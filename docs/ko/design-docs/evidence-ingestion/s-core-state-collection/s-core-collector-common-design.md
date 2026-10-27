# S-CORE 수집기 공통 설계

> **참고:** [../source-collector-interface-design.md](../source-collector-interface-design.md) — 이 문서가 적용하는 **공통** Source Collector 인터페이스: 수집기 I/O API, 라이프사이클, 오류/진단 보고는 모든 소스 카테고리(런타임, 하드웨어, S-CORE 모듈 모두)에 대해 그곳에서 한 번 정의됩니다.

## 개요

S-CORE 모듈 수집기(현재는 Lifecycle)는 공통 [Source Collector 인터페이스](../source-collector-interface-design.md)를 따릅니다 — 자체 I/O API, 라이프사이클 상태 머신, 진단 분류 체계를 정의하지 않습니다. 이 문서는 해당 공통 인터페이스 위에 추가되는 **S-CORE 특화 관례**만 다룹니다:

- S-CORE 모듈 수집기에 대해 `source_ref`와 `collector_id`가 어떻게 명명되는지.
- `s-core-api` 전송에 대해 공통 수집기 구성의 `source` 블록이 어떻게 채워지는지.
- 각 S-CORE 모듈의 모듈별 설계(API 표면, 상태 필드, 전송 경로)가 어디에 문서화되어 있는지.

## 1. S-CORE 명명 관례

### 1.1 `source_ref`

S-CORE 모듈 수집기는 공통 I/O API의 `source_ref` 매개변수(참조: [source-collector-interface-design.md Section 3.2](../source-collector-interface-design.md#32-입력-계약))를 `score.<module>` 관례로 채웁니다:

| 모듈 | `source_ref` |
| --- | --- |
| Lifecycle | `score.lifecycle` |

### 1.2 `collector_id`

S-CORE 모듈 수집기는 `<module>-collector` 명명 관례를 사용합니다 (예: `lifecycle-collector`), 이는 각 모듈의 `VehicleEvidenceCollector` 구성 YAML의 `metadata.name` 필드와 일치합니다 (참조: `interfaces/score-evidence-layer-package/collectors/`).

## 2. S-CORE 수집기 구성

S-CORE 모듈 수집기는 공통 수집기 구성(참조: [source-collector-interface-design.md Section 6](../source-collector-interface-design.md#6-공통-수집기-구성))을 `VehicleEvidenceCollector` 리소스 스키마 및 `transport: s-core-api`로 채웁니다:

```yaml
apiVersion: score.dev/v1alpha1
kind: VehicleEvidenceCollector
metadata:
  name: <module>-collector
  version: 1.0.0
spec:
  source:
    transport: s-core-api
    api: <module>-state-api
    messageType: Score<Module>State
  fieldBindingProvider:
    # generic: 내장, 설정 기반 바인딩 (기본값 -- 플러그인 없음, 새 코드 없음).
    # plugin: 독립적으로 빌드/배포된 바인딩 아티팩트, VEL 코어의 PluginLoader가
    #         안정적인 C-ABI 진입점을 통해 이 구성에서만 구동되어 로드함
    #         (참조: source-collector-interface-design.md Section 3.6).
    #         모듈의 네이티브 API를 선언적으로 도달할 수 없을 때에만 필요함
    #         (Lifecycle이 현재 예시 -- 해당 수집기 구성 참조).
    kind: <generic | plugin>
    plugin:                                   # kind: plugin일 때만 존재
      library: plugins/<module>-field-bindings
      abiVersion: 1
      entryPoint: vel_plugin_create_binding_provider
      settings: {}
  inputSchemaRef:
    name: score-<module>-state
    version: 1.0.0
    path: schemas/score-<module>-state-v1.0.0.yaml
  normalizationRuleRef:
    name: score-<module>-to-evidence
    version: 1.0.0
    path: mappings/score-<module>-to-evidence-v1.0.0.yaml
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

> **S-CORE 특화 오류 카테고리:** `SOURCE_UNAVAILABLE`([source-collector-interface-design.md Section 5.1](../source-collector-interface-design.md#51-오류-카테고리)에서 일반적으로 정의됨)은 S-CORE 모듈 수집기의 경우, S-CORE 모듈의 API(예: Launch Manager의 external monitor notification 채널)에 접근할 수 없거나 응답하지 않음을 의미합니다. 마찬가지로 `AUTHENTICATION_ERROR`와 `TIMEOUT`도 S-CORE API 호출에 구체적으로 적용됩니다.

## 3. 공유 상태 매핑 엔진

S-CORE 모듈 수집기는 각자 고유한 raw 레코드 매핑, 검증, 진단 로직을 구현하지 않습니다. 매핑/검증 엔진은 [source-collector-interface-design.md Section 3.5](../source-collector-interface-design.md#35-설정-기반config-driven-상태-매핑-공통-수집기-내부-구조)에서 한 번 정의된 **진정한 공통 인프라**입니다 (`IStateFieldBindingProvider`, `ConfigDrivenStateMapper`) — Lifecycle 수집기와 함께 도입되었지만 S-CORE 특화가 아닙니다. S-CORE 모듈 수집기는 다른 소스 카테고리와 동일한 방식으로 이를 재사용합니다:

- 모든 S-CORE 모듈 수집기는 `VehicleEvidenceCollector` YAML(`source_ref`, `collector_id`, schema/mapping 참조)에서 구성된 **동일한 제네릭 `SCoreCollector`**입니다 — VEL 코어에 컴파일된 모듈별 구체 수집기 하위 클래스(예: `LifecycleCollector` 클래스)는 없습니다.
- 각 S-CORE 모듈의 `IStateFieldBindingProvider` 실현은 해당 모듈의 `fieldBindingProvider` 구성(제네릭 또는 플러그인 — [source-collector-interface-design.md Section 3.6](../source-collector-interface-design.md#36-플러그인-검색-및-로딩))에 의해 선택됩니다: 기존 선언적 전송/공급자 기능으로 상태에 도달할 수 있는 모듈에는 **제네릭 바인딩**(VEL 코어에 내장, 설정 기반, 새 코드 없음); Lifecycle과 같이 제네릭 바인딩이 표현할 수 없는 네이티브 S-CORE 인터페이스가 실제로 필요한 모듈에만 **플러그인**(독립적으로 빌드/배포된 바인딩 아티팩트, 공유 `PluginLoader`가 안정적인 C-ABI 진입점을 통해 구성에서만 로드). 어느 쪽이든 이것이 모듈별 로직이 존재할 수 있는 유일한 곳이며, VEL 코어 컴파일 클래스가 절대 아닙니다.
- Enum-to-string 및 유사한 값 변환(예: Lifecycle의 `kRunning` -> `"running"`)은 각 필드의 `RawEvidenceSchema`에 있는 `conversion` 블록에 선언되고 `ConfigDrivenStateMapper`가 일반적으로 실행합니다([source-collector-interface-design.md Section 3.7](../source-collector-interface-design.md#37-필드-값-변환-공통-수집기-내부-구조)) — 바인딩 어댑터나 플러그인에 하드코딩되지 않습니다.
- 공유 `ConfigDrivenStateMapper`는 모듈의 `RawEvidenceSchema` 구성과 해당 바인딩으로부터 schema에 부합하는 raw 레코드를 구성합니다 — 모듈별 매퍼 코드는 없습니다.
- **Schema Validator / Normalization Processor** — 공유 evidence 처리 컴포넌트(참조: [vel_architectural_design_draft_kor.md Section 3](../../../vel_architectural_design_draft_kor.md#3-구성-경계)), `RawEvidenceSchema`와 `EvidenceNormalizationRule` 구성에 의해 구동됩니다. 소스 모듈과 무관하게 동일하게 적용됩니다.
- **`CollectorDiagnostics`** — 공유 오류/진단 보고, 참조: [source-collector-interface-design.md Section 5](../source-collector-interface-design.md#5-수집기-오류-및-진단-보고).

따라서 새로운 S-CORE 모듈 수집기를 추가하려면: 새로운 `RawEvidenceSchema`, 새로운 `EvidenceNormalizationRule`, 새로운 `VehicleEvidenceCollector` 구성만 필요합니다 — 그리고, **제네릭 바인딩이 해당 모듈의 네이티브 인터페이스에 도달할 수 없는 경우에만**, `IStateFieldBindingProvider`를 구현하는 독립적으로 빌드/배포된 플러그인 하나가 필요합니다. 새로운 Mapper, Validator, Normalizer, 수집기 하위 클래스는 절대 작성되지 않으며, 하드코딩된 팩토리/등록 목록이 갱신되지 않으며, VEL 코어가 다시 빌드되지 않습니다. 이는 FR-VEL-012 (플랫폼 수집기 분리)와 FR-VEL-013/014/015 (설정 기반 입력, 출력, 정규화)를 schema 수준뿐만 아니라 수집기 구현 수준에서도 충족합니다.

공유 엔진과 플러그인 로딩 모델에 대해서는 [공통 수집기 클래스 다이어그램](../../../../features/diagrams/Collector_common_class_diagram.puml)을, S-CORE Lifecycle에서의 구체적 적용(플러그인 기반 바인딩 — Lifecycle의 네이티브 인터페이스가 필요하므로)에 대해서는 [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) Section 7과 [Lifecycle 수집기 클래스 다이어그램](../../../../features/diagrams/lifecycle/Lifecycle_collector_class_diagram.puml)을 참조하십시오.

## 4. 모듈별 설계 문서와의 관계

| 측면 | 문서 |
|--------|----------|
| 수집기 I/O API | [source-collector-interface-design.md](../source-collector-interface-design.md) (공통, 모든 소스 카테고리) |
| 수집기 라이프사이클 | [source-collector-interface-design.md](../source-collector-interface-design.md) (공통, 모든 소스 카테고리) |
| 오류 및 진단 보고 | [source-collector-interface-design.md](../source-collector-interface-design.md) (공통, 모든 소스 카테고리) |
| S-CORE 명명 관례 및 수집기 구성 | **이 문서** |
| Lifecycle 모듈 API 표면 / 상태 필드 / 전송 경로 | [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) |

## 5. 추적성

| 항목 | 참조 |
| --- | --- |
| 설계 | S-CORE 수집기 공통 설계 |
| 적용된 공통 인터페이스 | [source-collector-interface-design.md](../source-collector-interface-design.md) |
| 요구사항 | FR-VEL-003, FR-VEL-012 |
