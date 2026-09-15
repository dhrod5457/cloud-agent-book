# Current Phase

Phase 5 - 장별 설계 진행 중 / 새 Cloud Agent 중심 목차에 1~5장 설계 반영 완료

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

최종 독자 판단:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

# Core Definition

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

# Current TOC

`planning/toc.md`가 최신 목차다.

- 11개 Part
- 18개 장

큰 흐름:

1. Cloud Agent 이해
2. 왜 Cloud Agent인가
3. 어떤 작업을 Cloud로 보낼 것인가
4. Cloud Agent의 Token 절약
5. Cloud 컴퓨팅 자원 활용
6. 여러 Cloud Agent 사용
7. Local + Cloud Hybrid Workflow
8. CI/CD와 Cloud Agent
9. 실전 프로젝트
10. Cloud Agent를 언제 쓰지 말아야 하는가
11. Cloud Agent 중심 개발환경의 미래

# Phase 5 Chapter Alignment

## 1장 - Coding Agent에서 Cloud Worker로

정식 파일:

- `chapters/01/plan.md`

핵심 역할:

- Chat LLM과 Coding Agent 구분
- Cloud Agent를 Remote Development Worker로 정의
- `LLM + Repository + Execution Environment + Compute + Tools` 모델
- 결과를 자연어 응답이 아니라 Commit/Test/Artifact/PR까지 포함한 작업 상태로 정의

이 장에서는 Agent Ready 일반론이나 Token 최적화를 확장하지 않는다.

## 2장 - Local Agent와 Cloud Agent

정식 파일:

- `chapters/02/plan.md`

핵심 역할:

- 실행 위치 기준 Local/Cloud 비교
- 내부망 / VPN / HSM / DB / Human Steering 판단
- Local 중심 작업과 Cloud 중심 작업 구분
- `Local → Cloud → Local` Handoff 개념
- Git 기반 handoff의 필요성 소개
- Agent Execution Time과 Developer Blocking Time 구분

`chapters/02/execution-platform.md`는 이전 확장 설계 자료로 유지하지만 2장 본문 전체를 의미하지 않는다.

범위 안내:

- `chapters/02/README.md`

해당 확장 문서의 내용은 3, 4, 7, 8, 9, 10, 11, 12, 13, 14장으로 분산한다.

## 3장 - Cloud Session, Container, Compute와 Token

정식 파일:

- `chapters/03/plan.md`

핵심 역할:

- Cloud Session / Workspace / Container 개념
- CPU/RAM/Disk와 LLM Token 분리
- Brain/Hands를 간단한 설명 모델로만 사용
- Build/Test 실행시간과 Token 사용량의 차이
- Compute-heavy / Context-light 작업
- Parallel Compute와 Parallel Reasoning 구분
- Compute / LLM / Human Cost 분리

제품별 CPU/RAM/Disk 숫자는 `research/`에서 기준일과 함께 관리하며 본문 핵심 논리에 고정하지 않는다.

## 4장 - 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

정식 파일:

- `chapters/04/plan.md`

핵심 역할:

- 개발자 PC와 Cloud Worker 실행환경 분리
- 장시간 Build/Test 작업의 비동기 위임
- `Agent Execution Time`과 `Developer Blocking Time` 분리
- 독립 Task의 병렬 실행
- Parallelizable Task와 Dependent Task 구분
- 병렬화가 선형적으로 생산성을 높이지 않는 이유
- Local Resource Occupancy 감소를 Cloud 활용 가치로 포함
- `campus-platform`의 Unit/Integration/E2E/Docker 검증 병렬 예제

4장의 핵심 문장:

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> Cloud Agent가 오래 실행되는 것과 개발자가 오래 기다리는 것은 같은 의미가 아니다.

> 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.

## 5장 - Task Routing: Local인가 Cloud인가

정식 파일:

- `chapters/05/plan.md`

핵심 역할:

- Task 특성을 기준으로 Local / Cloud / Hybrid / Runner-first 경로 선택
- Scope 명확성, 완료 조건, 독립 검증 가능 여부 평가
- Human Steering 빈도와 Context 크기를 Routing 기준으로 사용
- Internal Network / VPN / DB / HSM 같은 Hard Constraint 반영
- File Conflict와 dependency를 병렬 Cloud 작업 전에 확인
- Git을 통해 입력/결과를 전달할 수 있는지 판단
- 너무 작은 Task의 Cloud overhead와 너무 큰 Task의 Context/Retry/Review 비용 비교
- `campus-platform` 작업을 실제 Local/Cloud/Hybrid로 분류

5장의 핵심 문장:

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

# Preserved Topics

기존 3장의 `Agent Ready 프로젝트의 기준`은 독립 장에서 제외했지만 삭제하지 않았다.

`planning/future-topics.md`에 `Agent Ready Profile` 후속 주제로 이동했다.

기존 평가 기준:

- Reproducibility
- Discoverability
- Executability
- Testability
- Verifiability
- Isolation
- Parallelizability
- Observability
- Security Boundary

현재 책에서는 Cloud Agent 활용에 직접 필요한 부분만 각 관련 장에 분산한다.

# Cloud Agent Efficiency Decisions

## Local / Cloud Handoff

```text
Local
→ 요구사항 분석 / Architecture / Task Split
→ Commit / Push
      ↓
Cloud
→ 독립 Work / Build / Test / E2E / CI Fix
→ Evidence / PR
      ↓
Local
→ 내부망 검증 / Review / Integration / Merge
```

Task는 Local 또는 Cloud에 영구적으로 속하지 않는다.

## Task Routing

Routing은 다음 순서로 판단한다.

```text
1. Runner로 해결 가능한가?
2. Cloud에 독립 실행 가능한가?
3. Human Steering이 필요한가?
4. Internal Network가 필요한가?
```

주요 판단 기준:

- Scope 명확성
- 완료 조건
- 독립 검증 가능 여부
- File Conflict
- Human Steering
- Context 크기
- Internal Network
- 실행시간/Compute 요구량
- Git Handoff 가능 여부
- Cloud start overhead

## Git as Handoff Boundary

Git은 형상관리뿐 아니라 Local과 Remote Worker 사이의 작업 전달 경계로 다룬다.

상태 후보:

- Task ID
- Session ID
- Branch
- Commit SHA
- Status
- Test Result
- PR

## Token / Compute 분리

```text
Cloud Container
→ 많은 Build/Test 실행

Result Filter / Gateway
→ 필요한 실패 정보 추출

Cloud Agent
→ 필요한 Context만 분석
```

## Prepared Cloud Environment

주요 개념:

- Cloud Environment as Code
- Prepared Environment
- Task-specific Environment
- Snapshot
- Cache
- Warm Worker
- Cold Start
- Fresh Source/Branch/Task 상태

## Evidence-based Result

Cloud Agent 완료 결과에는 가능한 경우 다음을 포함한다.

- Commit / Diff
- Build Result
- Unit / Integration Test Result
- E2E Result
- Screenshot / Video
- Log Reference
- PR

## Async Delegation / Time Separation

Cloud Agent 활용 효과를 Agent Execution Time만으로 평가하지 않는다.

함께 볼 값:

- Agent Execution Time
- Queue Time
- Developer Blocking Time
- Review Time
- Retry Time
- Local Resource Occupancy

Cloud Task가 오래 실행되어도 개발자가 다음 작업을 계속할 수 있다면 전체 Workflow 관점에서는 이점이 있을 수 있다.

## Parallel Execution

독립 Task만 병렬화한다.

좋은 대상:

- Unit Test
- Integration Test
- E2E
- Docker Build
- 서로 다른 서비스/모듈 작업

같은 파일, 공통 schema, 강한 dependency가 있는 작업은 병렬화보다 순차 처리를 우선한다.

# Future Topics

다음 주제는 본문 핵심에서 제외하고 `planning/future-topics.md`에 보존한다.

- Agent-Native Development Environment 전체 아키텍처
- Agent Ready Assessment 일반론
- Agent Memory Architecture
- Agent Security Platform
- Agent Chaos Engineering
- Garbage Collector Agent
- Shadow Agent
- Canary Agent
- Agent Platform Governance
- Agent-native Observability Platform
- Multi-agent 조직론

필요하면 18장 미래 전망에서 짧게 언급한다.

# Completed

- Phase 1 초기 방향 정의
- Phase 2 초기 목차 설계
- Phase 3 초기 목차 검증
- Phase 4 `campus-platform` 예제 설계
- Cloud Agent 중심 Concept/Scope 재정의
- 11 Part / 18 Chapter 목차 재정렬
- Remote Worker 운영 모델 설계
- Agent Platform 일반론을 Future Topics로 이동
- 새 목차 기준 1장 재설계 완료
- 새 목차 기준 2장 재설계 완료
- 새 목차 기준 3장 재설계 완료
- 새 목차 기준 4장 설계 완료
- 새 목차 기준 5장 Task Routing 설계 완료
- 기존 Agent Ready Profile을 Future Topics로 보존

# In Progress

Phase 5 장별 설계.

새 목차 기준 1~5장 설계가 완료되었다.

# Next

다음 대상:

`chapters/06/plan.md` - Cloud에 보내기 좋은 개발 작업

6장에서 다룰 핵심:

- Build
- Unit Test
- Integration Test
- E2E
- Docker Build
- Static Analysis / Lint
- Migration Validation
- Refactoring
- 작은 Bug Fix
- Documentation
- PR Review
- Dependency Update
- CI Failure Fix
- 작업별 Cloud Runner / Cloud Agent 선택
- `campus-platform`의 실제 Cloud Task 예제

그 다음 7장부터 새 목차 순서대로 설계한다.

Phase 6 본문 집필은 시작하지 않는다.
