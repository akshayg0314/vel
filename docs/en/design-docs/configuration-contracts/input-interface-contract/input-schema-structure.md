# Input Schema Structure and Required Fields

> **Related:** FR-VEL-013 (Input Interface Configuration), STKH-VEL-013
> **Reference example:** [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml)

## 1. Purpose

This document defines the **Input Interface Definition** configuration artifact: the structure and required fields of the raw source data VEL accepts from a given source. It is the foundational contract for the input side of the Evidence Layer, upon which collectors, validation, normalization, quality, and persistence all depend.

## 2. Design Position

In the VEL architecture, the input contract is configuration-driven. Each source (runtime vendor, hardware device, S-CORE module) provides its own Input Interface Definition. Platform-specific collectors are kept separate from the common Evidence Layer (FR-VEL-012). This document specifies the data representation and configuration format only; it does not define collector implementation, transport behavior, or pipeline processing.

## 3. Source Data Model

### 3.1 Input Interface Definition as a Configuration Artifact

The input interface is defined in configuration, not hard-coded. Each source provides its own Input Interface Definition, which enables adding new sources without modifying the common Evidence Layer (FR-VEL-013).

### 3.2 Canonical Contract Kind: RawEvidenceSchema

The input contract uses the `RawEvidenceSchema` kind, mirroring the reference example package.

### 3.3 Contract Structure

Each Input Interface Definition has:

- **Metadata** — name, version, owner.
- **Encoding** — the serialization format of the raw source record. `json` is the canonical encoding; other encodings (protobuf, CBOR) are future extensions.
- **Message type** — the logical message type name of the source record.
- **Field list** — each field with:
  - `name` — logical field name used in configuration references.
  - `path` — JSONPath (RFC 9535) to the field in the raw record.
  - `type` — the data type (string, number, integer, uint64, boolean, enum, etc.).
  - `required` — whether the field must be present.
  - Optional constraints: `range`, `allowedValues`, `pattern`, `minimum`, `maximum`.
- **Additional fields policy** — `reject` or `allow` for fields not declared in the schema.

The input contract is **validator-agnostic**: the contract format is independent of any specific schema validator engine. The concrete validator and its supported constraint types are selected by the deployment.

### 3.4 Required Fields

The following field categories are **required** in every Input Interface Definition because downstream processing depends on them:

| Category | Purpose |
| --- | --- |
| `message_id` | A unique identifier for the raw source message, used for traceability. |
| `source_id` | Identifies the source that produced the record. |
| `workload_id` | Identifies the workload or execution context. |
| `execution_id` | Identifies the execution instance; also used as correlation identifier. |
| `resource_id` | Identifies the specific hardware/software resource being observed. |
| `timestamp_ns` | The observation timestamp in nanoseconds. |

These are the **common required fields**. Source-specific fields (e.g., `inference_latency_us`, `device_state`) are declared as additional fields with their own requiredness.

### 3.5 Field Type Taxonomy

The input contract supports the following field types:

| Type | Description | Example |
| --- | --- | --- |
| `string` | UTF-8 text | `source_id` |
| `uint64` / `uint32` / `uint8` | Unsigned integers | `timestamp_ns`, `device_state` |
| `int64` / `int32` | Signed integers | signed metrics |
| `number` | Floating point | latency in ms |
| `boolean` | True/false | flags |
| `enum` | Restricted set of values | `device_state` allowed values |
| `array` | List of values | multi-value metrics |

### 3.6 Constraints

Fields may declare constraints that the Schema Validator enforces:

- `range: {minimum, maximum}` — numeric bounds.
- `allowedValues: [..]` — enumerated values.
- `pattern` — regex pattern for strings.
- `required: true/false` — presence requirement.

## 4. Configuration Format

### 4.1 Example Input Interface Definition

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
    - name: inference_latency_us
      path: $.inference_latency_us
      type: uint32
      required: false
      range: {minimum: 0, maximum: 10000000}
    - name: device_state
      path: $.device_state
      type: uint8
      required: true
      allowedValues: [0, 1, 2, 3]
    - {name: vendor_error_code, path: $.vendor_error_code, type: string, required: false}
```

### 4.2 Example Raw Source Record (JSON)

```json
{
  "message_id": "msg-0001",
  "source_id": "vendor-a-npu",
  "workload_id": "inference-workload",
  "execution_id": "exec-42",
  "resource_id": "npu-0",
  "timestamp_ns": 1710000000000000000,
  "inference_latency_us": 38400,
  "device_state": 2,
  "vendor_error_code": "0xA17"
}
```

## 5. Behavioral Semantics

- A source record is **schema-valid** when all `required: true` fields are present, present fields match their declared type, and all declared constraints are satisfied.
- The `additionalFields` policy determines whether undeclared fields cause rejection (`reject`) or are tolerated (`allow`). The default recommendation is `reject` to enforce strict contracts.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-013, STKH-VEL-013 |
| Reference example | [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml) |
