# Current Phase

Phase 7 - 전체 초고 편집/교정 진행 중

Phase 6에서 1~18장 본문 초고 작성과 전체 정합성 점검을 완료했다.

현재 **1~15장 편집/교정을 완료했다.**

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

## 1~12장

상태: `편집/교정 완료`

핵심 흐름:

```text
1장  Cloud Agent 정의
2장  Local / Cloud / Hybrid 비교
3장  Compute / LLM / Human Cost
4장  독립 실행환경 / 비동기 / 병렬 실행 가치
5장  Task Routing Framework
6장  개발 작업 Catalog
7장  Task Contract / Small Input
8장  Result Gateway / Small Output / Evidence
9장  Prepared Environment / Startup Cost
10장 Runner-first / Agent-on-failure
11장 Source / Runtime / Evidence Isolation
12장 Independent Task Fan-out / Fan-in Cost
```

## 13장 - Local → Cloud → Local Handoff

상태: `편집/교정 완료`

주요 변경:

- Local/Cloud 특성 반복 설명을 제거하고 Handoff 경계에 집중
- Local→Cloud 입력을 `Git + Task Contract`로 정리
- Cloud→Local 반환을 `Evidence + Commit / PR`로 정리
- 미커밋 Local State 처리 원칙 명시
- Internal Validation을 Hybrid 마지막 Stage로 정리
- Multi-Repository는 필요한 Repository와 각 SHA만 전달
- Local Fallback 상세 기준은 17장으로 이동
- Handoff의 목적을 Developer Blocking Time 감소와 연결

## 14장 - Task Queue와 Event-driven Cloud Agent

상태: `편집/교정 완료`

주요 변경:

- Event를 Agent 호출이 아니라 Task Candidate로 정의
- `Event → Dedup / Classification → Runner / Tool → 필요 시 Agent` 공통 흐름으로 통일
- CI / Review / Nightly / Dependency Update를 같은 실행 규칙으로 연결
- 같은 SHA + Failure Fingerprint 기반 중복 Task 억제
- Agent Push → CI FAIL → Agent 재호출 Loop에 Budget/Fingerprint 종료 조건 적용
- Issue가 불명확하면 Local Investigation으로 보내는 경계 유지
- 결과는 기존 PR Commit 또는 Review 가능한 PR로 반환

## 15장 - campus-platform Cloud Agent Workflow 설계

상태: `편집/교정 완료`

주요 변경:

- 앞 장의 개념을 다시 설명하지 않고 하나의 운영 모델로 통합
- `Local Agent / Cloud Runner / Cloud Agent` 역할표로 실행 주체 구분
- Task Type과 Prepared Environment 연결
- 반복 작업을 Task Catalog로 정리
- Task ID / Base SHA / Result SHA / Verification / Evidence / PR 상태 연결
- 같은 SHA에서 Unit / Integration / Docker / E2E 병렬 검증
- Result Gateway를 프로젝트 공통 반환 인터페이스로 사용
- Failure를 Environment 경로와 Code Agent 경로로 분리
- Tibero / HSM / Internal API를 Internal Validation Stage로 유지
- Event-driven Task를 동일 운영 모델의 입력 채널로 통합
- 운영 플랫폼 구축보다 반복 가능한 Workflow를 먼저 만든다는 원칙 유지

# Current Edited Flow

```text
Task Routing
      ↓
Task Contract / Small Input
      ↓
Prepared Environment
      ↓
Runner-first
      ↓
필요한 경우 Cloud Agent
      ↓
Evidence / Small Output
      ↓
Git / Task Isolation
      ↓
독립 Task 병렬화
      ↓
Local → Cloud → Local Handoff
      ↓
Event-driven Task Input
      ↓
campus-platform 운영 모델
```

# Phase 7 Remaining Focus

남은 장:

```text
16장 하나의 기능을 Local + Cloud로 끝까지 개발하기
17장 Cloud가 항상 정답은 아니다
18장 다음 단계: Harness와 Orchestration
```

마지막 편집 기준:

- 16장은 15장의 운영 모델을 다시 설명하지 않고 시간순 End-to-End 사례에 집중
- 17장은 5장의 시작 Routing을 반복하지 않고 Cloud 중단 / 역판단 / Local Fallback에 집중
- 18장은 현재 책의 범위를 닫고 Harness / Orchestration을 미래 방향으로만 소개
- 제품별 변경 가능한 사실과 출처를 최종 점검
- 장간 참조 번호와 용어 표기를 최종 통일

# Next

1. 16장 `하나의 기능을 Local + Cloud로 끝까지 개발하기` 편집
2. 17장 `Cloud가 항상 정답은 아니다` 편집
3. 18장 `다음 단계: Harness와 Orchestration` 편집
4. 1~18장 전체 편집 결과 최종 정합성 점검
5. Phase 7 완료 후 최종 교정/출판 준비 단계로 전환
