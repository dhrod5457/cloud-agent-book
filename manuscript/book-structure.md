# 출판 원고 구조

## 책 정보

제목:

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

표지용 한 줄 소개:

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

세부 포지셔닝은 `manuscript/title-and-positioning.md`를 따른다.

이 문서는 1~18장 본문을 실제 책으로 조립할 때 사용하는 순서를 정의한다.

## Front Matter

1. 서문
2. 이 책이 다루는 문제
3. 이 책의 독자
4. 이 책을 읽는 방법
5. 전체 목차

관련 파일:

- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/title-and-positioning.md`

---

# Part I. Cloud Agent를 이해한다

전환 페이지:

`manuscript/parts/part-01.md`

Cloud Agent를 단순한 원격 AI가 아니라 Repository와 독립 실행환경을 가진 Remote Development Worker로 정의한다.

## 1장. Coding Agent에서 Cloud Worker로

`chapters/01/draft.md`

## 2장. Local Agent와 Cloud Agent

`chapters/02/draft.md`

## 3장. Cloud Session, Container, Compute와 Token

`chapters/03/draft.md`

## 4장. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

`chapters/04/draft.md`

---

# Part II. 어떤 Task를 Cloud로 보낼 것인가

전환 페이지:

`manuscript/parts/part-02.md`

Cloud를 사용할 수 있는 Task와 Cloud로 보내는 것이 유리한 Task를 구분한다.

## 5장. Task Routing: Local인가 Cloud인가

`chapters/05/draft.md`

## 6장. Cloud에 보내기 좋은 개발 작업

`chapters/06/draft.md`

## 7장. Cloud Agent Task Contract: 작은 Task와 작은 Context

`chapters/07/draft.md`

## 8장. Tool Output을 줄이고 Evidence를 남기기

`chapters/08/draft.md`

---

# Part III. Cloud 실행환경과 검증을 설계한다

전환 페이지:

`manuscript/parts/part-03.md`

Task가 Cloud에 들어간 뒤 어떻게 시작하고 실행하고 검증할지 다룬다.

## 9장. Prepared Cloud Environment, Cache, Snapshot

`chapters/09/draft.md`

## 10장. Cloud Agent를 Test Runner처럼 사용하기

`chapters/10/draft.md`

## 11장. Git, Branch, Worktree, Container로 작업 격리하기

`chapters/11/draft.md`

## 12장. 병렬 Worker와 중복 Context 비용

`chapters/12/draft.md`

---

# Part IV. Local과 Cloud를 연결한다

전환 페이지:

`manuscript/parts/part-04.md`

독립 Cloud Task를 기존 개발 Workflow 안에 넣는 방법을 다룬다.

## 13장. Local → Cloud → Local Handoff

`chapters/13/draft.md`

## 14장. Task Queue와 Event-driven Cloud Agent

`chapters/14/draft.md`

---

# Part V. 실제 프로젝트에 적용한다

전환 페이지:

`manuscript/parts/part-05.md`

앞 장의 원칙을 Java/Spring Boot 기반 `campus-platform`에 결합한다.

## 15장. campus-platform Cloud Agent Workflow 설계

`chapters/15/draft.md`

## 16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

`chapters/16/draft.md`

---

# Part VI. Cloud의 한계와 다음 단계를 정한다

전환 페이지:

`manuscript/parts/part-06.md`

Cloud를 계속 사용할지, Local로 되돌릴지, 어느 시점에 자동화를 확장할지 판단한다.

## 17장. Cloud가 항상 정답은 아니다

`chapters/17/draft.md`

## 18장. 다음 단계: Harness와 Orchestration

`chapters/18/draft.md`

---

# Back Matter

다음 단계에서 확정한다.

- 참고자료
- 용어집
- 부록 필요 여부
- 제품별 공식 근거 기준일 표기 방식

제품 기능과 가격처럼 변경 가능한 정보는 본문과 분리하고 Research를 근거로 관리한다.

# 최종 결합 순서

```text
Title / Copyright Page
→ Preface
→ About This Book
→ How to Read
→ Table of Contents
→ Part I transition
→ Chapters 1~4
→ Part II transition
→ Chapters 5~8
→ Part III transition
→ Chapters 9~12
→ Part IV transition
→ Chapters 13~14
→ Part V transition
→ Chapters 15~16
→ Part VI transition
→ Chapters 17~18
→ References
→ Glossary / Appendix if retained
```
