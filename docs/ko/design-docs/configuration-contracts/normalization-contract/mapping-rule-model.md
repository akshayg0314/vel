# 단위, Enum 및 상태 매핑 규칙 모델

> **관련:** FR-VEL-005 (소스 정규화), FR-VEL-015 (정규화 매핑 구성), STKH-VEL-005, STKH-VEL-015
> **참조 예시:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

## 1. 목적

이 문서는 **정규화 매핑(Normalization Mapping)** 구성 산출물, 즉 소스별 표현을 정규화된 Vehicle Evidence로 변환하는 규칙 모델을 정의합니다. 단위 변환, enum/상태 매핑, 매핑 규칙의 구조를 다룹니다.

## 2. 설계 위치

VEL 아키텍처에서 정규화 매핑은 구성 기반(configuration-driven)입니다. 각 소스는 입력 인터페이스 정의와 출력 Evidence 정의를 참조하는 자체 정규화 매핑을 가집니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 매핑 규칙 모델

### 3.1 구성 산출물로서의 정규화 매핑

정규화 매핑은 하드코딩이 아닌 구성으로 정의됩니다. 각 소스는 입력 인터페이스 정의와 출력 Evidence 정의를 참조하는 자체 정규화 매핑을 가집니다.

### 3.2 표준 계약 종류: EvidenceNormalizationRule

정규화 계약은 참조 예시를 반영하여 `EvidenceNormalizationRule` 종류를 사용합니다:

- [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

### 3.3 매핑 규칙 모델 구조

정규화 매핑은 다음 섹션으로 구성됩니다:

| 섹션 | 목적 |
| --- | --- |
| `compatibleInputSchemas` | 이 규칙이 적용되는 입력 스키마 (버전 범위 포함). |
| `outputSchema` | 이 규칙이 생성하는 출력 evidence 스키마. |
| `defaults` | 출력 필드의 기본값 (evidence 도메인, 품질, 시간, 추적성). |
| `commonMappings` | 모든 evidence 레코드에 적용되는 필드 간 매핑 (식별자, 시간, 추적성). |
| `evidenceMappings` | 하나 이상의 evidence 레코드를 생성하는 소스별 매핑 (값/상태/이벤트 매핑). |
| `failureHandling` | 매핑 실패 처리 방법. |

### 3.4 변환 연산

규칙 모델은 단위 변환, enum/상태 매핑, 타임스탬프 처리를 위한 변환 연산을 지원합니다:

| 연산 | 설명 | 예시 |
| --- | --- | --- |
| `scale` | 숫자 값을 계수로 곱함 (단위 변환). | `inference_latency_us` × 0.001 → ms. |
| `enumMap` | enum 값을 정규화된 상태로 매핑. | `device_state: 2` → `degraded`. |
| `timestamp` | 타임스탬프를 표준 형식으로 변환. | `timestamp_ns` 및 클록 도메인. |
| `lookup` | 테이블에서 값을 조회하여 evidence 생성. | `vendor_error_code: 0xA17` → `accelerator.execution.timeout`. |

### 3.5 매핑 규칙 범주

규칙 모델은 세 가지 매핑 범주를 구분합니다:

1. **공통 매핑(Common mappings)** — 모든 evidence 레코드에 적용됩니다 (식별자, 타임스탬프, 추적성). `commonMappings` 섹션입니다.
2. **Evidence 매핑(Evidence mappings)** — evidence 레코드를 생성하는 소스별 매핑입니다. 각 매핑은 다음을 가집니다:
   - `id` — 고유 식별자.
   - `when` — 매핑이 적용되는 조건. `fieldExists` (필드 존재) 및 `fieldEquals` (필드가 특정 값)를 지원합니다. 상태/오류 매핑에는 값 기반 조건이 필요합니다.
   - `output` — evidence 유형 및 정규화된 클래스.
   - `value` — 소스 경로, 대상 경로 및 변환.
   - `constants` — 고정 출력 값.
   - `lookup` — 선택적 조회 테이블.
3. **기본값(Defaults)** — 매핑 조건이 일치하지 않을 때 적용되는 기본값.

### 3.6 단위 변환 모델

단위 변환은 `factor`가 있는 `scale` 변환으로 표현됩니다. 소스 단위와 대상 단위는 매핑 컨텍스트에 암시됩니다 (예: 소스 `inference_latency_us`가 `unit: ms` 상수로 대상 `observation.value`로 스케일링됨). `scale` 계수는 단위 변환에 충분합니다. 오프셋 기반 변환 (예: 온도)은 지원되지 않습니다.

## 4. 구성 형식

### 4.1 정규화 매핑 예시

```yaml
apiVersion: score.dev/v1alpha1
kind: EvidenceNormalizationRule
metadata:
  name: vendor-a-npu-to-score-evidence
  version: 3.1.0
spec:
  compatibleInputSchemas:
    - {name: vendor-a-npu-status, versionRange: ">=1.2.0 <2.0.0"}
  outputSchema: {name: score-normalized-evidence, version: 1.0.0}
  defaults:
    evidenceDomain: accelerator
    quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
    time: {clockDomain: monotonic}
    traceability: {inputSchemaVersion: "1.2.0", normalizationRuleVersion: "3.1.0"}
  commonMappings:
    - {sourcePath: $.source_id, targetPath: $.identity.sourceId, required: true}
    - {sourcePath: $.workload_id, targetPath: $.identity.workloadId, required: true}
    - {sourcePath: $.execution_id, targetPath: $.identity.executionInstanceId, required: true}
    - {sourcePath: $.resource_id, targetPath: $.identity.resourceId, required: true}
    - {sourcePath: $.execution_id, targetPath: $.identity.correlationId, required: true}
    - {sourcePath: $.message_id, targetPath: $.traceability.sourceMessageId, required: true}
    - sourcePath: $.timestamp_ns
      targetPath: $.time.eventTimestampNs
      required: true
      transform: {operation: timestamp, sourceUnit: ns, clockDomain: monotonic}
  evidenceMappings:
    - id: inference-latency
      when: {fieldExists: $.inference_latency_us}
      output: {evidenceType: workload.execution.latency}
      value:
        sourcePath: $.inference_latency_us
        targetPath: $.observation.value
        transform: {operation: scale, factor: 0.001}
      constants:
        - {targetPath: $.observation.unit, value: ms}
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

## 5. 동작 의미론

- 정규화 매핑은 호환 가능한 입력 스키마를 참조하고 출력 스키마를 준수하는 레코드를 생성합니다.
- 공통 매핑은 모든 evidence 레코드에 적용됩니다. Evidence 매핑은 소스 메시지당 하나 이상의 evidence 레코드를 생성합니다.
- 단일 소스 메시지는 **여러** evidence 레코드를 생성할 수 있습니다 (예: 지연 시간용 하나, 장치 상태용 하나, 벤더 오류용 하나). 이것이 **표준 모델**입니다. 각 evidence 매핑은 별도의 레코드를 생성하며, 동일한 소스 메시지의 모든 레코드는 동일한 상관 식별자를 공유합니다. 레코드는 독립적으로 게시됩니다 (원자성 보장 없음). 소비자는 상관 식별자를 통해 레코드를 그룹화합니다. 레코드 간 순서는 보장되지 않지만 각 레코드는 독립적으로 유효합니다.
- 변환 연산은 소스 값이 변환되는 방식을 정의합니다 (scale, enumMap, timestamp, lookup). 이 네 가지 연산은 규칙 모델이 지원하는 완전한 집합입니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-005, FR-VEL-015, STKH-VEL-005, STKH-VEL-015 |
| 참조 예시 | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) |