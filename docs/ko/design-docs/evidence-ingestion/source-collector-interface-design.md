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

### 3.5 설정 기반(config-driven) 상태 매핑 (공통 수집기 내부 구조)

수집기가 내부적으로 `raw_state_records`(Section 3.3)를 구성하는 방식 또한 각 소스 카테고리나 모듈이 재구현하는 것이 아니라 그 자체로 공통 인프라이며, 핵심적으로 — 새 소스를 추가한다고 해서 새 소스별 클래스를 작성해야 한다는 뜻은 **아닙니다**:

- **`IStateFieldBindingProvider`** — 모든 소스의 필드 값이 읽히는 하나의 공유되고 모듈 중립적인 인터페이스(`field_names()` / `read_field(name)` / `subscribe(on_update)`). 이 인터페이스는 또한 **안정적인 플러그인 계약**(Section 3.6)입니다: 주어진 소스의 구현이 VEL 코어 내부에 살든(제네릭 바인딩) VEL 코어 밖에서 독립적으로 로드된 플러그인으로 살든, `ConfigDrivenStateMapper`와 그 외 모든 공통 컴포넌트는 이 하나의 인터페이스만 볼 수 있습니다. `read_field()`는 동기/풀 기반 소스를 다루며, `subscribe()`는 이벤트 기반/푸시 소스(예: Lifecycle의 `ext_monitor_notify` 콜백)를 다룹니다 — 구현이 `subscribe()`로 등록하고 최신 값을 내부에 캐시하면, 이후 `read_field()` 호출이 그 값을 반환합니다. **푸시 기반 소스의 일관성 규약:** 각 푸시 이벤트마다 구현은 전체 이벤트 페이로드로부터 하나의 불변 스냅샷을 생성하고 이를 원자적으로 교체합니다(예: `std::atomic<std::shared_ptr<const T>>`). 필드 단위 가변 상태를 갱신하는 방식은 사용하지 않습니다. `ConfigDrivenStateMapper`는 각 수집 주기 시작 시 스냅샷을 한 번만 로드하므로, 하나의 raw 레코드 내에서 푸시 기반 필드에 대한 모든 `read_field()` 호출은 동일한 이벤트 세대를 참조합니다 — 수집 주기 도중 새로운 푸시 이벤트가 도착하더라도 마찬가지입니다. 이를 통해 서로 다른 두 이벤트의 필드가 섞인 불일치(torn) 레코드를 방지합니다. **풀 전용(pull-only) 소스:** `subscribe()`는 소스 범주와 무관하게 모든 어댑터의 인터페이스 시그니처에 포함되어 있으므로, Evidence Runtime과 `ConfigDrivenStateMapper`는 소스별 조건 분기 없이 이를 균일하게 호출할 수 있습니다. 이벤트 기반/푸시 API가 없는 순수 풀 기반 소스의 구현은 `subscribe()`를 no-op으로 구현합니다 — `on_update` 콜백을 받아 저장하거나(또는 단순히 반환하고) 실제로 호출하지는 않습니다. 해당 구현이 노출하는 모든 필드는 `read_field()`만으로 충분히 제공되기 때문입니다. 이는 오류나 누락된 기능이 아니라, 풀 전용 소스에 대한 예상되고 유효한 구현입니다.
- **주어진 소스에 대해 `IStateFieldBindingProvider`를 실현하는 두 가지 방법** (수집기별로 구성으로 선택 — Section 3.6 참조):
  1. **제네릭 바인딩 (새 코드가 전혀 필요 없음).** VEL 코어에 내장된 설정 기반 구현으로, 소스의 `fieldBindingProvider` 구성을 해석하여 기존 전송/공급자 기능(예: 제네릭 DDS/파일/명령 리더)을 재사용합니다 — 어떤 필드가 존재하는지, 각 필드가 기본 전송 읽기/구독 호출에 어떻게 매핑되는지, 어떤 것이 포함/제외되는지. 이것이 **기본, 선호 경로**입니다: 대부분의 소스는 `RawEvidenceSchema`, `EvidenceNormalizationRule`, 수집기 구성 외에는 아무것도 필요로 하지 않습니다.
  2. **플러그인 바인딩 (제네릭 바인딩이 네이티브 인터페이스를 표현할 수 없을 때에만).** 소스의 네이티브 API가 제네릭 바인딩의 선언적 전송/공급자 기능으로 실제로 도달할 수 없을 때(예: Lifecycle의 두 개의 서로 다른 `mw::com` 서비스 연결, 하나는 풀 하나는 푸시, 각각 소스별 페이로드 디코딩 필요), 해당 모듈의 바인딩 로직은 **독립적으로 빌드 및 배포된 플러그인** — VEL 코어와 별도로 빌드/버전 관리되는 바인딩 아티팩트로, 이 안정적인 `IStateFieldBindingProvider` 계약만 구현하고 Section 3.6에 따라 런타임에 로드됩니다. 이 플러그인은 그러한 모듈이 필요한 **유일한** 소스별 코드입니다; VEL 코어에 컴파일되지 않으며, VEL 코어 수집기 하위 클래스로 추가되지 않으며, 하드코딩된 팩토리 목록에 등록되지 않습니다.
- **`ConfigDrivenStateMapper`** — 한 번만 구축되어 소스 카테고리나 바인딩 종류와 무관하게 모든 수집기가 공유합니다. 소스의 `RawEvidenceSchema` 구성과 `IStateFieldBindingProvider`(제네릭 또는 플러그인 제공 — 매퍼는 차이를 알 수 없고 알 필요도 없음)가 주어지면, 구성된 필드(name/path/type/required/`conversion` — Section 3.7)를 순회하며 바인딩에서 각 값을 가져와 schema에 부합하는 `raw_state_records`를 일반적으로 구성합니다. 소스별 로직을 포함하지 않습니다.

이는 FR-VEL-012 (플랫폼 수집기 분리)와 FR-VEL-013 (입력 인터페이스 구성)을 schema 수준뿐만 아니라 수집기 구현 수준에서도 직접 충족합니다: 새로운 소스를 추가하려면 새로운 `RawEvidenceSchema`, 새로운 `EvidenceNormalizationRule`, 수집기 구성만 필요합니다 — 그리고, **제네릭 바인딩이 소스의 네이티브 인터페이스에 도달할 수 없는 경우에만**, 독립적으로 빌드/배포된 플러그인 하나가 필요합니다. 어느 쪽이든 VEL 코어 자체는 수정되지 않으며, 수집기 하위 클래스가 추가되지 않으며, 하드코딩된 팩토리/등록 목록이 갱신되지 않으며, 모듈을 추가하기 위해 VEL 코어가 다시 빌드되지 않습니다.

![공통 수집기 및 설정 기반 상태 매핑 클래스 다이어그램](../../../features/assets/Collector_common_class_diagram.svg)

[PlantUML 소스](../../../features/diagrams/Collector_common_class_diagram.puml)

### 3.6 플러그인 검색 및 로딩

수집기의 구성은 Section 3.5에 따라, 해당 수집기의 `IStateFieldBindingProvider`가 내장 제네릭 바인딩인지 외부에서 로드된 플러그인인지 선택합니다:

```yaml
spec:
  fieldBindingProvider:
    kind: generic   # 또는: plugin
    plugin:                                   # kind: plugin일 때만 존재
      library: plugins/<module>-field-bindings
      abiVersion: 1
      entryPoint: vel_plugin_create_binding_provider
      settings:                               # 불투명, 플러그인 정의 설정
        transport: s-core-api
        api: <module>-state-api
```

- **`kind: generic`** — Evidence Runtime이 내장 제네릭 바인딩을 직접 인스턴스화하며, 라이브러리가 로드되지 않습니다.
- **`kind: plugin`** — Evidence Runtime의 **`PluginLoader`** 컴포넌트(공유, VEL 코어의 일부, 모듈 특화가 아님)가 이 구성에 의해서만 구동되어 다음을 수행합니다:
  1. `plugin.library`를 Evidence Runtime의 구성된 플러그인 디렉토리에 대해 해석합니다. 해석된 경로는 해당 디렉토리 안에 있어야 하며, 이를 벗어나는 경로(외부 절대 경로, `..` 세그먼트)는 `CONFIGURATION_ERROR`로 거부됩니다.
  2. 플러그인 아티팩트를 로드합니다(배포 모델 — `dlopen`을 통한 인프로세스 공유 라이브러리 또는 IPC를 통한 아웃오브프로세스 프로바이더 — 은 배포 결정이며, 아래 C-ABI 계약은 두 경우 모두 동일합니다).
  3. 이름(`plugin.entryPoint`, 예: `vel_plugin_create_binding_provider`)으로 **안정적이고 버전 관리된 C-ABI 진입점**을 해석합니다.
  4. 플러그인이 보고한 ABI 버전(내보낸 `extern "C" int vel_plugin_abi_version()`, 또는 진입점이 반환하는 동등한 버전 필드)을 VEL 코어가 지원하는 ABI 버전과 대조하고, 불일치 시 로드를 거부합니다(`CONFIGURATION_ERROR`).
  5. 진입점을 호출하여 `plugin.settings`(VEL 코어에는 불투명, 플러그인에만 의미 있음)를 전달하고, `IStateFieldBindingProvider*` 인스턴스와 함께 일치하는 `extern "C" void vel_plugin_destroy_binding_provider(IStateFieldBindingProvider*)` 정리 함수를 받습니다.
- **실패 처리.** 1-5단계의 모든 실패(라이브러리 없음, 심볼 누락, ABI 불일치, 인스턴스화 오류)는 `CollectorDiagnostics`를 통해 `CONFIGURATION_ERROR`로 보고됩니다; 해당 수집기는 시작하지 않으며, 다른 수집기는 영향을 받지 않습니다. 수집기 중지 시 로더는 플러그인의 destroy 함수를 호출한 다음 플러그인을 언로드합니다.
- **하드코딩된 레지스트리 없음.** `PluginLoader`는 알려진 플러그인이나 모듈 이름의 내장 목록을 보유하지 않습니다. 모든 플러그인은 해당 수집기 자체 구성에 선언된 `library`/`entryPoint` 경로에서만 검색됩니다. 새 플러그인 기반 모듈을 추가한다는 것은: 플러그인을 자체 독립 버전 아티팩트로 빌드하고, VEL 옆에 배포하고, 새 수집기 구성을 그 아티팩트를 가리키게 하는 것입니다 — **VEL 코어 소스 변경 0, 새 수집기 하위 클래스 0, VEL 코어 재빌드 0.**
- **안정적인 플러그인 계약.** 플러그인은 `IStateFieldBindingProvider`(Section 3.5)와 위의 두 C-ABI 진입점 외에는 아무것도 구현하지 않습니다. `ConfigDrivenStateMapper`, `CollectorDiagnostics`, 스키마 검증, 정규화를 구현하지 않습니다 — 그것들은 공유 VEL 코어 인프라로 남아 있으며, 어떤 플러그인도 건드리지 않습니다.
- **배포 모델 참고.** 플러그인 배포 모델(인프로세스 공유 라이브러리 대 아웃오브프로세스 프로바이더)은 이 계약에 의해 의도적으로 고정되지 않습니다; 둘 다 위의 안정적인 C-ABI 진입점과 호환됩니다. 인프로세스 공유 라이브러리를 선택하면 플러그인 충돌이 호스팅 Evidence Runtime 프로세스에 영향을 줄 수 있습니다; 이 트레이드오프는 더 낮은 지연 시간과 더 단순한 배포를 위해 수용되며, 완화책은 `CollectorDiagnostics`/`health_probe()`(Section 4.3)와 Evidence Runtime 호스트의 프로세스 수준 감독(예: 충돌 시 재시작)입니다. 아웃오브프로세스 프로바이더는 대신 IPC 경계 뒤에서 플러그인 결함을 격리하지만, 추가 배포 및 IPC 오버헤드 비용이 듭니다.

### 3.7 필드 값 변환 (공통 수집기 내부 구조)

인터페이스별 상태 및 오류 값(예: Lifecycle `ProcessState` enum 값)은 바인딩(제네릭 또는 플러그인)의 소스별 코드가 아니라, `ConfigDrivenStateMapper` 내부의 **공유되고 설정 기반인 변환 단계**에 의해 evidence 문자열 표현으로 변환됩니다:

- 각 `RawEvidenceSchema` 필드는 `type`/`path`/`required` 속성과 함께 선택적 `conversion` 블록을 선언할 **수** 있습니다:
  ```yaml
  - name: state
    path: $.state
    type: string
    required: true
    conversion:
      operation: enumMap
      sourceValueKind: enum
      values: {kIdle: idle, kStarting: starting, kRunning: running, kTerminating: terminating, kTerminated: terminated, kFailed: failed}
      unmappedValue: unknown
  ```
- 바인딩(제네릭 또는 플러그인)은 `read_field()`/`subscribe()`에서 소스에서 관찰된 그대로의 **raw 네이티브 값**(예: 리터럴 enum 라벨 `kRunning`)을 반환합니다 — 자체적으로 문자열 변환을 수행하지 않습니다.
- `ConfigDrivenStateMapper`는 해당 raw 값을 필드의 `conversion.values` 맵에서 조회하고 매핑된 evidence 문자열로 대체합니다. raw 값이 `values`에 **없으면**(진정으로 새롭거나 인식되지 않은 값 — 예: VEL 구성이 아직 매핑하도록 갱신되지 않은 미래의 `ProcessState` 멤버), 매퍼는 표준 문자열 `unmappedValue`(기본값 `"unknown"`)를 기록합니다 — raw 레코드는 거부되지 않으며, 원래 raw 값은 그대로 보존되지 않습니다. 이는 인식되지 않은 인터페이스별 값이 보존되거나 거부되지 않고 표준 `"unknown"` 문자열로 표시된다는 리뷰어 정렬 입장과 일치합니다.
- 결과적으로 인터페이스별 상태/오류 필드는 스키마에서 `type: string`으로 선언됩니다(`type: enum`/`allowedValues`가 아닌), VEL이 모든 현재 및 미래의 인터페이스별 값의 폐쇄된 열거형을 소유하지 않기 때문입니다 — 그리고 아키텍처상 소유해서도 안 됩니다. `conversion.values`는 가독성과 일관성을 위해 **현재 알려진** 별칭을 문서화하고 정규화합니다; 필드가 가질 수 있는 값을 제한하지 않으며, 인식되지 않은 값은 거부되지 않고 `"unknown"`으로 매핑됩니다.
- 예시: Lifecycle 시퀀스 다이어그램의 Lifecycle `kRunning → "running"` 단계가 이 메커니즘을 사용합니다. 변환 규칙은 `score-lifecycle-state-v1.0.0.yaml`에 한 번 선언되고 `ConfigDrivenStateMapper`가 일반적으로 실행합니다 — `LifecycleFieldBindings`(제네릭 또는 플러그인 제공)에는 자체 enum-to-string 로직이 없습니다.

## 4. 수집기 라이프사이클

### 4.1 라이프사이클 상태

모든 수집기는 Evidence Runtime 내에서 동일한 라이프사이클 상태 머신을 따릅니다:

![수집기 라이프사이클 상태 머신](../../../features/assets/Collector_lifecycle_state_machine.svg)

[PlantUML 원본](../../../features/diagrams/Collector_lifecycle_state_machine.puml)

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

> **입력 계약의 `AdapterDiagnostic`과의 관계:** 이는 [검증 실패 처리 계약](../configuration-contracts/input-interface-contract/validation-failure-handling.md)에서 참조하는 것과 동일한 `AdapterDiagnostic` 종류이며, 서로 다른 파이프라인 단계를 다룹니다. 이 문서의 형태는 **수집기 I/O 수준** 실패(`error_category` 값: `CONFIGURATION_ERROR`, `SOURCE_UNAVAILABLE`, `AUTHENTICATION_ERROR`, `TIMEOUT`, `SCHEMA_VIOLATION`, `NORMALIZATION_ERROR`, `RESOURCE_EXHAUSTION`, `INTERNAL_ERROR` — Section 5.1)를 다루며, 이는 raw 상태 레코드가 검증될 수 있기 **이전에** 발생할 수 있으므로(예: 소스 API에 접근 불가), `sourceMessageId`/`field` 같은 필드는 적용되지 않습니다. 입력 계약의 형태는 레코드가 읽힌 후 **스키마 검증 수준** 실패(`failureCategory` 값: `missingRequiredField`, `invalidType`, `outOfRange`, `notAllowedValue`, `patternMismatch`, `additionalFieldPresent`)를 다룹니다. 수집기 수준 진단에 `error_category: SCHEMA_VIOLATION`이 있으면, 입력 계약의 `AdapterDiagnostic`에서 더 세분화된 `failureCategory`(및 문제 `field`)가 동일한 진단 레코드에 포함되어 어떤 검증 규칙이 실패했는지 식별합니다. 두 문서는 하나의 `AdapterDiagnostic` 종류의 상호 보완적 측면을 설명하며, 서로 다른 두 스키마가 아닙니다.

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
 
모든 수집기는 `VehicleEvidenceCollector` 리소스 스키마를 준수하는 이 공통 구성 구조를 공유합니다 (`transport`/`api` 바인딩과 같은 소스별 필드는 각 소스 카테고리의 자체 설계 문서에서 정의됨):
 
```yaml
apiVersion: score.dev/v1alpha1
kind: VehicleEvidenceCollector
metadata:
  name: <source>-collector
  version: 1.0.0
spec:
  source:
    transport: <s-core-api | dds | file | command | os-interface>
    api: "<source-specific-api-or-topic>"
    messageType: <MessageType>
  fieldBindingProvider:
    kind: <generic | plugin>          # Section 3.6
    plugin:                            # kind: plugin일 때만 존재
      library: plugins/<source>-field-bindings
      abiVersion: 1
      entryPoint: vel_plugin_create_binding_provider
      settings: {}                     # 불투명, 플러그인 정의
  inputSchemaRef:
    name: <input-schema-name>
    version: 1.0.0
    path: schemas/<input-schema>.yaml
  normalizationRuleRef:
    name: <mapping-rule-name>
    version: 1.0.0
    path: mappings/<normalization-mapping>.yaml
  outputSchemaRef:
    name: score-normalized-evidence
    version: 1.0.0
    path: ../vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml
  publication:
    transport: dds
    topic: ScoreNormalizedEvidence
    deliveryClass: reliable
  velHealth:
    topic: ScoreEvidenceLayerHealth
    heartbeatPeriod: 1s
  resourceLimits:
    inputQueueDepth: 256
    maximumMessageRate: 1000
```

## 7. 소스 카테고리 설계 문서와의 관계

| 측면 | 문서 |
|--------|----------|
| 수집기 I/O API | **이 문서** (공통, 모든 소스 카테고리) |
| 수집기 라이프사이클 | **이 문서** (공통, 모든 소스 카테고리) |
| 오류 및 진단 보고 | **이 문서** (공통, 모든 소스 카테고리) |
| 설정 기반 상태 매핑 (`IStateFieldBindingProvider`, `ConfigDrivenStateMapper`) | **이 문서, Section 3.5** (공통, 모든 소스 카테고리) |
| 플러그인 검색/로딩 (제네릭 vs. 플러그인 바인딩, `PluginLoader`, ABI 계약) | **이 문서, Section 3.6** (공통, 모든 소스 카테고리) |
| 필드 값 변환 (`conversion` 블록, 매핑되지 않은 값 보존) | **이 문서, Section 3.7** (공통, 모든 소스 카테고리) |
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
