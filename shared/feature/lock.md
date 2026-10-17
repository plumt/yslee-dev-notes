# lock 그룹

앱 잠금(패턴·PIN·생체 인증)을 계약, 구현, 화면 세 모듈로 나눈다. 현재 채택한 앱은 없다(`lock-impl` 은 settings 등록만).

## lock-api

- **목적**: 모델과 계약만 담는다. 저장·암호화·생체 인증 라이브러리 의존이 없다.
- **구현**: `LockType` 은 PATTERN / PIN / BIOMETRIC / NONE 이고 방식은 하나만 선다. `LockRepository` 는 DataStore·Keystore 를, `BiometricAuthenticator` 는 `androidx.biometric` 을 숨기지만, 생체 계약은 `FragmentActivity` 를 받아야 해서 노출한다. 원문을 돌려주는 연산은 계약에 없고 저장과 대조만 있다.
- **의존**: `kotlinx-coroutines-android`(api), `androidx-fragment-ktx`(api).

## lock-impl

- **목적**: 잠금 설정 저장, 키 암호화, 생체 인증 실행. 밖에는 계약 바인딩만 내놓는다(구현은 `internal`).
- **기술**: Android Keystore(AES 256비트 키, 밖으로 나오지 않음), AES/GCM/NoPadding, DataStore Preferences(`lock_settings`), `androidx.biometric`, Hilt.
- **구현**: 패턴과 PIN 은 암호문과 IV 를 Base64 로 저장하고, 대조는 복호화해 상수 시간으로 비교한다(조기 반환은 타이밍 공격 경로). 잠김 여부는 값이 없으면 잠김으로 읽는다.
- **의존**: `lock-api`(api), `core:fundamental:common`.

## lock-ui

- **목적**: 잠금 화면과 설정 화면. 계약만 의존하고 구현을 모른다.
- **구현**: `LockGuard` 가 상태홀더를 소유하고 `LockScreen` 에는 상태와 입력 콜백만 내려 단독 재사용과 미리보기가 쉽다. 액션은 `onIntent` 단일 진입점이고 처리 함수는 `private` 이다. 검증 실패 문구는 효과가 아니라 상태에 남긴다. 생체는 방식이 생체이고 가용할 때만 띄우며 호스트는 `FragmentActivity` 여야 한다.
- **의존**: `lock-api`(api), `core:fundamental:common`, `convention.feature`. 바인딩은 앱이 `lock-impl` 을 함께 넣어 붙인다.
