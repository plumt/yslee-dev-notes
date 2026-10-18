# QR Scanner (Q-Ring)

## Basic Information

- **App Name**: Q-Ring
- **Package ID**: `com.yunseong.qring` (dev 플레이버는 `.dev` 접미사)
- **Version Code**: 1
- **Version Name**: 1.0.0
- **Min SDK**: 29
- **Target SDK**: 36
- **Status**: Active Development
- **딥링크 스킴**: `yunseong-qring`

상세 설계는 [architecture.md](architecture.md) 를 본다.

## Description

Q-Ring 은 QR 코드 스캔·생성·기록을 4개 탭으로 묶은 독립 앱이다. 다중 QR 동시 인식(ML Kit), 로고와 색을 넣은 QR 생성(ZXing), 타입별 액션이 붙는 스캔 기록 관리를 제공한다. 앱 모듈은 조립만 하고 화면은 feature 모듈과 공유 모듈에 있다.

## API Keys & Credentials

별도 외부 API 키는 없다. 설정은 `clients/qrscanner/config.json` 의 약관·처리방침 URL 키 둘과 전역 설정의 문의 메일 하나를 NativeConfig(XOR 암호화 NDK)로 읽는다. 현재 약관·처리방침 URL 은 비어 있어 더보기 탭에서 해당 두 행이 보이지 않는다. 외부 서비스 의존은 Firebase(Crashlytics, Remote Config)뿐이며 설정 파일은 플레이버별 소스셋에 두고 저장소에 공개하지 않는다.

## Features

- **스캔(홈)**: 공용 ML Kit 다중 스캐너 화면을 띄워 여러 QR 을 동시에 인식하고 결과를 기록에 저장. 최근 스캔 카드를 누르면 타입별 액션(URL 열기 등) 실행
- **생성**: 로고 이미지와 색상을 지정해 QR 생성, 갤러리 저장(MediaStore, 저장소 권한 불필요), 공유
- **히스토리**: 스캔 기록 목록, 타입 분류, 즐겨찾기 필터, 삭제, 조회 실패 재시도
- **더보기**: 앱 버전, 문의, 약관·처리방침, 오픈소스 라이선스, 기록 전체 삭제
- **강제 업데이트**: 스플래시에서 Remote Config 기반 버전 확인 게이트

## 화면 구성

스플래시(시작) -> 홈 · 생성 · 히스토리 · 더보기 4개 탭. 개인정보 처리방침/이용약관은 바텀바를 끄는 보조 목적지이고 오픈소스 라이선스는 별도 Activity 다.

## Dependencies

### App-specific Modules
- `:apps:qrscanner:feature:splash`, `home`, `history`, `more`

### Shared Feature Modules
- `qr:generator` (생성 탭 화면, 앱 feature 로 감싸지 않고 직접 배선), `qr:common:qr-common-impl` (기록 저장소 구현)
- `image-picker`, `image-saver`, `share`, `oss-licenses`, `webview`, `legal`

### Shared Core Modules
- `common:app-shell` (셸·딥링크 호스트), `firebase:crashlytics`, `fundamental:common`

### External Services
- **Firebase**: Crashlytics, Remote Config

## Build Configuration

플레이버(`dev`, `prod`)·서명·빌드 타입은 Convention Plugin 이 만들며 앱은 재선언하지 않는다. 서명 키는 빌드 타입이 아닌 플레이버 기준(dev 플레이버는 dev 키, prod 는 운영 키)으로 갈린다.

```bash
./gradlew :apps:qrscanner:assembleDevDebug     # dev 디버그
./gradlew :apps:qrscanner:assembleProdRelease  # 운영 릴리스
./gradlew :apps:qrscanner:installDevDebug      # 기기 설치
```

빌드 전 검증(`preBuildVerify`)은 딥링크 스킴과 매니페스트의 일치, release 서명 설정만 검사한다(필수 키·광고 키 목록은 비어 있음).

## Testing

```bash
./gradlew :apps:qrscanner:test                 # 유닛 테스트
./gradlew :apps:qrscanner:connectedAndroidTest # 계측 테스트
```

`DeepLinkModuleMappingTest` 가 허용 host 집합과 라우터 매핑의 일치를 검증한다. E2E 는 testTag 기반 Appium 하네스를 쓰며 dev 플레이버 고정이다.

## Architecture 요약

- **UI**: Jetpack Compose + Material 3, 밝은 종이 테마 하나(시스템 다크 모드를 따르지 않음)
- **DI**: Hilt
- **Navigation**: Navigation Compose, 각 목적지가 바텀바 표시를 직접 제어
- **Security**: NDK/JNI XOR 암호화 설정
- **Build System**: Convention Plugins + Version Catalog

자세한 내용은 [architecture.md](architecture.md).

## Notes

- 단일 스캔(ZXing)은 공유 모듈 `qr:scanner` 가 있고 Lottomate 가 쓴다. Q-Ring 은 ML Kit 다중 스캐너를 쓴다
- 앱 매니페스트에는 권한이 없다. 카메라 등은 쓰는 공유 모듈이 병합한다
- 릴리스 빌드는 R8/ProGuard 가 적용된다

## 스크린샷

스크린샷은 [screenshots](../../screenshots/qrscanner/) 폴더를 참고한다.
