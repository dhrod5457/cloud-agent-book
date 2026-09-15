# Phase 5 전체 정합성 점검

## 결론

새 Cloud Agent 중심 목차의 1~18장 설계가 모두 존재하며, 현재 `planning/toc.md`의 순서와 역할에 맞게 연결되어 있다.

Phase 6 본문 집필을 시작해도 되는 상태다.

---

# 1. 전체 흐름

```text
1~3장
Cloud Agent 정의 / Local vs Cloud / Compute vs Token
        ↓
4~6장
왜 Cloud인가 / 어디로 보낼 것인가 / 어떤 작업이 적합한가
        ↓
7~8장
작은 Input / 작은 Output
        ↓
9~10장
Prepared Environment / Runner-first
        ↓
11~12장
Git 격리 / 병렬 Worker와 비용
        ↓
13~14장
Local↔Cloud Handoff / Event-driven Task
        ↓
15~16장
campus-platform 통합 예제 / End-to-End 기능 개발
        ↓
17장
Cloud를 쓰지 않을 조건 / Local Fallback
        ↓
18장
Harness / Orchestration 미래 전망
```

중간에 Agent Platform 일반론으로 확장되지 않고, 마지막까지 Cloud Agent 활용 질문으로 돌아온다.

---

# 2. 장별 역할 점검

## 1장 - Coding Agent에서 Cloud Worker로

역할:

- Cloud Agent 정의
- Remote Development Worker 관점
- LLM + Repository + Execution Environment + Compute + Tools

중복 방지:

- Local/Cloud 선택은 2장
- Compute/Token 상세는 3장
- 환경 최적화는 9장

판정: PASS

## 2장 - Local Agent와 Cloud Agent

역할:

- 실행 위치 차이
- Local/Cloud 기본 역할
- Hybrid 전제

중복 방지:

- Routing 세부 기준은 5장
- Handoff 절차는 13장

판정: PASS

## 3장 - Cloud Session, Container, Compute와 Token

역할:

- Compute와 Token 분리
- Brain/Hands 간단 모델
- Execution Resource와 Reasoning Resource 구분

중복 방지:

- Runner 구현은 10장
- Agent Platform Brain/Hands 상세는 후속 주제

판정: PASS

## 4장 - 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

역할:

- Cloud의 실질적 가치
- 비동기 위임
- Agent Execution Time vs Developer Blocking Time

중복 방지:

- 병렬 작업 격리는 11장
- 병렬화 비용은 12장

판정: PASS

## 5장 - Task Routing

역할:

- Local / Cloud / Hybrid / Runner-first 선택
- Task 크기와 Hard Constraint

중복 방지:

- 개별 작업 유형은 6장
- Cloud를 쓰지 않을 조건의 종합은 17장

판정: PASS

## 6장 - Cloud에 보내기 좋은 개발 작업

역할:

- Build/Test/E2E/Docker/Refactoring/Bug Fix 등의 실제 분류
- Cloud Runner와 Cloud Agent 구분

중복 방지:

- Runner-first 구조 자체는 10장

판정: PASS

## 7장 - Cloud Agent Task Contract

역할:

- 작은 Task
- 작은 Context
- Goal / Scope / Relevant Files / Forbidden Changes / Validation / Evidence

중복 방지:

- 결과 반환 상세는 8장

판정: PASS

## 8장 - Tool Output과 Evidence

역할:

- Result Filter / Result Gateway
- Artifact First
- Failure Fingerprint
- Evidence-based Result
- Demos over Diffs

중복 방지:

- 입력 Context는 7장
- Runner-first 실행은 10장

판정: PASS

## 9장 - Prepared Cloud Environment

역할:

- Cold Start
- Cloud Environment as Code
- Cache / Snapshot / Warm Worker
- Reusable Cache vs Fresh State

중복 방지:

- 실제 Runner 실행 구조는 10장

판정: PASS

## 10장 - Cloud Agent를 Test Runner처럼 사용하기

역할:

- Deterministic First
- Runner-first / Agent-on-failure
- CPU/RAM 실행과 LLM 판단 분리

중복 방지:

- Result Gateway 상세는 8장
- Event trigger는 14장

판정: PASS

## 11장 - Git, Branch, Worktree, Container

역할:

- Git Handoff Boundary
- Base SHA
- Branch per Task
- Workspace 격리

중복 방지:

- 병렬 Worker 수와 Context 비용은 12장

판정: PASS

## 12장 - 병렬 Worker와 중복 Context 비용

역할:

- Fan-out / Fan-in
- Context Duplication
- Merge / Review Cost
- Best-of-N Advanced Pattern

중복 방지:

- Git 격리 자체는 11장
- Best-of-N은 고급 사례 수준으로 제한

판정: PASS

## 13장 - Local → Cloud → Local Handoff

역할:

- 하나의 Task가 실행 위치를 이동하는 Workflow
- Multi-Repository 범위 제한
- 내부망 최종 검증

중복 방지:

- Local/Cloud 기본 차이는 2장
- Task Routing은 5장

판정: PASS

## 14장 - Task Queue와 Event-driven Cloud Agent

역할:

- Issue / CI / Review / Schedule 기반 Task 생성
- 이벤트가 없으면 Agent도 실행하지 않음

중복 방지:

- Runner-first 원리는 10장에서 정의하고 여기서는 Event source에 적용

판정: PASS

## 15장 - campus-platform Cloud Agent Workflow

역할:

- 1~14장 원칙 통합
- Java/Spring Boot 운영 모델

새 개념 추가를 최소화하고 통합 장으로 유지한다.

판정: PASS

## 16장 - 하나의 기능을 끝까지 개발하기

역할:

- 시간 순서 End-to-End 사례
- 요구사항 → Task Split → Cloud 실행 → Evidence → 내부망 검증 → Merge

15장과의 차이:

- 15장 = 시스템/운영 구조
- 16장 = 하나의 기능 실행 과정

판정: PASS

## 17장 - Cloud가 항상 정답은 아니다

역할:

- Local Fallback
- Cloud 부적합 조건
- 중단 기준

5장과의 차이:

- 5장 = 시작 전 Routing
- 17장 = 부적합 조건 종합 + 실행 중 Fallback

판정: PASS

## 18장 - 다음 단계: Harness와 Orchestration

역할:

- 미래 발전 방향
- Harness / Routing Automation / Brain-Hands / Agent-native Observability / Best-of-N

범위 제한:

- Agent Platform 상세 설계로 확장하지 않음
- 상세 내용은 `planning/future-topics.md`로 넘김

판정: PASS

---

# 3. 핵심 메시지 일관성

다음 문장은 전체 설계에서 일관되게 유지된다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> 작은 Task와 작은 Context를 전달한다.

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

판정: PASS

---

# 4. 용어 정리

Phase 6에서 우선 사용할 용어:

- Cloud Agent
- Cloud Worker
- Cloud Runner
- Local Agent
- Remote Development Worker
- Task Routing
- Task Contract
- Result Gateway
- Evidence
- Prepared Cloud Environment
- Cold Start
- Handoff Boundary
- Developer Blocking Time
- Agent Execution Time

제한적으로 사용할 용어:

- Brain / Hands: Compute와 Token 구분 및 미래 전망에서만 사용
- Best-of-N: 12장 고급 병렬화와 18장 미래 전망 수준
- Agent-native Observability: 18장 미래 전망 수준
- Orchestration: 18장 이후 발전 방향 수준

본문 핵심에서 제외:

- Agent Memory Architecture
- Agent Security Platform
- Agent OS
- Agent Chaos Engineering
- Garbage Collector Agent
- Shadow Agent
- Canary Agent
- Agent Governance Platform
- General Multi-Agent Theory

---

# 5. 기존 자료 처리

다음 문서는 삭제하지 않는다.

- `chapters/02/execution-platform.md`
- `planning/agent-native-development-environment.md`
- `planning/toc-amendment-agent-native.md`
- `examples/campus-platform/agent-native-development-environment.md`
- `research/anthropic/agent-native-development-environment.md`

단, 현행 장 번호와 범위 판단에는 사용하지 않는다.

우선순위:

```text
planning/concept.md
→ planning/scope.md
→ planning/toc.md
→ chapters/NN/plan.md
→ planning/future-topics.md
→ 과거 확장 설계 자료
```

`chapters/02/README.md`에 `execution-platform.md` 내부의 과거 장 번호는 현행 참조가 아니라는 점을 명시했다.

---

# 6. Phase 6 집필 규칙

본문 집필 시 다음을 유지한다.

- 1장부터 순차 작성
- 장 설계의 역할을 넘지 않음
- 뒤 장 내용을 앞 장에서 과도하게 설명하지 않음
- 제품 기능은 원칙을 설명하는 사례로만 사용
- 변경 가능한 CPU/RAM/가격/한도는 research 자료와 분리
- Java/Spring Boot `campus-platform` 예제를 반복 사용
- 좋은 사례/나쁜 사례 비교
- 실행 가능한 명령과 Evidence를 가능한 범위에서 제시
- 수치가 필요한 경우 실제 근거 또는 명시적인 예시로 구분
- Agent Platform 일반론으로 확장하지 않음

---

# Phase 5 판정

## 완료 조건

- [x] Cloud Agent 중심 Concept 확정
- [x] Scope 재정의
- [x] 11 Part / 18 Chapter 목차 확정
- [x] 1~18장 설계 완료
- [x] Agent Platform 확장 주제 Future Topics로 이동
- [x] 장별 역할 중복 점검
- [x] 핵심 용어/메시지 정합성 점검
- [x] Phase 6 집필 경계 정의

## 최종 판정

`Phase 5 - 완료`

다음 단계:

`Phase 6 - 1장부터 본문 초고 작성`
