# 최종 원고 조립 순서

이 문서는 출판 원고를 하나의 파일이나 PDF / EPUB / 인쇄 원고로 결합할 때 사용하는 실제 파일 순서다.

내부 기획 문서는 포함하지 않는다.

## Front Matter

1. `manuscript/title-page.md`
2. `manuscript/preface.md`
3. `manuscript/about-this-book.md`
4. `manuscript/how-to-read.md`
5. `manuscript/table-of-contents.md`

## Part I. Cloud Agent를 이해한다

6. `manuscript/parts/part-01.md`
7. `chapters/01/draft.md`
8. `chapters/02/draft.md`
9. `chapters/03/draft.md`
10. `chapters/04/draft.md`

## Part II. 어떤 Task를 Cloud로 보낼 것인가

11. `manuscript/parts/part-02.md`
12. `chapters/05/draft.md`
13. `chapters/06/draft.md`
14. `chapters/07/draft.md`
15. `chapters/08/draft.md`

## Part III. Cloud 실행환경과 검증을 설계한다

16. `manuscript/parts/part-03.md`
17. `chapters/09/draft.md`
18. `chapters/10/draft.md`
19. `chapters/11/draft.md`
20. `chapters/12/draft.md`

## Part IV. Local과 Cloud를 연결한다

21. `manuscript/parts/part-04.md`
22. `chapters/13/draft.md`
23. `chapters/14/draft.md`

## Part V. 실제 프로젝트에 적용한다

24. `manuscript/parts/part-05.md`
25. `chapters/15/draft.md`
26. `chapters/16/draft.md`

## Part VI. Cloud의 한계와 다음 단계를 정한다

27. `manuscript/parts/part-06.md`
28. `chapters/17/draft.md`
29. `chapters/18/draft.md`

## Back Matter

30. `manuscript/appendix/task-contract-evidence-template.md`
31. `manuscript/glossary.md`
32. `manuscript/references.md`

## 출판 대상에서 제외하는 내부 문서

다음 파일은 기획과 검증을 위한 Source of Truth이지만 독자용 원고에는 직접 포함하지 않는다.

```text
planning/*
research/*
review/*
PROJECT.md
STATUS.md
README.md
manuscript/title-and-positioning.md
```

`manuscript/title-and-positioning.md`의 확정 결과만 `manuscript/title-page.md`에 반영한다.

## 결합 규칙

- 각 Part 전환 파일은 해당 Part의 첫 장 앞에 둔다.
- 장 번호와 제목은 `planning/toc.md`와 일치시킨다.
- Markdown 내부 상대 경로는 PDF / EPUB 변환 전 확인한다.
- 제품 URL과 공식 자료 기준일은 References에서 관리한다.
- 출판 원고 생성 뒤 본문을 다시 확장하지 않는다.
- 교정쇄에서 발견된 오탈자, 사실 오류, 깨진 링크만 원본 파일에 역반영한다.

최종 조립 대상은 총 32개 파일이다.
