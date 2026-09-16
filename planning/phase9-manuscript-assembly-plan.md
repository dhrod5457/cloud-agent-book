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
- 참고자료와 용어집은 본문 조립 이후 최종 필요 여부를 판단한다.

## 2. 최종 제목

제목:

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

표지용 한 줄 소개:

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

세부 기준은 `manuscript/title-and-positioning.md`에서 관리한다.

## 3. 출판용 Part 구조

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

Part 전환 파일:

- `manuscript/parts/part-01.md`
- `manuscript/parts/part-02.md`
- `manuscript/parts/part-03.md`
- `manuscript/parts/part-04.md`
- `manuscript/parts/part-05.md`
- `manuscript/parts/part-06.md`

### Part I. Cloud Agent를 이해한다

- Cloud Agent를 Remote Development Worker로 정의한다.
- Local과 Cloud의 차이를 실행 위치 관점에서 설명한다.
- Compute와 LLM 자원을 분리한다.
- 독립 실행환경과 비동기 위임의 가치를 설명한다.

### Part II. 어떤 Task를 Cloud로 보낼 것인가

- Task의 실행 위치를 결정한다.
- 실제 개발 작업을 Runner / Agent / Local로 분류한다.
- Cloud Agent에게 전달할 입력을 줄인다.
- 결과는 Evidence 중심으로 반환한다.

### Part III. Cloud 실행환경과 검증을 설계한다

- Cloud Task의 시작 비용을 줄인다.
- 결정론적 검증은 Runner가 우선한다.
- Source / Runtime / Evidence를 격리한다.
- 병렬화의 이득과 Fan-in 비용을 함께 본다.

### Part IV. Local과 Cloud를 연결한다

- 사람이 시작하는 Handoff를 설계한다.
- Event가 Task Candidate를 만드는 자동 경로를 설계한다.
- 두 경로 모두 Git과 Evidence를 공통 경계로 사용한다.

### Part V. 실제 프로젝트에 적용한다

- 앞 장의 원칙을 하나의 프로젝트 운영 모델로 합친다.
- 하나의 기능을 Requirement부터 Merge까지 시간 순서로 따라간다.

### Part VI. Cloud의 한계와 다음 단계를 정한다

- Cloud 이점이 사라지는 조건을 판단한다.
- Local Fallback을 정상적인 Routing으로 설명한다.
- 반복 Workflow가 안정된 뒤 Harness와 Orchestration으로 넘어가는 순서를 제시한다.

## 4. Front Matter 구조

출판 원고 앞부분은 다음 순서를 사용한다.

```text
서문
→ 이 책이 다루는 문제
→ 이 책의 독자
→ 이 책을 읽는 방법
→ 전체 목차
→ Part / Chapter 본문
```

현재 파일:

- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/book-structure.md`
- `manuscript/title-and-positioning.md`

## 5. 읽는 방법

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

## 6. Phase 9 진행 상태

완료:

1. 6개 Part 구조 확정
2. 서문 작성
3. 책 소개 / 독자 대상 작성
4. 읽는 방법 작성
5. 제목 / 부제 / 표지용 한 줄 소개 확정
6. 6개 Part 전환 페이지 작성
7. 최종 원고 결합 순서 초안 작성
8. README / PROJECT를 현재 상태와 동기화

남음:

1. 전체 목차의 출판용 표현 정리
2. 참고자료 구조 확정
3. 용어집 필요 여부 판단 및 필요 시 작성
4. 부록 필요 여부 결정
5. PDF / EPUB / 인쇄 포맷 결정
6. 교정쇄 생성 단계 정의

## 7. Phase 9 완료 조건

- 출판용 Part 구조가 확정되어 있다.
- Front Matter가 준비되어 있다.
- 제목과 부제가 확정되어 있다.
- Part 전환 문구가 준비되어 있다.
- 참고자료 및 용어집 정책이 정해져 있다.
- 1~18장 파일을 어떤 순서로 출판 원고에 결합할지 명확하다.
- 프로젝트 설명 파일이 현재 진행 상태를 반영한다.

Phase 9에서는 본문을 다시 쓰지 않는다. 책으로 조립하는 데 필요한 앞뒤 구조만 만든다.
