# 12장. 병렬 Worker와 중복 Context 비용

Cloud Worker를 여러 개 띄우면 여러 작업을 동시에 처리할 수 있다.

하지만 Agent 수를 늘린다고 개발 속도가 같은 비율로 증가하지는 않는다.

병렬화에는 반대쪽 비용이 있다.

```text
Startup Overhead
Context Duplication
Merge Cost
Review Cost
Result Integration Cost
Coordination Cost
```

따라서 병렬화의 핵심 질문은 `몇 개의 Agent를 띄울 것인가`가 아니다.

> 서로 독립적으로 실행하고 검증할 수 있는 Task가 몇 개인가?

11장이 병렬화를 위한 격리 조건을 만들었다면, 12장은 **어디까지 병렬화하는 것이 실제로 이득인가**를 판단한다.

---

## 1. 먼저 Parallel Compute와 Parallel Reasoning을 구분한다

하나의 Git SHA에서 다음 검증이 필요하다고 하자.

```text
Unit Test
Integration Test
E2E
Docker Build
```

이 작업은 대부분 Cloud Runner로 병렬화할 수 있다.

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

이것이 `Parallel Compute`다.

반면 다음은 다르다.

```text
Agent A → Repository 분석
Agent B → Repository 분석
Agent C → Repository 분석
```

같은 문제에 여러 LLM이 각각 Context를 읽고 판단한다. 이것은 `Parallel Reasoning`이다.

기본값은 먼저 Compute를 병렬화하고, 판단이 실제로 필요한 Task에만 Agent를 붙이는 것이다.

---

## 2. Fan-out 전에 Dependency를 확인한다

Task가 여러 개 있다고 모두 동시에 시작하지 않는다.

```text
Task A → common DTO 변경
Task B → API 변경
Task C → UI 변경
```

의존 관계가 다음이라면:

```text
A → B → C
```

세 Worker를 동시에 시작해도 B와 C는 다시 작업해야 할 가능성이 높다.

반대로 다음처럼 독립성이 높으면 Fan-out하기 쉽다.

```text
Task A → attendance module
Task B → notification module
Task C → library module
```

판단 기준:

```text
File Scope 분리
낮은 Dependency
독립 Validation
낮은 Merge 순서 의존성
```

병렬화는 Task Dependency를 먼저 본 뒤 시작한다.

---

## 3. 가장 먼저 병렬화하기 좋은 것은 Read-only 검증이다

Source를 수정하지 않는 검증은 Merge Conflict가 없다.

```text
Unit Test
Integration Test
E2E
Docker Build
Static Analysis
```

같은 SHA를 읽고 서로 다른 결과를 만든다.

```text
abc123
├─ Unit
├─ Integration
├─ E2E
└─ Docker
```

결과는 마지막에 Fan-in하면 된다.

여러 Agent가 동시에 코드를 수정하는 것보다 여러 Cloud Runner가 검증을 병렬 실행하는 것이 더 단순한 시작점이다.

Cloud 병렬화의 효과를 확인할 때도 이 경로부터 측정하는 편이 좋다.

---

## 4. Context Duplication은 숨은 비용이다

Agent가 여러 개면 각 Agent가 자신의 Context를 읽는다.

좋지 않은 구조:

```text
Agent A → Repository 전체 탐색
Agent B → Repository 전체 탐색
Agent C → Repository 전체 탐색
Agent D → Repository 전체 탐색
```

같은 README, Architecture 문서, 공통 Source를 반복해서 읽을 수 있다.

병렬 실행시간은 줄어도 Token과 탐색시간은 중복된다.

권장 구조:

```text
Agent A → attendance 관련 Context
Agent B → notification 관련 Context
Agent C → admin UI 관련 Context
```

7장의 Task Contract가 병렬화에서도 중요하다.

Task별 Context가 작아질수록 Agent 수 증가에 따른 중복 비용도 작아진다.

---

## 5. 코드 변경 병렬화는 Change Locality를 본다

다음은 병렬화하기 쉽다.

```text
Agent A → attendance module
Agent B → notification module
Agent C → library module
```

각 Module이 별도 Test를 가지고 공통 변경이 적다면 Fan-out하기 좋다.

반대로 다음 구조는 병렬성이 낮다.

```text
Agent A → common-auth
Agent B → common-auth
Agent C → common-auth
```

실행 중에는 서로 다른 Container에서 성공하더라도 통합 시 비용이 커진다.

```text
Parallel Execution
→ 빠름

Fan-in
→ Conflict / Review / Rework 증가
```

따라서 병렬 코딩에서는 Agent 수보다 `어디를 바꾸는가`를 먼저 본다.

---

## 6. Agent Count는 Task Count와 다르다

설명용 예로 Task가 10개 있다고 해서 Agent 10개를 즉시 시작할 필요는 없다.

실제 독립성이 세 개뿐이라면 다음처럼 그룹을 만들 수 있다.

```text
Task Queue: 10
   ↓
Dependency 확인
   ↓
Parallel Group A: 3
   ↓
Fan-in
   ↓
Parallel Group B: 3
```

병렬도에 영향을 주는 것은 Worker 한도만이 아니다.

```text
Task Dependency
Startup Overhead
Context Duplication
Compute Quota
Review Capacity
Merge Risk
```

Agent를 만들 수 있는 최대 수가 팀이 사용해야 할 병렬도는 아니다.

---

## 7. Fan-in이 병목이 될 수 있다

병렬화는 Worker가 작업을 시작하는 Fan-out으로 끝나지 않는다.

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

Fan-in에서는 다음 비용이 발생한다.

```text
PR Review
Conflict Resolution
Regression Test
Architecture Consistency 확인
Migration Ordering
Rework
```

Cloud Worker가 여러 PR을 빠르게 만들어도 Review와 Integration이 따라가지 못하면 전체 Lead Time은 줄지 않는다.

> 병렬화를 설계할 때 Fan-out만큼 Fan-in 비용을 본다.

---

## 8. Review Capacity가 병렬도의 상한이 된다

Agent Throughput이 Reviewer 처리량보다 크면 Queue가 쌓인다.

설명용 예:

```text
Agent가 생성 가능한 PR
20 / day

팀이 검토 가능한 PR
5 / day
```

이 숫자는 처리량 관계를 설명하기 위한 예시이며 실제 기준값이 아니다.

이 경우 병목은 Agent가 아니라 Review다.

볼 수 있는 지표:

```text
Review Queue Time
Review Duration
Merge Conflict Rate
Rework Rate
Integration Failure Rate
```

Cloud Agent 생산성을 `몇 개의 PR을 만들었는가`로만 평가하면 안 되는 이유다.

실제로 중요한 것은 변경이 **통합 가능한 상태**까지 얼마나 빨리 도달하는가다.

---

## 9. Startup Overhead도 병렬 수만큼 반복될 수 있다

Worker마다 다음 비용이 생길 수 있다.

```text
Provisioning
Repository Checkout
Dependency Restore
Image Pull
Browser 준비
```

9장의 Prepared Environment와 Cache는 이 비용을 줄인다.

```text
Reusable
→ Runtime / Tool / Dependency Cache

Fresh per Worker
→ Source / Branch / DB State / Temp / Test Output
```

병렬 Worker를 늘릴수록 Environment 준비 비용도 같이 늘 수 있으므로, 짧은 Task를 지나치게 잘게 쪼개지 않는다.

---

## 10. 병렬화 이득은 Worker 수로 계산하지 않는다

개념적으로 다음처럼 볼 수 있다.

```text
Parallel Benefit
≈ Saved Execution Time
- Startup Overhead
- Context Duplication Cost
- Merge Cost
- Review Cost
- Coordination Cost
```

정확한 수식을 만들려는 목적은 아니다.

측정해야 할 항목을 놓치지 않기 위한 개념 모델이다.

예를 들어 Worker를 2개에서 8개로 늘렸을 때 다음 현상이 동시에 나타날 수 있다.

```text
Execution Time 감소
Review Queue 증가
Merge Conflict 증가
```

이 숫자 역시 설명용 예다.

전체 Lead Time은 오히려 비슷하거나 길어질 수 있다.

병렬도는 실제 결과를 보고 조정한다.

---

## 11. Dependency-aware Parallel Group

Task Dependency를 명시하면 병렬 그룹을 만들 수 있다.

```text
Group 1
common-dto
   ↓
Group 2
├─ attendance-api
├─ notification-api
└─ library-api
   ↓
Group 3
admin-ui
```

같은 Group에는 서로 결과를 기다리지 않아도 되는 Task만 둔다.

이 장의 목적은 범용 Scheduler를 구현하는 것이 아니다.

Task를 병렬화할 때 `동시에 실행 가능한가`를 Agent 수가 아니라 Dependency로 판단하는 원칙을 세우는 것이다.

---

## 12. Java 21 Migration 예

세 서비스가 있다고 하자.

```text
service-a
service-b
service-c
```

각 서비스가 독립 Build를 가진다면 Java 21 대응을 병렬로 진행할 수 있다.

```text
+-----------+-----------+-----------+
|           |           |           |
service-a service-b   service-c
|           |           |           |
PR A      PR B        PR C
 \          |          /
        Fan-in
          ↓
    Full Validation
```

그러나 세 서비스가 같은 `common-build-plugin` 변경을 필요로 한다면 선행 Task를 만든다.

```text
Task 0
common-build-plugin Java 21 대응
   ↓
새 Base SHA
   ↓
service-a / b / c 병렬 실행
```

Dependency를 무시하고 동시에 시작하면 여러 Agent가 같은 공통 문제를 반복 해결할 수 있다.

---

## 13. Best-of-N은 일반 병렬화와 다르다

일반 병렬화는 서로 다른 Task를 나눈다.

```text
Task A → Worker A
Task B → Worker B
Task C → Worker C
```

Best-of-N은 같은 어려운 문제를 여러 Agent가 각각 푼다.

```text
같은 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
```

같은 Context와 추론 비용이 N번 반복되므로 기본값은 `N=1`로 둔다.

검토할 수 있는 조건:

```text
어려운 Bug
해결 실패 비용이 큼
후보를 자동 검증 가능
각 후보를 독립 Branch에서 실행 가능
```

후보 선택도 가능한 한 Cloud Runner의 결정론적 검증을 먼저 사용한다.

Best-of-N은 일반적인 병렬 전략이 아니라 제한된 고급 기법이다.

---

## 14. 12장에서 기억할 판단

병렬화 전에 다음 순서로 본다.

```text
1. Task가 독립적인가?
2. Read-only 검증부터 병렬화할 수 있는가?
3. 코드 변경 영역이 겹치지 않는가?
4. 각 Agent의 Context가 작게 유지되는가?
5. Startup Overhead가 Task보다 크지 않은가?
6. Review / Merge / Integration을 감당할 수 있는가?
```

정리하면 다음과 같다.

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

> Agent 수를 늘린다고 생산성이 선형 증가하지 않는다.

> Fan-out만큼 Fan-in 비용도 설계해야 한다.

다음 장에서는 이렇게 나눈 Task를 Local에서 Cloud로 넘기고, Evidence와 PR을 다시 Local로 가져오는 Handoff 흐름을 다룬다.
