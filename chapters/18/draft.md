# 18장. 다음 단계: Harness와 Orchestration

17장까지 이 책은 Cloud Agent를 실제 개발 Workflow에 넣는 방법을 다뤘다.

핵심 구조는 이미 완성되어 있다.

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

이 구조가 잘 동작하면 다음 질문이 생긴다.

```text
이 반복 결정을 어디까지 자동화할 수 있을까?
```

이 장은 새로운 Agent Platform을 설계하는 장이 아니다.

앞 장까지 만든 Workflow를 기준으로, 반복되는 준비·Routing·검증·결과 회수 과정을 조금 더 자동화하면 어떤 주제가 자연스럽게 나타나는지만 살펴본다.

> 먼저 Cloud Agent를 잘 사용하는 Workflow를 만들고, 그 다음에 Orchestration을 자동화한다.

그리고 자동화의 방향도 Agent 수를 늘리는 쪽보다 다음에 가깝다.

```text
Task Routing
Environment 선택
Runner 선택
Validation
Evidence 수집
Retry / Stop 조건
```

---

## 1. 지금까지 만든 구조가 출발점이다

이 책에서 Cloud Agent는 Remote Development Worker다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

이 Worker를 잘 사용하려면 앞에서 다음 조건을 만들었다.

```text
작은 Task
작은 Context
Prepared Environment
Deterministic Runner
Result Gateway
Git Handoff
Evidence
```

이 조건이 안정화되지 않은 상태에서 Multi-Agent Orchestrator부터 만들면 자동화가 복잡해질 가능성이 크다.

예를 들어 Build 명령도 매번 Agent가 찾아야 하고, Test 결과도 긴 로그를 직접 읽어야 하고, Environment도 Task마다 새로 설치한다면 Orchestrator는 결국 Agent에게 다음을 계속 시키게 된다.

```text
setup 방법 추론
build 방법 추론
test 방법 추론
failure 위치 탐색
결과 형식 추론
```

자동화가 늘수록 불확실성도 같이 늘어난다.

반대로 실행 규칙이 정리되어 있다면 자동화할 대상이 명확해진다.

```text
Task Type
→ Environment
→ Runner
→ PASS/FAIL
→ Agent 호출 여부
→ Evidence
```

Orchestration은 이 결정의 반복을 줄이는 다음 단계다.

---

## 2. Harness는 Agent가 반복해서 추론할 일을 줄인다

Agent가 Repository에 들어올 때마다 다음을 묻는다면 비용이 계속 발생한다.

```text
어떻게 setup하지?
어떻게 build하지?
어떻게 test하지?
어떤 파일부터 읽지?
무엇이 PASS지?
실패 로그는 어디 있지?
```

이 질문에 대한 답을 Prompt에 매번 길게 적는 대신 Repository와 실행환경이 제공할 수 있다.

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

이 책에서는 이런 구조를 Harness 관점으로 본다.

Harness는 거대한 프레임워크를 의미하지 않는다.

Agent가 반복해서 탐색하고 추론해야 했던 개발환경의 규칙을 실행 가능한 형태로 꺼내놓는 것이다.

예를 들어 다음 명령이 있다고 하자.

```bash
./scripts/verify-auth.sh
```

이 명령이 다음을 수행한다.

```text
Auth compile
Auth unit test
Auth integration test
Result Summary 생성
```

Agent는 세부 명령을 다시 조합할 필요가 없다.

```text
Task Contract
→ ./scripts/verify-auth.sh
→ result.json
```

Prompt보다 실행환경이 더 많은 정보를 제공한다.

> Agent가 반복해서 실패하면 Prompt보다 Harness와 실행환경을 먼저 점검한다.

---

## 3. Harness는 Repository의 사용 설명서와 실행 인터페이스를 함께 제공한다

Agent에게 도움이 되는 Repository는 단순히 README가 많은 Repository가 아니다.

어디서 시작하고 무엇을 실행해야 하는지 쉽게 찾을 수 있어야 한다.

예:

```text
AGENTS.md
├─ backend task → docs/backend.md
├─ auth task → docs/security/auth.md
├─ test → scripts/test
├─ verify → scripts/verify
└─ artifacts → artifacts/<task-id>/
```

이 구조에서 문서는 탐색 경로를 제공하고 Script는 실행 경로를 제공한다.

```text
Discoverability
+
Executability
+
Verifiability
```

7장의 Progressive Context와 10장의 Runner-first가 Harness에서 만난다.

좋지 않은 구조:

```text
Agent
→ Repository 전체 grep
→ README 여러 개 확인
→ build.gradle 분석
→ 테스트 명령 추론
```

권장 구조:

```text
Task Type
→ 관련 문서
→ 표준 명령
→ 표준 Evidence
```

Harness의 목적은 Agent의 자유도를 없애는 것이 아니다.

필요하지 않은 탐색을 줄이는 것이다.

---

## 4. Task Routing도 반복되면 자동화 후보가 된다

5장에서 Developer가 Task를 다음 기준으로 분류했다.

```text
Runner로 끝나는가?
Cloud에서 독립 실행 가능한가?
Human Steering이 필요한가?
Internal Network가 필요한가?
```

처음에는 사람이 판단하는 편이 낫다.

하지만 같은 패턴이 반복되면 일부 Routing은 규칙으로 만들 수 있다.

예:

```text
Task Type: unit-test
→ Cloud Runner

Task Type: e2e
→ frontend-e2e Runner

Task Type: reproducible-bug-fix
→ Cloud Agent 후보

Requires: HSM
→ Local
```

Task Metadata가 다음처럼 들어온다고 하자.

```yaml
type: bug-fix
scope: auth
reproducible: true
internal_network: false
validation: ./gradlew test --tests AuthServiceTest
```

Routing Rule은 다음처럼 단순해질 수 있다.

```text
reproducible=true
+ validation exists
+ internal_network=false
→ cloud-agent candidate
```

이 단계에서 중요한 것은 모든 결정을 LLM에게 맡기는 것이 아니다.

Hard Constraint는 코드로 처리할 수 있다.

```text
internal_network=true
→ local/hybrid
```

그리고 애매한 Task만 사람이 보거나 Agent가 보조한다.

---

## 5. Orchestration은 Task와 실행환경을 연결한다

현재까지의 Workflow에서 Developer는 여러 결정을 직접 했다.

```text
Task를 어디로 보낼지
어떤 Environment를 사용할지
Runner인지 Agent인지
어떤 Validation을 실행할지
```

이 결정이 반복되면 하나의 Orchestration 흐름으로 묶을 수 있다.

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

여기서 Orchestrator의 역할은 `Agent에게 모든 일을 시키는 상위 Agent`로 한정되지 않는다.

오히려 많은 부분은 일반 프로그램이 맡을 수 있다.

예:

```text
Task Type Mapping
Environment Lookup
Queue
Retry Counter
Failure Fingerprint Check
Artifact Index
```

LLM이 필요한 부분만 남긴다.

이 구조는 10장의 Runner-first 원칙과 같다.

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

---

## 6. Brain과 Hands를 다시 보면

3장에서 Compute와 Token을 구분하기 위해 다음 단순 모델을 사용했다.

```text
Brain
→ LLM 판단

Hands
→ Container / Browser / DB / Tool 실행
```

Cloud Agent Workflow가 발전하면 이 구분이 더 중요해질 수 있다.

하나의 판단이 여러 실행환경을 사용할 수 있기 때문이다.

예:

```text
Agent
→ backend-test Runner 호출
→ failure 확인
→ frontend-e2e Runner 호출
→ 결과 비교
```

이 구조에서 LLM Session이 실행환경 하나와 영구적으로 묶일 필요는 없다.

```text
Reasoning Lifecycle
!=
Compute Lifecycle
```

하지만 이 책에서는 여기까지만 다룬다.

Session Hibernate, One Brain Multiple Hands, Worker lifecycle 같은 상세 구조는 후속 연구 주제다.

현재 독자에게 필요한 것은 다음 구분이다.

```text
LLM이 판단하는 시간
Compute가 실행하는 시간
```

둘을 분리하면 비용과 병렬성을 다르게 설계할 수 있다는 점이다.

---

## 7. Agent-native Observability는 Evidence의 확장이다

8장에서 Result Gateway를 만들었다.

Agent가 처음 읽는 것은 작은 Summary이고 필요할 때 Raw Artifact를 조회한다.

```text
Summary
→ Failure Detail
→ Stack Trace
→ Specific Log
→ Raw Artifact
```

이 구조를 더 발전시키면 Agent가 실행상태를 구조화된 방식으로 직접 조회할 수 있다.

예:

```text
get_failed_tests(task)
get_errors(service, since)
get_http_failures(status)
get_browser_console_errors(task)
```

사람은 Dashboard를 볼 수 있다.

Agent는 Query 가능한 Evidence를 사용할 수 있다.

```text
Dashboard for Humans
+
Queryable Evidence for Agents
```

중요한 것은 새 Observability Platform을 만드는 것 자체가 아니다.

현재의 긴 로그를 Agent가 반복해서 읽지 않도록 한다는 8장의 원칙이 그대로 확장된 것이다.

이 책에서는 Query 가능한 Evidence라는 방향만 소개한다.

---

## 8. Compute-aware Scheduling

Task마다 필요한 Compute가 다르다.

```text
Unit Test
→ 작은 CPU/RAM

Integration Test
→ Docker / 더 많은 RAM

E2E
→ Browser Runtime

Large Build
→ CPU 중심
```

후속 Scheduler는 Agent만 고르는 것이 아니라 적절한 실행환경도 고를 수 있다.

예:

```text
Task: unit-test
→ environment: backend-test
→ runner: unit

Task: admin-e2e
→ environment: frontend-e2e
→ runner: e2e

Task: reproducible auth bug
→ environment: backend-test
→ cloud-agent
```

제품별 CPU/RAM 숫자를 고정할 필요는 없다.

중요한 것은 Task Metadata와 Environment Capability를 연결하는 것이다.

```text
Task Requirement
↔
Environment Capability
```

9장의 Task-specific Environment가 자동 선택으로 발전하는 셈이다.

---

## 9. Warm Worker와 Ephemeral Worker의 선택도 자동화될 수 있다

9장에서 Warm Worker와 Ephemeral Worker를 구분했다.

짧고 반복적인 Task:

```text
lint
small unit test
PR verification
```

는 READY Worker를 재사용하면 시작시간을 줄일 수 있다.

```text
READY
→ Task
→ Reset
→ READY
```

반대로 다음 작업은 새 환경이 더 단순할 수 있다.

```text
대형 Integration
Full E2E
독립 Feature 작업
격리가 중요한 변경
```

```text
Task
→ Ephemeral Worker
→ Evidence
→ Destroy
```

후속 Orchestration은 Task 특성에 따라 둘 중 하나를 선택할 수 있다.

하지만 선택 기준은 여전히 같다.

```text
Startup Cost
Isolation Requirement
Execution Duration
Reuse Value
```

Warm Worker Pool 자체의 운영 상세는 이 책의 범위를 넘는다.

---

## 10. Best-of-N은 기본 전략이 아니다

12장에서 Best-of-N을 일반 Fan-out과 구분했다.

일반 Fan-out:

```text
서로 다른 Task
→ 서로 다른 Worker
```

Best-of-N:

```text
같은 Task
→ 여러 Agent
→ 여러 Patch
```

예:

```text
어려운 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
        ↓
Deterministic Validation
        ↓
Candidate 선택
```

Cloud 환경에서는 기술적으로 실행하기 쉽다.

그러나 Context와 LLM 비용도 N배 가까이 늘 수 있다.

따라서 기본값은 N=1이다.

Best-of-N을 검토할 조건:

```text
Task 난도가 높음
실패 비용이 큼
후보 자동 검증 가능
독립 Branch 가능
Review 가치가 비용보다 큼
```

작은 Bug나 문서 수정에는 사용하지 않는다.

여기서도 핵심은 Agent 수가 아니라 검증 구조다.

---

## 11. Candidate 선택도 Agent 선호보다 Validation을 우선한다

여러 Patch가 만들어졌다고 하자.

좋지 않은 구조:

```text
LLM에게 Patch A/B/C 중 가장 좋아 보이는 것을 고르라고 한다.
```

먼저 다음을 실행한다.

```text
Target Test
Regression Test
Static Analysis
Security Check
Diff Scope
```

예:

```text
Patch A
Target PASS
Regression FAIL

Patch B
Target PASS
Regression PASS

Patch C
Target FAIL
```

이 경우 후보를 상당 부분 프로그램으로 줄일 수 있다.

그 다음 사람이 Review한다.

```text
Validation
→ Candidate 좁힘
→ Human Review
```

Best-of-N도 Runner-first 원칙을 벗어나지 않는다.

---

## 12. Replay는 Workflow 자체를 평가하는 방법이 될 수 있다

Cloud Agent와 Harness도 시간이 지나면 바뀐다.

```text
모델 변경
Prompt 변경
Task Contract 변경
Environment 변경
Runner 변경
```

이때 같은 과거 Task를 다시 실행해 비교할 수 있다.

예:

```text
Task AUTH-142
```

비교 항목:

```text
성공 여부
Agent Invocation Count
Token Usage
Execution Time
Retry Count
Human Intervention
Changed Files
Regression
```

이를 Replay 또는 Benchmark 관점으로 볼 수 있다.

예를 들어 새 Harness 도입 후 다음처럼 비교할 수 있다.

```text
Before
Repository 탐색 8회
Agent Retry 3회

After
Relevant Context 직접 제공
Agent Retry 1회
```

이 책에서는 평가 플랫폼을 설계하지 않는다.

다만 Cloud Agent Workflow도 코드처럼 변경 전후를 비교할 수 있다는 관점을 남긴다.

---

## 13. 자동화의 순서는 작은 반복부터다

좋지 않은 접근:

```text
첫날부터
Task Router
Scheduler
Multi-Agent Manager
Memory
Observability Platform
자동 Merge
```

먼저 반복되고 안정적인 부분을 찾는다.

예:

```text
1. Unit Test를 Cloud Runner로 분리
2. Result Summary 생성
3. CI Failure에서 Agent-on-failure
4. Prepared Environment 구성
5. Task Contract 표준화
6. Routing Rule 일부 자동화
```

이 순서에서는 각 단계의 효과를 측정할 수 있다.

문제가 생기면 어느 계층에서 발생했는지도 찾기 쉽다.

자동화는 복잡한 시스템을 한 번에 만드는 작업이 아니라 반복되는 수동 결정을 하나씩 코드로 옮기는 작업에 가깝다.

---

## 14. Cloud Agent 중심 개발환경의 발전 단계를 보면

이 책의 흐름을 발전 단계로 정리하면 다음과 같다.

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

이 책의 목표는 6단계까지 실제 프로젝트에 적용할 수 있게 만드는 것이다.

7단계부터는 팀과 프로젝트의 반복 패턴에 따라 선택적으로 진행한다.

모든 팀이 Agent Platform을 만들어야 하는 것은 아니다.

Runner와 Git, CI, Script만으로 충분한 팀도 있다.

자동화 수준은 문제 크기에 맞춘다.

---

## 15. 지금 하지 않을 것

Cloud Agent를 다루다 보면 다음 주제로 쉽게 확장된다.

```text
Agent Memory Architecture
Agent Security Platform
Agent OS
Agent Chaos Engineering
Garbage Collector Agent
Shadow Agent
Canary Agent
Agent Governance Platform
General Multi-Agent Theory
Agent 조직론
```

이 주제들은 흥미롭지만 이 책의 핵심 질문과는 거리가 있다.

이 책의 질문은 끝까지 다음이다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

따라서 위 주제는 후속 연구로 남긴다.

현재 책에서는 Cloud Agent Workflow를 이해하는 데 직접 필요한 범위만 사용한다.

---

## 16. 마지막으로 전체 Workflow를 다시 보자

책 전체에서 만든 흐름은 다음과 같다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture
        ↓
Task Split
        ↓
Local or Cloud Routing
        ↓
Task Contract
        ↓
Git Handoff
        ↓
Prepared Cloud Environment
        ↓
Runner
   ├─ PASS → Evidence
   └─ FAIL
        ↓
   Result Gateway
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
Local Review
        ↓
Internal Validation
        ↓
Merge
```

병렬화할 수 있다면 독립 Task만 Fan-out한다.

```text
Unit
Integration
E2E
Docker
```

같은 파일이나 Schema를 공유한다면 순차화하거나 재분해한다.

Cloud에서 재현하지 못하거나 내부망이 필요하면 Local로 돌아온다.

```text
Cloud
→ Evidence
→ Local Fallback
```

이 Workflow가 먼저다.

Orchestration은 이 반복을 자동화하는 다음 단계다.

---

## 17. 독자가 마지막에 가져갈 판단 기준

책의 마지막에서 기억해야 할 것은 특정 Cloud Agent 제품의 사용법이 아니다.

다음 질문에 답할 수 있어야 한다.

```text
이 Task는 Runner로 끝낼 수 있는가?

Agent 판단이 필요한가?

Cloud에 독립 실행 가능한가?

작은 Context로 만들 수 있는가?

Prepared Environment가 있는가?

Evidence로 완료를 확인할 수 있는가?

내부망 검증은 어디서 할 것인가?

병렬화했을 때 Fan-in 비용은 얼마인가?

Cloud가 불리해지면 언제 Local로 돌아올 것인가?
```

그리고 최종적으로 다음 문장을 직접 말할 수 있어야 한다.

```text
이 작업은 Local에서 하자.
이 검증은 Cloud Runner로 보내자.
이 실패는 Cloud Agent에게 맡기자.
이 마지막 검증은 내부망에서 하자.
```

이 선택이 자연스러워지면 Cloud Agent는 별도의 신기한 도구가 아니라 개발 Workflow의 하나의 실행 위치가 된다.

---

## 장을 마치며

Cloud Agent 활용이 안정화된 뒤에는 Harness와 Orchestration이 자연스럽게 다음 주제로 나타난다.

하지만 순서는 바꾸지 않는다.

```text
Cloud Agent Workflow
→ 반복 패턴 확인
→ Harness
→ Routing 자동화
→ Orchestration
```

처음부터 더 많은 Agent를 관리하는 시스템을 만드는 것이 목적이 아니다.

> 더 많은 Agent보다 더 나은 Task Routing, Harness, Validation이 먼저다.

Cloud Agent는 Local Agent를 없애지 않는다.

Runner도 없애지 않는다.

CI도 없애지 않는다.

각각의 역할을 분리하고 필요한 위치에 배치한다.

이 책의 최종 목적은 하나다.

> 독자가 자신의 개발 흐름에서 Local과 Cloud의 역할을 직접 나눌 수 있게 하는 것.
