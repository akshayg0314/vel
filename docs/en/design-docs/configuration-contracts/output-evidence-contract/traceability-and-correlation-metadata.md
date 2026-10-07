# Traceability and Correlation Metadata Fields

> **Related:** FR-VEL-006 (Source Identity Metadata), FR-VEL-007 (Observation Time Metadata), FR-VEL-008 (Correlation Metadata), STKH-VEL-006, STKH-VEL-007, STKH-VEL-008
> **Reference example:** [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

## 1. Purpose

This document defines the **traceability** and **correlation** metadata fields in the normalized Vehicle Evidence record. These fields preserve the provenance of observations: where the data came from (source identity), when it was observed (time), and how it was transformed (rule versions), plus the correlation identifier that links related evidence records.

## 2. Design Position

In the VEL architecture, traceability, identity, and time fields are part of the Output Evidence Definition. They are populated by the Traceability Enricher from the source record's identity/timestamp fields and the normalization rule versions. This document specifies the data representation and configuration format only.

## 3. Traceability and Correlation Model

### 3.1 Traceability metadata fields

The `traceability` section of the Output Evidence Definition declares provenance fields:

| Field | Type | Description |
| --- | --- | --- |
| inputSchemaVersion | string | Version of the Input Interface Definition used to validate the source record. |
| normalizationRuleVersion | string | Version of the Normalization Mapping rule used to transform the source record. |
| sourceMessageId | string | The unique identifier of the raw source message. |

These fields record **how** the evidence was produced, enabling audit and reproducibility.

### 3.2 Identity metadata fields

The `identity` section carries the source identity and correlation identifiers:

| Field | Type | Description |
| --- | --- | --- |
| `sourceId` | string | The source that produced the record. |
| `workloadId` | string | The workload or execution context. |
| `executionInstanceId` | string | The execution instance. |
| `resourceId` | string | The specific resource observed. |
| `correlationId` | string | The correlation identifier linking related evidence records. |

### 3.3 Time metadata fields

The `time` section carries the observation time:

| Field | Type | Description |
| --- | --- | --- |
| `eventTimestampNs` | uint64 | The observation timestamp in nanoseconds. |
| `clockDomain` | string | The clock domain of the timestamp (`monotonic` or `wall-clock`). |

### 3.4 Correlation identifier semantics

- The `correlationId` links multiple evidence records originating from the same source event or execution instance. A single `correlationId` field is sufficient; multiple correlation dimensions (e.g., per-batch) are not needed since records are grouped hierarchically by source/workload/resource and additionally by correlationId.
- In the reference example, `execution_id` is mapped to both `executionInstanceId` and `correlationId`. This means all evidence records from one execution share the same correlation identifier.
- The `correlationId` field is declared `required: true` in the Output Evidence Definition (per the reference schema). It is **always populated** by mapping the source's `execution_id` to `correlationId` in the common mappings. This satisfies FR-VEL-008 ("when source context provides it") because every source in the current scope provides an `execution_id`.
- The `clockDomain` field is recorded per evidence record. When sources with different clock domains are aggregated, each record retains its own domain; no mixing occurs within a single record, so a per-record domain is sufficient. Cross-source clock alignment is handled at the deployment level, not by VEL.
- Provenance includes exactly the three fields in Section 3.1 of this document. Additional provenance (e.g., collector or adapter version) is not needed because the Input Interface Definition name and version (referenced by inputSchemaVersion) already identify the collector/adapter.

### 3.5 Propagation responsibility

The Traceability Enricher is responsible for populating these fields during normalization. This document defines the **contract** for those fields.

## 4. Configuration Format

### 4.1 Traceability, identity, and time fields in the Output Evidence Definition

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
    - {path: $.identity.sourceId, type: string, required: true}
    - {path: $.identity.workloadId, type: string, required: true}
    - {path: $.identity.executionInstanceId, type: string, required: true}
    - {path: $.identity.resourceId, type: string, required: true}
    - {path: $.identity.correlationId, type: string, required: true}
    - {path: $.time.eventTimestampNs, type: uint64, required: true}
    - {path: $.time.clockDomain, type: string, required: true}
    - {path: $.traceability.inputSchemaVersion, type: string, required: true}
    - {path: $.traceability.normalizationRuleVersion, type: string, required: true}
    - {path: $.traceability.sourceMessageId, type: string, required: true}
```

### 4.2 Example traceability/identity/time sections in a record

```json
{
  "identity": {
    "sourceId": "vendor-a-npu",
    "workloadId": "inference-workload",
    "executionInstanceId": "exec-42",
    "resourceId": "npu-0",
    "correlationId": "exec-42"
  },
  "time": {
    "eventTimestampNs": 1710000000000000000,
    "clockDomain": "monotonic"
  },
  "traceability": {
    "inputSchemaVersion": "1.2.0",
    "normalizationRuleVersion": "3.1.0",
    "sourceMessageId": "msg-0001"
  }
}
```

## 5. Behavioral Semantics

- Traceability, identity, and time fields are mandatory in every normalized Vehicle Evidence record.
- These fields are populated by the Traceability Enricher from the source record's identity/timestamp fields and the normalization rule versions.
- The `correlationId` enables consumers to group evidence records from the same source event. It is populated from the source's correlation identifier when provided.
- The `sourceMessageId` provides exact provenance back to the raw source message.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-006, FR-VEL-007, FR-VEL-008, STKH-VEL-006, STKH-VEL-007, STKH-VEL-008 |
| Reference example | [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml) |
