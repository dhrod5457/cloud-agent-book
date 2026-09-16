# Table of Contents

> Phase 5 Cloud Agent 중심 재정렬본. Agent Platform 일반론은 축소하고 실제 Cloud Agent 활용, 비용, 실행환경, Handoff, Evidence 중심으로 구성한다.

# 책의 중심 질문

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

독자가 마지막에 답할 수 있어야 하는 질문:

- 왜 이 Task를 Local이 아니라 Cloud로 보내는가?
- Cloud Agent에게 보내기 전에 무엇을 준비해야 하는가?
- 어떤 Cloud Environment가 필요한가?
- Task와 Context 범위를 어디까지 줄여야 하는가?
- Token과 CPU/RAM을 어떻게 효율적으로 사용할 것인가?
- Cloud Agent에게 어떤 Evidence를 돌려받아야 하는가?
- 여러 Cloud Agent를 어디까지 병렬화할 것인가?
- Local과 Cloud 사이에서 작업을 어떻게 넘길 것인가?

# 전체 흐름

```text
Cloud Agent를 Remote Worker로 이해
        ↓
Local과 Cloud 차이 이해
        ↓
Compute와 Token 분리
        ↓
Cloud에 적합한 Task 선택
        ↓
Task / Context 축소
        ↓
Prepared Environment / Cache
        ↓
Runner / Agent 분리
        ↓
Branch / Container로 병렬화
        ↓
Git으로 Local ↔ Cloud Handoff
        ↓
CI / Event에서 Cloud Agent 호출
        ↓
Evidence Result / PR
        ↓
Local Review / Merge
```

Java/Spring Boot 기반 `campus-platform`을 책 전체의 실전 예제로 사용한다.

---

# Part I. Cloud Agent 이해

## 1장. Coding Agent에서 Cloud Worker로

### 목적

Cloud Agent를 단순히 클라우드에서 실행되는 AI가 아니라 Repository와 독립 실행환경을 가진 Remote Development Worker로 정의한다.

### 핵심 정의

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

후반부에서 다시 사용할 정의:

> 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker.

### 핵심 개념

- Coding Agent
- Repository
- Tool Use
- Remote Worker
- Execution Environment
- Evidence Result

### 실전 예제

`campus-platform` Repository를 checkout하고 build/test를 수행한 뒤 Commit, Test Result, Artifact, PR을 반환하는 Cloud Worker를 설명한다.

---

## 2장. Local Agent와 Cloud Agent

### 목적

Local과 Cloud를 경쟁 관계가 아니라 서로 다른 작업 특성에 맞는 실행 위치로 설명한다.

### 핵심 개념

- Local Agent
- Cloud Agent
- Human Steering
- Internal Network
- Isolation
- Hybrid Workflow

### Local 중심 예

- Architecture 설계
- 복잡한 디버깅
- VPN / 사내 DB / HSM
- 큰 Context
- 빠른 질의/수정 반복
- 최종 Review / Integration

### Cloud 중심 예

- Unit / Integration / E2E
- Build / Docker Build
- Static Analysis / Lint
- Migration Validation
- 작은 Bug Fix
- 반복 Refactoring
- Documentation
- PR Review
- CI Failure Fix
- 장시간 작업

핵심 질문:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

---

## 3장. Cloud Session, Container, Compute와 Token

### 목적

Cloud Agent의 CPU/RAM/Disk와 LLM Token을 서로 다른 자원으로 설명한다.

### 핵심 개념

- Cloud Session
- Container / VM
- CPU
- RAM
- Disk
- LLM Token
- Brain / Hands 간단 모델

### 핵심 흐름

```text
Agent
→ 명령 결정

Container
→ 10,000 tests 실행

Result Gateway
→ 실패 3건 추출

Agent
→ 실패 3건만 분석
```

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

---

# Part II. 왜 Cloud Agent인가

## 4장. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

### 목적

Cloud Agent의 핵심 가치를 더 많은 Token이 아니라 독립 실행환경, 비동기 작업 위임, 병렬성에서 찾는다.

### 핵심 개념

- Isolation
- Long-running Task
- Async Delegation
- Parallel Execution
- Independent Workspace
- Developer Blocking Time
- Agent Execution Time

### 시간 분리 예

```text
10:00 Cloud Task 위임
10:01 개발자는 다음 Feature 진행
10:40 Cloud Task 완료
11:20 개발자 Review
```

Agent Execution Time과 Developer Blocking Time을 구분한다.

### 실전 예제

Backend Test, Integration Test, Frontend E2E, Docker Build를 여러 Cloud Worker에 병렬 위임하고 개발자는 다음 기능 작업을 계속한다.

---

# Part III. 어떤 작업을 Cloud로 보낼 것인가

## 5장. Task Routing: Local인가 Cloud인가

### 목적

Task 특성을 기준으로 Local/Cloud 실행 위치를 선택한다.

### 핵심 판단 기준

- Scope 명확성
- 완료 조건
- 독립 검증 가능 여부
- File Conflict 가능성
- Human Steering 빈도
- Context 크기
- Internal Network 필요 여부
- Build/Test 실행시간
- Git으로 결과 회수 가능 여부

### Task 크기

너무 작은 Task는 Cold Start와 checkout overhead가 커지고, 너무 큰 Task는 Context/Retry/Review 비용이 커진다.

절대 시간 규칙 대신 프로젝트에서 측정해 적정 크기를 정한다.

### 실전 예제

Architecture 설계, JWT Bug Fix, 전체 Unit Test, HSM 검증, 한 줄 수정, E2E Test를 각각 Local/Cloud로 분류한다.

---

## 6장. Cloud에 보내기 좋은 개발 작업

### 목적

Cloud Worker에 맡기기 좋은 실제 작업 유형을 설명한다.

### 핵심 개념

- Build
- Unit Test
- Integration Test
- E2E
- Docker Build
- Static Analysis
- Lint
- Migration Validation
- Refactoring
- Bug Fix
- Documentation
- PR Review
- Dependency Update
- CI Failure Fix

### 실전 예제

`campus-platform`의 Unit Test, Docker Build, Migration Validation, Web E2E를 서로 다른 Cloud Task로 정의한다.

---

# Part IV. Cloud Agent의 Token 절약

## 7장. Cloud Agent Task Contract: 작은 Task와 작은 Context

### 목적

Cloud Agent가 Repository 전체를 반복 탐색하지 않도록 Task와 Context의 범위를 제한한다.

### Task Contract 최소 형태

```text
Task
AuthService expired token 처리 수정

Goal
Expired JWT → HTTP 401

Scope
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest

Do Not Change
- DB Schema
- OAuth 전체 구조
- 공통 Exception format

Output
- commit
- changed files
- test result
- short summary
```

### 핵심 개념

- Task Scope
- Context Scope
- Relevant Files
- Forbidden Changes
- Validation
- Expected Result
- Progressive Context
- AGENTS.md

> Task Contract의 목적은 Prompt를 길게 만드는 것이 아니라 Agent가 탐색해야 하는 범위를 줄이는 것이다.

---

## 8장. Tool Output을 줄이고 Evidence를 남기기

### 목적

대형 로그를 LLM에 직접 넣지 않고 필요한 정보만 제공하면서 원본 결과는 Artifact로 보존한다.

### 핵심 개념

- Result Filter
- Result Gateway
- Raw Artifact
- Failure Summary
- Log Lookup
- Artifact Lookup
- Retry Limit
- Token / Cost Budget
- Failure Fingerprint
- Evidence-based Result

### Evidence Result 예

```text
Implementation: DONE
Build: PASS
Unit Tests: 314 / 314 PASS
Integration Tests: 42 / 42 PASS
E2E: PASS

Artifacts:
- junit.xml
- screenshot
- video
- build.log

Commit: abc123
PR: #142
```

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

### Demos over Diffs

UI 변경에서는 Build/Test와 Before/After Screenshot, E2E Video를 먼저 확인한 뒤 필요한 Diff를 본다.

코드 Review를 생략한다는 의미는 아니다.

---

# Part V. Cloud 컴퓨팅 자원 활용

## 9장. Prepared Cloud Environment, Cache, Snapshot

### 목적

Cloud Agent가 일을 시작하기 전 환경 구성 비용을 줄인다.

### 핵심 개념

- Prepared Cloud Environment
- Cloud Environment as Code
- Base Image
- Snapshot
- Cold Start
- Gradle / Maven Cache
- npm Cache
- Docker Layer Cache
- Playwright Browser Cache
- Warm Worker
- Fresh State

### 작업별 Environment

```text
backend-test
frontend-e2e
migration-test
fullstack
```

Task 특성에 맞는 Environment를 선택한다.

### Cache / Fresh State 분리

```text
Prepared Environment
- Runtime
- Tools
- Dependencies
- Cache

+

Fresh State
- Source Code
- Branch
- Task
- Test Result
- Temporary Data
```

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

---

## 10장. Cloud Agent를 Test Runner처럼 사용하기

### 목적

CPU/RAM 중심의 결정론적 작업과 LLM 추론을 분리한다.

### 기본 구조

```text
Task
  ↓
Runner
  ├─ PASS → 종료
  └─ FAIL → Cloud Agent
                ↓
             분석/수정
                ↓
             Runner 재검증
```

### 핵심 개념

- Cloud Runner
- Agent Worker
- Deterministic Validation
- Runner-first
- Agent-on-failure
- Parallel Test
- Test Container

정상 경로에서 LLM 호출을 줄이는 것을 Cloud Agent 비용 최적화로 설명한다.

---

# Part VI. 여러 Cloud Agent 사용

## 11장. Git, Branch, Worktree, Container로 작업 격리하기

### 목적

Git을 Local과 Cloud 사이의 Handoff Boundary이자 Cloud 병렬성의 기반으로 설명한다.

### 핵심 흐름

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Task Branch
→ Work / Test
→ Commit / Push
→ PR
```

> Cloud Agent 시대에는 Git이 개발자와 Remote Worker 사이의 작업 전달 프로토콜 역할까지 수행한다.

### Task 상태 후보

- Task ID
- Session ID
- Branch
- Commit SHA
- Status
- Test Result
- PR

### 핵심 개념

- Task Branch
- Git Worktree
- Independent Clone
- Cloud Container
- File Scope
- Shared File
- Migration Conflict

같은 파일이나 Schema를 수정하는 Task는 처음부터 병렬화하지 않는다.

---

## 12장. 병렬 Worker와 중복 Context 비용

### 목적

여러 Cloud Agent를 사용할 때 Parallel Compute의 이점과 중복 비용을 함께 관리한다.

### 핵심 개념

- Fan-out
- Fan-in
- Context Duplication
- Parallel Compute
- Agent Count
- Merge Conflict
- Review Cost
- Best-of-N Advanced Pattern

### 좋은 병렬화

- Backend Unit Test
- Frontend E2E
- Docker Build
- Documentation

### 나쁜 병렬화

여러 Agent가 동시에 `UserService` 구조를 변경하는 작업.

Agent 수를 늘린다고 생산성이 선형 증가하지 않는다는 점을 설명한다.

---

# Part VII. Local + Cloud Hybrid Workflow

## 13장. Local → Cloud → Local Handoff

### 목적

하나의 Task가 단계에 따라 Local과 Cloud 사이를 이동하는 실제 개발 흐름을 설계한다.

### 기본 흐름

```text
Local
→ 요구사항 분석
→ Architecture
→ 핵심 구현 / Task Split
→ Commit / Push
       ↓
Cloud
→ Build / Test / E2E
→ Refactoring / CI Fix
→ Evidence Result / PR
       ↓
Local
→ 내부망 검증
→ Review
→ Integration / Merge
```

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하는 것이 아니다.

### Multi-Repository

Backend, Frontend, Common Library가 분리된 경우 현재 Task에서 함께 수정될 가능성이 높은 Repository만 Cloud Workspace에 제공한다.

무관한 Repository까지 연결하지 않는다.

---

# Part VIII. CI/CD와 Cloud Agent

## 14장. Task Queue와 Event-driven Cloud Agent

### 목적

개발자 PC가 아니라 Issue, CI, Review, Schedule 등의 이벤트가 Cloud Task를 생성하도록 설계한다.

### Task Source

- Issue
- CI Failure
- PR Review
- Scheduled Test
- Dependency Update
- Nightly Build

### 구조

```text
Task Queue
      ↓
Cloud Worker Pool
      ↓
Runner / Agent
```

```text
Git Push
→ CI
   ├─ PASS → 종료
   └─ FAIL → Cloud Agent → 수정 → 재검증 → PR
```

> 이벤트가 없으면 Agent도 실행하지 않는다.

Review Comment, Nightly Failure, Dependency Update도 같은 패턴으로 연결한다.

---

# Part IX. 실전 프로젝트

## 15장. campus-platform Cloud Agent Workflow 설계

### 목적

Java/Spring Boot 프로젝트에 앞 장의 원칙을 하나의 운영 모델로 통합한다.

### 실전 구조

```text
Developer / Local Agent
        ↓
Architecture / Task Split
        ↓
Git
        ↓
+----------------+----------------+----------------+
|                |                |                |
Cloud #1       Cloud #2        Cloud #3        Cloud #4
Unit Test      Integration     Docker Build      E2E
|                |                |                |
+----------------+----------------+----------------+
        ↓
Evidence Result
        ↓
PR
        ↓
Local Review / Internal Validation / Merge
```

### 적용 요소

- Java 21
- Spring Boot
- Gradle
- MyBatis
- PostgreSQL/Testcontainers
- Docker
- Playwright
- Migration Validation
- Prepared Environment
- Result Gateway
- Task Contract

---

## 16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

### 목적

작은 기능 하나를 요구사항 분석부터 PR, Review, Merge까지 따라간다.

### 실전 단계

- Local Requirement / Architecture
- Task Contract 작성
- Commit / Push
- Cloud Task 분산
- Unit / Integration / E2E / Docker 검증
- 실패 시 작은 Context로 Agent 분석
- Evidence Result 수집
- PR
- Local Review
- 내부망 검증
- Merge

UI가 포함되면 Demos over Diffs를 사용한다.

---

# Part X. Cloud Agent를 언제 쓰지 말아야 하는가

## 17장. Cloud가 항상 정답은 아니다

### 목적

Cloud Agent의 환경 준비 비용, Context 비용, 제약이 이점보다 큰 상황을 판단한다.

### Cloud에 부적합한 예

- 내부망 의존이 강함
- Repository를 Cloud에 제공할 수 없음
- 매우 큰 Context 필요
- 지속적 Human Steering 필요
- Cloud에서 재현 불가능
- 한 줄 수정처럼 Cloud overhead가 더 큼
- 여러 Agent가 같은 파일/Schema를 동시에 수정해야 함
- 전체 Architecture를 새로 설계하는 탐색적 작업

Local Fallback 기준을 제공한다.

---

# Part XI. Cloud Agent 중심 개발환경의 미래

## 18장. 다음 단계: Harness와 Orchestration

### 목적

Cloud Agent 활용에서 자연스럽게 이어지는 후속 방향만 짧게 소개한다.

### Advanced Topic

- Harness Engineering
- Agent Orchestration
- Agent-native Observability
- Brain / Hands
- Best-of-N
- Agent Platform

이 주제들을 독립 Agent Platform 이론으로 확장하지 않는다.

상세한 후속 연구는 `planning/future-topics.md`에 둔다.

---

# 범위에서 제외한 독립 장

다음은 현재 책의 독립 장으로 사용하지 않는다.

- Agent Memory Architecture
- Agent Security Platform
- Agent OS
- Agent Chaos Engineering
- Garbage Collector Agent
- Shadow Agent
- Canary Agent
- Agent Governance
- Agent-native Observability Platform
- General Multi-Agent Theory

# 상세 운영 설계

이번 목차에 반영된 Cloud Remote Worker 운영 원칙은 다음 문서에서 관리한다.

`planning/cloud-agent-remote-worker-model.md`

# 최종 독자 판단

이 책을 읽은 뒤 독자가 가장 먼저 할 수 있어야 하는 말은 다음이다.

> 이 작업은 Local에서 하고, 이 작업은 Cloud Agent에게 보내자.