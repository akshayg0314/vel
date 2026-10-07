# Mapping Rule Model for Units, Enums, and States

> **Related:** FR-VEL-005 (Source Normalization), FR-VEL-015 (Normalization Mapping Configuration), STKH-VEL-005, STKH-VEL-015
> **Reference example:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

## 1. Purpose

This document defines the **Normalization Mapping** configuration artifact: the rule model that transforms source-specific representations into normalized Vehicle Evidence. It covers unit conversion, enum/state mapping, and the structure of mapping rules.

## 2. Design Position

In the VEL architecture, the normalization mapping is configuration-driven. Each source has its own Normalization Mapping that references an Input Interface Definition and an Output Evidence Definition. This document specifies the data representation and configuration format only.

## 3. Mapping Rule Model

### 3.1 Normalization Mapping as a Configuration Artifact

The normalization mapping is defined in configuration, not hard-coded. Each source has its own Normalization Mapping that references an Input Interface Definition and an Output Evidence Definition.

### 3.2 Canonical Contract Kind: EvidenceNormalizationRule

The normalization contract uses the `EvidenceNormalizationRule` kind, mirroring the reference example:

- [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

### 3.3 Mapping rule model structure

A Normalization Mapping has these sections:

| Section | Purpose |
| --- | --- |
| `compatibleInputSchemas` | The input schemas this rule applies to (with version ranges). |
| `outputSchema` | The output evidence schema this rule produces. |
| `defaults` | Default values for output fields (evidence domain, quality, time, traceability). |
| `commonMappings` | Field-to-field mappings applied to every evidence record (identity, time, traceability). |
| `evidenceMappings` | Source-specific mappings that produce one or more evidence records (value/state/event mappings). |
| `failureHandling` | How mapping failures are handled. |

### 3.4 Transform operations

The rule model supports transform operations for unit conversion, enum/state mapping, and timestamp handling:

| Operation | Description | Example |
| --- | --- | --- |
| `scale` | Multiply a numeric value by a factor (unit conversion). | `inference_latency_us` × 0.001 → ms. |
| `enumMap` | Map an enum value to a normalized state. | `device_state: 2` → `degraded`. |
| `timestamp` | Convert a timestamp to the canonical form. | `timestamp_ns` with clock domain. |
| `lookup` | Look up a value in a table to produce evidence. | `vendor_error_code: 0xA17` → `accelerator.execution.timeout`. |

### 3.5 Mapping rule categories

The rule model distinguishes three mapping categories:

1. **Common mappings** — applied to every evidence record (identity, timestamp, traceability). These are the `commonMappings` section.
2. **Evidence mappings** — source-specific mappings that produce evidence records. Each mapping has:
   - `id` — unique identifier.
   - `when` — condition for the mapping to apply. Supports `fieldExists` (field is present) and `fieldEquals` (field has a specific value). Value-based conditions are needed for state/error mappings.
   - `output` — evidence type and normalized class.
   - `value` — source path, target path, and transform.
   - `constants` — fixed output values.
   - `lookup` — optional lookup table.
3. **Defaults** — default values applied when no mapping condition matches.

### 3.6 Unit conversion model

Unit conversion is expressed as a `scale` transform with a `factor`. The source unit and target unit are implicit in the mapping context (e.g., source `inference_latency_us` scaled to target `observation.value` with `unit: ms` constant). A `scale` factor is sufficient for unit conversions; offset-based conversions (e.g., temperature) are not supported.

## 4. Configuration Format

### 4.1 Example Normalization Mapping

```yaml
apiVersion: score.dev/v1alpha1
kind: EvidenceNormalizationRule
metadata:
  name: vendor-a-npu-to-score-evidence
  version: 3.1.0
spec:
  compatibleInputSchemas:
    - {name: vendor-a-npu-status, versionRange: ">=1.2.0 <2.0.0"}
  outputSchema: {name: score-normalized-evidence, version: 1.0.0}
  defaults:
    evidenceDomain: accelerator
    quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
    time: {clockDomain: monotonic}
    traceability: {inputSchemaVersion: "1.2.0", normalizationRuleVersion: "3.1.0"}
  commonMappings:
    - {sourcePath: $.source_id, targetPath: $.identity.sourceId, required: true}
    - {sourcePath: $.workload_id, targetPath: $.identity.workloadId, required: true}
    - {sourcePath: $.execution_id, targetPath: $.identity.executionInstanceId, required: true}
    - {sourcePath: $.resource_id, targetPath: $.identity.resourceId, required: true}
    - {sourcePath: $.execution_id, targetPath: $.identity.correlationId, required: true}
    - {sourcePath: $.message_id, targetPath: $.traceability.sourceMessageId, required: true}
    - sourcePath: $.timestamp_ns
      targetPath: $.time.eventTimestampNs
      required: true
      transform: {operation: timestamp, sourceUnit: ns, clockDomain: monotonic}
  evidenceMappings:
    - id: inference-latency
      when: {fieldExists: $.inference_latency_us}
      output: {evidenceType: workload.execution.latency}
      value:
        sourcePath: $.inference_latency_us
        targetPath: $.observation.value
        transform: {operation: scale, factor: 0.001}
      constants:
        - {targetPath: $.observation.unit, value: ms}
    - id: npu-device-state
      when: {fieldExists: $.device_state}
      output: {evidenceType: resource.operational.state}
      value:
        sourcePath: $.device_state
        targetPath: $.observation.state
        transform:
          operation: enumMap
          values: {0: available, 1: busy, 2: degraded, 3: unavailable}
          unmappedValue: unknown
```

## 5. Behavioral Semantics

- The Normalization Mapping references a compatible input schema and produces records conforming to the output schema.
- Common mappings are applied to every evidence record; evidence mappings produce one or more evidence records per source message.
- A single source message can produce **multiple** evidence records (e.g., one for latency, one for device state, one for a vendor error). This is the **standard model**: each evidence mapping produces a separate record, and all records from the same source message share the same correlation identifier. Records are published independently (no atomicity guarantee); consumers group them via the correlation identifier. Ordering across records is not guaranteed, but each record is independently valid.
- Transform operations define how source values are converted (scale, enumMap, timestamp, lookup). These four operations are the complete set supported by the rule model.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-005, FR-VEL-015, STKH-VEL-005, STKH-VEL-015 |
| Reference example | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) |
