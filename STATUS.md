# Current Phase

Phase 7 - 전체 초고 편집/교정 진행 중

Phase 6에서 1~18장 본문 초고 작성과 전체 정합성 점검을 완료했다.

현재 1~6장 편집/교정을 완료했다.

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

`planning/toc.md`가 최신 목차다.

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

# Phase 6 Result

Phase 6 완료.

- `chapters/01/draft.md` ~ `chapters/18/draft.md` 작성 완료
- 1~18장 설계 대비 검토 완료
- 전체 역할/중복/용어/범위 정합성 점검 완료
- `planning/phase6-draft-consistency-check.md` 작성
- 1장 제목을 `Coding Agent에서 Cloud Worker로`로 통일
- Agent Platform 일반론은 18장의 미래 전망 수준으로 제한

# Phase 7 Editing Rules

`planning/phase7-editing-plan.md`를 기준으로 편집한다.

우선순위:

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
- `AuthService expired token` 반복 예제는 연결 장치로 유지하되 배경 설명은 반복하지 않는다.
- 한 문단에 하나의 판단을 두고 같은 의미의 문장을 연속해서 반복하지 않는다.
- Part 전환부와 장간 연결 문장을 점검한다.

# Phase 7 Progress

## 1장 - Coding Agent에서 Cloud Worker로

상태: `편집/교정 완료`

- Cloud Agent 정의와 Remote Development Worker 모델에 집중
- Task Contract / Result Gateway / Runner 상세를 뒤 장으로 이동
- 제품 사례를 공통 실행 모델 설명 수준으로 축약

## 2장 - Local Agent와 Cloud Agent

상태: `편집/교정 완료`

- Local / Cloud / Hybrid 판단 재료에 집중
- 5장의 Routing Framework와 중복되는 절차 설명 축약
- 10장의 Runner-first와 11~13장의 Git/Handoff 상세를 예고 수준으로 축약

## 3장 - Cloud Session, Container, Compute와 Token

상태: `편집/교정 완료`

- Reasoning Resource와 Execution Resource 분리 강화
- Build wall-clock time과 LLM Usage 분리
- Tool Output 상세는 8장, Runner-first 상세는 10장으로 이동
- Parallel Compute / Parallel Reasoning과 Compute / LLM / Human Cost 구조로 정리

## 4장 - 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

상태: `편집/교정 완료`

주요 변경:

- 독립 실행환경 / 비동기 위임 / Developer Blocking Time / 병렬 실행 가치에 집중
- 병렬화의 중복 Context, Merge, Review, Fan-in 비용은 12장으로 이동
- Agent Execution Time과 Developer Blocking Time을 핵심 측정축으로 유지
- Local Resource Occupancy를 Cloud 활용 효과에 포함
- 병렬화는 독립 Task가 있을 때만 가치가 있다는 조건만 유지

## 5장 - Task Routing: Local인가 Cloud인가

상태: `편집/교정 완료`

주요 변경:

- Task 시작 시점의 Routing Framework에 집중
- `Hard Constraint → Runner 가능 여부 → Scope/Validation → Handoff → Steering/Context → Conflict/Overhead` 순서로 재구성
- 17장의 Cloud 중단/Local Fallback 상세를 제거
- Hybrid를 단계별 Routing으로 정의
- Score는 보조 수단이고 Hard Constraint가 우선한다는 기준 강화

## 6장 - Cloud에 보내기 좋은 개발 작업

상태: `편집/교정 완료`

주요 변경:

- 작업별 장문 설명을 `기본 실행 주체 + Agent 호출 조건 + Evidence` 중심 Catalog로 압축
- Build / Unit / Integration / E2E / Docker / Static / Migration / Refactoring / Bug / Documentation / PR Review / Dependency / CI Failure 정리
- Runner / Agent / Local-Hybrid가 하나의 Task 안에서 단계별로 바뀔 수 있음을 유지
- `Runner → 필요한 순간에 Agent → 다시 Runner`를 장의 실행 원칙으로 정리
- campus-platform 반복 Task를 RUN-* / FIX-BUG / REFACTOR-MODULE Catalog로 연결

# Phase 7 Remaining Focus

반복 압축 대상:

```text
7 ↔ 16
13 ↔ 15 ↔ 16
```

추가 점검:

- 설명용 Test Count / 시간 / Retry 횟수 표기
- 영문 용어 표기 통일
- 제품 사례 기준일/공식 출처
- 7→8→9→10 연결
- 11→12→13 연결
- 15→16→17 연결
- 18장 결론이 Agent Platform 일반론으로 확장되지 않는지 재확인

# Next

1. 7장 `Cloud Agent Task Contract: 작은 Task와 작은 Context` 편집
2. 8장 `Tool Output을 줄이고 Evidence를 남기기` 편집
3. 9장 `Prepared Cloud Environment, Cache, Snapshot` 편집
4. 이후 10~18장 순차 편집
5. 전체 편집 완료 후 최종 교정/출판 준비 단계로 전환
