# Agent-Native Development Environment 장 설계

## Phase

Phase 5 - 장별 설계 보강

본문은 작성하지 않는다. 이 문서는 새 장의 위치, 경계, 핵심 주장, 절 구성, 사례, 조사 근거를 정의한다.

## 배치 결정

이 주제는 2장 `Local Agent, Cloud Agent, Hybrid Agent`의 확장 절만으로 처리하지 않는다.

이유:

- 2장은 실행 위치와 Runner/Agent 분리의 기초를 설명한다.
- 이번 주제는 Repository, Validation, Observability, Orchestration, Agent Lifecycle을 모두 전제로 한다.
- 4장, 8장, 10장, 17장, 19장의 개념을 종합한 뒤 설명해야 중복이 줄어든다.

따라서 Phase 5 목차 보강 시 다음 위치에 독립 장으로 추가한다.

```text
20장 실패, 재시도, 충돌, 통합
        ↓
21장 Agent-Native Development Environment
        ↓
22장 VPN과 내부망이 있는 Hybrid Agent 시스템
        ↓
23장 Full Agentic Development로의 진화와 Governance
```

기존 21장은 22장으로, 기존 22장은 23장으로 이동한다.

## 장 제목

`Agent-Native Development Environment: Agent가 일하기 좋은 개발환경 설계`

## 장의 목적

Cloud Agent를 원격 개발자 또는 원격 container로 보는 수준을 넘어, Agent가 탐색·실행·검증·관찰·복구할 수 있도록 개발환경 자체를 재설계하는 방법을 설명한다.

핵심 관점:

> 좋은 Agent 시스템은 좋은 모델 하나로 만들어지는 것이 아니라, Model + Context + Harness + Tools + Compute + Validation + Observability + Orchestration이 함께 만들어낸다.

최종 변화:

> Agent를 개발환경에 넣는 것이 아니라 개발환경을 Agent 중심으로 다시 설계한다.

## 선행 장

필수 선행:

- 4장 Repository as Interface
- 8장 Agent 실행 인터페이스
- 10장 자동 검증과 Definition of Done
- 13장 Git / Worktree / Cloud Sandbox
- 16장 Project Memory와 Long Running Task
- 17장 Agent Observability와 Quality Evaluation
- 19장 PM Agent
- 20장 실패, 재시도, 충돌, 통합

2장에서 정의한 `Agent-friendly Execution Platform`을 기초 전제로 사용하되 반복 설명하지 않는다.

## 독자가 얻는 것

- Agent 성능을 model 성능과 runtime 환경으로 분리해 평가할 수 있다.
- Brain, Session, Hands의 lifecycle을 분리할 수 있다.
- 하나의 Agent가 Linux, Browser, Android, Database 등 여러 execution hand를 사용하게 설계할 수 있다.
- full test와 deterministic sampling을 작업 단계별로 배치할 수 있다.
- Agent-friendly log와 query interface를 설계할 수 있다.
- Agent를 sleep/wake할 수 있는 durable session 구조를 설계할 수 있다.
- PR Babysitter, Garbage Collector, Shadow Agent, Canary Agent의 적용 조건을 판단할 수 있다.
- Best-of-N과 deterministic validation을 결합할 수 있다.
- execution replay와 Agent regression test를 설계할 수 있다.
- PM Agent가 model뿐 아니라 CPU, RAM, timeout, hand, agent count까지 scheduling하도록 확장할 수 있다.

---

# 핵심 개념 모델

## Agent Performance

개념식:

```text
Agent Performance
= f(
    Model,
    Context,
    Harness,
    Tools,
    CPU,
    RAM,
    Runtime,
    Network,
    Validation,
    Observability,
    Orchestration
  )
```

핵심 메시지:

> 같은 모델이라도 실행환경이 다르면 다른 수준의 개발자처럼 행동할 수 있다.

Anthropic infrastructure-noise 연구는 동일 model, 동일 harness, 동일 task set에서도 resource configuration 차이로 성공률이 달라질 수 있음을 실제 사례로 사용한다.

구체적 수치는 본문 핵심 논리와 분리해 research 문서에 둔다.

---

# 절 구성

## 21.1 Agent 성능은 모델 성능만으로 결정되지 않는다

설명 범위:

- CPU/RAM 부족이 단순 latency 문제가 아닌 이유
- dependency install, subprocess, integration test, build 전략 자체가 제한될 수 있음
- infrastructure failure를 Agent가 reasoning 문제로 오인하면 retry와 token이 증가할 수 있음

비교 사례:

```text
충분한 환경
Agent
→ dependency 설치
→ 여러 subprocess
→ integration test
→ large build
→ 문제 해결
```

```text
제한된 환경
Agent
→ dependency 설치 실패
→ test 실패
→ environment error 분석
→ retry
→ token 소비
```

공식 근거:

- `research/anthropic/agent-native-development-environment.md`

## 21.2 Brain과 Hands의 lifecycle을 분리한다

기존 coupled 구조:

```text
Agent Container
- LLM
- Harness
- Shell
- Repository
- Browser
- Tools
```

권장 추상화:

```text
Session
|
+-- Brain
|    - LLM
|    - Agent Harness
|    - Task State
|
+-- Hands
     - Linux Sandbox
     - Browser
     - Android Emulator
     - Database
     - External Tool
```

원칙:

> Agent 안에 컴퓨터가 있는 것이 아니라, Agent가 필요할 때 컴퓨터를 호출한다.

효과:

- sandbox failure와 session state 분리
- brain restart와 durable state 분리
- 필요 없는 sandbox provisioning 생략
- 하나의 brain에서 여러 hands 사용

## 21.3 One Brain, Multiple Hands

예제:

```text
                 Agent Brain

       +-------------+-------------+
       |             |             |
    Linux         Browser       Android
       |             |             |
 Backend Test      Web UI      Mobile App
       \             |             /
        +------------+------------+
                     |
                  Database
                     |
              Migration Test
```

`campus-platform` 예제:

하나의 Verification Agent가 다음을 다른 execution hand에서 검증한다.

- Spring Boot backend
- 관리자 Web
- Android app
- DB migration

VM, Container, Browser, Emulator를 모두 `Agent가 호출하는 실행 도구`로 추상화한다.

## 21.4 Deterministic Sampling

목표:

- 모든 Agent가 매번 full suite를 실행하는 비용 감소
- 빠른 feedback과 distributed coverage를 동시에 확보

구조:

```text
Fast Feedback Test
+
Distributed Deterministic Sampling
+
Final Full Validation
```

예시 정책:

```text
개발 중
→ small deterministic sample

PR
→ wider sample

Merge Queue
→ full test
```

중요 조건:

- 같은 Task/Agent의 retry에서는 같은 sample을 사용해 재현성 유지
- 다른 Worker는 다른 sample을 사용해 aggregate coverage 확대
- merge 전에는 full validation 유지

장점과 위험을 함께 설명한다.

장점:

- feedback latency 감소
- CPU 낭비 감소
- Agent waiting 감소

위험:

- sample 밖 regression 누락 가능
- sample seed 관리 필요
- final full validation이 없으면 품질 gate 약화

Anthropic C compiler harness 사례를 근거로 사용한다.

## 21.5 Agent-friendly Log

사람에게 보기 좋은 verbose log와 Agent가 처리하기 좋은 machine-readable output을 구분한다.

좋지 않은 출력:

```text
Running test 1
Running test 2
DEBUG ...
INFO ...
WARN ...
...
```

권장 출력:

```text
TOTAL=10000
PASS=9997
FAIL=3

ERROR AuthTest.expiredToken expected=401 actual=200
ERROR UserTest.delete NullPointerException UserService.java:142
ERROR MapperTest.insert duplicate_key
```

상세 log는 Artifact Store에 보존한다.

핵심 원칙:

> Agent가 읽기 쉬운 출력 형식도 개발환경의 일부다.

2장의 Result Gateway와 연결한다.

## 21.6 Agent Hibernate

Agent/Worker를 항상 RUNNING 상태로 유지하지 않는다.

상태 모델:

```text
RUNNING
→ IDLE
→ SNAPSHOT
→ OFF
→ WAKE
→ RUNNING
```

snapshot 후보:

- repository state
- branch
- dependency/cache reference
- build artifact
- durable task state

PR follow-up 예:

```text
PR 수정
→ Agent 작업
→ CI 대기
→ Agent sleep

Review Comment
→ Agent wake
→ workspace/session 복구
→ 수정
→ 테스트
```

중요 구분:

- durable session/task state
- disposable sandbox/runtime

2장의 Prebuilt Environment/Cache와 16장의 Project Memory를 연결한다.

## 21.7 PR Babysitter Agent

PR 생성 후 merge-ready까지 lifecycle을 관리하는 Agent role을 소개한다.

```text
PR
|
v
PR Babysitter
|
+-- CI 확인
+-- failure 대응
+-- review comment 처리
+-- conflict 감지
+-- test 재실행
+-- merge-ready 상태 유지
|
v
Developer Review
```

적용 후보:

- CI failure response
- reviewer feedback
- merge conflict detection
- flaky test retry 정책 실행
- dependency update verification
- merge queue monitoring

자동 merge 권한과는 분리한다. Trust Boundary와 CI Gate를 적용한다.

## 21.8 Speculative Coding / Best-of-N

어려운 Task에서 여러 후보를 병렬 생성하고 deterministic validation으로 선택하는 패턴을 설명한다.

```text
Bug
|
+---------+---------+
|         |         |
Agent A   Agent B   Agent C
|         |         |
Patch A   Patch B   Patch C
 \         |         /
  +--------+--------+
           |
       Validation
           |
       Evaluation
           |
         Winner
```

후보 선택 기준:

1. tests
2. security validation
3. regression
4. diff size
5. complexity
6. cost
7. execution time

핵심:

> 후보를 LLM의 선호만으로 고르지 않고 가능한 항목은 코드로 검증한다.

메시지:

> 비싼 모델 하나에게 오래 고민시키는 것과 여러 후보를 병렬로 만든 뒤 자동 검증으로 선택하는 것 중 어느 것이 더 싼지는 Task마다 다르다.

## 21.9 Best-of-N 비용 제어

Candidate count를 task difficulty에 따라 결정한다.

정책 예시는 고정 규칙이 아니라 설명용으로만 사용한다.

```text
Task Difficulty
→ Candidate Count
→ Parallel Execution
→ Validation
→ Winner
```

N 증가 시 LLM cost도 증가하므로 기본값은 N=1이다.

Best-of-N은 어려운 bug, security-critical change, production incident 등 검증 가치가 높은 작업에서 제한적으로 사용한다.

## 21.10 Time-travel Debugging

Agent가 실패 시점의 execution artifact를 재생하거나 조회할 수 있게 한다.

```text
Bug
→ Execution Recording
→ failure point lookup
→ network/DOM/state/call-chain 확인
→ root cause 분석
```

Browser/UI artifact 예:

- screenshot
- video
- network log
- DOM snapshot
- browser trace
- Playwright trace

이 절은 특정 replay 제품을 전제로 하지 않는다.

2장의 Artifact First / Result Gateway를 `실패 당시 상태 조회`로 확장한다.

## 21.11 Garbage Collector Agent

Feature Agent의 지속적인 변경으로 쌓일 수 있는 다음 항목을 정기적으로 검사한다.

- duplication
- architecture drift
- stale documentation
- inconsistent naming
- dead code
- outdated dependency
- abandoned feature flag
- temporary workaround

구조:

```text
Nightly / Weekly
→ Repository Scan
→ Garbage Collector Agent
→ deterministic validation
→ Small Cleanup PR
```

대규모 자동 refactoring보다 작은 cleanup PR을 반복하는 방식으로 제한한다.

메시지:

> Agent가 만든 기술부채를 다른 Agent가 지속적으로 회수한다.

## 21.12 Agent-native Observability

Observability를 사람이 Dashboard에서 보는 기능으로 제한하지 않는다.

Agent가 구조화된 interface로 직접 조회할 수 있게 한다.

```text
Agent
  |
  +-- Source
  +-- Logs
  +-- Metrics
  +-- Trace
  +-- DOM
  +-- Screenshot
  +-- Database
  +-- Browser
  +-- Video
```

예:

```text
get_errors(service, since)
get_slow_spans(service, threshold_ms=2000)
get_metric(name="startup_time")
get_http_failures(status=500)
get_browser_console_errors()
```

핵심:

> Observability for Humans에서 Observability for Agents로 확장한다.

17장에서는 무엇을 관찰할지 정의하고, 21장에서는 Agent가 직접 사용할 query interface를 설계한다.

## 21.13 Compute-aware Orchestration

19장의 PM Agent scheduling을 compute까지 확장한다.

PM/Scheduler가 판단할 수 있는 항목:

- Model
- Context
- CPU
- RAM
- Timeout
- Network
- Number of Agents
- Execution Hand
- Retry Budget

Task 예:

```text
Task A
- type: unit-test
- model: none
- cpu: 2
- memory: 4GB
- execution: cloud-runner

Task B
- type: spring-integration-test
- model: none
- cpu: 4
- memory: 16GB

Task C
- type: complex-debug
- model: strong
- cpu: 8
- memory: 32GB
- agent_count: 2

Task D
- type: e2e
- hand: browser

Task E
- type: mobile-test
- hand: android-emulator
```

숫자는 예시이며 특정 제품 사양으로 고정하지 않는다.

## 21.14 Warm Pool

짧고 빈번한 Task의 cold start를 줄이기 위해 준비된 Runner pool을 유지할 수 있다.

```text
Warm Pool

Runner #1 READY
Runner #2 READY
Runner #3 READY

Task
→ 즉시 할당
→ 작업
→ reset
→ READY
```

적합한 작업:

- unit test
- lint
- compile
- small PR verification

장시간 독립 작업은 별도 ephemeral worker를 사용할 수 있다.

## 21.15 Shadow Agent

실제 수정 권한은 Main Agent만 갖고 Shadow Agent는 read-only review를 수행한다.

```text
Main Agent
→ 실제 수정

Shadow Agent
→ 변경 없이 위험요소 평가
```

판단이 크게 다르면 Human Escalation으로 전환한다.

적용 후보:

- security-sensitive change
- schema/migration
- production-critical change

## 21.16 Canary Agent

새 model, harness, toolchain을 모든 Task에 즉시 적용하지 않는다.

```text
기존 Agent/Harness
→ 대부분의 Task

신규 Agent/Harness
→ 작은 비율의 Task
```

비교 지표:

- success rate
- token usage
- execution time
- retry count
- human intervention
- regression

Agent platform 자체에도 canary rollout을 적용한다.

## 21.17 Agent Replay

Agent 실패와 성공을 재현할 수 있도록 다음을 저장한다.

- Task input
- Prompt/instruction version
- Context references
- Model/version
- Tool calls
- Tool results
- Environment version
- Git SHA
- Agent output
- Test result

사용:

```text
Failed Task
→ Replay with new model
→ Replay with new harness
→ Replay with changed tool
→ Result comparison
```

Agent regression test의 기반으로 사용한다.

민감정보와 credential은 replay artifact에서 제거하거나 참조형으로 관리한다.

## 21.18 Agent 자체도 테스트한다

Application CI와 별도로 Agent Platform CI를 설계한다.

Benchmark Suite 예:

- Simple NullPointerException
- JWT expiration bug
- DB migration failure
- Frontend validation bug
- Architecture rule violation

비교 지표:

- 해결 성공률
- 평균 token usage
- 평균 execution time
- retry count
- regression
- human intervention

핵심:

> Agent도 배포하고, 관찰하고, 테스트하고, 회귀 검증해야 하는 소프트웨어 시스템이다.

## 21.19 Agent Platform 전체 구조

```text
                     PM / Orchestrator
                            |
                    Task Classification
                            |
       +--------------------+--------------------+
       |                    |                    |
 Deterministic          Agent Brain           Best-of-N
    Runner                  |                    |
       |            +-------+-------+            |
 Build/Test         Linux Browser Android     Candidate Agents
 Lint/Scan           Hand   Hand   Hand           |
       |                    |                    |
       +--------------------+--------------------+
                            |
                      Result Gateway
                            |
                  Deterministic Validation
                            |
                     +------+------+
                     |             |
                    PASS          FAIL
                     |             |
               PR Babysitter   Retry/Budget
                     |             |
                   Merge      Human Escalation
                     |
              Garbage Collector
                     |
              Continuous Cleanup
```

기반 계층:

- Prebuilt Environment
- Snapshot
- Cache
- Warm Pool
- Artifact Store
- Agent-native Observability
- Execution Replay
- Repository Harness
- Progressive Context
- Isolated Worktree
- Budget Management

## 21.20 Agent-Native Repository

4장 Repository as Interface를 플랫폼 수준에서 재조합한다.

예시 구조:

```text
repository/
├─ AGENTS.md
├─ docs/
│  ├─ architecture/
│  ├─ database/
│  ├─ security/
│  └─ testing/
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

목표는 문서를 많이 만드는 것이 아니라 Agent가 탐색, 실행, 검증, 관찰할 수 있는 interface를 Repository가 제공하는 것이다.

---

# 현재 적용 가능한 구조와 확장 구조를 분리한다

## 현재 개발팀이 우선 적용할 수 있는 구조

- deterministic build/test/verify
- Result Gateway / Artifact First
- Progressive Context
- Task Context Package
- Prebuilt Environment / Cache
- isolated branch/worktree/container
- Budget / Retry / Failure Fingerprint
- PR Babysitter의 제한된 event-driven workflow
- Agent-friendly logs
- Agent benchmark 최소 세트

## 플랫폼 성숙 후 확장할 구조

- Brain / Hands / Session 완전 분리
- One Brain, Multiple Hands
- Agent Hibernate
- Compute-aware dynamic scheduling
- Warm Pool
- Best-of-N
- Shadow Agent
- Canary Agent
- Execution Replay
- Agent-native Observability query plane
- Garbage Collector Agent

이 구분을 통해 현재 구현 가능한 설계와 장기 플랫폼 방향을 섞지 않는다.

---

# 다른 장과의 경계

## 2장

Local/Cloud/Hybrid, Runner-first, Result Gateway, Prebuilt/Cache의 기본 원리만 설명한다.

21장에서는 이 요소들이 하나의 Agent-Native Development Environment로 통합될 때의 구조를 다룬다.

## 4장

Repository 탐색성, Progressive Disclosure, canonical source를 상세히 정의한다.

21장은 이를 Agent-Native Repository의 구성요소로 재사용한다.

## 8장

`setup`, `test`, `verify` 실행 interface 자체를 정의한다.

21장은 해당 interface를 Runner, Brain, Hands가 어떻게 호출하는지 다룬다.

## 10장

Definition of Done과 deterministic validation을 정의한다.

21장은 Best-of-N, PR Babysitter, Garbage Collector 결과를 해당 validation으로 평가한다.

## 16장

Project Memory와 durable task state를 정의한다.

21장은 Agent Hibernate, Session recovery와 연결한다.

## 17장

관찰 지표와 품질 평가를 정의한다.

21장은 Agent가 직접 observability data를 query하는 interface로 확장한다.

## 19장

PM Agent의 task decomposition과 scheduling을 정의한다.

21장은 scheduling 대상을 Model뿐 아니라 Compute, Hand, Agent Count로 확장한다.

## 20장

retry, failure, human escalation의 일반 정책을 정의한다.

21장은 replay, fingerprint, budget, environment retention과 통합한다.

## 22장 Hybrid Enterprise

21장의 execution abstraction을 VPN/HSM/Tibero/Jenkins 등 기업 내부망에 적용한다.

## 23장 Governance

21장에서 만든 Agent Platform 자체의 canary, benchmark, budget, audit 결과를 Governance 입력으로 사용한다.

---

# `campus-platform` 실전 예제

통합 앱 예제로 다음 execution hands를 사용한다.

```text
Verification Brain
|
+-- Backend Hand
|    Spring Boot / Gradle / Testcontainers
|
+-- Web Hand
|    Admin UI / Browser / Playwright
|
+-- Android Hand
|    Android Emulator / Instrumented Test
|
+-- DB Hand
     PostgreSQL/Tibero-compatible migration validation
```

예제 Task:

`로그인 토큰 만료 처리 회귀 검증`

1. Backend Hand에서 AuthService unit/integration test
2. Web Hand에서 만료 세션 UI 확인
3. Android Hand에서 재로그인 flow 확인
4. Result Gateway가 각 artifact를 통합
5. PASS면 종료
6. FAIL이면 해당 hand의 artifact만 Agent Brain에 전달
7. 수정 후 해당 scope 재검증
8. merge 전 Full Validation

---

# 조사 자료

- `research/anthropic/agent-native-development-environment.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/github/continuous-ai-runner-first.md`

제품별 변경 가능한 사양, 비용, session limit은 장의 핵심 논리와 분리한다.

---

# 장의 최종 핵심 문장

1. `LLM은 판단하고, 컨테이너는 실행한다.`
2. `Brain과 Hands의 생명주기를 분리한다.`
3. `하나의 정답을 기다리지 말고 필요하면 여러 후보를 만들고 코드로 검증한다.`
4. `Observability는 사람뿐 아니라 Agent도 직접 조회할 수 있어야 한다.`
5. `Agent도 배포하고, 관찰하고, 테스트하고, 회귀 검증해야 하는 소프트웨어 시스템이다.`
6. `좋은 Agent 시스템을 만드는 핵심은 더 긴 Prompt가 아니라 더 좋은 Environment와 Harness다.`
7. `Agent가 개발환경을 사용하는 단계를 넘어, 개발환경 자체가 Agent를 위해 설계되는 방향으로 간다.`

## 발전 단계

```text
1. AI에게 코드 작성을 요청
2. AI가 Repository에서 직접 작업
3. 여러 Agent 병렬 실행
4. Runner와 Agent 분리
5. Brain과 Hands 분리
6. Observability/Test/Artifact/Replay를 Agent가 직접 사용
7. PM Agent가 Model + Compute + Agent 수를 scheduling
8. Agent가 일하기 좋은 개발환경과 Repository 자체를 설계
```
