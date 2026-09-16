# 클라우드 코딩 에이전트 실전

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

Cloud Agent를 실제 개발 Workflow에 어떻게 배치하고 활용할지 다루는 Software Engineering 책 프로젝트입니다.

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

책의 중심 질문:

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

독자가 마지막에 답할 수 있어야 하는 질문:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

## 핵심 관점

- Cloud Agent는 Local Agent를 대체하지 않습니다.
- Cloud Agent의 핵심 가치는 더 많은 Token보다 독립 실행환경과 병렬성에 있습니다.
- CPU / RAM / Disk와 LLM Token은 서로 다른 자원으로 봅니다.
- Runner가 할 수 있는 Build / Test / E2E / Docker는 Runner에게 먼저 맡깁니다.
- Cloud Agent는 판단과 제한된 코드 수정이 필요한 구간에 사용합니다.
- 작은 Task와 작은 Context를 전달합니다.
- 결과는 Evidence와 Artifact로 확인합니다.
- 병렬화의 대상은 Agent 수가 아니라 독립 Task입니다.
- Cloud 이점이 사라지면 Local 또는 Hybrid로 재Routing합니다.

## 현재 상태

```text
Phase 5  1~18장 설계                완료
Phase 6  1~18장 초고                완료
Phase 7  전체 편집 / 중복 압축       완료
Phase 8  최종 교정 / 출판 정합성      완료
Phase 9  출판 원고 조립              완료
Phase 10 교정쇄 생성 / 시각 검수      준비 중
```

Phase 9에서 다음 출판 구성 요소를 완료했습니다.

- 최종 제목 / 부제
- Front Matter
- 6개 Part 전환 페이지
- 출판용 전체 목차
- Appendix A
- Glossary
- References
- 32개 파일 Assembly Manifest
- PDF / EPUB / 인쇄 포맷 정책

Phase 10의 첫 작업은 `manuscript/master.md` 생성입니다.

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

## 출판 원고 주요 파일

```text
manuscript/title-page.md
manuscript/preface.md
manuscript/about-this-book.md
manuscript/how-to-read.md
manuscript/table-of-contents.md
manuscript/parts/part-01.md ~ part-06.md
chapters/01/draft.md ~ chapters/18/draft.md
manuscript/appendix/task-contract-evidence-template.md
manuscript/glossary.md
manuscript/references.md
```

실제 결합 순서:

- `manuscript/assembly-manifest.md`

출판 포맷 정책:

- `manuscript/publication-format.md`

Phase 10 계획:

- `planning/phase10-proof-plan.md`

## 실전 예제

Java/Spring Boot 기반 `campus-platform`을 사용합니다.

```text
Local / Local Agent
→ 요구사항 / Architecture / Task Split

Cloud Runner
→ Build / Unit / Integration / E2E / Docker

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정

Local
→ Internal Validation / Review / Merge
```

특정 제품 사용 설명서가 아니라 이 실행 구조를 설계하는 방법을 설명합니다.

## Source of Truth

현재 우선순위가 높은 문서:

- `planning/concept.md`
- `planning/scope.md`
- `planning/toc.md`
- `planning/phase8-publication-consistency-check.md`
- `planning/phase9-manuscript-assembly-check.md`
- `planning/phase10-proof-plan.md`
- `manuscript/assembly-manifest.md`
- `manuscript/book-structure.md`
- `STATUS.md`
