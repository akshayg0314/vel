# Vehicle Evidence Layer 개략설계 초안

## 1. 범위와 설계 방향

Vehicle Evidence Layer(VEL)은 구성된 runtime, hardware 및 S-CORE 모듈 상태 정보를 수집하고, 이기종 source 데이터를 정규화하여 지정된 consumer에게 Vehicle Evidence를 노출한다.

VEL은 evidence producer다. VEL은 multi-node coordination, boot sequence, workload lifecycle 실행, process/container 제어, hardware 제어, policy 결정 및 OEM 최종 차량 의사결정을 수행하지 않는다.

배포 방식과 기능 경계를 구분한다. 하나의 process가 여러 VEL 컴포넌트를 호스팅할 수 있지만, 설계에서는 각 기능 경계를 명시적으로 유지한다.

## 2. 아키텍처 개요

개요도는 Vehicle Evidence Layer의 경계를 보여준다. 구성된 source와 interface configuration이 VEL에 입력되고, VEL은 지정된 consumer에게 Vehicle Evidence를 노출한다. 선택적 persistence와 운영 상태 출력은 OEM 의사결정 및 제어와 분리된다.

![Vehicle Evidence Layer 시스템 개요](../features/assets/VEL_architecture_overview.svg)

[개요 PlantUML 원본](../features/diagrams/VEL_architecture_overview.puml)

컴포넌트 그림은 수집, 처리, 발행, 선택적 저장, health 및 logging을 Vehicle Evidence Layer 경계 안에 배치하고, 구성된 source, 제공되는 interface definition 및 지정된 consumer는 경계 밖에 둔다. 화살표에는 컴포넌트 간 전달되는 data 또는 configuration을 표시하며, 아래의 내부 처리 순서도는 처리 순서와 실패 분기를 별도로 설명한다. 각 컴포넌트를 별도 process 또는 배포 단위로 고정하지 않는다. 선택적 persistence는 evidence record를 받지만, Publisher가 저장된 evidence를 다시 조회할지는 미결 연계 사항이며 필수 경로가 아니다.

![Vehicle Evidence Layer 기능 컴포넌트](../features/assets/VEL_architecture_components.svg)

[컴포넌트 PlantUML 원본](../features/diagrams/VEL_architecture_components.puml)

Vehicle Evidence Layer는 다음 다섯 기능 영역으로 구성된다.

- **Evidence Ingestion**: Source Collector가 file, command, 지원 OS interface 또는 S-CORE API를 통해 구성된 runtime, hardware 및 S-CORE state를 읽는다.
- **Evidence Processing**: Schema Validator가 source record를 검증하고, Normalization Processor가 mapping을 적용하며, Quality Evaluator가 evidence quality를 부여하고, Traceability Enricher가 source identity, observation time 및 제공된 correlation 정보를 추가한다.
- **Evidence Management**: 배포 환경에서 persistence를 구성한 경우에만 정규화된 evidence를 저장한다.
- **Evidence Publication**: Vehicle Evidence Publisher가 normalized Vehicle Evidence contract를 노출하고, VEL Health Publisher가 VEL 수집 pipeline의 health를 노출한다.
- **Observability**: Collection and Audit Logging이 수집, 검증, 정규화 및 발행 event를 기록하며 Vehicle Evidence contract와는 분리된다.

## 3. Configuration 경계

Configuration은 source와 evidence contract를 정의한다.

- Input Interface Definition은 source field, type 및 requiredness를 정의한다.
- Normalization Mapping은 source-to-evidence field mapping, unit conversion, state conversion 및 quality rule을 정의한다.
- Output Evidence Definition은 VEL이 consumer에게 노출하는 normalized Vehicle Evidence field를 정의한다.

이를 통해 새로운 source를 추가할 때 common evidence processing behavior를 변경하지 않고 configuration과 platform-specific collector를 확장할 수 있다.

Output Evidence Definition은 담당 data-format 이해관계자와 합의한 aggregation data format을 담는다(FR-VEL-016). Vehicle Evidence Publisher는 구성된 output contract에 따라 evidence를 노출한다. 이는 별도의 aggregation 또는 multi-node coordination 컴포넌트를 뜻하지 않는다. format은 합의 전까지 미결 설계 항목이다.

## 4. 컴포넌트 책임

| 컴포넌트 | 책임 | 경계 |
| --- | --- | --- |
| Source Collector | 구성된 source data와 source context를 읽는다 | platform-specific 구현이며 normalization policy를 결정하지 않는다 |
| Schema Validator | input definition에 따라 source record를 검증한다 | 잘못된 source data를 reject하거나 진단한다 |
| Normalization Processor | source representation을 Vehicle Evidence representation으로 변환한다 | configuration을 적용하며 OEM 결정을 내리지 않는다 |
| Quality Evaluator | validity, freshness, completeness 및 mapping quality를 부여한다 | evidence quality를 설명하며 차량 동작을 해석하지 않는다 |
| Traceability Enricher | source identity, observation time 및 제공된 correlation 정보를 추가한다 | observation provenance를 보존한다 |
| Evidence Persistence | 구성된 경우 normalized evidence를 저장한다 | 선택적 deployment capability다 |
| Vehicle Evidence Publisher | 구성된 output interface를 통해 normalized Vehicle Evidence를 노출한다 | evidence publication만 담당한다 |
| VEL Health Publisher | collection 및 processing health를 노출한다 | VEL operational status만 담당한다 |
| Collection and Audit Logging | operational 및 audit event를 기록한다 | evidence content와 분리된다 |

Vehicle Evidence Publisher에서 지정된 consumer로 이어지는 경계에서 VEL은 배포 환경이 선택한 security mechanism과 consumer access rule을 적용한다(SEC-VEL-001, SEC-VEL-004). authentication이 필요한 경우 배포 환경이 제공한 caller identity를 사용하고(SEC-VEL-003), 선택된 integrity method를 지원한다(SEC-VEL-005). VEL은 자체 authentication authority를 정의하거나 운영하지 않는다(SEC-VEL-002). 배포 환경별 mechanism과 transport는 여기서 확정하지 않는다.

## 5. S-CORE 연계

![S-CORE integration view with Vehicle Evidence Layer](../features/assets/SCORE_architecture_with_VEL.svg)

[PlantUML 원본](../features/diagrams/SCORE_architecture_with_VEL.puml)

이 그림은 컴포넌트 아키텍처의 S-CORE 연계 경계를 상세히 보여주며, 수집→처리→발행 경로는 동일하게 유지한다. Source Collector는 module state를 관측하고, Evidence Processing은 Vehicle Evidence Publisher로 전달할 정규화된 evidence를 생성한다. Collection and Audit Logging은 운영 이벤트를 S-CORE Logging으로 보낸다. S-CORE Communication은 module state의 출처가 아니라 module API에 접근할 때 조건부로 사용하는 수단이다. 필요한지 여부는 배포 환경의 module API 계약이 결정하며, 이 설계는 특정 communication profile이나 state field를 확정하지 않는다. 선택적 evidence 저장에는 S-CORE Persistency를 사용할 수 있다. VEL Health는 여기서 S-CORE service에 직접 의존하지 않으므로 컴포넌트 아키텍처 그림에서 설명한다.

VEL은 다음 경계에서만 S-CORE service를 사용한다.

- S-CORE Modules는 배포 환경이 노출한 API를 통해 관측 가능한 module state를 제공한다.
- 배포된 module API가 S-CORE Communication을 사용하는 경우에만 Source Collector가 이를 사용한다.
- S-CORE Logging은 VEL collection 및 audit log를 수신한다.
- 구성된 evidence persistence가 필요한 경우 S-CORE Persistency를 사용할 수 있다.

이 연계는 VEL을 coordinator, controller, policy manager 또는 decision-maker로 만들지 않는다.

## 6. VEL Internal Component Flow

![VEL internal component flow](../features/assets/VEL_internal_component_flow.svg)

[PlantUML 원본](../features/diagrams/VEL_internal_component_flow.puml)

이 view는 architecture overview를 반복하지 않고 VEL 내부 processing flow에 집중한다. 각 단계의 data artifact, invalid record의 diagnostic 경로, normalized Vehicle Evidence와 optional persistence, VEL Health 및 operational logging의 분리를 보여준다.

```text
source data
    -> schema validation
    -> normalization
    -> quality evaluation
    -> traceability enrichment
    -> persistence and/or publication
```

Logging은 이 흐름을 관찰하고, configuration은 각 처리 단계가 사용하는 contract와 rule을 제공한다.

## 7. Vehicle Evidence 내용

![Vehicle Evidence record 내용과 별도의 VEL Health](../features/assets/VEL_evidence_record_content.svg)

[PlantUML 원본](../features/diagrams/VEL_evidence_record_content.puml)

각 evidence record에는 정규화된 관측값, source identity 및 observation timestamp가 포함된다. source가 correlation identifier를 제공한 경우에는 이를 record에 포함한다. Evidence quality는 각 record 또는 evidence batch에 적용되고, VEL Health는 수집·처리 pipeline의 상태를 별도로 나타낸다. 그림은 논리적 내용을 보여주며 최종 field name이나 output schema를 확정하지 않는다.

구성된 input definition과 normalization mapping에 따라 source data가 관측값으로 변환된다. 아래는 예시이며 필수 source format이나 고정된 변환 규칙이 아니다.

| Source 예시 | 가능한 구성 기반 처리 | Evidence 관측값 |
| --- | --- | --- |
| Runtime metric | source unit 변환 및 range 검증 | 정규화된 숫자와 단위 |
| Hardware metric 또는 status | measurement 검증 또는 availability state 매핑 | resource measurement 또는 operational state |
| S-CORE module state | unknown value를 포함한 source state 매핑 | 정규화된 module state |
| 구성된 event 또는 fault | code 매핑 또는 미등록 원본값 보존 | 유형화된 event 또는 fault 관측값 |

source가 execution context를 제공하면 해당 관측값의 identity 및 correlation 정보에 사용한다. execution context가 반드시 별도 evidence record를 생성하는 것은 아니다. 구성된 output interface는 최종 Vehicle Evidence를 지정된 consumer에게 노출한다.

## 8. 미결 설계 항목

- 담당 data-format 이해관계자와 input interface definition, output evidence definition 및 normalization mapping format 확정
- 각 배포 환경의 platform-specific collector interface 및 S-CORE API profile 확인
- integration별 normalized evidence persistence 및 특정 external transport 필요 여부 결정
