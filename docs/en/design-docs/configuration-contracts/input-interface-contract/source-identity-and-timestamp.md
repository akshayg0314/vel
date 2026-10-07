# Source Identity and Timestamp Requirements

> **Related:** FR-VEL-006 (Source Identity Metadata), FR-VEL-007 (Observation Time Metadata), FR-VEL-008 (Correlation Metadata), STKH-VEL-006, STKH-VEL-007, STKH-VEL-008
> **Reference example:** [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml)

## 1. Purpose

This document defines how a source record identifies itself (source identity) and how it expresses its observation time (timestamp). These fields are prerequisites for the traceability requirements (FR-VEL-006, FR-VEL-007) and are propagated to the normalized Vehicle Evidence by the Traceability Enricher.

## 2. Design Position

In the VEL architecture, identity and timestamp fields are declared per-source in the Input Interface Definition. They are the common required fields that every source must provide, enabling downstream traceability and correlation. This document specifies the data representation and configuration format only.

## 3. Source Data Model

### 3.1 Source Identity fields

Each Input Interface Definition MUST declare the following identity fields:

| Field | Description | Requiredness | Notes |
| --- | --- | --- | --- |
| `source_id` | Identifies the source (vendor/hardware/OS/S-CORE module) that produced the record. | required | Used as `identity.sourceId` in normalized output. |
| `workload_id` | Identifies the workload or execution context the record belongs to. | required | Used as `identity.workloadId`. |
| `resource_id` | Identifies the specific resource (e.g., NPU-0, CPU-2) being observed. | required | Used as `identity.resourceId`. |
| `execution_id` | Identifies the execution instance. Doubles as the correlation identifier. | required | Used as both `identity.executionInstanceId` and `identity.correlationId`. |
| `message_id` | Unique identifier of the raw message itself. | required | Used as `traceability.sourceMessageId`. Distinct from execution_id; multiple messages may share an execution_id. |

### 3.2 Timestamp requirements

The Input Interface Definition MUST declare a timestamp field. Requirements:

- **Field name convention:** `timestamp_ns` (nanoseconds since epoch or since boot depending on clock domain).
- **Type:** `uint64` (nanoseconds, non-negative).
- **Requiredness:** `required: true`.
- **Clock domain:** declared **per-source** in the Input Interface Definition. Accepted domains: `monotonic` (since boot) or `wall-clock` (epoch). Default is `monotonic` for runtime/hardware metrics, matching the reference example, but each source may declare its own domain. This allows mixed-domain sources to coexist in one deployment.
- **Precision:** nanoseconds. If the source provides a coarser timestamp, it should be converted to nanoseconds by the normalizer using the transform operation.

### 3.3 Identity/timestamp placement in contract

Identity and timestamp fields are part of the Input Interface Definition's `spec.fields` list. They are distinguishable by their declared `name`, and downstream components reference them by name.

### 3.4 Uniqueness and correlation semantics

- Each raw message MUST have a unique `message_id` to allow exact provenance. If the source does not guarantee uniqueness, VEL generates a synthetic message id to enforce it.
- `execution_id` is always the correlation identifier (FR-VEL-008). It MAY be shared across multiple messages (e.g., all messages from one inference run), enabling downstream consumers to group evidence records belonging to the same source event. No separate correlation field is needed.
- VEL does not guarantee cross-source clock synchronization; each source declares its own clock domain, and consumers must account for domain differences when aggregating records from multiple sources.

## 4. Configuration Format

### 4.1 Identity and timestamp fields in Input Interface Definition

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
  fields:
    - {name: message_id, path: $.message_id, type: string, required: true}
    - {name: source_id, path: $.source_id, type: string, required: true}
    - {name: workload_id, path: $.workload_id, type: string, required: true}
    - {name: execution_id, path: $.execution_id, type: string, required: true}
    - {name: resource_id, path: $.resource_id, type: string, required: true}
    - {name: timestamp_ns, path: $.timestamp_ns, type: uint64, required: true}
    # ... additional source-specific fields omitted for brevity
```

### 4.2 Clock domain declaration (optional extension proposal)

```yaml
spec:
  owner: runtime-vendor
  encoding: json
  messageType: VendorANpuStatus
  additionalFields: reject
  time:
    clockDomain: monotonic   # or wall-clock
    precision: ns
  fields: [...]
```

> Note: the `time` block is a proposed extension. Whether the clock domain lives here or in the Normalization Mapping contract is an open design item.

## 5. Behavioral Semantics

- A record missing any required identity or timestamp field is schema-invalid and MUST be handled per the validation failure handling policy.
- The identity values are copied into the normalized output by the Traceability Enricher into the `identity` and `traceability` sections.
- Timestamp is propagated as `time.eventTimestampNs` along with the clock domain as `time.clockDomain` in the normalized Vehicle Evidence record.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-006, FR-VEL-007, FR-VEL-008, STKH-VEL-006, STKH-VEL-007, STKH-VEL-008 |
| Reference example | [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml) |
