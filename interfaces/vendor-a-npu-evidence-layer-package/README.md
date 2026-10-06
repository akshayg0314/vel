# Vendor-A NPU Evidence Layer Example Package

> 설계 예제입니다. 공개 S-CORE의 실제 schema가 아니며, DDS topic과 필드도 구현 계약을 설명하기 위한 예시입니다.

## 목적

Vendor-A NPU runtime 메시지를 검증하고 S-CORE Evidence Layer의 정규화 Evidence로 변환하는 최소 패키지 구조를 보여줍니다. VEL은 관측값을 변환하며 Scenario 실행, workload restart, MRM 같은 차량 행동은 결정하지 않습니다.

## 구조

```text
vendor-a-npu-evidence-layer-package/
├── manifest.yaml
├── README.md
├── collectors/
│   └── npu-runtime-collector.yaml
├── schemas/
│   └── vendor-a-npu-status-v1.2.0.yaml
├── mappings/
│   └── vendor-a-npu-to-evidence-v3.1.0.yaml
├── signatures/
│   ├── content-digests.yaml
│   └── manifest.sig
└── tests/
    └── normalization-vectors.yaml
```

> 정규화 출력 schema(`score-normalized-evidence-v1.0.0.yaml`)는 모든 source package가 공유하므로 더 이상 이 package 안에 있지 않습니다. [`../vel-output-evidence-package/`](../vel-output-evidence-package/)를 참고하세요.

## 파일 설명

- `manifest.yaml`: 검증 및 배포의 최상위 패키지 manifest입니다. 구성 파일과 artifact 역할, 호환 버전을 고정합니다.
- `collectors/npu-runtime-collector.yaml`: DDS 입력 topic, 사용할 schema/rule, 출력 topic, VEL health 및 resource limit을 연결합니다.
- `schemas/vendor-a-npu-status-v1.2.0.yaml`: Vendor 메시지의 필드, 타입, 필수성, 범위를 정의합니다. 의미 판단은 포함하지 않습니다.
- `mappings/vendor-a-npu-to-evidence-v3.1.0.yaml`: 필드 전달, us에서 ms로의 단위 변환, enum 변환, vendor 오류 코드 매핑 및 실패 처리를 정의합니다.
- `../vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml`: 정규화 출력 record가 만족할 논리 계약을 정의합니다 (모든 source package가 공유하는 별도 package).
- `tests/normalization-vectors.yaml`: 정상, 미등록 오류 코드, 필수 필드 누락의 입출력 기대 결과를 정의합니다.
- `signatures/content-digests.yaml`: 주요 artifact의 실제 SHA-256 digest를 담습니다.
- `signatures/manifest.sig`: 예제는 서명되지 않았음을 명시하는 자리표시자입니다. 실제 보안 서명이 아닙니다.

## 정규화 예

입력 `inference_latency_us: 38400`은 mapping rule에 따라 `38.4 ms`가 됩니다. `device_state: 2`는 `degraded` 관측 상태로 변환됩니다. `vendor_error_code: 0xA17`은 `accelerator.execution.timeout` Evidence를 추가 생성합니다. 하나의 입력 메시지에서 세 개의 Evidence record가 만들어지며 동일 `execution_id`를 correlation ID로 사용합니다.

## 책임 경계

- Execution Layer: Vendor runtime 메시지와 canonical runtime state 소유
- Evidence Layer, S-CORE: schema validation, normalization, quality, traceability, publication
- Orchestrator Layer, S-CORE: Evidence 해석과 scenario/workload governance
- OEM Policy Tier: 최종 차량 행동과 MRM 결정

## 적용 전 필수 확정 사항

실제 DDS IDL, topic QoS, timestamp clock domain, evidence ID 생성법, schema validator, rule engine 지원 연산, 오류 진단 schema, 서명 및 OTA 승인 절차를 별도 Engineering View 계약으로 확정해야 합니다.
