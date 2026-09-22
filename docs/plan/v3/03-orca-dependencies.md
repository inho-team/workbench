# OMT 런타임의 Orca 의존 전수 목록

기준 커밋: `622ca0e` (참고 저장소 `oh-my-teams` HEAD)

OMT 런타임(`plugins/oh-my-teams/scripts/*.mjs`)은 6개 스크립트(`orca-adapter.mjs`, `role-terminal.mjs`, `teams-org.mjs`, `director.mjs`, `resources.mjs`, `dependencies.mjs`)에서 총 22개 고유 형태의 Orca CLI 하위 명령을 호출한다.
Orca가 사라지면 워크트리 생성 영수증 발급, 대화형 역할 터미널 생성 및 제어, Orca 오케스트레이션 기반의 worker 시작·정산·메시지 수신, PM 생존 감지 등 감독 터미널 기반의 협업 기능 전체가 멈춘다.
다만 OMT는 이미 자체 `local-adapter.mjs`(`git worktree` 기반 작업 공간)와 `headless.mjs`(직접 자식 프로세스 구동 및 턴 제어 런타임) 대체 경로를 구축해 두었으므로, 워크트리 생성 및 에이전트 실행 자체는 Orca 없이도 구동 가능한 대체 구조를 보유하고 있다.

## 탐색 방법 및 재현 패턴

`plugins/oh-my-teams/scripts/*.mjs` 전체를 대상으로 Orca CLI 호출 지점을 전수로 도출하기 위해 정규표현식 기반 grep 탐색과 코드 검증을 수행했다.

1. **어댑터 및 런타임 선택 지점 탐색**:
   - 패턴: `runOrcaJson|selectOrcaExecutable`
   - 결과: `orca-adapter.mjs`, `role-terminal.mjs`, `teams-org.mjs` 3개 모듈에서 Orca JSON 래퍼 호출 식별.
2. **Orca 주요 하위 명령 키워드 탐색**:
   - 패턴: `\b(worktree|terminal|orchestration|skills|status|browser|repo)\b`
   - 결과: `worktree`, `terminal`, `orchestration`, `skills`, `status` 5개 명령군에서 호출 확인. `browser`와 `repo`는 코드 및 문서 어디에서도 호출되거나 참조되지 않음을 전수 확인.
3. **프로세스 실행 함수 탐색**:
   - 패턴: `(\bspawn\b|\brun\b|\bexecute\b)\s*\(\s*\[`
   - 결과: `director.mjs`(`terminal send`, `terminal list`), `resources.mjs`(`orchestration worker-list`), `dependencies.mjs`(`status --json`)의 직접 호출 지점 발굴.
4. **바이너리 이름 탐색**:
   - 패턴: `\borca\b`
   - 결과: 모든 모듈 내 주석, 매개변수 기본값(`orcaExecutable ?? "orca"`), 도움말 문구 대조 완료.

## Orca 명령 호출 전수 목록

| Orca 명령 | 호출 위치(파일:줄) | 기대는 OMT 기능 | Orca가 사라지면 깨지는 것 | 이미 있는 대체 경로 |
|---|---|---|---|---|
| `orca --version` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:193`, `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:64` | Orca 런타임 발견(`discoverOrcaRuntime`) 및 버전 호환성 검증 | Orca 설치 유무 및 버전 확인 실패로 런타임 디스커버리 영수증 발급 불가 | 없음 (단, headless 경로는 Orca 버전을 검사하지 않고 프로바이더 CLI만 직접 실행함: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1-50`) |
| `orca skills get orca-cli` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:199` | Orca 런타임 발견 시 스킬 가이드 해시(`guide hash`) 추출 및 일치 검증 | Orca 런타임 가이드 검증 실패로 `discoverOrcaRuntime` 실패 | 없음 (headless 모드는 가이드 해시를 검증하지 않음: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1-50`) |
| `orca status --json` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:207`, `oh-my-teams/plugins/oh-my-teams/scripts/dependencies.mjs:114` | Orca 데스크톱 및 런타임 데몬 준비 상태(`reachable`, `ready`) 확인, 의존성 사전 점검 | Orca 런타임 준비 상태 판정 불가 및 의존성 점검 실패 | 없음 |
| `orca worktree create --name ... --parent-worktree active --base-branch ... --setup ...` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:276` | 격리된 자식 작업 공간(worktree) 생성 및 작업 디렉터리 할당 | Orca 관리 하의 worktree 생성 및 ID 영수증 발급 불가 | 있음 (`local-adapter.mjs`의 `git worktree add` 기반 로컬 워크트리 생성: `oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:120-145`) |
| `orca worktree show --worktree id:<claim.id>` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:380` | 기존 워크트리 영수증 재확인 및 런타임 인스턴스 일관성 검증(`reobserveOrcaWorkspaceClaim`) | 재접속 시 워크트리 존재 여부 및 런타임 일치 검증 불가 | 있음 (`local-adapter.mjs`의 로컬 워크트리 경로 확인: `oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:150-170`) |
| `orca terminal create --worktree ... --title ... --command ...` | `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:577` | 권한 우회 플래그를 적용한 대화형 에이전트 CLI 터미널 생성(`role-terminal`) | Orca GUI 상에서 역할 전용 대화형 터미널 개설 불가 | 있음 (`headless.mjs`의 백그라운드 프로세스 직접 스폰 방식: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:620-650`) |
| `orca terminal list [--worktree ...]` | `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:399`, `oh-my-teams/plugins/oh-my-teams/scripts/director.mjs:332` | 워크트리 내 터미널 목록 조회(미사용 셸 정리), PM 터미널 생존 확인(`findPmTerminal`) | 고아 터미널 탭 정리 실패, 이사(Director)의 PM 터미널 위치 식별 불가 | 있음 (`headless-list`를 통한 백그라운드 워커 목록 조회: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:520-550`) |
| `orca terminal rename --terminal ... --title ...` | `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:345` | 역할 식별 태그(`[PM]`, `[Senior]` 등)로 터미널 제목 고정(`pinTerminalTitle`) | 터미널 탭 제목 변경 실패 (치명적이지 않음) | 없음 |
| `orca terminal read --terminal ... --screen` | `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:546`, `oh-my-teams/plugins/oh-my-teams/scripts/teams-org.mjs:1348` | 터미널 화면 관찰(에이전트 시작 여부, 폴더 신뢰 질문, 셸 판정), `terminal-idle-check` | 화면 기반 대기/신뢰 질문 감지 및 상태 확인 불가 | 있음 (`headless-status` 및 로그 파일 기반 턴 관찰: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:430-460`) |
| `orca terminal wait --terminal ... --for tui-idle` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:438`, `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:526` | 작업 주입 전 또는 에이전트 구동 후 TUI 유휴(idle) 상태 대기 | TUI 유휴 상태 감지 실패로 작업 주입 불가 또는 조기 실패 | 있음 (`headless.mjs`의 턴 완료 파일 락 대기: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:670-710`) |
| `orca terminal send --terminal ... --text ... --enter` | `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:607`, `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:625`, `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:895`, `oh-my-teams/plugins/oh-my-teams/scripts/director.mjs:297` | 시작 엔터 전송, 신뢰 질문 수락 엔터 전송, `/clear` 명령 전송, 이사 응답 알림(`[omt reply]`) 전송 | 터미널 키스트로크 입력 자동화 불가 | 있음 (`headless-answer`를 통한 stdin/턴 답변 전송: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:480-510`) |
| `orca terminal close --terminal ...` | `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:419`, `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:758` | 미사용 터미널 정리, 신뢰 질문이 잔존한 터미널 재오픈 시 기존 탭 종료 | 터미널 탭 누적으로 인한 호스트 자원 누수 발생 | 있음 (`headless-stop`을 통한 프로세스 종료: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:560-590`) |
| `orca orchestration task-create --spec ...` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:594` | 작업 주입(`injectTask`)을 위한 Orca 오케스트레이션 Task 객체 생성 | Orca 오케스트레이션 task ID 발급 실패 | 있음 (OMT 자체 workflow task 스키마 사용: `oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1-50`) |
| `orca orchestration dispatch --task ...` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:605` | 생성된 Orca Task를 터미널로 디스패치 | Orca 디스패치 인스턴스 생성 불가 | 있음 (OMT 자체 디스패치 및 headless 실행: `oh-my-teams/plugins/oh-my-teams/scripts/teams-org.mjs:1450-1490`) |
| `orca orchestration task-update --id ... --status failed` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:628` | 디스패치 실패 시 생성했던 Orca Task를 실패 상태로 갱신 | Orca 내부 태스크 상태 불일치 | 있음 (OMT 워크플로 저장소의 실패 정산: `oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:800-850`) |
| `orca orchestration worker-start` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:740` | 지정된 터미널/워크트리에서 작업자 시작 및 Run 바인딩 영수증 획득 | Orca 오케스트레이션 기반 감독 worker 구동 전체 마비 | 있음 (`headless-start` 명령: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:620-660`) |
| `orca orchestration worker-stop --dispatch ...` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:815`, `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:838` | 작업 완료가 입증된 worker의 감독 터미널 종료 | 정상 종료된 worker 터미널 회수 불가 | 있음 (`headless-stop`: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:560-590`) |
| `orca orchestration worker-abandon --dispatch ...` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:815`, `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:853` | 프로세스 상태 미확인 worker를 종료로 단정하지 않고 격리(fencing) | 비정상 worker 자원 격리 불가 | 있음 (`headless-runner.mjs`의 프로세스 타임아웃 강제 회수: `oh-my-teams/plugins/oh-my-teams/scripts/headless-runner.mjs:115-130`) |
| `orca orchestration worker-release --dispatch ...` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:815`, `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:868` | 정산이 완료된 worker 자원 공식 해제 | Orca 오케스트레이션 자원 점유 지속 | 있음 (OMT 워크플로 정산 및 워크트리 회수: `oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:900-950`) |
| `orca orchestration check --run ... --wait` | `oh-my-teams/plugins/oh-my-teams/scripts/orca-adapter.mjs:939` | Run 코디네이터 우편함 대기 및 비-하트비트 메시지(`worker_done` 등) 수신 | worker 완료 및 진행 보고 수신 불가 | 있음 (`headless.mjs`의 턴 상태 파일 폴링: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:430-470`) |
| `orca orchestration worker-read --dispatch ...` | `oh-my-teams/plugins/oh-my-teams/scripts/teams-org.mjs:1503` | worker 실행 출력 및 로그 스트림 읽기 | worker 표준 출력 수집 불가 | 있음 (headless의 `turn.log` 직접 읽기: `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:450-470`) |
| `orca orchestration worker-list --json` | `oh-my-teams/plugins/oh-my-teams/scripts/resources.mjs:267` | PM 프로세스 생존 상태(`queryPmLiveness`) 질의 | PM 워크트리의 에이전트 생존 판정 불가 | 없음 |

## Orca 내부 판정 규칙에 기대는 부분

Orca 번들 소스(`out/main/index.js`, `out/shared/shell-process-detection.js`) 및 분석 문서(`docs/plan/agy-terminal-path.md`)에 정의된 판정 규칙에 의존하는 항목이다.

| 판정 영역 | Orca 내부 판정 규칙 및 내용 | 인용 절 및 근거 위치 | OMT에 미치는 영향 및 한계 |
|---|---|---|---|
| `tui-idle` 판정 | 소문자 변환된 화면 버퍼(`e`)에서 `e.lastIndexOf('antigravity cli')` 배너를 찾고, 이어진 줄에서 온전히 `gemini`로 시작하는 모델 줄(`e.startsWith('gemini', o)`)과 길이가 1인 `>` 프롬프트 줄이 모두 존재해야 idle로 판정한다. Claude는 제목(`✳` 접두사)과 상태 훅, Codex는 `openai codex` 배너와 `model:`, `directory:` 줄로 판정한다. | `docs/plan/agy-terminal-path.md` 2절 (`terminal wait --for tui-idle` 판정: `oh-my-teams/docs/plan/agy-terminal-path.md:16-22`), 4.2절/Orca 수정안 (`oh-my-teams/docs/plan/agy-terminal-path.md:281-286`) | Agy에서 `claude`, `gpt-oss` 등 비-Gemini 모델을 사용하면 배너 뒤 모델 줄이 `gemini`로 시작하지 않아 `tui-idle`이 영구히 만족되지 않는다. Windows에서는 화면 폭(44) 조정 시 에이전트 식별과 양립할 수 없어 대화형 주입이 차단된다. |
| 에이전트 식별 (`agentIdentity` / `isShellProcess`) | 전경 프로세스가 셸(`powershell.exe`, `cmd.exe`, `bash`, `zsh` 등)이면 `isShellProcess`가 참이 되어 bare shell로 간주하고 `isterminalrunningagent`가 `false`를 반환한다. 증거 우선순위는 `live-hook` > `process` > `launch` > `completed-hook` > `sleeping-session` > `sibling` > `title` 순이며 충돌 시 `null`로 처리된다. | `docs/plan/agy-terminal-path.md` 1절 (에이전트 식별: `oh-my-teams/docs/plan/agy-terminal-path.md:11-15`), 6절 (`agentIdentity` 결정 규칙: `oh-my-teams/docs/plan/agy-terminal-path.md:55-76`) | PowerShell에서 `mode con: cols=44; agy ...`와 같은 복합 명령을 실행하면 전경 프로세스가 `powershell.exe`로 유지되어 `no_agent_detected` 또는 `agent_unconfigured` 거부가 발생한다. 따라서 OMT는 복합 명령을 사전 차단한다. |
| 생존 판정 (`liveness`) | `isTerminalRunningAgent`를 통해 셸의 자식 프로세스 트리를 탐색해 에이전트 프로세스 생존을 판정한다. 오케스트레이션에서는 `worker-list`의 `projection.liveness.verdict`가 `live`, `exited`, `idle`, `stalled` 중 하나인지 확인한다. | `docs/plan/agy-terminal-path.md` 6.2절 (`isTerminalRunningAgent`: `oh-my-teams/docs/plan/agy-terminal-path.md:63-66`), `resources.mjs` (`oh-my-teams/plugins/oh-my-teams/scripts/resources.mjs:231-236`, `oh-my-teams/plugins/oh-my-teams/scripts/resources.mjs:281-287`) | PM 생존 여부 판단 시 Orca 어휘를 그대로 보존하며, 응답 실패나 미확인 값은 `unverifiable`로 처리하여 프로세스가 살아있을 가능성이 있는 자원을 함부로 회수하지 않는다. |

## 문서·스킬에만 나오고 코드에서는 호출하지 않는 Orca 명령

스킬(`skills/*.md`) 및 가이드 문서(`references/*.md`)에는 안내되어 있으나, OMT 스크립트 코드(`plugins/oh-my-teams/scripts/*.mjs`) 내부에서는 직접 실행하지 않는 명령 목록이다.

| Orca 명령 | 문서/스킬 위치 | 용도 및 서술 맥락 | 코드 미호출 사유 |
|---|---|---|---|
| `orca skills get orchestration` | `oh-my-teams/plugins/oh-my-teams/references/orca-runtime.md:9` | 사용자가 수동으로 오케스트레이션 CLI 가이드를 읽도록 안내하는 문서 서술 | 스크립트 코드는 런타임 발견 시 `orca-cli` 가이드 해시만 조회(`skills get orca-cli`)하도록 구현되어 있다. |
| `orca orchestration run-create` | `oh-my-teams/plugins/oh-my-teams/skills/pm/SKILL.md:19`, `oh-my-teams/plugins/oh-my-teams/skills/pl/SKILL.md:19`, `oh-my-teams/plugins/oh-my-teams/references/kickoff-registry.md:111` | 상위 역할(PM/PL)이 새 작업을 위한 오케스트레이션 Run을 생성하는 지침 | OMT 스크립트는 외부에서 이미 생성되어 전달된 `--run` ID를 바인딩할 뿐 자체적으로 Run을 생성하지 않는다. |
| `orca orchestration send` | `oh-my-teams/plugins/oh-my-teams/skills/junior/SKILL.md:18`, `oh-my-teams/plugins/oh-my-teams/skills/senior/SKILL.md:21`, `oh-my-teams/plugins/oh-my-teams/skills/pl/SKILL.md:37`, `oh-my-teams/plugins/oh-my-teams/references/orca-runtime.md:228` | 하위 역할이 상위 역할에게 질문, 진행 상황, 에스컬레이션(`--type escalation`)을 발송하는 프롬프트 지침 | 에이전트 모델이 프롬프트 지시에 따라 터미널에서 대화형으로 직접 실행하는 사용자 명령이며 스크립트 자동화 대상이 아니다. |
| `orca orchestration reply` | `oh-my-teams/plugins/oh-my-teams/skills/junior/SKILL.md:18`, `oh-my-teams/plugins/oh-my-teams/skills/senior/SKILL.md:21`, `oh-my-teams/plugins/oh-my-teams/references/orca-runtime.md:236` | 상위 역할의 진행 상황 질문에 하위 역할이 회신하는 프롬프트 지침 | 에이전트 모델이 상위 질문에 답변하기 위해 직접 호출하는 상호작용 명령이다. |
| `orca orchestration ask` | `oh-my-teams/plugins/oh-my-teams/skills/junior/SKILL.md:18`, `oh-my-teams/plugins/oh-my-teams/skills/senior/SKILL.md:21` | 작업자가 추가 결정 사항을 배정자에게 질의하는 프롬프트 지침 | 에이전트 모델의 자발적 질의 명령이다. |
| `orca terminal show` | `oh-my-teams/plugins/oh-my-teams/references/orca-runtime.md:185`, `oh-my-teams/docs/plan/agy-terminal-path.md:60` | 터미널 속성 및 `agentIdentity` 표시 상태를 설명하는 문서 상의 예시 | 스크립트 코드는 `terminal list`와 `terminal read --screen`을 사용해 필요한 상태를 직접 확인한다. |
