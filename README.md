# omt-console

[oh my teams](https://github.com/inho-team/oh-my-teams)(OMT)를 대화형 화면으로 다루는 데스크톱·모바일 앱입니다. 사용자는 채팅 창에서 이사(Director)와 대화하고, 앱은 Orca로 PM과 worker 터미널을 열어 메시지를 전달하며 kickoff가 멈추지 않게 감독합니다.

> 상태: 설계 단계. 설계 문서는 `docs/` 아래에 작성합니다.

## 목표

- **대화 중심 화면:** 왼쪽 대화 목록, 가운데 대화, 아래 입력창으로 이뤄진 채팅형 화면에서 이사와 이야기합니다.
- **정지 위험 최소화:** 종료된 PM 터미널 자동 재시작, 결정 요청 알림과 즉시 승인, 할당량 소진 시 계정 홈 전환, 여유 메모리에 따른 작업 순서 조정을 앱이 맡습니다.
- **데스크톱이 본체, 모바일은 원격:** 데스크톱 앱이 Orca와 OMT를 직접 다루고, 모바일 앱은 같은 화면으로 본체에 접속합니다. 화면 코드는 OTA로 갱신합니다.

## 기술 조합

| 구성 | 기술 |
|---|---|
| 화면 | React |
| 데스크톱 | Tauri |
| 모바일 | Capacitor, JS 번들 OTA |
| 연결 | Tailscale + 토큰 인증 |
| OMT 연동 | `teams-org.mjs`의 JSON 명령과 Orca CLI |

## 설계 문서

- [아키텍처](docs/architecture.md)
- [감독과 정지 위험 대응](docs/supervision.md)
- [계정 격리 실행기 설계](docs/account-isolation.md)
- [로드맵](docs/roadmap.md)
- [위험과 제약](docs/risks.md)
- [OMT 연동 계약](docs/omt-contract.md)
- [디자인 시스템](docs/design-system.md)

**기술 선택 기록 (ADR):**
- [0001 React](docs/decisions/0001-react.md)
- [0002 Tauri](docs/decisions/0002-tauri.md)
- [0003 Capacitor와 JS 번들 OTA](docs/decisions/0003-capacitor-js-ota.md)
- [0004 Tailscale과 토큰 인증](docs/decisions/0004-tailscale-token-auth.md)
- [0005 이사 세션 실행 방식](docs/decisions/0005-director-session.md)
- [0006 계정 격리 방식](docs/decisions/0006-account-isolation.md)

## 라이선스

MIT
