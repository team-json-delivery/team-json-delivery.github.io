---
title: '내 근처에는 누가 살고 있을까? : 위치 기반 서비스 설계하기'
date: 2024-02-18T06:27:22+09:00
author: "njhyuk"
cover:
  image: "/images/nearby-service/cover.jpeg"
---

오늘은 스터디원 분들과  `가상 면접 사례로 배우는 대규모 시스템 설계 기초 2권` 의 주변친구 챕터를 스터디를 하며 나눈 내용을 바탕으로 포스팅을 해보려 해요.

면접에서 주변 친구 서비스를 설계해 보라고 하면 어떻게 할까요?

## 문제를 이해하고 설계 범위를 확정하자

면접 상황에서는 문제를 이해하고 기능의 스펙을 최소화 하는데 집중해야 합니다.

국내 서비스 수준이 아닌 글로벌 서비스 수준의 설계를 원하는지, 주변 친구의 범위가 어느 정도인지, 채팅과 같은 부가기능을 지원해야 되는지에 대해서도 논의해 보면 좋아요. 면접 상황에서는 가급적 핵심 기능에 집중하고 면접관과 이해도를 맞추도록 해요.

책에서 설명하는 면접관과 논의 예시는 다음과 같아요.

* Q : 주변 친구의 범위는 어느정도로 할까요?
* A : 5마일(mile) 입니다.
* Q : 사용자 사이에 강이 있을수 있는데 어떻게 하는게 좋을까요?
* A : 사용자 사이의 직선거리라고 가정하죠.
* Q : 사용자 수(DAU)는 어떻게 가정할까요?
* A : 10억명으로 가정하고 주변 친구 기능은 10%가 사용 하는것으로 가정하죠.
* Q : 이동이력은 보관할까요?
* A : 추후 머신러닝 데이터를 확보하기 위해 영구 보관이 필요해요.
* Q : 사용자가 10분이상 비활성화 상태면 주변 친구 목록에서 사라지게 할까요?
* A : 사라지게 합시다.
* Q : GDPR과 CCPA 같은 데이터 보호법을 고려할까요?
* A : 고려하지 않겠습니다.

GDPR, 즉 일반 데이터 보호 규정(General Data Protection Regulation)은 유럽연합(EU)의 데이터 보호와 개인정보의 자유로운 이동을 규제하는 법률이에요.
ISMS(Information Security Management System)가 정보 보안 관리 체계 전반에 초점을 맞춘다면, GDPR은 개인 데이터의 안전한 처리와 투명성을 강조하고 있어요.

💡 한 스터디원에 따르면, 사용자는 자신의 개인 데이터에 대한 이력을 요청할 수 있고, 기업은 해당 데이터를 제공하는 기능을 구현하는데 많은 어려움이 있었다고 해요.

### 기능 요구사항
* 사용자는 앱에서 주변 친구 확인 가능
* 해당 친구까지의 거리와 마지막으로 갱신된 시각 (timestamp) 표시
* 친구 목록은 N초마다 한번씩 갱신 되어야 함

### 비기능 요구사항
* 낮은 지연시간 (low latency)
* 안정성, 일부의 데이터 유실 용인
* 결과적 일관성 (eventually consistency)

### 개략적 규모 추정
* 총 사용자 10억명 중 DAU는 1억 명으로 가정
* 동시 사용 접속자 수는 1억명의 10%인 천만명으로 가정
* 사용자는 모두 30초마다 위치 갱신
* 1,000,000명/30초 = 334,000 QPS

## 개략적 설계안을 제시하자

### 개략적 설계안

먼저 백엔드의 역할을 정의하면 다음과 같아요.

* 모든 활성 상태 사용자의 위치 변화 내역을 수신하기
* 사용자 위치 변경 내역을 수신할 때마다, 관련 친구들에게 변경 내역을 갱신하기
* 특정 임계치보다 너무 먼 경우 변경내역 전하지 않기

단 위와같이 천만명의 유저의 위치 내역을 수신하도록 구현하면 30초마다 천만명의 유저를 처리해야 하기에 자체 DDOS를 발생시킬 수 있어 주의가 필요해요.

스터디원들끼리 서로 클라이언트로 인한 자체 DDOS 사례들을 공유했어요.
클라이언트가 API를 여러번 호출하는 버그, N시간 마다 동작하는 클라이언트의 배치로 인한 장애 경험 등을 공유해 주셨답니다! 😅

![](https://github.com/team-json-delivery/study-archive/blob/main/System-Design-Interview-vol2/02.%EC%A3%BC%EB%B3%80%20%EC%B9%9C%EA%B5%AC/devhwang/archi.png?raw=true)

####  웹소켓 서버
* 친구 위치 정보 변경을 거의 실시간에 가깝게 처리하는 유상태 서버 클러스터에요.
* 검색 반경 내 친구 위치가 변경되면, 웹소켓을 통해 클라이언트로 전송되어요.
* 앱을 킬때, 온라인 상태인 모든 주변 친구 위치가 웹소켓으로 해당 앱에 전송되어요.

#### 레디스 위치 정보 캐시
* 활성 상태인 사용자의 가장 최근 위치 정보를 캐시하는데 사용되어요.
* 레디스의 TTL 기능을 이용해 활성 상태 만료를 관리하고, 캐시 정보가 갱신될때 TTL이 갱신되면서 활성상태를 연장시키도록 구현해요.
* 위치 정보를 DB가 아닌 레디스에 저장하는 이유는, 주변친구 기능은 사용자의 “현재위치”만 친구들이 알수 있으면 되기 때문이에요.
  * 즉 위치 정보에 대해 영속성을 보장할 필요 없기 때문에 휘발될 수 있지만 성능이 좋은 레디스를 활용하는게 좋아요.

#### 사용자 데이터베이스
* 사용자 데이터와 친구 관계 정보를 저장해요.
* 복잡한 SNS 친구관계를 만든다면 GraphDB를 이용해야 할 수 있지만, 사용자 데이터 및 친구 관계는 1:N이라 가정하면 RDB와 NoSQL 어느쪽이든 사용 가능해요.
* RDB를 사용하기로 했다면, 한대의 DB로 감당할 수 없을 수 있기 때문에, 사용자 ID를 기준으로 데이터를 샤딩하는것을 고려 해야 해요.

#### 위치 이동 이력 데이터베이스
* 사용자의 위치 변동 이력을 보관해요.
* 해당 이력은 기능구현과 관련은 없지만, 추후 머신러닝 학습에 이용 될 수 있어요.
* 위치 이력 DB는 카산드라를 채택해요.
  * 막대한 쓰기 연산 부하를 감당할 수 있고, 수평적 규모 확장이 가능해요.
  * RDB는 쓰기보단 읽기 위주 서비스에 적합하고, 대량의 데이터엔 샤딩 전략이 고려되어야 해요.


#### 카산드라의 수평적 규모 확장은 무한할까? 🤔

한 스터디원의 경험에 따르면, AWS에 카산드라 N백대 클러스터가 있는상태로 수평적 규모 확장으로 버티던 서비스가 더이상 늘리기 어려운 시점이 오게되어 카산드라를 RDB 처럼 샤딩했던 경험이 있었고, [최종적 일관성](https://www.popit.kr/%EA%B2%B0%EA%B3%BC%EC%A0%81-%EC%9D%BC%EA%B4%80%EC%84%B1%EC%9D%B8%EA%B0%80-%EC%B5%9C%EC%A2%85%EC%A0%81-%EC%9D%BC%EA%B4%80%EC%84%B1%EC%9D%B8%EA%B0%80/) 으로 동작하기 때문에 주의가 필요하다고 해요.

#### 레디스 펍/섭 서버
* 웹소켓 서버를 통해 수신한 위치 정보 이벤트를 펍/섭 채널에 발행해요.
* 수신한 위치 정보를 사용자의 채널에 펍(발행) 하면 사용자의 친구들이 섭(구독) 하는 형태에요.

## 조금 더 설계해보자

### 무상태 서버 VS 유상태 서버
API 서버는 무상태 특성이 있기 때문에 CPU 사용률 또는 I/O 상태에 따라 오토 스케일링 할 수 있지만, 웹소켓 서버는 노드를 제거하기 전에 기존에 연결된 클라이언트들이 있어 유상태 특성이 있어요.

웹소켓 서버는 교체할때 로드밸런서에 연결 종료중 상태를 전달하고 새로운 커넥션은 맺지 않고, 모든 커넥션이 종료되면 노드에서 제거하는 전략을 고려해야해요.

### 레디스 펍/섭 서버
* 레디스 펍/섭 서버를 위치 변경 내역 메시지를 라우팅 하는 계층으로 활용했어요.
* 레디스 펍/섭 채널 비용이 아주 저렴하기 때문이에요.
* 구독자가 없는 채널로 전송된 메시지는 버려지는 특성이 있어 부담이 적어요.
* 구독자 관계를 추적하기 위한 자료 구조로 소량의 메모리만 사용하는 해시 테이블 또는 연결 리스트를 사용해요.

### 얼마나 많은 레디스 펍/섭 서버가 필요할까?

#### 메모리 사용량
* 채널의 수는 1억개 (10억 사용자의 10%)
* 해시 테이블과 연결 리스트는 20바이트로 가정한다면, 1억*20바이트=200GB 메모리
* 100GB의 메모리가 탑재된 서버라면 2대가 필요해요.

#### CPU 사용량
* 위치 정보 업데이트 양은 초당 1400만건
* 서버 한대로 감당 가능한 수는 십만건이라고 가정 한다면, 140대의 서버가 필요해요.

본 설계안의 결론은 병목은 메모리가 아닌 CPU 사용량으로 보이기에 분산 레디스 펍/섭 클러스터 도입이 필요하다는 점을 알 수 있어요.


#### 이론적 계산으로 서버 대수를 선정하는게 가능할까? 🤔

개인적인 경험으로는, 목표 TPS를 산정하고 해당 TPS를 견딜수 있는지 nGrinder 같은 성능테스트 툴로 반복적으로 성능테스트 시도와 실패 결과를 통해 최종 서버의 스펙을 산정하는 경우가 많았어요.

대다수의 스터디원들도 이에 공감했지만, 카프카 클러스터의 스펙을 선정할때 [제로 카피\(Zero-Copy\)](https://soft.plusblog.co.kr/7) 의 특성 덕분에 책과 같이 계산만으로 스펙을 선정한 경험을 공유 주셨어요.

성능테스트 기반으로 서버 스펙을 산정하는 방식은 [신규 서비스 배포 전에 실험과 개선을 반복한 이야기](https://helloworld.kurly.com/blog/vsms-performance-experiment/) 포스팅을 추천해요.

### 분산 레디스 펍/섭 서버 클러스터
* 서버에는 필연적으로 장애가 생기기 마련이에요.
* etcd나 주키퍼 같은 서비스 탐색 컴포넌트 도입으로 문제를 해결 할 수 있어요.
* 예를들어 [p_1, p_2, p_3, p_4] 와 같이 가용한 서버 목록을 유지하는 해시링 키-값 저장소를 구현해야 해요.

#### 레디스 펍/섭 클러스터의 확장성
* 데이터는 채널의 모든 구독자에게 전송되고 나면 바로 삭제되기에 무상태 특성이 있지만, 펍/섭 서버는 채널에 대한 상태 정보 보관를 보관하기에 유상태 특성이 있어요.
* 펍/섭 서버를 교체하거나 제거하는 경우 채널은 다른 서버로 이동시켜야 하고, 해당 채널의 모든 구독자에게 알려야해요.
* 이러한 유상태 서버 클러스터는 늘리고 줄이는데 부담되기에 오버 프로비지닝 해 두는것이 좋아요.


#### 펍/섭 구조가 항상 옳은걸까?  🤔

상품 DB 엑셀 업로드를 레디스 펍/섭을 통해 구현한 경험이 있었어요.

데이터가 하나의 엑셀 파일을 공유 해야하는 유상태라는 특성이 있어 후처리 서버 확장을 하는데 제약이 있었던 경험이 있어 펍/섭 구조가 유상태일땐 여러 고민이 필요하다 느꼈죠.


## 마무리

이렇게 책에서 배운 '주변친구' 기능의 설계 방법과 스터디원들의 실제 경험을 결합해보니, 대규모 시스템 설계의 복잡함을 더 잘 이해할 수 있었어요. 또한 핵심 컴포넌트 3가지를 사용할 수 있다는 점을 배우게 되었답니다.

* **웹소켓** : 클라이언트와 서버 사이의 실시간 통신을 위해 사용
* **레디스**는 위치 데이터의 빠른 읽기/쓰기를 위해 사용
* **레디스 펍/섭** : 메시지 라우팅 계층으로써 모든 온라인 친구에게 위치변경 내역을 전달하는데 사용

여러분도 대규모 시스템 설계에 대한 이해를 넓히는 데 이 글이 도움이 되었으면 해요.

다음 포스팅에서 더 재미있는 내용으로 찾아뵐게요!