# qr 그룹

QR 스캔·생성·히스토리를 담당하는 5개 모듈이다. `qr/` 와 `qr/common/` 폴더는 소스가 없으므로 project 의존으로 선언하지 않고, 쓸 하위 모듈 경로를 직접 적는다.

| 모듈 | 역할 | 핵심 기술 |
|---|---|---|
| `qr-common-api` | 타입 판별·액션·히스토리 저장소 계약 | 순수 Kotlin 모델, `Flow` |
| `qr-common-impl` | 히스토리 Room 구현 | Room, SQLCipher, Hilt, KSP |
| `scanner` | 단일 스캐너 다이얼로그 | CameraX, ZXing core |
| `multi-scanner` | 멀티 스캐너 + 바운딩 박스 | CameraX, ML Kit barcode |
| `generator` | QR 생성 화면 | ZXing core, Compose |

## qr-common-api / qr-common-impl

- api 는 Room·KSP·Hilt·Compose 를 끌어오지 않는다. `QRHistory` 는 `@Entity` 가 아닌 순수 모델이고 타임스탬프 기본값이 없다(시계를 읽으면 소비 앱이 기기 시각에 묶인다).
- `QRType.fromContent` 는 접두사 순서가 곧 우선순위다(URL, EMAIL, SMS, PHONE, WIFI, VCARD, 그 외 TEXT). 문자 갈래를 전화 매처보다 먼저 두지 않으면 `SMS` 가 나오지 않는다.
- impl 은 공용 `DatabaseFactory.createDatabase` 로만 Room 을 만든다. 스캔 기록에는 URL·와이파이 비밀번호·연락처가 들어갈 수 있어 SQLCipher 암호화와 Keystore 키 관리를 거친다. 엔티티는 밖으로 노출하지 않고 매퍼로 변환한다. 저장 시각은 주입된 `TimeProvider` 로 정한다. 같은 내용도 합치지 않고 행을 더한다.
- 의존: impl → api(api), `core:fundamental:common`, `core:common:database`.

## scanner

- 전체화면 다이얼로그에 프리뷰·오버레이(검정 50% + 가운데 가이드 박스)·닫기 버튼을 쌓는다. 후면 카메라 고정이다.
- 권한 상태(허용/거부/영구 거부)에 따라 프리뷰, 재시도 안내, 설정 이동 안내를 보여 준다. 권한 UseCase 는 화면 전용 ViewModel 이 직접 주입받는다(호출부에서 열면 MVI 단일 진입점이 뚫린다).
- 첫 인식 한 건을 콜백에 넘기면 호출부가 닫는다. 이 모듈을 의존하면 `CAMERA` 권한이 병합되어 스토어 페이지에 노출된다.

## multi-scanner

- 미리보기와 분석을 후면 카메라에 함께 묶고, 분석은 최신 프레임만 남기는 backpressure 전략으로 전용 단일 스레드에서 돌린다. QR 포맷만 켠다.
- 안정화: 빈 프레임이 3회 연속일 때까지 직전 결과를 유지해 오버레이 깜빡임을 막는다.
- 좌표 변환: 미리보기 `FIT_CENTER` 기준으로 회전, 작은 축 비율, 중앙 정렬 여백 순으로 적용해 바운딩 박스를 그린다. 사용자가 고른 한 건만 콜백에 넘긴다.
- `scanner` 와 한 화면에 동시에 띄우지 않는다.

## generator

- 화면이 입력·크기·색을 들고 생성 버튼만 ViewModel 로 넘긴다. 이전 생성 Job 을 취소하고 `Dispatchers.Default` 에서 인코딩한다.
- 오류 정정 수준 H, 여백 1, UTF-8, 로고 기본 크기는 변의 1/5(면적 4%)라 로고가 중앙을 가려도 인식이 남는다.
- 로고 선택·갤러리 저장·공유는 구현하지 않고 `onPickLogoClick`·`onSaveClick`·`onShareClick` 으로 호출자에 위임한다. 그룹 간 직접 참조는 빌드 가드가 막기 때문이다.
