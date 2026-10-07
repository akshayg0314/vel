# Source Collector 인터페이스 설계

> **참고:** [s-core-state-collection/s-core-collector-common-design.md](s-core-state-collection/s-core-collector-common-design.md) — 이 공통 인터페이스의 S-CORE 특화 구체적 적용 사례입니다.

## 1. 목적

이 문서는 **공통 Source Collector 인터페이스**를 정의합니다: 수집기 I/O API, Evidence Runtime 내 수집기 라이프사이클, 수집기 오류/진단 보고. 이 세 가지 측면은 수집기가 어떤 소스 카테고리를 읽든 **모든 VEL 수집기에 동일하게 적용**됩니다:

| 소스 카테고리 | 예시 소스 | 수집기 설계 문서 |
| --- | --- | --- |
| 런타임 메트릭 | CPU, GPU, NPU 사용률 (예: vendor-a-npu) | 아직 작성되지 않음 |
| 하드웨어 데이터 | 센서 상태, 장치 상태 | 아직 작성되지 않음 |
| S-CORE 모듈 상태 | Lifecycle | [s-core-state-collection/s-core-collector-common-design.md](s-core-state-collection/s-core-collector-common-design.md) |

이 문서는 입력 인터페이스 계약(Input Interface Contract)의 input-schema-structure 문서에 대응하는 수집기 인터페이스 짝입니다: 해당 계약은 모든 소스가 따르는 공통 **데이터 형태**(`RawEvidenceSchema`)를 정의하고, 이 문서는 모든 수집기가 따르는 공통 **컴포넌트 동작**(I/O 계약, 라이프사이클, 진단)을 정의합니다. 둘 다 런타임, 하드웨어, S-CORE 모듈 소스 전반에 균일하게 적용되며, 어느 쪽도 S-CORE 특화가 아닙니다.

**충족 요구사항:** FR-VEL-002 (접근 가능한 메트릭 소스 메커니즘), FR-VEL-003 (S-CORE 모듈 상태 수집), FR-VEL-012 (플랫폼 수집기 분리).

## 2. 설계 위치

- VEL은 **evidence 생산자**입니다; 수집기의 책임은 schema에 부합하는 raw 상태 레코드를 생성하는 것으로 끝납니다. [VEL 아키텍처 설계 초안](../../vel_architectural_design_draft_kor.md) (Section 4, 컴포넌트 책임 표)에 따르면: "Source Collectors | 구성된 소스 데이터를 읽고 소스 컨텍스트를 제공 | 플랫폼별 구현; **정규화 정책 없음**." 수집기는 정규화 매핑을 직접 적용하지 않습니다 — 이는 Evidence Processing의 Normalization Processor의 책임입니다.
- 플랫폼별 수집기 구현(런타임, 하드웨어, S-CORE)은 이 공통 인터페이스와 분리되어 있습니다(FR-VEL-012): 새로운 소스 카테고리는 이 인터페이스를 구현하는 새 수집기를 추가함으로써 도입되며, 인터페이스 자체를 변경하지 않습니다.
- 이 문서는 **전송 방식에 독립적**입니다: DDS, S-CORE API 바인딩, 파일, 명령을 전송 방식으로 고정하지 않습니다. 수집기의 `source` 구성(transport, api/topic, messageType)에서 구체적인 수집기가 자신의 전송 바인딩을 선언합니다.

## 3. 수집기 I/O API

### 3.1 목적

모든 VEL 수집기는 **균일한 I/O API**를 노출하여, 소스 카테고리(런타임 메트릭, 하드웨어 상태, S-CORE 모듈 상태)와 무관하게 Evidence Runtime이 단일하고 예측 가능한 인터페이스를 통해 상호작용할 수 있도록 합니다.

### 3.2 입력 계약

모든 수집기는 다음 입력 매개변수를 받습니다:

| 매개변수 | 타입 | 필수 | 설명 |
|-----------|------|----------|-------------|
| `collector_id` | `string` | 예 | 수집기 인스턴스의 고유 식별자 (수집기 YAML의 `collector_id`와 일치). |
| `source_ref` | `string` | 예 | 수집 대상 소스에 대한 참조 (예: `score.lifecycle`, `vendor-a.npu`, `hardware.sensor-0`). |
| `collection_trigger` | `enum` | 예 | 다음 중 하나: `periodic`, `event`, `on_demand`. |
| `collection_window` | `object` | 아니오 | 경계가 있는 수집을 위한 선택적 시간 창(`start`, `end`). |
| `parameters` | `object` | 아니오 | 소스별 수집 매개변수 (예: 필터 기준). |

### 3.3 출력 계약

모든 수집기는 다음 출력을 생성합니다:

| 출력 | 타입 | 설명 |
|--------|------|-------------|
| `raw_state_records` | `array` | 소스의 `RawEvidenceSchema`(입력 인터페이스 정의)에 부합하는 raw 상태 레코드 — 정규화된 evidence가 **아닙니다**. 정규화는 Evidence Processing 영역(Schema Validator, Normalization Processor)이 하류에서 수행하며, 수집기는 이에 대한 정책을 갖지 않습니다. |
| `collection_metadata` | `object` | 수집 실행에 대한 메타데이터 (타임스탬프, 지속 시간, 레코드 수). |
| `diagnostics` | `object` | 진단 정보 (Section 5 참조). |

### 3.4 I/O 동작

- **동기** 수집: 수집기는 소스 읽기가 완료된 직후 raw 상태 레코드를 반환합니다.
- **비동기** 수집: 수집기는 Evidence Runtime이 폴링하여 준비된 raw 상태 레코드를 가져올 수 있는 `collection_handle`을 반환합니다.
- **멱등성**: 동일한 `collection_trigger`와 `collection_window`로 반복 수집하면 결정적인 결과를 생성합니다.
- **타임아웃**: 소스 읽기가 구성된 타임아웃을 초과하면, 수집기는 타임아웃 진단을 반환하고 raw 상태 레코드는 반환하지 않습니다.

## 4. 수집기 라이프사이클

### 4.1 라이프사이클 상태

모든 수집기는 Evidence Runtime 내에서 동일한 라이프사이클 상태 머신을 따릅니다:

![수집기 라이프사이클 상태 머신](../../features/assets/Collector_lifecycle_state_machine.svg)

[PlantUML 원본](../../features/diagrams/Collector_lifecycle_state_machine.puml)

| 상태 | 설명 |
|-------|-------------|
| `PROVISIONED` | 수집기가 구성되고 Evidence Runtime에 등록되었지만 아직 활성화되지 않음. |
| `ACTIVE` | 수집기가 수집 요청을 받을 준비가 됨. |
| `RUNNING` | 수집기가 현재 수집 주기를 실행 중. |
| `SUSPENDED` | 수집기가 일시적으로 일시 중지됨 (예: 리소스 제약, 유지보수). |
| `ERROR` | 수집기가 복구 불가능한 오류를 만나 계속할 수 없음. |
| `TERMINATED` | 수집기가 영구적으로 종료됨. |

### 4.2 상태 전이

| From | To | 트리거 |
|------|----|---------|
| `PROVISIONED` | `ACTIVE` | Evidence Runtime이 `activate` 명령을 전송. |
| `ACTIVE` | `RUNNING` | 수집 요청 수신. |
| `RUNNING` | `ACTIVE` | 수집 주기가 성공적으로 완료. |
| `ACTIVE` | `SUSPENDED` | Evidence Runtime이 `suspend` 명령을 전송. |
| `SUSPENDED` | `ACTIVE` | Evidence Runtime이 `resume` 명령을 전송. |
| `ACTIVE` | `ERROR` | 수집 중 복구 불가능한 오류. |
| `RUNNING` | `ERROR` | 수집 중 복구 불가능한 오류. |
| `ERROR` | `TERMINATED` | Evidence Runtime이 `terminate` 명령을 전송. |
| `SUSPENDED` | `TERMINATED` | Evidence Runtime이 `terminate` 명령을 전송. |
| `ACTIVE` | `TERMINATED` | Evidence Runtime이 `terminate` 명령을 전송. |

### 4.3 상태 보고

각 수집기는 Evidence Runtime이 폴링하는 **상태 프로브**를 노출합니다:

```yaml
health_status:
  state: ACTIVE              # 현재 라이프사이클 상태
  last_collection: "2026-10-01T08:00:00Z"
  collection_count: 42
  error_count: 0
  last_error: null
  uptime_seconds: 86400
```

## 5. 수집기 오류 및 진단 보고

### 5.1 오류 카테고리

모든 수집기는 소스 카테고리와 무관하게 공통 오류 분류 체계를 사용합니다:

| 카테고리 | 코드 | 설명 | 복구 가능 |
|----------|------|-------------|-------------|
| `CONFIGURATION_ERROR` | `1001` | 잘못되었거나 누락된 수집기 구성. | 예 |
| `SOURCE_UNAVAILABLE` | `1002` | 소스(S-CORE API, 하드웨어 인터페이스, 런타임 엔드포인트)에 접근할 수 없거나 응답하지 않음. | 예 |
| `AUTHENTICATION_ERROR` | `1003` | 소스 접근 시 인증/권한 부여 실패. | 아니오 |
| `TIMEOUT` | `1004` | 소스 읽기가 구성된 타임아웃을 초과. | 예 |
| `SCHEMA_VIOLATION` | `1005` | 수집된 데이터가 예상된 `RawEvidenceSchema`에 부합하지 않음. | 예 |
| `NORMALIZATION_ERROR` | `1006` | raw 소스 상태에서 evidence 레코드로의 매핑 실패. | 예 |
| `RESOURCE_EXHAUSTION` | `1007` | 수집기가 메모리, 디스크 또는 기타 리소스를 소진함. | 예 |
| `INTERNAL_ERROR` | `1999` | 예상치 못한 내부 오류. | 아니오 |

### 5.2 진단 출력

모든 수집기는 `AdapterDiagnostic` schema를 균일하게 사용합니다:

```yaml
adapter_diagnostic:
  collector_id: "lifecycle-collector"
  source_ref: "score.lifecycle"
  timestamp: "2026-10-01T08:00:00.123Z"
  severity: ERROR                 # INFO | WARNING | ERROR | FATAL
  error_code: "1002"
  error_category: "SOURCE_UNAVAILABLE"
  message: "S-CORE lifecycle API unreachable"
  retry_count: 3
  recovery_hint: "Check S-CORE lifecycle service availability"
  evidence_record_id: null        # 영향받은 evidence에 대한 선택적 링크
```

### 5.3 실패 처리 통합

- **일시적 오류** (예: `SOURCE_UNAVAILABLE`, `TIMEOUT`): 수집기는 지수 백오프로 재시도(최대 3회).
- **영구적 오류** (예: `AUTHENTICATION_ERROR`, `INTERNAL_ERROR`): 수집기는 `ERROR` 상태로 전이하고 Evidence Runtime에 통지.
- **Schema 위반**: 수집기는 진단을 기록하고, 잘못된 레코드를 폐기하며, 수집을 계속함.
- **Evidence 품질 영향**: evidence 수집을 방해하는 모든 오류는 evidence 품질 메타데이터에 반영됨 (예: `completeness`, `integrity` 플래그).

### 5.4 속도 제한

수집기는 로그 범람을 방지하기 위해 **진단 속도 제한**을 구현합니다:

- 수집기당 60초 창에 최대 10개의 진단.
- 초과 진단은 요약 진단으로 집계됨.
- Evidence Runtime은 수집기 구성 YAML을 통해 속도 제한을 구성할 수 있음.

## 6. 공통 수집기 구성

모든 수집기는 이 공통 구성 구조를 공유합니다 (`transport`/`api` 바인딩과 같은 소스별 필드는 각 소스 카테고리의 자체 설계 문서에서 정의됨):

```yaml
collector:
  kind: VehicleEvidenceCollector
  version: 1.0.0
  collector_id: "<source>-collector"
  source:
    transport: <s-core-api | dds | file | command | os-interface>
    api: "<source-specific-api-or-topic>"
  collection:
    trigger: periodic
    interval_seconds: 60
    timeout_seconds: 10
    retry:
      max_retries: 3
      backoff_seconds: 2
  diagnostics:
    rate_limit_per_minute: 10
  lifecycle:
    auto_start: true
```

## 7. 소스 카테고리 설계 문서와의 관계

| 측면 | 문서 |
|--------|----------|
| 수집기 I/O API | **이 문서** (공통, 모든 소스 카테고리) |
| 수집기 라이프사이클 | **이 문서** (공통, 모든 소스 카테고리) |
| 오류 및 진단 보고 | **이 문서** (공통, 모든 소스 카테고리) |
| S-CORE 모듈 수집기 관례 | [s-core-state-collection/s-core-collector-common-design.md](s-core-state-collection/s-core-collector-common-design.md) |
| S-CORE 모듈 API 표면 / 상태 필드 / 전송 경로 | [s-core-state-collection/collectors/lifecycle-collector-design.md](s-core-state-collection/collectors/lifecycle-collector-design.md) (모듈별) |
| 런타임 및 하드웨어 수집기 관례 | 아직 작성되지 않음 |

## 8. 추적성

| 항목 | 참조 |
| --- | --- |
| 설계 | Source Collector 인터페이스 설계 |
| 요구사항 | FR-VEL-002, FR-VEL-003, FR-VEL-012 |
| 구체적 적용 사례 | S-CORE 상태 수집 (작성됨), 런타임 및 하드웨어 수집 (아직 작성되지 않음) |
| 참조 아키텍처 | [`vel_architectural_design_draft_kor.md`](../../vel_architectural_design_draft_kor.md) |
