# yslee-dev-notes

비공개로 운영하는 개인 프로젝트를 공개용으로 정리한 문서 모음입니다.
소스 코드는 공개하지 않으며, 구조, 설계 결정, 겪은 문제와 해결 과정만 다룹니다.

## 프로젝트

| 디렉터리 | 내용 |
|----------|------|
| [yunseong-apps](yunseong-apps/README.md) | Android 앱 6개와 Firebase 백엔드, 웹을 한 저장소로 묶은 모노레포의 구조와 설계 |
| [homeserver](homeserver/README.md) | 집에서 운영하는 서버: 부팅 자동화, 리버스 프록시, 인증서 자동 갱신 |
| [mattermost](mattermost/README.md) | 셀프호스팅 Mattermost 스택을 git 하나로 묶고 스크립트로 복원하기 |
| [dev-tools](dev-tools/README.md) | 개발 보조 도구: 로컬 LLM 전환, 작업 위임 스크립트 |
| [ci-cd](ci-cd/README.md) | Jenkins 파이프라인과 빌드 실패 분석 자동화 |

## 공개 원칙

도메인, IP, 계정명, 토큰, 비밀번호, 키스토어, 내부 경로, 비공개 저장소 주소는 쓰지 않습니다.
예시가 필요하면 `example.com`, `<TOKEN>` 같은 가짜 값을 씁니다.
