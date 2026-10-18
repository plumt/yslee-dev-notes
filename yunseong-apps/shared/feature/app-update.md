# app-update 그룹

앱 시작 시점의 관문을 맡는 세 모듈이다. 버전 확인(`app-update`), 스플래시 뼈대(`splash-base`), 루팅 검사(`security-root`).

## app-update

- **목적**: Remote Config 로 강제 업데이트 여부를 판정하고, 외관만 바꿔 끼울 수 있는 안내 다이얼로그를 제공한다.
- **기술**: Firebase Remote Config, Compose 다이얼로그(기본 렌더러는 공용 확인 다이얼로그).
- **구현**: `check()` 는 세 분기(`UpToDate` / `UpdateRequired` / `FetchFailed`)로만 답한다. 초기화·조회·비교 전체가 한 try 안에 있어 어디서 예외가 나도 `FetchFailed` 이고, 취소만 다시 던진다. 스플래시마다 최신값을 받으려고 최소 fetch 간격을 0 으로 둔다.
- **비교 규칙**: 강제 플래그가 꺼져 있으면 마지막 자리를 뺀 앞자리만 비교하고, 켜져 있으면 모든 자리를 비교한다.
- **경계**: 문구 4종·버튼 2개·순서·닫힘 동작은 슬롯으로 열지 않는다. `onDismiss` 는 앱 종료에만 쓴다. 우회로가 생기면 강제가 무의미하다.
- **의존**: `core:fundamental:common`(버전 비교), `core:firebase:remote-config`, `core:common:ui`.

## splash-base

- **목적**: 스플래시 상태홀더와 화면의 공통 뼈대. 앱은 실패 정책·후속 작업·브랜드 표시만 정한다. 4개 앱(fxrate, lottomate, lunar, qrscanner)이 상속한다.
- **구현**: Kotlin 은 상위 `init` 을 하위 프로퍼티 초기화보다 먼저 실행하므로, base 는 `init` 에서 흐름을 시작하지 않고 화면이 `startVersionCheck()` 로 트리거한다. 같은 이유로 추상 멤버는 `val` 이 아니라 `fun` 으로 연다.
- **정책**: 확인 실패 정책(막기 / 통과)은 앱마다 갈리는 유일한 축이라 기본값 없이 반드시 고르게 한다. 후속 작업은 별개의 축이다.
- **보안 검사**: 보안 검사 집합은 멀티바인딩이며 기본은 빈 집합이다. `SecurityBlocked` 에는 재시도 경로를 열지 않는다.
- **의존**: `app-update`(api), `core:fundamental:common`.

## security-root

- **목적**: 루팅 흔적이 보이면 앱 진입을 막는 보안 검사. 의존을 걸면 Hilt 로 등록되고, 앱이 검사 집합을 스플래시 상위 생성자로 넘길 때 실행된다.
- **구현**: 빌드 태그의 테스트 키, `su` 바이너리 경로 목록, 루팅 관리 앱 패키지 조회 등을 신호로 본다. 첫 실패에서 순회가 끝난다. `QUERY_ALL_PACKAGES` 대신 매니페스트 `<queries>` 로 6개 앱만 조회한다.
- **한계**: 완전한 방어가 아니다. Magisk 류는 신호를 숨길 수 있고 커스텀 ROM 사용자가 걸릴 수도 있다. 진짜 무결성이 필요하면 서버 검증을 함께 건다. 사용자 문구에는 경로·패키지명을 넣지 않는다.
- **의존**: `splash-base`, `core:fundamental:common`(로거).
