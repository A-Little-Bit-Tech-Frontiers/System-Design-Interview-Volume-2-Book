# 분산 메시지 큐

<img width="652" alt="스크린샷 2024-06-30 오후 3 50 30" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/e34f9a30-f89b-4afb-a303-ccf62164b38f">

- 생산자는 메시지를 메시지 큐에 발행
- 소비자는 큐를 구독하고 구독한 메시지를 소비
- 메시지 큐는 생산자와 소비자 사이의 결합을 느슨하게 하는 서비스로, 생산자와 소비자늬 독립적인 운영 및 규모 확장을 가능하게 하는 역할

## 메시지 모델

### 일대일 모델

큐에 전송된 메시지는 오직 한 소비자만 가져갈 수 있다. <br>
어떤 소비자가 메시지를 가져갔다는 사실을 큐에 알리면 해당 메시지는 큐에서 삭제된다.

<img width="604" alt="스크린샷 2024-06-30 오후 3 52 05" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/9cc703e4-db70-49eb-82d5-ad38b6d6df65">

### 발행-구독 모델

메시지를 보내고 받을때 토픽을 이용한다. <br>
토픽에 전달된 메시지는 해당 토픽을 구독하는 모든 소비자에 전달된다.

<img width="624" alt="스크린샷 2024-06-30 오후 3 53 29" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/dc02540b-bb53-4434-a5bf-3c1fa752e0f6">

<br>

## 데이터 저장소

메시지를 어떻게 지속적으로 저장할지에 대한 내용이다. <br>
메시지 큐의 트래픽 패턴은 아래와 같다.

- 읽기/쓰기가 빈번하게 일어난다.
- 갱신/삭제 연산은 발생하지 않는다.
- 순차적인 읽기/쓰기가 대부분이다.

### 선택지 1: 데이터베이스

데이터베이스라면 데이터 저장 요구사항을 맞출 수는 있다. <br>
하지만 이상적인 방법일 수는 없는데, 읽기/쓰기 연산이 동시에 대규모로 빈번하게 발생하는 상황을 잘 처리하는 데이터베이스는 설계하기 어렵다.

### 선택지 2: 쓰기 우선 로그 (Write-Ahead Log, WAL)

WAL은 새로운 항목이 추가되기만 하는 일반 파일이다. <br>
지속성을 보장해야 하는 메시지는 디스크에 WAL로 보관하는 것을 추천한다. <br>
WAL에 대한 접근 패턴은 읽기/쓰기 전부 순차적이고, 접근 패턴이 순차적일때 디스크는 큰 용량을 저렴한 가격에 제공한다.

<img width="664" alt="스크린샷 2024-06-30 오후 3 58 29" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/d859d976-883d-42c4-a6a5-3f830c9a8a8b">

<br>


## 메시지 자료구조

메시지 자료구조는 생산자, 메시지 큐, 소비자 사이의 계약이다. <br>
아래는 메시지 자료구조의 스키마 사례이다.

<img width="284" alt="스크린샷 2024-06-30 오후 3 59 47" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/aca5cf6e-26eb-41ff-975b-cbfe5f5d9956">

<br>

## 소비자 재조정 (consumer rebalancing)

소비자 재조정은 어떤 소비자가 어떤 파티션을 책임지는지 다시 정하는 프로세스이다. <br>
새로운 소비자가 합류하거나, 기존 소비자가 그룹을 떠나거나, 어떤 소비자에 장애가 발생하거나, 파티션들이 재조정되는 경우 시작될 수 있다.

이 절차에 코디네이터가 중요한 역할을 하고, 코디네이터는 소비자 재조정을 위해 소비자들과 통신하는 브로커 노드다.

<img width="680" alt="스크린샷 2024-06-30 오후 4 02 35" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/15b37821-5456-46b8-bdb9-b269a2702063">

<br>

## 상태 저장소

메시지 큐 브로커의 상태 저장소에는 다음과 같은 정보가 저장된다.

- 소비자에 대한 파티션의 배치 관계
- 각 소비자 그룹이 각 파티션에서 마지막으로 가져간 메시지의 오프셋

<img width="556" alt="스크린샷 2024-06-30 오후 4 06 45" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/4f8806a4-2b2e-46f3-a8af-870ea50d79fc">

> 소비자 상태 정보 데이터가 이용되는 패턴은 다음과 같다.
> - 읽기와 쓰기가 빈번하게 발생하지만 양은 많지 않다.
> - 데이터 갱신은 빈번하게 일어나지만 삭제되는 일은 거의 없다.
> - 읽기와 쓰기 연산은 무작위적 패턴을 보인다.
> - 데이터의 일관성이 중요하다.

<br>

## 메타데이터 저장소

토픽 설정이나 속성 정보를 보관한다. <br>
파티션 수, 메시지 보관 기간, 사본 배치 정보 등이 해당한다. <br>
자주 변경되지 않으며 양도 적다. 하지만 높은 일관성을 요구한다.

<br>

## 복제

짙은 색은 해당 파티션의 리더이고, 나머지는 단순 사본이다. <br>
생산자는 파티션에 메시지를 보낼때 리더에게만 보내고, 다른 사본은 리더에서 새 메시지를 지속적으로 가져와 동기화한다. <br>
메시지를 완전히 동기화한 사본의 개수가 지정된 임계값을 넘으면 리더는 생산자에게 메시지를 잘 받았다는 응답을 보낸다

<img width="725" alt="스크린샷 2024-06-30 오후 4 10 40" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/0b209fa4-5fee-4766-ab86-4c930c0bc499">








