# kickoff 브리프: omt-console 설계 문서와 단계별 계획

> 대상 저장소: `inho-team/omt-console`(공개, 기준 커밋 `27abfa0`). 연동 대상 OMT는 `inho-team/oh-my-teams` 2.8.7(`ee01340`)이다.

## 목표

OMT를 채팅형 화면으로 다루는 데스크톱·모바일 앱 omt-console의 설계 문서를 쓴다. 구현은 하지 않는다. 문서는 단계(Phase)로 나눈 계획과 각 단계의 완료 조건을 담아, 다음 kickoff가 Phase 1 구현을 바로 시작할 수 있어야 한다.

## 수용 기준

1. **문서 구성:** `docs/` 아래에 다음이 있다. 파일을 나누는 방식은 PM이 정하되 README에서 모든 문서로 링크한다.
   - `docs/architecture.md`: 전체 구조, 구성 요소, 데이터 흐름, 신뢰 경계.
   - `docs/roadmap.md`: Phase별 범위·완료 조건·선행 조건·예상 크기. 순서대로 진행할 수 있어야 한다.
   - `docs/omt-contract.md`: OMT와의 연동 계약.
   - `docs/decisions/`: 기술 선택 기록(ADR). 한 결정당 한 파일.
   - `docs/design-system.md`: 화면 구성과 디자인 원칙.
2. **구조:** 데스크톱 앱이 본체다. 본체가 Orca CLI로 PM·worker 터미널을 열고, `orca terminal send`·`read`·`wait`로 메시지를 전달하며, OMT의 `director-signal`·`director-inbox`·`director-reply`·`director-watch`·`resource-acquire`로 kickoff를 감독한다. 모바일 앱은 같은 화면으로 본체에 원격 접속하며 Orca를 직접 다루지 않는다. 채팅 상대는 이사(Director)이며, 이사 세션은 OMT headless 런타임(`claude -p --resume`, 스트림 JSON)으로 감싸는 방안을 검토한다.
3. **정지 위험 대응 설계:** 다음 각 위험에 대해 감지 방법, 자동 대응, 사용자에게 알리는 방식, 실패 시 대체 동작을 적는다. 모두 이전 OMT 운영에서 실제로 일어난 일이다.
   - PM·worker 터미널이 메모리 부족으로 종료됨 → 같은 워크트리에 다시 띄우고 저장된 상태에서 재개.
   - 감시 프로세스가 강제 종료됨 → 감시를 앱 프로세스가 맡아 호스트 도구의 정리 대상이 되지 않게 함.
   - 결정 요청을 몇 시간 놓침 → 신호함 기반 푸시 알림과 알림 안의 승인·거절.
   - 할당량 소진 → 계정별 홈(`CLAUDE_CONFIG_DIR`, `CODEX_HOME` 등) 분리와 전환. OMT의 계정 pool, OpenCodex 계정 격리, handoff(2.8.5~2.8.6)를 먼저 활용하고, 앱이 새로 만들 부분만 설계한다.
   - 메모리 부족 → 여유 메모리에 따른 작업 순서 조정과 자원 슬롯.
4. **OMT 연동 계약:** 앱이 호출하는 `teams-org.mjs` 명령과 Orca CLI 명령을 목록으로 적고, 각 명령의 입력과 JSON 출력 형태를 OMT 2.8.7의 실제 출력으로 확인해 예시를 붙인다. 확인하지 못한 것은 "미확인"으로 표시한다. OMT 버전이 바뀌어도 앱이 깨지지 않도록 계약 버전 확인 방법을 정한다. OMT에 새 명령이나 출력 필드가 필요하면 "OMT에 요청할 변경" 절에 모은다.
5. **기술 선택 기록(ADR):** 다음을 각각 결정·대안·근거·결과로 기록한다. 이 조합은 사용자가 확정했으므로 대안 비교는 근거를 남기는 목적이다.
   - 화면: React
   - 데스크톱: Tauri(Electron 대비 메모리 사용량. 이 PC는 여유 메모리가 자주 0.5GB 아래로 떨어진다)
   - 모바일: Capacitor와 JS 번들 OTA(OTA 방식과 도구, Apple·Google 정책상 허용 범위)
   - 연결: Tailscale + 토큰 인증(기존 OMT 대시보드와 같은 방식)
   - 이사 세션 실행 방식
6. **디자인:** 대화형 AI 앱에서 흔한 배치(왼쪽 대화 목록, 가운데 대화, 아래 입력창)를 따르되, 특정 회사의 로고·이름·아이콘·고유 에셋을 쓰지 않는다는 원칙을 적는다. kickoff 목록, 신호함, 결정 버튼, 여유 메모리·할당량 표시가 이 배치 안 어디에 들어가는지 화면별로 설명한다. 데스크톱과 모바일이 같은 컴포넌트를 쓰는 방식도 적는다.
7. **단계 계획:** 최소한 다음 순서를 포함한다. 각 Phase는 혼자 쓸모가 있어야 하고, 완료 조건이 검증 가능해야 한다.
   - Phase 0: 저장소 기반(빌드·검사·CI, 코드 규칙)
   - Phase 1: 데스크톱 본체(Orca 연동, 터미널 수명 관리와 자동 재시작, 신호함, 결정 버튼, 이사와 채팅)
   - Phase 2: 감독 강화(할당량·메모리·자원 슬롯 표시와 자동 조정, PR·CI 상태)
   - Phase 3: 모바일 원격(같은 화면, Tailscale 접속, 푸시 알림, OTA)
   - Phase 4: 배포(코드 서명, 자동 업데이트, 스토어)
8. **위험과 제약:** 계정 여러 개 전환의 약관 문제(업무·개인처럼 성격이 다른 계정 사이의 전환으로 한정), 원격 접속의 보안(토큰 저장, 권한 범위, 폰 분실), 스토어 OTA 정책, Windows·macOS 차이를 적는다.
9. **품질:** 문서는 두괄식으로 쓰고(OMT `references/bluf.md`의 원칙), 한국어 문서는 fluent-korean 기준을 따른다. 저장소 안의 상대 링크가 모두 유효하다. Senior 검토에서 승인된다.

## 비목표

- 앱 코드, 빌드 설정, CI 워크플로를 만들지 않는다. Phase 0부터는 다음 kickoff에서 한다.
- OMT 저장소를 고치지 않는다. 필요한 변경은 「OMT에 요청할 변경」 절에 적는다.
- mobius에 의존하지 않는다. 계정 전환 아이디어의 참고로만 쓴다(`github.com/chussum/mobius`, macOS 전용 Swift 앱, MIT).

## 제약

- 한국어 문서는 fluent-korean 기준을 적용한다. 형식이 정해진 출력(JSON 예시, 명령)은 원문을 유지한다.
- 이 PC의 여유 메모리가 넉넉하지 않다. 같은 PC에서 OMT 저장소의 kickoff 두 개(`bluf-all-roles`, `omt-audit-r2`)가 동시에 돌고 있다. worker는 한 번에 하나만 띄우고, OMT 명령 출력을 확인할 때도 무거운 작업을 피한다.
- 브랜치는 `docs/design`을 사용한다.

## 전달 방식

- `delivery`: `pull-request`, base 브랜치 `main`. 사용자는 kickoff가 완료되는 대로 이사가 `main`에 병합하도록 승인했다.

## 근거 위치

- OMT 저장소: `C:/Users/kjsun/orca/oh-my-teams`(2.8.7). 이사 명령은 `plugins/oh-my-teams/skills/director/SKILL.md`, 런타임 계약은 `plugins/oh-my-teams/references/orca-runtime.md`, headless 런타임은 `docs/plan/headless-runtime.md`, 대시보드 인증 방식은 같은 문서의 「대시보드」 절, 전수 조사 보고서는 `docs/plan/omt-audit-2026-09.md`에 있다.
- 설치된 OMT 런타임: `C:/Users/kjsun/.claude/plugins/cache/oh-my-teams/oh-my-teams/2.8.7/scripts/teams-org.mjs`.
- Orca CLI 가이드: `orca skills get orca-cli`, `orca skills get orchestration`.
- mobius 참고: 한도 감지는 로컬 세션 로그의 새 줄만 15초마다 읽고, 우선 계정 → 대체 계정 → 초기화 뒤 복귀 순서로 전환한다. 자격 증명은 동기화하지 않는다.
- 사용자 요청 원문: "omt를 좀 더 쉽게 다룰 수 있고 … chatgpt처럼 생긴 툴" / "orca cli를 적극 활용해서 각 터미널을 열어 그 안에서 메세지를 전달하면서 돌 수 있게끔 해 정지 위험을 좀 최소화 … 홈을 좀 다르게 … 데스크탑앱, 모바일도 OTA" / "Phase를 만들어서 순서별로"
