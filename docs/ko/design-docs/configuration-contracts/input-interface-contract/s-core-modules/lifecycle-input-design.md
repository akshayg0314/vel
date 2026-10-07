# S-CORE Lifecycle — 입력 인터페이스 설계

> **참조:** [lifecycle-collector-design.md](../../../evidence-ingestion/s-core-state-collection/collectors/lifecycle-collector-design.md) — 수집기가 S-CORE Lifecycle API에서 이 상태를 읽는 방법.

## 1. 관찰 가능한 상태

**S-CORE 컴포넌트:** Launch Manager, Health Monitor (PHM — Platform Health Management).

| 관찰 가능 항목 | S-CORE 소스 |
| --- | --- |
| 컴포넌트 상태 (idle/starting/running/terminating/terminated/failed) | `ProcessState` 열거형 (`lifecycle/score/launch_manager/src/daemon/src/process_group_manager/process_state.hpp`) |
| 실행 대상 (debug/production/test) | 활성 실행 대상 구성 |
| 프로세스 그룹 / 종속성 그래프 상태 | `GraphState` 열거형 |
| 프로세스 그룹 식별자 | `IdentifierHash` (문자열 기반; 섹션 2 참고) |
| 프로세스 종료 코드, PID | Launch Manager 외부 모니터 알림 |
| Health Monitor 감독 상태 | 데드라인 / 로직 / 하트비트 모니터 상태, 집계됨 |
| 복구 작업 상태 | `RestartAction` / `SwitchRunTargetAction` 구성 구조체 |

**VEL 수집기가 사용하는 S-CORE API:** Launch Manager 외부 모니터 알림 (`comp_req__launch_man__ext_monitor_notify`).

> **접근성 참고:** VEL은 안정적이고 공개된 외부 API를 통해 노출된 데이터만 수집할 수 있습니다. `pid`, `exit_code`, `process_execution_error`를 포함한 모든 라이프사이클 필드는 `comp_req__launch_man__ext_monitor_notify`를 통해 노출됩니다. 전체 접근성 경계는 evidence 수집 문서의 S-CORE 모듈 수집기 API 설계를 참조하세요.

## 2. 소스별 참고 사항

- **`process_group_id`는 `string`이며 `uint32`가 아닙니다.** `IdentifierHash` (`lifecycle/score/launch_manager/src/daemon/src/common/identifier_hash.hpp`)로, 숫자 핸들이 아닌 문자열 파생 식별자를 래핑합니다.
- **`previous_state`는 `enum`이며 `string`이 아닙니다.** `state`와 동일한 `allowedValues`를 사용합니다 (둘 다 `ProcessState`).
- **`recovery_action`은 의도적으로 이 스키마의 일부가 아닙니다.** 소스는 서로 다른 선택적 구성 슬롯 (`ready_recovery_action` / `recovery_action`)에서 사용되는 두 개의 별개 구성 구조체 (`recovery_action_config.hpp`의 `RestartAction`, `SwitchRunTargetAction`)를 정의하며, 단일 태그된 유니온/열거형이 아닙니다. 현재는 `recovery_state` (복구 프로세스의 상태, 수행된 작업이 아님)만 수집됩니다.
- **Evidence 유형 명명:** 각 `evidenceType`의 마지막 경로 세그먼트는 소스 필드의 밑줄을 유지합니다 (예: `deadline_error`, `process_group_id`). 구조적/범주 세그먼트 (예: `process-group`, `health`)는 하이픈을 유지합니다.

## 3. 소스 식별 (예시 인스턴스화)

| 필드 | 예시 |
| --- | --- |
| `source_id` | `score-lifecycle` |
| `workload_id` | `launch-manager` |
| `resource_id` | `component:networking` |

## 4. 예시 원시 상태 레코드

```json
{
  "message_id": "msg-0001",
  "source_id": "score-lifecycle",
  "workload_id": "launch-manager",
  "execution_id": "exec-42",
  "resource_id": "component:networking",
  "timestamp_ns": 1710000000000000000,
  "state": "running",
  "previous_state": "starting",
  "transition_time_ns": 1710000000000000000,
  "details": {
    "run_target": "production",
    "exit_code": 0,
    "pid": 4821,
    "graph_state": "success",
    "supervision_status": "failed",
    "failed_supervision_cycles": 2,
    "failed_supervision_type": "deadline",
    "deadline_error": "too_late",
    "deadline_violation": true,
    "alive_indication_count": 42,
    "process_group_id": "mini_adas_group",
    "recovery_state": "idle"
  }
}
```

> **예시 참고:** `heartbeat_error`와 `logic_error`는 데드라인 모니터만 실패를 보고했으므로 (`failed_supervision_type: deadline`) 여기서 생략됩니다. 유형별 오류 필드는 실제로 실패한 감독 유형에 대해서만 채워지며 전체 레코드에 대해 채워지지 않습니다. 실행 오류가 발생하지 않았으므로 `process_execution_error`는 생략됩니다 (이 필드에는 소스 `ExecErrc` 열거형에 정의된 "오류 없음" 센티널 값이 없습니다 — 섹션 5 참조).

## 5. `RawEvidenceSchema` 및 정규화 매핑

표준 권위 아티팩트는 인터페이스 패키지에서 유지 관리됩니다 (문서와 계약 간의 드리프트를 피하기 위해 여기에 인라인으로 중복하지 않음):

- **입력 스키마:** `schemas/score-lifecycle-state-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/schemas/score-lifecycle-state-v1.0.0.yaml`)
- **정규화 매핑:** `mappings/score-lifecycle-to-evidence-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/mappings/score-lifecycle-to-evidence-v1.0.0.yaml`)
- **수집기 구성:** `collectors/lifecycle-collector.yaml` (`interfaces/score-evidence-layer-package/collectors/lifecycle-collector.yaml`)

스키마에 대한 주요 사항 (소스에 대해 검증됨):

- `state` / `previous_state`: `enum`, 값 `[idle, starting, running, terminating, terminated, failed]` (`ProcessState`와 일치).
- `process_group_id`: `string` (섹션 2 참조).
- `process_execution_error`: `uint32` (스키마에서 제한된 열거형이 아님) — 소스 `ExecErrc` 값은 1부터 시작합니다 (`kGeneralError`); "오류 없음"에 대한 정의된 값이 없으므로 오류가 발생하지 않은 경우 `0`으로 설정하는 대신 필드를 생략해야 합니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 설계 | S-CORE Lifecycle 입력 설계 |
| 상위 설계 | [s-core-module-input-design.md](../s-core-module-input-design.md) — 공유 S-CORE 소스 모델 |
| 요구사항 | FR-VEL-003, FR-VEL-012, FR-VEL-013, AOU-VEL-002 |
| 참조 S-CORE 모듈 | `lifecycle/` |
