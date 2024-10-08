# 증권 거래소

## 개략적 설계안

<img width="777" alt="스크린샷 2024-08-15 오후 7 00 10" src="https://github.com/user-attachments/assets/fac6e988-fa40-42bb-8fe5-aefeb7b3dc27">

1. 고객이 브로커의 웹/모바일을 통해 주문한다.
2. 브로커가 주문을 거래소에 전송한다.
3. 주문이 클라이언트 게이트웨이를 통해 거래소로 들어간다. 클라이언트 게이트웨이는 입력 유효성 검사, 속도 제한, 인증, 정규화 같은 기능울 수행한다.
4. 설정한 규칙에 따라 위험성 점검을 수행한다.
5. 위험성 점검 과정을 통과한 주문에 대해, 주문 관리자는 지갑에 주문 처리 자금이 충분한지 확인한다.
6. 주문이 체결 엔진으로 전송된다. 체결 가능 주문이 발견되면 체결 엔진은 매수 측과 매도 측에 각각 하나씩 두 개의 집행 기록을 생성한다. 나중에 그 과정을 재생할때 항상 결정론적으로 동일한 결과가 나오도록 보장하기 위해 시퀀서는 주문 및 집행 기록을 일정 순서로 정렬한다.
7. 주문 집행 사실을 클라이언트에 전송한다.

### 체결 엔진

체결 엔진은 다음과 같은 역할을 한다.

1. 각 주식 심볼에 대한 주문서, 호가창을 유지 관리한다.
2. 매수/매도 주문을 연결한다. 주문 체결 결과로 두 개의 집행 기록이 만들어진다. (매수 1건, 매도 1건)
3. 집행 기록 스트림을 시장 데이터로 배포한다.

### 시퀀서

시퀀서는 체결 엔진에 주문을 전달하기 전에 **순서 ID**를 붙여 보낸다. <br>
또한 체결 엔진이 처리를 끝낸 모든 집행 기록 쌍에도 순서 ID를 붙인다. <br>
시퀀서에는 입력/출력 두 가지 시퀀서가 있으며, 각각 고유한 순서를 유지한다.

<img width="462" alt="스크린샷 2024-08-15 오후 7 06 48" src="https://github.com/user-attachments/assets/7671353e-715b-4948-bb31-068adeb5569f">


### 주문 관리자

주문 관리자는 클라이언트 게이트웨이를 통해 주문을 수신하고 다음을 실행한다.

- 종합정 위험 점검 담당 컴포넌트에 주문을 보내 위험성을 검토한다. 
- 사용자의 지갑에 거래를 처리하기에 충분한 자금이 있는지 확인한다.
- 주문을 시퀀서에 전달한다. 시퀀서는 해당 주문에 순서 ID를 찍고 체결 엔진에 보내어 처리한다.

### 시장 데이터 흐름

시장 데이터 게시 서비스는 체결 엔진에서 집행 기록을 수신하고 집행 기록 스트림에서 호가 창과 봉 차트를 만들어낸다. <br>
시장 데이터는 데이터 서비스로 전송되어 해당 서비스의 구독자가 사용할 수 있게 된다.

### 호가 창

매수/매도 주문 목록으로 가격 수준별로 정리되어 있다. 호가창의 자료구조는 다음 요구사항을 만족해야 한다.

- 일정한 조회 시간
- 빠른 추가/취소/실행 속도 (가급적 O(1) 시간복잡도를 만족해야 한다)
- 빠른 업데이트
- 최고 매수 호가/최저 매도 호가 질의
- 가격 수준 순회

<img width="530" alt="스크린샷 2024-08-15 오후 7 13 09" src="https://github.com/user-attachments/assets/c305f775-6074-471e-bcbc-a591b5e856db">


<br>

다음 코드는 호가 창이 실제로 어떻게 구현되는지 보여준다.

```java
class PriceLevel {
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

이 코드는 일반 연결 리스트를 사용하므로 지정가 주문을 추가/취소하는 시간 복잡도가 O(1)이 아니다. (private List<Order> orders) <br>
보다 효율적인 호가 창을 만드려면 orders의 자료구조는 이중 연결 리스트로 변경하여 모든 삭제 연산이 O(1)이 되도록 해야 한다.

1. 새 주문을 넣는것은 PriceLevel 리스트 마지막에 새 Order를 추가하는 것이다. 이중 연결 리스트에서 이 연산은 O(1)이다.
2. 주문을 체결하는 것은 PriceLevel 리스트의 맨 앞에 있는 Order를 삭제하는 것이다. 이중 연결 리스트는 시간 복잡도 O(1)이다.
3. 주문을 취소한다는 것은 호가창, OrderBook에서 Order를 삭제한다는 것이다. OrderBook에 있는 Map<OrderID, Order> orderMap을 활용하여 O(1) 시간 내에 취소할 주문을 찾을 수 있다.

<img width="842" alt="스크린샷 2024-08-15 오후 7 23 47" src="https://github.com/user-attachments/assets/7adb8082-fe60-4b90-84f1-cd06beb1917d">

<br>

## 상세 설계

### 성능

고성능을 위해 거래소는 주로 네트워크 및 디스크 액세스 지연 시간을 줄이거나 없애는 방안을 통해 중요 경로의 e2e 지연시간을 줄였다. <br>
오랜 시간동안 검증된 설계안은, 모든 것을 동일한 서버에 배치해여 네트워크를 통하는 구간을 없애는 것이다. <br>
같은 서버 내 컴포넌트 간 통신은 이벤트 저장소인 mmap를 통한다.

<img width="466" alt="스크린샷 2024-08-15 오후 7 35 58" src="https://github.com/user-attachments/assets/15b8697a-9838-4c84-9c34-7e671e040791">

CPU 효율성을 극대화하기 위해 애플리케이션 루프는 단일 스레드이며, 특정 CPU 코어에 고정시킨다. <br>
즉, 컨텍스트 스위치를 없애고 상태를 업데이트하는 스레드가 하나라서 락을 사용할 필요가 없다.

### 고가용성

클라이언트 게이트웨이같은 stateless 서비스는 쉽게 scale-out이 가능하다. <br>
하지만 주문 관리자나 체결 엔진처럼 상태를 저장하는 컴포넌트는 사본 간에 상태 데이터를 복사할 수 있어야한다.

주 체결 엔진은 주 인스턴스이고, 부 체결 엔진은 동일한 이벤트를 수신하고 처리하지만 이벤트 저장소로 이벤트를 전송하지는 않는다. <br>
주 인스턴스가 다운되면 부 인스턴스는 즉시 주 인스턴스 지위를 승계한 후 이벤트를 전송한다.

<img width="443" alt="스크린샷 2024-08-15 오후 7 39 57" src="https://github.com/user-attachments/assets/b6a9a3f3-28a4-4076-a65f-f1a327680b0d">










