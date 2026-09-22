# 흡수 결정표

결론: legacy-workbench의 브라우저, 정책, 스케줄러 기능은 다시 작성하거나 이식하며, DB는 SQLite로 교체하여 재작성한다. 워크트리와 터미널 통제는 OMT의 헤드리스 런타임과 단일 프로세스 구조로 대체하여 IPC 비용과 유휴 메모리를 최소화한다.

## 결정 목록

| 기능 | 결정 | 효율 근거 | 검증 정도 | 유지 비용 | 공개 가능 여부 | 관련 설계 | 근거 위치 |
|---|---|---|---|---|---|---|---|
| internal/api | 다시 작성 | IPC 통신 지연 제거 (미측정) | 구현됨 | 낮음 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/api/server.go:1-50 |
| internal/automation | 다시 작성 | 데몬 메모리 소모 없음 (미측정) | 구현됨 | 보통 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/automation/automation.go:1-35 |
| internal/browser | 이식 | 단일 런타임에서 즉각 통신 (미측정) | 구현됨 | 높음 | 검토 필요 | [04-orca-replacement.md](04-orca-replacement.md) | legacy-workbench/internal/browser/cdp.go:1-40 |
| internal/codex | OMT에 이미 있어 legacy 것은 버림 | OpenCodex 활용으로 자원 절약 (미측정) | 일부 | 없음 | 해당 없음 | [04-orca-replacement.md](04-orca-replacement.md) | legacy-workbench/internal/codex/client.go:1-50 |
| internal/computer | 보류 | OS 의존성으로 배포 크기 증대 | 구현됨 | 매우 높음 | 검토 필요 | [04-orca-replacement.md](04-orca-replacement.md) | legacy-workbench/internal/computer/computer.go:1-30 |
| internal/config | 다시 작성 | 호스트 환경 변수 조회 단순화 | 구현됨 | 낮음 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/config/config.go:1-35 |
| internal/embed | 둘 다 버림 | 별도 무거운 데몬 상주 배제 | 일부 | 없음 | 해당 없음 | [06-efficiency-budget.md](06-efficiency-budget.md) | legacy-workbench/internal/embed/embed.go:1-40 |
| internal/gate | OMT에 이미 있어 legacy 것은 버림 | 기존 게이트 증거 파일 사용 | 구현됨 | 없음 | 해당 없음 | [04-orca-replacement.md](04-orca-replacement.md) | legacy-workbench/internal/gate/gate.go:1-35 |
| internal/git | OMT에 이미 있어 legacy 것은 버림 | 기존 local-adapter 사용 | 구현됨 | 없음 | 해당 없음 | [04-orca-replacement.md](04-orca-replacement.md) | legacy-workbench/internal/git/git.go:1-35 |
| internal/lego | 보류 | 레고 엔진 대신 프롬프트로 대체 | 구현됨 | 보통 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/lego/block/manifest.go:1-35 |
| internal/mcp | OMT에 이미 있어 legacy 것은 버림 | OMT 도구 생태계 사용 | 일부 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/mcp/doc.go:1-36 |
| internal/media | 보류 | 미디어 파이프라인 외부 의존 | 구현됨 | 매우 높음 | 불가 | [06-efficiency-budget.md](06-efficiency-budget.md) | legacy-workbench/internal/media/media.go:1-40 |
| internal/node | OMT에 이미 있어 legacy 것은 버림 | 통신 직렬화 오버헤드 0 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/node/agy.go:1-40 |
| internal/org | OMT에 이미 있어 legacy 것은 버림 | OMT 설정 구조 재사용 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/org/agentsmd.go:1-35 |
| internal/policy | 다시 작성 | OMT 역할 권한과 병합 적용 | 구현됨 | 낮음 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/policy/bridge.go:1-35 |
| internal/quota | OMT에 이미 있어 legacy 것은 버림 | 기존 limit-check 재사용 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/quota/quota.go:1-35 |
| internal/schedule | 다시 작성 | SQLite 스케줄러로 교체 | 구현됨 | 보통 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/schedule/cron.go:1-35 |
| internal/session | OMT에 이미 있어 legacy 것은 버림 | OMT 워크플로에 흡수 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/session/tracker.go:1-35 |
| internal/store | 다시 작성 | DB 데몬 제거로 100MB 달성 | 구현됨 | 낮음 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/store/store.go:1-40 |
| internal/tools | OMT에 이미 있어 legacy 것은 버림 | OMT 제공 기능 사용 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/tools/registry.go:1-40 |
| internal/toolsock | 둘 다 버림 | 프로세스 분리 없어 소켓 불필요 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/internal/toolsock/server.go:1-35 |
| internal/workspace | OMT에 이미 있어 legacy 것은 버림 | local-adapter 워크트리 재사용 | 구현됨 | 없음 | 해당 없음 | [04-orca-replacement.md](04-orca-replacement.md) | legacy-workbench/internal/workspace/clone.go:1-35 |
| cmd/orchestrator | 둘 다 버림 | 데스크톱 앱 기반 구조로 재설계 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/cmd/orchestrator/main.go:1-40 |
| cmd/node-agent | 둘 다 버림 | 데스크톱 앱 런타임으로 대체 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/cmd/node-agent/main.go:1-30 |
| cmd/ | 둘 다 버림 | 데스크톱 앱으로 대체 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/cmd/orchestrator/main.go:1-40 |
| web/ | 다시 작성 | Tauri/React 기반 앱으로 통합 | 구현됨 | 보통 | 검토 필요 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/web/app/page.tsx:1-40 |
| 
ode-agent/ | 둘 다 버림 | 단일 바이너리 구조 미사용 | 구현됨 | 없음 | 해당 없음 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/docs/workbench-spec.html:2061 |
| migrations/ | 다시 작성 | SQLite 스키마로 이관 | 구현됨 | 보통 | 가능 | [05-target-architecture.md](05-target-architecture.md) | legacy-workbench/migrations/001_init.sql:1-40 |
| 역할 | OMT에 이미 있어 legacy 것은 버림 | 프롬프트 재사용 | 검증 완료 | 낮음 | 가능 | [05-target-architecture.md](05-target-architecture.md) | oh-my-teams/plugins/oh-my-teams/scripts/role-launch.mjs:1-50 |
| kickoff 등록부 | OMT에 이미 있어 legacy 것은 버림 | 기존 워크트리 바인딩 로직 재사용 | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/scripts/kickoff-registry.mjs:1-45 |
| workflow | OMT에 이미 있어 legacy 것은 버림 | 상태 기계 재사용 (미측정) | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/scripts/workflow.mjs:1-60 |
| 게이트와 증거 | OMT에 이미 있어 legacy 것은 버림 | 1급 게이트 증거 모델 보존 | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/scripts/gates.mjs:1-40 |
| 이사 신호 | OMT에 이미 있어 legacy 것은 버림 | 직접 호출로 전달 지연 100ms 달성 목적 | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/scripts/director.mjs:1-50 |
| 자원 슬롯 | OMT에 이미 있어 legacy 것은 버림 | 동시성 제한으로 메모리 고갈 방지 | 검증 완료 | 낮음 | 가능 | [06-efficiency-budget.md](06-efficiency-budget.md) | oh-my-teams/plugins/oh-my-teams/scripts/resources.mjs:1-50 |
| headless 런타임 | OMT에 이미 있어 legacy 것은 버림 | TUI 미사용으로 오버헤드 최소화 | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/scripts/headless.mjs:1-50 |
| 호환성 표 | OMT에 이미 있어 legacy 것은 버림 | 사전 진단 및 호환성 보장 | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/scripts/launch-matrix.mjs:1-50 |
| 사용 한도 handoff | OMT에 이미 있어 legacy 것은 버림 | 쿼터 소진 방지 방어층 구축 | 검증 완료 | 낮음 | 가능 | [06-efficiency-budget.md](06-efficiency-budget.md) | oh-my-teams/plugins/oh-my-teams/scripts/limit-check.mjs:1-50 |
| advise·assist | OMT에 이미 있어 legacy 것은 버림 | 자체 코드 참조 체계 지속 | 검증 완료 | 낮음 | 가능 | [04-orca-replacement.md](04-orca-replacement.md) | oh-my-teams/plugins/oh-my-teams/references/advise.md:1-40 |
| OpenCodex | OMT에 이미 있어 legacy 것은 버림 | 로컬 백엔드 연동 지속 유지 | 검증 완료 | 낮음 | 가능 | [05-target-architecture.md](05-target-architecture.md) | oh-my-teams/plugins/oh-my-teams/scripts/opencodex.mjs:1-50 |
| 사용량 보고 | OMT에 이미 있어 legacy 것은 버림 | 비용 집계 모델 보존 | 검증 완료 | 낮음 | 가능 | [06-efficiency-budget.md](06-efficiency-budget.md) | oh-my-teams/plugins/oh-my-teams/scripts/usage-report.mjs:1-50 |
| 대시보드 | OMT에 이미 있어 legacy 것은 버림 | 대시보드 뷰 통합 재작성 | 검증 완료 | 낮음 | 검토 필요 | [05-target-architecture.md](05-target-architecture.md) | oh-my-teams/plugins/oh-my-teams/scripts/dashboard.mjs:1-50 |
