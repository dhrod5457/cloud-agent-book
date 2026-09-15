# Cloud Agent Book

Cloud Agent를 실제 개발팀에서 어떻게 활용할지 다루는 책 프로젝트입니다.

현재 책의 중심 질문은 다음과 같습니다.

> 클라우드 에이전트를 왜 사용하고, 로컬 에이전트와 어떻게 조합하며, 어떤 작업을 맡기고, 토큰과 클라우드 컴퓨팅 자원을 어떻게 효율적으로 활용할 것인가?

기존 가제 `AI Agent Ready Software Engineering`은 초기 방향에서 사용한 이름이며, 최종 제목은 Cloud Agent 중심 목차가 안정된 뒤 다시 결정합니다.

## 핵심 관점

- Cloud Agent는 Local Agent를 대체하지 않습니다.
- Cloud Agent의 핵심 가치는 더 많은 Token보다 독립 실행환경과 병렬성에 있습니다.
- CPU/RAM과 LLM Token은 서로 다른 자원으로 봅니다.
- Build/Test/E2E 같은 실행 작업은 Cloud Compute를 적극 활용합니다.
- Cloud Agent에는 작은 Task와 작은 Context를 전달합니다.
- 여러 Cloud Agent를 사용할 때 Repository 전체 분석을 반복하지 않도록 합니다.
- Local에서는 설계와 통합을, Cloud에서는 독립 작업과 병렬 검증을 맡기는 Hybrid Workflow를 중심으로 설명합니다.

## 실전 예제

Java/Spring Boot 기반 `campus-platform`을 사용합니다.

예:

```text
Local
→ 기능 설계 / Task 분리

Cloud #1
→ Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Docker Build

Cloud #4
→ Web E2E

Local
→ 내부망 검증 / 통합 / 최종 Review
```

## 집필 원칙

- 책 전체를 한 번에 작성하지 않습니다.
- Phase 단위로 설계, 검토, 작성합니다.
- 프로젝트 파일을 Source of Truth로 사용합니다.
- 특정 AI 제품 사용 설명서로 작성하지 않습니다.
- Claude Code, Codex, GitHub Copilot Coding Agent, Cursor 등의 사례는 설계 패턴을 설명하기 위한 근거로만 사용합니다.
- 변경 가능성이 높은 제품 사양, 가격, Session 제한은 본문의 핵심 논리와 분리합니다.
- Java/Spring Boot를 주요 실전 예제로 사용합니다.
- 새로운 주제는 `Cloud Agent를 더 잘 사용하는 방법과 직접 관련이 있는가?`를 기준으로 본문 포함 여부를 판단합니다.

## 현재 상태

Phase 5 장별 설계 단계입니다.

기존 Agent Platform 중심 확장을 축소하고 Cloud Agent 중심으로 범위와 목차를 다시 정렬했습니다.

현재 최신 기준:

- `planning/concept.md`
- `planning/scope.md`
- `planning/toc.md`
- `planning/future-topics.md`
- `STATUS.md`

본문 초고는 아직 시작하지 않습니다.
