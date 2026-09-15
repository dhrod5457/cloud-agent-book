# 15장 설계 - campus-platform Cloud Agent Workflow 설계

## 장의 목표

1~14장에서 정의한 Cloud Agent 활용 원칙을 하나의 Java/Spring Boot 프로젝트 운영 모델로 통합한다.

이 장은 새로운 개념을 추가하는 장이 아니라 앞 장의 원칙을 `campus-platform`에 배치해 실제 팀 Workflow로 보이게 만드는 장이다.

핵심 질문:

> Java/Spring Boot 프로젝트에서 Local Agent, Cloud Runner, Cloud Agent를 실제로 어떻게 배치하면 되는가?

---

## 핵심 주장

> Local에서는 설계와 통합을 하고, Cloud에서는 독립적인 작업을 병렬로 처리한다.

> Cloud Agent는 Remote Development Worker이고, Cloud Runner는 반복 가능한 실행 노드다.

> Task Contract로 작업을 넘기고 Evidence로 결과를 회수한다.

---

## 독자가 얻는 것

- Java/Spring Boot 프로젝트에 Local/Cloud 역할을 배치할 수 있다.
- Build/Test/E2E/Docker/Migration 작업을 Cloud Runner에 분산할 수 있다.
- 작은 Bug Fix와 Refactoring을 Cloud Agent에 위임할 수 있다.
- Prepared Environment를 Backend/E2E/Migration용으로 나눌 수 있다.
- Git Branch/Task 상태를 운영할 수 있다.
- Result Gateway와 Evidence Result를 실제 Workflow에 연결할 수 있다.
- Tibero/HSM/Jenkins 같은 내부망 검증을 Local Handoff로 남길 수 있다.
- 전체 Workflow의 Compute/Token/Human 비용 지점을 식별할 수 있다.

---

# 예제 프로젝트 기준

`campus-platform`의 책용 축약 구조:

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

기술 예:

- Java 21
- Spring Boot 3.x
- Gradle
- MyBatis
- PostgreSQL/Testcontainers
- Redis/Kafka는 필요한 예제 범위에서 사용
- Docker
- Playwright 기반 Web E2E

실제 기업 최종 검증 후보:

- Tibero/Oracle
- HSM
- Internal Jenkins
- VPN/Internal API

---

# 15.1 전체 운영 구조

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

이 그림을 Part IX의 기준 아키텍처로 사용한다.

---

## 15.2 Local Workspace 역할

Local에서 수행:

- 요구사항 분석
- Architecture 결정
- 큰 Context 탐색
- Task 분해
- 내부망 확인
- 최종 Review
- 최종 Integration

Local Agent가 모든 테스트까지 직접 실행해야 하는 것은 아니다.

---

## 15.3 Cloud Environment 구성

### backend-test

```text
Java 21
Gradle
Docker CLI
PostgreSQL client
Testcontainers image/cache
```

### frontend-e2e

```text
Node
Playwright
Chrome
npm cache
```

### migration-test

```text
Java 21
Migration tool
PostgreSQL
DB client
```

### fullstack

Cross-stack bug에서만 제한적으로 사용한다.

---

## 15.4 Cloud Task Catalog

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
runtime: Testcontainers
```

### RUN-E2E

```text
execution: runner
runtime: Playwright
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

---

## 15.5 Task Contract 예

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

---

## 15.6 Git Task 상태

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

Task 상태와 Evidence를 같은 ID로 추적한다.

---

## 15.7 Build/Test 병렬화

기능 변경 후:

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

모든 결과는 동일 SHA 기준이어야 한다.

---

## 15.8 Result Gateway

Artifact 구조 예:

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

Agent/Developer가 처음 읽는 것은 `result.json`이다.

---

## 15.9 실패 처리

예:

```text
Unit PASS
Integration FAIL
Docker PASS
E2E PASS
```

Integration Failure:

```text
Result Gateway
→ failure classification
→ TEST_FAILURE
→ relevant failure summary
→ Cloud Agent
```

수정 후:

```text
Target Integration Test
→ PASS
→ Integration Suite
→ PASS
```

---

## 15.10 Infra Failure는 코드 Agent에게 보내지 않는다

예:

```text
Docker registry timeout
Testcontainers image pull failure
Worker disk full
```

처리:

```text
Infra Retry / Environment Fix
```

코드 수정 Agent 호출을 피한다.

---

## 15.11 Migration Hybrid Workflow

```text
Cloud
→ PostgreSQL disposable DB
→ migration validation
        ↓
Local
→ Tibero syntax/behavior validation
```

Cloud에서 가능한 범위와 실제 기업 DB 검증을 분리한다.

---

## 15.12 HSM/내부 API Hybrid Workflow

핵심 비즈니스 로직:

```text
Cloud
→ Fake/Mock 기반 Unit/Contract Test
```

실물/내부망 검증:

```text
Local
→ HSM / Internal API
```

Cloud Agent가 내부망에 접근하지 못한다고 전체 Cloud Workflow를 포기하지 않는다.

---

## 15.13 Web UI Evidence

UI Task 결과:

```text
Build PASS
E2E PASS
Before Screenshot
After Screenshot
Video/Trace
```

Developer는 먼저 Demo Evidence를 보고 필요한 Diff를 확인한다.

---

## 15.14 Event-driven CI 연결

```text
PR
→ CI Runner
→ PASS → Review
→ FAIL → Result Gateway
           ↓
        Agent Fix
           ↓
        Runner PASS
           ↓
        PR Update
```

---

## 15.15 비용 지점 표시

### Compute Cost

- Gradle Build
- Integration
- Docker
- Browser E2E

### LLM Cost

- Task 이해
- Failure 분석
- 코드 수정
- Review 보조

### Human Cost

- Task 분해
- Review
- Internal Validation
- Conflict 해결

책에서는 세 비용을 분리해 표시한다.

---

## 15.16 Developer Blocking Time 관찰

예:

```text
10:00 Task Cloud 위임
10:01 다음 기능 개발
10:45 Cloud Evidence 생성
11:10 Review
```

Cloud 실행시간과 Developer Blocking Time을 분리해 기록한다.

---

## 15.17 운영 대시보드 대신 최소 상태부터 시작한다

현재 책에서는 거대한 Agent Platform UI를 만들지 않는다.

최소 상태:

```text
Task
Branch
SHA
Execution
Status
Evidence
PR
```

파일/CI/PR만으로도 운영 가능하게 한다.

---

## 15.18 성공 기준

이 예제의 목적은 `Agent를 많이 실행하는 것`이 아니다.

성공 기준 후보:

- Local CPU/RAM 점유 감소
- Developer Blocking Time 감소
- Cloud PASS 경로의 Agent 호출 감소
- Failure Context 크기 감소
- Review 가능한 Evidence 확보
- 병렬 검증 Lead Time 감소

---

# 좋은 사례와 나쁜 사례

## 모든 것을 Agent에게

좋지 않은 방식:

```text
Agent
→ 환경 설치
→ build
→ test
→ 로그 전체 분석
→ 수정
```

권장:

```text
Prepared Env
+ Runner
+ Result Gateway
+ 필요한 경우 Agent
```

## 내부망 때문에 Cloud 포기

좋지 않은 판단:

```text
Tibero/HSM 필요
→ 모든 작업 Local
```

권장:

```text
Cloud 가능한 80% 검증
→ Local 최종 검증
```

## Evidence 없는 PR

좋지 않은 결과:

```text
PR + "완료"
```

권장:

```text
PR + Build/Test/Artifact Evidence
```

---

# 필요한 그림

1. campus-platform 전체 Workflow
2. Cloud Task Catalog
3. Parallel Verification
4. Failure → Agent → Reverification
5. Internal Hybrid Validation
6. Compute/LLM/Human Cost Map

---

# Phase 6 구현 후보

```text
cloud-env/
scripts/
tasks/
artifacts/
```

샘플 Task와 실행 스크립트를 하나의 작은 Repository 구조로 제시한다.

---

# 앞뒤 장 연결

1~14장:
개별 원칙

15장:
전체 프로젝트 운영 모델로 통합

16장:
하나의 실제 기능을 시간 순서대로 끝까지 실행

---

# 의도적으로 다루지 않을 내용

- 완전한 Agent Platform 구현
- Kubernetes 운영 상세
- 실제 대학 운영망 구성 상세
- 제품별 Cloud Agent UI 튜토리얼

---

# 장의 결론 메시지

> Local에서는 설계와 통합을 하고, Cloud에서는 독립적인 작업을 병렬로 처리한다.

> Task Contract로 작업을 넘기고 Evidence로 결과를 돌려받는다.

> 내부망 검증이 남아 있어도 Cloud에서 가능한 작업은 분리해 위임할 수 있다.
