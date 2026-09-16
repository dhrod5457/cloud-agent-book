# Current Phase

Phase 7 - 전체 초고 편집/교정 진행 중

Phase 6에서 1~18장 본문 초고 작성과 전체 정합성 점검을 완료했다.

현재 1~9장 편집/교정을 완료했다.

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

## 1~6장

상태: `편집/교정 완료`

핵심 정리:

- 1장: Cloud Agent 정의와 Remote Development Worker 모델에 집중
- 2장: Local / Cloud / Hybrid 판단 재료에 집중
- 3장: Reasoning Resource와 Execution Resource 분리
- 4장: 독립 실행환경 / 비동기 / Developer Blocking Time / 병렬성 가치에 집중
- 5장: Task Routing Framework로 압축
- 6장: 개발 작업 Catalog와 `Runner → Agent → Runner` 구조로 압축

## 7장 - Cloud Agent Task Contract: 작은 Task와 작은 Context

상태: `편집/교정 완료`

주요 변경:

- Task Contract를 `입력 경계`로 정의
- `Task / Goal / Scope / Relevant Files / Forbidden Changes / Validation / Expected Result / Evidence`를 핵심 Contract로 정리
- Progressive Context를 Relevant Files → Dependency → Document → Wider Context 순으로 단순화
- Base SHA / Environment / Budget은 선택적 실행 메타데이터로 정리
- AUTH-142 예제를 이후 장에서 재사용할 기준 예제로 고정
- 16장의 End-to-End 절차와 겹치는 실행 설명 축약

## 8장 - Tool Output을 줄이고 Evidence를 남기기

상태: `편집/교정 완료`

주요 변경:

- `Tool Output → Result Filter → Result Gateway → Evidence` 흐름으로 재구성
- Raw Artifact 보존과 Agent Context 축소를 분리
- `result.json`은 예시 결과 인터페이스로만 유지
- 자연어 Summary와 실행 Evidence 역할 분리
- Demos over Diffs를 UI 검증 순서로 한정
- Failure Fingerprint는 반복 실패 식별까지 다루고 중단/Local Fallback 판단은 17장으로 이동
- 7장 Small Input과 8장 Small Output의 대칭 구조 강화

## 9장 - Prepared Cloud Environment, Cache, Snapshot

상태: `편집/교정 완료`

주요 변경:

- Cold Start를 Provisioning / Checkout / Dependency Restore / Warm-up / First Command로 분해
- Prepared Environment와 Cloud Environment as Code에 집중
- Task-specific Environment 구분
- Reusable Cache와 Fresh Source/Runtime State 경계 강화
- Cache Invalidation / Snapshot / Warm Worker를 준비 비용 관점으로 정리
- 반복 환경 실패는 Prompt가 아니라 Environment를 수정한다는 원칙 유지
- Task Contract에는 설치 절차 대신 Environment 이름을 사용하도록 연결
- 10장 Runner-first로 이어지는 연결부 정리

# Current Edited Flow

```text
7장
Task Contract
→ Small Input / Context
        ↓
8장
Result Gateway
→ Small Output / Evidence
        ↓
9장
Prepared Environment
→ Small Startup Overhead
        ↓
10장
Runner-first / Agent-on-failure
```

# Phase 7 Remaining Focus

반복 압축 대상:

```text
10 ↔ 6 / 8
11 ↔ 12
13 ↔ 15 ↔ 16
5 ↔ 17
```

추가 점검:

- 설명용 Test Count / 시간 / Retry 횟수 표기
- 영문 용어 표기 통일
- 제품 사례 기준일/공식 출처
- 10→11→12→13 연결
- 13→14 Human-driven / Event-driven 경계
- 15→16 정적 운영 모델 / 시간순 실행 구분
- 17→18 결론과 미래 주제 경계
- 18장이 Agent Platform 일반론으로 확장되지 않는지 재확인

# Next

1. 10장 `Cloud Agent를 Test Runner처럼 사용하기` 편집
2. 11장 `Git, Branch, Worktree, Container로 작업 격리하기` 편집
3. 12장 `병렬 Worker와 중복 Context 비용` 편집
4. 이후 13~18장 순차 편집
5. 전체 편집 완료 후 최종 교정/출판 준비 단계로 전환
