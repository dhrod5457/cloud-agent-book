# 18장 설계 - 다음 단계: Harness와 Orchestration

## 장의 목표

이 책에서 설명한 Cloud Agent 활용 구조를 기반으로, 다음 단계에서 어떤 방향으로 확장할 수 있는지 짧게 정리한다.

이 장은 Agent Platform 일반론을 새로 시작하는 장이 아니다. 앞 장까지 만든 Local/Cloud Routing, Prepared Environment, Runner-first, Task Contract, Result Gateway, Git Handoff, Event-driven Workflow를 더 자동화할 때 자연스럽게 나타나는 후속 주제만 소개한다.

핵심 질문:

> Cloud Agent 활용이 안정화된 뒤, 다음으로 자동화할 수 있는 것은 무엇인가?

---

## 핵심 주장

> 먼저 Cloud Agent를 잘 사용하는 Workflow를 만들고, 그 다음에 Orchestration을 자동화한다.

> 긴 Prompt보다 잘 준비된 Harness와 실행환경이 반복 작업의 품질을 더 안정적으로 만든다.

> 미래의 개발환경은 Agent 수를 늘리는 방향보다 Task Routing, Compute, Validation, Evidence를 자동 연결하는 방향으로 발전할 수 있다.

이 장의 모든 개념은 미래 전망 또는 Advanced Topic으로 제한한다.

---

## 독자가 얻는 것

- Harness Engineering을 Cloud Agent 관점에서 설명할 수 있다.
- Task Routing을 자동화하는 Orchestration의 역할을 이해할 수 있다.
- Brain / Hands를 Token과 Compute 분리를 설명하는 후속 모델로 이해할 수 있다.
- Agent-native Observability가 왜 필요한지 방향을 이해할 수 있다.
- Best-of-N을 일반 전략이 아니라 제한된 고급 병렬화 사례로 판단할 수 있다.
- 현재 책의 범위와 후속 연구 범위를 구분할 수 있다.

---

# 절 구성

## 18.1 지금까지 만든 구조가 출발점이다

책의 기본 Workflow:

```text
Developer / Local Agent
        ↓
Task Routing
        ↓
Task Contract
        ↓
Git Handoff
        ↓
Prepared Cloud Environment
        ↓
Runner-first
        ↓
필요한 경우 Cloud Agent
        ↓
Result Gateway / Evidence
        ↓
PR / Local Review
```

이 구조가 안정화되기 전에 복잡한 Orchestration부터 도입하지 않는다.

---

## 18.2 Harness Engineering

Cloud Agent가 반복해서 다음을 추론하게 하지 않는다.

- 어떻게 setup하는가
- 어떻게 build하는가
- 어떻게 test하는가
- 어떤 파일을 읽어야 하는가
- 무엇이 PASS인가
- 실패 로그를 어디서 찾는가

Repository와 실행환경이 이를 제공하도록 만든다.

예:

```text
AGENTS.md
scripts/build
scripts/test
scripts/verify
Task Contract
Result Gateway
Prepared Environment
```

핵심:

> Agent가 반복해서 실패하면 Prompt보다 실행환경과 도구를 먼저 점검한다.

이 책에서는 Harness를 Cloud Agent의 탐색/실행/검증 비용을 줄이는 수단으로만 다룬다.

---

## 18.3 Task Routing의 자동화

현재 책에서는 Developer 또는 Local Agent가 Task를 분류한다.

후속 단계에서는 Scheduler가 다음 정보를 보고 실행 위치를 고를 수 있다.

```text
Task
- type
- scope
- context size
- internal network
- validation
- expected duration
- compute demand
```

예:

```text
Unit Test
→ Cloud Runner

재현 가능한 Bug Fix
→ Cloud Agent

HSM 검증
→ Local

Migration
→ Cloud validation + Local Tibero validation
```

이를 Agent Platform 전체 설계로 확장하지 않고 Routing 자동화의 방향으로만 소개한다.

---

## 18.4 Brain / Hands

3장에서 사용한 간단 모델을 다시 연결한다.

```text
Brain
→ LLM 판단

Hands
→ Container / Browser / Emulator / DB에서 실행
```

핵심은 LLM lifecycle과 Compute lifecycle을 반드시 같은 것으로 볼 필요가 없다는 점이다.

후속 시스템에서는 하나의 판단 주체가 여러 실행환경을 필요할 때 호출할 수 있다.

상세 Session lifecycle, Hibernate, One Brain Multiple Hands 설계는 `planning/future-topics.md`로 넘긴다.

---

## 18.5 Agent-native Observability

현재 책에서는 Evidence와 Artifact를 사람이 Review하고 Agent가 필요한 실패 정보를 조회한다.

후속 단계에서는 Agent가 구조화된 Query로 직접 상태를 조회할 수 있다.

예:

```text
get_errors(service, since)
get_failed_tests(task)
get_http_failures(status)
get_browser_console_errors(task)
```

방향:

```text
Dashboard for Humans
+
Queryable Evidence for Agents
```

상세 Observability Platform은 후속 주제로 남긴다.

---

## 18.6 Best-of-N

어려운 Bug에 한해 여러 Cloud Agent가 독립 후보를 만들고 Runner가 검증하는 방식을 소개한다.

```text
Bug
 |
 +-- Agent A → Patch A
 +-- Agent B → Patch B
 +-- Agent C → Patch C
            ↓
      Deterministic Validation
            ↓
         Candidate 선택
```

기본값은 N=1이다.

적합한 경우:

- 해결 실패 비용이 큼
- 후보를 자동 검증 가능
- 서로 독립 Branch에서 실행 가능

부적합한 경우:

- 작은 Task
- 검증 기준 불명확
- Merge/Review 비용이 더 큼

---

## 18.7 Compute-aware Scheduling

후속 Scheduler는 Agent만 선택하는 것이 아니라 실행 자원도 선택할 수 있다.

예:

```text
unit-test
→ Runner / small compute

integration-test
→ Runner / Docker available

frontend-e2e
→ Browser environment

complex-debug
→ Agent + larger context
```

CPU/RAM 값 자체는 제품과 환경에 따라 달라지므로 본문 고정값으로 제시하지 않는다.

---

## 18.8 Warm Worker와 Snapshot의 발전

9장의 Prepared Environment를 더 발전시키면 짧고 반복적인 Task용 Warm Worker Pool을 생각할 수 있다.

```text
READY Worker
→ Task
→ Reset
→ READY
```

반면 장시간/격리 요구가 큰 작업은 별도 Ephemeral Worker가 더 적합할 수 있다.

이 장에서는 운영 플랫폼 상세 설계로 확장하지 않는다.

---

## 18.9 Replay와 Agent 평가

Cloud Agent Workflow 자체도 변경될 수 있다.

후속 단계에서는 같은 Task를 다시 실행하여 다음을 비교할 수 있다.

- 성공 여부
- Token 사용량
- 실행시간
- Retry
- Human Intervention
- Regression

이를 위한 Replay/Benchmark 상세 구조는 후속 주제로 넘긴다.

---

## 18.10 Cloud Agent 중심 개발환경의 발전 단계

```text
1. Local Agent 사용
        ↓
2. Cloud Agent에 독립 Task 위임
        ↓
3. Prepared Environment / Runner-first
        ↓
4. 여러 Cloud Worker 병렬화
        ↓
5. Local + Cloud Hybrid Workflow
        ↓
6. Event-driven Cloud Task
        ↓
7. Routing / Environment / Validation 자동화
```

이 책의 목표는 6단계까지 독자가 실제 프로젝트에서 구성할 수 있게 하는 것이다.

7단계 이후는 후속 연구 영역이다.

---

## 18.11 무엇을 지금 하지 않을 것인가

현재 책의 독립 주제로 확장하지 않는다.

- Agent Memory Architecture
- Agent Security Platform
- Agent OS
- Agent Chaos Engineering
- Garbage Collector Agent
- Shadow Agent
- Canary Agent
- Agent Governance Platform
- General Multi-Agent Theory
- Agent 조직론

필요한 설계는 `planning/future-topics.md`에 보존한다.

---

## 18.12 최종 Workflow 다시 보기

책 전체의 최종 구조:

```text
Developer / Local Agent
        ↓
Architecture / Task Split
        ↓
Local or Cloud Routing
        ↓
Git Handoff
        ↓
Prepared Cloud Environment
        ↓
Runner
   ├─ PASS → Evidence
   └─ FAIL → Cloud Agent
                  ↓
                Fix
                  ↓
              Runner
        ↓
Evidence / PR
        ↓
Local Review / Internal Validation
        ↓
Merge
```

독자가 마지막에 기억해야 할 질문:

> 이 작업은 Local에서 하고, 이 작업은 Cloud Agent에게 보내자.

---

# 좋은 사례와 나쁜 사례

## Orchestration 도입

좋지 않은 방식:

```text
Cloud Task 기준도 없음
→ 복잡한 Multi-Agent Orchestrator부터 구축
```

권장:

```text
Task Routing / Runner / Evidence가 먼저 안정화
→ 반복되는 결정부터 자동화
```

## Agent 실패 대응

좋지 않은 방식:

```text
Agent 실패
→ Prompt만 계속 길게 수정
```

권장:

```text
환경 / Harness / Validation / Context Boundary 점검
```

## Best-of-N

좋지 않은 방식:

```text
모든 Task에 Agent 5개
```

권장:

```text
검증 가치가 높은 어려운 Task에 제한 적용
```

---

# 필요한 그림

1. 현재 Cloud Workflow → 미래 자동화 단계
2. Harness 구성 요소
3. Task Routing 자동화
4. Brain / Hands 간단 모델
5. Best-of-N 검증 흐름
6. Cloud Agent 발전 단계

---

# 앞 장과의 연결

17장:
Cloud를 사용하지 않거나 Local로 되돌리는 기준

18장:
Cloud 활용이 안정화된 이후 자동화 가능한 다음 단계

책 종료:
Agent Platform 구축이 아니라 실제 Local/Cloud 선택 능력으로 돌아온다.

---

# 후속 자료

- `planning/future-topics.md`
- `planning/agent-native-development-environment.md`
- `research/anthropic/agent-native-development-environment.md`

이 자료는 현재 책의 핵심 범위보다 확장된 연구 자료다.

---

# 장의 결론 메시지

> Cloud Agent 활용이 먼저이고 Agent Platform은 그 다음 문제다.

> 더 많은 Agent보다 더 나은 Task Routing, Harness, Validation이 먼저다.

> 이 책의 최종 목적은 독자가 자신의 개발 흐름에서 Local과 Cloud의 역할을 직접 나눌 수 있게 하는 것이다.
