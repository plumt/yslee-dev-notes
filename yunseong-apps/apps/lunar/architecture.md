# Lunar - 아키텍처

## 모듈 구성과 경계

```
:apps:lunar                    앱 조립 (Application, MainActivity, di/, navigation/)
 ├─ core:common                달 그림·포맷·모델·알림 설정·테마
 ├─ feature:splash             시작 화면 (앱 NavGraph 가 직접 연결)
 ├─ feature:home               탭: 홈
 ├─ feature:calendar           탭: 달력
 ├─ feature:explore            탭: 달 탐구
 ├─ feature:sky                탭: 이달의 하늘
 ├─ feature:settings           탭: 설정
 └─ feature:notification       FCM 핸들러 (조립 전용, 화면 없음)
shared:*                       앱 셸, 달 계산(calculator), 내비게이션, 디자인시스템, Firebase 등
```

- 앱 모듈은 Application·MainActivity 와 `di/`·`navigation/` 패키지만 가진다. 화면 코드가 앱 모듈에 들어오면 `AppModuleShapeGuard` 가 구성 단계에서 빌드를 멈춘다.
- 의존 방향은 `feature -> core:common -> shared` 한쪽뿐이다. `core:common` 은 feature 를 역참조하지 않는다.
- 달 천문 계산(삭·상현·망·하현 시각, 월령, 블루문, 슈퍼문 판정)은 앱이 아니라 `:shared:core:lunar:calculator` 에 있고, Hilt 제공은 `core:common` 이 맡는다.
- 각 feature 는 `convention.feature` 규약 플러그인으로 library·hilt·compose 설정과 presentation-base·ui-base·designsystem 의존을 받는다.

## 화면 소유와 내비게이션

| 화면 | 그래프 등록 주체 |
|---|---|
| 스플래시 | 앱 `NavGraph` 가 직접 연결하고 시작 목적지로 둔다 |
| 홈, 달력, 달 탐구, 하늘, 설정 | 각 feature 모듈이 `FeatureDestination` 을 `@IntoSet` 으로 자가등록 |
| 알림함 | 앱 `NavGraph` 가 `notifications` route 로 연결, 바텀바 숨김 |

부팅 뒤 어디로 갈지는 앱 정책이라 스플래시만 앱이 소유한다. 나머지는 feature 쪽에서 route 상수와 등록 순서(홈 20, 달력 30, 달 탐구 40, 하늘 45, 설정 50)를 정하고, 앱은 `featureDestinations(...)` 로 그 집합을 순서대로 펼 뿐이다. 앱이 아는 것은 route 상수와 탭 라벨·아이콘뿐이다. 탭을 늘릴 때는 feature 의존, `Screen` 객체, `bottomNavItems` 항목을 함께 더해야 바텀바에 보인다.

## 앱 셸 조립 순서

`MainActivity` 가 edge-to-edge 를 켜고 콜드 스타트 딥링크를 큐에 넣은 뒤 화면을 세운다.

1. `LunarPhaseTheme` - 공용 `AppThemeHost` 에 Midnight 팔레트와 달 그림 색을 얹는다. Material 색도 이 팔레트에서 파생되어 공용 다이얼로그도 같은 톤이다
2. `DeepLinkWiringEffect` - 라우터 부착·해제와 큐 소비
3. 바텀바 표시 상태와 종료 확인 다이얼로그(셸 밖에서 만든다)
4. `AppShell` - Scaffold 셸. 바텀 슬롯에 배너 광고와 바텀바를 세로로 얹어 둘이 함께 나타나고 사라진다
5. `NavGraph` - 셸이 계산한 하단 여백만 받는다

## 딥링크

host -> route 매핑 `HOST_TO_ROUTE` 하나가 정본이다. 화이트리스트(`provideRoutes`)는 이 맵의 키로 만들고 라우터는 이 맵 자체를 받는다. 두 곳을 따로 적던 시절 한쪽만 고쳐 조용히 어긋난 일이 있어 한 곳으로 합쳤다.

- 경로 없는 host 형태만 통과하고 쿼리는 버린다. 신뢰 수준은 모두 PUBLIC
- 콜드 스타트는 `onCreate` 에서, 실행 중에는 `onNewIntent` 에서 URI 를 큐(최근 1개)에 넣고 `DeepLinkWiringEffect` 가 소비한다
- 자기 패키지 판정은 `BuildConfig.APPLICATION_ID` 를 써서 dev 의 `.dev` 접미사를 따라간다

## 데이터 흐름

서버 API 없이 계산 중심으로 흐른다.

```
TimeProvider.dayTicks(KST)  ->  ViewModel  ->  calculator (Meeus)  ->  UiState  ->  Compose
        (지금 + KST 자정마다)       (init/구독 중 적재)  (위상·이벤트·블루문)
```

- 홈은 계산 값을 먼저 내보내 `isLoading` 을 끄고, 이어서 관측 지점·월출·월몰과 날씨를 덧붙여 다시 내보낸다
- 시각은 항상 `TimeProvider` 로만 읽어 테스트에서 고정할 수 있고, 날짜 경계는 KST 로 자른다
- 달력의 이벤트·블루문 레이블링은 계산 모듈의 `MonthlyMoonCalendar` 가 하고 feature 는 표시 모델로 옮겨 적는다
- 달력 적재 실패는 1회성 이벤트가 아니라 상태(`error`)로 알린다. 실패는 그 달을 보는 동안 계속 참인 사실이라서다
- 달 탐구의 슈퍼문·미니문 분류 임계값(360,000km 이하 / 405,000km 이상)은 천문학 정의가 아니라 앱이 정한 값이다

## 알림 파이프라인

```
FCM (data-only) -> AppFcmService(공용) -> FcmMessageRouter -> 달 이벤트 핸들러
   -> MoonNotificationSettings 토글 확인 -> 로컬 알림 + 알림함 저장
```

- 서버가 달 이벤트를 data-only 로 보낸다. notification 페이로드를 붙이면 백그라운드에서 핸들러가 불리지 않는다
- 토픽이 두 개다: `moon_phase`(새달·보름달), `moon_special`(슈퍼문·미니문·블루문). 토글 두 개는 서로 독립이고 마스터 토글은 없으며 기본값은 둘 다 꺼짐(opt-in)이다
- 토글은 토픽 구독과 핸들러 쪽에서 두 번 확인한다. 끄기는 저장이 먼저, 구독 해제가 나중이라 해제가 실패해도 핸들러가 저장 값을 보고 막는다. 발신 쪽이 토픽과 eventType 짝을 틀려도 수신 쪽에서 걸러진다
- 켜기는 반대로 토픽 구독이 성공해야 ON 을 저장한다. 실패하거나 Firebase 가 준비되지 않았으면 OFF 를 저장하고 안내 효과를 보낸다
- 안내 문구는 상태 필드가 아니라 일회성 `SettingsEffect` 로 보낸다. 상태에 두면 회전 시 같은 토스트가 반복된다
- 알림 설정은 SharedPreferences 기반 공용 저장소 대신 DataStore(Preferences)에 둔다. 이 기능 전용이고 Flow 구독이 필요해서다

## 달 위상 그림

`MoonPhaseImage` 는 위상 값 p(0~1)로 Canvas 에 직접 그린다.

- 신월(p <= 0.01, >= 0.99)은 그림자 원만, 보름(0.48~0.52)은 달 사진 또는 달빛 원 전체를 그린다
- 반달 부근(0.23~0.27, 0.73~0.77)은 정확한 반원으로 붙이고, 그 밖은 명암 경계를 타원 호로 만든 경로(`buildMoonPath`)로 그린다
- 사진이 있으면 사진 위에 블러 그림자를, 없으면 색 경로로 달빛을 칠한다
- 접근성 설명은 같은 p 로 위상 이름을 만들어 `contentDescription` 에 넣는다

## 빌드와 보안 구성

- **규약 플러그인**: 플레이버·서명·가드는 `convention.android.application` 이 만든다. 앱 `build.gradle.kts` 에서 `flavorDimensions`·`productFlavors`·`signingConfigs` 를 다시 선언하지 않는다. 플러그인 선언 순서는 `ModuleConventionGuard` 가 검사한다
- **NativeConfig**: 설정 JSON -> 생성 스크립트 -> XOR 암호화 C 소스 -> NDK 빌드 -> JNI 복호화. 키가 소스나 리소스에 평문으로 남지 않는다
- **빌드 전 검증(`preBuildVerify`)**: prodRelease 의 테스트 광고 ID 검사, 전 variant 의 딥링크 스킴 선언 검사, release 서명 검사. `AdIdVariantGuard` 는 소스에 광고 ID 리터럴이 있으면 평가 직후 빌드를 멈춘다
- **크래시 리포트**: crashlytics 모듈 의존만으로 `Logger` 에 리포터가 꽂힌다. 릴리스에서 `Logger.e` 가 버려져 딥링크·알림 경로 오류를 볼 수 없던 문제를 해결하려고 붙였다

## 주요 설계 결정

| 시기 | 결정 | 이유 |
|---|---|---|
| 2026-10 | 알림함 앱 전용 Material 색 래퍼 삭제 | 공용 `AppThemeHost` 가 루트에서 이미 같은 톤으로 파생한다 |
| 2026-10 | 화면이 실제 상태바 높이만큼 위 여백을 둔다 | 고정 여백만 쓰면 홈의 날짜·종 버튼이 상태바와 겹쳐 눌리지 않았다 |
| 2026-09 | 공공데이터 API 경로 제거, Meeus 일원화 | 하루 한 번 월령만 제공되어 삭망 보간 오차가 컸다 |
| 2026-09 | 크래시 리포터를 로거에 연결 | 릴리스에서 오류 로그가 통째로 버려졌다 |
| 2026-08 | 화면 다섯을 feature 모듈로 분리 | 앱 직속 코드 비율이 앱 중 가장 높았다 |
| 2026-08 | 탭을 `FeatureDestination` 자가등록으로, 스플래시만 앱 소유 | 자가등록하면 스플래시가 홈 route 를 알아야 한다 |
| 2026-08 | 딥링크 host 매핑을 `HOST_TO_ROUTE` 로 통합 | 화이트리스트와 라우터를 따로 적어 조용히 어긋났다 |
| 2026-08 | dev·prod 플레이버를 규약 플러그인 allowlist 로 이관 | 앱마다 같은 블록을 선언하면 서명 분기가 앱 수만큼 갈라진다 |
| 2026-08 | 앱 셸을 `shared/core/common/app-shell` 로 이전 | Activity·Scaffold·바텀바 골격이 앱마다 복제되어 있었다 |

## 알려진 함정 요약

- 스킴 문자열은 `DeepLinkConfig`·매니페스트·`preBuildVerify` 세 곳에 있는데 빌드 검증은 앞의 둘만 비교한다
- 조립 전용 의존 중 `feature:notification`(FCM 핸들러 등록)과 `permission-notification`(권한 Provider)은 빼면 빌드는 통과해도 기능이 사라지거나 구성 단계에서 멈춘다
- Activity 재생성 때 같은 링크가 다시 큐에 들어가 같은 화면이 한 번 더 쌓일 수 있다
- 시스템 바 스타일은 `SystemBarStyle.dark` 로 고정한다. 앱은 늘 밤하늘 테마인데 기본값은 기기 모드를 따라 라이트 모드에서 상태바 아이콘이 묻힌다
