# Current Phase

Phase 6 - 본문 초고 작성 진행 중

Phase 5의 1~18장 설계와 전체 정합성 점검을 완료했다.

현재 1~16장 초고를 작성했다.

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

> 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.

> Task는 Local 또는 Cloud에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

> 이벤트가 없으면 Agent도 실행하지 않는다.

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
- `chapters/02/execution-platform.md`는 과거 확장 참고자료로만 유지

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

## 1~12장

상태: `초고 작성 및 설계 대비 1차 검토 완료`

- `chapters/01/draft.md`
- `chapters/02/draft.md`
- `chapters/03/draft.md`
- `chapters/04/draft.md`
- `chapters/05/draft.md`
- `chapters/06/draft.md`
- `chapters/07/draft.md`
- `chapters/08/draft.md`
- `chapters/09/draft.md`
- `chapters/10/draft.md`
- `chapters/11/draft.md`
- `chapters/12/draft.md`

핵심 흐름:

```text
Cloud Worker 정의
→ Local / Cloud 비교
→ Compute / Token 분리
→ 독립 실행환경 / 비동기 / 병렬성
→ Task Routing
→ 작업 유형별 Runner / Agent / Local-Hybrid 배치
→ Small Task / Small Context
→ Small Output / Evidence
→ Prepared Environment
→ Runner-first / Agent-on-failure
→ Source / Runtime 격리
→ 병렬 Fan-out / Fan-in 비용 관리
```

## 13장 - Local → Cloud → Local Handoff

상태: `초고 작성 완료`

- 설계: `chapters/13/plan.md`
- 초고: `chapters/13/draft.md`

핵심:

- Task는 단계에 따라 Local과 Cloud 사이를 이동
- Local은 요구사항/Architecture/내부망/최종 통합에 사용
- Cloud는 독립 실행/장시간 검증/병렬 처리에 사용
- Task Contract와 Git Commit을 Local→Cloud 입력으로 사용
- Git을 Handoff Boundary로 사용
- Evidence / Commit / PR을 Cloud→Local Return Boundary로 사용
- 내부망 자원은 Local Validation으로 남겨 Hybrid 구성
- Cloud 실패 시 Result Gateway/Agent-on-failure를 먼저 적용하고 필요한 경우 Local Fallback
- Multi-Repository는 필요한 Repository만 연결하고 각 Base SHA를 고정

## 14장 - Task Queue와 Event-driven Cloud Agent

상태: `초고 작성 및 설계 대비 1차 검토 완료`

- 설계: `chapters/14/plan.md`
- 초고: `chapters/14/draft.md`

핵심:

- Push / CI Failure / Review Comment / Nightly / Dependency Update / Issue를 Task Source로 사용
- Event → Task Candidate → Dedup / Classification → Runner / Agent 흐름
- 정상 경로는 Runner에서 종료
- Infrastructure Failure와 Code Failure 분리
- 동일 SHA / Failure Fingerprint 기반 중복 Task 억제
- Agent Push → CI FAIL → Agent 재호출 Loop를 Budget/Fingerprint로 제한
- 자동 결과는 Draft PR 또는 기존 PR Commit으로 반환

14장의 핵심 원칙:

> 이벤트가 없으면 Agent도 실행하지 않는다.

> 정상 경로에는 LLM이 필요하지 않다.

> 자동화에는 시작 조건뿐 아니라 중복 제거와 종료 조건도 필요하다.

## 15장 - campus-platform Cloud Agent Workflow 설계

상태: `초고 작성 완료`

- 설계: `chapters/15/plan.md`
- 초고: `chapters/15/draft.md`

핵심:

- 1~14장의 원칙을 하나의 Java/Spring Boot 프로젝트 운영 모델로 통합
- Local Workspace / Cloud Runner / Cloud Agent 역할 분리
- backend-test / frontend-e2e / migration-test Environment 구성
- RUN-BUILD / RUN-UNIT / RUN-INTEGRATION / RUN-E2E / RUN-DOCKER / FIX-BUG / REFACTOR-MODULE Task Catalog
- Task Contract를 운영 입력 형식으로 사용
- Task ID / Base SHA / Branch / Session / Current SHA / PR 상태 연결
- 동일 SHA 기준 Unit / Integration / Docker / E2E 병렬 검증
- Result Gateway / result.json / Artifact 구조
- Infrastructure Failure를 코드 Agent에서 분리
- Tibero / HSM / Internal API를 Local Hybrid Validation으로 남김
- UI Task는 Screenshot / Video / Trace Evidence 사용
- Compute / LLM / Human Cost를 분리해 관찰
- 성공 기준을 Agent 수가 아니라 Blocking Time / Lead Time / Evidence / Rework로 정의

15장의 핵심 원칙:

> Task Contract로 작업을 넘기고 Evidence로 결과를 돌려받는다.

> Local에서는 설계와 통합을 하고, Cloud에서는 독립적인 작업을 병렬로 처리한다.

## 16장 - 하나의 기능을 Local + Cloud로 끝까지 개발하기

상태: `초고 작성 완료`

- 설계: `chapters/16/plan.md`
- 초고: `chapters/16/draft.md`

실전 시나리오:

```text
학생 출결 API 인증 변경
```

핵심:

- Local Requirement/Impact Analysis
- Base Commit 고정
- Agent Task와 Runner Task 분리
- Task Contract 작성
- Prepared Environment + Fresh Branch
- Agent는 Relevant Files부터 시작하고 Context를 단계적으로 확대
- Target Test → Failure Summary → Retry / Failure Fingerprint
- 수정 SHA에서 Unit / Integration / Docker / E2E 병렬 Regression
- UI Failure는 Screenshot / Video / Trace부터 확인
- 최종 Cloud Evidence와 PR 생성
- Local에서 Evidence → Changed Files → Diff → Architecture 순으로 Review
- Tibero/Internal API/Jenkins 등 내부망 최종 검증
- PR 기준 SHA와 Cloud/Internal Evidence 일치 확인
- Merge 전 Full Validation
- Merge 후 최소 Task 이력 보존
- Local-only와 Hybrid의 Blocking Time / Overhead 비교
- 재현 불가 / 내부 DB 의존 / Scope 확대 시 Local Fallback 또는 Task 재분해

16장의 핵심 원칙:

> Cloud Agent 활용은 별도 도구 사용법이 아니라 개발 Workflow 설계다.

> 작은 Task를 Git으로 넘기고, Runner와 Agent가 작업한 Evidence를 다시 Local로 가져온다.

# Current Execution Flow

현재 7~16장의 연결은 다음과 같다.

```text
7장  Task Contract / Small Input
  ↓
8장  Result Gateway / Small Output / Evidence
  ↓
9장  Prepared Environment
  ↓
10장 Runner-first / Agent-on-failure
  ↓
11장 Source / Runtime / Artifact 격리
  ↓
12장 독립 Task Fan-out / Fan-in 비용 관리
  ↓
13장 Local → Git → Cloud → Evidence → Local Handoff
  ↓
14장 CI / Review / Schedule 이벤트가 Task 생성
  ↓
15장 campus-platform 전체 운영 모델로 통합
  ↓
16장 하나의 기능을 Requirement → Merge까지 시간 순서로 실행
```

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

1. 15~16장 설계 대비 자체 검토
2. 필요한 수정 반영
3. 17장 `Cloud가 항상 정답은 아니다` 초고 작성
4. 18장 `다음 단계: Harness와 Orchestration` 초고 작성
5. 1~18장 초고 전체 정합성 점검
6. Phase 6 초고 완료 처리

Phase 6에서는 장별로 `초고 → 설계 대비 검토 → 수정 → 다음 장` 순서로 진행한다.
