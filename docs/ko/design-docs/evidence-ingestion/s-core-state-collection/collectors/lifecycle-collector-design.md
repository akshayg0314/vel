# Lifecycle 수집기 설계

## 개요

이 문서는 S-CORE Lifecycle 수집기의 **모듈별** 설계를 정의합니다. Lifecycle 모듈에 특정한 API 표면, 관찰 가능한 상태 필드, 소스 식별, 전송 경로를 다룹니다.

> **공통 측면** (수집기 I/O API, 라이프사이클, 오류 및 진단 보고)은 공유 문서에서 정의됩니다: [`../../source-collector-interface-design.md`](../../source-collector-interface-design.md) (모든 VEL 수집기에 공통) 및 [`../s-core-collector-common-design.md`](../s-core-collector-common-design.md) (S-CORE 특화 명명 관례).

---

## 1. 수집기 구성

참조: `lifecycle-collector.yaml` (`interfaces/score-evidence-layer-package/collectors/lifecycle-collector.yaml`)

```yaml
apiVersion: score.dev/v1alpha1
kind: VehicleEvidenceCollector
metadata:
  name: lifecycle-collector
  version: 1.0.0
spec:
  source:
    transport: s-core-api
    api: lifecycle-state-api
    messageType: ScoreLifecycleState
  inputSchemaRef:
    name: score-lifecycle-state
    version: 1.0.0
    path: schemas/score-lifecycle-state-v1.0.0.yaml
  normalizationRuleRef:
    name: score-lifecycle-to-evidence
    version: 1.0.0
    path: mappings/score-lifecycle-to-evidence-v1.0.0.yaml
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

---

## 2. S-CORE API 표면

**컴포넌트:** Launch Manager, Health Monitor (PHM - Platform Health Management).

### 2.1 관찰 가능한 상태

- **실행 대상 상태**: 활성 실행 대상 (문자열 이름, 예: `"Startup"`, `"Off"`, `"Fallback"`).
- **컴포넌트 상태**: 시작됨, 실행 중, 중지됨 (Lifecycle 인터페이스 기준).
- **프로세스 상태**: 활성/생존, 종료 코드, 리소스 사용량.
- **프로세스 런타임 식별자**: 프로세스 식별자 (`pid`), lifecycle API를 통해 노출되며 스키마에서 `details.pid`로 표시됩니다.
- **준비 조건**: 컴포넌트가 준비 상태에 도달했는지 여부 (프로세스 상태 전환에서 유추).
- **Health Monitor 상태**: 데드라인 모니터, 로직 모니터, 하트비트 모니터 상태. 감독 상태 (ok/failed, 유형별 오류 존재 여부에서 파생), 실패한 감독 유형 (heartbeat/deadline/logic), 특정 유형별 오류 (deadline_error, heartbeat_error, logic_error), 데드라인 위반, 실패한 감독 주기, 활성 표시 횟수, 프로세스 실행 오류, 프로세스 그룹 식별자, 복구 상태를 포함합니다.

### 2.2 데이터 정의 (호출 가능한 API가 아닌 유형)

- **`ProcessState` 열거형** (`lifecycle/score/launch_manager/src/daemon/src/process_group_manager/process_state.hpp`):
  ```cpp
  enum class ProcessState : std::uint8_t {
      kIdle = 0,         // process in idle state.
      kStarting = 1,     // process in starting state.
      kRunning = 2,      // process in running state.
      kTerminating = 3,  // process in terminating state.
      kTerminated = 4,   // process in terminated state.
      kFailed = 5        // process failed to start.
  };
  ```

- **`GraphState` 열거형** (`lifecycle/score/launch_manager/src/daemon/src/process_group_manager/details/graph.hpp`):
  ```cpp
  enum class GraphState : std::uint8_t {
      kSuccess = 0U,        // Graph is not running and process group state is known
      kInTransition = 1U,   // Graph is running, process group state is in transition
      kAborting = 2U,       // Graph is running but has been aborted due to error
      kCancelled = 3U,      // Graph is running but has been cancelled because a new transition is pending
      kUndefinedState = 4U  // Graph is not running but process group state is not known
  };
  ```

아래 다이어그램은 `ProcessState`, `GraphState`, 복구 작업 구성 유형, Health Monitor 감독 유형을 유형 모델로 보여줍니다. `RestartAction`과 `SwitchRunTargetAction`은 **별개의** 구성 구조체로, **서로 다른** 선택적 슬롯 (`ready_recovery_action` / `recovery_action`)을 차지합니다 — 태그된 유니온이나 단일 열거형이 아닙니다.

![Lifecycle 유형 모델](../../../../../features/assets/lifecycle/Lifecycle_type_model.svg)

[PlantUML 소스](../../../../../features/diagrams/lifecycle/Lifecycle_type_model.puml)

아래 클래스 다이어그램은 VEL 측 수집기 구현 클래스 (LifecycleCollector, LifecycleStateReader, LifecycleStateMapper, CollectorDiagnostics)와 이들이 읽는 소스 측 S-CORE 유형을 보여줍니다:

![Lifecycle 수집기 클래스 다이어그램](../../../../../features/assets/lifecycle/Lifecycle_collector_class_diagram.svg)

[PlantUML 소스](../../../../../features/diagrams/lifecycle/Lifecycle_collector_class_diagram.puml)

### 2.3 Lifecycle API 필드

- **`pid` / `exit_code` / `process_execution_error`** — lifecycle API를 통해 외부 모니터 알림 API (`comp_req__launch_man__ext_monitor_notify` 요구사항)로 노출됩니다.
- **활성 보고** — Health Monitor는 Alive API (`report_alive()` / `report_failure()`)를 통해 Launch Manager에 생존을 보고합니다. 참고: 이는 **쓰기** 경로입니다 (감독되는 프로세스가 자체적으로 보고). VEL은 이를 사용하여 다른 프로세스의 상태를 읽을 수 없습니다.
- **Health Monitor 감독 유형** — Health Monitor는 세 가지 감독 유형 (데드라인, 하트비트, 로직)을 평가하며, 각각 고유한 오류 열거형과 상태 스냅샷을 가지며 외부 모니터 알림 API를 통해 노출됩니다.

### 2.4 수집기 접근 패턴

```cpp
#include "score/mw/launch_manager/process_group_manager/process_state.hpp"
#include "score/mw/launch_manager/process_group_manager/details/graph.hpp"

// 1. 현재 프로세스 그룹 (그래프) 상태를 얻습니다.
score::mw::lifecycle::GraphState graph_state = graph.getState();  // kSuccess, kInTransition, ...

// 2. 그룹의 각 프로세스에 대해 프로세스 상태를 얻습니다.
score::mw::lifecycle::ProcessState proc_state = process_node.getState();  // kIdle, kStarting, kRunning, ...

// 3. 종료 코드, PID, 프로세스 실행 오류는 ProcessInfoNode의 getter로 노출되지 않습니다
//    (공개 API는 getPid(), getState(), getTerminationTimeout(),
//    getControlClientChannel()로 제한됨 — 공개 getExitCode() 없음). 이러한 필드는
//    수집기가 구독하는 외부 모니터 알림 콜백 페이로드
//    (comp_req__launch_man__ext_monitor_notify)를 통해서만 관찰할 수 있습니다.

// 4. 수집기는 이를 score-lifecycle-state 스키마를 준수하는 원시 상태 레코드로 매핑합니다.
```

아래 시퀀스는 `pid`/`exit_code`/`process_execution_error`에 대한 외부 모니터 알림 콜백 경로를 포함한 전체 접근 패턴 상호작용을 보여줍니다:

![Lifecycle 수집기 접근 패턴 시퀀스](../../../../../features/assets/lifecycle/Lifecycle_collector_sequence.svg)

[PlantUML 소스](../../../../../features/diagrams/lifecycle/Lifecycle_collector_sequence.puml)

---

## 3. 소스 식별

| 필드 | 값 |
|-------|-------|
| `source_id` | `score-lifecycle` |
| `workload_id` | `launch-manager` 또는 `health-monitor` |
| `resource_id` | `component:<이름>` 또는 `run-target:<이름>` 또는 `monitor:<유형>` |

---

## 4. 상태 필드 (원시 Evidence 스키마)

수집기는 `score-lifecycle-state` 스키마 (`interfaces/score-evidence-layer-package/schemas/score-lifecycle-state-v1.0.0.yaml`)를 준수하는 원시 상태 레코드를 생성합니다:

| 필드 | 경로 | 유형 | 필수 |
|-------|------|------|----------|
| `message_id` | `$.message_id` | string | ✓ |
| `source_id` | `$.source_id` | string | ✓ |
| `workload_id` | `$.workload_id` | string | ✓ |
| `execution_id` | `$.execution_id` | string | ✓ |
| `resource_id` | `$.resource_id` | string | ✓ |
| `timestamp_ns` | `$.timestamp_ns` | uint64 | ✓ |
| `state` | `$.state` | enum | ✓ |
| `previous_state` | `$.previous_state` | enum | – |
| `transition_time_ns` | `$.transition_time_ns` | uint64 | – |
| `run_target` | `$.details.run_target` | string | – |
| `exit_code` | `$.details.exit_code` | int32 | – |
| `pid` | `$.details.pid` | uint32 | – |
| `graph_state` | `$.details.graph_state` | enum | – |
| `supervision_status` | `$.details.supervision_status` | enum | – |
| `failed_supervision_cycles` | `$.details.failed_supervision_cycles` | uint32 | – |
| `failed_supervision_type` | `$.details.failed_supervision_type` | enum | – |
| `deadline_error` | `$.details.deadline_error` | enum | – |
| `heartbeat_error` | `$.details.heartbeat_error` | enum | – |
| `logic_error` | `$.details.logic_error` | enum | – |
| `deadline_violation` | `$.details.deadline_violation` | bool | – |
| `alive_indication_count` | `$.details.alive_indication_count` | uint32 | – |
| `process_execution_error` | `$.details.process_execution_error` | uint32 | – |
| `process_group_id` | `$.details.process_group_id` | string | – |
| `recovery_state` | `$.details.recovery_state` | string | – |

---

## 5. 전송 경로

아래 정적 뷰는 VEL Evidence Ingestion 아키텍처에서 Lifecycle 수집기의 위치와 Schema Validator, Normalization Processor, DDS 게시 토픽과의 상호작용을 보여줍니다:

![Lifecycle 수집기 정적 뷰](../../../../../features/assets/lifecycle/Lifecycle_collector_static_view.svg)

[PlantUML 소스](../../../../../features/diagrams/lifecycle/Lifecycle_collector_static_view.puml)

![Lifecycle 수집기 전송 흐름](../../../../../features/assets/lifecycle/Lifecycle_collector_transport_flow.svg)

[PlantUML 소스](../../../../../features/diagrams/lifecycle/Lifecycle_collector_transport_flow.puml)

---

## 6. S-CORE API 경계

VEL은 S-CORE가 **안정적이고 공개된 외부 API**를 통해 노출하는 데이터만 수집할 수 있습니다. Lifecycle 모듈의 경우:

- **허용**: 외부 모니터 알림 API를 통해 `ProcessState`, `GraphState`, `pid`, `exit_code`, `process_execution_error`, Health Monitor 감독 상태 읽기.
- **허용되지 않음**: 제어 명령 전송 (프로세스 시작/중지/재시작), `report_alive()` / `report_failure()` 호출 (쓰기 경로), 또는 모든 라이프사이클/프로세스/하드웨어 제어 작업.

이는 `saf_req__vel__*` (라이프사이클/프로세스/컨테이너/하드웨어 제어 없음, 조정 없음, OEM 결정 없음)를 충족합니다.

---

## 7. 추적성

| 항목 | 참조 |
|------|-----------|
| 설계 | Lifecycle 수집기 설계 |
| 요구사항 | FR-VEL-003, AOU-VEL-002 |
| 수집기 구성 | `lifecycle-collector.yaml` (`interfaces/score-evidence-layer-package/collectors/lifecycle-collector.yaml`) |
| 입력 스키마 | `score-lifecycle-state-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/schemas/score-lifecycle-state-v1.0.0.yaml`) |
| 정규화 매핑 | `score-lifecycle-to-evidence-v1.0.0.yaml` (`interfaces/score-evidence-layer-package/mappings/score-lifecycle-to-evidence-v1.0.0.yaml`) |
