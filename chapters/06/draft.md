# 6장. Cloud에 보내기 좋은 개발 작업

5장에서 Task를 `Local / Cloud / Hybrid / Runner-first` 중 어디에 배치할지 판단하는 기준을 만들었다.

이제 그 기준을 실제 개발 작업에 적용한다.

중요한 것은 작업 이름만 보고 Cloud Agent를 호출하지 않는 것이다.

같은 `테스트 작업`도 실제 Workflow에서는 다음처럼 나뉠 수 있다.

```text
테스트 실행
→ Cloud Runner

실패 원인 분석
→ Cloud Agent

수정 후 재검증
→ Cloud Runner
```

이 장에서는 작업을 세 범주로 본다.

```text
Deterministic Work
→ Runner / Tool

Reasoning + Code Change
→ Cloud Agent

Internal / Interactive Work
→ Local 또는 Hybrid
```

> Cloud에 보내기 좋은 작업과 Cloud Agent가 직접 해야 하는 작업은 같은 개념이 아니다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> Agent는 판단과 수정이 필요한 구간에만 사용한다.

10장에서는 이 원칙을 Runner-first 실행 구조로 더 구체화한다. 이번 장의 역할은 **실제 개발 작업을 어떤 실행 주체에 배치할지 Catalog를 만드는 것**이다.

---

## 1. 작업 이름보다 실행 구조를 본다

Dependency Update를 예로 들어보자.

```text
Version Update
      ↓
Runner
→ Build / Test
      ↓
PASS ─────────→ Done
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent
 ↓
Compatibility Fix
 ↓
Runner 재검증
```

처음부터 Agent가 Repository 전체를 분석할 필요는 없다.

작업을 볼 때 다음 질문을 사용한다.

```text
실행 명령이 명확한가?
판정 기준이 명확한가?
코드 변경이 필요한가?
판단이 필요한가?
내부 시스템 의존성이 있는가?
어떤 Evidence를 남겨야 하는가?
```

이 질문을 Build, Test, E2E, Migration, Bug Fix 같은 실제 작업에 적용한다.

---

## 2. Build는 Runner 작업이다

Java/Spring Boot 프로젝트의 Build는 보통 명령과 판정 기준이 명확하다.

```bash
./gradlew clean build
```

기본 흐름:

```text
Git SHA
  ↓
Cloud Runner
  ↓
Build
  ↓
PASS / FAIL
```

PASS라면 Agent를 호출하지 않는다.

FAIL이고 코드 수정이 필요하면 실패 요약을 Agent에게 넘긴다.

```text
BUILD: FAIL
AuthService.java:142
cannot find symbol
JwtExpiredException
```

### 기본 Evidence

```text
Git SHA
Build Status
Duration
Artifact Path
Compile Error Summary
```

Build 방법이 이미 표준화되어 있다면 Agent에게 `어떻게 빌드해야 할지` 다시 추론시키지 않는다.

---

## 3. Unit Test는 Cloud Compute에 잘 맞는다

Unit Test는 실행 명령과 성공 조건이 명확하다.

```bash
./gradlew test
```

또는 대상만 좁혀 실행할 수 있다.

```bash
./gradlew test --tests AuthServiceTest
```

기본 구조:

```text
Runner
→ Unit Test
→ PASS → 종료
→ FAIL → 실패 Test 목록
```

Agent가 필요하다면 실패한 Test부터 본다.

```text
AuthServiceTest.expiredToken
expected: 401
actual: 200
```

전체 로그를 처음부터 Context에 넣는 방법은 피한다. Tool Output을 줄이는 방법은 8장에서 다룬다.

Unit Test는 다음 조건에서 Cloud 활용 가치가 커진다.

```text
Test 수가 많음
실행시간이 김
모듈별 분할 가능
Local CPU/RAM 점유가 큼
```

---

## 4. Integration Test는 재현 가능한 환경이 핵심이다

Integration Test는 Unit Test보다 Runtime 의존성이 크다.

예:

```text
Spring Boot
PostgreSQL
Redis
Kafka
Testcontainers
Mock HTTP Server
```

Cloud에서 이 환경을 재현할 수 있다면 Runner에 적합하다.

```text
Prepared Environment
        ↓
Integration Runner
        ↓
Testcontainers / DB / Mock
        ↓
Integration Test
```

반대로 실제 내부 자원이 필수라면 Hybrid가 된다.

```text
Cloud
→ 일반 Integration Validation

Local/Internal
→ Tibero / Internal API 최종 검증
```

Integration Test의 핵심 질문은 `Cloud에서 돌릴 수 있는가`보다 **Cloud에서 필요한 상태를 재현할 수 있는가**다.

---

## 5. E2E와 Browser Test는 Cloud 분리에 적합하다

Browser E2E는 실행시간과 자원 점유가 크고 Screenshot, Video, Trace 같은 Artifact를 남길 수 있다.

```text
Cloud Runner
→ Browser 시작
→ E2E 실행
→ Screenshot / Video / Trace 저장
```

시나리오가 독립적이라면 여러 Worker로 나눌 수 있다.

```text
Worker #1 → auth scenarios
Worker #2 → attendance scenarios
Worker #3 → admin scenarios
```

실패 시 Agent에게 필요한 것은 전체 Trace가 아니라 우선 다음 정보다.

```text
Failed Scenario
Expected / Actual
Screenshot
Console Error Summary
```

### 기본 Evidence

```text
Scenario Count
Failed Scenario List
Screenshot
Video / Trace Reference
Console Error Summary
```

UI 변경은 8장의 `Demos over Diffs`와 연결된다.

---

## 6. Docker Build도 Runner가 먼저다

```bash
docker build -t campus-platform:abc123 .
```

Docker Build는 일반적으로 다음 구조로 충분하다.

```text
Git SHA
→ Runner
→ Docker Build
→ PASS / FAIL
→ Image Digest / Build Log
```

실패했다고 모두 코드 Agent 문제는 아니다.

```text
Dockerfile 문제
Dependency 문제
Registry/Network 문제
Base Image 문제
```

코드나 Dockerfile 수정이 필요한 경우에만 Agent를 호출한다.

Prepared Environment와 Docker Layer Cache는 9장에서 다룬다.

---

## 7. Static Analysis와 Lint는 Tool-first다

다음은 기본적으로 일반 도구가 처리한다.

```text
Formatter
Lint
Checkstyle
SpotBugs
Architecture Check
Dependency Check
Security Scan
```

기본:

```text
Tool
→ PASS / FAIL
```

자동 수정이 가능하면 Agent보다 Tool을 먼저 사용한다.

```text
FAIL
 ↓
Auto-fix 가능?
 ├─ YES → Tool Fix → Recheck
 └─ NO  → Agent 후보
```

> 코드로 판정하고 코드로 수정할 수 있는 규칙은 Agent보다 도구를 먼저 사용한다.

---

## 8. Migration은 작성과 검증을 분리한다

DB Migration은 한 덩어리의 Agent Task로 보지 않는다.

```text
Migration 작성
→ Developer / Agent

Migration 검증
→ Runner
```

Cloud에서 Disposable DB를 사용할 수 있다면 다음을 검증할 수 있다.

```text
clean DB 적용
기존 Schema에서 upgrade
syntax
순서
기본 Integration Test
```

실제 대상이 내부 Tibero/Oracle이라면 마지막 경계만 Local에 남긴다.

```text
Cloud
→ 일반 Migration Validation
        ↓
Local/Internal
→ 실제 DB 검증
```

Agent가 Migration을 작성했다고 Agent의 자연어 판단으로 검증을 끝내지 않는다.

---

## 9. 반복 Refactoring은 Cloud Agent 후보다

판단과 코드 변경이 필요하지만 범위를 제한할 수 있는 Refactoring은 Cloud Agent에 잘 맞는다.

예:

```text
deprecated API 교체
Java 17 → 21 호환 수정
반복 import 변경
동일 패턴 수정
```

좋은 Task:

```text
attendance 모듈에서 deprecated API를 교체하고
./gradlew :attendance:test를 통과한다.
```

좋지 않은 Task:

```text
프로젝트 전체 구조를 더 좋게 리팩터링해.
```

Cloud Agent 후보의 조건:

```text
변경 패턴 명확
Scope 제한
검증 명령 존재
독립 Branch 가능
Review 가능한 Diff
```

Architecture 방향은 Local에서 정하고 반복 실행 부분만 Cloud로 분리한다.

---

## 10. 작은 Bug Fix는 재현 가능성이 먼저다

Cloud Agent에 보내기 좋은 Bug는 실패와 완료 조건이 명확하다.

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

기본 흐름:

```text
Reproducible Failure
       ↓
Cloud Agent
       ↓
Analyze / Fix
       ↓
Runner
       ↓
PASS
```

반대로 다음 Bug는 Local에 더 가깝다.

```text
Production에서만 간헐 발생
실제 HSM 의존
내부 DB 상태 의존
재현 절차 불명확
여러 시스템을 동시에 조사해야 함
```

Cloud Agent의 모델 성능보다 **재현 가능성**이 먼저다.

---

## 11. Documentation은 사실 기반 갱신부터 보낸다

Cloud에 적합한 문서 작업:

```text
API 변경에 맞춘 README 수정
특정 Module 문서 갱신
Release Note 초안
변경 코드 기반 운영 문서 수정
```

이 작업은 Repository와 Commit을 기준으로 범위를 제한하기 쉽다.

반대로 다음은 Local 중심이 자연스럽다.

```text
새 Architecture 원칙 정의
조직 전체 개발 정책 결정
요구사항이 정해지지 않은 문서
```

즉 **사실 기반 갱신**과 **방향 결정**을 구분한다.

---

## 12. PR Review는 보조 Worker로 사용한다

Cloud Agent는 PR Review를 보조할 수 있다.

검토 후보:

```text
변경 범위
Test 누락
명확한 오류
규칙 위반
문서/코드 불일치
```

권장 순서:

```text
PR
→ Deterministic Checks
→ Cloud Review Agent
→ Findings
→ Human Review
```

Cloud Agent의 Review를 최종 승인과 동일하게 취급하지 않는다.

기계적으로 검증할 수 있는 항목은 CI/Tool이 먼저 처리하고, Agent는 의미 판단을 보조한다.

---

## 13. Dependency Update는 Runner-first 대표 사례다

```text
Dependency Version 변경
      ↓
Runner
→ Build / Test
      ↓
PASS → PR
FAIL
 ↓
Failure Summary
 ↓
Agent Compatibility Fix
 ↓
Runner 재검증
```

PASS하면 LLM이 필요 없다.

FAIL일 때도 Agent에게 Repository 전체를 다시 설명하지 않는다.

```text
변경 Dependency
이전/신규 Version
실패 Command
실패 Test
관련 파일
```

이 패턴은 Event-driven Workflow에도 그대로 연결된다.

---

## 14. CI Failure Fix는 Agent-on-failure에 잘 맞는다

```text
Git Push / PR
     ↓
    CI
     ↓
PASS → Done
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent
 ↓
Fix
 ↓
Runner
 ↓
PASS
```

이 구조에서 Agent는 상시 실행되지 않는다.

실패가 발생했고 코드 판단이 필요한 경우에만 호출한다.

CI Failure, Review Comment, Nightly Test 같은 Event가 실제 Task를 만드는 구조는 14장에서 다룬다.

---

## 15. 하나의 작업 안에서도 실행 주체는 바뀐다

`Unit Test는 Runner`, `Bug Fix는 Agent`처럼 작업 이름에 실행 주체를 영구적으로 붙이지 않는다.

하나의 Workflow 안에서도 바뀐다.

```text
Unit Test 실행
→ Runner

실패 원인 분석
→ Agent

코드 수정
→ Agent

재검증
→ Runner
```

Migration도 마찬가지다.

```text
Migration 작성
→ Developer / Agent

Disposable DB Validation
→ Cloud Runner

Tibero 실검증
→ Local/Internal
```

실행 주체는 Task의 **현재 단계**에 따라 선택한다.

---

## 16. 작업별 기본 Catalog

| 작업 | 기본 실행 주체 | Agent 호출 조건 | 대표 Evidence |
| --- | --- | --- | --- |
| Build | Cloud Runner | 코드/Build 수정 필요 | Status, Artifact, Error Summary |
| Unit Test | Cloud Runner | 실패 원인 분석/수정 | Failed Tests, JUnit Result |
| Integration Test | Cloud Runner | 재현 가능한 Code Failure | Test Result, Logs/Artifacts |
| E2E | Cloud Runner | UI/Code 분석 필요 | Screenshot, Video/Trace |
| Docker Build | Cloud Runner | Dockerfile/코드 수정 필요 | Image Digest, Build Result |
| Lint / Static Analysis | Tool / Runner | Auto-fix 불가 | Violation Summary |
| Migration Validation | Cloud Runner | Migration 수정 필요 | Apply Result, Test Result |
| 반복 Refactoring | Cloud Agent | 처음부터 판단/수정 필요 | Commit, Test Result |
| 작은 Bug Fix | Cloud Agent | 재현 가능해야 함 | Commit, Target Test |
| Documentation | Cloud Agent 후보 | 사실 기반 범위 명확 | Changed Files, Review |
| PR Review | Cloud Agent 보조 | 의미 검토가 필요 | Findings |
| Dependency Update | Runner-first | FAIL일 때 | Build/Test Result |
| CI Failure Fix | Agent-on-failure | Code Failure일 때 | Fix Commit, Revalidation |

이 표는 기본값이다.

Internal Network, Human Steering, Context 크기 같은 조건은 5장의 Routing 기준을 다시 적용한다.

---

## 17. campus-platform Task Catalog

프로젝트에서 반복되는 Task는 이름과 실행 방식을 미리 정해둘 수 있다.

```text
RUN-BUILD
→ ./gradlew build

RUN-UNIT
→ ./gradlew test

RUN-INTEGRATION
→ integrationTest + Testcontainers

RUN-E2E
→ npx playwright test

RUN-DOCKER
→ docker build

FIX-BUG
→ Failure Summary + Relevant Files + Target Validation

REFACTOR-MODULE
→ One Module + Module Test
```

Migration은 다음처럼 나눈다.

```text
Cloud
→ Disposable DB Validation

Local/Internal
→ Tibero 실제 검증
```

이런 Catalog가 있으면 매번 Agent에게 `무엇을 어떻게 실행할지` 처음부터 설명할 필요가 줄어든다.

15장에서는 이 Catalog를 `campus-platform` 전체 운영 모델에 배치한다.

---

## 18. 이 장에서 기억할 실행 순서

실제 작업을 받으면 다음처럼 생각한다.

```text
1. Cloud로 보낼 가치가 있는가?
2. Tool/Runner만으로 처리 가능한가?
3. 실패 또는 코드 판단이 발생했는가?
4. Agent가 필요한가?
5. Agent 수정 후 Runner로 재검증했는가?
6. Evidence를 남겼는가?
```

한 줄로 줄이면 다음과 같다.

```text
Runner
→ 필요한 순간에 Agent
→ 다시 Runner
```

다음 장에서는 Agent에게 실제 수정 Task를 넘길 때 Repository 전체를 다시 탐색하지 않도록 `Task Contract`와 작은 Context를 구성한다.
