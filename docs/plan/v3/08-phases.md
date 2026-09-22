# 단계 계획과 OMT 버전 이정표

결론: Phase 1에서 실행 계층 어댑터 도입으로 OMT 2.x 기반을 마련하고, Phase 2에서 Orca 없이 구동되는 첫 워크플로를 실현한다. 마지막 Phase 3에서 OMT 3.0 전환과 함께 Orca 의존성을 완전히 제거한다.

## Phase 1: 실행 계층 어댑터 도입 (OMT 2.x)
- 범위: OrcaAdapter와 WorkbenchAdapter 인터페이스 설계 및 OMT 내 어댑터 패턴 적용. 이식/재작성 대상 기능 중 `internal/api`, `internal/config`, `internal/policy`를 구현한다.
- 완료 조건: 기존 OMT 설정에서 OrcaAdapter를 거쳐 모든 킥오프 기능이 정상 동작하고 관련 테스트가 통과한다.
- 선행 조건: 선행 조건은 없다.
- 예상 크기: S (인터페이스 분리와 기존 코드 래핑 위주로, 변경 파일 수 10개 미만 수준이다).
- 담당 저장소: oh-my-teams
- 전환 중 운영 보장: 기존 OrcaAdapter를 기본값으로 유지하여 현재 운영 중인 킥오프 프로세스에 장애를 주지 않는다.
- 작업 단위:
  1. `adapters.mjs`에 `RunnerAdapter` 인터페이스와 완료 조건을 정의한다.
  2. `orca-adapter.mjs`에 `OrcaAdapter` 클래스를 작성하고 기존 Orca CLI 명령을 캡슐화하여 단위 테스트를 통과하게 한다.
  3. `workbench-adapter.mjs`에 `WorkbenchAdapter` 모의(Mock) 클래스를 작성하고 기본 반환값을 설정한다.

## Phase 2: Orca 없는 첫 킥오프 완주 (OMT 2.x)
- 범위: WorkbenchAdapter 본체 구현, SQLite 연동, React/Tauri 데스크톱 껍데기 구축. 이식/재작성 대상 기능 중 `internal/store`, `migrations/`, `internal/automation`, `internal/schedule`, `web/`을 구현한다.
- 완료 조건: WorkbenchAdapter 활성화 시 Orca 바이너리 개입 없이 PM부터 Junior까지의 전체 워크플로가 에러 없이 끝까지 동작한다.
- 선행 조건: Phase 1 인터페이스 어댑터 구조 안정화가 완료되어야 한다.
- 예상 크기: L (본격적인 DB 및 백그라운드 런타임 제어로, 신규 모듈 5개 이상 및 DB 마이그레이션을 포함한다).
- 담당 저장소: workbench
- 전환 중 운영 보장: 신규 킥오프만 WorkbenchAdapter를 사용하도록 플래그를 두며, 기존 킥오프는 영향을 받지 않는다.
- 효율 목표 달성: 이 단계에서 '유휴 메모리 100MB 이하', '에이전트당 추가 메모리 50MB 이하', '자원 부족 멈춤 0회' 측정 및 달성을 확인한다 ([06-efficiency-budget.md](06-efficiency-budget.md)).
- 위험 대응: `docs/supervision.md`의 놓친 결정 요청, 메모리·할당량 고갈 시 대응은 이 단계의 백그라운드 런타임 제어에서 모니터링 기능으로 반영한다. 다중 계정의 서비스 약관 위반 위험은 UI 경고 노출로 다룬다. Windows와 macOS 환경 차이 위험은 크로스 플랫폼 경로 정규화로 다룬다. 컨테이너 런타임 라이선스와 메모리 비용 위험은 [05-target-architecture.md](05-target-architecture.md)의 계정 격리 실행기 절 결정을 따른다.

## Phase 3: Orca 어댑터 제거와 OMT 3.0 (OMT 3.0)
- 범위: OMT 코드베이스에서 OrcaAdapter 코드 제거, 모바일 접속 및 Tailscale 인증 릴리스. 이식/재작성 대상 기능 중 `internal/browser`를 구현한다.
- 완료 조건: OMT 저장소에서 orca-adapter.mjs 및 관련 CLI 로직이 모두 삭제되고 단위 테스트가 통과한다.
- 선행 조건: Phase 2에서 완주한 킥오프 결과물의 무결점이 실측으로 입증되어야 한다.
- 예상 크기: M (OMT 코드 제거 및 모바일 릴리스 위주로, 변경 파일 수 10~20개 수준이다).
- 담당 저장소: oh-my-teams, workbench
- 전환 중 운영 보장: Phase 2의 충분한 검증 후 3.0 릴리스로 전환하여 공백을 방지한다.
- 효율 목표 달성: 이 단계에서 '턴 시작 지연 500ms 이하', '의사 신호 전달 지연 100ms 이하' 달성을 검증한다 ([06-efficiency-budget.md](06-efficiency-budget.md)).
- 위험 대응: 원격 접속 보안 위험(접속 토큰 보호 및 권한)과 모바일 앱 OTA 정책 위험은 [05-target-architecture.md](05-target-architecture.md)의 화면 (구조 확정 사항) 절 결정을 따른다.
