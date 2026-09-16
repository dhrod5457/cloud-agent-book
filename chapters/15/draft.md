# 15장. campus-platform Cloud Agent Workflow 설계

지금까지는 Cloud Agent를 잘 사용하기 위한 원칙을 하나씩 나눠서 설명했다.

```text
Task Routing
Task Contract
Small Context
Result Gateway
Prepared Environment
Runner-first
Git Isolation
Parallel Worker
Local ↔ Cloud Handoff
Event-driven Task
```

15장에서는 이 원칙들을 하나의 Java/Spring Boot 프로젝트 운영 흐름으로 합친다.

이 장의 목적은 새로운 Agent Platform을 만드는 것이 아니다.

`campus-platform`이라는 예제 프로젝트에서 Local Agent, Cloud Runner, Cloud Agent를 어디에 배치할지 결정하는 것이다.

핵심 구조는 다음과 같다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture
        ↓
Task Routing / Task Contract
        ↓
Commit / Push
        ↓
Git Repository
        ↓
+----------------+----------------+----------------+----------------+
|                |                |                |                |
Cloud Runner   Cloud Runner     Cloud Runner     Cloud Agent
Unit Test      Integration      Docker/E2E       Bug Fix/Refactor
|                |                |                |
+----------------+----------------+----------------+----------------+
        ↓
Result Gateway / Evidence
        ↓
PR
        ↓
Developer / Local Agent
        ↓
Internal Validation
        ↓
Review / Merge
```

> Local에서는 설계와 통합을 하고, Cloud에서는 독립적인 작업을 병렬로 처리한다.

그리고 두 실행 위치를 연결하는 경계는 이미 앞에서 정리했다.

```text
Input Boundary
→ Git + Task Contract

Return Boundary
→ Evidence + PR
```

---

## 1. 예제 프로젝트 구조

책에서 사용할 `campus-platform`은 실제 대학 시스템을 그대로 복제하지 않는다.

Cloud Agent Workflow를 설명하기 위한 축약 구조를 사용한다.

```text
campus-platform/
├─ auth/
├─ student/
├─ attendance/
├─ notification/
├─ integration/
├─ common/
├─ admin-web/
└─ scripts/
```

기술 기준은 다음 정도로 둔다.

```text
Java 21
Spring Boot 3.x
Gradle
MyBatis
PostgreSQL / Testcontainers
Docker
Playwright
```

기업 환경의 실제 최종 검증에서는 다음과 같은 내부 자원이 있을 수 있다.

```text
Tibero / Oracle
HSM
Internal Jenkins
VPN-only API
사내 Nexus
```

이 자원은 Cloud에 억지로 모두 복제하지 않는다.

Cloud에서 재현 가능한 영역과 내부망에서만 검증 가능한 영역을 나눈다.

---

## 2. Local Workspace의 역할

Local은 프로젝트 전체를 이해하고 방향을 정하는 위치다.

대표 작업:

```text
요구사항 해석
Architecture 결정
큰 Context 탐색
Task 분해
내부망 확인
최종 Review
통합
```

예를 들어 `학생 출결 인증 오류`가 보고됐다고 하자.

처음부터 Cloud Agent에게 다음처럼 맡기지 않는다.

```text
출결 인증 쪽을 전체적으로 보고 고쳐줘.
```

먼저 Local에서 문제를 좁힌다.

```text
현상
Expired token 요청이 200을 반환

영향 영역
auth + attendance boundary

DB Schema
변경 없음

HSM
변경 없음

Expected
HTTP 401
```

이 단계에서 Task를 작게 만들 수 있어야 Cloud로 넘기기 쉬워진다.

---

## 3. Cloud Environment는 Task별로 나눈다

모든 Cloud Worker에 같은 도구를 넣을 필요는 없다.

`campus-platform`에서는 최소 세 종류의 실행환경을 가정할 수 있다.

### backend-test

```text
Java 21
Gradle
Docker CLI
PostgreSQL client
Testcontainers image/cache
Gradle dependency cache
```

### frontend-e2e

```text
Node
Playwright
Chrome
npm cache
browser cache
```

### migration-test

```text
Java 21
Migration Tool
PostgreSQL
DB client
```

Cross-stack 문제가 있을 때만 fullstack 환경을 사용한다.

기본 원칙은 다음과 같다.

```text
Task Type
→ 필요한 Environment 선택
```

큰 Image 하나를 모든 Task에 쓰기보다 필요한 Runtime만 제공한다.

---

## 4. Cloud Task Catalog를 만든다

팀이 매번 Prompt부터 새로 쓰지 않으려면 반복 작업을 Task Type으로 정리할 수 있다.

예를 들어 다음 정도면 충분하다.

### RUN-BUILD

```text
execution: runner
command: ./gradlew build
```

### RUN-UNIT

```text
execution: runner
command: ./gradlew test
```

### RUN-INTEGRATION

```text
execution: runner
environment: backend-test
runtime: Testcontainers
```

### RUN-E2E

```text
execution: runner
environment: frontend-e2e
command: npx playwright test
```

### RUN-DOCKER

```text
execution: runner
command: docker build ...
```

### FIX-BUG

```text
execution: cloud-agent
input: failure summary + relevant files
validation: target test + regression
```

### REFACTOR-MODULE

```text
execution: cloud-agent
scope: one module
validation: module test
```

Task Catalog의 목적은 Prompt Template을 늘리는 것이 아니다.

`이 작업은 Runner인가 Agent인가`, `어떤 Environment를 쓰는가`, `무엇을 Evidence로 남기는가`를 미리 정해두는 것이다.

---

## 5. Task Contract는 운영 입력 형식이 된다

작은 Bug Fix를 Cloud Agent에 넘긴다고 하자.

예:

```text
Task
AuthService expired token 처리 수정

Base SHA
abc123

Goal
Expired JWT → HTTP 401

Scope
- auth module

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest

Do Not Change
- DB schema
- OAuth 전체 구조
- 공통 Exception format

Output
- commit
- changed files
- test result
- result.json
```

Task Contract는 Agent에게 생각하는 방법을 지시하는 문서가 아니다.

운영 관점에서는 다음 네 가지를 고정한다.

```text
어디서 시작하는가?
어디까지 바꿀 수 있는가?
무엇이 성공인가?
무엇을 반환해야 하는가?
```

---

## 6. Git Task 상태를 하나로 묶는다

Cloud Task가 여러 개 생기면 자연어 상태만으로 추적하기 어렵다.

Task ID와 Git 상태를 연결한다.

예:

```text
Task ID: task-142
Base SHA: abc123
Branch: agent/task-142
Session: cloud-142
Status: verifying
Current SHA: def456
PR: #142
```

그리고 Artifact도 같은 Task ID를 사용한다.

```text
artifacts/task-142/
```

이렇게 하면 다음 관계가 명확해진다.

```text
Task
→ Base SHA
→ Branch
→ Result Commit
→ Verification
→ Evidence
→ PR
```

Git은 Source 관리 도구이면서 Cloud Task의 기준점이 된다.

---

## 7. 검증은 같은 SHA에서 병렬 실행한다

Cloud Agent가 수정 Commit `def456`을 만들었다고 하자.

이 Commit에서 다음 검증을 동시에 실행할 수 있다.

```text
Git SHA def456
      ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit   Integration Docker    E2E
|         |         |         |         |
+---------+---------+---------+---------+
          ↓
        Fan-in
```

여기서 가장 중요한 조건은 모든 Runner가 같은 SHA를 검증하는 것이다.

좋지 않은 상태:

```text
Unit        → def456
Integration → def456
Docker      → def456
E2E         → def789
```

이 결과를 한 묶음의 Evidence로 사용하면 안 된다.

검증 결과는 Source State와 항상 연결한다.

---

## 8. Result Gateway는 프로젝트의 결과 인터페이스다

Runner가 여러 개면 결과 형식도 제각각이 되기 쉽다.

이를 Task 단위 Artifact로 모은다.

```text
artifacts/task-142/
├─ result.json
├─ unit-junit.xml
├─ integration-junit.xml
├─ build.log
├─ docker-build.log
├─ screenshots/
└─ e2e-trace/
```

Agent와 Developer가 처음 읽는 것은 `result.json`이다.

예:

```json
{
  "taskId": "task-142",
  "gitSha": "def456",
  "status": "FAIL",
  "checks": {
    "unit": "PASS",
    "integration": "FAIL",
    "docker": "PASS",
    "e2e": "PASS"
  },
  "failures": [
    "AttendanceApiTest.expiredToken"
  ]
}
```

이 파일을 표준 제품 포맷으로 정하려는 것은 아니다.

중요한 것은 `큰 로그보다 작은 구조화 결과가 먼저`라는 점이다.

---

## 9. 실패는 분류한 뒤 Agent에게 보낸다

병렬 검증 결과가 다음과 같다고 하자.

```text
Unit        PASS
Integration FAIL
Docker      PASS
E2E         PASS
```

먼저 Integration Failure를 분류한다.

```text
Result Gateway
→ Failure Classification
```

결과:

```text
TEST_FAILURE
Test: AttendanceApiTest.expiredToken
expected: 401
actual: 200
```

이제 Agent Task를 만들 수 있다.

```text
Task Scope
attendance auth boundary

Relevant Files
AttendanceController.java
AuthService.java
AttendanceApiTest.java
```

Agent가 수정한 뒤 다시 Runner로 검증한다.

```text
Target Integration Test
→ PASS

Integration Suite
→ PASS
```

`Agent가 수정했기 때문에 성공`이 아니다.

Runner가 같은 조건에서 통과했기 때문에 성공이다.

---

## 10. Infrastructure Failure는 코드 Agent에게 보내지 않는다

Cloud 환경에서는 다음 실패도 자주 생길 수 있다.

```text
Docker registry timeout
Testcontainers image pull failure
Worker disk full
Network timeout
```

이 Failure를 Agent에 넘기면 잘못된 코드 수정이 일어날 수 있다.

예:

```text
Registry timeout
→ Agent
→ Dockerfile 수정
```

문제의 원인이 아니다.

권장:

```text
Infra Failure
→ retry
→ environment fix
→ operator escalation
```

코드 Agent는 코드 판단이 필요한 실패에만 붙인다.

---

## 11. Migration은 Hybrid Workflow로 둔다

DB Migration은 Cloud에서 모든 것을 끝내기 어려울 수 있다.

예를 들어 개발용 검증은 PostgreSQL/Testcontainers로 수행하고 실제 운영 대상은 Tibero라고 하자.

Cloud:

```text
Migration 작성
→ disposable PostgreSQL
→ syntax / order / basic integration
```

Local/Internal:

```text
Tibero
→ 실제 syntax
→ behavior
→ deployment procedure
```

구조:

```text
Cloud Validation
      ↓
Evidence
      ↓
Local Tibero Validation
```

내부 DB 때문에 전체 작업을 Local로 되돌릴 필요는 없다.

Cloud에서 가능한 일반 검증을 먼저 끝내고 마지막 경계만 Local에 남긴다.

---

## 12. HSM과 Internal API도 같은 방식으로 나눈다

학생증이나 인증 관련 프로젝트에서는 HSM이나 내부 API가 필요할 수 있다.

Cloud에서 다음은 가능하다.

```text
Pure Logic
Mock / Fake
Contract Test
Input/Output Validation
```

실제 장비와 내부망 검증은 Local/Internal에서 한다.

```text
Cloud
→ Crypto flow logic test
→ Mock HSM response test
      ↓
Local
→ Real HSM
→ Internal AP
```

Cloud Agent가 내부망에 접근하지 못한다는 이유로 Cloud Workflow 전체를 포기하지 않는다.

---

## 13. UI Task는 Demo Evidence를 함께 남긴다

Admin Web 변경은 Test 결과만으로 빠르게 판단하기 어려울 수 있다.

다음 결과를 함께 남긴다.

```text
Build PASS
E2E PASS
Before Screenshot
After Screenshot
Video / Trace
```

Developer는 먼저 Demo Evidence를 확인한다.

```text
Screenshot / Video
→ 실제 동작 확인
→ 필요한 Diff Review
```

Diff를 생략하는 것이 아니다.

Review 순서를 바꾸는 것이다.

결과를 먼저 확인하면 CSS와 UI 변경을 이해하는 시간이 줄어들 수 있다.

---

## 14. PR과 CI가 Event-driven 흐름을 만든다

Cloud Agent Workflow가 수동 Task 위임으로만 끝날 필요는 없다.

PR이 만들어지면 CI Runner가 실행된다.

```text
PR
→ CI Runner
→ PASS
   ↓
 Review
```

FAIL이라면:

```text
PR
→ CI FAIL
→ Result Gateway
→ Failure Classification
→ Agent Fix 필요?
   ├─ NO → Retry / Tool / Escalation
   └─ YES
        ↓
      Agent
        ↓
      Runner
        ↓
      PR Update
```

이 흐름은 Developer가 직접 터미널에서 Agent를 다시 시작하지 않아도 된다.

이벤트가 다음 Cloud Task를 만든다.

---

## 15. 비용을 Compute / LLM / Human으로 나눠 본다

이 Workflow에서 비용이 발생하는 위치를 나눠보자.

### Compute

```text
Gradle Build
Unit Test
Integration Test
Docker Build
Browser E2E
```

### LLM

```text
Task 이해
Failure 분석
코드 수정
의미 기반 Review 보조
```

### Human

```text
요구사항 해석
Task 분해
Architecture 판단
Review
Internal Validation
Conflict 해결
```

Cloud Agent 최적화는 이 세 가지 중 하나만 줄이는 문제가 아니다.

예를 들어 Cloud Compute가 조금 늘어도 Developer Blocking Time이 크게 줄 수 있다.

반대로 Agent 호출을 많이 줄였더라도 Review Queue가 쌓이면 전체 Lead Time은 줄지 않는다.

---

## 16. Developer Blocking Time을 별도 측정한다

예:

```text
10:00 Task Cloud 위임
10:01 Developer 다음 기능 개발
10:45 Cloud Evidence 생성
11:10 Developer Review
```

Cloud Worker는 45분 실행됐다.

그러나 Developer는 대부분의 시간 동안 다른 일을 했다.

측정:

```text
Cloud Execution Time
Developer Blocking Time
Review Time
Retry Time
```

Cloud Workflow의 효과는 실행시간 하나로 판단하지 않는다.

---

## 17. 처음부터 운영 대시보드를 만들 필요는 없다

Cloud Agent를 도입하면 곧바로 복잡한 Agent Platform UI를 만들고 싶어질 수 있다.

하지만 초기에 필요한 정보는 많지 않다.

```text
Task
Branch
Base SHA
Current SHA
Execution Type
Status
Evidence
PR
```

예:

```text
TASK-142
branch: agent/task-142
sha: def456
execution: cloud-agent
status: verifying
result: artifacts/task-142/result.json
pr: #142
```

이 정보는 파일, CI metadata, PR만으로도 관리할 수 있다.

먼저 Workflow가 실제로 가치가 있는지 검증하고, 반복되는 운영 문제가 생길 때 자동화를 확장한다.

---

## 18. 성공 기준은 Agent 사용량이 아니다

Cloud Agent Workflow를 도입했다고 다음 수치가 늘어나는 것을 성공으로 보지 않는다.

```text
Agent Session 수
Agent가 만든 Commit 수
Agent가 만든 PR 수
```

대신 다음을 본다.

```text
Local CPU/RAM 점유 감소
Developer Blocking Time 감소
PASS 경로 Agent 호출 감소
Failure Context 크기 감소
Review 가능한 Evidence 확보
병렬 검증 Lead Time 감소
Rework 감소
```

Cloud Agent를 많이 사용한 팀이 아니라 **필요한 곳에만 사용한 팀**이 목표다.

---

## 19. campus-platform 전체 Workflow

지금까지의 구조를 한 번에 정리하면 다음과 같다.

```text
Developer / Local Agent
        ↓
Requirement
Architecture
Task Split
        ↓
Task Routing
        ↓
Task Contract
        ↓
Commit / Push
        ↓
Git Repository
        ↓
Task-specific Environment
        ↓
+-------------+-------------+-------------+-------------+
|             |             |             |             |
Unit Runner  Integration   Docker Runner  E2E Runner
|             Runner        |             |
+-------------+-------------+-------------+-------------+
                      ↓
                Result Gateway
                      ↓
              PASS / Failure Class
                      ↓
            Code Reasoning Needed?
                ├─ NO → Done / Retry
                └─ YES
                     ↓
                 Cloud Agent
                     ↓
                    Fix
                     ↓
                 Runner Verify
                     ↓
                 Evidence / PR
                     ↓
              Developer / Local
                     ↓
          Tibero / HSM / Internal API
                     ↓
                Review / Merge
```

이 Workflow에서 Cloud Agent는 전체 시스템의 중심이 아니다.

**판단과 코드 수정이 필요한 구간에 들어가는 Remote Worker**다.

Runner, Git, Environment, Evidence가 함께 있어야 실제 개발 흐름이 된다.

---

## 20. 기억할 운영 원칙

`campus-platform` 예제에서 남겨야 할 원칙은 다음과 같다.

```text
Local
→ 설계 / 탐색 / 내부망 / 통합

Cloud Runner
→ Build / Test / E2E / Docker / Validation

Cloud Agent
→ Failure 분석 / 작은 Bug Fix / 제한된 Refactoring

Git
→ Handoff Boundary

Evidence
→ Return Boundary

Internal Validation
→ Cloud에서 끝낼 수 없는 마지막 경계
```

Task Contract로 작업을 넘기고 Evidence로 결과를 돌려받는다.

Prepared Environment로 시작 시간을 줄이고, Runner가 정상 경로를 처리한다.

실패가 생기면 필요한 Context만 Agent에 전달한다.

병렬화는 독립 Task만 수행하고 Fan-in 비용까지 함께 본다.

다음 장에서는 이 운영 모델을 하나의 기능 개발에 시간 순서대로 적용한다.
