# omt-console

[oh my teams](https://github.com/inho-team/oh-my-teams)(OMT)를 대화형 화면으로 다루는 데스크톱·모바일 앱입니다. 사용자는 채팅 창에서 이사(Director)와 대화하고, 앱은 Orca로 PM과 worker 터미널을 열어 메시지를 전달하며 kickoff가 멈추지 않게 감독합니다.

> 상태: 설계 단계. 설계 문서는 `docs/` 아래에 작성하며, 설계 kickoff의 브리프는 [`docs/briefs/`](docs/briefs/)에 있습니다.

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

## 라이선스

MIT
