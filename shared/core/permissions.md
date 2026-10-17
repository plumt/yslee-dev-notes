# shared/core/permissions

권한 하나에 Gradle 모듈 하나를 두는 Provider 구현 묶음입니다(11개). 경로는 `:shared:core:permissions:permission-<이름>` 이며 우산 디렉터리 `permissions` 자체는 Gradle 모듈이 아닙니다. 계약(`PermissionType`, `PermissionProvider`, `PermissionProviderRegistry` 등)은 [fundamental/permission](fundamental.md) 이 갖습니다.

## 설계

- 각 모듈은 Provider 와 Hilt 모듈 하나로 구성되고 `@Provides @IntoSet` 으로 Registry 에 기여합니다. 즉 모듈을 의존하는 것만으로 등록됩니다.
- `uses-permission` 은 해당 Provider 모듈 매니페스트가 선언합니다. 앱은 실제 쓰는 권한의 모듈만 의존해 불필요한 권한이 스토어에 표기되지 않게 합니다.
- 한 `PermissionType` 을 지원하는 Provider 는 저장소 전체에서 하나이며 `PermissionProviderGuard` 의 `TYPE_TO_MODULE` 에 등록해 빌드에서 검증합니다.
- 호출은 Provider 를 직접 주입하지 않고 Registry 나 UseCase 로 합니다. SDK 버전 경계는 `permissionsFor` 와 매니페스트의 `maxSdkVersion` 이 일치해야 합니다.
- 하드웨어 필요 여부는 `required="false"` feature 로 선언하고, 사용 전 `CheckSensorUseCase` 로 확인합니다.
- 모든 모듈의 의존은 `fundamental/common`(`ActivityProvider`), `fundamental/permission`(계약, Base, 모델) 두 개입니다.
- 빌드 스크립트는 `convention.android.library` 다음에 `convention.android.hilt` 순서입니다.

## 모듈 표

| 모듈 | 요청 대상 | 선언 권한 | 비고 |
|---|---|---|---|
| permission-bluetooth | BLUETOOTH | BLUETOOTH_CONNECT, BLUETOOTH(max 30) | API 31 이 요청 권한 경계, 하드웨어 판정 포함 |
| permission-calendar | CALENDAR | READ_CALENDAR, WRITE_CALENDAR | 읽기, 쓰기를 한 타입으로 요청, 읽기 전용이 필요하면 새 타입과 모듈로 분리 |
| permission-camera | CAMERA | CAMERA, camera feature(optional) | 하드웨어 판정 포함 |
| permission-contacts | CONTACTS | READ_CONTACTS | 읽기 전용, Play 권한 사용 선언 필요 |
| permission-core | 배터리 최적화 제외 | REQUEST_IGNORE_BATTERY_OPTIMIZATIONS | 런타임 권한이 아닌 설정 인텐트로 요청, 허용 여부는 복귀 후 `isGranted` 로 재확인 |
| permission-gallery | 갤러리 이미지 | READ_MEDIA_IMAGES, READ_EXTERNAL_STORAGE(max 32) | API 33 경계, 파일 선택만이면 Photo Picker 우선 |
| permission-location | 전경 위치 | FINE, COARSE, gps feature(optional) | GPS 하드웨어 판정, 시스템 위치 설정 여부는 별도 |
| permission-location-background | 배경 위치 | ACCESS_BACKGROUND_LOCATION | permission-location 과 함께 의존, Play 별도 심사 대상 |
| permission-microphone | RECORD_AUDIO | RECORD_AUDIO, microphone feature(optional) | 하드웨어 판정 포함 |
| permission-motion | ACTIVITY_RECOGNITION | ACTIVITY_RECOGNITION | 디렉터리 이름만 motion, API 29 미만 분기 보존 |
| permission-notification | NOTIFICATION | POST_NOTIFICATIONS | API 33 경계로 요청 경로 분기, `Unavailable` 갈래 처리 필요 |
