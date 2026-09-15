# Current Phase

Phase 5 - 장별 설계 진행 중 / 목차 보강

본문은 아직 작성하지 않는다.

# Completed

- Phase 1 방향 정의
- Phase 2 전체 목차 설계
- Phase 3 목차 검증
- Phase 4 `campus-platform` 예제 프로젝트 설계
- Phase 5 1장 설계
- Phase 5 2장 설계
- Phase 5 2장 `Agent-friendly Execution Platform` 확장 설계
- Phase 5 3장 `Agent Ready 프로젝트의 기준` 설계
- `Agent-Native Development Environment` 독립 장 추가 결정
- Anthropic infrastructure noise / Managed Agents / parallel C compiler 공식 사례 조사
- `campus-platform` Agent-Native Development Environment 적용 설계

# Phase 4 / Example Artifacts

- `examples/campus-platform/README.md`
- `examples/campus-platform/architecture.md`
- `examples/campus-platform/testing.md`
- `examples/campus-platform/evolution.md`
- `examples/campus-platform/cloud-test-runner.md`
- `examples/campus-platform/execution-platform.md`
- `examples/campus-platform/agent-ready-baseline.md`
- `examples/campus-platform/agent-native-development-environment.md`

# Phase 5 / Planning Artifacts

- `chapters/01/plan.md`
- `chapters/02/plan.md`
- `chapters/02/execution-platform.md`
- `chapters/03/plan.md`
- `planning/agent-native-development-environment.md`
- `planning/toc-amendment-agent-native.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/agent-native-development-environment.md`
- `research/github/continuous-ai-runner-first.md`

# Book Direction

- 상위 개념은 `Agent Ready Software Engineering`이다.
- Java/Spring Boot는 주요 실전 예제지만 원칙은 언어와 제품에 종속되지 않는다.
- 특정 AI 제품 사용법보다 Repository, 실행환경, 검증, 관찰, orchestration 설계를 중심으로 다룬다.
- 프로젝트 파일을 Source of Truth로 사용한다.
- 변경 가능성이 높은 제품 CPU/RAM/가격/세션 제한은 일반 원칙과 분리한다.

# Cloud / Execution 핵심 원칙

2장에서 다음 원칙을 정의했다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

추가 운영 원칙:

- 환경은 미리 준비한다.
- 가능한 판정은 코드로 처리한다.
- Context는 필요할 때 필요한 만큼만 제공한다.
- 실패 결과 전체를 전달하지 않고 조회 가능한 형태로 저장한다.
- Agent의 자율성에는 시간, retry, token, 비용 한도를 둔다.
- Agent가 반복해서 실패하면 프롬프트보다 실행환경과 도구를 먼저 개선한다.

확장된 관점:

> 클라우드 에이전트의 최종 형태는 Agent를 계속 실행하는 시스템이 아니라, Agent가 필요할 때 붙을 수 있도록 잘 준비된 실행 플랫폼이다.

# Agent-friendly Execution Platform

핵심 구성:

- Prebuilt Environment
- Reusable Cache / Disposable Runtime State
- Cloud Runner / Agent Worker 분리
- Failure-driven Agent
- Result Gateway / Artifact First
- Progressive Context
- Task Context Package
- Isolated Worktree / Container
- Failure Container Retention
- Budgeted Autonomy
- Failure Fingerprint
- Event-driven Agent
- Continuous Small Task
- Fan-out / Fan-in
- Harness Engineering

핵심 원칙:

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

> Agent의 자율성은 무제한 실행 권한이 아니라 예산 안에서 스스로 해결할 수 있는 권한이다.

> Agent가 반복해서 실패하면 프롬프트보다 Harness를 먼저 개선한다.

# Agent Ready 평가

3장에서는 Agent Ready를 다음 9개 기준의 Profile로 평가한다.

1. Reproducibility
2. Discoverability
3. Executability
4. Testability
5. Verifiability
6. Isolation
7. Parallelizability
8. Observability
9. Security Boundary

각 기준은 `PASS / PARTIAL / FAIL`과 실행 가능한 Evidence로 평가한다.

# New Chapter Decision - Agent-Native Development Environment

Phase 5 설계 보강에서 새 독립 장을 추가하기로 결정했다.

이유:

2장의 실행 위치/Runner 원칙을 넘어 Repository, Validation, Observability, PM Agent, Retry/Governance를 종합해야 하는 주제가 추가되었기 때문이다.

목차 변경안:

```text
20장 실패, 재시도, 충돌, 통합
→ 21장 Agent-Native Development Environment
→ 22장 VPN과 내부망이 있는 Hybrid Agent 시스템
→ 23장 Full Agentic Development로의 진화와 Governance
```

기존 21장은 22장으로, 기존 22장은 23장으로 이동한다.

현재 `planning/toc.md`는 Phase 3 확정본이고, Phase 5에서는 `planning/toc-amendment-agent-native.md`가 후반부 배치에 대한 최신 결정을 보완한다. Phase 5 checkpoint에서 두 문서를 통합한다.

# Agent-Native Development Environment 핵심 모델

> 좋은 Agent 시스템은 좋은 모델 하나로 만들어지는 것이 아니라, Model + Context + Harness + Tools + Compute + Validation + Observability + Orchestration이 함께 만들어낸다.

개념 모델:

```text
Agent Performance
= f(Model, Context, Harness, Tools, CPU, RAM, Runtime,
    Network, Validation, Observability, Orchestration)
```

공식 사례 조사에서는 동일 model/harness/task set에서도 runtime resource configuration에 따라 agentic coding 성공률이 달라질 수 있음을 확인했다.

# 21장 핵심 설계

## Brain / Hands / Session

Brain과 실행환경을 같은 container lifecycle에 고정하지 않는다.

```text
Session
|
+-- Brain
|    - LLM
|    - Harness
|    - Task State
|
+-- Hands
     - Linux Sandbox
     - Browser
     - Android Emulator
     - Database
     - External Tool
```

핵심 원칙:

> Agent 안에 컴퓨터가 있는 것이 아니라, Agent가 필요할 때 컴퓨터를 호출한다.

## One Brain, Multiple Hands

하나의 Agent Brain이 Backend, Browser, Android, Database execution hand를 작업 성격에 따라 호출할 수 있게 한다.

`campus-platform`에서는 Spring Boot Backend, 관리자 Web, Android App, DB Migration을 서로 다른 hand로 검증한다.

## Deterministic Sampling

대형 test suite는 다음 계층으로 설계할 수 있다.

```text
Fast Feedback Test
+
Distributed Deterministic Sampling
+
Final Full Validation
```

동일 retry에서는 같은 sample을 사용해 재현성을 유지하고, 다른 Worker는 다른 sample을 사용해 aggregate coverage를 넓힌다.

Merge 전 full validation은 유지한다.

## Agent-friendly Log

상세 로그 전체를 stdout/context로 보내지 않는다.

- 짧은 summary
- machine-readable error line
- searchable artifact
- 필요 시 Result Gateway lookup

핵심 원칙:

> Agent가 읽기 쉬운 출력 형식도 개발환경의 일부다.

## Agent Hibernate

Brain/session state와 expensive execution hand lifecycle을 분리한다.

```text
RUNNING → IDLE → SNAPSHOT → OFF → WAKE → RUNNING
```

PR CI 대기나 reviewer 대기 동안 execution resource를 끄고 후속 event에서 복원할 수 있게 설계한다.

## PR Babysitter

PR 생성 이후 다음을 관리하는 제한된 role을 둔다.

- CI 확인
- 실패 대응
- review comment 처리
- conflict 감지
- test 재실행
- merge-ready 상태 유지

자동 merge 권한은 Trust Boundary와 별도로 관리한다.

## Speculative Coding / Best-of-N

어려운 Task에서는 복수 후보를 병렬 생성할 수 있지만 최종 선택은 가능한 한 deterministic validation으로 수행한다.

N의 기본값은 1이며 task difficulty와 budget에 따라 제한적으로 늘린다.

## Time-travel Debugging

Browser/UI/실행 실패 시 screenshot, video, network log, DOM snapshot, trace 등 실패 시점 artifact를 저장하고 필요한 시점을 조회할 수 있게 한다.

## Garbage Collector Agent

Agent가 생성한 변경으로 쌓일 수 있는 duplication, architecture drift, stale docs, dead code, unused dependency, temporary workaround를 정기적으로 점검한다.

대규모 자동 refactoring보다 작은 cleanup PR을 반복한다.

## Agent-native Observability

17장에서 정의한 Observability를 Agent query interface로 확장한다.

예:

```text
get_errors(service, since)
get_slow_spans(service, threshold_ms)
get_metric(name)
get_http_failures(status)
get_browser_console_errors()
```

핵심 관점:

> Observability for Humans에서 Observability for Agents로 확장한다.

## Compute-aware Orchestration

PM/Scheduler가 model만 고르지 않는다.

판단 대상:

- Model
- Context
- CPU
- RAM
- Timeout
- Network
- Number of Agents
- Execution Hand
- Retry Budget

## Warm Pool

unit test, lint, compile, small PR verification처럼 짧고 빈번한 Task는 READY 상태의 Runner pool을 사용할 수 있다.

장시간 작업은 별도 ephemeral worker를 사용할 수 있다.

## Shadow Agent

실제 수정 권한은 Main Agent만 갖고 Shadow Agent는 read-only risk review를 수행한다.

판단 불일치가 큰 경우 Human Escalation으로 전환한다.

## Canary Agent

새 model/harness/toolchain을 전체 Task에 즉시 적용하지 않고 일부 Task에 먼저 적용한다.

비교 지표:

- success rate
- token usage
- execution time
- retry
- human intervention
- regression

## Agent Replay

다음 정보를 replay 가능하게 기록한다.

- Task input
- Prompt/instruction version
- Context reference
- Model/version
- Tool calls/results
- Environment version
- Git SHA
- Agent output
- Test result

새 model/harness/tool 변경 전후를 동일 Task로 비교한다.

## Agent Platform CI

Application만 테스트하지 않고 Agent workflow도 benchmark suite로 평가한다.

지표:

- 해결 성공률
- 평균 token
- 평균 execution time
- retry
- human intervention
- regression

핵심 원칙:

> Agent도 배포하고, 관찰하고, 테스트하고, 회귀 검증해야 하는 소프트웨어 시스템이다.

# Agent-Native Repository

향후 Repository는 사람이 읽는 문서뿐 아니라 Agent가 탐색·실행·검증·관찰할 interface를 제공한다.

후보 구조:

```text
repository/
├─ AGENTS.md
├─ docs/
├─ scripts/
│  ├─ build
│  ├─ test
│  ├─ verify
│  └─ architecture-check
├─ agent/
│  ├─ task-template
│  ├─ result-schema
│  ├─ failure-patterns
│  └─ benchmarks
└─ observability/
   ├─ metrics
   ├─ traces
   └─ queries
```

# Current vs Future

현재 개발팀이 우선 적용할 수 있는 범위:

- deterministic command/validation
- Result Gateway / Artifact First
- Progressive Context
- Task Context Package
- Prebuilt Environment / Cache
- isolated branch/worktree/container
- Budget / Retry / Failure Fingerprint
- event-driven PR workflow
- Agent-friendly log
- 최소 Agent benchmark

플랫폼 성숙 후 확장 범위:

- Brain / Hands / Session 완전 분리
- One Brain, Multiple Hands
- Agent Hibernate
- Compute-aware dynamic scheduling
- Warm Pool
- Best-of-N
- Shadow / Canary Agent
- Execution Replay
- Agent-native Observability query plane
- Garbage Collector Agent

# Research Sources

- `research/anthropic/agent-native-development-environment.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/github/continuous-ai-runner-first.md`

# In Progress

Phase 5 장별 설계.

1장, 2장, 3장의 순차 설계는 완료했다.

21장 신규 주제는 본문이 아니라 planning 수준의 설계만 선반영했다.

# Next

순차 설계는 기존 계획대로 4장으로 진행한다.

`chapters/04/plan.md` - Repository as Interface: Agent가 이해할 수 있는 저장소

4장에서 Progressive Disclosure, Context Budget, canonical source, Agent-Native Repository의 기초를 상세히 설계한다.

Phase 6 본문 초고는 시작하지 않는다.

# Open Questions

- `Agent-Native Development Environment`를 최종 21장 제목으로 유지할지
- Brain / Hands / Session을 책의 정식 용어로 사용할지
- deterministic sampling 정책을 예제 코드까지 구현할지
- Result Gateway interface를 CLI/API 중 어디까지 표준화할지
- Agent Hibernate의 snapshot 범위를 workspace까지 포함할지 task/session state로 제한할지
- Best-of-N 적용 기준을 Task Contract 필드로 둘지
- Shadow Agent와 Canary Agent를 본문 또는 보조 패턴으로 둘지
- Agent Replay artifact의 보안/개인정보 제거 규칙을 어느 장에서 상세히 다룰지
- Agent Platform benchmark suite를 예제 프로젝트에 실제 구현할지
