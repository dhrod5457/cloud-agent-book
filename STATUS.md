# Current Phase

Phase 8 - 최종 교정 / 출판 준비 진행 중

Phase 7에서 1~18장 전체 편집/교정과 정합성 점검을 완료했다.

현재 **1~6장 최종 교정을 완료했다.**

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase5-consistency-check.md`
6. `planning/phase6-draft-consistency-check.md`
7. `planning/phase7-editing-plan.md`
8. `planning/phase7-editing-consistency-check.md`
9. `planning/phase8-publication-checklist.md`
10. 각 `chapters/NN/plan.md`
11. `planning/future-topics.md`
12. `STATUS.md`

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

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, Test와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

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
- Local Fallback은 실패가 아니라 Routing의 일부다.

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

# Phase 5 Result

Phase 5 완료.

- 1~18장 설계 완료
- 장별 역할/중복/용어/범위 정합성 점검 완료
- `planning/phase5-consistency-check.md` 작성

# Phase 6 Result

Phase 6 완료.

- `chapters/01/draft.md` ~ `chapters/18/draft.md` 초고 작성 완료
- 설계 대비 장별 검토 완료
- 전체 초고 정합성 점검 완료
- `planning/phase6-draft-consistency-check.md` 작성

# Phase 7 Result

Phase 7 완료.

- 1~18장 전체 편집/교정 완료
- 중복 설명 압축
- 장별 역할 경계 명확화
- 핵심 용어 표기 정리
- 설명용 숫자와 제품별 변경 가능한 사실의 본문 원칙 분리
- 장간 연결부 정리
- `AUTH-142 / expired token` 반복 예제를 공통 연결 사례로 정리
- Agent Platform 일반론 확장 방지
- `planning/phase7-editing-consistency-check.md` 작성

# Final Edited Flow

```text
1장  Cloud Agent = Remote Development Worker
  ↓
2장  Local / Cloud / Hybrid
  ↓
3장  Compute / LLM / Human Cost
  ↓
4장  독립 실행환경 / 비동기 / 병렬 실행 가치
  ↓
5장  Task Routing
  ↓
6장  개발 작업 Catalog
  ↓
7장  Task Contract / Small Input
  ↓
8장  Result Gateway / Evidence / Small Output
  ↓
9장  Prepared Environment / Startup Cost
  ↓
10장 Runner-first / Agent-on-failure
  ↓
11장 Source / Runtime / Evidence Isolation
  ↓
12장 Independent Task Fan-out / Fan-in Cost
  ↓
13장 Human-driven Handoff
  ↓
14장 Event-driven Handoff
  ↓
15장 campus-platform 운영 모델
  ↓
16장 End-to-End 기능 Timeline
  ↓
17장 Cloud Stop / Local Fallback / 재Routing
  ↓
18장 Harness / Orchestration 미래 방향 + 결론
```

# Phase 8 Rules

`planning/phase8-publication-checklist.md`를 기준으로 진행한다.

```text
문장 단위 교정
→ 제목 / 절 제목 / 용어 표기 통일
→ 코드블록 / 표 형식 통일
→ 장간 참조 검사
→ 설명용 수치 표기 검사
→ 제품 사례 / 공식 출처 검사
→ 참고자료 형식 통일
→ 도입 / 결론 연결 검사
→ 최종 목차와 본문 제목 대조
```

새 구조와 새 개념은 추가하지 않는다.

# Phase 8 Progress

## 1~3장

상태: `최종 교정 완료`

주요 교정:

- 1장: `Base SHA`, Task/Evidence 핵심 문장, Build / Test 표기 통일
- 2장: 결정론적 검증을 `Cloud Runner`로 명확화하고 설명용 시간 예시 표기
- 3장: Reasoning / Execution Resource, 설명용 수치, Compute / LLM / Human 비용 표기 정리

## 4장 - 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

상태: `최종 교정 완료`

주요 교정:

- 설명용 시간 예시를 명시
- `Cloud Runner`, `Base SHA`, `DB Schema`, `CPU / RAM / Disk` 표기 통일
- Parallel Compute와 Parallel Reasoning 표현 유지
- 12장의 병렬화 비용 참조 점검

## 5장 - Task Routing: Local인가 Cloud인가

상태: `최종 교정 완료`

주요 교정:

- `Local 상태`, `DB Schema`, Hard Constraint 표기 정리
- Routing Decision Matrix와 판단 순서 용어 통일
- Score를 실제 기준값이 아닌 설명용 예로 명시
- `Runner-first`와 Cloud Agent 후보 구분 유지

## 6장 - Cloud에 보내기 좋은 개발 작업

상태: `최종 교정 완료`

주요 교정:

- `Cloud Runner → Cloud Agent → Cloud Runner 재검증` 표현 통일
- Build / Unit / Integration / E2E / Docker / Migration의 실행 주체 표기 정리
- 기본 Catalog의 Evidence 표기 통일
- Tool-first / Runner-first / Agent-on-failure 용어 연결 확인

# Phase 8 Remaining Focus

```text
7~9장
Task Contract / Evidence / Prepared Environment 형식 통일

10~12장
Runner / Isolation / Parallel Cost 용어 통일

13~15장
Handoff / Event / 운영 모델 장간 참조 점검

16~18장
Timeline / Fallback / 결론 문장 최종 교정
```

전체 장 교정 후 다음을 별도 점검한다.

- 제품명을 직접 언급한 문장의 공식 출처와 기준일
- 장 제목과 `planning/toc.md` 일치
- 장간 참조 번호
- 참고자료 형식
- 설명용 수치 표기
- 코드블록 언어 지정
- 표 형식

# Next

1. 7장 `Cloud Agent Task Contract: 작은 Task와 작은 Context` 최종 교정
2. 8장 `Tool Output을 줄이고 Evidence를 남기기` 최종 교정
3. 9장 `Prepared Cloud Environment, Cache, Snapshot` 최종 교정
4. 이후 10~18장 순차 진행
5. 전체 장 교정 완료 후 제품 출처 / 참조 / 목차 / 참고자료 최종 검사
