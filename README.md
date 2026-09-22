# workbench

결론: 이 프로젝트는 OMT를 위한 React/Tauri 기반의 대화형 다중 에이전트 데스크톱/모바일 애플리케이션이다. 현재 계획 단계에 있으며, 상세 계획 문서는 [docs/plan/README.md](docs/plan/README.md)에서 확인할 수 있다.

[oh my teams](https://github.com/inho-team/oh-my-teams)(OMT)를 다시 짠 새 Orca CLI 의존성 없는 대화형 데스크톱·모바일 앱입니다. 사용자는 채팅 창에 이사(Director)로 대화하며, 뒤에서 동작하는 백그라운드 프로세스의 PM과 worker 조직에 작업을 지시하고 결과를 받습니다.

> 주의: 현재 계획 단계이다. 최신 계획 문서는 [docs/plan/README.md](docs/plan/README.md)에 작성하며, 각 kickoff의 요구사항은 [docs/briefs/](docs/briefs/)에 확정한다.

## 주요 목표

- **대화 중심 화면:** 새 대화 시작, 이전 대화, 아래 입력창이 있는 채팅 화면에서 이사와 이야기한다.
- **백그라운드 최소화:** 구 PM 터미널과 노드 에이전트, 타사 클라우드 연동 등 무거운 데몬을 모두 걷어내고, 램 메모리와 백그라운드 작업을 최소화한 단일 프로세스로 실행한다.
- **데스크톱 우선, 모바일 대비:** 데스크톱 기반 OMT 단독 실행을 최우선 목표로 삼으며, 모바일 접근을 위해 웹 화면의 반응형을 우선 지원한다. 화면 구성은 향후 모바일 OTA를 대비한다.

## 기술 스택

| 영역 | 기술 |
|---|---|
| 화면 | React |
| 데스크톱 | Tauri |
| 모바일 | Capacitor, JS 기반 OTA |
| 원격 | Tailscale + 토큰 인증 |
| OMT 실행 | Node.js 기반 백그라운드 프로세스 및 SQLite 연동 |

## 라이선스

MIT
