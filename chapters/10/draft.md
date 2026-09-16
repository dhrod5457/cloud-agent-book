# 10장. Cloud Agent를 Test Runner처럼 사용하기

Cloud 환경에서 실행되는 모든 작업에 LLM이 필요한 것은 아니다.

9장에서 Prepared Cloud Environment를 만들었다면 이제 그 환경은 단순히 Agent가 코드를 수정하는 공간이 아니라 반복 가능한 실행 노드가 된다.

예를 들어 다음 작업은 결과를 프로그램으로 판정할 수 있다.

```text
Build
Unit Test
Integration Test
E2E
Docker Build
Static Analysis
Migration Validation
```

이 작업의 정상 경로에 LLM을 붙일 이유는 적다.

이 장에서는 Cloud 환경의 실행 주체를 두 가지로 나눈다.

```text
Cloud Runner
→ 정해진 명령 실행
→ CPU / RAM 중심
→ PASS / FAIL 판정

Cloud Agent
→ 실패 원인 분석
→ 코드 탐색
→ 수정 판단
→ 코드 변경
```

핵심 원칙은 다음과 같다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

---

## 1. Cloud Runner는 Agent가 아니다

Cloud Runner는 AI가 아니다.

정해진 환경에서 정해진 명령을 실행하고 결과를 반환하는 실행 노드다.

예:

```bash
./gradlew test
```

이 명령에 필요한 것은 다음이다.

```text
Repository
Java
Gradle
CPU
RAM
Test Runtime
```

LLM은 없어도 된다.

반대로 다음 작업은 판단이 필요하다.

```text
AuthServiceTest.expiredToken이 왜 실패하는지 분석하고
필요한 코드를 수정한다.
```

여기서는 Agent가 필요할 수 있다.

두 작업을 한 덩어리로 묶지 않는다.

```text
Runner
→ Test 실행
→ Failure 생성

Agent
→ Failure 분석
→ Code Fix

Runner
→ 다시 Test
```

Cloud Agent를 많이 사용하는 것보다 Agent가 필요한 구간을 정확히 좁히는 편이 낫다.

---

## 2. Deterministic First

개발 과정에는 사람이 판단하지 않아도 되는 검증이 많다.

예:

```text
Compile 성공 여부
Unit Test 성공 여부
Lint 통과 여부
Architecture Rule 위반 여부
Migration 적용 여부
Docker Build 성공 여부
```

이 결과를 Agent에게 다음처럼 물어볼 필요는 없다.

```text
코드를 살펴보고 문제가 없는 것 같으면 알려줘.
```

실행한다.

```bash
./gradlew test
./gradlew check
```

그리고 exit code와 report를 확인한다.

```text
PASS
```

또는:

```text
FAIL
```

가능하면 Validation은 자연어 판단이 아니라 executable validation으로 만든다.

```text
테스트 성공 여부
→ Test Runner

API Schema Compatibility
→ Contract Test

Architecture Dependency
→ Architecture Test

Migration
→ Disposable DB

Formatting
→ Formatter / Linter
```

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

---

## 3. 성공 경로에서 Agent를 제거한다

100개의 Task가 있다고 가정하자.

```text
100 Task
95 PASS
5 FAIL
```

모든 Task마다 Agent Session을 실행하면 다음 구조가 된다.

```text
100 Task
→ 100 Agent Session
→ 각각 Context 로딩
→ 각각 명령 실행
```

그러나 95개가 단순 PASS라면 Agent는 아무 판단도 하지 않았다.

권장 구조는 다음과 같다.

```text
100 Runner Execution
      ↓
95 PASS → 종료
5 FAIL  → 분석 필요 여부 확인
```

그중 코드 수정 판단이 필요한 실패만 Agent에게 보낸다.

```text
5 FAIL
├─ 2 Infra Failure → Environment / Retry
├─ 1 Auto-fix 가능 → Tool
└─ 2 Code Failure → Agent
```

이 경우 Agent 호출은 100번이 아니라 2번일 수 있다.

Cloud Agent 비용을 줄이는 가장 직접적인 방법 중 하나는 Token을 조금 덜 쓰게 만드는 것이 아니라 **Agent 호출 자체를 줄이는 것**이다.

---

## 4. Runner는 역할별로 나눌 수 있다

모든 Runner가 같은 일을 할 필요는 없다.

논리적으로 다음처럼 나눌 수 있다.

```text
runner-build
runner-unit
runner-integration
runner-e2e
runner-docker
runner-migration
runner-static
```

각 Runner는 필요한 Environment와 실행 명령이 다르다.

예:

```text
runner-unit
Environment: backend-test
Command: ./gradlew test

runner-e2e
Environment: frontend-e2e
Command: npx playwright test

runner-migration
Environment: migration-test
Command: migration validation
```

물리적으로 반드시 서로 다른 VM일 필요는 없다.

역할을 나누는 목적은 다음을 명확하게 만드는 것이다.

```text
어떤 Environment를 쓰는가?
어떤 Command를 실행하는가?
Timeout은 얼마인가?
어떤 Artifact를 남기는가?
어떤 Compute가 필요한가?
```

9장의 Task-specific Environment가 Runner 선택과 연결된다.

---

## 5. Runner도 Input과 Output 계약이 필요하다

Runner는 단순 shell 실행이지만 작업 상태를 추적하려면 입력과 출력이 구조화되어야 한다.

입력 예:

```text
Task ID
Git SHA
Branch
Command
Environment
Timeout
Artifact Rules
```

출력 예:

```text
Status
Exit Code
Duration
Result Summary
Artifact References
```

예:

```text
TASK=AUTH-142
SHA=abc123
COMMAND=./gradlew test --tests AuthServiceTest
STATUS=FAIL
EXIT=1
DURATION=42s
RESULT=artifacts/AUTH-142/result.json
```

이 정보는 Agent 없이도 자동화 시스템이 해석할 수 있다.

8장에서 만든 `result.json`과 같은 구조를 Runner의 결과 인터페이스로 사용할 수 있다.

---

## 6. FAIL이라고 모두 Agent를 호출하지 않는다

실패에도 종류가 있다.

예를 들어 Docker Build가 실패했다고 하자.

원인은 다음 중 하나일 수 있다.

```text
Dockerfile compile/build 문제
Registry timeout
Disk full
Network unavailable
Base image download failure
```

Registry timeout을 Agent에게 보내면 Agent는 코드나 Dockerfile을 수정하려 할 수 있다.

하지만 문제는 코드가 아니다.

그래서 FAIL 후에는 Failure Classification이 필요하다.

```text
FAIL
 ↓
Failure Classification
 ├─ INFRA_FAILURE
 ├─ BUILD_FAILURE
 ├─ TEST_FAILURE
 ├─ E2E_FAILURE
 └─ MIGRATION_FAILURE
```

그 다음 처리한다.

```text
Infra Failure
→ Retry / Environment 처리

Deterministic Auto-fix 가능
→ Tool

Code Reasoning 필요
→ Cloud Agent
```

Agent 호출 조건을 `FAIL이면 무조건`으로 만들지 않는다.

---

## 7. Infrastructure Failure와 Code Failure를 구분한다

Cloud 환경에서는 다음과 같은 인프라 실패가 발생할 수 있다.

```text
Network Timeout
Registry Unavailable
Worker Disk Full
Dependency Mirror Failure
Container Start Failure
```

이 Failure는 Source Code를 수정한다고 해결되지 않는다.

좋지 않은 흐름:

```text
Registry timeout
→ Cloud Agent
→ Dockerfile 분석
→ Dockerfile 수정
```

권장:

```text
Registry timeout
→ INFRA_FAILURE
→ Retry 또는 Environment Escalation
```

반면 다음은 코드 실패에 가깝다.

```text
Compilation Error
Assertion Failure
NullPointerException
Selector mismatch caused by changed UI
Migration syntax error
```

이 중에서도 자동으로 고칠 수 있는 것은 Tool-first로 처리하고, 의미 판단이 필요한 경우만 Agent를 부른다.

---

## 8. Result Gateway가 Agent 호출 경계를 만든다

Runner의 Raw Log를 Agent에게 그대로 보내지 않는다.

8장의 구조를 그대로 사용한다.

```text
Runner
→ Raw Artifact
→ Result Gateway
→ Failure Summary
→ Agent
```

Agent에게 처음 전달하는 정보는 작다.

예:

```text
STATUS: TEST_FAILURE

Test:
AuthServiceTest.expiredToken

Expected:
401

Actual:
200

Top Frame:
AuthServiceTest.java:94
```

이 정도 정보로 문제를 좁힌 뒤 필요할 때만 상세 로그를 조회한다.

Runner-first와 Result Gateway를 같이 사용하면 Compute와 LLM Context를 분리할 수 있다.

```text
Runner
→ 많이 실행

Gateway
→ 필요한 결과만 추출

Agent
→ 필요한 실패만 판단
```

---

## 9. Agent-on-failure

코드 판단이 필요한 Failure라면 Cloud Agent를 호출한다.

기본 흐름:

```text
Runner
  ↓
FAIL
  ↓
Result Gateway
  ↓
Cloud Agent
  ↓
Analyze
  ↓
Fix
  ↓
Commit
  ↓
Runner
  ↓
Verification
```

예를 들어 다음 테스트가 실패했다.

```text
AuthServiceTest.expiredToken
expected: 401
actual: 200
```

Agent는 관련 Context를 읽는다.

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

수정하고 Commit을 만든다.

그러나 여기서 끝나지 않는다.

Agent의 `수정 완료` 선언은 최종 성공 조건이 아니다.

다시 Runner가 검증해야 한다.

---

## 10. 수정 후 반드시 같은 검증을 다시 실행한다

좋지 않은 구조:

```text
Agent
→ Code Fix
→ "수정했습니다"
→ PR
```

권장 구조:

```text
Agent
→ Code Fix
→ Runner
→ PASS / FAIL
```

가능하면 처음 실패한 명령을 그대로 사용한다.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

Before:

```text
FAIL
expected: 401
actual: 200
```

After:

```text
PASS
```

그다음 회귀 검증 범위를 넓힌다.

```text
Target Test
→ Related Test Class
→ Module Test
→ Full Verification
```

이 과정을 통해 Agent의 자연어 판단과 실행 사실을 분리한다.

---

## 11. Retry도 Runner가 기준이 된다

Agent가 수정한 뒤 다시 실패하면 8장의 Failure Fingerprint를 사용할 수 있다.

```text
Agent Fix
→ Runner
→ FAIL
→ Fingerprint
```

첫 번째 실패:

```text
Compilation Error
```

두 번째 실패:

```text
AuthServiceTest.expiredToken assertion failure
```

실패 종류가 바뀌었다.

작업이 앞으로 진행된 것으로 볼 수 있다.

반대로 동일 Fingerprint가 반복되면 다음 Retry를 제한한다.

```text
Retry #1 → Fingerprint A
Retry #2 → Fingerprint A
```

이때 Agent에게 무한히 같은 수정을 반복시키지 않는다.

Runner 결과가 Retry 지속 여부를 결정하는 Evidence가 된다.

---

## 12. 여러 검증은 병렬 Runner로 실행한다

하나의 Git SHA에 여러 독립 검증이 필요할 수 있다.

```text
Git SHA: abc123
```

검증:

```text
Unit Test
Integration Test
E2E
Docker Build
```

서로 독립적이라면 병렬로 실행한다.

```text
             abc123
                |
    +-----------+-----------+-----------+
    |           |           |           |
  Unit      Integration    E2E       Docker
    |           |           |           |
    +-----------+-----------+-----------+
                |
              Fan-in
                |
         Verification Result
```

여기서 중요한 조건이 있다.

모든 Runner는 같은 Git SHA를 검증해야 한다.

```text
Unit → abc123
Integration → abc123
E2E → abc123
Docker → def456
```

처럼 서로 다른 상태의 결과를 하나의 Evidence로 합치면 안 된다.

Git SHA는 실행 결과의 기준점이다.

---

## 13. Test Suite가 크면 Sharding을 고려한다

테스트가 10,000개 있다고 하자.

하나의 Runner가 모두 실행하는 대신 여러 shard로 나눌 수 있다.

```text
10,000 tests
   ↓
Shard A
Shard B
Shard C
Shard D
```

조건은 다음과 같다.

```text
Shard 간 State 공유가 적음
각 결과를 합산 가능
실패 Test를 식별 가능
모든 Shard가 같은 SHA 사용
```

하지만 Shard 수를 무조건 늘리는 것은 좋지 않다.

각 Runner에는 시작 비용이 있다.

```text
Worker Provision
Checkout
Environment Restore
Scheduling
Artifact Merge
```

테스트가 30초인데 Worker 준비에 40초가 걸리면 과도한 Sharding이 더 느릴 수 있다.

병렬화는 Task 실행시간과 시작 비용을 함께 보고 결정한다.

---

## 14. Runner에도 Resource와 Timeout이 필요하다

모든 작업이 같은 Compute를 요구하지 않는다.

예:

```text
unit-test
→ CPU 중심
→ 짧은 Timeout

integration-test
→ Memory / Docker 필요
→ 긴 Timeout

e2e
→ Browser 필요
→ Wall-clock 길 수 있음
```

개념적으로 다음처럼 관리할 수 있다.

```text
runner-unit
cpu: small
timeout: project-defined

runner-integration
memory: larger
docker: required
timeout: project-defined

runner-e2e
browser: required
timeout: project-defined
```

구체적인 CPU/RAM 숫자를 책의 표준값으로 고정하지 않는다.

프로젝트에서 측정한다.

Timeout이 발생했을 때도 코드 Failure와 Resource 부족을 구분한다.

```text
Test Deadlock
vs
Worker OOM
```

둘은 처리 방식이 다르다.

---

## 15. Runner-first가 Token을 줄이는 이유

3장에서 Compute와 Token을 분리했다.

Runner-first는 그 개념을 실제 Workflow로 구현한다.

Token이 필요한 순간:

```text
Failure 분석
Code 탐색
수정 판단
Code 생성
Review
```

Token이 필요하지 않은 순간:

```text
Gradle Compile 진행
JUnit 8,000건 실행
Docker Layer Build
Browser Scenario 반복
Static Analysis 실행
```

그래서 다음 구조를 사용한다.

```text
LLM
→ 무엇을 할지 판단

Runner
→ 많이 실행

Result Gateway
→ 필요한 결과만 전달

LLM
→ 필요한 실패만 판단
```

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

이 문장이 10장에서 실제 실행 구조가 된다.

---

## 16. CI와 Runner-first는 자연스럽게 연결된다

Runner는 개발자가 직접 실행할 수도 있고 Event가 실행할 수도 있다.

예:

```text
Git Push
→ Runner
→ PASS
→ 종료
```

또는:

```text
Git Push
→ Runner
→ FAIL
→ Failure Summary
→ Agent
→ Fix
→ Runner
```

Dependency Update도 같은 구조다.

```text
Dependency 변경
→ Build / Test Runner
→ PASS → 종료
→ FAIL → Agent 후보
```

Nightly Test도 같다.

```text
Schedule
→ Runner
→ PASS → 아무 일 없음
→ FAIL → Task 생성
```

14장에서는 이 구조를 Issue, CI, Review, Schedule Event와 Task Queue까지 확장한다.

10장에서는 `Runner 결과가 Agent 호출 조건이 된다`는 점까지만 확정한다.

---

## 17. campus-platform Runner 구성

`campus-platform`을 기준으로 다음 Runner를 구성한다고 가정하자.

### runner-unit

```text
Environment
backend-test

Command
./gradlew test
```

### runner-integration

```text
Environment
backend-test

Runtime
PostgreSQL / Testcontainers

Command
integration test
```

### runner-e2e

```text
Environment
frontend-e2e

Command
npx playwright test
```

### runner-docker

```text
Environment
container-build

Command
docker build
```

### runner-migration

```text
Environment
migration-test

Command
PostgreSQL migration validation
```

Tibero 실제 검증과 HSM 연동 검증은 Cloud Runner에 억지로 넣지 않는다.

내부망이 필요한 단계는 13장의 Local Handoff로 남긴다.

---

## 18. 하나의 PR을 검증하는 흐름

PR SHA가 `abc123`이라고 하자.

```text
abc123
  |
  +-- runner-unit ───────── PASS
  +-- runner-integration ─ PASS
  +-- runner-docker ────── PASS
  +-- runner-e2e ───────── FAIL
                             ↓
                       Result Gateway
                             ↓
                    E2E Failure Summary
                             ↓
                       Cloud Agent
                             ↓
                           Fix
                             ↓
                         def456
                             ↓
                       runner-e2e
                             ↓
                           PASS
```

이제 전체 검증이 필요하다면 수정 Commit `def456` 기준으로 필요한 Runner를 다시 실행한다.

```text
def456
→ Unit
→ Integration
→ E2E
→ Docker
```

Evidence도 `def456`에 연결한다.

이렇게 해야 어떤 Source 상태가 실제로 검증됐는지 명확하다.

---

## 이 장에서 기억할 것

Cloud Agent를 잘 사용하는 것은 모든 Cloud 작업에 Agent를 붙이는 것이 아니다.

정상 경로는 가능한 한 프로그램으로 처리한다.

```text
Task
→ Runner
→ PASS
→ Done
```

실패가 발생하면 먼저 분류한다.

```text
FAIL
→ Infra?
→ Auto-fix 가능?
→ Code Reasoning 필요?
```

그중 판단과 코드 수정이 필요한 구간만 Agent가 맡는다.

```text
Runner
→ FAIL
→ Agent
→ Fix
→ Runner
→ PASS
```

그리고 성공 여부는 Agent의 설명이 아니라 Runner의 재검증으로 확정한다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

> Agent가 수정한 결과도 Runner가 다시 검증한다.

다음 장에서는 여러 Task와 Worker가 동시에 실행될 때 Git, Branch, Worktree, Container를 이용해 서로의 작업 상태를 격리하는 방법을 다룬다.