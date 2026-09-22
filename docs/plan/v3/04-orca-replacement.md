# Orca 의존 대체 설계

이 문서는 OMT 2.x가 의존하는 Orca 기능을 분리하고 독립 구현체로 교체하는 방안을 규정한다. 실행 환경, 워크트리, 감독 프로세스 등 핵심 기능은 legacy-workbench의 네이티브 구현과 OMT의 headless 프로세스로 완전히 대체한다. 이를 통해 메모리 오버헤드와 지연 시간을 최소화하고 안정성을 높인다. 전환기에는 실행 계층 어댑터 패턴을 도입하여 기존 운영 중단 없이 이관한다.

## Orca 기능 대체 방안

[참고 03 문서](03-orca-dependencies.md)의 두 표(코드가 호출하는 명령, 문서·스킬에만 나오는 명령)에 있는 명령을 대체하는 방법은 다음과 같다. 효율이 최우선 기준이며 측정되지 않은 값은 "미측정"으로 둔다.

| Orca 의존 명령 | 대체 방법 | 재사용 후보 및 근거 위치 | 효율 영향 |
|---|---|---|---|
| `orca version` | CLI 바이너리를 직접 검증한다. | 새로 작성 (구현 없음) | 미측정 |
| `orca skills get orca-cli` | 런타임 가이드 검증 생략. | 새로 작성 (구현 없음) | 미측정 |
| `orca status --json` | 상태 확인 불필요. CLI 실행 가능 여부만 판단. | 새로 작성 (구현 없음) | 미측정 |
| `orca worktree create` | 로컬 작업 디렉터리를 `git worktree add`로 직접 생성. | `local-adapter.mjs`의 로컬 워크트리 생성 (`oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:117-150`) | 미측정 |
| `orca worktree show` | 디렉터리 존재 여부와 `.git` 파일 경로 확인. | `local-adapter.mjs`의 로컬 워크트리 경로 확인 (`oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:200-227`) | 미측정 |
| `orca terminal create` | 비대화형 모드(stdio: ignore)로 백그라운드 프로세스 스폰. | `headless.mjs`의 턴 스폰 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:603-650`) | 미측정 |
| `orca terminal list` | OMT가 직접 생성한 프로세스 PID와 생존 상태 추적. | `headless.mjs`의 백그라운드 워커 조회 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1016-1025`) | 미측정 |
| `orca terminal rename` | 터미널 탭 UI가 없으므로 불필요. 메타데이터 사용. | 새로 작성 (구현 없음) | 미측정 |
| `orca terminal read` | GUI 화면 버퍼 대신 기록된 stream.jsonl, turn.json 파일 파싱. | `headless.mjs`의 턴 기록 읽기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:714-750`) | 미측정 |
| `orca terminal wait` | 정규식 화면 폴링 대신 liveness와 exit.json 생성 대기. | `headless.mjs`의 턴 상태 파일 폴링 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:950-970`) | 미측정 |
| `orca terminal send` | stdin이 아닌, 기존 세션을 이어받는 새 턴 프로세스를 스폰. | `headless.mjs`의 다음 턴 스폰 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:971-994`) | 미측정 |
| `orca terminal close` | stop.request 파일을 기록하여 runner가 리소스 정리. | `headless.mjs`의 프로세스 종료 요청 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:995-1015`) | 미측정 |
| `orca orchestration task-create` | 자체 워크플로 task 상태 객체 인스턴스화. | OMT 자체 task 상태 생성 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:169-183`) | 미측정 |
| `orca orchestration dispatch` | 디스패처가 워커 프로세스 직접 실행. | OMT 자체 디스패치 실행 (`oh-my-teams/plugins/oh-my-teams/scripts/teams-org.mjs:878-947`) | 미측정 |
| `orca orchestration task-update` | 메모리의 로컬 워크플로 객체 상태 갱신. | OMT 워크플로의 task 상태 정산 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1077-1133`) | 미측정 |
| `orca orchestration worker-start` | 워커 프로세스를 로컬 환경에서 직접 스폰. | `headless.mjs`의 워커 시작 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:666-713`) | 미측정 |
| `orca orchestration worker-stop` | 감독 worker 종료: stop.request로 안전하게 종료. | `headless.mjs`의 워커 중지 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:995-1015`) | 미측정 |
| `orca orchestration worker-abandon` | 비정상 worker 격리: 프로세스 트리 강제 종료(taskkill). | `headless-runner.mjs`의 트리 강제 종료 (`oh-my-teams/plugins/oh-my-teams/scripts/headless-runner.mjs:111-135`) | 미측정 |
| `orca orchestration worker-release` | worker 자원 해제: 워크플로 슬롯 해제. | OMT 워크플로 정산 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1169-1238`) | 미측정 |
| `orca orchestration check` | 진행/완료 신호 수신: 프로세스 상태 폴링. | `headless.mjs`의 상태 폴링 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:950-970`) | 미측정 |
| `orca orchestration worker-read` | 턴 출력 읽기: stream.jsonl 파일에서 턴 정보 읽기. | headless의 턴 읽기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:714-750`) | 미측정 |
| `orca orchestration worker-list` | PM 생존 확인. | 새로 작성 (구현 없음) | 미측정 |
| `orca skills get orchestration` | [문서·스킬 전용] OMT 프롬프트 내규로 대체. | 새로 작성 (구현 없음) | 미측정 |
| `orca orchestration run-create` | [문서·스킬 전용] 배정 작업 ID 연동. | 새로 작성 (구현 없음) | 미측정 |
| `orca orchestration send` | [문서·스킬 전용] OMT 통신 인터페이스 직접 호출. | 새로 작성 (구현 없음) | 미측정 |
| `orca orchestration reply` | [문서·스킬 전용] OMT 통신 인터페이스 사용 회신. | 새로 작성 (구현 없음) | 미측정 |
| `orca orchestration ask` | [문서·스킬 전용] 지정 출력 포맷으로 OMT 파서 위임. | 새로 작성 (구현 없음) | 미측정 |
| `orca terminal show` | [문서·스킬 전용] 프로세스 메타데이터 직접 확인 참조. | 새로 작성 (구현 없음) | 미측정 |

## 대화형 TUI 역할의 PTY 수명 관리와 화면 읽기

v3는 대화형 TUI를 OMT headless처럼 완전히 비대화형 모드로 대체하여 PTY 없이 실행하는 것으로 결정한다. GUI 터미널 렌더링과 PTY에 의존하던 기존 Orca의 방식을 버리고, 백그라운드 프로세스를 파일 시스템과 상태 폴링으로 관리한다.

1. **띄우기 (`spawnWorker`)**: TUI 프로세스를 GUI나 PTY 없이 백그라운드로 스폰하되, 통신을 끊기 위해 `stdio: 'ignore'`로 강제 실행한다.
   - **재사용 후보**: `headless.mjs`의 `launchTurn` 스폰 방식 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:603-650`).
2. **입력 보내기 (`sendTurn`)**: 터미널 키스트로크나 stdin 파이프를 통하지 않는다. 이전 세션을 이어받는 새로운 비대화형 턴 프로세스를 다시 스폰하여 입력을 전달한다.
   - **재사용 후보**: `headless.mjs`의 `answerHeadless`를 통한 새 턴 시작 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:971-994`).
3. **화면 및 출력 읽기 (`readOutput`)**: 화면 버퍼나 stdout을 직접 읽지 않는다. 턴이 기록한 파일(`stream.jsonl`, `turn.json`)을 파싱하여 출력을 확인한다.
   - **재사용 후보**: `headless.mjs`의 `readTurn` 파일 읽기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:714-750`).
4. **준비 및 완료 판정 (`checkAgentReady`, `waitCompletion`)**: 화면 유휴 배너 정규식이나 파일 락 이벤트 대기 방식이 아니라, `exit.json` 파일 생성 여부와 프로세스 생존(Liveness)을 폴링(polling)하여 판정한다.
   - **재사용 후보**: `headless.mjs`의 `waitHeadless` 상태 폴링 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:950-970`).
5. **닫기 (`stopWorker`)**: 터미널 탭을 닫거나 SIGTERM을 직접 보내는 대신, `stop.request` 파일을 기록하여 백그라운드 러너가 스스로 종료하도록 유도한다.
   - **재사용 후보**: `headless.mjs`의 `stopHeadless` 종료 요청 파일 생성 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:995-1015`).

### 비대화형 모드로 대체 시 잃는 것
- 대화 중 즉각적인 질문 응답 및 키스트로크 입력 (입력을 주려면 턴이 끝나길 기다렸다가 새 턴으로 이어서 실행해야 함).
- 실시간 화면 관찰 및 진행률 렌더링.
- 터미널 크기나 인터럽트 시그널(Ctrl+C) 등 TUI 전용 상호작용.

이러한 수명 관리 방식은 불필요한 GUI 렌더링과 IPC 오버헤드를 완전히 제거하여 프로세스 자원 효율을 극대화한다.

## 교체 가능한 실행 계층 어댑터 인터페이스

OMT 2.x 전환기에 기존 Orca 의존성을 단계적으로 걷어내기 위해 `RunnerAdapter` 인터페이스로 제어망을 추상화한다. 어댑터 인터페이스 서명은 다음과 같다.

- `createWorktree(task): Workspace` (워크트리 생성)
- `reclaimWorktree(workspace): void` (워크트리 회수)
- `spawnWorker(task, workspace): WorkerId` (프로세스 구동)
- `checkAgentReady(workerId): boolean` (에이전트 준비 판정)
- `checkLiveness(workerId): LivenessState` (생존 확인)
- `sendTurn(workerId, input): boolean` (입력 전송)
- `waitCompletion(workerId, timeout): CompletionSignal` (완료 신호)
- `readOutput(workerId): string` (출력/화면 읽기)
- `connectBrowser(workerId): BrowserSession` (브라우저 연결 연산)
- `stopWorker(workerId, force): boolean` (워커 종료)

## OMT 2.x 전환기 어댑터 운영 방안

OMT 2.x는 전환 기간 동안 기존 방식인 Orca 어댑터로 계속 동작할 수 있어야 한다.

- **어댑터 선택 방식:** 환경 변수(`OMT_ADAPTER`) 또는 워크플로 설정으로 어댑터 구현체(`OrcaAdapter` 또는 `WorkbenchAdapter`)를 동적으로 주입한다.
- **기본값:** 전환기 동안은 `OrcaAdapter`를 기본값으로 유지하여 안정성을 보장한다.
- **되돌리기:** `WorkbenchAdapter` 사용 중 예기치 않은 오류나 검증 실패 시, 환경 변수 변경을 통해 즉시 `OrcaAdapter`로 되돌릴 수 있는 Fallback 경로를 지원한다.

## Orca 어댑터 제거 조건

Orca 어댑터를 완전히 걷어내기 위한 검증 가능한 조건은 다음과 같다.

1. **독립 구동 완수**: `WorkbenchAdapter`만으로 PM부터 Junior까지 모든 하위 역할을 포함한 워크플로가 에러 없이 끝까지 동작해야 한다.
2. **보안 관리 무결성**: 워커 프로세스가 격리된 워크트리 밖의 시스템 자원에 접근하려 할 때, 차단됨을 입증하는 테스트를 통과해야 한다.
3. **자원 회수 누수 제로**: 워커 종료 및 예외 상황 이후 고아 프로세스나 잔류 워크트리가 없음을 자동화된 검증으로 증명해야 한다.
