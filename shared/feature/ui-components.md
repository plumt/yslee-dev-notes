# UI 컴포넌트 모듈

문구·데이터를 호출부가 주입하는 재사용 Compose 모듈들이다. 모두 `convention.feature` 를 쓰고 직접 선언한 project 의존이 거의 없다.

| 모듈 | 목적 | 핵심 구현 | 채택 |
|---|---|---|---|
| chart | 월별 꺾은선·막대 차트 | 꺾은선은 Vico `CartesianChartHost`(축·포맷터를 라이브러리가 제공), 막대는 순수 Compose `Box`/`Row`(최댓값 뱃지·강조가 Vico 축 API 밖). 앱 도메인 타입 없이 컬렉션과 포맷터만 받고 색은 `MaterialTheme` 로 주입 | lottomate |
| coachmark | UI 요소를 스크림·구멍·말풍선으로 단계 안내 | `coachMarkTarget` 이 `boundsInRoot()` 를 수집하고 오버레이는 대상과 같은 최상위 `Box` 의 형제로 둔다(Dialog 는 좌표가 어긋남). 회전 시 단계 인덱스만 저장하고 좌표는 재수집. 문구는 `CoachMarkStep`, 완료 여부는 `CoachMarkCompletedStore` 포트 | 없음 |
| onboarding | 스와이프 온보딩 | `HorizontalPager`, `PaperStepIndicator`, 건너뛰기 기본 콜백은 `onFinish`. 페이지는 `OnboardingPage` 로 주입, 열람 여부는 `OnboardingSeenStore` 포트이며 구현은 앱 몫. 빈 `pages` 는 `require` 로 거부 | 없음 |
| legal | 약관 화면과 표시 방식 포트 | `LegalContentSource.Url`(웹뷰, 배포 없이 수정) 또는 `Embedded`(오프라인 가능)를 값으로 표현. 웹뷰는 다른 그룹이라 직접 부를 수 없어 앱 NavGraph 가 분기. `LegalDocumentScreen` 은 상태홀더 없이 조항을 카드로 펼침 | qrscanner |
| oss-licenses | 오픈소스 라이선스 화면과 설정 행 | AboutLibraries Gradle 플러그인이 의존 그래프에서 `aboutlibraries.json` 을 생성하고 닫힌(exported=false) Activity 가 `LibrariesContainer` 로 렌더. 진입은 `OssLicensesLauncher` | 6개 앱 |
| settings-ui | 설정·더보기 화면 골격 | `SettingsScaffold` 가 배경·여백(20dp)·섹션 순서(앱 섹션, 버전, 정보, 꼬리말)를 고정하고 앱은 제목과 고유 섹션만 정함. 값이 없으면 행을 그리지 않는 것이 계약. 라이선스 행은 슬롯 | diding, fxrate, lottomate, lunar, moduta, qrscanner |
| webview | 제목과 URL 을 받는 범용 WebView 컴포저블 | `AndroidView` factory 에서 JS·DOM Storage 켜고 캐시 `LOAD_NO_CACHE`, 줌 끔. `WebViewSchemeValidator`(순수 문자열 판별, JVM 테스트 가능), `ExternalAppRegistry`(카드사·간편결제 스킴), `WebViewSchemeClient`(인텐트 실행·마켓 폴백), `WebViewPopupChromeClient`(새 창 URL 을 외부로). `intent://` 는 selector 를 지우고 `NEW_TASK` 만 남겨 실행. `INTERNET` 권한 선언 | lottomate, qrscanner |

## 공통 설계 결정

- 문구·이미지·약관·페이지 같은 내용은 모듈에 두지 않고 호출부가 주입한다.
- 새 파라미터는 기존 채택 앱의 동작이 바뀌지 않도록 기본값을 보수적으로(webview 는 `false`) 둔다.
- chart, oss-licenses 등 모듈 간 직접 의존은 두지 않고 슬롯과 콜백으로 연결한다.
