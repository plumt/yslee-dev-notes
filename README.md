# yunseongApps

Android 앱 6개와 Firebase 백엔드, React 웹을 한 저장소에서 관리하는 모노레포의 공개 문서 저장소입니다.
소스 저장소는 비공개이며, 이 저장소는 구조·설계 결정·모듈 책임을 설명하는 문서만 담습니다.

## 한눈에 보기

- 앱 6개: Lottomate, Lunar, QRScanner, Diding, Moduta, Fxrate
- Gradle 모듈 135개 (shared core·feature 모듈과 앱 전용 core·feature 모듈 포함)
- Clean Architecture + Modular Architecture, 의존 방향은 `apps → shared/feature → shared/core` 단방향
- 계층 규칙·화면 모듈 규격·앱 모듈 모양을 문서가 아니라 빌드(Convention Plugin의 Guard)가 강제
- 기술 스택: Kotlin 2.2.10, AGP 9.0.1, Compose BOM 2025.02.00, Hilt 2.56.2, Room 2.7.0-alpha12
- SDK: minSdk 29, targetSdk 36, compileSdk 36, Java 17

## 앱 현황

| 앱 | 한 줄 소개 | 패키지 | Flavor | 문서 |
|----|-----------|--------|--------|------|
| Lottomate | 로또 번호 관리, 당첨 확인, 통계, 당첨 푸시 알림 | `com.yunseong.lottomate` | dev / prod | [apps/lottomate](apps/lottomate/README.md) |
| Lunar | 달 위상 대시보드와 달력, 달 이벤트 알림 | `com.yun.lunarphase` | dev / prod | [apps/lunar](apps/lunar/README.md) |
| QRScanner (Q-Ring) | QR 다중 스캔, 생성, 이력 | `com.yunseong.qring` | dev / prod | [apps/qrscanner](apps/qrscanner/README.md) |
| Diding | 알람 기반 TODO·일정 관리 | `com.yunseong.diding` | dev / prod | [apps/diding](apps/diding/README.md) |
| Moduta | 서울시 버스 실시간 정보와 지도 | `com.yunseong.moduta` | dev / prod | [apps/moduta](apps/moduta/README.md) |
| Fxrate | 환율 조회와 임계값 알림, 홈 위젯 | `com.yunseong.fxrate` | dev / prod | [apps/fxrate](apps/fxrate/README.md) |

6개 앱 모두 dev/prod 환경 flavor를 갖습니다. 앱 모듈에는 조립 코드(Application, MainActivity, 내비게이션, DI 바인딩)만 두고,
기능은 `apps/<앱>/core`, `apps/<앱>/feature` 하위 모듈에 둡니다.

## 모듈 구성

```
yunseongApps/
├── apps/                  앱 6개 (각 앱은 조립 모듈 + core/feature 하위 모듈)
├── shared/
│   ├── core/              공통 인프라 (fundamental, common, firebase, permissions, lunar)
│   └── feature/           여러 앱이 공유하는 기능 (qr, lock, social-login, notification-* 등)
├── build-logic/           Convention Plugin, Guard, NDK 설정 생성기 (별도 Gradle 빌드)
├── clients/               앱별 클라이언트 설정 (빌드 시 NDK로 암호화되는 값의 출처)
├── backend/               Firebase Cloud Functions (TypeScript)
├── functions/             Cloud Functions (JavaScript, 로또·달 이벤트 푸시)
├── frontend/              React 웹 (방침·약관, 관리자 콘솔)
└── scripts/               스캐폴딩, 문서 검사, 테스트 집계, Appium E2E, 배포 스크립트
```

| 계층 | 모듈 수 | 역할 |
|------|--------|------|
| Shared Core - Fundamental | 5 | 필수 인프라 (common, permission 계약, local-notification, alarm, presentation-base) |
| Shared Core - Permissions | 11 | 권한별 Provider 구현, 의존만 추가하면 활성화 |
| Shared Core - Common | 29 | network, database, ui, designsystem, admob, deeplink, navigation-base, app-shell 등 |
| Shared Core - Firebase | 6 | core, auth, crashlytics, firestore, messaging, remote-config |
| Shared Core - Lunar | 1 | 달 위상 계산 |
| Shared Feature | 30 | 2개 이상 앱이 쓰는 기능 (api/impl/ui 분리 포함) |
| 앱 전용 | 앱별 5~15 | Lottomate 15, Lunar 8, Moduta 8, Diding 8, Fxrate 8, QRScanner 5 |

shared 모듈의 상세는 [shared/core](shared/core.md), [shared/feature](shared/feature.md)를 참고하세요.

## 핵심 설계 요약

- 의존 방향 규칙을 빌드가 막음: 역참조, 앱 간 참조, 계층 역전, feature 간 교차 참조는 configure 단계에서 실패
- 공용 모듈이 앱 전용 구현을 필요로 하면 포트 인터페이스를 두고 각 앱이 구현·바인딩
- Hilt 멀티바인딩(`@Multibinds` + `@IntoSet`)으로 권한·소셜 로그인·화면 목적지·FCM 핸들러를 의존 추가만으로 등록
- 화면 모듈은 `convention.feature` 하나로 구성하고 `StateViewModel`/`MviViewModel` 골격 사용
- API 키 등 런타임 값은 BuildConfig 대신 NDK(XOR + JNI)로 숨김
- 빌드 전 검증으로 "빌드는 성공하지만 출시 후에야 드러나는" 오류(빈 키, 테스트 광고 ID, 딥링크 스킴 불일치, 서명 누락)를 차단

자세한 내용은 [ARCHITECTURE.md](ARCHITECTURE.md)에 있습니다.

## 문서 목록

| 문서 | 내용 |
|------|------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | 계층 구조, 의존성 규칙, 주요 아키텍처 패턴, 보안 구조 |
| [docs/build-and-tooling.md](docs/build-and-tooling.md) | Convention Plugin, Guard, 빌드 커맨드, 테스트 인프라, 문서 계약과 동기화 방식 |
| [docs/backend-and-web.md](docs/backend-and-web.md) | Cloud Functions, 웹 관리자 콘솔 개요 |
| [shared/core](shared/core.md) | Shared Core 모듈 문서 |
| [shared/feature](shared/feature.md) | Shared Feature 모듈 문서 |
| `apps/<앱>/README.md` | 앱별 구조와 설계 (위 앱 현황 표의 링크) |

## 스크린샷

스크린샷은 `screenshots/` 폴더에 앱별로 정리합니다.

## 백엔드와 웹

- Cloud Functions (Node.js 20, firebase-functions v2, 리전 `asia-northeast3`): 로또 당첨 확인·구매 알림, 달 이벤트 알림, 관리자 API
- 웹 (React 19, Vite 7, TypeScript): 방침·약관 페이지와 관리자 콘솔(통계, 수동 알림, Remote Config, Secret)

상세는 [docs/backend-and-web.md](docs/backend-and-web.md)를 참고하세요.

## 이 저장소에 대하여

소스 저장소의 `AGENTS.md`와 `docs/`를 기준으로 구조 문서를 추려 공개용으로 정리한 것입니다.
키, 서명 정보, 내부 주소 같은 민감 값은 싣지 않습니다.
