# 구성 계약 (Configuration Contracts)

이 폴더에는 S-CORE 모듈이 Vehicle Evidence Layer(VEL)와 통합될 때의 입력 인터페이스 계약 문서가 포함되어 있습니다. 구성 계약은 세 가지 하위 계약으로 구성됩니다: 입력 인터페이스 계약(Input Interface Contract), 정규화 계약(Normalization Contract), 출력 evidence 계약(Output Evidence Contract).

## 목차

```{toctree}
:maxdepth: 2

input-interface-contract/s-core-module-input-design
input-interface-contract/s-core-modules/lifecycle-input-design
input-interface-contract/input-schema-structure
input-interface-contract/source-identity-and-timestamp
input-interface-contract/validation-failure-handling
normalization-contract/mapping-rule-model
normalization-contract/unmapped-and-unknown-value-handling
normalization-contract/mapping-quality-outputs
output-evidence-contract/normalized-evidence-record-structure
output-evidence-contract/evidence-quality-metadata
output-evidence-contract/traceability-and-correlation-metadata
```

## 관련 패키지

S-CORE Lifecycle 모듈의 표준 인터페이스 아티팩트는 패키지 디렉터리에 있습니다: `interfaces/score-evidence-layer-package/` (스키마, 매핑, 수집기 구성, 테스트, 서명). 벤더-NPU 참조 패키지는 `interfaces/vendor-a-npu-evidence-layer-package/`에, 출력 evidence 패키지는 `interfaces/vel-output-evidence-package/`에 있습니다.
