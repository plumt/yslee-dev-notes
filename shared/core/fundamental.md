# shared/core/fundamental

5개 모듈. Gradle 경로는 `:shared:core:fundamental:<이름>`. 저장소 전체의 최하위 기반입니다.

| 모듈 | 목적 | 핵심 기술 | project 의존 |
|---|---|---|---|
| common | 시각, 앱 정체성, 로깅, 설정 저장의 포트와 유틸 | Hilt, 코루틴, BuildConfig | 없음 |
| permission | 런타임 권한 계약 (구현은 permissions 그룹) | Hilt Multibinds, activity-compose | common |
| presentation-base | 상태홀더 베이스와 화면 상태 계약 | ViewModel, StateFlow | common |
| alarm | 정확 알람 유틸과 부팅 후 재예약 리시버 | AlarmManager, BroadcastReceiver | common |
| local-notification | 로컬 알림 채널 생성, 발송, 취소 | NotificationManager | 없음 |

## 모듈별 설명

- **common** (`com.yunseong.common`): 가장 넓게 쓰이는 모듈로 직접 의존하는 모듈이 82개입니다. `TimeProvider`(서버 시간 보정이 반영된 시각), `AppVersionProvider`, `AppLaunchInfo`(Optional 로 받음), `Logger`, `CrashReporter` 포트, `ActivityProvider`, `AppPreferences`(`@Inject` 생성자로만 바인딩), `rethrowIfCancellation()`, `coroutineErrorHandler`, 포맷과 인텐트 유틸을 담습니다. `SupportEmail` 문자열은 앱이 바인딩합니다. 수명이 긴 스코프에는 에러 핸들러를 얹어 예외 한 건이 프로세스를 끝내지 않게 합니다.
- **permission**: 권한 요청의 계약(`PermissionType`, `PermissionProvider`, Registry, `EnsurePermissionUseCase`, `CheckSensorUseCase` 등)만 둡니다. 구체 Provider 는 permissions 그룹 모듈이 `@Provides @IntoSet` 으로 기여합니다. 매니페스트에 `uses-permission` 을 넣지 않아 계약만 의존해도 권한이 병합되지 않으며, Compose 의 활동 결과 런처로 요청합니다. `@Multibinds` 와 `@JvmSuppressWildcards` 를 함께 유지해야 Hilt 배선이 유지됩니다.
- **presentation-base**: Compose 를 모르는 상태홀더 베이스입니다. `UiState`(불변 data class), `LoadState`(로딩, 오류, 데이터를 하나로), Intent 입력(`onIntent`), 1회성 사건은 `sendEffect` 로 보내는 MVI 풍 구조를 표준화합니다. 코루틴 실패는 `launchErrorHandler` 로 보고합니다. Compose 측 소비는 `common/ui-base` 가 담당합니다.
- **alarm**: `SCHEDULE_EXACT_ALARM` 계열 정확 알람 예약과 `PendingIntent` 플래그의 SDK 분기 유틸, 재부팅 및 앱 교체 뒤 알람을 다시 거는 `BootRescheduleReceiver` 를 제공합니다. 다시 걸 대상은 소비 모듈이 `AlarmRescheduler` 를 `@IntoSet` 으로 기여해 정하고, 리시버는 외부에 열려 있어 액션 검사를 유지합니다. 정확 알람 권한은 예약하는 소비 모듈이 선언합니다.
- **local-notification**: 앱이 직접 발행하는 알림의 채널 생성과 발송, 취소를 담습니다(원격 푸시는 firebase/messaging). `sendNotification` 은 권한 미허용과 `SecurityException` 을 `false` 로 접어 반환하므로 호출부가 결과를 확인해야 합니다. 권한 요청은 소비처 몫이고, 순환을 피하려고 permission 모듈을 의존하지 않고 권한 확인을 자체 구현합니다.
