# 근접성 서비스

## 데이터 모델

읽기 연산은 굉장히 자주 수행되는데, 쓰기 연산 실행 빈도는 낮다. 사업장 정보를 추가하거나 삭제, 편집하는 행위는 빈번하지 않기 때문이다. <br>
이 시스템의 핵심이 되는 테이블은 business 테이블과 지리적 위치 색인 테이블이다.

## 개략적 설계

<img width="640" alt="스크린샷 2024-06-20 오후 2 25 58" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/8e1f871b-bd16-44bf-9759-c8f567de2d78">

### 데이터베이스 클러스터

primary-secondary로 DB 형태를 구성하여, 주 데이터베이스는 쓰기 요청을 처리하며, 부 데이터베이스는 읽기 요청을 처리한다. <br>
데이터는 일단 주 데이터베이스에 기록된 다음 사본 데이터베이스로 복사된다.

### 사업장 서비스와 위치 기반 서비스의 규모 확장성

둘 다 무상태 서비스이므로 점심시간 등의 특정 시간대에 집중적으로 몰리는 트래픽에는 서버를 추가하고, 야간 시간에는 서버를 삭제하도록 구성할 수 있다. <br>
시스템을 클라우드에 둔다면 여러 지역, 여러 가용성 구역에 서버를 두어 시스템 가용성을 높일 수 있다.

## 상세 설계

### 데이터베이스 규모 확장성

- 사업장 테이블
    - 사업장 테이블 데이터는 한 서버에 담을 수 없을 수도 있다. 따라서 샤딩을 적용하기 좋고, 샤딩하는 가장 간단한 방법은 사업장 ID를 기준으로 하는 것이다.
- 지리 정보 색인 테이블
    - 각각의 지오 해시에 연결되는 모든 사업장 ID를 JSON 배열로 만들어 같은 열에 저장하는 방식이 있다. 특정한 지오해시에 속한 모든 사업장 ID가 한 열에 보관된다.
        - 사업장 정보를 갱신하려면 JSON 배열을 읽고 갱신할 사업장 ID를 찾아내야 한다. 또한 병렬적으로 실행되는 갱신 연산 결과로 데이터가 소실되는 경우를 막기 위해 락을 사용해야 한다.
    - 같은 지오해시에 속한 사업장 ID 각각을 별도 열로 저장하는 방식이 있다.
        - 지오해시와 사업장 ID 컬럼을 합쳐 복합키로 사용하면 사업장 정보를 추가하고 삭제하기가 쉽다. (락을 사용할 필요가 없다)

### 캐시

캐시 도입을 논의할때는 벤치마킹과 비용 분석에 주의해야 한다.

#### 캐시 키

사용자 위치 정보는 캐시 키로 적절하지 않다. 위치가 조금 달라지더라도 변화가 없어야 이상적이다. <br>
지오해시나 쿼드트리는 이 문제를 효과적으로 해결한다. 같은 격자 내 모든 사업장이 같은 해시 값을 갖도록 만들 수 있다.

특정 지오해시에 해당하는 사업장 ID 목록을 미리 계산한 다음 레디스같은 저장소에 캐시할 수 있다.

<img width="724" alt="스크린샷 2024-06-20 오후 2 40 21" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/e49b2cf8-3a99-4580-9ca9-59767dd6fa03">

### 최종 설계도

<img width="715" alt="스크린샷 2024-06-20 오후 2 41 00" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/aed47f8d-f040-4f43-8aff-9bd370d283e8">