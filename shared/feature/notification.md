# notification 그룹

알림함(`notification-inbox-*`)과 알림 설정(`notification-settings`).

## notification-inbox-api

- 알림 계약과 도메인 모델만 담는다. 읽지 않은 개수 배지나 알림 적재처럼 일부만 필요한 소비자가 DB 컴파일러·메시징·화면 의존을 받지 않도록 분리했다.
- 조회 연산은 전부 `Flow`, 변경 연산만 suspend 다. 알림은 앱 밖에서 도착하므로 배지와 목록이 같은 시각의 값을 보게 하려는 결정이다.
- 딥링크는 저장 시점에 검증하지 않는다. 막으면 알림이 알림함에서 통째로 사라지고, 판정 기준(화이트리스트)은 앱마다 달라 이동 시점에 두는 편이 낫다.

## notification-inbox-impl

- **기술**: Room(KSP), 공용 암호화 DB 팩토리, FCM 메시지 핸들러 계약, Hilt. 스키마는 `notifications` 테이블 버전 2이며 스키마 내보내기를 켜 마이그레이션 테스트의 전제로 쓴다.
- **수신 경로**: 라우터가 비어 있지 않은 `type` 을 확인하고, 핸들러는 `canHandle` 을 재정의해 제목·본문 키 유무로 선별한다. 딥링크는 `deeplink`, `link` 순으로 본다. 목록은 `receivedAt` 내림차순이고 읽지 않은 수는 읽음 플래그 0 의 개수다.
- DB 는 공용 팩토리로만 만든다(직접 빌더를 부르면 암호화와 키 관리가 갈라진다). 스키마 버전을 올리면 마이그레이션을 함께 등록한다.
- **의존**: api(api), `core:fundamental:common`, `core:common:database`, `core:firebase:messaging`.

## notification-inbox-ui

- 계약만 의존하는 알림함 화면. 래퍼 화면(상태홀더·진입 시 전체 읽음 발행) / 목록 본문(상태홀더를 모름) / 상태홀더 세 층이다.
- 목록과 읽지 않은 수는 계약 흐름을 그대로 쓰고 화면 상태로 합치지 않는다. 합치면 구독 수명과 초기 방출 계약이 깨진다. 날짜 그룹 순서는 라벨 문자열이 아니라 그룹 안 최신 수신 시각으로 정한다. 현재 lottomate 가 채택한다.

## notification-settings

- **목적**: FCM 토픽 구독/해제를 켜고 끄는 UseCase. 연타를 흡수한다.
- **구현**: `ConcurrentHashMap<String, Job>` 으로 토픽당 Job 하나만 남기고, 새 호출 진입 시 이전 Job 을 취소한 뒤 등록한다. 완료 시에는 자기 Job 만 지운다. `@Singleton` 이라 같은 토픽을 쓰는 화면끼리는 칸을 공유한다. 취소된 토글은 콜백 없이 끝나며 `onFailure` 로 흘리지 않는다.
- 권한 확인은 호출부 몫이다. 같은 토픽의 호출은 한 스레드에서 차례로 와야 한다.
- **의존**: `core:fundamental:common`, `core:firebase:messaging`. 소비 앱도 이 둘을 직접 선언한다.
