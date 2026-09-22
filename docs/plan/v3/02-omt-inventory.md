# OMT 기능 재고

기준 커밋: `622ca0e` (참고 저장소 `oh-my-teams` HEAD)

OMT(oh-my-teams)는 다중 에이전트 조직 운영에 필요한 13개 핵심 기능(역할 분담, kickoff 등록부, 워크플로 상태 기계, 게이트 검증 및 증거 보존, 비동기 이사 신호, 자원 슬롯 동시성 제어, headless 런타임, 호환성 판정 매트릭스, 사용 한도 handoff, advise·assist 보조, OpenCodex 프록시 연동, 토큰 사용량 보고서, 웹 대시보드)을 제공한다.
런타임 스크립트는 `plugins/oh-my-teams/scripts/` 최상위 기준 42개 파일 20,400줄(하위 `providers/` 7개 781줄 및 HTML 포함 시 총 21,614줄)이며, `tests/` 디렉터리 내 41개 테스트 파일 23,294줄의 테스트 스위트를 갖추고 있다.
대화형 역할 실행, headless 런타임, OpenCodex 플랫폼 연동, Agy 호환성 표, 사용 한도 handoff 등은 실측 검증 문서로 뒷받침되며, kickoff 수명주기, workflow 상태 전이(handoff 제외), 이사 신호, 게이트와 증거, 대시보드·사용량 보고·자원 슬롯 등은 단위 테스트 위주로 검증되어 있다.

## 기능 표

| 기능 | 하는 일 | 주요 스크립트와 명령 | 코드 줄 수 | 테스트 파일과 줄 수 | 검증 정도 | 근거 위치 |
|---|---|---|---|---|---|---|
| 역할(PM·PL·Senior·Junior, 역할 접기·실행 깊이) | PM(전체 조율/작업 배정), PL(팀 리드/작업 분해), Senior(구현/검토/가이드), Junior(구현/검증)의 계층적 책임을 정의하고, 조직 규모에 따른 역할 접기(`canonicalRole`)와 중첩 실행 깊이(`run-depth`) 제한을 집행한다. | `role-launch.mjs`, `role-terminal.mjs`, `core.mjs`, `teams-org.mjs` (`role-terminal`, `role-dispatch`, `worker-start`) | `role-launch.mjs` (590줄), `role-terminal.mjs` (906줄), `core.mjs` (1,232줄) | `tests/role-dispatch.test.mjs` (941줄), `tests/role-terminal.test.mjs` (1,183줄), `tests/run-depth.test.mjs` (304줄) | 실제 환경 검증 완료 (Claude Code 2.1.274 + Haiku 4.5로 `worker_done` 수신 실측 완료: `oh-my-teams/plugins/oh-my-teams/references/orca-runtime.md:153-154`) | `oh-my-teams/plugins/oh-my-teams/scripts/role-launch.mjs:1-50`, `oh-my-teams/plugins/oh-my-teams/scripts/role-terminal.mjs:1-40`, `oh-my-teams/plugins/oh-my-teams/skills/pm/SKILL.md:1-30`, `oh-my-teams/plugins/oh-my-teams/skills/junior/SKILL.md:1-25` |
| kickoff 등록부 | PM 워크트리와 Run 바인딩을 등록하고 활성 워크트리 목록 조회, 브랜치 정리, 상태 해제 및 종료 준비 검사를 수행한다. | `kickoff-registry.mjs`, `teams-org.mjs` (`kickoff-register`, `kickoff-show`, `kickoff-bind`, `kickoff-release`, `kickoff-branch-cleanup`, `kickoff-check-close-ready`, `kickoff-merge-record`) | `kickoff-registry.mjs` (740줄) | `tests/kickoff-registry.test.mjs` (581줄) | 단위 테스트만 수행 (PM 워크트리 바인딩 및 종료 절차 실측 문서는 있으나 실제 환경 실행 기록 없음) | `oh-my-teams/plugins/oh-my-teams/scripts/kickoff-registry.mjs:1-45`, `oh-my-teams/plugins/oh-my-teams/references/kickoff-registry.md:1-40` |
| workflow(예약·정산·재작업·handoff·깊이) | 상태 기계(reserved, active, completed, rework, handoff) 기반의 작업 생명주기를 관리하고 시도(attempt) 예약/정산, 핸드오프 전환, 중첩 깊이 초과 방지를 집행한다. | `workflow.mjs`, `workflow-store.mjs`, `handoff.mjs`, `handoff-snapshot.mjs`, `teams-org.mjs` (`workflow-reserve`, `workflow-settle`, `workflow-rework`, `workflow-handoff-start`, `workflow-handoff-finish`) | `workflow.mjs` (1,701줄), `workflow-store.mjs` (173줄), `handoff.mjs` (128줄), `handoff-snapshot.mjs` (102줄) | `tests/workflow-safety.test.mjs` (1,072줄), `tests/workflow-recovery.test.mjs` (116줄), `tests/handoff.test.mjs` (213줄), `tests/handoff-transition.test.mjs` (696줄) | 단위 테스트 및 일부 실제 환경 검증 (핸드오프 전환 실측: `oh-my-teams/docs/plan/role-handoff.md:207-226`) | `oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1-60`, `oh-my-teams/plugins/oh-my-teams/scripts/handoff.mjs:1-40` |
| 게이트와 증거(verify·merge-check·review-record·accept) | 작업 계약 검증 스크립트 실행(`verify`), Git 워크트리 청결성 검사(`merge-check`), 상위 검토 기록(`review-record`), 증거 파일 봉인 및 최종 수용(`workflow-accept` / `deliver`)을 수행한다. | `gates.mjs`, `evidence.mjs`, `delivery.mjs`, `contracts.mjs`, `teams-org.mjs` (`verify`, `merge-check`, `review-record`, `workflow-accept`, `deliver`) | `gates.mjs` (465줄), `evidence.mjs` (440줄), `delivery.mjs` (294줄), `contracts.mjs` (310줄) | `tests/boundary-and-gate.test.mjs` (345줄), `tests/delivery.test.mjs` (361줄), `tests/review-format.test.mjs` (86줄) | 단위 테스트만 수행 (안전 검사 및 감사 기록 문서는 있으나 실제 환경 실행 기록 없음) | `oh-my-teams/plugins/oh-my-teams/scripts/gates.mjs:1-40`, `oh-my-teams/plugins/oh-my-teams/scripts/evidence.mjs:1-50` |
| 이사 신호(director-signal·inbox·watch) | 이사(Director)와 PM 간 비동기 신호(decision, close-ready, blocked, progress)를 파일 락 기반 인박스로 교환하고, 활성 PM 터미널로 실시간 알림을 전송하며 신호 대기(`watch`)를 제공한다. | `director.mjs`, `teams-org.mjs` (`director-signal`, `director-inbox`, `director-reply`, `director-ack`, `director-watch`) | `director.mjs` (468줄) | `tests/director-role.test.mjs` (956줄), `tests/director-signal.test.mjs` (917줄) | 단위 테스트만 수행 (이사 신호 프로토콜 실측 문서는 있으나 실제 환경 실행 기록 없음) | `oh-my-teams/plugins/oh-my-teams/scripts/director.mjs:1-50`, `oh-my-teams/plugins/oh-my-teams/skills/director/SKILL.md:1-40` |
| 자원 슬롯(resource-acquire/release) | 동시 테스트/빌드/워커 실행 수를 제한하는 슬롯 락을 획득/반환하고, 메모리 여유량 점검 및 종료된 프로세스의 고아 락을 회수(`resource-gc`)한다. | `resources.mjs`, `teams-org.mjs` (`resource-acquire`, `resource-release`, `resource-list`, `resource-gc`) | `resources.mjs` (342줄) | `tests/lock-recovery-and-release.test.mjs` (406줄) | 단위 테스트만 수행 (파일 락 획득/해제 및 타임아웃 회수 단위 테스트 검증, 실측 문서는 미측정) | `oh-my-teams/plugins/oh-my-teams/scripts/resources.mjs:1-50` |
| headless 런타임 | Orca GUI 터미널 없이 프로바이더(Agy, Claude, Codex, Ollama) CLI를 직접 백그라운드 프로세스로 구동하고 턴(turn) 기반 I/O 및 감독자 응답을 처리한다. | `headless.mjs`, `headless-runner.mjs`, `teams-org.mjs` (`headless-start`, `headless-status`, `headless-answer`, `headless-stop`, `headless-list`) | `headless.mjs` (1,025줄), `headless-runner.mjs` (408줄) | `tests/headless.test.mjs` (1,263줄), `tests/supervision.test.mjs` (455줄) | 실제 환경 검증 완료 (Windows 환경 Agy CLI 1.2.4 headless 구동 실측: `oh-my-teams/docs/plan/headless-runtime.md:112-125`) | `oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1-50`, `oh-my-teams/docs/plan/headless-runtime.md:1-40` |
| 호환성 표(launch-matrix) | 실행기(runner), 모델 계열, OS 플랫폼, 셸, 워크트리 신뢰, 권한 질문 설정의 조합별 실행 가능 경로(`supervised-terminal`, `headless`, `blocked`) 및 사전 차단 사유를 판정한다. | `launch-matrix.mjs` | `launch-matrix.mjs` (393줄) | `tests/launch-matrix.test.mjs` (705줄) | 실제 환경 검증 완료 (Orca 1.4.204 및 Agy CLI 환경 실측 매트릭스 검증: `oh-my-teams/docs/plan/agy-terminal-path.md:277-300`) | `oh-my-teams/plugins/oh-my-teams/scripts/launch-matrix.mjs:1-50`, `oh-my-teams/docs/plan/agy-terminal-path.md:79-110` |
| 사용 한도 handoff | 프로바이더 사용량 한도 초과(429, quota exhaustion)를 감지하고 다른 프로필이나 계정으로 작업 컨텍스트를 이관하는 handoff를 수행한다. | `limit-check.mjs`, `quota.mjs`, `handoff.mjs`, `teams-org.mjs` (`worker-limit-check`, `quota-check`) | `limit-check.mjs` (385줄), `quota.mjs` (196줄) | `tests/limit-check.test.mjs` (516줄), `tests/handoff-transition.test.mjs` (696줄) | 실제 환경 검증 완료 (한도 도달 감지 및 프로필 라우팅 실측: `oh-my-teams/docs/plan/shared-quota-model-routing.md:123-150`) | `oh-my-teams/plugins/oh-my-teams/scripts/limit-check.mjs:1-50`, `oh-my-teams/plugins/oh-my-teams/scripts/quota.mjs:1-40` |
| advise·assist | 작업 시작 전 설계 조언(`advise`)과 작업 중 코드 탐색/보조(`assist`)를 모델에 요청하고, 지시문 대필 방지 및 토큰 한도를 강제한다. | `core.mjs`, `teams-org.mjs` (`advise`, `assist`) | `core.mjs` (1,232줄), `teams-org.mjs` (1,837줄) | `tests/advise.test.mjs` (360줄) | 단위 테스트 및 참조 가이드만 구비 (참조 가이드 `oh-my-teams/plugins/oh-my-teams/references/advise.md:1-40`, `oh-my-teams/plugins/oh-my-teams/references/assist.md:1-30` 구비, 실측 문서는 미측정) | `oh-my-teams/plugins/oh-my-teams/references/advise.md:1-40`, `oh-my-teams/plugins/oh-my-teams/references/assist.md:1-40` |
| OpenCodex | 로컬 OpenCodex 백엔드/프록시 구동 상태를 감지하고 플러그인 설치 확인 및 인증 프록시 라이프사이클을 관리한다. | `opencodex.mjs`, `teams-org.mjs` (`opencodex-probe`, `opencodex-proxy-start`, `opencodex-proxy-stop`) | `opencodex.mjs` (808줄) | `tests/opencodex.test.mjs` (1,173줄) | 실제 환경 검증 완료 (OpenCodex 백엔드 런타임 플랫폼 연동 실측: `oh-my-teams/docs/plan/opencodex-w2-platform-proof.md:12-51`) | `oh-my-teams/plugins/oh-my-teams/scripts/opencodex.mjs:1-50`, `oh-my-teams/docs/OPENCODEX_RUNTIME.md:1-40` |
| 사용량 보고(usage-report) | 세션/역할/프로바이더별 토큰 사용량과 비용 원장을 집계하고 일일/월간 사용량 통계 보고서를 생성한다. | `usage-report.mjs`, `usage-sources.mjs`, `usage-ledger.mjs`, `usage.mjs`, `teams-org.mjs` (`usage-report`, `usage-collect`) | `usage-report.mjs` (799줄), `usage-sources.mjs` (845줄), `usage-ledger.mjs` (190줄), `usage.mjs` (206줄) | `tests/usage-report.test.mjs` (687줄), `tests/usage-sources.test.mjs` (781줄) | 단위 테스트만 수행 (보고서 파싱 및 집계 단위 테스트 검증, 실측 문서는 미측정) | `oh-my-teams/plugins/oh-my-teams/scripts/usage-report.mjs:1-50`, `oh-my-teams/plugins/oh-my-teams/scripts/usage-sources.mjs:1-50` |
| 대시보드 | headless 워커들의 상태, 턴별 로그, 질문/답변 인터페이스를 웹 브라우저에서 관제할 수 있는 로컬 HTTP 서버 및 SSE 스트림을 제공한다. | `dashboard.mjs`, `dashboard.html`, `teams-org.mjs` (`dashboard`) | `dashboard.mjs` (185줄), `dashboard.html` (433줄) | `tests/dashboard.test.mjs` (302줄) | 단위 테스트만 수행 (HTTP 서버 구동 및 대시보드 API 응답 단위 테스트 검증, 실측 문서는 미측정) | `oh-my-teams/plugins/oh-my-teams/scripts/dashboard.mjs:1-50` |

## 스크립트별 규모 표

규모 측정 방법: Node.js의 `fs.readFileSync`로 파일을 읽어 줄바꿈 정규식(`/\r?\n/`)으로 분할한 전체 라인 수를 기준으로 집계했다(`wc -l` 측정값과 동일).

### plugins/oh-my-teams/scripts/*.mjs 파일별 규모

| 파일명 | 코드 줄 수 (`wc -l`) | 주요 역할 |
|---|---|---|
| `adapters.mjs` | 107 | 런타임 어댑터 매핑 및 신호 래퍼 정의 |
| `contracts.mjs` | 310 | 작업 계약(contract) 스키마 파싱 및 유효성 검증 |
| `core.mjs` | 1,232 | 공통 유틸리티, canonicalRole 해석, 프로세스 스폰 헬퍼 |
| `dashboard.mjs` | 185 | Headless 워커 관제용 웹 서버 및 SSE 스트림 제공 |
| `delivery.mjs` | 294 | 산출물 납품 및 최종 보고 증거 수집 |
| `dependencies.mjs` | 461 | 외부 도구(Node, Git, Orca, CLI 등) 의존성 사전 점검 |
| `director.mjs` | 468 | 이사 신호 교환, 인박스 락 관리, PM 터미널 알림 전송 |
| `evidence.mjs` | 440 | 워크트리 변경 지문(fingerprint) 및 검증 증거 봉인 |
| `execution.mjs` | 146 | 실행 컨텍스트 및 포트 어댑터 보조 |
| `failures.mjs` | 148 | 오류 분류 및 정규화된 실패 코드 변환 |
| `gates.mjs` | 465 | 계약 기반 검증(`verify`) 및 워크트리 게이트 통과 검사 |
| `handoff-snapshot.mjs` | 102 | 핸드오프 전환 시점의 Git 상태 스냅샷 저장 |
| `handoff.mjs` | 128 | 역할 간 컨텍스트 이관 및 전환 지점 기록 |
| `headless-runner.mjs` | 408 | Headless 모드 자식 프로세스 생명주기 및 I/O 관리 |
| `headless.mjs` | 1,025 | Headless 감독관, 턴 기반 제어 및 대시보드 상태 관리 |
| `host-defaults.mjs` | 152 | 호스트 환경 기본값 및 프로파일 해석 |
| `incidents.mjs` | 266 | 인시던트 보고서 생성 및 상태 기록 |
| `jev.mjs` | 444 | JEV(Job Execution Verification) 증거 파싱 |
| `kickoff-registry.mjs` | 740 | PM 워크트리 등록부 및 Run 바인딩 관리 |
| `launch-matrix.mjs` | 393 | 실행기·모델·OS 조합별 호환성 및 실행 경로 예측 매트릭스 |
| `lessons.mjs` | 71 | 작업 완료 후 학습 교훈(lessons learned) 수집 |
| `limit-check.mjs` | 385 | 프로바이더 한도 초과(429/quota) 감지 및 방어 |
| `local-adapter.mjs` | 265 | Git 기반 로컬 워크트리 생성 및 관리 어댑터 |
| `opencodex.mjs` | 808 | OpenCodex 로컬 프록시 구동 및 백엔드 연동 |
| `orca-adapter.mjs` | 992 | Orca CLI 호출 및 오케스트레이션 결과 파싱 어댑터 |
| `org-draft.mjs` | 133 | 조직 구성 초안 작성 및 템플릿 처리 |
| `presets.mjs` | 276 | 조직 구성 프리셋 정의 및 로딩 |
| `providers.mjs` | 358 | 프로바이더 공통 인터페이스 및 어댑터 라우팅 |
| `quota.mjs` | 196 | 토큰 쿼터 스냅샷 및 잔여량 추적 |
| `resources.mjs` | 342 | 동시 실행 제한 슬롯(자원 락) 관리 및 GC |
| `role-launch.mjs` | 590 | 역할별 실행 인자 구성 및 프로파일 해석 |
| `role-terminal.mjs` | 906 | Orca 터미널 생성, 화면 판정, 신뢰 질문 대응 |
| `status.mjs` | 277 | 조직 및 작업 진행 상태 종합 요약 |
| `teams-org.mjs` | 1,837 | CLI 엔트리포인트 및 하위 명령 디스패처 |
| `usage-ledger.mjs` | 190 | 토큰 및 비용 원장 로깅 |
| `usage-report.mjs` | 799 | 토큰 사용량 집계 및 보고서 생성 |
| `usage-sources.mjs` | 845 | 프로바이더별 사용량 데이터 파싱 및 수집 |
| `usage.mjs` | 206 | 사용량 집계 헬퍼 |
| `worker.mjs` | 942 | 작업자 프로세스 제어 및 환경 설정 |
| `workflow-store.mjs` | 173 | 워크플로 상태 파일 저장 및 동기화 |
| `workflow.mjs` | 1,701 | 워크플로 상태 기계 및 작업 생명주기 제어 |
| `workspace.mjs` | 194 | 워크스페이스 경로 및 Git 공통 디렉터리 해석 |
| **최상위 42개 스크립트 합계** | **20,400** | **`plugins/oh-my-teams/scripts/*.mjs` 전체** |

참고: 하위 디렉터리인 `plugins/oh-my-teams/scripts/providers/` 내 7개 모듈(781줄)과 `dashboard.html`(433줄)을 포함한 전체 `scripts/` 디렉터리의 규모는 총 21,614줄이다.

### tests/ 테스트 파일 규모 합계

`tests/` 디렉터리 내 41개 테스트 파일의 전체 라인 수 합계는 **23,294줄**이다.
주요 테스트 스위트로는 `runtime.test.mjs`(2,493줄), `headless.test.mjs`(1,263줄), `skill-instructions.test.mjs`(1,251줄), `role-terminal.test.mjs`(1,183줄), `opencodex.test.mjs`(1,173줄), `workflow-safety.test.mjs`(1,072줄), `director-role.test.mjs`(956줄), `role-dispatch.test.mjs`(941줄), `director-signal.test.mjs`(917줄) 등이 있다.
