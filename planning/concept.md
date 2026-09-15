# Concept

## 중심 주제

이 책의 중심 주제는 **Cloud Agent를 실제 개발팀에서 어떻게 활용할 것인가**이다.

`Agent Ready Software Engineering`은 책 전체를 지지하는 설계 철학으로 남기지만, 독자가 따라가는 주된 질문은 Cloud Agent 활용이다.

## 이 책이 답할 핵심 질문

> 클라우드 에이전트를 왜 사용하고, 로컬 에이전트와 어떻게 조합하며, 어떤 작업을 맡기고, 토큰과 클라우드 컴퓨팅 자원을 어떻게 효율적으로 활용할 것인가?

이 질문을 다음 실전 판단으로 나눈다.

- 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?
- Cloud Agent가 직접 추론해야 하는가, 일반 Runner로 충분한가?
- 여러 Cloud Worker로 병렬화할 수 있는가?
- Cloud Agent가 읽어야 하는 Context를 어디까지 줄일 수 있는가?
- Build/Test/E2E 같은 CPU/RAM 작업을 어떻게 Cloud에 분산할 것인가?
- 실패한 경우에만 Agent를 호출하도록 만들 수 있는가?
- Cloud와 Local 결과를 어떤 흐름으로 통합할 것인가?

## 핵심 정의

Cloud Agent를 단순한 원격 AI 개발자로 설명하지 않는다.

이 책에서는 Cloud Agent를 다음 요소가 결합된 **Remote Worker**로 본다.

```text
Cloud Agent
= LLM
+ 독립 실행환경
+ Repository
+ CPU / RAM / Disk
+ Tools
```

따라서 Cloud Agent의 가치는 추론 능력뿐 아니라 다음에도 있다.

- 개발자 PC와 분리된 실행환경
- 장시간 작업 위임
- 여러 독립 작업의 병렬 처리
- 별도 Branch/Container에서의 격리
- Build/Test/E2E/Docker 작업의 분산 실행

## 핵심 메시지

책 전체에서 다음 메시지를 유지한다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Cloud Agent에게 Repository 전체를 반복해서 이해시키지 않는다.

> 작은 Task와 작은 Context를 전달한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

## Local과 Cloud의 역할

### Local Agent에 적합한 작업

- Architecture 설계
- 복잡한 디버깅
- 개발자의 빈번한 개입이 필요한 작업
- 내부망/VPN/사내 DB/HSM 접근
- 여러 모듈을 동시에 이해해야 하는 작업
- 빠르게 질문과 수정이 반복되는 작업
- 최종 통합과 리뷰

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

이 분류는 절대 규칙이 아니라 Task 특성에 따른 기본 판단 기준이다.

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

## 프로젝트와 환경 설계의 역할

Cloud Agent를 잘 활용하려면 프로젝트도 준비되어 있어야 한다.

다만 이 책에서 Repository 구조, Agent Contract, 실행 인터페이스, Harness는 독립 주제가 아니라 Cloud Agent 활용을 돕는 수단으로 설명한다.

예:

- 명확한 build/test 명령
- 작은 Task 범위
- Progressive Context
- Result Gateway
- Prebuilt Environment
- Dependency Cache
- Snapshot/Warm Environment
- Branch/Worktree 격리
- Retry/Token/Cost Budget

## 최종 독자 역량

책을 읽은 독자는 Agent Platform 전체를 설계할 필요가 없다.

대신 자신의 프로젝트에서 다음 판단과 구성을 할 수 있어야 한다.

- 왜 Cloud Agent를 쓸 것인가
- 어떤 Task를 Cloud로 보낼 것인가
- 어떤 Task를 Local에 남길 것인가
- Cloud Agent 여러 개를 어떻게 병렬화할 것인가
- Token을 어떻게 줄일 것인가
- Cloud CPU/RAM을 어떻게 활용할 것인가
- Build/Test를 어떻게 Cloud에 분산할 것인가
- Context와 Tool Output을 어떻게 줄일 것인가
- Local + Cloud Hybrid Workflow를 어떻게 구성할 것인가

최종적으로 독자가 다음 문장을 실제 프로젝트에 적용할 수 있어야 한다.

> 이 작업은 Local에서 하고, 이 작업은 Cloud Agent에게 보내자.

## 범위 판단 기준

새로운 주제를 추가하기 전에 다음 질문을 먼저 사용한다.

> 이 내용이 Cloud Agent를 더 잘 사용하는 방법과 직접 관련이 있는가?

- YES: 본문에 포함
- 간접적: Tip / Advanced Topic / 마지막 전망 장에서 짧게 소개
- NO: 후속 주제 후보로 이동
