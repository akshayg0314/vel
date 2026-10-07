# Design: S-CORE Module Input Handling

> **Related:** FR-VEL-003 (S-CORE Module State Collection), AOU-VEL-002, FR-VEL-013 (Input Interface Configuration)
> **See also:** [s-core-modules](s-core-modules/lifecycle-input-design.md) — module-specific input designs. [Collectors](../../evidence-ingestion/s-core-state-collection/collectors/lifecycle-collector-design.md) — how a collector reads this state from S-CORE APIs.

## 1. Purpose

This document defines how VEL's Input Interface Contract handles **S-CORE module state** as a raw input source. It covers the S-CORE runtime modules that expose observable state through S-CORE APIs, as tracked by the S-CORE reference integration workspace.

This design is a concrete application of the input schema structure: it defines the `RawEvidenceSchema` instances that S-CORE module sources must conform to, using the same common required fields and contract structure. It complements the vendor-NPU example by modeling S-CORE modules as a distinct source category with state-based data.

## 2. Design Position

Per the VEL architecture:

- **FR-VEL-003**: "The system shall collect configured S-CORE module state information through S-CORE APIs exposed by the deployed environment."
- **AOU-VEL-002**: "Each deployment environment provides the S-CORE APIs and operating-system access required by its configured collectors."
- **FR-VEL-013**: "The system shall support configuration-driven input interfaces, so that adding a new source does not require changes to the common Evidence Layer."
- VEL is an **evidence producer**, not a coordinator/controller. It only **reads** S-CORE module state; it never invokes control operations (satisfies `saf_req__vel__*` — no lifecycle/process/container/hardware control, no coordination, no OEM decisions).
- S-CORE data is **commonly state-based** (per the Evidence Content Map, architecture Section 7).

This means S-CORE module inputs are modeled as **state observations** read through S-CORE APIs, not as command/control interactions.

## 3. S-CORE Module Source Model

### 3.1 Source categories

VEL distinguishes three source categories, each with its own Input Interface Definition:

| Category | Example Sources | Data Nature |
| --- | --- | --- |
| Runtime metrics | CPU, GPU, NPU utilization | Unit- and range-oriented |
| Hardware data | Sensor states, device status | Measurement or operational state |
| **S-CORE module state** | Lifecycle | **State-based** |

### 3.2 S-CORE module inventory (VEL-relevant subset — Lifecycle)

The following S-CORE module is a **VEL-relevant source** — it exposes observable state through real S-CORE APIs and is pinned in the reference integration `known_good.json`:

| Module | Observable State (examples) | S-CORE API Used by VEL Collector | Source ID |
| --- | --- | --- | --- |
| **Lifecycle** | Launch Manager: component states (started/running/stopped), run targets, ready states, process health, exit codes. Health Monitor: deadline/logic/heartbeat monitor status. | Launch Manager external monitor notification (`comp_req__launch_man__ext_monitor_notify`), `ProcessState` enum, `GraphState` enum | `score-lifecycle` |

> **Accessibility note (Lifecycle):** VEL can only collect data exposed through a stable, public, external-facing API. All lifecycle fields — including `pid`, `exit_code`, and `process_execution_error` — are exposed through the S-CORE external monitor notification API (`comp_req__launch_man__ext_monitor_notify`). See [lifecycle-input-design.md](s-core-modules/lifecycle-input-design.md) for module-specific detail.

### 3.3 S-CORE module source identity

Each S-CORE module is a distinct source with a stable identity, conforming to the common required fields:

| Field | Example | Description |
| --- | --- | --- |
| `message_id` | `msg-0001` | Unique identifier for the raw source message (traceability). |
| `source_id` | `score-lifecycle` | The S-CORE module identifier. |
| `workload_id` | `launch-manager` | The workload/process exposing the module state. |
| `execution_id` | `exec-42` | The execution instance; also used as correlation identifier. |
| `resource_id` | `component:networking` | The specific resource/entity observed within the module. |
| `timestamp_ns` | `1710000000000000000` | The observation timestamp in nanoseconds. |

### 3.4 S-CORE module state observation

S-CORE module state is observed as a **state record** with a common structure, extending the common required fields with module-specific state fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `message_id` | string | ✓ | Unique message identifier. |
| `source_id` | string | ✓ | S-CORE module identifier. |
| `workload_id` | string | ✓ | Workload/process exposing the state. |
| `execution_id` | string | ✓ | Execution instance / correlation identifier. |
| `resource_id` | string | ✓ | Resource/entity observed. |
| `timestamp_ns` | uint64 | ✓ | Observation timestamp. |
| `state` | string | ✓ | The observed state (module-specific). |
| `previous_state` | string | – | The previous state, when a transition is observed. |
| `transition_time_ns` | uint64 | – | The time of the state transition. |
| `details` | object | – | Module-specific detail fields (e.g., run target, component, parameter set). |

This generic state model allows VEL to collect state from any S-CORE module through configuration, without hard-coding module-specific logic.

## 4. Per-Module Design Documents

Each S-CORE module's example raw state record, `RawEvidenceSchema`, Normalization Mapping, and source-specific notes (verified against `lifecycle/`) are documented separately, to keep each module's contract independently reviewable and to avoid duplicating the canonical interface artifacts (which live in `interfaces/score-evidence-layer-package/`):

| Module | Design Document |
| --- | --- |
| Lifecycle | [s-core-modules/lifecycle-input-design.md](s-core-modules/lifecycle-input-design.md) |

Each per-module document follows the same structure: Observable State, Source-Specific Notes (verified against source), Source Identity example, Example Raw State Record, `RawEvidenceSchema`/Normalization Mapping references, and Traceability.

## 5. Cross-Cutting Notes (apply to all modules)

- **Evidence Type naming:** the final path segment of an `evidenceType` preserves the exact source field name's underscores (e.g. `deadline_error`, `process_group_id`, `supervision_status`) rather than converting `_` to `-`. Structural/category path segments (e.g. `process-group`, `parameter-set`, `health`, `message-passing`) may keep hyphens since they aren't literal field names.
- **`previous_state`** in every module's `RawEvidenceSchema` is `type: enum` with the same `allowedValues` as that module's own `state` field, not `type: string` — a transition can only be between the states the module actually has.
- **"No error" sentinel values:** optional error/status fields generally have no defined "no error" value in the underlying S-CORE enum. When no error is present, the field should be **omitted** from the raw record rather than set to an invented value like `none` that doesn't appear in the schema's `allowedValues`.
- **TODO:** Extend cross-cutting notes for other modules (e.g., config_error, kvs_last_error_code, connection_stop_reason handling) as they are added.

## 6. Normalization Mapping Pattern

The S-CORE module state is normalized into Vehicle Evidence using the standard Normalization Mapping. All modules follow the same pattern — `commonMappings` project the shared identity/timestamp fields, and `evidenceMappings` project each `details.*` field to its own `evidenceType` — as shown in the canonical `score-lifecycle-to-evidence-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/mappings/score-lifecycle-to-evidence-v1.0.0.yaml`).

The clock domain is declared **per-source** in the Input Interface Definition (default `monotonic`, may be `wall-clock`). The S-CORE module mappings use `wall-clock` as the default because S-CORE events commonly carry wall-clock timestamps. Individual collector deployments MAY configure `monotonic` instead when reading hardware/steady clock sources, by overriding the `time.clockDomain` default in the `EvidenceNormalizationRule` for that source.

## 7. Configuration-Driven Source Addition

Adding a new S-CORE module as a VEL source requires only configuration, following the standard pattern:

1. Define the **Input Interface Definition** (`RawEvidenceSchema`) for the module's state.
2. Define the **Normalization Mapping** (`EvidenceNormalizationRule`) for the module's state.
3. Implement a **platform-specific collector** that reads the module's state via its S-CORE API (see [lifecycle-collector-design.md](../../evidence-ingestion/s-core-state-collection/collectors/lifecycle-collector-design.md)).

![Configuration-driven source addition](../../../../features/assets/Configuration_driven_source_addition.svg)

[PlantUML source](../../../../features/diagrams/Configuration_driven_source_addition.puml)

No changes to common VEL processing behavior are required. This satisfies FR-VEL-012 (Platform Collector Separation) and FR-VEL-013/014/015 (configuration-driven interfaces).

## 8. Traceability

| Item | Reference |
| --- | --- |
| Design | S-CORE Module Input Handling |
| Requirements | FR-VEL-003, FR-VEL-012, FR-VEL-013, AOU-VEL-002 |
| Reference architecture | [`vel_architectural_design_draft_eng.md`](../../../vel_architectural_design_draft_eng.md) |
| Reference S-CORE module | `lifecycle/` |