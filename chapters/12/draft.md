# 12장. 병렬 Worker와 중복 Context 비용

클라우드 작업자를 여러 개 띄우면 여러 작업을 동시에 처리할 수 있다.

하지만 에이전트 수를 늘린다고 개발 속도도 같은 비율로 빨라지는 것은 아니다. 병렬로 처리하면서 다음과 같은 추가 비용이 들기 때문이다.

```text
Startup Overhead
Context Duplication
Merge Cost
Review Cost
Result Integration Cost
Coordination Cost
```

따라서 병렬화의 핵심 질문은 `몇 개의 Agent를 띄울 것인가`가 아니다.

> 서로 독립적으로 실행하고 검증할 수 있는 작업이 몇 개인가?

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

이 작업은 대부분 클라우드 실행기로 병렬화할 수 있다.

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

이처럼 계산 작업을 동시에 실행하는 것이 `Parallel Compute`다.

반면 다음은 다르다.

```text
Agent A → Repository 분석
Agent B → Repository 분석
Agent C → Repository 분석
```

같은 문제에 여러 LLM이 각각 맥락 정보를 읽고 판단한다. 이것은 `Parallel Reasoning`이다.

기본값은 먼저 실행 자원을 병렬화하고, 판단이 실제로 필요한 작업에만 에이전트를 붙이는 것이다.

---

## 2. Fan-out 전에 Dependency를 확인한다

작업이 여러 개 있다고 모두 동시에 시작하지 않는다.

```text
Task A → common DTO 변경
Task B → API 변경
Task C → UI 변경
```

의존 관계가 다음이라면:

```text
A → B → C
```

세 작업자를 동시에 시작해도 B와 C는 다시 작업해야 할 가능성이 높다.

반대로 다음처럼 독립성이 높으면 분산 실행하기 쉽다.

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

병렬화는 작업 간 의존 관계를 먼저 본 뒤 시작한다.

---

## 3. 가장 먼저 병렬화하기 좋은 것은 Read-only 검증이다

소스 코드를 수정하지 않는 검증은 병합 충돌이 없다.

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

마지막에 결과를 모아 통합하면 된다.

여러 에이전트가 동시에 코드를 수정하는 것보다 여러 클라우드 실행기가 검증을 병렬 실행하는 것이 더 단순한 시작점이다.

클라우드 병렬화의 효과를 확인할 때도 이 경로부터 측정하는 편이 좋다.

---

## 4. Context Duplication은 숨은 비용이다

에이전트가 여러 개면 각 에이전트가 자신의 맥락 정보를 읽는다.

좋지 않은 구조:

```text
Agent A → Repository 전체 탐색
Agent B → Repository 전체 탐색
Agent C → Repository 전체 탐색
Agent D → Repository 전체 탐색
```

같은 README, 시스템 구조 문서, 공통 소스 코드를 반복해서 읽을 수 있다.

병렬 실행시간은 줄어도 토큰과 탐색시간은 중복된다.

권장 구조:

```text
Agent A → attendance 관련 Context
Agent B → notification 관련 Context
Agent C → admin UI 관련 Context
```

7장의 작업 명세가 병렬화에서도 중요하다.

작업별 맥락 정보가 작아질수록 에이전트 수 증가에 따른 중복 비용도 작아진다.

---

## 5. 코드 변경 병렬화는 Change Locality를 본다

다음은 병렬화하기 쉽다.

```text
Agent A → attendance module
Agent B → notification module
Agent C → library module
```

각 모듈이 별도 테스트를 가지고 공통 변경이 적다면 분산 실행하기 좋다.

반대로 다음 구조는 병렬성이 낮다.

```text
Agent A → common-auth
Agent B → common-auth
Agent C → common-auth
```

실행 중에는 서로 다른 컨테이너에서 성공하더라도 통합 시 비용이 커진다.

```text
Parallel Execution
→ 빠름

Fan-in
→ Conflict / Review / Rework 증가
```

따라서 병렬 코딩에서는 에이전트 수보다 `어디를 바꾸는가`를 먼저 본다.

---

## 6. Agent Count는 Task Count와 다르다

설명용 예로 작업이 10개 있다고 해서 에이전트 10개를 즉시 시작할 필요는 없다.

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

병렬도에 영향을 주는 것은 작업자 한도만이 아니다.

```text
Task Dependency
Startup Overhead
Context Duplication
Compute Quota
Review Capacity
Merge Risk
```

에이전트를 만들 수 있는 최대 수가 팀이 사용해야 할 병렬도는 아니다.

---

## 7. Fan-in이 병목이 될 수 있다

병렬화는 작업을 나눠 동시에 시작하는 분산 실행(Fan-out)으로 끝나지 않는다.

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

나뉘어 실행된 결과를 모으는 결과 통합(Fan-in)에서는 다음 비용이 발생한다.

```text
PR Review
Conflict Resolution
Regression Test
Architecture Consistency 확인
Migration Ordering
Rework
```

클라우드 작업자가 여러 PR을 빠르게 만들어도 검토와 통합이 따라가지 못하면 전체 소요시간은 줄지 않는다.

> 병렬화를 설계할 때 분산 실행만큼 결과 통합 비용을 본다.

---

## 8. Review Capacity가 병렬도의 상한이 된다

에이전트 처리량이 검토자의 처리량보다 크면 대기열이 쌓인다.

설명용 예:

```text
Agent가 생성 가능한 PR
20 / day

팀이 검토 가능한 PR
5 / day
```

이 숫자는 처리량 관계를 설명하기 위한 예시이며 실제 기준값이 아니다.

이 경우 병목은 에이전트가 아니라 검토다.

볼 수 있는 지표:

```text
Review Queue Time
Review Duration
Merge Conflict Rate
Rework Rate
Integration Failure Rate
```

클라우드 에이전트 생산성을 `몇 개의 PR을 만들었는가`로만 평가하면 안 되는 이유다.

실제로 중요한 것은 변경이 **통합 가능한 상태**까지 얼마나 빨리 도달하는가다.

---

## 9. Startup Overhead도 병렬 수만큼 반복될 수 있다

작업자마다 다음 비용이 생길 수 있다.

```text
Provisioning
Repository Checkout
Dependency Restore
Image Pull
Browser 준비
```

9장의 미리 준비한 실행환경과 캐시는 이 비용을 줄인다.

```text
Reusable
→ Runtime / Tool / Dependency Cache

Fresh per Worker
→ Source / Branch / DB State / Temp / Test Output
```

병렬 작업자를 늘릴수록 실행환경 준비 비용도 같이 늘 수 있으므로, 짧은 작업을 지나치게 잘게 쪼개지 않는다.

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

이 식은 정확한 계산식을 제시하려는 것이 아니다. 무엇을 측정해야 하는지 빠뜨리지 않도록 관계를 정리한 것이다.

예를 들어 작업자를 2개에서 8개로 늘렸을 때 다음 현상이 동시에 나타날 수 있다.

```text
Execution Time 감소
Review Queue 증가
Merge Conflict 증가
```

이 숫자 역시 설명용 예다.

전체 소요시간은 오히려 비슷하거나 길어질 수 있다.

병렬도는 실제 결과를 보고 조정한다.

---

## 11. Dependency-aware Parallel Group

작업 간 의존 관계를 명시하면 병렬 그룹을 만들 수 있다.

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

같은 그룹에는 서로 결과를 기다리지 않아도 되는 작업만 둔다.

이 장의 목적은 범용 작업 스케줄러를 구현하는 것이 아니다.

작업을 병렬화할 때 `동시에 실행 가능한가`를 에이전트 수가 아니라 의존 관계로 판단하는 원칙을 세우는 것이다.

---

## 12. Java 21 Migration 예

세 서비스가 있다고 하자.

```text
service-a
service-b
service-c
```

각 서비스가 독립 빌드를 가진다면 Java 21 대응을 병렬로 진행할 수 있다.

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

그러나 세 서비스가 같은 `common-build-plugin` 변경을 필요로 한다면 선행 작업을 만든다.

```text
Task 0
common-build-plugin Java 21 대응
   ↓
새 Base SHA
   ↓
service-a / b / c 병렬 실행
```

의존 관계를 무시하고 동시에 시작하면 여러 에이전트가 같은 공통 문제를 반복 해결할 수 있다.

---

## 13. Best-of-N은 일반 병렬화와 다르다

일반 병렬화는 서로 다른 작업을 나눈다.

```text
Task A → Worker A
Task B → Worker B
Task C → Worker C
```

Best-of-N은 같은 어려운 문제를 여러 에이전트가 각각 푼다.

```text
같은 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
```

같은 맥락 정보와 추론 비용이 N번 반복되므로 기본값은 `N=1`로 둔다.

검토할 수 있는 조건:

```text
어려운 Bug
해결 실패 비용이 큼
후보를 자동 검증 가능
각 후보를 독립 Branch에서 실행 가능
```

후보 선택도 가능한 한 클라우드 실행기의 결정론적 검증을 먼저 사용한다.

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

> 병렬화의 대상은 에이전트가 아니라 독립 작업이다.

> 에이전트 수를 늘린다고 생산성이 선형 증가하지 않는다.

> 분산 실행만큼 결과 통합 비용도 설계해야 한다.

다음 장에서는 이렇게 나눈 작업을 로컬에서 클라우드로 넘기고, 검증 근거와 PR을 다시 로컬로 가져오는 작업 전달 흐름을 다룬다.
