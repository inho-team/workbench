# Orca 의존 대체 설계

이 문서는 OMT 2.x가 의존하는 Orca 기능을 분리하고 독립 구현체로 교체하는 방안을 규정한다. 실행 환경, 워크트리, 감독 프로세스 등 핵심 기능은 legacy-workbench의 네이티브 구현과 OMT의 headless 프로세스로 완전히 대체한다. 이를 통해 메모리 오버헤드와 지연 시간을 최소화하고 안정성을 높인다. 전환기에는 실행 계층 어댑터 패턴을 도입하여 기존 운영 중단 없이 이관한다.

## Orca 기능 대체 방안

[참고 03 문서](03-orca-dependencies.md)의 모든 의존 명령을 대체하는 방법은 다음과 같다. 효율이 최우선 기준이며 측정되지 않은 값은 "미측정"으로 둔다.

| Orca 의존 명령 | 대체 방법 | 효율 영향 |
|---|---|---|
| `orca version` | 불필요. CLI 바이너리를 직접 검증한다. | 상태 확인 RPC 지연 0초로 단축 (미측정) |
| `orca terminal list --json` | 에이전트 식별은 OMT가 직접 생성한 프로세스 PID와 상태로 추적한다. | 불필요한 폴링 제거 (미측정) |
| `orca terminal read --screen` | **대화형 TUI 역할 화면 읽기 대체:** GUI 렌더링을 거치지 않고 파일 시스템의 턴 로그를 직접 읽는다. | 렌더링 메모리 절약 (미측정) |
| `orca orchestration worker-start` | **대화형 TUI 역할 PTY 수명 관리:** PTY 할당 없이 `stdio` 기반 자식 프로세스로 구동 및 수명 관리.<br>**일반:** 전역 워크트리 클론 대신 격리된 로컬 워크트리를 생성한다. | 파일 I/O 및 GUI 렌더링 생략 (미측정) |
| `orca orchestration worker-stop` | 감독 worker 종료: OS 프로세스를 직접 종료한다. | RPC 지연 회피 (미측정) |
| `orca orchestration worker-abandon` | 비정상 worker 격리: 프로세스 강제 종료 및 격리 처리한다. | RPC 지연 회피 (미측정) |
| `orca orchestration worker-release` | worker 자원 해제: OMT 워크플로가 종료 즉시 로컬 워크트리를 삭제한다. | 워크트리 누수 방지 (미측정) |
| `orca orchestration check` | 진행/완료 신호 수신: 프로세스 상태 및 파일 이벤트를 감시하여 `worker_done`을 파싱한다. | API 폴링 최소화로 CPU 점유 개선 (미측정) |
| `orca orchestration worker-read` | 턴 출력 읽기: 턴 로그 파일에서 직접 읽는다. | IPC 지연 감소 (미측정) |
| `orca orchestration worker-list` | PM 생존 확인: OS 프로세스 Liveness로 판별한다. | 상태 확인 RPC 지연 제거 (미측정) |

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
