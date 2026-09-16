# Current Phase

Phase 10 - 교정쇄 생성 / 시각 검수 준비 중

Phase 9에서 출판 원고 구조를 확정했다.

현재 다음 항목이 모두 준비되어 있다.

- 최종 제목 / 부제
- Front Matter
- 6개 Part 전환 페이지
- 1~18장 본문
- Appendix
- Glossary
- References
- 출판용 전체 목차
- 32개 파일 Assembly Manifest
- PDF / EPUB / 인쇄 포맷 정책

다음 작업은 **32개 Markdown 파일을 하나의 Master 원고로 결합하고 PDF 교정쇄를 생성하는 것**이다.

# Final Book Title

제목:

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

표지용 한 줄 소개:

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

# Source of Truth

현재 우선순위는 다음과 같다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase8-publication-consistency-check.md`
6. `planning/phase9-manuscript-assembly-plan.md`
7. `planning/phase9-manuscript-assembly-check.md`
8. `planning/phase10-proof-plan.md`
9. `manuscript/assembly-manifest.md`
10. `manuscript/book-structure.md`
11. `manuscript/publication-format.md`
12. 각 `manuscript/*` 출판 원고 파일
13. 각 `chapters/NN/draft.md`
14. `STATUS.md`

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
- 결과는 자연어 완료 보고보다 Evidence와 Artifact로 확인한다.
- 병렬화의 대상은 Agent가 아니라 독립 Task다.
- Task는 Local 또는 Cloud에 영구적으로 속하지 않는다.
- Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.
- Local Fallback은 실패가 아니라 Routing의 일부다.

# Phase Results

## Phase 5

- 1~18장 설계 완료

## Phase 6

- 1~18장 초고 완료

## Phase 7

- 전체 편집 / 중복 압축 완료

## Phase 8

- 최종 교정 / 출판 정합성 검사 완료
- `planning/toc.md`와 장 제목 18 / 18 일치
- 제품 공식 근거 재검증

## Phase 9

- 최종 제목 / 부제 확정
- Front Matter 완료
- 6개 Part 구조와 전환 페이지 완료
- 출판용 전체 목차 완료
- Appendix / Glossary / References 완료
- 최종 조립 순서 확정
- 포맷 정책 확정
- `planning/phase9-manuscript-assembly-check.md` 작성

# Publication Structure

```text
Front Matter
→ Part I / Chapters 1~4
→ Part II / Chapters 5~8
→ Part III / Chapters 9~12
→ Part IV / Chapters 13~14
→ Part V / Chapters 15~16
→ Part VI / Chapters 17~18
→ Appendix A
→ Glossary
→ References
```

실제 파일 단위 순서는 다음 문서를 따른다.

`manuscript/assembly-manifest.md`

총 32개 파일이다.

# Publication Format

기준:

`manuscript/publication-format.md`

```text
1. Markdown Source
2. PDF Proof
3. EPUB
4. Print-ready PDF
```

교정 결과는 PDF 자체가 아니라 Markdown Source에 역반영한다.

# Phase 10

계획 문서:

`planning/phase10-proof-plan.md`

진행 순서:

```text
32개 원고 파일
→ manuscript/master.md 생성
→ 구조 검사
→ PDF Proof 생성
→ PDF 페이지 렌더링
→ 시각 검수
→ review/proof-01.md 기록
→ 원본 Markdown 수정
→ Master / PDF 재생성
```

# Next

1. `manuscript/assembly-manifest.md` 기준으로 `manuscript/master.md` 생성
2. Master 구조 검사
3. PDF Proof 생성
4. 페이지 렌더링과 시각 검수
5. 교정 이슈를 원본 Markdown에 역반영
