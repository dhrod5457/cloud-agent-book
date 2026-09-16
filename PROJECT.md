# 프로젝트 목적

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

Cloud Agent를 실제 개발팀에서 어떻게 활용할지 정리하는 Software Engineering 책을 작성한다.

이 책은 특정 AI 제품 사용 설명서가 아니다.

핵심 질문:

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

최종 독자 판단:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

## 표지 메시지

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

## 핵심 정의

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

## 핵심 원칙

1. 제품 기능과 일반적인 활용 원칙을 구분한다.
2. Runner가 할 수 있는 일은 Runner에게 맡긴다.
3. Cloud Agent에는 작은 Task와 작은 Context를 전달한다.
4. 결과는 Evidence와 Artifact로 검증한다.
5. Git을 Local과 Cloud 사이의 Handoff Boundary로 사용한다.
6. 병렬화의 대상은 Agent가 아니라 독립 Task다.
7. Local Fallback을 정상적인 Routing으로 다룬다.
8. 제품별 변경 가능한 가격, CPU / RAM, Session 제한은 본문 핵심 논리와 분리한다.
9. Java/Spring Boot를 주요 실전 예제로 사용한다.
10. Agent Platform 일반론은 현재 책의 핵심 범위로 확장하지 않는다.

## 현재 진행 상태

```text
Phase 5
1~18장 설계 완료

Phase 6
1~18장 초고 완료

Phase 7
전체 편집 / 중복 압축 완료

Phase 8
최종 교정 / 출판 정합성 검사 완료

Phase 9
출판 원고 조립 완료

Phase 10
교정쇄 생성 / 시각 검수 준비 중
```

## Phase 9 결과

출판 원고에 필요한 구성 요소를 모두 준비했다.

- 최종 제목 / 부제
- Title Page
- Preface
- About This Book
- How to Read
- 출판용 Table of Contents
- Part I ~ VI 전환 페이지
- Chapters 1 ~ 18
- Appendix A
- Glossary
- References
- 32개 파일 Assembly Manifest
- Markdown → PDF Proof → EPUB → Print-ready PDF 포맷 정책

완료 점검:

- `planning/phase9-manuscript-assembly-check.md`

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

## 출판 원고 결합

파일 단위 Source of Truth:

`manuscript/assembly-manifest.md`

총 32개 파일을 다음 구조로 결합한다.

```text
5 Front Matter
+ 6 Part Transition
+ 18 Chapter
+ 3 Back Matter
```

## Phase 10

계획:

`planning/phase10-proof-plan.md`

진행 순서:

```text
assembly-manifest.md
→ manuscript/master.md
→ PDF Proof
→ 페이지 렌더링
→ 시각 검수
→ review/proof-01.md
→ 원본 Markdown 수정
→ PDF 재생성
```

PDF나 EPUB에서 직접 내용을 수정하지 않는다. 모든 교정 결과는 Markdown Source에 역반영한다.

## 주요 실전 예제

Java/Spring Boot 기반 `campus-platform`을 사용한다.

```text
Local / Local Agent
→ Requirement / Architecture / Human Steering / Internal Validation / Review

Cloud Runner
→ Build / Test / E2E / Docker / 결정론적 검증

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정
```

Tibero, Oracle, HSM, Internal Jenkins, VPN-only API 등은 Local / Cloud 경계를 설명하는 데 필요한 범위에서만 사용한다.

## 현재 Source of Truth

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase8-publication-consistency-check.md`
6. `planning/phase9-manuscript-assembly-check.md`
7. `planning/phase10-proof-plan.md`
8. `manuscript/assembly-manifest.md`
9. `manuscript/book-structure.md`
10. `manuscript/publication-format.md`
11. `STATUS.md`
