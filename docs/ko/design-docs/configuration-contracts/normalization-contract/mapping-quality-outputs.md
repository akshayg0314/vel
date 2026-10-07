# 매핑 품질 출력

> **관련:** FR-VEL-009 (Evidence 품질 메타데이터), STKH-VEL-009
> **참조 예시:** [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml)

## 1. 목적

이 문서는 정규화 매핑이 **매핑 품질(mapping quality)** 출력을 생성하는 방법을 정의합니다. 매핑 품질은 소스 값이 정규화된 표현으로 얼마나 충실하게 매핑되었는지를 설명합니다.

## 2. 설계 위치

VEL 아키텍처에서 매핑 품질은 정규화된 Vehicle Evidence 레코드의 네 가지 Evidence 품질 필드 중 하나입니다. 적용된 매핑 규칙에 따라 정규화 프로세서가 할당합니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 매핑 품질 모델

### 3.1 품질 메타데이터 필드로서의 매핑 품질

매핑 품질은 네 가지 Evidence 품질 필드 중 하나입니다. 세 가지 값을 가진 enum으로 표현됩니다:

| 값 | 설명 |
| --- | --- |
| `exact` | 소스 값이 충실도 손실 없이 직접/정확하게 매핑됨. |
| `approximate` | 소스 값이 약간의 부정확성을 도입할 수 있는 변환 (예: 단위 스케일링, 반올림)으로 매핑됨. |
| `unmapped` | 소스 값을 정규화된 표현으로 매핑할 수 없음 (예: 알 수 없는 enum, 매핑되지 않은 오류 코드). |

이 세 값 enum은 매핑 품질의 완전한 집합입니다. 중간 값은 사용되지 않습니다. 값은 정확히 매핑되거나, 근사적으로 매핑되거나, 매핑되지 않습니다. 부분 매핑은 지원되지 않습니다.

### 3.2 매핑 품질 할당 방법

매핑 품질은 적용된 매핑 규칙에 따라 정규화 프로세서가 할당합니다:

| 매핑 상황 | mappingQuality |
| --- | --- |
| 직접 필드 복사 (공통 매핑) | `exact` |
| `scale`을 통한 단위 변환 | `approximate` (또는 무손실인 경우 `exact`) |
| 일치하는 값이 있는 enum/상태 매핑 | `exact` |
| `unmappedValue` (unknown)가 있는 enum/상태 매핑 | `unmapped` |
| 일치하는 키가 있는 조회 | `exact` |
| `onUnmapped` 폴백이 있는 조회 | `unmapped` |
| 발행된 잘못된 값 (`emit-invalid-evidence`) | `unmapped` (및 `validity: invalid`) |

### 3.3 기본값

정규화 매핑은 품질에 영향을 주는 조건이 감지되지 않을 때 적용되는 기본 품질 값을 선언합니다:

```yaml
defaults:
  quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
```

### 3.4 매핑 품질 출력 전파

매핑 품질 값은 정규화된 Vehicle Evidence 레코드의 `$.quality.mappingQuality`에 기록됩니다. 변환 중 정규화 프로세서가 설정하며 추가 품질 조건이 감지되면 품질 평가기가 재정의할 수 있습니다. 품질 평가기는 매핑 규칙 자체로 포착되지 않은 품질 문제 (예: 정확히 매핑되지만 의미적으로 신뢰할 수 없는 값)가 평가에서 드러나면 값을 `unmapped` 또는 `approximate`로 변경할 수 있습니다.

### 3.5 다른 품질 필드와의 구분

- **mappingQuality**는 **변환**의 충실도를 설명합니다 (소스 값이 정규화된 값이 된 방식).
- **validity**는 관찰이 유효한지 여부를 설명합니다.
- **freshness**는 적시성을 설명합니다.
- **completeness**는 예상된 모든 데이터가 있는지 여부를 설명합니다.

이들은 독립적인 차원입니다. 예를 들어 값은 `exact` 매핑이지만 `stale` freshness일 수 있습니다.

### 3.6 exact vs. approximate 임계값

변환이 충실도 손실 없이 값을 보존하면 매핑은 **`exact`** 입니다 (직접 복사, 무손실 단위 스케일링, 일치하는 enum/조회 항목). 변환이 단위 변환 중 반올림 또는 정확히 표현할 수 없는 스케일링 값과 같은 부정확성을 도입하면 매핑은 **`approximate`** 입니다. 직접 매핑의 기본값은 `exact`입니다. `approximate`는 변환이 손실이 있는 것으로 알려진 경우에만 사용됩니다.

## 4. 구성 형식

### 4.1 정규화 매핑의 기본 품질

```yaml
defaults:
  evidenceDomain: accelerator
  quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
  time: {clockDomain: monotonic}
  traceability: {inputSchemaVersion: "1.2.0", normalizationRuleVersion: "3.1.0"}
```

### 4.2 실패 처리의 매핑 품질

```yaml
failureHandling:
  unmappedEnum: {result: emit-unknown, mappingQuality: unmapped}
```

### 4.3 onUnmapped 폴백의 매핑 품질

```yaml
onUnmapped:
  evidenceType: accelerator.vendor-event
  normalizedClass: unmapped
  mappingQuality: unmapped
  preserveOriginalValue: true
```

### 4.4 출력 레코드의 매핑 품질

```json
{
  "quality": {
    "validity": "valid",
    "freshness": "current",
    "completeness": "complete",
    "mappingQuality": "exact"
  }
}
```

## 5. 동작 의미론

- 모든 정규화된 Vehicle Evidence 레코드는 `mappingQuality` 값을 가집니다. 출력 레코드의 필수 필드입니다.
- 매핑 품질은 적용된 매핑 규칙과 변환 성공 여부에 따라 설정됩니다.
- `unmapped` 매핑 품질은 정규화된 값이 소스 값을 충실하게 나타내지 않을 수 있음을 소비자에게 알립니다.
- 매핑 품질은 레코드별로 할당됩니다. 매핑 품질의 배치 수준 집계는 선택 사항이며 배포에서 제공할 수 있습니다. 이는 레코드별 값을 대체하지 않습니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-009, STKH-VEL-009 |
| 참조 예시 | [`vendor-a-npu-to-evidence-v3.1.0.yaml`](../../../interfaces/vendor-a-npu-evidence-layer-package/mappings/vendor-a-npu-to-evidence-v3.1.0.yaml) |