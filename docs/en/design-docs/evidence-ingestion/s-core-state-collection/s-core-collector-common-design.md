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

S-CORE module collectors populate the common collector configuration (see [source-collector-interface-design.md Section 6](../source-collector-interface-design.md#6-common-collector-configuration)) with `transport: s-core-api`:

```yaml
collector:
  kind: VehicleEvidenceCollector
  version: 1.0.0
  collector_id: "<module>-collector"
  source:
    transport: s-core-api
    api: "<module-specific-api>"
  collection:
    trigger: periodic
    interval_seconds: 60
    timeout_seconds: 10
    retry:
      max_retries: 3
      backoff_seconds: 2
  diagnostics:
    rate_limit_per_minute: 10
  lifecycle:
    auto_start: true
```

> **S-CORE-specific error category:** `SOURCE_UNAVAILABLE` (defined generically in [source-collector-interface-design.md Section 5.1](../source-collector-interface-design.md#51-error-categories)) means, for an S-CORE module collector, that the S-CORE module's API is not reachable or not responding (e.g., the Launch Manager's external monitor notification channel). Likewise `AUTHENTICATION_ERROR` and `TIMEOUT` apply to the S-CORE API call specifically.

## 3. Relationship to Per-Module Design Docs

| Aspect | Document |
|--------|----------|
| Collector I/O API | [source-collector-interface-design.md](../source-collector-interface-design.md) (common, all source categories) |
| Collector lifecycle | [source-collector-interface-design.md](../source-collector-interface-design.md) (common, all source categories) |
| Error & diagnostic reporting | [source-collector-interface-design.md](../source-collector-interface-design.md) (common, all source categories) |
| S-CORE naming conventions & collector configuration | **This document** |
| Lifecycle module API surface / state fields / transport path | [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) |

## 4. Traceability

| Item | Reference |
| --- | --- |
| Design | S-CORE Collector Common Design |
| Common interface applied | [source-collector-interface-design.md](../source-collector-interface-design.md) |
| Requirements | FR-VEL-003, FR-VEL-012 |
