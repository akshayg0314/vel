# Normalized Evidence Record Structure

> **Related:** FR-VEL-004 (Vehicle Evidence), FR-VEL-011 (Evidence Publication), FR-VEL-014 (Output Evidence Configuration), STKH-VEL-001, STKH-VEL-011, STKH-VEL-014
> **Reference example:** [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

## 1. Purpose

This document defines the **Output Evidence Definition** configuration artifact: the canonical structure of a normalized Vehicle Evidence record. It specifies the six canonical sections (`header`, `identity`, `observation`, `time`, `quality`, `traceability`) and their required fields.

## 2. Design Position

In the VEL architecture, the output evidence contract is configuration-driven. It defines the canonical data representation that all downstream consumers depend on. This document specifies the data representation and configuration format only; it does not define publication behavior or persistence.

## 3. Output Evidence Record Model

### 3.1 Canonical Contract Kind: NormalizedEvidenceSchema

The output evidence contract uses the `NormalizedEvidenceSchema` kind, mirroring the reference example:

- [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

### 3.2 Record structure

Each normalized Vehicle Evidence record MUST conform to the Output Evidence Definition. The `requiredSections` list defines the mandatory top-level sections. Deployments MUST NOT add deployment-specific sections beyond the six canonical sections:

| Section | Description |
| --- | --- |
| `header` | Evidence identification and metadata. |
| `identity` | Source identity and correlation identifiers. |
| `observation` | The observed value or state. |
| `time` | Observation timestamp and clock domain. |
| `quality` | Evidence quality metadata (validity, freshness, completeness, mapping quality). |
| `traceability` | Provenance (input schema version, normalization rule version, source message id). |

### 3.3 Section details

- **header** — contains `evidenceId`, a UUID (v4) that uniquely identifies the evidence record. It is generated as a UUID (v4), which is universally unique, requires no coordination, and supports distributed collectors.
- **identity** — contains `sourceId`, `workloadId`, `executionInstanceId`, `resourceId`, and `correlationId`.
- **observation** — may be value-based, state-based, or both. **At least one observation field MUST be populated** for a record to be meaningful evidence.
- **time** — contains `eventTimestampNs` and `clockDomain`.
- **quality** — contains `validity`, `freshness`, `completeness`, and `mappingQuality`.
- **traceability** — contains `inputSchemaVersion`, `normalizationRuleVersion`, and `sourceMessageId`.

### 3.4 Aggregation data format

The aggregation data format is `json`.

## 4. Configuration Format

### 4.1 Example Output Evidence Definition

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
    - {path: $.header.evidenceId, type: string, required: true}
    - {path: $.identity.sourceId, type: string, required: true}
    - {path: $.identity.workloadId, type: string, required: true}
    - {path: $.identity.executionInstanceId, type: string, required: true}
    - {path: $.identity.resourceId, type: string, required: true}
    - {path: $.identity.correlationId, type: string, required: true}
    - {path: $.time.eventTimestampNs, type: uint64, required: true}
    - {path: $.time.clockDomain, type: string, required: true}
    - {path: $.quality.validity, type: enum, values: [valid, invalid, unknown], required: true}
    - {path: $.quality.freshness, type: enum, values: [current, delayed, stale], required: true}
    - {path: $.quality.completeness, type: enum, values: [complete, partial, missing], required: true}
    - {path: $.quality.mappingQuality, type: enum, values: [exact, approximate, unmapped], required: true}
    - {path: $.traceability.inputSchemaVersion, type: string, required: true}
    - {path: $.traceability.normalizationRuleVersion, type: string, required: true}
    - {path: $.traceability.sourceMessageId, type: string, required: true}
```

### 4.2 Example normalized Vehicle Evidence record

```json
{
  "header": {
    "evidenceId": "550e8400-e29b-41d4-a716-446655440000"
  },
  "identity": {
    "sourceId": "vendor-a-npu",
    "workloadId": "inference-workload",
    "executionInstanceId": "exec-42",
    "resourceId": "npu-0",
    "correlationId": "exec-42"
  },
  "observation": {
    "value": 38.4,
    "unit": "ms"
  },
  "time": {
    "eventTimestampNs": 1710000000000000000,
    "clockDomain": "monotonic"
  },
  "quality": {
    "validity": "valid",
    "freshness": "current",
    "completeness": "complete",
    "mappingQuality": "exact"
  },
  "traceability": {
    "inputSchemaVersion": "1.2.0",
    "normalizationRuleVersion": "3.1.0",
    "sourceMessageId": "msg-0001"
  }
}
```

## 5. Behavioral Semantics

- Every normalized Vehicle Evidence record MUST conform to the Output Evidence Definition.
- The `requiredSections` list defines the mandatory top-level sections. Deployments MUST NOT add deployment-specific sections beyond the six canonical sections (`header`, `identity`, `observation`, `time`, `quality`, `traceability`).
- The `header.evidenceId` uniquely identifies the evidence record. It is generated as a **UUID (v4)**, which is universally unique, requires no coordination, and supports distributed collectors.
- The `observation` section may be value-based, state-based, or both. **At least one observation field MUST be populated** for a record to be meaningful evidence.
- The aggregation data format is `json`.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-004, FR-VEL-011, FR-VEL-014, STKH-VEL-001, STKH-VEL-011, STKH-VEL-014 |
| Reference example | [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml) |
