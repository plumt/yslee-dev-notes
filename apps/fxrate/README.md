# FxRate (환율)

## Basic Information

- **App Name**: 환율 (dev 빌드는 "환율 DEV")
- **Package ID**: `com.yunseong.fxrate` (dev 플레이버는 `.dev` 접미사)
- **Version Code**: 1
- **Version Name**: 1.0.0
- **Min SDK**: 29
- **Target SDK**: 36
- **Status**: In Development (서버 연동 미구현)
- **딥링크 스킴**: `yunseong-fxrate`

## Description

통화별 환율을 보여주고 금액을 환산하며, 임계값 알림 규칙을 관리하고, 홈 화면 위젯으로 즐겨찾기 환율을 띄우는 앱이다. 화면과 위젯은 환율을 Room 캐시에서만 읽는 구조다. 캐시를 서버 스냅샷으로 채우는 경로는 계획 단계이며, 지금은 debug 빌드의 개발용 목 데이터만 캐시를 채운다. release 빌드는 캐시가 비어 있다. 구조는 [architecture.md](architecture.md), 기능별 상세는 [features.md](features.md) 를 본다.

## API Keys & Credentials

현재 이 앱에는 필요한 키가 없다.

- 앱이 외부 환율 API 를 직접 부르지 않으므로 환율 API 키를 앱에 두지 않는다 (서버 중계 계획의 키는 서버 비밀 관리에만 둔다)
- `clients/fxrate/config.json` 의 앱 섹션은 비어 있고 필수 키 목록(`requiredConfigKeys`)도 비어 있다
- 전 앱 공통 설정의 `support_email`(문의 메일)만 `NativeConfig` 접근자로 쓴다
- Firebase 설정 파일도 없다. google-services 없이 빌드되며, 업데이트 확인만 실패로 통과 처리된다

## Features

- **환율 탭**: 기준 통화 환산, 목록 전체 금액 환산, 즐겨찾기 순서 편집, 통화 상세 시트
- **알림 탭**: 임계값 알림 규칙 목록·추가·수정·삭제·켜고 끄기 (최대 20개)
- **설정 탭**: 기준 통화 변경, 출처·라이선스·문의·앱 버전
- **스플래시**: 강제 업데이트 게이트
- **홈 화면 위젯**: 즐겨찾기 통화 환율을 런처에 표시 (Glance)

## Dependencies

### App-specific Modules
- `:apps:fxrate:core:common` - 환산·카탈로그·표기·소스 계약 (순수 JVM 로직)
- `:apps:fxrate:core:database` - Room 캐시, 즐겨찾기, 알림 규칙, 기준 통화 설정
- `:apps:fxrate:core:widget` - Glance 위젯
- `:apps:fxrate:feature:rates` / `alerts` / `settings` / `splash`

### Shared Modules
- `app-shell`(앱 셸·딥링크), `app-update`(업데이트 게이트), `settings-ui`, `oss-licenses`, `favorites`, `widget-base`, `designsystem`, `fundamental:common`

### External Services
- 없음 (Firebase·광고 미사용). 계획 중인 서버 중계는 Firestore + Functions + FCM

## Build Configuration

```bash
./gradlew :apps:fxrate:assembleDevDebug
./gradlew :apps:fxrate:assembleProdRelease
./gradlew :apps:fxrate:installDevDebug
```

### Flavors

`dev`, `prod`. 규약 플러그인의 allowlist 로 생성되며 서명은 플레이버 기준이다. debug 소스셋에만 목 데이터가 있고 release 소스셋의 시드 구현은 아무 일도 하지 않는다.

## Testing

```bash
./gradlew :apps:fxrate:test
```

환산·서식·DAO 변동 기준·알림 편집 상태 같은 로직이 JVM 단위 테스트로 검증된다.

## Notes

- 서버 연동(스냅샷 구독, 알림 반영, FCM)은 계획 문서만 있고 구현되지 않았다. 알림 규칙은 로컬에만 쌓이고 "반영 대기" 로 표시된다
- 환율 소스 후보의 약관(상업 이용·캐싱·재배포)은 출시 전 확인이 남아 있다
