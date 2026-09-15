# Current Phase

Phase 5 - 장별 설계 진행 중

# Completed

- Phase 1 방향 정의
- Phase 2 전체 목차 설계
- Phase 3 목차 검증
- Phase 4 `campus-platform` 예제 프로젝트 설계
- Phase 5 1장 설계
- Phase 5 2장 설계
- Phase 5 2장 `Agent-friendly Execution Platform` 확장 설계
- Phase 5 3장 `Agent Ready 프로젝트의 기준` 설계
- Cloud compute resource와 LLM usage 분리
- Runner-first / Agent-on-exception 구조
- Result Gateway / Artifact First 설계
- Prebuilt Environment / Cache 정책
- Deterministic First 원칙
- Progressive Context / Task Context Package
- Budgeted Autonomy / Failure Fingerprint / Retry 정책
- Event-driven Agent / Continuous Small Task
- Fan-out / Fan-in
- Harness Engineering
- Agent Ready 9개 평가 기준
- `campus-platform` Stage 0 Agent Ready baseline

# Phase 4 Artifacts

- `examples/campus-platform/README.md`
- `examples/campus-platform/architecture.md`
- `examples/campus-platform/testing.md`
- `examples/campus-platform/evolution.md`
- `examples/campus-platform/cloud-test-runner.md`
- `examples/campus-platform/execution-platform.md`
- `examples/campus-platform/agent-ready-baseline.md`

# Phase 5 Artifacts

- `chapters/01/plan.md`
- `chapters/02/plan.md`
- `chapters/02/execution-platform.md`
- `chapters/03/plan.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/github/continuous-ai-runner-first.md`

# Decisions

## 책의 기본 방향

- 상위 개념은 `Agent Ready Software Engineering`이다.
- Java/Spring Boot는 주요 실전 예제지만 일반 원칙은 언어와 제품에 종속되지 않는다.
- 특정 제품의 사용법보다 프로젝트 구조, 실행환경, 검증, orchestration 설계를 중심으로 다룬다.
- 프로젝트 파일을 Source of Truth로 사용한다.

## Cloud Agent 핵심 원칙

2장에서는 다음 원칙을 유지한다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

추가 원칙:

- 환경은 미리 준비한다.
- 가능한 판정은 코드로 처리한다.
- Context는 필요할 때 필요한 만큼만 제공한다.
- 실패 결과 전체를 전달하지 않고 조회 가능한 형태로 저장한다.
- Agent의 자율성에는 시간, retry, token, 비용 한도를 둔다.
- Agent가 반복해서 실패하면 프롬프트보다 실행환경과 도구를 먼저 개선한다.

클라우드 실행 구조의 목표:

> AI를 최대한 많이 사용하는 시스템이 아니라, AI가 반드시 필요한 순간에만 호출되는 시스템.

더 확장된 관점:

> 클라우드 에이전트의 최종 형태는 Agent를 계속 실행하는 시스템이 아니라, Agent가 필요할 때 붙을 수 있도록 잘 준비된 실행 플랫폼이다.

## Agent-friendly Execution Platform

권장 구조:

```text
                PM / Orchestrator
                       |
                Task Scheduler
                       |
               Task Classification
                       |
        +--------------+--------------+
        |                             |
 Deterministic                   Reasoning Required
        |                             |
        v                             v
  Cloud Runner                   Agent Worker
        |                             |
 Build/Test/E2E                  Analyze/Fix
        |                             |
        +------------+----------------+
                     |
                Result Gateway
                     |
              +------+------+
              |             |
             PASS          FAIL
              |             |
             Done       Budget Check
                            |
                       +----+----+
                       |         |
                     Retry   Human Escalation
```

기반 계층:

- Prebuilt Environment
- Reusable Cache
- Artifact Store
- Repository Harness
- Progressive Documentation
- Isolated Worktree / Container

## Prebuilt Environment

Agent/Runner가 매번 JDK, Node, dependency, Playwright, Docker tool을 처음부터 준비하지 않도록 한다.

핵심 원칙:

> Agent에게 개발환경을 설치하게 하지 않는다. 이미 작업할 수 있는 환경을 준다.

## Cache

Reusable Cache:

- Gradle/Maven dependency
- npm cache
- Docker layer
- Playwright browser
- compiler cache
- immutable code generation result

Disposable Runtime State:

- DB
- temporary file
- mutable test data
- test output
- browser/process/session state

Cache 재사용과 테스트 상태 격리를 분리한다.

## Deterministic First

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

대상:

- build
- test
- lint
- architecture rule
- migration validation
- security scan
- dependency check

가능한 검증은 executable validation으로 만든다.

## Result Gateway / Artifact First

Result Filter를 다음 구조로 확장한다.

```text
Raw Artifact
→ Result Gateway
→ Summary / Failure Index / Stack Trace Lookup / Log Search / Artifact Lookup
→ Agent
```

원본 로그를 요약 후 폐기하지 않는다.

각 Task는 `result.json`, `junit.xml`, `coverage.xml`, `build.log`, `git.diff`, screenshot/video 등의 artifact를 남길 수 있다.

Agent는 기본적으로 `result.json`만 읽고 필요할 때 특정 artifact를 조회한다.

핵심 원칙:

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

## Progressive Context

Repository 전체 문서를 한꺼번에 Context로 전달하지 않는다.

```text
AGENTS.md
→ 작업 유형별 문서
→ 관련 소스
→ 관련 테스트
→ 필요한 failure artifact
```

핵심 원칙:

> Context를 줄이는 것뿐 아니라 Context를 필요할 때 가져오게 만든다.

## Task Context Package

Agent 호출 시 다음 범위를 작게 제공한다.

- Task
- Failure
- 관련 파일
- 검증 명령
- 변경 금지 영역
- 완료 조건
- Budget

정식 형식은 6장 Task Contract에서 다룬다.

## Isolation

병렬 Agent는 같은 Working Directory를 공유하지 않는다.

- task branch
- worktree
- 독립 VM/container
- 독립 artifact path

같은 파일/schema를 수정하는 작업은 처음부터 병렬화하지 않는다.

## Failure Container Retention

성공 container는 즉시 폐기할 수 있다.

실패 container는 짧은 TTL 동안 freeze/retain하여 Agent나 사람이 실패 당시 상태를 확인할 수 있게 한다.

유용한 대상:

- Testcontainers/DB state
- race condition
- browser state
- filesystem/process issue
- network timeout

## Budgeted Autonomy

> Agent의 자율성은 무제한 실행 권한이 아니라 예산 안에서 스스로 해결할 수 있는 권한이다.

Budget 후보:

- max wall-clock time
- max turns
- max retry
- max tokens
- max cost
- max changed files
- max diff size

Budget 소진 시 Human Escalation으로 전환한다.

## Retry / Failure Fingerprint

`성공할 때까지 계속 수정` 정책을 사용하지 않는다.

Retry마다 실패가 달라졌는지 확인한다.

동일 failure fingerprint가 반복되면 중단한다.

Fingerprint 후보:

- failing test id
- exception type
- assertion message
- error code
- top stack frame

## Event-driven Agent

Agent는 상시 프로세스가 아니다.

활성화 이벤트 후보:

- Runner/CI failure
- review comment
- nightly regression failure
- security alert
- dependency update failure
- human escalation

PASS 경로에서는 Agent를 호출하지 않는다.

## Continuous Small Task

거대한 Agent 작업보다 작은 검증 가능한 Task를 지속적으로 반복한다.

장점:

- 작은 Context
- 작은 실패 범위
- 쉬운 rollback
- 쉬운 verification
- 작은 PR
- 예측 가능한 token budget

GitHub Continuous AI 공개 사례는 이 패턴의 사례로만 사용한다.

## Fan-out / Fan-in

서로 다른 repository/module/file scope처럼 독립 검증 가능한 작업만 fan-out한다.

공통 schema/common file 의존성이 있으면 dependency를 먼저 분석하고 순차 작업으로 전환한다.

## Harness Engineering

Agent가 반복해서 실패하면 먼저 Harness 부족을 확인한다.

예:

- 테스트 명령을 못 찾음 → AGENTS.md/실행 인터페이스 개선
- 로그가 너무 큼 → Result Gateway
- 환경 설치 실패 → Prebuilt Image
- Architecture 위반 반복 → architecture-check
- 동일 오류 반복 → failure fingerprint

핵심 원칙:

> Agent가 반복해서 실패하면 프롬프트보다 Harness를 먼저 개선한다.

## 비용 모델

세 종류를 분리한다.

### Compute Cost

- CPU
- RAM
- Storage
- Container runtime

### LLM Cost

- input/output token
- reasoning
- tool result processing

### Human Cost

- waiting
- review
- reproduction
- context switching

핵심 기준:

> 싼 deterministic compute로 해결할 수 있는 문제에 비싼 probabilistic reasoning을 사용하지 않는다.

다만 전체 비용은 token 하나가 아니라 compute, LLM, 사람 시간을 함께 본다.

## Agent Ready 평가 기준

3장에서는 Agent Ready를 9개 기준의 `Agent Ready Profile`로 평가한다.

1. Reproducibility
2. Discoverability
3. Executability
4. Testability
5. Verifiability
6. Isolation
7. Parallelizability
8. Observability
9. Security Boundary

각 기준은 `PASS / PARTIAL / FAIL`과 실행 가능한 Evidence를 사용한다.

Cloud Runner 도입에는 Reproducibility, Executability, Testability, Isolation, Observability를 우선한다.

Cloud Agent Worker에는 Discoverability, Verifiability, Security Boundary가 추가로 중요하다.

# In Progress

Phase 5 장별 설계.

현재 1장, 2장, 3장의 설계가 작성되었다.

2장은 `chapters/02/plan.md`와 `chapters/02/execution-platform.md` 두 문서로 관리한다.

`execution-platform.md`는 Cloud Agent에서 Agent-friendly Execution Platform으로 확장한 세부 설계다.

# Next

Phase 5를 계속 진행한다.

다음 대상:

`chapters/04/plan.md` - Repository as Interface: Agent가 이해할 수 있는 저장소

4장에서는 다음을 구체화한다.

- Repository Layout
- canonical source
- Progressive Disclosure
- Context Budget
- Progressive Context
- naming
- generated file
- migration 위치
- change locality
- Agent가 전체 Repository를 읽지 않고 필요한 영역을 찾는 구조

장 설계가 확정되기 전에는 Phase 6 본문 초고를 시작하지 않는다.

# Open Questions

Phase 5 이후 구현/집필 과정에서 검증한다.

- Result Gateway API/CLI 형식을 어디까지 표준화할지
- Runner result schema를 JSON Schema로 고정할지
- failure fingerprint의 일반 형식을 정의할지
- 실패 container retention을 예제 구현까지 포함할지
- Prepared Image를 Dockerfile/Dev Container 중 어떤 형태로 예시할지
- Agent Budget을 Task Contract의 필수 필드로 둘지 선택 필드로 둘지
- Agent Harness와 Harness Engineering을 책의 정식 용어로 확정할지
- Agent Ready Profile을 YAML/JSON machine-readable 형식으로 제공할지
