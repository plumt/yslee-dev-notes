# Lottomate 모듈 구성

앱 전용 모듈의 책임과 경계를 정리한다. 모든 모듈은 Convention Plugin 과 Version Catalog 로만 빌드를 구성한다.

## core 모듈 (데이터·도메인, UI 없음)

| 모듈 | 책임 | 경계·설계 포인트 |
|---|---|---|
| `core:common` | 회차 모델, 당첨 판정(`WinningRank`), 번호 계산, 번호 UI, 테마, 설정 상수, 토픽 구독 | 외부 URL·문의 메일·FCM 토픽은 평문 상수 대신 네이티브 설정 getter 로 읽는다. feature 를 의존하지 않는다. |
| `core:database` | SQLCipher 암호화 Room DB `LottoDatabase`(version 4) | 테이블 3개(`lotto`, `my_numbers`, `favorite_numbers`), 외래 키 없이 회차 값으로만 연결. 파괴적 마이그레이션 금지, 스키마 JSON 을 커밋. |
| `core:winning-numbers` | 당첨번호 조회·동기화 | Firestore(최신/회차) + 외부 API(전 회차) -> 로컬 DB. 최신 회차는 프로세스당 하나의 메모리 캐시(Mutex 보호). API base URL 은 네이티브 설정으로만 받는다. |
| `core:my-numbers` | 저장 번호·즐겨찾기 도메인/저장소/집계 | 번호는 저장 전에 오름차순 정렬. 여러 건 변경은 트랜잭션 UseCase 로 감싼다. |
| `core:statistics` | 당첨번호와 내 구매 게임 집계 | 저장소 계약은 domain, 구현은 data 에 두고 UseCase 는 구현을 import 하지 않는다. 현재 시각은 `TimeProvider`. |
| `core:winning-checker` | 저장 번호와 당첨번호 대조, 당첨/구매 로컬 알림 발송 | 대조 트리거를 갖지 않는다. 등수는 1~5 숫자, 낙첨은 null. 알림 문구는 등수만 보고 고른다(1등은 금액을 몰라도 낙첨 문구가 나가면 안 되기 때문). |
| `core:countdown` | 추첨까지 남은 시간 판정 | 시각을 인자로만 받는 순수 계산. 토요일 20:35 추첨 시작, 20:55 결과 발표 기준 3상태. |

## feature 모듈 (화면)

| 모듈 | 책임 |
|---|---|
| `feature:splash` | 공용 `SplashScaffold` 로 강제 업데이트 게이트를 지난 뒤 당첨번호를 동기화하고 홈으로 이동. 확인 실패 시 진행을 막는(BLOCK) 정책. |
| `feature:home` | 최신 당첨 결과, 카운트다운, QR 저장, 회차 조회, 번호 이력 진입. QR 카메라 UI 는 공유 스캐너 화면을 재사용한다. |
| `feature:my-numbers-overview` | 내 번호 탭. 이번주 구매 / 달력 / 즐겨찾기 3세그먼트와 네이티브 광고. |
| `feature:my-numbers` | 번호 입력 시트와 수동 입력 화면. 저장·조회는 `core:my-numbers` UseCase 로만. |
| `feature:statistics` | 통계 탭. 집계는 `core:statistics` 를 쓰고 숫자 서식은 `Locale.KOREA` 를 명시. |
| `feature:more` | 더보기 탭. 버전 확인, 알림 토글(권한 확보는 ViewModel), 저장 번호 삭제, 약관 진입. |
| `feature:messaging` | FCM 메시지 핸들러(`lotto_check`, `lotto_purchase_reminder`). 수신 서비스는 공용이고 핸들러만 `@IntoSet` 으로 등록한다. |

## 공유 모듈 사용 지점

- `shared:core:common:app-shell`: `AppShell`, 딥링크 호스트 액티비티, 바텀바 상태
- `shared:feature:qr:scanner`: ZXing 기반 단일 QR 스캔 (qrscanner 앱은 ML Kit 다중 스캔 사용)
- `shared:feature:chart`: Vico 기반 차트
- `shared:feature:notification-inbox` / `notification-settings`: 알림함, FCM 토픽 토글
- `shared:feature:app-update`, `oss-licenses`, `webview`, `settings-ui`
- `shared:core:firebase:*`, `shared:core:common:admob`, `shared:core:common:time-sync`

## 의존 방향 규칙

1. 앱 -> feature, core, shared
2. feature -> core, shared (다른 feature 참조는 계층 가드 예외 소수만)
3. core -> shared (feature 금지)
4. 서로 의존하는 두 feature 중 한쪽은 상위(`my-numbers-overview` 가 `my-numbers` 와 `winning-checker` 를 의존하고 반대는 없다)
