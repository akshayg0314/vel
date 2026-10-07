# Evidence Quality Metadata Fields

> **Related:** FR-VEL-009 (Evidence Quality Metadata), STKH-VEL-009
> **Reference example:** [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

## 1. Purpose

This document defines the **Evidence Quality** metadata fields attached to each normalized Vehicle Evidence record (or batch). Evidence Quality describes the observation's validity, freshness, completeness, and mapping quality. It does not interpret vehicle behavior (per VEL scope).

## 2. Design Position

In the VEL architecture, quality metadata is part of the Output Evidence Definition. It is assigned by the Quality Evaluator based on the normalization mapping rules and validation results. This document specifies the data representation and configuration format only.

## 3. Quality Metadata Model

### 3.1 Quality metadata is part of the Output Evidence Definition

The quality metadata fields are declared in the Output Evidence Definition's `quality` section. They are mandatory in every record.

### 3.2 Quality fields

The Output Evidence Definition declares four quality fields:

| Field | Type | Allowed Values | Description |
| --- | --- | --- | --- |
| `validity` | enum | `valid`, `invalid`, `unknown` | Whether the evidence observation is valid. |
| `freshness` | enum | `current`, `delayed`, `stale` | How current the observation is relative to expectations. |
| `completeness` | enum | `complete`, `partial`, `missing` | Whether all expected observation data is present. |
| `mappingQuality` | enum | `exact`, `approximate`, `unmapped` | How faithfully the source value was mapped to the normalized representation. |

### 3.3 Quality semantics

- **validity** — indicates whether the observation is considered valid. `invalid` is used when an invalid-value record is emitted. `unknown` is used when validity cannot be determined.
- **freshness** — indicates the timeliness of the observation. `current` means the observation is within the expected freshness window; `delayed` means it is late but usable; `stale` means it is too old to be considered current. The freshness window (the threshold between `current`, `delayed`, and `stale`) is **deployment-specific** and declared in the deployment configuration.
- **completeness** — indicates whether all expected fields are present. `complete` means all expected data present; `partial` means some expected data missing; `missing` means the expected observation data is entirely absent.
- **mappingQuality** — indicates the fidelity of the normalization mapping. `exact` means a direct/exact mapping; `approximate` means a scaled or approximate conversion; `unmapped` means the source value could not be mapped to a normalized representation.

### 3.4 Quality assignment

Quality values are assigned by the Quality Evaluator. The Output Evidence Contract defines the **allowed values and structure**; the assignment rules are defined in the normalization mapping contract and the Quality Evaluation design.

### 3.5 Defaults

The reference example shows default quality values in the normalization mapping:

```yaml
defaults:
  quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
```

These defaults apply when no quality-affecting condition is detected.

## 4. Configuration Format

### 4.1 Quality fields in the Output Evidence Definition

```yaml
apiVersion: score.dev/v1alpha1
kind: NormalizedEvidenceSchema
metadata:
  name: score-normalized-evidence
  version: 1.0.0
spec:
  owner: score
  encoding: json
  requiredSections: [header, identity, observation, time, quality, traceability]
  fields:
    # ... header, identity, observation, time fields omitted for brevity
    - {path: $.quality.validity, type: enum, values: [valid, invalid, unknown], required: true}
    - {path: $.quality.freshness, type: enum, values: [current, delayed, stale], required: true}
    - {path: $.quality.completeness, type: enum, values: [complete, partial, missing], required: true}
    - {path: $.quality.mappingQuality, type: enum, values: [exact, approximate, unmapped], required: true}
    # ... traceability fields omitted for brevity
```

### 4.2 Example quality section in a record

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

- Quality metadata is attached to each evidence record and is mandatory. Batch-level quality aggregation is optional and may be provided by the deployment.
- Quality describes the **observation**, not the VEL pipeline. VEL pipeline health is exposed separately via VEL Health.
- Quality values are assigned by the Quality Evaluator based on the normalization mapping rules and validation results. The four quality dimensions (`validity`, `freshness`, `completeness`, `mappingQuality`) are the complete set; no additional dimensions are added.
- Consumers use quality metadata to determine how much trust to place in the evidence observation.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-009, STKH-VEL-009 |
| Reference example | [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml) |
