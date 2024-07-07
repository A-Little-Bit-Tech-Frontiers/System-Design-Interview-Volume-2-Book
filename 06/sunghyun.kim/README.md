# 광고 클릭 이벤트 집계

## 개략적 설계안

### 질의 API 설계

- 기능 요구사항
  - 지난 M분 동안 각 ad_id에 발생한 클릭 수 집계
  - 지난 M분 동안 가장 많은 클릭이 발생한 상위 N개 ad_id 목록 반환
  - 다양한 속성을 기준으로 집계 결과를 필터링하는 기능 지원


#### API 1: 지난 M분 동안 각 ad_id에 발생한 클릭 수 집계

GET /v1/ads/{ad_id}/aggregated_count (주어진 ad_id에 발생한 이벤트 수를 집계하여 반환)

- request
  - from: 집계 시작 시간
  - to: 집계 종료 시간
  - filter: 필터링 전략 식별자
- response
  - ad_id: 광고 식별자
  - count: 집계된 클릭 횟수

#### API 2: 지난 M분 동안 가장 많은 클릭이 발생한 상위 N개 ad_id 목록 반환

GET /v1/ads/popular_ads (지난 M분간 가장 많은 클릭이 발생한 상위 N개 광고 목록 반환)

- request
  - count: 상위 몇 개의 광고를 반환할 것인가
  - window: 분 단위로 표현된 집계 윈도 크기
  - filter: 필터링 전략 식별자
- response
  - ad_ids: 광고 식별자 목록

### 데이터 모델

이 시스템이 다루는 데이터는 **원시 데이터**와 **집계 결과 데이터**이다.

<img width="303" alt="스크린샷 2024-07-07 오후 8 40 45" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/c5e19748-bbd0-40ea-9f38-27204d6157e9">
<img width="533" alt="스크린샷 2024-07-07 오후 8 40 41" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/53d4c811-9ad6-442c-bcbf-7c64d8400f11">

<br>

<img width="609" alt="스크린샷 2024-07-07 오후 8 40 57" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/64a4547b-0bed-46df-88d0-417d9ed52dc1">

### 올바른 데이터베이스 선택

**원시 데이터**는 백업과 재계산 용도로만 이용되므로 이론적으로는 읽기 연산 빈도는 낮다. <br>
쓰기 및 시간 범위 질의에 최적화된 카산드라나 InfluxDB가 적절하다.

**집계 데이터**는 본질적으로 시계열 데이터이며 이 데이터를 처리하는 워크플로는 읽기 연산과 쓰기 연산 둘 다 많이 사용한다. <br>
각 광고에 대해 매 분마다 데이터베이스에 질의를 던져 고객에게 최신 집계 결과를 제시해야 하기 때문이다.

<br>


<img width="693" alt="스크린샷 2024-07-07 오후 8 44 56" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/e455a63b-7428-4c78-922f-6d0f32971061">


### 집계 서비스

#### 클릭 이벤트 수 집계

<img width="761" alt="스크린샷 2024-07-07 오후 8 46 34" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/7fca8692-c569-47a4-9f8a-972c8f1109b6">

#### 가장 많이 클릭된 상위 N개 광고 반환

<img width="694" alt="스크린샷 2024-07-07 오후 8 47 18" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/a7a1755f-908c-417c-b1cc-2602c13b335e">

#### 데이터 필터링

**미국 내 광고 ad001에 대해 집계된 클릭 수만 표시**같은 데이터 필터링을 지원하려면 필터링 기준을 사전에 정의한 다음 기준에 따라 집계하면 된다.

















