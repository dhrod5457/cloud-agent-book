# Current Phase

Phase 5 - 장별 설계 진행 중 / Cloud Agent 중심 범위 재정렬 완료

본문은 아직 작성하지 않는다.

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/future-topics.md`
6. `STATUS.md`

이전 Agent-Native 독립 장 설계는 현재 목차 결정보다 우선하지 않는다.

# Book Direction

책의 중심 주제는 **Cloud Agent 활용**이다.

핵심 질문:

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

`Agent Ready Software Engineering`은 지원 철학으로 남기지만 책의 중심축으로 확장하지 않는다.

# Cloud Agent Definition

Cloud Agent를 단순히 Cloud에서 실행되는 LLM으로 정의하지 않는다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

책에서 사용할 확장 정의:

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

# Core Messages

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Cloud Agent에게 Repository 전체를 반복해서 이해시키지 않는다.

> 작은 Task와 작은 Context를 전달한다.

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

# Local / Cloud Handoff

하나의 Task는 Local 또는 Cloud에 영구적으로 속하지 않는다.

```text
Local
→ 요구사항 분석
→ Architecture
→ Task Split
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

핵심 원칙:

> Task는 작업 단계에 따라 실행 위치를 이동할 수 있다.

# Git as Handoff Boundary

Git은 단순 형상관리뿐 아니라 Local과 Cloud 사이의 작업 전달 경계로 사용한다.

```text
Local
→ Commit / Push
→ Git
→ Cloud Task Branch
→ Work / Test
→ Commit / Push / PR
```

Cloud Task 상태 후보:

- Task ID
- Session ID
- Branch
- Commit SHA
- Status
- Test Result
- PR

# Cloud Agent Efficiency

## Token / Compute 분리

Cloud Container가 CPU/RAM을 오래 사용해 Build/Test를 수행하는 것과 LLM Token 사용량은 같은 개념이 아니다.

Token은 주로 다음 시점에 증가한다.

- 코드/문서 읽기
- 계획
- Tool Result 읽기
- 로그 분석
- 수정 방향 판단

권장 구조:

```text
Container
→ 많은 Test 실행

Result Filter / Gateway
→ 실패 정보만 추출

Agent
→ 필요한 실패만 분석
```

## Runner-first

```text
Task
  ↓
Runner
  ├─ PASS → 종료
  └─ FAIL → Cloud Agent
```

이 구조는 Agent Platform 일반론이 아니라 Cloud Agent의 Token/비용 최적화 패턴으로 다룬다.

## Context 최소화

- 작은 Task
- 관련 파일만 전달
- 검증 명령 명시
- 변경 금지 범위 명시
- Progressive Context
- Repository 전체 재탐색 방지

## Result Gateway / Evidence

원본 로그와 Artifact는 저장하고 Agent에는 필요한 정보만 먼저 제공한다.

- PASS / FAIL
- Failed Test
- 핵심 Error
- Stack Trace 위치
- Artifact 경로

Cloud Agent 완료 결과에는 가능한 경우 다음 Evidence를 포함한다.

- Commit / Diff
- Build Result
- Unit / Integration Test Result
- E2E Result
- Screenshot / Video
- Log Reference
- PR

UI 작업에서는 `Demos over Diffs`를 빠른 1차 검증 방식으로 사용할 수 있다.

# Prepared Cloud Environment

Cloud Worker가 매번 JDK, Node, Dependency, Browser, Docker Image를 처음부터 준비하지 않게 한다.

핵심 개념:

- Prepared Cloud Environment
- Cloud Environment as Code
- 작업별 Environment
- Snapshot
- Gradle/Maven/npm Cache
- Docker Layer Cache
- Playwright Browser Cache
- Warm Worker
- Cold Start

작업별 환경 후보:

```text
backend-test
frontend-e2e
migration-test
fullstack
```

환경 상태를 나눈다.

```text
Reusable
- Runtime
- Tools
- Dependencies
- Cache

Fresh
- Source
- Branch
- Task
- Test Result
- Temporary Data
```

# Task Size

Cloud Task가 너무 작으면 다음 overhead가 커진다.

- Environment start
- Repository checkout
- Agent startup
- Context loading

너무 크면 다음 비용이 커진다.

- Context
- Token
- Failure Scope
- Retry
- Review

절대 시간 규칙 대신 프로젝트별로 측정해 적정 크기를 정한다.

# Parallel Cloud Workers

독립 Task만 병렬화한다.

- 다른 Repository
- 다른 Module
- 다른 File Scope
- 독립 검증 가능

같은 파일/Schema를 여러 Agent가 동시에 수정하지 않는다.

병렬 Agent 수가 늘면 다음 비용도 함께 증가한다.

- Context 중복
- Dependency setup 중복
- Merge Conflict
- Review
- LLM usage
- PR 관리

# Task Queue / Event-driven Agent

Cloud Agent 실행 시작점이 개발자 PC일 필요는 없다.

Task Source:

- Issue
- CI Failure
- PR Review
- Scheduled Test
- Dependency Update
- Nightly Build

핵심:

> 이벤트가 없으면 Agent도 실행하지 않는다.

PASS 경로에서는 Agent를 호출하지 않는다.

# Developer Time

Cloud Agent 생산성을 Agent 실행시간만으로 평가하지 않는다.

분리 지표:

- Agent Execution Time
- Developer Blocking Time

Cloud Agent의 중요한 가치는 Developer Blocking Time과 Context Switching을 줄이는 데 있다.

# Multi-Repository

Task에 여러 Repository가 필요할 수 있지만 모든 Repository를 항상 제공하지 않는다.

원칙:

> 현재 Task에서 함께 수정될 가능성이 높은 Repository만 Cloud Workspace에 제공한다.

# Current TOC

`planning/toc.md`는 현재 다음 구조를 정의한다.

- 11개 Part
- 18개 장
- Cloud Agent 이해
- 왜 Cloud Agent인가
- 어떤 작업을 Cloud로 보낼 것인가
- Cloud Agent의 Token 절약
- Cloud 컴퓨팅 자원 활용
- 여러 Cloud Agent 사용
- Local + Cloud Hybrid Workflow
- CI/CD와 Cloud Agent
- 실전 프로젝트
- Cloud Agent를 언제 쓰지 말아야 하는가
- Cloud Agent 중심 개발환경의 미래

새로 반영된 배치:

- 1장: Remote Development Worker 정의
- 4장: Developer Blocking Time
- 5장: Task Size / Cloud 적합 조건
- 7장: Cloud Agent Task Contract
- 8장: Evidence Result / Demos over Diffs
- 9장: Prepared Cloud Environment / Cold Start / 작업별 Environment
- 11장: Git Handoff Boundary / Branch-Session-Task 상태
- 12장: 병렬화 상한 / 중복 비용
- 13장: Local→Cloud→Local Handoff / Multi-Repository
- 14장: Task Queue / Event-driven Cloud Agent
- 15~16장: 전체 Hybrid Workflow와 Evidence-based Result

상세 운영 모델:

- `planning/cloud-agent-remote-worker-model.md`

# Future Topics

다음 주제는 본문 핵심에서 제외하고 `planning/future-topics.md`에 보존한다.

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

필요하면 마지막 미래 장에서 짧게 소개한다.

# Completed

- Phase 1 초기 방향 정의
- Phase 2 초기 목차 설계
- Phase 3 초기 목차 검증
- Phase 4 `campus-platform` 예제 설계
- Phase 5 1~3장 초기 설계
- Cloud Agent 중심 Concept / Scope / TOC 재정렬
- Cloud Agent를 Remote Development Worker로 정의
- Token / Compute 분리
- Runner-first
- Result Gateway
- Prepared Environment / Cache
- Git Handoff Boundary
- Evidence-based Result
- Demos over Diffs
- Task Queue / Event-driven Agent
- Task Size 최적화
- Parallelization 상한
- Developer Blocking Time 분리
- Cloud Agent Remote Worker 운영 모델 설계

# In Progress

Phase 5 장별 설계를 새 18장 목차 기준으로 재정렬한다.

기존 `chapters/01~03/plan.md`는 참고 자료로 유지하되 새 목차와 번호/범위가 달라졌으므로 다시 맞춘다.

# Next

다음 작업은 **새 목차 기준 장별 설계 정합성 점검**이다.

우선 순서:

1. `chapters/01~03/plan.md`를 새 1~3장과 맞춘다.
2. 기존 2장 Execution Platform 내용을 3/8/9/10장으로 재배치한다.
3. 새 4장부터 순차적으로 `plan.md`를 작성한다.
4. Agent Platform 일반론은 `planning/future-topics.md` 참조로 유지한다.

Phase 6 본문 집필은 시작하지 않는다.
