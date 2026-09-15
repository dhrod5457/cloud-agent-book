# Concept

## 중심 주제

이 책의 중심 주제는 **Cloud Agent를 실제 개발팀에서 어떻게 활용할 것인가**이다.

`Agent Ready Software Engineering`은 책 전체를 지지하는 설계 철학으로 남기지만, 독자가 따라가는 주된 질문은 Cloud Agent 활용이다.

## 이 책이 답할 핵심 질문

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

이를 다음 판단으로 나눈다.

- 왜 이 Task를 Local이 아니라 Cloud로 보내는가?
- Cloud Agent에게 보내기 전에 무엇을 준비해야 하는가?
- 어떤 Cloud Environment를 선택해야 하는가?
- Task와 Context 범위를 어디까지 줄여야 하는가?
- Token과 Cloud CPU/RAM을 어떻게 효율적으로 사용할 것인가?
- 어떤 Evidence를 결과로 돌려받아야 하는가?
- 여러 Cloud Agent를 어디까지 병렬로 사용할 것인가?
- Local과 Cloud 사이에서 작업을 어떻게 넘길 것인가?

## Cloud Agent의 핵심 정의

Cloud Agent를 단순히 클라우드에서 실행되는 LLM으로 설명하지 않는다.

```text
Cloud Agent
= LLM
+ Repository
+ 독립 실행환경
+ CPU
+ RAM
+ Disk
+ Development Tools
```

책에서 사용할 확장 정의:

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

따라서 Cloud Agent의 가치는 LLM 추론 능력뿐 아니라 독립 실행환경까지 포함해서 평가한다.

## 핵심 메시지

책 전체에서 다음 메시지를 유지한다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Cloud Agent에게 Repository 전체를 반복해서 이해시키지 않는다.

> 작은 Task와 작은 Context를 전달한다.

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

## Local과 Cloud의 역할

### Local Agent에 적합한 작업

- Architecture 설계
- 복잡한 디버깅
- 개발자의 빈번한 개입이 필요한 작업
- 내부망/VPN/사내 DB/HSM 접근
- 여러 모듈을 동시에 이해해야 하는 작업
- 빠르게 질문과 수정이 반복되는 작업
- 최종 통합과 Review

### Cloud Agent에 적합한 작업

- Unit Test
- Integration Test
- E2E Test
- Build / Docker Build
- Static Analysis / Lint
- Migration Validation
- 반복적인 Refactoring
- 작은 Bug Fix
- 독립적인 Feature
- Documentation
- PR Review
- CI 실패 수정
- 장시간 실행 작업
- 서로 독립적인 병렬 작업

이 분류는 절대 규칙이 아니다. 하나의 Task도 단계에 따라 Local과 Cloud 사이를 이동할 수 있다.

```text
Local
→ 요구사항 분석 / Architecture / Task 분해

Cloud
→ 독립 구현 / Build / Test / E2E / 반복 검증

Local
→ 내부망 검증 / 최종 Review / 통합 / Merge
```

핵심 원칙:

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하는 것이 아니다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

## Git은 Handoff Boundary다

Cloud Agent가 Remote Repository의 clean state에서 작업하는 구조에서는 Git이 단순한 형상관리 시스템을 넘어 작업 전달 경계가 된다.

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Task Branch
→ Test
→ Commit / Push
→ PR
```

> Cloud Agent 시대에는 Git이 개발자와 Remote Worker 사이의 작업 전달 프로토콜 역할까지 수행한다.

Task 상태에는 다음 정보를 연결할 수 있다.

- Task ID
- Cloud Session ID
- Branch
- Commit SHA
- Status
- Test Result
- PR

## Token과 Compute를 분리한다

Cloud Container에서 Gradle 전체 테스트가 오래 실행되고 CPU/RAM을 많이 사용하더라도, 그 실행 시간 자체가 같은 비율의 LLM Token 사용을 뜻하지 않는다.

Token은 주로 다음 시점에 사용된다.

- Agent가 코드를 읽을 때
- 계획할 때
- 명령 결과를 읽을 때
- 로그를 분석할 때
- 수정 방향을 판단할 때

따라서 권장 흐름은 다음과 같다.

```text
Agent
→ Test 실행 결정

Container
→ 10,000 tests 실행

Result Filter / Gateway
→ 실패 3건 추출

Agent
→ 실패 3건만 분석
```

## Runner와 Cloud Agent의 관계

모든 Build/Test 실행에 LLM이 필요한 것은 아니다.

```text
Task
  ↓
Runner
  ├─ PASS → 종료
  └─ FAIL → Cloud Agent 호출
```

이 구조는 Agent Platform 일반론이 아니라 **Cloud Agent의 Token과 비용을 줄이는 실전 패턴**으로 다룬다.

## Cloud Environment도 팀 자산이다

Cloud Worker가 매번 JDK, Node, Gradle dependency, Playwright, Docker image를 처음부터 준비하게 하지 않는다.

Prepared Cloud Environment를 버전 관리하고 작업 특성에 맞게 선택할 수 있게 한다.

예:

```text
backend-test
frontend-e2e
migration-test
fullstack
```

환경은 다음 두 영역으로 분리한다.

```text
Prepared Environment
- Runtime
- Tools
- Dependencies
- Cache

+

Fresh State
- Source Code
- Branch
- Task
- Test Result
- Temporary Data
```

Cloud Agent의 Cold Start는 모델 응답시간과 별도로 관리한다.

- VM/Container 생성
- Repository clone
- Dependency 설치
- Build cache 생성
- Docker image pull

최적화 목표는 `Task submitted → useful work started` 시간을 줄이는 것이다.

## Result는 Evidence로 반환한다

Cloud Agent 작업은 `완료했습니다`라는 설명만으로 끝내지 않는다.

가능하면 다음 Evidence를 함께 반환한다.

- Commit / Diff
- Build Result
- Unit Test Result
- Integration Test Result
- E2E Result
- Screenshot / Video
- Log / Artifact Reference
- PR

UI 작업에서는 빠른 1차 검증을 위해 Diff보다 Demo를 먼저 볼 수도 있다.

> Cloud Agent의 결과는 Diff보다 Demo가 먼저일 수 있다.

단 코드 Review를 생략한다는 의미는 아니다.

## Cloud Task는 작되 너무 작지 않아야 한다

너무 작은 Task는 Environment start, checkout, Agent startup, Context loading overhead가 커진다.

너무 큰 Task는 Context, Token, 실패 범위, Retry와 Review 비용이 커진다.

따라서 Task 크기는 절대 시간 기준으로 고정하지 않고 프로젝트별 실제 측정으로 결정한다.

## 병렬화에는 상한이 있다

Cloud Agent 수를 늘린다고 생산성이 선형 증가하지 않는다.

증가하는 비용:

- Repository Context 중복
- Dependency setup 중복
- Merge Conflict
- Review 증가
- LLM usage 증가
- PR 관리 증가

독립적으로 검증 가능한 Task만 병렬화한다.

## 시간 효과도 분리해서 본다

Cloud Agent 평가에서는 다음을 구분한다.

- Agent Execution Time
- Developer Blocking Time

Cloud Task가 40분 걸려도 개발자가 그동안 다른 작업을 진행했다면 실제 개발자 대기시간은 훨씬 짧을 수 있다.

Cloud Agent의 가치는 실행시간 단축뿐 아니라 Developer Blocking Time과 Context Switching 감소에도 있다.

## 프로젝트와 환경 설계의 역할

Cloud Agent를 잘 활용하려면 프로젝트와 실행환경이 준비되어 있어야 한다.

다만 Repository 구조, Task Contract, Result Gateway, Prepared Environment 등은 독립 Agent Platform 이론이 아니라 Cloud Agent 효율화를 위한 수단으로 설명한다.

상세 운영 모델은 `planning/cloud-agent-remote-worker-model.md`에서 관리한다.

## 최종 독자 역량

책을 읽은 독자는 Agent Platform 전체를 설계할 필요가 없다.

대신 자신의 프로젝트에서 다음 판단과 구성을 할 수 있어야 한다.

- 왜 Cloud Agent를 쓸 것인가
- 어떤 Task를 Cloud로 보낼 것인가
- 어떤 Task를 Local에 남길 것인가
- 어떤 Environment를 준비할 것인가
- Git으로 Local/Cloud 작업을 어떻게 넘길 것인가
- 여러 Cloud Agent를 어떻게 병렬화할 것인가
- Token을 어떻게 줄일 것인가
- Cloud CPU/RAM을 어떻게 활용할 것인가
- Build/Test를 어떻게 Cloud에 분산할 것인가
- Context와 Tool Output을 어떻게 줄일 것인가
- 어떤 Evidence를 결과로 받을 것인가
- Local + Cloud Hybrid Workflow를 어떻게 구성할 것인가

최종적으로 독자가 다음 문장을 실제 프로젝트에 적용할 수 있어야 한다.

> 이 작업은 Local에서 하고, 이 작업은 Cloud Agent에게 보내자.

## 범위 판단 기준

새로운 주제를 추가하기 전에 다음 질문을 먼저 사용한다.

> 이 내용이 개발자가 Cloud Agent를 더 잘 사용하는 데 직접적인 도움이 되는가?

- YES: 본문에 포함
- 간접적: Tip / Advanced Topic / 마지막 전망 장에서 짧게 소개
- NO: 후속 주제 후보로 이동
