# 목표 구조

이 문서는 엔진(oh-my-teams)과 제품(workbench)의 분리를 전제로 목표 아키텍처를 규정한다. 제품의 호스트 설비 언어, 상태 저장소, 계정 격리 실행기, 화면 및 권한 구조를 효율 관점에서 비교하여 결정한다. 또한, 구형 설계 초안인 8028b89의 유산을 선별 수용하여 시스템의 안정성과 확장성을 확보한다.

## 엔진과 제품 경계

사용자의 "엔진과 제품 분리" 결정에 따라 역할은 나뉜다. 엔진(oh-my-teams)은 역할, 워크플로, 비동기 호출 등 다중 에이전트 조율 로직과 순수 도구 스크립트를 제공한다. 제품(workbench)은 엔진을 내장하고 데이터베이스, 컨테이너 계정 격리, 브라우저 제어, 데스크톱 및 모바일 화면을 통합하는 애플리케이션이다.

## 비교 및 결정

### 호스트 설비 언어

에이전트 CLI가 호스트 언어와 무관한 별도 프로세스(node-agent 등)로 실행된다는 전제하에, 호스트 자체의 상주 비용과 OMT(Node)와의 결합 비용(프로세스 재사용 vs IPC)을 따져 결정한다.

| 대안 | 상주 메모리 | 배포 크기 | OMT 결합 비용 |
|---|---|---|---|
| **Node.js (잠정 결정)** | 단일 프로세스 통합 시 V8 중복 없음 | 보통 | **없음 (같은 프로세스 재사용 시 IPC 불필요)** |
| Go | 매우 낮음 | 작음 (단일 바이너리) | 높음 (Node 프로세스와의 IPC 및 직렬화 비용 발생) |

- **결정 사유 (잠정 결정)**: Go의 단일 바이너리와 낮은 상주 메모리(8.9MB)는 장점이나, OMT가 Node.js 기반이므로 Go 호스트를 사용할 경우 OMT를 위한 별도 Node 프로세스를 띄우고 IPC 통신을 해야 하는 결합 비용이 발생한다. Node.js 호스트를 사용하여 OMT와 같은 프로세스를 재사용하면 IPC 직렬화 비용을 없앨 수 있어 효율적이다.
- **확정 기준**: 현재 측정값이 없으므로 잠정 결정한다. 추후 Node 단일 프로세스 체제의 총 메모리 사용량과 Go+Node 다중 프로세스 체제의 총 상주 메모리 및 IPC 지연 시간을 직접 측정하여, 더 효율적인 쪽으로 결정을 확정하거나 뒤집는다.

### 상태 저장소

| 대안 | 가시성 | 백업 | 설치 부담 | 메모리 오버헤드 |
|---|---|---|---|---|
| 파일 시스템 | 높음 | 단순 | 없음 | 낮음 |
| **SQLite (결정)** | 보통 | 단순 (단일 파일 복사) | 없음 (내장) | 낮음 |
| PostgreSQL | 매우 높음 | 복잡 | 매우 높음 | 높음 (데몬 상주) |

- **결정 사유**: 워크스테이션에서 별도 DB 프로세스(상주 메모리 100MB+)가 에이전트 구동을 방해하지 않도록 배제한다.
- **효율 근거**: PostgreSQL 설치 및 백그라운드 메모리 오버헤드를 없애고 SQLite를 애플리케이션 내장으로 채택하여 유휴 비용을 0으로 한다 (미측정).

### 계정 격리 실행기
| 대안 | 효율 (메모리·속도) | 계정 격리 강도 |
|---|---|---|
| Docker Desktop | 낮음 (무거운 VM 1~2GB) | 높음 |
| **가벼운 실행기 (결정)** | 높음 (가벼운 VM 또는 루트리스) | 높음 |

- **결정 사유**: 다중 계정을 위해 컨테이너 격리를 하되, OS별 최적화 경량 실행기(macOS의 OrbStack/Colima, Linux의 Podman, Windows의 WSL2 직접 배포)를 선택한다.
- **도입 근거**: legacy M-14 (`legacy-workbench/docs/16-구독-설계.md`) 문서에 따르면 OrbStack이 Docker Desktop 대비 RAM 사용량이 1/5 수준이며 유휴 CPU 자원을 거의 소모하지 않는다는 서술을 바탕으로, 효율성 향상을 위한 채택 근거로 인용한다 (직접 측정값 아님).

### 화면 (구조 확정 사항)

초안 8028b89에 담긴 결정 사항들을 수용하여 다음 구조를 확정한다.
- **데스크톱**: Tauri 기반 네이티브 데스크톱 앱 (`docs/decisions/0002-tauri.md`).
- **모바일**: Capacitor를 이용한 JS 번들 OTA 앱 업데이트 (`docs/decisions/0003-capacitor-js-ota.md`).
- **원격 연결**: Tailscale VPN과 보안 토큰을 결합해 폐쇄망 내에서만 접근을 허용 (`docs/decisions/0004-tailscale-token-auth.md`).
- **화면 프레임워크**: React (`docs/decisions/0001-react.md`).

### 도구 권한 강제와 OMT 역할 결합

- **결정**: legacy의 도구 화이트리스트 및 브리지 통제를 OMT 직무 역할 명세와 연동한다.
- **이유**: [참고 01 문서](01-legacy-inventory.md)의 `entrypoint.sh` 관측(`legacy-workbench/deploy/agy/entrypoint.sh:1-35`)에서 증명된 차단 방식을 OMT의 `permissions.allow` 배열과 병합하여, 역할이 허락하지 않은 도구 호출을 OS 쉘에서 원천 차단한다.

## 초안 (8028b89) 반영 내역

초안 8028b89의 실제 12개 파일 내용을 바탕으로 다음 사항을 반영한다. 거짓 정보를 배제하고 실제 파일 기준으로 작성했다.

- `README.md`, `docs/architecture.md`: 제품과 엔진 분리 아키텍처 수용.
- `docs/account-isolation.md`, `docs/decisions/0006-account-isolation.md`: OS별(macOS/Linux/Windows) 경량 컨테이너 격리 및 다중 계정 격리 설계 수용.
- `docs/decisions/0001-react.md`: React 프레임워크 채택 반영.
- `docs/decisions/0002-tauri.md`: 데스크톱 앱 프레임워크로 Tauri 채택 반영.
- `docs/decisions/0003-capacitor-js-ota.md`: 모바일 프레임워크로 Capacitor 및 OTA 업데이트 방식 수용.
- `docs/decisions/0004-tailscale-token-auth.md`: Tailscale VPN 및 토큰 기반 원격 접속 인가 체계 채택.
- `docs/decisions/0005-director-session.md`: 에이전트 활동을 감독하는 디렉터 세션 통제 구조 반영.
- `docs/risks.md`: 무단 접근 방지 기본 비활성화 방침 및 보안 위협 방어 대책 반영.
- `docs/roadmap.md`: 단계별 마일스톤 및 릴리스 로드맵 반영.
- `docs/supervision.md`: 에이전트 감독 및 권한 통제 모델 수용.
