# 16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

15장에서는 `campus-platform` 전체 운영 모델을 설계했다.

이번 장에서는 하나의 기능을 처음부터 끝까지 따라간다.

예제 기능은 다음과 같다.

```text
학생 출결 API 인증 변경
```

요구사항:

```text
만료된 인증 토큰으로 출결 API를 호출하면 HTTP 401을 반환한다.
정상 토큰의 기존 출결 등록 동작은 유지한다.
관리자 Web에서는 인증 실패 상태를 확인할 수 있어야 한다.
```

이 기능은 Backend 수정, Unit/Integration Test, Web E2E, Docker Build, 내부 DB 검증이 필요하다고 가정한다.

이 장에서 중요한 것은 특정 제품의 버튼을 누르는 순서가 아니다.

> Cloud Agent 활용은 별도 도구 사용법이 아니라 개발 Workflow 설계다.

그리고 각 단계에는 다음 단계로 넘길 수 있는 명시적인 입력과 Evidence가 있어야 한다.

---

## 1. Local에서 요구사항을 먼저 해석한다

처음부터 Cloud Agent에게 기능 전체를 맡기지 않는다.

Local에서 다음을 먼저 결정한다.

```text
정상 동작
실패 동작
영향 모듈
변경 금지 영역
내부 시스템 의존성
Cloud로 분리 가능한 검증
```

이번 기능의 초기 판단은 다음과 같다.

```text
Architecture 판단
→ Local

Backend 수정
→ Cloud Agent 후보

Unit / Integration / E2E / Docker
→ Cloud Runner

Tibero 실제 검증
→ Local
```

이 단계에서 중요한 것은 `Cloud가 가능한가`보다 `어떤 부분을 Cloud로 분리할 수 있는가`다.

---

## 2. 영향 범위를 좁힌다

예상 관련 파일을 확인한다.

```text
AuthService.java
JwtTokenProvider.java
AttendanceController.java
AuthServiceTest.java
AttendanceApiTest.java
```

그리고 변경하지 않을 영역도 확인한다.

```text
DB schema 변경 없음
HSM 변경 없음
공통 Exception format 변경 없음
OAuth 전체 구조 변경 없음
```

이 정보는 그대로 Task Contract의 Scope와 Forbidden Changes가 된다.

Task 범위를 좁히는 작업은 Agent의 일을 줄이는 동시에 Review 범위도 줄인다.

---

## 3. Base Commit을 만든다

Cloud Worker가 Local Working Directory의 미커밋 상태를 자동으로 아는 것을 기대하지 않는다.

먼저 전달 가능한 Git 상태를 만든다.

```text
Local
→ 필요한 수정 정리
→ Fast Test
→ Commit
→ Push
```

예:

```text
base_sha: abc123
```

이 SHA가 Cloud Task의 기준점이 된다.

이후 모든 검증 결과도 어떤 SHA에서 실행됐는지 기록한다.

---

## 4. 기능을 실행 가능한 Task로 분해한다

하나의 기능을 한 Agent에게 통째로 넘기지 않는다.

예:

```text
Task A
Expired token backend fix

Task B
Backend unit/integration validation

Task C
Admin Web E2E validation

Task D
Docker build validation
```

실행 주체:

```text
Task A → Cloud Agent
Task B → Cloud Runner
Task C → Cloud Runner
Task D → Cloud Runner
```

Task A는 판단과 코드 수정이 필요하고, B~D는 정해진 명령을 실행하면 된다.

---

## 5. Task Contract를 작성한다

Task A를 다음처럼 정의한다.

```text
Task
Expired token 처리 수정

Goal
Attendance API 인증 실패 → HTTP 401

Base SHA
abc123

Scope
- auth
- attendance auth boundary

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Do Not Change
- DB schema
- OAuth 전체 구조
- Common Exception format

Validation
./gradlew test --tests AuthServiceTest

Expected Result
expired token → 401
normal token → 기존 테스트 PASS

Output
- commit
- changed files
- test evidence
- result.json
```

Task Contract는 Agent에게 모든 배경을 설명하는 문서가 아니다.

Cloud에서 바로 시작할 수 있도록 범위와 완료 조건을 고정하는 입력이다.

---

## 6. Cloud Branch와 Environment를 준비한다

Task별 Branch를 만든다.

```text
branch: agent/task-auth-expired
```

Environment:

```text
backend-test
```

준비 상태:

```text
Java 21
Gradle
Docker/Testcontainers
Dependency Cache
```

Source와 Branch는 Fresh하게 checkout한다.

```text
Prepared Environment
        +
Fresh Source / Branch
        ↓
Task 시작
```

Agent가 JDK 설치나 Browser 설치부터 시작하게 하지 않는다.

---

## 7. Cloud Agent는 작은 Context에서 시작한다

Agent에게 처음 제공하는 정보는 다음 정도다.

```text
Task Contract
Relevant Files
Known Failure
```

처음부터 Repository 전체를 읽히지 않는다.

기본 탐색 순서:

```text
Relevant Files
      ↓
Direct Dependency
      ↓
Related Auth Document
      ↓
Wider Module Context
```

문제가 세 파일 안에서 해결되면 Context 확장은 필요 없다.

필요한 경우에만 한 단계씩 넓힌다.

> Cloud에서 실패하면 Context를 한 번에 넓히지 말고 필요한 정보만 추가한다.

---

## 8. 수정 후 Target Test부터 실행한다

Agent가 코드를 수정했다고 하자.

바로 전체 Test Suite를 실행하기보다 가장 좁은 검증부터 시작한다.

```bash
./gradlew test --tests AuthServiceTest
```

결과:

```text
PASS
```

이면 다음 단계로 이동한다.

FAIL이면 Agent가 긴 로그 전체를 읽지 않는다.

Result Gateway가 Failure Summary를 만든다.

---

## 9. 실패하면 작은 Result로 다시 진입한다

예를 들어 Target Test가 여전히 실패했다고 하자.

```text
STATUS: FAIL
Test: AuthServiceTest.expiredToken
expected: 401
actual: 200
Stack: AuthServiceTest.java:94
```

Agent에게 처음 전달하는 것은 이 Summary다.

필요하면 다음 순서로 추가한다.

```text
Failure Detail
→ Stack Trace
→ Specific Test Log
→ Related Artifact
→ Raw Log
```

100MB build.log를 처음부터 다시 Context에 넣지 않는다.

Source Context와 Result Context를 모두 단계적으로 확대한다.

---

## 10. Retry는 Failure Fingerprint로 제어한다

Agent가 수정하고 다시 Runner를 실행한다.

Retry #1:

```text
AuthServiceTest.expiredToken
expected 401
actual 200
```

Retry #2도 동일하다.

```text
AuthServiceTest.expiredToken
expected 401
actual 200
```

동일 Failure Fingerprint가 반복되면 다음 Retry를 무조건 수행하지 않는다.

```text
same fingerprint
+ retry budget reached
        ↓
stop
        ↓
Local / Human escalation
```

반대로 Failure가 바뀌었다면 진행 중일 수 있다.

```text
Retry #1
Compilation Failure

Retry #2
Target Test Failure
```

Compile 단계는 통과했다는 의미일 수 있다.

Retry 횟수뿐 아니라 Failure 변화 여부를 함께 본다.

---

## 11. 수정 Commit이 나오면 병렬 Regression을 실행한다

Agent가 결과 Commit을 만들었다.

```text
result_sha: def456
```

이제 같은 SHA에서 검증을 병렬 실행한다.

```text
Commit def456
      ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit   Integration Docker    E2E
|         |         |         |         |
+---------+---------+---------+---------+
```

예:

```text
Unit        PASS
Integration PASS
Docker      PASS
E2E         FAIL
```

Developer는 이 동안 다른 작업을 할 수 있다.

Cloud 실행시간과 Developer Blocking Time을 분리하는 지점이다.

---

## 12. E2E Failure는 UI Evidence부터 본다

E2E Failure가 발생했다고 하자.

Artifact:

```text
screenshot
browser video
trace
console log
```

Agent에게 처음 전달하는 정보:

```text
Scenario
attendance expired token error view

Expected
401 error state visible

Actual
generic error page

Artifact
screenshot-fail.png
```

UI 문제는 Diff만 읽는 것보다 실제 화면 Evidence가 빠를 수 있다.

```text
Screenshot / Video
→ 동작 확인
→ 필요 시 Source / Diff 확인
```

이것이 8장의 `Demos over Diffs`를 실제 기능 Workflow에 적용한 예다.

---

## 13. E2E 수정 후 다시 Runner가 검증한다

Cloud Agent가 UI 또는 API mapping을 수정했다고 하자.

새 Commit:

```text
def789
```

다시 같은 E2E를 실행한다.

```text
runner-e2e
→ PASS
```

그다음 필요한 회귀 범위를 다시 실행한다.

```text
Target E2E
→ related E2E suite
```

Agent가 성공했다고 설명하는 것이 아니라 Runner 결과가 다음 단계 진입 조건이 된다.

---

## 14. 최종 Cloud Evidence를 만든다

모든 Cloud 검증이 끝났다고 하자.

최종 결과:

```text
Task: task-auth-expired
Commit: def789

Build: PASS
Unit: 314/314 PASS
Integration: 42/42 PASS
Docker: PASS
E2E: PASS

Artifacts:
- junit.xml
- integration-junit.xml
- screenshot-after.png
- e2e-trace.zip
- build.log

PR: #142
```

Cloud Worker가 반환하는 핵심은 `완료했습니다`라는 문장이 아니다.

```text
Commit
Test Result
Artifact
PR
```

이 Evidence를 Local에서 검토한다.

---

## 15. Local에서는 Evidence를 먼저 확인한다

Developer나 Local Agent가 PR을 검토한다.

권장 순서:

```text
1. Evidence
2. Changed Files
3. Diff
4. Architecture 영향
```

UI 변경이 있으면:

```text
Screenshot / Video
→ E2E Result
→ Diff
```

순서로 볼 수 있다.

중요한 것은 Cloud Agent의 자연어 Summary를 그대로 승인하는 것이 아니다.

실행 사실을 Evidence로 먼저 확인한다.

---

## 16. 내부망 검증은 Local로 돌아온다

Cloud에서 모든 검증을 수행할 필요는 없다.

이번 기능에서 실제 운영 DB가 Tibero이고 내부 Jenkins를 사용한다고 가정한다.

Local/Internal에서 확인한다.

```text
Tibero
Internal API
Jenkins deployment job
```

HSM 변경이 없다면 HSM 검증까지 할 필요는 없다.

Task와 관련된 내부 검증만 수행한다.

```text
Cloud
→ 일반적으로 재현 가능한 검증
      ↓
Local
→ 실제 내부환경 경계 검증
```

Hybrid Workflow의 핵심이다.

---

## 17. Cloud 결과와 내부 결과를 같은 PR 기준으로 맞춘다

Cloud에서 검증한 SHA와 Local 내부 검증 SHA가 달라지면 Evidence가 섞인다.

예:

```text
Cloud E2E
→ def789

Local Tibero
→ ghi000
```

그 사이 코드가 변경됐다면 동일 결과로 볼 수 없다.

따라서 최종 검증은 PR 기준 SHA를 확인한다.

```text
PR SHA
→ Cloud Required Checks
→ Internal Validation
→ Review
```

변경이 생기면 필요한 검증을 다시 실행한다.

---

## 18. Merge 전 Full Validation을 수행한다

모든 개별 Task가 PASS했더라도 통합 상태에서 필요한 검증을 확인한다.

```text
PR SHA
→ required checks
→ internal validation
→ review complete
→ merge
```

Cloud Worker가 만든 개별 Evidence만 보고 바로 Merge하지 않는다.

공통 모듈이나 다른 PR과 통합되는 과정에서 새로운 문제가 생길 수 있기 때문이다.

Fan-out 결과는 마지막 Fan-in Validation에서 다시 확인한다.

---

## 19. Merge 후 최소 기록을 남긴다

Task 종료 후 모든 Agent 대화나 실행 로그를 영구 보관할 필요는 없다.

다음 정도를 남기면 추적에 충분할 수 있다.

```text
Task ID
Merge Commit
PR
Final Evidence
주요 Failure / Retry 정보
```

예:

```text
Task: task-auth-expired
PR: #142
Merge: 12ab34c
Retries: 2
Final: PASS
```

이 기록은 장기 Agent Memory를 만들기 위한 것이 아니다.

나중에 `왜 이 변경이 들어갔는가`, `어떤 검증을 했는가`를 확인할 최소 작업 이력이다.

---

## 20. 시간 흐름으로 보면

이 기능이 실제로 다음처럼 진행됐다고 하자.

```text
09:30 Requirement / Design
10:00 Base Commit
10:05 Cloud Task 시작
10:06 Developer 다음 작업
10:22 Agent Fix Commit
10:23 Parallel Validation
10:40 E2E Failure
10:45 Agent Fix
10:58 All PASS
11:20 Developer Review
11:35 Internal Validation
11:45 Merge
```

Cloud Task는 약 53분 동안 실행됐다.

하지만 Developer가 그 시간 전체를 기다린 것은 아니다.

```text
Cloud elapsed time
!= Developer blocking time
```

Developer는 10:06부터 다음 작업을 진행할 수 있었다.

Cloud Agent Workflow의 가치를 단순한 완료시간 하나로 측정하지 않는 이유다.

---

## 21. 비용 흐름을 분리해서 본다

이 기능의 비용을 나눠보면 다음과 같다.

### Compute

```text
Unit Test
Integration Test
Docker Build
E2E
```

### LLM

```text
Backend Failure 분석
Backend 코드 수정
E2E Failure 분석
UI/API 수정
```

### Human

```text
Requirement 해석
Task 분해
Review
Internal Validation
Merge 판단
```

만약 전체 시간이 길다면 어느 구간이 병목인지 봐야 한다.

```text
Compute가 느린가?
Agent Retry가 많은가?
Review Queue가 긴가?
내부망 검증이 오래 걸리는가?
```

Cloud Agent 사용량 자체는 목표가 아니다.

---

## 22. Local-only와 Hybrid를 비교한다

같은 기능을 전부 Local에서 처리한다고 하자.

```text
Developer
→ 구현
→ Unit Test 대기
→ Integration 대기
→ Docker 대기
→ E2E 대기
→ Failure 분석
→ 수정
→ 다시 검증
```

Hybrid에서는 다음처럼 나뉜다.

```text
Developer
→ 설계 / Task 분해 / 위임
→ 다음 작업

Cloud
→ Build / Test / E2E
→ Failure Fix
→ Evidence

Developer
→ Review / Internal Validation
```

항상 Hybrid가 더 빠르다고 단정하지 않는다.

Task가 너무 작거나 Cloud Cold Start가 크거나 Human Steering이 계속 필요하면 Local-only가 더 나을 수 있다.

비교해야 할 것은 다음이다.

```text
Total Lead Time
Developer Blocking Time
Cloud Overhead
Agent Invocation
Review Cost
```

---

## 23. 실패 분기: Cloud에서 재현되지 않는다

Cloud Runner에서 문제가 재현되지 않을 수 있다.

```text
Cloud
→ Test PASS

Production/Internal
→ Failure
```

이 경우 Cloud Agent를 계속 붙잡아 두지 않는다.

```text
cannot reproduce
→ Local Fallback
→ 실제 환경 조사
```

재현할 수 없는 Task를 Cloud에서 무한 분석하는 것은 비용만 늘릴 수 있다.

---

## 24. 실패 분기: 내부 DB 의존성이 발견된다

처음에는 일반 인증 문제로 보였지만 실제로는 Tibero-specific SQL이나 상태와 연결될 수 있다.

```text
Cloud Analysis
→ Tibero-specific behavior 발견
```

이때 실행 위치를 바꾼다.

```text
Cloud
→ 현재까지 Evidence 저장
      ↓
Local
→ Tibero 실제 검증
```

Local Fallback은 Cloud Task 실패가 아니다.

새롭게 발견된 Hard Constraint에 따라 Routing을 다시 한 것이다.

---

## 25. 실패 분기: Task Scope가 너무 커진다

처음에는 세 파일 변경을 예상했다.

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

하지만 분석 중 다음이 드러났다고 하자.

```text
auth
attendance
common exception
admin web
gateway
DB migration
```

여러 모듈에 걸친 Architecture 변경으로 확대됐다.

이 경우 Agent에게 계속 Context를 추가하지 않는다.

```text
Task Scope expanded
      ↓
Stop Cloud Task
      ↓
Local Analysis
      ↓
Task Re-split
```

Task Contract가 커지기 시작하는 것은 Routing을 다시 봐야 한다는 신호다.

---

## 26. 전체 Workflow를 한 번에 정리하면

```text
Local
Requirement / Architecture
        ↓
Impact Analysis
        ↓
Base Commit
        ↓
Task Split
        ↓
Task Contract
        ↓
Git Handoff
        ↓
Cloud Agent
Analyze / Fix
        ↓
Target Runner
        ↓
Failure?
 ├─ YES → Small Result → Agent Retry / Escalation
 └─ NO
        ↓
Parallel Regression
Unit / Integration / Docker / E2E
        ↓
Result Gateway
        ↓
Evidence / PR
        ↓
Local Review
        ↓
Internal Validation
        ↓
Final Checks
        ↓
Merge
```

Cloud Agent는 이 흐름 전체를 대신하는 존재가 아니다.

Workflow 중 판단과 코드 수정이 필요한 구간을 맡는 Remote Worker다.

Runner는 실행하고, Git은 상태를 넘기고, Evidence는 결과를 증명한다.

---

## 27. 이 기능에서 기억할 기준

하나의 기능을 Local + Cloud로 끝까지 개발할 때 중요한 기준은 다음과 같다.

```text
작은 Task로 넘긴다.
명확한 SHA에서 시작한다.
Runner가 할 수 있는 일은 Runner에게 맡긴다.
Agent에는 필요한 Context만 준다.
실패 로그 전체보다 작은 Failure Summary부터 준다.
Agent 수정 뒤 Runner가 재검증한다.
독립 검증은 같은 SHA에서 병렬화한다.
Evidence를 가지고 Local로 돌아온다.
내부망 검증은 Local에 남긴다.
Scope가 커지면 다시 Task를 분해한다.
```

> 작은 Task를 Git으로 넘기고, Runner와 Agent가 작업한 Evidence를 다시 Local로 가져온다.

이것이 이 책에서 사용하는 Local + Cloud Hybrid Workflow의 기본 형태다.
