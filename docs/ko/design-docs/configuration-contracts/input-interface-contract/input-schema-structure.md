# 입력 스키마 구조 및 필수 필드

> **관련:** FR-VEL-013 (입력 인터페이스 구성), STKH-VEL-013
> **참조 예시:** [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml) — Vendor A NPU 런타임 상태의 표준 입력 인터페이스 정의.

## 1. 목적

이 문서는 **입력 인터페이스 정의** 구성 아티팩트를 정의합니다: VEL이 특정 소스에서 수락하는 원시 소스 데이터의 구조와 필수 필드. 수집기, 검증, 정규화, 품질, 지속성과 같은 모든 다운스트림 컴포넌트가 입력 스키마 형태를 알아야 하므로 Evidence Layer 입력 측의 기반 계약입니다.

## 2. 설계 위치

VEL 아키텍처에서 입력 계약은 구성 기반입니다. 각 소스(런타임 벤더, 하드웨어 장치, S-CORE 모듈)는 자체 입력 인터페이스 정의를 제공합니다. 플랫폼별 수집기는 공통 Evidence Layer와 분리됩니다 (FR-VEL-012). 이 문서는 데이터 표현 및 구성 형식만 지정하며 수집기 구현, 전송 동작 또는 파이프라인 처리는 정의하지 않습니다.

## 3. 소스 데이터 모델

### 3.1 입력 인터페이스 정의는 구성 아티팩트

입력 인터페이스는 하드코딩이 아닌 구성으로 정의됩니다. 각 소스는 자체 입력 인터페이스 정의를 제공하며, 이를 통해 공통 Evidence Layer를 수정하지 않고 새 소스를 추가할 수 있습니다 (FR-VEL-013).

### 3.2 표준 계약 종류: RawEvidenceSchema

입력 계약은 참조 예시 패키지를 미러링하는 `RawEvidenceSchema` 종류를 사용합니다.

### 3.3 계약 구조

각 입력 인터페이스 정의는 다음을 가집니다:

- **메타데이터** — 이름, 버전, 소유자.
- **인코딩** — 원시 소스 레코드의 직렬화 형식. `json`이 표준 인코딩입니다; 다른 인코딩(protobuf, CBOR)은 향후 확장입니다.
- **메시지 유형** — 소스 레코드의 논리적 메시지 유형 이름.
- **필드 목록** — 각 필드는 다음을 가집니다:
  - `name` — 구성 참조에 사용되는 논리적 필드 이름.
  - `path` — 원시 레코드의 필드에 대한 JSONPath (RFC 9535).
  - `type` — 데이터 유형 (string, number, integer, uint64, boolean, enum 등).
  - `required` — 필드가 반드시 존재해야 하는지 여부.
  - 선택적 제약 조건: `range`, `allowedValues`, `pattern`, `minimum`, `maximum`.
- **추가 필드 정책** — 스키마에 선언되지 않은 필드에 대한 `reject` 또는 `allow`.

입력 계약은 **검증기 중립적**입니다: 계약 형식은 특정 스키마 검증 엔진과 독립적입니다. 구체적인 검증기와 지원되는 제약 조건 유형은 배포 시 선택됩니다.

### 3.4 필수 필드

다운스트림 처리가 이에 의존하므로 다음 필드 범주는 모든 입력 인터페이스 정의에서 **필수**입니다:

| 범주 | 목적 |
| --- | --- |
| `message_id` | 추적성을 위한 원시 소스 메시지의 고유 식별자. |
| `source_id` | 레코드를 생성한 소스 식별. |
| `workload_id` | 워크로드 또는 실행 컨텍스트 식별. |
| `execution_id` | 실행 인스턴스 식별; 상관 식별자로도 사용. |
| `resource_id` | 관찰 중인 특정 하드웨어/소프트웨어 리소스 식별. |
| `timestamp_ns` | 관찰 타임스탬프 (나노초). |

이들은 **공통 필수 필드**입니다. 소스별 필드(예: `inference_latency_us`, `device_state`)는 자체 필수 여부와 함께 추가 필드로 선언됩니다.

### 3.5 필드 유형 분류

입력 계약은 다음 필드 유형을 지원합니다:

| 유형 | 설명 | 예시 |
| --- | --- | --- |
| `string` | UTF-8 텍스트 | `source_id` |
| `uint64` / `uint32` / `uint8` | 부호 없는 정수 | `timestamp_ns`, `device_state` |
| `int64` / `int32` | 부호 있는 정수 | 부호 있는 메트릭 |
| `number` | 부동 소수점 | ms 단위 지연 시간 |
| `boolean` | 참/거짓 | 플래그 |
| `enum` | 제한된 값 집합 | `device_state` 허용 값 |
| `array` | 값 목록 | 다중 값 메트릭 |

### 3.6 제약 조건

필드는 Schema Validator가 적용할 제약 조건을 선언할 수 있습니다:

- `range: {minimum, maximum}` — 숫자 범위.
- `allowedValues: [..]` — 열거된 값.
- `pattern` — 문자열의 정규식 패턴.
- `required: true/false` — 존재 요구 사항.

## 4. 구성 형식

### 4.1 입력 인터페이스 정의 예시

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

### 4.2 원시 소스 레코드 예시 (JSON)

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

## 5. 동작 의미론

- 소스 레코드는 모든 `required: true` 필드가 존재하고, 존재하는 필드가 선언된 유형과 일치하며, 모든 선언된 제약 조건을 충족할 때 **스키마 유효**입니다.
- `additionalFields` 정책은 선언되지 않은 필드가 거부(`reject`) 또는 허용(`allow`)되는지 결정합니다. 엄격한 계약을 적용하려면 기본 권장 사항은 `reject`입니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-013, STKH-VEL-013 |
| 참조 예시 | [`vendor-a-npu-status-v1.2.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/schemas/vendor-a-npu-status-v1.2.0.yaml) |