# Mapping Quality Outputs

> **Related:** FR-VEL-009 (Evidence Quality Metadata), STKH-VEL-009
> **Reference example:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

## 1. Purpose

This document defines how the Normalization Mapping produces **mapping quality** outputs. Mapping quality describes how faithfully a source value was mapped to its normalized representation.

## 2. Design Position

In the VEL architecture, mapping quality is one of the four Evidence Quality fields in the normalized Vehicle Evidence record. It is assigned by the Normalization Processor based on the mapping rule applied. This document specifies the data representation and configuration format only.

## 3. Mapping Quality Model

### 3.1 Mapping quality is a quality metadata field

Mapping quality is one of the four Evidence Quality fields. It is expressed as an enum with three values:

| Value | Description |
| --- | --- |
| `exact` | The source value was mapped directly/exactly with no loss of fidelity. |
| `approximate` | The source value was mapped with a conversion that may introduce minor imprecision (e.g., unit scaling, rounding). |
| `unmapped` | The source value could not be mapped to a normalized representation (e.g., unknown enum, unmapped error code). |

This three-value enum is the complete set for mapping quality. No intermediate values are used; a value is either exactly mapped, approximately mapped, or unmapped. Partial mapping is not supported.

### 3.2 How mapping quality is assigned

Mapping quality is assigned by the Normalization Processor based on the mapping rule applied:

| Mapping situation | mappingQuality |
| --- | --- |
| Direct field copy (common mapping) | `exact` |
| Unit conversion via `scale` | `approximate` (or `exact` if lossless) |
| Enum/state mapping with a matched value | `exact` |
| Enum/state mapping with `unmappedValue` (unknown) | `unmapped` |
| Lookup with a matched key | `exact` |
| Lookup with `onUnmapped` fallback | `unmapped` |
| Invalid value emitted (`emit-invalid-evidence`) | `unmapped` (and `validity: invalid`) |

### 3.3 Defaults

The Normalization Mapping declares default quality values that apply when no quality-affecting condition is detected:

```yaml
defaults:
  quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
```

### 3.4 Mapping quality output propagation

The mapping quality value is written to `$.quality.mappingQuality` in the normalized Vehicle Evidence record. It is set by the Normalization Processor during transformation and can be overridden by the Quality Evaluator when additional quality conditions are detected. The Quality Evaluator MAY change the value to `unmapped` or `approximate` if the evaluation reveals quality issues not captured by the mapping rule itself (e.g., a value that maps exactly but is semantically unreliable).

### 3.5 Distinction from other quality fields

- **mappingQuality** describes the fidelity of the **transformation** (how the source value became the normalized value).
- **validity** describes whether the observation is valid.
- **freshness** describes timeliness.
- **completeness** describes whether all expected data is present.

These are independent dimensions; a value can be `exact` mapping but `stale` freshness, for example.

### 3.6 exact vs. approximate threshold

A mapping is **`exact`** when the transformation preserves the value with no loss of fidelity (direct copy, lossless unit scaling, or a matched enum/lookup entry). A mapping is **`approximate`** when the transformation introduces imprecision, such as rounding during unit conversion or a scaled value that cannot be represented exactly. The default is `exact` for direct mappings; `approximate` is used only when the transformation is known to be lossy.

## 4. Configuration Format

### 4.1 Default quality in Normalization Mapping

```yaml
defaults:
  evidenceDomain: accelerator
  quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
  time: {clockDomain: monotonic}
  traceability: {inputSchemaVersion: "1.2.0", normalizationRuleVersion: "3.1.0"}
```

### 4.2 Mapping quality in failure handling

```yaml
failureHandling:
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
```

### 4.3 Mapping quality in onUnmapped fallback

```yaml
onUnmapped:
  evidenceType: accelerator.vendor-event
  normalizedClass: unmapped
  mappingQuality: unmapped
  preserveOriginalValue: true
```

### 4.4 Mapping quality in the output record

```json
{
  "quality": {
    "validity": "valid",
    "freshness": "current",
    "completeness": "complete",
    "mappingQuality": "exact"
  }
}
```

## 5. Behavioral Semantics

- Every normalized Vehicle Evidence record carries a `mappingQuality` value; it is a required field in the output record.
- Mapping quality is set based on the mapping rule applied and the success of the transformation.
- `unmapped` mapping quality signals to consumers that the normalized value may not faithfully represent the source value.
- Mapping quality is assigned per record. Batch-level aggregation of mapping quality is optional and may be provided by the deployment; it does not replace the per-record value.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-009, STKH-VEL-009 |
| Reference example | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) |
