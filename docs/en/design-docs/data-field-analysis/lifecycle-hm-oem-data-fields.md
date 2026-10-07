# Lifecycle / Health Monitoring — OEM Data Fields

> **Purpose:** This document lists the data fields from the S-CORE Lifecycle and Health Monitoring modules that are logically useful for OEM (Original Equipment Manufacturer) use cases such as vehicle-state monitoring, diagnostics, safety analysis, and fleet telemetry.
>
> **Accessibility:** This document lists the **logically useful** OEM fields exposed by the S-CORE Lifecycle and Health Monitoring modules.

---

## 1. Process Lifecycle State

| Field | Type | Allowed Values | Description | OEM Relevance |
|-------|------|----------------|-------------|---------------|
| `state` | enum | `idle`, `starting`, `running`, `terminating`, `terminated`, `failed` | Current state of a managed process. | Indicates whether a software component is up, down, starting, or failed — essential for vehicle-level availability and diagnostics. |
| `previous_state` | enum (`ProcessState`) | `idle` (0), `starting` (1), `running` (2), `terminating` (3), `terminated` (4), `failed` (5) | The state the process was in before the current state. Derived field (not persisted in source; computed from state transitions). | Enables transition analysis (e.g., how often a component goes through `failed`). |
| `transition_time_ns` | uint64 | — | Timestamp (ns) when the state transition occurred. | Correlates software state changes with vehicle events or other evidence. |
| `exit_code` | int32 | — | Process exit code when terminated. 0 = success; non-zero = failure/unexpected termination (OS-reported status from wait()). | Distinguishes clean shutdown from abnormal termination; supports root-cause analysis. |
| `process_execution_error` | uint32 | 1=GeneralError, 2=InvalidArguments, 3=CommunicationError, 4=MetaModelError, 5=Cancelled, 6=Failed, 7=FailedUnexpectedTerminationOnExit, 8=FailedUnexpectedTerminationOnEnter, 9=InvalidTransition, 10=AlreadyInState, 11=InTransitionToSameState, 12=NoTimeStamp, 13=CycleOverrun, 14=ActivationInProgress, 15=RequestQueueIsFull, 16=RunTargetDoesntExist | Error code from the process launcher (`ExecErrc`). The current implementation hardcodes `kGeneralError` (1) on abort. | Identifies launch failures (missing binary, permissions, resource exhaustion). |
| `pid` | uint32 | — | Process identifier exposed by the lifecycle API (zero if never started). | Identifies the exact process instance for forensics and cross-referencing lifecycle API logs. |

---

## 2. Process Group / Graph State

| Field | Type | Allowed Values | Description | OEM Relevance |
|-------|------|----------------|-------------|---------------|
| `graph_state` | enum | `success`, `in-transition`, `aborting`, `cancelled`, `undefined-state` | State of the process group (dependency graph). | Shows whether a coordinated set of components (e.g., an ADAS stack) is fully up, transitioning, or aborted. |
| `process_group_id` | string | — | Identifier of the process group. It is an `IdentifierHash` derived from a string name. | Groups related components for fleet-level health aggregation. |
| `run_target` | string | — | Active run target name (integrator-defined, e.g. `"Startup"`, `"Off"`, `"Fallback"`, `"Running"`, `"SafeState"`). | Indicates which software configuration is active — important for regulatory and safety traceability. |

---

## 3. Health Monitor Supervision Status

| Field | Type | Allowed Values | Description | OEM Relevance |
|-------|------|----------------|-------------|---------------|
| `supervision_status` | enum | `ok`, `failed` | Overall supervision status of a monitored component (derived from whether any per-type error is present). | Primary health indicator for a component under active supervision. |
| `failed_supervision_type` | enum | `heartbeat`, `deadline`, `logic` | Which supervision type detected the failure. | Distinguishes failure mode: missed heartbeat, missed deadline, or invalid state transition — critical for diagnostics. |
| `failed_supervision_cycles` | uint32 | — | Number of consecutive failed supervision cycles. | Indicates severity / persistence of a fault (transient vs. persistent). |
| `deadline_violation` | bool | `true`/`false` | Whether a deadline was violated. | Directly indicates a real-time scheduling failure (e.g., FEO cycle overrun). |
| `alive_indication_count` | uint32 | — | Count of alive indications reported. | Confirms the component is actively reporting liveness; absence indicates a stall. |

---

## 4. Supervision Type Details

### 4.1 Deadline Supervision

| Field | Type | Description | OEM Relevance |
|-------|------|-------------|---------------|
| `deadline_error` | enum (`too_early`, `too_late`) | Deadline evaluation error. | `too_late` = real-time overrun; `too_early` = premature completion (can be a logic fault). |
| `deadline_state.timestamp_ms` | uint32 | Timestamp in milliseconds. | Snapshot of deadline monitor state. |
| `deadline_state.is_running` | bool | Running flag. | Indicates the deadline monitor is actively running. |
| `deadline_state.is_stopped` | bool | Stopped flag. | Indicates the deadline monitor has been stopped. |
| `deadline_state.is_underrun` | bool | "Finished too early" flag. | `is_underrun` indicates the cycle finished before the minimum allowed time — a logic anomaly. |

### 4.2 Heartbeat Supervision

| Field | Type | Description | OEM Relevance |
|-------|------|-------------|---------------|
| `heartbeat_error` | enum (`too_early`, `too_late`, `multiple_heartbeats`) | Heartbeat evaluation error. | `too_late` = component stalled; `multiple_heartbeats` = double-execution / re-entrancy bug. |
| `heartbeat_state.heartbeat_timestamp` | uint64 (62-bit) | Heartbeat timestamp. | Confirms periodic liveness; timestamp drift correlates with clock/load issues. |
| `heartbeat_state.counter` | uint8 (2-bit) | Heartbeat counter, saturated at 3. | Tracks heartbeat cadence; saturation indicates sustained liveness. |

### 4.3 Logic Supervision

| Field | Type | Description | OEM Relevance |
|-------|------|-------------|---------------|
| `logic_error` | enum (`invalid_state`, `invalid_transition`, `unmapped_error`) | Logic evaluation error. | Indicates the component entered an unexpected state or made an illegal transition — a software logic fault. |
| `logic_state.current_state_index` | uint64 (56-bit) | Current state index. | Tracks the current state in a state machine; useful for understanding component behavior at failure. |
| `logic_state.monitor_status` | enum | `ok` (0), `invalid_state` (1), `invalid_transition` (2), `unmapped_error` (3) | Monitor status. | Indicates whether the logic monitor is healthy or detected an anomaly. |

---

## 5. Recovery State

| Field | Type | Allowed Values | Description | OEM Relevance |
|-------|------|----------------|-------------|---------------|
| `recovery_state` | string | — | Fallback run target name (`"fallback"` — hardcoded `IdentifierHash` in `process_group_manager.hpp`). Not a recovery state machine; it identifies the run target the system transitions to on recovery. | Shows which run target the system falls back to during recovery — critical for availability and safety analysis. |
| `recovery_action` | struct | `RestartAction { number_of_attempts: uint32, delay_before_restart_ms: uint32 }` or `SwitchRunTargetAction { run_target: string }` | The recovery action taken. `ready_recovery_action` uses `RestartAction`; `recovery_action` uses `SwitchRunTargetAction`. | Identifies the mitigation strategy; supports fleet-level policy evaluation. |

> **Note on `recovery_action` typing:** This is **not** a single runtime enum. In the source (`recovery_action_config.hpp`), `RestartAction` and `SwitchRunTargetAction` are two distinct configuration structs, each used in a different optional config slot (`ready_recovery_action: Optional<RestartAction>`, `recovery_action: Optional<SwitchRunTargetAction>`). Which action applies is determined by which slot is populated, not by a tagged union/enum. `recovery_action` is not currently part of the VEL evidence schema (only `recovery_state` is collected); it is documented here for completeness.
>
> **TODO:** If `recovery_action` is added to the schema, represent it as an object with a `type` discriminator (e.g., `restart` / `switch-run-target`), not a plain enum, since each action carries different parameters.

---

## 6. Alive Reporting (Bridge between HM and LM)

| Field | Type | Description | OEM Relevance |
|-------|------|-------------|---------------|
| `alive_status` | enum (`alive`, `failure`) | Whether the component reported alive or failure. | A component that stops reporting `alive` without reporting `failure` is presumed stalled — key for watchdog-style monitoring. |
| `identifier` | string | Process identity used for alive reporting (from `IDENTIFIER` env). | Maps alive notifications to a specific component instance. |

---

## 7. Correlation / Identity Fields

| Field | Type | Description | OEM Relevance |
|-------|------|-------------|---------------|
| `source_id` | string | Module identity (e.g., `score-lifecycle`). | Identifies the data source for traceability. |
| `workload_id` | string | Component identity (e.g., `launch-manager`, `health-monitor`). | Links evidence to a specific S-CORE workload. |
| `resource_id` | string | Resource identity (e.g., `component:<name>`, `run-target:<name>`, `monitor:<type>`). | Pinpoints the exact resource being monitored. |
| `execution_id` | string | Execution instance identifier. | Correlates multiple evidence records from the same execution. |
| `timestamp_ns` | uint64 | Event timestamp (nanoseconds, wall-clock). | Enables temporal correlation across all vehicle evidence. |

---

## 8. Mapping to VEL Evidence Types

Each field above maps to a normalized evidence type in the VEL package:

| Evidence Type | Source Field(s) |
|---------------|-----------------|
| `s-core.lifecycle.component.state` | `state` |
| `s-core.lifecycle.process-group.state` | `graph_state` |
| `s-core.lifecycle.process.exit_code` | `exit_code` |
| `s-core.lifecycle.process.pid` | `pid` |
| `s-core.lifecycle.run-target` | `run_target` |
| `s-core.lifecycle.health.supervision_status` | `supervision_status` |
| `s-core.lifecycle.health.failed_supervision_type` | `failed_supervision_type` |
| `s-core.lifecycle.health.deadline_error` | `deadline_error` |
| `s-core.lifecycle.health.heartbeat_error` | `heartbeat_error` |
| `s-core.lifecycle.health.logic_error` | `logic_error` |
| `s-core.lifecycle.health.deadline_violation` | `deadline_violation` |
| `s-core.lifecycle.health.failed_supervision_cycles` | `failed_supervision_cycles` |
| `s-core.lifecycle.health.alive_indication_count` | `alive_indication_count` |
| `s-core.lifecycle.process.process_execution_error` | `process_execution_error` |
| `s-core.lifecycle.process-group.process_group_id` | `process_group_id` |
| `s-core.lifecycle.recovery.recovery_state` | `recovery_state` |

> **Naming convention:** Evidence Type path segments that correspond directly to a source field name preserve the source field's underscores (e.g. `deadline_error`, `process_group_id`) rather than converting them to hyphens. Hyphenated segments (e.g. `process-group`, `health`) are structural category names, not source field names.

---

## 9. Notes

- **Read-only**: VEL only reads these fields; it never sends control commands (per `saf_req__vel__*`).
- **Enum stability**: Field names and enum values are aligned with the S-CORE lifecycle API to ensure a stable contract.
- **OEM filtering**: Fields marked as OEM-relevant are those that support vehicle-level monitoring, diagnostics, safety, and fleet analytics. Internal counters (e.g., raw FFI handle values) are intentionally excluded.
- **Collectability**: The S-CORE-specific fields in sections 1–5 are exposed through the S-CORE external monitor notification API (`comp_req__launch_man__ext_monitor_notify`).