# 14장. Task Queue와 Event-driven Cloud Agent

13장까지의 Handoff는 사람이 Task를 정리해 Cloud로 넘기는 흐름이었다.

```text
Developer
→ Task Contract
→ Git Handoff
→ Cloud Worker
```

하지만 Cloud Agent의 시작점이 항상 개발자일 필요는 없다.

다음 이벤트도 Task를 만들 수 있다.

```text
CI Failure
PR Review Comment
Nightly Test Failure
Dependency Update
Static Analysis Failure
Issue
Scheduled Validation
```

이 장에서는 Cloud Agent를 상시 실행 중인 AI 프로세스로 보지 않는다.

필요한 이벤트가 발생했을 때 Task가 생성되고, 그 Task를 처리할 Worker가 실행되는 구조로 본다.

> 이벤트가 없으면 Agent도 실행하지 않는다.

그리고 이벤트가 발생했다고 곧바로 Agent를 호출하지 않는다.

> 정상 경로는 Runner가 처리하고, 실패와 판단이 필요한 순간에만 Cloud Agent를 호출한다.

---

## 1. Cloud Task의 시작점은 사람만이 아니다

개발자가 직접 다음처럼 Task를 만들 수 있다.

```text
Task
AuthService expired token fix
```

하지만 실제 개발 Workflow에는 이미 많은 이벤트가 존재한다.

```text
Git Push
CI Result
PR Review
Issue
Schedule
Dependency Bot
Static Analysis
Security Scan
```

이 이벤트에서 Cloud Task를 만들 수 있다.

기본 구조:

```text
Event Source
    ↓
Task Creation
    ↓
Task Queue
    ↓
Task Classification
    ↓
Runner / Cloud Agent / Local Escalation
```

이 장의 목적은 Kafka나 범용 Queue 제품을 설계하는 것이 아니다.

어떤 이벤트가 어떤 Task를 만들고, 언제 Agent가 실제로 호출되어야 하는지를 정리하는 것이다.

---

## 2. Git Push는 먼저 CI를 실행한다

가장 기본적인 Event-driven 흐름은 Git Push다.

```text
Git Push
   ↓
  CI
   ↓
+--+--+
|     |
PASS FAIL
|     |
Done  Failure Summary
```

PASS라면 종료한다.

Agent를 호출하지 않는다.

```text
Push
→ Runner
→ PASS
→ Done
```

FAIL이라면 먼저 Failure Classification과 Result Gateway를 거친다.

```text
Push
→ CI FAIL
→ Result Gateway
→ Failure Classification
```

그다음 판단이 필요한 경우에만 Agent를 호출한다.

```text
Code Reasoning Required
→ Cloud Agent
→ Fix
→ Runner Revalidation
```

전체 흐름:

```text
Git Push
   ↓
Runner / CI
   ↓
PASS ─────────────→ Done
   ↓ FAIL
Result Gateway
   ↓
Failure Classification
   ↓
Code Fix Needed?
 ├─ NO → Retry / Tool / Escalation
 └─ YES
       ↓
  Cloud Agent
       ↓
      Fix
       ↓
     Runner
       ↓
      PASS
```

10장의 Runner-first 구조가 이벤트에 연결된 형태다.

---

## 3. Event-driven이라고 모든 이벤트에 Agent를 붙이지 않는다

다음 구조는 비효율적이다.

```text
Git Push → Agent
CI Start → Agent
CI PASS → Agent
PR Open → Agent
Review Comment → Agent
```

이벤트 수와 Agent 호출 수가 거의 같아진다.

Cloud Agent의 비용은 단순 Compute가 아니라 Context와 Reasoning도 포함한다.

따라서 다음처럼 단계적으로 처리한다.

```text
Event
→ Deterministic Rule
→ Runner / Tool
→ Result
→ 판단 필요 여부
→ Agent
```

예를 들어 Lint Failure가 auto-fix 가능한 경우:

```text
Lint FAIL
→ Formatter / Auto-fix
→ Recheck
→ PASS
```

Agent는 필요하지 않다.

반면 Unit Test가 의미 있는 Regression을 발견했다면:

```text
Unit Test FAIL
→ Failure Summary
→ Code Reasoning Required
→ Agent
```

핵심은 Agent를 이벤트 소비자의 기본값으로 두지 않는 것이다.

---

## 4. Task Queue에는 최소 정보만 있으면 된다

Event-driven 구조에서 여러 Task가 동시에 생길 수 있다.

예:

```text
CI Failure 3건
Review Comment 2건
Nightly Failure 4건
Dependency Update 1건
```

이를 바로 Worker에 흩뿌리는 대신 Task 상태를 관리할 수 있다.

최소 정보 예:

```text
Task ID
Source Event
Repository
Base SHA
Task Type
Priority
Environment
Status
Budget
```

예:

```yaml
id: ci-91
source: ci_failure
repository: campus-platform
base_sha: abc123
task_type: test_failure_fix
environment: backend-test
status: queued
max_retry: 2
```

이 구조가 거대한 Scheduler를 의미하지는 않는다.

GitHub Issue, PR metadata, CI run ID, 간단한 DB나 파일만으로도 개념을 구현할 수 있다.

중요한 것은 Task가 어떤 이벤트에서 만들어졌고, 어떤 Git 상태를 기준으로 하는지 추적할 수 있는 것이다.

---

## 5. Task Queue는 실행 순서를 정리하는 완충지대다

이벤트가 발생하는 속도와 사람이 Review할 수 있는 속도는 다를 수 있다.

예:

```text
한 시간 동안
CI Failure 12건
Review Comment 8건
Nightly Failure 5건
```

모든 이벤트마다 즉시 Agent를 띄우면 다음 문제가 생길 수 있다.

```text
Context Duplication
Compute Burst
Review Queue 증가
같은 파일 동시 수정
중복 Failure 처리
```

Task Queue를 두면 다음을 먼저 확인할 수 있다.

```text
중복인가?
이미 처리 중인가?
Dependency가 있는가?
Cloud에서 실행 가능한가?
Runner로 끝낼 수 있는가?
Review Capacity가 있는가?
```

12장의 병렬도와 Review Capacity 기준이 Event-driven 실행에도 그대로 적용된다.

---

## 6. Worker는 Task가 있을 때만 실행해도 된다

Cloud Agent를 항상 켜 둔 프로세스로 생각할 필요는 없다.

기본 구조:

```text
Task Queue
    ↓
Worker Allocation
    ↓
Task 실행
    ↓
Evidence 반환
    ↓
Worker 종료 또는 READY 복귀
```

짧고 빈번한 Task는 9장의 Warm Worker를 사용할 수 있다.

```text
READY Worker
→ Task 할당
→ 실행
→ 초기화
→ READY
```

격리가 중요하거나 장시간 실행되는 Task는 Ephemeral Worker가 단순하다.

```text
Task
→ New Worker
→ Execute
→ Artifact
→ Destroy
```

이 장에서는 Worker Pool 제품을 설계하지 않는다.

핵심은 Agent가 상시 존재해야 하는 것이 아니라 Task와 Event에 따라 실행된다는 관점이다.

---

## 7. CI Failure는 가장 명확한 Event-driven Agent 후보다

예를 들어 PR #142에서 Integration Test가 실패했다고 하자.

```text
PR #142
SHA: abc123

runner-integration
FAIL
```

Result Gateway가 다음 Summary를 만든다.

```text
Failure
AuthServiceTest.expiredToken

Expected
401

Actual
200

Top Frame
AuthServiceTest.java:94
```

Failure Classification 결과:

```text
TEST_FAILURE
reproducible: yes
code_reasoning: required
```

이때 Task를 생성한다.

```yaml
id: ci-pr142-auth
source: ci_failure
base_sha: abc123
scope: auth
validation: ./gradlew test --tests AuthServiceTest.expiredToken
```

Cloud Agent가 수정한다.

```text
Agent
→ Analyze
→ Fix
→ Commit def456
```

그리고 Runner가 다시 검증한다.

```text
runner-integration
on def456
→ PASS
```

결과는 기존 PR에 Commit으로 추가하거나 별도 Draft PR로 반환할 수 있다.

중요한 것은 CI Failure가 발생했다는 이유만으로 Agent를 호출한 것이 아니라, 재현 가능한 코드 실패로 분류된 뒤 호출했다는 점이다.

---

## 8. Infrastructure Failure는 Agent Task로 만들지 않는다

다음 Failure를 생각해 보자.

```text
Docker Registry Timeout
Dependency Mirror Unavailable
Worker Disk Full
Network Failure
```

이 이벤트도 CI Failure이지만 Source Code 문제는 아니다.

권장 흐름:

```text
CI FAIL
→ INFRA_FAILURE
→ Retry / Environment Handling
```

좋지 않은 흐름:

```text
Registry Timeout
→ Cloud Agent
→ Dockerfile 수정 시도
```

Event-driven 구조에서는 Failure Classification이 Agent 호출 비용뿐 아니라 잘못된 코드 변경도 막아준다.

---

## 9. PR Review Comment도 Task Source가 될 수 있다

Reviewer가 다음 Comment를 남겼다고 하자.

```text
이 예외 처리에 회귀 테스트를 추가해주세요.
```

이 Comment를 Cloud Task로 만들 수 있다.

```text
PR Review Comment
      ↓
Task Candidate
```

하지만 모든 Comment를 자동 실행하지 않는다.

먼저 확인한다.

```text
실제 수정 요청인가?
권한 있는 Reviewer가 남겼는가?
현재 PR Scope 안인가?
Task가 명확한가?
Validation을 정의할 수 있는가?
```

조건을 만족하면 다음 입력을 만든다.

```text
PR SHA
Review Comment
Relevant Files
Current Changed Files
Validation
```

예:

```yaml
source: review_comment
pr: 142
base_sha: def456
goal: expired token regression test 추가
scope: auth test only
validation: ./gradlew test --tests AuthServiceTest
```

Cloud Agent가 수정한 뒤 같은 PR Branch에 Commit을 추가할 수 있다.

---

## 10. PR Follow-up에서는 Context를 재사용한다

하나의 PR은 여러 이벤트를 거친다.

```text
PR 생성
→ CI
→ Review
→ 수정
→ CI 재실행
→ 추가 Review
```

각 이벤트마다 Repository 전체를 다시 설명하면 Context가 중복된다.

PR 단위로 다음 정보를 유지할 수 있다.

```text
PR 번호
Branch
Base SHA
Current SHA
Task Summary
Relevant Files
Previous Evidence
Review Comments
```

예:

```text
PR #142
Branch: agent/auth-142
Current SHA: def456
Scope: auth expired token
Relevant Files: 3
Previous Test: 24/24 PASS
```

새 Review Comment가 왔을 때 이 최소 Context에서 이어간다.

이것을 장기 Agent Memory Architecture로 확장하지 않는다.

PR Task Context를 재사용하는 수준으로만 본다.

---

## 11. Nightly Test도 같은 패턴을 사용한다

매일 새벽 전체 E2E를 실행한다고 하자.

```text
02:00
Nightly Runner
→ Full E2E
```

결과:

```text
1,200 scenarios
1,197 PASS
3 FAIL
```

세 Failure를 바로 Agent 세 개에 보내지 않는다.

먼저 분류한다.

```text
Failure #1
browser startup error
→ INFRA

Failure #2
attendance scenario regression
→ reproducible

Failure #3
notification scenario regression
→ reproducible
```

그다음 두 개만 Task를 만든다.

```text
Nightly
→ 3 FAIL
→ 1 Infra 제외
→ 2 Cloud Agent 후보
```

각 Task는 자신에게 필요한 Failure와 Relevant Context만 받는다.

7~8장의 작은 Input/Output 원칙을 Event-driven 경로에서도 유지한다.

---

## 12. Dependency Update는 Runner-first Event의 대표 사례다

Dependency Bot이나 Schedule이 버전을 변경했다고 하자.

```text
spring-library
1.2.0 → 1.3.0
```

처음부터 Agent가 변경점을 분석하지 않는다.

```text
Dependency Update
      ↓
Runner
→ Build
→ Test
```

PASS라면 바로 Review 가능한 PR을 만든다.

```text
PASS
→ PR
```

FAIL일 때만 Failure Summary를 만든다.

```text
Compile Error
Method xyz removed
```

그다음 Compatibility Fix가 필요하면 Agent를 호출한다.

```text
FAIL
→ Result Gateway
→ Cloud Agent
→ Compatibility Fix
→ Runner
```

Agent 입력도 작게 만든다.

```text
package name
old/new version
failed command
failed tests
relevant files
```

Dependency Update는 Runner-first / Agent-on-failure 원칙을 자동화하기 좋은 작업이다.

---

## 13. Issue는 바로 Agent Task가 아닐 수 있다

Issue가 생성됐다고 모두 Cloud Agent에게 보낼 수 있는 것은 아니다.

좋은 Issue:

```text
Goal 명확
Reproduction 있음
Expected Result 있음
Scope 추정 가능
Validation 가능
```

예:

```text
Expired token 요청이 200을 반환함
expected: 401
reproduction test 존재
```

이 Issue는 Cloud Task로 변환하기 쉽다.

반대로 다음 Issue는 다르다.

```text
시스템 전체가 가끔 느립니다. 개선해주세요.
```

여기에는 다음이 없다.

```text
재현 조건
Scope
병목 위치
Validation
```

이 경우 먼저 Local 분석이 필요하다.

```text
Issue
→ Local Investigation
→ Task Split
→ Cloud Candidate 생성
```

Event-driven이라고 해서 불명확한 문제를 자동으로 Cloud에 넘기지 않는다.

---

## 14. Event에서 Task Contract를 생성한다

이벤트에는 원래부터 일부 Context가 들어 있다.

CI Failure라면:

```text
Repository
SHA
Failed Job
Failed Test
Log
```

Review Comment라면:

```text
PR
Current SHA
Changed Files
Comment
```

이 정보를 7장의 Task Contract로 변환한다.

예:

```text
Event
CI Failure
       ↓
Task Contract

Task
AuthService expired token failure fix

Base SHA
abc123

Relevant Files
AuthService.java
AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Output
Commit + Test Evidence
```

Event-driven 구조에서도 Agent가 받는 것은 Raw Event가 아니라 정리된 Task다.

---

## 15. 같은 이벤트가 중복 Task를 만들 수 있다

CI 시스템에서는 같은 실패가 여러 이벤트를 만들 수 있다.

예:

```text
PR synchronize
CI retry
workflow rerun
review update
```

모두 같은 SHA와 같은 Failure를 가리킬 수 있다.

Task Dedup Key를 둘 수 있다.

예:

```text
repository
+ sha
+ event_type
+ failure_fingerprint
```

개념적으로:

```text
campus-platform
abc123
ci_failure
AuthServiceTest.expiredToken/JWTExpiredException
```

같은 Key가 이미 `queued` 또는 `running`이면 새 Agent Task를 만들지 않는다.

이 장의 목적은 분산 Dedup 시스템을 설계하는 것이 아니다.

같은 실패를 여러 Agent가 중복 처리하지 않도록 최소한의 식별자를 둔다는 원칙이다.

---

## 16. Agent Fix가 다시 이벤트를 만드는 Loop를 제어한다

Event-driven 구조에서는 Agent가 Push한 결과가 다시 CI를 실행한다.

```text
Agent Fix
→ Push
→ CI
→ FAIL
→ Agent Fix
→ Push
→ CI
→ FAIL
→ ...
```

제어가 없으면 무한 반복이 가능하다.

8장의 Budget과 Failure Fingerprint를 사용한다.

```text
max_retry
same fingerprint stop
changed failure 확인
max cost/token
Human Escalation
```

예:

```text
Retry #1
Compilation Error

Retry #2
Unit Test Failure

Retry #3
Same Unit Test Failure
```

첫 번째에서 두 번째로 실패가 바뀌었다면 작업이 진전되었다.

하지만 같은 Failure가 반복되면 중단한다.

```text
Same Fingerprint
→ Stop
→ Human Review
```

Event-driven Agent의 핵심은 자동 반복이 아니라 자동 반복의 종료 조건까지 포함하는 것이다.

---

## 17. Draft PR을 기본 Return Point로 사용할 수 있다

자동으로 생성된 변경을 즉시 Merge하는 것은 별도 정책 문제다.

이 책에서는 기본값을 다음처럼 둔다.

```text
Event
→ Cloud Agent
→ Fix
→ Runner PASS
→ Draft PR
→ Human Review
```

기존 PR의 Follow-up Task라면 같은 Branch에 Commit을 추가할 수 있다.

중요한 것은 Merge 권한이 아니다.

> 검증 가능한 결과를 Review 가능한 형태로 반환하는 것이 먼저다.

자동 Merge는 팀의 별도 정책과 권한 모델에서 결정한다.

---

## 18. Event-driven 구조는 Developer Monitoring을 줄인다

사람이 매번 CI 결과를 보고 Agent를 수동 호출한다고 하자.

```text
Developer
→ CI 확인
→ Failure 읽음
→ Agent 호출
→ Task 설명
→ 결과 확인
```

이 과정도 Developer Blocking Time을 만든다.

Event-driven 구조에서는 다음 일부를 자동화할 수 있다.

```text
CI Failure
→ Result Filter
→ Task 생성
→ Agent 실행
→ Runner Revalidation
→ PR Update
```

Developer는 Review가 필요한 시점에 들어온다.

```text
Developer
→ Evidence / PR Review
```

Cloud Agent가 비동기 Remote Worker라는 특성이 여기서 더 분명해진다.

---

## 19. campus-platform CI Failure 예제

PR #142가 있다고 하자.

```text
PR #142
SHA: abc123
```

Integration Runner가 실패한다.

```text
runner-integration
FAIL

AuthServiceTest.expiredToken
expected: 401
actual: 200
```

Event 처리:

```text
CI Failure
→ TEST_FAILURE
→ reproducible
→ Cloud Task 생성
```

Task:

```text
Task ID
ci-142-auth

Base SHA
abc123

Scope
auth expired token

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

Agent 수정:

```text
Commit
def456
```

재검증:

```text
Target Test PASS
AuthServiceTest PASS
Integration Runner PASS
```

PR은 `def456`으로 업데이트된다.

사람은 최종 Evidence와 Diff를 Review한다.

---

## 20. campus-platform Nightly 예제

매일 새벽 Admin E2E를 실행한다고 하자.

```text
02:00
runner-e2e
```

결과:

```text
3 FAIL
```

분류:

```text
Failure A
Chrome startup timeout
→ Infra

Failure B
attendance page selector mismatch
→ Code/UI candidate

Failure C
notification retry assertion failure
→ Code candidate
```

Task 생성:

```text
Task B → admin UI scope
Task C → notification scope
```

두 Task가 서로 다른 파일 Scope라면 병렬 실행할 수 있다.

```text
Cloud Agent B
Cloud Agent C
```

각 결과는 별도 Evidence와 PR로 반환한다.

이때도 12장의 병렬화 조건을 먼저 확인한다.

---

## 21. Event-driven Cloud Agent의 기본 흐름

이 장의 전체 흐름을 하나로 정리하면 다음과 같다.

```text
Event Source
- Push
- CI Failure
- Review Comment
- Nightly
- Dependency Update
- Issue
      ↓
Task Candidate
      ↓
Dedup / Classification
      ↓
Runner로 끝낼 수 있는가?
   ├─ YES → Runner → PASS → Done
   └─ NO / FAIL
          ↓
   Code Reasoning 필요한가?
      ├─ NO → Tool / Retry / Local Escalation
      └─ YES
             ↓
         Cloud Agent
             ↓
            Fix
             ↓
          Runner
             ↓
          Evidence
             ↓
          PR / Review
```

Cloud Agent는 이 흐름 중 판단과 수정이 필요한 구간에서만 등장한다.

---

## 22. 이벤트를 설계할 때 확인할 질문

Event Source를 Cloud Agent와 연결하기 전에 다음을 확인한다.

```text
이 이벤트가 실제 작업을 의미하는가?
Runner가 먼저 처리할 수 있는가?
실패를 분류할 수 있는가?
Task Scope를 만들 수 있는가?
Base SHA가 명확한가?
중복 Task를 식별할 수 있는가?
Retry 종료 조건이 있는가?
결과를 Evidence/PR로 반환할 수 있는가?
```

이 질문에 답하기 어렵다면 자동 Agent 호출보다 사람의 분류가 먼저다.

---

## 이 장에서 기억할 것

Cloud Agent를 항상 켜 둔 AI 개발자로 볼 필요는 없다.

```text
Event
→ Task
→ Runner
→ 필요 시 Agent
→ Evidence
→ Review
```

이라는 Remote Worker 실행 모델로 볼 수 있다.

핵심 원칙은 세 가지다.

> 이벤트가 없으면 Agent도 실행하지 않는다.

> 정상 경로에는 LLM이 필요하지 않다.

> 자동화에는 시작 조건뿐 아니라 중복 제거와 종료 조건도 필요하다.

다음 장에서는 지금까지 만든 Task Routing, Prepared Environment, Runner-first, Isolation, Handoff, Event-driven 구조를 `campus-platform` 하나의 운영 모델로 통합한다.
