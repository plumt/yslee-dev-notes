# yunseongApps 아키텍처

## 개요

yunseongApps는 Android 앱 6개(Lottomate, Lunar, QRScanner, Diding, Moduta, Fxrate)를 담은 멀티 앱 모노레포입니다.
Clean Architecture와 Modular Architecture를 기반으로, 새 앱을 만들 때 기존 모듈을 조립하는 것이 기본이 되도록 설계했습니다.
규칙은 문서에만 두지 않고 Gradle Convention Plugin의 Guard로 빌드 시점에 강제합니다.

## 모듈 계층 구조

```
┌──────────────────────────────────────────────────────┐
│              Apps (apps/*)                           │  조립 전용: Application, MainActivity,
│  lottomate, lunar, qrscanner, diding, moduta, fxrate │  내비게이션, DI 바인딩
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│   App Core / App Feature (apps/<app>/core, feature)  │  앱 전용 도메인·데이터·화면
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│         Shared Feature (shared/feature/*)            │  여러 앱이 공유하는 기능 (30개)
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│         Shared Core (shared/core/*)                  │  공통 인프라 (52개)
│  fundamental / common / firebase / permissions / lunar│
└──────────────────────────────────────────────────────┘
```

총 135개 모듈(MODULE_INDEX 기준)입니다. Shared Core는 fundamental 5, permissions 11, common 29, firebase 6, lunar 1이고,
Shared Feature는 30개, 앱 전용 모듈은 앱마다 5~15개입니다.

shared 모듈 문서는 [shared/core](shared/core.md), [shared/feature](shared/feature.md)에 있습니다.

## 의존성 규칙

| 모듈 | 허용 | 금지 |
|------|------|------|
| App | Shared Core, Shared Feature, 자기 앱의 core/feature | 다른 앱 |
| App Feature | 자기 앱 core, Shared Core, Shared Feature | 다른 feature(같은 앱 포함, 화면 부품 재사용만 사유와 함께 예외 등록) |
| App Core | Shared Core, 자기 앱의 다른 core | 같은 앱의 feature(계층 역전) |
| Shared Feature | Shared Core, 같은 feature 그룹 내부 형제 모듈 | 다른 그룹의 shared feature, 앱 |
| Shared Core | 다른 Shared Core(신중하게) | feature, 앱 |

### 빌드가 강제하는 6규칙 (`DependencyLayerGuard`)

1. `shared/*`는 `apps/*`를 참조할 수 없다 (역참조 금지)
2. `apps/A`는 `apps/B`를 참조할 수 없다 (앱 간 교차 금지)
3. `shared/core`는 `shared/feature`를 참조할 수 없다 (계층 역전 금지)
4. `shared/feature` 그룹 간 교차 참조 금지 (같은 그룹 내부 형제는 허용)
5. 앱 feature 간 참조 금지 (화면 부품 재사용만 allowlist)
6. 앱 core는 같은 앱의 feature를 참조할 수 없다

위반하면 configure 단계에서 빌드가 실패합니다. 규칙 4의 allowlist는 현재 비어 있습니다.
모듈 안쪽 계층(`domain` 패키지가 Room, Retrofit, `android.*`를 import하지 않기)은 별도 검사 스크립트가 봅니다.

### 포트 인터페이스 패턴

방향 규칙만으로는 "공용 모듈이 앱 전용 구현을 필요로 할 때" 답이 없어 역참조로 메워지기 쉽습니다.
이럴 때는 공용 모듈이 계약(인터페이스)만 소유하고 각 앱이 구현·바인딩합니다.

딥링크가 대표 사례입니다. `DeepLinkRouter` 계약은 deeplink 모듈이 갖고, `navigation-base`는 host-to-route 매핑 표만 받는
표준 구현체(`MapRouteDeepLinkRouter`)를 제공합니다. 앱은 매핑 표만 주입하므로 앱마다 라우터 클래스를 새로 만들지 않습니다.
`AppLaunchInfo`처럼 하위 모듈이 `MainActivity`를 알아야 할 때도 직접 참조하지 않고 포트로 받습니다.

## 앱 모듈 규격

앱은 주제만 다르고 뼈대는 같아야 하므로 세 가지를 빌드로 막습니다.

- 앱 모듈 패키지 루트에는 `Application`, `MainActivity`, `di/`, `navigation/`만 허용합니다 (`AppModuleShapeGuard`).
  `di/` 아래로 숨기는 우회와 이름을 바꾼 Service·Receiver·Provider·Activity·Worker도 기능으로 보고 막습니다.
- 화면 모듈은 `convention.feature`를 씁니다. `library + hilt + compose` 손조합은 금지입니다 (`ModuleConventionGuard`).
- Firebase 모듈을 쓰면 `google-services` 설정이 있어야 하고 (`FirebaseConfigGuard`), 쓰는 권한의 Provider 모듈을 의존해야 합니다 (`PermissionProviderGuard`).

예외 목록(`ALLOWLIST`)은 비어 있고, 6개 앱이 모두 규격을 지킵니다.

## 새 기능을 어디에 만드는가

1. 만들기 전에 모듈 인덱스에서 같은 일을 하는 모듈이 있는지 확인합니다. 있는데 모양이 안 맞으면 새로 만들기보다 그 모듈을 고칩니다.
2. 배치 판정: 앱 고유 도메인이면 `apps/<앱>`, 화면이 있으면 `shared/feature`, 없으면 `shared/core`
   (홀로 서면 `fundamental`, 아니면 `common`).
3. 승격은 두 번째 앱이 필요로 할 때 합니다. 미리 shared로 올리면 쓰이지 않는 재고만 늘어납니다.
4. `shared/`에 두는 기준은 "여러 앱이 쓴다"가 아니라 "어떤 앱에 넣어도 동작한다"입니다.
   권한은 쓰는 모듈의 매니페스트가 선언하고, DB 이름이나 채널 id는 모듈에 하드코딩하지 않습니다.

## 확장점 규약

공용 모듈에 확장점을 둘 때는 메커니즘보다 "앱이 바인딩을 빠뜨리면 무슨 일이 일어나야 하는가"를 먼저 정합니다.

| 부재 시 있어야 할 결과 | 메커니즘 |
|---|---|
| 그 기능을 안 쓰는 게 정답 (옵트인) | `@BindsOptionalOf`, 소비처에서 건너뜀 |
| 참여하는 것만 모으면 됨 | `@Multibinds` + `@IntoSet` |
| 동작하는 기본값이 필요 | `@BindsOptionalOf` + `orElseGet { 기본값 }` |
| 앱마다 반드시 달라야 함 | 기본 바인딩을 만들지 않아 누락 시 컴파일 실패 |
| 앱이 쓰겠다고 한 기능이 조용히 안 도는 것 | 금지 (no-op 기본 바인딩 등) |

판정 질문은 "앱이 그 기능을 쓰려고 의도했는가"입니다. 의도했는데 배선이 빠졌다면 컴파일 실패로 만들어야 하고,
조용히 안 도는 구조는 만들지 않습니다.

## 아키텍처 패턴

### 화면 공장화 골격 (`convention.feature` + `presentation-base`)

신규 화면 모듈은 `convention.feature` 하나만 적용하면 library + hilt + compose, `presentation-base`,
`ui-base`, `designsystem`, 테스트 의존(JUnit, MockK, coroutines-test, Turbine)이 준비됩니다.

상태 홀더는 공개 액션 수로 고릅니다.

| 상황 | 베이스 |
|---|---|
| 공개 액션 0~3개 단순 화면 | `StateViewModel<S, E>` |
| 공개 액션 4개 이상, 또는 UI/ViewModel 독립 개발 | `MviViewModel<S, I, E>` (단일 `onIntent`) |
| 상태·액션이 없는 값 전달 | 평범한 `ViewModel` |

설계 근거는 다음과 같습니다.
- `UiState`는 `data class`로 둡니다. `sealed` 계층은 `copy` 부분 갱신이 안 돼 로딩과 에러가 겹치는 화면을 표현하지 못합니다.
- `LoadState<T>`가 `isLoading`/`error`/`data` 세 필드를 대체해 "로딩 중이면서 에러" 같은 불가능한 조합을 타입으로 막습니다.
- 효과(effect)는 `replay = 0`입니다. 화면이 없는 동안 쌓아 두면 복귀 시 과거 내비게이션이 한꺼번에 실행됩니다.
- 기존 모듈을 일괄 이관하지 않고, 신규 화면부터 적용하며 손댈 일이 생길 때 옮깁니다.

### 앱 셸과 화면 자가등록

- `app-shell`: 바텀 탭 바(`AppBottomNavigationBar`), Scaffold 셸(`AppShell`), 딥링크 호스트 Activity(`DeepLinkHostActivity`),
  라우터 포트 바인딩을 공용화했습니다. 앱은 탭 목록, 테마, NavGraph 호출만 남깁니다.
- `navigation-base`의 `FeatureDestination` 계약으로 feature 모듈이 자기 화면을 `@IntoSet`으로 등록하면, 앱 NavGraph는
  등록된 목적지를 소비만 합니다. 옵트인이라 등록이 없는 앱은 빈 집합으로 기존 방식 그대로 동작합니다.
  앱 정책을 담는 화면(스플래시 등)은 앱이 계속 소유합니다.

### api/impl 컨테이너 모듈 패턴

무거운 의존성(Room+KSP, 생체인증, ML Kit, Compose)을 소비자 컴파일 경로에서 빼기 위해 계약과 구현을 분리합니다.
상위 디렉터리는 `build.gradle.kts`가 없는 컨테이너입니다.

```
shared/feature/notification-inbox/
├── notification-inbox-api/    도메인 모델 + Repository 계약 (Room·KSP·Compose 없음)
├── notification-inbox-impl/   Room + SQLCipher + FCM 핸들러 + DI 바인딩
└── notification-inbox-ui/     Compose 화면 + ViewModel (api만 의존)
```

적용 대상은 `notification-inbox`, `lock`, `qr:common`, `qr:multi-scanner`입니다. 핵심 규칙은 다음과 같습니다.
- impl의 Room은 `implementation`으로 선언합니다. `api`로 노출하면 소비자 classpath로 다시 전파돼 분리 효과가 사라집니다.
- api 모듈에는 `convention.android.hilt`를 적용하지 않습니다 (KSP 태스크가 생기기 때문).
- Room `@Entity`는 api에 두지 않고, 공개 API가 ML Kit, Biometric, Compose 타입을 노출하지 않게 합니다.
- 앱은 보통 `-impl`과 `-ui`를 의존합니다.

### Provider + Registry 패턴 (권한, 소셜 로그인)

권한과 소셜 로그인은 계약을 한 모듈에, 구현을 기능별 모듈에 두고 Hilt 멀티바인딩으로 모읍니다.

- 권한: `fundamental:permission`이 `PermissionProvider` 계약, `PermissionProviderRegistry`, UseCase를 갖고,
  `permissions/permission-camera` 같은 구현 모듈이 의존 추가만으로 `@IntoSet` 등록됩니다.
  `@Multibinds` 선언 덕에 Provider가 하나도 없는 앱도 빈 Set이 주입됩니다.
  등록되지 않은 권한 타입은 런타임 예외가 나므로 `PermissionProviderGuard`가 빌드 시점에 의존 누락을 잡습니다.
- 소셜 로그인: `social-login-core` 계약과 Google, Kakao, Naver 구현을 모듈로 분리해, 카카오만 쓰는 앱의 컴파일 경로에서
  Credential Manager, Firebase Auth, 네이버 SDK가 빠집니다. 미등록 타입은 예외 대신 `NotSupported` 결과를 돌려줍니다.
  키는 앱이 NativeConfig 경유로 제공합니다.

### 시간 보정 패턴 (`TimeProvider` + `TimeCorrection`)

시각 의존 로직은 `System.currentTimeMillis()` 직접 호출 대신 `TimeProvider`를 주입받아 테스트에서 시각을 고정할 수 있게 합니다.
`time-sync` 모듈을 의존에 추가하면 `TimeCorrection`이 `@IntoSet`으로 등록돼 서버 기준 보정 시각으로 자동 승격됩니다.
Hilt는 다른 모듈의 바인딩을 대체할 수 없어 교체가 아니라 위임 구조를 씁니다. 보정이 실패하거나 null이면 기기 시간으로
폴백하는 것이 계약의 일부입니다.

### 딥링크 화이트리스트

커스텀 스킴은 OS가 발신 앱을 제한하지 않으므로 네 겹으로 막습니다.

| 겹 | 수단 | 막는 것 |
|----|------|--------|
| 1 | host/path 화이트리스트 (`DeepLinkRoute` `@IntoSet`) | 등록하지 않은 경로 진입 |
| 2 | 발신자 신뢰 수준 (`PUBLIC` < `FIRST_PARTY` < `SELF`) | 외부 앱이 민감 화면을 여는 것 |
| 3 | 쿼리 파라미터 화이트리스트 | 임의 파라미터 주입 |
| 4 | `DeepLinkTarget` 생성자·`copy()`를 `internal`로 | 검증을 우회한 인스턴스 생성 |

referrer 판정은 위조 가능하므로 심층 방어의 한 겹으로만 쓰고, 결제나 계정 변경 같은 동작은 앱 내 인증을 다시 요구합니다.

## 빌드 시점 검증

Convention Plugin이 "빌드는 성공하지만 런타임이나 출시 후에야 드러나는" 문제를 미리 막습니다.
플러그인과 Guard 목록은 [docs/build-and-tooling.md](docs/build-and-tooling.md)에 정리했습니다.

| 검사 (`convention.verify.prebuild`) | debug | release |
|---|---|---|
| 필수 설정 키가 비어 있음 | 경고 | 실패 |
| 구글 테스트 AdMob 퍼블리셔 ID 사용 | - | 실패 |
| 매니페스트 딥링크 스킴 불일치 | 실패 | 실패 |
| release 서명 키스토어·자격증명 누락 | - | 실패 |
| `versionCode`가 최소값 미만 | - | 실패 |

심각도를 나눈 이유는, debug에서 막으면 개발자가 우회 플래그를 상시 켜게 되어 검증 자체가 무의미해지기 때문입니다.
개발 중에는 경고, 출시 경로에서는 차단합니다. 검증 태스크는 variant별로 만들어 `assembleDebug`가 release 규칙에 막히지 않게 합니다.

## ProGuard/R8 규칙 배치

- SDK 규칙은 그 SDK를 쓰는 모듈의 `consumer-rules.pro`에 둡니다. 앱 파일에 두면 새 앱이 모듈을 붙일 때마다 복사해야 하고,
  빠뜨리면 release에서만 조용히 깨집니다.
- 앱 `proguard-rules.pro`에는 앱 고유 정책만 둡니다.
- Gson·SimpleXML 같은 리플렉션 직렬화 DTO는 반드시 keep합니다. 난독화되면 예외 없이 전 필드가 null이 됩니다.
- JNI 심볼 보존 규칙(`proguard-nativeconfig.pro`)은 NativeConfig 플러그인이 자동 생성합니다.

## 보안 아키텍처: NDK NativeConfig

런타임 API 키를 BuildConfig에 넣으면 APK에 평문으로 남기 때문에, 앱이 런타임에 읽는 값은 네이티브(C) 레이어로 숨깁니다.

```
clients/<app>/config.json            키의 유일한 출처
  → build-logic generate_native_config.py   XOR 암호화된 C 소스, CMakeLists, Kotlin 래퍼, ProGuard 규칙 생성
  → CMake가 .so로 컴파일
  → NativeConfig.<getter>()가 JNI로 런타임에 복호화
```

- `convention.ndk.nativeconfig`를 적용하면 배선이 끝납니다. 모듈 경로에서 client·library·package를 추론합니다.
- XOR 키는 필드명의 SHA-256에서 파생해 결정론적이고 증분 빌드에 안정적입니다.
- Kotlin 래퍼까지 생성하는 이유는, 손으로 쓰던 래퍼가 config 변경을 따라가지 못해 실제로 드리프트가 났기 때문입니다.
- `keys`를 지정하면 그 모듈의 `.so`에 필요한 키만 들어갑니다.
- 매니페스트 meta-data로 SDK가 직접 읽는 값(지도 SDK 키, AdMob App ID)은 NDK로 숨길 수 없어
  `loadClientConfig()`로 읽어 placeholder로 주입합니다.
- 서명 자격증명은 APK에 들어가지 않는 빌드 전용 값이라 NDK 대상이 아니며, 로컬 설정이나 환경변수에서 읽습니다.
- JNI 값을 ViewModel에서 직접 부르지 않고 얇은 wrapper object를 거쳐야 단위 테스트에서 목킹할 수 있습니다.

## 버전 관리

모든 의존성 버전은 `gradle/libs.versions.toml` Version Catalog 한 곳에서 관리하고 하드코딩하지 않습니다.
SDK 버전과 flavor, 서명 이름 같은 전역 상수는 convention 모듈의 `ProjectConfiguration`에 모았습니다.

## 기술 스택

| 영역 | 기술 |
|------|------|
| Android | Kotlin 2.2.10, AGP 9.0.1, Jetpack Compose(BOM 2025.02.00, Material 3), Hilt 2.56.2, Room 2.7.0-alpha12, Firebase BoM |
| Backend | Node.js 20, firebase-functions v2, Firebase Admin SDK, Firestore |
| Frontend | React 19, TypeScript 5.9, Vite 7, React Router 7, Firebase JS SDK |

## 더 알아보기

- [README](README.md): 저장소 개요와 앱 현황
- [빌드와 도구](docs/build-and-tooling.md): Convention Plugin, Guard, 빌드 커맨드, 테스트 인프라, 문서 체계
- [백엔드와 웹](docs/backend-and-web.md): Cloud Functions와 관리자 웹
