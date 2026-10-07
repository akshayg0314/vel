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

S-CORE 모듈 수집기는 공통 수집기 구성(참조: [source-collector-interface-design.md Section 6](../source-collector-interface-design.md#6-공통-수집기-구성))을 `transport: s-core-api`로 채웁니다:

```yaml
collector:
  kind: VehicleEvidenceCollector
  version: 1.0.0
  collector_id: "<module>-collector"
  source:
    transport: s-core-api
    api: "<module-specific-api>"
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

> **S-CORE 특화 오류 카테고리:** `SOURCE_UNAVAILABLE`([source-collector-interface-design.md Section 5.1](../source-collector-interface-design.md#51-오류-카테고리)에서 일반적으로 정의됨)은 S-CORE 모듈 수집기의 경우, S-CORE 모듈의 API(예: Launch Manager의 external monitor notification 채널)에 접근할 수 없거나 응답하지 않음을 의미합니다. 마찬가지로 `AUTHENTICATION_ERROR`와 `TIMEOUT`도 S-CORE API 호출에 구체적으로 적용됩니다.

## 3. 모듈별 설계 문서와의 관계

| 측면 | 문서 |
|--------|----------|
| 수집기 I/O API | [source-collector-interface-design.md](../source-collector-interface-design.md) (공통, 모든 소스 카테고리) |
| 수집기 라이프사이클 | [source-collector-interface-design.md](../source-collector-interface-design.md) (공통, 모든 소스 카테고리) |
| 오류 및 진단 보고 | [source-collector-interface-design.md](../source-collector-interface-design.md) (공통, 모든 소스 카테고리) |
| S-CORE 명명 관례 및 수집기 구성 | **이 문서** |
| Lifecycle 모듈 API 표면 / 상태 필드 / 전송 경로 | [`collectors/lifecycle-collector-design.md`](collectors/lifecycle-collector-design.md) |

## 4. 추적성

| 항목 | 참조 |
| --- | --- |
| 설계 | S-CORE 수집기 공통 설계 |
| 적용된 공통 인터페이스 | [source-collector-interface-design.md](../source-collector-interface-design.md) |
| 요구사항 | FR-VEL-003, FR-VEL-012 |
