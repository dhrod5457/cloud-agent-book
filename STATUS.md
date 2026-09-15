# Current Phase

Phase 5 - 장별 설계 진행 중 / Cloud Agent 중심 범위 재정렬 완료

본문은 아직 작성하지 않는다.

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/future-topics.md`
5. `STATUS.md`

이전 `planning/toc-amendment-agent-native.md`의 독립 장 추가 결정은 현재 목차 결정보다 우선하지 않는다.

# Book Direction

책의 중심 주제는 **Cloud Agent 활용**이다.

핵심 질문:

> 클라우드 에이전트를 왜 사용하고, 로컬 에이전트와 어떻게 조합하며, 어떤 작업을 맡기고, 토큰과 클라우드 컴퓨팅 자원을 어떻게 효율적으로 활용할 것인가?

`Agent Ready Software Engineering`은 지원 철학으로 남기지만 책의 중심축으로 확장하지 않는다.

# Core Messages

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Cloud Agent에게 Repository 전체를 반복해서 이해시키지 않는다.

> 작은 Task와 작은 Context를 전달한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

# Local / Cloud 판단 기준

## Local 중심

- Architecture 설계
- 복잡한 디버깅
- Human Steering이 잦은 작업
- 내부망 / VPN / 사내 DB / HSM
- 여러 모듈을 동시에 이해해야 하는 작업
- 빠른 질문/수정 반복
- 최종 통합 / Review

## Cloud 중심

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

독자가 최종적으로 다음을 판단할 수 있어야 한다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

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

## Result Gateway

목적은 Cloud Agent가 읽어야 하는 Tool Output을 줄이는 것이다.

원본 로그와 Artifact는 저장하고 다음만 먼저 제공한다.

- PASS / FAIL
- Failed Test
- 핵심 Error
- Stack Trace 위치
- Artifact 경로

필요할 때만 상세 내용을 조회한다.

## Prebuilt Environment / Cache

Cloud Worker가 매번 JDK, Node, Dependency, Browser, Docker Image를 처음부터 준비하지 않게 한다.

사용 후보:

- Base Image
- Snapshot
- Gradle/Maven Cache
- npm Cache
- Docker Layer Cache
- Playwright Browser Cache
- Preinstalled Tools
- Warm Worker

핵심:

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

# Parallel Cloud Workers

독립 Task만 병렬화한다.

- 다른 Repository
- 다른 Module
- 다른 File Scope
- 독립 검증 가능

공통 파일이나 Schema를 수정한다면 먼저 dependency를 분석하고 순차 실행으로 전환한다.

여러 Agent를 사용할 때 같은 Repository 전체를 각 Agent가 반복 분석하지 않도록 한다.

# Hybrid Workflow

```text
Local / PM
→ Architecture
→ Task 분리

Cloud Worker
→ 독립 구현 / Build / Test / E2E

Local
→ 내부망 검증
→ 통합
→ 최종 Review
```

# CI/CD

Agent는 항상 실행될 필요가 없다.

호출 후보:

- CI Failure
- Review Comment
- Nightly Test Failure
- Dependency Update Failure
- 반복 검증 실패

PASS 경로에서는 Agent 호출을 생략할 수 있다.

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

이 목차가 현재 최신 목차다.

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

# Existing Artifacts

기존에 만든 다음 문서는 삭제하지 않는다.

- `planning/agent-native-development-environment.md`
- `planning/toc-amendment-agent-native.md`
- `examples/campus-platform/agent-native-development-environment.md`
- `research/anthropic/agent-native-development-environment.md`

이들은 현재 책의 핵심 목차가 아니라 후속 주제 연구 자료로 취급한다.

# Completed

- Phase 1 초기 방향 정의
- Phase 2 초기 목차 설계
- Phase 3 초기 목차 검증
- Phase 4 `campus-platform` 예제 설계
- Phase 5 1~3장 초기 설계
- Cloud Runner / Agent Worker 구분
- Token / Compute 분리 관점
- Result Gateway
- Prebuilt Environment / Cache
- Parallel Cloud Session
- GitHub Continuous AI 사례 조사
- Claude Code Web 실행환경 사례 조사
- Cloud Agent 중심 Concept 재정의
- Cloud Agent 중심 Scope 재정의
- 11 Part / 18 Chapter 목차 재정렬
- 고급 Agent Platform 내용을 Future Topics로 이동

# In Progress

Phase 5 장별 설계를 새 목차 기준으로 재정렬한다.

기존 `chapters/01~03/plan.md`는 참고 자료로 유지하되 새 목차와 번호/범위가 달라졌으므로 순차적으로 다시 맞춘다.

# Next

다음 작업은 **새 목차 기준 장별 설계 정합성 점검**이다.

우선 순서:

1. 기존 `chapters/01~03/plan.md`를 새 1~3장과 비교
2. 2장의 Execution Platform 내용을 Cloud Agent 활용 범위로 축소/재배치
3. 새 4장부터 순차 설계
4. 기존 Agent Platform 중심 내용은 `planning/future-topics.md` 참조로 전환

Phase 6 본문 집필은 시작하지 않는다.
