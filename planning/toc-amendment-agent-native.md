# TOC Amendment - Agent-Native Development Environment

## 상태

Phase 5 설계 보강에서 확정한 목차 수정안이다.

`planning/toc.md`의 Phase 3 확정본보다 이 문서의 후반부 장 배치 결정을 우선한다. Phase 5 장별 설계가 일정 수준 진행된 뒤 전체 `planning/toc.md`를 일괄 정리한다.

## 변경 이유

2장의 `Agent-friendly Execution Platform` 설계가 확장되면서 다음 주제가 2장의 역할 범위를 넘어섰다.

- Brain / Hands / Session lifecycle 분리
- One Brain, Multiple Hands
- deterministic sampling
- Agent Hibernate
- PR Babysitter
- Best-of-N
- Time-travel Debugging
- Garbage Collector Agent
- Agent-native Observability
- Compute-aware Orchestration
- Warm Pool
- Shadow / Canary Agent
- Agent Replay
- Agent Platform CI
- Agent-Native Repository

이 주제들은 Repository, Validation, Observability, PM Agent, Retry/Governance를 모두 선행해서 이해해야 하므로 책 후반부 독립 장으로 분리한다.

## 장 수 변경

기존:

- 7개 Part
- 22개 장

변경안:

- 7개 Part
- 23개 장

## Part VII 수정안

# Part VII. Agent-Native Development Environment와 기업 운영

## 21장. Agent-Native Development Environment

### 목적

Agent가 개발환경을 사용하는 단계를 넘어, Repository와 실행환경 자체를 Agent가 탐색·실행·검증·관찰·복구하기 쉬운 형태로 설계하는 방법을 통합한다.

### 독자가 얻는 것

- Model 외에 Context, Harness, Tools, Compute, Validation, Observability, Orchestration이 Agent 성능에 미치는 영향을 설명할 수 있다.
- Brain / Hands / Session lifecycle을 분리할 수 있다.
- 하나의 Brain이 Linux, Browser, Android, Database 같은 여러 execution hand를 사용하게 설계할 수 있다.
- Agent-friendly log, deterministic sampling, Artifact/Replay 구조를 설계할 수 있다.
- PR Babysitter, Best-of-N, Garbage Collector, Shadow/Canary Agent의 적용 조건을 판단할 수 있다.
- PM Agent의 scheduling을 Model에서 Compute/Hand/Agent Count까지 확장할 수 있다.
- Agent Platform 자체를 benchmark, replay, canary 방식으로 검증할 수 있다.

### 핵심 개념

- Agent Performance Model
- Brain / Hands / Session
- One Brain, Multiple Hands
- Deterministic Sampling
- Agent-friendly Log
- Agent Hibernate
- PR Babysitter
- Speculative Coding / Best-of-N
- Time-travel Debugging
- Garbage Collector Agent
- Agent-native Observability
- Compute-aware Orchestration
- Warm Pool
- Shadow Agent
- Canary Agent
- Agent Replay
- Agent Benchmark Suite
- Agent Platform CI
- Agent-Native Repository

### 선행 장

4장, 8장, 10장, 13장, 16장, 17장, 19장, 20장

### 실전 예제

`campus-platform`의 하나의 Verification Brain이 Spring Boot Backend, 관리자 Web, Android App, DB Migration을 각각 다른 execution hand에서 검증하고 Result Gateway를 통해 결과를 통합하는 구조를 설계한다.

상세 설계:

- `planning/agent-native-development-environment.md`
- `examples/campus-platform/agent-native-development-environment.md`

---

## 22장. VPN과 내부망이 있는 Hybrid Agent 시스템

기존 21장의 내용을 유지하되 번호를 22장으로 이동한다.

21장에서 정의한 execution hand abstraction을 기업 내부망에 적용한다.

추가 연결:

- Local Hand: Tibero / Oracle / HSM / 내부 Jenkins / VPN API
- Cloud Hand: 독립 build/test/E2E
- PM / Scheduler: internal-network requirement를 기준으로 실행 위치 선택

선행 장에 21장을 추가한다.

---

## 23장. Full Agentic Development로의 진화와 Governance

기존 22장의 내용을 유지하되 번호를 23장으로 이동한다.

21장에서 추가된 다음 운영 항목을 Governance 입력으로 포함한다.

- Agent Benchmark
- Harness version
- Environment version
- Canary Agent 결과
- Shadow Agent disagreement
- Compute Budget
- Agent Budget
- Replay 결과
- Human Escalation rate

선행 장은 1장부터 22장까지로 변경한다.

## 의존성 변경

```text
... → 18 → 19 → 20 → 21 → 22 → 23
                    |     |     |
                    |     |     └─ Governance
                    |     └─ Enterprise Hybrid
                    └─ Agent-Native Environment
```

21장은 앞 장의 개념을 반복하지 않고 종합한다.

- 4장: Repository Interface
- 8장: Execution Interface
- 10장: Deterministic Validation
- 16장: Durable State
- 17장: Observability/Evaluation
- 19장: PM Scheduling
- 20장: Retry/Escalation

## Phase 5 이후 정리

Phase 5 중 다음을 유지한다.

- 1~3장 설계는 기존 순서대로 완료 상태 유지
- 다음 순차 설계 대상은 4장
- 21장은 본문 작성 없이 planning 문서만 우선 확보

Phase 5 checkpoint에서 `planning/toc.md`를 이 amendment와 통합한다.
