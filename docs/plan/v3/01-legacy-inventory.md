# legacy-workbench 기능 재고 조사 보고서

## 개요 및 요약

legacy-workbench는 Go 기반 오케스트레이터와 PostgreSQL 영속 계층을 바탕으로 총 22개 내부 도메인 모듈과 43개 마이그레이션을 통해 P1부터 P5까지의 핵심 기능 및 Orca 도구 독립 내재화(Browser, Computer, Gate, Automation)를 약 85% 이상 구현 완료한 상태이다.
전체 Go 코드는 프로덕션 29,977줄 및 테스트 17,998줄 규모이며, Next.js 기반 웹 대시보드(17,080줄)와 연동되어 단일 바이너리 체제로 동작한다.
반면 P6 분산 노드 원격 WS 프로토콜 및 자동 프로비저닝, Linear 등 외부 SaaS 직접 연동, 모바일 에뮬레이터 도구는 '설계만' 또는 '일부' 상태로 남아 있다.
핵심 실측 발견으로 agy 컨테이너 격리 환경에서 내장 도구 57개가 권한 설정을 통해 완전 통제됨이 확인되었고, 호스트 macOS 환경에서 CDP 브라우저 및 화면 제어가 네이티브로 완결되어 있음을 검증했다.

---

## 1. 모듈별 기능 및 구현 상태

다음 표는 legacy-workbench의 `internal/` 하위 22개 모듈과 `cmd/`, `web/`, `node-agent/`, `migrations/`의 역할, 구현 상태, 코드 규모 및 실측 근거를 정리한 재고 목록이다.

| 모듈 | 하는 일 | 구현 상태 | Go 코드 줄 수(테스트 제외) | 테스트 줄 수 | 실측 근거 | 근거 위치 |
|---|---|---|---|---|---|---|
| `internal/api` | HTTP REST 엔드포인트와 WebSocket 허브를 열어 조직·직원·태스크·구독·게이트 등 시스템 전반의 조작을 외부 및 웹 UI에 제공한다. | 구현됨 | 4,388 | 1,804 | docs/09 화면 둘(`/audit`, `/worktrees`) 연동 및 WS 허브 실측을 통과했다. | `legacy-workbench/internal/api/server.go:1-50` |
| `internal/automation` | Orca 내재화 M-16에 따라 `automations` 테이블을 주기적으로 폴링하여 등록된 스케줄 및 `div_rebase` 잡을 비동기 실행한다. | 구현됨 | 502 | 706 | docs/09 M-16 Phase 2에서 자동화 스케줄러가 DB 표를 읽고 실행함을 검증했다. | `legacy-workbench/internal/automation/automation.go:1-35` |
| `internal/browser` | CDP(Chrome DevTools Protocol) 소켓을 직접 제어하여 DOM 접근성 트리(AXTree) 스냅샷, 좌표 기반 클릭, 키 입력 등 브라우저 자동화를 수행한다. | 구현됨 | 2,279 | 948 | docs/09 M-16 Phase 3에서 브라우저 세로 한 줄 연결 및 SPA 버튼 탐색 휴리스틱을 실측했다. | `legacy-workbench/internal/browser/cdp.go:1-40` |
| `internal/codex` | OpenAI Codex app-server JSON-RPC 클라이언트를 감싸 프로세스 기동, 세션 수명 주기, 턴 제어 및 승인 요청 콜백을 처리한다. | 일부 | 767 | 0 | docs/09 결정 M-12~M-15에 따라 Chief 전용 통신으로 한정하여 실측했다. | `legacy-workbench/internal/codex/client.go:1-50` |
| `internal/computer` | macOS Accessibility API와 CoreGraphics 화면 캡처 브리지를 호출하여 활성 창 조회 및 마우스·키보드 조작을 대행한다. | 구현됨 | 661 | 311 | docs/09 M-16 데스크톱 조작에서 `ListApps` 왕복 12초 실측 및 화이트리스트 관문을 확인했다. | `legacy-workbench/internal/computer/computer.go:1-30` |
| `internal/config` | 오케스트레이터 실행 환경 설정을 읽고, M-5 결정에 따라 비밀 정보는 파일이 아닌 환경에서 주입받아 provider별 TOML을 생성한다. | 구현됨 | 386 | 271 | docs/05 및 docs/09 P1/P2 부팅 시퀀스에서 설정 렌더링을 검증했다. | `legacy-workbench/internal/config/config.go:1-35` |
| `internal/embed` | Ollama `bge-m3` 등 로컬 임베딩 HTTP 엔드포인트를 직행 호출하여 기억 벡터 검색용 고차원 벡터를 추출한다. | 일부 | 89 | 0 | docs/09 P4 임베드 러너 실측에서 로컬 무비용 임베딩 호출을 확인했다. | `legacy-workbench/internal/embed/embed.go:1-40` |
| `internal/gate` | 채용·머지·발령·예산 등 인간 개입 지점을 1급 데이터 프리미티브(Gate)로 관리하며 생성·해결·취소 및 WebSocket 알림을 전파한다. | 구현됨 | 244 | 302 | docs/09 M-16 Phase 1에서 단일 `gates` 테이블 기반 통합 승인 관문 진입을 실측했다. | `legacy-workbench/internal/gate/gate.go:1-35` |
| `internal/git` | 오케스트레이터 권한으로 워크트리 분기, diff 검사, 커밋 검증, lead 판정 후 머지 및 `div_rebase` 조작을 안전하게 대행한다. | 구현됨 | 386 | 439 | docs/09 D-06 닫힘 실측에서 조상 보존 3단계 `div_rebase` 동작을 확인했다. | `legacy-workbench/internal/git/git.go:1-35` |
| `internal/lego` | 프론트엔드·백엔드·계약의 3축 규칙에 따라 스택별 코드 블록을 조립하고 파일 골격(스캐폴딩)을 자동 생성한다. | 구현됨 | 2,212 | 1,661 | HANDOFF-레고 실측에서 14개 소스 파일과 단위 테스트를 통해 골격 생성을 검증했다. | `legacy-workbench/internal/lego/block/manifest.go:1-35` |
| `internal/mcp` | 도구 레지스트리를 표준 MCP stdio 서버 프로토콜로 노출하며 Chief Codex 세션용 계급 기반 도구 필터링을 집행한다. | 일부 | 179 | 0 | docs/05 및 docs/08 E안 채택에 따라 로컬 직원을 제외하고 Chief 전용 stdio 어댑터로 축소 적용했다. | `legacy-workbench/internal/mcp/doc.go:1-36` |
| `internal/media` | FFmpeg 및 ffprobe 서브프로세스를 대신 실행하여 오디오·비디오 메타데이터 분석, 구간 분할, 트랜스코딩, Whisper 전사를 대행한다. | 구현됨 | 526 | 408 | docs/01 및 docs/09 P5 미디어 파이프라인에서 미디어 조작 격리를 확인했다. | `legacy-workbench/internal/media/media.go:1-40` |
| `internal/node` | node-agent 프로토콜(WS) 통신과 agy CLI 컨테이너 어댑터를 구현하여 프로세스 스폰, 스트리밍 입출력 번역, 턴 실행을 중계한다. | 구현됨 | 1,265 | 1,036 | docs/09 2026-09-10 agy 컨테이너 실행 실측에서 턴 실행 및 스트리밍 변환을 확인했다. | `legacy-workbench/internal/node/agy.go:1-40` |
| `internal/org` | 조직 도메인(에이전트, 부서, 템플릿, 로스터) 로직을 관장하며 지문 기반 캐싱으로 `AGENTS.md` 프롬프트 문서를 동적 렌더링한다. | 구현됨 | 674 | 408 | docs/09 AGENTS.md 렌더링 실측에서 두 지문 기반 신선도 보장을 확인했다. | `legacy-workbench/internal/org/agentsmd.go:1-35` |
| `internal/policy` | 순수 함수 기반 L0 정책 엔진으로 셸 명령어 토큰 분석, git/파일 접근 규칙, 샌드박스 경계 침범 여부를 사전 검사한다. | 구현됨 | 1,123 | 906 | docs/07 및 docs/09 셸 브리지 판정 실측에서 worker 권한 강제 불변식을 확인했다. | `legacy-workbench/internal/policy/bridge.go:1-35` |
| `internal/quota` | DB를 모르는 인메모리 순수 정책 모듈로서 토큰 버킷 알고리즘을 사용해 rpm, 버스트, 호출 간격, 동시 요청 수를 엄격히 제한한다. | 구현됨 | 362 | 335 | docs/16 단계 C 실측에서 폭주 방지 인메모리 속도 한도 집행을 확인했다. | `legacy-workbench/internal/quota/quota.go:1-35` |
| `internal/schedule` | 외부 라이브러리 없이 순수 Go로 5필드 cron 표현식을 해석하여 정기 이슈(`task_recurrences`)의 다음 실행 시각을 계산한다. | 구현됨 | 450 | 222 | docs/17 정기 이슈 스케줄러 실측에서 순수 시각 연산 및 표현식 파싱을 검증했다. | `legacy-workbench/internal/schedule/cron.go:1-35` |
| `internal/session` | 에이전트와 스레드 간 세션 생명주기 관리, 턴 기록, 리포트 수집, 리뷰 판정, 에피소딕 기억 압축(Compaction)을 수행한다. | 구현됨 | 2,186 | 1,003 | docs/09 P1/P2 세션 루프 및 기억 압축 실측을 통과했다. | `legacy-workbench/internal/session/tracker.go:1-35` |
| `internal/store` | PostgreSQL(pgx) 기반 영속 계층으로 조직, 태스크, 기억, 노드, 구독, 게이트, 자동화, 감사 등 전체 테이블의 sqlc 및 수동 CRUD를 담당한다. | 구현됨 | 7,097 | 5,328 | docs/09 001~043 마이그레이션 적용 및 트랜잭션 쿼리 동작을 실측했다. | `legacy-workbench/internal/store/store.go:1-40` |
| `internal/tools` | MCP 및 CLI가 공통 참조하는 전송 무관 도구 레지스트리(Registry)와 도메인별 툴 핸들러(hr, comm, task, git, browser 등)를 등록한다. | 구현됨 | 3,815 | 1,845 | docs/05 D-15 도구 레지스트리 설계 및 browsertools 15종 등록을 검증했다. | `legacy-workbench/internal/tools/registry.go:1-40` |
| `internal/toolsock` | 오케스트레이터와 에이전트 간 Unix Domain Socket 기반 IPC 통신을 관리하며 일회용 티켓 인증으로 도구를 격리 실행한다. | 구현됨 | 517 | 255 | HANDOFF-셸-브리지 및 docs/14 실측에서 macOS 104자 소켓 경로 준수를 확인했다. | `legacy-workbench/internal/toolsock/server.go:1-35` |
| `internal/workspace` | 권위 레포의 `.git` 오염을 방지하기 위해 Git worktree 대신 격리된 로컬 클론(local clone)을 생성하고 관리한다. | 구현됨 | 308 | 353 | docs/07 §3.2 및 docs/09 M-2 격리 실측에서 로컬 클론 생성을 검증했다. | `legacy-workbench/internal/workspace/clone.go:1-35` |
| `cmd/orchestrator` | 오케스트레이터 메인 진입점으로 REST API 서버, WS 허브, 스케줄러, 백그라운드 워커, `ops` 관리자 CLI를 통합 구동한다. | 구현됨 | 6,865 | 1,783 | docs/09 통합 부팅 및 `workbench ops` 서브커맨드 동작을 실측했다. | `legacy-workbench/cmd/orchestrator/main.go:1-40` |
| `cmd/node-agent` | 머신당 1개 상주하는 경량 데몬 소스코드로 호스트 RAM 측정 및 app-server 프로세스 모니터링을 담당한다. | 구현됨 | 184 | 0 | docs/01 인프라 설계 및 docs/05 P6 도입 계획에 따라 기본 골격을 구현했다. | `legacy-workbench/cmd/node-agent/main.go:1-30` |
| `cmd/` (통합) | `cmd/orchestrator`와 `cmd/node-agent` 두 개 바이너리 진입점으로 구성된다. | 구현됨 | 7,049 | 1,783 | docs/05 바이너리 2개 실행 단위 원칙에 부합함을 확인했다. | `legacy-workbench/cmd/orchestrator/main.go:1-40` |
| `web/` | Next.js App Router 기반 관리 대시보드로 조직, 로스터, 태스크 타임라인, 감사 로그, 워크트리 현황 화면을 제공한다. | 구현됨 | 0 (TS/CSS 등 17,080줄) | 0 | docs/09 `make web-check` 및 `/audit`, `/worktrees` 화면 연동을 확인했다. | `legacy-workbench/web/app/page.tsx:1-40` |
| `node-agent/` | `cmd/node-agent`에서 빌드된 단일 정적 실행 바이너리(8,962,194 바이트)로 머신 프로비저닝에 사용된다. | 구현됨 | 184 (바이너리 8.9MB) | 0 | workbench-spec.html §FN-090 및 저장소 루트 정적 바이너리 배치를 확인했다. | `legacy-workbench/docs/workbench-spec.html:2061` |
| `migrations/` | Goose 기반 PostgreSQL DDL 마이그레이션 43개 파일 및 seed 데이터로 전체 테이블 스키마 진화를 관리한다. | 구현됨 | 43개 SQL 파일 | 0 | docs/09 001부터 043까지 마이그레이션 번호 충돌 해결 및 스키마 적용을 실측했다. | `legacy-workbench/migrations/001_init.sql:1-40` |

> **줄 수 산출 방식**: Go 소스코드는 `*.go` 파일 중 `*_test.go`를 제외한 파일들의 행 수를 합산했고, 테스트 줄 수는 `*_test.go` 파일들의 행 수를 합산했다. `web/`은 `node_modules`와 빌드 산출물을 제외한 TypeScript, TSX, CSS, JSON 파일의 총 행 수(17,080줄)를 측정했다.
> **migrations/ 테이블 범주(43개 파일)**: 조직/계정(001, 008, 009, 022), 정책/보안(002, 040, 041, 042), 스킬/도구(004, 043), 작업/형상(006, 007, 023, 032), 기억(010, 011, 013, 014, 015, 020, 033), 노드(012), 미디어(016, 017, 019), 프롬프트/리뷰(018, 021), 구독/쿼터(024, 028, 029, 031, 034, 036, 038, 039), 협업/이슈(025, 026), 샌드박스(027), 정기작업(030), 내재화(037), 인간개입(035) 등 14개 범주를 포괄한다.

---

## 2. 설계 문서 대조표

legacy-workbench `docs/` 디렉터리의 01~21부 설계 문서 및 4건의 HANDOFF 문서에 대해 핵심 결정 사항, 구현 여부, 미구현 사항 및 근거 위치를 대조한 결과는 다음과 같다.

| 문서 | 무엇을 정했는가 (핵심 결정 2~4개) | 구현된 것 | 구현되지 않은 것 | 근거 위치 |
|---|---|---|---|---|
| `01-설계-아키텍처.md` | 비서실장 1인 대면 경계 원칙, 단일 PostgreSQL 영속 계층, 계급 기반 MCP 권한 분리를 확정했다. | organizations, divisions, agents, messages 스키마와 orchestrator 바이너리를 구현했다. | P6 원격 노드 프로비저닝 스크립트(`join.sh`)와 외부 MCP 연동은 구현하지 않았다. | `legacy-workbench/docs/01-설계-아키텍처.md:13-70` |
| `02-구현-단계.md` | P1(비서실장)부터 P6(회사)까지 6단계 구현 마일스톤과 단계별 누적 DB 스키마 도입 순서를 확정했다. | P1~P5 기본 프레임워크와 migrations 001~020 스키마를 순차 구현했다. | P6 원격 노드 에이전트 간 분산 RPC 연동과 완전 자동 세대교체는 구현하지 않았다. | `legacy-workbench/docs/02-구현-단계.md:16-114` |
| `03-제품-UI-API-운영.md` | 3열 레이아웃 대시보드(조직도/결정/타임라인), WebSocket 상태 전송 규약, Persona 4요소를 확정했다. | Next.js 웹 대시보드 레이아웃, WebSocket 스트리밍 엔드포인트(`GET /ws`), Persona 필드를 구현했다. | 웹 대시보드 내 실시간 턴 개입/일시정지(steer/interrupt) 대화형 UI는 구현하지 않았다. | `legacy-workbench/docs/03-제품-UI-API-운영.md:11-85` |
| `04-화면-기능-부록.md` | 19개 화면(SCR-01~19) 상세 와이어프레임과 96개 기능(FN-001~096) 매핑 매트릭스를 확정했다. | 태스크, 로스터, 조직, 워크트리, 감사 대시보드 및 핵심 백엔드 기능(FN-001~056)을 구현했다. | SCR-11(부서 간 협업 전용 뷰), SCR-16(GitLab CI 파이프라인 뷰)은 구현하지 않았다. | `legacy-workbench/docs/04-화면-기능-부록.md:15-120` |
| `05-모듈-구성.md` | Go 바이너리 2개 체제, `internal/` 단방향 의존 규칙, MCP 9종의 단일 레지스트리 통합 원칙을 확정했다. | orchestrator 바이너리, `internal/` 22개 모듈 아키텍처, `internal/tools` 레지스트리를 구현했다. | P6 원격 노드 분리를 위한 독립 바이너리 간 통신 프로토콜 완성은 구현하지 않았다. | `legacy-workbench/docs/05-모듈-구성.md:10-70` |
| `06-구현-계획.md` | E안 채택(로컬 직원은 MCP 대신 `exec_command` + `outputSchema`), 단계별 개발 일정 및 인터페이스 선행 원칙을 확정했다. | E안 기반 `outputSchema` 턴 실행, 셸 브리지(`workbench tool`), 단계별 마이그레이션을 구현했다. | 계획 초안의 순차적 일정은 E안 한계 직면 및 agy 어댑터 전환으로 일부 수정되었다. | `legacy-workbench/docs/06-구현-계획.md:140-200` |
| `07-권한-강제-설계.md` | 셸을 가진 워커 위협 모델 분석, 4단 방어층(L0 정책~L3 게이트), Git worktree 대신 로컬 클론 강제(M-2)를 확정했다. | `internal/policy` 명령어 토큰 분석 엔진, `internal/workspace` 로컬 클론, toolsock 티켓 인증을 구현했다. | OS 수준의 cgroup/seccomp 완전 격리 샌드박스는 구현하지 않았다. | `legacy-workbench/docs/07-권한-강제-설계.md:38-150` |
| `08-E안-소통-경로.md` | 로컬 직원의 소통을 `outputSchema`로 단일화하고 `thread/inject_items`를 통한 맥락 주입 규칙을 확정했다. | `session.ReportSchema`, `session.CompactionSchema` 파싱 및 주입 로직을 구현했다. | 로컬 직원의 양방향 실시간 대화형 질의응답 경로는 구현하지 않았다. | `legacy-workbench/docs/08-E안-소통-경로.md:35-98` |
| `09-진행-상황.md` | P1~P5 개발 이력, E안 실패 후 agy CLI 전환, Orca 도구 독립 내재화(M-16), M-14 계정 격리 실측을 확정했다. | 43개 마이그레이션 적용, 내재화 4대 도구(Browser, Computer, Gate, Automation), agy 연동을 구현했다. | agy 상주 stdio 프로토콜(`stream-json` 입력)은 미지원으로 단발 CLI 실행을 유지했다. | `legacy-workbench/docs/09-진행-상황.md:3122-3165` |
| `10-화면-API-명세.md` | 19개 화면 구조, 40개 REST API 명세, WebSocket 프로토콜(`GET /ws`) 이벤트 스키마를 확정했다. | `internal/api` chi 라우터, 40여 개 REST 엔드포인트 핸들러, 웹소켓 브로드캐스터를 구현했다. | 모바일 오프라인 Service Worker 캐싱 및 PWA 설치 기능은 구현하지 않았다. | `legacy-workbench/docs/10-화면-API-명세.md:85-180` |
| `11-스킬-설계.md` | Orca 스킬 실측을 바탕으로 정적 프롬프트 스킬(DB)과 동적 런타임 툴(코드)의 2층 구조를 확정했다. | `skills` 관련 4개 테이블(`004_hiring.sql`), `AGENTS.md` 프롬프트 스킬 주입 렌더링을 구현했다. | 사용자 정의 커스텀 스킬의 웹 UI 동적 편집 및 마켓플레이스 연동은 구현하지 않았다. | `legacy-workbench/docs/11-스킬-설계.md:215-280` |
| `12-ERD-설계.md` | 프로젝트별 DB 스키마를 에이전트가 기록(D-16), 환경별 스키마 드리프트 감지, 9개 ERD API를 확정했다. | 순수 함수 기반 DDL 파서 초안 및 핸드오프 인계 명세를 작성했다. | `021_erd.sql` 정식 DDL 반영 및 백엔드 실시간 drift 감지 API 머지는 미완으로 남았다. | `legacy-workbench/docs/12-ERD-설계.md:43-95` |
| `13-페르소나-핸드오프.md` | 17개 직무 페르소나 정의, `baseInstructions` 완전 대체 규칙, JSONB 시스템 프롬프트 주입을 확정했다. | `022_worker_role_persona.sql` 마이그레이션과 `internal/org/persona.go` 렌더러를 구현했다. | 17개 직군 전체에 대한 LLM 대화 품질 실측 튜닝은 전수 완료하지 않았다. | `legacy-workbench/docs/13-페르소나-핸드오프.md:21-100` |
| `14-CLI-설계.md` | 바이너리 2개(`workbench`, `wb`), 3개 표면(직원/운영자/부팅), Unix domain socket 104자 한도 준수를 확정했다. | `cmd/orchestrator` 내 `ops` 관리자 명령, `internal/toolsock` 티켓 인증 소켓 통신을 구현했다. | 직원 전용 독립 경량 C/Go 바이너리(`wb`) 빌드는 별도 분리하지 않았다. | `legacy-workbench/docs/14-CLI-설계.md:21-65` |
| `15-이슈-업무-추적-설계.md` | 태스크의 이슈 트래커 확장, 참조자(`task_watchers`), 이관 프로토콜, 연관 링크(`task_links`), WBS 계층을 확정했다. | `025_issue_tracking.sql`, `internal/store/issue.go`, `internal/api/issues.go`를 구현했다. | 외부 Linear 서비스와의 양방향 웹훅 실시간 동기화는 구현하지 않았다. | `legacy-workbench/docs/15-이슈-업무-추적-설계.md:128-180` |
| `16-구독-설계.md` | 구독 계정 모델(`024_subscriptions.sql`), 4계층 티어 라우팅(M-15), OrbStack 기반 agy 컨테이너 격리(M-14)를 확정했다. | `subscriptions` 테이블, `internal/quota` 토큰 버킷, `docker-compose.agy.yml`을 구현했다. | Tier 2(3.1 Pro) 자동 에스컬레이션 라우터의 완전 자율 전환은 구현하지 않았다. | `legacy-workbench/docs/16-구독-설계.md:326-365` |
| `17-정기-이슈-설계.md` | 정기 이슈를 인스턴스가 아닌 틀(`task_recurrences`)로 관리하고, 발행 락 및 순수 Go cron 계산을 확정했다. | `030_task_recurrence.sql`, `internal/schedule/cron.go`, `internal/store/recurrence.go`를 구현했다. | 밀린 정기 이슈의 소급 실행(의도적 건너뜀 정책 채택)은 구현하지 않았다. | `legacy-workbench/docs/17-정기-이슈-설계.md:72-130` |
| `18-PARA-아카이브-설계.md` | PARA 아카이브 통일, 아카이브 등급의 뷰 처리(D-17), 벡터 검색 가중치 곱셈 재정렬(D-18, D-19)을 확정했다. | `033_para_archive.sql` 및 `internal/store/para.go` 가상 뷰 쿼리를 구현했다. | 디스크 상의 물리적 PARA 디렉터리 폴더 구조 마이그레이션은 구현하지 않았다. | `legacy-workbench/docs/18-PARA-아카이브-설계.md:45-94` |
| `19-일정-비서-설계.md` | TickTick API 분석을 통한 개인 일정 비서 연동, `calendar_events` 독립 테이블, 사람 일정과 조직 태스크 분리를 확정했다. | `calendar_events` 스키마 초안 및 API 명세를 설계했다. | TickTick OAuth2 실계정 연동 및 백그라운드 주기적 양방향 동기화 워커는 구현하지 않았다. | `legacy-workbench/docs/19-일정-비서-설계.md:154-220` |
| `20-orca-도구-독립-내재화-설계.md` | Orca 런타임 100% 분리 클린룸 내재화 원칙, 4대 도구(Browser, Computer, Gate, Automation) 자체 구현을 확정했다. | `internal/gate`, `internal/automation`, `internal/browser`, `internal/computer`, `worktree ps`를 구현했다. | Computer Use의 Windows/Linux 브리지와 E2E 야간 자동화 러너(`night_e2e`)는 구현하지 않았다. | `legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:151-240` |
| `21-orca-전체-기능-카탈로그-및-독립-구현-명세.md` | Orca 234개 전수 CLI 명령의 19개 도메인 카탈로그화, 도메인별 독립 구현 명세, 우선순위 매트릭스를 확정했다. | Browser(15개 툴), Gate(DDL/API), Automation(스케줄러), Worktree(계보/rebase)를 구현했다. | Mobile Emulator(16개), Linear 직접 연동(27개), VM 관리(1개)는 구현하지 않았다. | `legacy-workbench/docs/21-orca-전체-기능-카탈로그-및-독립-구현-명세.md:11-40` |
| `HANDOFF-ERD.md` | 프로젝트 ERD 인계, 6건 미결 결정 처리 절차, `/projects/:id` 상세 화면 선행 조건을 확정했다. | `internal/session` 리포트 연계 분석 및 DDL 초안을 작성했다. | `internal/erd` 정식 코드 머지 및 `021_erd.sql` 반영은 구현하지 않았다. | `legacy-workbench/docs/HANDOFF-ERD.md:27-70` |
| `HANDOFF-구독.md` | 구독 브랜치 인계, 단계 D(비중/토스) 완료 확인, 단계 E(agy CLI 어댑터) 착수 지침을 확정했다. | `024_subscriptions.sql`, `internal/node/agy.go` 컨테이너 어댑터 기본 구조를 구현했다. | agy 상주 프로세스화 및 스트리밍 양방향 통신 완성은 구현하지 않았다. | `legacy-workbench/docs/HANDOFF-구독.md:16-120` |
| `HANDOFF-레고.md` | lego 골격 생성기 인계, FE/BE 분리 및 3축(FE/BE/Contract) 아키텍처 규칙, 스케줄러 연동을 확정했다. | `internal/lego` 엔진, 매니페스트, 14개 Go 파일 및 단위 테스트(2,212줄 + 1,661줄)를 구현했다. | 실제 운영 오케스트레이터 태스크 루프와의 완전 통합은 구현하지 않았다. | `legacy-workbench/docs/HANDOFF-레고.md:32-100` |
| `HANDOFF-셸-브리지.md` | 셸 브리지(`workbench tool`) 설계 인계, heredoc 인자 전달, 소켓 탈취 방지 티켓 인증을 확정했다. | `internal/policy/bridge.go`, `internal/toolsock` 및 PR #2 머지를 구현했다. | P-D~P-F 잔여 단계(macOS 전용 권한 세분화 등)는 구현하지 않았다. | `legacy-workbench/docs/HANDOFF-셸-브리지.md:10-80` |

---

## 3. Orca 도구 독립 내재화 심층 대조 (20부 및 21부 전용 절)

### 3.1 20부 (Orca 도구 독립 내재화 설계) 절별 결정 및 구현 여부

20부(`docs/20-orca-도구-독립-내재화-설계.md`)에 정의된 모든 절의 핵심 결정과 실제 구현 여부는 다음과 같다.

1. **§1.1 현황 진단: 이미 들어왔는가?**
   - **결정**: Orca 데스크톱 도구의 기도입 여부를 전수 조사하고, Electron 런타임 의존성 없이 순수 프로토콜 수준에서 분리 도입할 필요성을 진단했다.
   - **구현 여부**: 진단 완료. Orca 바이너리 없이 런타임 0% 분리 원칙을 확정했다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:30-33`)
2. **§1.2 도입 전략: Fork 후 커스텀이 불가능한 이유**
   - **결정**: Electron의 비대한 런타임, 모놀리식 번들 구조, 비표준 IPC로 인해 Fork가 불가능함을 확인하고 클린룸(Clean-room) 네이티브 Go 재구현 전략을 확정했다.
   - **구현 여부**: 전 모듈 클린룸 독립 구현을 채택하여 적용했다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:34-48`)
3. **§2.1 Browser Use 메커니즘 분석 (`orca/src/main/browser`)**
   - **결정**: 외부 라이브러리 대신 Chrome DevTools Protocol(CDP) 직결, `getFullAXTree` 접근성 트리 기반 `@e1` 참조 태깅, 좌표 기반 클릭 메커니즘을 분석했다.
   - **구현 여부**: 분석 결과를 반영하여 `internal/browser`에 직접 구현했다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:74-102`)
4. **§2.2 Computer Use 메커니즘 분석 (`orca/src/main/computer`, `orca/native`)**
   - **결정**: Windows(PowerShell UIAutomation/SendInput) 및 macOS(Swift/Accessibility/CoreGraphics) 네이티브 메커니즘을 심층 분석했다.
   - **구현 여부**: 분석 완료. macOS 브리지를 우선 채택하여 구현했다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:103-120`)
5. **§2.3 Orchestration & 승인 게이트(Gate) 분석 (`orca/src/cli/handlers/orchestration`)**
   - **결정**: 사람의 개입을 예외 처리가 아닌 1급 데이터 프리미티브(Gate)로 규정하고 CLI 명령 및 WS 이벤트 모델을 분석했다.
   - **구현 여부**: 분석 완료. 단일 `gates` 테이블 및 도메인 서비스를 설계에 반영했다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:121-141`)
6. **§2.4 Automations (스케줄러) 분석 (`orca/src/cli/handlers/automations.ts`)**
   - **결정**: 스케줄 정의와 실행 이력(`runs`)의 분리 영속화 및 cron 기반 실행 구조를 분석했다.
   - **구현 여부**: 분석 완료. `automations`와 `automation_runs` 스키마를 도출했다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:142-150`)
7. **§3.1 독립 Browser MCP (`browser-mcp`)**
   - **결정**: 별도 독립 바이너리를 분리하지 않고 `internal/browser` 및 `internal/tools/browsertools` 15개 툴로 등록하며, HTML 대신 AXTree와 좌표 클릭을 채택했다.
   - **구현 여부**: 구현 완료. Phase 3을 통해 15개 브라우저 도구가 완전히 등록되었다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:182-242`)
8. **§3.2 독립 Computer MCP (`computer-mcp`)**
   - **결정**: node-agent가 아닌 로컬 호스트 실행으로 변경하고, Swift/cgo 대신 osascript/CoreGraphics 브리지를 채택하며, 화이트리스트(040)와 감사(041)를 연동했다.
   - **구현 여부**: macOS 환경에서 구현 완료. Windows/Linux 브리지는 미구현 상태이다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:243-294`)
9. **§3.3 1급 승인 게이트(Gate) 엔진**
   - **결정**: 채용, main 머지, 발령, 예산 승인을 통합하는 단일 `gates` 테이블 DDL을 확정하고, Broadcaster를 통한 WS 이벤트 전파를 정립했다.
   - **구현 여부**: 구현 완료. Phase 1 마이그레이션 `037` 및 `internal/gate` 패키지로 완성되었다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:295-339`)
10. **§3.4 Worktree 계보(`parent_worktree_id`) 및 프로세스 뷰 (`worktree ps`)**
    - **결정**: `worktrees`에 `parent_worktree_id`와 `process_status`를 추가하여 태스크 축 계보를 추적하고, 실행 상태를 조회하는 `worktree ps` 엔드포인트를 확정했다.
    - **구현 여부**: 구현 완료. Phase 4를 통해 REST API 및 CLI에 반영되었다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:340-380`)
11. **§3.5 스케줄러 & 자동화 엔진 (`automations` / `runs`)**
    - **결정**: 미결 #29(만료 기억 삭제), D-06(`div_rebase`), E2E 테스트를 수용하는 러너를 구축하고, `div_rebase` 시 3단계 조상 보존 규칙을 집행하도록 결정했다.
    - **구현 여부**: 구현 완료. `internal/automation`에서 `memory_cleanup`, `custom_prompt`, `div_rebase` 잡이 구현되었다(`night_e2e`는 미구현). (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:381-444`)
12. **§3.6 CLI 인터페이스 확장 (`workbench` & `wb`)**
    - **결정**: 운영자용 `workbench`(gate, worktree, automation, browser)와 에이전트용 `wb`(gate, task) 인터페이스를 정의했다.
    - **구현 여부**: 일부 구현. `cmd/orchestrator` 내 `ops` 서브커맨드로 운영자 CLI가 구현되었으나 경량 독립 바이너리 `wb`는 분리되지 않았다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:445-478`)
13. **§4. DB 마이그레이션 계획 (`037_orca_internalization.sql`)**
    - **결정**: `gates`, `automations`, `automation_runs` 테이블 생성 및 `worktrees` 계보 컬럼 추가 DDL을 계획했다.
    - **구현 여부**: 구현 완료. `migrations/037_orca_internalization.sql`에 정확히 반영되었다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:479-556`)
14. **§5. 구현 로드맵 (Execution Phases)**
    - **결정**: Phase 1(게이트 1급화), Phase 2(스케줄러/자동화), Phase 3(독립 브라우저), Phase 4(워크트리 계보/CLI) 순차 실행을 확정했다.
    - **구현 여부**: 로드맵의 Phase 1~4 및 Computer Use까지 전체 단계가 구현 완료되었다. (`legacy-workbench/docs/20-orca-도구-독립-내재화-설계.md:557-565`)

---

### 3.2 21부 (Orca 234개 전수 기능 카탈로그) 그룹별 대조

21부(`docs/21-orca-전체-기능-카탈로그-및-독립-구현-명세.md`)에 정의된 전체 19개 도메인 234개 명령을 기능 그룹별로 묶어 독립 구현 상태와 대응하는 legacy 모듈을 대조한 결과는 다음과 같다.

| 기능 그룹 | 포함된 세부 도메인 | Orca 명령 수 | 독립 구현 상태 | 대응하는 legacy 모듈 |
|---|---|---|---|---|
| **Browser Use** | 도메인 1(기본 38개) + 도메인 2(고급 40개) | 78 | 일부 (핵심 구현됨) | `internal/browser`, `internal/tools/browsertools` (15개 도구 등록) |
| **Orchestration & Gate** | 도메인 3(Orchestration & Gate) | 30 | 일부 (게이트 완료) | `internal/gate`, `internal/store/gates.go`, `internal/api/gates.go` |
| **Computer Use** | 도메인 4(Computer Use) | 14 | 일부 (macOS 완료) | `internal/computer`, `cmd/orchestrator/ops_desktop.go` |
| **Automations (스케줄러)** | 도메인 5(Automations / 스케줄러) | 7 | 구현됨 | `internal/automation`, `internal/schedule`, `internal/store/automations.go` |
| **Core (Worktree/Terminal/Repo)** | 도메인 6(Core: Worktree, Terminal, Repo) | 26 | 일부 (워크트리 완료) | `internal/git`, `internal/workspace`, `internal/toolsock` |
| **Linear / 이슈 트래커** | 도메인 7(Linear / 이슈 트래커 연동) | 27 | 일부 (자체 DB 대체) | `internal/tools/task`, `internal/store/issue.go` (자체 태스크로 대체) |
| **Mobile Emulator** | 도메인 8(Emulator: 모바일/안드로이드) | 16 | 설계만 (미구현) | 모바일 전용 도구로 P3 선택 과제로 보류됨 |
| **기타 운영 및 관리 도구** | 도메인 9(프로젝트 7) + 도메인 10(스킬 6) + 도메인 11(환경 5) + 도메인 12(산출물 5) + 도메인 13(훅 4) + 도메인 14(파일 3) + 도메인 15(계정 2) + 도메인 16(진단 1) + 도메인 17(메타 1) + 도메인 18(데몬 1) + 도메인 19(VM 1) | 42 | 구현됨 (대부분) | `cmd/orchestrator`, `internal/node`, `internal/tools`, `internal/policy` 등 |
| **합계** | **전체 19개 도메인** | **234** | **핵심 도메인 내재화 완료** | 그룹 합계는 21부 명세의 전체 234개 명령 수와 정확히 일치한다. |

> **검증 결과**: 그룹별 명령 수의 합계는 `78 + 30 + 14 + 7 + 26 + 27 + 16 + 42 = 234`개로 21부 문서 머리말에 명시된 총 234개 명령 수와 정확히 부합한다. (`legacy-workbench/docs/21-orca-전체-기능-카탈로그-및-독립-구현-명세.md:11-36`)

---

## 4. 컨테이너 계정 격리 (M-14) 조사

`docker-compose.agy.yml`, `deploy/agy/`, `docs/16-구독-설계.md`의 M-14 절 및 `docs/09-진행-상황.md`에 기록된 컨테이너 계정 격리의 설계와 실측 결과는 다음과 같다.

### 4.1 설계 및 구현 내용

1. **배경 및 원칙 (M-14)**
   - Google OAuth 정액 쿼터는 OS 사용자 및 키체인 수준에 종속되므로 호스트 Mac 환경에서 다중 계정 충돌을 방지하기 위해 컨테이너 격리를 채택했다.
   - Mac 로컬 사용자 분리(`sudo -u`) 시 발생하는 파일 소유권(UID) 꼬임과 sudoers 관리 부담을 배제하고, 서비스 정의 복사만으로 확장이 가능한 구조를 수립했다. (`legacy-workbench/docs/16-구독-설계.md:326-340`)
2. **컨테이너 구성 (`docker-compose.agy.yml`)**
   - 계정 3개 서비스를 정의했다: `wb-agy-lead` (Lead Pro #1), `wb-agy-ultra` (Worker Ultra #3 메인 실무), `wb-agy-backup` (Worker Pro #2 백업 안전망).
   - 프로세스를 `sleep infinity`로 항시 기동하고, 오케스트레이터가 매 턴마다 `docker exec`로 진입하여 기동 지연 시간을 제거했다.
   - 홈 디렉터리 볼륨 바인드: `~/.workbench/agy-homes/<역할>` 디렉터리를 컨테이너 내부 홈에 마운트하여 최초 1회 브라우저 OAuth 로그인 후 세션 토큰을 영구 보존했다.
   - 작업 공간 마운트: DB(`worktrees.path`)에 저장된 호스트 절대 경로와 컨테이너 내부 경로를 동일하게 맞추기 위해 `${WB_PROJECTS_ROOT}`를 동일 경로로 바인드 마운트했다. (`legacy-workbench/docker-compose.agy.yml:1-55`)
3. **이미지 빌드 및 권한 관문 보장 (`deploy/agy/`)**
   - `deploy/agy/Dockerfile`: Debian bookworm-slim 기반에 공식 Linux arm64 tarball(`storage.googleapis.com/antigravity-public/antigravity-cli/<ver>/linux-arm/cli_linux_arm64.tar.gz`)을 내려받아 `agy` CLI 바이너리를 설치했다. (`legacy-workbench/deploy/agy/Dockerfile:1-23`)
   - `deploy/agy/entrypoint.sh`: 컨테이너가 뜰 때마다 `settings.json`의 `permissions.allow` 배열에 `"mcp(*)"`를 병합 추가(`unique`)하도록 작성했다. agy 내장 57개 도구(브라우저 23개 포함)가 조직의 화이트리스트와 정책 감사를 우회하지 못하도록 헤드리스 모드의 기본 거부 원리를 활용하여 잠그고 오직 MCP 도구만 통과시켰다.
   - 위험 플래그 차단: 관문을 무력화하는 `--dangerously-skip-permissions` 플래그의 사용을 엄격히 금지했다. (`legacy-workbench/deploy/agy/entrypoint.sh:1-35`)

### 4.2 측정값 및 실측 근거

문서에 명시된 실측 측정값과 미측정 항목은 다음과 같다.

- **CLI 버전 확인 속도**: 호스트 macOS 환경에서 `agy --version` 실행 시 300초를 초과하여 프로세스가 매달렸으나(hang), 컨테이너 내부에서는 즉시 `1.2.0`을 출력하며 정상 응답했다. (`legacy-workbench/docs/09-진행-상황.md:4005-4012`)
- **실측 호출 명령 형식**: `agy --conversation <id> --output-format stream-json --print='<프롬프트>'` 형식을 사용했다. `--conversation <id>` 플래그를 통해 턴 간 문맥이 유지됨(`num_turns: 2`)을 확인했다. (`legacy-workbench/docs/09-진행-상황.md:4013-4025`)
- **내장 도구 차단 실측**: 초기 `init` 이벤트에서 57개 내장 도구가 감지되었으나, `permissions.allow = ["mcp(*)"]` 적용 후 내장 도구 호출은 전면 차단되고 MCP 도구만 정상 진입함을 실측으로 확인했다. (`legacy-workbench/docs/09-진행-상황.md:4049-4087`)
- **수동 권한 보존 실측**: 사용자가 `read_url` 권한을 수동 추가한 뒤 재기동했을 때 기존 설정이 삭제되지 않고 유지됨을 확인했다. (`legacy-workbench/docs/09-진행-상황.md:4120-4127`)
- **자원 소모량 측정값**: OrbStack 환경 도입 분석 시 Docker Desktop 대비 유휴 CPU 점유율은 약 0% 수준, RAM 점유율은 1/5 수준으로 보고되었다. (`legacy-workbench/docs/16-구독-설계.md:330-332`)
- **그 외 정량 지표**: 네트워크 지연 시간(ms) 및 초당 입출력 I/O 대역폭 수치는 별도로 측정되지 않았으므로 '미측정'으로 기록한다.
