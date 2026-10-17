# 빌드와 도구

모노레포의 빌드 설정, 빌드 시점 검증, 개발 스크립트, 테스트 인프라, 문서 체계를 정리합니다.
구조 규칙은 [ARCHITECTURE.md](../ARCHITECTURE.md)에 있습니다.

## build-logic

`build-logic`은 루트 설정의 `pluginManagement`가 `includeBuild`로 들이는 별도 Gradle 빌드입니다.
메인 빌드의 서브프로젝트가 아니어서 플러그인 ID로만 참조하고, 내부는 둘로 나뉩니다.

| 위치 | 담는 것 |
|------|--------|
| `build-logic/convention` | Convention Plugin 10종과 Guard 8종 (Kotlin DSL 플러그인 모듈) |
| `build-logic/scripts` | NativeConfig 생성기(Python)와 그 단위 테스트 |

버전 카탈로그는 메인 빌드와 같은 `gradle/libs.versions.toml`을 읽습니다.

## Convention Plugin

모듈의 `build.gradle.kts`는 `plugins { id("convention.xxx") }` 한 줄로 설정을 받습니다.

| 플러그인 | 하는 일 |
|---------|--------|
| `convention.android.application` | SDK·Java 기본값, 공통 의존, dev/prod flavor, flavor 기준 서명 분기, buildTypes, Guard, Detekt |
| `convention.android.library` | SDK·Java 기본값, `consumer-rules.pro` 보장, 공통 의존, Guard, Detekt |
| `convention.android.hilt` | Hilt, KSP, hilt-android, hilt-compiler |
| `convention.android.compose` | Compose 컴파일러, BOM, 기본 Compose 의존 |
| `convention.buildconfig` | BuildConfig 생성 활성화 |
| `convention.detekt` | 정적 분석(report-only), 모듈별 baseline, 공유 설정. library/application이 자동 적용 |
| `convention.feature` | library + hilt + compose, presentation-base, ui-base, designsystem, 테스트 의존 일괄. core 경로에서는 실패 |
| `convention.ndk.nativeconfig` | config.json에서 XOR C 소스, CMakeLists, JNI 래퍼, ProGuard 규칙 생성 |
| `convention.verify.prebuild` | variant마다 패키징 전 설정·서명·딥링크 검증 (앱 전용) |
| `convention.play.publish` | Gradle Play Publisher 배선 |

설계 포인트는 다음과 같습니다.

- 공통 상수(`ProjectConfiguration`)에 SDK 버전, flavor 이름, 서명 설정 이름을 모아 SDK 변경 시 한 곳만 고칩니다.
- 앱 공통 빌드 설정(flavor, 서명, buildTypes)은 convention이 먼저 만들어 두고 앱은 재오픈해 고유 설정만 덧붙입니다.
  6개 앱이 같은 buildTypes 블록을 각자 갖고 있던 것을 승격한 결과입니다.
- flavor를 쓰는 앱에는 `testDebugUnitTest` 집합 태스크를 자동 등록합니다. AGP가 만들지 않는 이 태스크가 없으면
  루트에서 테스트를 돌릴 때 flavor 앱이 조용히 건너뛰어지기 때문입니다.
- NDK flavor 인자는 `afterEvaluate`가 아니라 `configureEach`로 넘깁니다.
- lint는 NewApi를 오류로 고정해 minSdk 하위 호환을 게이트로 삼고, 기존 지적은 앱별 baseline으로 그랜드파더링합니다.
- 클라이언트 설정은 `loadClientConfig`, 빌드 전용 시크릿은 `getSecret`으로 읽는 확장 함수로 제공합니다.

### Guard 8종

Guard는 플러그인이 적용될 때 configure 단계에서 위반을 찾아 빌드를 세웁니다.

| Guard | 막는 것 |
|-------|--------|
| `DependencyLayerGuard` | 의존성 계층 6규칙 위반 (역참조, 앱 간 교차, 계층 역전, feature 교차) |
| `ModuleConventionGuard` | 화면 모듈의 손조합, 플러그인 선언 순서 불일치 |
| `AppModuleShapeGuard` | 앱 모듈에 조립 이외의 파일·중첩 디렉터리·Android 컴포넌트 |
| `FirebaseConfigGuard` | Firebase 모듈에 닿는데 `google-services` 설정이 없음 |
| `PermissionProviderGuard` | 쓰는 권한 타입의 Provider 모듈이 의존 그래프에 없음 |
| `ConsumerRulesGuard` | 직렬화 DTO가 있는데 `consumer-rules.pro`에 실질 규칙이 없음 |
| `MigrationSafetyGuard` | 마이그레이션도 허용 플래그도 없는 `fallbackToDestructiveMigration()` |
| `AdIdVariantGuard` | AdMob 테스트 퍼블리셔 ID가 debug 블록 밖에 고정됨 |

각 Guard는 긴급 우회용 `-Pskip...=true` 플래그를 갖습니다. 값이 정확히 `true`일 때만 꺼지고 경고 로그를 남기며,
커밋 전에는 반드시 원복합니다. 플래그 존재만으로 꺼지면 `gradle.properties` 한 줄로 전 빌드의 검증이 조용히 해제되기 때문입니다.

Firebase·권한 Guard는 다른 모듈의 Gradle 모델 대신 build 스크립트 텍스트를 읽어 판정합니다.
그 모듈이 아직 평가되지 않았어도 판정할 수 있게 한 선택입니다.

## 빌드 전 검증과 NativeConfig

`convention.verify.prebuild`의 검사 항목과 심각도 분리 이유, NativeConfig의 동작 흐름은
[ARCHITECTURE.md](../ARCHITECTURE.md)의 "빌드 시점 검증"과 "보안 아키텍처" 절에 있습니다.
NativeConfig 생성기는 설정 병합 우선순위(공통 → 앱 → 환경)를 convention과 같은 순서로 구현하고,
생성 산출물(C 소스, CMakeLists, Kotlin 래퍼, keep 규칙)은 손대지 않고 config 또는 생성기를 고치는 방식으로 운영합니다.

## 정적 분석

Detekt는 전 모듈에 적용돼 있고 현재 report-only(위반이 있어도 exit 0) + 모듈별 baseline 그랜드파더링 단계입니다.
게이트로 쓰기에는 아직 이르다는 점을 문서에 명시했고, 규칙을 끌 때는 근거를 주석으로 남깁니다.
AGP Lint와는 별개 도구로, Lint는 Android API·리소스 오용을, Detekt는 Kotlin 스타일·복잡도를 봅니다.

## 개발 스크립트 (`scripts/`)

빌드·설치·테스트·의존 조회는 `scripts/dev.py` 한 진입점으로 합니다. 태스크 이름과 모듈 목록을 손으로 조립하지 않고
`settings.gradle.kts`와 build 파일에서 읽습니다.

| 명령 | 하는 일 |
|------|--------|
| `dev.py build/install <앱>` | flavor·variant에 맞는 태스크로 빌드·설치, 끝나면 build 디렉터리 정리 |
| `dev.py test <앱>` | 앱 하나의 전 모듈 유닛 테스트 |
| `dev.py test-changed` | 바뀐 모듈과 역의존 모듈만 테스트 |
| `dev.py rdeps <모듈>` | 역의존 조회 |
| `dev.py unused` | 어느 앱도 쓰지 않는 shared 모듈 |
| `dev.py lint <앱>` | minSdk 하위 호환 게이트 lint, 실패 시 오류 요약 |
| `dev.py wire <앱> <모듈>` | 모듈을 앱 build 파일·문서·딥링크·탭에 배선 |
| `dev.py cert <앱>` | APK·설치본 서명 인증서 지문 확인 (빌드하지 않음) |

### 스캐폴딩

- `create_module.py`: 계층별(shared-core, shared-feature, 우산, app-core, app-feature) 모듈 골격을 만듭니다.
  코드, 단위 테스트, `consumer-rules.pro`, `AGENTS.md`, 필수 `docs/`, `settings.gradle.kts` 등록, 모듈 인덱스 행까지 한 번에 생성합니다.
- `create_app.py`: dev/prod flavor 배선, 앱 셸, 문서 골격, 테스트·Appium 하네스 디렉터리를 포함한 새 앱을 만듭니다.
- `delete_module.py`, `delete_app.py`: `--dry-run`으로 대상을 확인한 뒤 되돌립니다.
- `--verify`는 새 모듈이 문서 검사기 두 종과 Gradle 단위 테스트(앱은 빌드 포함)를 통과하는지 확인합니다.

스캐폴더와 셸 스크립트는 앱 이름이나 namespace prefix를 상수로 적지 않고 설정에서 조회합니다.
루트 스크립트는 표준 라이브러리만 사용하고, 서드파티 의존은 Appium 러너의 requirements에만 둡니다.

### 배포

- Firebase 배포 스크립트: 전체, Functions만, Hosting만
- App Distribution 배포 스크립트: 앱·환경·빌드 타입을 지정하는 대화형 또는 인자 방식
- Jenkins 원격 호출 스크립트: 로컬에서 빌드하지 않고 CI 서버에 위임
- GitHub Actions: Android 빌드는 자동/수동, Firebase 배포는 수동 실행만 허용

## 테스트 인프라

### 유닛 테스트

- 모든 앱의 테스트는 `:apps:<앱>:testDebugUnitTest` 집합 태스크 경로를 탑니다.
- 앱 단위 러너(`apps/<앱>/script/run-unit-tests.sh`)는 `src/test`가 있는 하위 모듈을 찾아 하드코딩 없이 실행합니다.
- `collect-test-results.sh`는 전체 테스트를 돌려 HTML 리포트, Jenkins 이메일 본문용 표, JUnit XML, properties 파일로 집계합니다.
- 변경분 테스트는 바뀐 파일을 모듈로 환산하고 역의존 전이 폐포까지 대상으로 삼습니다. 빌드 설정이 바뀌면 전체 테스트를 권합니다.
- 커버리지는 Kover 기반 스크립트가 라인 80%, 분기 70% 기준으로 판정합니다 (현재 적용 모듈은 제한적).
- 테스트 대상 모듈을 성격별(계산·보안, SDK 위임 브릿지, 상태전이·매핑, 딥링크 핸들러, 권한 Provider DI)로 나누고,
  순수 UI·DI 계약 모듈은 테스트를 생략하는 분류 기준을 문서로 고정했습니다.

### Appium E2E 하네스

화면 흐름을 testTag 기반으로 검증하는 자체 구현 러너(`scripts/appium_runner`)입니다.
외부 저장소 의존 없이 Android 전용, Hilt, Compose native 화면이라는 전제로 범위를 줄였습니다.

- **testTag 컨벤션**: 별도 TestTags 파일을 만들지 않고, 태그 상수를 화면 파일 맨 아래에 `internal const val TAG_X = "snake_case"`로
  둡니다. 동적 목록 항목은 `"${TAG_X}_${id}"`로 조합하고, 루트에 강제 태그를 달지 않습니다.
- **`testTagsAsResourceId`**: Appium(UiAutomator2)이 Compose testTag를 resource-id로 읽도록 앱 최상위 Composable에 한 번만 설정합니다.
- **`test-tags.yaml`**: 소스의 태그 상수를 정적 스캔해 앱별로 자동 생성합니다. 직접 수정하지 않습니다.
- **시나리오 YAML**: 앱별 `appiumTest/scenarios`에 두고, 30여 개 액션 중 native Compose에 필요한 서브셋을 지원합니다.
  요소 탐색은 resourceId 우선 경로와 xpath 경로로 나뉩니다.
- **기기 선택**: 기본은 로컬 에뮬레이터이고 실기기는 파라미터로 opt-in합니다.
- **자동 배선**: `create_app.py`가 새 앱에 실행 스크립트와 스모크 시나리오를 함께 만듭니다.
  실행 스크립트는 본문을 이중 관리하지 않고 기존 앱의 것을 복제해 이름만 치환합니다.
- 일괄 실행 스크립트는 기본이 안내이고 `--run`을 붙여야 실제로 실행합니다.

## 문서 체계

### 모듈 문서 계약

모든 모듈은 `AGENTS.md`(얇은 인덱스)와 `docs/`(상세)를 가집니다. 모듈은 저장소 루트를 뺀 `AGENTS.md` 보유 디렉터리 전부이며
Gradle 모듈, 우산 디렉터리, 비 Gradle(backend, functions, frontend, scripts)을 모두 포함합니다.

- `AGENTS.md`는 네 절만 허용합니다: 모듈 정보, 반드시 지킬 것(최대 8개), 의존성, 문서 인덱스. 분량 상한을 두어 매번 읽어도 부담이 없게 합니다.
- 상세 문서는 범주마다 개별 파일입니다: 구조, 유의사항, 히스토리, 에러 코드, 비즈니스 로직, 트러블슈팅, 공개 계약 등.
- 구조·실행·히스토리 문서는 코드와 이력에서 뽑아 생성하고 손으로 고치지 않습니다. 판단이 필요한 문서만 사람이 씁니다.
- 같은 문장을 여러 문서에 복제하지 않습니다. 예를 들어 유의사항 문서는 규칙 문장을 옮기지 않고 이유와 위반 시 증상을 적습니다.
- 문서마다 프론트매터에 작성자 서명을 적고, 재검증 없이 검증일을 갱신하지 않습니다.
- 저장소에 `CLAUDE.md`를 두지 않습니다. 있으면 그 경로의 `AGENTS.md` 자동 로드가 꺼지기 때문입니다.

판정은 두 검사 스크립트(문서 계약 검사, 컨텍스트 드리프트 검사)가 축별 건수로 합니다.
규약을 바꾸면 스캐폴더의 문서 템플릿도 같은 변경에서 고쳐, 새로 만든 모듈이 처음부터 검사를 통과하게 합니다.

### 공개 문서 동기화

비공개 모노레포와 이 공개 문서 저장소는 Git Submodule로 연결됩니다.
공개 대상은 개요와 구조 문서뿐이고, 시크릿 관리 문서, 키·토큰, 서비스 계정, 상세 구현 코드는 공개하지 않습니다.
공개 저장소 문서를 갱신한 뒤 메인 저장소가 submodule 참조를 갱신하는 순서로 반영합니다.

## 코딩 규약 요약

- 버전은 Version Catalog만 사용하고 하드코딩하지 않습니다.
- 모든 `@Composable` UI 함수에 `@Preview`를 붙이되, Hilt 화면 진입점은 본문을 `Content`로 분리해 그쪽에 붙입니다.
- 사용자에게 보이는 문구는 `strings.xml`과 `stringResource()`를 씁니다.
- 시각은 `TimeProvider`를 주입합니다.
- 공용 초기화(Crashlytics, AdMob 등)는 모듈이 `AppStartupTask`로 기여하고 앱 Application은 `StartupApplication`을 상속합니다.
- 커밋 메시지의 타입과 범위는 영문(feat, fix, refactor, docs, build, test, perf, chore), 제목과 본문은 한글입니다.
