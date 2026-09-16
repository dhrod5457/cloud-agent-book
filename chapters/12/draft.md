# 12장. 병렬 Worker와 중복 Context 비용

Cloud Agent를 여러 개 동시에 실행하면 많은 일을 한꺼번에 처리할 수 있다.

하지만 Agent 수를 늘린다고 개발 속도가 같은 비율로 증가하지는 않는다.

병렬화에는 항상 반대쪽 비용이 있다.

```text
Startup Overhead
Context Duplication
Merge Conflict
Review Cost
Result Integration
Coordination Cost
```

따라서 병렬화의 핵심 질문은 `몇 개의 Agent를 띄울 것인가`가 아니다.

> 서로 독립적으로 실행하고 검증할 수 있는 Task가 몇 개인가?

이 장에서는 Cloud Agent의 병렬성을 Agent 수가 아니라 Task 구조 관점에서 본다.

---

## 1. 병렬화의 첫 번째 대상은 Compute다

하나의 Commit에 대해 다음 검증이 필요하다고 하자.

```text
Unit Test
Integration Test
E2E
Docker Build
```

서로 결과를 기다릴 필요가 없다면 동시에 실행할 수 있다.

```text
Git SHA
  ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit    Integration E2E      Docker
|         |         |         |
+---------+---------+---------+---------+
          ↓
        Fan-in
```

여기서 중요한 점이 있다.

네 개의 LLM이 필요한 것은 아니다.

대부분의 작업은 Cloud Runner 네 개면 충분하다.

```text
Parallel Compute
→ 여러 Runner

Parallel Reasoning
→ 여러 Agent
```

두 구조를 구분해야 한다.

Cloud 병렬화의 첫 단계는 `Agent 여러 개`가 아니라 `독립 실행 가능한 Compute 작업을 분리하는 것`이다.

---

## 2. Fan-out 전에 Dependency를 확인한다

Task가 여러 개 있다고 모두 동시에 실행하지 않는다.

예를 들어 다음 세 작업이 있다고 하자.

```text
Task A
common DTO 변경

Task B
API 변경

Task C
UI 변경
```

의존 관계가 다음이라면:

```text
A → B → C
```

세 Agent를 동시에 실행하는 것은 의미가 적다.

B는 A의 결과를 필요로 하고 C는 B의 결과를 필요로 한다.

이 경우 다음처럼 순서를 만든다.

```text
Task A
  ↓
Task B
  ↓
Task C
```

반대로 다음은 병렬화하기 쉽다.

```text
Task A → attendance module
Task B → notification module
Task C → library module
```

조건은 다음과 같다.

```text
다른 Module
다른 File Scope
독립 Validation
낮은 Merge Dependency
```

병렬화 전에 Dependency Graph를 먼저 본다.

---

## 3. Context Duplication은 숨은 비용이다

Agent를 여러 개 띄우면 각 Agent가 자신만의 Context를 필요로 한다.

다음 구조를 보자.

```text
Agent A
→ Repository 전체 분석

Agent B
→ Repository 전체 분석

Agent C
→ Repository 전체 분석

Agent D
→ Repository 전체 분석
```

네 Agent가 모두 같은 README, Architecture 문서, 공통 Source, Dependency 구조를 다시 읽는다.

이 과정에서 Token과 시간이 중복된다.

좋지 않은 병렬화:

```text
4 Agent
×
Repository 전체 Context
```

권장 구조:

```text
Agent A
→ attendance 관련 Context

Agent B
→ notification 관련 Context

Agent C
→ admin UI 관련 Context
```

7장의 Task Contract가 여기서 중요해진다.

병렬 Task마다 작은 Relevant Context를 주면 병렬화에 따른 중복 비용도 줄일 수 있다.

---

## 4. Agent Count는 Task Count와 다르다

Task가 10개 있다고 Agent 10개를 바로 띄우는 것이 최적은 아니다.

다음 요소를 같이 봐야 한다.

```text
Task Dependency
Cloud Start Overhead
Token Budget
Compute Quota
Review Capacity
Merge Conflict
Result Integration
```

예를 들어 10개 Task 중 실제로 독립적인 것이 세 개뿐일 수 있다.

```text
Task Queue: 10
   ↓
Dependency 분석
   ↓
Parallel Group A: 3
   ↓
Fan-in
   ↓
Parallel Group B: 3
   ↓
Fan-in
```

이 방식이 10개를 한 번에 실행하는 것보다 전체 Flow를 안정적으로 유지할 수 있다.

병렬도는 Agent가 만들 수 있는 수가 아니라 팀이 **통합하고 검토할 수 있는 수**까지 고려해서 정한다.

---

## 5. Read-only 검증은 병렬화하기 쉽다

가장 먼저 병렬화하기 좋은 작업은 Source를 수정하지 않는 검증이다.

예:

```text
Unit Test
Integration Test
E2E
Docker Build
Static Analysis
Documentation Validation
```

같은 Git SHA를 읽고 서로 다른 검증을 수행한다.

```text
abc123
 ├─ unit
 ├─ integration
 ├─ e2e
 └─ docker
```

Merge Conflict가 없고 결과만 Fan-in하면 된다.

따라서 Cloud 병렬화의 시작점은 여러 Agent가 코드를 동시에 수정하는 것이 아니라 **검증 작업을 여러 Runner로 분산하는 것**이 될 수 있다.

이 방식은 구현 복잡도도 낮고 병렬화 이점도 확인하기 쉽다.

---

## 6. 코드 변경 병렬화에서는 Change Locality를 본다

코드를 동시에 수정하려면 `어디를 바꾸는가`가 중요하다.

좋은 예:

```text
Agent A
→ attendance module

Agent B
→ notification module

Agent C
→ library module
```

각 모듈이 독립적으로 Build/Test되고 공통 파일 수정이 적다면 병렬로 진행하기 좋다.

나쁜 예:

```text
Agent A
→ common-auth

Agent B
→ common-auth

Agent C
→ common-auth
```

세 Agent가 같은 핵심 영역을 수정하면 실행 중에는 독립적이어도 통합 시 비용이 커진다.

```text
Parallel Execution
→ 빨라짐

Fan-in
→ Conflict / Review / Rework 증가
```

따라서 병렬 코딩에서는 Agent 수보다 Change Locality를 먼저 본다.

---

## 7. Fan-in이 실제 병목이 될 수 있다

병렬화는 Fan-out만으로 끝나지 않는다.

결과를 다시 하나로 합쳐야 한다.

```text
PR A
PR B
PR C
PR D
  ↓
Fan-in
  ↓
Integration Validation
```

Fan-in에서 다음 비용이 생긴다.

```text
PR Review
Conflict Resolution
Full Regression
Architecture Consistency 확인
Migration Ordering
Release Note 정리
```

Cloud Agent가 네 개의 PR을 30분 안에 만들 수 있어도 사람이 네 개를 검토하는 데 반나절이 걸릴 수 있다.

이 경우 Fan-out 속도는 전체 Lead Time을 설명하지 못한다.

> 병렬화를 설계할 때 Fan-out만큼 Fan-in 비용도 본다.

---

## 8. Review Capacity는 병렬도의 상한이 된다

Agent가 많은 코드를 만들수록 Reviewer의 일이 늘어난다.

예를 들어 Agent가 하루 30개의 PR을 만들 수 있다고 하자.

그러나 팀이 하루 5개만 제대로 검토할 수 있다면 나머지는 Queue에 쌓인다.

```text
Agent Throughput
30 PR/day

Review Capacity
5 PR/day
```

병목은 Agent가 아니다.

Review다.

측정할 수 있는 값:

```text
PR 생성 수
Review Queue Time
Review Duration
Merge Conflict Rate
Rework Rate
Merge Success Rate
```

Cloud Agent 생산성을 `PR 수`만으로 평가하지 않는 이유다.

실제로 중요한 것은 Task가 얼마나 빨리 통합 가능한 상태까지 도달하는가다.

---

## 9. 중복 Environment 비용도 생긴다

병렬 Worker마다 환경을 처음부터 준비하면 다음 비용이 반복된다.

```text
Repository Checkout
Dependency Restore
Docker Image Pull
Playwright Browser 준비
Build Cache 생성
```

9장의 Prepared Environment와 Cache를 재사용하면 이 비용을 줄일 수 있다.

권장 구조:

```text
Shared Reusable Cache
        +
Fresh Runtime per Worker
```

예:

```text
공유 가능
- Gradle dependency cache
- Docker layer cache
- Browser binary

공유 금지
- mutable DB state
- temp output
- test user data
- process state
```

병렬 Worker가 빠르게 시작하되 서로의 Runtime State는 오염시키지 않게 한다.

---

## 10. 병렬화 이득은 단순히 Worker 수로 계산하지 않는다

개념적으로 병렬화의 이득은 다음처럼 생각할 수 있다.

```text
Parallel Benefit
≈ Saved Execution Time
- Startup Overhead
- Duplicate Context Cost
- Merge Cost
- Review Cost
- Coordination Cost
```

정확한 수식으로 계산할 필요는 없다.

중요한 것은 어떤 비용을 측정해야 하는지 아는 것이다.

예를 들어 Worker 수를 2개에서 8개로 늘렸는데:

```text
Execution Time은 감소
Review Queue는 증가
Merge Conflict도 증가
```

했다면 전체 Lead Time이 줄지 않을 수 있다.

병렬도는 실제 결과를 보고 조절한다.

---

## 11. Dependency-aware Parallel Group

Task에 Dependency를 명시하면 병렬 그룹을 만들 수 있다.

예:

```yaml
task: common-dto
parallel_group: 1

---

task: attendance-api
depends_on:
  - common-dto
parallel_group: 2

---

task: admin-ui
depends_on:
  - attendance-api
parallel_group: 3
```

실행:

```text
Group 1
common-dto
   ↓
Group 2
attendance-api
   ↓
Group 3
admin-ui
```

반면 서로 의존성이 없는 Task는 같은 Group에 둘 수 있다.

```text
Group 2
├─ attendance-test
├─ notification-test
└─ library-test
```

이 장의 목적은 범용 Scheduler를 구현하는 것이 아니다.

Task Dependency를 병렬도 판단에 포함한다는 원칙을 설명하는 것이다.

---

## 12. Java 17 → 21 Migration 예제

여러 서비스를 Java 21로 올린다고 하자.

```text
service-a
service-b
service-c
```

각 서비스가 독립 Build를 가진다면 병렬로 진행할 수 있다.

```text
Migration Controller
      ↓
+-----+-----+-----+
|           |     |
service-a service-b service-c
|           |     |
Agent/Runner 각각 실행
|           |     |
PR A      PR B   PR C
 \          |     /
       Fan-in
         ↓
Full Validation
```

하지만 모든 서비스가 같은 Gradle Plugin이나 공통 Library를 사용한다고 하자.

```text
common-build-plugin
```

그 Plugin 수정이 선행돼야 한다면 먼저 처리한다.

```text
Task 0
common-build-plugin Java 21 대응
   ↓
새 Base
   ↓
service-a / b / c 병렬 Migration
```

Dependency를 무시하고 세 Agent를 동시에 시작하면 각 Agent가 같은 공통 문제를 반복해서 해결할 수 있다.

---

## 13. Best-of-N은 일반 병렬화와 다르다

일반 Fan-out은 서로 다른 Task를 여러 Worker가 수행한다.

```text
Task A → Worker A
Task B → Worker B
Task C → Worker C
```

Best-of-N은 같은 어려운 Task를 여러 Agent가 각각 해결한다.

```text
같은 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
```

그다음 후보를 검증한다.

```text
Patch A → Test
Patch B → Test
Patch C → Test
```

Best-of-N은 비용이 훨씬 크다.

같은 Context와 같은 문제를 N번 반복해서 읽고 추론하기 때문이다.

따라서 기본값은 N=1로 둔다.

다음과 같은 상황에서만 검토할 수 있다.

```text
어려운 Bug
검증 가치가 큼
자동 검증 기준 존재
실패 비용이 큼
```

일반적인 개발 Task의 기본 병렬화 전략으로 사용하지 않는다.

---

## 14. Best-of-N 후보 선택도 가능한 한 검증으로 한다

세 Patch가 만들어졌다고 하자.

좋지 않은 방식:

```text
LLM에게 세 Patch 중 가장 좋아 보이는 것을 골라달라고 한다.
```

먼저 결정론적 검증을 사용한다.

```text
Unit Test
Integration Test
Regression
Static Analysis
Security Check
```

그 뒤에도 여러 후보가 남으면 다음을 비교할 수 있다.

```text
Diff Size
Changed Files
Execution Time
Complexity
```

LLM의 선호가 아니라 **검증 가능한 결과**를 우선한다.

Best-of-N 역시 8장의 Evidence 원칙을 따른다.

---

## 15. campus-platform 병렬화 예제

좋은 검증 병렬화:

```text
Cloud #1
→ attendance unit test

Cloud #2
→ notification integration test

Cloud #3
→ admin E2E

Cloud #4
→ Docker build
```

좋은 코드 병렬화:

```text
Agent A
→ attendance expired-status bug

Agent B
→ notification retry bug
```

두 작업이 서로 다른 Module과 테스트를 가진다고 가정한다.

나쁜 병렬화:

```text
Agent A
Agent B
Agent C
  ↓
모두 common-auth 수정
```

또 다른 나쁜 예:

```text
Agent A
→ migration V142

Agent B
→ migration V143

하지만 V143은 V142 Schema를 전제로 함
```

겉으로는 파일이 달라도 실제 Dependency가 있다.

병렬화는 File Conflict뿐 아니라 Semantic Dependency도 봐야 한다.

---

## 16. 병렬화 체크리스트

Task를 Fan-out하기 전에 다음을 확인한다.

```text
서로 독립적인가?
같은 Base SHA에서 시작하는가?
같은 핵심 파일을 수정하지 않는가?
공통 DTO/Schema/Migration 의존이 없는가?
각 Task를 독립적으로 검증할 수 있는가?
Task별 Context를 작게 제한할 수 있는가?
Worker 시작 비용보다 실행시간이 충분히 큰가?
Reviewer가 결과를 처리할 수 있는가?
Fan-in 후 통합 검증 방법이 있는가?
```

이 질문 중 여러 개가 불명확하면 Agent 수를 늘리기 전에 Task 구조를 다시 본다.

---

## 17. 무엇을 측정할 것인가

병렬화를 최적화하려면 Agent 수만 기록하지 않는다.

다음 값을 함께 본다.

```text
Task Lead Time
Worker Startup Time
Agent Execution Time
Duplicate Context Usage
Review Queue Time
Review Duration
Merge Conflict Rate
Rework Rate
Integration Failure Rate
```

예를 들어:

```text
Worker 2개
→ Lead Time 4시간

Worker 8개
→ Coding 1시간
→ Review/Conflict 4시간
→ Lead Time 5시간
```

이라면 Worker 수를 늘린 것이 전체 Flow를 악화시킨 것이다.

Cloud Agent의 병렬성은 `얼마나 많이 동시에 실행했는가`보다 `전체 개발시간이 실제로 줄었는가`로 평가한다.

---

## 18. 이 장에서 기억할 것

Cloud Agent 환경에서는 Worker를 추가하기 쉽다.

그러나 병렬화의 가치가 생기려면 Task가 먼저 독립적이어야 한다.

```text
Task Dependency 확인
   ↓
Source / Runtime 격리
   ↓
작은 Context 제공
   ↓
Fan-out
   ↓
Evidence 수집
   ↓
Fan-in
   ↓
Integration Validation
```

병렬화의 이득은 실행시간 단축에서 나오지만 비용은 Context, Merge, Review, Integration에서 발생한다.

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

> Agent 수를 늘린다고 생산성이 선형 증가하지 않는다.

> Fan-out만큼 Fan-in 비용도 설계해야 한다.

다음 장에서는 이 병렬 결과를 다시 Local 개발 흐름으로 가져와 `Local → Cloud → Local` Handoff를 하나의 실제 Workflow로 연결한다.
