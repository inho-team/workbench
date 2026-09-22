# workbench v3 계획

결론: workbench v3 계획은 OMT를 핵심 엔진으로 두고 Orca 의존성을 제거하여 유휴 자원을 100MB 이하로 낮추고 반응 속도를 끌어올린다. 이를 위해 로컬 데스크톱 앱(Tauri)을 본체로 삼으며 어댑터 패턴 기반으로 3단계에 걸쳐 무정지 전환을 완수한다.

- [01-legacy-inventory.md](01-legacy-inventory.md): legacy-workbench의 22개 모듈 및 구현 현황 재고를 정리했다.
- [02-omt-inventory.md](02-omt-inventory.md): OMT의 13개 핵심 기능과 런타임 코드 규모를 정리했다.
- [03-orca-dependencies.md](03-orca-dependencies.md): OMT 런타임이 의존하는 Orca CLI 명령을 전수 조사했다.
- [04-orca-replacement.md](04-orca-replacement.md): Orca 기능을 대체할 구체적 어댑터와 구현 방향을 세웠다.
- [05-target-architecture.md](05-target-architecture.md): 엔진과 제품을 분리하여 호스트 언어와 DB, 계정 격리 수단을 정했다.
- [06-efficiency-budget.md](06-efficiency-budget.md): 턴 지연 500ms 등 효율 목표와 측정 수단을 정량적으로 명시했다.
- [07-absorption-decisions.md](07-absorption-decisions.md): 기능 단위 이식, 재작성, 폐기 결정과 효율 검증 근거를 표로 정리했다.
- [08-phases.md](08-phases.md): 전환 과정에서 기존 운영을 멈추지 않는 3단계 마일스톤을 세웠다.
- [09-public-scope.md](09-public-scope.md): 오픈 소스 전환을 위해 기능별 라이선스 및 내부 정보 제거 지침을 적었다.
- [10-draft-review.md](10-draft-review.md): 8028b89 이전 초안의 아키텍처 및 화면 결정을 목표 구조에 수용했다.
