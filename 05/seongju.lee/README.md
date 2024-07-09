## 1. 데이터 모델
지표 데이터는 통상 시계열 데이터 형태로 기록한다. 값 집합에 타임스탬프가 붙은 형태로 기록한다는 뜻이다. 시계열 각각에는 고유한 이름이 붙고, 선택적으로 레이블을 붙이기도 한다.

<br>

## 2. 데이터 접근 패턴

<img width="652" alt="image" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/67941526/e8d5f011-5837-4c7e-bc73-e396ed84992f">  

이 시스템은 언제나 많은 양의 쓰기 연산 부하를 감당해야 하지만, 읽기 연산의 부하는 잠시 급증하였다가 곧 사라지곤 한다.  
이러한 패턴을 바탕으로 어떤 데이터 저장소를 선택할 수 있다.  

----

<br>

## 3. 지표 수집

### 3.1 풀 모델  

<img width="665" alt="image" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/67941526/8a2ab057-a79e-44be-98b1-df5ad9d26632">

지표 수집기는 데이터를 가져올 서비스 목록을 알아야 한다.  
‘지표 수집기’ 서버 안에 모든 서비스 엔드포인트의 DNS/IP 정보를 담은 파일을 두면 가장 간단하다. 하지만 이 방안은 서버가 수시로 추가/삭제되는 대규모 운영 환경에는 적용하기 어렵다.  
etcd나 아파치 주키퍼 같은 서비스 디스커버리 기술을 활용하면 이 문제는 해결할 수 있다.   
다시 말해 각 서비스는 자신의 가용성(availability) 관련 정보를 서비스 디스커버리 서비스(SDS)에 기록하고, SDS는 서비스 엔드포인트 목록에 변화가 생길 때마다 지표 수집기에 통보하는 것이다.

<img width="585" alt="image" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/67941526/a922ed83-d167-44f1-b224-03a459e5bcad">  

수천 대 서버가 만들어 내는 지표 데이터를 수집하려면 지표 수집기 서버 한 대로는 부족하다. 지표 수집기 서버 풀을 만들어야 대용량 지표 데이터 규모를 감당할 수 있다. 지표 수집기 서버를 여러 대 둘 때 흔히 빚어지는 문제는 여러 서버가 같은 출처에서 데이터를 중복해서 가져올 가능성이 있다는 것이다. 따라서 지표 수집 서버 간에 중재 메커니즘이 존재해야 한다.

### 3.2 푸시 모델

<img width="554" alt="image" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/67941526/3818bcfa-2e20-4a36-a600-846c430af14c">  

푸시 모델은 지표 출처에 해당하는 서버가 직접 지표를 수집기에 전송하는 모델이다. 푸시 모델의 경우, 모니터링 대상 서버에 통상 수집 에이전트라고 부르는 소프트웨어를 설치한다. 수집 에이전트는 해당 장비에서 실행되는 서비스가 생산하는 지표 데이터를 받아 모은 다음 주기적으로 수집기에 전달한다.
간단한 카운터 지표의 경우에는 수집기에 보내기 전에 에이전트가 직접 데이터 집계 등의 작업을 처리할 수도 있다.  

푸시 모델을 채택한 지표 수집기가 밀려드는 지표 데이터를 제때 처리하지 못하는 일을 방지하려면, 지표 수집기 클러스터 자체도 자동 규모 확장이 가능하도록 구성하고 그 앞에 로드밸런서를 두는 것이 바람직하다. 
<img width="690" alt="image" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/67941526/fc6ccefb-4df8-400a-9196-26ee932ca7be">

<br>

## 4. 경보 시스템

<img width="656" alt="image" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/67941526/253d973f-679b-45d9-8977-31a9cdca2967">  

- 경보 관리자는 경보 설정 내역을 캐시에서 가져온다.
- 설정에 근거하여 경보 관리자는 지정된 시간마다 질의 서비스를 호출한다. 그리고 질의 결과가 설정된 threshold를 위반하면 경보 이벤트를 발행한다. 그 외에도 경보 관리자는 다음과 같은 작업을 수행한다.
  - 경보 필터링, 병합, 중복 제거
  - 접근 제어
  - 재시도

