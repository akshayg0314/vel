# 정규화된 Evidence 레코드 구조

> **관련:** FR-VEL-004 (Vehicle Evidence), FR-VEL-011 (Evidence 게시), FR-VEL-014 (출력 Evidence 구성), STKH-VEL-001, STKH-VEL-011, STKH-VEL-014
> **참조 예시:** [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

## 1. 목적

이 문서는 **출력 Evidence 정의(Output Evidence Definition)** 구성 산출물, 즉 정규화된 Vehicle Evidence 레코드의 표준 구조를 정의합니다. 여섯 개의 표준 섹션 (`header`, `identity`, `observation`, `time`, `quality`, `traceability`)과 필수 필드를 지정합니다.

## 2. 설계 위치

VEL 아키텍처에서 출력 evidence 계약은 구성 기반입니다. 모든 다운스트림 소비자가 의존하는 표준 데이터 표현을 정의합니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다. 게시 동작이나 영속성은 정의하지 않습니다.

## 3. 출력 Evidence 레코드 모델

### 3.1 표준 계약 종류: NormalizedEvidenceSchema

출력 evidence 계약은 참조 예시를 반영하여 `NormalizedEvidenceSchema` 종류를 사용합니다:

- [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

### 3.2 레코드 구조

각 정규화된 Vehicle Evidence 레코드는 출력 Evidence 정의를 준수해야 합니다. `requiredSections` 목록은 필수 최상위 섹션을 정의합니다. 배포는 여섯 개의 표준 섹션을 넘어 배포별 섹션을 추가해서는 안 됩니다:

| 섹션 | 설명 |
| --- | --- |
| `header` | Evidence 식별 및 메타데이터. |
| `identity` | 소스 식별자 및 상관 식별자. |
| `observation` | 관찰된 값 또는 상태. |
| `time` | 관찰 타임스탬프 및 클록 도메인. |
| `quality` | Evidence 품질 메타데이터 (유효성, 신선도, 완전성, 매핑 품질). |
| `traceability` | 출처 (입력 스키마 버전, 정규화 규칙 버전, 소스 메시지 ID). |

### 3.3 섹션 세부 사항

- **header** — evidence 레코드를 고유하게 식별하는 `evidenceId`(UUID v4)를 포함합니다. UUID (v4)로 생성되며 전역적으로 고유하고 조정이 필요 없으며 분산 수집기를 지원합니다.
- **identity** — `sourceId`, `workloadId`, `executionInstanceId`, `resourceId`, `correlationId`를 포함합니다.
- **observation** — 값 기반, 상태 기반 또는 둘 다일 수 있습니다. 레코드가 의미 있는 evidence가 되려면 **최소 하나의 observation 필드가 채워져야 합니다**.
- **time** — `eventTimestampNs` 및 `clockDomain`을 포함합니다.
- **quality** — `validity`, `freshness`, `completeness`, `mappingQuality`를 포함합니다.
- **traceability** — `inputSchemaVersion`, `normalizationRuleVersion`, `sourceMessageId`를 포함합니다.

### 3.4 집계 데이터 형식

집계 데이터 형식은 `json`입니다.

## 4. 구성 형식

### 4.1 출력 Evidence 정의 예시

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

### 4.2 정규화된 Vehicle Evidence 레코드 예시

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

## 5. 동작 의미론

- 모든 정규화된 Vehicle Evidence 레코드는 출력 Evidence 정의를 준수해야 합니다.
- `requiredSections` 목록은 필수 최상위 섹션을 정의합니다. 배포는 여섯 개의 표준 섹션 (`header`, `identity`, `observation`, `time`, `quality`, `traceability`)을 넘어 배포별 섹션을 추가해서는 안 됩니다.
- `header.evidenceId`는 evidence 레코드를 고유하게 식별합니다. **UUID (v4)** 로 생성되며 전역적으로 고유하고 조정이 필요 없으며 분산 수집기를 지원합니다.
- `observation` 섹션은 값 기반, 상태 기반 또는 둘 다일 수 있습니다. 레코드가 의미 있는 evidence가 되려면 **최소 하나의 observation 필드가 채워져야 합니다**.
- 집계 데이터 형식은 `json`입니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-004, FR-VEL-011, FR-VEL-014, STKH-VEL-001, STKH-VEL-011, STKH-VEL-014 |
| 참조 예시 | [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml) |