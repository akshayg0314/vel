# 매핑되지 않은 값 및 알 수 없는 값 처리

> **관련:** FR-VEL-005 (소스 정규화), FR-VEL-009 (Evidence 품질 메타데이터), STKH-VEL-005, STKH-VEL-009
> **참조 예시:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

## 1. 목적

이 문서는 정규화 매핑이 정규화된 표현으로 매핑할 수 없는 값(알 수 없는 enum 값, 매핑되지 않은 오류 코드, 정의된 매핑 규칙 밖의 기타 값)을 처리하는 방법을 정의합니다.

## 2. 설계 위치

VEL 아키텍처에서 정규화 매핑은 구성 기반입니다. 매핑되지 않은/알 수 없는 값 처리는 정규화 매핑 계약의 일부이며 매핑별로 구성할 수 있습니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 매핑되지 않은 값 처리 모델

### 3.1 매핑되지 않은 값 vs. 알 수 없는 값

관련이 있지만 구별되는 두 개념입니다:

| 개념 | 설명 | 예시 |
| --- | --- | --- |
| **매핑되지 않은 값(Unmapped value)** | 존재하지만 구성된 매핑 규칙이 없는 값. | 조회 테이블에 없는 `vendor_error_code: 0xC99`. |
| **알 수 없는 값(Unknown value)** | 의미를 결정할 수 없는 값. | 허용 집합 밖의 enum 값이 `unknown`으로 매핑됨. |

`unknown`은 알 수 없는/매핑되지 않은 값의 표준 표현입니다. 다른 센티널 값 (예: `null`, `-1`)은 사용되지 않습니다. 이 값은 정당한 데이터 값과 모호할 수 있습니다.

### 3.2 처리 전략

정규화 매핑은 매핑되지 않은/알 수 없는 값이 처리되는 방식을 정의합니다:

| 전략 | 설명 | 품질에 대한 효과 |
| --- | --- | --- |
| `emit-unknown` | `unknown` (또는 구성된 `unmappedValue`)의 정규화된 값을 생성. | `mappingQuality: unmapped`. |
| `emit-invalid-evidence` | 원래 값을 보존하며 무효로 표시된 evidence 레코드 생성. | `validity: invalid`, `mappingQuality: unmapped`. |
| `preserve-original` | `observation.originalValue`에 원래 값을 유지. | `mappingQuality: unmapped`. |
| `reject` | 레코드를 완전히 폐기. | 생성된 evidence 없음. |

### 3.3 enumMap unmappedValue

enum/상태 매핑 (`enumMap`)의 경우 변환은 소스 값이 매핑 테이블에 없을 때 사용되는 `unmappedValue`를 선언합니다:

```yaml
transform:
  operation: enumMap
  values: {0: available, 1: busy, 2: degraded, 3: unavailable}
  unmappedValue: unknown
```

`device_state: 5` (테이블에 없음)가 도착하면 `mappingQuality: unmapped`로 `unknown`에 매핑됩니다.

### 3.4 lookup onUnmapped

조회 기반 매핑 (예: 벤더 오류 코드)의 경우 매핑은 폴백 동작을 정의하는 `onUnmapped` 블록을 선언합니다:

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

`vendor_error_code: 0xC99` (조회 테이블에 없음)가 도착하면 `normalizedClass: unmapped`, `mappingQuality: unmapped`, 원래 값이 보존된 `accelerator.vendor-event` evidence를 생성합니다.

### 3.5 실패 처리 통합

정규화 매핑의 `failureHandling` 블록은 입력 계약의 실패 처리와 조정합니다:

```yaml
failureHandling:
  missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
  invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
  transformationError: {result: reject, emitAdapterDiagnostic: true}
```

- `unmappedEnum` — 기본값은 `mappingQuality: unmapped`를 사용한 `emit-unknown`입니다.
- `invalidValue` — 기본값은 원래 값을 보존한 `emit-invalid-evidence`입니다.
- `transformationError` — 기본값은 어댑터 진단을 사용한 `reject`입니다.

**기본 매핑되지 않은 동작:** 매핑되지 않은 enum/조회 값의 기본값은 `mappingQuality: unmapped`를 사용한 **`emit-unknown`** 입니다. 그러나 각 매핑은 엄격한 거부를 요구하는 배포를 위해 **`reject`** 로 재정의할 수 있습니다. 이는 품질 신호로 evidence를 보존하는 것과 일부 배포에 필요한 엄격성 사이의 균형을 유지합니다.

## 4. 구성 형식

### 4.1 unmappedValue가 있는 enumMap

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

### 4.2 onUnmapped가 있는 lookup

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

### 4.3 failureHandling 블록

```yaml
failureHandling:
  missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
  invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
  transformationError: {result: reject, emitAdapterDiagnostic: true}
```

## 5. 동작 의미론

- 매핑되지 않은 enum 값은 구성된 `unmappedValue` (일반적으로 `unknown`)로 변환되고 `mappingQuality: unmapped`로 표시됩니다.
- 매핑되지 않은 조회 키는 원래 값이 보존된 `mappingQuality: unmapped`로 폴백 evidence (예: 일반 vendor-event)를 생성합니다.
- 출력 레코드의 `mappingQuality` 필드는 값이 정확히 매핑되지 않았음을 소비자에게 알립니다.
- `reject`가 사용되면 evidence가 생성되지 않고 어댑터 진단이 발행됩니다. 매핑에서 참조하는 소스 값을 해석하거나 변환할 수 없는 경우 매핑 결과는 구성된 failureHandling에 따라 emit-unknown 또는 reject입니다. 복합 소스 값의 부분 매핑은 없습니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-005, FR-VEL-009, STKH-VEL-005, STKH-VEL-009 |
| 참조 예시 | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) |