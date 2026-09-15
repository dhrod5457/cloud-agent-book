# Current Phase

Phase 5 - 장별 설계 진행 중 / 새 Cloud Agent 중심 목차에 1~7장 설계 반영 완료

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

확장 정의:

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

- `chapters/01/plan.md`
- Chat LLM과 Coding Agent 구분
- Cloud Agent를 Remote Development Worker로 정의
- `LLM + Repository + Execution Environment + Compute + Tools`
- Commit/Test/Artifact/PR을 포함한 결과 모델

## 2장 - Local Agent와 Cloud Agent

- `chapters/02/plan.md`
- 실행 위치 기준 Local/Cloud 비교
- 내부망 / VPN / HSM / DB / Human Steering 판단
- `Local → Cloud → Local` Handoff 소개
- Agent Execution Time과 Developer Blocking Time 구분

`chapters/02/execution-platform.md`는 이전 확장 설계 자료이며 범위 안내는 `chapters/02/README.md`에서 관리한다.

## 3장 - Cloud Session, Container, Compute와 Token

- `chapters/03/plan.md`
- Cloud Session / Workspace / Container
- CPU/RAM/Disk와 LLM Token 분리
- Brain/Hands 간단 모델
- Compute-heavy / Context-light 작업
- Parallel Compute와 Parallel Reasoning 구분
- Compute / LLM / Human Cost 분리

## 4장 - 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

- `chapters/04/plan.md`
- 개발자 PC와 Cloud Worker 환경 분리
- 장시간 작업의 비동기 위임
- Agent Execution Time vs Developer Blocking Time
- 독립 Task 병렬 실행
- Parallelizable Task vs Dependent Task
- Local Resource Occupancy 감소

핵심:

> 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.

## 5장 - Task Routing: Local인가 Cloud인가

- `chapters/05/plan.md`
- Local / Cloud / Hybrid / Runner-first 경로 선택
- Scope, 완료 조건, 독립 검증, Human Steering, Context 크기 평가
- Internal Network / VPN / DB / HSM Hard Constraint
- File Conflict / dependency 확인
- Git Handoff 가능 여부
- 너무 작은 Task와 너무 큰 Task의 비용 비교

핵심:

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

## 6장 - Cloud에 보내기 좋은 개발 작업

- `chapters/06/plan.md`
- Build, Unit Test, Integration Test, E2E, Docker Build를 Cloud Runner 중심으로 분류
- Static Analysis / Lint / Migration Validation을 deterministic 실행으로 분류
- 반복 Refactoring / 작은 Bug Fix / Documentation / PR Review를 Cloud Agent 후보로 분류
- Dependency Update와 CI Failure Fix를 `Runner → Agent → Runner` 패턴으로 설계
- 작업별 Input / Validation / Evidence 정의
- `campus-platform` Cloud Task Catalog 정의

핵심:

> Runner가 할 수 있으면 Runner에게 맡긴다.

> Agent는 판단과 수정이 필요한 구간에만 사용한다.

## 7장 - Cloud Agent Task Contract: 작은 Task와 작은 Context

- `chapters/07/plan.md`
- Task / Goal / Scope를 분리
- Relevant Files를 initial context boundary로 사용
- Forbidden Changes로 변경 경계 정의
- Validation을 실행 가능한 명령으로 명시
- Expected Result를 관찰 가능한 결과로 작성
- Output을 Commit/Test/Artifact 기반 Evidence로 정의
- AGENTS.md → 관련 문서 → 관련 소스 → 관련 테스트 순서의 Progressive Context
- Context 확대 단계를 정의해 Repository 전체 재탐색 방지
- Base SHA / Task Branch를 작업 입력에 포함
- Environment와 선택적 Budget을 Task Contract에 연결
- Task Contract가 지나치게 커지면 Task 분해 또는 Local/Hybrid 전환을 검토
- `campus-platform` AuthService expired-token Task Contract 예제

7장의 핵심 원칙:

> Task Contract의 목적은 Prompt를 길게 만드는 것이 아니라 Agent가 탐색해야 하는 범위를 줄이는 것이다.

> Goal은 결과를 제한하고 Scope는 탐색과 변경 범위를 제한한다.

> Context를 줄이는 것뿐 아니라 Context를 필요할 때 가져오게 만든다.

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

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

```text
1. Runner로 해결 가능한가?
2. Cloud에 독립 실행 가능한가?
3. Human Steering이 필요한가?
4. Internal Network가 필요한가?
```

## Runner / Agent Separation

```text
Development Task
      ↓
Deterministic?
  ├─ YES → Cloud Runner
  │          ↓
  │        PASS / FAIL
  │          ↓
  │      FAIL + 판단 필요
  │          ↓
  │       Cloud Agent
  └─ NO → Cloud Agent 또는 Local/Hybrid
```

## Small Input / Small Output

7장과 8장을 다음 한 쌍으로 설계한다.

```text
7장
Task Contract
→ 작은 Input / Context
        ↓
Cloud Agent / Runner
        ↓
8장
Result Gateway / Evidence
→ 작은 Output / Tool Result
```

## Token / Compute Separation

```text
Cloud Container
→ 많은 Build/Test 실행

Result Filter / Gateway
→ 필요한 실패 정보 추출

Cloud Agent
→ 필요한 Context만 분석
```

## Prepared Cloud Environment

- Cloud Environment as Code
- Prepared Environment
- Task-specific Environment
- Snapshot
- Cache
- Warm Worker
- Cold Start
- Fresh Source/Branch/Task 상태

## Evidence-based Result

Cloud 결과에는 가능한 경우 다음을 포함한다.

- Commit / Diff
- Build Result
- Unit / Integration Test Result
- E2E Result
- Screenshot / Video
- Log Reference
- PR

# Preserved / Future Topics

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
- 새 목차 기준 6장 Cloud Task Catalog 설계 완료
- 새 목차 기준 7장 Cloud Agent Task Contract 설계 완료
- 기존 Agent Ready Profile을 Future Topics로 보존

# In Progress

Phase 5 장별 설계.

새 목차 기준 1~7장 설계가 완료되었다.

# Next

다음 대상:

`chapters/08/plan.md` - Tool Output을 줄이고 Evidence를 남기기

8장에서 다룰 핵심:

- Raw log 전체를 LLM에 전달하지 않는 구조
- Result Filter / Result Gateway
- Summary / Failed Test Index / Exception / Stack Trace Lookup
- Artifact First
- build.log / junit.xml / coverage.xml / screenshot / video
- 필요한 경우에만 상세 로그 조회
- Evidence-based Cloud Agent Result
- Demos over Diffs
- Failure Fingerprint
- Retry / Token / Cost Budget과 결과 처리의 연결
- `campus-platform` 테스트 실패 결과 예제

그 다음 9장부터 새 목차 순서대로 설계한다.

Phase 6 본문 집필은 시작하지 않는다.
