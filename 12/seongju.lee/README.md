# 전자 지갑

## 1. 분산 트랜잭션

<img width="656" alt="image" src="https://github.com/user-attachments/assets/1b550961-60b1-41a6-bf68-d552f48fd44a">

위 상황은 '클라이언트 A에서 클라이언트 B로 1달러를 이체'하라는 명령을 받은 상황이다. 즉, 두 업데이트 연산이 하나의 원자적 트랜잭션으로 실행되어야만 하는 상황이다.  
위와 같이 서로 다른 두 개 저장소 노드를 갱신하는 연산을 원자적으로 수행하려면 어떻게 해야 할까?  

우선, 인메모리 기반의 레디스 노드를 트랜잭션을 지원하는 관계형 데이터베이스 노드로 교체하는 것이다.  
<img width="456" alt="image" src="https://github.com/user-attachments/assets/15af5cf5-cd65-4bea-af14-b1d4107e58e4">

각 업데이트는 원자성 보장이 되었다고 하자. 이제는 첫 번째 계정과 두 번째 계정의 각 업데이트를 하나의 트랜잭션으로 묶어야 한다.

<br>

## 2. 분산 트랜잭션: 2단계 커밋(2PC: Two Phase Commit)

<img width="698" alt="image" src="https://github.com/user-attachments/assets/9349d83e-160e-4469-9c31-f29c4b0ac7af">

**데이터베이스 자체에 의존하는 방안**  

1. 조정자(지갑 서비스)는 정상적으로 여러 데이터베이스에 읽기 및 쓰기 작업을 수행 -> 그 결과로 데이터베이스 A와 C에는 락이 걸림
2. 애플리케이션이 트랜잭션을 커밋하려 할 때 조정자는 모든 데이터베이스에 트랜잭션 준비를 요청
3. 두번째 단계에서 조정자는 모든 데이터베이스의 응답을 받아 다음 절차를 수행
    1. 모든 데이터베이스가 ‘예’라고 응답하면 조정자는 모든 데이터베이스에 해당 트랜잭션 커밋을 요청
    2. 어느 한 데이터베이스라도 ‘아니요’를 응답하면 조정자는 모든 데이터베이스에 트랜잭션 중단을 요청
  

이 방안에서 준비 단계를 실행하려면 데이터베이스 트랜잭션 실행 방식을 변경해야 한다. 
2PC의 가장 큰 문제점은 **다른 노드의 메시지를 기다리는 동안 락이 오랫동안 잠긴 상태로 남을 수 있어**서 성능이 좋지 않다는 것이다. 또 다른 문제는 아래 그림과 같이 **조정자가 SPOF**가 될 수 있다는 것이다.  

<br>

## 3. 분산 트랜잭션: TC/C

TC/C(시도-확정/취소, Try-Confirm/Cancel)는 **두 단계로 구성된 보상 트랜잭션**이다. 

1. 조정자는 모든 데이터베이스에 트랜잭션에 필요한 자원 예약을 요청  
2. 조정자는 모든 데이터베이스로부터 회신을 받음
    1. 모두 ‘예’라고 응답하면 조정자는 모든 데이터베이스에 작업 확인을 요청하는데, 이것이 바로 `Try-Confirm` 절차
    2. 어느 하나라도 ‘아니요’라고 응답하면 조정자는 모든 데이터베이스에 작업 취소를 요청하며, 이것이 바로 `Try-Cancel` 절차
  
**2PC의 두 단계는 한 트랜잭션이지만, TC/C에서는 각 단계가 별도 트랜잭션**

- 시도 - 확정
  
  <img width="570" alt="image" src="https://github.com/user-attachments/assets/fea3be2b-8899-4aad-a646-53cfc2ffec28">

- 시도 - 실패
  
  <img width="573" alt="image" src="https://github.com/user-attachments/assets/d2e8e70d-8af5-4c84-8e93-33621818a4e8">

  **시도 단계의 트랜잭션에서 계정 A의 잔액은 이미 바뀌었고 트랜잭션은 종료되었다. 이미 종료된 트랜잭션의 효과를 되돌리려면 지갑 서비스는 또 다른 트랜잭션을 시작하여 계정 A에 1달러를 다시 추가해야 한다.**
  시도 단계에서 계정 C의 잔액은 업데이트하지 않았으므로, 계정 C의 데이터베이스에는 NOP 명령을 전송하기만 하면 된다.


- TC/C는 여러 개의 독립적인 로컬 트랜잭션으로 구성
- TC/C의 실행 주체는 애플리케이션이며, 애플리케이션은 이런 독립적 로컬 트랜잭션이 만드는 중간 결과를 볼 수 있음
- 반면, 데이터베이스 트랜잭션이나 2PC같은 분산 트랜잭션의 경우 실행 주체는 데이터베이스이며 애플리케이션은 그 중간 실행 결과를 알 수 없다.


### 3.1 잘못된 순서로 실행된 경우

<img width="698" alt="image" src="https://github.com/user-attachments/assets/c7f851f3-ac76-4ea5-bf76-ef11b63c9fe7">  

위 그림은 시도 단계에서 계정 A에 대한 작업이 실패하여 지갑 서비스에 실패를 반환한 다음 취소 단계로 진입하여 계정 A와 계정 C 모두에 취소 명령을 전송하는 과정을 보여준다.  
이때 계정 C를 관리하는 데이터베이스에 네트워크 문제가 있어서 시도 명령 전에 취소 명령부터 받게 되었다고 해보자. 그 시점에는 취소할 것이 없는 상태  

순서가 바뀌어 도착하는 명령도 처리할 수 있도록 하려면 기존 로직을 다음과 같이 수정
- 취소 명령이 먼저 도착하면 데이터베이스에 아직 상응하는 시도 명령을 못보았음을 나타내는 플래그를 참으로 설정하여 저장
- 시도 명령이 도착하면 항상 먼저 도착한 취소 명령이 있었는지 확인 -> 있었으면 바로 실패를 반환

<br>

## 4. 분산 트랜잭션: 사가

선형적 명령 수행

사가는 유명한 분산 트랜잭션 솔루션 가운데 하나로 마이크로서비스 아키텍처에서는 사실상 표준이다. 

1. 모든 연산은 순서대로 정렬된다. 각 연산은 자기 데이터베이스에 독립 트랜잭션으로 실행된다.
2. 연산은 첫 번째부터 마지막까지 순서대로 실행된다. 한 연산이 완료되면 다음 연산이 개시된다.
3. 연산이 실패하면 전체 프로세스는 실패한 연산부터 맨 처음 연산까지 역순으로 보상 트랜잭션을 통해 롤백된다. 따라서 n개의 연산을 실행하는 분산 트랜잭션은, 보상 트랜잭션을 위한 n개 연산까지 총 2n개의 연산을 준비해야 한다.

<img width="472" alt="image" src="https://github.com/user-attachments/assets/1419a9e0-e172-4cdb-a363-9822b100be90">  

|  | TC/C | 사가 |
| --- | --- | --- |
| 보상 트랜잭션 실행 | 취소 단계에서 | 롤백 단계에서 |
| 중앙 조정 | 예 | 예(오케스트레이션) |
| 작업 실행 순서 | 임의  | 선형 |
| 병렬 실행 가능성 | 예 | 아니요(선형적 실행) |
| 일시적으로 일관되지 않은 상태 허용 | 예 | 예 |
| 구현 계층: 애플리케이션 또는 데이터베이스 | 애플리케이션 | 애플리케이션 |

지연 시간(latency) 요구사항에 따라 다르다. 위 표에서 봤듯이 **사가의 연산은 순서대로 실행**해야 하지만 **TC/C에서는 병렬로 실행할 수 있다.** 

1. 지연 시간 요구사항이 없거나 앞서 살펴본 송금 사례처럼 서비스 수가 매우 적다면 아무것이나 사용하면 됨
2. 지연 시간에 민감하고 많은 서비스/운영이 관계된 시스템이라면 TC/C가 더 나음

> 개인적으로는 작업 실행 순서가 선형적인지 여부가 가장 큰 차이점이 아닐까 싶다.
결국 TC/C는 조정자가 일괄적으로 작업 실행을 각 트랜잭션을 통해 지시한다. 여기서 바로 작업 실행 순서가 보장이 안되는 문제가 발생하는 것인다.
이에 따라서 실패 시, 보상 트랜잭션을 어떻게 주는지도 flag를 통해 구분해야 하므로 기본적인 복잡도가 올라가는 것 같다고 생각하기 때문이다.
——
또한, Try, Confirm, Cancel 상태가 명시적으로 관리되어야 한다는 점도 관리의 복잡도를 너무 높이는 것 같다.

<br>

## 5. 이벤트 소싱

### 5.1 명령

명령은 외부에서 전달된, 의도가 명확한 요청이다. 예를 들어 고객 A에서 C로 $1를 이체하라는 요청은 명령이다.  
이벤트 소싱에서 순서는 아주 중요하다. 따라서 명령은 일반적으로 FIFO 큐에 저장된다.

### 5.2 이벤트

명령은 의도가 명확하지만 사실은 아니기 때문에 유효하지 않을 수도 있다. 유효하지 않은 명령은 실패할 수 있다. 가령 이체 후에 잔액이 음수가 된다면 이체는 실패한다.  
**작업 이행 전에는 반드시 명령의 유효성을 검사**해야 한다. 그리고 검사를 통과한 명령은 반드시 이행(fulfill)되어야 한다. **명령 이행 결과를 이벤트라고 부른다.**

명령과 이벤트 사이에는 두 가지 중요한 차이점이 있다.

1. **이벤트는 검증된 사실로, 실행이 끝난 상태다. 그래서 이벤트에 대해 이야기할 때는 과거 시제를 사용**한다. 따라서 명령이 “A에서 C로 $1 송금”인 경우, 해당 이벤트는 “A에서 C로  $1 **송금을 완료하였음**”이 된다.  
2. 명령에는 무작위성이나 I/O가 포함될 수 있지만 이벤트는 결정론적이다. 이벤트는 과거에 실제로 있었던 일이다.


### 5.3 재현성

이벤트 소싱이 다른 아키텍처에 비해 갖는 가장 중요한 장점은 **재현성(reproducibility)** 이다.  
앞서 언급한 **분산 트랜잭션 방안의 경우 지갑 서비스는 갱신한 계정 잔액(즉, 상태)을 데이터베이스에 저장한다.  
계정 잔액이 변경된 이유는 알기가 어렵다.** 또한 **한번 업데이트가 이루어지고 나면 과거 잔액이 얼마였는지는 알 수 없다.** 데이터베이스는 특정 시점의 잔액이 얼마인지만 보여준다.  
하지만 이벤트를 처음부터 다시 재생하면 과거 잔액 상태는 언제든 재구성 할 수 있다. 
**이벤트 리스트는 불변(immutable)이고 상태 기계 로직은 결정론적이므로 이벤트 이력을 재생하여 만들어낸 상태는 언제나 동일**하다. 아래 그림은 이벤트를 재생하여 지갑 서비스 상태를 재현하는 과정의 사례다.

<img width="498" alt="image" src="https://github.com/user-attachments/assets/9e9bcd9c-d796-4d84-bcd2-392c6dbec9a4">  

> 이벤트가 불변이자 (검증이 완료된)단일 진실 공급원이라는 말을 가장 적절하게 사용하는 예시가 아닌가 싶다.


### 5.4 CQRS

<img width="564" alt="image" src="https://github.com/user-attachments/assets/21a54689-8478-4b6f-b3f5-569c0fe609de">

이벤트 소싱에서의 이벤트는 주로 쓰기 작업에 사용된다. 따라서, 조회를 하는 과정에서 이벤트를 재구성하여 조회하는 것은 매우 비효율적이다.  
그렇기 때문에 상태 기록을 하는 상태 기계를 하나 두고, 이벤트를 해당 기계에도 발행한다. 그러면, 해당 기계가 저장하는 상태 이력 데이터베이스에 질의하여 조회 성능을 높일 수 있다.
