# Unmapped and Unknown Value Handling

> **Related:** FR-VEL-005 (Source Normalization), FR-VEL-009 (Evidence Quality Metadata), STKH-VEL-005, STKH-VEL-009
> **Reference example:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

## 1. Purpose

This document defines how the Normalization Mapping handles values that cannot be mapped to a normalized representation: unknown enum values, unmapped error codes, and other values that fall outside the defined mapping rules.

## 2. Design Position

In the VEL architecture, the normalization mapping is configuration-driven. Unmapped/unknown value handling is part of the Normalization Mapping contract and is configurable per mapping. This document specifies the data representation and configuration format only.

## 3. Unmapped Value Handling Model

### 3.1 Unmapped vs. unknown values

Two related but distinct concepts:

| Concept | Description | Example |
| --- | --- | --- |
| **Unmapped value** | A value that exists but has no configured mapping rule. | `vendor_error_code: 0xC99` not in the lookup table. |
| **Unknown value** | A value whose semantics cannot be determined. | An enum value outside the allowed set, mapped to `unknown`. |

`unknown` is the canonical representation for unknown/unmapped values. No other sentinel values (e.g., `null`, `-1`) are used; they would be ambiguous with legitimate data values.

### 3.2 Handling strategies

The normalization mapping defines how unmapped/unknown values are handled:

| Strategy | Description | Effect on quality |
| --- | --- | --- |
| `emit-unknown` | Produce a normalized value of `unknown` (or the configured `unmappedValue`). | `mappingQuality: unmapped`. |
| `emit-invalid-evidence` | Produce an evidence record marked invalid, preserving the original value. | `validity: invalid`, `mappingQuality: unmapped`. |
| `preserve-original` | Keep the original value in `observation.originalValue`. | `mappingQuality: unmapped`. |
| `reject` | Discard the record entirely. | No evidence produced. |

### 3.3 enumMap unmappedValue

For enum/state mappings (`enumMap`), the transform declares an `unmappedValue` that is used when the source value is not in the mapping table:

```yaml
transform:
  operation: enumMap
  values: {0: available, 1: busy, 2: degraded, 3: unavailable}
  unmappedValue: unknown
```

If `device_state: 5` arrives (not in the table), it is mapped to `unknown` with `mappingQuality: unmapped`.

### 3.4 Lookup onUnmapped

For lookup-based mappings (e.g., vendor error codes), the mapping declares an `onUnmapped` block that defines the fallback behavior:

```yaml
lookup:
  "0xA17": {evidenceType: accelerator.execution.timeout, normalizedClass: execution-timeout}
  "0xB03": {evidenceType: accelerator.memory.failure, normalizedClass: device-memory-failure}
onUnmapped:
  evidenceType: accelerator.vendor-event
  normalizedClass: unmapped
  mappingQuality: unmapped
  preserveOriginalValue: true
```

If `vendor_error_code: 0xC99` arrives (not in the lookup table), it produces an `accelerator.vendor-event` evidence with `normalizedClass: unmapped`, `mappingQuality: unmapped`, and the original value preserved.

### 3.5 Failure handling integration

The `failureHandling` block in the Normalization Mapping coordinates with the Input Contract's failure handling:

```yaml
failureHandling:
  missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
  invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
  transformationError: {result: reject, emitAdapterDiagnostic: true}
```

- `unmappedEnum` — default `emit-unknown` with `mappingQuality: unmapped`.
- `invalidValue` — default `emit-invalid-evidence` with original value preserved.
- `transformationError` — default `reject` with adapter diagnostic.

**Default unmapped behavior:** The default for unmapped enum/lookup values is **`emit-unknown`** with `mappingQuality: unmapped`. However, each mapping MAY override this to **`reject`** for deployments that require strict rejection of any unmapped value. This balances preserving evidence with a quality signal against the strictness some deployments require.

## 4. Configuration Format

### 4.1 enumMap with unmappedValue

```yaml
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

### 4.2 lookup with onUnmapped

```yaml
- id: vendor-error
  when: {fieldExists: $.vendor_error_code}
  value:
    sourcePath: $.vendor_error_code
    targetPath: $.observation.originalValue
  lookup:
    "0xA17": {evidenceType: accelerator.execution.timeout, normalizedClass: execution-timeout}
    "0xB03": {evidenceType: accelerator.memory.failure, normalizedClass: device-memory-failure}
  onUnmapped:
    evidenceType: accelerator.vendor-event
    normalizedClass: unmapped
    mappingQuality: unmapped
    preserveOriginalValue: true
```

### 4.3 failureHandling block

```yaml
failureHandling:
  missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
  invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
  transformationError: {result: reject, emitAdapterDiagnostic: true}
```

## 5. Behavioral Semantics

- Unmapped enum values are converted to the configured `unmappedValue` (typically `unknown`) and marked with `mappingQuality: unmapped`.
- Unmapped lookup keys produce a fallback evidence (e.g., generic vendor-event) with `mappingQuality: unmapped` and the original value preserved.
- The `mappingQuality` field in the output record signals to consumers that the value was not exactly mapped.
- When `reject` is used, no evidence is produced and an adapter diagnostic is emitted. If any source value referenced by a mapping cannot be resolved or transformed, the mapping outcome is either emit-unknown or reject per the configured failureHandling; there is no partial mapping of a composite source value.

## 6. Traceability

| Item | Reference |
| --- | --- |
| Requirements | FR-VEL-005, FR-VEL-009, STKH-VEL-005, STKH-VEL-009 |
| Reference example | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) |
