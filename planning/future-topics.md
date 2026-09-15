# 후속 주제 후보

이 문서는 Cloud Agent 중심 본문 범위에서 제외하거나 축소한 주제를 보존한다.

삭제 목록이 아니다. 현재 책의 중심 질문과 직접 연결되지 않아 후속 책, 부록, 별도 연구 주제로 이동한 항목이다.

## 이동 기준

현재 책의 중심 질문:

> 클라우드 에이전트를 왜 사용하고, 로컬 에이전트와 어떻게 조합하며, 어떤 작업을 맡기고, 토큰과 클라우드 컴퓨팅 자원을 어떻게 효율적으로 활용할 것인가?

이 질문과 직접 연결되지 않는 Agent Platform 일반론은 본문 핵심에서 제외한다.

---

# 1. Agent-Native Development Environment

기존 설계 문서:

- `planning/agent-native-development-environment.md`
- `planning/toc-amendment-agent-native.md`
- `examples/campus-platform/agent-native-development-environment.md`
- `research/anthropic/agent-native-development-environment.md`

후속 주제 후보:

- Model + Context + Harness + Tools + Compute + Validation + Observability + Orchestration
- Brain / Hands / Session lifecycle
- One Brain, Multiple Hands
- Agent Hibernate
- Compute-aware Orchestration
- Warm Pool
- Agent-native Repository

현재 책에서는 다음 정도만 남긴다.

- Brain = LLM 판단
- Hands = Cloud Container 실행
- Cloud Agent 성능은 모델뿐 아니라 실행환경 영향도 받음

이를 별도 장으로 확장하지 않는다.

---

# 2. Agent Platform 운영

후속 주제:

- Agent Platform control plane
- Capability Broker
- Sandbox Broker
- Tool Registry
- Resource Scheduler
- Agent lifecycle service
- Multi-agent 조직 구조
- 조직 단위 quota / accounting

현재 책에서는 Cloud Agent 실행환경과 Task routing에 필요한 수준만 다룬다.

---

# 3. Agent Memory Architecture

후속 주제:

- 장기 Agent Memory
- Episodic / Semantic Memory
- Memory lifecycle
- cross-session memory
- vector/graph memory
- organization-wide agent knowledge

현재 책에서는 Cloud Agent Context 최소화와 Repository 문서를 이용한 필요한 정보 조회만 다룬다.

---

# 4. Agent-native Observability Platform

후속 주제:

- Agent용 observability query API
- Trace/Metric/Log 자동 탐색
- Agent execution telemetry
- token/compute/human cost correlation
- agent behavior tracing
- autonomous performance debugging

현재 책에서는 Cloud Task의 실행시간, Agent invocation, Token 사용량, 실패율 정도만 다룬다.

---

# 5. Agent Security Platform

후속 주제:

- Capability Broker
- Tool permission broker
- Agent Security Telemetry
- Agent-native Security
- dynamic least privilege
- policy engine
- autonomous secret boundary enforcement

현재 책에서는 Cloud Repository/Secret/Network 접근을 판단하는 실전 경계만 다룬다.

---

# 6. Agent Chaos Engineering

후속 주제:

- Agent runtime failure injection
- Tool failure simulation
- Context corruption tests
- model/provider failure handling
- sandbox loss/recovery tests

현재 책에서는 Cloud Task 실패와 retry 제한만 다룬다.

---

# 7. Garbage Collector Agent

후속 주제:

- duplication cleanup
- architecture drift cleanup
- stale docs
- dead code
- unused dependency
- abandoned feature flag
- temporary workaround cleanup

현재 책에서는 핵심 범위에서 제외한다.

---

# 8. Shadow Agent / Canary Agent

## Shadow Agent

실제 변경 권한 없이 Main Agent의 판단을 별도로 검토하는 구조.

## Canary Agent

새 Model/Harness를 일부 Task에만 적용하고 성공률, Token, Retry, Human Intervention을 비교하는 구조.

현재 책에서는 미래 전망에서 짧게 언급할 수 있다.

---

# 9. Agent Replay / Agent Regression Test

후속 주제:

- Task input 저장
- Prompt/Context 저장
- Model/Harness version 저장
- Tool calls/results 저장
- Environment version
- Git SHA
- Test result
- 동일 Task 재실행
- Model/Harness regression 비교

현재 책에서는 본문 핵심에서 제외한다.

---

# 10. Speculative Coding / Best-of-N의 확장

현재 책에서는 Best-of-N을 Cloud 병렬화의 고급 사례로만 짧게 사용한다.

후속 주제에서는 다음을 상세히 다룰 수 있다.

- Candidate count policy
- Model diversity
- deterministic ranking
- patch complexity scoring
- security scoring
- cost-aware selection
- multi-stage tournament

---

# 11. Time-travel Debugging

후속 주제:

- execution recording
- browser trace
- network replay
- DOM snapshot
- application state snapshot
- failure-time environment replay

현재 책에서는 E2E Artifact 보존 사례 수준만 사용한다.

---

# 12. PR Babysitter Agent

후속 주제:

- CI monitoring
- review comment response
- conflict detection
- flaky test retry
- merge queue monitoring
- long-lived PR lifecycle automation

현재 책에서는 CI failure / Review Comment → Cloud Agent 호출 패턴 정도만 다룬다.

---

# 13. Agent Ready Profile

기존 3장 설계에서 정의했던 `Agent Ready 프로젝트의 기준`은 Cloud Agent 중심 목차 재정렬로 독립 장에서 제외했다.

기존 평가 기준:

1. Reproducibility
2. Discoverability
3. Executability
4. Testability
5. Verifiability
6. Isolation
7. Parallelizability
8. Observability
9. Security Boundary

평가 방식:

- PASS / PARTIAL / FAIL
- 설명이 아니라 실행 가능한 Evidence 요구
- 단일 평균 Score보다 기준별 Profile 사용

이 개념은 여전히 유효하지만 현재 책에서는 독립 방법론으로 확장하지 않는다.

Cloud Agent 활용에 직접 필요한 부분만 각 장에 분산한다.

- Reproducibility → 9장 Prepared Cloud Environment
- Executability/Testability → 6장, 10장
- Isolation/Parallelizability → 11장, 12장
- Discoverability → 7장 Context 최소화
- Verifiability → 8장 Evidence-based Result
- Observability → Cloud Task 실행시간/결과 추적 수준
- Security Boundary → 2장/17장의 Cloud 사용 제한 조건

기존 상세 설계는 Git history의 이전 `chapters/03/plan.md`와 `examples/campus-platform/agent-ready-baseline.md`에서 확인할 수 있다.

후속 책 또는 부록에서 `Agent Ready Assessment`로 다시 확장할 수 있다.

---

# 14. 향후 별도 책 후보

가칭:

`Agent-Native Software Engineering`

주제:

> Agent가 개발환경을 사용하는 단계를 넘어 개발환경과 플랫폼 자체를 Agent 중심으로 설계하는 방법

현재 책의 Cloud Agent 활용 경험을 전제로 다음을 확장할 수 있다.

- Agent Platform Architecture
- Agent-native Repository
- Agent-native Observability
- Agent Security
- Agent Memory
- Agent Evaluation
- Agent Operations
- Compute-aware Orchestration
- Agent CI / Regression
- Agent Ready Assessment
