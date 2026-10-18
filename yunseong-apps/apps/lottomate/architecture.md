# Lottomate 아키텍처

앱 모듈(`:apps:lottomate`)은 조립만 하고, 데이터·도메인은 `core`, 화면은 `feature` 모듈이 갖는다. 규약 플러그인의 `AppModuleShapeGuard` 가 앱 모듈에 허용 밖 파일·디렉터리·컴포넌트 선언이 생기면 구성 단계에서 빌드를 멈춘다. 이 규칙은 앱 직속에 쌓였던 코드 약 1만 줄을 feature 모듈로 분해한 뒤 지키고 있다.

## 모듈 계층

```
:apps:lottomate  (Application · MainActivity · di/ · navigation/)
 ├─ feature/*    화면 + 상태홀더 (splash, home, my-numbers-overview, my-numbers, statistics, more, messaging)
 ├─ core/*       데이터·도메인 (common, database, winning-numbers, my-numbers, statistics, winning-checker, countdown)
 └─ shared/*     여러 앱이 공유하는 기반 (app-shell, firebase, qr, chart, designsystem ...)
```

- feature 는 core 를 의존하고, core 는 feature 를 의존하지 않는다. 한때 화면 모듈끼리 서로 15건을 참조해 data·domain 을 core 로 옮겼고, 화면 부품 재사용 3건만 계층 가드 예외로 남았다.
- `core/common` 은 회차 모델, 당첨 판정, 번호 계산, 번호 UI, 테마, 설정 상수, 토픽 구독을 모은 공용 모듈이다. feature 를 의존하지 않는다.

## 시작 순서

| 순서 | 어디 | 하는 일 |
|---|---|---|
| 1 | `LottomateApplication` | 메인 스레드에서 광고 SDK 초기화, 로거에 Crashlytics 연결 |
| 2 | 같은 곳, IO 코루틴 A | Remote Config 초기화 (fetch 최소 간격 0초) |
| 3 | 같은 곳, IO 코루틴 B | 당첨 확인 토픽 갱신, 구매 알림이 켜져 있으면 토픽 구독 |
| 4 | `MainActivity` | edge-to-edge, Crashlytics 수집 설정, 콜드 스타트 딥링크 적재 |
| 5 | `setContent` | 앱 테마 -> 딥링크 배선 -> `AppShell` -> `NavGraph` |

초기화 코루틴 스코프는 `SupervisorJob` 기반이고 `coroutineErrorHandler` 를 문맥에 가진다. 초기화 중 예외가 빠져나오면 로그만 남고 프로세스는 살아 있다. 과거에 이 처리기가 없어 예외가 콜드 스타트를 끝내는 문제가 있었고 그 뒤 추가했다.

## 내비게이션

시작 목적지는 `splash` 이고 홈으로 갈 때 `popUpTo(inclusive)` 로 백스택에서 뺀다.

| 라우트 | 화면 | 바텀바 |
|---|---|---|
| `splash` | 스플래시 | 숨김 |
| `home` · `my_numbers` · `statistics` · `more` | 탭 4개 | 표시 |
| `notifications` | 알림함 | 숨김 |
| `privacy_policy` · `terms_of_service` | 약관 웹뷰 | 숨김 |

바텀바는 공용 `AppBottomNavigationBar` 에 탭 목록만 넘기는 래퍼이며 높이는 전 앱 공통 60dp 다. 표시 여부는 각 목적지가 `LaunchedEffect` 로 직접 켜고 끈다. 배너 광고가 바텀바 슬롯 안에 있어 바텀바가 숨는 화면에서는 광고도 함께 숨는다.

## 딥링크

`HOST_TO_ROUTE` 맵 하나에서 허용 목록(화이트리스트)과 라우터 매핑이 함께 파생된다. 두 곳에 따로 적으면 한쪽만 고쳤을 때 링크가 조용히 거부되거나 이동하지 않기 때문이다.

| 호스트 | 라우트 | 발신자 신뢰 |
|---|---|---|
| `home` | `home` | PUBLIC |
| `notifications` | `notifications` | SELF |
| `my-numbers` | `my_numbers` | PUBLIC |
| `statistics` | `statistics` | PUBLIC |

외부 앱이 열면 안 되는 호스트는 `SELF_ONLY_HOSTS` 에 넣는다. 알림함 항목을 누르면 앱이 자기 패키지를 발신자로 디스패치해 SELF 경로를 통과한다. 이전에는 모든 항목이 PUBLIC 으로 만들어져 외부 앱이 알림함을 열 수 있었고 이를 수정했다.

## 데이터 흐름

```
Firestore(최신/회차) ─┐
동행복권 API(전 회차) ─┴─> core/winning-numbers ──> core/database (SQLCipher Room)
                                                      ▲
feature/my-numbers* ──> core/my-numbers ───────────────┤
feature/statistics ───> core/statistics ───────────────┘
FCM 데이터 메시지 ──> feature/messaging ──> core/winning-checker ──> 로컬 알림
```

- **당첨번호 조회**: 최신 회차는 메모리 캐시 -> Firestore, 특정 회차는 로컬 DB -> Firestore, 전 회차는 최신 회차 비교 후 외부 API 일괄 저장. Firestore 응답에서 0 인 금액·당첨자 수는 같은 회차의 기존 행 값으로 채운다(백엔드가 1등 당첨금만 채우기 때문).
- **로컬 DB**: SQLCipher 로 암호화한 Room DB(`lotto`, `my_numbers`, `favorite_numbers` 3테이블). 키는 Android Keystore 로 보호한 passphrase 를 공용 모듈에서 만든다. 파괴적 마이그레이션을 켜지 않고 명시적 `Migration` 으로만 올린다. 사용자가 저장한 번호를 스키마 변경으로 조용히 지우지 않기 위해서다.
- **당첨 확인**: 단말이 주기적으로 깨어나 확인하지 않는다. 발표 시각을 아는 쪽이 서버라서 서버가 FCM 데이터 메시지(`lotto_check`, `lotto_purchase_reminder`)를 보내면 `feature/messaging` 의 핸들러가 `core/winning-checker` 를 호출한다. 푸시 payload 의 당첨번호로 대조해 DB 동기화를 기다리지 않는다.
- **카운트다운**: `core/countdown` 은 시각을 인자로만 받는 순수 판정이다(`TimeRemaining` / `DrawInProgress` / `ResultAnnounced`). 분 경계 갱신 상태홀더는 `feature/home` 에 둔다. 서버 시각 보정은 `time-sync` 공유 모듈이 맡는다.
- **시각 처리**: 엔티티와 도메인 모델의 `createdAt`·`updatedAt` 에 기본값을 두지 않고 호출부가 `TimeProvider` 로 채운다.

## UI 패턴

- 상태는 `state` 로 읽고 사용자 액션은 `onIntent(Intent)` 로만 보내는 MVI 스타일이다. 토스트·URL 열기 같은 1회성 이벤트는 별도 채널로 보낸다.
- 화면은 Paper 컴포넌트(티켓 스텁 디자인)로 분해돼 있다. 분해 직후 `HomeScreen` 760 -> 158줄, `MyNumbersScreen` 1,353 -> 324줄, `StatisticsScreen` 1,520 -> 249줄이 됐다.
- 스플래시·설정 화면 뼈대는 공용 `SplashScaffold`, `SettingsScaffold` 에 위임해 앱마다 문구와 모양이 갈라지지 않게 한다.
- 통계 ViewModel 은 UseCase 호출마다 예외를 따로 받아 한 섹션의 실패가 다른 섹션을 비우지 않는다.

## 설정과 시크릿

- 설정은 `clients/lottomate/config.json` 의 `common`·`dev`·`prod` 중첩 구조다. 원래 이중 키 워크어라운드가 있었으나 광고 앱 id 만 갈리면 되므로 이 구조로 재설계했다.
- 빌드 시 설정 생성 스크립트가 XOR 암호화 C 소스를 만들고 NDK 로 빌드해 JNI 로 복호화한다. 이렇게 하면 소스 리터럴과 달리 dex 에 평문이 남지 않는다. 키마다 접근자가 생성되므로 키를 지우면 그것을 부르는 코드가 컴파일되지 않는다.
- 광고 앱 id 는 `manifestPlaceholders` 로 주입하고 배너 id 가 비어 있으면 배너를 그리지 않는다. 운영 광고 키에 구글 테스트 id 가 포함돼 있으면 prodRelease 가 검증에서 멈춘다.
- 딥링크 스킴 문자열이 코드 상수·매니페스트·`preBuildVerify` 세 곳에 있어, 앞의 둘만 빌드 검증이 대조한다는 점을 주의해야 한다.

## 주요 설계 결정

| 날짜 | 결정 | 이유 |
|---|---|---|
| 2026-10 | 데이터·도메인을 `core` 로 이동 | 화면 모듈 간 순환 참조 제거 |
| 2026-10 | 브랜드 테마를 한 곳(`BrandTheme`)에서 정의 | Material 주색과 종이 포인트 색이 갈려 있었음 |
| 2026-09 | `notifications` 딥링크를 SELF 전용으로 | 외부 앱의 알림함 열기 차단 |
| 2026-08 | 앱 직속 코드를 feature 모듈 5개로 분해 | 저장소에서 가장 큰 앱 직속 코드 해소 |
| 2026-08 | 딥링크 화이트리스트와 라우터를 한 맵에서 파생 | 두 곳 불일치로 링크가 조용히 죽는 문제 |
| 2026-08 | SDK 난독화 규칙을 소유 모듈의 `consumer-rules.pro` 로 이관 | 앱에 규칙을 쌓으면 새 앱에서 빠지고 정본이 둘이 됨 |
| 2026-08 | `prodDebug` 에도 배포 서명 적용 | 내부 테스터 배포에서 인증서가 다르면 설치가 갈림 |
| 2026-06 | 전 화면 티켓 스텁 디자인으로 통일 | 홈에서 시작한 컨셉을 앱 전체로 확장 |

## 알려진 유의사항

- 탭 뱃지 상태(`BadgeState`)는 값을 쓰는 코드가 없어 항상 0 이다.
- 앱 `allowBackup` 이 꺼져 있어 재설치 시 내 번호가 복원되지 않는다.
- 릴리스에서 `android.util.Log` 호출을 모두 제거하므로 릴리스 원인은 Crashlytics 로 본다.
- 스플래시 테마 속성은 API 31 부터 적용되어 minSdk 29~30 기기는 기본 시작 창을 본다.
