# shared/feature

여러 앱이 함께 쓰는 기능 모듈 묶음이다. 비즈니스 로직과 화면(Compose UI)을 함께 제공하며, 인프라는 `shared/core` 에 맡긴다. 모듈별 상세는 [feature/](feature/) 아래 문서를 본다.

## 모듈 표

| 그룹 | 모듈 | 목적 | 상세 |
|---|---|---|---|
| app-update | `app-update` | Remote Config 로 강제 업데이트 판정, 외관 교체형 안내 다이얼로그 | [app-update](feature/app-update.md) |
| | `splash-base` | 스플래시 상태홀더·화면 공통 뼈대 | |
| | `security-root` | 루팅 흔적 검사(값싼 1차 필터) | |
| qr | `qr-common-api` / `qr-common-impl` | QR 타입 판별·액션·히스토리 계약 / Room+SQLCipher 구현 | [qr](feature/qr.md) |
| | `scanner` | ZXing 단일 스캐너 다이얼로그 | |
| | `multi-scanner` | ML Kit 멀티 스캐너 + 바운딩 박스 | |
| | `generator` | ZXing QR 생성 화면 | |
| social-login | `social-login-core` / `-base` | 계약·모델·UseCase / 로그인 화면 뼈대 | [social-login](feature/social-login.md) |
| | `-google` / `-kakao` / `-naver` | 제공자별 구현 | |
| lock | `lock-api` / `lock-impl` / `lock-ui` | 앱 잠금(패턴·PIN·생체) 계약 / 저장·암호화 / 화면 | [lock](feature/lock.md) |
| notification | `notification-inbox-api` / `-impl` / `-ui` | 알림함 계약 / 저장·수신 / 화면 | [notification](feature/notification.md) |
| | `notification-settings` | FCM 토픽 토글 UseCase | |
| image | `image-picker` | 갤러리 선택 후 축소 디코딩 | [image](feature/image.md) |
| | `image-saver` | MediaStore 로 PNG 저장 | |
| | `share` | FileProvider 이미지 공유 인텐트 | |
| UI 컴포넌트 | `chart` | 월별 꺾은선·막대 차트 | [ui-components](feature/ui-components.md) |
| | `coachmark` | 스크림·구멍·말풍선 안내 오버레이 | |
| | `onboarding` | 스와이프 온보딩 화면 | |
| | `legal` | 약관 화면과 표시 방식 포트 | |
| | `oss-licenses` | 오픈소스 라이선스 화면 | |
| | `settings-ui` | 설정·더보기 화면 골격 | |
| | `webview` | 범용 WebView 화면 | |

`qr` 디렉터리 자체는 소스가 없는 묶음 폴더이고, 하위 5개 모듈만 Gradle 에 include 된다.

## 공통 패턴

- **api / impl / ui 분리**: 저장·수신 등 무거운 의존이 있는 기능은 계약(`-api`), 구현(`-impl`), 화면(`-ui`)으로 나눈다. api 는 Room·KSP·Hilt·SDK 를 끌어오지 않아 배지나 적재처럼 일부만 필요한 소비자가 가볍게 쓴다. 화면은 계약만 의존하고 구현 바인딩은 앱이 함께 넣어 붙인다. (lock, notification-inbox, qr-common, social-login)
- **구현 라이브러리는 `implementation`**: `api` 로 선언하면 소비자에게 전파되므로, 계약 모듈이 노출해야 하는 타입(`Flow`, `FragmentActivity` 등)만 `api` 로 둔다.
- **포트와 슬롯**: 문구·이미지·저장 여부는 모듈이 갖지 않고 호출부가 주입한다(`legal`, `onboarding`, `coachmark`). 열람 여부 같은 상태는 포트 인터페이스만 두고 구현은 앱이 한다.
- **교차 참조 가드**: 서로 다른 그룹의 shared-feature 를 직접 참조하면 빌드가 막는다. 조합(예: 약관 URL이면 webview, 아니면 내장 화면)은 앱의 NavGraph 가 한다.
- **규약 플러그인**: 화면 모듈은 `convention.feature` 가 `presentation-base`·`ui-base`·`designsystem` 을 붙여 준다. 계약·구현 모듈은 `convention.android.library`(+ `convention.android.hilt`)를 쓴다. 버전은 Version Catalog 로만 참조한다.
- **MVI 상태홀더**: 화면은 `state` 와 `onIntent` 단일 진입점을 쓰고, 처리 함수는 `private` 으로 둔다. 재시도 전에 사라지면 안 되는 문구는 1회성 효과가 아니라 상태에 둔다.
- **취소 전파**: 넓은 `catch` 의 첫 줄에서 `rethrowIfCancellation()` 을 호출해 코루틴 취소를 실패로 바꾸지 않는다.
- **실패는 값으로**: 소셜 로그인의 `AuthResult`, 저장 계열의 `false`/`null` 처럼 예외 대신 값으로 돌려줘 호출부 분기를 단순하게 한다.
- **키는 앱이 제공**: SDK 키나 클라이언트 ID 는 모듈에 넣지 않고 한정자 바인딩으로 앱이 제공한다.
- **모듈 문서**: 각 모듈은 얇은 `AGENTS.md` 와 `docs/`(API, DESIGN, CAVEATS, DECISIONS 등)를 갖는다.
