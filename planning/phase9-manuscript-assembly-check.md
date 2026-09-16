# Phase 9 - 출판 원고 조립 완료 점검

기준일: 2026-09-16

Phase 9의 목적은 Phase 8에서 교정이 끝난 1~18장 본문을 실제 책의 Front Matter, Part, Back Matter와 연결해 출판 가능한 원고 구조로 만드는 것이었다.

본문에 새 개념을 추가하지 않고 출판 구조만 조립하는 원칙을 유지했다.

## 1. 책 정보

제목:

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

표지용 한 줄 소개:

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

독자용 제목 페이지:

- `manuscript/title-page.md`

내부 포지셔닝 문서:

- `manuscript/title-and-positioning.md`

## 2. Front Matter

준비 완료:

- `manuscript/title-page.md`
- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/table-of-contents.md`

Front Matter는 제품 기능 소개보다 책의 핵심 질문을 먼저 제시한다.

```text
이 Task는 Local에서 해야 하는가,
Cloud로 보내야 하는가?
```

## 3. Part 구조

18개 Chapter를 6개 Part로 묶었다.

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

Part 전환 페이지:

- `manuscript/parts/part-01.md`
- `manuscript/parts/part-02.md`
- `manuscript/parts/part-03.md`
- `manuscript/parts/part-04.md`
- `manuscript/parts/part-05.md`
- `manuscript/parts/part-06.md`

각 전환문은 앞 장 내용을 다시 설명하지 않고 다음 Part의 판단 질문과 역할만 연결한다.

## 4. Main Text

본문은 다음 18개 파일을 그대로 사용한다.

```text
chapters/01/draft.md
...
chapters/18/draft.md
```

Phase 9에서 구조 조립을 이유로 본문을 확장하지 않았다.

본문 수정 허용 범위는 다음으로 제한한다.

```text
명확한 오탈자
사실 오류
깨진 링크
출판 조립에서 발견한 참조 오류
```

## 5. Back Matter

다음 세 항목만 유지한다.

### Appendix A

- `manuscript/appendix/task-contract-evidence-template.md`

본문 이론을 반복하지 않고 Task Contract, Cloud Runner, Evidence, Handoff, Local Fallback의 복사용 Template만 제공한다.

### Glossary

- `manuscript/glossary.md`

Cloud Agent, Cloud Runner, Task Contract, Evidence, Result SHA, Local Fallback처럼 책 전체에서 의미가 고정된 용어만 정리한다.

### References

- `manuscript/references.md`

GitHub, OpenAI, Anthropic 등의 공식 자료와 본문 원칙을 확인하는 데 사용한 Engineering 사례를 정리한다.

제품별 변경 가능한 정보는 출판 직전에 다시 확인한다.

## 6. 최종 조립 순서

Source of Truth:

- `manuscript/assembly-manifest.md`

최종 독자용 원고는 32개 파일로 조립한다.

```text
5 Front Matter
+ 6 Part Transition
+ 18 Chapter
+ 3 Back Matter
= 32 files
```

기획, Research, Review 문서는 출판 대상에 직접 포함하지 않는다.

## 7. 출판 포맷

기준 문서:

- `manuscript/publication-format.md`

원본과 파생 산출물의 우선순위는 다음과 같다.

```text
1. Markdown Source
2. PDF Proof
3. EPUB
4. Print-ready PDF
```

PDF 또는 EPUB에서 직접 내용을 수정하지 않는다. 교정 결과는 Markdown Source에 역반영한다.

인쇄용 PDF는 판형과 출판사 사양이 정해진 뒤 만든다.

## 8. Phase 9 완료 조건 점검

- [x] 출판용 Part 구조 확정
- [x] Front Matter 준비
- [x] 제목 / 부제 확정
- [x] Part 전환 문구 준비
- [x] 출판용 전체 목차 준비
- [x] References 정책 및 파일 준비
- [x] Glossary 정책 및 파일 준비
- [x] Appendix 필요 범위 확정 및 파일 준비
- [x] 최종 파일 결합 순서 확정
- [x] 출판 포맷 정책 확정
- [x] 교정쇄 생성 절차 정의

## 9. Phase 9 결과

Phase 9 완료.

이제 원고 구조를 다시 설계하지 않는다.

다음 단계에서는 `manuscript/assembly-manifest.md` 순서대로 실제 원고를 결합하고 PDF 교정쇄를 생성한 뒤 시각적으로 확인한다.

교정쇄 단계에서 발견한 문제는 원본 Markdown에 수정한다.
