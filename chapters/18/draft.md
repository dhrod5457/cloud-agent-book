# 18장. 다음 단계: Harness와 Orchestration

17장까지 이 책은 Cloud Agent를 실제 개발 Workflow에 넣는 방법을 정리했다.

핵심 구조는 이미 완성되어 있다.

```text
Local / Cloud Routing
→ Task Contract
→ Git Handoff
→ Prepared Environment
→ Runner-first
→ 필요한 경우 Cloud Agent
→ Evidence / PR
→ Local Review / Internal Validation
```

이 구조가 반복해서 동작하면 다음 질문이 생긴다.

```text
반복되는 준비와 판단을 어디까지 자동화할 수 있을까?
```

이 장은 새로운 Agent Platform을 설계하는 장이 아니다.

앞에서 만든 Workflow를 안정화한 뒤 자연스럽게 나타나는 다음 단계만 짧게 정리한다.

> 먼저 Cloud Agent를 잘 사용하는 Workflow를 만들고, 그 다음에 Orchestration을 자동화한다.

---

## 1. 다음 단계는 더 많은 Agent가 아니라 더 적은 반복 판단이다

앞 장까지 사람이 반복해서 결정한 것은 대체로 다음과 같다.

```text
어떤 Task를 Cloud로 보낼 것인가?
어떤 Environment를 사용할 것인가?
Runner인가 Agent인가?
어떤 Validation을 실행할 것인가?
어떤 Evidence를 남길 것인가?
언제 Retry하고 언제 멈출 것인가?
```

이 결정이 프로젝트에서 반복되고 기준이 안정되면 일부를 코드와 설정으로 옮길 수 있다.

```text
Task Type
→ Environment
→ Runner / Agent
→ Validation
→ Evidence
```

자동화의 목적은 Agent 수를 늘리는 것이 아니라 같은 결정을 매번 다시 하지 않는 것이다.

---

## 2. Harness는 Agent가 반복해서 추론할 일을 줄인다

Agent가 Repository에 들어올 때마다 다음을 다시 찾아야 한다면 비용이 반복된다.

```text
어떻게 setup하는가?
어떻게 build하는가?
어떻게 test하는가?
어떤 파일부터 읽는가?
무엇이 PASS인가?
결과는 어디에 남는가?
```

이 정보를 Prompt에 계속 추가하기보다 Repository와 실행환경이 제공하도록 만들 수 있다.

예:

```text
AGENTS.md
scripts/build
scripts/test
scripts/verify
Task Contract
Prepared Environment
Result Gateway
```

예를 들어 다음 명령 하나가 인증 모듈의 표준 검증을 수행한다고 하자.

```bash
./scripts/verify-auth.sh
```

Agent 입장에서는 다음 구조가 된다.

```text
Task Contract
→ 표준 Validation
→ 구조화된 Result
```

Harness는 거대한 프레임워크가 아니다.

> Agent가 반복해서 탐색하고 추론해야 했던 개발 규칙을 발견 가능하고 실행 가능한 형태로 꺼내놓는 것에 가깝다.

Agent가 같은 종류의 환경·빌드·검증 문제로 반복 실패한다면 Prompt를 길게 만들기 전에 Harness와 실행환경을 먼저 점검한다.

---

## 3. Routing도 반복되면 일부 자동화할 수 있다

5장과 17장에서 사람이 Routing을 판단했다.

반복 패턴이 안정되면 일부는 규칙으로 만들 수 있다.

```text
unit-test
→ Cloud Runner

admin-e2e
→ frontend-e2e Runner

reproducible-bug-fix
→ Cloud Agent 후보

requires-hsm
→ Local / Hybrid
```

Task Metadata가 있다면 다음처럼 연결할 수 있다.

```yaml
type: bug-fix
scope: auth
reproducible: true
internal_network: false
validation: ./gradlew test --tests AuthServiceTest
```

Hard Constraint는 일반 코드로 처리할 수 있다.

```text
internal_network = true
→ Local / Hybrid
```

애매한 Task만 사람이 판단하거나 Agent가 보조한다.

모든 Routing 결정을 다시 LLM에게 물어보는 것이 자동화는 아니다.

---

## 4. Orchestration은 Task와 실행환경을 연결한다

Workflow가 반복되면 다음 흐름을 하나의 Orchestration으로 묶을 수 있다.

```text
Task
 ↓
Classification
 ↓
Environment Selection
 ↓
Runner / Agent Selection
 ↓
Execution
 ↓
Validation
 ↓
Evidence
```

여기서 Orchestrator가 반드시 `상위 Agent`일 필요는 없다.

많은 기능은 일반 프로그램으로 처리할 수 있다.

```text
Task Type Mapping
Environment Lookup
Queue
Retry Counter
Failure Fingerprint Check
Artifact Index
```

LLM은 여전히 판단과 코드 수정이 필요한 구간에만 들어간다.

10장의 Runner-first 원칙을 Orchestration 단계에서도 유지하는 것이다.

---

## 5. Compute와 Reasoning의 Lifecycle도 분리할 수 있다

3장에서 다음 모델을 사용했다.

```text
Brain
→ LLM 판단

Hands
→ Container / Browser / DB / Tool 실행
```

후속 시스템에서는 하나의 판단이 여러 실행환경을 호출할 수도 있다.

```text
Agent
→ backend-test Runner
→ failure 분석
→ frontend-e2e Runner
→ 결과 비교
```

이 경우 다음 두 Lifecycle을 같은 것으로 볼 필요가 없다.

```text
Reasoning Lifecycle
!=
Compute Lifecycle
```

Session Hibernate, One Brain Multiple Hands, Worker Pool 같은 상세 설계는 이 책의 범위를 넘는다.

여기서는 LLM과 Compute를 분리해 설계할 수 있다는 방향만 남긴다.

---

## 6. Evidence는 Query 가능한 인터페이스로 발전할 수 있다

8장에서는 Raw Artifact를 보존하고 작은 Summary부터 읽었다.

후속 단계에서는 Agent가 필요한 결과를 구조화된 방식으로 조회할 수 있다.

```text
get_failed_tests(task)
get_errors(service, since)
get_http_failures(status)
get_browser_console_errors(task)
```

개념은 다음과 같다.

```text
Dashboard for Humans
+
Queryable Evidence for Agents
```

핵심은 새 Observability Platform을 만드는 것이 아니다.

긴 로그를 Agent가 반복해서 읽지 않게 한다는 기존 원칙을 인터페이스로 확장하는 것이다.

---

## 7. Scheduling도 Task 특성에 맞게 발전할 수 있다

Task마다 필요한 실행환경은 다르다.

```text
Unit Test
→ backend-test Runner

Integration Test
→ Docker / DB 가능한 Environment

E2E
→ Browser Environment

Bug Fix
→ Agent + 필요한 Validation
```

후속 Scheduler는 Task Metadata와 Environment Capability를 연결할 수 있다.

```text
Task Requirement
↔
Environment Capability
```

또 9장의 Prepared Environment를 바탕으로 Warm Worker와 Ephemeral Worker를 선택할 수 있다.

```text
짧고 반복적인 검증
→ Warm Worker 후보

격리 요구가 크거나 장시간 작업
→ Ephemeral Worker 후보
```

제품별 CPU/RAM 숫자나 Worker Pool 운영 상세는 현재 책의 원칙으로 고정하지 않는다.

---

## 8. Best-of-N과 Replay는 제한된 고급 기법이다

같은 어려운 Task를 여러 Agent가 독립적으로 풀게 하는 Best-of-N은 일반 병렬화와 다르다.

```text
같은 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
        ↓
Deterministic Validation
```

Context와 LLM 비용도 반복되므로 기본값은 N=1이다.

자동 검증 가능하고 실패 비용이 큰 어려운 Task에서만 제한적으로 검토한다.

Workflow 자체가 바뀌었을 때는 과거 Task를 다시 실행해 비교하는 Replay도 생각할 수 있다.

```text
같은 Task
→ 이전 Harness / Environment
→ 새로운 Harness / Environment
→ 성공률 / Retry / Token / Human Intervention 비교
```

이 책에서는 Best-of-N 시스템이나 평가 플랫폼을 설계하지 않는다. Cloud Workflow를 개선할 때 사용할 수 있는 후속 관점으로만 소개한다.

---

## 9. 자동화는 작은 반복부터 시작한다

처음부터 다음을 한꺼번에 만들 필요는 없다.

```text
Task Router
Scheduler
Multi-Agent Manager
Memory
Observability Platform
자동 Merge
```

먼저 반복되고 검증 가능한 부분을 찾는다.

예:

```text
Cloud Runner로 Test 분리
→ Result Summary 생성
→ CI Failure에서 Agent-on-failure
→ Prepared Environment 정리
→ Task Contract 표준화
→ 반복 Routing 일부 자동화
```

각 단계에서 실제로 Blocking Time, Retry, Review Cost가 줄었는지 확인한다.

복잡한 Platform보다 반복되는 수동 결정을 하나씩 코드로 옮기는 편이 이 책의 방향에 맞다.

---

## 10. 이 책의 범위는 여기까지다

Cloud Agent를 다루다 보면 다음 주제로 쉽게 확장된다.

```text
Agent Memory Architecture
Agent Security Platform
Agent OS
Agent Chaos Engineering
Shadow / Canary Agent
Agent Governance Platform
General Multi-Agent Theory
Agent 조직론
```

이 주제는 후속 연구로 남긴다.

현재 책의 질문은 끝까지 하나다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

이 질문에 직접 필요하지 않은 Platform 일반론은 여기서 확장하지 않는다.

---

## 11. 최종 Workflow와 마지막 판단

책 전체의 실행 흐름을 마지막으로 한 번만 정리한다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture
        ↓
Task Split / Routing
        ↓
Task Contract / Git Handoff
        ↓
Prepared Cloud Environment
        ↓
Runner
   ├─ PASS → Evidence
   └─ FAIL
        ↓
   Reasoning Required?
      ├─ NO → Tool / Retry / Escalation
      └─ YES
            ↓
       Cloud Agent
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

독자가 마지막에 답할 수 있어야 하는 질문은 다음과 같다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
Cloud 이점이 사라지면 언제 Local로 돌아올 것인가?
```

그리고 실제 개발에서는 다음 문장이 자연스럽게 나와야 한다.

```text
이 설계는 Local에서 하자.
이 검증은 Cloud Runner로 보내자.
이 재현 가능한 실패는 Cloud Agent에게 맡기자.
이 마지막 검증은 내부망에서 하자.
```

> 더 많은 Agent보다 더 나은 Task Routing, Harness, Validation이 먼저다.

Cloud Agent는 Local Agent를 없애지 않는다. Runner나 CI도 없애지 않는다.

각 역할을 분리하고, Task 특성에 맞는 실행 위치에 배치한다.

이 책의 최종 목적은 독자가 자신의 개발 흐름에서 **Local과 Cloud의 역할을 직접 나눌 수 있게 하는 것**이다.