# Moduta 아키텍처

## 모듈 구성

앱 모듈은 조립만 맡는다. 진입점, 내비게이션, 딥링크, 키 공급이 전부다. 버스 도메인은 앱 레벨에 없고 `core:seoul-bus` 가 쥔다.

```
:apps:moduta
├── core:database      SQLCipher Room (노선·정류소 캐시, 즐겨찾기)
├── core:seoul-bus     서울 버스 API 9개 → 도메인 (화면 없음, Compose 미의존)
├── feature:bus-search     탭 1 (MviViewModel)
├── feature:nearby-station 탭 2 (MviViewModel, 위치 권한)
├── feature:air-quality    탭 3 (StateViewModel)
├── feature:settings       탭 4
└── feature:bus-map        지도 + 정류장 상세 (바텀바 숨김)
```

같은 앱의 feature 끼리는 서로 의존하지 않고, 화면 사이 이동은 앱의 `ModutaNavGraph` 가 맡는다. `feature:seoul-bus` 는 화면이 없는 데이터 모듈이라 2026-10 에 `core:seoul-bus` 로 옮겼다.

## 내비게이션

| 라우트 | 화면 | 바텀바 |
|---|---|---|
| `bus_search` (시작) / `nearby_station` / `air_quality` / `settings` | 4개 탭 | 표시 |
| `bus_map/{busRouteId}/{routeName}` | 버스 지도 | 숨김 |
| `station_detail/{arsId}/{stationName}/{lat}/{lng}` | 정류장 상세 | 숨김 |

- 노선명·정류장명은 한글·공백·괄호가 들어가므로 `Uri.encode` 로 한 번만 인코딩해 넘기고, 꺼내는 쪽은 다시 복호하지 않는다. 예전 `URLEncoder` + 재복호 조합은 `%` 가 든 이름에서 크래시하고 `+` 를 공백으로 바꿨다
- 탭 이동은 공용 바텀바가 그래프 시작 목적지까지 상태를 저장하며 pop 하고 단일 top·상태 복원으로 탭별 백스택을 살린다
- 바텀바 표시는 각 목적지 진입 때 `LocalBottomBarState` 로 바꾸고, 서브 목적지는 하단 패딩도 0 으로 둔다

## 딥링크

host 에서 route 로 가는 맵 `HOST_TO_ROUTE` 하나가 단일 출처다. 화이트리스트와 라우터 매핑이 이 맵에서 파생되고, 두 host 집합이 같은지 단위 테스트가 확인한다. 예전에는 둘을 따로 적어 한쪽만 고치면 링크가 조용히 거부됐다. `yunseong-moduta://<host>` 형태만 받고 쿼리는 버리며, 서브 목적지로 가는 링크는 없다. 콜드 스타트 링크는 내비게이션 컨트롤러보다 먼저 도착하므로 큐에 넣어 두고 화면의 배선 이펙트가 라우터를 붙인 뒤 소비한다.

## 서울 버스 데이터 레이어 (core:seoul-bus)

`BusApiService` → `BusRemoteDataSource` → `BusRepositoryImpl` → `BusUseCase` 로 얇게 겹친다. 엔드포인트는 `buspos` 3, `busRouteInfo` 4, `stationinfo` 2, 모두 GET 이다.

- **응답 판정**: 이 API 는 키 만료·한도 초과도 HTTP 200 으로 보내고 `msgHeader.headerCd` 로만 알린다. 그래서 2xx 도 본문 전에 헤더 코드를 보고, 정상(0)과 결과 없음(4)이 아니면 전용 예외를 던진다
- **캐시**: HTTP 캐시는 공용 `OkHttpClient` 에 있다. 기본은 1분 캐시이고, 노선 전체 버스 위치와 정류소 도착 안내 두 조회만 `Cache-Control: no-cache` 로 매번 네트워크를 탄다. 초 단위로 바뀌는 값이라 캐시가 곧 오답이기 때문이다
- **좌표**: 응답마다 경도·위도 필드명이 다르다(`tmX/tmY` 대 `gpsX/gpsY`). 주변 정류장 요청은 `tmX` 에 경도, `tmY` 에 위도를 넣고 응답은 `gpsX/gpsY` 만 읽는다. 숫자로 못 읽으면 0.0 을 쓰고 모델의 `hasValidCoordinate` 로 걸러낸다
- **실행**: 저장소는 콜드 `Flow` 를 돌려주고 `flowOn(IO)` 로 요청과 매핑을 옮긴다. 저장소 안에는 `catch` 가 없고 실패 처리는 화면 쪽 상태 변환이 맡는다
- 서비스 키와 베이스 URL 은 Qualifier(`@SeoulBusServiceKey`, `@SeoulBusBaseUrl`)로만 주입받는다

## 화면 설계

- **bus-search**: `onIntent` 하나만 공개하는 MVI. 오류 문구는 상태가 아니라 1회성 효과로 보낸다(상태에 담으면 회전 때 스낵바가 다시 뜬다). 목록은 `LoadState`
- **nearby-station**: 반경 500m·가까운 순은 UseCase 가 정한다(서울 버스 API 권장 최대값). 권한 거부는 오류가 아니라 별도 권한 축으로 표현하고, 권한 축과 목록 축은 겹칠 수 있으므로 화면 상태 전체를 sealed 로 만들지 않는다. 진입 가드로 탭을 오갈 때마다 권한 팝업이 뜨는 것을 막는다
- **bus-map**: 30초 갱신 루프를 ViewModel 이 아니라 화면의 `repeatOnLifecycle(STARTED)` 안에서 돌린다(ViewModel 무한 루프는 테스트를 멈춘다). 카카오맵 라벨 아이콘은 벡터를 Bitmap 으로 래스터화해 등록한다(벡터를 그대로 넘기면 마커가 안 보인다). 지도와 상세는 같은 모듈이지만 지도는 MVI, 상세는 단순 ViewModel 이다
- **air-quality**: 에어코리아 호출·변환·등급 판정은 공용 모듈에 있고 이 모듈은 표시만 한다

## 로컬 저장소 (core:database)

`moduta_database` 는 SQLCipher 로 암호화한 Room DB 다(버전 2, 엔티티 5). 생성은 공용 `DatabaseFactory` 로만 하며, `Room.databaseBuilder` 를 직접 부르면 암호화가 빠져 평문 파일이 남는다. 파괴적 마이그레이션은 쓰지 않고 스키마를 바꾸면 버전과 `Migration` 을 함께 올린다. 즐겨찾기 시각은 주입한 `TimeProvider` 로 채운다.

## 키 공급과 빌드

- `clients/moduta/config.json` → 빌드 플러그인이 XOR·NDK 산출물 `NativeConfig` 로 굽고 `ModutaKeyModule` 이 Qualifier 로 각 feature 에 내려준다. feature 모듈에는 `NativeConfig` 심볼이 없다
- 카카오맵 앱 키만 두 길로 나간다. SDK 초기화는 `NativeConfig` 값으로, 매니페스트 meta-data 는 플레이버별 manifestPlaceholder 로 넣는다 (SDK 가 매니페스트를 직접 읽어 NDK 만으로는 숨길 수 없다)
- `preBuildVerify` 가 필수 키, 딥링크 스킴 일치(매니페스트·검증 블록·`DeepLinkConfig` 세 곳)를 빌드 전에 검사한다
- 테마는 공용 `AppThemeHost` 에 앱 테마를 넘겨 조립한다. 라이트 고정, 브랜드 파랑 하나가 포인트다

## 주요 설계 결정

| 시기 | 결정 | 이유 |
|---|---|---|
| 2026-10 | `feature:seoul-bus` 를 `core:seoul-bus` 로 이동 | 화면 없는 데이터 모듈이 feature 에 있어 화면 모듈 셋이 feature 를 참조했다 |
| 2026-10 | 죽은 `core:common` 제거, 브랜드 테마를 앱 모듈 한 곳에 | 색 상수 하나만 남았고 쓰는 곳도 앱뿐이었다 |
| 2026-09 | 경로 인자를 `Uri.encode` 한 번으로 | 재복호가 `%` 포함 이름에서 크래시했다 |
| 2026-08 | dev·prod 플레이버 도입 | 앱마다 서명·패키지 분기 방식이 갈려 있었다. 카카오 콘솔에 dev 앱을 따로 등록한 뒤 합류 |
| 2026-08 | 딥링크 host 맵 단일화 | 화이트리스트와 라우터를 따로 적어 한쪽만 고치면 링크가 조용히 실패 |
| 2026-08 | 베이스 URL 도 `NativeConfig` 로 | 값이 두 곳에 적히면 갈라진다 |
| 2026-06 | 날씨 기능 제외 | 데이터 출처가 달라 이 앱 범위를 넘는다 |

## 테스트

단위 테스트는 앱 모듈의 딥링크 매핑·경로 인자 인코딩 둘과 하위 모듈의 `src/test` 이다. E2E 는 `appiumTest/` 의 testTag 기반 콜드 스타트 스모크이고 실행 엔진은 저장소 공용 러너를 쓴다. 하네스는 dev 플레이버 고정이다.
