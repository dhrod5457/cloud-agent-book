# Current Phase

Phase 9 - 출판 원고 조립 진행 중

Phase 8에서 1~18장 본문 최종 교정과 출판 정합성 점검을 완료했다.

현재는 새 본문을 작성하는 단계가 아니라 **완성된 18개 장을 실제 책 구조로 묶는 단계**다.

Phase 9 첫 작업으로 다음을 완료했다.

- 출판용 6개 Part 구조 정의
- 서문 작성
- 책 소개 / 독자 대상 작성
- 읽는 방법 작성
- 출판 원고 조립 순서 문서화
- README / PROJECT의 과거 진행 상태 정리

# Source of Truth

현재 우선순위는 다음과 같다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase8-publication-consistency-check.md`
6. `planning/phase9-manuscript-assembly-plan.md`
7. `manuscript/book-structure.md`
8. `manuscript/preface.md`
9. `manuscript/about-this-book.md`
10. `manuscript/how-to-read.md`
11. 각 `chapters/NN/draft.md`
12. `STATUS.md`

과거 Agent-Native 독립 장 설계와 초기 22~23장 체계는 현행 18장 원고보다 우선하지 않는다.

# Book Direction

책의 중심 주제는 **Cloud Agent 활용과 Local + Cloud 개발 Workflow**다.

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
- Agent에게 개발환경을 설치하게 하지 않고 바로 작업 가능한 환경을 제공한다.
- 결과는 자연어 완료 보고보다 Evidence와 Artifact로 확인한다.
- 병렬화의 대상은 Agent가 아니라 독립적으로 실행하고 검증할 수 있는 Task다.
- Task는 Local 또는 Cloud에 영구적으로 속하지 않는다.
- Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.
- Local Fallback은 실패가 아니라 Routing의 일부다.

# Phase Results

## Phase 5

- 1~18장 설계 완료
- 장별 역할과 범위 정합성 점검 완료

## Phase 6

- `chapters/01/draft.md` ~ `chapters/18/draft.md` 초고 작성 완료
- 전체 초고 정합성 점검 완료

## Phase 7

- 1~18장 전체 편집 완료
- 중복 설명 압축
- 장별 역할 경계 명확화
- `AUTH-142 / expired token` 공통 사례 연결
- Agent Platform 일반론 확장 방지

## Phase 8

- 1~18장 최종 교정 완료
- 용어 / 제목 / 코드블록 / 표 / 장간 참조 점검
- 설명용 수치 표기 점검
- 제품 공식 근거 재검증
- `planning/toc.md`와 장 제목 **18 / 18 일치** 확인
- 깨진 Anthropic Research 내부 경로 복구
- `planning/phase8-publication-consistency-check.md` 작성

# Phase 9 Publication Structure

출판용 원고는 기존 장 번호를 유지하면서 6개 Part로 묶는다.

## Part I. Cloud Agent를 이해한다

1. Coding Agent에서 Cloud Worker로
2. Local Agent와 Cloud Agent
3. Cloud Session, Container, Compute와 Token
4. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

## Part II. 어떤 Task를 Cloud로 보낼 것인가

5. Task Routing: Local인가 Cloud인가
6. Cloud에 보내기 좋은 개발 작업
7. Cloud Agent Task Contract: 작은 Task와 작은 Context
8. Tool Output을 줄이고 Evidence를 남기기

## Part III. Cloud 실행환경과 검증을 설계한다

9. Prepared Cloud Environment, Cache, Snapshot
10. Cloud Agent를 Test Runner처럼 사용하기
11. Git, Branch, Worktree, Container로 작업 격리하기
12. 병렬 Worker와 중복 Context 비용

## Part IV. Local과 Cloud를 연결한다

13. Local → Cloud → Local Handoff
14. Task Queue와 Event-driven Cloud Agent

## Part V. 실제 프로젝트에 적용한다

15. campus-platform Cloud Agent Workflow 설계
16. 하나의 기능을 Local + Cloud로 끝까지 개발하기

## Part VI. Cloud의 한계와 다음 단계를 정한다

17. Cloud가 항상 정답은 아니다
18. 다음 단계: Harness와 Orchestration

세부 조립 순서:

`manuscript/book-structure.md`

# Front Matter

현재 작성 완료:

- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`

Front Matter의 역할은 다음과 같다.

```text
서문
→ 왜 Cloud Agent Workflow가 필요한가

책 소개
→ 무엇을 다루고 누구를 위한 책인가

읽는 방법
→ 독자 목적별 장 선택 경로
```

# Publication Rules

Phase 9에서는 1~18장 본문을 다시 확장하지 않는다.

본문 변경은 다음 경우로 제한한다.

```text
명확한 오탈자
사실 오류
깨진 링크
출판 조립 과정에서 발견된 참조 오류
```

제품별 변경 가능한 가격, CPU / RAM, Session 제한은 본문에 새로 고정하지 않는다.

# Current Manuscript Components

```text
Front Matter
- preface.md
- about-this-book.md
- how-to-read.md

Book Structure
- book-structure.md

Main Text
- chapters/01/draft.md
  ...
- chapters/18/draft.md

Research
- GitHub / OpenAI / Anthropic 공식 근거
```

# Phase 9 Remaining Work

1. 표지용 제목 / 부제 후보 작성 및 확정
2. 6개 Part 전환 페이지 문구 작성
3. 전체 목차의 출판용 표현 정리
4. 참고자료 구조 확정
5. 용어집 필요 여부 판단 및 필요 시 작성
6. 부록 필요 여부 결정
7. 최종 원고 결합 순서와 파일 목록 확정
8. PDF / EPUB / 인쇄 포맷 결정
9. 교정쇄 생성 단계로 이동

# Next

다음 작업은 **제목 / 부제 / 표지용 한 줄 소개**다.

제목은 특정 제품명보다 책의 핵심 판단인 `Cloud Agent`, `Local + Cloud`, `Task Routing`, `Development Workflow`를 중심으로 검토한다.

제목 확정 후 6개 Part의 전환 문구를 작성한다.
