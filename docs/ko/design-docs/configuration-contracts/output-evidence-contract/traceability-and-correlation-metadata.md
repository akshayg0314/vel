# 추적성 및 상관 메타데이터 필드

> **관련:** FR-VEL-006 (소스 식별 메타데이터), FR-VEL-007 (관찰 시간 메타데이터), FR-VEL-008 (상관 메타데이터), STKH-VEL-006, STKH-VEL-007, STKH-VEL-008
> **참조 예시:** [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

## 1. 목적

이 문서는 정규화된 Vehicle Evidence 레코드의 **추적성(traceability)** 및 **상관(correlation)** 메타데이터 필드를 정의합니다. 이 필드는 관찰의 출처를 보존합니다: 데이터가 어디서 왔는지 (소스 식별자), 언제 관찰되었는지 (시간), 어떻게 변환되었는지 (규칙 버전), 그리고 관련 evidence 레코드를 연결하는 상관 식별자입니다.

## 2. 설계 위치

VEL 아키텍처에서 추적성, 식별자 및 시간 필드는 출력 Evidence 정의의 일부입니다. 소스 레코드의 식별자/타임스탬프 필드와 정규화 규칙 버전에서 추적성 강화기가 채웁니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 추적성 및 상관 모델

### 3.1 추적성 메타데이터 필드

출력 Evidence 정의의 `traceability` 섹션은 출처 필드를 선언합니다:

| 필드 | 유형 | 설명 |
| --- | --- | --- |
| `inputSchemaVersion` | string | 소스 레코드를 검증하는 데 사용된 입력 인터페이스 정의의 버전. |
| `normalizationRuleVersion` | string | 소스 레코드를 변환하는 데 사용된 정규화 매핑 규칙의 버전. |
| `sourceMessageId` | string | 원시 소스 메시지의 고유 식별자. |

이 필드는 evidence가 **어떻게** 생성되었는지 기록하여 감사 및 재현성을 가능하게 합니다.

### 3.2 식별 메타데이터 필드

`identity` 섹션은 소스 식별자 및 상관 식별자를 전달합니다:

| 필드 | 유형 | 설명 |
| --- | --- | --- |
| `sourceId` | string | 레코드를 생성한 소스. |
| `workloadId` | string | 워크로드 또는 실행 컨텍스트. |
| `executionInstanceId` | string | 실행 인스턴스. |
| `resourceId` | string | 관찰된 특정 리소스. |
| `correlationId` | string | 관련 evidence 레코드를 연결하는 상관 식별자. |

### 3.3 시간 메타데이터 필드

`time` 섹션은 관찰 시간을 전달합니다:

| 필드 | 유형 | 설명 |
| --- | --- | --- |
| `eventTimestampNs` | uint64 | 나노초 단위의 관찰 타임스탬프. |
| `clockDomain` | string | 타임스탬프의 클록 도메인 (`monotonic` 또는 `wall-clock`). |

### 3.4 상관 식별자 의미론

- `correlationId`는 동일한 소스 이벤트 또는 실행 인스턴스에서 시작된 여러 evidence 레코드를 연결합니다. 단일 `correlationId` 필드로 충분합니다. 레코드는 소스/워크로드/리소스별로 계층적으로 그룹화되고 추가로 `correlationId`로 그룹화되므로 여러 상관 차원 (예: 배치별)은 필요하지 않습니다.
- 참조 예시에서 `execution_id`는 `executionInstanceId`와 `correlationId` 모두에 매핑됩니다. 이는 한 실행의 모든 evidence 레코드가 동일한 상관 식별자를 공유함을 의미합니다.
- `correlationId` 필드는 출력 Evidence 정의에서 `required: true`로 선언됩니다 (참조 스키마 기준). 공통 매핑에서 소스의 `execution_id`를 `correlationId`에 매핑하여 **항상 채워집니다**. 이는 FR-VEL-008 ("소스 컨텍스트가 제공할 때")을 충족합니다. 현재 범위의 모든 소스가 `execution_id`를 제공하기 때문입니다.
- `clockDomain` 필드는 evidence 레코드별로 기록됩니다. 서로 다른 클록 도메인의 소스가 집계될 때 각 레코드는 자체 도메인을 유지합니다. 단일 레코드 내에서 혼합이 발생하지 않으므로 레코드별 도메인으로 충분합니다. 소스 간 클록 정렬은 VEL이 아닌 배포 수준에서 처리됩니다.
- 출처에는 이 문서의 섹션 3.1의 정확히 세 필드가 포함됩니다. 추가 출처 (예: 수집기 또는 어댑터 버전)는 필요하지 않습니다. 입력 인터페이스 정의 이름과 버전 (`inputSchemaVersion`이 참조)이 이미 수집기/어댑터를 식별하기 때문입니다.

### 3.5 전파 책임

추적성 강화기는 정규화 중에 이 필드를 채울 책임이 있습니다. 이 문서는 해당 필드에 대한 **계약**을 정의합니다.

## 4. 구성 형식

### 4.1 출력 Evidence 정의의 추적성, 식별자 및 시간 필드

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

### 4.2 레코드의 추적성/식별자/시간 섹션 예시

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

## 5. 동작 의미론

- 추적성, 식별자 및 시간 필드는 모든 정규화된 Vehicle Evidence 레코드에서 필수입니다.
- 이 필드는 소스 레코드의 식별자/타임스탬프 필드와 정규화 규칙 버전에서 추적성 강화기가 채웁니다.
- `correlationId`는 소비자가 동일한 소스 이벤트의 evidence 레코드를 그룹화할 수 있게 합니다. 소스의 상관 식별자가 제공될 때 해당 식별자에서 채워집니다.
- `sourceMessageId`는 원시 소스 메시지까지 정확한 출처를 제공합니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-006, FR-VEL-007, FR-VEL-008, STKH-VEL-006, STKH-VEL-007, STKH-VEL-008 |
| 참조 예시 | [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml) |