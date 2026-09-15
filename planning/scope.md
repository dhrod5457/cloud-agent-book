# Scope

## 범위 원칙

이 책의 중심은 **Cloud Agent를 왜 사용하고, 어떤 Task를 보내고, Local Agent와 어떻게 조합하며, Token과 Cloud Compute를 어떻게 효율적으로 사용할 것인가**이다.

Repository 설계, Harness, Validation, Observability 같은 개념은 이 질문에 직접 도움이 되는 범위에서만 다룬다.

새로운 주제를 추가할 때 다음 기준을 사용한다.

> 이 내용이 Cloud Agent를 더 잘 사용하는 방법과 직접 관련이 있는가?

- YES: 핵심 본문
- 간접적: Tip / Advanced Topic / 미래 전망
- NO: `planning/future-topics.md`

---

# 핵심 본문 범위

## 1. Cloud Agent 이해

- Coding Agent의 기본 개념
- Local Agent와 Cloud Agent 차이
- Cloud Session / Container / Repository
- 독립 실행환경
- CPU / RAM / Disk
- LLM Token과 Compute Resource의 차이
- Cloud Agent를 Remote Worker로 보는 관점

## 2. Cloud Agent를 쓰는 이유

- Local 개발환경 점유 감소
- 장시간 작업 위임
- 독립 실행환경
- 병렬 처리
- 여러 Branch/Container 기반 격리
- 개발자 PC와 분리된 Build/Test 실행

## 3. Local / Cloud Task Routing

### Local 중심

- Architecture 설계
- 복잡한 디버깅
- Human Steering이 잦은 작업
- 내부망 / VPN
- 사내 DB / HSM
- 큰 Context가 필요한 작업
- 최종 통합과 Review

### Cloud 중심

- Build
- Unit Test
- Integration Test
- E2E Test
- Docker Build
- Static Analysis
- Lint
- Migration Validation
- 반복적인 Refactoring
- 작은 Bug Fix
- 독립 Feature
- Documentation
- PR Review
- CI 실패 수정
- 장시간 작업
- 독립적인 병렬 작업

핵심 독자 판단:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

## 4. Cloud Agent의 Token 절약

- Task Scope 축소
- Context Scope 축소
- Repository 전체 재탐색 방지
- Progressive Context
- 필요한 문서만 읽기
- Test Log / Tool Output 최소화
- Result Filter / Result Gateway
- Raw Artifact 저장 후 필요한 부분만 조회
- Retry 제한
- Token / Cost Budget
- 작은 Task 단위
- 여러 Agent의 중복 Context 문제

## 5. Compute Resource 활용

- Cloud Runner
- Build/Test Runner
- Test Container
- Parallel Build/Test
- UI/E2E 반복 실행
- Prebuilt Environment
- Preinstalled Tool
- Gradle/Maven/npm Cache
- Docker Layer Cache
- Snapshot
- Warm Environment / Warm Worker
- Cloud Session 재사용

핵심 원칙:

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

## 6. Runner와 Cloud Agent 분리

```text
Task
  ↓
Runner
  ├─ PASS → 종료
  └─ FAIL → Cloud Agent
```

이 구조는 Agent Platform 일반론이 아니라 Cloud Agent 비용 최적화 방법으로 다룬다.

핵심 설명:

- 모든 테스트 실행에 LLM이 필요하지 않다.
- 정상 경로는 일반 Runner가 처리한다.
- 실패 분석과 코드 수정이 필요할 때 Cloud Agent를 호출한다.
- CPU/RAM 작업과 LLM 판단을 분리한다.

## 7. Cloud Agent 병렬 실행

- Task 분해
- Branch per Task
- Git Worktree
- Independent Clone/Container
- 서로 독립적인 Module/File Scope
- Parallel Worker
- Fan-out / Fan-in
- 병렬 Agent의 Context 중복 비용
- 같은 파일/schema를 수정하는 작업의 병렬화 제한
- Best-of-N은 고급 사례로만 소개

## 8. Local + Cloud Hybrid Workflow

예:

```text
PM / Local Agent
→ Architecture
→ Task 분리

Cloud Worker
→ 독립 구현
→ Build/Test/E2E
→ 검증

Local Agent
→ 내부망 검증
→ 통합
→ 최종 Review
```

기업 환경 사례:

- VPN
- 내부 Git
- Nexus
- Tibero / Oracle
- Redis / Kafka
- HSM
- Jenkins
- Internal API

## 9. CI/CD와 Cloud Agent

- CI Failure → Cloud Agent
- 수정 → 재검증 → PR
- Review Comment 대응
- Nightly Test Failure
- Dependency Update 검증
- Event-driven 호출
- PR 수정과 follow-up session

## 10. Cloud Agent를 쓰지 말아야 하는 경우

- 내부망 의존이 강함
- Repository를 Cloud에 제공할 수 없음
- 매우 큰 Context가 필요함
- 지속적인 Human Steering 필요
- 수정이 너무 작아서 Cloud 환경 준비 비용이 더 큼
- Cloud에서 재현할 수 없는 문제
- 동일 파일을 여러 Agent가 동시에 변경해야 함

## 11. 실전 Java/Spring Boot 프로젝트

`campus-platform`을 이용한다.

핵심 예:

```text
Local
→ 기능 설계 / 구현 / Task 분리

Cloud #1
→ Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Docker Build

Cloud #4
→ Web E2E

FAIL
→ Agent 분석

PASS
→ PR / 통합
```

Java/Spring Boot 예제에서 다음을 실전 적용한다.

- Java 21
- Spring Boot
- Gradle
- MyBatis
- PostgreSQL/Testcontainers
- Docker
- Playwright 또는 Web E2E
- Migration Validation

---

# Cloud Agent 효율화에 직접 연결되는 보조 개념

다음 내용은 Cloud Agent 사용을 설명하는 수단으로 유지하되 독립된 플랫폼 이론으로 확장하지 않는다.

## Agent Contract / Repository 안내

- README / AGENTS.md
- build/test/verify 명령
- 문서 위치 안내
- 변경 금지 영역

목적은 Cloud Agent가 Repository 전체를 다시 추론하는 비용을 줄이는 것이다.

## Task Contract

- Goal
- Scope
- Relevant Files
- Forbidden Changes
- Acceptance Criteria
- Verification
- Budget

목적은 작은 Task와 작은 Context를 Cloud Agent에 전달하는 것이다.

## Progressive Context

Cloud Agent가 현재 Task에 필요한 문서만 읽게 한다.

예:

```text
AGENTS.md
├─ Backend → docs/backend.md
├─ Auth → docs/security/auth.md
├─ DB → docs/database.md
└─ Test → docs/testing.md
```

Agent Memory Architecture로 확장하지 않는다.

## Result Gateway

목적은 Cloud Agent가 읽어야 하는 Tool Output을 줄이는 것이다.

- Summary
- Failed Test Index
- Stack Trace Lookup
- Log Search
- Artifact Lookup

거대한 Agent Platform 구성요소로 확장하지 않는다.

## Brain / Hands

CPU/RAM과 Token의 차이를 이해하기 위한 간단한 설명으로만 사용한다.

```text
Brain → LLM 판단
Hands → Cloud Container에서 명령 실행
```

별도 아키텍처 장으로 확장하지 않는다.

---

# Advanced Topic으로 제한할 범위

## Best-of-N

어려운 Bug에서 여러 Cloud Agent가 독립 Patch를 만들고 Test Runner가 검증하는 병렬화 사례로 짧게 다룬다.

본문 핵심 개념으로 만들지 않는다.

## Harness Engineering

Cloud Agent가 실행 명령과 검증 방법을 쉽게 찾도록 하는 개발환경 개선 관점까지만 설명한다.

## Agent-native Observability / Orchestration

마지막 미래 전망에서 방향만 소개한다.

---

# 현재 책에서 핵심 범위 밖으로 이동할 내용

다음 주제는 독립 장으로 만들지 않는다.

- Agent Chaos Engineering
- Agent Security Platform
- Agent Memory Architecture
- Agent OS
- Garbage Collector Agent
- Canary Agent
- Shadow Agent
- Agent Security Telemetry
- Capability Broker 상세 설계
- Agent Governance Platform
- Agent Observability Platform
- Agent-native Security
- Multi-agent 조직론
- Agent Platform 운영

기존에 작성한 설계는 삭제하지 않고 `planning/future-topics.md`에서 후속 주제 후보로 보존한다.

---

# 특정 제품의 사용 범위

Claude Code, Codex, GitHub Copilot Coding Agent, Cursor 등의 제품은 설계 패턴을 설명하는 실제 사례로만 사용한다.

제품별 CPU/RAM, 가격, Session 제한처럼 변경 가능성이 높은 정보는 `research/`에서 기준일과 출처를 관리하고 본문의 핵심 논리와 분리한다.

제품 사용 설명서 형태로 작성하지 않는다.

---

# 최종 독자 역량

독자는 책을 읽고 다음을 판단하고 구현할 수 있어야 한다.

- 왜 Cloud Agent를 써야 하는가
- Local Agent와 무엇이 다른가
- 어떤 Task를 Cloud에 보내야 하는가
- 어떤 Task를 Local에 남겨야 하는가
- 여러 Cloud Agent를 어떻게 병렬로 사용하는가
- Token을 어떻게 절약하는가
- Cloud CPU/RAM을 어떻게 활용하는가
- Build/Test를 어떻게 Cloud에 분산하는가
- Cloud Agent Context를 어떻게 줄이는가
- Local + Cloud Hybrid 개발환경을 어떻게 구성하는가

책의 목표는 독자가 Agent Platform 전체를 설계하게 만드는 것이 아니다.
