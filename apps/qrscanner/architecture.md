# Q-Ring 아키텍처

앱 모듈(`:apps:qrscanner`)은 조립 전용이다. 소스는 `QRScannerApplication`, `MainActivity`, `di/` 3파일, `navigation/` 3파일뿐이고 규약 플러그인의 가드가 그 밖의 파일·디렉터리를 구성 단계에서 막는다. 원래 이 앱만 feature 골격이 없어 화면 코드가 전부 앱 모듈에 있었으나 2026-08 에 탭 화면 넷을 `apps/qrscanner/feature/*` 로 분리했다.

## 모듈 구성

```
:apps:qrscanner
 ├─ feature:splash    버전 확인 게이트
 ├─ feature:home      스캔 탭 (공용 다중 스캐너 호출 + 결과 표시 + 기록 저장)
 ├─ feature:history   히스토리 탭
 ├─ feature:more      더보기 탭
 └─ shared:*          qr:generator, qr:common(-impl), image-picker, image-saver, share,
                      legal, webview, oss-licenses, app-shell, crashlytics ...
```

| 모듈 | 책임 |
|---|---|
| `feature:splash` | 공용 `SplashScaffold` 와 `AppUpdateChecker` 로 버전 확인. 실패 정책은 진행 차단(BLOCK). 로직과 다이얼로그를 직접 만들지 않는다. |
| `feature:home` | 카메라 UI 를 만들지 않고 `MultiQRScannerScreen`(ML Kit)을 호출. 권한 UseCase 는 해석하지 않고 스캐너에 그대로 전달. 스캔 결과를 `QRHistoryRepository` 로 저장하며 저장 실패해도 화면에 보인 결과는 지우지 않는다. |
| `feature:history` | 기록 목록과 타입별 액션, 즐겨찾기 필터(화면에서 거르지 않고 저장소 질의를 교체), 삭제, 조회 실패 시 빈 목록 대신 실패 상태와 재시도를 그린다. |
| `feature:more` | 버전·정보 행(`SettingsScaffold` 위임)과 기록 전체 삭제. 삭제 실패는 사용자에게 알리고 진행 중에는 클릭을 막는다. 문의 메일은 `@SupportEmail` 로만 받는다. |

각 feature 는 `uiState` 로 상태를 노출하고 `convention.feature` 규약 플러그인(Hilt·Compose 포함)을 쓴다.

## 내비게이션과 화면 조립

| 라우트 | 화면 | 소유 | 바텀바 |
|---|---|---|---|
| `splash` | 스플래시 | `feature:splash` | 숨김 |
| `home` | 스캔 | `feature:home` | 표시 |
| `generator` | QR 생성 | `shared:feature:qr:generator` | 표시 |
| `history` | 히스토리 | `feature:history` | 표시 |
| `more` | 더보기 | `feature:more` | 표시 |
| `privacy_policy` · `terms_of_service` | 웹뷰 또는 내장 문서 | `webview` · `legal` | 숨김 |
| `open_source_license` | 별도 Activity | `oss-licenses` | 숨김 |

- 시작 목적지는 `splash` 이고 홈으로 갈 때 `popUpTo(inclusive)` 로 백스택에서 제거한다.
- `MainActivity` 는 edge-to-edge, Crashlytics 초기화, 콜드 스타트 딥링크 적재, `setContent` 순으로 돌고 `AppThemeHost` -> 딥링크 배선 -> `AppShell` -> `NavGraph` 로 조립한다.
- `QRScannerApplication` 은 `onCreate` 에서 로거에 크래시 리포터를 연결한다. 이 한 줄이 빠졌을 때 릴리스의 `Logger.e` 가 전부 버려졌던 적이 있어 지우면 안 되는 호출로 관리한다.
- 테마는 밝은 종이 단일 테마이며 시스템 다크 모드를 따르지 않는다.

## 생성 탭 배선

생성 화면 자체가 공유 모듈(`qr:generator`)이라 앱 feature 로 한 번 더 감싸지 않았다. 대신 화면이 필요로 하는 주변 동작을 콜백으로 앱이 직접 연결한다.

| 동작 | 사용하는 것 |
|---|---|
| 로고 선택 | `rememberImagePicker` (Photo Picker, 권한 불필요) |
| 갤러리 저장 | `SaveImageToGalleryUseCase` (MediaStore). IO 전환은 UseCase 가, 토스트는 메인에서 |
| 공유 | `ImageSharer.share` |

갤러리 저장은 처음에 클릭한 메인 스레드에서 PNG 인코딩을 해 화면이 멈췄기 때문에 `rememberCoroutineScope` + `Dispatchers.IO` 로 옮겼다(2026-10). 로고는 `remember` 상태로만 들고 있어 탭 전환이나 회전 시 사라지는 점이 알려진 제약이다.

## 법적 문서와 라이선스

처리방침·이용약관은 설정 URL 을 `LegalContentSource.fromUrl` 에 넘겨, URL 이 있으면 웹뷰, 없으면 내장 문서 화면을 그린다. URL 이 비어 있는 현재는 더보기 탭이 두 행 자체를 그리지 않는다(앱이 콜백을 null 로 내림). URL 값은 NDK 라이브러리에 빌드 때 들어가므로 채우면 다시 빌드해야 반영된다.

## 딥링크

`HOST_TO_ROUTE` 맵 하나가 단일 출처이며 화이트리스트와 라우터 매핑이 여기서 파생된다. 현재 열린 host 는 탭 넷이고 모두 PUBLIC 이다. 자사 앱 간 연동과 App Links 는 없어 신뢰 패키지·App Links 호스트는 비어 있다. 경로 없는 `yunseong-qring://<host>` 형태만 통과한다. 스킴 문자열은 코드, 매니페스트, 빌드 검증 세 곳에 있고 빌드 검증은 매니페스트와 검증 설정만 대조한다.

## 데이터와 QR 모듈 분리

- 기록 저장소는 계약 모듈(`qr-common-api`)과 구현(`qr-common-impl`, Room)으로 나뉜다. feature 는 계약만 의존하고 앱이 구현 모듈을 의존해 Hilt 그래프에 `@Binds` 를 넣는다. 2026-08 에 한 번 이 의존을 뺐다가 되살렸다.
- 기록의 타입은 저장된 이름을 `QRType.fromStoredName` 으로 되읽어 모르는 이름에서도 예외 없이 처리한다. 카드 라벨과 실제 액션은 같은 타입 판정 하나에서 나온다.
- 스캐너는 앱마다 다르다. Q-Ring 은 ML Kit 다중 스캔, Lottomate 는 ZXing 단일 스캔이다. 생성은 ZXing 이다.

## 빌드 구성

| 항목 | 값 |
|---|---|
| 규약 플러그인 | android.application · hilt · compose · buildconfig · ndk.nativeconfig · verify.prebuild (선언 순서 고정) |
| 버전 | versionCode 1, versionName 1.0.0 |
| 서명 | 플레이버 기준 분기 (devDebug 도 dev 키) |
| 설정 | 전역 -> 앱 common -> 플레이버 순으로 병합해 NativeConfig 라이브러리에 포함 |
| 배포 | Firebase App Distribution 으로 테스터 그룹에 APK 업로드 |
| 리소스 | 앱 이름은 Q-Ring 하나 (`generateLocaleConfig = false`) |

컴파일에 필요한 의존이 두 가지 있다. `fundamental:common` 은 `@SupportEmail` 한정자와 `CrashReporter` 상위 타입 해석에 필요해 지우면 컴파일이 깨진다(2026-08 제거를 시도했다가 되돌림). 반면 `fragment-ktx` 는 app-shell 이 이미 노출하므로 빼도 깨지지 않는다.

## 주요 설계 결정

| 날짜 | 결정 | 이유 |
|---|---|---|
| 2026-10 | 갤러리 저장을 IO 로 이동 | 메인 스레드 인코딩으로 화면 정지 |
| 2026-09 | Application 에서 로거에 크래시 리포터 연결 | 연결이 없어 릴리스 오류가 전부 사라지고 있었음 |
| 2026-08 | 스플래시 흐름을 공용 base 로 승격 | 앱마다 버전 확인 흐름을 복제해 분기가 갈림 |
| 2026-08 | `qr-common-impl` 을 앱 의존으로 복원 | 히스토리가 계약 모듈만 의존하게 되며 저장소 바인딩이 전이되지 않음 |
| 2026-08 | 자리 표시 화면을 실제 화면으로 교체 | 약관·처리방침·라이선스 라우트가 빈 화면이었음 |
| 2026-08 | 탭 화면을 feature 모듈로 분리 | 화면 코드가 전부 앱 모듈에 있던 구조 정리 |

## 알려진 유의사항

- 로고 선택 창을 취소하면 이미 넣어 둔 로고가 지워진다(선택기가 취소 시 null 을 넘기고 그대로 대입).
- 공유 콜백은 아직 메인 스레드에서 캐시 파일을 쓴다. 쓰기 실패 시 화면에 드러나지 않는다.
- Activity 가 재생성되면 시작 딥링크가 한 번 더 처리된다.
- 창 테마가 플랫폼 다크 테마라 Compose 첫 프레임 전 배경은 어둡다.
