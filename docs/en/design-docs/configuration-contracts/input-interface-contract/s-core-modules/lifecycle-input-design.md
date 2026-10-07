# S-CORE Lifecycle — Input Interface Design

> **See also:** [lifecycle-collector-design.md](../../../evidence-ingestion/s-core-state-collection/collectors/lifecycle-collector-design.md) — how the collector reads this state from the S-CORE Lifecycle API.

## 1. Observable State

**S-CORE components:** Launch Manager, Health Monitor (PHM — Platform Health Management).

| Observable | S-CORE Source |
| --- | --- |
| Component state (idle/starting/running/terminating/terminated/failed) | `ProcessState` enum (`lifecycle/score/launch_manager/src/daemon/src/process_group_manager/process_state.hpp`) |
| Run target (string name, e.g. "Startup", "Off", "Fallback") | `RunTargetConfig.name` (arbitrary string, not a fixed enum) |
| Process group / dependency graph state | `GraphState` enum |
| Process group identity | `IdentifierHash` (string-backed; see Section 2 note) |
| Process exit code, PID | Launch Manager external monitor notification |
| Health Monitor supervision status | Deadline / logic / heartbeat monitor status, aggregated (VEL-derived) |
| Recovery state | `recovery_state_` — hardcoded `IdentifierHash("fallback")` (fallback run target name) |

**S-CORE API used by the VEL collector:** Launch Manager external monitor notification (`comp_req__launch_man__ext_monitor_notify`).

> **Accessibility note:** VEL can only collect data exposed through a stable, public, external-facing API. All lifecycle fields — including `pid`, `exit_code`, and `process_execution_error` — are exposed through `comp_req__launch_man__ext_monitor_notify`. See the S-CORE module collector API design in the evidence-ingestion documentation for the full accessibility boundary.

## 2. Source-Specific Notes

- **`process_group_id` is `string`, not `uint32`.** It is an `IdentifierHash` (`lifecycle/score/launch_manager/src/daemon/src/common/identifier_hash.hpp`), which wraps a string-derived identifier, not a numeric handle.
- **`previous_state` is `enum`, not `string`.** It uses the same `allowedValues` as `state` (both are `ProcessState`).
- **`run_target` is `string`, not a constrained enum.** `RunTargetConfig.name` (`config.hpp`) is an arbitrary string (e.g. "Startup", "Off", "Fallback", "Running", "SafeState"). Run target names are integrator-defined, not a fixed `[debug, production, test]` set.
- **`recovery_state` is `string`, not a recovery process state machine.** The source defines `recovery_state_` as a hardcoded `const IdentifierHash recovery_state_{"fallback"}` (`process_group_manager.hpp`). It represents the fallback run target name, not an `[idle, timeout, sending, waiting_for_response]` state machine. The recovery mechanism is a simple `sendRecoveryRequest` → `handleRecoveryRequest` → transition to the fallback run target.
- **`recovery_action` is intentionally not part of this schema.** The source defines two distinct config structs (`RestartAction`, `SwitchRunTargetAction` in `recovery_action_config.hpp`) used in different optional config slots (`ready_recovery_action` / `recovery_action`), not a single tagged union/enum. Only `recovery_state` (the fallback run target name) is currently collected.
- **`supervision_status`, `failed_supervision_cycles`, and `deadline_violation` are VEL-derived aggregations**, not direct source fields. The health monitor reports per-monitor errors via `MonitorEvaluationError` (`Deadline`, `Heartbeat`, `Logic` variants in `common.rs`). VEL aggregates these into `supervision_status` (`ok`/`failed`), counts failed cycles, and derives `deadline_violation` from the presence of a `DeadlineEvaluationError`.
- **Evidence Type naming:** the final path segment of each `evidenceType` preserves the source field's underscores (e.g. `deadline_error`, `process_group_id`) rather than converting them to hyphens, per the field-name-as-evidence-identity convention. Structural/category segments (e.g. `process-group`, `health`) remain hyphenated.

## 3. Source Identity (example instantiation)

| Field | Example |
| --- | --- |
| `source_id` | `score-lifecycle` |
| `workload_id` | `launch-manager` |
| `resource_id` | `component:networking` |

## 4. Example Raw State Record

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
    "run_target": "Running",
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
    "recovery_state": "fallback"
  }
}
```

> **Example note:** `heartbeat_error` and `logic_error` are omitted here because only the deadline monitor reported a failure (`failed_supervision_type: deadline`); per-type error fields are only populated for the supervision type(s) that actually failed, not for the whole record. `process_execution_error` is omitted because no execution error occurred (the field has no defined "no error" sentinel value in the source `ExecErrc` enum — see Section 5).

## 5. `RawEvidenceSchema` and Normalization Mapping

The canonical, authoritative artifacts are maintained in the interface package (not duplicated inline here, to avoid drift between documentation and contract):

- **Input schema:** `schemas/score-lifecycle-state-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/schemas/score-lifecycle-state-v1.0.0.yaml`)
- **Normalization mapping:** `mappings/score-lifecycle-to-evidence-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/mappings/score-lifecycle-to-evidence-v1.0.0.yaml`)
- **Collector configuration:** `collectors/lifecycle-collector.yaml` (`interfaces/score-evidence-layer-package/collectors/lifecycle-collector.yaml`)

Key points about the schema (verified against source):

- `state` / `previous_state`: `enum`, values `[idle, starting, running, terminating, terminated, failed]` (matches `ProcessState`).
- `process_group_id`: `string` (see Section 2).
- `run_target`: `string` (see Section 2) — arbitrary run target name, not a fixed enum.
- `recovery_state`: `string` (see Section 2) — the fallback run target name (`"fallback"`), not a recovery state machine.
- `process_execution_error`: `uint32` (not a constrained enum in the schema) — the source `ExecErrc` values start at 1 (`kGeneralError`); there is no defined value for "no error", so the field should be omitted rather than set to `0` when no error has occurred.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Design | S-CORE Lifecycle Input Design |
| Parent design | [s-core-module-input-design.md](../s-core-module-input-design.md) — shared S-CORE source model |
| Requirements | FR-VEL-003, FR-VEL-012, FR-VEL-013, AOU-VEL-002 |
| Reference S-CORE module | `lifecycle/` |
