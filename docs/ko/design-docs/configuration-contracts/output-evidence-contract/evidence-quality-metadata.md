# Evidence 품질 메타데이터 필드

> **관련:** FR-VEL-009 (Evidence 품질 메타데이터), STKH-VEL-009
> **참조 예시:** [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml)

## 1. 목적

이 문서는 각 정규화된 Vehicle Evidence 레코드 (또는 배치)에 첨부되는 **Evidence 품질(Evidence Quality)** 메타데이터 필드를 정의합니다. Evidence 품질은 관찰의 유효성, 신선도, 완전성 및 매핑 품질을 설명합니다. 차량 동작을 해석하지 않습니다 (VEL 범위 기준).

## 2. 설계 위치

VEL 아키텍처에서 품질 메타데이터는 출력 Evidence 정의의 일부입니다. 정규화 매핑 규칙과 검증 결과에 따라 품질 평가기가 할당합니다. 이 문서는 데이터 표현 및 구성 형식만 지정합니다.

## 3. 품질 메타데이터 모델

### 3.1 출력 Evidence 정의의 일부로서의 품질 메타데이터

품질 메타데이터 필드는 출력 Evidence 정의의 `quality` 섹션에 선언됩니다. 모든 레코드에서 필수입니다.

### 3.2 품질 필드

출력 Evidence 정의는 네 가지 품질 필드를 선언합니다:

| 필드 | 유형 | 허용 값 | 설명 |
| --- | --- | --- | --- |
| `validity` | enum | `valid`, `invalid`, `unknown` | evidence 관찰이 유효한지 여부. |
| `freshness` | enum | `current`, `delayed`, `stale` | 관찰이 기대치에 비해 얼마나 최신인지. |
| `completeness` | enum | `complete`, `partial`, `missing` | 예상된 모든 관찰 데이터가 있는지 여부. |
| `mappingQuality` | enum | `exact`, `approximate`, `unmapped` | 소스 값이 정규화된 표현으로 얼마나 충실하게 매핑되었는지. |

### 3.3 품질 의미론

- **validity** — 관찰이 유효한 것으로 간주되는지 여부를 나타냅니다. 잘못된 값 레코드가 발행될 때 `invalid`가 사용됩니다. 유효성을 결정할 수 없을 때 `unknown`이 사용됩니다.
- **freshness** — 관찰의 적시성을 나타냅니다. `current`는 관찰이 예상 신선도 창 내에 있음을 의미합니다. `delayed`는 늦었지만 사용 가능함을 의미합니다. `stale`은 최신으로 간주하기에는 너무 오래되었음을 의미합니다. 신선도 창 (`current`, `delayed`, `stale` 사이의 임계값)은 **배포별** 이며 배포 구성에 선언됩니다.
- **completeness** — 예상된 모든 필드가 있는지 여부를 나타냅니다. `complete`는 예상된 모든 데이터가 있음을 의미합니다. `partial`은 일부 예상 데이터가 없음을 의미합니다. `missing`은 예상된 관찰 데이터가 완전히 없음을 의미합니다.
- **mappingQuality** — 정규화 매핑의 충실도를 나타냅니다. `exact`는 직접/정확한 매핑을 의미합니다. `approximate`는 스케일링되거나 근사적인 변환을 의미합니다. `unmapped`는 소스 값을 정규화된 표현으로 매핑할 수 없음을 의미합니다.

### 3.4 품질 할당

품질 값은 품질 평가기가 할당합니다. 출력 Evidence 계약은 **허용 값과 구조**를 정의합니다. 할당 규칙은 정규화 매핑 계약과 품질 평가 설계에 정의됩니다.

### 3.5 기본값

참조 예시는 정규화 매핑의 기본 품질 값을 보여줍니다:

```yaml
defaults:
  quality: {validity: valid, freshness: current, completeness: complete, mappingQuality: exact}
```

이 기본값은 품질에 영향을 주는 조건이 감지되지 않을 때 적용됩니다.

## 4. 구성 형식

### 4.1 출력 Evidence 정의의 품질 필드

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
    # ... header, identity, observation, time 필드는 간결성을 위해 생략
    - {path: $.quality.validity, type: enum, values: [valid, invalid, unknown], required: true}
    - {path: $.quality.freshness, type: enum, values: [current, delayed, stale], required: true}
    - {path: $.quality.completeness, type: enum, values: [complete, partial, missing], required: true}
    - {path: $.quality.mappingQuality, type: enum, values: [exact, approximate, unmapped], required: true}
    # ... traceability 필드는 간결성을 위해 생략
```

### 4.2 레코드의 품질 섹션 예시

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

- 품질 메타데이터는 각 evidence 레코드에 첨부되며 필수입니다. 배치 수준 품질 집계는 선택 사항이며 배포에서 제공할 수 있습니다.
- 품질은 VEL 파이프라인이 아닌 **관찰**을 설명합니다. VEL 파이프라인 상태는 VEL Health를 통해 별도로 노출됩니다.
- 품질 값은 정규화 매핑 규칙과 검증 결과에 따라 품질 평가기가 할당합니다. 네 가지 품질 차원 (`validity`, `freshness`, `completeness`, `mappingQuality`)은 완전한 집합입니다. 추가 차원은 추가되지 않습니다.
- 소비자는 품질 메타데이터를 사용하여 evidence 관찰에 얼마나 신뢰를 둘지 결정합니다.

## 6. 추적성

| 항목 | 참조 |
| --- | --- |
| 요구사항 | FR-VEL-009, STKH-VEL-009 |
| 참조 예시 | [`score-normalized-evidence-v1.0.0.yaml`](../../../interfaces/vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml) |