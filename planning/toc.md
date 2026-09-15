# Table of Contents

> Phase 3 검증 반영본. 전체 책의 구조를 정의하며 본문은 포함하지 않는다.

# 전체 구성 원칙

책은 하나의 Java/Spring Boot 예제 프로젝트 `campus-platform`을 기본 축으로 사용한다.

각 장은 먼저 제품과 언어에 독립적인 문제와 설계 원칙을 설명하고, 그 다음 Java/Spring Boot 적용 예를 보여준다.

독자는 일반적인 프로젝트에서 시작해 다음 순서로 프로젝트를 변화시킨다.

```text
일반 프로젝트
  ↓
Agent가 이해할 수 있는 프로젝트
  ↓
재현 가능한 실행환경
  ↓
Agent Contract / Task Contract
  ↓
자동 테스트 / 자동 검증
  ↓
외부 의존성 격리
  ↓
병렬 작업 가능한 모듈 구조
  ↓
Local / Cloud Agent
  ↓
Trust Boundary / CI Gate
  ↓
Project Memory / Observability
  ↓
Agent Lifecycle / Roles
  ↓
PM Agent
  ↓
Hybrid Enterprise Agent System
  ↓
Agentic Development Governance
```

각 장은 가능하면 다음 구조를 따른다.

- 문제
- 원인
- 설계 원칙
- 구조
- 실전 적용
- 실패 사례
- 체크리스트

---

# Part I. AI Agent가 바꾸는 개발 방식

## 1장. 개발자가 코드를 작성하던 프로젝트에서 Agent가 작업하는 프로젝트로

### 목적

AI Coding Agent의 등장이 단순한 개발 도구 추가가 아니라 프로젝트 구조와 개발 프로세스의 전제를 어떻게 바꾸는지 정의한다.

### 독자가 얻는 것

- 기존 IDE 중심 개발 방식과 Agent 기반 개발 방식의 차이를 이해한다.
- 왜 기존 프로젝트가 Agent에게 불친절한지 설명할 수 있다.
- Agent Ready Software Engineering이 해결하려는 문제를 이해한다.

### 핵심 개념

- Human-driven Development
- Agent-driven Development
- Repository as Interface
- Machine-verifiable Development
- Agent Ready Software Engineering

### 선행 장

없음

### 실전 예제

기존 `campus-platform` 프로젝트를 Agent에게 처음 전달했을 때 발생하는 환경 설정, 문서 탐색, 테스트, 외부 시스템 의존 문제를 분석한다.

---

## 2장. Local Agent, Cloud Agent, Hybrid Agent

### 목적

Agent의 실행 위치에 따라 가능한 작업과 제한이 어떻게 달라지는지 정의한다.

### 독자가 얻는 것

- Local Agent와 Cloud Agent의 차이를 설명할 수 있다.
- 내부망, 개발자 PC, Cloud Sandbox 중 작업 위치를 선택할 수 있다.
- Hybrid Agent 구조가 필요한 조건을 판단할 수 있다.

### 핵심 개념

- Local Agent
- Cloud Agent
- Hybrid Agent
- Sandbox
- Repository-based Execution
- Network Boundary

### 선행 장

1장

### 실전 예제

`campus-platform`에서 코드 리팩터링은 Cloud Agent에, Tibero/HSM/Jenkins 확인은 Local Agent에 배정하는 구조를 설계한다.

---

## 3장. Agent Ready 프로젝트의 기준

### 목적

Agent Ready를 추상적인 표현이 아니라 확인 가능한 프로젝트 특성으로 정의한다.

### 독자가 얻는 것

- 프로젝트의 Agent Ready 수준을 평가할 수 있다.
- Agent 도입 전에 해결해야 할 구조적 문제를 찾을 수 있다.
- 이후 장에서 사용할 공통 평가 기준을 이해한다.

### 핵심 개념

- Reproducibility
- Discoverability
- Executability
- Testability
- Verifiability
- Isolation
- Parallelizability
- Observability
- Security Boundary

### 선행 장

1장, 2장

### 실전 예제

일반적인 Spring Boot 프로젝트를 위 기준으로 평가하여 개선 항목을 도출한다.

---

# Part II. Agent가 이해하고 실행할 수 있는 프로젝트

## 4장. Repository as Interface: Agent가 이해할 수 있는 저장소

### 목적

Repository 자체를 Agent가 탐색하고 판단하는 인터페이스로 보고 구조, 명명, 문서 위치와 context 탐색 비용을 설계한다.

### 독자가 얻는 것

- Agent가 저장소 구조를 빠르게 파악하도록 만들 수 있다.
- canonical source와 파생 문서를 구분할 수 있다.
- 거대한 파일, 숨겨진 규칙, generated file이 만드는 탐색 비용을 줄일 수 있다.

### 핵심 개념

- Repository Layout
- Discoverability
- Canonical Source
- Progressive Disclosure
- Context Budget
- Naming
- Generated File
- Migration Location
- Change Locality

### 선행 장

3장

### 실전 예제

`campus-platform`의 최상위 구조와 모듈별 진입점을 정리하고, Agent가 기능 위치와 변경 범위를 추론하기 쉬운 형태로 재구성한다.

---

## 5장. Agent Contract: 프로젝트를 설명하는 계약

### 목적

대화 프롬프트에 의존하지 않고 Repository 자체가 Agent에게 프로젝트 규칙을 설명하도록 만든다.

### 독자가 얻는 것

- Agent Contract에 포함할 최소 정보를 정의할 수 있다.
- README, AGENTS.md, CLAUDE.md, architecture.md 등의 역할을 구분할 수 있다.
- 제품별 instruction file과 프로젝트의 canonical rule을 분리할 수 있다.
- 문서 중복과 규칙 충돌을 줄일 수 있다.

### 핵심 개념

- Agent Contract
- AGENTS.md
- CLAUDE.md
- README.md
- Architecture Document
- Development Guide
- Testing Guide
- ADR
- Source of Truth
- Instruction Precedence
- Product Adapter Document

### 선행 장

4장

### 실전 예제

`campus-platform`의 프로젝트 목적, 모듈 경계, 빌드·테스트·검증 명령, 금지 규칙을 canonical contract로 작성하고 제품별 instruction file은 이를 참조하도록 설계한다.

---

## 6장. Task Contract: Agent에게 일을 넘기는 방법

### 목적

Agent에게 자유 형식 지시를 전달하는 대신 실행 가능한 작업 계약을 정의한다.

### 독자가 얻는 것

- 작업의 범위와 완료 조건을 명확하게 전달할 수 있다.
- Agent의 과도한 변경을 줄일 수 있다.
- 작업 의존성과 병렬 실행 가능성을 판단할 수 있다.

### 핵심 개념

- Goal
- Scope
- Allowed Files
- Forbidden Changes
- Dependencies
- Acceptance Criteria
- Verification
- Deliverables
- Task Dependency

### 선행 장

5장

### 실전 예제

`학생 조회 API 추가` 작업을 Task Contract 형태로 변환하고 자유 형식 프롬프트와 비교한다.

---

## 7장. 재현 가능한 개발환경

### 목적

Agent가 새로운 환경에서도 최소한의 명령으로 프로젝트를 실행할 수 있게 한다.

### 독자가 얻는 것

- 개발자 개인 PC에 숨겨진 환경 의존성을 제거할 수 있다.
- Local, CI, Cloud Agent의 실행 환경 차이를 줄일 수 있다.
- 자동 Setup의 범위를 결정할 수 있다.

### 핵심 개념

- Reproducible Environment
- Bootstrap
- Dependency Pinning
- Container
- Dev Container
- Environment Variables
- Toolchain Version
- Idempotent Setup

### 선행 장

4장, 5장

### 실전 예제

Java 버전, Gradle, Docker, 환경변수, 테스트 의존성을 자동 구성하는 `setup` 절차를 설계한다.

---

## 8장. Agent 실행 인터페이스: setup, test, verify

### 목적

Agent와 사람이 동일한 명령으로 프로젝트를 조작할 수 있는 표준 실행 인터페이스를 설계한다.

### 독자가 얻는 것

- Agent가 빌드 도구와 CI 구조를 추측하지 않게 만들 수 있다.
- 로컬 명령과 CI 명령을 통일할 수 있다.
- 검증 진입점을 하나로 수렴시킬 수 있다.

### 핵심 개념

- setup
- build
- test
- verify
- Makefile
- scripts/
- CI Parity
- Exit Code
- Deterministic Command

### 선행 장

7장

### 실전 예제

`./scripts/setup.sh`, `./scripts/test.sh`, `./scripts/verify.sh` 또는 동등한 인터페이스를 설계한다.

---

# Part III. Agent가 스스로 검증할 수 있는 아키텍처

## 9장. 외부 시스템 없이 테스트 가능한 구조

### 목적

DB, Redis, Kafka, HSM, 외부 API가 없어도 대부분의 작업을 Agent가 검증할 수 있도록 애플리케이션 경계를 설계한다.

### 독자가 얻는 것

- 외부 시스템 의존이 Agent 실행을 막는 이유를 이해한다.
- Fake, Mock, Testcontainers의 역할을 구분할 수 있다.
- Adapter 경계를 테스트 가능한 형태로 설계할 수 있다.

### 핵심 개념

- Unit Test
- Integration Test
- Contract Test
- E2E Test
- Testcontainers
- Mock Server
- Fake Adapter
- Ports and Adapters

### 선행 장

7장, 8장

### 실전 예제

Tibero, Redis, Kafka, HSM, 외부 학사 API를 각각 Testcontainer, Fake 또는 Mock으로 대체한다.

---

## 10장. 자동 검증과 Agent Definition of Done

### 목적

Agent의 `완료했습니다`라는 설명 대신 기계적으로 판단 가능한 완료 조건을 만든다.

### 독자가 얻는 것

- 자동 검증 파이프라인을 설계할 수 있다.
- 작업 완료 여부를 PASS/FAIL로 판단할 수 있다.
- 신규 기능과 기존 기능의 회귀를 동시에 검증할 수 있다.

### 핵심 개념

- Definition of Done
- Format
- Lint
- Compile
- Unit Test
- Integration Test
- Security Check
- Secret Detection
- Diff Validation
- Exit Code

### 선행 장

8장, 9장

### 실전 예제

`verify.sh`가 코드 형식, 컴파일, 테스트, Secret, 변경 범위를 검사하도록 설계한다.

---

## 11장. Architecture Rule을 코드로 검증하기

### 목적

문서에 적힌 아키텍처 규칙을 일부 실행 가능한 테스트로 전환한다.

### 독자가 얻는 것

- 의존성 방향과 모듈 경계를 자동 검증할 수 있다.
- 문서 규칙과 실제 코드 간 차이를 줄일 수 있다.
- Agent가 구조를 훼손하는 변경을 조기에 차단할 수 있다.

### 핵심 개념

- Architecture Test
- Dependency Rule
- Package Rule
- Module Rule
- Forbidden Dependency
- Static Analysis

### 선행 장

4장, 10장

### 실전 예제

`attendance` 모듈이 `student` 내부 구현에 직접 의존하지 못하도록 규칙을 정의한다.

---

# Part IV. 여러 Agent가 동시에 일하는 프로젝트

## 12장. 병렬 Agent 개발을 위한 모듈 경계

### 목적

모듈화를 유지보수 목적뿐 아니라 병렬 Agent 작업의 충돌 제어 수단으로 사용한다.

### 독자가 얻는 것

- 병렬 작업 가능성을 기준으로 모듈 경계를 평가할 수 있다.
- shared file과 common module이 만드는 충돌을 줄일 수 있다.
- 작업을 독립적으로 분할하기 쉬운 구조를 만들 수 있다.

### 핵심 개념

- Bounded Context
- Module Ownership
- Change Locality
- Parallelizability
- Shared State
- Common Module
- Migration Conflict

### 선행 장

4장, 11장

### 실전 예제

`auth`, `attendance`, `notification`을 세 Agent가 동시에 변경할 때 충돌 지점을 분석한다.

---

## 13장. Git, Worktree, Branch, Cloud Sandbox

### 목적

여러 Agent가 동일 Repository를 병렬로 변경하는 실행 전략을 설계한다.

### 독자가 얻는 것

- branch per agent, worktree, independent clone의 차이를 이해한다.
- Local Agent와 Cloud Agent에 맞는 Git 격리 전략을 선택할 수 있다.
- Merge와 충돌 해결 흐름을 설계할 수 있다.

### 핵심 개념

- Task Branch
- Git Worktree
- Independent Clone
- Cloud Sandbox
- Commit Policy
- Merge Strategy
- Conflict Handling

### 선행 장

6장, 12장

### 실전 예제

여러 Task Branch와 Worktree를 만들어 세 작업을 독립 실행하고 결과를 통합하는 흐름을 설계한다.

---

## 14장. Trust Boundary: Agent에게 어디까지 권한을 줄 것인가

### 목적

Agent의 실행 편의성과 시스템 보안 사이의 경계를 설계한다.

### 독자가 얻는 것

- Local과 Cloud Agent의 권한 차이를 설계할 수 있다.
- Secret, Network, Repository, Production 접근을 분리할 수 있다.
- 비신뢰 입력이 Agent 행동에 영향을 주는 위험을 통제할 수 있다.
- Agent별 최소 권한 원칙을 적용할 수 있다.

### 핵심 개념

- Least Privilege
- Secret Boundary
- Network Boundary
- Repository Permission
- PR Permission
- Merge Permission
- Deployment Permission
- Production Access
- Untrusted Input
- Prompt Injection
- Secret Exfiltration
- Tool Allowlist
- Command Execution Boundary

### 선행 장

2장, 13장

### 실전 예제

Worker Agent는 feature branch 쓰기만 허용하고, Integrator만 merge 권한을 갖게 한다. 외부 문서와 Issue 내용은 비신뢰 입력으로 취급하고 Secret 및 명령 실행 경계를 분리한다.

---

## 15장. CI/CD Gate와 Agent 권한 연결

### 목적

Agent의 변경이 실제 Merge와 배포로 이어질 때 필요한 기계적 보호 장치와 승인 단계를 설계한다.

### 독자가 얻는 것

- Agent와 CI의 검증 책임을 구분할 수 있다.
- PR, Merge, Deploy 권한 단계를 설계할 수 있다.
- Production 배포를 Agent에게 직접 허용할지 판단할 수 있다.

### 핵심 개념

- CI Gate
- PR Check
- Merge Gate
- Deployment Gate
- Approval
- Environment Protection
- Audit Log
- Separation of Duties

### 선행 장

10장, 14장

### 실전 예제

Worker Agent → PR → CI Verify → Reviewer → Merge → Jenkins Deploy 흐름을 설계한다.

---

# Part V. 장기 작업과 Project Memory

## 16장. Project Memory와 Long Running Task

### 목적

Agent의 대화 컨텍스트와 프로젝트가 장기간 유지해야 하는 정보를 분리하고, 몇 시간 또는 며칠 동안 이어지는 작업을 중단·재개할 수 있게 한다.

### 독자가 얻는 것

- 세션 종료 후에도 유지되어야 할 정보를 판단할 수 있다.
- 여러 Agent가 동일한 프로젝트 상태를 공유할 수 있다.
- 작업 중단과 재개가 가능한 상태 구조를 만들 수 있다.
- Agent 교체 후에도 작업을 이어갈 수 있다.

### 핵심 개념

- Agent Memory
- Project Memory
- Decision Log
- ADR
- Task State
- Progress
- Failed Attempts
- Test Result
- Next Task
- Checkpoint
- Resume
- Idempotency
- Source of Truth

### 선행 장

5장, 6장, 10장

### 실전 예제

`docs/decisions`, `docs/tasks`, `docs/progress`, `docs/knowledge`와 작업 상태 파일을 사용해 대규모 모듈 분리 작업을 여러 세션에 걸쳐 이어가는 구조를 설계한다.

---

## 17장. Agent Observability와 Quality Evaluation

### 목적

여러 Agent의 실행 상태와 결과물의 품질을 사람이 추측하지 않고 관찰하고 평가할 수 있게 한다.

### 독자가 얻는 것

- Agent 상태 모델을 정의할 수 있다.
- 테스트 통과와 작업 품질을 구분할 수 있다.
- 실행 시간, 재시도 횟수, 비용과 자원 사용량을 관찰할 수 있다.
- PM Agent가 Worker 상태와 결과를 판단하는 기준을 만들 수 있다.

### 핵심 개념

- queued
- running
- blocked
- failed
- verifying
- completed
- Requirement Coverage
- Change Scope
- Unnecessary Changes
- Retry Count
- Execution Time
- Cost
- Resource Usage
- Audit Trail

### 선행 장

10장, 15장, 16장

### 실전 예제

Worker Agent의 작업 상태, 검증 결과, 변경 파일, 실패 원인, 재시도 횟수와 실행 비용을 수집하는 구조를 설계한다.

---

# Part VI. Agent Orchestration

## 18장. Agent Lifecycle과 역할 분리

### 목적

Agent를 작업 단위의 일시적 실행 자원으로 정의하고, 작업 복잡도에 따라 Planner, Worker, Tester, Reviewer 등의 역할을 선택하는 방법을 설계한다.

### 독자가 얻는 것

- 작업에 필요한 Agent 수와 역할을 결정할 수 있다.
- Agent 생성, 실행, 검증, 종료 흐름을 설계할 수 있다.
- 구현 Agent와 검증 Agent를 분리할 수 있다.
- 역할을 과도하게 늘리지 않고 필요한 수준으로 구성할 수 있다.

### 핵심 개념

- Task Analysis
- Agent Allocation
- Spawn
- Execute
- Verify
- Terminate
- Planner
- Worker
- Tester
- Reviewer
- Security Reviewer
- Integrator
- Separation of Duties

### 선행 장

6장, 13장, 17장

### 실전 예제

`학생 출결 기능` 작업을 분석해 Backend Worker와 Tester를 생성하고 Reviewer가 결과를 검토한 뒤 작업이 끝나면 Agent를 종료한다.

---

## 19장. PM Agent: 여러 Agent를 운영하는 Agent

### 목적

사람이 모든 Worker를 직접 지시하는 구조에서 프로젝트 상태를 읽고 작업을 분해·할당·검증하는 PM Agent 구조로 확장한다.

### 독자가 얻는 것

- PM Agent의 책임 범위를 정의할 수 있다.
- Task Queue와 의존성 기반 작업 배분 구조를 설계할 수 있다.
- 필요한 Agent 수를 동적으로 결정할 수 있다.
- 실패 작업의 재할당과 완료 판단 흐름을 설계할 수 있다.

### 핵심 개념

- PM Agent
- Project State
- Task Queue
- Dependency Graph
- Task Decomposition
- Agent Allocation
- Scheduling
- Result Collection
- Retry
- Completion Decision

### 선행 장

16장, 17장, 18장

### 실전 예제

PM Agent가 프로젝트 상태를 읽고 다음 작업 목록을 생성한 뒤 의존성을 분석해 필요한 Agent 수를 계산하고 병렬 배정한다.

---

## 20장. 실패, 재시도, 충돌, 통합

### 목적

Agent 시스템을 정상 흐름만이 아니라 실패를 전제로 설계한다.

### 독자가 얻는 것

- 실패 유형을 구분하고 재시도 여부를 판단할 수 있다.
- 동일 실패의 무한 반복을 방지할 수 있다.
- 병렬 작업 결과를 안전하게 통합할 수 있다.
- 사람이 개입해야 하는 조건을 정의할 수 있다.

### 핵심 개념

- Retry Policy
- Failure Classification
- Blocked Task
- Conflict
- Rebase
- Integration
- Rollback
- Human Escalation
- Retry Budget

### 선행 장

13장, 16장, 19장

### 실전 예제

테스트 실패, Git 충돌, 외부 시스템 접근 실패, 요구사항 불명확 상황을 각각 다른 방식으로 처리한다.

---

# Part VII. 기업 환경과 Agentic Development 운영

## 21장. VPN과 내부망이 있는 Hybrid Agent 시스템

### 목적

Cloud Agent가 모든 시스템에 접근할 수 없다는 현실을 전제로 기업 프로젝트의 작업 경계와 handoff를 설계한다.

### 독자가 얻는 것

- Cloud Agent에 보내도 되는 작업과 Local Agent가 수행해야 하는 작업을 구분할 수 있다.
- 내부망 의존 작업을 Agent 시스템에 포함할 수 있다.
- Cloud와 Local Agent 사이의 결과 전달 구조를 설계할 수 있다.

### 핵심 개념

- VPN
- Internal Git
- Nexus
- Tibero / Oracle
- Redis
- Kafka
- HSM
- Jenkins
- Internal API
- Hybrid Execution
- Handoff

### 선행 장

2장, 14장, 15장, 19장

### 실전 예제

Cloud Agent는 독립 코드 개발과 테스트를 수행하고 Local Agent는 Tibero, HSM, Jenkins 통합 검증을 수행한 뒤 PM Agent가 결과를 통합한다.

---

## 22장. Full Agentic Development로의 진화와 Governance

### 목적

앞 장들의 원칙을 하나의 프로젝트 개발 프로세스로 통합하고, 자동화 수준을 단계적으로 높이면서 사람의 책임과 운영 기준을 유지하는 방법을 정리한다.

### 독자가 얻는 것

- 기존 프로젝트를 단계적으로 Agent Ready 프로젝트로 전환할 수 있다.
- 모든 기능을 한 번에 자동화하지 않고 성숙도에 따라 도입할 수 있다.
- 사람과 Agent의 최종 책임 경계를 설계할 수 있다.
- 비용, 권한, 감사, 모델 선택과 자동화 수준을 Governance 관점에서 관리할 수 있다.

### 핵심 개념

- Agent Ready Maturity
- Incremental Adoption
- Human-in-the-loop
- Human-on-the-loop
- Full Agentic Development
- Governance
- Cost Budget
- Resource Budget
- Auditability
- Accountability
- Model Independence

### 선행 장

1장부터 21장까지

### 실전 예제

`campus-platform`이 일반적인 Spring Boot 프로젝트에서 Agent Contract, 재현 가능한 환경, 자동 검증, 병렬 Agent, Project Memory, PM Agent, Hybrid Agent 구조를 갖춘 프로젝트로 변하는 전체 과정을 정리하고 단계별 도입 기준을 정의한다.

---

# 부록 후보

## 부록 A. Agent Contract 템플릿

- README.md
- AGENTS.md
- 제품별 instruction adapter
- architecture.md
- development.md
- testing.md

## 부록 B. Task Contract 템플릿

- Goal
- Scope
- Allowed Files
- Forbidden Changes
- Dependencies
- Acceptance Criteria
- Verification
- Deliverables

## 부록 C. Agent Ready 체크리스트

새 프로젝트와 기존 프로젝트를 평가할 수 있는 점검표를 제공한다.

## 부록 D. 제품별 구현 사례

Claude Code, Codex, GitHub Copilot Coding Agent, Devin 등의 현재 기능을 책의 일반 원칙과 연결해 비교한다.

제품 기능은 변경될 수 있으므로 본문보다 부록 또는 별도 Research 문서에 가깝게 유지한다.

---

# 장 의존성 요약

```text
1 → 2 → 3
        ↓
        4 → 5 → 6 → 7 → 8
        ↓               ↓
       12 ← 11 ← 10 ← 9
        ↓       ↓
       13 → 14 → 15
        ↓         ↓
        └────→ 16 → 17 → 18 → 19 → 20
                           ↓          ↓
                           └────→ 21 → 22
```

# Phase 3에서 확정한 사항

- 전체 목차는 7개 Part, 22개 장으로 구성한다.
- 하나의 `campus-platform` 예제를 책 전체의 기본 축으로 사용한다.
- 4장은 Repository의 탐색성과 context 구조에 집중하고, 12장은 병렬 변경과 충돌 제어에 집중한다.
- `Task Contract`와 `Trust Boundary`는 독립 장으로 유지한다.
- `Agent Memory와 Project Memory`, `Long Running Agent`는 하나의 장으로 통합한다.
- `Agent Lifecycle`, `Planner/Worker/Tester/Reviewer`는 하나의 장으로 통합한다.
- CI/CD Gate는 기업 환경이 아니라 일반 Trust Boundary의 연장선으로 다룬다.
- Trust Boundary에는 Untrusted Input과 Prompt Injection을 포함한다.
- Agent Observability에는 품질뿐 아니라 실행 시간, 재시도, 비용과 자원 사용량을 포함한다.
- 제품별 기능 설명은 본문의 중심이 아니라 사례 또는 부록으로 둔다.
- Java/Spring Boot는 주요 실전 예제지만 일반 원칙보다 먼저 등장하지 않는다.
- PM Agent는 Repository, Verification, Parallel Development, Memory, Observability를 설명한 뒤 도입한다.
- 기업 내부망과 Hybrid Agent는 일반 원칙을 설명한 이후 적용 사례로 다룬다.
