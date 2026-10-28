# Lifecycle Collector Design

## Overview

This document defines the **module-specific** design for the S-CORE Lifecycle collector. It covers the API surface, observable state fields, source identity, and transport path specific to the Lifecycle module.

> **Common aspects** (collector I/O API, lifecycle, error & diagnostic reporting) are defined in the shared documents: [`../../source-collector-interface-design.md`](../../source-collector-interface-design.md) (common to every VEL collector) and [`../s-core-collector-common-design.md`](../s-core-collector-common-design.md) (S-CORE-specific naming conventions).

---

## 1. Collector Configuration

Reference: `lifecycle-collector.yaml` (`interfaces/score-evidence-layer-package/collectors/lifecycle-collector.yaml`)

```yaml
apiVersion: score.dev/v1alpha1
kind: VehicleEvidenceCollector
metadata:
  name: lifecycle-collector
  version: 1.0.0
spec:
  source:
    transport: s-core-api
    api: lifecycle-state-api
    messageType: ScoreLifecycleState
  fieldBindingProvider:
    # kind: plugin -- Lifecycle's native interface (two distinct mw::com
    # service connections: ILmControl pull + IExtMonitorNotify push) cannot
    # be expressed by VEL core's generic, declarative binding, so its
    # IStateFieldBindingProvider realization is an independently built and
    # deployed plugin, loaded by the shared PluginLoader purely from this
    # config (see source-collector-interface-design.md Section 3.6). This is
    # the ONLY place this selection is made -- VEL core has no Lifecycle-
    # specific collector subclass or hard-coded factory entry.
    kind: plugin
    plugin:
      library: plugins/lifecycle-field-bindings
      abiVersion: 1
      entryPoint: vel_plugin_create_binding_provider
      settings:
        transport: s-core-api
        api: lifecycle-state-api
  inputSchemaRef:
    name: score-lifecycle-state
    version: 1.0.0
    path: schemas/score-lifecycle-state-v1.0.0.yaml
  normalizationRuleRef:
    name: score-lifecycle-to-evidence
    version: 1.0.0
    path: mappings/score-lifecycle-to-evidence-v1.0.0.yaml
  outputSchemaRef:
    name: score-normalized-evidence
    version: 1.0.0
    path: ../vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml
  publication:
    transport: dds
    topic: ScoreNormalizedEvidence
    deliveryClass: reliable
  velHealth:
    topic: ScoreEvidenceLayerHealth
    heartbeatPeriod: 1s
  resourceLimits:
    inputQueueDepth: 256
    maximumMessageRate: 1000
```

Field-level binding and conversion for the module's state fields are declared in `schemas/score-lifecycle-state-v1.0.0.yaml` (not duplicated here). For example, `state`'s declaration is:

```yaml
- name: state
  path: $.state
  type: string
  required: true
  conversion:
    operation: enumMap
    sourceValueKind: enum
    values: {kIdle: idle, kStarting: starting, kRunning: running, kTerminating: terminating, kTerminated: terminated, kFailed: failed}
    unmappedValue: unknown
```

`LifecycleFieldBindings.read_field("state")` returns the raw native label observed from the source (e.g. `"kRunning"`); the shared `ConfigDrivenStateMapper` looks that value up in `conversion.values` and writes `"running"` to the raw record. If a future S-CORE release adds a `ProcessState` member VEL's configuration has not yet been updated for (e.g. a hypothetical `kDegraded`), `unmappedValue: unknown` means the mapper writes the canonical string `"unknown"` as the evidence string, rather than rejecting the record or preserving the raw label verbatim. See [source-collector-interface-design.md Section 3.7](../../source-collector-interface-design.md#37-field-value-conversion-common-collector-internals) for the full mechanism, which is shared by every S-CORE module, not Lifecycle-specific.

---

## 2. S-CORE API Surface

**Components:** Launch Manager, Health Monitor (PHM - Platform Health Management).

### 2.1 Observable State

- **Run target state**: which run target is active (string name, e.g. `"Startup"`, `"Off"`, `"Fallback"`).
- **Component states**: started, running, stopped (per the Lifecycle Interface).
- **Process health**: alive/liveliness, exit codes, resource usage.
- **Process runtime identity**: process identifier (`pid`), delivered via the external monitor notification capability (see Section 2.4) and surfaced in the schema as `details.pid`.
- **Ready conditions**: whether components have reached ready state (inferred from process state transitions).
- **Health Monitor status**: deadline monitor, logic monitor, heartbeat monitor status. This includes the supervision status (ok/failed, derived from whether any per-type error is present), the supervision type that failed (heartbeat/deadline/logic), the specific per-type errors (deadline_error, heartbeat_error, logic_error), deadline violations, failed supervision cycles, alive indication counts, process execution errors, process group identity, and recovery state.

### 2.2 Data Definitions (types, not callable APIs)

- **`ProcessState` enum** (`lifecycle/score/launch_manager/src/daemon/src/process_group_manager/process_state.hpp`):
  ```cpp
  enum class ProcessState : std::uint8_t {
      kIdle = 0,         // process in idle state.
      kStarting = 1,     // process in starting state.
      kRunning = 2,      // process in running state.
      kTerminating = 3,  // process in terminating state.
      kTerminated = 4,   // process in terminated state.
      kFailed = 5        // process failed to start.
  };
  ```

- **`GraphState` enum** (`lifecycle/score/launch_manager/src/daemon/src/process_group_manager/details/graph.hpp`):
  ```cpp
  enum class GraphState : std::uint8_t {
      kSuccess = 0U,        // Graph is not running and process group state is known
      kInTransition = 1U,   // Graph is running, process group state is in transition
      kAborting = 2U,       // Graph is running but has been aborted due to error
      kCancelled = 3U,      // Graph is running but has been cancelled because a new transition is pending
      kUndefinedState = 4U  // Graph is not running but process group state is not known
  };
  ```

The diagram below shows `ProcessState`, `GraphState`, the recovery-action config types, and the Health Monitor supervision types as a type model. `RestartAction` and `SwitchRunTargetAction` are two **distinct** config structs occupying two **different** optional slots (`ready_recovery_action` / `recovery_action`) — not a tagged union or single enum.

![Lifecycle type model](../../../../../features/assets/lifecycle/Lifecycle_type_model.svg)

[PlantUML source](../../../../../features/diagrams/lifecycle/Lifecycle_type_model.puml)

The class diagram below shows the VEL-side collector implementation classes. `SCoreCollector` (S-CORE-specific conventions, extending the abstract `SourceCollector`) is instantiated **directly from `lifecycle-collector.yaml`** with `source_ref=score.lifecycle`, `collector_id=lifecycle-collector`. The mapping/validation/diagnostics engine (`ConfigDrivenStateMapper`, `CollectorDiagnostics`) is **shared across every S-CORE module collector**, not Lifecycle-specific. Lifecycle's `IStateFieldBindingProvider` realization (`LifecycleFieldBindings`) is loaded at runtime by the shared `PluginLoader` as an **independently built and deployed plugin** (a binding artifact, selected purely by `lifecycle-collector.yaml`'s `fieldBindingProvider` config) because Lifecycle's native interface -- two distinct `mw::com` service connections -- cannot be expressed by VEL core's generic, declarative binding. `LifecycleFieldBindings` is the only Lifecycle-specific artifact in the whole design, and it is never compiled into VEL core. See Section 7 for how this keeps future modules config-driven, and `source-collector-interface-design.md` Sections 3.5-3.7 for the shared plugin-loading and field-conversion mechanisms this realization relies on.

![Lifecycle collector class diagram](../../../../../features/assets/lifecycle/Lifecycle_collector_class_diagram.svg)

[PlantUML source](../../../../../features/diagrams/lifecycle/Lifecycle_collector_class_diagram.puml)

### 2.3 Lifecycle API Fields

- **`run_target`** — exposed via the synchronous `ILmControl::get_active_run_target()` API (`lm_control/src/ilm_control.hpp`), which queries the Launch Manager for the name of the Run Target it has currently settled on. This is the **only pull/synchronous** field; every other field below is push-based.
- **`state`, `pid`, `graph_state`, `exit_code`, `process_execution_error`** — delivered through the external monitor notification capability (the `comp_req__launch_man__ext_monitor_notify` requirement) as part of its `ProcessMonitorNotify` payload. There is no separate pull API for any of these; `ProcessInfoNode` and `Graph`, which internally hold this state, are Launch Manager daemon-internal classes not reachable from an external VEL process.
- **Alive reporting** — the Health Monitor reports liveness to the Launch Manager through the Alive API (`report_alive()` / `report_failure()`). Note: this is a **write** path (the supervised process reports itself); VEL cannot use it to read another process's status.
- **Health Monitor supervision types** — the Health Monitor evaluates three supervision types (deadline, heartbeat, logic), each with its own error enum and state snapshot, delivered through the same external monitor notification capability.

### 2.4 Collector Access Pattern

```cpp
#include "score/mw/lifecycle/ilm_control.hpp"
#include "score/mw/lifecycle/ext_monitor/ext_monitor_notify.hpp"

// 1. run_target is the only field read synchronously, via the public ILmControl API.
score::Result<score::mw::lifecycle::RunTargetName> run_target = lm_control->get_active_run_target();

// 2. Every other field (state, pid, graph_state, exit_code, process_execution_error,
//    process_group_id, recovery_state, and all Health Monitor supervision fields) is
//    delivered through the external monitor notification capability
//    (comp_req__launch_man__ext_monitor_notify) as a ProcessMonitorNotify payload.
//    LifecycleFieldBindings registers for this payload via the shared
//    IStateFieldBindingProvider.subscribe(on_update) method (push-based), as opposed to
//    run_target which is read via read_field() (pull-based). There is no synchronous
//    getter for any of these fields on a type reachable from an external VEL process --
//    ProcessInfoNode and Graph, which hold this state internally, are Launch Manager
//    daemon-internal classes.
//
//    read_field()/subscribe() return the RAW native value exactly as observed (e.g. the
//    literal ProcessState enum label "kRunning") -- LifecycleFieldBindings performs NO
//    enum-to-string conversion itself. That conversion is applied generically by the
//    shared ConfigDrivenStateMapper, driven by each field's `conversion` block in
//    score-lifecycle-state-v1.0.0.yaml (see Section 2.2's config example and Section 4),
//    with unrecognized raw values mapped to the canonical string "unknown" (unmappedValue)
//    rather than rejected or preserved verbatim. This keeps LifecycleFieldBindings a thin,
//    source-only plugin.
//
//    Consistency guarantee: on each notification, LifecycleFieldBindings builds one
//    immutable snapshot struct from the full payload and atomically swaps a reference to
//    it in, rather than updating per-field mutable state. The ConfigDrivenStateMapper takes
//    a single snapshot once per collection cycle, before iterating push-origin fields, so
//    every read_field() call for push fields within one raw record resolves against the
//    same notification generation -- even if a new notification arrives mid-cycle. This
//    avoids a torn record where, e.g., supervision_status reflects one fault event while
//    failed_supervision_type reflects a later one. This is an internal implementation detail
//    of LifecycleFieldBindings; it requires no schema or mapper changes.

// 3. LifecycleFieldBindings exposes these values through the shared IStateFieldBindingProvider
//    interface (field_names() / read_field() / subscribe()); the shared ConfigDrivenStateMapper
//    (not Lifecycle-specific) then builds the raw state record generically by iterating the
//    score-lifecycle-state RawEvidenceSchema, pulling each field's value from the bindings, and
//    applying each field's configured `conversion` rule where declared.
```

The sequence below shows the full access-pattern interaction, including the `get_active_run_target()` read path and the external monitor notification callback path for the push-based fields:

![Lifecycle collector access pattern sequence](../../../../../features/assets/lifecycle/Lifecycle_collector_sequence.svg)

[PlantUML source](../../../../../features/diagrams/lifecycle/Lifecycle_collector_sequence.puml)

---

## 3. Source Identity

| Field | Value |
|-------|-------|
| `source_id` | `score-lifecycle` |
| `workload_id` | `launch-manager` or `health-monitor` |
| `resource_id` | `component:<name>` or `run-target:<name>` or `monitor:<type>` |

---

## 4. State Fields (Raw Evidence Schema)

The collector produces raw state records conforming to the `score-lifecycle-state` schema (`interfaces/score-evidence-layer-package/schemas/score-lifecycle-state-v1.0.0.yaml`):

| Field | Path | Type | Required | Conversion |
|-------|------|------|----------|------------|
| `message_id` | `$.message_id` | string | ✓ | – |
| `source_id` | `$.source_id` | string | ✓ | – |
| `workload_id` | `$.workload_id` | string | ✓ | – |
| `execution_id` | `$.execution_id` | string | ✓ | – |
| `resource_id` | `$.resource_id` | string | ✓ | – |
| `timestamp_ns` | `$.timestamp_ns` | uint64 | ✓ | – |
| `state` | `$.state` | string | ✓ | `enumMap`, `unmappedValue: unknown` |
| `previous_state` | `$.previous_state` | string | – | `enumMap`, `unmappedValue: unknown` |
| `transition_time_ns` | `$.transition_time_ns` | uint64 | – | – |
| `run_target` | `$.details.run_target` | string | – | – |
| `exit_code` | `$.details.exit_code` | int32 | – | – |
| `pid` | `$.details.pid` | uint32 | – | – |
| `graph_state` | `$.details.graph_state` | string | – | `enumMap`, `unmappedValue: unknown` |
| `supervision_status` | `$.details.supervision_status` | string | – | `enumMap`, `unmappedValue: unknown` |
| `failed_supervision_cycles` | `$.details.failed_supervision_cycles` | uint32 | – | – |
| `failed_supervision_type` | `$.details.failed_supervision_type` | string | – | `enumMap`, `unmappedValue: unknown` |
| `deadline_error` | `$.details.deadline_error` | string | – | `enumMap`, `unmappedValue: unknown` |
| `heartbeat_error` | `$.details.heartbeat_error` | string | – | `enumMap`, `unmappedValue: unknown` |
| `logic_error` | `$.details.logic_error` | string | – | `enumMap`, `unmappedValue: unknown` |
| `deadline_violation` | `$.details.deadline_violation` | bool | – | – |
| `alive_indication_count` | `$.details.alive_indication_count` | uint32 | – | – |
| `process_execution_error` | `$.details.process_execution_error` | uint32 | – | – |
| `process_group_id` | `$.details.process_group_id` | string | – | – |
| `recovery_state` | `$.details.recovery_state` | string | – | – |

> **Why `string`, not `enum`:** these fields carry interface-specific state/error values that originate from the S-CORE Lifecycle module's own enums (`ProcessState`, `GraphState`, `SupervisionStatus`, `FailedSupervisionType`, `DeadlineError`, `HeartbeatError`, `LogicError`). VEL does not own or enumerate these value sets as a closed schema constraint (`allowedValues`) — a future S-CORE release may add a new member to any of them. Instead, each field's `conversion` block (declared once in `schemas/score-lifecycle-state-v1.0.0.yaml`, shown in Section 2.2's configuration example) maps the **currently known** raw native labels to their evidence string form, and `unmappedValue: unknown` guarantees an unrecognized raw value is surfaced as the canonical string `"unknown"` — never rejected, and never preserved verbatim. See [source-collector-interface-design.md Section 3.7](../../source-collector-interface-design.md#37-field-value-conversion-common-collector-internals) for the shared mechanism that executes this.

---

## 5. Transport Path

The static view below shows where the Lifecycle Collector sits in the VEL Evidence Ingestion architecture and its interactions with the Schema Validator, Normalization Processor, and the DDS publication topic:

![Lifecycle collector static view](../../../../../features/assets/lifecycle/Lifecycle_collector_static_view.svg)

[PlantUML source](../../../../../features/diagrams/lifecycle/Lifecycle_collector_static_view.puml)

![Lifecycle collector transport flow](../../../../../features/assets/lifecycle/Lifecycle_collector_transport_flow.svg)

[PlantUML source](../../../../../features/diagrams/lifecycle/Lifecycle_collector_transport_flow.puml)

---

## 6. S-CORE API Boundary

VEL can only collect data that S-CORE exposes through a **stable, public, external-facing API**. For the Lifecycle module:

- **Allowed**: Reading `run_target` via `ILmControl::get_active_run_target()`, and `state`, `pid`, `ProcessState`, `GraphState`, `exit_code`, `process_execution_error`, `process_group_id`, `recovery_state`, and Health Monitor supervision status (supervision_status, failed_supervision_type, deadline_error, heartbeat_error, logic_error, deadline_violation, failed_supervision_cycles, alive_indication_count) via the external monitor notification capability.
- **Not allowed**: Sending control commands (start/stop/restart processes), invoking `report_alive()` / `report_failure()` (write path), or any lifecycle/process/hardware control operations.

This satisfies `saf_req__vel__*` (no lifecycle/process/container/hardware control, no coordination, no OEM decisions).

---

## 7. Extensibility: Adding a New S-CORE Module

The mapping/validation engine shown in Section 2.2 is **shared infrastructure**, not Lifecycle-specific, and adding a new S-CORE module collector does **not** mean adding a new module-specific class to VEL core. It means:

1. A new `RawEvidenceSchema` config (`schemas/score-<module>-state-v1.0.0.yaml`), with a `conversion` block on any interface-specific state/error field (Section 4 and `source-collector-interface-design.md` Section 3.7) -- never a closed `enum`/`allowedValues` for such fields.
2. A new `EvidenceNormalizationRule` config (`mappings/score-<module>-to-evidence-v1.0.0.yaml`).
3. A new `VehicleEvidenceCollector` config (`collectors/<module>-collector.yaml`), including a `fieldBindingProvider` block that selects how the module's `IStateFieldBindingProvider` is realized:
   - **`kind: generic`** (the default, preferred path) -- the module's state is reachable through VEL core's built-in, config-driven generic binding (reusing existing transport/provider capabilities declared purely in configuration). **This requires no new code at all.**
   - **`kind: plugin`** -- only if the module's native interface genuinely cannot be expressed by the generic binding (as is the case for Lifecycle, whose state requires two distinct native `mw::com` service connections with source-specific payload decoding). In this case, implement the module's `IStateFieldBindingProvider` realization as its own **independently built and deployed binding artifact**, versioned separately from VEL core, and point the collector config's `fieldBindingProvider.plugin` block at it. The shared `PluginLoader` (`source-collector-interface-design.md` Section 3.6) discovers and loads it **purely from that configuration** -- at no point is VEL core source modified, a collector subclass added, a hard-coded factory/registration list updated, or VEL core rebuilt.

No new Mapper, Validator, Normalizer class, or `SCoreCollector` subclass is ever written per module -- `SCoreCollector` itself is one generic class, instantiated per module purely from its `VehicleEvidenceCollector` config, and `ConfigDrivenStateMapper`, the Schema Validator, and the Normalization Processor are built once and reused by every S-CORE module collector regardless of which binding kind that module uses.

**Lifecycle's own realization is the `kind: plugin` case**, not the default: its `LifecycleFieldBindings` plugin is required specifically because of its native dual-connection interface (Section 2.4), and is the only Lifecycle-specific artifact in this entire design -- it is built and deployed independently of VEL core, as described in `lifecycle-collector.yaml`'s `fieldBindingProvider` block (Section 1).

## 8. Traceability

| Item | Reference |
|------|-----------|
| Design | Lifecycle Collector Design |
| Requirements | FR-VEL-003, AOU-VEL-002 |
| Collector config | `lifecycle-collector.yaml` (`interfaces/score-evidence-layer-package/collectors/lifecycle-collector.yaml`) |
| Input schema | `score-lifecycle-state-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/schemas/score-lifecycle-state-v1.0.0.yaml`) |
| Normalization mapping | `score-lifecycle-to-evidence-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/mappings/score-lifecycle-to-evidence-v1.0.0.yaml`) |