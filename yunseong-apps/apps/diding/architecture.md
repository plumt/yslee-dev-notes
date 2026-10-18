# Diding - 아키텍처

## 모듈 구성과 계층

```
:apps:diding                   앱 조립 (Application, MainActivity, NavGraph, di/)
 ├─ feature:home               오늘 탭
 ├─ feature:management         관리 탭
 ├─ feature:calendar           캘린더 탭
 ├─ feature:setting            설정 탭
 ├─ core:alarm                 정확 알람 스케줄러 + 리시버 2개
 ├─ core:database              Room DB, 저장소 구현, 알람 포트, 알림 스위치
 └─ core:domain                모델·저장소 계약·판정·유스케이스 (순수 Kotlin 성격)
```

의존 방향은 `feature -> core:domain` 이고, 구현은 `core:database` 가 갖는다.

- `core:domain` 은 Room·Hilt·Compose 를 모른다. 저장소 인터페이스와 모델만 있고 구현과 Hilt 바인딩은 `core:database` 소관이다. 유스케이스에는 `@Inject` 를 달지 않고 ViewModel 이 저장소를 넘겨 만든다.
- `core:database` 는 Room 을 `implementation` 으로 가두고 `core:domain` 을 `api` 로 재노출한다. 그래서 database 를 붙인 쪽의 컴파일 경로에는 계약 타입만 오르고 Room 은 오르지 않는다. 저장소 구현체(`*RepositoryImpl`)는 `internal` 이다.
- 화면(feature)은 `core:domain` 계약만 본다. 홈·캘린더는 `core:database` 를 아예 의존하지 않고, 관리·설정만 알람 스케줄러 포트를 쓰려고 의존한다(DAO·엔티티는 참조하지 않는다).
- `core:alarm` 은 `MainActivity`·앱 `R` 을 직접 참조하지 않고 공용 `AppLaunchInfo` 포트를 주입받는다. 직접 참조하면 core 가 앱을 거꾸로 의존한다. `AppLaunchInfo` 바인딩은 앱 모듈에만 둔다.
- 도메인 모듈을 `shared/core` 로 올리지 않은 이유는 요일 규약(월=1 ... 일=7)과 날짜 키 형식이 이 앱의 약속이라 올리면 소비 앱 전체의 계약이 되기 때문이다. 계층 가드가 다른 앱의 참조도 막는다.

## 화면 배선

앱은 화면을 나열하지 않는다. feature 모듈 넷이 `FeatureDestination` 을 `@IntoSet` 으로 올리고 `DidingNavGraph` 는 레지스트리의 집합만 편다.

| 앱이 소유 | feature 가 소유 |
|---|---|
| 탭 라벨·아이콘(`BottomNavItem`) | route 문자열 상수와 등록 순서 |
| `Scaffold`·바텀바·`NavHost` 골격 | `composable(ROUTE)` 등록과 화면 본문 |

- 탭 전환 동작(선택 판정, `popUpTo`, 상태 저장·복원)은 공용 `AppBottomNavigationBar` 가 정한다. 이 앱은 탭 목록과 testTag 만 넘긴다.
- `Scaffold` 는 앱이 직접 둔다. 바텀바를 숨기는 화면이 없어 공용 셸의 표시 상태 제어가 필요 없기 때문이다.
- 화면을 추가하는 순서: 앱에 feature 의존 추가 -> (탭이면) `BottomNavItem` 추가 -> (딥링크면) `HOST_TO_ROUTE` 에 host 추가.

## 딥링크

`HOST_TO_ROUTE` 한 map 이 화이트리스트와 라우터 매핑을 함께 낳는다. 둘을 따로 적으면 한쪽만 고쳤을 때 딥링크가 조용히 거부되거나 이동하지 않는다.

- host 네 개(home, management, calendar, setting), 경로 없는 형태만 허용, 쿼리는 버린다. 신뢰 수준은 PUBLIC
- 자기 패키지는 `BuildConfig.APPLICATION_ID` 로 판정해 dev 의 `.dev` 접미사를 따라간다. 자사 앱 연동과 App Links 는 쓰지 않는다
- 콜드 스타트 URI 는 `MainActivity.onCreate` 가, 실행 중 링크는 `onNewIntent` 가 큐에 넣고 `DeepLinkWiringEffect` 가 `NavController` 에 붙은 뒤 소비한다
- 탭 클릭과 달리 `popUpTo`·`launchSingleTop` 이 없어 같은 링크를 거듭 열면 같은 목적지가 백스택에 쌓인다

## 데이터 모델과 저장

- Room DB(스키마 v1)는 `todo`(제목, 반복 유형 문자열, 요일 문자열, 월 일자, 알림 켜짐, 알림 시·분, 보관 여부, 생성 시각)와 `completion`(todo_id + 완료 날짜 복합키, 완료 시각, 삭제 CASCADE) 두 테이블이다.
- DB 는 공용 암호화 DB 팩토리로 연다. 파일 이름과 Keystore 별칭이 DB 이름에서 파생되므로 `DATABASE_NAME` 을 바꾸면 기존 DB 를 열지 못한다.
- 완료는 "날짜별 기록"이다. 반복 할 일의 같은 날 같은 할 일 완료는 기본키에서 한 건으로 합쳐지고, 완료 해제는 그 행을 지운다.
- 이미 저장된 할 일은 반드시 `update` 로 고친다. id 가 있는 할 일을 `add`(REPLACE)로 다시 넣으면 외래키 CASCADE 로 완료 이력이 지워진다.
- 시각은 엔티티 기본값이 아니라 `TimeProvider` 로 정한다. 기본값이 있으면 호출부가 빠뜨렸을 때 조용히 기기 시각이 박힌다.

## 반복 규칙과 날짜 규약

반복은 `sealed interface TodoRepeat`(Daily, Weekly, Monthly, None)이다. 새 종류가 생기면 `else` 없는 `when` 이 컴파일 오류로 알려 준다.

| 규칙 | 해당 조건 | 경계 |
|---|---|---|
| Daily | 언제나 | 없음 |
| Weekly | 요일이 지정 집합에 든다 | 빈 집합이면 어느 날도 해당 없음 |
| Monthly | 일자 == min(지정일, 그 달 일 수) | 29~31일 지정은 일자가 없는 달의 말일에 해당 |
| None | 없음 | 어느 날도 해당 없음 |

- 요일은 월=1 ... 일=7 (`Calendar.DAY_OF_WEEK` 는 일=1 이라 변환 함수를 거친다)
- 날짜 키는 0 을 채운 `yyyy-MM-dd` 문자열(`CompletionDateKey`)이다. 이 형식에서만 문자열 사전순이 날짜순과 같아 기간 조회가 문자열 범위 비교(BETWEEN)로 성립한다. 로캘과 무관하게 ASCII 숫자로 만든다
- 오늘 목록 필터를 SQL 로 내리지 않는다. 반복 판정이 도메인 타입 안에 있어 질의로 옮기면 규칙이 둘로 갈리기 때문이다. 홈 목록과 캘린더 수행률 분모가 같은 판정 함수를 불러 같은 날 같은 답을 낸다

## 오늘 화면과 시간 처리

- 기준 시각은 `TimeProvider.dayTicks()` 가 기기 시간대 자정마다 새로 내고, 그 시각 하나에서 날짜 키·요일·일자·라벨을 모두 뽑아 한 화면이 같은 날을 본다
- 구독 중일 때만 자정 대기가 돈다. 화면이 보이지 않으면 대기도 조회도 멈춘다
- 완료 토글은 화면 상태를 먼저 바꾸지 않는다. 저장 뒤 완료 Flow 가 다시 내보낼 때 체크 표시가 바뀐다
- 기기 시계를 직접 읽지 않는 규칙은 시각 의존 로직을 고정 시각으로 테스트하기 위함이다

## 알람 파이프라인

```
앱 시작 / 설정 토글 / 할 일 저장·보관 / 발화 후 / 부팅·앱 교체
        └─> TodoAlarmScheduler.scheduleAll()   (IO 디스패처)
              0. 마스터 스위치 OFF 면 전부 취소하고 종료
              1. 보관되지 않은 + 알림 켜진 할 일만 선별
              2. 기존 PendingIntent 를 찾아 알람 취소
              3. 할 일마다 다음 발화 시각 계산 -> one-shot 정확 알람 등록
TodoAlarmReceiver (발화) -> 재검증 -> 알림 발송 -> scheduleAll() 로 다음 발화 예약
```

- 반복 알람 API 대신 매번 one-shot 을 다시 건다. 주간·월간 규칙이 고정 간격이 아니기 때문이다. 요청 코드가 할 일 id 라 여러 번 호출해도 할 일당 알람은 하나다.
- 마스터 스위치 판정을 `scheduleAll()` 의 0단계에 둔 이유: 예약을 요청하는 경로가 여섯이라 판정을 호출부에 두면 하나만 빠뜨려도 꺼 둔 알림이 되살아난다. 꺼져 있을 때는 그냥 돌아가지 않고 거둔다(켜 둔 동안 예약된 알람이 시스템에 남아 있기 때문이다).
- 발화 시점에도 보관 여부와 알림 여부를 다시 본다. 예약 뒤 상태가 바뀌었을 수 있다. 알림 발송이 실패해도 재예약은 따로 `try` 로 감싸 돈다.
- 리시버는 `goAsync()` 로 IO 에서 처리하고 `finally` 에서 `finish()` 를 부른다.
- 다음 발화 시각: Daily 는 오늘 후보가 지났으면 +1일, Weekly 는 오늘부터 8개 날짜를 훑어 첫 일치일, Monthly 는 일자를 말일로 clamp 한 뒤 지났으면 다음 달(다시 clamp).
- 리시버는 `android:permission` 을 걸지 않는다. 세 액션이 모두 protected broadcast 인데 검사를 걸면 `MY_PACKAGE_REPLACED` 가 조용히 배달되지 않는다.
- 리시버 선언은 앱이 아니라 `core:alarm` 매니페스트에 둔다. 의존에 넣는 행위 자체가 배선이 되게 하려는 것이다. 권한(`RECEIVE_BOOT_COMPLETED`, `SCHEDULE_EXACT_ALARM`/`USE_EXACT_ALARM`, `POST_NOTIFICATIONS`)도 그것을 쓰는 모듈이 선언해 병합된다.
- 전면 취소 대상 id 는 저장소에서 받는다. id 범위를 훑으면 호출 스레드가 멈추고 상한 밖 id 가 남는다.

## 앱 시작

`DidingApplication.onCreate` 는 알림 채널 `diding_todo` 를 만들고 `scheduleAll()` 을 `SupervisorJob + Dispatchers.IO` 스코프에서 부른다. 스코프 문맥의 `coroutineErrorHandler` 를 빼면 재예약 예외가 콜드 스타트를 죽이므로 반드시 둔다. 알림 설정 확인은 시작 코드가 하지 않고 스케줄러에 맡긴다.

## 관리 화면의 상태 모델

관리 화면은 MVI(State, Intent, Effect 없음)를 쓴다. 공개 액션이 네 개 이상이라 `onIntent` 단일 진입점이 낫고, 하위 컴포저블은 콜백만 받아 ViewModel 을 모른다. 목록은 Room Flow 가 출처라 로딩·오류 상태를 타입에 만들지 않는다. 저장은 시트를 먼저 닫은 뒤 재예약을 부른다(재예약 실패가 저장 실패 처리로 번져 시트가 남고 재저장 시 중복되는 것을 막는다). 시트 입력의 반복 유형은 문자열 토글이고 도메인 타입 변환은 `fromTypeCode`/`typeCode` 로만 한다.

## 설정 화면과 권한

알림 켜기는 권한이 `Granted` 일 때만 설정 값과 토글을 올리고, 영구 거부면 토글을 끄고 앱 설정으로 안내한다. 권한 확보, 설정 저장, 알람 예약/취소는 `TodoNotificationToggleUseCase` 하나로 묶는다. 설정 키 문자열의 사본을 만들지 않는다(설정 화면과 스케줄러가 같은 값을 읽어야 어긋나지 않는다).

## 빌드

- 규약 플러그인 여섯 줄이 빌드 구성 전부다. 선언 순서는 `ModuleConventionGuard` 가 검사한다
- 플레이버·서명은 규약 플러그인이 만들고 앱에서 다시 선언하지 않는다
- `preBuildVerify` 는 설정 JSON 구조, 매니페스트 딥링크 스킴, release 서명만 검사한다(필수 키 목록과 광고 키 목록은 비어 있다)
- 릴리스는 R8 축소를 켠다. 앱 규칙 파일은 `keepattributes` 뿐이고 SDK 규칙은 각 모듈 몫이다

## 주요 설계 결정

| 시기 | 결정 | 이유 |
|---|---|---|
| 2026-10 | 인라인 바텀바를 공용 `AppBottomNavigationBar` 로 교체 | 동작 차이가 재선택 하나만 남았다 |
| 2026-09 | 시작 시 알림 설정 판정 삭제, 스케줄러에 일임 | 같은 키 사본이 두 곳에 있어 판정이 갈라질 수 있었다 |
| 2026-09 | `appScope` 에 `coroutineErrorHandler` 추가 | 재예약 예외가 IO 스레드 미처리 예외로 콜드 스타트를 끝냈다 |
| 2026-08 | 화면 나열을 NavGraph 에서 feature 자가등록으로 이동 | 화면을 더할 때마다 앱을 고쳐야 했고 route 가 두 곳에 있었다 |
| 2026-08 | 딥링크 host 매핑을 `HOST_TO_ROUTE` 로 통합 | 한쪽만 고치면 조용히 실패했다 |
| 2026-08 | 화면의 Room DAO 직접 사용을 도메인 계약으로 분리 | 저장 스키마가 바뀌면 UI 가 곧바로 깨졌다 |
| 2026-08 | 테마 2단 중첩을 `AppThemeHost` 로 대체 | 다섯 앱이 같은 조립을 손으로 반복했다 |

## 알려진 함정

- `allowDestructiveMigration = true` 인 채 스키마 버전을 올리면 데이터가 전부 삭제된다
- `TodoAlarmReceiver` 의 FQCN 과 `ACTION_FIRE` 문자열을 옮기거나 바꾸면 이미 예약된 알람이 발화도 취소도 되지 않는다
- 앱은 `Logger.init` 을 부르지 않아 릴리스에서는 시작 예외 기록이 남지 않는다
- 앱 모듈의 단위 테스트는 없다. 알람 계산 테스트만 `core:alarm` 에 있다
