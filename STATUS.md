# Current Phase

Phase 7 - 전체 초고 편집/교정 진행 중

Phase 6에서 1~18장 본문 초고 작성과 전체 정합성 점검을 완료했다.

현재 **1~12장 편집/교정을 완료했다.**

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase5-consistency-check.md`
6. `planning/phase6-draft-consistency-check.md`
7. `planning/phase7-editing-plan.md`
8. 각 `chapters/NN/plan.md`
9. `planning/future-topics.md`
10. `STATUS.md`

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

- Cloud Agent는 Local Agent를 대체하지 않는다.
- Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.
- CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.
- CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.
- Runner가 할 수 있으면 Runner에게 맡긴다.
- 작은 Task와 작은 Context를 전달한다.
- Agent에게 개발환경을 설치하게 하지 말고 바로 작업 가능한 환경을 제공한다.
- Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.
- 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.
- Task는 Local 또는 Cloud에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.
- 이벤트가 없으면 Agent도 실행하지 않는다.
- Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.

# Current TOC

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

# Phase 7 Editing Rules

`planning/phase7-editing-plan.md`를 기준으로 편집한다.

```text
중복 압축
→ 용어 통일
→ 장간 연결
→ 예제 정리
→ 수치/근거 구분
→ 문장 교정
```

편집 원칙:

- 최초 설명 장에서 개념을 정의하고 후속 장에서는 적용 중심으로 줄인다.
- 새 개념을 추가하지 않는다.
- 과거 장 번호를 사용하지 않는다.
- 제품별 변경 가능한 사양은 본문 원칙과 분리한다.
- `AUTH-142 / expired token` 예제는 연결 장치로 유지하되 배경 설명은 반복하지 않는다.
- 설명용 숫자는 실제 운영 수치처럼 보이지 않게 구분한다.
- Part 전환부와 장간 연결 문장을 점검한다.

# Phase 7 Progress

## 1~6장

상태: `편집/교정 완료`

- 1장: Cloud Agent 정의와 Remote Development Worker 모델
- 2장: Local / Cloud / Hybrid 판단 재료
- 3장: Reasoning Resource / Execution Resource / Compute·LLM·Human Cost
- 4장: 독립 실행환경 / 비동기 / Developer Blocking Time / 병렬 실행 가치
- 5장: Task Routing Framework
- 6장: 개발 작업 Catalog와 `Runner → Agent → Runner`

## 7~9장

상태: `편집/교정 완료`

```text
7장 Task Contract
→ Small Input / Context

8장 Result Gateway
→ Small Output / Evidence

9장 Prepared Environment
→ Small Startup Overhead
```

주요 정리:

- 7장은 Task의 입력 경계와 Progressive Context에 집중
- 8장은 Raw Artifact 보존과 작은 Evidence Summary에 집중
- 9장은 Prepared Environment / Cache / Snapshot / Fresh State 경계에 집중

## 10장 - Cloud Agent를 Test Runner처럼 사용하기

상태: `편집/교정 완료`

주요 변경:

- 6장의 작업 Catalog 반복을 제거하고 Runner-first 실행 규칙에 집중
- `Deterministic First`와 정상 PASS 경로에서 Agent 제거
- FAIL 후 Infrastructure / Tool-fix / Code Reasoning을 분류
- `Agent-on-failure → Runner 재검증`을 장의 핵심 흐름으로 정리
- Result Gateway 상세는 8장 참조로 축약
- 독립 검증의 병렬 Runner는 소개하되 병렬화 비용은 12장으로 이동
- Sharding은 실행시간과 Startup Overhead를 같이 보는 선택지로 정리

## 11장 - Git, Branch, Worktree, Container로 작업 격리하기

상태: `편집/교정 완료`

주요 변경:

- Git을 Local↔Cloud Handoff Boundary와 Source 기준점으로 정리
- Base SHA / Branch per Task / Verification SHA 연결 강화
- Source Isolation과 Runtime Isolation을 분리
- Worktree는 Source 격리, Container/VM은 Runtime 격리로 명확화
- DB / Port / Temp / Artifact Path도 Task별로 분리
- 같은 파일 / Shared Module / Migration / Schema의 논리적 충돌은 별도 문제로 유지
- Multi-Repository는 필요한 Repository와 각 Base SHA만 연결

## 12장 - 병렬 Worker와 중복 Context 비용

상태: `편집/교정 완료`

주요 변경:

- Parallel Compute와 Parallel Reasoning 구분
- Read-only 검증을 병렬화의 첫 대상으로 정리
- Fan-out 전 Task Dependency와 Change Locality 확인
- Agent 수 증가에 따른 Context Duplication 비용 명시
- Fan-in / Review / Merge / Rework 비용을 병렬화 판단에 포함
- Review Capacity를 병렬도의 상한으로 설명
- Startup Overhead와 Prepared Environment 연결
- Dependency-aware Parallel Group 예제 유지
- Best-of-N은 기본값 N=1인 제한적 고급 기법으로 정리

# Current Edited Flow

```text
7장  Task Contract / Small Input
  ↓
8장  Result Gateway / Small Output / Evidence
  ↓
9장  Prepared Environment / Startup Cost
  ↓
10장 Runner-first / Agent-on-failure
  ↓
11장 Source / Runtime / Evidence Isolation
  ↓
12장 Independent Task Fan-out / Fan-in Cost
  ↓
13장 Local → Cloud → Local Handoff
```

# Phase 7 Remaining Focus

주요 중복 압축 대상:

```text
13 ↔ 15 ↔ 16
5 ↔ 17
```

추가 점검:

- 13→14 Human-driven / Event-driven Handoff 경계
- 15→16 정적 운영 모델 / 시간순 실행 구분
- 17→18 현재 범위 / 미래 자동화 경계
- 설명용 Test Count / 시간 / Retry 횟수 표기
- 영문 용어 표기 통일
- 제품 사례 기준일/공식 출처
- 18장이 Agent Platform 일반론으로 확장되지 않는지 재확인

# Next

1. 13장 `Local → Cloud → Local Handoff` 편집
2. 14장 `Task Queue와 Event-driven Cloud Agent` 편집
3. 15장 `campus-platform Cloud Agent Workflow 설계` 편집
4. 이후 16~18장 편집
5. 전체 편집 완료 후 최종 교정/출판 준비 단계로 전환
