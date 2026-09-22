# Orca 대체 설계

이 문서는 OMT 런타임이 의존하는 Orca 기능을 분리하고 독립 구현체로 교체하는 방안을 확정한다. 터미널 제어, 워크트리, 감독 프로세스 등 핵심 기능을 legacy-workbench의 네이티브 구현과 OMT의 headless 런타임으로 온전히 대체한다. 이를 통해 메모리 오버헤드와 턴 지연을 해소하고 안정성을 높인다. 전환기에는 실행 계층 어댑터 패턴을 도입하여 기존 운영을 중단 없이 유지한다.

## Orca 기능 대체 방안

[재고 03 문서](03-orca-dependencies.md)의 의존 명령을 대체하는 방법과 재사용 후보, 효율 영향을 정의한다.

| Orca 의존 기능 | 대체 방법 | 재사용 후보 및 근거 위치 | 효율 영향 |
|---|---|---|---|
| **터미널·PTY 수명 관리와 화면 읽기** | GUI 터미널을 생략하고 파이프(`stdio`) 기반 자식 프로세스로 구동하며 턴 로그를 직접 읽는다. | OMT headless 런타임 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:620-650`) | 터미널 렌더링 메모리를 아끼고 읽기 지연을 줄인다(미측정, 측정 계획: 턴당 I/O 지연 측정). |
| **워크트리 생성·계보·회수** | 전역 워크트리 대신 격리된 로컬 클론을 생성하고 완료 시 자동 회수한다. | legacy `internal/workspace` (`legacy-workbench/internal/workspace/clone.go:1-35`), OMT 로컬 어댑터 (`oh-my-teams/plugins/oh-my-teams/scripts/local-adapter.mjs:120-145`) | 워크트리 생성 속도와 파일 시스템 I/O 부담을 줄인다(미측정, 측정 계획: 생성 소요 시간 실측). |
| **감독 worker (Task·Dispatch·생존 확인·완료 신호)** | OMT 자체 워크플로 엔진으로 직접 프로세스를 띄우고 OS 프로세스 트리와 파일 이벤트 락으로 상태를 관찰한다. | OMT 워크플로 (`oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1-50`), headless 런타임 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:430-470`) | 오케스트레이션 API 폴링을 없애 CPU 유휴 비용을 개선한다(미측정). |
| **에이전트 준비 판정 (tui-idle 등)** | 텍스트 배너 매칭 대신 워커의 stdout 완료 마커나 턴 완료 파일 락을 직접 대기한다. | OMT headless 런타임 (`oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:670-710`) | 해상도 의존성에 따른 준비 지연과 실패 재시도 비용을 제거한다(미측정). |
| **내장 브라우저** | Electron 브라우저 대신 호스트의 Chrome에 CDP로 직접 연결해 DOM(AXTree)을 추출한다. | legacy `internal/browser` (`legacy-workbench/internal/browser/cdp.go:1-40`) | 브라우저 렌더러 중복 구동을 방지하여 추가 메모리 사용을 막는다(미측정). |
| **런타임 호환성 검증 (`version`, `status`)** | 데몬 상태 폴링을 생략하고 프로바이더 CLI 바이너리만 직접 검증한다. | OMT headless 의존성 점검 경로 | 불필요한 상태 확인 RPC 지연을 0초로 단축한다. |

## 교체 가능한 실행 계층 어댑터 인터페이스

OMT 2.x 전환기에 기존 Orca 의존성을 단계적으로 걷어내기 위해 `RunnerAdapter` 인터페이스로 런타임을 추상화하고 Orca 어댑터와 workbench 어댑터를 교체 가능하게 만든다.

- `spawnWorker(task, workspace)`
  - 입력: 작업 정의, 워크트리 경로.
  - 출력: 워커 프로세스 ID.
  - 실패 의미: 실행 환경(도커/바이너리) 구성 실패 또는 할당량 초과.
- `sendTurn(workerId, input)`
  - 입력: 워커 ID, 입력 문자열.
  - 출력: 전송 성공 여부(boolean).
  - 실패 의미: 프로세스 응답 없음 또는 입력 스트림 닫힘.
- `waitTurnReady(workerId, timeout)`
  - 입력: 워커 ID, 대기 시간 제한.
  - 출력: 턴 완료 상태.
  - 실패 의미: 에이전트 응답 타임아웃 또는 충돌 종료.
- `readOutput(workerId)`
  - 입력: 워커 ID.
  - 출력: 현재 턴의 표준 출력 및 상태 로그.
  - 실패 의미: 로그 파일 소실.
- `stopWorker(workerId, force)`
  - 입력: 워커 ID, 강제 종료 여부.
  - 출력: 성공 여부.
  - 실패 의미: 고아 프로세스 격리 실패.

이 구조를 통해 전환 기간 중에도 OMT 2.x는 기존의 `OrcaAdapter`로 운영을 이어갈 수 있다.

## Orca 어댑터 제거 조건

Orca 어댑터를 완전히 걷어내기 위한 검증 가능한 조건은 다음과 같다.

1. **독립 구동 완수**: `WorkbenchAdapter`만으로 PM부터 Junior까지 모든 역할이 포함된 워크플로가 에러 없이 끝까지 동작해야 한다.
2. **보안 관문 무결성**: `WorkbenchAdapter` 상의 워커가 워크트리 외부의 호스트 자원에 접근하려 할 때 legacy의 `internal/policy` 셸 브리지가 100% 차단함을 검사 명령이 통과해야 한다.
3. **자원 회수 누수 제로**: 워커 종료 시 고아 프로세스와 임시 워크트리가 호스트에 남지 않고 완전히 삭제됨을 실측으로 증명해야 한다.
