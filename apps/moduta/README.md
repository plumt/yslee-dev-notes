# Moduta

## Basic Information

- **App Name**: Moduta (dev 빌드는 "서울버스 DEV", prod 는 "서울버스")
- **Package ID**: `com.yunseong.moduta` (dev 플레이버는 `.dev` 접미사)
- **Version Code**: 1
- **Version Name**: 1.0.0
- **Min SDK**: 29
- **Target SDK**: 36
- **Status**: Active Development
- **딥링크 스킴**: `yunseong-moduta`

## Description

서울시 버스 정보, 내 주변 정류장, 대기질을 4개 탭으로 묶은 독립 앱이다. 서울 열린데이터광장의 버스 API 9개를 도메인 계약으로 감싸 노선 검색과 실시간 위치를 제공하고, 카카오맵 위에 버스와 정류장을 그린다. 상세 구조는 [architecture.md](architecture.md) 를 본다.

## API Keys & Credentials

키 값은 저장소에 두지 않는다. `clients/moduta/config.json` 한 곳에서 관리하고, 빌드 때 NDK `NativeConfig`(XOR + JNI)로 구워 넣는다. `BuildConfig` 는 APK 에 평문으로 남기 때문에 키 노출에 쓰지 않는다.

| Key Name | Purpose | Where to get it |
|----------|---------|-----------------|
| `seoul_bus_service_key` | 서울 버스 공공 API 서비스 키 | 서울 열린데이터광장 (data.seoul.go.kr) |
| `kakao_map_app_key` | 카카오맵 SDK 앱 키 (dev·prod 별도 등록) | Kakao Developers |
| `air_quality_service_key` | 대기질(에어코리아) 서비스 키 | 공공데이터포털. 현재 빈 값 |
| `support_email` | 문의 메일 | 전 앱 공통 설정(`global-config`)에서 공급 |

- release 빌드는 서울 버스 키와 카카오맵 키가 없거나 비면 빌드 전 검증에서 실패하고, debug 는 경고만 낸다
- 서명 키스토어는 dev 플레이버와 prod 플레이버가 서로 다르며, 경로와 비밀번호는 환경변수 또는 `local.properties` 로 받는다

## Features

- **버스 검색**: 노선 번호로 검색하고 즐겨찾기를 토글한다 (첫 번째 탭)
- **내 주변**: 위치 권한과 GPS 를 거쳐 반경 500m 정류장과 즐겨찾는 정류장을 나열한다
- **대기질**: 에어코리아 측정소 대기질을 한 화면으로 보여준다
- **더보기**: 앱 버전, 문의, 오픈소스 라이선스 진입점
- **버스 지도**: 카카오맵 위에 노선 버스의 실시간 위치를 30초 주기로 갱신한다
- **정류장 상세**: 정류장의 도착 안내와 경유 노선 카드

## Dependencies

### App-specific Modules
- `:apps:moduta:core:database` - SQLCipher 로 암호화한 Room DB (캐시, 즐겨찾기)
- `:apps:moduta:core:seoul-bus` - 서울 버스 API 도메인과 데이터 레이어
- `:apps:moduta:feature:bus-search` / `nearby-station` / `air-quality` / `settings` - 4개 탭
- `:apps:moduta:feature:bus-map` - 지도 화면과 정류장 상세

### Shared Modules
- `app-shell` (앱 셸, 딥링크 호스트, 내비게이션·테마), `location-provider`, `permission-location`, `air-quality`, `fundamental:common`, `fundamental:permission`

### External Services
- Kakao Map SDK, 서울 열린데이터광장 버스 API, 에어코리아 대기질 API
- 광고 SDK 와 Firebase 는 쓰지 않는다

## Build Configuration

```bash
./gradlew :apps:moduta:assembleDevDebug
./gradlew :apps:moduta:assembleProdRelease
./gradlew :apps:moduta:installDevDebug
```

### Flavors

`dev`, `prod` 두 개. 규약 플러그인이 allowlist 기준으로 생성하므로 앱 모듈에서 `productFlavors`·`signingConfigs` 를 다시 선언하지 않는다. 서명은 buildType 이 아니라 플레이버 기준으로 갈린다. release 는 R8 축소를 켠다.

## Testing

```bash
./gradlew :apps:moduta:test
apps/moduta/script/run-appium-tests.sh 00_smoke   # Appium E2E (dev 플레이버 고정)
```

단위 테스트는 딥링크 host 매핑 일치와 경로 인자 인코딩을 검증한다. E2E 는 testTag 기반 콜드 스타트 스모크 시나리오다.

## Notes

- 카카오맵 SDK 는 `ModutaApplication.onCreate` 에서 초기화하고, 같은 키를 매니페스트 placeholder 로도 넣는다. 두 경로가 같은 플레이버 값을 보게 유지한다
- 서울 버스 API 는 평문 HTTP 만 되므로 `network_security_config.xml` 에 `ws.bus.go.kr` 한 도메인만 예외로 둔다
- `allowBackup` 은 false 다. 암호화 DB 는 복원해도 열리지 않는다
- 날씨 기능은 데이터 출처가 달라 범위에서 제외했다
