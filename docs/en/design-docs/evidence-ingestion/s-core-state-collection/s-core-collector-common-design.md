# S-CORE Collector Common Design

> **See also:** [../source-collector-interface-design.md](../source-collector-interface-design.md) — the **common** Source Collector Interface this document applies: collector I/O API, lifecycle, and error/diagnostic reporting are defined there once, for every source category (runtime, hardware, and S-CORE modules alike).

## Overview

S-CORE module collectors (currently: Lifecycle) conform to the common [Source Collector Interface](../source-collector-interface-design.md) — they do not define their own I/O API, lifecycle state machine, or diagnostic taxonomy. This document covers only the **S-CORE-specific conventions** layered on top of that common interface:

- How `source_ref` and `collector_id` are named for S-CORE module collectors.
- How the common collector configuration's `source` block is populated for the `s-core-api` transport.
- Where each S-CORE module's module-specific design (API surface, state fields, transport path) is documented.

## 1. S-CORE Naming Conventions

### 1.1 `source_ref`

S-CORE module collectors populate the common I/O API's `source_ref` parameter (see [source-collector-interface-design.md Section 3.2](../source-collector-interface-design.md#32-input-contract)) using the `score.<module>` convention:

| Module | `source_ref` |
| --- | --- |
| Lifecycle | `score.lifecycle` |

### 1.2 `collector_id`

S-CORE module collectors use the `<module>-collector` naming convention (e.g., `lifecycle-collector`), matching the `metadata.name` field in each module's `VehicleEvidenceCollector` configuration YAML (see `interfaces/score-evidence-layer-package/collectors/`).

## 2. S-CORE Collector Configuration

S-CORE module collectors populate the common collector configuration (see [source-collector-interface-design.md Section 6](../source-collector-interface-design.md#6-common-collector-configuration)) conforming to the `VehicleEvidenceCollector` resource schema with `transport: s-core-api`:

```yaml
apiVersion: score.dev/v1alpha1
kind: VehicleEvidenceCollector
metadata:
  name: <module>-collector
  version: 1.0.0
spec:
  source:
    transport: s-core-api
    api: <module>-state-api
    messageType: Score<Module>State
  inputSchemaRef:
    name: score-<module>-state
    version: 1.0.0
    path: schemas/score-<module>-state-v1.0.0.yaml
  normalizationRuleRef:
    name: score-<module>-to-evidence
    version: 1.0.0
    path: mappings/score-<module>-to-evidence-v1.0.0.yaml
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

> **S-CORE-specific error category:** `SOURCE_UNAVAILABLE` (defined generically in [source-collector-interface-design.md Section 5.1](../source-collector-interface-design.md#51-error-categories)) means, for an S-CORE module collector, that the S-CORE module's API is not reachable or not responding (e.g., the Launch Manager's external monitor notification channel). Likewise `AUTHENTICATION_ERROR` and `TIMEOUT` apply to the S-CORE API call specifically.

## 3. Shared State-Mapping Engine

S-CORE module collectors do not each implement their own raw-record mapping, validation, or diagnostics logic. The mapping/validation engine is **truly common infrastructure** -- defined once in [source-collector-interface-design.md Section 3.5](../source-collector-interface-design.md#35-config-driven-state-mapping-common-collector-internals) (`IStateFieldBindingProvider`, `ConfigDrivenStateMapper`) -- not S-CORE-specific, even though it was introduced alongside the Lifecycle collector. S-CORE module collectors reuse it exactly as any other source category would:

- Each S-CORE module implements exactly one thin `<Module>FieldBindings` adapter against the shared `IStateFieldBindingProvider` interface, wrapping that module's native S-CORE API calls (e.g. `LifecycleFieldBindings`). This is the only module-specific code.
- The shared `ConfigDrivenStateMapper` builds schema-conformant raw records from the module's `RawEvidenceSchema` config and that adapter -- no per-module mapper code.
- **Schema Validator / Normalization Processor** -- shared evidence-processing components (see [vel_architectural_design_draft_eng.md Section 3](../../../vel_architectural_design_draft_eng.md#3-configuration-boundary)), driven by `RawEvidenceSchema` and `EvidenceNormalizationRule` configuration. They apply identically regardless of source module.
- **`CollectorDiagnostics`** -- shared error/diagnostic reporting per [source-collector-interface-design.md Section 5](../source-collector-interface-design.md#5-collector-error--diagnostic-reporting).

Consequently, adding a new S-CORE module collector requires: a new `RawEvidenceSchema`, a new `EvidenceNormalizationRule`, a new `VehicleEvidenceCollector` config, and a thin `<Module>FieldBindings` adapter -- no new Mapper, Validator, or Normalizer class. This satisfies FR-VEL-012 (Platform Collector Separation) and FR-VEL-013/014/015 (configuration-driven input, output, and normalization) at the collector-implementation level, not only at the schema level.

See the [common collector class diagram](../../../../features/diagrams/Collector_common_class_diagram.puml) for the shared engine, and [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) Section 7 and the [Lifecycle collector class diagram](../../../../features/diagrams/lifecycle/Lifecycle_collector_class_diagram.puml) for its S-CORE Lifecycle realization.

## 4. Relationship to Per-Module Design Docs

| Aspect | Document |
|--------|----------|
| Collector I/O API | [source-collector-interface-design.md](../source-collector-interface-design.md) (common, all source categories) |
| Collector lifecycle | [source-collector-interface-design.md](../source-collector-interface-design.md) (common, all source categories) |
| Error & diagnostic reporting | [source-collector-interface-design.md](../source-collector-interface-design.md) (common, all source categories) |
| S-CORE naming conventions & collector configuration | **This document** |
| Lifecycle module API surface / state fields / transport path | [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) |

## 5. Traceability

| Item | Reference |
| --- | --- |
| Design | S-CORE Collector Common Design |
| Common interface applied | [source-collector-interface-design.md](../source-collector-interface-design.md) |
| Requirements | FR-VEL-003, FR-VEL-012 |
