# shared/core 개요

`shared/core` 는 모든 앱과 shared feature 가 공통으로 기대는 인프라 계층입니다. 총 53개 Gradle 모듈이 5개 그룹으로 나뉩니다. 화면이 있는 기능은 `shared/feature` 가, 앱 전용 코드는 `apps/*` 가 맡고, 이 계층은 "계약, 순수 규칙, 얇은 SDK 래퍼, 공통 UI 골격"만 둡니다.

## 그룹별 모듈 표

| 그룹 | 모듈 수 | 역할 | 상세 |
|---|---|---|---|
| common | 30 | 네트워크, DB, 디자인 시스템, 광고, 공공 API 클라이언트, 기능별 계약 | [core/common.md](core/common.md) |
| firebase | 6 | Firebase SDK 를 Hilt 와 코루틴에 맞게 감싼 래퍼 | [core/firebase.md](core/firebase.md) |
| fundamental | 5 | 최하위 기반(시각, 로깅, 설정), 권한 계약, 상태홀더 계약, 알람, 로컬 알림 | [core/fundamental.md](core/fundamental.md) |
| lunar | 1 | 네트워크 없는 순수 Kotlin 천문 계산 | [core/lunar.md](core/lunar.md) |
| permissions | 11 (+ 우산 디렉터리) | 권한 하나당 모듈 하나인 Provider 구현 | [core/permissions.md](core/permissions.md) |

### common (30)
admob, air-quality, analytics, app-shell, astro-event, backup, calendar, clipboard, connectivity, database, deeplink, designsystem, favorites, image-compression, location-provider, moon-viewing, navigation-base, network, paging, review-prompt, search, theme-mode, threshold-alert, time-sync, ui, ui-base, weather-kma, widget-base, wikipedia

### firebase (6)
core, auth, crashlytics, firestore, messaging, remote-config

### fundamental (5)
common, permission, presentation-base, alarm, local-notification

### lunar (1)
calculator

### permissions (11)
permission-core, permission-bluetooth, permission-calendar, permission-camera, permission-contacts, permission-gallery, permission-location, permission-location-background, permission-microphone, permission-motion, permission-notification

## 의존 방향

```
common  ──►  fundamental/common (최하위)
   │            ▲
   ├──► firebase/* ──► firebase/core
   └──► fundamental/permission ◄── permissions/permission-*
lunar/calculator : 어느 모듈도 의존하지 않는 순수 모듈 (moon-viewing 이 소비)
```

- `fundamental/common` 은 다른 project 를 전혀 의존하지 않는 최하위 모듈이며, 저장소에서 직접 의존하는 모듈 수가 가장 많습니다.
- 계약만 필요한 모듈(analytics, favorites, theme-mode, search, threshold-alert, backup, connectivity 등)은 project 의존이 0개입니다. 무거운 그래프를 소비자에게 끌고 가지 않기 위해서입니다.
- 공개 시그니처에 노출되는 타입의 의존은 `api`, 내부 구현용은 `implementation` 으로 구분합니다.

## 공통 설계 원칙

1. **포트(계약)와 구현의 분리**: 저장이 필요한 기능(favorites, theme-mode, search, review-prompt)은 인터페이스(포트)만 두고 구현은 앱이 Hilt 로 바인딩합니다. Room 처럼 제네릭 구현이 불가능하거나 앱마다 저장소가 다르기 때문입니다.
2. **플러그인식 확장은 Hilt Multibindings**: analytics 로거, FCM 핸들러, 권한 Provider, 알람 재예약자(`AlarmRescheduler`)는 `@IntoSet` 으로 기여하고, 모듈이 `@Multibinds` 로 빈 집합을 선언해 아무도 등록하지 않아도 빌드가 깨지지 않게 합니다.
3. **앱별 값은 주입, 모듈에는 박지 않음**: API 서비스 키, base URL, DB 이름, 광고 단위 ID, 분석 SDK 는 모두 한정자(qualifier)나 파라미터로 받습니다. 키는 NativeConfig 에서 앱이 읽어 제공합니다.
4. **빌드 시점 가드**: `FirebaseConfigGuard`(google-services 적용 확인), `ConsumerRulesGuard`(직렬화 DTO 의 keep 규칙 확인), `PermissionProviderGuard`(권한 타입별 Provider 모듈 등록 확인)가 규칙 위반을 빌드에서 막습니다.
5. **실패는 값으로, 취소는 전파**: 외부 호출 실패는 `null` / `false` / `Result` / `Failure` 값으로 접어 호출부로 예외가 새지 않게 하되, 넓은 `catch` 첫 줄에서 `rethrowIfCancellation()` 을 불러 코루틴 취소는 삼키지 않습니다.
6. **시각 주입**: 시각은 `TimeProvider` 로만 읽어 서버 시간 보정을 받게 하고, 테스트에서 고정할 수 있게 합니다.
7. **UI 문구와 도메인 분리**: 판정 모듈(moon-viewing, threshold-alert, location-provider 등)은 문구를 만들지 않고 값만 돌려줍니다. 표시는 소비 화면이 합니다.
8. **규약 플러그인과 Version Catalog**: 모든 모듈이 `convention.android.library` 를 기본으로, 필요에 따라 `convention.android.hilt`, `convention.android.compose`, `convention.buildconfig` 를 얹고, 라이브러리 버전은 Version Catalog 로만 참조합니다. minSdk 는 29 입니다.
9. **R8/keep 은 consumer-rules.pro**: 소비 앱에 전파해야 할 keep 규칙은 각 모듈의 `consumer-rules.pro` 에만 둡니다.
10. **모듈 문서의 2층 구조**: 모듈마다 짧은 `AGENTS.md`(지켜야 할 것, 의존)와 `docs/`(API, DESIGN, CAVEATS, DECISIONS, ERRORS, TROUBLESHOOTING 등)로 문서가 나뉩니다. 이 레포의 그룹별 문서는 그 요약입니다.
