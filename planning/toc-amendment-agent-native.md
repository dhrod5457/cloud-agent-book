# TOC Amendment - Agent-Native Development Environment

## 상태

이 문서는 **이전 설계 기록**이다.

Phase 5 범위 재정렬에서 책의 중심을 Cloud Agent 활용으로 다시 좁혔기 때문에, 이 문서의 `Agent-Native Development Environment` 독립 장 추가 결정은 더 이상 현재 목차에 적용하지 않는다.

현재 최신 목차는 다음 문서를 따른다.

- `planning/toc.md`

현재 범위는 다음 문서를 따른다.

- `planning/concept.md`
- `planning/scope.md`

이 문서에서 제안했던 고급 Agent Platform 주제는 삭제하지 않고 다음 문서로 이동했다.

- `planning/future-topics.md`

## 이전 결정

과거에는 다음 구조를 검토했다.

```text
20장 실패, 재시도, 충돌, 통합
→ 21장 Agent-Native Development Environment
→ 22장 VPN과 내부망이 있는 Hybrid Agent 시스템
→ 23장 Full Agentic Development로의 진화와 Governance
```

이 구조는 현재 사용하지 않는다.

## 현재 처리 방식

다음 주제는 현재 Cloud Agent 중심 책에서 독립 장으로 만들지 않는다.

- Brain / Hands / Session lifecycle
- One Brain, Multiple Hands
- Agent Hibernate
- Garbage Collector Agent
- Agent-native Observability Platform
- Compute-aware Orchestration
- Shadow Agent
- Canary Agent
- Agent Replay
- Agent Platform CI
- Agent-Native Repository 전체 아키텍처

현재 책에서는 Cloud Agent 활용과 직접 연결되는 일부 개념만 제한적으로 사용한다.

예:

- Brain / Hands: Token과 Compute의 차이를 설명하는 간단한 모델
- Best-of-N: Cloud 병렬화의 고급 사례
- Harness Engineering: Cloud Agent가 실행/검증하기 좋은 프로젝트 준비 관점
- Agent-native Observability: 마지막 미래 전망에서 간단히 소개

상세 설계는 후속 주제 후보로 보존한다.
