# Diding

반복 규칙이 있는 할 일을 관리하고 정해진 시각에 알림을 울려 주는 앱이다. 서버 없이 기기 안에서만 동작한다(완전 오프라인). Jetpack Compose 와 멀티모듈 구조를 쓰고, 이 모노레포의 앱 중 앱 안에 도메인 모듈을 따로 둔 유일한 앱이다.

상세 문서: [architecture.md](architecture.md) (구조·설계 결정) · [features.md](features.md) (화면·기능별 동작)

## Basic Information

- **App Name**: Diding (표시 이름 prod "디딩", dev "디딩 DEV")
- **Package ID**: `com.yunseong.diding` (dev 플레이버는 `.dev` 접미사)
- **Version Code**: 1
- **Version Name**: 1.0.0
- **Min SDK**: 29
- **Target SDK**: 36
- **Status**: Active Development
- **Gradle 경로**: `:apps:diding`
- **플레이버**: `dev` · `prod` (규약 플러그인 allowlist 가 생성)
- **딥링크 스킴**: `yunseong-diding`

## Description

하단 탭 네 개(오늘, 관리, 캘린더, 설정)로 이루어진다. 오늘 해당하는 할 일을 체크하고, 반복(매일/매주/매월) 할 일을 추가·수정·보관하며, 월 달력에서 날짜별 수행률을 색으로 확인하고, 할 일별 알림 시각에 `AlarmManager` 정확 알람으로 알림을 받는다. 외부 API 키나 Firebase 에 의존하지 않는다.

## Features

- **오늘**: 오늘 해당하는 할 일 목록과 진행 카드, 완료 토글. 자정마다 목록이 갱신된다
- **관리**: 반복 할 일 추가·수정·보관(바텀시트), 매일/매주(요일 선택)/매월(일자 선택) 반복, 할 일별 알림 켜기와 시각 지정
- **캘린더**: 월별 달력에 날짜마다 할 일 수행률을 색으로 표시
- **설정**: 알림 마스터 스위치, 앱 버전, 문의, 오픈소스 라이선스
- **알람**: 정확 알람 예약, 발화 후 다음 알람 재예약, 재부팅·앱 교체 뒤 자동 복구
- **딥링크**: `yunseong-diding://home | management | calendar | setting`

## 외부 서비스와 키

없다. Firebase, AdMob, 외부 API 를 쓰지 않으며 플랫폼 키 의존도 없다. 이 때문에 dev/prod 플레이버 롤아웃의 첫 파일럿 앱이 되었다. 문의 메일 주소만 공용 설정 파일의 값을 NDK `NativeConfig` 로 암호화해 읽는다(값은 이 문서에 두지 않는다).

## Dependencies

### 앱 전용 모듈
- `:apps:diding:core:domain` - 할 일·완료 모델, 저장소 계약, 반복 판정, 달성률, 유스케이스 (Room·Hilt·Compose 를 모르는 순수 계층)
- `:apps:diding:core:database` - 암호화 Room DB, 저장소 구현, 알람 스케줄러 포트, 알림 마스터 스위치
- `:apps:diding:core:alarm` - 정확 알람 스케줄러 구현과 발화·재예약 리시버
- `:apps:diding:feature:home` / `management` / `calendar` / `setting` - 탭 화면

### 공유 모듈
- `:shared:core:common:app-shell` - 앱 셸·딥링크 호스트 Activity
- `:shared:core:fundamental:common` - 시각 공급(`TimeProvider`), 설정 저장소, 로거
- `:shared:core:fundamental:alarm` - 정확 알람·`PendingIntent` 호환 계층, 재부팅 재예약 리시버
- `:shared:core:fundamental:local-notification` - 알림 채널·발송
- `:shared:core:fundamental:permission`, `:shared:core:permissions:permission-notification` - 권한 요청
- `:shared:core:common:ui`, `designsystem`, `navigation-base`, `database`(암호화 DB 팩토리)
- `:shared:feature:oss-licenses`, `settings-ui`

### 주요 라이브러리
Hilt, Jetpack Compose + Material 3, Navigation Compose, Room, Kotlin Coroutines/Flow, `AlarmManager`

## Build Configuration

```bash
# Debug build (dev 플레이버)
./gradlew :apps:diding:assembleDevDebug

# Release build (prod 플레이버)
./gradlew :apps:diding:assembleProdRelease

# Install on device
./gradlew :apps:diding:installDevDebug
```

규약 플러그인은 `convention.android.application`, `hilt`, `compose`, `buildconfig`, `ndk.nativeconfig`, `verify.prebuild` 여섯 개다. 서명은 buildType 이 아니라 flavor 로 갈리고(dev 플레이버는 dev 키, prod 는 prod 키) 서명 정보는 환경변수 또는 로컬 설정 파일에서 읽는다.

## Testing

```bash
./gradlew :apps:diding:core:alarm:test     # 알람 다음 발화 시각 계산 등
```

앱 모듈 자체의 `src/test` 에는 파일이 없다. 알람 계산 테스트는 `core:alarm` 이 갖고, E2E 는 Appium 스모크 하네스(`appiumTest/`, dev 플레이버)가 맡는다.

## Notes

- 알림 채널 `diding_todo` 는 `DidingApplication` 이 시작할 때 만들고, 알림 설정 판정은 시작 코드가 아니라 스케줄러가 스스로 한다
- 테마는 밝은 종이 톤 고정(accent 그린)이며 시스템 다크 설정을 읽지 않는다
- `allowDestructiveMigration = true` 상태라 스키마 버전을 올리려면 먼저 `Migration` 을 쓰고 이 플래그를 지워야 한다(안 그러면 할 일·완료 이력이 전부 삭제된다)
- 앱 이름은 `src/dev`·`src/prod` 의 `strings.xml` 이 가르며 `src/main` 에는 `app_name` 을 두지 않는다
- 자세한 한계와 함정은 [architecture.md](architecture.md) 의 마지막 절 참고
