# shared/core/firebase

6개 모듈. Gradle 경로는 `:shared:core:firebase:<이름>`. Firebase SDK 를 Hilt 와 코루틴 환경에 맞게 얇게 감쌉니다.

| 모듈 | 목적 | 핵심 기술 | project 의존 |
|---|---|---|---|
| core | Firebase BOM, `@FirebaseGson`, INTERNET 권한의 뿌리 | Firebase BOM, Hilt | 없음 |
| auth | FirebaseAuth 콜백 API 를 suspend 로 변환 | firebase-auth, 코루틴 | core |
| crashlytics | `CrashReporter` 포트 구현 | Crashlytics | core, fundamental/common |
| firestore | 앱이 지정한 DB 이름으로 Firestore 제공 | Firestore, Hilt | core |
| messaging | FCM 수신 서비스, 타입 기반 라우팅, 토픽 구독 | FCM, Hilt Multibinds | core, fundamental/common |
| remote-config | Remote Config fetch 와 키 조회 | Remote Config, Gson | core, fundamental/common |

## 모듈별 설명

- **core**: firebase 계열의 최하위 모듈입니다. Firebase BOM 버전을 올리는 곳이 여기 하나뿐이고, 새 Firebase 모듈은 이 모듈을 먼저 의존합니다. `INTERNET` 권한도 여기서 한 번만 선언하고 `@FirebaseGson` 을 제공합니다. `retrofit-gson` 은 `api` 로 노출하지 않고, 직렬화 DTO 는 두지 않습니다.
- **auth**: 콜백 기반 인증 API 를 suspend 함수로 감쌉니다. 예외는 잡지 않고 호출부가 성공, 실패를 분기하며, `FirebaseAuth` 인스턴스는 이 모듈만 제공해 중복 바인딩을 막습니다.
- **crashlytics**: `CrashlyticsManager` 가 `fundamental/common` 의 `CrashReporter` 포트를 구현합니다. 스택트레이스의 파일명과 줄번호를 유지하는 `-keepattributes SourceFile,LineNumberTable` 규칙을 `consumer-rules.pro` 로 전파합니다.
- **firestore**: `FirebaseFirestore` 를 `@FirestoreDatabaseId` 로 앱이 바인딩한 데이터베이스 이름으로 만듭니다. 기본값이 없어 빠뜨리면 `Dagger/MissingBinding` 으로 빌드가 멈춥니다. Repository 같은 데이터 접근 로직은 두지 않습니다.
- **messaging**: `AppFcmService` 가 의존만으로 앱 매니페스트에 병합됩니다. 수신 메시지는 `canHandle` 조건으로 타입에 맞는 핸들러에 라우팅되며 핸들러는 Hilt `@IntoSet` 으로 등록합니다. `@Multibinds` 선언이 있어 핸들러가 0개여도 컴파일됩니다. 서비스 keep 규칙을 지우면 R8 릴리스에서만 수신이 멈춥니다.
- **remote-config**: fetch 와 키 조회를 한 클래스로 감쌉니다. 앱 중립 키 이름만 `RemoteConfigConstants` 에 두고 기본값은 호출부가 넘깁니다. 실패는 값으로 접고 취소만 전파하며, `AppVersion` 모델(앱 업데이트 판정용)은 R8 keep 규칙과 함께 관리합니다.

## 공통 사항

- 이 모듈들에 닿는 앱은 `google-services` 플러그인을 적용해야 하며, 없으면 `FirebaseConfigGuard` 빌드 가드가 실패시킵니다(remote-config 는 `shared/feature/app-update` 경유 경로를 제외).
- 로깅은 `fundamental/common` 의 `Logger` 를 씁니다.
- 직렬화 DTO 에 keep 규칙이 없으면 `ConsumerRulesGuard` 가 막습니다.
