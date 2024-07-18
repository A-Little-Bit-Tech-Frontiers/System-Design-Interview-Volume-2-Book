# 호텔 예약 시스템

## 개략적 설계안

### 호텔 관련 API

| API                  | 설명                     |
|----------------------|------------------------|
| GET /v1/hotels/id    | 호텔의 상세 정보 반환           |
| POST /v1/hotels      | 신규 호텔 추가. 호텔 직원만 사용 가능 |
| PUT /v1/hotels/id    | 호텔 정보 갱신. 호텔 직원만 사용 가능 |
| DELETE /v1/hotels/id | 호텔 정보 삭제. 호텔 직원만 사용 가능 |


### 객신 관련 API

| API                           | 설명                     |
|-------------------------------|------------------------|
| GET /v1/hotels/id/rooms/id    | 객실 상세 정보 반환            |
| POST /v1/hotles/id/rooms      | 신규 객실 추가. 호텔 직원만 사용 가능 |
| PUT /v1/hotels/id/rooms/id    | 객실 정보 갱신. 호텔 직원만 사용 가능 |
| DELETE /v1/hotels/id/rooms/id | 객실 정보 삭제. 호텔 직원만 사용 가능 |

### 예약 관련 API

| API                        | 설명                |
|----------------------------|-------------------|
| GET /v1/reservations       | 로그인 사용자의 예약 이력 반환 |
| GET /v1/reservations/id    | 특정 예약의 상세 정보 반환   |
| POST /v1/reservations      | 신규 예약             |
| DELETE /v1/reservations/id | 예약 취소             |

### 데이터 모델

<img width="622" alt="스크린샷 2024-07-14 오후 2 06 05" src="https://github.com/user-attachments/assets/72ea1b75-09bd-4384-b18c-007bd4fbafa6">
<img width="499" alt="스크린샷 2024-07-14 오후 2 06 13" src="https://github.com/user-attachments/assets/2f66c5cc-a1bc-49a0-a574-1c566548cf0c">

### 개략적 설계안

<img width="694" alt="스크린샷 2024-07-14 오후 2 06 30" src="https://github.com/user-attachments/assets/155ad9bd-3753-4d57-b95e-1ddce82582ec">


## 상세 설계

### 개선된 데이터 모델

호텔 객실을 예약할 때는 특정 객실이 아니라 특정한 객실 유형을 예약하게 된다. <br>
이 요구사항을 수용하려면 API와 스키마의 어떤 부분을 변경해야 할까 ? <br>
예약 API의 호출 인자인 roomId는 roomTypeId로 변경한다.

<img width="610" alt="스크린샷 2024-07-14 오후 2 10 03" src="https://github.com/user-attachments/assets/67f27e2b-7d33-4e90-b91e-9eb566cf5f09">

- room: 객실에 관계된 정보를 담는다.
- room_type_rate: 특정 객실 유형의 특정 일자 요금 정보를 담는다.
- reservation: 투숙객 예약 정보를 담는다.
- room_type_inventory: 호텔의 모든 객실 유형을 담는 테이블
  - hotel_id: 호텔 식별자
  - room_type_id: 객실 유형 식별자
  - date: 일자
  - total_inventory: 총 객실 수에서 일시적으로 제외한 객실 수를 뺀 값. 일부 객실은 유지보수를 위해 예약 가능 목록에서 빼 둘 수 있어야 한다.
  - total_reserved: 지정된 hotel_id, room_type_id, date에 예약된 모든 객실의 수

> 예약 데이터가 단일 데이터베이스에 담기에 너무 크다면 다음과 같은 방안을 생각해 볼 수 있다.
> 1. 현재 및 향후 예약 데이터만 저장한다. 예약 이력은 자주 접근하지 않으므로 아카이빙하거나 cold storage로 옮길 수도 있다.
> 2. 데이터베이스를 샤딩한다.

### 동시성 문제

#### 같은 예약자가 예약 버튼을 두 번 누를 수 있다.

클라이언트 측은 요청을 전송하고 난 후 예약 버튼을 회색으로 표시하거나, 숨기거나, 비활성화한다. <br>
서버 측은 예약 API 요청에 멱등키를 추가한다. reservation_id를 멱등 키로 사용하여 이중 예약 문제를 해결한다.

#### 여러 사용자가 같은 객실을 동시에 예약하려 할 수 있다.

<img width="672" alt="스크린샷 2024-07-14 오후 2 17 40" src="https://github.com/user-attachments/assets/0b1ad3ac-8af4-4d4d-9c62-affa3b7d7aee">

이 문제를 해결하려면 락을 활용해야 한다.

- 방안 1: 비관적 락
  - 장점
    - 구현이 쉽고 모든 갱신 연산을 직렬화하여 충돌을 막는다.
  - 단점
    - 여러 레코드에 락을 걸면 데드락이 발생할 수 있다.
    - 확장성이 낮다. 트랜잭션의 수명이 길거나 많은 엔티티에 관련된 경우, 데이터베이스 성능에 영향을 끼친다.
- 방안 2: 낙관적 락
  - 장점
    - 데이터베이스 자원에 락을 걸 필요가 없다. 버전 번호를 통해 데이터 일관성을 유지할 책임은 애플리케이션에 있다.
    - 데이터에 대한 경쟁이 치열하지 않은 상황에 적합하다. 락을 관리하는 비용 없이 트랜잭션을 실행할 수 있다.
  - 단점
    - 데이터에 대한 경쟁이 치열한 상황에서는 성능이 좋지 못하다.
- 방안 3: 데이터베이스 제약 조건
  - 장점
    - 구현이 쉽다.
    - 데이터에 대한 경쟁이 심하지 않을 때 잘 동작한다.
  - 단점
    - 데이터에 대한 경쟁이 심하면 실패하는 연산 수가 엄청나게 늘어난다.
    - 데이터베이스 제약 조건은 애플리케이션 코드와 달라서 버전을 통제하기 어렵다.

### 시스템 규모 확장

#### 데이터베이스 샤딩

<img width="692" alt="스크린샷 2024-07-14 오후 2 25 05" src="https://github.com/user-attachments/assets/3a1314a4-bcfa-45a0-909e-a3acc27a3f6e">


#### 캐시

호텔 잔여 객실 데이터는 오직 현재와 미래의 데이터만이 중요하다. 고객이 과거의 어떤 객실을 예약하려 하지는 않기 때문이다. <br>
따라서 데이터를 보관할 때 낡은 데이터는 자동으로 소멸되도록 TTL을 설정할 수 있다면 바람직하다. <br>
이런 상황에서 레디스의 TTL과 LRU(Least Recently Used) 캐시 교체 정책을 사용하여 메모리를 최적으로 활용할 수 있다.

데이터베이스 앞에 캐시 계층을 두고 잔여 객실 확인 및 객실 예약 로직이 해당 계층에서 실행되도록 할 수 있다. <br>
요청 가운데 일부만 잔여 객실 데이터베이스가 처리하고, 나머지는 캐시가 담당한다.

- 장점
  - 읽기는 캐시가 처리하므로 데이터베이스의 부하가 크게 줄어든다.
  - 읽기를 메모리에서 실행하므로 높은 성능을 보장한다.
- 단점
  - 데이터베이스와 캐시 사이의 데이터 일관성을 유지하는 것은 어려운 문제다.

### 서비스 간 데이터 일관성

<img width="782" alt="스크린샷 2024-07-14 오후 2 31 45" src="https://github.com/user-attachments/assets/93d54563-c5f2-4d5d-97bc-1b5e7ef4d798">
<img width="546" alt="스크린샷 2024-07-14 오후 2 33 32" src="https://github.com/user-attachments/assets/680e110a-fb19-43ec-b7f2-64d9a6afcaae">

마이크로 서비스 환경에서는 하나의 트랜잭션으로 데이터 일관성을 보증하는 기법을 사용할 수 없다. <br>
데이터 일관성 문제를 해결하기 위해 아래와 같은 방법을 사용할 수 있다.

- 2PC commit: 모든 노드가 성공하든 아니면 실패하든 둘 중 하나로 트랜잭션이 마무리되도록 보증한다. 어느 한 노드에 장애가 발생하면 해당 장애가 복구될 때까지 진행이 중단된다. 성능이 뛰어나진 않다.
- SAGA: 각 노드에 국지적으로 발생하는 트랜잭션을 하나로 엮는 것. 각각의 트랜잭션이 완료되면 다음 트랜잭션을 시작하는 트리거로 쓰일 메시지를 만들어 보낸다. 어느 한 트랜잭션이라도 실패하면 사가는 그 이전 트랜잭션의 결과를 전부 되돌리는 트랜잭션들을 순차적으로 싱핸한다.












