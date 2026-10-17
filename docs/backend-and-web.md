# 백엔드와 웹

Android 앱을 지원하는 Firebase 백엔드와 React 웹의 개요입니다. 세 모듈 모두 Gradle 모듈이 아니며 각자 `AGENTS.md`와 `docs/`를 가집니다.

| 모듈 | 런타임 | 역할 |
|------|--------|------|
| `backend` | Node.js 20, TypeScript 5, firebase-functions 5 (v2) | Lottomate용 Cloud Functions와 관리자 API |
| `functions` | Node.js 20, firebase-functions 5 (v2), firebase-admin 12, astronomia, axios | 로또 당첨 확인과 Lunar 달 이벤트 푸시 |
| `frontend/lottomate-web` | React 19, React Router 7, Vite 7, TypeScript 5.9, Firebase JS SDK 12 | 방침·약관 페이지와 관리자 콘솔 |

모든 함수는 서울 리전(`asia-northeast3`)에 배포하며, 앱마다 백엔드 디렉터리를 따로 두고 다른 앱의 소스를 import하지 않습니다.

## backend (Lottomate 백엔드)

TypeScript로 작성한 앱별 Firebase 백엔드 묶음이며 지금은 Lottomate의 Cloud Functions와 Firebase 배포 설정을 담습니다.
배포 단위는 함수 11개입니다.

### 구성

| 영역 | 역할 |
|------|------|
| 진입점 | Admin SDK 초기화와 함수 export |
| `fcm/` | 회차 조회·저장, FCM 발송, 스케줄·HTTP 트리거 |
| `api/` | 관리자용 Remote Config, Secret Manager, 통계 HTTP 함수 |
| `auth/` | 관리자 HTTP 함수를 감싸는 호출자 검증 헬퍼 |
| `types/` | 응답·결과 타입 |

### 함수

- 스케줄 함수 2개: 매시 정각 새 회차 확인 후 당첨 알림 발송, 토요일 정오 구매 알림 발송
- 관리자 HTTP 함수 9개: 수동 알림 발송, 통계, Remote Config 조회·수정·삭제, Secret 목록·조회·생성/수정·삭제

### 설계 포인트

- **당첨 확인 흐름**: 저장된 최신 회차에 1을 더한 회차를 외부 추첨 결과 API에 묻고, 응답이 없으면 미발표로 보고 끝냅니다.
  응답이 있으면 Firestore에 저장한 뒤 최신 회차를 다시 읽어 FCM을 보냅니다.
  한 실행은 한 회차만 전진하므로 저장이 밀려 있으면 매시 한 회차씩 따라잡습니다.
- **저장과 발송은 한 단위가 아닙니다.** 발송 여부는 기록하지 않고 저장된 최신 회차 번호만으로 다음 실행이 할 일을 정합니다.
  단순하지만 저장 뒤 발송이 실패한 회차는 다시 보내지 않는다는 트레이드오프가 있어 유의사항으로 문서화했습니다.
- **구매 알림**: 새 회차 확인 없이 다음 회차 번호를 실어 보내고, 앱의 핸들러가 로컬 구매 이력을 보고 없을 때만 알림을 띄웁니다.
- **관리자 인증**: 관리자 HTTP 함수는 공통 래퍼(`onAdminRequest`)로 정의합니다. 래퍼가 OPTIONS 처리,
  Bearer ID 토큰 추출·검증, 관리자 목록 확인을 본문 핸들러보다 먼저 수행하므로 `onRequest`로 직접 열면 인증 없이 배포되는 실수를 막습니다.
- **초기화 순서**: `admin.initializeApp()`을 모든 export보다 위에 두고, Firestore는 공용 인스턴스로만 접근합니다.
- 시크릿 값은 소스나 설정 파일이 아니라 Secret Manager에 둡니다.
- 함수 이름, FCM 토픽, data 키, Firestore 필드를 바꾸면 Android 앱과 웹 소비처를 같은 변경에서 고칩니다.
- 배포 전에는 반드시 빌드를 거칩니다.

## functions (로또 + 달 이벤트)

JavaScript로 작성한 Cloud Functions로, 로또 당첨 확인과 Lunar 앱의 달 이벤트 푸시를 한 파일에서 내보냅니다.
함수 6개가 있습니다.

| 함수 | 트리거 | 하는 일 |
|------|--------|--------|
| `scheduledMoonEventCheck` | 매일 09:00 (KST) | 그날 달 이벤트가 있으면 토픽별 FCM 발송 |
| `testMoonEventNotification` | HTTP | 지정 날짜의 달 이벤트 계산 결과 조회, 요청 시 발송 |
| `scheduledLottoCheck` | 토요일 20:55 | 다음 회차 발표 시 저장 후 당첨 알림 |
| `sendLottoCheckNotification` | HTTP | 확인·저장·발송을 한 번 실행 |
| `testLottoCheckNotification` | HTTP | 고정 테스트 데이터로 알림 발송 |
| `initializeLottoData` | HTTP | 과거 회차 초기 적재 |

### 달 이벤트 계산

- 외부 API 없이 Meeus 알고리즘 구현 라이브러리(`astronomia`)로 삭·망 시각과 지구-달 거리를 계산합니다.
- 계산 로직은 별도 파일에 격리하고 Firestore나 로또 코드를 부르지 않아 순수 함수로 검증할 수 있습니다.
  고정 기대값과 대조하는 로컬 검증 스크립트가 있습니다.
- 망의 거리로 슈퍼문, 미니문, 일반 보름달을 분류하고, 같은 양력 달에 망이 두 번이면 두 번째를 블루문으로 판정합니다.
  분류 임계값은 앱의 감지기와 같은 값이어야 하므로 한쪽을 바꾸면 반대쪽도 같이 바꿉니다.
- 토픽은 둘로 나눕니다. 삭과 일반 망은 위상 토픽, 슈퍼문·미니문·블루문은 특별 이벤트 토픽입니다.
  같은 날 여러 이벤트가 겹치면 우선순위 점수로 대표 이벤트를 고릅니다.
- KST 날짜는 시간대 설정에 기대지 않고, 9시간을 더한 값을 UTC 게터로 읽어 만듭니다.
- 로또 회차 문서 ID는 4자리 0 채움 형식이며, 형식을 바꾸면 이전 스크립트로 기존 문서도 옮깁니다.
- 서비스 계정 키를 소스에 두지 않고 Admin SDK는 실행 환경의 기본 자격 증명으로 초기화합니다.
- 배포는 대상 프로젝트를 확인하고 함수 이름을 지정해 로또와 달 함수를 갈라 올립니다.

## frontend/lottomate-web

Lottomate의 개인정보처리방침·이용약관 페이지와 관리자 콘솔을 담은 React SPA입니다. Firebase Hosting에 올라가고 Authentication으로 로그인합니다.

### 라우트

| 경로 | 화면 | 로그인 |
|------|------|--------|
| `/` | 홈 (방침·약관·관리자 링크) | 불필요 |
| `/privacy`, `/terms` | 방침·약관 (앱 설정이 이 경로를 참조하므로 변경 금지) | 불필요 |
| `/login` | 관리자 로그인 | 불필요 |
| `/admin/dashboard` | 통계와 수동 알림 발송 | 필요 |
| `/admin/remote-config` | Remote Config 파라미터 관리 | 필요 |
| `/admin/secrets` | Secret Manager 관리 | 필요 |

### 설계 포인트

- **인증 게이트**: `AuthProvider`가 로그인 상태 리스너를 걸고, `PrivateRoute`가 `/admin` 부모 라우트를 감싸 비로그인 사용자를 로그인으로 보냅니다.
  게이트는 화면 진입 조건일 뿐이며 실제 권한은 백엔드의 관리자 검사가 403으로 막습니다.
- **백엔드 호출**: 관리자 화면의 모든 요청은 `adminFetch`를 거쳐 Firebase ID 토큰을 Bearer 헤더에 싣습니다.
  `fetch` 직접 호출은 토큰이 빠지므로 금지합니다.
- **관리자 화면 구성**: 사이드바 레이아웃 아래 대시보드(최소 앱 버전, 최신 회차, 마지막 업데이트 카드와 수동 알림 버튼),
  Remote Config 목록·추가·수정·삭제, Secret 목록·조회·생성·삭제로 이루어집니다.
- Remote Config 값은 웹이 형식을 검사하지 않으므로 앱이 읽는 형식 그대로 입력해야 합니다.
- 빌드 산출물은 `dist`이며, 이 경로는 Hosting 설정과 배포 스크립트가 기대하므로 바꾸지 않습니다.
- 소스에 Secret 값이나 서비스 계정 키를 두지 않습니다.

## 모듈 간 계약

| 변경 대상 | 같이 고쳐야 하는 곳 |
|-----------|------------------|
| 함수 이름 | 웹 관리자 화면의 호출 경로 |
| FCM 토픽, data 키 | Android 앱의 메시지 핸들러, 클라이언트 설정 |
| 달 분류 임계값 | 앱의 달 이벤트 감지기 |
| Firestore 필드 | 앱과 웹의 소비처 |
| 웹 `/privacy`, `/terms` 경로 | 앱 클라이언트 설정의 방침·약관 URL |

앱, 백엔드, 웹이 한 저장소에 있어 이런 계약 변경을 한 커밋에서 맞출 수 있다는 점이 모노레포의 이점입니다.

## 배포

- Firebase Functions와 Hosting은 배포 스크립트(전체, Functions만, Hosting만)와 GitHub Actions 수동 워크플로로 올립니다.
- 웹 빌드 후 `dist`를 Hosting으로 올리며, 모든 경로를 `index.html`로 rewrite합니다.
- 상세 명령은 [빌드와 도구](build-and-tooling.md)를 참고하세요.
