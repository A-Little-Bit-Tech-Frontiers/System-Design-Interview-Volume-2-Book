# S3와 유사한 객체 저장소

## 블록 저장소

HDD나 SSD처럼 서버에 물리적으로 연결되는 형태의 드라이브

- raw block을 서버에 볼륨 형태로 제공한다.
- 가장 유연하고 융통성이 높은 저장소

## 파일 저장소

- 파일과 디렉터리를 손쉽게 다루는데 필요한 높은 수준의 추상화를 제공
- 가장 널리 사용되는 범용 저장소 솔루션

## 객체 저장소

- 데이터 영속성을 높이고 대규모 애플리케이션을 지원하며 비용을 낮추기 위해 의도적으로 성능을 희생
- 실시간 갱신이 필요가 없는 상대적으로 cold data 보관에 초점을 맞추며 데이터 아카이브나 백업에 주로 사용됨

<img width="687" alt="image" src="https://github.com/user-attachments/assets/0e190ca8-6a74-4ff1-bdd4-01b6ef239d7f">

## 개략적 설계안

<img width="706" alt="image" src="https://github.com/user-attachments/assets/9ccb3b44-f438-43f6-bf36-5949961da2f8">

## 객체 업로드

<img width="697" alt="image" src="https://github.com/user-attachments/assets/23f118d6-09f3-4dc9-9de5-4b16d1243d97">

## 객체 다운로드

![image](https://github.com/user-attachments/assets/6c0f50e9-9649-439a-9017-eb292be83599)


## 데이터 저장소

<img width="680" alt="image" src="https://github.com/user-attachments/assets/9b02059e-a361-4622-b72c-9fae653459b1">

## 데이터 저장 흐름

<img width="685" alt="image" src="https://github.com/user-attachments/assets/369a724b-a64c-4ea5-9ecf-3a9bf499aa87">

1. API 서비스가 객체 데이터를 저장소로 포워딩
2. 데이터 라우팅 서비스가 배치 서비스에게 해당 객체를 어느 데이터 노드에 보관할지 질의 → 배치서비스는 주 노드 반환
3. 데이터 라우팅 서비스가 직접 주 노드에 접근해서 저장
4. 주 데이터 노드는 다중화 진행

* 2단계에서 배치 서비스가 데이터 노드를 선택하는 방법은 보통 안정 해시를 사용

* 부노드에 데이터를 다중화 하는 것은 일관성을 보장하기 위함 → 지연시간과 트레이드 오프

## 데이터 전확성 검증

디스크  혹은 메모리에서 데이터 훼손이 일어나는 문제는 체크섬을 두어 해결 가능원본 체크섬을 두면 전송 받은 데이터를 검증할 수 있다.

원본 체크섬을 두면 전송 받은 데이터를 검증할 수 있다.

<img width="655" alt="image" src="https://github.com/user-attachments/assets/4c73327c-2084-465c-a6ee-d3e6ceafedbb">

- 새로 계산한 체크섬이 원본 체크섬과 다르면 데이터가 훼손된 것
- 같은 경우에는 높안 확률로 데이터가 온전하다고 볼 수 있다.

## 버킷 내 객체 목록

객체 저장소는 객체를 파일 시스템처럼 계층적 구조로 보간하지 않는다. `s3://<버킷명>/<객체명>` 의 수평적 경로로 접근한다.

## 객체 버전

객체 버전은 버킷 안에 한 객체의 여러 버전을 둘 수 있도록 하는 기능이다.

실수로 지우거나 덮어쓴 객체를 복구 가능

### 버전이 유지되는 객체의 업로드

<img width="461" alt="image" src="https://github.com/user-attachments/assets/8efcbcf9-c41c-4c3d-9442-22f19387880c">


1. HTTP PUT 요청
2. API 서비스는 사용자 ROLE에 버킷 쓰기 권한을 확인한다.
3. 업로드
4. 새 객체의 메타데이터 저장
5. 버전기능을 지원하기 위해 메타데이터 저장소의 객체 테이블에는 `object_version` 컬럼이 존재한다. 버전 기능이 활성화 되어 있는 경우에만 데이터가 저장된다. 기존 레코드를 덮어 쓰는 대신에 `bucket_id`와 `object_name`은 같지만 `object_id`와 `object_version은` 다름. 같은 데이터들 중에 TIMEUUID 값이 가장 큰 녀석이 최신 데이터
