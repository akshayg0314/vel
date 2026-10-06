# 설계 문서

이 폴더에는 Vehicle Evidence Layer(VEL)에 대한 설계 문서가 포함되어 있습니다. 일반적인 VEL 설계 문서(입력 인터페이스 계약, 수집기 설계 패턴)와 모듈별 통합 설계 문서를 모두 포함합니다.

폴더 구조는 VEL 초안 문서 구조를 따릅니다. 현재는 S-CORE Lifecycle 모듈 문서만 포함되어 있습니다.

## 목차

```{toctree}
:maxdepth: 2

configuration-contracts/README
evidence-ingestion/README
data-field-analysis/README
```

## 공통 참조

- [VEL 용어집](../glossary_kor.rst): Vehicle Evidence, Source, Source Collector, Input Interface Definition, Normalization Mapping, Evidence Quality, S-CORE Module, Evidence Runtime, Designated Consumer, Correlation Identifier, VEL Health, Output Evidence Definition.
- [VEL 아키텍처 설계 초안](../vel_architectural_design_draft_kor.md): VEL Evidence Layer 원칙 및 경계.
- [요구사항](../requirements_kor.rst): FR-VEL-003, FR-VEL-008, FR-VEL-012, FR-VEL-013, FR-VEL-014, FR-VEL-015, AOU-VEL-002, saf_req_vel_*.

## 범위 및 경계

- VEL은 Evidence 생산자입니다. 다중 노드 조정, 라이프사이클 실행, 프로세스/컨테이너/하드웨어 제어 또는 최종 OEM 결정을 수행하지 않습니다.
- 플랫폼별 수집기는 공통 Evidence Layer와 분리되어 있습니다.