# 14장. Task Queue와 Event-driven Cloud Agent

13장까지는 Developer가 Task를 정리해 Cloud로 넘기는 Handoff를 다뤘다.

```text
Developer
→ Task Contract
→ Git Handoff
→ Cloud Worker
```

하지만 실제 개발에서는 사람이 직접 시작하지 않아도 Task 후보가 생긴다.

```text
CI Failure
PR Review Comment
Nightly Failure
Dependency Update
Issue
Scheduled Validation
```

이 장에서는 Cloud Agent를 상시 실행되는 AI 프로세스로 보지 않는다.

**Event가 Task 후보를 만들고, 필요한 경우에만 Worker와 Agent가 실행되는 구조**로 본다.

> 이벤트가 없으면 Agent도 실행하지 않는다.

또 하나의 원칙을 그대로 유지한다.

> 이벤트가 발생했다고 Agent를 호출하지 않는다. 먼저 결정론적 경로를 통과시킨다.

---

## 1. Event는 Agent 호출이 아니라 Task Candidate다

다음 구조는 단순하지만 비용이 커지기 쉽다.

```text
Git Push → Agent
PR Open → Agent
Review Comment → Agent
Nightly Fail → Agent
```

이벤트 수가 곧 Agent 호출 수가 된다.

권장 구조는 다르다.

```text
Event
→ Task Candidate
→ Dedup / Classification
→ Runner / Tool
→ 판단 필요 여부
→ Cloud Agent
```

즉 Event는 작업을 시작할 **계기**이지 실행 주체를 결정하는 답이 아니다.

---

## 2. Event를 Task 입력으로 정규화한다

이벤트 종류는 달라도 Cloud Task가 필요로 하는 정보는 비슷하다.

예:

```text
Source Event
Repository
Base / Current SHA
Task Type
Goal 또는 Failure
Scope
Validation
Environment
Status
```

CI Failure 예:

```yaml
source: ci_failure
repository: campus-platform
base_sha: abc123
task_type: test_failure_fix
failure: AuthServiceTest.expiredToken
validation: ./gradlew test --tests AuthServiceTest.expiredToken
environment: backend-test
```

PR Review Comment도 같은 형태로 바꿀 수 있다.

```yaml
source: review_comment
pr: 142
base_sha: def456
goal: expired token regression test 추가
scope: auth test only
validation: ./gradlew test --tests AuthServiceTest
```

7장의 Task Contract를 Event-driven 경로에서도 재사용하는 셈이다.

---

## 3. 정상 경로는 Runner에서 끝낸다

Git Push를 예로 보자.

```text
Git Push
   ↓
  CI
   ↓
PASS → Done
FAIL → Failure Summary
```

PASS라면 Agent를 호출하지 않는다.

FAIL이어도 바로 Agent를 호출하지 않는다.

```text
CI FAIL
→ Result Gateway
→ Failure Classification
→ Code Reasoning Needed?
```

처리 경로는 다음처럼 나뉠 수 있다.

```text
Infra Failure
→ Retry / Environment 처리

Auto-fix 가능
→ Tool

Code Reasoning 필요
→ Cloud Agent
```

10장의 Runner-first 규칙을 Event의 시작점에 연결한 구조다.

---

## 4. Task Queue는 Event와 실행 사이의 완충지대다

짧은 시간에 여러 Event가 발생할 수 있다.

```text
CI Failure
Review Comment
Nightly Failure
Dependency Update
```

이를 모두 즉시 Worker로 보내기 전에 Queue 또는 상태 저장소에서 다음을 확인할 수 있다.

```text
중복인가?
이미 처리 중인가?
현재 SHA가 최신인가?
Dependency가 있는가?
Runner로 끝낼 수 있는가?
Cloud 실행이 가능한가?
```

Queue 구현이 반드시 별도 메시지 시스템일 필요는 없다.

```text
GitHub Issue / PR metadata
CI metadata
DB
파일 기반 Task 목록
```

어떤 형태든 핵심은 Event 발생과 Worker 실행을 분리하는 것이다.

---

## 5. 같은 실패를 중복 Task로 만들지 않는다

CI Retry, Push, Nightly 실행이 겹치면 같은 실패가 여러 번 Task로 만들어질 수 있다.

중복 판단 후보:

```text
Repository
Git SHA
Task Type
Failure Fingerprint
```

예:

```text
SHA: abc123
Failure: AuthServiceTest.expiredToken
Fingerprint: TEST_FAILURE/401-200/AuthServiceTest:94
```

같은 SHA와 같은 Failure가 이미 처리 중이라면 새 Agent를 시작하지 않는다.

```text
Duplicate Event
→ Existing Task에 연결
```

8장의 Failure Fingerprint가 여기서는 Event deduplication에도 사용된다.

---

## 6. CI Failure는 가장 단순한 Agent-on-failure Event다

예를 들어 PR #142의 Integration Test가 실패했다고 하자.

```text
PR #142
SHA: abc123
Integration: FAIL
```

Result Gateway가 실패를 요약한다.

```text
Test
AuthServiceTest.expiredToken

Expected
401

Actual
200
```

분류 결과가 재현 가능한 Code Failure라면 Agent Task를 만든다.

```text
CI Failure
→ Task Contract
→ Cloud Agent
→ Fix Commit def456
→ Runner Revalidation
```

결과는 기존 PR에 Commit으로 추가하거나 별도 Draft PR로 반환할 수 있다.

중요한 것은 `CI가 실패했기 때문`이 아니라 **수정 판단이 필요한 재현 가능한 실패로 분류됐기 때문**에 Agent가 호출된다는 점이다.

---

## 7. Review Comment는 수정 요청일 때만 Task가 된다

모든 Review Comment가 실행 가능한 Task는 아니다.

예:

```text
이 예외 처리에 회귀 테스트를 추가해주세요.
```

이 Comment는 다음을 확인한 뒤 Task로 바꿀 수 있다.

```text
수정 요청인가?
현재 PR Scope 안인가?
변경 목표가 명확한가?
Validation을 정의할 수 있는가?
```

반면 의견 교환이나 설계 질문은 자동 Agent Task로 만들지 않는다.

```text
이 구조 자체를 다시 고민해야 하지 않을까요?
```

이런 Comment는 Human Steering 비중이 높다.

> Event-driven 자동화에서도 Task가 명확해야 한다는 5장과 7장의 조건은 그대로 유지된다.

---

## 8. Nightly와 Dependency Update도 같은 패턴을 사용한다

Nightly Test:

```text
Schedule
→ Full E2E Runner
→ PASS → Done
→ FAIL → Failure 분류
→ 재현 가능한 실패만 Task 생성
```

Dependency Update:

```text
Version Update
→ Build / Test Runner
→ PASS → Review 가능한 PR
→ FAIL → Compatibility Fix가 필요할 때 Agent
```

이벤트 종류가 달라도 핵심 경로는 같다.

```text
Event
→ Runner
→ Result
→ 필요할 때만 Agent
```

Event별로 별도 Agent Workflow를 만드는 것보다 공통 실행 규칙을 재사용하는 편이 단순하다.

---

## 9. 불명확한 Issue는 먼저 Local Investigation으로 보낸다

Issue가 생성됐다고 곧바로 Cloud Task가 되는 것은 아니다.

Cloud Task로 바꾸기 쉬운 Issue:

```text
Reproduction 있음
Expected Result 있음
Scope 추정 가능
Validation 가능
```

반대로 다음 Issue는 바로 위임하기 어렵다.

```text
시스템이 가끔 느립니다. 개선해주세요.
```

이 경우 먼저 문제를 좁힌다.

```text
Issue
→ Local Investigation
→ Reproduction / Scope
→ Task Split
→ Cloud Candidate
```

Event-driven 구조는 Task Routing을 생략하는 자동화가 아니다.

---

## 10. Agent가 만든 변경이 다시 Event를 만들 수 있다

Cloud Agent가 Commit을 Push하면 CI가 다시 실행된다.

```text
Agent Fix
→ Push
→ CI
→ FAIL
→ Event
→ Agent Fix
→ ...
```

종료 조건이 없으면 Loop가 생긴다.

따라서 Task에는 다음 중 일부를 연결한다.

```text
Task ID
Attempt Count
Failure Fingerprint
Budget
Parent Event
```

예:

```text
같은 Failure Fingerprint 반복
또는
Retry Budget 소진
→ 자동 Agent 재호출 중단
→ Evidence 남김
```

중단 이후 Local로 되돌릴지, Task를 다시 분해할지는 17장의 Routing 역판단 기준을 사용한다.

---

## 11. 결과는 PR 또는 기존 PR의 Commit으로 돌아온다

Event-driven 작업도 반환 경계는 13장과 같다.

```text
Cloud Task
→ Result Commit
→ Evidence
→ PR 또는 기존 PR Update
```

자동화가 결과를 만들었다고 바로 Merge하지 않는다.

```text
Runner Evidence
→ Agent Change
→ Runner Revalidation
→ Review 가능한 상태
```

Human Review를 유지하면서 Event 생성과 반복 실행만 자동화할 수 있다.

---

## 12. Worker는 Task가 있을 때만 필요하다

Task Queue에 작업이 생기면 Worker를 할당한다.

```text
Task Queue
→ Worker Allocation
→ Execute
→ Evidence
→ 종료 또는 Reset
```

짧고 반복되는 Task는 9장의 Warm Worker를 사용할 수 있고, 격리가 중요한 Task는 Ephemeral Worker를 사용할 수 있다.

이 장의 목적은 Worker Pool을 설계하는 것이 아니다.

핵심은 다음이다.

> Cloud Agent는 상시 존재해야 하는 서비스가 아니라 Event에서 만들어진 Task를 처리하는 실행 주체가 될 수 있다.

---

## 13. campus-platform의 Event-driven 흐름

`campus-platform`의 기본 자동 경로를 정리하면 다음과 같다.

```text
Push / PR / Schedule / Review
             ↓
         Event
             ↓
      Task Candidate
             ↓
   Dedup / Classification
             ↓
     Runner / Tool First
             ↓
       PASS → Done
             ↓ FAIL
      Reasoning Needed?
       ├─ NO → Retry / Escalation
       └─ YES
             ↓
        Cloud Agent
             ↓
            Fix
             ↓
          Runner
             ↓
      Evidence / PR Update
```

이 구조에서 Agent는 Event 시스템의 중심이 아니다.

Task가 명확하고 판단이 필요한 순간에만 호출된다.

다음 장에서는 지금까지 만든 Routing, Environment, Runner, Agent, Git, Evidence, Handoff, Event 흐름을 `campus-platform` 하나의 운영 모델로 합친다.
