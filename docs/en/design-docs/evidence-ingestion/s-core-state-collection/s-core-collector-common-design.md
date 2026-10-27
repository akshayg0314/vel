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
  fieldBindingProvider:
    # generic: built-in, config-driven binding (default -- no plugin, no new code).
    # plugin: independently built/deployed binding artifact, loaded by VEL core's
    #         PluginLoader via a stable C-ABI entry point, purely from this
    #         config (see source-collector-interface-design.md Section 3.6).
    #         Required only when the module's native API cannot be reached
    #         declaratively (Lifecycle is the current example -- see its
    #         collector config).
    kind: <generic | plugin>
    plugin:                                   # present only when kind: plugin
      library: plugins/<module>-field-bindings
      abiVersion: 1
      entryPoint: vel_plugin_create_binding_provider
      settings: {}
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

- Every S-CORE module collector is the **same generic `SCoreCollector`** configured from its `VehicleEvidenceCollector` YAML (`source_ref`, `collector_id`, schema/mapping refs) -- there is no per-module concrete collector subclass (e.g. no `LifecycleCollector` class) compiled into VEL core.
- Each S-CORE module's `IStateFieldBindingProvider` realization is selected by that module's `fieldBindingProvider` configuration (generic or plugin -- [source-collector-interface-design.md Section 3.6](../source-collector-interface-design.md#36-plugin-discovery-and-loading)): the **generic binding** (built into VEL core, config-driven, no new code) for modules whose state is reachable through existing declarative transport/provider capabilities; a **plugin** (an independently built and deployed binding artifact, loaded by the shared `PluginLoader` purely from configuration via a stable C-ABI entry point) only for modules like Lifecycle whose native S-CORE interface genuinely requires code that the generic binding cannot express. Either way, this is the only place any module-specific logic can exist, and it is never a VEL-core-compiled class.
- Enum-to-string and similar value conversions (e.g. Lifecycle's `kRunning` -> `"running"`) are declared in each field's `conversion` block in its `RawEvidenceSchema` and executed generically by `ConfigDrivenStateMapper` ([source-collector-interface-design.md Section 3.7](../source-collector-interface-design.md#37-field-value-conversion-common-collector-internals)) -- never hard-coded in a binding adapter or plugin.
- The shared `ConfigDrivenStateMapper` builds schema-conformant raw records from the module's `RawEvidenceSchema` config and that binding -- no per-module mapper code.
- **Schema Validator / Normalization Processor** -- shared evidence-processing components (see [vel_architectural_design_draft_eng.md Section 3](../../../vel_architectural_design_draft_eng.md#3-configuration-boundary)), driven by `RawEvidenceSchema` and `EvidenceNormalizationRule` configuration. They apply identically regardless of source module.
- **`CollectorDiagnostics`** -- shared error/diagnostic reporting per [source-collector-interface-design.md Section 5](../source-collector-interface-design.md#5-collector-error--diagnostic-reporting).

Consequently, adding a new S-CORE module collector requires: a new `RawEvidenceSchema`, a new `EvidenceNormalizationRule`, and a new `VehicleEvidenceCollector` config -- and, **only if the generic binding cannot reach that module's native interface**, one independently built/deployed plugin implementing `IStateFieldBindingProvider`. No new Mapper, Validator, Normalizer, or collector subclass is ever written, no hard-coded factory/registration list is updated, and VEL core is never rebuilt. This satisfies FR-VEL-012 (Platform Collector Separation) and FR-VEL-013/014/015 (configuration-driven input, output, and normalization) at the collector-implementation level, not only at the schema level.

See the [common collector class diagram](../../../../features/diagrams/Collector_common_class_diagram.puml) for the shared engine and plugin-loading model, and [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) Section 7 and the [Lifecycle collector class diagram](../../../../features/diagrams/lifecycle/Lifecycle_collector_class_diagram.puml) for its S-CORE Lifecycle realization (a plugin-backed binding, since Lifecycle's native interface requires it).

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
