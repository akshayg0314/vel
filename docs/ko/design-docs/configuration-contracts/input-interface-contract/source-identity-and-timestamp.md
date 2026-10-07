# 소스 식별 및 타임스탬프 요구사항

> **관련:** FR-VEL-006 (소스 식별 메타데이터), FR-VEL-007 (관찰 시간 메타데이터), FR-VEL-008 (상관 메타데이터), STKH-VEL-006, STKH-VEL-007, STKH-VEL-008
> **참조 예시:** [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml)

## 1. 목적

이 문서는 소스 레코드가 자신을 식별하는 방법(소스 식별)과 관찰 시간을 표현하는 방법(타임스탬프)을 정의합니다. 이러한 필드는 추적성 요구사항(FR-VEL-006, FR-VEL-007)의 전제 조건이며 Traceability Enricher에 의해 정규화된 Vehicle Evidence로 전파됩니다.

## 2. 설계 위치

VEL 아키텍처에서 식별 및 타임스탬프 필드는 입력 인터페이스 정의에 소스별로 선언됩니다. 이는 모든 소스가 제공해야 하는 공통 필수 필드이며 다운스트림 추적성 및 상관을 가능하게 합니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 소스 데이터 모델

### 3.1 소스 식별 필드

각 입력 인터페이스 정의는 다음 식별 필드를 반드시 선언해야 합니다:

| 필드 | 설명 | 필수 여부 | 참고 |
| --- | --- | --- | --- |
| `source_id` | 레코드를 생성한 소스(벤더/하드웨어/OS/S-CORE 모듈) 식별. | required | 정규화 출력에서 `identity.sourceId`로 사용. |
| `workload_id` | 레코드가 속한 워크로드 또는 실행 컨텍스트 식별. | required | `identity.workloadId`로 사용. |
| `resource_id` | 관찰 중인 특정 리소스(예: NPU-0, CPU-2) 식별. | required | `identity.resourceId`로 사용. |
| `execution_id` | 실행 인스턴스 식별. 상관 식별자 역할도 함. | required | `identity.executionInstanceId` 및 `identity.correlationId`로 사용. |
| `message_id` | 원시 메시지 자체의 고유 식별자. | required | `traceability.sourceMessageId`로 사용. execution_id와 구별됨; 여러 메시지가 동일한 execution_id를 공유할 수 있음. |

### 3.2 타임스탬프 요구사항

입력 인터페이스 정의는 타임스탬프 필드를 반드시 선언해야 합니다. 요구사항:

- **필드 이름 규칙:** `timestamp_ns` (클록 도메인에 따라 epoch 이후 또는 부팅 이후의 나노초).
- **유형:** `uint64` (나노초, 음수가 아님).
- **필수 여부:** `required: true`.
- **클록 도메인:** 입력 인터페이스 정의에서 **소스별로** 선언됩니다. 허용 도메인: `monotonic` (부팅 이후) 또는 `wall-clock` (epoch). 기본값은 런타임/하드웨어 메트릭의 경우 `monotonic`이며 참조 예시와 일치하지만 각 소스는 자체 도메인을 선언할 수 있습니다. 이를 통해 혼합 도메인 소스가 한 배포에서 공존할 수 있습니다.
- **정밀도:** 나노초. 소스가 더 거친 타임스탬프를 제공하는 경우 정규화기가 변환 작업을 사용하여 나노초로 변환해야 합니다.

### 3.3 계약에서의 식별/타임스탬프 배치

식별 및 타임스탬프 필드는 입력 인터페이스 정의의 `spec.fields` 목록의 일부입니다. 선언된 `name`으로 구별할 수 있으며 다운스트림 컴포넌트는 이름으로 참조합니다.

### 3.4 고유성 및 상관 의미론

- 각 원시 메시지는 정확한 출처를 허용하도록 고유한 `message_id`를 가져야 합니다. 소스가 고유성을 보장하지 않으면 VEL은 이를 적용하기 위해 합성 메시지 ID를 생성합니다.
- `execution_id`는 항상 상관 식별자입니다 (FR-VEL-008). 여러 메시지에서 공유될 수 있으며(예: 한 추론 실행의 모든 메시지), 다운스트림 소비자가 동일한 소스 이벤트에 속한 evidence 레코드를 그룹화할 수 있습니다. 별도의 상관 필드는 필요하지 않습니다.
- VEL은 소스 간 클록 동기화를 보장하지 않습니다; 각 소스는 자체 클록 도메인을 선언하며 소비자는 여러 소스의 레코드를 집계할 때 도메인 차이를 고려해야 합니다.

## 4. 구성 형식

### 4.1 입력 인터페이스 정의의 식별 및 타임스탬프 필드

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
    # ... 간결성을 위해 추가 소스별 필드 생략
```

### 4.2 클록 도메인 선언 (선택적 확장 제안)

```yaml
spec:
  owner: runtime-vendor
  encoding: json
  messageType: VendorANpuStatus
  additionalFields: reject
  time:
    clockDomain: monotonic   # 또는 wall-clock
    precision: ns
  fields: [...]
```

> 참고: `time` 블록은 제안된 확장입니다. 클록 도메인이 여기에 있는지 아니면 정규화 매핑 계약에 있는지는 공개 설계 항목입니다.

## 5. 동작 의미론

- 필수 식별 또는 타임스탬프 필드가 누락된 레코드는 스키마 무효이며 검증 실패 처리 정책에 따라 처리되어야 합니다.
- 식별 값은 Traceability Enricher에 의해 `identity` 및 `traceability` 섹션의 정규화 출력에 복사됩니다.
- 타임스탬프는 정규화된 Vehicle Evidence 레코드에서 클록 도메인과 함께 `time.eventTimestampNs`로 `time.clockDomain`으로 전파됩니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-006, FR-VEL-007, FR-VEL-008, STKH-VEL-006, STKH-VEL-007, STKH-VEL-008 |
| 참조 예시 | [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml) |