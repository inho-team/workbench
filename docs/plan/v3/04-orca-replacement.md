# Orca 의존 대체 설계

이 문서는 OMT 2.x가 의존하는 Orca 기능을 분리하고 독립 구현체로 교체하는 방안을 규정한다. 실행 환경, 워크트리, 감독 프로세스 등 핵심 기능은 legacy-workbench의 네이티브 구현과 OMT의 headless 프로세스로 완전히 대체한다. 이를 통해 메모리 오버헤드와 지연 시간을 최소화하고 안정성을 높인다. 전환기에는 실행 계층 어댑터 패턴을 도입하여 기존 운영 중단 없이 이관한다.

## Orca 기능 대체 방안

[참고 03 문서](03-orca-dependencies.md)의 두 표(코드가 호출하는 명령, 문서·스킬에만 나오는 명령)에 있는 명령을 대체하는 방법은 다음과 같다. 효율이 최우선 기준이며 측정되지 않은 값은 "미측정"으로 둔다.

| Orca 의존 명령 | 대체 방법 | 재사용 후보 및 근거 위치 | 효율 영향 |
|---|---|---|---|
| `orca version` | CLI 바이너리를 직접 검증한다. | `headless.mjs`의 프로바이더 CLI 직접 실행 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1-50`) | 상태 확인 RPC 지연 0초로 단축 (미측정) |
| `orca skills get orca-cli` | 런타임 가이드 검증 생략. | `headless.mjs`의 가이드 해시 검증 생략 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1-50`) | 해시 추출 오버헤드 제거 (미측정) |
| `orca status --json` | 상태 확인 불필요. CLI 실행 가능 여부만 판단. | 없음 | 사전 점검 RPC 지연 제거 (미측정) |
| `orca worktree create` | 로컬 작업 디렉터리를 `git worktree add`로 직접 생성. | `local-adapter.mjs`의 로컬 워크트리 생성 (`oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:120-145`) | 중앙 데몬 병목 제거 (미측정) |
| `orca worktree show` | 디렉터리 존재 여부와 `.git` 파일 경로 확인. | `local-adapter.mjs`의 로컬 워크트리 경로 확인 (`oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:150-170`) | 상태 조회 IPC 지연 제거 (미측정) |
| `orca terminal create` | PTY 또는 stdio를 연결하여 백그라운드 프로세스 직접 스폰. | `headless.mjs`의 백그라운드 프로세스 스폰 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:620-650`) | GUI 렌더링 및 IPC 오버헤드 제거 (미측정) |
| `orca terminal list` | OMT가 직접 생성한 프로세스 PID와 생존 상태(Liveness) 추적. | `headless-list` 기반 백그라운드 워커 조회 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:520-550`) | 불필요한 전역 터미널 폴링 제거 (미측정) |
| `orca terminal rename` | 터미널 탭 UI가 없으므로 불필요. 로컬 프로세스 메타데이터 사용. | 없음 | 타이틀 갱신 오버헤드 제거 (미측정) |
| `orca terminal read` | GUI 화면 버퍼 대신 턴 로그 파일(`turn.log`) 직접 파싱. | `headless-status` 및 로그 파일 기반 관찰 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:430-460`) | 렌더링 메모리 절약, 화면 파싱 비용 제거 (미측정) |
| `orca terminal wait` | 정규식 화면 폴링 대신 턴 완료 파일 생성/이벤트 대기. | `headless.mjs`의 턴 완료 파일 락 대기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:670-710`) | 화면 폴링 주기 삭제로 CPU 점유 개선 (미측정) |
| `orca terminal send` | stdin 파이프 또는 입력 큐 파일로 명령/엔터 전송. | `headless-answer` 기반 stdin 전송 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:480-510`) | 키스트로크 에뮬레이션 오버헤드 제거 (미측정) |
| `orca terminal close` | 자식 프로세스에 SIGTERM/SIGKILL 전송으로 리소스 정리. | `headless-stop`을 통한 프로세스 종료 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:560-590`) | 호스트 자원 누수 즉시 회수 (미측정) |
| `orca orchestration task-create` | OMT 자체 workflow task 스키마 객체 직접 인스턴스화. | OMT 자체 workflow task 스키마 사용 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1-50`) | 오케스트레이션 IPC 지연 제거 (미측정) |
| `orca orchestration dispatch` | OMT 디스패처가 워커 프로세스 직접 실행. | OMT 자체 디스패치 및 headless 실행 (`oh-my-teams/plugins/oh-my-teams/scripts/teams-org.mjs:1450-1490`) | 중앙 디스패치 큐 대기 시간 제거 (미측정) |
| `orca orchestration task-update` | 메모리의 로컬 워크플로 객체 상태 갱신 및 파일 기록. | OMT 워크플로 저장소의 실패 정산 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:800-850`) | 상태 동기화 IPC 지연 제거 (미측정) |
| `orca orchestration worker-start` | 워커 프로세스를 로컬 환경에서 직접 spawn. | `headless-start` 명령 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:620-660`) | 감독 터미널 생성 비용 및 IPC 지연 제거 (미측정) |
| `orca orchestration worker-stop` | 감독 worker 종료: OS 프로세스를 직접 종료한다. | `headless-stop` (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:560-590`) | RPC 지연 회피 (미측정) |
| `orca orchestration worker-abandon` | 비정상 worker 격리: 프로세스 강제 종료 및 격리 처리한다. | `headless-runner.mjs`의 프로세스 타임아웃 강제 회수 (`oh-my-teams/plugins/oh-my-teams/scripts/headless-runner.mjs:115-130`) | RPC 지연 회피 (미측정) |
| `orca orchestration worker-release` | worker 자원 해제: 종료 즉시 로컬 워크트리 삭제. | OMT 워크플로 정산 및 워크트리 회수 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:900-950`) | 워크트리 누수 방지 (미측정) |
| `orca orchestration check` | 진행/완료 신호 수신: 파일 이벤트를 감시하여 `worker_done` 파싱. | `headless.mjs`의 턴 상태 파일 폴링 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:430-470`) | API 폴링 최소화로 CPU 점유 개선 (미측정) |
| `orca orchestration worker-read` | 턴 출력 읽기: 턴 로그 파일에서 직접 읽는다. | headless의 `turn.log` 직접 읽기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:450-470`) | IPC 지연 감소 (미측정) |
| `orca orchestration worker-list` | PM 생존 확인: OS 프로세스 Liveness 식별. | 없음 | 상태 확인 RPC 지연 제거 (미측정) |
| `orca skills get orchestration` | [문서·스킬 전용] OMT 프롬프트 내규로 대체. | 없음 | 불필요한 도움말 검색 생략 (미측정) |
| `orca orchestration run-create` | [문서·스킬 전용] 에이전트 생성 없이 배정 작업 ID 연동. | 없음 | 런타임 생성 절차 간소화 (미측정) |
| `orca orchestration send` | [문서·스킬 전용] CLI 대신 OMT 통신 인터페이스 직접 호출. | 없음 | CLI 실행 오버헤드 감소 (미측정) |
| `orca orchestration reply` | [문서·스킬 전용] OMT 통신 인터페이스 사용 회신. | 없음 | CLI 파싱 및 IPC 오버헤드 감소 (미측정) |
| `orca orchestration ask` | [문서·스킬 전용] 지정 출력 포맷으로 OMT 파서 위임. | 없음 | CLI 오버헤드 없이 신속 질의 (미측정) |
| `orca terminal show` | [문서·스킬 전용] 프로세스 메타데이터 직접 확인 참조. | 없음 | 정보 조회 IPC 지연 제거 (미측정) |

## 대화형 TUI 역할의 PTY 수명 관리와 화면 읽기

Claude Code나 Codex처럼 자체 대화형 터미널(TUI)을 가지는 역할을 구동하기 위해, GUI 터미널 렌더링에 의존하던 기존 Orca의 방식을 대체하여 백그라운드에서 직접 프로세스를 관리하고 입출력을 제어한다. 이를 어댑터 인터페이스 연산과 대응하면 다음과 같다.

1. **띄우기 (`spawnWorker`)**: TUI 프로세스를 GUI 없이 백그라운드로 스폰하되, `stdio` 기반 통신을 위해 PTY를 우회하여 파이프로 연결한다.
   - **재사용 후보**: `headless.mjs`의 직접 스폰 방식 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:620-650`).
2. **입력 보내기 (`sendTurn`)**: 터미널 키스트로크를 에뮬레이션하지 않고, 프로세스의 `stdin` 파이프로 명령어 문자열과 개행문자를 직접 주입한다.
   - **재사용 후보**: `headless.mjs`의 stdin 전송 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:480-510`).
3. **화면 읽기 (`readOutput`)**: 화면 버퍼 폴링과 정규식 매칭을 버리고, 프로세스가 출력하는 stdout 스트림을 가로채거나 턴 단위 로그 파일(`turn.log`)에서 직접 읽어온다.
   - **재사용 후보**: `headless.mjs`의 로그 파일 직접 읽기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:450-470`).
4. **준비 및 완료 판정 (`checkAgentReady`, `waitCompletion`)**: 화면의 유휴(idle) 배너 정규식을 폴링하는 방식은 비효율적이므로, 턴 완료 알림 파일 생성 이벤트(file lock)를 대기하여 판정한다.
   - **재사용 후보**: `headless.mjs`의 턴 완료 파일 락 대기 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:670-710`).
5. **닫기 (`stopWorker`)**: 터미널 GUI 탭을 닫는 대신 워커 자식 프로세스에 SIGTERM/SIGKILL 시그널을 보내 리소스를 즉시 해제한다.
   - **재사용 후보**: `headless-stop`의 자식 프로세스 종료 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:560-590`).

이러한 수명 관리 방식은 불필요한 GUI 렌더링, 키스트로크 에뮬레이션, IPC 오버헤드를 모두 제거하여 프로세스 자원 효율을 극대화한다.

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
