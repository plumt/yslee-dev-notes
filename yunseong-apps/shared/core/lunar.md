# shared/core/lunar

1개 모듈. `:shared:core:lunar:calculator` (namespace `com.yunseong.lunar.calculator`).

## calculator

- **목적**: 달 위상, 거리, 출몰, 24절기와 화면용 달력 조립을 네트워크 없이 계산하는 순수 Kotlin 천문 모듈입니다.
- **기술**: `java.time` 과 `kotlin.math` 만 사용하며 외부 런타임 의존과 Hilt 가 없습니다. 모든 결과 시각은 KST `LocalDateTime` 입니다.
- **구현 방식**: UseCase 와 알고리즘이 순수 클래스이고, 주입 배선은 소비 앱(`apps/lunar/core/common` 의 `LunarCalculatorModule`)이 합니다. 외부 데이터를 받아 위상을 계산하는 `calculateFromData` 경로를 쓰며, 같은 위상을 한 달 동안 빠짐없이 모을 때는 `eventsOfTypeInMonth` 를 씁니다.
- **주의**: `ApiLunAgeAlgorithm.calculate` 는 `UnsupportedOperationException`, `Elp2000Algorithm` 은 미구현 stub 이라 프로덕션에서 쓰지 않습니다. 슈퍼문, 미니문 임계값은 천문학 정의가 아닌 앱이 정한 값입니다.
- **의존 관계**: 다른 모듈에 의존하지 않으며, `common/moon-viewing` 이 관측 지점 `GeoLocation` 타입을 받아 쓰고 lunar 앱이 주 소비자입니다.
