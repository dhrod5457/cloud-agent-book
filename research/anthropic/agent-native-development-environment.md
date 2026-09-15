# Agent-Native Development Environment 조사

작성 기준일: 2026-09-16

이 문서는 `Agent-Native Development Environment` 장의 설계 근거를 정리한다. 제품 기능이나 수치를 책의 일반 원칙으로 직접 일반화하지 않는다.

## 1. Infrastructure configuration이 Agent 성능에 미치는 영향

Anthropic Engineering의 `Quantifying infrastructure noise in agentic coding evals`는 agentic coding benchmark에서 runtime infrastructure가 결과에 영향을 줄 수 있음을 실험했다.

실험 조건은 다음 요소를 고정했다.

- 동일 Claude model
- 동일 harness
- 동일 Terminal-Bench 2.0 task set

변경한 것은 container resource configuration이다.

공개된 결과에서는 가장 제한적인 구성과 가장 여유 있는 구성 사이에 Terminal-Bench 2.0 성공률 차이가 6 percentage points 발생했다.

또한 strict resource enforcement 환경에서 infrastructure error가 높아질 수 있음을 설명한다. 예시로 3x ceiling을 적용했을 때 infra error rate가 5.8%에서 2.1%로 감소했다고 공개했다.

이 사례에서 책이 추출할 원칙:

> 같은 모델이라도 실행환경이 다르면 실제로 수행할 수 있는 전략과 성공률이 달라질 수 있다.

메모리나 CPU가 부족하면 단순히 실행이 느려지는 데 그치지 않는다.

- dependency install 실패
- subprocess 실행 제한
- integration test 실패
- large build 실패
- transient spike에 따른 OOM kill
- environment failure를 Agent가 분석하고 retry하면서 추가 token 소비

따라서 Agent 성능을 model score만으로 설명하지 않고 다음 함수의 결과로 본다.

```text
Agent Performance
= f(Model, Context, Harness, Tools, CPU, RAM, Runtime,
    Network, Validation, Observability, Orchestration)
```

출처:

- Anthropic Engineering, `Quantifying infrastructure noise in agentic coding evals`, 2026-02-05
- https://www.anthropic.com/engineering/infrastructure-noise

## 2. Brain / Hands / Session 분리

Anthropic Engineering의 `Scaling Managed Agents: Decoupling the brain from the hands`는 hosted agent architecture에서 session, harness, sandbox를 분리하는 구조를 설명한다.

초기 구조에서는 agent components가 하나의 container lifecycle에 묶여 있었으나 이후 다음을 분리했다.

- Session: durable event log / state
- Brain: Claude + harness
- Hands: sandbox와 external tools

핵심 인터페이스 관점은 sandbox를 Agent 내부의 고정된 환경으로 보는 것이 아니라 필요할 때 호출하는 tool로 취급하는 것이다.

일반 원칙:

> Agent 안에 컴퓨터가 있는 것이 아니라, Agent가 필요할 때 컴퓨터를 호출한다.

설계상 얻는 효과:

- sandbox failure와 session state를 분리
- harness restart와 durable session을 분리
- sandbox가 필요 없는 task에서는 provisioning 생략 가능
- 하나의 brain이 여러 execution environment를 사용할 수 있음
- sandbox와 credential/security boundary를 분리 가능

Anthropic은 brain/hands 분리 후 container가 필요하지 않은 session에서는 container provisioning을 늦출 수 있었고, 해당 architecture에서 p50 time-to-first-token이 약 60%, p95가 90% 이상 감소했다고 공개했다. 이 수치는 해당 시스템의 구현 결과이며 일반 성능 예상값으로 사용하지 않는다.

출처:

- Anthropic Engineering, `Scaling Managed Agents: Decoupling the brain from the hands`, 2026-04-08
- https://www.anthropic.com/engineering/managed-agents

## 3. One Brain, Multiple Hands

같은 Managed Agents 자료는 한 brain이 여러 hands를 사용할 수 있는 구조를 설명한다.

`execute(name, input) -> string`처럼 실행환경을 추상화하면 hand는 다음이 될 수 있다.

- Linux sandbox
- browser
- mobile/emulator 환경
- custom tool
- MCP server
- external resource

책에서는 이 구조를 특정 API가 아니라 다음 일반 패턴으로 사용한다.

```text
Agent Brain
   |
   +-- Linux Hand
   +-- Browser Hand
   +-- Android Hand
   +-- Database Hand
```

## 4. Agent-friendly test output과 deterministic sampling

Anthropic Engineering의 `Building a C compiler with a team of parallel Claudes`는 장기 병렬 coding agent harness를 만들면서 test harness를 Agent에 맞게 설계한 사례를 공개했다.

### Agent-friendly log

공개 글은 test harness가 수천 바이트의 불필요한 출력을 context에 넣지 않도록 해야 하며, 중요한 정보는 file에 저장하고 오류는 자동 검색하기 쉬운 형식으로 출력하는 것이 유리하다고 설명한다.

책에서는 다음 원칙으로 추상화한다.

> Agent가 읽기 쉬운 출력 형식도 개발환경의 일부다.

### Deterministic sampling

같은 실험에서 fast test mode는 전체 suite의 1% 또는 10% random sample을 실행하도록 설계됐다.

특징:

- 동일 Agent에서는 deterministic sample을 사용해 regression 재현 가능
- 서로 다른 VM/Agent에서는 다른 sample을 사용해 aggregate coverage 확대
- 장시간 full suite 때문에 Agent가 진행을 멈추는 문제를 줄임

책에서는 이를 다음 구조로 확장한다.

```text
Fast Feedback Test
+
Distributed Deterministic Sampling
+
Final Full Validation
```

예시 정책은 프로젝트별로 결정한다.

```text
개발 중: 작은 deterministic sample
PR: 더 넓은 sample
Merge Queue: full validation
```

숫자 자체를 일반 규칙으로 고정하지 않는다.

### Parallel task 설계

C compiler 실험에서는 독립 failure가 많을 때 여러 Agent가 서로 다른 문제를 처리할 수 있었지만, Linux kernel compile처럼 하나의 큰 sequential bottleneck에서는 여러 Agent가 같은 문제를 반복해 병렬성이 제대로 작동하지 않았다.

책에서는 다음 원칙으로 사용한다.

- 독립 task만 fan-out한다.
- 같은 bottleneck이나 shared file을 여러 Agent가 동시에 수정하게 하지 않는다.
- fan-out 전 dependency와 validation independence를 확인한다.

출처:

- Anthropic Engineering, `Building a C compiler with a team of parallel Claudes`, 2026-02-05
- https://www.anthropic.com/engineering/building-c-compiler

## 5. 책에서 직접 일반화하지 않을 사항

다음은 특정 Anthropic 구현 또는 실험 결과이므로 일반 설계 원칙과 분리한다.

- 특정 resource multiplier가 모든 workload에서 적절하다는 주장
- p50/p95 TTFT 개선율을 다른 Agent platform에도 적용하는 것
- C compiler 실험의 Agent 수나 비용을 일반적인 개발팀 기준으로 사용하는 것
- 1%/10% sampling 비율을 모든 프로젝트의 기본값으로 사용하는 것
- Anthropic Managed Agents의 구체적인 API를 표준 interface로 취급하는 것

## 6. 장 설계에 반영할 근거

이 조사에서 다음 주제를 `Agent-Native Development Environment` 설계에 반영한다.

- Agent Performance = Model + Context + Harness + Tools + Compute + Validation + Observability + Orchestration
- Brain / Hands / Session lifecycle 분리
- One Brain, Multiple Hands
- Compute-aware Orchestration
- Agent-friendly Log
- Distributed Deterministic Sampling
- independent work만 fan-out
- execution environment를 Agent capability의 일부로 평가

제품별 변경 가능한 수치와 구현 세부사항은 본문의 핵심 주장과 분리한다.
