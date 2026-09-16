# Phase 9 - 출판 원고 조립 계획

기준일: 2026-09-16

Phase 8에서 1~18장 본문 최종 교정과 출판 정합성 검사를 완료했다.

Phase 9의 목적은 새 본문을 추가하는 것이 아니라 완성된 장을 실제 책의 읽기 순서와 출판 구조로 묶는 것이다.

## 1. 작업 원칙

- 1~18장 본문 내용과 장 번호는 유지한다.
- 새 개념을 본문에 추가하지 않는다.
- Part는 장 역할을 묶는 상위 구조로만 사용한다.
- Front Matter는 책의 문제의식, 독자 대상, 읽는 방법을 설명한다.
- 제품별 변경 가능한 정보는 Research에 남긴다.
- 제목과 부제는 별도 단계에서 확정한다.
- 참고자료와 용어집은 본문 조립 이후 최종 필요 여부를 판단한다.

## 2. 출판용 Part 구조

기존 장 설계 단계에서는 주제를 세밀하게 나누기 위해 Part가 많았다.

출판 원고에서는 한두 장짜리 Part가 반복되지 않도록 다음 6개 Part로 묶는다.

```text
Part I. Cloud Agent를 이해한다
1~4장

Part II. 어떤 Task를 Cloud로 보낼 것인가
5~8장

Part III. Cloud 실행환경과 검증을 설계한다
9~12장

Part IV. Local과 Cloud를 연결한다
13~14장

Part V. 실제 프로젝트에 적용한다
15~16장

Part VI. Cloud의 한계와 다음 단계를 정한다
17~18장
```

### Part I. Cloud Agent를 이해한다

대상 장:

1. Coding Agent에서 Cloud Worker로
2. Local Agent와 Cloud Agent
3. Cloud Session, Container, Compute와 Token
4. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

역할:

- Cloud Agent를 Remote Development Worker로 정의한다.
- Local과 Cloud의 차이를 실행 위치 관점에서 설명한다.
- Compute와 LLM 자원을 분리한다.
- 독립 실행환경과 비동기 위임의 가치를 설명한다.

### Part II. 어떤 Task를 Cloud로 보낼 것인가

대상 장:

5. Task Routing: Local인가 Cloud인가
6. Cloud에 보내기 좋은 개발 작업
7. Cloud Agent Task Contract: 작은 Task와 작은 Context
8. Tool Output을 줄이고 Evidence를 남기기

역할:

- Task의 실행 위치를 결정한다.
- 실제 개발 작업을 Runner / Agent / Local로 분류한다.
- Cloud Agent에게 전달할 입력을 줄인다.
- 결과는 Evidence 중심으로 반환한다.

### Part III. Cloud 실행환경과 검증을 설계한다

대상 장:

9. Prepared Cloud Environment, Cache, Snapshot
10. Cloud Agent를 Test Runner처럼 사용하기
11. Git, Branch, Worktree, Container로 작업 격리하기
12. 병렬 Worker와 중복 Context 비용

역할:

- Cloud Task의 시작 비용을 줄인다.
- 결정론적 검증은 Runner가 우선한다.
- Source / Runtime / Evidence를 격리한다.
- 병렬화의 이득과 Fan-in 비용을 함께 본다.

### Part IV. Local과 Cloud를 연결한다

대상 장:

13. Local → Cloud → Local Handoff
14. Task Queue와 Event-driven Cloud Agent

역할:

- 사람이 시작하는 Handoff를 설계한다.
- Event가 Task Candidate를 만드는 자동 경로를 설계한다.
- 두 경로 모두 Git과 Evidence를 공통 경계로 사용한다.

### Part V. 실제 프로젝트에 적용한다

대상 장:

15. campus-platform Cloud Agent Workflow 설계
16. 하나의 기능을 Local + Cloud로 끝까지 개발하기

역할:

- 앞 장의 원칙을 하나의 프로젝트 운영 모델로 합친다.
- 하나의 기능을 Requirement부터 Merge까지 시간 순서로 따라간다.

### Part VI. Cloud의 한계와 다음 단계를 정한다

대상 장:

17. Cloud가 항상 정답은 아니다
18. 다음 단계: Harness와 Orchestration

역할:

- Cloud 이점이 사라지는 조건을 판단한다.
- Local Fallback을 정상적인 Routing으로 설명한다.
- 반복 Workflow가 안정된 뒤 Harness와 Orchestration으로 넘어가는 순서를 제시한다.

## 3. Front Matter 구조

출판 원고 앞부분은 다음 순서를 사용한다.

```text
서문
→ 이 책이 다루는 문제
→ 이 책의 독자
→ 이 책을 읽는 방법
→ Part / Chapter 본문
```

초기 파일:

- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/book-structure.md`

## 4. 서문 작성 기준

서문에서는 기술 목록을 먼저 나열하지 않는다.

다음 문제에서 시작한다.

```text
Coding Agent가 코드를 작성할 수 있게 됐다.
그 다음 문제는 어디서 실행할 것인가다.
```

서문이 답해야 할 질문:

- 왜 Local Agent만으로는 설명이 부족한가?
- Cloud Agent의 핵심 가치는 무엇인가?
- 왜 더 많은 Agent보다 Task Routing이 중요한가?
- 이 책이 특정 제품 사용 설명서가 아닌 이유는 무엇인가?

## 5. 책 소개 작성 기준

책 소개는 독자가 얻을 수 있는 판단 능력을 중심으로 작성한다.

독자는 책을 읽은 뒤 다음 질문에 답할 수 있어야 한다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
어떤 Evidence를 받아야 하는가?
언제 Cloud를 중단하고 Local로 돌아올 것인가?
```

## 6. 읽는 방법 작성 기준

처음부터 모든 장을 순서대로 읽는 독자뿐 아니라 필요한 문제부터 찾아오는 독자도 고려한다.

권장 경로:

```text
처음 읽는 독자
→ 1~18장 순서

Cloud 도입 판단이 필요한 독자
→ 1~6장 → 17장

비용 / Token / Compute 최적화
→ 3장 → 7~10장 → 12장

CI / 자동화
→ 10장 → 13~15장 → 18장

실제 적용 예가 먼저 필요한 독자
→ 15~16장 → 필요한 앞 장으로 역참조
```

## 7. Phase 9 후속 작업

1. 6개 Part 구조 확정
2. 서문 작성
3. 책 소개 작성
4. 읽는 방법 작성
5. 제목 / 부제 후보 작성 및 확정
6. Part 전환 페이지 문구 작성
7. 참고자료 구조 정리
8. 용어집 필요 여부 판단
9. 최종 원고 조립 순서 문서화
10. README / PROJECT / STATUS를 실제 상태와 동기화

## 8. Phase 9 완료 조건

- 출판용 Part 구조가 확정되어 있다.
- Front Matter가 준비되어 있다.
- 제목과 부제가 확정되어 있다.
- Part 전환 문구가 준비되어 있다.
- 참고자료 및 용어집 정책이 정해져 있다.
- 1~18장 파일을 어떤 순서로 출판 원고에 결합할지 명확하다.
- 프로젝트 설명 파일이 현재 진행 상태를 반영한다.

Phase 9에서는 본문을 다시 쓰지 않는다. 책으로 조립하는 데 필요한 앞뒤 구조만 만든다.
