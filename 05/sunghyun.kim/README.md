# 지표 모니터링 및 경보 시스템

## 개략적 설계안

지표 모니터링 및 경보 시스템은 일반적으로 다섯 가지 컴포넌트를 이용한다.

- 데이터 수집
- 데이터 전송
- 데이터 저장소
- 경보
- 시각화

### 데이터 모델

지표 데이터는 시계열 데이터 형태로 기록한다. <br>
시계열 각각에는 고유한 이름이 붙고, 선택적으로 레이블을 붙이기도 한다.

### 데이터 저장소 시스템

시장에서 인기있는 시계열 DB 두가지는 InfluxDB, Prometheus이다. <br>
다량의 시계열 데이터를 저장하고 빠른 실시간 분석을 지원하는 것이 특징이고, 메모리 캐시와 디스크 저장소를 함께 사용한다. <br>
영속성 요건과 높은 성능 요구사항도 잘 만족한다.

좋은 시계열 데이터베이스는 막대한 양의 시계열 데이터를 레이블 기준으로 집계하고 분석하는 기능을 제공한다. <br>
InfluxDB는 레이블 기반의 신속한 데이터 질의를 지원하기 위해 레이블별로 인덱스를 구축한다. <br>
레이블을 이용할 때 데이터베이스 과부하를 피하는 지침도 제공한다. 핵심은 각 레이블이 가질 수 있는 값의 가짓수 (cardinality)가 낮아야 한다는 것. <br>
이 기능은 데이터 시각화에 중요하며, 범용 데이터베이스로 구축하기는 매우 까다롭다.

<img width="767" alt="스크린샷 2024-07-07 오후 8 15 02" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/021e91b2-9b6b-4c2c-8409-06830e9954dc">

<br>


## 상세 설계

### 지표 수집

카운터나 CPU 사용량같은 지표를 수집할 때는 데이터가 소실되어도 심각한 문제가 아니다. <br>
지표를 보내는 클라이언트는 성공적으로 데이터가 전송되었는지 신경쓰지 않아도 상관없다.

#### 풀 vs 푸시 모델

<img width="502" alt="스크린샷 2024-07-07 오후 8 19 02" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/5c467ea8-f1dc-4500-856c-18f3b938fae3">
<img width="541" alt="스크린샷 2024-07-07 오후 8 19 46" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/263c2346-0bb5-4b4d-b0c9-a291e3850381">
<img width="673" alt="스크린샷 2024-07-07 오후 8 20 37" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/3d3a534e-23ab-4919-a35a-e961f630a3d3">


### 지표 전송 파이프라인의 규모 확장

지표 수집기는 서버 클러스터 형태이며 엄청난 양의 데이터를 받아 처리해야 한다. <br>
이 클러스터는 자동으로 규모 확장이 가능하도록 설정하여 언제나 데이터 처리에 충분한 수집기 서버가 존재하도록 해야한다.

### 데이터 집계 지점

지표 집계는 다양한 지점에서 실행 가능하다.

- 수집 에이전트가 집계
  - 클라이언트에 설치된 수집 에이전트는 복잡한 집계 로직은 지원하기 어렵다. 어떤 카운터 값을 분 단위로 집계하여 지표 수집기에 보내는 정도는 가능하다.
- 데이터 수집 파이프라인에서 집계
  - 데이터를 저장소에 기록하기 전에 집계할 수 있으려면 플링크같은 스트림 프로세싱 엔진이 필요하다. 데이터베이스에는 계산 결과만 기록하므로 실제로 기록되는 양은 엄청나게 줄어든다. 하지만 늦게 도착하는 지표 데이터의 처리가 어렵고, 원본 데이터를 보관하지 않아 정밀도나 유연성 측면에서 손해를 본다
- 질의 시에 집계
  - 데이터를 날것 그래도 보관하고, 질의할 때 필요한 시간 구간에 맞게 집계한다. 데이터 손실 문제는 없으나 질의를 처리하는 순간에 전체 데이터 세트를 대상으로 집계 결과를 계산해야 하므로 속도는 느리다.

### 질의 서비스

질의 서비스는 질의 서버 클러스터 형태로 구현되며, 시각화 또는 경보 시스템에서 접수된 요청을 시계열 데이터베이스를 통해 처리하는 역할을 담당한다. <br>
이런 질의 처리 전담 서비스를 두면 클라이언트와 시계열 데이터베이스 사이의 결합도를 낮출 수 있다.

### 경보 시스템

<img width="694" alt="스크린샷 2024-07-07 오후 8 28 17" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/16dfd044-155e-445e-93e7-f65fa92a3cc5">














