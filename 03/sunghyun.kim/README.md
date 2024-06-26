# 구글 맵

<img width="597" alt="스크린샷 2024-06-30 오후 3 14 34" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/99279872-5f85-4e69-b041-05c1bf61a553">

## 위치 서비스

위치 서비스는 사용자의 위치를 기록하는 역할이다. <br>
클라이언트는 t초마다 자기 위치를 전송한다고 가정한다. <br>
이렇게 주기적으로 위치 정보를 전송하면

1. 해당 데이터 스트림을 활용하여 시스템을 점차 개선할 수 있다.
2. 클라이언트가 보내는 위치 정보가 거의 실시간 정보에 가까우므로 ETA를 좀 더 정확하게 산출할 수 있고, 교통 상황에 따라 다른 경로를 안내할 수도 있다.

*사용자의 위치 이력을 클라이언트에 버퍼링해 두었다가, batch request하면 전송 빈도를 줄일 수 있다.*

<img width="594" alt="스크린샷 2024-06-30 오후 3 18 14" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/73d18be3-4801-487e-90e9-6c2fd9f7886a">

> 위치 갱신 요청 빈도를 줄여도 여전히 많은 쓰기 요청을 처리해야 한다. <br>
> 아주 높은 쓰기 요청 빈도에 최적화된 카산드라 DB, 카프카같은 스트림 처리 엔진 등을 이용할 수 있다. <br>
> 통신 프로토콜로는 HTTP를 keep-alive 옵션과 함께 사용하여 효율을 높일 수 있다.

## 경로 안내 서비스

이 컴포넌트는 A에서 B로 가는 합리적으로 빠른 경로를 찾아주는 역할을 담당한다. **결과를 얻는데 드는 시간 지연은 어느정도 감내할 수 있다. 계산된 경로는 최단 시간 경로일 필요는 없으나 정확도는 보장되어야 한다.** <br>

## 지도 표시

확대 수준별로 한 벌씩 지도 타일을 저장하려면 수백 PB가 필요하다. 그 모두를 클라이언트가 가지고 있는 것은 실용적이지 않다. <br>
클라이언트의 위치 및 현재 클라이언트가 보는 확대 수준에 따라 필요한 타일을 서버에서 가져오는 접근법이 바람직하다. <br>
아래는 클라이언트가 지도 타일을 서버에서 가져오는 몇 가지 시나리오이다.

- 사용자가 지도를 확대 또는 이동시키며 주변을 탐색한다.
- 경로 안내가 진행되는 동안 사용자의 위치가 현재 지도 타일을 벗어나 인접한 타일로 이동한다.

### 선택지 1

클라이언트의 위치, 현재 클라이언트가 보는 지도의 확대 수준에 근거하여 필요한 지도 타일을 즉석에서 만드는 방안이다. <br>
사용자 위치 및 확대 수준의 조합은 무한하다는 점을 감안하면, 이 방안에는 몇 가지 심각한 문제가 있다. <br>

- 모든 지도 타일을 동적으로 만들어야 하는 서버 클러스터에 심각한 부하가 걸린다.
- 캐시를 활용하기 어렵다.

### 선택지 2

확대 수준별로 미리 만들어둔 지도 타일을 클라이언트에 전달한다. <br>
각 지도 타일이 담당하는 지리적 영역은 지오해싱같은 분할법을 사용해 만든 고정된 사각형 격자로 표현되므로 정적이다. <br>
이렇게 미리 만들어둔 정적 이미지는 CDN을 통해 서비스한다. <br>
이 접근법은 규모 확장이 용이하고 성능 측면에서도 유리하다.

<br>

## 데이터 모델

### 경로 안내 타일

도로 데이터로는 외부 사업자나 기관이 제공한 것을 이용한다. 이 데이터는 방대한 양의 도로 및 메타데이터로 구성된다. <br>
경로 안내 타일 처리 서비스라 불리는 오프라인 데이터 가공 파이프라인을 주기적으로 실행하여 경로 안내 타일로 변환하다.

경로 안내 타일을 저장하는 효율적인 방법은 S3같은 객체 저장소에 파일을 보관하고, 그 파일을 이용할 경로 안내 서비스에서 적극적으로 캐싱하는 것이다. <br>
타일을 객체 저장소에 보관할떄는 지오해시 기준으로 분류해서, 위도와 경도가 주어졌을때 신속하게 찾는다.

### 사용자 위치 데이터

이 데이터는 도로 데이터 및 경로 안내 타일을 갱신하는데 사용되며, 실시간 교통 상황 데이터나 교통 상황 이력 데이터베이스를 구축하는데 사용된다. <br>
아울러 데이터 스트림 프로세싱 서비스는 이 위치 데이터를 처리하여 지도 데이터를 갱신한다. <br>
사용자 위치 데이터를 저장하려면 엄청난 양의 쓰기 연산을 잘 처리할 수 있으며, 규모 확장이 가능한 카산드라가 적절하다.

### 지오코딩 데이터베이스

위도/경도 쌍으로 변환하는 정보를 보관한다. <br>
레디스처럼 빠른 읽기 연산을 제공하는 key-value 저장소가 적당하다.

### 미리 만들어둔 지도 이미지

이미지는 한 번만 계산하고 그 결과는 캐시해두는 전략을 쓰는것이 좋다. <br>
CDN 원본 서버로는 S3같은 클라우드 저장소를 활용한다.

<br>

## 사용자 위치 데이터는 어떻게 이용되는가

<img width="712" alt="스크린샷 2024-06-30 오후 3 37 52" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/6968ae68-9fe2-40ab-a0ff-94fd7b613475">

## 최종 설계안

<img width="770" alt="스크린샷 2024-06-30 오후 3 39 16" src="https://github.com/A-Little-Bit-Tech-Frontiers/System-Design-Interview-Volume-2-Book/assets/87420630/3b9265c5-eec5-406a-8c36-dbc346bf50bb">













