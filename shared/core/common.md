# shared/core/common

30개 모듈. Gradle 경로는 `:shared:core:common:<이름>` 입니다. 공통 규약은 [../core.md](../core.md) 를 봅니다.

## 요약 표

| 모듈 | 목적 | 핵심 기술 | project 의존 |
|---|---|---|---|
| admob | AdMob 초기화와 배너, 네이티브 광고 | play-services-ads, Compose | fundamental/common |
| air-quality | 에어코리아 실시간 대기질 조회 | Retrofit, Gson, Hilt | network |
| analytics | 분석 이벤트 계약과 방송 Registry | Hilt Multibinds | 없음 |
| app-shell | 바텀 탭 바, Scaffold 셸, 딥링크 호스트 | Compose, Navigation | navigation-base, designsystem |
| astro-event | KASI 천문현상 일정과 해설 | Retrofit, 문자열 파서 | network |
| backup | zip 백업과 검증, 복원 | JDK zip, 체크섬, 코루틴 | 없음 |
| calendar | 시스템 캘린더 삽입 화면 호출 | Intent.ACTION_INSERT | fundamental/common |
| clipboard | 클립보드 복사와 피드백 필요 여부 | ClipboardManager | 없음 |
| connectivity | 네트워크 상태 Flow | ConnectivityManager, Flow | 없음 |
| database | 암호화 Room 팩토리 | Room, SQLCipher, Keystore | fundamental/common |
| deeplink | 딥링크 화이트리스트 검증, 신뢰 판정 | 순수 Kotlin, Hilt | fundamental/common |
| designsystem | 테마 토큰과 공통 컴포넌트 | Compose Material3 | 없음 |
| favorites | 즐겨찾기 포트와 순서 계산 | 순수 Kotlin | 없음 |
| image-compression | 이미지 축소 디코딩과 압축 | Bitmap, EXIF | fundamental/common |
| location-provider | 현재 GPS 위치 1회 조회 | Hilt, 코루틴 | fundamental/permission, fundamental/common |
| moon-viewing | 달 보기 좋음 판단 | 순수 규칙 | weather-kma, lunar/calculator |
| navigation-base | 딥링크 라우터의 Compose 구현 | Navigation Compose, Hilt | deeplink, fundamental/common |
| network | OkHttp, Retrofit 공용 인프라 | OkHttp, Retrofit, Hilt | fundamental/common |
| paging | Paging3 기본값과 목록 상태 분기 | Paging3, Compose | designsystem |
| review-prompt | 인앱 리뷰 요청 시점 판정 | Play In-App Review | fundamental/common |
| search | 검색 디바운스, 최근 검색어 계약 | Flow, 순수 함수 | 없음 |
| theme-mode | 라이트, 다크, 시스템 모드 계약 | 열거형, 포트 | 없음 |
| threshold-alert | 히스테리시스 임계값 알림 판정 | 순수 Kotlin | 없음 |
| time-sync | 서버 시간 기반 기기 시간 보정 | Hilt, 캐시 | fundamental/common |
| ui | 앱 공통 Compose 컴포넌트 | Compose, Material3 | fundamental/common, designsystem, navigation-base |
| ui-base | presentation-base 의 Compose 골격 | Compose, lifecycle-compose | presentation-base, designsystem |
| weather-kma | 기상청 단기예보 | Retrofit, 격자 변환 | network |
| widget-base | Glance 홈 위젯 공통 골격 | Glance | designsystem |
| wikipedia | 위키백과 첫 단락 요약 | Retrofit | network |

## 인프라와 데이터

- **network**: 앱 전체가 공유하는 OkHttp 클라이언트와 Retrofit 팩토리, 표준 결과 타입을 둡니다. 인증 헤더 등 세 인증 포트는 `@BindsOptionalOf` 로만 선언해 기본 구현 없이 앱이 선택적으로 채우며, 셋 다 비면 클라이언트를 건드리지 않습니다. baseUrl 은 호출부가 넘기고, 본문 로깅은 `ENABLE_LOGGING`(debug 만 true) 분기 안에서만 합니다. 취소 예외는 `rethrowIfCancellation()` 으로 전파합니다.
- **database**: Keystore 로 키를 관리하는 암호화 Room 데이터베이스 팩토리입니다. DB 이름당 한 번만 `createDatabase` 를 부르고 싱글턴으로 공유합니다. 마이그레이션을 먼저 작성하고, 키 획득 실패 시 열 수 없는 파일은 버리므로 캐시가 아닌 데이터는 재동기 수단을 함께 둡니다.
- **backup**: 파일을 zip 으로 묶어 스트림에 내보내고, 버전과 체크섬 검증을 통과한 뒤에만 되돌려 씁니다. 교체 직전 사본을 남기고 실패하면 롤백합니다. DB, DataStore 지식은 `BackupSource` 구현부가 갖고 SAF 는 직접 부르지 않아(스트림과 경로만 받음) 의존이 없습니다.
- **connectivity**: 기본 네트워크 하나를 보고 온라인 여부, 연결 종류, 계량 여부를 `Flow` 로 냅니다. 온라인 판정에 검증 능력을 포함해 캡티브 포털을 온라인으로 보지 않습니다.
- **time-sync**: 서버 시간으로 기기 시간을 보정하며 의존성 추가만으로 활성화됩니다. 보정 조회는 캐시만 읽고 경과 시간은 `ElapsedRealtimeSource` 로 재며, 실패는 `null` 과 백오프로 처리합니다. `Optional<TimeSyncConfig>` 로 앱이 설정을 교체할 수 있습니다.
- **image-compression**: 축소 디코딩과 EXIF 회전을 거쳐 한 번에 압축합니다. 중간 Bitmap 은 `finally` 에서 회수하며 스레드는 호출부가 정합니다. Hilt, Compose 를 쓰지 않는 `object` 입니다.
- **clipboard**: 텍스트를 복사하고 피드백 표시 필요 여부를 반환합니다(플랫폼이 자체 안내를 띄우는 버전 판별). 민감 정보는 `sensitive` 플래그로 표시합니다.
- **calendar**: 런타임 권한 없이 시스템 캘린더 앱의 이벤트 삽입 화면을 `Intent.ACTION_INSERT` 로 띄웁니다. 앱이 없을 때의 `ActivityNotFoundException` 은 실패 결과로 돌려줍니다.

## 공공 API 클라이언트 (모두 network 위에 구현)

- **weather-kma**: 기상청 단기예보를 격자 단위로 받아 시간별 하늘상태, 강수, 기온으로 변환합니다. 서비스 키는 `@KmaServiceKey` 로 받고, 발표 시각은 `KmaBaseTime.latest` 로만 고르며, 응답은 문자열로 받아 전용 파서로 해석합니다.
- **air-quality**: 에어코리아 측정소별 실시간 대기질을 `AirQualityInfo` 로 돌려주는 data, domain 모듈입니다. 기본 측정소는 `중구` 이며 키와 baseUrl 은 앱이 한정자로 제공합니다. 화면은 없습니다.
- **astro-event**: 한국천문연구원 천문현상 API 에서 한 달치 해설과 일정을 받아 종류별로 분류합니다. 발표 전 달(빈 응답)은 캐시하지 않고 실패는 `AstroEventException` 하나로 던집니다.
- **wikipedia**: 한국어 위키백과 문서의 첫 단락 요약으로 용어 설명을 제공합니다. 동음이의, 없는 문서는 `null` 이며, 표시 시 출처, 라이선스, 원문 링크를 함께 둬야 합니다. 요청에는 ASCII 로만 구성한 User-Agent 가 필요합니다.
- **moon-viewing**: weather-kma 예보와 lunar 위치 타입을 받아 달을 볼 시각의 하늘상태, 강수로 "좋음, 구름 사이로, 어려움" 판단을 냅니다. 문구는 두지 않고 예보 실패는 `null` 입니다.

## 광고, 분석, 알림 판정

- **admob**: SDK 초기화는 `AdMobStartupTask` 가 `StartupApplication` 에서 수행하고 앱은 App ID 만 `manifestPlaceholders` 로 채웁니다. 잘못된 ID 로 인한 크래시를 막는 try/catch 를 유지하며 Hilt 를 요구하지 않습니다. 배너와 네이티브 광고 Composable 을 제공하고 광고 단위 ID 는 호출부가 넘깁니다.
- **analytics**: 이벤트 타입 계약(`AnalyticsLogger`)과 이를 방송하는 Registry 만 둡니다. SDK 연동은 앱이 만들어 `@Binds @IntoSet` 으로 등록하고, 문자열 기반 `logEvent(name, params)` 를 금지해 오타를 컴파일로 잡습니다.
- **threshold-alert**: 히스테리시스가 붙은 순수 임계값 판정만 담습니다. 도메인 어휘 없이 문자열 키 하나로 대상을 다루며 경계값은 포함입니다. 서버 구현과 짝으로 변경합니다.
- **review-prompt**: 인앱 리뷰를 언제 띄울지 `ReviewPromptPolicy` 로 판정하고 요청합니다. 상태 저장은 `ReviewPromptStore` 포트로 앱에 맡기며 실패는 전파하지 않습니다.
- **location-provider**: 현재 GPS 위치를 1회 조회하며 실패는 `LocationResult.Failure` 로 돌려줍니다. 권한 확보까지 한 번에 하는 `RequestCurrentLocationUseCase` 는 `EnsurePermissionUseCase` 에 위임합니다. 지역 이름은 시도, 시군구까지만 만듭니다(위치 과노출 방지). lunar, moduta 가 함께 사용합니다.

## 포트 계약 (구현은 앱)

- **favorites**: 대상 타입과 무관한 즐겨찾기 포트와 순서 계산 함수. Room 은 제네릭 DAO 가 불가능해 저장소는 앱이 구현합니다.
- **theme-mode**: 라이트, 다크, 시스템 모드와 해석 함수. 저장은 `ThemeModeStore` 포트, 시스템 모드 값은 파라미터로 받습니다.
- **search**: 검색 디바운스, `RecentSearchStore` 포트, 일치 구간 계산. UI 는 designsystem 컴포넌트를 조합합니다.

## 딥링크와 내비게이션

- **deeplink**: 화이트리스트 검증과 발신자 `TrustLevel` 판정으로 라우팅하는 Compose 비의존 계약입니다. `DeepLinkTarget` 생성자는 공개하지 않아 신뢰 수준 위조를 막고, 결제나 계정 변경은 딥링크만으로 수행하지 않습니다.
- **navigation-base**: deeplink 계약의 Compose 측 구현입니다. 딥링크 라우터와 화면 자가등록(`FeatureDestinationModule` 의 멀티바인딩)을 제공하고, 중복 route 는 예외로 알립니다.
- **app-shell**: `BottomNavDestination` 데이터로 받는 바텀 탭 바, Scaffold 셸, `DeepLinkHostActivity` 와 라우터 별칭 바인딩을 제공합니다. 앱별 분기는 두지 않습니다.

## UI

- **designsystem**: 앱별 강조색만 주입하면 색, 타이포, 간격, 형태, 컴포넌트가 따라오는 디자인 시스템입니다. 앱 루트는 `AppThemeHost(appTheme)` 하나로 세우고, `AppTheme` 추상 함수에는 기본값 인자를 두지 않습니다(`AppThemeDefaults.kt` 확장으로 제공). 소셜 로그인 버튼 브랜드 색은 심사 규정상 팔레트로 대체하지 않습니다. 공개 컴포저블은 기본값을 붙인 인자 추가만 허용합니다.
- **ui**: 다이얼로그, 바텀시트, 버튼, 하단탭 상태 등 공통 Compose 컴포넌트. 소비처 없는 컴포넌트는 추가하지 않고 빈 상태와 로딩은 designsystem 을 씁니다.
- **ui-base**: `presentation-base` 계약(UiState, Effect, LoadState)을 Compose 에서 구독해 렌더링하는 골격입니다. 효과 수집은 라이프사이클 게이트 안에서만 합니다.
- **paging**: Paging3 기본 설정과 Compose 목록 상태 분기(로딩, 빈 상태, 에러)를 표준화하며 페이지 소스는 두지 않습니다.
- **widget-base**: Glance 홈 위젯의 공통 골격으로 `PaperPalette` 변환과 위젯 루트, 공통 조각을 담습니다. 앱 어휘와 의미 색은 두지 않고 위젯에는 `@Preview` 를 붙이지 않습니다.
