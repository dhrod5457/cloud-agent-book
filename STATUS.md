# Current Phase

Phase 6 - 본문 초고 작성 진행 중

Phase 5의 1~18장 설계와 전체 정합성 점검을 완료했다.

현재 1~2장 초고를 작성했다.

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase5-consistency-check.md`
6. 각 `chapters/NN/plan.md`
7. `planning/future-topics.md`
8. `STATUS.md`

과거 Agent-Native 독립 장 설계와 초기 22~23장 체계는 현행 목차보다 우선하지 않는다.

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

> Runner가 할 수 있으면 Runner에게 맡긴다.

> 작은 Task와 작은 Context를 전달한다.

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

# Current TOC

`planning/toc.md`가 최신 목차다.

- 11개 Part
- 18개 장

1. Coding Agent에서 Cloud Worker로
2. Local Agent와 Cloud Agent
3. Cloud Session, Container, Compute와 Token
4. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리
5. Task Routing: Local인가 Cloud인가
6. Cloud에 보내기 좋은 개발 작업
7. Cloud Agent Task Contract: 작은 Task와 작은 Context
8. Tool Output을 줄이고 Evidence를 남기기
9. Prepared Cloud Environment, Cache, Snapshot
10. Cloud Agent를 Test Runner처럼 사용하기
11. Git, Branch, Worktree, Container로 작업 격리하기
12. 병렬 Worker와 중복 Context 비용
13. Local → Cloud → Local Handoff
14. Task Queue와 Event-driven Cloud Agent
15. campus-platform Cloud Agent Workflow 설계
16. 하나의 기능을 Local + Cloud로 끝까지 개발하기
17. Cloud가 항상 정답은 아니다
18. 다음 단계: Harness와 Orchestration

# Phase 5 Result

Phase 5 완료.

- 1~18장 설계 완료
- 전체 역할/중복/용어/범위 정합성 점검 완료
- `planning/phase5-consistency-check.md` 작성
- Agent Platform 확장 내용은 `planning/future-topics.md`로 이동
- 과거 `chapters/02/execution-platform.md`는 참고자료로만 유지

# Phase 6 Writing Rules

본문은 1장부터 순차 작성한다.

- 각 장의 `plan.md`를 우선한다.
- 뒤 장 내용을 앞 장에서 과도하게 설명하지 않는다.
- 제품은 설계 원칙을 설명하기 위한 실제 사례로만 사용한다.
- 변경 가능한 제품 사양/가격/한도는 `research/`에 분리한다.
- Java/Spring Boot `campus-platform`을 반복 예제로 사용한다.
- 좋은 사례와 나쁜 사례를 비교한다.
- 가능한 경우 실행 명령과 검증 Evidence를 사용한다.
- 수치는 근거 있는 사실과 설명용 예시를 구분한다.
- Agent Platform 일반론으로 확장하지 않는다.

# Phase 6 Progress

## 1장 - Coding Agent에서 Cloud Worker로

상태: `초고 작성 및 설계 대비 1차 검토 완료`

파일:

- 설계: `chapters/01/plan.md`
- 초고: `chapters/01/draft.md`
- 공식 근거: `research/chapter-01-cloud-worker-official-sources.md`

주요 내용:

- Chat LLM과 Coding Agent 구분
- Remote Development Worker 정의
- Cloud Agent = LLM + Repository + Execution Environment + Compute + Tools
- Prompt보다 Task 관점
- Evidence 기반 결과
- Git Handoff Boundary 소개
- `campus-platform` 병렬 검증 예제

## 2장 - Local Agent와 Cloud Agent

상태: `초고 작성 완료`

파일:

- 설계: `chapters/02/plan.md`
- 초고: `chapters/02/draft.md`
- 공식 근거: `research/chapter-02-local-cloud-official-sources.md`

주요 내용:

- 모델보다 실행 위치를 우선하는 비교
- Local에 적합한 Interactive / Internal 작업
- Cloud에 적합한 독립 / 장시간 / 검증 가능한 작업
- Local → Cloud → Local Handoff
- Internal Network를 Hard Constraint로 판단
- Human Steering / Context 크기 기준
- Agent Execution Time vs Developer Blocking Time
- Git Handoff Boundary
- `campus-platform` 작업별 Local / Cloud / Hybrid 분류

# Preserved / Future Topics

다음은 현재 책의 본문 핵심이 아니다.

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
- General Multi-Agent Theory

필요하면 18장에서 짧게 소개하고 상세 설계는 `planning/future-topics.md`에 둔다.

# Next

1. 2장 초고 설계 대비 자체 검토
2. 필요한 수정 반영
3. 3장 `Cloud Session, Container, Compute와 Token` 초고 작성

Phase 6에서는 장별로 `초고 → 설계 대비 검토 → 수정 → 다음 장` 순서로 진행한다.
