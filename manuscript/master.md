# 클라우드 코딩 에이전트 실전

## Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계

더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법


# 서문

Coding Agent가 코드를 작성하고, 테스트를 실행하고, Pull Request를 만들 수 있게 되면서 개발자의 질문도 달라지고 있다.

처음에는 어떤 모델이 더 코드를 잘 쓰는지가 중요했다. 실제 프로젝트에 Agent를 넣기 시작하면 다른 문제가 더 크게 보인다.

```text
이 작업을 내 개발환경에서 계속할 것인가?
아니면 Cloud에 넘길 것인가?
```

Cloud Agent를 단순히 `클라우드에서 실행되는 AI`로 보면 이 질문에 답하기 어렵다.

이 책에서는 Cloud Agent를 다음과 같이 본다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

즉 Cloud Agent는 대화만 하는 모델이 아니라 Repository와 실행환경을 받아 실제 개발 작업을 수행할 수 있는 Remote Development Worker다.

이 관점으로 보면 Cloud Agent의 장점도 달라진다.

핵심 가치는 단순히 더 많은 Token이나 더 큰 모델에 있지 않다. Local 개발환경과 분리된 곳에서 장시간 작업을 실행하고, 여러 독립 Task를 병렬로 처리하며, 개발자가 그 시간 동안 다른 일을 할 수 있다는 점이 중요하다.

하지만 모든 작업을 Cloud로 보내는 것이 좋은 방법은 아니다.

아키텍처를 결정해야 하는 작업, 내부망과 실제 장비가 필요한 작업, 요구사항이 계속 바뀌는 작업, 개발자의 짧은 Feedback Loop가 필요한 작업은 Local이 더 적합할 수 있다.

반대로 실행 명령과 완료 조건이 분명한 Build, Test, E2E, Docker Build 같은 작업은 Cloud Runner에 맡기기 쉽다. 재현 가능한 실패를 분석하고 제한된 범위의 코드를 수정하는 작업은 Cloud Agent 후보가 될 수 있다.

그래서 이 책의 중심 질문은 `어떤 Agent를 사용할 것인가`가 아니다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

이 질문에 답하려면 모델 선택만으로는 부족하다.

Task의 범위, Context 크기, 실행환경, Git 상태, 테스트 방법, Evidence, 내부망 의존성, 병렬화 비용, Review 비용을 함께 봐야 한다.

이 책은 그 판단을 실제 개발 Workflow 안에서 다룬다.

Java/Spring Boot 기반 `campus-platform` 예제를 사용하지만 특정 제품 사용 설명서는 아니다. GitHub Copilot cloud agent, OpenAI Codex, Claude 계열의 hosted agent 사례는 공통 실행 패턴과 설계 원칙을 확인하기 위한 근거로만 사용한다.

제품 이름과 기능은 바뀔 수 있다. 하지만 다음 문제는 계속 남는다.

```text
작업을 어떻게 나눌 것인가?
어디서 실행할 것인가?
어떤 상태를 넘길 것인가?
무엇으로 검증할 것인가?
어떤 결과를 받아야 하는가?
```

이 책에서는 다음 방향을 일관되게 유지한다.

- Runner가 할 수 있으면 Runner에게 맡긴다.
- Agent는 판단과 수정이 필요한 구간에만 사용한다.
- 작은 Task와 작은 Context를 전달한다.
- 원본 결과는 Artifact로 보존하고 필요한 정보만 Agent에게 보여준다.
- Git을 Local과 Cloud 사이의 Handoff Boundary로 사용한다.
- 병렬화의 대상은 Agent가 아니라 독립 Task다.
- Cloud 이점이 사라지면 Local로 돌아온다.

Cloud Agent는 Local 개발을 없애지 않는다. CI와 Runner도 없애지 않는다.

오히려 각 역할을 더 분명하게 나누게 만든다.

이 책을 다 읽은 뒤에는 새로운 Cloud Agent 제품을 볼 때 기능 목록부터 확인하기보다 자신의 작업을 먼저 보고 다음을 판단할 수 있기를 바란다.

```text
이 설계는 Local에서 하자.
이 검증은 Cloud Runner로 보내자.
이 실패는 Cloud Agent에게 맡기자.
이 마지막 검증은 내부망에서 하자.
```

그 판단을 반복 가능한 개발 Workflow로 만드는 것이 이 책의 목적이다.


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


# 이 책을 읽는 방법

이 책은 1장부터 18장까지 순서대로 읽을 수 있도록 구성했다.

하지만 Cloud Agent를 이미 사용하고 있거나 특정 문제를 해결하려는 독자는 필요한 경로부터 읽어도 된다.

# 처음 읽는다면

처음 Cloud Agent Workflow를 설계한다면 순서대로 읽는 것을 권장한다.

```text
Part I
Cloud Agent를 이해한다
        ↓
Part II
어떤 Task를 Cloud로 보낼 것인가
        ↓
Part III
Cloud 실행환경과 검증을 설계한다
        ↓
Part IV
Local과 Cloud를 연결한다
        ↓
Part V
실제 프로젝트에 적용한다
        ↓
Part VI
Cloud의 한계와 다음 단계를 정한다
```

앞 장에서 만든 개념을 뒤 장에서 다시 정의하지 않고 적용하는 구조이기 때문이다.

# Cloud 도입 여부를 먼저 판단하고 싶다면

다음 순서로 읽는다.

```text
1장
Cloud Agent 정의
↓
2장
Local / Cloud 차이
↓
4장
Cloud 실행의 가치
↓
5장
Task Routing
↓
6장
실제 개발 작업 분류
↓
17장
Cloud를 쓰지 말아야 할 조건
```

이 경로를 읽으면 특정 제품을 설치하기 전에 어떤 Task가 Cloud에 적합한지 먼저 판단할 수 있다.

# Token과 비용을 줄이는 것이 관심사라면

다음 장을 연결해서 읽는다.

```text
3장
Compute와 LLM 자원 분리
↓
7장
작은 Task / 작은 Context
↓
8장
작은 Output / Evidence
↓
9장
Startup Cost
↓
10장
Runner-first
↓
12장
병렬화와 중복 Context 비용
```

핵심은 `Agent Prompt를 짧게 쓰는 방법`만 찾는 것이 아니다.

```text
Compute에는 실행을 맡기고
LLM에는 필요한 판단만 맡긴다.
```

# CI/CD와 자동화가 관심사라면

다음 경로가 빠르다.

```text
10장
Runner-first / Agent-on-failure
↓
11장
작업 격리
↓
13장
Handoff
↓
14장
Event-driven Task
↓
15장
통합 운영 모델
↓
18장
Harness / Orchestration
```

CI Failure, PR Review, Nightly Test 같은 Event가 어떻게 Task가 되고, 어떤 경우에만 Agent를 호출할지 연결해서 볼 수 있다.

# 실제 적용 예를 먼저 보고 싶다면

15~16장을 먼저 읽어도 된다.

```text
15장
campus-platform 운영 모델
↓
16장
하나의 기능 End-to-End Timeline
```

모르는 개념이 나오면 다음처럼 앞 장으로 돌아간다.

```text
Task Routing → 5장
Task Contract → 7장
Evidence → 8장
Prepared Environment → 9장
Runner-first → 10장
Isolation → 11장
Parallel Cost → 12장
Handoff → 13장
```

# 내부망이 있는 프로젝트라면

다음 장을 함께 읽는다.

```text
2장
Internal Network가 Local / Cloud 선택에 미치는 영향

5장
Hard Constraint 기반 Routing

13장
Cloud 검증과 Local Internal Validation 분리

15장
Tibero / HSM / Internal API를 포함한 운영 모델

17장
Local Fallback과 재Routing
```

Cloud에서 내부망을 억지로 복제하는 방법보다 Cloud에서 끝낼 조건과 Local에서 확인할 조건을 나누는 데 초점을 둔다.

# 코드블록과 수치 읽는 방법

본문에는 Workflow를 설명하기 위한 코드블록과 숫자가 자주 나온다.

예:

```text
Test 10,000개
Build 20분
Retry 2회
PR 20개 / day
```

이 값은 별도 언급이 없는 한 제품의 성능 기준이나 권장값이 아니다.

설명용 수치는 구조와 판단 방법을 보여주기 위한 예다. 실제 프로젝트에서는 Build Time, Cold Start, Review Capacity, Retry Rate를 직접 측정해 기준을 잡는다.

# 제품 이름을 읽는 방법

GitHub, OpenAI, Anthropic 등의 제품 사례가 나오더라도 이 책은 특정 제품을 표준으로 두지 않는다.

제품 사례에서는 다음만 가져온다.

```text
Repository를 받을 수 있는가?
독립 실행환경이 있는가?
명령을 실행할 수 있는가?
코드를 변경할 수 있는가?
검증 결과를 반환할 수 있는가?
```

가격, CPU / RAM, 동시 Session 수처럼 변경 가능성이 높은 정보는 본문의 핵심 원칙과 분리한다.

# 반복해서 등장하는 AUTH-142 예제

여러 장에서 `AUTH-142 / expired token` 예제가 이어진다.

각 장에서 새로운 문제를 만드는 대신 같은 Task를 다음 관점으로 계속 본다.

```text
7장  Task Contract
8장  Evidence
10장 Runner 재검증
11장 Git / SHA 추적
13장 Handoff
15장 운영 모델
16장 End-to-End Timeline
```

따라서 같은 예제가 다시 나와도 처음부터 다시 설명하는 것으로 읽기보다 **하나의 Task가 Workflow의 다음 단계로 이동하는 과정**으로 보면 된다.

# 마지막에는 이 질문으로 돌아온다

책을 읽는 동안 다음 질문을 계속 기준으로 삼는다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

그리고 Cloud로 보낸다면 한 단계 더 묻는다.

```text
Cloud Runner인가?
Cloud Agent인가?
어떤 Evidence를 받아야 하는가?
언제 Local로 돌아와야 하는가?
```

이 네 질문이 책 전체를 연결하는 읽기 기준이다.


# 목차

## 서문

## 이 책이 다루는 문제와 독자

## 이 책을 읽는 방법

---

# Part I. Cloud Agent를 이해한다

1장. Coding Agent에서 Cloud Worker로

2장. Local Agent와 Cloud Agent

3장. Cloud Session, Container, Compute와 Token

4장. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

---

# Part II. 어떤 Task를 Cloud로 보낼 것인가

5장. Task Routing: Local인가 Cloud인가

6장. Cloud에 보내기 좋은 개발 작업

7장. Cloud Agent Task Contract: 작은 Task와 작은 Context

8장. Tool Output을 줄이고 Evidence를 남기기

---

# Part III. Cloud 실행환경과 검증을 설계한다

9장. Prepared Cloud Environment, Cache, Snapshot

10장. Cloud Agent를 Test Runner처럼 사용하기

11장. Git, Branch, Worktree, Container로 작업 격리하기

12장. 병렬 Worker와 중복 Context 비용

---

# Part IV. Local과 Cloud를 연결한다

13장. Local → Cloud → Local Handoff

14장. Task Queue와 Event-driven Cloud Agent

---

# Part V. 실제 프로젝트에 적용한다

15장. campus-platform Cloud Agent Workflow 설계

16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

---

# Part VI. Cloud의 한계와 다음 단계를 정한다

17장. Cloud가 항상 정답은 아니다

18장. 다음 단계: Harness와 Orchestration

---

# 부록 A. Task Contract / Evidence / Handoff 템플릿

# 용어집

# 참고자료


# Part I. Cloud Agent를 이해한다

Coding Agent가 Local에서 파일을 읽고 수정하는 것만으로는 Cloud Agent를 설명하기 어렵다.

Cloud Agent는 Repository와 독립 실행환경을 함께 사용한다. Build와 Test를 실행하고, 장시간 작업을 비동기적으로 처리하며, 여러 Task를 서로 다른 실행환경에 나눠 맡길 수 있다.

이 Part에서는 먼저 Cloud Agent를 **Remote Development Worker**로 정의한다. 이어서 Local과 Cloud의 차이를 실행 위치와 접근 범위로 나누고, LLM이 판단하는 자원과 CPU / RAM / Disk가 실행하는 자원을 구분한다.

마지막으로 독립 실행환경이 왜 장시간 작업과 병렬 실행에 의미가 있는지 살펴본다.

이 Part를 읽은 뒤에는 다음 질문을 구분할 수 있어야 한다.

```text
Cloud Agent는 Local Agent와 무엇이 다른가?
Compute와 LLM Token은 어떻게 다른가?
Cloud가 개발자의 대기시간을 어떻게 줄일 수 있는가?
어떤 병렬성이 실제 개발에 도움이 되는가?
```

다음 Part부터는 이 실행환경에 **어떤 Task를 보낼 것인지** 결정한다.


# 1장. Coding Agent에서 Cloud Worker로

소프트웨어 개발에서 AI를 사용할 때 먼저 구분해야 할 것은 `대화하는 AI`와 `작업하는 Agent`다.

대화형 LLM은 질문을 받고 답을 만든다. 코드 조각을 제안하거나 오류 메시지를 해석할 수 있다. Coding Agent는 여기서 한 단계 더 나아가 Repository를 읽고, 파일을 수정하고, 명령을 실행하며, Build와 Test 결과를 바탕으로 다시 작업한다.

Cloud Agent는 Coding Agent에 독립된 원격 실행환경이 결합된 형태로 볼 수 있다.

이 책에서는 Cloud Agent를 다음과 같이 정의한다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

확장하면 다음과 같다.

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

이 정의가 책 전체의 출발점이다.

---

## 1. Chat LLM과 Coding Agent는 다르다

개발자가 일반적인 LLM에 다음과 같이 질문한다고 하자.

```text
이 Java 코드에서 NullPointerException이 발생할 가능성이 있는지 확인해줘.
```

LLM은 전달받은 코드를 읽고 가능성을 설명할 수 있다.

하지만 실제 프로젝트의 문제는 코드 한 조각으로 끝나지 않는다.

```text
UserService.java
UserMapper.java
UserMapper.xml
UserServiceTest.java
application-test.yml
DB migration
```

오류를 재현하려면 Test를 실행해야 하고, Mapper XML이나 설정 파일까지 확인해야 할 수도 있다.

Coding Agent는 다음 흐름에 참여한다.

```text
Repository 탐색
→ 관련 파일 확인
→ 코드 수정
→ Build / Test 실행
→ 실패 확인
→ 재수정
→ 검증 결과 반환
```

차이는 더 긴 답변을 만드는 데 있지 않다.

Agent가 **Repository와 Tool을 사용해 실제 작업 상태를 변경한다**는 점이 중요하다.

예를 들어 다음 명령을 Agent가 직접 실행할 수 있다고 하자.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

Test가 실패하면 Agent는 실패 내용을 확인하고 관련 파일을 수정한 뒤 다시 실행할 수 있다.

이 순간 AI는 코드 설명 도구를 넘어 개발 Workflow의 작업 주체가 된다.

---

## 2. Cloud Agent에는 실행할 컴퓨터가 필요하다

Coding Agent가 Repository를 수정하고 검증하려면 코드를 실행할 장소가 필요하다.

Java 프로젝트라면 다음 자원이 필요할 수 있다.

```text
JDK
Gradle 또는 Maven
Source Code
Dependency
CPU
RAM
Disk
Shell
Git
Test Runtime
```

Frontend까지 포함되면 Node와 Browser가 필요할 수 있고, Docker Build나 Testcontainers를 사용한다면 Container Runtime도 필요하다.

따라서 Cloud Agent를 LLM 하나로만 보면 실제 동작을 설명하기 어렵다.

```text
Developer
   ↓
Task
   ↓
Cloud Worker
├─ LLM
├─ Repository
├─ CPU / RAM / Disk
├─ Development Tools
└─ Build / Test Runtime
   ↓
Commit / Test Result / Artifact / PR
```

제품마다 구현 방식과 제약은 다르지만, 이 책에서 보는 공통 구조는 같다.

```text
LLM
+
Repository
+
Execution Environment
+
Tools
```

이 요소가 결합되어야 Repository 안에서 실제 작업을 수행할 수 있다.

---

## 3. Cloud의 가치는 모델 성능만으로 설명되지 않는다

Cloud Agent를 처음 접하면 모델 성능과 Token 사용량에 관심이 집중되기 쉽다.

하지만 Cloud 실행환경이 주는 가치도 따로 봐야 한다.

개발자의 Local 환경에서 다음 프로그램을 동시에 사용한다고 하자.

```text
IDE
Local Agent
Docker
Database
Browser
Emulator
```

여기에 전체 Gradle Test, Testcontainers, Docker Build까지 실행하면 Local CPU와 RAM을 오래 점유할 수 있다.

일부 검증을 Cloud Worker로 분리하면 구조가 달라진다.

```text
Developer PC
├─ IDE
├─ Local Agent
└─ 현재 기능 개발

Cloud Worker
├─ Repository
├─ Gradle
├─ Docker
├─ Testcontainers
└─ Test 실행
```

서로 독립적인 작업이라면 여러 실행환경에서 동시에 처리할 수도 있다.

```text
             Git SHA
                |
      +---------+---------+
      |         |         |
   Unit      E2E      Docker Build
      |         |         |
      +---------+---------+
                |
             Evidence
```

따라서 이 책에서는 다음 관점을 유지한다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

LLM이 얼마나 잘 판단하는지와 어떤 환경에서 무엇을 실행할 수 있는지는 서로 다른 문제다. 3장에서 이 차이를 Compute와 Token 관점으로 분리한다.

---

## 4. Remote Developer보다 Remote Worker로 보는 편이 낫다

Cloud Agent를 `인터넷에 있는 AI 개발자`라고 표현하면 이해하기는 쉽다. 그러나 실제 Workflow를 설계할 때는 `Remote Worker` 관점이 더 유용하다.

개발자 한 명을 추가했다고 생각하면 다음처럼 넓은 요청을 만들기 쉽다.

```text
Repository 전체를 살펴보고 문제가 있으면 알아서 고쳐줘.
```

이 요청에는 작업 범위와 완료 조건이 없다.

반대로 Remote Worker에게 전달할 작업은 기준점과 검증 방법을 가질 수 있다.

```text
Task
AuthService expired token 처리 수정

Base SHA
abc123

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
HTTP 401
```

결과도 대화가 아니라 작업 결과로 받는다.

```text
Commit
abc789

Test
AuthServiceTest.expiredToken PASS

Changed Files
3
```

세부 Task Contract는 7장에서 다룬다. 1장에서는 한 가지만 기억하면 된다.

> Cloud Agent에게 넘기는 것은 Prompt가 아니라 Task다.

---

## 5. 입력과 출력 모두 작업 상태를 가진다

Cloud Task의 입력은 자연어 요청 하나로 끝나지 않는다.

보통 다음과 같은 작업 상태가 함께 필요하다.

```text
Task
Repository
Base SHA
Environment
Context
Validation
Expected Result
```

출력도 자연어 완료 선언만으로는 부족하다.

```text
Commit
Changed Files
Build / Test Result
Artifact
PR
```

예를 들어 다음 응답만으로는 완료 여부를 검증하기 어렵다.

```text
수정했습니다. 테스트도 문제없습니다.
```

반면 실행 결과가 있으면 확인할 수 있다.

```text
Commit: abc789
Test: AuthServiceTest.expiredToken PASS
Artifact: junit.xml
```

어떤 Evidence를 남기고 어떻게 큰 Tool Output을 줄일지는 8장에서 다룬다.

---

## 6. Git은 Local과 Cloud 사이의 기준점을 만든다

Local Agent는 개발자의 현재 Working Directory를 직접 사용할 수 있다. Cloud Agent는 별도 작업공간에서 시작하는 경우가 많다.

따라서 Local과 Cloud 사이에는 전달 가능한 기준점이 필요하다.

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Work / Test
→ Commit / Push
→ PR
```

여기서 Git은 코드 이력 관리뿐 아니라 Local과 Remote Worker 사이에서 작업 상태를 넘기는 경계가 된다.

```text
Task ID: task-142
Base SHA: abc123
Result SHA: def456
PR: #142
```

Branch, Worktree, Container를 이용한 구체적인 격리는 11장에서 다룬다. Local에서 Cloud로 넘기고 다시 결과를 회수하는 전체 Handoff는 13장에서 연결한다.

---

## 7. Cloud Agent가 모든 개발을 맡는 것은 아니다

Cloud Agent가 독립 실행환경을 가진다고 해서 모든 Task를 Cloud로 보내야 하는 것은 아니다.

다음 작업은 사람의 중간 판단이 많이 필요할 수 있다.

```text
새 인증 Architecture를 어떻게 설계할 것인가?
```

요구사항이 계속 바뀌고 여러 모듈을 함께 검토해야 한다면 Local에서 Developer와 Agent가 짧은 Feedback Loop를 유지하는 편이 나을 수 있다.

반면 다음 작업은 독립적인 Cloud Task로 만들기 쉽다.

```text
AuthServiceTest.expiredToken 실패 수정
```

재현 방법, 관련 범위, 완료 조건이 명확하기 때문이다.

즉 Cloud Agent는 Local Agent를 대체하는 도구가 아니다.

2장에서는 같은 Task를 Local과 Cloud 중 어디에서 실행할지 비교한다.

---

## 8. campus-platform을 Remote Worker 관점으로 보기

이 책에서는 Java/Spring Boot 기반 `campus-platform`을 반복 예제로 사용한다.

축약 구조는 다음과 같다.

```text
campus-platform/
├─ auth/
├─ student/
├─ attendance/
├─ notification/
├─ integration/
├─ common/
├─ admin-web/
└─ scripts/
```

출결 인증 변경을 수행한 뒤 여러 검증이 필요하다고 하자.

```text
Local Developer / Agent
→ 요구사항 확인
→ 변경 기준점 확정
→ Commit / Push
        |
        +-- Cloud Worker #1: Unit Test
        +-- Cloud Worker #2: Integration Test
        +-- Cloud Worker #3: Docker Build
        +-- Cloud Worker #4: Web E2E
```

각 Worker는 정해진 Git 상태를 기준으로 실행하고 검증 결과를 반환한다.

여기서 중요한 것은 Worker 개수가 아니다.

`어떤 Task를 독립적으로 실행할 수 있는가`, `결과를 무엇으로 검증할 것인가`가 먼저다.

---

## 9. 이 책에서 사용할 Cloud Agent 모델

책 전체의 기본 구조를 정리하면 다음과 같다.

```text
Developer / Local Agent
        ↓
      Task
        ↓
       Git
        ↓
Cloud Execution Environment
        ↓
Work / Build / Test
        ↓
Evidence / Commit / PR
        ↓
Developer / Local Agent
        ↓
Review / Integration
```

후속 장에서는 이 구조를 나눠서 다룬다.

- 2장: Local Agent와 Cloud Agent
- 3장: Compute와 Token
- 5장: Task Routing
- 7장: Task Contract
- 8장: Evidence와 Result Gateway
- 9장: Prepared Environment
- 10장: Cloud Runner와 Agent
- 11~13장: 격리와 Handoff

1장에서 기억할 기준은 단순하다.

> Cloud Agent는 Repository와 독립 실행환경을 가진 Remote Development Worker다.

> Cloud Agent는 Local Agent를 대체하지 않는다.

> Cloud Agent에게 넘기는 것은 Task이며, 결과는 검증 가능한 작업 상태여야 한다.

다음 장에서는 이 Remote Worker와 Local Agent를 비교해 실행 위치를 선택하는 기준을 만든다.

---

## 참고자료

제품별 기능은 변경될 수 있으므로 본문에서는 공통 실행 모델만 사용한다. 제품 사례의 세부 사실은 `research/chapter-01-cloud-worker-official-sources.md`에서 기준일과 출처를 관리한다.

- GitHub Docs, `About GitHub Copilot cloud agent`
- GitHub Docs, `Configure the development environment`
- OpenAI, Codex 관련 공식 자료


# 2장. Local Agent와 Cloud Agent

Cloud Agent를 도입할 때 먼저 결정해야 하는 것은 어떤 제품을 쓸지가 아니다.

더 중요한 질문은 다음이다.

> 이 Task를 어디에서 실행할 것인가?

같은 계열의 Coding Agent라도 개발자 PC에서 실행되는 경우와 별도 Cloud 환경에서 실행되는 경우에는 사용할 수 있는 상태와 자원이 다르다.

Local은 현재 Workspace, 미커밋 파일, VPN, 내부 DB 같은 자원에 가깝다. Cloud는 개발자 PC와 분리된 환경에서 장시간 작업을 실행하거나 여러 독립 Task를 동시에 처리하기 쉽다.

따라서 Local Agent와 Cloud Agent를 경쟁 관계로 볼 필요는 없다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

이 장에서는 두 실행 위치의 차이와 Hybrid 사용의 기본 형태만 정리한다. 실제 Routing Framework는 5장에서 만든다.

---

## 1. 차이는 모델보다 실행 위치에서 시작한다

Local Agent는 개발자의 현재 환경을 직접 사용할 수 있다.

```text
Developer PC
├─ Repository
├─ IDE
├─ Local Agent
├─ Git Working Tree
├─ Docker
├─ Database
├─ VPN
└─ Internal Tools
```

현재 수정 중인 파일도 바로 읽을 수 있다.

```text
modified: AuthService.java
untracked: debug-config.yml
```

반면 Cloud Agent는 별도 실행환경에서 Repository와 Task를 기준으로 시작한다.

```text
Cloud Environment
├─ Repository Checkout
├─ Workspace
├─ CPU / RAM / Disk
├─ Development Tools
└─ Agent
```

다음 두 작업을 비교해 보자.

### 현재 Local 상태가 필요한 작업

```text
미커밋 코드와 Local DB 상태를 같이 보면서
간헐적으로 발생하는 오류를 조사한다.
```

### 독립적으로 전달 가능한 작업

```text
Base SHA: abc123
Task: AuthService expired token 수정
Validation: ./gradlew test --tests AuthServiceTest.expiredToken
```

두 작업의 차이는 모델 성능보다 **현재 상태를 어디까지 전달할 수 있는가**에 가깝다.

---

## 2. Local이 유리한 작업

Local의 가장 큰 장점은 개발자와 현재 작업 상태에 가깝다는 점이다.

다음과 같은 상황을 생각해 보자.

```text
Developer:
인증 모듈을 OAuth 기준으로 바꿀지 기존 JWT 구조를 유지할지 비교해보자.

Agent:
현재 auth와 연동 구조를 보면 두 선택지가 있습니다.

Developer:
DB Schema는 변경하지 않는 방향으로 다시 보자.
```

이 작업은 중간 판단이 계속 바뀐다.

이런 경우에는 비동기 위임보다 짧은 Feedback Loop가 중요하다.

Local에 적합한 대표 조건은 다음과 같다.

- 요구사항이 아직 불명확함
- Architecture 판단 비중이 큼
- 여러 모듈을 넓게 탐색해야 함
- 개발자와 질문/수정이 반복됨
- 현재 미커밋 상태를 사용해야 함
- 로컬 파일이나 디바이스에 의존함
- VPN이나 내부 시스템 접근이 필요함
- 최종 Integration과 Review가 필요함

Local은 단순히 `작은 작업을 하는 곳`이 아니다. 사람의 개입과 현재 환경 의존성이 큰 작업에 가깝다.

---

## 3. Cloud가 유리한 작업

Cloud Agent는 Task가 명확하고 독립적으로 검증할 수 있을수록 사용하기 쉽다.

```text
Task
AuthService expired token 처리 수정

Failure
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

이 Task는 다음 특성을 가진다.

```text
Scope가 작음
완료 조건이 있음
독립 Test 가능
Git 기준점을 만들 수 있음
중간 질문이 적음
```

Cloud가 유리한 대표 조건은 다음과 같다.

- Scope와 완료 조건이 명확함
- 독립적으로 검증 가능함
- 다른 Task와 파일 충돌이 적음
- Git으로 상태를 전달할 수 있음
- 실행시간이 길거나 Local 자원을 많이 사용함
- 사람의 중간 개입이 적음
- 여러 독립 작업으로 분리할 수 있음

Cloud의 장점은 단순히 원격에 있다는 데 있지 않다.

독립 실행환경을 이용해 Local 자원을 비우고, 장시간 작업을 위임하며, 서로 독립적인 작업을 동시에 실행할 수 있다는 데 있다.

---

## 4. 하나의 기능도 Local과 Cloud 사이를 이동한다

Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다.

출결 API 인증 변경을 예로 들어보자.

처음에는 Local에서 요구사항과 영향 범위를 확인한다.

```text
Local
→ 요구사항 분석
→ Architecture 확인
→ 영향 범위 확인
```

변경 기준이 정해지면 독립 검증을 Cloud로 보낼 수 있다.

```text
Cloud
→ Unit Test
→ Integration Test
→ Docker Build
→ Web E2E
```

마지막에 실제 내부 DB나 장비 검증이 필요하면 Local로 돌아온다.

```text
Local
→ Tibero 검증
→ HSM 확인
→ 최종 Review
→ Merge
```

즉 기본 형태는 다음과 같다.

```text
Local
  ↓
Cloud
  ↓
Local
```

13장에서는 이 흐름을 Git과 Evidence를 이용한 Handoff로 구체화한다.

---

## 5. Internal Network는 강한 제약이 된다

기업 환경에서는 다음 자원이 내부망에만 있을 수 있다.

```text
Internal Git
Nexus
Jenkins
Tibero / Oracle
HSM
Internal API
VPN-only Server
```

Cloud 환경이 이 자원에 접근할 수 없다면 Task 전체를 Local에 남기거나, Cloud에서 가능한 부분만 분리해야 한다.

```text
Cloud
→ Unit Test
→ Testcontainers 기반 Integration
→ Docker Build

Local
→ Tibero 실제 검증
→ HSM 최종 검증
```

이런 형태가 Hybrid다.

중요한 것은 모든 내부 환경을 Cloud에 복제하는 것이 아니다.

**재현 가능한 검증과 내부 환경에서만 가능한 검증을 분리하는 것**만으로도 Cloud를 활용할 수 있다.

---

## 6. Human Steering이 많으면 비동기 위임의 이점이 줄어든다

Cloud Agent의 장점 중 하나는 Task를 맡기고 개발자가 다른 일을 할 수 있다는 점이다.

하지만 작업 중간에 사람이 계속 방향을 바꿔야 한다면 이 장점은 작아진다.

예를 들어 UI 구조를 탐색한다고 하자.

```text
A안 구현
→ 사람 확인
→ B안으로 변경
→ 다시 확인
→ 데이터 흐름도 변경
```

이 작업은 짧은 주기로 상호작용이 반복된다.

반대로 다음 Task는 중간 개입이 적다.

```text
Goal
AuthService가 Java 21에서 기존 Test를 모두 통과하도록 deprecated API를 수정한다.

Validation
./gradlew :auth:test
```

따라서 Local/Cloud 판단에서 다음 질문이 중요하다.

> 작업 중간에 사람이 얼마나 자주 개입해야 하는가?

이 책에서는 이를 `Human Steering`으로 부른다.

---

## 7. Context가 너무 크면 먼저 문제를 줄인다

Cloud Agent가 Repository를 읽을 수 있다고 해서 매 Task마다 전체 시스템을 다시 이해시키는 것이 유리한 것은 아니다.

다음 두 범위를 비교해 보자.

```text
작은 Context
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java
```

```text
큰 Context
- Auth
- Student
- Permission
- Payment
- Database Schema
- Common Exception
- External API
- Admin Web
```

전체 시스템 관계를 보며 방향을 정해야 한다면 Local에서 먼저 구조를 좁히는 편이 낫다.

```text
큰 문제
→ Local에서 분석/결정
→ 작은 Task로 분해
→ Cloud 위임
```

7장에서 Cloud Task의 Context 경계를 구체적으로 정의한다.

---

## 8. Agent 실행시간과 개발자 대기시간은 다르다

Cloud Task가 오래 걸린다고 개발자가 같은 시간 동안 멈춰 있어야 하는 것은 아니다.

다음은 설명용 예다.

```text
10:00 Cloud Task 위임
10:01 개발자는 다음 작업 시작
10:40 Cloud Task 완료
11:20 개발자가 결과 Review
```

이 경우 Cloud Task의 실행시간과 개발자의 실제 대기시간은 다르다.

```text
Agent Execution Time
= Worker가 Task를 수행한 시간

Developer Blocking Time
= 개발자가 해당 Task 때문에 실제로 멈춘 시간
```

Cloud 사용의 가치를 평가할 때는 `Agent가 몇 분 걸렸는가`만 보지 않는다.

Local CPU / RAM 점유를 얼마나 줄였는지, 개발자가 다른 작업을 계속할 수 있었는지도 함께 본다. 이 차이는 4장에서 더 자세히 다룬다.

---

## 9. Cloud가 항상 더 빠른 것은 아니다

Cloud에서는 Worker 준비, Repository Checkout, 환경 준비, 결과 회수 같은 비용이 생길 수 있다.

예를 들어 문구 한 줄을 수정하고 짧은 검증만 하면 되는 Task라면 Cloud 준비 비용이 작업 자체보다 클 수 있다.

반대로 장시간 Integration Test나 E2E처럼 Local 자원을 오래 점유하는 작업은 Cloud 분리의 이점이 커질 수 있다.

따라서 다음 두 질문은 다르다.

```text
Cloud에서 실행할 수 있는가?

Cloud로 보내는 것이 유리한가?
```

첫 번째가 가능하더라도 두 번째의 답이 `아니오`일 수 있다.

5장에서는 이 판단을 Local / Cloud / Hybrid / Runner-first Routing Framework로 정리한다.

---

## 10. campus-platform에서 실행 위치를 나누면

`campus-platform`에 다음 작업이 있다고 가정하자.

| Task | 기본 위치 | 이유 |
| --- | --- | --- |
| 신규 인증 구조 설계 | Local | 넓은 Context와 Human Steering 필요 |
| expired token 수정 | Cloud Agent 후보 | 작은 Scope, 독립 검증 가능 |
| 전체 Unit Test | Cloud Runner | 결정론적 검증, Compute 중심 |
| Web E2E | Cloud Runner | Browser / 장시간 검증 |
| Tibero 실제 Migration 검증 | Local / Hybrid | 내부 DB 의존 |
| HSM 오류 분석 | Local | 내부 장비 / 네트워크 의존 |
| Docker Build | Cloud Runner | 독립 실행 가능 |

이 표는 고정 규칙이 아니다.

프로젝트의 네트워크와 실행환경, 보안 정책, Task 크기에 따라 결과가 달라질 수 있다.

중요한 것은 제품 이름이 아니라 Task 특성을 실행 위치에 연결하는 것이다.

---

## 11. 이 장에서 기억할 질문

Local과 Cloud를 비교할 때 다음 항목을 먼저 본다.

```text
현재 Local 상태가 필요한가?
Internal Network가 필요한가?
Human Steering이 많은가?
Context가 큰가?
독립적으로 검증 가능한가?
장시간 Compute가 필요한가?
Git으로 기준점을 전달할 수 있는가?
Cloud 준비 비용보다 Task 가치가 큰가?
```

이 장에서는 판단 재료만 만들었다.

5장에서는 이 질문을 실제 Routing 순서로 정리하고, 17장에서는 반대로 Cloud Task를 중단하거나 Local로 Fallback해야 하는 조건을 다룬다.

다음 장에서는 Cloud 환경에서 사용되는 CPU, RAM, Disk와 LLM Token을 분리해 본다.

---

## 참고자료

제품별 기능은 변경될 수 있으므로 본문에서는 실행 위치에 따른 공통 차이만 사용한다. 제품 사례의 세부 사실은 `research/chapter-02-local-cloud-official-sources.md`에서 기준일과 출처를 관리한다.


# 3장. Cloud Session, Container, Compute와 Token

Cloud Agent를 실제 개발에 사용하면 비용과 성능을 설명할 때 자주 섞이는 개념이 있다.

바로 **Cloud에서 코드를 실행하는 자원**과 **LLM이 읽고 판단하는 자원**이다.

예를 들어 Cloud 환경에서 전체 Test를 30분 동안 실행했다고 하자. 이 시간 동안 CPU와 RAM이 사용되고, Testcontainers가 Container를 띄우며, Disk에는 로그와 Test 결과가 쌓일 수 있다.

그렇다고 이 30분이 그대로 30분치 LLM 추론이나 일정한 비율의 Token 사용을 의미하지는 않는다.

이 책에서는 먼저 두 자원을 분리한다.

```text
Reasoning Resource
- Input Context
- Output
- Reasoning
- Source / Context 읽기
- Tool Result 분석

Execution Resource
- CPU
- RAM
- Disk
- Process
- Container / VM
- Browser
- Build / Test Tools
```

두 자원은 연결되어 있지만 같은 것은 아니다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

---

## 1. Cloud Session은 하나의 작업 실행 단위다

제품마다 Session, Task, Workspace, Environment 같은 이름을 다르게 사용할 수 있다.

이 책에서는 제품명과 관계없이 다음 요소가 결합된 작업 단위를 `Cloud Session`으로 본다.

```text
Cloud Session
├─ Repository / Branch
├─ Workspace
├─ CPU
├─ RAM
├─ Disk
├─ Development Tools
└─ Agent / LLM Interaction
```

중요한 것은 `계정에 원격 컴퓨터 한 대가 붙어 있다`는 식으로 단순화하지 않는 것이다.

개발자가 확인해야 할 것은 다음과 같다.

```text
이 Task의 Git 기준점은 무엇인가?
어느 Workspace에서 실행되는가?
어떤 Tool이 준비되어 있는가?
다른 Task와 실행환경이 격리되어 있는가?
결과는 어떻게 반환되는가?
```

제품별 vCPU, RAM, Disk 수치는 바뀔 수 있다. 이 장에서는 특정 수치보다 **Task가 독립 실행환경을 사용한다는 구조**에 집중한다.

---

## 2. 모델이 Build와 Test를 직접 계산하는 것은 아니다

다음 요청을 생각해 보자.

```text
전체 Test를 실행하고 실패 원인을 찾아 수정해줘.
```

한 문장 안에 성격이 다른 작업이 섞여 있다.

LLM은 무엇을 실행할지 판단할 수 있다.

```text
./gradlew test
```

실제 Test는 Shell, JVM, 운영체제, CPU, RAM에서 실행된다.

```text
Gradle
→ JVM
→ JUnit
→ Application Context
→ Testcontainers
→ DB Container
```

결과가 나오면 LLM이 실패 내용을 읽고 다음 행동을 판단한다.

```text
LLM
명령 결정
   ↓
Execution Environment
명령 실행
   ↓
CPU / RAM / Disk
   ↓
Result
   ↓
LLM
결과 분석 / 다음 행동 결정
```

Agent가 `Test를 실행했다`고 표현하더라도 컴파일, Test, Browser 렌더링, Docker Build 자체는 일반 프로그램이 수행한다.

이 구분이 후반부의 Cloud Runner 설계로 이어진다.

---

## 3. Brain과 Hands는 자원 분리를 설명하기 위한 모델이다

이 책에서는 제한적으로 다음 표현을 사용한다.

```text
Brain
= LLM
= 계획 / 판단 / 분석 / 수정 방향 결정

Hands
= Execution Environment
= Shell / CPU / RAM / Disk / Build / Test / Browser
```

예를 들어 `AuthServiceTest.expiredToken` 실패를 수정한다고 하자.

Brain이 하는 일:

```text
실패 의미 확인
→ 관련 코드 판단
→ 수정 방향 결정
→ 어떤 검증을 다시 실행할지 결정
```

Hands가 하는 일:

```text
파일 읽기/쓰기
→ Gradle 실행
→ JUnit 실행
→ Testcontainers 실행
→ 결과 파일 생성
```

이 모델을 별도의 Agent Platform 아키텍처로 확장하지 않는다.

목적은 하나다.

> LLM이 판단하는 비용과 실행환경이 계산하는 비용을 분리해서 보기 위해서다.

---

## 4. Build 시간과 LLM 사용량은 다른 지표다

다음 수치는 설명용 예다. 전체 Build가 20분 걸린다고 하자.

```bash
./gradlew clean build
```

이 시간 동안 다음 작업이 실행될 수 있다.

```text
Java Compile
Test Compile
Unit Test
Integration Test
Static Analysis
Package
```

주로 사용하는 자원은 CPU, RAM, Disk I/O, Network I/O, Container Runtime이다.

LLM은 다른 순간에 사용된다.

```text
Task 읽기
Repository 구조 확인
실행 명령 선택
결과 확인
오류 분석
수정 코드 생성
```

다음 두 상황을 비교해 보자.

### A. 실행 후 결과만 확인

```text
Agent
→ Build 실행

Environment
→ 20분 Build

Agent
→ 최종 결과 확인
```

### B. 계속 상태를 읽음

```text
Agent
→ Build 실행
→ 반복 Polling
→ 로그 누적 읽기
→ 중간 상태 분석
→ 다시 Polling
```

두 경우 Build 시간은 같아도 LLM이 읽는 Tool Output과 Context의 양은 다르다.

따라서 `Wall-clock Time`과 `LLM Usage`를 같은 지표로 보면 비용을 잘못 해석할 수 있다.

---

## 5. Token은 Source뿐 아니라 Tool Output에서도 사용된다

Agent가 판단하기 위해 읽는 것은 Source Code만이 아니다.

```text
Task / Prompt
Source Code
Documentation
Git Diff
Tool Output
Build Log
Test Log
Stack Trace
Agent Response
```

이 정보가 Context가 된다.

예를 들어 Container가 Test 10,000개를 실행했다고 하자. 이 숫자는 설명용 예다.

결과가 다음과 같을 수 있다.

```text
Tests
total: 10,000
passed: 9,997
failed: 3
```

실행된 Test 수가 많다고 해서 Agent가 10,000개 Test의 모든 출력을 읽어야 하는 것은 아니다.

중요한 것은 **실행량과 Agent가 읽는 정보량을 분리하는 것**이다.

7장에서는 Agent가 처음 읽는 Source / Context를 줄이고, 8장에서는 Tool Output과 Evidence를 다룬다.

---

## 6. Test를 많이 실행하는 것과 로그를 많이 읽는 것은 다르다

다음 수치 역시 설명용 예다.

```text
JUnit Tests: 10,000
Execution Time: 25분
Log: 100MB
Failures: 3
```

비효율적인 구조는 다음과 같다.

```text
Execution Environment
→ Test 실행
→ 100MB 로그 생성
      ↓
LLM
→ 전체 로그 분석
```

Test Report가 실패 3건을 구조화해서 제공할 수 있다면 LLM에게 전체 정상 로그를 다시 읽힐 필요는 없다.

```text
Execution Environment
→ Test 실행
      ↓
Structured Test Result
→ 실패 3건 식별
      ↓
LLM
→ 필요한 실패부터 분석
```

원본 로그는 보존할 수 있다. 다만 Agent에게 처음 보여주는 결과를 작게 만든다.

이 원칙은 8장의 Result Gateway에서 구체화한다.

---

## 7. Compute-heavy / Context-light 작업을 찾는다

Cloud에 보내기 쉬운 작업 중에는 다음 특성을 가진 것이 있다.

```text
Compute는 큼
Context는 작음
완료 조건은 명확함
```

예:

```text
전체 Unit Test
Integration Test
Docker Build
Web E2E
Static Analysis
Migration Validation
```

이미 실행 명령이 정해진 검증이라면 Agent가 Repository 전체를 이해할 필요가 없을 수 있다.

```text
Base SHA
Command
Environment
Expected Exit Code
Artifact Location
```

실제 계산은 실행환경이 담당한다.

이때 중요한 질문은 다음이다.

> 이 작업은 계속 LLM의 판단이 필요한가, 아니면 대부분 Compute만 필요한가?

구체적인 Runner-first 구조는 10장에서 다룬다.

---

## 8. Parallel Compute와 Parallel Reasoning은 다르다

여러 Cloud Session을 동시에 사용한다고 하자.

다음은 Parallel Compute다.

```text
Worker A → Unit Test
Worker B → Integration Test
Worker C → Web E2E
Worker D → Docker Build
```

서로 다른 실행환경이 정해진 작업을 동시에 수행한다.

반면 다음은 Parallel Reasoning이다.

```text
Agent A → Repository 전체 분석
Agent B → Repository 전체 분석
Agent C → Repository 전체 분석
Agent D → Repository 전체 분석
```

후자는 각 Agent가 같은 Source와 문서를 반복해서 읽을 수 있다.

따라서 Session 수가 늘었다고 유효 작업량이 같은 비율로 증가하는 것은 아니다.

기본 순서는 다음과 같다.

```text
Task를 독립적으로 나눈다
      ↓
각 Task의 Context를 줄인다
      ↓
필요한 Compute 또는 Agent를 배치한다
```

병렬화의 이득과 중복 Context 비용은 12장에서 자세히 다룬다.

---

## 9. 비용은 Compute, LLM, Human으로 나눠 본다

Cloud Agent 비용을 Token 하나로만 표현하면 실제 Workflow의 병목을 놓치기 쉽다.

이 책에서는 최소한 세 종류로 나눈다.

### Compute Cost

```text
CPU
RAM
Disk
Container / VM Runtime
Network
Browser Runtime
```

### LLM Cost

```text
Input Context
Output
Reasoning
Tool Result Processing
Agent Invocation
```

### Human Cost

```text
Developer Blocking Time
Review Time
Failure Reproduction
Context Switching
Manual Environment Setup
```

예를 들어 Cloud에서 장시간 Test를 별도 실행하면 Compute 사용은 늘 수 있다. 하지만 개발자가 Local CPU / RAM 점유에서 벗어나 다음 작업을 진행할 수 있다면 Human Cost는 줄 수 있다.

반대로 작은 문구 수정 하나를 위해 Cloud Environment를 준비하고 Agent가 Repository를 탐색하고 PR까지 만든다면 전체 비용이 더 커질 수 있다.

따라서 최적화 목표는 Token 하나를 최소화하는 것이 아니다.

> 전체 개발 Workflow에서 Compute, LLM, Human 비용을 어디에 쓰는 것이 유리한지 판단하는 것이다.

---

## 10. campus-platform에서 자원을 나눠 보면

출결 기능 변경 뒤 다음 검증이 필요하다고 하자.

```text
Unit Test
Integration Test
Docker Build
Web E2E
```

같은 Git SHA를 기준으로 독립 검증이 가능하다면 다음처럼 분리할 수 있다.

```text
Git SHA: abc123
        |
        +-- Execution #1: Unit Test
        +-- Execution #2: Integration Test
        +-- Execution #3: Docker Build
        +-- Execution #4: Web E2E
```

각 실행환경은 CPU / RAM을 사용해 작업을 수행한다.

결과가 다음처럼 모였다고 하자.

```text
Unit: PASS
Integration: PASS
Docker: PASS
E2E: FAIL 2
```

이때 LLM이 필요한 지점은 E2E 실패의 원인을 판단하고 코드를 수정하는 구간일 수 있다.

즉 실행환경이 먼저 많은 작업을 하고, 판단이 필요한 순간에 LLM이 참여한다.

이것이 다음 문장의 실제 의미다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

---

## 11. 제품별 자원 수치는 원칙과 분리한다

실제 제품을 선택할 때는 다음 정보가 중요하다.

```text
vCPU
RAM
Disk
동시 Session 수
Session 유지시간
가격
Rate Limit
```

하지만 이런 값은 제품과 요금제에 따라 바뀔 수 있다.

따라서 책에서는 다음처럼 분리한다.

```text
본문
→ Compute와 Token을 분리하는 원칙
→ Task별 자원 활용 방법
→ 병렬화 판단 방법

Research
→ 기준일별 제품 사양
→ 가격 / 제한
```

독자가 제품을 바꾸더라도 본문의 판단 기준은 유지되어야 한다.

---

## 12. 이 장에서 기억할 구조

Cloud Agent 비용을 하나의 숫자로 보지 않는다.

```text
Task
  ↓
LLM
판단
  ↓
Execution Environment
CPU / RAM / Disk
  ↓
Build / Test / Browser / Docker
  ↓
Result
  ↓
필요한 결과만 LLM
  ↓
다음 판단
```

그리고 비용은 다음처럼 나눈다.

```text
Compute
+
LLM
+
Human
```

이 구조를 이해하면 다음 질문을 할 수 있다.

```text
이 작업에 LLM 판단이 계속 필요한가?
대부분 Compute만 필요한가?
Agent가 전체 로그를 읽어야 하는가?
여러 Session이 실제 독립 Compute를 수행하는가?
같은 Context를 여러 Agent가 반복해서 읽고 있지는 않은가?
```

다음 장에서는 독립 실행환경과 비동기 위임이 개발자의 실제 시간에 어떤 차이를 만드는지 살펴본다.

---

## 참고자료

제품별 자원 사양과 제한은 본문 원칙과 분리해 조사 문서에서 관리한다.

- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/infrastructure-noise.md`


# 4장. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

Cloud Agent를 사용하는 이유를 `AI가 코드를 대신 작성한다`로만 설명하면 Cloud의 장점이 잘 보이지 않는다.

Local Agent도 Repository를 읽고 코드를 수정하며 Test를 실행할 수 있다. 같은 Agent라도 Cloud에서 실행할 때 달라지는 것은 **실행 위치와 작업 방식**이다.

이 장에서는 Cloud의 가치를 네 가지로 본다.

```text
독립 실행환경
장시간 작업의 비동기 위임
독립 Task의 병렬 실행
Agent Execution Time과 Developer Blocking Time의 분리
```

핵심은 다음 두 문장이다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> Cloud Agent가 오래 실행되는 것과 개발자가 오래 기다리는 것은 같은 의미가 아니다.

3장에서 Compute와 LLM을 분리했다면, 이번 장에서는 그 Compute를 개발 Workflow에서 어떻게 활용하는지 본다.

---

## 1. 독립 실행환경이 첫 번째 가치다

Local Agent는 개발자의 현재 컴퓨터와 자원을 공유하는 경우가 많다.

```text
Developer Mac
├─ IDE
├─ Local Agent
├─ Docker
├─ Database
├─ Browser
├─ Emulator
└─ Gradle
```

여기에 전체 Test, Testcontainers, Docker Build, Browser E2E까지 동시에 실행하면 CPU, RAM, Disk I/O가 현재 개발 작업과 경쟁한다.

Cloud에서는 이 실행을 별도 Worker로 분리할 수 있다.

```text
Developer Mac
├─ IDE
├─ Local Agent
└─ 현재 기능 개발

Cloud Worker
├─ Repository
├─ CPU / RAM / Disk
├─ Gradle
├─ Docker
├─ Browser
└─ Test Runtime
```

Cloud 환경이 항상 Local보다 빠르다는 뜻은 아니다.

핵심은 **개발자의 현재 Workspace와 실행 자원을 분리할 수 있다는 것**이다.

이 분리는 다음 효과를 만든다.

- 무거운 Build / Test가 Local 작업을 덜 방해한다.
- Task별로 독립된 Runtime을 사용할 수 있다.
- 하나의 개발자 PC보다 많은 실행 작업을 동시에 수행할 수 있다.
- Task가 끝날 때까지 터미널을 계속 지켜볼 필요가 줄어든다.

---

## 2. Cloud Worker는 독립 작업 노드다

1장에서 Cloud Agent를 다음처럼 정의했다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

실행 관점에서는 다음처럼 볼 수 있다.

```text
Cloud Worker
= Task
+ Repository State
+ Runtime
+ Compute
+ Tools
+ Result
```

Cloud Worker가 가치 있으려면 Local 개발자의 현재 화면을 계속 따라가는 것이 아니라, 일정한 기준점에서 독립적으로 작업할 수 있어야 한다.

예를 들어 다음 정도가 명확하다고 하자.

```text
Base SHA
abc123

Task
AuthService expired token 수정

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
HTTP 401
```

Cloud Worker는 이 상태를 기준으로 작업할 수 있다. 그동안 개발자는 다른 Branch나 다른 기능을 진행할 수 있다.

Task 입력 형식은 7장의 Task Contract에서 구체화한다. 여기서는 한 가지 원칙만 기억하면 된다.

> Cloud의 비동기성은 Task가 현재 개발자의 Workspace와 분리될 수 있을 때 생긴다.

---

## 3. 장시간 작업은 Local에서 분리할 가치가 크다

Cloud에 보내기 좋은 대표 작업은 시간이 오래 걸리면서 중간에 사람의 판단이 거의 필요하지 않은 작업이다.

예:

```text
전체 Unit Test
Integration Test
Docker Build
Frontend E2E Regression
Static Analysis
Migration Validation
반복 Refactoring 후 전체 검증
```

예를 들어 Integration Test가 오래 걸린다고 하자.

Local에서는 다음 자원을 계속 점유할 수 있다.

```text
JVM
Docker
Testcontainers
Database Container
Disk I/O
Network
```

Cloud로 분리하면 구조가 달라진다.

```text
Developer
→ Cloud Task 위임
→ 다음 기능 진행

Cloud Worker
→ Integration Test
→ Evidence / Artifact 생성
```

여기서 `오래 걸리면 무조건 Cloud`라는 규칙을 만들지는 않는다.

장시간 Task라도 Scope가 불명확하거나 Human Steering이 계속 필요하거나 내부망에서만 재현된다면 Local이 더 적합할 수 있다.

어떤 Task를 실제로 Cloud로 보낼지는 5장에서 판단한다.

---

## 4. Agent Execution Time과 Developer Blocking Time은 다르다

Cloud Agent의 성능을 `몇 분 만에 끝냈는가`만으로 보면 비동기 작업의 장점을 놓친다.

아래 시간은 설명용 예다.

```text
10:00 Cloud Task 위임
10:01 Developer는 다음 기능 시작
10:40 Cloud Task 완료
11:20 Developer가 결과 Review
```

Cloud Task의 실행시간은 40분이다.

하지만 Developer가 40분 동안 멈춰 있었던 것은 아니다.

두 값을 분리한다.

```text
Agent Execution Time
= Worker가 Task를 수행한 시간

Developer Blocking Time
= 해당 Task 때문에 Developer가 실제로 멈춘 시간
```

Cloud Task가 Local보다 조금 늦게 끝나더라도 Developer Blocking Time은 줄어들 수 있다.

그래서 Cloud 활용을 평가할 때 다음을 함께 본다.

```text
Agent Execution Time
Queue Time
Developer Blocking Time
Review Time
Retry Time
Local Resource Occupancy
```

이 값들은 서로 같은 방향으로 움직이지 않는다.

Cloud Compute 사용량이 늘어도 Developer Blocking Time은 줄 수 있고, Agent가 많은 결과를 만들어도 Review Time은 늘 수 있다.

---

## 5. Local Resource Occupancy도 비용이다

개발자의 컴퓨터가 충분히 빠르더라도 여러 작업이 동시에 실행되면 현재 개발 흐름에 영향을 줄 수 있다.

```text
IDE
Local Agent
Docker
PostgreSQL
Redis
Browser
Emulator
Gradle Test
Playwright
```

Cloud Worker로 무거운 검증을 분리하면 Local에서는 다음 작업에 자원을 남길 수 있다.

```text
코드 탐색
Architecture 판단
빠른 수정 반복
내부망 작업
최종 Review
```

따라서 Cloud 비용을 Token이나 Cloud Compute만으로 보지 않는다.

개발자 장비의 점유와 작업 전환 비용도 전체 Workflow 비용에 포함한다.

---

## 6. 비동기 위임에 적합한 Task는 따로 있다

Cloud Worker에 맡겨 두고 개발자가 다른 일을 하려면 Task가 일정 시간 독립적으로 진행될 수 있어야 한다.

비동기 위임에 유리한 신호:

```text
Scope가 명확함
완료 조건이 있음
중간 질문이 적음
독립 검증 가능
Git / Artifact로 결과 회수 가능
```

예:

```text
특정 Service Test 추가
Module 전체 Test
Docker Build
E2E Regression
Dependency Update 검증
```

반대로 다음 작업은 개발자의 개입이 계속 필요할 수 있다.

```text
새 인증 Architecture 방향 탐색
재현 조건이 없는 Production 장애 조사
여러 UI 안을 빠르게 비교하는 작업
```

이런 Task는 Local에서 짧은 Feedback Loop를 유지하는 편이 자연스럽다.

Cloud를 쓸 수 있는가보다 **사람이 없어도 이 Task가 의미 있게 전진할 수 있는가**를 먼저 본다.

---

## 7. 병렬화의 대상은 Agent가 아니라 독립 Task다

Cloud에서는 여러 Worker를 동시에 실행할 수 있다. 하지만 `Agent를 몇 개 띄울 수 있는가`가 첫 질문은 아니다.

먼저 묻는다.

> 서로 독립적으로 실행하고 검증할 수 있는 Task가 몇 개인가?

하나의 Git SHA를 다음처럼 검증한다고 하자.

```text
                  abc123
                     |
        +------------+------------+------------+
        |            |            |            |
      Unit       Integration     E2E        Docker
        |            |            |            |
        +------------+------------+------------+
                     |
                  Evidence
```

이 경우 네 개의 Agent가 필요한 것은 아니다.

대부분은 네 개의 Cloud Runner가 각각 독립적인 Compute 작업을 실행하면 된다. 실패 분석이나 코드 수정이 필요할 때만 Agent가 들어간다.

즉 Cloud 병렬성의 첫 사용처는 대개 **Parallel Reasoning이 아니라 Parallel Compute**다.

여러 Worker를 실제로 얼마나 동시에 실행해야 하는지, 중복 Context와 Review / Fan-in 비용이 어디서 생기는지는 12장에서 다룬다.

---

## 8. 병렬화 전에는 독립성만 확인한다

이 장에서는 병렬화의 상세 비용 계산까지 들어가지 않는다. 다만 시작 조건 하나는 명확히 한다.

좋은 병렬 Task:

```text
Unit Test
Integration Test
E2E
Docker Build
```

같은 SHA를 읽고 각자 독립적으로 검증할 수 있다.

또 서로 다른 모듈의 제한된 변경도 병렬화 후보가 될 수 있다.

```text
Task A → attendance module
Task B → notification module
```

반대로 같은 핵심 파일이나 같은 DB Schema를 동시에 바꿔야 한다면 실제 독립 Task가 아니다.

```text
Task A → UserService.java
Task B → UserService.java
```

이런 경우의 Merge / Review 비용과 병렬화 상한은 11~12장에서 자세히 다룬다.

4장에서 기억할 것은 하나다.

> Cloud의 병렬성은 독립 Task가 있을 때만 가치가 생긴다.

---

## 9. Evidence가 있어야 비동기 위임이 끝난다

Developer가 Cloud Worker의 실행 과정을 계속 지켜보지 않는다면 완료 시 검증 가능한 결과가 필요하다.

예:

```text
Git SHA: abc123
Unit: PASS
Integration: PASS
Docker: PASS
E2E: FAIL 1
Artifact: e2e-failure-screenshot.png
```

이런 결과가 있으면 Developer는 전체 로그를 처음부터 읽지 않고 실패한 지점부터 확인할 수 있다.

Evidence의 구조와 Tool Output을 줄이는 방법은 8장에서 다룬다.

여기서는 다음 관계만 연결한다.

```text
비동기 위임
→ 작업 과정을 계속 관찰하지 않음
→ 완료 시 검증 가능한 Evidence 필요
```

---

## 10. campus-platform에 적용하면

출결 API를 수정한 뒤 다음 검증이 필요하다고 하자.

```text
Unit Test
Integration Test
Admin Web E2E
Docker Build
```

Local에서 순차적으로 실행할 수도 있다.

또는 Base SHA를 기준으로 Cloud에 분리할 수 있다.

```text
Local
→ Attendance / Auth 수정
→ Base SHA abc123

Cloud
├─ Cloud Runner #1 :attendance:test
├─ Cloud Runner #2 integrationTest
├─ Cloud Runner #3 admin-web E2E
└─ Cloud Runner #4 Docker Build
```

Developer는 Cloud 검증이 진행되는 동안 다음 기능을 작업한다.

검증 결과가 돌아오면 실패 항목만 추가로 분석한다.

이 예제에서 Cloud의 가치는 `Test가 반드시 더 빨리 끝난다`는 것이 아니다.

```text
Local Compute 점유 감소
+
Developer Blocking Time 감소
+
독립 검증 병렬 실행
```

이 세 효과를 얻을 수 있다는 점이다.

---

## 11. 이 장에서 측정할 값

Cloud Agent를 도입한 뒤 단순히 `Agent가 몇 분 걸렸는가`만 측정하지 않는다.

최소한 다음을 나눠 본다.

| 항목 | 의미 |
| --- | --- |
| Agent Execution Time | Cloud Task 자체 실행시간 |
| Queue Time | 실행 시작 전 대기시간 |
| Developer Blocking Time | Developer가 실제로 멈춘 시간 |
| Review Time | 결과 확인과 승인에 사용한 시간 |
| Retry Time | 실패 후 재실행에 사용한 시간 |
| Local Resource Occupancy | Local CPU / RAM / Disk를 무거운 Task가 점유한 정도 |

모든 팀에 같은 목표값을 적용하지 않는다.

중요한 것은 Cloud 도입 전후에 **개발자의 전체 Workflow가 어떻게 바뀌었는가**다.

---

## 12. 이 장에서 기억할 구조

```text
Developer / Local Agent
        |
     Task Split
        |
        +------ Local 작업 계속
        |
     Cloud Worker
        |
 Build / Test / Independent Work
        |
      Evidence
        |
       Review
```

Cloud Agent의 가치는 Agent 한 명의 응답시간만으로 평가할 수 없다.

독립 실행환경으로 Local 자원을 분리하고, 사람이 기다리지 않아도 되는 작업을 비동기로 보내고, 독립 Task를 병렬 실행하는 데 의미가 있다.

다음 장에서는 이 가치를 실제 Task에 적용하기 위해 `Local / Cloud / Hybrid / Runner-first` Routing 기준을 만든다.

# Part II. 어떤 Task를 Cloud로 보낼 것인가

Cloud에서 실행할 수 있다는 이유만으로 모든 작업을 Cloud에 보내는 것은 좋은 Routing이 아니다.

작업마다 필요한 Context, Human Steering, 내부망 접근, 실행시간, 검증 방식이 다르다. 같은 Bug Fix라도 재현 가능하고 Scope가 작다면 Cloud Agent에 적합할 수 있지만, Architecture 판단과 내부 시스템 접근이 필요하다면 Local이 더 적합할 수 있다.

이 Part에서는 먼저 Task의 실행 위치를 결정하는 기준을 만든다. 다음으로 Build, Test, E2E, Docker, Migration, Bug Fix, Refactoring 같은 실제 개발 작업을 `Local / Cloud Runner / Cloud Agent / Hybrid` 관점에서 분류한다.

Cloud Agent에게 작업을 넘기기로 했다면 입력도 줄여야 한다. Task Contract로 Scope와 Validation을 고정하고, Repository 전체가 아니라 필요한 Context부터 제공한다. 결과 역시 대형 로그가 아니라 검증 가능한 Evidence를 중심으로 받는다.

이 Part의 흐름은 다음과 같다.

```text
Task 선택
→ 실행 주체 결정
→ 입력 범위 축소
→ 결과 / Evidence 경계 정의
```

다음 Part에서는 이렇게 선택한 Task가 Cloud에서 바로 실행될 수 있도록 **환경과 검증 경로**를 설계한다.


# 5장. Task Routing: Local인가 Cloud인가

Cloud Agent를 잘 사용하는 팀은 모든 작업을 Cloud로 보내지 않는다.

더 중요한 것은 `이 Task를 어디에서 실행하는 것이 유리한가`를 빠르게 판단하는 것이다.

이 장에서는 Task를 다음 네 경로로 분류한다.

```text
Task
  ↓
Task Classification
  ↓
Local / Cloud / Hybrid / Runner-first
```

4장에서 Cloud의 독립 실행환경과 비동기·병렬 실행 가치를 설명했다면, 이번 장에서는 그 가치를 **어떤 Task에 적용할지** 결정한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

17장에서는 이 판단을 반대로 적용해 Cloud Task를 중단하거나 Local로 되돌릴 조건을 다룬다. 이번 장의 역할은 **Task 시작 시점의 Routing Framework**를 만드는 것이다.

---

## 1. Routing은 모델 선택보다 실행 위치 선택에서 시작한다

Task를 받으면 `어떤 모델이 잘할까`보다 먼저 다음을 본다.

```text
이 작업은 어디에서 실행해야 하는가?
어떤 실행 주체가 필요한가?
```

예를 들어 신규 인증 Architecture를 설계하는 Task에는 다음이 필요할 수 있다.

```text
여러 모듈 탐색
요구사항 비교
Developer와 반복 대화
내부 시스템 제약 확인
```

이 작업은 Local에 가깝다.

반면 다음은 Cloud Runner에 가깝다.

```bash
./gradlew test
```

명령과 성공 조건이 정해져 있기 때문이다.

다음처럼 재현 가능한 작은 Bug는 Cloud Agent 후보가 된다.

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

Routing은 작업 이름이 아니라 **작업 상태와 완료 조건**을 보고 결정한다.

---

## 2. 첫 질문은 `Runner로 끝낼 수 있는가`다

Cloud에 보내는 Task라고 모두 Agent가 필요한 것은 아니다.

가장 먼저 결정론적으로 끝낼 수 있는지 확인한다.

```text
Task
  ↓
결정론적 실행으로 판정 가능한가?
  ├─ YES → Runner / Tool
  └─ NO  → Agent 또는 Human 판단 후보
```

Runner에 적합한 대표 작업:

```text
Build
Unit Test
Integration Test
E2E
Docker Build
Lint
Static Analysis
Migration Validation
```

이 작업은 명령과 PASS / FAIL 판정 기준이 명확하다.

```text
Runner
→ 실행
→ PASS / FAIL
```

PASS라면 종료한다.

FAIL이고 원인 분석이나 코드 수정이 필요할 때 Agent가 들어간다.

이 원칙은 6장에서 작업 유형별로 적용하고, 10장에서 Runner-first 실행 구조로 상세히 다룬다.

---

## 3. Hard Constraint를 먼저 확인한다

Cloud에 유리한 조건을 여러 개 합산하기 전에 **Cloud 실행 자체를 막는 조건**이 있는지 본다.

대표 Hard Constraint:

```text
Internal Network 필수
Repository / 데이터를 Cloud에 제공할 수 없음
현재 Local 상태를 그대로 사용해야 함
특정 장비 / 디바이스에 직접 접근해야 함
```

예:

```text
HSM 실제 장비 검증
→ Local

Tibero 운영환경에서만 재현되는 SQL 문제
→ Local 또는 Hybrid
```

다른 조건이 좋아도 Hard Constraint가 있으면 실행 위치가 먼저 제한된다.

```text
Hard Constraint
→ 실행 가능한 위치 결정
        ↓
그 안에서 비용 / 시간 최적화
```

보안·정책·네트워크 제약을 우회하는 것이 Routing의 목적은 아니다.

---

## 4. Local에 가까운 Task

다음 신호가 많을수록 Local이 유리하다.

```text
요구사항이 아직 불명확함
Human Steering이 잦음
큰 Context가 필요함
미커밋 Local 상태 의존
내부망 / 장비 의존
재현 절차가 불명확함
Architecture 판단 비중이 큼
```

예:

```text
새 인증 구조를 JWT로 유지할지 다른 구조로 바꿀지 검토
```

이 작업은 독립 실행보다 탐색과 의사결정에 가깝다.

```text
Developer + Local Agent
→ 탐색
→ 선택지 비교
→ 결정
→ Task Split
```

방향이 정해진 뒤 구현이나 검증을 작은 Cloud Task로 나눌 수 있다.

Local의 핵심 강점은 **Developer와 Agent 사이의 짧은 Feedback Loop**다.

---

## 5. Cloud에 가까운 Task

다음 신호가 많을수록 Cloud로 보내기 쉽다.

```text
Scope 명확
완료 조건 명확
Git으로 상태 전달 가능
독립 검증 가능
중간 질문 적음
장시간 실행
Compute 사용량 큼
다른 Task와 충돌 적음
Evidence로 결과 반환 가능
```

예:

```text
Task
attendance 모듈 Java 21 호환성 수정

Scope
attendance module only

Validation
./gradlew :attendance:test
```

또는:

```text
Task
전체 Web E2E 실행

Validation
npx playwright test
```

Cloud가 유리한 이유는 단순히 원격에서 실행되기 때문이 아니다.

Task를 독립적으로 위임하고, Developer는 다른 작업을 진행하며, 결과를 나중에 Evidence로 받을 수 있기 때문이다.

---

## 6. Hybrid는 단계별 Routing이다

기업 프로젝트에서는 Local과 Cloud 하나만으로 끝나지 않는 Task가 많다.

예를 들어 DB Migration을 수정한다고 하자.

```text
Local
→ 요구사항 / 실제 DB 제약 확인
        ↓
Cloud
→ Migration 작성 / 일반 검증
→ Testcontainers 기반 Test
        ↓
Local
→ Tibero 실제 적용 검증
```

이 경우 하나의 기능 안에서 실행 위치가 바뀐다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

Hybrid는 예외 처리라기보다 **재현 가능한 부분은 Cloud로 보내고 실제 내부 경계는 Local에 남기는 방식**이다.

13장에서 이 Handoff를 전체 Workflow로 확장한다.

---

## 7. Human Steering과 Context 크기를 함께 본다

Cloud Agent는 비동기 위임에 유리하다. 그러나 작업 중 사람이 계속 방향을 바꿔야 한다면 이 장점은 줄어든다.

예:

```text
관리자 UI 개선
→ A안 구현
→ Developer 확인
→ 구조 변경
→ B안 구현
→ 다시 확인
```

이런 Task는 Local에 가깝다.

반대로 다음 Task는 중간 개입이 적다.

```text
Goal
Expired JWT → HTTP 401

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

Context도 같은 방식으로 본다.

작은 Context:

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

큰 Context:

```text
Auth
Student
Permission
Database
Common Exception
External API
Admin Web
```

큰 문제는 Local에서 먼저 범위를 좁힌 뒤 작은 Task로 Cloud에 보내는 편이 낫다.

```text
큰 문제
→ Local 탐색
→ 결정
→ Task 분해
→ Cloud 위임
```

Context를 실제로 어떻게 작게 구성하는지는 7장에서 다룬다.

---

## 8. Git으로 기준 상태를 전달할 수 있는가

Cloud Worker는 Local IDE의 현재 상태를 자동으로 공유하지 않는다.

다음 상태에 강하게 의존하면 Handoff가 어렵다.

```text
미커밋 파일
untracked config
로컬 DB 임시 데이터
IDE 내부 상태
특정 프로세스 상태
```

Cloud Task는 가능한 한 다음처럼 명시적인 기준점에서 시작하는 편이 좋다.

```text
Repository
Base SHA
Branch
Task
Validation
```

예:

```text
repository: campus-platform
base_sha: abc123
branch: agent/auth-expired-token
```

Git과 Branch를 실제 Handoff Boundary로 사용하는 방법은 11장과 13장에서 구체화한다.

이번 장에서의 판단은 단순하다.

> 현재 작업 상태를 다른 실행환경으로 명확히 전달할 수 있는가?

---

## 9. 독립 검증과 변경 충돌을 본다

Cloud Task는 독립 검증이 가능할수록 관리하기 쉽다.

좋은 예:

```text
Task A → attendance module test
Task B → notification module test
```

반대로 다음은 독립 Task라고 보기 어렵다.

```text
Task A → UserService 구조 변경
Task B → UserService 오류 처리 변경
Task C → UserService 기반 DTO 변경
```

각 Branch에서 성공하더라도 통합 시 충돌할 수 있다.

Routing 단계에서는 다음 정도만 확인한다.

```text
예상 변경 파일
shared / common module
DB Schema / Migration
공통 DTO / API
순서 의존성
```

Source / Runtime 격리는 11장에서, 병렬화 비용과 Fan-in은 12장에서 상세히 다룬다.

---

## 10. Task 크기는 Cloud Overhead와 함께 본다

Cloud에는 작업 내용 외의 준비 비용이 있다.

```text
Worker Start
Checkout
Environment 준비
Context Load
Commit / PR
Review
```

따라서 너무 작은 Task는 Cloud가 불리할 수 있다.

```text
문구 한 줄 수정
검증 수초
```

반대로 너무 큰 Task도 문제가 된다.

```text
프로젝트 전체 Architecture 개선
```

Task가 커지면 다음 비용이 함께 커진다.

```text
Context
변경 파일 수
Retry 범위
검증 범위
Review 부담
Merge Risk
```

절대적인 `몇 분 이하이면 Local` 같은 규칙은 두지 않는다.

프로젝트마다 Cold Start와 Review 비용이 다르기 때문이다.

적정 Task는 대체로 다음 특징을 가진다.

```text
독립 실행 가능
독립 검증 가능
Review 가능한 변경 범위
명확한 완료 조건
```

7장의 Task Contract가 이 범위를 고정하는 입력이 된다.

---

## 11. Routing Decision Matrix

다음 표는 시작점이다.

| 조건 | Local | Cloud | Hybrid |
| --- | --- | --- | --- |
| 요구사항 불명확 | 유리 | 불리 | 불리 |
| Human Steering 잦음 | 유리 | 불리 | 가능 |
| 큰 Context 필요 | 유리 | 불리 | 가능 |
| 장시간 Build / Test | 가능 | 유리 | 유리 |
| Compute 사용 큼 | 가능 | 유리 | 유리 |
| Internal Network 필수 | 유리 | 불리 | 유리 |
| Git Handoff 가능 | 가능 | 유리 | 유리 |
| 독립 검증 가능 | 가능 | 유리 | 유리 |
| 동일 파일 / DB Schema 충돌 큼 | 유리 | 불리 | 불리 |
| Architecture 판단 | 유리 | 불리 | 가능 |

이 표는 점수 합산으로 정답을 만드는 도구가 아니다.

Hard Constraint 하나가 다른 조건보다 우선할 수 있다.

---

## 12. Score는 보조 수단일 뿐이다

팀 내에서 빠르게 대화하기 위해 간단한 Score를 사용할 수 있다.

아래 점수는 설명용 예다.

```text
Scope 명확성             +2
독립 검증 가능            +2
장시간 실행               +1
Compute 사용 큼           +1
Human Steering 많음       -2
Internal Network 필요     -3
큰 Context 필요           -2
파일 충돌 가능성 높음     -2
```

점수만으로 자동 결정하지 않는다.

판단 순서는 다음이 낫다.

```text
1. Hard Constraint
2. Runner로 끝낼 수 있는가
3. Scope / 완료 조건
4. 독립 검증 가능성
5. Handoff 가능성
6. Human Steering / Context
7. 충돌 가능성
8. Cloud Overhead와 Task 가치
```

Score는 이 대화를 짧게 하기 위한 보조 도구다.

---

## 13. campus-platform 작업을 분류해보자

| Task | 기본 경로 | 이유 |
| --- | --- | --- |
| 신규 인증 Architecture | Local | 큰 Context, Human Steering |
| expired token 수정 | Cloud Agent 후보 | 작은 Scope, 재현 / 검증 가능 |
| 전체 Unit Test | Cloud Runner | 결정론적, Compute 중심 |
| Web E2E | Cloud Runner | Browser 기반 독립 검증 |
| Tibero Migration 실제 검증 | Local / Hybrid | 내부 DB 의존 |
| HSM 오류 분석 | Local | 내부 장비 / 네트워크 의존 |
| Docker Build | Cloud Runner | 결정론적 Build |
| Dependency Update | Runner-first | PASS면 Agent 불필요 |
| README 한 줄 수정 | Local 후보 | Cloud Overhead가 상대적으로 큼 |

프로젝트의 실행환경이 바뀌면 Routing 결과도 바뀔 수 있다.

그래서 작업 이름보다 기준을 유지한다.

---

## 14. Routing 판단 순서

실제 Task를 받으면 다음 순서로 본다.

```text
1. Hard Constraint가 있는가?
2. Runner로 끝낼 수 있는가?
3. Scope와 완료 조건이 명확한가?
4. 독립 검증 가능한가?
5. Git으로 상태를 전달할 수 있는가?
6. Human Steering이 많이 필요한가?
7. Context가 과도하게 큰가?
8. 파일 / DB Schema 충돌 가능성이 큰가?
9. Cloud Overhead보다 Task 가치가 큰가?
```

이 질문에 답하면 대부분의 작업은 `Local / Cloud / Hybrid / Runner-first` 중 하나로 좁혀진다.

Routing은 고정된 분류표가 아니라 현재 Task 상태에 대한 판단이다.

다음 장에서는 이 기준을 Build, Test, E2E, Docker, Migration, Refactoring, Bug Fix, Documentation, PR Review, Dependency Update, CI Failure 같은 실제 작업에 적용한다.

# 6장. Cloud에 보내기 좋은 개발 작업

5장에서 Task를 `Local / Cloud / Hybrid / Runner-first` 중 어디에 배치할지 판단하는 기준을 만들었다.

이제 그 기준을 실제 개발 작업에 적용한다.

중요한 것은 작업 이름만 보고 Cloud Agent를 호출하지 않는 것이다.

같은 `Test 작업`도 실제 Workflow에서는 다음처럼 나뉠 수 있다.

```text
Test 실행
→ Cloud Runner

실패 원인 분석
→ Cloud Agent

수정 후 재검증
→ Cloud Runner
```

이 장에서는 작업을 세 범주로 본다.

```text
Deterministic Work
→ Runner / Tool

Reasoning + Code Change
→ Cloud Agent

Internal / Interactive Work
→ Local 또는 Hybrid
```

> Cloud에 보내기 좋은 작업과 Cloud Agent가 직접 해야 하는 작업은 같은 개념이 아니다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> Agent는 판단과 수정이 필요한 구간에만 사용한다.

10장에서는 이 원칙을 Runner-first 실행 구조로 더 구체화한다. 이번 장의 역할은 **실제 개발 작업을 어떤 실행 주체에 배치할지 Catalog를 만드는 것**이다.

---

## 1. 작업 이름보다 실행 구조를 본다

Dependency Update를 예로 들어보자.

```text
Version Update
      ↓
Cloud Runner
→ Build / Test
      ↓
PASS ─────────→ Done
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent
 ↓
Compatibility Fix
 ↓
Cloud Runner 재검증
```

처음부터 Agent가 Repository 전체를 분석할 필요는 없다.

작업을 볼 때 다음 질문을 사용한다.

```text
실행 명령이 명확한가?
판정 기준이 명확한가?
코드 변경이 필요한가?
판단이 필요한가?
내부 시스템 의존성이 있는가?
어떤 Evidence를 남겨야 하는가?
```

이 질문을 Build, Test, E2E, Migration, Bug Fix 같은 실제 작업에 적용한다.

---

## 2. Build는 Runner 작업이다

Java / Spring Boot 프로젝트의 Build는 보통 명령과 판정 기준이 명확하다.

```bash
./gradlew clean build
```

기본 흐름:

```text
Git SHA
  ↓
Cloud Runner
  ↓
Build
  ↓
PASS / FAIL
```

PASS라면 Agent를 호출하지 않는다.

FAIL이고 코드 수정이 필요하면 실패 요약을 Agent에게 넘긴다.

```text
BUILD: FAIL
AuthService.java:142
cannot find symbol
JwtExpiredException
```

### 기본 Evidence

```text
Git SHA
Build Status
Duration
Artifact Path
Compile Error Summary
```

Build 방법이 이미 표준화되어 있다면 Agent에게 `어떻게 빌드해야 할지` 다시 추론시키지 않는다.

---

## 3. Unit Test는 Cloud Compute에 잘 맞는다

Unit Test는 실행 명령과 성공 조건이 명확하다.

```bash
./gradlew test
```

또는 대상만 좁혀 실행할 수 있다.

```bash
./gradlew test --tests AuthServiceTest
```

기본 구조:

```text
Cloud Runner
→ Unit Test
→ PASS → 종료
→ FAIL → 실패 Test 목록
```

Agent가 필요하다면 실패한 Test부터 본다.

```text
AuthServiceTest.expiredToken
expected: 401
actual: 200
```

전체 로그를 처음부터 Context에 넣는 방법은 피한다. Tool Output을 줄이는 방법은 8장에서 다룬다.

Unit Test는 다음 조건에서 Cloud 활용 가치가 커진다.

```text
Test 수가 많음
실행시간이 김
모듈별 분할 가능
Local CPU / RAM 점유가 큼
```

---

## 4. Integration Test는 재현 가능한 환경이 핵심이다

Integration Test는 Unit Test보다 Runtime 의존성이 크다.

예:

```text
Spring Boot
PostgreSQL
Redis
Kafka
Testcontainers
Mock HTTP Server
```

Cloud에서 이 환경을 재현할 수 있다면 Cloud Runner에 적합하다.

```text
Prepared Environment
        ↓
Integration Runner
        ↓
Testcontainers / DB / Mock
        ↓
Integration Test
```

반대로 실제 내부 자원이 필수라면 Hybrid가 된다.

```text
Cloud
→ 일반 Integration Validation

Local / Internal
→ Tibero / Internal API 최종 검증
```

Integration Test의 핵심 질문은 `Cloud에서 돌릴 수 있는가`보다 **Cloud에서 필요한 상태를 재현할 수 있는가**다.

---

## 5. E2E와 Browser Test는 Cloud 분리에 적합하다

Browser E2E는 실행시간과 자원 점유가 크고 Screenshot, Video, Trace 같은 Artifact를 남길 수 있다.

```text
Cloud Runner
→ Browser 시작
→ E2E 실행
→ Screenshot / Video / Trace 저장
```

시나리오가 독립적이라면 여러 Worker로 나눌 수 있다.

```text
Worker #1 → auth scenarios
Worker #2 → attendance scenarios
Worker #3 → admin scenarios
```

실패 시 Agent에게 필요한 것은 전체 Trace가 아니라 우선 다음 정보다.

```text
Failed Scenario
Expected / Actual
Screenshot
Console Error Summary
```

### 기본 Evidence

```text
Scenario Count
Failed Scenario List
Screenshot
Video / Trace Reference
Console Error Summary
```

UI 변경은 8장의 `Demos over Diffs`와 연결된다.

---

## 6. Docker Build도 Runner가 먼저다

```bash
docker build -t campus-platform:abc123 .
```

Docker Build는 일반적으로 다음 구조로 충분하다.

```text
Git SHA
→ Cloud Runner
→ Docker Build
→ PASS / FAIL
→ Image Digest / Build Log
```

실패했다고 모두 코드 Agent 문제는 아니다.

```text
Dockerfile 문제
Dependency 문제
Registry / Network 문제
Base Image 문제
```

코드나 Dockerfile 수정이 필요한 경우에만 Agent를 호출한다.

Prepared Environment와 Docker Layer Cache는 9장에서 다룬다.

---

## 7. Static Analysis와 Lint는 Tool-first다

다음은 기본적으로 일반 도구가 처리한다.

```text
Formatter
Lint
Checkstyle
SpotBugs
Architecture Check
Dependency Check
Security Scan
```

기본 흐름:

```text
Tool
→ PASS / FAIL
```

자동 수정이 가능하면 Agent보다 Tool을 먼저 사용한다.

```text
FAIL
 ↓
Auto-fix 가능?
 ├─ YES → Tool Fix → Recheck
 └─ NO  → Agent 후보
```

> 코드로 판정하고 코드로 수정할 수 있는 규칙은 Agent보다 도구를 먼저 사용한다.

---

## 8. Migration은 작성과 검증을 분리한다

DB Migration은 한 덩어리의 Agent Task로 보지 않는다.

```text
Migration 작성
→ Developer / Agent

Migration 검증
→ Cloud Runner
```

Cloud에서 Disposable DB를 사용할 수 있다면 다음을 검증할 수 있다.

```text
clean DB 적용
기존 DB Schema에서 upgrade
syntax
순서
기본 Integration Test
```

실제 대상이 내부 Tibero / Oracle이라면 마지막 경계만 Local에 남긴다.

```text
Cloud
→ 일반 Migration Validation
        ↓
Local / Internal
→ 실제 DB 검증
```

Agent가 Migration을 작성했다고 Agent의 자연어 판단으로 검증을 끝내지 않는다.

---

## 9. 반복 Refactoring은 Cloud Agent 후보다

판단과 코드 변경이 필요하지만 범위를 제한할 수 있는 Refactoring은 Cloud Agent에 잘 맞는다.

예:

```text
deprecated API 교체
Java 17 → 21 호환 수정
반복 import 변경
동일 패턴 수정
```

좋은 Task:

```text
attendance 모듈에서 deprecated API를 교체하고
./gradlew :attendance:test를 통과한다.
```

좋지 않은 Task:

```text
프로젝트 전체 구조를 더 좋게 리팩터링해.
```

Cloud Agent 후보의 조건:

```text
변경 패턴 명확
Scope 제한
검증 명령 존재
독립 Branch 가능
Review 가능한 Diff
```

Architecture 방향은 Local에서 정하고 반복 실행 부분만 Cloud로 분리한다.

---

## 10. 작은 Bug Fix는 재현 가능성이 먼저다

Cloud Agent에 보내기 좋은 Bug는 실패와 완료 조건이 명확하다.

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

기본 흐름:

```text
Reproducible Failure
       ↓
Cloud Agent
       ↓
Analyze / Fix
       ↓
Cloud Runner
       ↓
PASS
```

반대로 다음 Bug는 Local에 더 가깝다.

```text
Production에서만 간헐 발생
실제 HSM 의존
내부 DB 상태 의존
재현 절차 불명확
여러 시스템을 동시에 조사해야 함
```

Cloud Agent의 모델 성능보다 **재현 가능성**이 먼저다.

---

## 11. Documentation은 사실 기반 갱신부터 보낸다

Cloud에 적합한 문서 작업:

```text
API 변경에 맞춘 README 수정
특정 Module 문서 갱신
Release Note 초안
변경 코드 기반 운영 문서 수정
```

이 작업은 Repository와 Commit을 기준으로 범위를 제한하기 쉽다.

반대로 다음은 Local 중심이 자연스럽다.

```text
새 Architecture 원칙 정의
조직 전체 개발 정책 결정
요구사항이 정해지지 않은 문서
```

즉 **사실 기반 갱신**과 **방향 결정**을 구분한다.

---

## 12. PR Review는 보조 Worker로 사용한다

Cloud Agent는 PR Review를 보조할 수 있다.

검토 후보:

```text
변경 범위
Test 누락
명확한 오류
규칙 위반
문서 / 코드 불일치
```

권장 순서:

```text
PR
→ Deterministic Checks
→ Cloud Review Agent
→ Findings
→ Human Review
```

Cloud Agent의 Review를 최종 승인과 동일하게 취급하지 않는다.

기계적으로 검증할 수 있는 항목은 CI / Tool이 먼저 처리하고, Agent는 의미 판단을 보조한다.

---

## 13. Dependency Update는 Runner-first 대표 사례다

```text
Dependency Version 변경
      ↓
Cloud Runner
→ Build / Test
      ↓
PASS → PR
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent Compatibility Fix
 ↓
Cloud Runner 재검증
```

PASS하면 LLM이 필요 없다.

FAIL일 때도 Agent에게 Repository 전체를 다시 설명하지 않는다.

```text
변경 Dependency
이전 / 신규 Version
실패 Command
실패 Test
관련 파일
```

이 패턴은 Event-driven Workflow에도 그대로 연결된다.

---

## 14. CI Failure Fix는 Agent-on-failure에 잘 맞는다

```text
Git Push / PR
     ↓
    CI
     ↓
PASS → Done
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent
 ↓
Fix
 ↓
Cloud Runner
 ↓
PASS
```

이 구조에서 Agent는 상시 실행되지 않는다.

실패가 발생했고 코드 판단이 필요한 경우에만 호출한다.

CI Failure, Review Comment, Nightly Test 같은 Event가 실제 Task를 만드는 구조는 14장에서 다룬다.

---

## 15. 하나의 작업 안에서도 실행 주체는 바뀐다

`Unit Test는 Runner`, `Bug Fix는 Agent`처럼 작업 이름에 실행 주체를 영구적으로 붙이지 않는다.

하나의 Workflow 안에서도 바뀐다.

```text
Unit Test 실행
→ Cloud Runner

실패 원인 분석
→ Cloud Agent

코드 수정
→ Cloud Agent

재검증
→ Cloud Runner
```

Migration도 마찬가지다.

```text
Migration 작성
→ Developer / Agent

Disposable DB Validation
→ Cloud Runner

Tibero 실제 검증
→ Local / Internal
```

실행 주체는 Task의 **현재 단계**에 따라 선택한다.

---

## 16. 작업별 기본 Catalog

| 작업 | 기본 실행 주체 | Agent 호출 조건 | 대표 Evidence |
| --- | --- | --- | --- |
| Build | Cloud Runner | 코드 / Build 수정 필요 | Status, Artifact, Error Summary |
| Unit Test | Cloud Runner | 실패 원인 분석 / 수정 | Failed Tests, JUnit Result |
| Integration Test | Cloud Runner | 재현 가능한 Code Failure | Test Result, Logs / Artifacts |
| E2E | Cloud Runner | UI / Code 분석 필요 | Screenshot, Video / Trace |
| Docker Build | Cloud Runner | Dockerfile / 코드 수정 필요 | Image Digest, Build Result |
| Lint / Static Analysis | Tool / Runner | Auto-fix 불가 | Violation Summary |
| Migration Validation | Cloud Runner | Migration 수정 필요 | Apply Result, Test Result |
| 반복 Refactoring | Cloud Agent | 처음부터 판단 / 수정 필요 | Commit, Test Result |
| 작은 Bug Fix | Cloud Agent | 재현 가능해야 함 | Commit, Target Test |
| Documentation | Cloud Agent 후보 | 사실 기반 범위 명확 | Changed Files, Review |
| PR Review | Cloud Agent 보조 | 의미 검토 필요 | Findings |
| Dependency Update | Runner-first | FAIL일 때 | Build / Test Result |
| CI Failure Fix | Agent-on-failure | Code Failure일 때 | Fix Commit, Revalidation |

이 표는 기본값이다.

Internal Network, Human Steering, Context 크기 같은 조건은 5장의 Routing 기준을 다시 적용한다.

---

## 17. campus-platform Task Catalog

프로젝트에서 반복되는 Task는 이름과 실행 방식을 미리 정해둘 수 있다.

```text
RUN-BUILD
→ ./gradlew build

RUN-UNIT
→ ./gradlew test

RUN-INTEGRATION
→ integrationTest + Testcontainers

RUN-E2E
→ npx playwright test

RUN-DOCKER
→ docker build

FIX-BUG
→ Failure Summary + Relevant Files + Target Validation

REFACTOR-MODULE
→ One Module + Module Test
```

Migration은 다음처럼 나눈다.

```text
Cloud
→ Disposable DB Validation

Local / Internal
→ Tibero 실제 검증
```

이런 Catalog가 있으면 매번 Agent에게 `무엇을 어떻게 실행할지` 처음부터 설명할 필요가 줄어든다.

15장에서는 이 Catalog를 `campus-platform` 전체 운영 모델에 배치한다.

---

## 18. 이 장에서 기억할 실행 순서

실제 작업을 받으면 다음처럼 생각한다.

```text
1. Cloud로 보낼 가치가 있는가?
2. Tool / Runner만으로 처리 가능한가?
3. 실패 또는 코드 판단이 발생했는가?
4. Agent가 필요한가?
5. Agent 수정 후 Cloud Runner로 재검증했는가?
6. Evidence를 남겼는가?
```

한 줄로 줄이면 다음과 같다.

```text
Cloud Runner
→ 필요한 순간에 Cloud Agent
→ 다시 Cloud Runner
```

다음 장에서는 Agent에게 실제 수정 Task를 넘길 때 Repository 전체를 다시 탐색하지 않도록 `Task Contract`와 작은 Context를 구성한다.

# 7장. Cloud Agent Task Contract: 작은 Task와 작은 Context

Cloud Agent에 작업을 맡길 때 필요한 것은 긴 Prompt가 아니다.

Repository 설명, Architecture, Coding Rule, 과거 장애 이력과 예상 원인을 한 번에 넣으면 정보량은 늘지만 Agent가 실제로 탐색해야 하는 범위도 함께 커진다.

이 장에서는 Cloud Worker가 바로 작업을 시작할 수 있도록 입력 경계를 정리한다. 이를 `Cloud Agent Task Contract`라고 부른다.

기본 형태는 다음과 같다.

```text
Task
Goal
Scope
Relevant Files
Forbidden Changes
Validation
Expected Result
Output / Evidence
```

필요하면 다음 정보를 추가한다.

```text
Base SHA / Branch
Environment
Related Document
Budget
```

핵심은 한 문장으로 정리할 수 있다.

> Task Contract의 목적은 Prompt를 길게 만드는 것이 아니라 Agent가 탐색해야 하는 범위를 줄이는 것이다.

---

## 1. Cloud Task는 Prompt가 아니라 작업 패키지다

다음 지시는 범위가 넓다.

```text
로그인 쪽에 문제가 있는 것 같은데 Repository 전체를 확인해서
관련된 부분을 분석하고 수정한 다음 테스트도 해줘.
```

Agent는 문제 정의, 관련 파일, 변경 범위, 검증 방법, 완료 조건을 모두 다시 추론해야 한다.

같은 작업을 다음처럼 바꿀 수 있다.

```text
Task
AuthService expired token 처리 수정

Goal
Expired JWT 요청이 HTTP 401을 반환한다.

Scope
auth 모듈의 token expiration 처리와 관련 테스트

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
expired token → HTTP 401

Output / Evidence
- Result SHA
- Changed Files
- Validation Result
```

Task Contract는 Agent의 사고 과정을 대신 작성하는 문서가 아니다.

다음 네 가지를 고정하는 작업 패키지다.

```text
어디서 시작하는가?
어디까지 바꿀 수 있는가?
무엇이 성공인가?
무엇을 반환해야 하는가?
```

---

## 2. Goal과 Scope를 분리한다

Goal은 원하는 결과를 설명한다.

```text
Goal
Expired JWT 요청이 HTTP 401을 반환한다.
```

Scope는 탐색과 변경 범위를 설명한다.

```text
Scope
auth 모듈의 expired token 처리와 해당 regression test
```

Goal만 있으면 Agent가 목표를 달성하기 위해 어디까지 바꿔도 되는지 알기 어렵다.

반대로 Scope만 있으면 어떤 결과를 만들어야 하는지 불분명하다.

> Goal은 결과를 제한하고 Scope는 탐색과 변경 범위를 제한한다.

---

## 3. Relevant Files는 첫 번째 Context다

Agent가 Repository를 처음 열었을 때 관련 파일을 찾는 탐색 자체가 Context를 소비한다.

사람이 이미 시작점을 알고 있다면 그 정보를 전달한다.

```text
Relevant Files
- src/main/java/.../AuthService.java
- src/main/java/.../JwtTokenProvider.java
- src/test/java/.../AuthServiceTest.java
```

이 목록은 whitelist가 아니다.

예상하지 못한 Dependency가 있을 수 있으므로 다음 순서를 기본으로 한다.

```text
Relevant Files
      ↓
Direct Dependency
      ↓
Related Test / Document
      ↓
Wider Module Context
```

즉 Relevant Files는 `Initial Context Boundary`다.

작은 Context의 목적은 정보를 없애는 것이 아니라 필요한 정보까지 도달하는 경로를 짧게 만드는 것이다.

---

## 4. Forbidden Changes는 Diff를 작게 유지한다

Bug 하나를 고치면서 주변 구조까지 함께 정리하면 Review 범위가 빠르게 커진다.

예를 들어 expired token 오류를 수정하다가 다음 변경까지 섞일 수 있다.

```text
Exception hierarchy 정리
JWT 구조 변경
공통 API response 변경
DB Migration 추가
OAuth 설정 변경
```

각 변경이 타당해도 원래 Task와 섞이면 회귀 위험과 Review 비용이 커진다.

따라서 하지 말아야 할 영역을 함께 적는다.

```text
Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API
- public response format
```

필요하면 경로로 제한할 수 있다.

```text
Do Not Change
- db/migration/**
- common/exception/**
- security/oauth/**
```

목적은 Governance가 아니라 Task의 변경 경계를 지키는 것이다.

---

## 5. Validation은 실행 가능한 명령으로 준다

다음 완료 조건은 판정하기 어렵다.

```text
기능이 정상 동작하는지 확인한다.
```

가능하면 Validation을 명령으로 만든다.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

필요하면 좁은 검증과 회귀 검증을 나눈다.

```text
Target Validation
./gradlew test --tests AuthServiceTest.expiredToken

Regression Validation
./gradlew test --tests AuthServiceTest
```

Agent가 코드를 읽고 `문제없어 보인다`고 판단하는 것과 실제 Test PASS는 다르다.

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

이 Validation은 10장의 Runner-first 구조에서 그대로 사용한다.

---

## 6. Expected Result와 Evidence를 미리 정한다

Validation 명령만으로는 무엇이 성공인지 충분하지 않을 수 있다.

Expected Result는 관찰 가능한 결과로 작성한다.

```text
Expired token
→ HTTP 401

Valid token
→ 기존 성공 동작 유지
```

그리고 Task가 반환할 결과도 미리 정한다.

```text
Output / Evidence
- Result SHA
- Changed Files
- Validation Result
- Artifact Reference if generated
- Short Summary
```

다음 결과만 받는 것은 부족하다.

```text
수정 완료했습니다.
테스트도 정상입니다.
```

Remote Worker는 자연어 설명보다 검증 가능한 결과를 남겨야 한다.

8장에서는 이 Evidence를 작게 전달하면서 Raw Artifact를 보존하는 방법을 다룬다.

---

## 7. Context는 필요할 때 단계적으로 넓힌다

모든 문서와 Source를 처음부터 Context에 넣지 않는다.

예를 들어 Repository에 다음 안내가 있다고 하자.

```text
AGENTS.md
├─ Backend → docs/backend.md
├─ Authentication → docs/security/auth.md
├─ Database → docs/database.md
├─ Testing → docs/testing.md
└─ Deployment → docs/deployment.md
```

AUTH-142라면 다음 경로부터 시작할 수 있다.

```text
Task Contract
→ AGENTS.md
→ docs/security/auth.md
→ Relevant Files
```

문제 해결에 실패해도 바로 Repository 전체로 확대하지 않는다.

```text
Level 1
Task Contract + Relevant Files
      ↓
Level 2
Direct Dependency + Related Test
      ↓
Level 3
Related Module Documentation
      ↓
Level 4
Specific Failure Artifact / Log
      ↓
Level 5
Wider Module / Repository Context
```

이 흐름을 `Progressive Context`로 사용할 수 있다.

> Context를 줄이는 것뿐 아니라 필요할 때 가져오게 만든다.

---

## 8. Base SHA와 Environment를 명시한다

Cloud Worker가 Remote Repository에서 작업하려면 시작점을 고정해야 한다.

```text
Repository
campus-platform

Base SHA
4f29abc

Task Branch
agent/auth-expired-token-142
```

결과도 같은 작업 상태와 연결한다.

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ PR
```

Branch와 Runtime 격리는 11장에서 다룬다.

Environment는 설치 방법보다 이름으로 선택하는 편이 낫다.

```text
Environment
backend-test
```

환경 구성 자체는 9장의 Prepared Environment에서 관리한다.

---

## 9. Budget은 선택 필드다

실패와 수정을 반복할 수 있는 Task에는 중단 조건을 둘 수 있다.

설명용 예:

```yaml
budget:
  max_retry: 2
```

표준값은 아니다. 프로젝트에 따라 다음 기준을 사용할 수 있다.

```text
max retry
max wall-clock time
max token/cost
max changed files
```

목적은 `성공할 때까지 계속해` 같은 무제한 Task를 피하는 것이다.

구체적인 Failure Fingerprint와 중단 판단은 8장과 17장에서 이어서 다룬다.

---

## 10. Task Contract가 너무 커지면 Task를 다시 본다

다음처럼 Contract 자체가 커졌다고 하자.

```text
Relevant Files: 수십 개
Related Modules: 다수
Validation Commands: 다수
Forbidden Changes: 수십 개
```

문서 형식을 더 복잡하게 만들기 전에 Task가 독립 실행하기에 너무 큰지 확인한다.

```text
Task Contract 과대
      ↓
Scope 재검토
      ↓
분리 가능한가?
  ├─ YES → 작은 Task로 분해
  └─ NO  → Local / Hybrid 재검토
```

Task를 작게 만드는 것 자체가 목적은 아니다.

Cloud Worker가 사람의 지속적인 개입 없이 끝낼 수 있는 단위인지 확인하는 것이 목적이다.

---

## 11. AUTH-142 최소 Contract

이 책에서 반복해서 사용할 expired token 예제를 하나의 Task로 정리하면 다음과 같다.

```text
Task ID
AUTH-142

Task
Expired token 처리 수정

Goal
Attendance API에 expired JWT 사용 시 HTTP 401 반환

Base SHA
4f29abc

Scope
auth 모듈의 token expiration handling과 관련 test

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Forbidden Changes
- DB Schema
- OAuth flow
- common exception response format

Environment
backend-test

Validation
./gradlew :auth:test --tests AuthServiceTest.expiredToken

Expected Result
- expired token → 401
- valid token 기존 동작 유지

Output / Evidence
- Result SHA
- Changed Files
- Validation Result
```

이 장 이후에는 같은 배경을 반복하지 않고 `AUTH-142`로 참조한다.

---

## 12. 작은 Input에서 작은 Output으로

Task Contract는 Cloud Agent의 입력 경계를 만든다.

```text
Task Contract
→ Small Input / Context
        ↓
Cloud Runner / Agent
```

하지만 Runner가 만든 대형 로그를 그대로 Agent에게 돌려주면 Context는 다시 커진다.

다음 장에서는 반대 방향을 다룬다.

```text
Cloud Runner / Agent
        ↓
Result Gateway
→ Small Output / Evidence
```

7장은 `어디까지 읽고 바꿀 것인가`를 줄이고, 8장은 `무엇을 먼저 읽을 것인가`를 줄인다.


# 8장. Tool Output을 줄이고 Evidence를 남기기

7장에서는 Cloud Agent에게 전달하는 입력을 줄였다.

```text
Task Contract
→ Small Input / Context
```

하지만 입력만 작게 만든다고 Context 비용이 줄어드는 것은 아니다.

Build, Test, E2E, Docker 작업은 많은 로그와 Artifact를 만든다. 이 결과를 그대로 LLM에 전달하면 몇 건의 실패를 찾기 위해 대량의 정상 로그까지 읽게 된다.

8장의 기본 구조는 다음과 같다.

```text
Cloud Runner / Agent
        ↓
Raw Result / Artifact
        ↓
Result Filter / Result Gateway
        ↓
Small Evidence Summary
        ↓
Agent / Developer
        ↓
필요할 때만 상세 Artifact 조회
```

핵심은 결과를 버리는 것이 아니다.

> 큰 결과는 보존하고, Agent에게는 판단에 필요한 부분만 보여준다.

그리고 Task 완료는 자연어 선언이 아니라 Evidence로 확인한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

---

## 1. Tool Output도 Context다

Agent가 읽는 다음 결과는 모두 Context가 된다.

```text
Gradle Build Log
JUnit Output
Spring Boot Startup Log
Testcontainers Log
Docker Build Output
Playwright Trace
Browser Console Log
Static Analysis Report
Git Diff
```

설명용 예로 전체 테스트가 다음 결과를 만들었다고 하자.

```text
Tests: 8,214
Passed: 8,211
Failed: 3
Raw Log: 100MB
```

좋지 않은 흐름:

```text
Cloud Runner
→ 100MB Log
→ Agent에게 전체 전달
→ Agent가 실패 3건 탐색
```

권장 흐름:

```text
Cloud Runner
→ Raw Log + Report 저장
→ 실패 3건 추출
→ Agent는 Failure Summary부터 확인
```

실행 결과의 크기와 LLM에 전달하는 결과의 크기를 분리한다.

---

## 2. Result Filter는 결정론적 프로그램으로 시작한다

대형 로그에서 다음 값은 LLM이 없어도 추출할 수 있다.

```text
Exit Code
Build Status
Total / Passed / Failed Count
Failed Test Name
Exception Type
Assertion Message
Top Stack Frame
Error / Warning Count
Artifact Path
```

예:

```text
BUILD: FAIL

Tests
- total: 8,214
- passed: 8,211
- failed: 3

Failures
1. AuthServiceTest.expiredToken
2. UserServiceTest.deleteUser
3. UserMapperTest.insert
```

이 정도 결과는 Shell Script, JUnit XML Parser, CI Post-processing Script로 만들 수 있다.

> 코드로 추출할 수 있는 결과를 다시 LLM에게 읽혀서 찾게 하지 않는다.

이 원칙은 10장의 Runner-first 구조와 연결된다.

---

## 3. Result Gateway는 원본을 버리지 않는다

Result Filter가 필요한 값을 추출하는 단계라면 Result Gateway는 Raw Artifact를 보존하고 필요할 때 다시 조회할 수 있게 하는 경계다.

```text
Raw Artifact
├─ build.log
├─ junit.xml
├─ coverage.xml
├─ docker.log
├─ git.diff
├─ screenshots/
├─ browser-video/
└─ browser-trace/
       ↓
Result Gateway
├─ Summary
├─ Failed Test Index
├─ Exception Index
├─ Log Search
└─ Artifact Lookup
       ↓
Agent / Developer
```

처음부터 `build.log` 전체를 읽지 않는다.

```text
Summary
  ↓
Failure Detail
  ↓
Specific Log
  ↓
Related Artifact
  ↓
Full Raw Log
```

7장의 Progressive Context를 실행 결과에도 적용한 구조다.

이 인터페이스가 반드시 HTTP API일 필요는 없다.

```text
CLI
JSON Report
CI Artifact Link
File Index
Object Storage Path
```

핵심은 원본을 유지하면서 Agent가 처음 읽는 결과를 작게 만드는 것이다.

---

## 4. result.json은 첫 번째 결과 인터페이스가 될 수 있다

Task 결과를 사람이 읽는 긴 로그와 별개로 구조화할 수 있다.

예:

```json
{
  "taskId": "AUTH-142",
  "gitSha": "abc123",
  "status": "FAIL",
  "build": "PASS",
  "tests": {
    "total": 8214,
    "passed": 8211,
    "failed": 3
  },
  "failures": [
    "AuthServiceTest.expiredToken",
    "UserServiceTest.deleteUser",
    "UserMapperTest.insert"
  ],
  "artifacts": {
    "junit": "junit.xml",
    "log": "build.log"
  }
}
```

이 JSON 형식을 표준으로 강제하려는 것은 아니다.

중요한 것은 Agent와 자동화가 먼저 읽을 수 있는 작은 구조화 결과가 있다는 점이다.

```text
Task Scope
AUTH-142 / expired token
       +
Result Summary
AuthServiceTest.expiredToken FAIL
       ↓
Agent
```

현재 Task에 필요하지 않은 실패와 로그는 처음부터 읽지 않는다.

---

## 5. Artifact First

Cloud Runner가 `FAIL` 한 줄만 반환하면 원인을 다시 재현해야 한다.

반대로 Raw Log 전체를 Agent에게 보내는 것도 과하다.

Task 단위로 Artifact를 보존한다.

```text
artifacts/AUTH-142/
├─ result.json
├─ junit.xml
├─ build.log
├─ git.diff
├─ screenshots/
├─ video/
└─ traces/
```

Agent는 기본적으로 `result.json`부터 읽고 필요한 파일만 추가로 조회한다.

Artifact First의 목적은 파일을 많이 남기는 것이 아니다.

다음 질문에 답할 수 있어야 한다.

```text
무엇을 실행했는가?
어느 Git 상태에서 실행했는가?
성공했는가?
무엇이 실패했는가?
원본 결과가 남아 있는가?
```

---

## 6. 자연어 설명과 실행 사실을 분리한다

Agent Summary는 변경 의도를 설명하는 데 유용하다.

```text
Agent Summary
expired token 처리 분기를 수정했습니다.
```

하지만 검증 결과와 같은 것으로 취급하지 않는다.

```text
Evidence
Result SHA: abc123
AuthServiceTest.expiredToken: PASS
AuthServiceTest: 24 / 24 PASS
```

Task별 Evidence는 다르다.

Bug Fix:

```text
Result SHA
Changed Files
Unit Test Result
```

UI Task:

```text
E2E Result
Before / After Screenshot
Browser Video / Trace
```

Docker Build:

```text
Build Result
Image Digest
Build Log Reference
```

7장의 `Output / Evidence`에서 필요한 결과를 미리 정하고, 8장에서는 그 결과를 구조화한다.

---

## 7. UI 변경은 Demo Evidence를 먼저 볼 수 있다

UI 변경은 Diff만 읽어서는 실제 결과를 빠르게 판단하기 어렵다.

예를 들어 Layout과 CSS가 바뀌었다면 다음 순서를 사용할 수 있다.

```text
Build PASS
E2E PASS
Before Screenshot
After Screenshot
Browser Video
      ↓
필요한 경우 Diff Review
```

이를 이 책에서는 `Demos over Diffs` 관점으로 사용한다.

코드 Review를 생략한다는 의미는 아니다.

동작 결과를 먼저 확인하고 구현 세부를 검토하는 순서다.

CLI Output, API Response, Generated Report처럼 결과를 직접 확인할 수 있는 작업에도 같은 원칙을 적용할 수 있다.

---

## 8. Failure Fingerprint는 반복 실패를 구분한다

Agent가 수정한 뒤 Runner가 다시 실패했다고 하자.

Retry마다 Raw Log 전체를 비교하지 않고 안정적인 Failure 정보를 조합할 수 있다.

예:

```text
AuthServiceTest.expiredToken
JWTExpiredException
expected=401
actual=200
AuthServiceTest.java:94
```

Fingerprint 후보:

```text
Failing Test ID
Exception Type
Assertion Message
Error Code
Top Stack Frame
```

Timestamp, Random Port, Container ID처럼 실행마다 바뀌는 값은 제외한다.

```text
Retry #1 → Fingerprint A
Retry #2 → Fingerprint A
```

같은 Fingerprint가 반복되면 수정이 실패를 바꾸지 못한 것이다.

반대로 Failure가 바뀌었다면 진행 중일 수 있다.

```text
Retry #1 → Compilation Error
Retry #2 → Unit Test Failure
```

이 장에서는 반복 실패를 식별하는 방법까지만 다룬다. 언제 중단하고 Local로 되돌릴지는 17장에서 정리한다.

---

## 9. PASS에도 최소 Evidence를 남긴다

성공한 Task도 다음 정도의 결과는 남긴다.

```text
Task ID
Git SHA
Validation Command
Status
Duration
Artifact Reference if needed
```

예:

```text
Task: AUTH-142
Git SHA: def456
Validation: ./gradlew :auth:test
Status: PASS
Tests: 24 / 24
```

성공 결과까지 대형 로그를 LLM이 읽을 필요는 없지만 어떤 상태를 검증했는지는 추적할 수 있어야 한다.

---

## 10. AUTH-142 결과 흐름

7장에서 만든 AUTH-142 Task Contract를 그대로 사용한다.

```text
Task Contract
→ Cloud Agent Fix
→ Result SHA def456
       ↓
Cloud Runner
→ ./gradlew :auth:test
       ↓
Raw Result
├─ junit.xml
└─ build.log
       ↓
Result Filter
→ 24 / 24 PASS
       ↓
result.json
       ↓
Evidence
- Result SHA def456
- Test PASS
```

실패했다면 다음처럼 시작한다.

```text
result.json
→ AuthServiceTest.expiredToken FAIL
→ Failure Detail
→ 필요한 경우 Specific Log
```

입력과 출력이 대칭을 이룬다.

```text
7장
Task Contract
→ Small Input / Context
        ↓
Cloud Runner / Agent
        ↓
8장
Result Gateway / Evidence
→ Small Output / Tool Result
```

다음 장에서는 이 작업이 시작되기 전에 발생하는 환경 준비 비용을 줄인다.


# Part III. Cloud 실행환경과 검증을 설계한다

Cloud에 보낼 Task를 골랐다면 다음 문제는 실행 준비다.

Agent가 Session을 시작할 때마다 JDK, Browser, Dependency, DB Tool을 설치한다면 Task보다 환경 준비에 더 많은 시간이 들 수 있다. 반대로 준비된 환경이 있어도 모든 실패를 Agent에게 넘기면 결정론적으로 처리할 수 있는 Build와 Test까지 LLM 비용을 사용하게 된다.

이 Part에서는 Cloud Task가 빠르게 시작하고, 필요한 경우에만 Agent가 개입하도록 실행 경로를 만든다.

Prepared Environment와 Cache로 시작 비용을 줄이고, 정상 검증은 Cloud Runner가 처리한다. Source는 Branch나 Worktree로, Runtime은 Container나 VM으로, 결과는 Artifact 경로로 분리한다. 병렬화할 때는 Worker 수보다 독립 Task와 Fan-in 비용을 먼저 본다.

핵심 흐름은 다음과 같다.

```text
Prepared Environment
→ Runner-first
→ Source / Runtime / Evidence Isolation
→ Independent Task Fan-out
→ Fan-in
```

이 구조가 만들어지면 Cloud 작업은 독립적으로 실행될 수 있다. 다음 Part에서는 이 독립 Task를 기존 개발 Workflow와 연결한다.


# 9장. Prepared Cloud Environment, Cache, Snapshot

Cloud Agent가 Task를 받았다고 바로 코드 수정이 시작되는 것은 아니다.

실제 작업 전에는 환경 준비가 필요하다.

```text
Worker 생성
→ Repository Checkout
→ Runtime 확인
→ Dependency 준비
→ Docker / Browser 준비
→ First Command
→ Task 시작
```

이 구간이 길면 모델이 빨라도 전체 Task는 느리다.

따라서 9장의 대상은 Agent의 추론 속도가 아니라 **환경 준비 비용**이다.

핵심 원칙은 다음과 같다.

> Agent에게 개발환경을 설치하게 하지 말고 바로 작업 가능한 환경을 제공한다.

> Cloud 환경도 Source Code처럼 버전 관리하고 재현 가능하게 만든다.

> Cache는 재사용하되 Source와 Runtime State는 Fresh하게 유지한다.

---

## 1. Cold Start를 구성 요소로 나눈다

Cloud Task의 시작시간을 `Agent 응답 시작시간` 하나로 보면 병목을 찾기 어렵다.

다음처럼 분해해서 본다.

```text
Provisioning Time
Checkout Time
Dependency Restore Time
Environment Warm-up Time
First Command Time
```

설명용 예로 환경 준비가 8분이고 실제 수정이 3분이라면 모델만 바꿔서는 전체 시간이 크게 줄지 않는다.

먼저 어느 구간이 반복되는지 측정해야 한다.

---

## 2. Prepared Environment를 만든다

매 Task마다 JDK나 Browser를 설치하지 않는다.

기본 구조:

```text
Base Image
   ↓
Runtime 설치
   ↓
Development Tools 설치
   ↓
Dependency 준비
   ↓
Cache Warming
   ↓
Prepared Environment
```

Task 실행 시에는 준비된 환경에 Fresh Source를 올린다.

```text
Prepared Environment
        +
Fresh Repository / Branch
        ↓
Task
```

포함 후보:

```text
Java / Node
Gradle / Maven
Docker CLI
Playwright Browser
Static Analysis Tool
DB Client
기본 OS Package
```

제품별 Snapshot 기능이 없어도 Dockerfile, Dev Container, Bootstrap Script 같은 방식으로 같은 원칙을 적용할 수 있다.

---

## 3. Cloud Environment as Code

환경 구성을 사람의 기억에 맡기지 않는다.

예:

```text
cloud-env/
├─ Dockerfile
├─ versions.env
├─ install-tools.sh
└─ warm-cache.sh
```

환경 버전도 코드처럼 관리한다.

```text
cloud-env: backend-test-v12
java: 21
node: 22
playwright: pinned
```

핵심은 특정 제품의 환경 설정 UI가 아니다.

`어떤 Runtime과 Tool을 사용했는지 다시 만들 수 있는가`가 중요하다.

---

## 4. Task별 Environment를 분리한다

모든 Worker에 모든 Tool을 넣을 필요는 없다.

`campus-platform`에서는 다음 정도로 나눌 수 있다.

### backend-test

```text
Java 21
Gradle
Docker
Testcontainers
DB Client
```

### frontend-e2e

```text
Node
Playwright
Chrome
```

### migration-test

```text
Java 21
Migration Tool
PostgreSQL
DB Client
```

Cross-stack 문제처럼 필요한 경우에만 더 큰 `fullstack` 환경을 사용한다.

Routing은 단순하다.

```text
Backend Unit / Integration
→ backend-test

Frontend E2E
→ frontend-e2e

Migration Validation
→ migration-test
```

환경이 클수록 항상 좋은 것은 아니다. 사용하지 않는 Tool은 Image 크기와 준비 비용을 늘린다.

---

## 5. 재사용할 것과 초기화할 것을 분리한다

Prepared Environment에서 가장 중요한 경계다.

재사용하기 좋은 상태:

```text
Gradle Dependency Cache
Maven Repository
npm Cache
Docker Layer
Playwright Browser
Compiler Cache
```

매 Task마다 새로 만들어야 할 상태:

```text
Source Checkout
Branch
Task Input
DB State
Temporary File
Mutable Fixture
Test Output
Process / Session State
```

구조:

```text
Reusable
- Runtime
- Tools
- Dependency Cache
        +
Fresh
- Source
- Branch
- Task
- Runtime Data
        ↓
Execution
```

목표는 `빠른 시작`과 `깨끗한 실행`을 동시에 얻는 것이다.

---

## 6. Cache는 Invalidation까지 설계한다

Cache는 오래 남기는 것보다 언제 버릴지 정하는 것이 중요하다.

Cache Key 후보:

```text
OS / Architecture
JDK Version
Gradle / Maven Lock State
Package Manifest Hash
Dockerfile Hash
Tool Version
```

예:

```text
gradle-cache
key = os + jdk + dependency-lock-hash
```

잘못된 Cache는 다음 문제를 만든다.

```text
Stale Dependency
이전 Generated Code 잔존
다른 Branch 결과 혼입
Test Pollution
```

Source 상태에 강하게 의존하는 Build Output이나 Test Result를 무조건 재사용하지 않는다.

---

## 7. Snapshot은 시작 상태를 미리 준비한다

Snapshot은 Task 시작 전에 Runtime과 Tool이 준비된 상태를 저장하는 방법으로 볼 수 있다.

```text
Base Image
→ Tool 설치
→ Dependency Restore
→ Browser 설치
→ Snapshot
```

Task 시작:

```text
Snapshot
→ Fresh Source Checkout
→ Task Branch
→ Execute
```

책의 기본값은 다음과 같다.

```text
Environment / Dependency
→ 재사용

Source / Branch / Runtime State
→ Fresh
```

Source까지 Snapshot에 포함하면 최신 Commit과의 차이 적용 비용과 Stale Source 위험을 같이 고려해야 한다.

---

## 8. Warm Worker는 선택지다

짧고 반복되는 Task는 READY 상태 Worker를 재사용해 시작시간을 줄일 수 있다.

```text
READY
→ Task
→ Reset
→ READY
```

후보:

```text
Lint
Compile
Small Unit Test
PR Verification
```

반대로 장시간 작업이나 격리가 중요한 Task는 Ephemeral Worker가 더 단순할 수 있다.

```text
Task
→ New Worker
→ Execute
→ Artifact
→ Destroy
```

Warm Worker 자체를 기본값으로 두지 않는다. Startup Cost와 격리 요구를 보고 선택한다.

---

## 9. 반복되는 환경 실패는 Prompt 문제가 아니다

다음 상황을 보자.

```text
Agent
→ JDK 없음
→ 설치 방법 추론
→ 설치 실패
→ 다시 추론
```

이 문제를 Prompt에 설치 방법을 더 길게 적어서 해결하지 않는다.

권장 대응:

```text
Prepared Environment 수정
→ Java 21 포함
→ Environment Version 갱신
```

Playwright Browser가 반복해서 없다면 `frontend-e2e` Environment에 포함한다.

> 반복되는 환경 실패는 Agent의 Reasoning 문제가 아니라 Environment 문제로 취급한다.

이 원칙은 18장의 Harness Engineering과도 연결된다.

---

## 10. Secret은 Image에 넣지 않는다

Prepared Environment와 실행 시점 Credential을 분리한다.

```text
Prepared Environment
- Runtime
- Tools
- Cache

Execution-time Injection
- Secret
- Token
- Temporary Credential
```

Secret을 Image에 포함하지 않는다.

Cloud에서 어떤 Credential을 제공할 수 있는지는 조직 정책과 5장의 Routing 기준을 따른다.

---

## 11. Cold Start를 측정한다

다음 값을 기록할 수 있다.

```text
worker_provision_ms
checkout_ms
dependency_restore_ms
image_pull_ms
browser_ready_ms
first_command_ms
```

설명용 예:

```text
Before
Worker ready: 7m 40s

After
Worker ready: 55s
```

이 숫자는 제품 기준이 아니라 측정 방법을 설명하기 위한 예다.

실제 프로젝트에서는 반복 Task의 준비시간을 측정해 Environment 개선 효과를 확인한다.

---

## 12. campus-platform에서는 Environment 이름으로 Task를 연결한다

예:

```text
backend-test
→ Java 21 / Gradle / Docker / Testcontainers

frontend-e2e
→ Node / Playwright / Chrome

migration-test
→ Java 21 / Migration Tool / PostgreSQL
```

Task Contract에는 설치 절차 대신 Environment 이름만 넣는다.

```text
Task: AUTH-142
Environment: backend-test
```

그리고 실행 상태는 Fresh하게 시작한다.

```text
Fresh Git Checkout
Task Branch
Disposable Test DB
New Test Output Directory
```

Tibero, HSM, Internal Jenkins처럼 Cloud에서 제공하지 않는 내부 자원을 Prepared Environment로 억지로 복제하지 않는다. 그런 작업은 Hybrid로 남긴다.

---

## 13. 작은 입력과 작은 출력 다음에는 빠른 시작이 필요하다

7~9장의 흐름은 다음과 같다.

```text
7장
Task Contract
→ Small Input / Context
        ↓
8장
Result Gateway
→ Small Output / Evidence
        ↓
9장
Prepared Environment
→ Small Startup Overhead
```

이제 Task의 입력, 결과, 시작 환경이 정리됐다.

다음 장에서는 이 Prepared Environment에서 어떤 작업을 LLM이 아닌 Runner에게 먼저 맡길지 다룬다.

```text
Prepared Environment
        ↓
Cloud Runner
        ↓
PASS → 종료
FAIL + 판단 필요 → Cloud Agent
```

9장이 시작 비용을 줄이는 장이라면 10장은 **LLM이 필요한 실행 구간을 줄이는 장**이다.


# 10장. Cloud Agent를 Test Runner처럼 사용하기

Cloud 환경에서 실행되는 모든 작업에 LLM이 필요한 것은 아니다.

9장에서 Prepared Cloud Environment를 만들었다면 그 환경은 Agent 전용 공간이 아니라 반복 가능한 실행 노드가 된다.

Build, Test, E2E, Docker Build처럼 명령과 판정 기준이 정해진 작업은 먼저 Cloud Runner가 처리할 수 있다.

```text
Cloud Runner
→ 정해진 명령 실행
→ PASS / FAIL 판정

Cloud Agent
→ 실패 원인 분석
→ 코드 수정 판단
→ 변경 수행
```

이 장의 원칙은 간단하다.

> 정상 경로는 Runner가 처리하고, 판단이 필요한 예외 경로에서만 Agent를 호출한다.

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

---

## 1. Cloud Runner와 Cloud Agent를 분리한다

다음 명령은 LLM이 없어도 실행할 수 있다.

```bash
./gradlew test
```

필요한 것은 Repository와 Runtime, CPU / RAM, Test Tool이다.

반대로 다음 질문은 판단이 필요하다.

```text
AuthServiceTest.expiredToken이 왜 실패했는가?
어느 코드를 바꿔야 하는가?
```

따라서 한 Task 안에서도 실행 주체가 바뀐다.

```text
Cloud Runner
→ Test
→ FAIL

Cloud Agent
→ Analyze / Fix

Cloud Runner
→ Verification
```

Cloud Agent를 많이 호출하는 것이 목적이 아니다. Agent가 필요한 구간을 좁히는 것이 목적이다.

---

## 2. Deterministic First

개발 과정에는 프로그램으로 판정할 수 있는 검증이 많다.

```text
Compile
Unit Test
Integration Test
Lint
Architecture Rule
Migration Validation
Docker Build
E2E
```

이런 결과를 자연어 판단으로 대체하지 않는다.

```text
좋지 않은 방식
→ 코드를 보고 문제없어 보이는지 판단해줘

권장 방식
→ ./gradlew test
→ Exit Code / Report 확인
```

가능하면 완료 조건을 실행 가능한 명령으로 만든다.

```text
Formatting
→ Formatter / Linter

API Compatibility
→ Contract Test

Migration
→ Disposable DB
```

판단을 코드로 만들 수 있다면 그 판단을 반복해서 LLM에 맡기지 않는다.

---

## 3. 성공 경로에서는 Agent를 호출하지 않는다

설명용 예시로 100개의 검증 Task가 있다고 하자.

```text
100 Runner Execution
→ 95 PASS
→ 5 FAIL
```

95개 PASS 경로는 그대로 종료할 수 있다.

실패 5개도 모두 Agent 작업은 아니다.

```text
5 FAIL
├─ Infrastructure Failure
├─ Auto-fix 가능한 규칙 위반
└─ Code Reasoning이 필요한 Failure
```

Agent는 마지막 경우에만 필요하다.

```text
Cloud Runner
→ PASS → Done
→ FAIL
    ↓
Failure Classification
    ↓
Reasoning 필요?
    ├─ NO → Retry / Tool / Environment 처리
    └─ YES → Cloud Agent
```

Cloud Agent 비용을 줄이는 가장 단순한 방법 중 하나는 Agent 한 번의 Token을 조금 줄이는 것이 아니라 불필요한 Agent 호출 자체를 없애는 것이다.

---

## 4. Runner 입력과 출력도 고정한다

Runner는 단순 Shell 실행이지만 어떤 Source를 검증했는지 추적할 수 있어야 한다.

입력 예:

```text
Task ID
Git SHA
Environment
Command
Timeout
Artifact Rule
```

출력 예:

```text
Status
Exit Code
Duration
Result Summary
Artifact Reference
```

예:

```text
TASK=AUTH-142
SHA=def456
COMMAND=./gradlew :auth:test
STATUS=FAIL
EXIT=1
RESULT=artifacts/AUTH-142/result.json
```

8장에서 만든 작은 구조화 결과를 Runner의 출력 경계로 사용할 수 있다.

핵심은 Runner의 결과가 특정 Git SHA와 연결되어 있다는 점이다.

---

## 5. FAIL이라고 모두 코드 문제는 아니다

Runner가 실패하면 먼저 실패 종류를 구분한다.

```text
FAIL
 ↓
Classification
 ├─ INFRA_FAILURE
 ├─ BUILD_FAILURE
 ├─ TEST_FAILURE
 ├─ E2E_FAILURE
 └─ MIGRATION_FAILURE
```

예를 들어 다음은 인프라 문제다.

```text
Registry timeout
Worker disk full
Dependency mirror unavailable
Container start failure
```

이런 실패를 Agent에게 보내 코드 수정을 시키면 잘못된 변경이 생길 수 있다.

```text
INFRA_FAILURE
→ Retry / Environment 처리
```

반대로 다음은 코드 분석 후보가 된다.

```text
Compilation Error
Assertion Failure
NullPointerException
UI selector mismatch
Migration syntax error
```

그중 자동 수정 도구가 해결할 수 있는 것은 Tool을 먼저 사용한다.

---

## 6. Agent-on-failure

코드 판단이 필요한 실패가 남으면 Agent를 호출한다.

```text
Cloud Runner
  ↓
FAIL
  ↓
Failure Summary
  ↓
Cloud Agent
  ↓
Analyze / Fix
  ↓
Commit
  ↓
Cloud Runner
  ↓
Verification
```

AUTH-142를 예로 들면 Agent가 처음 받아야 할 정보는 대형 로그가 아니다.

```text
Test
AuthServiceTest.expiredToken

Expected
401

Actual
200

Relevant Files
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

필요한 경우에만 8장의 Result Gateway에서 상세 Artifact를 추가 조회한다.

10장의 핵심은 Failure Summary 형식을 다시 정의하는 것이 아니라 **Runner와 Agent 사이의 호출 경계**를 만드는 것이다.

---

## 7. Agent가 수정한 뒤에는 Runner가 다시 판정한다

다음 흐름으로 끝내지 않는다.

```text
Cloud Agent
→ Code Fix
→ "수정 완료"
→ PR
```

수정 후 같은 검증을 다시 실행한다.

```text
Agent Fix
→ Cloud Runner
→ PASS / FAIL
```

가능하면 처음 실패한 명령부터 실행한다.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

Target Test가 PASS하면 필요한 범위까지 회귀 검증을 넓힌다.

```text
Target Test
→ Related Test
→ Module Test
→ 필요한 Full Verification
```

Agent의 설명과 실행 사실을 분리하는 과정이다.

---

## 8. Retry 판단도 Runner 결과를 기준으로 한다

수정 후 다시 실패하면 8장의 Failure Fingerprint를 사용한다.

```text
Agent Fix
→ Cloud Runner
→ FAIL
→ Failure Fingerprint
```

실패가 바뀌었다면 진행 중일 수 있다.

```text
Retry #1
Compilation Error

Retry #2
Assertion Failure
```

반대로 동일 Failure Fingerprint가 반복된다면 같은 시도를 계속할 이유가 줄어든다.

```text
Retry #1 → Fingerprint A
Retry #2 → Fingerprint A
```

구체적인 중단과 Local Fallback 기준은 17장에서 다룬다.

여기서는 Runner 결과가 다음 행동을 결정하는 Evidence가 된다는 점만 유지한다.

---

## 9. 독립 검증은 병렬 Runner로 실행할 수 있다

하나의 Git SHA에 여러 검증이 필요할 수 있다.

```text
Git SHA: def456
        ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit   Integration E2E      Docker
|         |         |         |
+---------+---------+---------+---------+
          ↓
        Fan-in
```

이때 모든 결과는 같은 Source 상태를 검증해야 한다.

```text
Unit        → def456
Integration → def456
E2E         → def456
Docker      → def456
```

서로 다른 SHA의 결과를 하나의 PASS 묶음으로 취급하면 안 된다.

병렬화의 중복 Context, Review, Merge 비용은 12장에서 다룬다. 10장에서는 먼저 **결정론적 검증을 여러 Runner로 분리할 수 있다**는 점만 사용한다.

---

## 10. Sharding은 실행시간과 시작 비용을 함께 본다

큰 Test Suite는 여러 Runner로 나눌 수 있다.

```text
Test Suite
→ Shard A
→ Shard B
→ Shard C
→ Shard D
```

적합한 조건:

```text
Shard 간 State 공유가 적음
각 결과를 합산 가능
실패 Test 식별 가능
같은 Git SHA 사용
```

하지만 Worker Provisioning과 Environment Restore에도 비용이 있다.

짧은 테스트를 너무 잘게 나누면 시작 비용이 실행시간보다 커질 수 있다.

따라서 Shard 수는 고정 규칙이 아니라 프로젝트의 실제 실행시간과 준비시간을 보고 정한다.

---

## 11. campus-platform에 적용하면

출결 인증 변경 Commit이 있다고 하자.

```text
Git SHA: def456
```

Cloud Runner가 먼저 검증한다.

```text
Unit        PASS
Integration FAIL
Docker      PASS
E2E         PASS
```

Integration Failure가 인프라 문제가 아니라 재현 가능한 Code Failure라면 Agent가 해당 실패만 분석한다.

```text
Integration Runner
→ FAIL
→ Failure Summary
→ Cloud Agent
→ Fix
→ Result SHA ghi789
→ Integration Runner
→ PASS
```

Result SHA가 바뀌었으므로 필요한 검증은 `ghi789` 기준으로 다시 맞춘다.

이 구조에서 Agent는 전체 검증 파이프라인을 대신하는 존재가 아니다.

판단과 변경이 필요한 지점에 들어가는 Worker다.

---

## 12. 10장에서 기억할 실행 규칙

```text
Prepared Environment
        ↓
Cloud Runner
  ├─ PASS → Evidence → Done
  └─ FAIL
       ↓
   Classification
       ↓
   Tool / Retry / Cloud Agent
       ↓
   수정이 있으면 Cloud Runner 재검증
```

정리하면 다음 세 문장으로 충분하다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> FAIL이라고 모두 Agent를 호출하지 않는다.

> Agent가 수정했으면 다시 Runner가 검증한다.

다음 장에서는 여러 Runner와 Agent가 동시에 작업할 때 Source와 Runtime이 서로 섞이지 않도록 Git, Branch, Worktree, Container를 이용해 작업을 격리한다.


# 11장. Git, Branch, Worktree, Container로 작업 격리하기

Cloud Worker를 여러 개 사용하려면 먼저 작업 상태를 분리해야 한다.

Agent 수만 늘리고 같은 Working Directory, 같은 Git Index, 같은 DB와 임시 경로를 공유하면 병렬화가 아니라 충돌을 만든다.

Cloud Task의 격리는 크게 두 층으로 나눈다.

```text
Source Isolation
→ Git Branch / Worktree / Independent Clone

Runtime Isolation
→ Container / VM / Process / Port / DB / Artifact Path
```

이 둘은 다른 문제를 해결한다.

> Branch는 Source 변경을 격리하고, Container나 VM은 실행 상태를 격리한다.

> Git은 Local과 Cloud 사이의 Handoff Boundary이자 작업 상태의 기준점이다.

---

## 1. Cloud Task는 명시적인 Git 상태에서 시작한다

Local 개발자의 IDE에는 Git에 없는 상태가 많다.

```text
modified files
untracked files
local config
local DB
running process
IDE state
```

Cloud Worker는 이 상태를 자동으로 알지 못한다.

따라서 시작점을 Git으로 고정한다.

```text
Repository
campus-platform

Base SHA
abc123

Task Branch
agent/auth-expired-token-142
```

기본 Handoff는 다음처럼 본다.

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Checkout
→ Task Branch
→ Work / Test
→ Commit / Push
→ PR
```

Git은 다음 질문에 답하는 기준점이 된다.

```text
어떤 코드에서 시작했는가?
무엇이 변경됐는가?
어떤 Commit이 검증됐는가?
어떤 결과가 어느 Source와 연결되는가?
```

---

## 2. Base SHA를 고정한다

Branch 이름만 기록하면 Task가 실제 어느 시점에서 시작했는지 불명확할 수 있다.

```text
Task 시작
main = abc123

이후
main = xyz999
```

따라서 Task 상태에 Base SHA를 남긴다.

```text
Task: AUTH-142
Base SHA: abc123
Branch: agent/auth-expired-token-142
```

Agent가 수정 결과를 만들면 검증 결과도 그 Result SHA와 연결한다.

```text
Base SHA abc123
   ↓
Result SHA def456
   ↓
Runner Verification def456
   ↓
PR
```

다음 결과는 Evidence가 아니다.

```text
Result SHA: def456
Validation Result: abc123 기준 PASS
```

> Source 상태와 Evidence는 같은 Commit 기준으로 연결한다.

---

## 3. Branch per Task

독립 Cloud Task는 가능한 한 독립 Branch와 연결한다.

```text
AUTH-142
→ agent/auth-expired-token-142

ATTEND-211
→ agent/attendance-retry-211
```

Branch 이름 규칙 자체보다 다음 대응관계가 중요하다.

```text
Task
↕
Branch
↕
Result SHA
↕
Validation Result
↕
PR
```

이 구조가 있으면 대화 기록을 뒤지지 않고 Git 상태로 작업을 추적할 수 있다.

Task 취소나 Retry도 특정 Branch 기준으로 처리하기 쉽다.

---

## 4. Task 상태를 Git과 연결한다

여러 Worker를 운영할 때 최소 상태는 다음 정도면 된다.

```text
Task ID
Session ID
Base SHA
Branch
Result SHA
Status
Validation Result
Artifact Path
PR
```

예:

```yaml
task_id: AUTH-142
session_id: cloud-142
base_sha: abc123
branch: agent/auth-expired-token-142
result_sha: def456
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/def456/
pr: 142
```

상태를 거대한 Agent Platform UI로 관리할 필요는 없다.

핵심은 `Task → Git SHA → Evidence → PR` 연결이 깨지지 않는 것이다.

---

## 5. Local에서는 Worktree로 Source를 나눌 수 있다

Local에서 여러 Branch를 동시에 열어야 한다면 Git Worktree가 유용하다.

```text
worktrees/
├─ auth-task/
├─ attendance-task/
└─ notification-task/
```

각 Worktree는 별도 Working Directory와 Branch를 가진다.

```text
auth-task
→ agent/auth

attendance-task
→ agent/attendance
```

하지만 Worktree는 Source 격리 수단이다.

두 Worktree가 여전히 다음 자원을 공유할 수 있다.

```text
localhost:8080
local PostgreSQL
/tmp/campus-test
same Docker network
same artifact folder
```

따라서 Worktree를 만들었다고 Runtime까지 격리됐다고 보지 않는다.

---

## 6. Cloud에서는 독립 Workspace와 Runtime이 단순하다

Cloud Worker는 Task마다 독립 Workspace를 주는 편이 관리하기 쉽다.

```text
Worker A
├─ clone A
├─ branch A
├─ runtime A
├─ temp A
└─ artifacts A

Worker B
├─ clone B
├─ branch B
├─ runtime B
├─ temp B
└─ artifacts B
```

격리 대상은 Source만이 아니다.

```text
Working Directory
Git Index
Process
Port
Temp Directory
Disposable DB
Test Output
Artifact Path
```

Integration Test가 PostgreSQL을 띄운다면 DB 상태도 Worker별로 분리한다.

```text
Worker A → postgres-A
Worker B → postgres-B
```

한 Task의 테스트 데이터가 다른 Task의 결과에 영향을 주지 않게 하는 것이 목적이다.

---

## 7. Branch와 Container는 다른 문제를 해결한다

다음 구분을 유지한다.

```text
Branch / Worktree / Clone
→ Source Isolation

Container / VM / Process Boundary
→ Runtime Isolation
```

Branch가 달라도 같은 DB를 쓰면 Runtime 충돌이 생길 수 있다.

```text
Branch A
Branch B
   ↓
same DB
```

반대로 Container가 달라도 두 Agent가 같은 핵심 파일을 바꾸면 Merge 시 충돌한다.

```text
Container A → UserService.java 수정
Container B → UserService.java 수정
```

따라서 Task를 병렬화하기 전에 두 질문을 모두 확인한다.

```text
Source 상태가 분리됐는가?
Runtime 상태가 분리됐는가?
```

---

## 8. Artifact 경로도 Task별로 분리한다

여러 Runner가 같은 경로에 결과를 쓰면 Evidence가 섞일 수 있다.

좋지 않은 구조:

```text
artifacts/result.json
artifacts/build.log
```

권장:

```text
artifacts/AUTH-142/def456/
├─ result.json
├─ junit.xml
└─ build.log
```

다른 Task는 별도 경로를 사용한다.

```text
artifacts/ATTEND-211/xyz789/
```

Artifact 내부에도 Git SHA를 기록하면 8장의 Evidence가 어느 Source 상태의 결과인지 추적하기 쉽다.

---

## 9. 물리적 격리가 논리적 충돌을 없애지는 않는다

완전히 다른 Container에서 작업해도 같은 파일을 바꾸면 통합 시 충돌할 수 있다.

```text
Agent A → UserService 예외 구조 변경
Agent B → UserService 성능 수정
Agent C → UserService logging 변경
```

실행 중에는 서로 방해하지 않아도 Fan-in 시점에는 다음 문제가 생길 수 있다.

```text
Merge Conflict
Semantic Conflict
Regression
```

따라서 병렬화 전에 예상 변경 영역을 본다.

```text
Expected Files
Shared Module
Public API
Common DTO
Central Config
Migration / DB Schema
```

여기서 병렬화의 비용을 계산하지는 않는다. 그 문제는 12장에서 다룬다.

11장의 역할은 **어떤 상태를 반드시 분리해야 하는가**를 정하는 것이다.

---

## 10. Migration과 DB Schema는 별도 경계가 필요하다

DB Migration은 일반 Source 파일보다 순서 의존성이 크다.

예:

```text
Task A
→ V142__add_student_index.sql

Task B
→ V142__add_attendance_status.sql
```

각 Branch에서는 문제가 없어도 Merge 시 번호가 충돌한다.

번호가 달라도 적용 순서가 중요할 수 있다.

```text
V142 → column 생성
V143 → index 생성
```

따라서 Migration Task에서는 다음을 별도로 본다.

```text
번호 정책
적용 순서
DB Schema dependency
통합 Migration Validation
```

Container를 나눈다고 DB Schema 변경의 논리적 의존성이 사라지는 것은 아니다.

---

## 11. Shared/Common Module은 선행 Task가 될 수 있다

처음에는 독립적으로 보이는 Task가 공통 모듈 때문에 연결될 수 있다.

```text
Task A → attendance
Task B → notification
Task C → student
```

세 Task가 모두 `common` DTO 변경을 요구한다면 실제 의존 관계는 다음과 같다.

```text
Task A ─┐
Task B ─┼→ common DTO
Task C ─┘
```

이 경우 공통 변경을 먼저 분리할 수 있다.

```text
Task 0
common DTO 변경
   ↓
새 Base SHA
   ↓
Task A / B / C
```

이 구조는 12장의 Dependency-aware Parallel Group으로 이어진다.

---

## 12. Multi-Repository도 필요한 범위만 연결한다

하나의 기능이 여러 Repository에 걸쳐 있을 수 있다.

```text
campus-api
campus-admin
campus-common
```

현재 Task에서 함께 수정될 가능성이 높은 Repository만 Workspace에 제공한다.

무관한 Repository까지 연결하면 다음 비용이 생긴다.

```text
Checkout 증가
검색 범위 증가
Context 증가
잘못된 변경 가능성 증가
```

Multi-Repository Task에서는 각 Repository의 기준점도 고정한다.

```text
campus-api    → Base SHA aaa111
campus-common → Base SHA bbb222
```

Git Handoff가 여러 Repository로 늘어났을 뿐 원칙은 같다.

---

## 13. campus-platform 격리 예

AUTH-142와 ATTEND-211을 동시에 진행한다고 하자.

```text
AUTH-142
Branch: agent/auth-expired-token-142
Result SHA: def456
Runtime: backend-test-A
Artifact Path: artifacts/AUTH-142/def456/

ATTEND-211
Branch: agent/attendance-retry-211
Result SHA: xyz789
Runtime: backend-test-B
Artifact Path: artifacts/ATTEND-211/xyz789/
```

각 Task는 Source, Runtime, Artifact를 분리한다.

그러나 두 Task가 같은 `common-auth`를 수정해야 한다면 병렬 실행 가능 여부를 다시 판단한다.

격리는 병렬화를 가능하게 만드는 조건이지, 모든 Task를 병렬로 만들어주는 장치가 아니다.

---

## 14. 11장에서 기억할 경계

Cloud Task를 격리할 때 다음 세 상태를 확인한다.

```text
Source
→ Branch / Worktree / Clone

Runtime
→ Process / Container / VM / DB / Port

Evidence
→ Task ID / Git SHA별 Artifact Path
```

그리고 논리적 충돌은 별도로 확인한다.

```text
same file
shared module
public API
migration
DB Schema
```

다음 장에서는 이렇게 격리된 Task 중 실제로 무엇을 병렬화할지, Agent 수를 늘릴수록 어떤 중복 Context와 Review/Fan-in 비용이 생기는지 살펴본다.


# 12장. 병렬 Worker와 중복 Context 비용

Cloud Worker를 여러 개 띄우면 여러 작업을 동시에 처리할 수 있다.

하지만 Agent 수를 늘린다고 개발 속도가 같은 비율로 증가하지는 않는다.

병렬화에는 반대쪽 비용이 있다.

```text
Startup Overhead
Context Duplication
Merge Cost
Review Cost
Result Integration Cost
Coordination Cost
```

따라서 병렬화의 핵심 질문은 `몇 개의 Agent를 띄울 것인가`가 아니다.

> 서로 독립적으로 실행하고 검증할 수 있는 Task가 몇 개인가?

11장이 병렬화를 위한 격리 조건을 만들었다면, 12장은 **어디까지 병렬화하는 것이 실제로 이득인가**를 판단한다.

---

## 1. 먼저 Parallel Compute와 Parallel Reasoning을 구분한다

하나의 Git SHA에서 다음 검증이 필요하다고 하자.

```text
Unit Test
Integration Test
E2E
Docker Build
```

이 작업은 대부분 Cloud Runner로 병렬화할 수 있다.

```text
Git SHA
  ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit    Integration E2E      Docker
|         |         |         |
+---------+---------+---------+---------+
          ↓
        Fan-in
```

이것이 `Parallel Compute`다.

반면 다음은 다르다.

```text
Agent A → Repository 분석
Agent B → Repository 분석
Agent C → Repository 분석
```

같은 문제에 여러 LLM이 각각 Context를 읽고 판단한다. 이것은 `Parallel Reasoning`이다.

기본값은 먼저 Compute를 병렬화하고, 판단이 실제로 필요한 Task에만 Agent를 붙이는 것이다.

---

## 2. Fan-out 전에 Dependency를 확인한다

Task가 여러 개 있다고 모두 동시에 시작하지 않는다.

```text
Task A → common DTO 변경
Task B → API 변경
Task C → UI 변경
```

의존 관계가 다음이라면:

```text
A → B → C
```

세 Worker를 동시에 시작해도 B와 C는 다시 작업해야 할 가능성이 높다.

반대로 다음처럼 독립성이 높으면 Fan-out하기 쉽다.

```text
Task A → attendance module
Task B → notification module
Task C → library module
```

판단 기준:

```text
File Scope 분리
낮은 Dependency
독립 Validation
낮은 Merge 순서 의존성
```

병렬화는 Task Dependency를 먼저 본 뒤 시작한다.

---

## 3. 가장 먼저 병렬화하기 좋은 것은 Read-only 검증이다

Source를 수정하지 않는 검증은 Merge Conflict가 없다.

```text
Unit Test
Integration Test
E2E
Docker Build
Static Analysis
```

같은 SHA를 읽고 서로 다른 결과를 만든다.

```text
abc123
├─ Unit
├─ Integration
├─ E2E
└─ Docker
```

결과는 마지막에 Fan-in하면 된다.

여러 Agent가 동시에 코드를 수정하는 것보다 여러 Cloud Runner가 검증을 병렬 실행하는 것이 더 단순한 시작점이다.

Cloud 병렬화의 효과를 확인할 때도 이 경로부터 측정하는 편이 좋다.

---

## 4. Context Duplication은 숨은 비용이다

Agent가 여러 개면 각 Agent가 자신의 Context를 읽는다.

좋지 않은 구조:

```text
Agent A → Repository 전체 탐색
Agent B → Repository 전체 탐색
Agent C → Repository 전체 탐색
Agent D → Repository 전체 탐색
```

같은 README, Architecture 문서, 공통 Source를 반복해서 읽을 수 있다.

병렬 실행시간은 줄어도 Token과 탐색시간은 중복된다.

권장 구조:

```text
Agent A → attendance 관련 Context
Agent B → notification 관련 Context
Agent C → admin UI 관련 Context
```

7장의 Task Contract가 병렬화에서도 중요하다.

Task별 Context가 작아질수록 Agent 수 증가에 따른 중복 비용도 작아진다.

---

## 5. 코드 변경 병렬화는 Change Locality를 본다

다음은 병렬화하기 쉽다.

```text
Agent A → attendance module
Agent B → notification module
Agent C → library module
```

각 Module이 별도 Test를 가지고 공통 변경이 적다면 Fan-out하기 좋다.

반대로 다음 구조는 병렬성이 낮다.

```text
Agent A → common-auth
Agent B → common-auth
Agent C → common-auth
```

실행 중에는 서로 다른 Container에서 성공하더라도 통합 시 비용이 커진다.

```text
Parallel Execution
→ 빠름

Fan-in
→ Conflict / Review / Rework 증가
```

따라서 병렬 코딩에서는 Agent 수보다 `어디를 바꾸는가`를 먼저 본다.

---

## 6. Agent Count는 Task Count와 다르다

설명용 예로 Task가 10개 있다고 해서 Agent 10개를 즉시 시작할 필요는 없다.

실제 독립성이 세 개뿐이라면 다음처럼 그룹을 만들 수 있다.

```text
Task Queue: 10
   ↓
Dependency 확인
   ↓
Parallel Group A: 3
   ↓
Fan-in
   ↓
Parallel Group B: 3
```

병렬도에 영향을 주는 것은 Worker 한도만이 아니다.

```text
Task Dependency
Startup Overhead
Context Duplication
Compute Quota
Review Capacity
Merge Risk
```

Agent를 만들 수 있는 최대 수가 팀이 사용해야 할 병렬도는 아니다.

---

## 7. Fan-in이 병목이 될 수 있다

병렬화는 Worker가 작업을 시작하는 Fan-out으로 끝나지 않는다.

```text
PR A
PR B
PR C
PR D
  ↓
Fan-in
  ↓
Integration Validation
```

Fan-in에서는 다음 비용이 발생한다.

```text
PR Review
Conflict Resolution
Regression Test
Architecture Consistency 확인
Migration Ordering
Rework
```

Cloud Worker가 여러 PR을 빠르게 만들어도 Review와 Integration이 따라가지 못하면 전체 Lead Time은 줄지 않는다.

> 병렬화를 설계할 때 Fan-out만큼 Fan-in 비용을 본다.

---

## 8. Review Capacity가 병렬도의 상한이 된다

Agent Throughput이 Reviewer 처리량보다 크면 Queue가 쌓인다.

설명용 예:

```text
Agent가 생성 가능한 PR
20 / day

팀이 검토 가능한 PR
5 / day
```

이 숫자는 처리량 관계를 설명하기 위한 예시이며 실제 기준값이 아니다.

이 경우 병목은 Agent가 아니라 Review다.

볼 수 있는 지표:

```text
Review Queue Time
Review Duration
Merge Conflict Rate
Rework Rate
Integration Failure Rate
```

Cloud Agent 생산성을 `몇 개의 PR을 만들었는가`로만 평가하면 안 되는 이유다.

실제로 중요한 것은 변경이 **통합 가능한 상태**까지 얼마나 빨리 도달하는가다.

---

## 9. Startup Overhead도 병렬 수만큼 반복될 수 있다

Worker마다 다음 비용이 생길 수 있다.

```text
Provisioning
Repository Checkout
Dependency Restore
Image Pull
Browser 준비
```

9장의 Prepared Environment와 Cache는 이 비용을 줄인다.

```text
Reusable
→ Runtime / Tool / Dependency Cache

Fresh per Worker
→ Source / Branch / DB State / Temp / Test Output
```

병렬 Worker를 늘릴수록 Environment 준비 비용도 같이 늘 수 있으므로, 짧은 Task를 지나치게 잘게 쪼개지 않는다.

---

## 10. 병렬화 이득은 Worker 수로 계산하지 않는다

개념적으로 다음처럼 볼 수 있다.

```text
Parallel Benefit
≈ Saved Execution Time
- Startup Overhead
- Context Duplication Cost
- Merge Cost
- Review Cost
- Coordination Cost
```

정확한 수식을 만들려는 목적은 아니다.

측정해야 할 항목을 놓치지 않기 위한 개념 모델이다.

예를 들어 Worker를 2개에서 8개로 늘렸을 때 다음 현상이 동시에 나타날 수 있다.

```text
Execution Time 감소
Review Queue 증가
Merge Conflict 증가
```

이 숫자 역시 설명용 예다.

전체 Lead Time은 오히려 비슷하거나 길어질 수 있다.

병렬도는 실제 결과를 보고 조정한다.

---

## 11. Dependency-aware Parallel Group

Task Dependency를 명시하면 병렬 그룹을 만들 수 있다.

```text
Group 1
common-dto
   ↓
Group 2
├─ attendance-api
├─ notification-api
└─ library-api
   ↓
Group 3
admin-ui
```

같은 Group에는 서로 결과를 기다리지 않아도 되는 Task만 둔다.

이 장의 목적은 범용 Scheduler를 구현하는 것이 아니다.

Task를 병렬화할 때 `동시에 실행 가능한가`를 Agent 수가 아니라 Dependency로 판단하는 원칙을 세우는 것이다.

---

## 12. Java 21 Migration 예

세 서비스가 있다고 하자.

```text
service-a
service-b
service-c
```

각 서비스가 독립 Build를 가진다면 Java 21 대응을 병렬로 진행할 수 있다.

```text
+-----------+-----------+-----------+
|           |           |           |
service-a service-b   service-c
|           |           |           |
PR A      PR B        PR C
 \          |          /
        Fan-in
          ↓
    Full Validation
```

그러나 세 서비스가 같은 `common-build-plugin` 변경을 필요로 한다면 선행 Task를 만든다.

```text
Task 0
common-build-plugin Java 21 대응
   ↓
새 Base SHA
   ↓
service-a / b / c 병렬 실행
```

Dependency를 무시하고 동시에 시작하면 여러 Agent가 같은 공통 문제를 반복 해결할 수 있다.

---

## 13. Best-of-N은 일반 병렬화와 다르다

일반 병렬화는 서로 다른 Task를 나눈다.

```text
Task A → Worker A
Task B → Worker B
Task C → Worker C
```

Best-of-N은 같은 어려운 문제를 여러 Agent가 각각 푼다.

```text
같은 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
```

같은 Context와 추론 비용이 N번 반복되므로 기본값은 `N=1`로 둔다.

검토할 수 있는 조건:

```text
어려운 Bug
해결 실패 비용이 큼
후보를 자동 검증 가능
각 후보를 독립 Branch에서 실행 가능
```

후보 선택도 가능한 한 Cloud Runner의 결정론적 검증을 먼저 사용한다.

Best-of-N은 일반적인 병렬 전략이 아니라 제한된 고급 기법이다.

---

## 14. 12장에서 기억할 판단

병렬화 전에 다음 순서로 본다.

```text
1. Task가 독립적인가?
2. Read-only 검증부터 병렬화할 수 있는가?
3. 코드 변경 영역이 겹치지 않는가?
4. 각 Agent의 Context가 작게 유지되는가?
5. Startup Overhead가 Task보다 크지 않은가?
6. Review / Merge / Integration을 감당할 수 있는가?
```

정리하면 다음과 같다.

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

> Agent 수를 늘린다고 생산성이 선형 증가하지 않는다.

> Fan-out만큼 Fan-in 비용도 설계해야 한다.

다음 장에서는 이렇게 나눈 Task를 Local에서 Cloud로 넘기고, Evidence와 PR을 다시 Local로 가져오는 Handoff 흐름을 다룬다.


# Part IV. Local과 Cloud를 연결한다

독립 실행환경에서 Task를 잘 처리하는 것만으로는 실제 개발 Workflow가 완성되지 않는다.

개발자는 Local에서 요구사항을 좁히고 Architecture와 내부 제약을 판단한다. Cloud에서는 독립 작업과 검증을 수행한다. 결과는 다시 Local로 돌아와 Review, Internal Validation, Merge를 거친다.

이 Part에서는 이 이동 경계를 다룬다.

사람이 직접 Task를 정리해 Cloud로 보내는 경우에는 `Git + Task Contract`가 입력 경계가 된다. Cloud에서 돌아오는 결과는 `Result SHA + Evidence + PR`로 연결한다.

같은 구조는 CI Failure, Review Comment, Schedule 같은 Event에도 적용할 수 있다. Event 자체가 Agent 호출 명령은 아니다. 먼저 Task Candidate로 만들고, 중복을 제거하고, Runner나 Tool로 끝낼 수 있는지 확인한 뒤 판단이 필요한 경우에만 Agent를 호출한다.

```text
Human-driven
Developer → Task Contract → Cloud

Event-driven
CI / Review / Schedule → Task Candidate → Cloud
```

두 경로의 공통점은 같다.

> Git으로 작업 상태를 넘기고 Evidence로 결과를 돌려받는다.

다음 Part에서는 이 구조를 하나의 실제 프로젝트 Workflow로 합친다.


# 13장. Local → Cloud → Local Handoff

Cloud Agent를 실제 개발에 넣으면 하나의 Task가 Local 또는 Cloud 한쪽에만 머물지 않는다.

Local에서 문제를 좁히고, Cloud에서 독립 작업과 검증을 수행한 뒤, 다시 Local에서 내부 환경을 확인하고 통합할 수 있다.

이 장의 대상은 실행 위치 자체가 아니라 **실행 위치 사이에서 작업 상태를 안전하게 넘기는 방법**이다.

기본 구조는 다음과 같다.

```text
Local
→ Task 정리
→ Git + Task Contract
      ↓
Cloud
→ Work / Runner / Agent
→ Evidence + Result SHA / PR
      ↓
Local
→ Review / Internal Validation / Merge
```

핵심 경계는 두 개다.

```text
Local → Cloud
Git + Task Contract

Cloud → Local
Evidence + Result SHA / PR
```

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

---

## 1. Handoff는 대화가 아니라 상태 전달이다

다음 지시는 Handoff 기준이 없다.

```text
내 Mac에서 하던 인증 수정 이어서 해줘.
```

Cloud Worker는 Local IDE의 미커밋 파일, 임시 설정, 로컬 DB 상태를 자동으로 알지 못한다.

Handoff에는 최소한 다음 질문의 답이 있어야 한다.

```text
어느 Source 상태에서 시작하는가?
무엇을 해야 하는가?
어디까지 바꿀 수 있는가?
무엇이 성공인가?
어떤 결과를 반환해야 하는가?
```

즉 Handoff는 자연어 대화를 옮기는 것이 아니라 **재현 가능한 작업 상태를 전달하는 과정**이다.

---

## 2. Local → Cloud 입력은 작게 고정한다

7장의 Task Contract를 Handoff 입력 형식으로 사용한다.

예:

```text
Task ID
AUTH-142

Base SHA
abc123

Goal
Expired JWT → HTTP 401

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Environment
backend-test

Expected Evidence
- Result SHA
- Changed Files
- Validation Result
```

Local에서 알고 있는 모든 프로젝트 정보를 넣는 것이 목적이 아니다.

Cloud Worker가 독립적으로 시작할 수 있는 최소 상태를 고정한다.

```text
Local Context
→ Task에 필요한 부분만 추출
→ Cloud Handoff
```

---

## 3. Git이 Source Handoff Boundary가 된다

Cloud Worker가 어느 코드에서 작업했는지 명확해야 한다.

기본 흐름은 다음과 같다.

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Checkout Base SHA
→ Task Branch
→ Work
→ Commit
```

예:

```text
Repository: campus-platform
Base SHA: abc123
Branch: agent/auth-142
```

결과도 SHA로 연결한다.

```text
Base SHA
abc123
   ↓
Result SHA
def456
   ↓
Verification SHA
def456
```

중요한 것은 Branch 이름보다 **실제로 검증한 Commit을 식별할 수 있는가**다.

11장의 Source Isolation이 여기서는 Local과 Cloud 사이의 작업 전달 경계가 된다.

---

## 4. 미커밋 Local State는 먼저 정리한다

Local에 다음 상태가 있다고 하자.

```text
modified: AuthService.java
untracked: debug.yml
local DB state changed
```

이 상태가 Task에 필요하다면 Cloud에 넘길 형태로 만들어야 한다.

선택지는 단순하다.

```text
필요한 변경을 Commit / Push
또는
현재 Task를 Local에 유지
```

Cloud에 보내기 위해 모든 실험 상태를 억지로 Commit할 필요는 없다.

아직 요구사항과 변경 범위가 정리되지 않았다면 Local에서 계속 좁힌 뒤 Handoff한다.

> Cloud Handoff가 어렵다는 것은 때로 Task가 아직 독립 실행 가능한 상태가 아니라는 신호다.

---

## 5. Cloud에서는 중간 확인보다 독립 실행을 우선한다

Handoff가 끝난 뒤 Cloud에서는 9~12장에서 만든 구조를 사용한다.

```text
Prepared Environment
      ↓
Runner-first
      ↓
필요한 경우 Cloud Agent
      ↓
Verification
      ↓
Evidence
```

예를 들어 같은 SHA에서 다음 검증을 병렬로 수행할 수 있다.

```text
Unit Test
Integration Test
Docker Build
E2E
```

Cloud Task를 위임한 뒤 개발자가 계속 상태를 확인하고 매 단계마다 방향을 정한다면 비동기 Handoff의 이점이 줄어든다.

따라서 Cloud로 넘기는 Task는 가능한 한 다음 조건을 가진다.

```text
완료 조건 명확
중간 Human Steering 적음
독립 검증 가능
결과를 Evidence로 반환 가능
```

---

## 6. Cloud → Local Return Boundary는 Evidence다

Cloud Worker의 자연어 응답만으로 작업 완료를 판단하지 않는다.

Local이 받아야 할 것은 검증 가능한 결과다.

예:

```text
Task: AUTH-142
Result SHA: def456

Changed Files
- AuthService.java
- AuthServiceTest.java

Validation Result
AuthServiceTest.expiredToken: PASS
AuthServiceTest: 24 / 24 PASS

Artifacts
- result.json
- junit.xml
```

Local에서는 다음 순서로 확인할 수 있다.

```text
Evidence
→ Changed Files
→ 필요한 Diff
→ Internal Validation
```

8장의 Result Gateway와 Evidence가 Handoff의 반환 인터페이스가 된다.

---

## 7. PR은 Review 가능한 Return Package다

팀 개발에서는 Result SHA와 Evidence를 PR로 묶을 수 있다.

```text
Cloud Worker
→ Result SHA
→ Push
→ PR
      ↓
Developer / Reviewer
```

PR에는 다음 관계가 연결되어야 한다.

```text
Task ID
Base SHA
Result SHA
Validation Result
Artifact Reference
```

이 책의 기본값은 자동 Merge가 아니다.

```text
Cloud
→ 검증 가능한 변경 반환

Local / Human
→ Review / Integration / Merge 판단
```

Cloud Agent가 코드를 만들었다는 사실과 main에 반영해도 된다는 판단은 분리한다.

---

## 8. 내부망 검증은 마지막 경계로 남길 수 있다

기업 프로젝트에서는 Cloud에서 직접 재현하기 어려운 자원이 있다.

```text
Tibero / Oracle
HSM
Internal Jenkins
VPN-only API
사내 시스템
```

이 때문에 전체 Task를 Local에서만 처리할 필요는 없다.

예를 들어 Migration은 다음처럼 나눌 수 있다.

```text
Cloud
→ Migration 일반 검증
→ Disposable DB
→ Integration Test
→ Evidence
      ↓
Local / Internal
→ Tibero 실제 적용 검증
```

HSM 연동도 같은 원칙을 사용할 수 있다.

```text
Cloud
→ Pure Logic / Mock / Unit Test
      ↓
Local / Internal
→ Real HSM Integration
```

Hybrid Workflow의 핵심은 Cloud를 내부망처럼 만드는 것이 아니라 **Cloud가 끝내야 할 조건과 Local이 확인해야 할 조건을 분리하는 것**이다.

---

## 9. Multi-Repository는 필요한 것만 넘긴다

하나의 기능이 여러 Repository에 걸쳐 있을 수 있다.

예:

```text
campus-api
campus-admin
campus-common
```

현재 Task가 API와 Common DTO를 함께 바꿔야 한다면 두 Repository가 필요할 수 있다.

반대로 관계없는 Repository까지 모두 연결하면 다음 비용이 늘어난다.

```text
Checkout
Context 탐색
검색 범위
잘못된 변경 가능성
```

따라서 각 Repository의 기준점을 명시하고 필요한 것만 전달한다.

```text
campus-api    @ abc123
campus-common @ 91de02
```

> Multi-Repository Handoff에서도 원칙은 같다. Task에 필요한 Source만 제공하고 각 기준점을 고정한다.

---

## 10. Handoff 상태는 최소한으로 추적한다

Task가 Local과 Cloud를 이동하면 현재 위치를 알 수 있어야 한다.

복잡한 Workflow Engine이 없어도 다음 정도면 시작할 수 있다.

```text
Task ID
Base SHA
Result SHA
Execution Location
Status
Validation Result
Artifact Path
PR
```

예:

```yaml
task_id: AUTH-142
base_sha: abc123
result_sha: def456
location: cloud
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/result.json
pr: 142
```

내부망 검증으로 넘어가면 상태만 바꾼다.

```text
CLOUD_DONE
→ LOCAL_VALIDATION
→ READY_TO_MERGE
```

목적은 상태 모델을 크게 만드는 것이 아니라 **누가 어떤 Source 상태를 검증하고 있는지 잃지 않는 것**이다.

---

## 11. 실행 중 Routing 조건이 바뀔 수 있다

Cloud에서 작업하다 다음 사실이 드러날 수 있다.

```text
Internal Dependency 발견
Cloud에서 재현 불가
예상보다 Scope 확대
Architecture 판단 필요
```

이 경우 Cloud에 계속 머물러야 한다는 규칙은 없다.

```text
Cloud
→ 새 제약 발견
→ Routing 재판단
→ Local 또는 Task 재분해
```

반대로 Local에서 조사하던 문제도 재현 Test와 Scope가 만들어지면 Cloud로 넘길 수 있다.

이 장에서는 Handoff 가능성만 다룬다. 언제 Cloud 작업을 중단하고 Local로 되돌릴지는 17장에서 정리한다.

---

## 12. Handoff의 목표는 Developer Blocking Time을 줄이는 것이다

좋은 Handoff는 Cloud에 보낸 뒤 개발자가 다른 일을 할 수 있게 한다.

설명용 예시:

```text
10:00  Cloud Task 위임
10:02  Developer 다음 작업 시작
10:40  Cloud Evidence 생성
11:10  Developer Review
```

위 시간은 Handoff와 Developer Blocking Time의 차이를 설명하기 위한 예시다.

Cloud 실행시간과 Developer Blocking Time은 다르다.

Task Contract와 Evidence를 미리 정하는 이유도 중간 질문을 줄이기 위해서다.

```text
Local
→ 판단 / 분해
→ Handoff
      ↓
Cloud
→ 독립 실행
→ Evidence 반환
      ↓
Local
→ Review / Internal Validation / Merge
```

다음 장에서는 이 Handoff의 시작점을 사람이 아니라 CI, Review, Schedule 같은 Event로 바꾼다.


# 14장. Task Queue와 Event-driven Cloud Agent

13장까지는 Developer가 Task를 정리해 Cloud로 넘기는 Handoff를 다뤘다.

```text
Developer
→ Task Contract
→ Git Handoff
→ Cloud Worker
```

하지만 실제 개발에서는 사람이 직접 시작하지 않아도 Task 후보가 생긴다.

```text
CI Failure
PR Review Comment
Nightly Failure
Dependency Update
Issue
Scheduled Validation
```

이 장에서는 Cloud Agent를 상시 실행되는 AI 프로세스로 보지 않는다.

**Event가 Task 후보를 만들고, 필요한 경우에만 Worker와 Agent가 실행되는 구조**로 본다.

> 이벤트가 없으면 Agent도 실행하지 않는다.

또 하나의 원칙을 그대로 유지한다.

> 이벤트가 발생했다고 Agent를 호출하지 않는다. 먼저 결정론적 경로를 통과시킨다.

---

## 1. Event는 Agent 호출이 아니라 Task Candidate다

다음 구조는 단순하지만 비용이 커지기 쉽다.

```text
Git Push → Agent
PR Open → Agent
Review Comment → Agent
Nightly Fail → Agent
```

이벤트 수가 곧 Agent 호출 수가 된다.

권장 구조는 다르다.

```text
Event
→ Task Candidate
→ Dedup / Classification
→ Runner / Tool
→ 판단 필요 여부
→ Cloud Agent
```

즉 Event는 작업을 시작할 **계기**이지 실행 주체를 결정하는 답이 아니다.

---

## 2. Event를 Task 입력으로 정규화한다

이벤트 종류는 달라도 Cloud Task가 필요로 하는 정보는 비슷하다.

예:

```text
Source Event
Repository
Git SHA
Task Type
Goal 또는 Failure
Scope
Validation
Environment
Status
```

CI Failure 예:

```yaml
source: ci_failure
repository: campus-platform
git_sha: abc123
task_type: test_failure_fix
failure: AuthServiceTest.expiredToken
validation: ./gradlew test --tests AuthServiceTest.expiredToken
environment: backend-test
```

PR Review Comment도 같은 형태로 바꿀 수 있다.

```yaml
source: review_comment
pr: 142
git_sha: def456
goal: expired token regression test 추가
scope: auth test only
validation: ./gradlew test --tests AuthServiceTest
```

7장의 Task Contract를 Event-driven 경로에서도 재사용하는 셈이다.

---

## 3. 정상 경로는 Runner에서 끝낸다

Git Push를 예로 보자.

```text
Git Push
   ↓
  CI
   ↓
PASS → Done
FAIL → Failure Summary
```

PASS라면 Agent를 호출하지 않는다.

FAIL이어도 바로 Agent를 호출하지 않는다.

```text
CI FAIL
→ Result Gateway
→ Failure Classification
→ Code Reasoning Needed?
```

처리 경로는 다음처럼 나뉠 수 있다.

```text
Infra Failure
→ Retry / Environment 처리

Auto-fix 가능
→ Tool

Code Reasoning 필요
→ Cloud Agent
```

10장의 Runner-first 규칙을 Event의 시작점에 연결한 구조다.

---

## 4. Task Queue는 Event와 실행 사이의 완충지대다

짧은 시간에 여러 Event가 발생할 수 있다.

```text
CI Failure
Review Comment
Nightly Failure
Dependency Update
```

이를 모두 즉시 Worker로 보내기 전에 Queue 또는 상태 저장소에서 다음을 확인할 수 있다.

```text
중복인가?
이미 처리 중인가?
현재 SHA가 최신인가?
Dependency가 있는가?
Runner로 끝낼 수 있는가?
Cloud 실행이 가능한가?
```

Queue 구현이 반드시 별도 메시지 시스템일 필요는 없다.

```text
GitHub Issue / PR metadata
CI metadata
DB
파일 기반 Task 목록
```

어떤 형태든 핵심은 Event 발생과 Worker 실행을 분리하는 것이다.

---

## 5. 같은 실패를 중복 Task로 만들지 않는다

CI Retry, Push, Nightly 실행이 겹치면 같은 실패가 여러 번 Task로 만들어질 수 있다.

중복 판단 후보:

```text
Repository
Git SHA
Task Type
Failure Fingerprint
```

예:

```text
SHA: abc123
Failure: AuthServiceTest.expiredToken
Fingerprint: TEST_FAILURE/401-200/AuthServiceTest:94
```

같은 SHA와 같은 Failure가 이미 처리 중이라면 새 Agent를 시작하지 않는다.

```text
Duplicate Event
→ Existing Task에 연결
```

8장의 Failure Fingerprint가 여기서는 Event deduplication에도 사용된다.

---

## 6. CI Failure는 가장 단순한 Agent-on-failure Event다

예를 들어 PR #142의 Integration Test가 실패했다고 하자.

```text
PR #142
Git SHA: abc123
Integration: FAIL
```

Result Gateway가 실패를 요약한다.

```text
Test
AuthServiceTest.expiredToken

Expected
401

Actual
200
```

분류 결과가 재현 가능한 Code Failure라면 Agent Task를 만든다.

```text
CI Failure
→ Task Contract
→ Cloud Agent
→ Result SHA def456
→ Cloud Runner 재검증
```

결과는 기존 PR에 Commit으로 추가하거나 별도 Draft PR로 반환할 수 있다.

중요한 것은 `CI가 실패했기 때문`이 아니라 **수정 판단이 필요한 재현 가능한 실패로 분류됐기 때문**에 Agent가 호출된다는 점이다.

---

## 7. Review Comment는 수정 요청일 때만 Task가 된다

모든 Review Comment가 실행 가능한 Task는 아니다.

예:

```text
이 예외 처리에 회귀 테스트를 추가해주세요.
```

이 Comment는 다음을 확인한 뒤 Task로 바꿀 수 있다.

```text
수정 요청인가?
현재 PR Scope 안인가?
변경 목표가 명확한가?
Validation을 정의할 수 있는가?
```

반면 의견 교환이나 설계 질문은 자동 Agent Task로 만들지 않는다.

```text
이 구조 자체를 다시 고민해야 하지 않을까요?
```

이런 Comment는 Human Steering 비중이 높다.

> Event-driven 자동화에서도 Task가 명확해야 한다는 5장과 7장의 조건은 그대로 유지된다.

---

## 8. Nightly와 Dependency Update도 같은 패턴을 사용한다

Nightly Test:

```text
Schedule
→ Full E2E Runner
→ PASS → Done
→ FAIL → Failure 분류
→ 재현 가능한 실패만 Task 생성
```

Dependency Update:

```text
Version Update
→ Build / Test Runner
→ PASS → Review 가능한 PR
→ FAIL → Compatibility Fix가 필요할 때 Agent
```

이벤트 종류가 달라도 핵심 경로는 같다.

```text
Event
→ Runner
→ Result
→ 필요할 때만 Agent
```

Event별로 별도 Agent Workflow를 만드는 것보다 공통 실행 규칙을 재사용하는 편이 단순하다.

---

## 9. 불명확한 Issue는 먼저 Local Investigation으로 보낸다

Issue가 생성됐다고 곧바로 Cloud Task가 되는 것은 아니다.

Cloud Task로 바꾸기 쉬운 Issue:

```text
Reproduction 있음
Expected Result 있음
Scope 추정 가능
Validation 가능
```

반대로 다음 Issue는 바로 위임하기 어렵다.

```text
시스템이 가끔 느립니다. 개선해주세요.
```

이 경우 먼저 문제를 좁힌다.

```text
Issue
→ Local Investigation
→ Reproduction / Scope
→ Task Split
→ Cloud Candidate
```

Event-driven 구조는 Task Routing을 생략하는 자동화가 아니다.

---

## 10. Agent가 만든 변경이 다시 Event를 만들 수 있다

Cloud Agent가 Result SHA를 Push하면 CI가 다시 실행된다.

```text
Agent Fix
→ Push
→ CI
→ FAIL
→ Event
→ Agent Fix
→ ...
```

종료 조건이 없으면 Loop가 생긴다.

따라서 Task에는 다음 중 일부를 연결한다.

```text
Task ID
Attempt Count
Failure Fingerprint
Budget
Parent Event
```

예:

```text
같은 Failure Fingerprint 반복
또는
Retry Budget 소진
→ 자동 Agent 재호출 중단
→ Evidence 남김
```

중단 이후 Local로 되돌릴지, Task를 다시 분해할지는 17장의 Routing 역판단 기준을 사용한다.

---

## 11. 결과는 PR 또는 기존 PR의 Commit으로 돌아온다

Event-driven 작업도 반환 경계는 13장과 같다.

```text
Cloud Task
→ Result SHA
→ Evidence
→ PR 또는 기존 PR Update
```

자동화가 결과를 만들었다고 바로 Merge하지 않는다.

```text
Runner Evidence
→ Agent Change
→ Cloud Runner 재검증
→ Review 가능한 상태
```

Human Review를 유지하면서 Event 생성과 반복 실행만 자동화할 수 있다.

---

## 12. Worker는 Task가 있을 때만 필요하다

Task Queue에 작업이 생기면 Worker를 할당한다.

```text
Task Queue
→ Worker Allocation
→ Execute
→ Evidence
→ 종료 또는 Reset
```

짧고 반복되는 Task는 9장의 Warm Worker를 사용할 수 있고, 격리가 중요한 Task는 Ephemeral Worker를 사용할 수 있다.

이 장의 목적은 Worker Pool을 설계하는 것이 아니다.

핵심은 다음이다.

> Cloud Agent는 상시 존재해야 하는 서비스가 아니라 Event에서 만들어진 Task를 처리하는 실행 주체가 될 수 있다.

---

## 13. campus-platform의 Event-driven 흐름

`campus-platform`의 기본 자동 경로를 정리하면 다음과 같다.

```text
Push / PR / Schedule / Review
             ↓
         Event
             ↓
      Task Candidate
             ↓
   Dedup / Classification
             ↓
     Runner / Tool First
             ↓
       PASS → Done
             ↓ FAIL
      Reasoning Needed?
       ├─ NO → Retry / Escalation
       └─ YES
             ↓
        Cloud Agent
             ↓
            Fix
             ↓
       Cloud Runner
             ↓
      Evidence / PR Update
```

이 구조에서 Agent는 Event 시스템의 중심이 아니다.

Task가 명확하고 판단이 필요한 순간에만 호출된다.

다음 장에서는 지금까지 만든 Routing, Environment, Runner, Agent, Git, Evidence, Handoff, Event 흐름을 `campus-platform` 하나의 운영 모델로 합친다.


# Part V. 실제 프로젝트에 적용한다

앞의 Part에서는 Cloud Agent Workflow를 구성하는 요소를 각각 나눠 설명했다.

이제 그 요소를 하나의 프로젝트 안에서 연결한다.

예제는 Java/Spring Boot 기반 `campus-platform`이다. Local에서는 요구사항과 Architecture, 내부망 검증을 담당하고, Cloud Runner는 Build/Test/E2E/Docker 같은 결정론적 실행을 맡는다. Cloud Agent는 재현 가능한 Failure 분석과 제한된 코드 수정에 사용한다.

15장에서는 이 구조를 **정적인 운영 모델**로 본다.

```text
Task Routing
→ Task Contract
→ Environment
→ Runner / Agent
→ Evidence
→ Handoff
```

16장에서는 같은 구조를 `학생 출결 API 인증 변경` 기능 하나에 적용해 Requirement부터 Merge까지 시간 순서대로 따라간다.

이 Part의 목적은 새로운 개념을 추가하는 것이 아니다.

앞에서 만든 원칙이 실제 기능 개발에서 어떻게 연결되는지 확인하는 것이다.

다음 Part에서는 이 Workflow를 언제 멈추거나 Local로 되돌려야 하는지, 그리고 반복되는 판단을 어느 단계부터 자동화할 수 있는지 정리한다.


# 15장. campus-platform Cloud Agent Workflow 설계

지금까지는 Cloud Agent 활용 원칙을 각각 나눠서 설명했다.

```text
Task Routing
Task Contract
Result Gateway
Prepared Environment
Runner-first
Git Isolation
Parallel Worker
Handoff
Event-driven Task
```

15장에서는 이 요소를 `campus-platform` 하나의 운영 모델로 합친다.

새로운 Agent Platform을 설계하는 장이 아니다.

**Local Agent, Cloud Runner, Cloud Agent를 실제 Java/Spring Boot 프로젝트의 어느 지점에 배치할지 연결하는 장**이다.

전체 구조는 다음과 같다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture / Task Split
        ↓
Task Routing
        ↓
Git + Task Contract
        ↓
Prepared Cloud Environment
        ↓
Runner-first
   ├─ PASS → Evidence
   └─ FAIL → 판단 필요 시 Cloud Agent
                    ↓
                  Fix
                    ↓
              Cloud Runner
        ↓
Evidence / PR
        ↓
Developer / Local Agent
        ↓
Internal Validation / Review / Merge
```

이 Workflow의 중심은 Agent가 아니다.

```text
Local 판단
+ Cloud Compute
+ 필요한 지점의 Agent Reasoning
+ Git Handoff
+ Evidence
```

이 다섯 요소를 연결하는 것이 목적이다.

---

## 1. 예제 프로젝트의 경계를 정한다

책에서 사용하는 `campus-platform`은 Cloud Agent Workflow를 설명하기 위한 축약 프로젝트다.

```text
campus-platform/
├─ auth/
├─ student/
├─ attendance/
├─ notification/
├─ integration/
├─ common/
├─ admin-web/
└─ scripts/
```

기본 기술 예:

```text
Java 21
Spring Boot 3.x
Gradle
MyBatis
PostgreSQL / Testcontainers
Docker
Playwright
```

실제 기업 환경의 마지막 검증에는 다음 자원이 있을 수 있다.

```text
Tibero / Oracle
HSM
Internal Jenkins
VPN-only API
사내 시스템
```

이 내부 자원을 Cloud에 모두 복제하는 것을 목표로 하지 않는다.

Cloud에서 재현 가능한 범위와 내부 환경에서만 검증 가능한 범위를 분리한다.

---

## 2. 실행 주체를 세 가지로 나눈다

이 프로젝트에서는 역할을 다음처럼 나눈다.

| 실행 주체 | 기본 역할 |
| --- | --- |
| Local Agent / Developer | 요구사항, Architecture, Task 분해, 내부망, 최종 통합 |
| Cloud Runner | Build, Test, E2E, Docker, Migration Validation |
| Cloud Agent | 재현 가능한 Failure 분석, 작은 Bug Fix, 제한된 Refactoring |

핵심은 Cloud Agent가 모든 개발 작업의 기본 실행 주체가 아니라는 점이다.

```text
판단과 상호작용이 많은 작업
→ Local

결정론적 실행
→ Cloud Runner

독립적인 판단 + 코드 수정
→ Cloud Agent
```

5~6장의 Routing과 Task Catalog를 프로젝트 수준에서 적용한 결과다.

---

## 3. Cloud Environment는 Task Type과 연결한다

9장에서 만든 Prepared Environment를 프로젝트 운영 단위로 연결한다.

예:

```text
backend-test
→ Java 21 / Gradle / Docker / Testcontainers

frontend-e2e
→ Node / Playwright / Browser

migration-test
→ Java / Migration Tool / Disposable DB
```

운영 시 중요한 것은 설치 절차를 매 Task마다 설명하는 것이 아니다.

```text
Task Type
→ Environment 이름 선택
```

예:

```text
RUN-UNIT        → backend-test
RUN-INTEGRATION → backend-test
RUN-E2E         → frontend-e2e
RUN-MIGRATION   → migration-test
```

필요한 도구가 반복해서 빠진다면 Prompt가 아니라 Environment를 수정한다.

---

## 4. 반복 작업은 Task Catalog로 고정한다

팀이 매번 새로운 Prompt부터 만들지 않도록 반복 작업의 실행 방식을 정리한다.

```text
RUN-BUILD
execution: runner
validation: ./gradlew build

RUN-UNIT
execution: runner
validation: ./gradlew test

RUN-INTEGRATION
execution: runner
environment: backend-test

RUN-E2E
execution: runner
environment: frontend-e2e

RUN-DOCKER
execution: runner
validation: docker build ...

FIX-BUG
execution: cloud-agent
input: failure summary + relevant files
validation: target test + regression

REFACTOR-MODULE
execution: cloud-agent
scope: one module
validation: module test
```

Task Catalog의 목적은 Prompt Template을 늘리는 것이 아니다.

다음 세 가지를 매번 다시 결정하지 않게 하는 것이다.

```text
누가 실행하는가?
어떤 Environment를 쓰는가?
무엇으로 검증하는가?
```

---

## 5. Task 상태는 Git 기준점과 연결한다

Cloud Task가 여러 개 실행될수록 대화창보다 Git 상태가 중요해진다.

최소 상태 예:

```text
Task ID
Base SHA
Branch
Result SHA
Execution Type
Status
Validation Result
Artifact Path
PR
```

예:

```yaml
task_id: AUTH-142
base_sha: abc123
branch: agent/auth-142
result_sha: def456
execution: cloud-agent
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/result.json
pr: 142
```

관계는 다음처럼 유지한다.

```text
Task
→ Base SHA
→ Branch
→ Result SHA
→ Verification
→ Evidence
→ PR
```

Agent의 대화 기록을 보지 않아도 어느 코드가 어느 검증을 통과했는지 알 수 있어야 한다.

---

## 6. 같은 SHA에서 검증을 병렬 실행한다

Result SHA가 만들어지면 독립 검증을 병렬 실행할 수 있다.

```text
Result SHA def456
      ↓
+---------+-------------+---------+---------+
|         |             |         |         |
Unit   Integration    Docker     E2E
|         |             |         |         |
+---------+-------------+---------+---------+
              ↓
            Fan-in
```

조건은 단순하다.

```text
모든 검증이 같은 Result SHA를 사용
각 검증이 독립적으로 실행 가능
결과를 Task 단위로 합산 가능
```

다른 SHA의 결과를 하나의 Evidence 묶음으로 사용하지 않는다.

```text
Unit        → def456
Integration → def456
Docker      → def456
E2E         → def789
```

위 상태라면 E2E를 다시 맞춰야 한다.

---

## 7. Result Gateway는 공통 반환 인터페이스가 된다

Runner마다 로그 형식이 달라도 Developer와 Agent가 처음 읽는 결과는 작게 맞출 수 있다.

Task Artifact 예:

```text
artifacts/AUTH-142/
├─ result.json
├─ unit-junit.xml
├─ integration-junit.xml
├─ build.log
├─ screenshots/
└─ e2e-trace/
```

첫 화면은 다음 정도면 충분하다.

```text
Task: AUTH-142
Result SHA: def456

Unit: PASS
Integration: PASS
Docker: PASS
E2E: PASS
```

실패가 있을 때만 8장의 Progressive Result Detail을 사용한다.

```text
Summary
→ Failure Detail
→ Specific Artifact
```

운영 모델에서는 `대형 로그 저장`과 `Agent에게 전달할 결과`를 분리한다.

---

## 8. Failure는 코드 Agent와 Environment 경로로 나눈다

Runner가 FAIL했다고 모두 Cloud Agent Task가 되지는 않는다.

```text
Runner FAIL
      ↓
Failure Classification
      ↓
+----------------------+----------------------+
|                                             |
Infrastructure / Environment              Code Reasoning
|                                             |
Retry / Environment Fix                    Cloud Agent
                                              ↓
                                             Fix
                                              ↓
                                        Cloud Runner
```

예:

```text
Registry timeout
→ Environment 경로

Assertion Failure
→ 재현 가능 + 코드 판단 필요
→ Cloud Agent 후보
```

10장의 Agent-on-failure를 실제 운영 Flow에 배치한 것이다.

---

## 9. Internal Validation을 별도 Stage로 둔다

Cloud에서 끝낼 수 없는 검증은 Workflow 밖의 예외가 아니다.

명시적인 Stage로 둔다.

예:

```text
Cloud Validation
→ PostgreSQL / Testcontainers
→ Unit / Integration
→ Evidence
      ↓
Internal Validation
→ Tibero
→ HSM
→ Internal API
```

Migration:

```text
Cloud
→ 일반 Migration Validation

Internal
→ Tibero 최종 검증
```

HSM 관련 Task:

```text
Cloud
→ Pure Logic / Mock / Contract Test

Internal
→ Real HSM Integration
```

이렇게 하면 내부망 제약이 Cloud Workflow 전체를 막지 않는다.

---

## 10. PR과 Event는 같은 운영 모델에 연결된다

수동 Task와 Event-driven Task를 별도 시스템으로 만들 필요는 없다.

사람이 만든 Task:

```text
Developer
→ Task Contract
→ Queue / Worker
```

Event가 만든 Task:

```text
CI / Review / Schedule
→ Task Candidate
→ Queue / Worker
```

둘 다 이후 흐름은 같다.

```text
Routing
→ Environment
→ Runner / Agent
→ Verification
→ Evidence / PR
```

14장의 Event-driven 구조는 이 운영 모델의 **Task 입력 채널 하나**로 들어온다.

---

## 11. 비용도 세 층으로 관찰한다

3장에서 구분한 비용을 프로젝트 지표에 적용한다.

```text
Compute
- Build / Test / Docker / Browser

LLM
- Task 이해 / Failure 분석 / 코드 수정

Human
- Task 분해 / Review / Internal Validation / Conflict 해결
```

Cloud Agent Workflow가 좋아졌는지는 Agent Session 수로 판단하지 않는다.

확인할 값은 다음과 같다.

```text
Developer Blocking Time
Review Queue Time
Retry / Rework
Cloud Startup Time
PASS 경로의 Agent 호출 수
Evidence 누락률
병렬 검증 Lead Time
```

Agent 호출 수가 줄어도 Review Queue가 늘면 전체 Lead Time은 개선되지 않을 수 있다.

---

## 12. 처음부터 운영 플랫폼을 만들 필요는 없다

초기에는 다음 정보만으로도 Workflow를 운영할 수 있다.

```text
Task ID
Branch
Base SHA
Result SHA
Execution Type
Status
Validation Result
Artifact Path
PR
```

이를 Git, CI metadata, PR, Artifact 저장소로 관리할 수 있다.

먼저 반복 가능한 실행 규칙을 만든다.

```text
Task Routing
→ Runner-first
→ Evidence
→ Handoff
```

그 다음 반복되는 수동 결정을 자동화한다.

18장에서 다룰 Harness와 Orchestration은 이 운영 흐름이 안정화된 이후의 문제다.

---

## 13. campus-platform 전체 운영 모델

지금까지를 하나의 그림으로 합치면 다음과 같다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture
Task Split
        ↓
Task Routing
        ↓
Git + Task Contract
        ↓
Task-specific Environment
        ↓
+--------------+--------------+--------------+
|              |              |              |
Unit Runner  Integration    E2E / Docker   Cloud Agent
|              Runner         Runner         |
+--------------+--------------+--------------+
                    ↓
              Result Gateway
                    ↓
            PASS / Failure Class
                    ↓
            필요하면 Agent Fix
                    ↓
            Cloud Runner Verify
                    ↓
               Evidence / PR
                    ↓
             Developer / Local
                    ↓
       Tibero / HSM / Internal API
                    ↓
              Review / Merge
```

이 모델의 핵심은 한 문장으로 정리할 수 있다.

> Task Contract로 작업을 넘기고, Runner가 가능한 일을 먼저 실행하며, 판단이 필요한 실패에만 Agent를 사용하고, Evidence로 결과를 다시 Local에 돌려준다.

다음 장에서는 이 정적인 운영 모델을 `학생 출결 API 인증 변경` 하나에 적용해 Requirement부터 Merge까지 시간 순서대로 따라간다.


# 16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

15장에서는 `campus-platform`의 전체 운영 모델을 정리했다. 이번 장에서는 그 구조를 하나의 기능에 적용해 요구사항부터 Merge까지 시간 순서대로 따라간다.

예제는 다음과 같다.

```text
학생 출결 API 인증 변경
```

요구사항은 단순화한다.

```text
만료된 인증 토큰으로 출결 API를 호출하면 HTTP 401을 반환한다.
정상 토큰의 기존 출결 등록 동작은 유지한다.
관리자 Web에서는 인증 실패 상태를 확인할 수 있어야 한다.
```

이 기능에는 Backend 수정, Unit / Integration Test, Web E2E, Docker Build, 내부 환경 검증이 필요하다고 가정한다.

이 장의 목적은 앞 장의 개념을 다시 설명하는 것이 아니다. 실제 기능 하나에서 **Local과 Cloud가 언제 바뀌고, 무엇을 다음 단계의 입력과 Evidence로 사용하는지** 보여주는 것이다.

> Cloud Agent 활용은 별도 도구 사용법이 아니라 개발 Workflow 설계다.

---

## 1. Local에서 요구사항과 영향 범위를 고정한다

처음부터 기능 전체를 Cloud Agent에게 맡기지 않는다.

Local에서 먼저 다음을 결정한다.

```text
정상 동작
실패 동작
영향 모듈
변경 금지 영역
내부 시스템 의존성
Cloud에서 검증 가능한 범위
```

이번 예제의 초기 판단은 다음과 같다.

```text
Architecture / 영향 분석
→ Local

Expired token 처리 수정
→ Cloud Agent 후보

Unit / Integration / E2E / Docker
→ Cloud Runner

Tibero / Internal API 최종 확인
→ Local
```

예상 관련 파일도 좁힌다.

```text
AuthService.java
JwtTokenProvider.java
AttendanceController.java
AuthServiceTest.java
AttendanceApiTest.java
```

그리고 이번 변경에서 건드리지 않을 영역을 정한다.

```text
DB Schema 변경 없음
HSM 변경 없음
OAuth 전체 구조 변경 없음
공통 Exception Format 변경 없음
```

이 판단이 뒤 단계의 Scope와 Review 범위를 만든다.

---

## 2. 전달 가능한 Git 기준점을 만든다

Cloud Worker가 Local Working Directory의 미커밋 상태를 자동으로 아는 것을 기대하지 않는다.

필요한 Local 작업을 정리한 뒤 기준 Commit을 만든다.

```text
Local
→ 빠른 검증
→ Commit
→ Push
```

예:

```text
Base SHA
abc123
```

이후 Cloud Task와 검증 결과는 이 Git 상태에서 출발한다.

```text
Base SHA
abc123
   ↓
Result SHA
   ↓
Verification
   ↓
PR
```

11장과 13장에서 설명한 Git Handoff를 실제 기능의 시작점으로 사용하는 것이다.

---

## 3. 기능을 Agent Task와 Runner Task로 나눈다

하나의 기능을 한 Agent에게 통째로 맡기지 않는다.

이번 기능은 다음처럼 나눌 수 있다.

```text
Task A
Expired token backend fix
→ Cloud Agent

Task B
Backend Unit / Integration Validation
→ Cloud Runner

Task C
Admin Web E2E Validation
→ Cloud Runner

Task D
Docker Build Validation
→ Cloud Runner
```

Task A에는 판단과 코드 수정이 필요하다. B~D는 정해진 명령과 판정 기준이 있으므로 Runner가 우선한다.

Task A의 입력은 7장의 Task Contract 형식을 그대로 사용한다.

```text
Task
Expired token 처리 수정

Goal
Attendance API 인증 실패 → HTTP 401

Base SHA
abc123

Scope
auth + attendance auth boundary

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Do Not Change
- DB Schema
- OAuth 전체 구조
- Common Exception Format

Validation
./gradlew test --tests AuthServiceTest

Expected Result
expired token → 401
normal token → 기존 테스트 PASS

Output / Evidence
- Result SHA
- Changed Files
- Validation Result
```

여기서 중요한 것은 형식 자체가 아니라 Cloud Worker가 **어디서 시작하고, 어디까지 바꾸며, 무엇을 통과해야 하는지**가 명시되어 있다는 점이다.

---

## 4. Prepared Environment에서 작은 Context로 시작한다

Task A는 `backend-test` 같은 준비된 Environment에서 시작한다.

```text
Prepared Environment
+ Fresh Source / Branch
        ↓
Cloud Task
```

Agent에게 처음 제공하는 정보도 작게 유지한다.

```text
Task Contract
Relevant Files
Known Failure
```

필요할 때만 Context를 넓힌다.

```text
Relevant Files
→ Direct Dependency
→ Related Document
→ Wider Module Context
```

Repository 전체를 처음부터 읽는 것이 기본 경로가 아니다.

> Cloud에서 막혔을 때 Context를 한 번에 넓히지 않고 필요한 정보만 추가한다.

---

## 5. 수정 뒤에는 Target Test로 가장 먼저 검증한다

Agent가 코드를 수정했다고 하자.

첫 번째 검증은 좁게 시작한다.

```bash
./gradlew test --tests AuthServiceTest
```

PASS라면 Regression 단계로 이동한다.

FAIL이면 8장의 Result Gateway를 통해 작은 Failure Summary부터 확인한다.

```text
STATUS: FAIL
Test: AuthServiceTest.expiredToken
Expected: 401
Actual: 200
Top Frame: AuthServiceTest.java:94
```

필요한 경우에만 상세 결과를 추가한다.

```text
Failure Detail
→ Stack Trace
→ Specific Test Log
→ Raw Artifact
```

Agent가 수정한 사실이 성공 조건이 아니다. Cloud Runner가 동일한 Validation을 통과해야 다음 단계로 이동한다.

---

## 6. 같은 실패가 반복되면 무한 Retry하지 않는다

Agent가 수정한 뒤에도 같은 Failure가 반복될 수 있다.

```text
Retry #1
AuthServiceTest.expiredToken
expected 401 / actual 200

Retry #2
AuthServiceTest.expiredToken
expected 401 / actual 200
```

위 Retry 횟수는 설명용 예다.

Failure Fingerprint가 그대로이고 Budget도 소진되고 있다면 Context와 Prompt만 계속 늘리지 않는다.

```text
Same Failure
+ Retry Budget 소진
        ↓
Stop / Escalate
```

반대로 실패가 `Compilation Error → Target Test Failure`처럼 바뀌었다면 작업이 진행된 것일 수 있다.

Retry 횟수와 Failure 변화 여부를 같이 본다.

구체적인 Local Fallback 판단은 17장에서 정리한다.

---

## 7. Result SHA에서 Regression을 병렬 실행한다

Agent가 다음 Result SHA를 만들었다고 하자.

```text
Result SHA
def456
```

이제 같은 SHA에서 독립 검증을 병렬로 실행한다.

```text
Result SHA def456
      ↓
+---------+-------------+---------+---------+
|         |             |         |         |
Unit   Integration    Docker     E2E
|         |             |         |         |
+---------+-------------+---------+---------+
          ↓
        Fan-in
```

예:

```text
Unit        PASS
Integration PASS
Docker      PASS
E2E         FAIL
```

이 단계에서 중요한 조건은 모든 Evidence가 같은 Source 상태를 검증한다는 점이다.

UI/E2E 실패라면 Screenshot, Video, Trace 같은 결과부터 본다.

```text
Scenario
attendance expired token error view

Expected
401 error state visible

Actual
generic error page

Artifact
screenshot-fail.png
```

필요한 경우 Agent가 수정하고, 새 Result SHA에서 다시 Cloud Runner가 검증한다.

---

## 8. Cloud 단계가 끝나면 Evidence와 PR을 반환한다

Cloud 검증이 끝났다면 Local로 자연어 완료 보고가 아니라 검증 가능한 결과를 돌려준다.

설명용 예:

```text
Task: AUTH-142
Result SHA: def789

Build: PASS
Unit: PASS
Integration: PASS
Docker: PASS
E2E: PASS

Artifacts:
- junit.xml
- integration-junit.xml
- screenshot-after.png
- e2e-trace.zip

PR: #142
```

실제 Test Count나 실행시간은 프로젝트 측정값을 사용한다. 여기서는 결과 구조만 보여준다.

Local Review에서는 다음 순서로 확인할 수 있다.

```text
Evidence
→ Changed Files
→ Diff
→ Architecture 영향
```

UI 변경이라면 Screenshot / Video를 먼저 확인한 뒤 Diff를 본다.

---

## 9. 내부망 검증을 Local에서 이어간다

Cloud에서 모든 검증을 끝낼 필요는 없다.

이번 기능에서 최종 대상이 내부 Tibero와 Internal API라고 가정한다.

```text
Cloud
→ 일반적으로 재현 가능한 Build / Test / E2E
      ↓
Local / Internal
→ Tibero
→ Internal API
→ Jenkins Deployment Validation
```

HSM 변경이 없다면 HSM 검증까지 추가하지 않는다. Task와 관련된 내부 경계만 확인한다.

Cloud Evidence와 내부 검증도 같은 PR SHA를 기준으로 맞춘다.

```text
PR SHA
→ Cloud Required Checks
→ Internal Validation
→ Review
```

검증 사이에 코드가 바뀌었다면 필요한 Check를 다시 실행한다.

---

## 10. Merge와 Task 종료

Merge 전에는 최종 기준 SHA에서 필요한 검증이 모두 연결되어 있는지 확인한다.

```text
PR SHA
→ Required Checks
→ Internal Validation
→ Review Complete
→ Merge
```

작업 종료 후에는 모든 Agent 대화나 원본 로그를 영구 보관할 필요는 없다.

최소 작업 이력은 다음 정도면 충분할 수 있다.

```text
Task ID
PR
Merge Commit
Final Evidence
주요 Failure / Retry 정보
```

이 기록의 목적은 장기 Agent Memory를 만드는 것이 아니라 나중에 `무엇을 변경했고 무엇으로 검증했는가`를 추적하는 것이다.

---

## 11. 시간 흐름으로 보면

설명용 Timeline을 만들어보자.

```text
09:30 Requirement / Impact Analysis
10:00 Base Commit
10:05 Cloud Task 시작
10:06 Developer 다음 작업 시작
10:22 Agent Fix Commit
10:23 Parallel Validation
10:40 E2E Failure
10:45 Fix
10:58 All Required Cloud Checks PASS
11:20 Developer Review
11:35 Internal Validation
11:45 Merge
```

이 시간은 실제 성능 수치가 아니라 Workflow를 설명하기 위한 예시다.

핵심은 다음 관계다.

```text
Cloud Elapsed Time
!=
Developer Blocking Time
```

Developer가 Cloud 실행 전체 시간 동안 기다리지 않았다는 점이 중요하다.

비용도 분리해서 본다.

```text
Compute
→ Unit / Integration / Docker / E2E

LLM
→ Failure 분석 / 코드 수정

Human
→ Requirement / Review / Internal Validation / Merge 판단
```

어느 구간이 병목인지 구분해야 다음 개선 지점을 찾을 수 있다.

---

## 12. 실패 분기는 실행 위치를 다시 판단하는 신호다

정상 Workflow가 항상 끝까지 Cloud에서 진행되는 것은 아니다.

### Cloud에서 재현되지 않음

```text
Cloud Runner
→ cannot reproduce
→ Evidence 반환
→ Local 조사
```

### 내부 DB 의존성 발견

```text
Cloud Analysis
→ Tibero-specific behavior 발견
→ Local / Internal Validation으로 이동
```

### Scope가 예상보다 크게 확대됨

```text
3개 파일 예상
→ 여러 Module / DB Schema 영향 발견
→ Cloud Task 중단
→ Local에서 재분해
```

위 파일 수는 설명용 예다.

이 분기에서 Cloud 작업은 실패한 것이 아니다. 실행 중 발견된 조건에 따라 Routing을 다시 한 것이다.

17장에서는 이런 중단과 Local Fallback 기준을 체계적으로 정리한다.

---

## 13. 하나의 기능을 끝까지 연결하면

이번 기능의 전체 흐름은 다음과 같다.

```text
Local
Requirement / Impact Analysis
        ↓
Base Commit
        ↓
Task Split / Task Contract
        ↓
Git Handoff
        ↓
Cloud Agent
Analyze / Fix
        ↓
Target Runner
        ↓
Parallel Regression
        ↓
Evidence / PR
        ↓
Local Review
        ↓
Internal Validation
        ↓
Merge
```

이 흐름에서 Cloud Agent는 전체 개발을 대신하지 않는다.

```text
Local
→ 결정과 통합

Cloud Runner
→ 실행과 검증

Cloud Agent
→ 필요한 판단과 수정
```

> 작은 Task를 Git으로 넘기고, Runner와 Agent가 만든 Evidence를 다시 Local로 가져온다.

이것이 이 책에서 사용하는 Local + Cloud Hybrid Workflow의 실제 실행 형태다.

# Part VI. Cloud의 한계와 다음 단계를 정한다

Cloud Agent를 잘 사용하는 방법에는 Cloud를 쓰지 않을 때를 판단하는 것도 포함된다.

Task가 너무 작거나, Scope가 계속 커지거나, 내부망과 장비가 필요하거나, 같은 실패가 반복된다면 Cloud 이점이 사라질 수 있다. 이때 Cloud Task를 끝까지 유지하는 것이 목표가 아니다. 현재 Evidence를 남기고 Local이나 Hybrid로 재Routing하는 것이 더 나은 선택일 수 있다.

17장에서는 실행 중 발견되는 Stop Signal과 Local Fallback을 다룬다.

18장에서는 Workflow가 안정된 뒤의 다음 단계를 본다. 반복되는 Environment 선택, Runner/Agent 분류, Validation, Retry 판단을 Harness와 일반 코드로 옮길 수 있다. 그 다음에야 Orchestration이 의미를 가진다.

순서는 다음과 같다.

```text
안정된 Task Routing
→ 반복 가능한 Validation
→ Prepared Environment
→ Evidence
→ Harness
→ 필요한 범위의 Orchestration
```

이 책은 더 많은 Agent를 만드는 방법으로 끝나지 않는다.

마지막에 남는 질문은 처음과 같다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
Cloud 이점이 사라지면 언제 Local로 돌아올 것인가?
```

> 더 많은 Agent보다 더 나은 Task Routing, Harness, Validation이 먼저다.


# 17장. Cloud가 항상 정답은 아니다

5장에서는 Task를 시작할 때 `Local / Cloud / Hybrid / Runner-first` 중 어디에 둘지 판단했다.

17장에서는 반대 방향을 본다.

> Cloud에서 시작한 Task를 언제 중단하고, 언제 Local이나 Hybrid로 되돌려야 하는가?

Cloud Agent의 장점은 분명하다.

```text
독립 실행환경
장시간 작업의 비동기 위임
Parallel Compute
Local Resource Occupancy 감소
Git 기반 Handoff
```

그러나 이 장점이 항상 이득으로 이어지는 것은 아니다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

> Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.

이 장의 핵심은 Cloud 사용률을 높이는 것이 아니라 **Cloud의 이점이 사라지는 신호를 빨리 찾는 것**이다.

---

## 1. Cloud 사용량은 성공 지표가 아니다

다음 숫자가 늘었다고 해서 개발 Workflow가 좋아졌다고 볼 수는 없다.

```text
Cloud Task 수
Agent Session 수
자동 생성 PR 수
병렬 Worker 수
```

대신 다음을 본다.

```text
Lead Time
Developer Blocking Time
검증 품질
Review Cost
Retry Cost
Compute / LLM / Human Cost
```

Agent가 많은 PR을 만들었지만 Review Queue가 쌓였다면 병목은 그대로다.

작은 수정 하나를 위해 Worker 준비, Checkout, Branch, PR 과정을 거친다면 Cloud Overhead가 작업 자체보다 클 수 있다.

목표는 다음이 아니다.

```text
Cloud Agent 사용률 최대화
```

목표는 다음에 가깝다.

```text
이 Task를 Cloud로 보냈을 때
전체 Workflow 비용이 실제로 줄어드는가?
```

---

## 2. Task보다 전달 비용이 크면 Local이 낫다

다음 작업을 생각해 보자.

```text
README 오타 수정
설정값 한 줄 변경
짧은 코드 수정
```

Cloud에는 고정 비용이 있다.

```text
Worker Start
Repository Checkout
Environment Ready
Context Load
Commit / Push / PR
Review
```

개념적으로 다음 관계라면 Local을 우선한다.

```text
Cloud Overhead > Task Work
```

`몇 분 이하이면 Local` 같은 절대 기준은 두지 않는다. 프로젝트마다 Cold Start, CI, Review 시간이 다르기 때문이다.

9장의 Prepared Environment로 시작 비용을 줄일 수 있어도 모든 작은 Task가 Cloud에 적합해지는 것은 아니다.

---

## 3. 큰 Context와 잦은 Human Steering은 Cloud 이점을 줄인다

다음처럼 여러 영역을 동시에 이해해야 하는 Task가 있다고 하자.

```text
Auth
Student
Permission
DB Schema
Common Exception
Admin Web
External API
```

이 작업이 Architecture 결정이나 요구사항 정리까지 포함한다면 독립적인 실행 Task라기보다 탐색과 의사결정에 가깝다.

```text
Developer + Local Agent
→ 구조 탐색
→ 선택지 비교
→ 결정
→ 작은 Task로 분해
```

Cloud Task Contract가 계속 커지는 것도 신호다.

```text
Relevant Files 증가
관련 Module 증가
Validation 증가
Forbidden Changes 증가
```

또 작업 중 사람이 계속 방향을 바꿔야 한다면 비동기 위임의 가치가 줄어든다.

```text
Cloud 결과 확인
→ 방향 수정
→ 다시 실행
→ 다시 확인
```

이 흐름이 반복된다면 Local의 짧은 Feedback Loop가 더 적합할 수 있다.

---

## 4. Internal Network와 정책은 Hard Constraint다

일부 조건은 단순한 비용 비교가 아니라 실행 위치를 제한한다.

예:

```text
Tibero / Oracle
HSM
Internal Jenkins
VPN-only API
Internal Git / Nexus
사내 Redis / Kafka
```

Cloud에서 이 자원에 접근할 수 없고 Task 완료에 반드시 필요하다면 Local 또는 Hybrid가 된다.

```text
Cloud
→ Pure Logic / Mock / 일반 검증
      ↓
Local
→ 실제 내부 자원 검증
```

조직 정책도 같은 종류의 제약이다.

```text
외부 SaaS에 Source 제공 금지
특정 Repository 반출 금지
운영 데이터 반출 금지
Secret 제공 제한
계약상 제3자 처리 제한
```

이 경우 제품 기능과 관계없이 Cloud 실행이 불가능할 수 있다.

책에서는 법률이나 보안 정책을 해석하지 않는다. Routing의 Hard Constraint로만 다룬다.

---

## 5. Cloud에서 재현되지 않는 문제를 계속 Retry하지 않는다

다음 문제는 Cloud에서 재현하기 어려울 수 있다.

```text
특정 Device 의존
HSM Firmware 의존
Internal Network Latency
Production-only Race Condition
운영 데이터 상태 의존
Local-only File / Process State
```

좋지 않은 흐름:

```text
재현 안 됨
→ Context 확대
→ Agent Retry
→ 재현 안 됨
→ 다시 Retry
```

먼저 다음을 확인한다.

```text
같은 입력을 만들 수 있는가?
같은 Runtime 조건을 만들 수 있는가?
같은 외부 의존성에 접근할 수 있는가?
```

아니라면 Cloud에서 얻은 Evidence를 보존하고 Local로 이동한다.

```text
Cloud
→ cannot reproduce
→ Evidence 저장
→ Local Fallback
```

운영 Incident도 마찬가지다. `가끔 느리다` 같은 문제는 먼저 운영/Local 환경에서 범위를 좁힌 뒤 재현 가능한 작은 Bug가 되었을 때 Cloud Task로 바꾸는 편이 낫다.

---

## 6. Local Working State가 기준점이라면 Handoff가 먼저다

Cloud Worker는 다음 상태를 자동으로 공유하지 않는다.

```text
uncommitted source
untracked config
IDE-only setting
local DB state
임시 patch
running process state
```

선택지는 두 가지다.

```text
명시적인 Git 상태로 정리
→ Cloud Handoff
```

또는:

```text
현재 Task는 Local에서 계속
```

Cloud 사용을 위해 중간 실험 상태를 억지로 정리하는 비용이 더 크다면 Local을 선택할 수 있다.

Git은 Handoff Boundary이지만 모든 작업이 즉시 Handoff 가능한 것은 아니다.

---

## 7. 격리할 수 있어도 통합 비용이 크면 병렬화하지 않는다

여러 Cloud Worker가 다른 Branch와 Container를 사용해도 논리적 충돌은 남는다.

```text
Agent A → UserService.java
Agent B → UserService.java
Agent C → UserService.java
```

또는:

```text
Agent A / B
→ 같은 DB Schema / Migration Sequence 수정
```

실행 중에는 독립적이어도 Fan-in에서 비용이 발생한다.

```text
Merge Conflict
Semantic Conflict
Regression
Review 증가
```

이 경우 선택은 다음과 같다.

```text
순차화
Task 재분해
공통 변경 선행
```

11~12장의 원칙을 역으로 적용한다.

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

---

## 8. Cloud Task 중단 기준을 명시한다

Cloud Task는 시작했다고 끝까지 Cloud에서 해결할 필요가 없다.

다음 신호가 나타나면 중단이나 재분류를 검토한다.

```text
동일 Failure Fingerprint 반복
Retry / Cost Budget 소진
Scope 예상보다 크게 증가
Internal Dependency 발견
필요 Repository 계속 증가
요구사항 불명확 발견
Human Steering 반복
Cloud 재현 불가
```

예를 들어 처음에는 제한된 파일 변경으로 예상했지만 실제로 여러 Module과 DB Schema까지 영향을 준다면 다음처럼 전환한다.

```text
Cloud Task Stop
→ 현재 Evidence 반환
→ Local Architecture Review
→ Task 재분해
```

중단은 실패가 아니다.

잘못된 Routing 가정을 빨리 수정한 것이다.

---

## 9. Local Fallback도 Handoff다

Fallback 시 다음 한 줄만 남기면 안 된다.

```text
Cloud에서 해결하지 못함
```

Local에서 그대로 이어갈 수 있는 Return Package를 만든다.

```text
Task ID
Base SHA
Result SHA
Changed Files
Failed Command
Failure Summary
Artifacts
Failure Fingerprint
Attempted Fixes
Retry / Budget 상태
Fallback Reason
```

예:

```text
Task: HSM-37
Status: LOCAL_FALLBACK
Base SHA: abc123
Result SHA: def456
Reason: actual HSM required

Cloud Validation Result
Unit: PASS
Mock Contract: PASS

Artifact Path
artifacts/HSM-37/def456/

Remaining
Actual HSM session validation
```

Cloud에서 한 작업을 버리지 않고 남은 경계부터 Local에서 이어간다.

> Local Fallback은 실패가 아니라 Routing의 일부다.

---

## 10. campus-platform에서 역방향 Routing을 적용하면

기능 이름보다 실행 조건을 본다.

| 상황 | 기본 판단 |
| --- | --- |
| HSM 실제 장비 장애 분석 | Local |
| Tibero에서만 재현되는 SQL 동작 | Local |
| 새 인증 Architecture 설계 | Local |
| 재현 가능한 AuthService Bug | Cloud Agent 후보 |
| Unit / Integration / E2E / Docker | Cloud Runner |
| Migration 일반 검증 | Cloud Runner |
| Tibero 실제 적용 | Local |
| HSM Pure Logic / Mock Test | Cloud Runner |
| 실제 HSM Session | Local |

같은 기능 안에서도 단계별 실행 위치는 달라질 수 있다.

```text
Architecture
→ Local

Implementation
→ Cloud Agent 후보

Build / Test
→ Cloud Runner

Internal Boundary
→ Local
```

---

## 11. Cloud 보내기 전과 실행 중에 묻는 질문은 다르다

5장에서는 Task 시작 전에 Routing을 판단했다.

```text
Scope가 명확한가?
독립 검증 가능한가?
Internal Network가 필요한가?
Git Handoff가 가능한가?
```

17장에서는 실행 중 조건이 바뀌었는지 본다.

```text
예상보다 Scope가 커졌는가?
Cloud에서 재현 가능한가?
같은 실패가 반복되는가?
새 Hard Constraint가 발견됐는가?
사람 개입이 계속 필요한가?
Fan-in 비용이 예상보다 큰가?
```

초기 Routing과 실행 중 재Routing을 구분하면 Cloud Task를 억지로 끝까지 유지하지 않아도 된다.

---

## 12. Cloud를 쓰지 않는 능력도 Cloud 활용 능력이다

Cloud Agent를 잘 사용한다는 말을 `많이 사용한다`로 이해하면 과도한 자동화로 이어질 수 있다.

이 책에서 말하는 활용 능력은 다음에 가깝다.

```text
Cloud가 이득인 Task를 찾는다.
Runner로 끝날 작업은 Runner에 둔다.
Local이 유리한 작업은 Local에 둔다.
Cloud 이점이 사라지면 빠르게 Fallback한다.
```

> Cloud Agent를 잘 사용하는 능력에는 Cloud를 쓰지 않을 때를 아는 것도 포함된다.

다음 장에서는 이 Workflow가 안정화된 뒤 반복 결정을 어디까지 Harness와 Orchestration으로 자동화할 수 있는지 살펴본다.

# 18장. 다음 단계: Harness와 Orchestration

17장까지 이 책은 Cloud Agent를 실제 개발 Workflow에 넣는 방법을 정리했다.

핵심 구조는 이미 완성되어 있다.

```text
Local / Cloud Routing
→ Task Contract
→ Git Handoff
→ Prepared Environment
→ Runner-first
→ 필요한 경우 Cloud Agent
→ Evidence / PR
→ Local Review / Internal Validation
```

이 구조가 반복해서 동작하면 다음 질문이 생긴다.

```text
반복되는 준비와 판단을 어디까지 자동화할 수 있을까?
```

이 장은 새로운 Agent Platform을 설계하는 장이 아니다.

앞에서 만든 Workflow를 안정화한 뒤 자연스럽게 나타나는 다음 단계만 짧게 정리한다.

> 먼저 Cloud Agent를 잘 사용하는 Workflow를 만들고, 그 다음에 Orchestration을 자동화한다.

---

## 1. 다음 단계는 더 많은 Agent가 아니라 더 적은 반복 판단이다

앞 장까지 사람이 반복해서 결정한 것은 대체로 다음과 같다.

```text
어떤 Task를 Cloud로 보낼 것인가?
어떤 Environment를 사용할 것인가?
Runner인가 Agent인가?
어떤 Validation을 실행할 것인가?
어떤 Evidence를 남길 것인가?
언제 Retry하고 언제 멈출 것인가?
```

이 결정이 프로젝트에서 반복되고 기준이 안정되면 일부를 코드와 설정으로 옮길 수 있다.

```text
Task Type
→ Environment
→ Runner / Agent
→ Validation
→ Evidence
```

자동화의 목적은 Agent 수를 늘리는 것이 아니라 같은 결정을 매번 다시 하지 않는 것이다.

---

## 2. Harness는 Agent가 반복해서 추론할 일을 줄인다

Agent가 Repository에 들어올 때마다 다음을 다시 찾아야 한다면 비용이 반복된다.

```text
어떻게 Setup하는가?
어떻게 Build하는가?
어떻게 Test하는가?
어떤 파일부터 읽는가?
무엇이 PASS인가?
결과는 어디에 남는가?
```

이 정보를 Prompt에 계속 추가하기보다 Repository와 실행환경이 제공하도록 만들 수 있다.

예:

```text
AGENTS.md
scripts/build
scripts/test
scripts/verify
Task Contract
Prepared Environment
Result Gateway
```

예를 들어 다음 명령 하나가 인증 모듈의 표준 검증을 수행한다고 하자.

```bash
./scripts/verify-auth.sh
```

Agent 입장에서는 다음 구조가 된다.

```text
Task Contract
→ 표준 Validation
→ 구조화된 Result
```

Harness는 거대한 프레임워크가 아니다.

> Agent가 반복해서 탐색하고 추론해야 했던 개발 규칙을 발견 가능하고 실행 가능한 형태로 꺼내놓는 것에 가깝다.

Agent가 같은 종류의 Environment / Build / Validation 문제로 반복 실패한다면 Prompt를 길게 만들기 전에 Harness와 실행환경을 먼저 점검한다.

---

## 3. Routing도 반복되면 일부 자동화할 수 있다

5장과 17장에서 사람이 Routing을 판단했다.

반복 패턴이 안정되면 일부는 규칙으로 만들 수 있다.

```text
unit-test
→ Cloud Runner

admin-e2e
→ frontend-e2e Runner

reproducible-bug-fix
→ Cloud Agent 후보

requires-hsm
→ Local / Hybrid
```

Task Metadata가 있다면 다음처럼 연결할 수 있다.

```yaml
type: bug-fix
scope: auth
reproducible: true
internal_network: false
validation: ./gradlew test --tests AuthServiceTest
```

Hard Constraint는 일반 코드로 처리할 수 있다.

```text
internal_network = true
→ Local / Hybrid
```

애매한 Task만 사람이 판단하거나 Agent가 보조한다.

모든 Routing 결정을 다시 LLM에게 물어보는 것이 자동화는 아니다.

---

## 4. Orchestration은 Task와 실행환경을 연결한다

Workflow가 반복되면 다음 흐름을 하나의 Orchestration으로 묶을 수 있다.

```text
Task
 ↓
Classification
 ↓
Environment Selection
 ↓
Runner / Agent Selection
 ↓
Execution
 ↓
Validation
 ↓
Evidence
```

여기서 Orchestrator가 반드시 `상위 Agent`일 필요는 없다.

많은 기능은 일반 프로그램으로 처리할 수 있다.

```text
Task Type Mapping
Environment Lookup
Queue
Retry Counter
Failure Fingerprint Check
Artifact Index
```

LLM은 여전히 판단과 코드 수정이 필요한 구간에만 들어간다.

10장의 Runner-first 원칙을 Orchestration 단계에서도 유지하는 것이다.

---

## 5. Compute와 Reasoning의 Lifecycle도 분리할 수 있다

3장에서 다음 모델을 사용했다.

```text
Brain
→ LLM 판단

Hands
→ Container / Browser / DB / Tool 실행
```

후속 시스템에서는 하나의 판단이 여러 실행환경을 호출할 수도 있다.

```text
Agent
→ backend-test Runner
→ Failure 분석
→ frontend-e2e Runner
→ 결과 비교
```

이 경우 다음 두 Lifecycle을 같은 것으로 볼 필요가 없다.

```text
Reasoning Lifecycle
!=
Compute Lifecycle
```

Session Hibernate, One Brain Multiple Hands, Worker Pool 같은 상세 설계는 이 책의 범위를 넘는다.

여기서는 LLM과 Compute를 분리해 설계할 수 있다는 방향만 남긴다.

---

## 6. Evidence는 Query 가능한 인터페이스로 발전할 수 있다

8장에서는 Raw Artifact를 보존하고 작은 Summary부터 읽었다.

후속 단계에서는 Agent가 필요한 결과를 구조화된 방식으로 조회할 수 있다.

```text
get_failed_tests(task)
get_errors(service, since)
get_http_failures(status)
get_browser_console_errors(task)
```

개념은 다음과 같다.

```text
Dashboard for Humans
+
Queryable Evidence for Agents
```

핵심은 새 Observability Platform을 만드는 것이 아니다.

긴 로그를 Agent가 반복해서 읽지 않게 한다는 기존 원칙을 인터페이스로 확장하는 것이다.

---

## 7. Scheduling도 Task 특성에 맞게 발전할 수 있다

Task마다 필요한 실행환경은 다르다.

```text
Unit Test
→ backend-test Runner

Integration Test
→ Docker / DB 가능한 Environment

E2E
→ Browser Environment

Bug Fix
→ Agent + 필요한 Validation
```

후속 Scheduler는 Task Metadata와 Environment Capability를 연결할 수 있다.

```text
Task Requirement
↔
Environment Capability
```

또 9장의 Prepared Environment를 바탕으로 Warm Worker와 Ephemeral Worker를 선택할 수 있다.

```text
짧고 반복적인 검증
→ Warm Worker 후보

격리 요구가 크거나 장시간 작업
→ Ephemeral Worker 후보
```

제품별 CPU / RAM 숫자나 Worker Pool 운영 상세는 현재 책의 원칙으로 고정하지 않는다.

---

## 8. Best-of-N과 Replay는 제한된 고급 기법이다

같은 어려운 Task를 여러 Agent가 독립적으로 풀게 하는 Best-of-N은 일반 병렬화와 다르다.

```text
같은 Bug
├─ Agent A → Patch A
├─ Agent B → Patch B
└─ Agent C → Patch C
        ↓
Deterministic Validation
```

Context와 LLM 비용도 반복되므로 기본값은 N=1이다.

자동 검증 가능하고 실패 비용이 큰 어려운 Task에서만 제한적으로 검토한다.

Workflow 자체가 바뀌었을 때는 과거 Task를 다시 실행해 비교하는 Replay도 생각할 수 있다.

```text
같은 Task
→ 이전 Harness / Environment
→ 새로운 Harness / Environment
→ 성공 여부 / Retry / Token / Human Intervention 비교
```

이 책에서는 Best-of-N 시스템이나 평가 플랫폼을 설계하지 않는다. Cloud Workflow를 개선할 때 사용할 수 있는 후속 관점으로만 소개한다.

---

## 9. 자동화는 작은 반복부터 시작한다

처음부터 다음을 한꺼번에 만들 필요는 없다.

```text
Task Router
Scheduler
Multi-Agent Manager
Memory
Observability Platform
자동 Merge
```

먼저 반복되고 검증 가능한 부분을 찾는다.

예:

```text
Cloud Runner로 Test 분리
→ Result Summary 생성
→ CI Failure에서 Agent-on-failure
→ Prepared Environment 정리
→ Task Contract 표준화
→ 반복 Routing 일부 자동화
```

각 단계에서 실제로 Developer Blocking Time, Retry, Review Cost가 줄었는지 확인한다.

복잡한 Platform보다 반복되는 수동 결정을 하나씩 코드로 옮기는 편이 이 책의 방향에 맞다.

---

## 10. 이 책의 범위는 여기까지다

Cloud Agent를 다루다 보면 다음 주제로 쉽게 확장된다.

```text
Agent Memory Architecture
Agent Security Platform
Agent OS
Agent Chaos Engineering
Shadow / Canary Agent
Agent Governance Platform
General Multi-Agent Theory
Agent 조직론
```

이 주제는 후속 연구로 남긴다.

현재 책의 질문은 끝까지 하나다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

이 질문에 직접 필요하지 않은 Platform 일반론은 여기서 확장하지 않는다.

---

## 11. 최종 Workflow와 마지막 판단

책 전체의 실행 흐름을 마지막으로 한 번만 정리한다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture
        ↓
Task Split / Routing
        ↓
Task Contract / Git Handoff
        ↓
Prepared Cloud Environment
        ↓
Cloud Runner
   ├─ PASS → Evidence
   └─ FAIL
        ↓
   Reasoning Required?
      ├─ NO → Tool / Retry / Escalation
      └─ YES
            ↓
       Cloud Agent
            ↓
           Fix
            ↓
      Cloud Runner
        ↓
Evidence / PR
        ↓
Local Review / Internal Validation
        ↓
Merge
```

독자가 마지막에 답할 수 있어야 하는 질문은 다음과 같다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
Cloud 이점이 사라지면 언제 Local로 돌아올 것인가?
```

그리고 실제 개발에서는 다음 문장이 자연스럽게 나와야 한다.

```text
이 설계는 Local에서 하자.
이 검증은 Cloud Runner로 보내자.
이 재현 가능한 실패는 Cloud Agent에게 맡기자.
이 마지막 검증은 내부망에서 하자.
```

> 더 많은 Agent보다 더 나은 Task Routing, Harness, Validation이 먼저다.

Cloud Agent는 Local Agent를 없애지 않는다. Cloud Runner나 CI도 없애지 않는다.

각 역할을 분리하고 Task 특성에 맞는 실행 위치에 배치한다.

이 책의 최종 목적은 독자가 자신의 개발 흐름에서 **Local과 Cloud의 역할을 직접 나눌 수 있게 하는 것**이다.

# 부록 A. Task Contract / Evidence / Handoff 템플릿

이 부록은 본문의 개념을 다시 설명하지 않는다. Local과 Cloud 사이에서 Task를 넘기고 결과를 검증할 때 바로 복사해 사용할 수 있는 최소 템플릿만 제공한다.

프로젝트 상황에 맞게 필드를 줄이거나 추가할 수 있다. 다만 Source 상태와 Validation Result가 연결되는 관계는 유지한다.

## A.1 Task Contract

```yaml
task_id: AUTH-142
base_sha: abc123

goal: >
  만료된 JWT로 Attendance API를 호출하면 HTTP 401을 반환한다.

scope:
  modules:
    - auth
    - attendance

relevant_files:
  - auth/src/main/java/.../AuthService.java
  - auth/src/main/java/.../JwtTokenProvider.java
  - auth/src/test/java/.../AuthServiceTest.java

do_not_change:
  - DB Schema
  - OAuth 전체 구조
  - 공통 Exception format

environment: backend-test

validation:
  - ./gradlew test --tests AuthServiceTest

expected_result:
  - expired token -> 401
  - normal token -> existing tests PASS

expected_output:
  - Result SHA
  - Changed Files
  - Validation Result
  - Artifact Reference
  - Limitations
```

핵심은 Prompt 길이가 아니라 다음 질문에 답하는 것이다.

```text
어디서 시작하는가?
무엇을 바꾸는가?
무엇을 바꾸면 안 되는가?
무엇이 성공인가?
무엇을 돌려받아야 하는가?
```

## A.2 Cloud Runner 입력

```yaml
task_id: AUTH-142
sha: def456
environment: backend-test
command: ./gradlew test --tests AuthServiceTest
timeout_seconds: 1800
artifact_path: artifacts/AUTH-142/def456/
```

Runner 입력에는 자연어 설명보다 어떤 Source 상태에서 어떤 명령을 실행하는지가 중요하다.

## A.3 Cloud Runner 출력

```yaml
task_id: AUTH-142
result_sha: def456
status: FAIL
exit_code: 1
duration_seconds: 73
validation_result: FAILED
artifact_path: artifacts/AUTH-142/def456/
```

실패가 발생하면 Result Gateway가 Agent가 먼저 읽을 작은 Failure Summary를 만든다.

```yaml
failure:
  type: TEST_FAILURE
  test: AuthServiceTest.expiredToken
  expected: 401
  actual: 200
  top_frame: AuthServiceTest.java:94
  fingerprint: TEST_FAILURE/401-200/AuthServiceTest:94
```

## A.4 Evidence 결과

```json
{
  "task_id": "AUTH-142",
  "base_sha": "abc123",
  "result_sha": "def456",
  "validation_result": "PASS",
  "changed_files": [
    "AuthService.java",
    "AuthServiceTest.java"
  ],
  "checks": {
    "target_test": "PASS",
    "module_test": "PASS",
    "integration": "PASS"
  },
  "artifacts": [
    "artifacts/AUTH-142/def456/result.json",
    "artifacts/AUTH-142/def456/junit.xml"
  ],
  "limitations": []
}
```

Evidence는 원본 Artifact를 대체하지 않는다. 처음 판단하는 데 필요한 정보만 작게 제공한다.

## A.5 Local → Cloud Handoff Package

```yaml
task_id: AUTH-142
repository: campus-platform
base_sha: abc123
branch: agent/auth-142
contract: task-contract.yaml
environment: backend-test
validation:
  - ./gradlew test --tests AuthServiceTest
expected_evidence:
  - Result SHA
  - Changed Files
  - Validation Result
  - Artifact Reference
```

Multi-Repository Task라면 Repository별 Base SHA를 따로 기록한다.

```yaml
repositories:
  campus-api: aaa111
  campus-common: bbb222
```

## A.6 Cloud → Local Return Package

```yaml
task_id: AUTH-142
base_sha: abc123
result_sha: def456
branch: agent/auth-142
validation_result: PASS
changed_files:
  - AuthService.java
  - AuthServiceTest.java
artifact_path: artifacts/AUTH-142/def456/
pr: 142
limitations:
  - Tibero 실제 환경 검증 필요
```

Local에서는 최소한 다음 관계를 확인한다.

```text
Result SHA
=
Evidence가 검증한 SHA
=
PR에서 Review하는 SHA
```

코드가 변경됐다면 필요한 Validation을 새 SHA에서 다시 실행한다.

## A.7 Local Fallback Package

Cloud 이점이 사라졌을 때 현재 작업을 버리지 않고 Local에서 이어가기 위한 반환 형식이다.

```yaml
task_id: HSM-37
status: LOCAL_FALLBACK
base_sha: abc123
result_sha: def456

validation_result:
  unit: PASS
  mock_contract: PASS

failed_command: null
failure_fingerprint: null

artifact_path: artifacts/HSM-37/def456/

attempted_fixes:
  - HSM 호출 경계를 Mock으로 분리

fallback_reason: actual HSM required

remaining:
  - 실제 HSM session validation
```

Cloud에서 확인한 사실과 아직 확인하지 못한 경계를 분리해서 남긴다.

## A.8 Event-driven Task Candidate

Event는 바로 Agent 호출이 아니라 Task Candidate로 변환한다.

```yaml
source_event: ci_failure
repository: campus-platform
git_sha: abc123
task_type: test_failure_fix
failure: AuthServiceTest.expiredToken
failure_fingerprint: TEST_FAILURE/401-200/AuthServiceTest:94
environment: backend-test
validation: ./gradlew test --tests AuthServiceTest.expiredToken
status: candidate
```

이후 공통 경로를 사용한다.

```text
Event
→ Task Candidate
→ Dedup / Classification
→ Cloud Runner / Tool
→ 판단이 필요할 때 Cloud Agent
→ Cloud Runner 재검증
→ Evidence / PR
```

## A.9 최소 운영 상태

처음부터 별도 Agent Platform을 만들지 않아도 다음 상태만 추적하면 기본 Workflow를 운영할 수 있다.

```yaml
task_id: AUTH-142
base_sha: abc123
branch: agent/auth-142
result_sha: def456
execution: cloud-agent
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/def456/
pr: 142
```

책 전체에서 유지하는 기본 관계는 다음과 같다.

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ Evidence / Artifact
→ PR
```

Template을 복잡하게 만드는 것보다 이 관계가 끊기지 않는 것이 중요하다.


# 용어집

이 용어집은 책 전체에서 반복해서 사용하는 용어의 의미를 고정하기 위한 것이다. 특정 제품의 명칭이나 구현 세부사항은 포함하지 않는다.

## Agent-on-failure

Cloud Runner가 먼저 검증을 실행하고, 재현 가능한 실패 중 코드 판단이 필요한 경우에만 Cloud Agent를 호출하는 방식.

## Artifact

Test Report, Log, Screenshot, Video, Trace처럼 실행 결과를 보존한 원본 또는 상세 결과물. Agent가 처음 읽는 작은 Evidence와 구분한다.

## Base SHA

Cloud Task가 시작한 Git 기준점. 어떤 Source 상태에서 작업을 시작했는지 식별한다.

## Cloud Agent

Repository와 독립 실행환경을 사용해 Task를 수행하는 Remote Development Worker. 이 책에서는 판단과 제한된 코드 수정이 필요한 구간에 사용한다.

## Cloud Runner

Build, Test, E2E, Docker Build처럼 명령과 판정 기준이 정해진 작업을 실행하는 Cloud 실행 주체. LLM 판단이 필요하지 않은 결정론적 경로를 우선 담당한다.

## Cloud Session

Repository, Workspace, 실행환경, Tool, Agent Interaction이 결합된 하나의 Cloud 작업 실행 단위. 제품별 Session 명칭과 정확히 일치한다는 의미는 아니다.

## Context Duplication

여러 Agent가 같은 Repository, 문서, 공통 Source를 반복해서 읽으면서 생기는 중복 Context와 탐색 비용.

## Developer Blocking Time

Cloud Task의 총 실행시간이 아니라 개발자가 해당 Task 때문에 다른 일을 진행하지 못하고 기다린 시간.

## Evidence

작업 결과를 검증할 수 있도록 정리한 작은 구조화 결과. Result SHA, Validation Result, 실패 요약, Artifact Reference 등이 포함될 수 있다.

## Failure Fingerprint

동일한 실패가 반복되는지 식별하기 위해 Test, Error Type, 핵심 Message, 위치 같은 정보를 조합한 실패 식별 정보.

## Fan-out

서로 독립적인 Task나 검증을 여러 Runner 또는 Worker로 나누어 동시에 실행하는 것.

## Fan-in

병렬 실행된 결과를 다시 합치고 Review, Merge, Regression, Integration Validation을 수행하는 단계.

## Handoff

Task와 Source 상태를 한 실행 위치에서 다른 실행 위치로 넘기는 과정. 이 책의 기본 경계는 Git, Task Contract, Evidence다.

## Harness

Agent가 반복해서 탐색하거나 추론해야 했던 개발 규칙을 발견 가능하고 실행 가능한 형태로 제공하는 구성. 예를 들어 표준 Build/Test Script, AGENTS.md, Task Contract, Result Gateway가 포함될 수 있다.

## Human Steering

작업 도중 사람이 방향을 자주 확인하고 수정해야 하는 정도. Human Steering이 높을수록 비동기 Cloud 위임의 이점이 줄어들 수 있다.

## Local Agent

개발자의 현재 Workspace와 짧은 Feedback Loop 안에서 사용하는 Agent. 요구사항 탐색, Architecture, 내부 자원 접근, Human Steering이 많은 작업에 유리할 수 있다.

## Local Fallback

Cloud Task를 계속 유지하는 이점이 사라졌을 때 현재 결과와 Evidence를 보존한 채 Local 또는 Hybrid Workflow로 이동하는 것. 실패가 아니라 재Routing의 한 형태다.

## Orchestration

Task Classification, Environment Selection, Runner / Agent Selection, Retry, Validation, Evidence 연결처럼 반복되는 실행 결정을 Workflow로 묶는 것.

## Prepared Environment

Task가 시작될 때 Runtime, Tool, Dependency, Cache가 이미 준비되어 있어 Agent가 개발환경 설치부터 반복하지 않도록 만든 Cloud 실행환경.

## Progressive Context

처음에는 Relevant Files 같은 작은 Context만 제공하고, Task 수행에 실제로 필요한 경우에만 Direct Dependency, 관련 문서, 넓은 Module Context 순으로 확장하는 방식.

## Result Gateway

대형 Tool Output과 Artifact를 그대로 Agent에게 전달하지 않고, 필요한 결과를 작은 구조화 Evidence로 변환하거나 필요한 상세 결과를 선택적으로 조회하게 하는 경계.

## Result SHA

Cloud Agent 또는 작업 결과로 만들어진 Git Commit의 SHA. Validation Result와 Evidence는 가능한 한 이 Source 상태와 연결한다.

## Runner-first

검증 가능한 작업은 먼저 Cloud Runner나 기존 CI가 실행하고, LLM 판단은 필요한 예외 경로에만 사용하는 원칙.

## Task

Cloud 또는 Local 실행 위치에 위임할 수 있도록 범위와 완료 조건을 가진 작업 단위.

## Task Candidate

Issue, CI Failure, Review Comment, Schedule 같은 Event에서 만들어진 작업 후보. Event가 발생했다고 즉시 Agent Task가 되는 것은 아니며 Dedup, Classification, Routing을 거친다.

## Task Contract

Cloud Worker가 독립적으로 작업을 시작할 수 있도록 Goal, Scope, Relevant Files, Forbidden Changes, Validation, Expected Result, Output / Evidence 등을 정의한 입력 경계.

## Task Routing

Task의 특성과 제약을 기준으로 Local, Cloud Runner, Cloud Agent, Hybrid 중 실행 위치와 실행 주체를 결정하는 과정.

## Validation Result

특정 Source 상태에서 실행한 Build/Test/E2E 등의 검증 결과. PASS / FAIL뿐 아니라 어떤 명령과 범위를 실행했는지 추적할 수 있어야 한다.

## Hybrid Workflow

하나의 Task나 기능을 단계별로 Local과 Cloud에 나누어 수행하는 방식. 예를 들어 Cloud에서 일반 검증을 끝낸 뒤 Tibero, HSM, VPN-only API 같은 내부 자원 검증을 Local에서 이어갈 수 있다.

## 이 책에서 유지하는 기본 관계

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ Evidence / Artifact
→ PR
```

실행 주체는 다음처럼 구분한다.

```text
Local / Local Agent
→ 요구사항 / Architecture / Human Steering / Internal Validation / Review

Cloud Runner
→ Build / Test / E2E / Docker / 결정론적 검증

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정
```


# 참고자료

기준일: 2026-09-16

이 책은 특정 제품의 기능 목록을 설명하는 책이 아니다. 참고자료는 Cloud Agent의 실행 모델, Local / Cloud 경계, 실행환경, Runner-first, Event-driven Workflow 같은 본문 원칙을 확인하는 데 사용한 공식 자료만 정리한다.

제품 UI, 가격, CPU / RAM, 동시 Session 제한처럼 변경 가능성이 높은 정보는 출판 직전에 다시 확인한다.

## GitHub Copilot cloud agent

- GitHub Docs, `About GitHub Copilot cloud agent`
  - https://docs.github.com/en/copilot/concepts/agents/cloud-agent/about-cloud-agent
- GitHub Docs, `Customize the development environment for Copilot cloud agent`
  - https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/customize-the-agent-environment
- GitHub Docs, `Get the best results with Copilot cloud agent`
  - https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results
- GitHub Docs, `About cloud and local sandboxes for GitHub Copilot`
  - https://docs.github.com/en/copilot/concepts/about-cloud-and-local-sandboxes
- GitHub Docs, `About the GitHub Copilot app`
  - https://docs.github.com/en/copilot/concepts/agents/github-copilot-app

본문에서 확인하는 범위:

- 독립된 Cloud 개발환경
- Repository 탐색과 코드 변경
- Test / Lint 실행
- 개발환경 사전 구성
- Issue / PR / Automation과 연결되는 비동기 작업
- Local / Cloud 실행 위치 구분

## OpenAI Codex

- OpenAI, `Addendum to OpenAI o3 and o4-mini system card: Codex`, 2025-05-16
  - https://openai.com/index/o3-o4-mini-codex-system-card-addendum/
- OpenAI, `Codex is now generally available`, 2025-10-06
  - https://openai.com/index/codex-now-generally-available/

본문에서 확인하는 범위:

- Cloud container / sandbox 기반 Coding Task
- Repository와 Development Environment 제공
- 파일 수정과 명령 실행
- Test / Lint / Type Check 실행
- Diff 검토와 Pull Request 연결

## Anthropic Engineering

- Anthropic Engineering, `Quantifying infrastructure noise in agentic coding evals`, 2026-02-05
  - https://www.anthropic.com/engineering/infrastructure-noise
- Anthropic Engineering, `Scaling Managed Agents: Decoupling the brain from the hands`, 2026-04-08
  - https://www.anthropic.com/engineering/managed-agents
- Anthropic Engineering, `Building a C compiler with a team of parallel Claudes`, 2026-02-05
  - https://www.anthropic.com/engineering/building-c-compiler

본문에서 확인하는 범위:

- 실행 자원이 Agent 성능에 영향을 줄 수 있다는 점
- Reasoning과 Execution Environment의 Lifecycle 분리
- Agent가 읽기 쉬운 Test Output과 Artifact 설계
- 독립 Task 병렬화와 Sequential Bottleneck의 차이

공개된 실험 수치는 해당 실험의 결과로만 사용하며 일반적인 성능 기대값으로 확대하지 않는다.

## GitHub Engineering / Continuous AI 사례

- GitHub Blog, `Continuous AI in practice: What developers can automate today with agentic CI`
  - https://github.blog/ai-and-ml/generative-ai/continuous-ai-in-practice-what-developers-can-automate-today-with-agentic-ci/
- GitHub Blog, `Automate repository tasks with GitHub Agentic Workflows`
  - https://github.blog/ai-and-ml/automate-repository-tasks-with-github-agentic-workflows/
- GitHub Next, `Continuous AI`
  - https://githubnext.com/projects/continuous-ai/
- GitHub Blog, `Improving token efficiency in GitHub Agentic Workflows`
  - https://github.blog/ai-and-ml/github-copilot/improving-token-efficiency-in-github-agentic-workflows/
- GitHub Changelog, `Schedule and automate tasks with Copilot cloud agent`, 2026-06-02
  - https://github.blog/changelog/2026-06-02-schedule-and-automate-tasks-with-copilot-cloud-agent/
- GitHub Changelog, `GitHub Agentic Workflows are now in technical preview`, 2026-02-13
  - https://github.blog/changelog/2026-02-13-github-agentic-workflows-are-now-in-technical-preview/

본문에서 확인하는 범위:

- Build / Test / Lint 같은 결정론적 작업과 Agent 판단의 분리
- 정상 경로에서 불필요한 LLM 호출을 제거하는 방식
- 작은 검증 가능한 Task와 PR 단위 반복
- CI Failure / Review / Schedule 같은 Event에서 필요할 때 Agent를 호출하는 방식

## Repository 내부 Research

출판 원고의 상세 근거와 재검증 기록은 다음 문서에서 관리한다.

- `research/chapter-01-cloud-worker-official-sources.md`
- `research/chapter-02-local-cloud-official-sources.md`
- `research/github/continuous-ai-runner-first.md`
- `research/anthropic/agent-native-development-environment.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/infrastructure-noise.md`

## 출판 시 확인 규칙

참고자료의 URL과 제품 기능은 출판 직전에 다시 확인한다. URL이 변경되더라도 본문의 일반 원칙과 특정 제품의 현재 기능을 같은 것으로 취급하지 않는다.

본문은 다음과 같은 구조적 원칙을 유지한다.

```text
Task Routing
→ Prepared Environment
→ Runner-first
→ 필요한 경우 Cloud Agent
→ Evidence
→ Local Review / Internal Validation
```

제품은 이 원칙을 설명하는 근거와 사례이며, 책의 정의 자체는 아니다.


