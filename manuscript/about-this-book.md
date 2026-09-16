# 이 책이 다루는 문제

Cloud Coding Agent를 실제 프로젝트에 넣으면 모델 성능만으로 설명되지 않는 문제가 생긴다.

```text
어떤 작업을 Cloud로 보낼 것인가?
어떤 작업은 Local에 남길 것인가?
Build와 Test까지 Agent가 판단하게 할 것인가?
여러 Cloud Worker를 어떻게 병렬화할 것인가?
Repository와 Context를 얼마나 넘길 것인가?
Cloud에서 만든 결과를 무엇으로 검증할 것인가?
```

이 책은 이 질문을 Software Engineering Workflow 관점에서 다룬다.

핵심 모델은 다음과 같다.

```text
Local / Local Agent
→ 요구사항 / Architecture / Human Steering / Internal Validation / Review

Cloud Runner
→ Build / Test / E2E / Docker / 결정론적 검증

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정
```

Cloud Agent를 모든 개발 작업의 기본 실행 주체로 두지 않는다.

Task의 성격에 따라 Local, Cloud Runner, Cloud Agent, Hybrid 중 적절한 실행 위치를 고른다.

## 이 책에서 다루는 것

- Cloud Agent를 Remote Development Worker로 이해하는 방법
- Local Agent와 Cloud Agent의 실행 위치 차이
- CPU / RAM / Disk와 LLM Token을 분리해서 보는 방법
- Cloud에 보내기 좋은 Task를 판단하는 기준
- Build / Test / E2E를 Cloud Runner로 분리하는 방법
- 작은 Task Contract와 Progressive Context
- Tool Output을 줄이고 Evidence를 남기는 방법
- Prepared Environment, Cache, Snapshot
- Git / Branch / Worktree / Container를 이용한 작업 격리
- 독립 Task의 병렬 실행과 Fan-in 비용
- Local → Cloud → Local Handoff
- CI / Review / Schedule 기반 Event-driven Task
- Java/Spring Boot 프로젝트에 적용하는 운영 모델
- Cloud Task 중단과 Local Fallback 기준
- 반복 Workflow를 Harness와 Orchestration으로 확장하는 순서

## 이 책에서 다루지 않는 것

이 책은 다음 주제를 Cloud Agent 활용에 직접 필요한 범위 이상으로 확장하지 않는다.

```text
범용 Agent Platform 설계
Agent OS
Agent Memory Architecture
Agent Governance Platform
일반적인 Multi-Agent 조직론
특정 제품의 전체 사용법
제품별 가격 비교표
제품별 고정 CPU / RAM 사양표
```

제품 기능, 가격, 제한은 바뀔 수 있다.

따라서 제품 사례는 원칙을 설명하는 근거로 사용하고, 변경 가능한 세부사항은 Research와 기준일을 따로 관리한다.

# 이 책의 독자

이 책은 다음 독자를 대상으로 한다.

## Coding Agent를 실제 개발에 사용하고 있는 개발자

Local Agent를 사용해 코드 작성과 수정은 하고 있지만, 장시간 Test, E2E, Build, 반복 수정 작업을 어떻게 Cloud로 분리할지 고민하는 경우에 적합하다.

## 팀 단위로 Cloud Agent 도입을 검토하는 개발자와 Tech Lead

다음과 같은 질문이 있다면 직접적인 대상이다.

```text
어떤 업무부터 Cloud에 맡길 것인가?
Agent가 만든 코드를 어떻게 검증할 것인가?
팀의 Review 병목은 어떻게 볼 것인가?
내부망이 있는 프로젝트에서 Cloud를 어디까지 사용할 것인가?
```

## CI/CD와 개발환경을 함께 다루는 Backend / Platform 개발자

이 책은 Agent Prompt만 다루지 않는다.

Git, Build, Test, Docker, Browser, DB, Artifact, CI, 실행환경이 Cloud Agent와 어떻게 연결되는지를 함께 다룬다.

## Java/Spring Boot 프로젝트를 운영하는 개발자

실전 예제는 `campus-platform`이라는 Java/Spring Boot 기반 프로젝트를 사용한다.

다만 핵심 원칙은 Java에만 한정되지 않는다. Task Routing, Runner-first, Evidence, Isolation, Handoff는 다른 언어와 개발환경에도 적용할 수 있다.

# 이 책을 읽고 나면

독자는 적어도 다음 질문에 자신의 프로젝트 기준으로 답할 수 있어야 한다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
```

그리고 Cloud로 보내기로 했다면 다음도 정할 수 있어야 한다.

```text
Base SHA는 무엇인가?
Scope는 어디까지인가?
어떤 Environment를 사용할 것인가?
Validation은 무엇인가?
어떤 Evidence를 반환받을 것인가?
어떤 조건에서 중단할 것인가?
```

이 책의 목적은 Cloud Agent 사용량을 늘리는 것이 아니다.

개발 Workflow 안에서 Cloud를 사용할 지점을 판단하고, 검증 가능한 방식으로 작업을 넘기고 돌려받는 기준을 만드는 것이다.
