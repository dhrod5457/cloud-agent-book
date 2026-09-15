# 12장 설계 - 병렬 Worker와 중복 Context 비용

## 장의 목표

여러 Cloud Worker를 병렬로 사용할 때 얻는 Compute 이점과 함께 Context 중복, Merge Conflict, Review Cost, PR 관리 비용을 같이 고려하는 병렬화 전략을 설계한다.

핵심 질문:

> Cloud Agent를 여러 개 띄우면 무엇이 빨라지고, 어디서부터 비용이 더 커지는가?

---

## 핵심 주장

> 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.

> Agent 수를 늘린다고 생산성이 선형 증가하지 않는다.

> Parallel Compute의 이득보다 Context, Merge, Review 비용이 커지기 시작하는 지점이 있다.

---

## 독자가 얻는 것

- Fan-out/Fan-in 구조를 설계할 수 있다.
- 독립 Task와 의존 Task를 구분할 수 있다.
- 여러 Agent가 같은 Repository를 반복 분석하는 Context 중복 비용을 줄일 수 있다.
- Agent 수를 정할 때 Compute와 Integration 비용을 함께 볼 수 있다.
- Multi-Repository/Module 작업을 병렬화할 조건을 판단할 수 있다.
- Best-of-N을 일반 병렬화와 구분할 수 있다.
- Review와 Merge가 병목이 되는 시점을 이해할 수 있다.

---

# 절 구성

## 12.1 병렬화는 Compute를 늘리는 방법이다

좋은 예:

```text
Unit Test
Integration Test
E2E
Docker Build
```

서로 독립적인 검증이면 다음처럼 병렬화할 수 있다.

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

이 경우 Agent가 여러 개 필요한 것이 아니라 여러 실행 노드가 필요한 것이다.

---

## 12.2 Fan-out 전 Dependency를 확인한다

병렬화 가능 조건:

- 다른 Repository
- 다른 Module
- 다른 File Scope
- 독립 검증 가능
- Merge 순서 의존성이 낮음

의존성이 있으면 순차 처리한다.

예:

```text
Task A: common DTO 변경
Task B: API 변경
Task C: UI 변경
```

의존 관계가:

```text
A → B → C
```

라면 무조건 동시에 실행하는 것보다 단계적으로 진행하는 편이 낫다.

---

## 12.3 Context Duplication

여러 Agent가 같은 Repository 전체를 처음부터 분석하면 다음 비용이 반복된다.

- README/Architecture 재탐색
- 동일 Source 읽기
- 동일 Tool 결과 읽기
- 동일 Dependency 구조 이해

예:

```text
Agent A → Repository 전체 분석
Agent B → Repository 전체 분석
Agent C → Repository 전체 분석
Agent D → Repository 전체 분석
```

Task별 작은 Context를 제공하면 중복을 줄일 수 있다.

```text
Agent A → attendance only
Agent B → notification only
Agent C → admin UI only
```

7장의 Task Contract를 병렬화 비용 절감 수단으로 연결한다.

---

## 12.4 Agent Count는 Task 수와 같지 않다

10개 Task가 있다고 Agent 10개가 항상 최적은 아니다.

고려 항목:

- Task dependency
- Cloud start overhead
- Token budget
- Compute quota
- Review capacity
- Merge conflict
- Result integration

개념:

```text
Task Queue 10개
↓
독립도 분석
↓
동시 실행 3개
↓
결과 통합
↓
다음 3개
```

병렬도는 프로젝트의 Review/Integration 처리량까지 고려해 정한다.

---

## 12.5 Read-only 검증은 병렬화하기 쉽다

특히 다음은 병렬화하기 쉽다.

- Unit Test
- Integration Test
- E2E
- Docker Build
- Static Analysis
- Documentation validation

Source를 수정하지 않거나 변경이 독립적이기 때문이다.

따라서 Cloud 병렬화의 첫 단계는 여러 Agent 코딩보다 `검증 작업 fan-out`이 될 수 있다.

---

## 12.6 Code Change 병렬화는 Change Locality가 중요하다

좋은 예:

```text
Agent A → attendance module
Agent B → notification module
Agent C → library module
```

나쁜 예:

```text
Agent A → common module
Agent B → common module
Agent C → common module
```

병렬 코딩에서는 `어디를 바꾸는가`가 Agent 수보다 중요하다.

---

## 12.7 Fan-in이 실제 병목이 될 수 있다

Fan-out은 쉬워도 결과 통합은 사람이 해야 할 수 있다.

비용:

- PR Review
- Conflict resolution
- full regression
- architecture consistency
- migration ordering

구조:

```text
A PR
B PR
C PR
D PR
  ↓
Fan-in
  ↓
Integration Validation
```

병렬화 설계에는 Fan-in 비용도 포함한다.

---

## 12.8 Review Capacity를 고려한다

Agent가 하루에 PR 30개를 만들 수 있어도 Reviewer가 5개만 검토할 수 있다면 전체 Flow는 빨라지지 않는다.

측정 후보:

```text
PR 생성 수
PR 대기시간
Review time
Merge conflict rate
Rework rate
```

Cloud Agent 생산성을 `생성량`만으로 평가하지 않는다.

---

## 12.9 중복 Build/Dependency 비용

Worker마다 같은 dependency를 다시 준비하면 병렬화 비용이 커진다.

9장의 Prepared Environment/Cache를 재사용한다.

```text
Parallel Worker
+ Shared Reusable Cache
+ Fresh Runtime State
```

단, mutable runtime state는 공유하지 않는다.

---

## 12.10 병렬화의 상한

개념식:

```text
Parallel Benefit
≈ Saved Execution Time
- Startup Overhead
- Duplicate Context Cost
- Merge Cost
- Review Cost
- Coordination Cost
```

정확한 수식으로 계산하기보다 팀이 측정할 항목을 제공한다.

Agent 수가 늘었는데 전체 Lead Time이 줄지 않는다면 병렬도를 다시 줄인다.

---

## 12.11 Best-of-N은 다른 종류의 병렬화다

일반 Fan-out:

```text
서로 다른 Task를 여러 Worker가 수행
```

Best-of-N:

```text
같은 어려운 Task를 여러 Agent가 각각 해결
```

예:

```text
Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
        ↓
Deterministic Validation
        ↓
Candidate 선택
```

기본값은 N=1이다.

Best-of-N은 어려운 Bug처럼 검증 가치가 큰 경우에만 Advanced Topic으로 사용한다.

---

## 12.12 후보 선택은 가능한 한 코드로 한다

Best-of-N 선택 기준 후보:

- test pass
- regression
- security check
- diff size
- changed files
- execution time

LLM 선호만으로 winner를 고르지 않는다.

---

## 12.13 Java 17 → 21 Migration Fan-out

예:

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

조건:

- 서비스별 독립 build
- 공통 library 변경 최소화
- 공통 build plugin 변경은 먼저 처리

공통 parent/build script가 바뀌어야 한다면 해당 작업을 선행 Task로 둔다.

---

## 12.14 campus-platform 병렬 예제

좋은 병렬화:

```text
Cloud #1 → attendance unit test
Cloud #2 → notification integration test
Cloud #3 → admin E2E
Cloud #4 → Docker build
```

코딩 병렬화:

```text
Agent A → attendance bug
Agent B → notification retry bug
```

나쁜 병렬화:

```text
Agent A/B/C
→ 모두 common-auth 수정
```

---

# 좋은 사례와 나쁜 사례

## Agent 수 최대화

좋지 않은 방식:

```text
Task 10개
→ Agent 10개 즉시 실행
```

권장 방식:

```text
Dependency/Conflict 분석
→ 실제 독립 Task만 동시 실행
```

## 전체 Repository 중복 Context

좋지 않은 방식:

```text
모든 Agent에게 Repository 전체 설명
```

권장 방식:

```text
Task Contract별 relevant context
```

## PR 생성이 목표

좋지 않은 지표:

```text
Agent PR 수
```

권장 지표:

```text
Lead Time
Review Time
Merge Success
Rework
```

---

# 필요한 그림

1. Fan-out / Fan-in
2. Dependency Graph → Scheduling
3. Context Duplication
4. Parallel Benefit vs Integration Cost
5. Best-of-N Advanced Pattern

---

# Phase 6 구현 후보

```text
tasks/
├─ task-a.yaml
├─ task-b.yaml
└─ task-c.yaml
```

각 Task에:

```text
scope
dependencies
parallel_group
validation
```

필드를 두는 예제를 사용할 수 있다.

---

# 앞뒤 장 연결

11장:
Task별 Source/Runtime 격리

12장:
격리된 Task를 얼마나 동시에 실행할지 결정

13장:
병렬 결과를 Local 개발 흐름으로 다시 Handoff

---

# 의도적으로 다루지 않을 내용

- 범용 Distributed Scheduler 구현
- Multi-Agent 조직론
- Agent Manager Framework 비교
- 자동 Dependency Graph 추론 플랫폼

---

# 장의 결론 메시지

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

> Agent 수를 늘린다고 생산성이 선형 증가하지 않는다.

> Fan-out만큼 Fan-in 비용도 설계해야 한다.
