# Validation Failure Handling for Invalid Source Records

> **Related:** FR-VEL-013 (Input Interface Configuration), STKH-VEL-013
> **Reference example:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) `failureHandling` block

## 1. Purpose

This document defines how VEL handles source records that fail schema validation. It establishes the contract for rejecting or diagnosing invalid source data before it enters the normalization pipeline. The concrete validation execution is detailed in the Schema Validation Design; this document defines the **contract-level** failure handling policy.

## 2. Design Position

In the VEL architecture, validation happens at the input boundary. The failure handling policy is part of the input contract and is configurable per source. This document specifies the data representation and configuration format only.

## 3. Failure Handling Model

### 3.1 Validation failure categories

A source record can fail validation in the following ways:

| Failure Category | Description | Example |
| --- | --- | --- |
| `missingRequiredField` | A field declared `required: true` is absent. | `timestamp_ns` missing. |
| `invalidType` | A present field has the wrong data type. | `device_state` is a string instead of uint8. |
| `outOfRange` | A numeric field violates its `range` constraint. | `inference_latency_us` > 10000000. |
| `notAllowedValue` | An enum field has a value outside `allowedValues`. | `device_state` = 5 (not in [0,1,2,3]). |
| `patternMismatch` | A string violates its `pattern`. | malformed `source_id`. |
| `additionalFieldPresent` | An undeclared field appears while `additionalFields: reject`. | extra field `foo` present. |

### 3.2 Failure handling policy

The input contract defines a **failure handling policy** that determines the outcome for each failure category. The reference example uses this policy (from the Normalization Mapping contract's `failureHandling` block):

```yaml
failureHandling:
  missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
  invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
  transformationError: {result: reject, emitAdapterDiagnostic: true}
```

### 3.3 Proposed failure handling outcomes

| Failure Category | Default Outcome | Notes |
| --- | --- | --- |
| `missingRequiredField` | `reject` | Record is rejected; adapter diagnostic emitted. |
| `invalidType` | `reject` | Record is rejected; adapter diagnostic emitted. |
| `outOfRange` | `reject` or `emit-invalid-evidence` | Configurable; default `reject`. |
| `notAllowedValue` | `emit-unknown` or `emit-invalid-evidence` | Configurable; default `emit-unknown` with `mappingQuality: unmapped`. |
| `patternMismatch` | `reject` | Record is rejected; adapter diagnostic emitted. |
| `additionalFieldPresent` | `reject` | When `additionalFields: reject`. |

### 3.4 Diagnostic output

When a record is rejected, VEL emits an **adapter diagnostic** that records the failure without becoming part of the Vehicle Evidence contract. The diagnostic schema is:

```yaml
apiVersion: score.dev/v1alpha1
kind: AdapterDiagnostic
metadata:
  name: <adapter-name>-diagnostic
spec:
  sourceMessageId: <message_id>
  sourceId: <source_id>
  failureCategory: <missingRequiredField|invalidType|outOfRange|notAllowedValue|patternMismatch|additionalFieldPresent>
  field: <offending-field-name>
  reason: <human-readable-reason>
  timestampNs: <observation-timestamp>
```

The diagnostic includes the failing source record identifier (`message_id`), the failure category, the offending field name, and a human-readable reason. This schema is shared with the audit logging design.

### 3.5 Invalid evidence vs. rejection

Two distinct handling modes exist:

- **Reject** — the record is discarded entirely; no Vehicle Evidence is produced. This is the **mandatory default** for all structural failures (missing required fields, wrong type, pattern mismatch, additional fields when `additionalFields: reject`).
- **Emit-invalid-evidence** — a Vehicle Evidence record is still produced but marked with quality `validity: invalid` and the original value preserved. This is available for value-level failures (out-of-range, not-allowed-value) where partial evidence is still useful. Deployments MAY configure strict rejection for these value-level failures.

Diagnostics are **rate-limited and aggregated** to prevent flooding at high invalid-record rates.

## 4. Configuration Format

### 4.1 Failure handling policy in the Input Interface Definition

```yaml
apiVersion: score.dev/v1alpha1
kind: RawEvidenceSchema
metadata:
  name: vendor-a-npu-status
  version: 1.2.0
spec:
  owner: runtime-vendor
  encoding: json
  messageType: VendorANpuStatus
  additionalFields: reject
  failureHandling:
    missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
    invalidType: {result: reject, emitAdapterDiagnostic: true}
    outOfRange: {result: reject, emitAdapterDiagnostic: true}
    notAllowedValue: {result: emit-unknown, mappingQuality: unmapped}
    patternMismatch: {result: reject, emitAdapterDiagnostic: true}
    additionalFieldPresent: {result: reject, emitAdapterDiagnostic: true}
  fields:
    - {name: message_id, path: $.message_id, type: string, required: true}
    # ... other fields
```

### 4.2 Adapter diagnostic record (proposed shape)

```yaml
apiVersion: score.dev/v1alpha1
kind: AdapterDiagnostic
metadata:
  name: vendor-a-npu-adapter-diagnostic
spec:
  sourceMessageId: msg-0001
  sourceId: vendor-a-npu
  failureCategory: missingRequiredField
  field: timestamp_ns
  reason: "required field 'timestamp_ns' is missing"
  timestampNs: 1710000000000000000
```

## 5. Behavioral Semantics

- Schema validation happens **before** normalization. A record that fails with `reject` never reaches the Normalization Processor.
- A record that fails with `emit-invalid-evidence` or `emit-unknown` proceeds to normalization with the failure recorded in the evidence quality metadata.
- Adapter diagnostics are operational/audit data, separate from the Vehicle Evidence contract (per the Collection and Audit Logging design).
- The failure handling policy is part of the input contract and is configurable per source.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-013, STKH-VEL-013 |
| Reference example | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) `failureHandling` block |
