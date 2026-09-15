# Table of Contents

> Phase 5 범위 재정렬본. 기존 Agent Platform 중심 확장을 축소하고 Cloud Agent 활용을 책의 중심축으로 복구한다.

# 책의 중심 질문

> 클라우드 에이전트를 왜 사용하고, 로컬 에이전트와 어떻게 조합하며, 어떤 작업을 맡기고, 토큰과 클라우드 컴퓨팅 자원을 어떻게 효율적으로 활용할 것인가?

# 전체 구성 원칙

책은 제품 사용 설명서가 아니다.

독자는 다음 순서로 판단 능력을 만든다.

```text
Cloud Agent가 무엇인가
        ↓
왜 Cloud를 사용하는가
        ↓
어떤 Task를 Cloud로 보낼 것인가
        ↓
Token과 Context를 어떻게 줄일 것인가
        ↓
Cloud CPU/RAM을 어떻게 활용할 것인가
        ↓
여러 Cloud Worker를 어떻게 병렬화할 것인가
        ↓
Local과 Cloud를 어떻게 조합할 것인가
        ↓
CI/CD와 어떻게 연결할 것인가
        ↓
실전 프로젝트에 어떻게 적용할 것인가
        ↓
언제 Cloud Agent를 쓰지 말아야 하는가
        ↓
향후 발전 방향
```

Java/Spring Boot 기반 `campus-platform`을 책 전체의 실전 예제로 사용한다.

---

# Part I. Cloud Agent 이해

## 1장. Coding Agent에서 Cloud Worker로

### 목적

Coding Agent를 IDE 보조 기능이나 Chat UI가 아니라 Repository를 읽고 명령을 실행하고 코드를 변경하는 작업 주체로 설명하고, Cloud Agent를 독립 실행환경을 가진 Remote Worker로 정의한다.

### 독자가 얻는 것

- Coding Agent와 일반 Chat LLM의 차이를 설명할 수 있다.
- Cloud Agent를 단순한 원격 AI 개발자가 아니라 실행환경을 가진 Worker로 이해할 수 있다.
- 책 전체에서 사용할 Cloud Worker 관점을 이해한다.

### 핵심 개념

- Coding Agent
- Repository
- Tool Use
- Remote Worker
- Execution Environment

### 선행 장

없음

### 실전 예제

`campus-platform` Repository를 Agent가 clone하고 build/test 명령을 실행할 수 있는 작업공간으로 본다.

---

## 2장. Local Agent와 Cloud Agent

### 목적

Local과 Cloud의 차이를 모델이 아니라 실행 위치, 네트워크, 격리, 컴퓨팅 자원, Human Steering 관점에서 설명한다.

### 독자가 얻는 것

- Local Agent가 적합한 작업과 Cloud Agent가 적합한 작업을 구분할 수 있다.
- 내부망, VPN, 사내 DB 같은 제약을 고려할 수 있다.
- Local과 Cloud를 경쟁 관계가 아니라 역할 분담 관계로 볼 수 있다.

### 핵심 개념

- Local Agent
- Cloud Agent
- Network Boundary
- Human Steering
- Isolation
- Hybrid Workflow

### 선행 장

1장

### 실전 예제

Architecture와 내부 HSM/Tibero 검증은 Local에 남기고 독립 Build/Test는 Cloud에 보내는 구조를 비교한다.

---

## 3장. Cloud Session, Container, Compute와 Token

### 목적

Cloud Agent의 CPU/RAM/Disk와 LLM Token을 서로 다른 자원으로 설명한다.

### 독자가 얻는 것

- 테스트 실행시간과 Token 사용량을 구분할 수 있다.
- LLM이 소비되는 시점과 Container가 계산하는 시점을 구분할 수 있다.
- Cloud Session을 독립 실행 노드로 이해할 수 있다.

### 핵심 개념

- Cloud Session
- Container / VM
- CPU
- RAM
- Disk
- LLM Token
- Brain / Hands 간단 모델

### 선행 장

1장, 2장

### 실전 예제

Cloud Container가 10,000개 테스트를 실행하고 Agent는 실패 3건만 읽는 흐름을 설명한다.

---

# Part II. 왜 Cloud Agent인가

## 4장. 독립 실행환경, 장시간 작업, 병렬성

### 목적

Cloud Agent의 핵심 가치를 더 많은 AI 호출이 아니라 개발자 PC와 분리된 실행환경, 장시간 작업 위임, 병렬 처리에서 찾는다.

### 독자가 얻는 것

- Cloud Agent가 실제로 유리한 이유를 설명할 수 있다.
- 로컬 컴퓨팅 자원을 점유하지 않고 장시간 작업을 위임할 수 있다.
- 독립 Branch/Container 기반 작업을 병렬화할 수 있다.

### 핵심 개념

- Isolation
- Long-running Task
- Parallel Execution
- Independent Workspace
- Remote Worker

### 선행 장

2장, 3장

### 실전 예제

Backend Test, Integration Test, Web E2E, Docker Build를 서로 다른 Cloud Worker에서 동시에 실행한다.

---

# Part III. 어떤 작업을 Cloud로 보낼 것인가

## 5장. Task Routing: Local인가 Cloud인가

### 목적

작업 특성을 보고 실행 위치를 선택하는 판단 기준을 만든다.

### 독자가 얻는 것

- Task마다 Local/Cloud 선택 기준을 적용할 수 있다.
- Cloud에 보내면 오히려 비효율적인 작업을 구분할 수 있다.
- 내부망, Context 크기, Human Steering, 실행시간, 병렬 가능성을 함께 판단할 수 있다.

### 핵심 개념

- Task Classification
- Context Size
- Internal Network
- Human Steering
- Independence
- Execution Cost

### 선행 장

2장, 4장

### 실전 예제

Architecture 설계, JWT bug fix, 전체 Unit Test, HSM 검증, 문서 수정 등 여러 Task를 Local/Cloud로 분류한다.

---

## 6장. Cloud에 보내기 좋은 개발 작업

### 목적

Cloud Agent와 Cloud Runner에 실제로 맡길 작업 유형을 구체적으로 설명한다.

### 독자가 얻는 것

- Build/Test 중심 작업을 Cloud에 분산할 수 있다.
- 반복 Refactoring과 작은 Bug Fix를 독립 Task로 만들 수 있다.
- PR Review와 Documentation도 Cloud Task로 설계할 수 있다.

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

### 선행 장

5장

### 실전 예제

`campus-platform`에서 모듈별 Test, Docker Build, Migration Validation을 독립 작업으로 분리한다.

---

# Part IV. Cloud Agent의 Token 절약

## 7장. 작은 Task와 작은 Context

### 목적

Cloud Agent가 Repository 전체를 반복해서 읽지 않도록 Task 범위와 Context 범위를 줄인다.

### 독자가 얻는 것

- Cloud Agent에 작은 Task Context Package를 전달할 수 있다.
- 관련 파일, 검증 명령, 변경 금지 범위를 명시할 수 있다.
- Progressive Context를 사용할 수 있다.

### 핵심 개념

- Task Scope
- Context Scope
- Relevant Files
- Forbidden Changes
- Acceptance Criteria
- Progressive Context
- AGENTS.md
- Task Contract의 최소 형태

### 선행 장

5장, 6장

### 실전 예제

`AuthServiceTest.expiredToken` 실패 수정에 필요한 파일 3개와 단일 테스트 명령만 Cloud Agent에 전달한다.

---

## 8장. Tool Output을 줄이고 필요한 결과만 보여주기

### 목적

Build/Test 로그 전체를 LLM에 전달하지 않고 필요한 실패 정보만 조회하게 한다.

### 독자가 얻는 것

- Tool Output이 Token 사용량에 미치는 영향을 이해한다.
- Result Filter / Result Gateway를 Cloud Token 절약에 사용할 수 있다.
- Retry, Token, Cost Budget을 적용할 수 있다.

### 핵심 개념

- Result Filter
- Result Gateway
- Raw Artifact
- Failure Summary
- Log Lookup
- Retry Limit
- Token Budget
- Cost Budget
- Failure Fingerprint

### 선행 장

3장, 7장

### 실전 예제

100MB build log는 Artifact로 저장하고 Agent에는 실패 테스트 3건만 먼저 전달한다.

---

# Part V. Cloud 컴퓨팅 자원 활용

## 9장. Prebuilt Environment, Cache, Snapshot

### 목적

Cloud Worker가 생성될 때마다 개발환경과 Dependency를 처음부터 구성하는 비용을 줄인다.

### 독자가 얻는 것

- 즉시 작업 가능한 Cloud 환경을 준비할 수 있다.
- 재사용 가능한 Cache와 초기화해야 하는 Runtime State를 구분할 수 있다.
- Snapshot과 Warm Worker의 적용 조건을 판단할 수 있다.

### 핵심 개념

- Prebuilt Environment
- Base Image
- Gradle / Maven Cache
- npm Cache
- Docker Layer Cache
- Playwright Browser Cache
- Snapshot
- Warm Environment
- Disposable Runtime State

### 선행 장

3장, 4장

### 실전 예제

Java 21, Gradle dependency, Node, Playwright, Docker CLI가 준비된 이미지에서 Worker를 시작한다.

---

## 10장. Cloud Agent를 Test Runner처럼 사용하기

### 목적

Cloud Agent의 실행환경을 Build/Test/Validation 노드로 적극 활용하되 LLM 호출과 실행 작업을 분리한다.

### 독자가 얻는 것

- 일반 Runner와 Cloud Agent를 역할별로 구분할 수 있다.
- 정상 경로에서는 LLM을 호출하지 않을 수 있다.
- Unit/Integration/E2E/Docker 작업을 여러 실행 노드로 분산할 수 있다.

### 핵심 개념

- Cloud Runner
- Agent Worker
- Deterministic Validation
- Runner-first
- Agent-on-failure
- Parallel Test
- Test Container

### 선행 장

6장, 8장, 9장

### 실전 예제

```text
Task
→ Runner
→ PASS: 종료
→ FAIL: Cloud Agent 분석
→ 수정
→ Runner 재검증
```

---

# Part VI. 여러 Cloud Agent 사용

## 11장. Branch, Worktree, Container로 작업 격리하기

### 목적

여러 Cloud Worker가 서로 간섭하지 않고 병렬 작업할 수 있게 한다.

### 독자가 얻는 것

- Task별 Branch/Worktree/Container를 설계할 수 있다.
- 같은 Working Directory를 공유할 때 발생하는 문제를 이해한다.
- 병렬화하면 안 되는 공통 파일/schema 작업을 구분할 수 있다.

### 핵심 개념

- Task Branch
- Git Worktree
- Independent Clone
- Cloud Container
- File Scope
- Shared File
- Migration Conflict

### 선행 장

4장, 5장

### 실전 예제

student, attendance, notification 작업을 서로 다른 Branch와 Worker에 배정한다.

---

## 12장. 병렬 Worker와 중복 Context 비용

### 목적

여러 Cloud Agent를 병렬로 사용할 때 Compute 이점과 LLM Context 중복 비용을 함께 관리한다.

### 독자가 얻는 것

- 독립 Task를 fan-out/fan-in할 수 있다.
- 여러 Agent가 같은 Repository를 반복 분석하는 낭비를 줄일 수 있다.
- Best-of-N을 필요한 어려운 Task에만 사용할 수 있다.

### 핵심 개념

- Fan-out
- Fan-in
- Context Duplication
- Parallel Compute
- Agent Count
- Best-of-N Advanced Pattern

### 선행 장

7장, 10장, 11장

### 실전 예제

Java 17→21 마이그레이션을 서비스별 Cloud Worker에 분산하고 마지막에 Full Validation을 수행한다.

---

# Part VII. Local + Cloud Hybrid Workflow

## 13장. Local과 Cloud를 하나의 개발 흐름으로 연결하기

### 목적

Local Agent와 Cloud Agent를 역할별로 연결한 실제 개발 Workflow를 만든다.

### 독자가 얻는 것

- Local에서 설계/분해하고 Cloud에서 독립 작업을 실행할 수 있다.
- 내부망 검증을 Local에 남기면서 Cloud 결과를 통합할 수 있다.
- PM 또는 개발자가 Task Routing을 수행할 수 있다.

### 핵심 개념

- Hybrid Workflow
- Handoff
- Local Integration
- Cloud Execution
- Internal Network
- Final Review

### 선행 장

5장, 10장, 12장

### 실전 예제

```text
Local
→ Architecture / Task 분리

Cloud
→ 구현 / Test / Docker / E2E

Local
→ Tibero/HSM/Jenkins 검증
→ 통합 / Review
```

---

# Part VIII. CI/CD와 Cloud Agent

## 14장. 실패와 이벤트가 Cloud Agent를 호출하게 만들기

### 목적

Agent를 항상 실행하지 않고 CI 실패, Review Comment, Nightly Failure 같은 이벤트에서만 Cloud Agent를 활성화한다.

### 독자가 얻는 것

- CI PASS 경로에서는 Agent 호출을 생략할 수 있다.
- CI Failure와 Review Comment를 Cloud Agent 작업으로 변환할 수 있다.
- Nightly 반복 작업을 설계할 수 있다.

### 핵심 개념

- Event-driven Agent
- CI Failure
- Review Comment
- Nightly Task
- Dependency Update
- Draft PR
- Revalidation

### 선행 장

8장, 10장, 13장

### 실전 예제

CI 실패 → Cloud Agent 수정 → Runner 재검증 → Draft PR 흐름을 구성한다.

---

# Part IX. 실전 프로젝트

## 15장. campus-platform Cloud Agent Workflow 설계

### 목적

앞 장의 원칙을 Java/Spring Boot 예제 프로젝트에 하나의 Workflow로 통합한다.

### 독자가 얻는 것

- 실제 Repository에서 Local/Cloud 역할을 설계할 수 있다.
- Cloud Task와 실행 명령을 정의할 수 있다.
- Prebuilt Environment, Cache, Result Gateway를 프로젝트에 연결할 수 있다.

### 핵심 개념

- Java 21
- Spring Boot
- Gradle
- MyBatis
- PostgreSQL/Testcontainers
- Docker
- Web E2E
- Task Routing

### 선행 장

1장부터 14장까지

### 실전 예제

기능 설계는 Local, Unit/Integration/Docker/E2E는 여러 Cloud Worker에 배정한다.

---

## 16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

### 목적

작은 기능 하나를 요구사항 분석부터 PR까지 실제 단계로 따라간다.

### 독자가 얻는 것

- Task 분해부터 Cloud 병렬 검증까지 전체 흐름을 실행할 수 있다.
- 실패했을 때 어떤 Context만 Cloud Agent에 추가할지 결정할 수 있다.
- 최종 Local Integration까지 수행할 수 있다.

### 핵심 개념

- Local Design
- Cloud Task
- Parallel Validation
- Failure Summary
- Agent Fix
- PR
- Final Integration

### 선행 장

15장

### 실전 예제

학생 출결 API 변경을 Local에서 설계하고 Cloud #1 Unit Test, #2 Integration, #3 Docker, #4 E2E로 검증한 뒤 결과를 통합한다.

---

# Part X. Cloud Agent를 언제 쓰지 말아야 하는가

## 17장. Cloud가 항상 정답은 아니다

### 목적

Cloud Agent 도입 비용과 제약이 이점보다 큰 상황을 판단한다.

### 독자가 얻는 것

- Cloud Agent를 사용하지 않아야 할 조건을 설명할 수 있다.
- 작은 수정, 큰 Context, 내부망, 보안, 재현 불가 문제를 구분할 수 있다.
- Local 작업으로 되돌리는 기준을 만들 수 있다.

### 핵심 개념

- Internal Network
- Large Context
- Human Steering
- Setup Overhead
- Security Restriction
- Non-reproducible Issue
- Local Fallback

### 선행 장

5장, 13장

### 실전 예제

HSM 장애, 미커밋 로컬 상태, 한 줄 수정, 전체 아키텍처 재설계 같은 작업을 Cloud에 보내지 않는 이유를 비교한다.

---

# Part XI. Cloud Agent 중심 개발환경의 미래

## 18장. 다음 단계: Harness, Orchestration, Agent-Native Environment

### 목적

현재 책의 범위를 넘어 발전할 수 있는 방향을 짧게 소개하되 Agent Platform 일반론으로 확장하지 않는다.

### 독자가 얻는 것

- Cloud Agent 활용이 향후 어떤 방향으로 발전할 수 있는지 이해한다.
- 현재 프로젝트에 필요한 범위와 후속 연구 주제를 구분할 수 있다.

### 핵심 개념

- Harness Engineering
- Agent Orchestration
- Agent-native Observability
- Brain / Hands
- Agent Platform
- Multi-agent

### 선행 장

1장부터 17장까지

### 실전 예제

현재 `campus-platform` Cloud Workflow를 기반으로 향후 자동 Task Scheduling이나 Agent-native 실행환경으로 확장할 수 있는 지점만 표시한다.

상세 설계는 `planning/future-topics.md`로 이동한다.

---

# 장 의존성 요약

```text
1 → 2 → 3 → 4
        ↓
        5 → 6 → 7 → 8
                ↓    ↓
                9 → 10
                     ↓
                11 → 12
                     ↓
                    13 → 14
                     ↓
                    15 → 16
                     ↓
                    17 → 18
```

# 범위에서 제외한 독립 장

다음 항목은 기존 설계에서 별도 장 후보였으나 현재 Cloud Agent 중심 책에서는 독립 장으로 사용하지 않는다.

- Agent-Native Development Environment 전체 아키텍처
- Agent Memory Architecture
- Agent Security Platform
- Agent Chaos Engineering
- Garbage Collector Agent
- Shadow Agent
- Canary Agent
- Agent Platform Governance
- Agent-native Observability Platform
- Multi-agent 조직론

필요한 내용은 18장에서 짧게 언급하고 상세 설계는 `planning/future-topics.md`에 보존한다.

# 최종 독자 판단

이 책을 읽은 뒤 독자가 가장 먼저 할 수 있어야 하는 말은 다음이다.

> 이 작업은 Local에서 하고, 이 작업은 Cloud Agent에게 보내자.
