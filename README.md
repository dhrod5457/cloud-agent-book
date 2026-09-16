# 클라우드 코딩 에이전트 실전

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

Cloud Agent를 실제 개발 Workflow에 어떻게 배치하고 활용할지 다루는 Software Engineering 책 프로젝트입니다.

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

책의 중심 질문은 다음과 같습니다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

독자가 마지막에 답할 수 있어야 하는 질문은 더 구체적입니다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

## 핵심 관점

- Cloud Agent는 Local Agent를 대체하지 않습니다.
- Cloud Agent의 핵심 가치는 더 많은 Token보다 독립 실행환경과 병렬성에 있습니다.
- CPU / RAM / Disk와 LLM Token은 서로 다른 자원으로 봅니다.
- Runner가 할 수 있는 Build / Test / E2E / Docker는 Runner에게 먼저 맡깁니다.
- Cloud Agent는 판단과 제한된 코드 수정이 필요한 구간에 사용합니다.
- 작은 Task와 작은 Context를 전달합니다.
- 결과는 자연어 완료 보고보다 Evidence와 Artifact로 확인합니다.
- 병렬화의 대상은 Agent 수가 아니라 독립적으로 실행하고 검증할 수 있는 Task입니다.
- Cloud 이점이 사라지면 Local 또는 Hybrid로 재Routing합니다.

## 현재 원고 상태

Phase 8까지 1~18장 설계, 초고, 전체 편집, 최종 교정, 출판 정합성 검사를 완료했습니다.

현재는 **Phase 9 - 출판 원고 조립**을 진행하고 있습니다.

Phase 9에서 현재까지 완료한 항목:

- 출판용 6개 Part 구조
- 서문
- 책 소개 / 독자 대상
- 읽는 방법
- 최종 제목 / 부제 / 표지용 한 줄 소개
- 6개 Part 전환 페이지
- 출판 원고 결합 순서 초안

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

출판 원고 구조는 `manuscript/book-structure.md`를 기준으로 관리합니다.

## 출판 원고 파일

- `manuscript/title-and-positioning.md`
- `manuscript/preface.md`
- `manuscript/about-this-book.md`
- `manuscript/how-to-read.md`
- `manuscript/book-structure.md`
- `manuscript/parts/part-01.md` ~ `part-06.md`

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

## 집필 원칙

- 프로젝트 파일을 Source of Truth로 사용합니다.
- 특정 AI 제품을 Cloud Agent의 정의로 사용하지 않습니다.
- 제품별 기능과 일반적인 활용 원칙을 구분합니다.
- 변경 가능성이 높은 가격, CPU / RAM, Session 제한은 본문 핵심 논리와 분리합니다.
- 제품 및 기술의 현재 기능은 공식 자료를 기준으로 검증합니다.
- Java/Spring Boot를 주요 실전 예제로 사용합니다.
- 새로운 주제는 `Cloud Agent를 더 잘 사용하는 방법과 직접 관련이 있는가?`를 기준으로 본문 포함 여부를 판단합니다.
- Agent Platform 일반론은 현재 책의 핵심 범위로 확장하지 않습니다.

## Source of Truth

현재 우선순위가 높은 문서:

- `planning/concept.md`
- `planning/scope.md`
- `planning/toc.md`
- `planning/cloud-agent-remote-worker-model.md`
- `planning/phase8-publication-consistency-check.md`
- `planning/phase9-manuscript-assembly-plan.md`
- `manuscript/book-structure.md`
- `manuscript/title-and-positioning.md`
- `STATUS.md`
