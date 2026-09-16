# 프로젝트 목적

**클라우드 코딩 에이전트 실전**

부제:

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

Cloud Agent를 실제 개발팀에서 어떻게 활용할지 정리하는 Software Engineering 책을 작성한다.

이 책은 특정 AI 제품 사용 설명서가 아니다.

핵심 질문은 다음과 같다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

최종 독자 판단은 다음과 같다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

독자는 책을 읽은 뒤 자신의 프로젝트에서 다음을 판단하고 구성할 수 있어야 한다.

- Local / Cloud / Hybrid 중 실행 위치 선택
- Build / Test / E2E를 Cloud Runner로 분리
- 재현 가능한 Failure를 Cloud Agent Task로 전환
- Cloud Agent Context와 Tool Output 축소
- Prepared Environment와 Cache 활용
- Git 기반 Local ↔ Cloud Handoff
- 독립 Task 병렬화와 Fan-in 비용 판단
- 내부망 검증과 Cloud 검증 경계 분리
- Cloud 이점이 사라질 때 Local Fallback

## 표지 메시지

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

세부 제목 기준은 `manuscript/title-and-positioning.md`에서 관리한다.

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

1. 프로젝트 파일을 Source of Truth로 사용한다.
2. 제품 기능과 일반적인 활용 원칙을 구분한다.
3. Java/Spring Boot를 주요 실전 예제로 사용한다.
4. 제품 및 기술의 현재 기능은 공식 자료로 검증한다.
5. 변경 가능한 가격, CPU / RAM, Session 제한은 본문 핵심 논리와 분리한다.
6. 새로운 주제는 `Cloud Agent를 더 잘 사용하는 방법과 직접 관련이 있는가?`를 기준으로 포함 여부를 판단한다.
7. Agent Platform 일반론은 현재 책의 핵심 범위로 확장하지 않는다.
8. Runner가 할 수 있는 일은 Runner에게 맡긴다.
9. Cloud Agent에는 작은 Task와 작은 Context를 전달한다.
10. 결과는 Evidence와 Artifact로 검증한다.

## 책 전체에서 유지할 메시지

> Cloud Agent는 Local Agent를 대체하지 않는다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

> Local Fallback은 실패가 아니라 Routing의 일부다.

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
출판 원고 조립 진행 중
```

Phase 9에서 현재까지 완료한 항목:

- 6개 Part 구조
- 서문
- 책 소개 / 독자 대상
- 읽는 방법
- 최종 제목 / 부제 / 표지용 한 줄 소개
- 6개 Part 전환 문구
- 최종 원고 결합 순서 초안

남은 주요 항목:

- 참고자료 구조
- 용어집 정책 및 필요 시 작성
- 부록 필요 여부
- 전체 목차의 출판용 표현
- PDF / EPUB / 인쇄 포맷 결정

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

각 Part 전환 페이지는 `manuscript/parts/part-01.md` ~ `part-06.md`에서 관리한다.

세부 조립 순서는 `manuscript/book-structure.md`를 따른다.

## 주요 실전 예제

Java/Spring Boot 기반 `campus-platform`을 사용한다.

기본 역할:

```text
Local / Local Agent
→ Requirement / Architecture / Human Steering / Internal Validation / Review

Cloud Runner
→ Build / Test / E2E / Docker / 결정론적 검증

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정
```

예제 기술:

- Java 21
- Spring Boot 3.x
- Gradle
- MyBatis
- PostgreSQL / Testcontainers
- Docker
- Playwright

Tibero, Oracle, HSM, Internal Jenkins, VPN-only API 등은 Local / Cloud 경계를 설명하는 데 필요한 범위에서만 사용한다.

## 현재 Source of Truth

우선순위가 높은 문서:

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase8-publication-consistency-check.md`
6. `planning/phase9-manuscript-assembly-plan.md`
7. `manuscript/title-and-positioning.md`
8. `manuscript/book-structure.md`
9. `STATUS.md`
