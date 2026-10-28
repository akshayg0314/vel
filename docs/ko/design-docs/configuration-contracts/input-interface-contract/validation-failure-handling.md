# 잘못된 소스 레코드의 검증 실패 처리

> **관련:** FR-VEL-013 (입력 인터페이스 구성), STKH-VEL-013
> **참조 예시:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) `failureHandling` 블록

## 1. 목적

이 문서는 VEL이 스키마 검증에 실패한 소스 레코드를 처리하는 방법을 정의합니다. 정규화 파이프라인에 들어가기 전에 잘못된 소스 데이터를 거부하거나 진단하기 위한 계약을 수립합니다. 구체적인 검증 실행은 스키마 검증 설계에서 상세히 다루며, 이 문서는 **계약 수준** 실패 처리 정책을 정의합니다.

## 2. 설계 위치

VEL 아키텍처에서 검증은 입력 경계에서 발생합니다. 실패 처리 정책은 입력 계약의 일부이며 소스별로 구성할 수 있습니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 실패 처리 모델

### 3.1 검증 실패 범주

소스 레코드는 다음과 같은 방식으로 검증에 실패할 수 있습니다:

| 실패 범주 | 설명 | 예시 |
| --- | --- | --- |
| `missingRequiredField` | `required: true`로 선언된 필드가 없음. | `timestamp_ns` 누락. |
| `invalidType` | 존재하는 필드의 데이터 유형이 잘못됨. | `device_state`가 uint8 대신 문자열. |
| `outOfRange` | 숫자 필드가 `range` 제약 조건을 위반. | `inference_latency_us` > 10000000. |
| `notAllowedValue` | enum 필드가 `allowedValues` 밖의 값을 가짐. | `device_state` = 5 ([0,1,2,3]에 없음). |
| `patternMismatch` | 문자열이 `pattern`을 위반. | 잘못된 형식의 `source_id`. |
| `additionalFieldPresent` | `additionalFields: reject`인 동안 선언되지 않은 필드가 나타남. | 추가 필드 `foo` 존재. |

### 3.2 실패 처리 정책

입력 계약은 각 실패 범주에 대한 결과를 결정하는 **실패 처리 정책**을 정의합니다. 참조 예시는 이 정책을 사용합니다 (정규화 매핑 계약의 `failureHandling` 블록에서):

```yaml
failureHandling:
  missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
  invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
  transformationError: {result: reject, emitAdapterDiagnostic: true}
```

### 3.3 제안된 실패 처리 결과

| 실패 범주 | 기본 결과 | 참고 |
| --- | --- | --- |
| `missingRequiredField` | `reject` | 레코드가 거부됨; 어댑터 진단이 발행됨. |
| `invalidType` | `reject` | 레코드가 거부됨; 어댑터 진단이 발행됨. |
| `outOfRange` | `reject` 또는 `emit-invalid-evidence` | 구성 가능; 기본값 `reject`. |
| `notAllowedValue` | `emit-unknown` 또는 `emit-invalid-evidence` | 구성 가능; 기본값 `emit-unknown` 및 `mappingQuality: unmapped`. |
| `patternMismatch` | `reject` | 레코드가 거부됨; 어댑터 진단이 발행됨. |
| `additionalFieldPresent` | `reject` | `additionalFields: reject`인 경우. |

### 3.4 진단 출력

레코드가 거부되면 VEL은 Vehicle Evidence 계약의 일부가 되지 않는 실패를 기록하는 **어댑터 진단**을 발행합니다. 진단 스키마는 다음과 같습니다:

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

진단에는 실패한 소스 레코드 식별자(`message_id`), 실패 범주, 문제 필드 이름, 사람이 읽을 수 있는 이유가 포함됩니다. 이 스키마는 감사 로깅 설계와 공유됩니다.

> **수집기 I/O 수준 `AdapterDiagnostic`과의 관계:** 이는 수집기의 I/O 수준 진단 보고에서 사용되는 것과 동일한 `AdapterDiagnostic` 종류입니다 ([Source Collector 인터페이스 설계 Section 5.2](../../evidence-ingestion/source-collector-interface-design.md#52-진단-출력)). 해당 문서의 진단은 수집기 `error_category` 세분성(예: `SOURCE_UNAVAILABLE`, `TIMEOUT`, `SCHEMA_VIOLATION`)의 실패를 다룹니다; 이 섹션의 `failureCategory` 값은 raw 레코드가 존재하고 스키마 검증이 실행된 후 `error_category: SCHEMA_VIOLATION` 하에서 구체적으로 보고되는 더 세분화된 분류입니다.

### 3.5 잘못된 evidence vs. 거부

두 가지 별개의 처리 모드가 존재합니다:

- **Reject** — 레코드가 완전히 폐기됨; Vehicle Evidence가 생성되지 않음. 이는 모든 구조적 실패(필수 필드 누락, 잘못된 유형, 패턴 불일치, `additionalFields: reject`일 때 추가 필드)에 대한 **필수 기본값**입니다.
- **Emit-invalid-evidence** — Vehicle Evidence 레코드가 여전히 생성되지만 품질 `validity: invalid`로 표시되고 원래 값이 보존됩니다. 이는 부분 evidence가 여전히 유용한 값 수준 실패(범위 초과, 허용되지 않은 값)에 사용할 수 있습니다. 배포는 이러한 값 수준 실패에 대해 엄격한 거부를 구성할 수 있습니다.

진단은 높은 잘못된 레코드 비율에서 플러딩을 방지하기 위해 **비율 제한 및 집계**됩니다.

## 4. 구성 형식

### 4.1 실패 처리 정책 및 스키마 구성
 
스키마 검증 규칙 및 제약 조건은 `RawEvidenceSchema`에 선언되며, 검증 및 변환 실패에 대한 런타임 처리 정책은 해당 `EvidenceNormalizationRule`의 `failureHandling` 블록에 구성됩니다:
 
**RawEvidenceSchema (검증 규칙 및 제약 조건 선언):**
 
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
 
**EvidenceNormalizationRule `failureHandling` (런타임 실패 결과 구성):**
 
```yaml
apiVersion: score.dev/v1alpha1
kind: EvidenceNormalizationRule
metadata:
  name: vendor-a-npu-to-score-evidence
  version: 3.1.0
spec:
  # ... mappings and defaults ...
  failureHandling:
    missingRequiredField: {result: reject, emitAdapterDiagnostic: true}
    invalidValue: {result: emit-invalid-evidence, preserveOriginalValue: true}
    unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
    transformationError: {result: reject, emitAdapterDiagnostic: true}
```

### 4.2 어댑터 진단 레코드 (제안된 형태)

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

## 5. 동작 의미론

- 스키마 검증은 정규화 **이전에** 발생합니다. `reject`로 실패한 레코드는 Normalization Processor에 도달하지 않습니다.
- `emit-invalid-evidence` 또는 `emit-unknown`으로 실패한 레코드는 evidence 품질 메타데이터에 실패가 기록된 상태로 정규화를 진행합니다.
- 어댑터 진단은 Vehicle Evidence 계약과 분리된 운영/감사 데이터입니다 (수집 및 감사 로깅 설계 기준).
- 실패 처리 정책은 입력 계약의 일부이며 소스별로 구성할 수 있습니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-013, STKH-VEL-013 |
| 참조 예시 | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) `failureHandling` 블록 |