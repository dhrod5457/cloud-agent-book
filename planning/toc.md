# Table of Contents

> Phase 2 초안. 이 문서는 전체 책의 구조를 설계하기 위한 목차이며 본문을 포함하지 않는다. Phase 3에서 중복, 누락, 순서, 난이도, 제품 편향을 검증한 뒤 수정할 수 있다.

# 전체 구성 원칙

책은 하나의 Java/Spring Boot 예제 프로젝트 `campus-platform`을 기본 축으로 사용한다.

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
Trust Boundary
  ↓
Project Memory / Observability
  ↓
Planner / Worker / Reviewer
  ↓
PM Agent
  ↓
Hybrid Enterprise Agent System
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

### 선행 장

1장, 2장

### 실전 예제

일반적인 Spring Boot 프로젝트를 위 기준으로 평가하여 개선 항목을 도출한다.

---

# Part II. Agent가 이해하고 실행할 수 있는 프로젝트

## 4장. Agent-friendly Repository 설계

### 목적

Agent가 저장소를 탐색하고 변경 범위를 판단하기 쉬운 프로젝트 구조를 설계한다.

### 독자가 얻는 것

- 디렉터리와 모듈 구조가 Agent 작업에 미치는 영향을 이해한다.
- 거대한 파일, 공통 모듈, generated file, migration이 만드는 문제를 줄일 수 있다.
- 사람이 읽기 좋은 구조와 Agent가 작업하기 좋은 구조의 공통점을 이해한다.

### 핵심 개념

- Repository Layout
- Module Boundary
- Dependency Direction
- Shared File
- Generated File
- Migration Ownership
- Discoverability

### 선행 장

3장

### 실전 예제

`campus-platform`을 `auth`, `student`, `attendance`, `notification`, `integration`, `common` 모듈로 나누고 변경 영향 범위를 분석한다.

---

## 5장. Agent Contract: 프로젝트를 설명하는 계약

### 목적

대화 프롬프트에 의존하지 않고 Repository 자체가 Agent에게 프로젝트 규칙을 설명하도록 만든다.

### 독자가 얻는 것

- Agent Contract에 포함할 최소 정보를 정의할 수 있다.
- README, AGENTS.md, CLAUDE.md, architecture.md 등의 역할을 구분할 수 있다.
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

### 선행 장

4장

### 실전 예제

`campus-platform`의 프로젝트 목적, 모듈 경계, 빌드·테스트·검증 명령, 금지 규칙을 Agent Contract로 작성한다.

---

## 6장. Task Contract: Agent에게 일을 넘기는 방법

### 목적

Agent에게 자유 형식 지시를 전달하는 대신 실행 가능한 작업 계약을 정의한다.

### 독자가 얻는 것

- 작업의 범위와 완료 조건을 명확하게 전달할 수 있다.
- Agent의 과도한 변경을 줄일 수 있다.
- 병렬 작업에서 Agent 간 경계를 정의할 수 있다.

### 핵심 개념

- Goal
- Scope
- Allowed Files
- Forbidden Changes
- Dependencies
- Acceptance Criteria
- Verification
- Deliverables

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
- Architecture Test
- Security Check
- Secret Detection
- Diff Validation

### 선행 장

8장, 9장

### 실전 예제

`verify.sh`가 코드 형식, 컴파일, 테스트, 아키텍처 규칙, Secret, 변경 범위를 검사하도록 설계한다.

---

## 11장. Architecture Rule을 코드로 검증하기

### 목적

문서에 적힌 아키텍처 규칙을 Agent가 우회하지 못하도록 일부 규칙을 실행 가능한 테스트로 전환한다.

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

PM Agent가 세 개의 Task Branch와 Worktree를 생성하고 결과를 통합하는 흐름을 설계한다.

---

## 14장. Trust Boundary: Agent에게 어디까지 권한을 줄 것인가

### 목적

Agent의 실행 편의성과 시스템 보안 사이의 경계를 설계한다.

### 독자가 얻는 것

- Local과 Cloud Agent의 권한 차이를 설계할 수 있다.
- Secret, Network, Repository, Production 접근을 분리할 수 있다.
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

### 선행 장

2장, 13장

### 실전 예제

Worker Agent는 feature branch 쓰기만 허용하고, Integrator만 main merge 권한을 갖는 정책을 설계한다.

---

# Part V. 장기 작업과 Project Memory

## 15장. Agent Memory와 Project Memory

### 목적

Agent의 대화 컨텍스트와 프로젝트가 장기간 유지해야 하는 정보를 분리한다.

### 독자가 얻는 것

- 세션 종료 후에도 유지되어야 할 정보를 판단할 수 있다.
- Git과 문서를 Project Memory로 사용할 수 있다.
- 여러 Agent가 동일한 프로젝트 상태를 공유할 수 있다.

### 핵심 개념

- Agent Memory
- Project Memory
- Decision Log
- ADR
- Task State
- Progress
- Knowledge
- Source of Truth

### 선행 장

5장, 6장

### 실전 예제

`docs/decisions`, `docs/tasks`, `docs/progress`, `docs/knowledge` 구조를 설계한다.

---

## 16장. Long Running Agent와 중단 가능한 작업

### 목적

몇 시간 또는 며칠 동안 이어지는 작업을 세션 하나의 컨텍스트에 의존하지 않고 지속하는 방법을 설계한다.

### 독자가 얻는 것

- 작업 중단과 재개가 가능한 상태 구조를 만들 수 있다.
- 실패한 시도와 다음 작업을 명시적으로 기록할 수 있다.
- Agent 교체 후에도 작업을 이어갈 수 있다.

### 핵심 개념

- Checkpoint
- Task State
- Progress
- Failed Attempts
- Test Result
- Next Task
- Resume
- Idempotency

### 선행 장

15장

### 실전 예제

대규모 모듈 분리 작업을 여러 세션에 걸쳐 이어가는 상태 파일을 설계한다.

---

## 17장. Agent Observability와 Evaluation

### 목적

여러 Agent의 상태와 작업 품질을 사람이 추측하지 않고 관찰하고 평가할 수 있게 한다.

### 독자가 얻는 것

- Agent 상태 모델을 정의할 수 있다.
- 테스트 통과와 작업 품질을 구분할 수 있다.
- PM Agent가 Worker 상태를 판단하는 기준을 만들 수 있다.

### 핵심 개념

- queued
- running
- blocked
- failed
- verifying
- completed
- Evaluation
- Requirement Coverage
- Change Scope
- Unnecessary Changes

### 선행 장

10장, 15장, 16장

### 실전 예제

Worker Agent의 작업 상태, 검증 결과, 변경 파일, 실패 원인을 PM Agent가 수집하는 구조를 설계한다.

---

# Part VI. Agent Orchestration

## 18장. Agent Lifecycle: 생성에서 종료까지

### 목적

Agent를 항상 실행하는 프로세스가 아니라 작업 단위의 일시적 실행 자원으로 정의한다.

### 독자가 얻는 것

- 작업에 필요한 Agent 수를 결정할 수 있다.
- Agent 생성, 실행, 검증, 종료 흐름을 설계할 수 있다.
- 불필요한 장기 Agent를 줄일 수 있다.

### 핵심 개념

- Task Analysis
- Agent Allocation
- Spawn
- Execute
- Verify
- Merge
- Terminate
- Retry

### 선행 장

6장, 13장, 17장

### 실전 예제

한 기능 개발을 위해 Backend Worker, Test Worker를 생성하고 작업 완료 후 종료하는 흐름을 설계한다.

---

## 19장. Planner, Worker, Tester, Reviewer

### 목적

한 Agent가 계획, 구현, 테스트, 리뷰를 모두 수행할 때 발생하는 자기검증 문제를 역할 분리로 다룬다.

### 독자가 얻는 것

- 작업 복잡도에 따라 필요한 역할을 선택할 수 있다.
- 구현 Agent와 검증 Agent를 분리할 수 있다.
- 역할을 과도하게 늘리지 않고 필요한 수준으로 구성할 수 있다.

### 핵심 개념

- Planner
- Worker
- Tester
- Reviewer
- Security Reviewer
- Integrator
- Separation of Duties

### 선행 장

18장

### 실전 예제

`학생 출결 기능`을 Planner → Backend Worker → Tester → Reviewer 흐름으로 실행한다.

---

## 20장. PM Agent: 여러 Agent를 운영하는 Agent

### 목적

사람이 모든 Worker를 직접 지시하는 구조에서 프로젝트 상태를 읽고 작업을 분해·할당·검증하는 PM Agent 구조로 확장한다.

### 독자가 얻는 것

- PM Agent의 책임 범위를 정의할 수 있다.
- Task Queue와 의존성 기반 작업 배분 구조를 설계할 수 있다.
- 실패 작업의 재할당과 완료 판단 흐름을 설계할 수 있다.

### 핵심 개념

- PM Agent
- Project State
- Task Queue
- Dependency Graph
- Agent Allocation
- Scheduling
- Result Collection
- Retry
- Completion Decision

### 선행 장

17장, 18장, 19장

### 실전 예제

PM Agent가 프로젝트 상태를 읽고 다음 작업 목록을 생성한 뒤 필요한 Agent 수를 계산하고 병렬 배정한다.

---

## 21장. 실패, 재시도, 충돌, 통합

### 목적

Agent 시스템을 정상 흐름만이 아니라 실패를 전제로 설계한다.

### 독자가 얻는 것

- 실패 유형을 구분하고 재시도 여부를 판단할 수 있다.
- 동일 실패의 무한 반복을 방지할 수 있다.
- 병렬 작업 결과를 안전하게 통합할 수 있다.

### 핵심 개념

- Retry Policy
- Failure Classification
- Blocked Task
- Conflict
- Rebase
- Integration
- Rollback
- Human Escalation

### 선행 장

13장, 16장, 20장

### 실전 예제

테스트 실패, Git 충돌, 외부 시스템 접근 실패, 요구사항 불명확 상황을 각각 다른 방식으로 처리한다.

---

# Part VII. 기업 환경의 Hybrid Agent 시스템

## 22장. VPN과 내부망이 있는 프로젝트

### 목적

Cloud Agent가 모든 시스템에 접근할 수 없다는 현실을 전제로 기업 프로젝트의 작업 경계를 설계한다.

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

### 선행 장

2장, 14장, 20장

### 실전 예제

Cloud Agent는 독립 코드 개발과 테스트를 수행하고 Local Agent는 Tibero, HSM, Jenkins 통합 검증을 수행한다.

---

## 23장. CI/CD와 Agent 권한 연결

### 목적

Agent 작업 결과가 실제 CI/CD와 배포 과정으로 이어질 때 필요한 보호 장치를 설계한다.

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

### 선행 장

10장, 14장, 22장

### 실전 예제

Worker Agent → PR → CI Verify → Reviewer → Merge → Jenkins Deploy 흐름을 설계한다.

---

## 24장. Full Agentic Development로의 진화

### 목적

앞 장들의 원칙을 하나의 프로젝트 개발 프로세스로 통합한다.

### 독자가 얻는 것

- 기존 프로젝트를 단계적으로 Agent Ready 프로젝트로 전환할 수 있다.
- 모든 기능을 한 번에 자동화하지 않고 성숙도에 따라 도입할 수 있다.
- 사람과 Agent의 최종 책임 경계를 설계할 수 있다.

### 핵심 개념

- Agent Ready Maturity
- Incremental Adoption
- Human-in-the-loop
- Human-on-the-loop
- Full Agentic Development
- Governance

### 선행 장

1장부터 23장까지

### 실전 예제

`campus-platform`이 일반적인 Spring Boot 프로젝트에서 Agent Contract, 자동 검증, 병렬 Agent, PM Agent, Hybrid Agent 구조를 갖춘 프로젝트로 변하는 전체 과정을 정리한다.

---

# 부록 후보

Phase 3 이후 확정한다.

## 부록 A. Agent Contract 템플릿

- README.md
- AGENTS.md
- CLAUDE.md
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
       13 → 14  15 → 16 → 17
        ↓                  ↓
        └──────→ 18 → 19 → 20 → 21
                    ↓          ↓
                    └────→ 22 → 23 → 24
```

# Phase 2에서 결정한 사항

- 하나의 `campus-platform` 예제를 책 전체의 기본 축으로 사용한다.
- 필요하면 특정 장에서 작은 보조 예제를 사용할 수 있지만 새로운 대형 예제 시스템은 추가하지 않는다.
- `Task Contract`는 독립 장으로 둔다.
- `Trust Boundary`는 독립 장으로 둔다.
- `Agent Observability와 Evaluation`은 하나의 독립 장으로 묶는다.
- 제품별 기능 설명은 본문의 중심이 아니라 사례 또는 부록으로 둔다.
- PM Agent는 책 후반부에 배치하고, 먼저 Repository, Verification, Parallel Development, Memory를 설명한다.
- 기업 내부망과 Hybrid Agent는 일반 원칙을 설명한 이후 별도 Part에서 다룬다.

# Phase 3 검증 항목

Phase 3에서는 다음을 검토한다.

- 24개 장이 지나치게 많은지
- 4장과 12장의 모듈 경계 설명이 중복되는지
- 10장과 17장의 검증/Evaluation 경계가 명확한지
- 15장과 16장의 Memory/Long Running 구분이 필요한지
- 18장과 20장의 Lifecycle/PM Agent 책임이 중복되는지
- 22장과 23장의 기업 환경 내용을 하나로 합칠지
- Java/Spring Boot 예제가 원칙 설명을 방해할 정도로 비중이 커지지 않는지
- 특정 AI 제품에 종속된 설명이 포함되지 않았는지
- 보안, 감사, 권한 회수, Human Escalation이 충분한지
