# social-login 그룹

소셜 로그인을 계약(`core`), 화면 뼈대(`base`), 제공자(`google`/`kakao`/`naver`)로 나눈 6개 모듈이다. 제공자는 의존을 추가하는 것만으로 활성화된다.

## social-login-core

- **목적**: 계약·모델·UseCase 만 담고 특정 소셜 SDK 의존이 없다.
- **구현**: `SocialLoginProviderRegistry` 가 Hilt 멀티바인딩으로 주입된 Provider 집합을 수단별 맵으로 접고, 요청 수단의 Provider 에 위임한다. 미등록 수단은 예외 대신 `AuthResult.Failure.NotSupported` 로 돌려준다. 한 수단에 구현은 하나다.
- **결과 계약**: `AuthResult` 는 2단 sealed 계층(성공: 로그인·로그아웃·연결 해제 / 실패: 취소·미지원·망·서버·인증·SDK·알 수 없음)이다. 실패를 값으로 두어 화면이 예외 처리 없이 분기한다. 토큰만 필수이고 이메일·인가 코드는 선택이다.
- **주의**: `SocialLoginCoreModule` 의 `@Multibinds` 를 지우면 Provider 를 붙이지 않은 앱이 컴파일에 실패한다.

## social-login-base

- **목적**: 로그인 화면 공통 상태홀더. 현재 어느 앱도 상속하지 않는 재고 모듈이다.
- **구현**: 화면이 `start()` 를 부를 때 등록 수단을 거른 `availableTypes` 를 계산한다(`init` 에서 부르면 하위 프로퍼티가 비어 있다). 결과는 성공, 취소, 나머지 실패 순으로 읽는다. 취소는 오류가 아니라 `LoginEffect.Cancelled` 이고, 이동은 `LoginEffect.SignedIn` 을 받은 화면이 정한다.
- **의존**: `social-login-core`(api), `core:fundamental:presentation-base`(api).

## 제공자 모듈

| 모듈 | 방식 | 요점 |
|---|---|---|
| google | Credential Manager 로 Google ID 토큰을 받아 Firebase 로 로그인 | 앱이 `@GoogleServerClientId` 만 제공. 포그라운드 Activity 로 조회. 계정 필터를 꺼 첫 로그인이 막히지 않게 함 |
| kakao | 카카오톡 앱 로그인 우선, 취소가 아니면 웹 로그인으로 한 단계 폴백 | 앱이 `@KakaoAppKey` 제공. SDK 초기화는 첫 호출 때 한 번 지연 수행 |
| naver | 네이버 OAuth 콜백을 코루틴에 이어 붙임. 로그인·로그아웃·탈퇴 지원 | 앱이 한정자 3개 제공. 초기화는 이중 검사 잠금으로 한 번 |

공통: 키는 모듈에 두지 않고 앱이 바인딩하며, 실패는 `AuthResult.Failure` 값으로 돌려주고, 넓은 `catch` 첫 줄에서 취소를 재던진다. 제공자 SDK 의존은 `api` 로 올리지 않는다. Google 을 쓰는 앱은 google-services 플러그인과 설정 파일을 갖춰야 한다.

의존: google → `core:fundamental:common`(ActivityProvider), `core:firebase:auth`. kakao → `core:fundamental:common`, 카카오 user SDK. naver → `core:fundamental:common`.
