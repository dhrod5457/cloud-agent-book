# Current Phase

Phase 6 완료 - 1~18장 본문 초고 작성 및 전체 정합성 점검 완료

다음 단계는 전체 초고 편집/교정이다.

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase5-consistency-check.md`
6. `planning/phase6-draft-consistency-check.md`
7. 각 `chapters/NN/plan.md`
8. `planning/future-topics.md`
9. `STATUS.md`

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

> Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.

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

# Phase 6 Result

Phase 6 완료.

- `chapters/01/draft.md` ~ `chapters/18/draft.md` 작성 완료
- 1~18장 설계 대비 검토 완료
- 전체 역할/중복/용어/범위 정합성 점검 완료
- `planning/phase6-draft-consistency-check.md` 작성
- 1장 제목을 `Coding Agent에서 Cloud Worker로`로 통일
- Agent Platform 일반론은 18장의 미래 전망 수준으로 제한

# Phase 6 Chapter Flow

```text
1~4장
Cloud Agent 정의 / Local-Cloud 차이 / Compute-Token / Cloud 실행 가치

5~6장
Task Routing / 실제 개발 작업 Catalog

7~8장
Small Input / Context
→ Small Output / Evidence

9~10장
Prepared Environment
→ Runner-first / Agent-on-failure

11~12장
Source / Runtime 격리
→ 독립 Task Fan-out / Fan-in 비용

13~14장
Human-driven Handoff
→ Event-driven Handoff

15~16장
campus-platform 운영 모델
→ 하나의 기능 End-to-End Timeline

17장
Cloud 부적합 조건 / Local Fallback

18장
Harness / Routing Automation / Orchestration 미래 방향
```

# Phase 6 Consistency Result

구조적 충돌 없음.

의도적으로 유지하는 반복:

- Cloud Agent = Remote Development Worker
- Runner-first
- Task Contract
- Result Gateway / Evidence
- Git Handoff
- Local Fallback
- `AuthService expired token` 반복 예제

다음 편집 단계에서 압축할 반복 구간:

```text
2 ↔ 5 ↔ 17
3 ↔ 8 ↔ 10
4 ↔ 12
7 ↔ 16
13 ↔ 15 ↔ 16
```

# Next - 전체 초고 편집/교정

새 개념 추가보다 다음 작업을 우선한다.

1. 장간 중복 설명 압축
2. 핵심 용어 표기 통일
3. 설명용 수치와 실제 근거 수치 구분
4. 장간 참조 번호 점검
5. 제품 사례의 기준일/공식 출처 재검증
6. 반복 예제의 중복 문장 축약
7. 문장 길이와 책 문체 교정
8. 1~18장 전체 흐름을 다시 읽고 Part 전환부 보강

편집 단계에서도 책의 범위를 Agent Platform 일반론으로 확장하지 않는다.
