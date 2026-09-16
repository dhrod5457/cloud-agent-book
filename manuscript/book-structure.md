# 출판 원고 구조

## 책 정보

제목:

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

표지용 한 줄 소개:

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

내부 포지셔닝 기준은 `manuscript/title-and-positioning.md`에서 관리하고, 실제 출판 원고에는 `manuscript/title-page.md`를 사용한다.

## Front Matter

출판 순서:

1. 제목 페이지
2. 서문
3. 이 책이 다루는 문제와 독자
4. 이 책을 읽는 방법
5. 전체 목차

파일:

- `manuscript/title-page.md`
- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/table-of-contents.md`

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

본문 반복을 피하기 위해 세 항목만 유지한다.

## 부록 A. Task Contract / Evidence / Handoff 템플릿

`manuscript/appendix/task-contract-evidence-template.md`

실무에서 복사해 사용할 수 있는 Task Contract, Runner 입력/출력, Evidence, Handoff, Local Fallback Template을 제공한다.

## 용어집

`manuscript/glossary.md`

책 전체에서 의미를 고정해서 사용하는 핵심 용어만 정리한다.

## 참고자료

`manuscript/references.md`

GitHub, OpenAI, Anthropic 등의 공식 자료와 본문 원칙에 사용한 Engineering 사례를 관리한다. 변경 가능한 제품 사양은 출판 직전에 다시 확인한다.

# 최종 조립 기준

실제 파일 단위 결합 순서는 다음 문서를 Source of Truth로 사용한다.

`manuscript/assembly-manifest.md`

현재 출판 원고는 총 32개 파일로 조립한다.

```text
Title Page
→ Preface
→ About This Book
→ How to Read
→ Table of Contents
→ Part I transition → Chapters 1~4
→ Part II transition → Chapters 5~8
→ Part III transition → Chapters 9~12
→ Part IV transition → Chapters 13~14
→ Part V transition → Chapters 15~16
→ Part VI transition → Chapters 17~18
→ Appendix A
→ Glossary
→ References
```

기획과 Research 문서는 최종 독자용 원고에 직접 포함하지 않는다.
