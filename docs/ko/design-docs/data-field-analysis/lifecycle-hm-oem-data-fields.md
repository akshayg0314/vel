# Lifecycle / Health Monitoring — OEM 데이터 필드

> **목적:** 이 문서는 OEM(Original Equipment Manufacturer) 사용 사례(차량 상태 모니터링, 진단, 안전 분석, 차량 원격 분석)에 논리적으로 유용한 S-CORE Lifecycle 및 Health Monitoring 모듈의 데이터 필드를 나열합니다.
>
> **접근성:** 이 문서는 S-CORE Lifecycle 및 Health Monitoring 모듈이 노출하는 **논리적으로 유용한** OEM 필드를 나열합니다.

---

## 1. 프로세스 Lifecycle 상태

| 필드 | 유형 | 허용 값 | 설명 | OEM 관련성 |
|-------|------|----------------|-------------|---------------|
| `state` | enum | `idle`, `starting`, `running`, `terminating`, `terminated`, `failed` | 관리되는 프로세스의 현재 상태. | 소프트웨어 컴포넌트가 실행 중인지, 중지되었는지, 시작 중인지, 실패했는지 나타냄 — 차량 수준 가용성 및 진단에 필수적. |
| `previous_state` | enum (`ProcessState`) | `idle` (0), `starting` (1), `running` (2), `terminating` (3), `terminated` (4), `failed` (5) | 현재 상태 이전의 프로세스 상태. 파생 필드 (소스에 유지되지 않음; 상태 전환에서 계산됨). | 전환 분석 가능 (예: 컴포넌트가 `failed`를 얼마나 자주 거치는지). |
| `transition_time_ns` | uint64 | — | 상태 전환이 발생한 타임스탬프 (ns). | 소프트웨어 상태 변경을 차량 이벤트 또는 다른 evidence와 상관시킴. |
| `exit_code` | int32 | — | 종료 시 프로세스 종료 코드. 0 = 성공; 0이 아닌 값 = 실패/예기치 않은 종료 (OS 보고 상태 from wait()). | 정상 종료와 비정상 종료를 구분; 근본 원인 분석 지원. |
| `process_execution_error` | uint32 | 1=GeneralError, 2=InvalidArguments, 3=CommunicationError, 4=MetaModelError, 5=Cancelled, 6=Failed, 7=FailedUnexpectedTerminationOnExit, 8=FailedUnexpectedTerminationOnEnter, 9=InvalidTransition, 10=AlreadyInState, 11=InTransitionToSameState, 12=NoTimeStamp, 13=CycleOverrun, 14=ActivationInProgress, 15=RequestQueueIsFull, 16=RunTargetDoesntExist | 프로세스 실행기(`ExecErrc`)의 오류 코드. 현재 구현은 중단 시 `kGeneralError` (1)을 하드코딩합니다. | 시작 실패 식별 (바이너리 누락, 권한, 리소스 고갈). |
| `pid` | uint32 | — | lifecycle API가 노출하는 프로세스 식별자 (시작된 적이 없으면 0). | 법의학 및 lifecycle API 로그 교차 참조를 위한 정확한 프로세스 인스턴스 식별. |

---

## 2. 프로세스 그룹 / 그래프 상태

| 필드 | 유형 | 허용 값 | 설명 | OEM 관련성 |
|-------|------|----------------|-------------|---------------|
| `graph_state` | enum | `success`, `in-transition`, `aborting`, `cancelled`, `undefined-state` | 프로세스 그룹 (종속성 그래프) 상태. | 조정된 컴포넌트 집합 (예: ADAS 스택)이 완전히 실행 중인지, 전환 중인지, 중단되었는지 보여줌. |
| `process_group_id` | string | — | 프로세스 그룹 식별자. 문자열 이름에서 파생된 `IdentifierHash`. | 차량 원격 분석 상태 집계를 위한 관련 컴포넌트 그룹화. |
| `run_target` | string | — | 활성 실행 대상 이름 (통합자 정의, 예: `"Startup"`, `"Off"`, `"Fallback"`, `"Running"`, `"SafeState"`). | 활성 소프트웨어 구성을 나타냄 — 규제 및 안전 추적성에 중요. |

---

## 3. Health Monitor 감독 상태

| 필드 | 유형 | 허용 값 | 설명 | OEM 관련성 |
|-------|------|----------------|-------------|---------------|
| `supervision_status` | enum | `ok`, `failed` | 감독되는 컴포넌트의 전체 감독 상태 (유형별 오류 존재 여부에서 파생). | 능동 감독 하의 컴포넌트에 대한 기본 상태 표시기. |
| `failed_supervision_type` | enum | `heartbeat`, `deadline`, `logic` | 실패를 감지한 감독 유형. | 실패 모드 구분: 하트비트 누락, 데드라인 누락, 잘못된 상태 전환 — 진단에 중요. |
| `failed_supervision_cycles` | uint32 | — | 연속 실패한 감독 주기 수. | 결함의 심각도/지속성 나타냄 (일시적 vs. 지속적). |
| `deadline_violation` | bool | `true`/`false` | 데드라인이 위반되었는지 여부. | 실시간 스케줄링 실패를 직접 나타냄 (예: FEO 주기 오버런). |
| `alive_indication_count` | uint32 | — | 보고된 활성 표시 횟수. | 컴포넌트가 능동적으로 생존을 보고하고 있음을 확인; 부재는 정지 상태를 나타냄. |

---

## 4. 감독 유형 세부 사항

### 4.1 데드라인 감독

| 필드 | 유형 | 설명 | OEM 관련성 |
|-------|------|-------------|---------------|
| `deadline_error` | enum (`too_early`, `too_late`) | 데드라인 평가 오류. | `too_late` = 실시간 오버런; `too_early` = 조기 완료 (로직 결함일 수 있음). |
| `deadline_state.timestamp_ms` | uint32 | 밀리초 단위 타임스탬프. | 데드라인 모니터 상태 스냅샷. |
| `deadline_state.is_running` | bool | 실행 플래그. | 데드라인 모니터가 능동적으로 실행 중인지 나타냄. |
| `deadline_state.is_stopped` | bool | 중지 플래그. | 데드라인 모니터가 중지되었는지 나타냄. |
| `deadline_state.is_underrun` | bool | "너무 일찍 완료됨" 플래그. | `is_underrun`은 주기가 최소 허용 시간보다 일찍 완료되었음을 나타냄 — 로직 이상. |

### 4.2 하트비트 감독

| 필드 | 유형 | 설명 | OEM 관련성 |
|-------|------|-------------|---------------|
| `heartbeat_error` | enum (`too_early`, `too_late`, `multiple_heartbeats`) | 하트비트 평가 오류. | `too_late` = 컴포넌트 정지; `multiple_heartbeats` = 이중 실행 / 재진입 버그. |
| `heartbeat_state.heartbeat_timestamp` | uint64 (62-bit) | 하트비트 타임스탬프. | 주기적 생존 확인; 타임스탬프 드리프트는 클록/부하 문제와 상관됨. |
| `heartbeat_state.counter` | uint8 (2-bit) | 하트비트 카운터, 3에서 포화. | 하트비트 케이던스 추적; 포화는 지속적인 생존을 나타냄. |

### 4.3 로직 감독

| 필드 | 유형 | 설명 | OEM 관련성 |
|-------|------|-------------|---------------|
| `logic_error` | enum (`invalid_state`, `invalid_transition`, `unmapped_error`) | 로직 평가 오류. | 컴포넌트가 예기치 않은 상태에 들어갔거나 잘못된 전환을 했음을 나타냄 — 소프트웨어 로직 결함. |
| `logic_state.current_state_index` | uint64 (56-bit) | 현재 상태 인덱스. | 상태 머신의 현재 상태 추적; 실패 시 컴포넌트 동작 이해에 유용. |
| `logic_state.monitor_status` | enum | `ok` (0), `invalid_state` (1), `invalid_transition` (2), `unmapped_error` (3) | 모니터 상태. | 로직 모니터가 정상인지 이상을 감지했는지 나타냄. |

---

## 5. 복구 상태

| 필드 | 유형 | 허용 값 | 설명 | OEM 관련성 |
|-------|------|----------------|-------------|---------------|
| `recovery_state` | string | — | 폴백 실행 대상 이름 (`"fallback"` — `process_group_manager.hpp`의 하드코딩된 `IdentifierHash`). 복구 상태 머신이 아닙니다; 복구 시 시스템이 전환되는 실행 대상을 식별합니다. | 복구 중 시스템이 폴백하는 실행 대상을 보여줌 — 가용성 및 안전 분석에 중요. |
| `recovery_action` | struct | `RestartAction { number_of_attempts: uint32, delay_before_restart_ms: uint32 }` 또는 `SwitchRunTargetAction { run_target: string }` | 수행된 복구 작업. `ready_recovery_action`은 `RestartAction`을 사용; `recovery_action`은 `SwitchRunTargetAction`을 사용. | 완화 전략 식별; 차량 원격 분석 정책 평가 지원. |

> **`recovery_action` 유형 참고:** 이는 **단일 런타임 열거형이 아닙니다.** 소스 (`recovery_action_config.hpp`)에서 `RestartAction`과 `SwitchRunTargetAction`은 두 개의 별개 구성 구조체이며, 각각 다른 선택적 구성 슬롯 (`ready_recovery_action: Optional<RestartAction>`, `recovery_action: Optional<SwitchRunTargetAction>`)에서 사용됩니다. 어떤 작업이 적용되는지는 태그된 유니온/열거형이 아닌 채워진 슬롯에 의해 결정됩니다. `recovery_action`은 현재 VEL evidence 스키마의 일부가 아닙니다 (`recovery_state`만 수집됨); 완전성을 위해 여기에 문서화됩니다.
>
> **TODO:** `recovery_action`이 스키마에 추가되면 각 작업이 다른 매개변수를 전달하므로 일반 열거형이 아닌 `type` 판별자 (예: `restart` / `switch-run-target`)가 있는 객체로 표현해야 합니다.

---

## 6. 활성 보고 (HM과 LM 사이의 브리지)

| 필드 | 유형 | 설명 | OEM 관련성 |
|-------|------|-------------|---------------|
| `alive_status` | enum (`alive`, `failure`) | 컴포넌트가 활성 또는 실패를 보고했는지 여부. | 실패를 보고하지 않고 `alive` 보고를 중지한 컴포넌트는 정지된 것으로 간주 — 워치독 스타일 모니터링에 중요. |
| `identifier` | string | 활성 보고에 사용되는 프로세스 식별자 (`IDENTIFIER` env에서). | 활성 알림을 특정 컴포넌트 인스턴스에 매핑. |

---

## 7. 상관 / 식별 필드

| 필드 | 유형 | 설명 | OEM 관련성 |
|-------|------|-------------|---------------|
| `source_id` | string | 모듈 식별자 (예: `score-lifecycle`). | 추적성을 위한 데이터 소스 식별. |
| `workload_id` | string | 컴포넌트 식별자 (예: `launch-manager`, `health-monitor`). | evidence를 특정 S-CORE 워크로드에 연결. |
| `resource_id` | string | 리소스 식별자 (예: `component:<이름>`, `run-target:<이름>`, `monitor:<유형>`). | 모니터링되는 정확한 리소스 지정. |
| `execution_id` | string | 실행 인스턴스 식별자. | 동일한 실행의 여러 evidence 레코드 상관. |
| `timestamp_ns` | uint64 | 이벤트 타임스탬프 (나노초, wall-clock). | 모든 차량 evidence의 시간적 상관 가능. |

---

## 8. VEL Evidence 유형 매핑

위의 각 필드는 VEL 패키지의 정규화된 evidence 유형에 매핑됩니다:

| Evidence 유형 | 소스 필드 |
|---------------|-----------------|
| `s-core.lifecycle.component.state` | `state` |
| `s-core.lifecycle.process-group.state` | `graph_state` |
| `s-core.lifecycle.process.exit_code` | `exit_code` |
| `s-core.lifecycle.process.pid` | `pid` |
| `s-core.lifecycle.run-target` | `run_target` |
| `s-core.lifecycle.health.supervision_status` | `supervision_status` |
| `s-core.lifecycle.health.failed_supervision_type` | `failed_supervision_type` |
| `s-core.lifecycle.health.deadline_error` | `deadline_error` |
| `s-core.lifecycle.health.heartbeat_error` | `heartbeat_error` |
| `s-core.lifecycle.health.logic_error` | `logic_error` |
| `s-core.lifecycle.health.deadline_violation` | `deadline_violation` |
| `s-core.lifecycle.health.failed_supervision_cycles` | `failed_supervision_cycles` |
| `s-core.lifecycle.health.alive_indication_count` | `alive_indication_count` |
| `s-core.lifecycle.process.process_execution_error` | `process_execution_error` |
| `s-core.lifecycle.process-group.process_group_id` | `process_group_id` |
| `s-core.lifecycle.recovery.recovery_state` | `recovery_state` |

> **명명 규칙:** 소스 필드 이름에 직접 해당하는 Evidence 유형 경로 세그먼트는 소스 필드의 밑줄을 유지합니다 (예: `deadline_error`, `process_group_id`). 하이픈 세그먼트 (예: `process-group`, `health`)는 소스 필드 이름이 아닌 구조적 범주 이름입니다.

---

## 9. 참고 사항

- **읽기 전용**: VEL은 이러한 필드만 읽습니다. 제어 명령을 보내지 않습니다 (`saf_req__vel__*` 기준).
- **열거형 안정성**: 필드 이름과 열거형 값은 안정적인 계약을 보장하기 위해 S-CORE lifecycle API와 정렬됩니다.
- **OEM 필터링**: OEM 관련으로 표시된 필드는 차량 수준 모니터링, 진단, 안전, 차량 원격 분석을 지원하는 필드입니다. 내부 카운터 (예: 원시 FFI 핸들 값)는 의도적으로 제외됩니다.
- **수집 가능성**: 섹션 1–5의 S-CORE 특정 필드는 S-CORE 외부 모니터 알림 API (`comp_req__launch_man__ext_monitor_notify`)를 통해 노출됩니다.
