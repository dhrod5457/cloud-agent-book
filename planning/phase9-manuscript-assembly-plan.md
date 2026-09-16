# Phase 9 - 출판 원고 조립 계획

기준일: 2026-09-16
상태: 완료

Phase 8에서 교정이 끝난 1~18장 본문을 실제 책 구조로 조립했다.

Phase 9에서는 새 본문을 추가하지 않고 Front Matter, Part, Back Matter, 제목, 전체 목차, 최종 결합 순서와 포맷 정책을 확정했다.

## 최종 책 정보

제목:

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

표지용 한 줄 소개:

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

관련 파일:

- `manuscript/title-page.md`
- `manuscript/title-and-positioning.md`

## 출판용 Part 구조

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

## Front Matter

준비 완료:

- `manuscript/title-page.md`
- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/table-of-contents.md`

## Main Text

본문은 다음 18개 파일을 사용한다.

```text
chapters/01/draft.md
...
chapters/18/draft.md
```

Phase 9에서는 구조 조립을 이유로 본문을 확장하지 않았다.

## Back Matter

### Appendix A

- `manuscript/appendix/task-contract-evidence-template.md`

Task Contract, Runner 입력/출력, Evidence, Handoff, Local Fallback의 실무 Template을 제공한다.

### Glossary

- `manuscript/glossary.md`

책 전체에서 의미가 고정된 핵심 용어만 정리한다.

### References

- `manuscript/references.md`

GitHub, OpenAI, Anthropic 공식 자료와 Engineering 사례를 관리한다.

## 최종 조립 순서

Source of Truth:

- `manuscript/assembly-manifest.md`

최종 원고는 다음 구성의 32개 파일로 조립한다.

```text
5 Front Matter
+ 6 Part Transition
+ 18 Chapter
+ 3 Back Matter
= 32 files
```

## 포맷 정책

기준 문서:

- `manuscript/publication-format.md`

산출물 우선순위:

```text
1. Markdown Source
2. PDF Proof
3. EPUB
4. Print-ready PDF
```

PDF / EPUB에서 직접 본문을 수정하지 않고 모든 교정은 Markdown Source에 역반영한다.

## Phase 9 완료 점검

완료 문서:

- `planning/phase9-manuscript-assembly-check.md`

완료 항목:

- [x] 출판용 Part 구조
- [x] Front Matter
- [x] 제목 / 부제
- [x] Part 전환문
- [x] 출판용 전체 목차
- [x] References
- [x] Glossary
- [x] Appendix
- [x] 최종 결합 순서
- [x] 포맷 정책
- [x] 교정쇄 생성 절차

## 다음 단계

Phase 10 - 교정쇄 생성 / 시각 검수

기준 문서:

- `planning/phase10-proof-plan.md`

다음 작업:

```text
assembly-manifest.md
→ manuscript/master.md 생성
→ PDF Proof 생성
→ 페이지 렌더링 검수
→ review/proof-01.md 기록
→ 원본 Markdown 수정
→ PDF 재생성
```
