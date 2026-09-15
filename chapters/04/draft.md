# 4장. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

Cloud Agent를 사용하는 이유를 단순히 `AI가 코드를 대신 작성해준다`로 설명하면 Cloud의 장점이 잘 보이지 않는다.

Local Agent도 코드를 읽고 수정할 수 있다. 좋은 모델이라면 Local에서도 충분히 복잡한 문제를 해결할 수 있다.

그렇다면 굳이 Cloud에서 Agent를 실행해야 하는 이유는 무엇인가.

이 장에서는 그 이유를 다음 네 가지로 정리한다.

```text
독립 실행환경
장시간 작업의 비동기 위임
서로 독립적인 작업의 병렬 실행
Agent 실행시간과 개발자 대기시간의 분리
```

핵심은 `Cloud Agent가 더 많이 생각한다`가 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

그리고 이 장에서 하나를 더 기억해야 한다.

> Cloud Agent가 오래 실행되는 것과 개발자가 오래 기다리는 것은 같은 의미가 아니다.

이 차이를 이해하면 Cloud Agent를 단순 Coding Assistant가 아니라 개발 Workflow의 실행 노드로 보기 시작하게 된다.

---

## 1. 독립 실행환경이 Cloud Agent의 첫 번째 가치다

Local Agent는 보통 개발자의 현재 컴퓨터를 함께 사용한다.

예를 들어 개발자의 Mac에서 다음 프로그램이 동시에 실행되고 있다고 하자.

```text
Developer Mac
├─ IDE
├─ Local Agent
├─ Docker
├─ Database
├─ Browser
├─ Android Emulator
└─ Gradle Daemon
```

여기에 전체 테스트를 실행한다.

```bash
./gradlew test
```

Integration Test가 Testcontainers까지 사용한다면 Docker Container도 추가된다.

Frontend가 있다면 Browser E2E가 같이 실행될 수 있다.

```text
Local Machine
├─ Java Compile
├─ Unit Test
├─ Integration Test
├─ PostgreSQL Container
├─ Redis Container
├─ Playwright Browser
└─ Docker Build
```

이 작업은 모두 개발자의 CPU, RAM, Disk를 사용한다.

Agent 자체가 빠르더라도 Local 실행환경이 포화되면 개발자는 다음 작업을 하기 불편해진다.

Cloud에서는 이 실행환경을 분리할 수 있다.

```text
Developer Mac
├─ IDE
├─ Local Agent
└─ 현재 기능 개발

Cloud Worker
├─ Repository
├─ CPU
├─ RAM
├─ Disk
├─ Gradle
├─ Docker
└─ Test Runtime
```

Cloud Worker가 전체 테스트를 실행하는 동안 Local 환경은 현재 개발 작업에 사용할 수 있다.

이 장에서 말하는 `독립 실행환경`의 가장 기본적인 의미다.

Cloud가 항상 Local보다 빠르다는 뜻은 아니다.

중요한 것은 **작업 자원을 분리할 수 있다**는 점이다.

---

## 2. Cloud Worker는 독립된 작업 노드다

1장에서 Cloud Agent를 다음처럼 정의했다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

이를 실제 작업 관점으로 다시 보면 다음과 같다.

```text
Cloud Worker
= Task
+ Repository State
+ Runtime
+ Compute
+ Tools
+ Result
```

Developer가 Cloud Worker에게 넘기는 것은 Prompt 한 줄이 아니다.

실제로 필요한 것은 다음과 같은 작업 단위다.

```text
Task
Base Git State
Context
Execution Environment
Validation
Expected Result
Output / Evidence
```

예를 들어 다음 Task를 보자.

```text
Task
AuthService expired token 수정

Base SHA
abc123

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected
HTTP 401
```

Cloud Worker는 이 기준점에서 독립적으로 작업한다.

Local 개발자는 같은 시간에 다른 Branch나 다른 기능을 계속 작업할 수 있다.

즉 Remote Worker의 핵심은 `사람 대신 대화하는 AI`가 아니다.

**현재 개발자의 Workspace와 분리된 상태에서 Task를 끝낼 수 있는 실행 단위**다.

Task Contract의 상세 형식은 7장에서 다룬다.

---

## 3. 장시간 작업은 Local에서 분리할 가치가 크다

Cloud에 보내기 좋은 작업 중 하나는 시간이 오래 걸리지만 중간에 사람의 판단이 거의 필요하지 않은 작업이다.

대표적인 예는 다음과 같다.

```text
전체 Unit Test
Integration Test
Docker Image Build
Frontend E2E Regression
Static Analysis
Migration Validation
반복 Refactoring 후 전체 검증
```

예를 들어 Integration Test가 25분 걸린다고 하자.

Local에서 실행하면 다음처럼 될 수 있다.

```text
Developer
→ Integration Test 시작
→ CPU/RAM 사용 증가
→ Docker Container 다수 실행
→ 결과를 기다림
```

개발자가 그동안 다른 작업을 할 수 있다고 하더라도 Local 환경의 자원을 같이 사용하므로 영향이 생길 수 있다.

Cloud로 보내면 작업을 분리할 수 있다.

```text
Developer
→ Cloud Task 생성
→ 다음 Feature 개발

Cloud Worker
→ Integration Test
→ Result / Artifact 생성
```

여기서 중요한 것은 `25분짜리 작업은 무조건 Cloud`라는 규칙이 아니다.

다음 조건이 같이 맞아야 한다.

```text
Scope가 명확한가?
중간 Human Steering이 적은가?
독립적으로 검증 가능한가?
Git을 통해 입력과 결과를 전달 가능한가?
```

장시간 실행이라는 이유 하나만으로 Cloud가 적합해지는 것은 아니다.

이 판단 기준은 5장에서 구체화한다.

---

## 4. Agent Execution Time과 Developer Blocking Time은 다르다

Cloud Agent의 성능을 평가할 때 흔히 `몇 분 걸렸는가`만 본다.

그러나 비동기 작업에서는 이 지표만으로 실제 개발 생산성을 설명하기 어렵다.

다음 상황을 보자.

```text
10:00
Developer가 Cloud Agent에 테스트 보강 Task 위임

10:01
Developer는 다음 Feature 개발 시작

10:40
Cloud Task 완료

11:20
Developer가 결과 Review
```

Cloud Task는 40분 동안 실행됐다.

하지만 Developer가 40분 동안 기다린 것은 아니다.

따라서 두 값을 분리한다.

```text
Agent Execution Time
= Cloud Task가 시작해서 완료될 때까지의 시간

Developer Blocking Time
= Developer가 해당 Task 때문에 실제로 멈춰 있던 시간
```

Cloud Agent가 40분 걸렸더라도 Developer Blocking Time은 Task를 넘기는 시간과 Review 시간 몇 분일 수 있다.

이 차이는 Cloud Agent를 `응답 속도`만으로 평가해서는 안 되는 이유다.

---

## 5. 작업시간이 길어도 전체 Workflow는 빨라질 수 있다

두 가지 개발 흐름을 비교해 보자.

### Local 순차 실행

```text
10:00 코드 수정 완료
10:01 전체 테스트 시작
10:31 테스트 완료
10:32 Docker Build 시작
10:44 Docker Build 완료
10:45 다음 Feature 개발
```

Developer의 다음 작업 시작은 10:45다.

### Cloud 비동기 실행

```text
10:00 코드 수정 완료
10:01 Cloud 검증 Task 생성
10:02 다음 Feature 개발 시작

Cloud
10:02 Unit / Integration / Docker 검증
10:35 결과 완료
```

Cloud 검증이 Local보다 몇 분 더 늦게 끝났다고 해도 Developer는 10:02부터 다음 작업을 시작할 수 있다.

그래서 Cloud Workflow에서는 다음 값들을 같이 봐야 한다.

```text
Agent Execution Time
Queue Time
Developer Blocking Time
Review Time
Retry Time
Local Resource Occupancy
```

Cloud Agent 도입의 목표가 `Agent 자체의 초당 속도`만은 아니라는 뜻이다.

---

## 6. Local Resource Occupancy도 개발 비용이다

개발자의 컴퓨터가 충분히 빠르더라도 동시에 실행되는 작업이 많아지면 개발 경험이 달라진다.

예를 들어 다음 조합을 생각해 보자.

```text
IDE
Local Agent
Docker
PostgreSQL
Redis
Browser
Android Emulator
```

여기에 다음이 추가된다.

```text
Gradle 전체 Test
Testcontainers
Docker Build
Playwright E2E
```

메모리가 부족하지 않더라도 CPU와 Disk I/O가 몰릴 수 있다.

팬이 돌고, Build가 느려지고, Browser Test가 다른 작업에 영향을 줄 수도 있다.

Cloud Worker로 무거운 검증을 분리하면 Local에서는 다음 작업에 집중할 수 있다.

```text
코드 탐색
Architecture 판단
빠른 수정 반복
내부망 작업
최종 Review
```

따라서 Cloud Agent를 사용할 때는 Token 사용량 외에 `Local Resource Occupancy`도 비용으로 본다.

---

## 7. 비동기 위임에 적합한 Task는 따로 있다

Cloud에 오래 맡길 수 있는 Task는 다음 성격을 가진다.

```text
Task가 명확함
완료 조건이 있음
중간 질문이 적음
독립 검증 가능
결과를 Git/Artifact로 반환 가능
```

예를 들면 다음과 같다.

```text
AuthService 테스트 추가
attendance 모듈 Java 21 호환성 수정
Docker Build 검증
E2E Regression
Dependency Update 검증
```

반대로 다음 작업은 비동기 위임에 적합하지 않을 수 있다.

```text
새 인증 Architecture를 어떻게 할지 같이 정해보자.
왜 Production이 가끔 느린지 조사해보자.
UI 방향을 여러 안으로 비교해보자.
```

이런 작업은 중간 판단과 질문이 반복된다.

Developer와 Agent가 짧은 주기로 대화하면서 범위를 좁히는 편이 더 낫다.

따라서 Cloud의 비동기성은 `사람이 없어도 계속 실행할 수 있는 Task`에서 가치가 크다.

---

## 8. 병렬화의 대상은 Agent가 아니라 Task다

Cloud Agent를 사용하다 보면 `Agent를 몇 개까지 동시에 띄울 수 있는가`에 관심을 가지기 쉽다.

그러나 병렬성을 결정하는 첫 번째 조건은 Agent 수가 아니다.

> 서로 독립적으로 실행하고 검증할 수 있는 Task가 몇 개인가?

예를 들어 하나의 Commit을 다음처럼 검증할 수 있다.

```text
Git SHA: abc123
        |
        +-- Task A: Unit Test
        +-- Task B: Integration Test
        +-- Task C: E2E
        +-- Task D: Docker Build
```

이 네 작업은 서로 결과를 기다리지 않아도 된다면 병렬로 실행할 수 있다.

```text
                abc123
                  |
      +-----------+-----------+-----------+
      |           |           |           |
   Unit Test  Integration    E2E      Docker Build
      |           |           |           |
      +-----------+-----------+-----------+
                  |
               Evidence
```

이 구조에서는 네 개의 LLM이 필요하지 않을 수도 있다.

대부분은 Cloud Runner 네 개면 충분하다.

실패 분석이나 코드 수정이 필요한 Task에만 Agent가 붙는다.

즉 Cloud의 병렬성은 먼저 **병렬 Compute**로 생각하는 것이 좋다.

---

## 9. 좋은 병렬화는 서로 방해하지 않는다

좋은 병렬 Task는 입력과 완료 조건이 서로 독립적이다.

예:

```text
Cloud #1
→ Backend Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Frontend E2E

Cloud #4
→ Docker Build
```

이 작업들은 같은 Git SHA를 읽지만 Source를 수정하지 않는 검증 작업일 수 있다.

또는 서로 다른 Module을 수정할 수도 있다.

```text
Agent A
→ attendance 모듈 deprecated API 변경

Agent B
→ notification 모듈 deprecated API 변경
```

두 모듈이 충분히 분리되어 있고 각각 별도 테스트가 있다면 병렬화하기 쉽다.

좋은 병렬화의 조건을 정리하면 다음과 같다.

```text
File Scope가 분리됨
Dependency가 약함
완료 조건이 독립적임
개별 검증 가능
Merge 순서 의존성이 낮음
```

---

## 10. 나쁜 병렬화는 작업 수만 늘린다

다음 구조를 보자.

```text
Agent A
→ UserService 구조 변경

Agent B
→ UserService 구조 변경

Agent C
→ UserService 테스트 구조 변경
```

세 Agent가 각각 작업에 성공하더라도 결과를 합치는 과정에서 문제가 생길 수 있다.

```text
Merge Conflict
Architecture Decision 충돌
Test 기대값 불일치
Review 증가
```

DB Schema도 비슷하다.

```text
Agent A
→ V121 migration 생성

Agent B
→ V121 migration 생성
```

각자 독립 환경에서는 성공하지만 통합 시 충돌한다.

Cloud Worker가 격리되어 있다는 사실은 **논리적 충돌까지 없애주지 않는다**.

따라서 병렬화 전에 Task Dependency를 먼저 본다.

```text
Independent Task
→ Parallelize

Dependent Task
→ Sequence 또는 재분해
```

---

## 11. Agent를 10개 띄워도 10배 빨라지지 않는다

병렬 Worker가 늘어날수록 다음 비용도 같이 증가한다.

```text
Repository Checkout
Environment Setup
Context Loading
LLM Usage
PR 수
Review 수
Merge Conflict
결과 통합
```

예를 들어 동일 Repository를 Agent 10개가 모두 처음부터 읽는다고 하자.

```text
Agent 1 → README / Architecture / Source 탐색
Agent 2 → README / Architecture / Source 탐색
...
Agent 10 → README / Architecture / Source 탐색
```

Task는 병렬이지만 Context 탐색 비용도 10번 발생할 수 있다.

따라서 Cloud 병렬화의 효과는 선형이 아니다.

병렬성이 유리한 지점과 Integration Cost가 더 커지는 지점이 존재한다.

이 비용은 12장에서 더 구체적으로 다룬다.

---

## 12. Parallel Compute와 Parallel Reasoning을 구분한다

3장에서 두 개념을 소개했다.

다시 비교해 보자.

### Parallel Compute

```text
Runner A → Unit Test
Runner B → Integration Test
Runner C → E2E
Runner D → Docker Build
```

입력은 명확하고 각 Runner가 정해진 명령을 실행한다.

### Parallel Reasoning

```text
Agent A → Repository 전체를 보고 개선안 작성
Agent B → Repository 전체를 보고 개선안 작성
Agent C → Repository 전체를 보고 개선안 작성
```

두 번째 구조는 경우에 따라 유용할 수 있지만 Context와 Token 사용이 훨씬 커질 수 있다.

Cloud를 잘 사용하는 기본 전략은 먼저 가능한 Compute를 병렬화하고, 정말 필요한 판단에만 Agent를 사용한다는 것이다.

```text
먼저 Runner 병렬화
      ↓
실패/판단 필요 지점 식별
      ↓
Agent 호출
```

---

## 13. Task가 완료되면 Evidence를 돌려받는다

비동기 작업의 문제는 Developer가 작업 과정을 계속 보지 않는다는 점이다.

따라서 완료 시 `무엇을 했는지`보다 `무엇으로 검증했는지`가 중요해진다.

예:

```text
Cloud #1
Unit Test
→ 1,284 / 1,284 PASS

Cloud #2
Integration Test
→ 143 / 143 PASS

Cloud #3
E2E
→ 87 / 88 PASS
→ failure screenshot

Cloud #4
Docker Build
→ PASS
→ image digest
```

Developer는 모든 로그를 처음부터 읽지 않고 Evidence를 기준으로 다음 행동을 결정할 수 있다.

E2E 한 건이 실패했다면 해당 실패만 추가 분석한다.

```text
E2E FAIL
→ Failure Summary
→ Agent 분석
```

Evidence의 상세 형식은 8장에서 다룬다.

---

## 14. campus-platform 검증을 병렬로 나눠보자

`campus-platform`에서 출결 API를 수정했다고 가정하자.

Local에서 핵심 코드를 수정하고 Commit을 만든다.

```text
Local
→ Attendance API 수정
→ Auth 처리 수정
→ Commit abc123
```

그다음 Cloud 검증을 분리한다.

```text
abc123
 |
 +-- Cloud Runner #1
 |    :attendance:test
 |
 +-- Cloud Runner #2
 |    integrationTest
 |
 +-- Cloud Runner #3
 |    admin-web E2E
 |
 +-- Cloud Runner #4
      Docker Build
```

각 Worker는 독립적인 실행환경을 사용한다.

Developer는 이 동안 다음 기능을 진행할 수 있다.

최종 결과가 다음과 같다고 하자.

```text
Unit Test: PASS
Integration: PASS
E2E: FAIL 1
Docker Build: PASS
```

이제 Agent가 E2E 실패 한 건만 분석한다.

```text
Failure
attendance-expired-token-view

Artifact
screenshot.png
trace.zip
console.log
```

전체 검증을 하나의 Agent가 순서대로 수행하는 것보다 각 실행 특성에 맞는 Worker를 사용한 것이다.

---

## 15. Cloud가 느려도 가치가 있을 수 있다

Local Test가 20분, Cloud Test가 Cold Start 때문에 24분 걸린다고 하자.

순수 실행시간만 보면 Local이 빠르다.

하지만 Local에서는 Developer가 CPU/RAM 점유 때문에 다음 작업을 제대로 못 하고, Cloud에서는 바로 다음 기능을 개발할 수 있다면 결과는 달라진다.

```text
Local
Test: 20분
Developer Blocking: 15분

Cloud
Test: 24분
Developer Blocking: 2분
```

이 숫자는 설명을 위한 예시다.

중요한 것은 `Agent Execution Time` 하나만으로 Cloud의 가치를 판단하지 않는다는 점이다.

팀에서 실제로 측정해야 할 항목은 다음과 같다.

```text
Queue Time
Environment Startup Time
Execution Time
Developer Blocking Time
Review Time
Retry Time
Local Resource Occupancy
```

이 데이터를 보면 어떤 Task가 Cloud에 적합한지 더 현실적으로 판단할 수 있다.

---

## 16. 장시간 Task에는 중간 개입보다 완료 조건이 중요하다

비동기 위임은 사람이 과정을 계속 지켜보지 않는다는 전제에서 가치가 있다.

따라서 Task가 명확해야 한다.

좋지 않은 예:

```text
Repository를 전반적으로 개선하고 테스트해줘.
```

Agent가 무엇을 끝내야 하는지 불분명하다.

더 좋은 예:

```text
Task
attendance 모듈 Java 21 deprecated API 수정

Validation
./gradlew :attendance:test

Expected
all tests pass

Forbidden
DB schema 변경 금지
```

장시간 실행이라고 해서 Task Scope를 넓게 잡는 것이 아니다.

오히려 사람이 중간에 개입하지 않아도 되도록 완료 조건을 더 명확하게 만든다.

이 형식은 7장의 Task Contract에서 구체화한다.

---

## 17. Cloud Worker가 실패해도 Local 작업은 계속될 수 있다

비동기 Workflow의 또 다른 장점은 실패가 개발자의 현재 Workspace를 직접 망가뜨리지 않는다는 점이다.

예를 들어 Cloud Worker에서 Dependency Update를 시도했다고 하자.

```text
Cloud Branch
→ dependency update
→ build fail
```

Local의 현재 Branch는 그대로다.

Developer는 실패 결과를 확인하고 다음처럼 선택할 수 있다.

```text
Cloud Agent에게 수정 계속
Local에서 직접 확인
Task 폐기
새 Base SHA로 다시 시작
```

작업 단위가 분리되어 있기 때문에 실패한 시도를 폐기하기도 쉽다.

이 격리 구조는 11장의 Branch/Worktree/Container에서 상세히 다룬다.

---

## 18. 이 장에서 기억할 판단 기준

Cloud Agent를 사용할 때 다음 질문을 순서대로 생각해 볼 수 있다.

```text
이 작업은 Local 컴퓨팅 자원을 많이 점유하는가?
장시간 실행되지만 중간 질문은 적은가?
Git을 통해 입력 상태를 고정할 수 있는가?
독립적으로 검증 가능한가?
다른 Task와 충돌하지 않는가?
여러 독립 Task로 나눌 수 있는가?
Cloud 작업 중 Developer는 다른 일을 할 수 있는가?
```

이 질문에서 긍정적인 답이 많을수록 Cloud의 독립 실행환경과 비동기성이 실제 가치로 이어질 가능성이 높다.

그러나 다음 조건이 강하다면 Local이 더 나을 수 있다.

```text
Human Steering이 계속 필요함
내부망에 강하게 의존함
전체 시스템 Context가 필요함
다른 Task와 같은 파일/Schema를 동시에 수정함
Task가 너무 작음
```

다음 장에서는 이 판단을 `Task Routing`이라는 형태로 정리해 Local, Cloud, Hybrid, Runner-first 중 무엇을 선택할지 구체적으로 다룬다.

---

## 장 요약

Cloud Agent의 가장 큰 장점은 모델 자체보다 **실행환경을 분리할 수 있다는 점**에서 시작한다.

장시간 Build/Test를 Cloud로 보내면 개발자 PC의 CPU/RAM을 비울 수 있다.

Task가 서로 독립적이라면 여러 Worker에서 동시에 실행할 수 있다.

하지만 Agent 수를 늘린다고 생산성이 같은 비율로 증가하지는 않는다.

병렬화의 대상은 Agent가 아니라 독립 Task다.

그리고 Cloud Agent의 실행시간과 Developer가 실제로 기다린 시간은 따로 측정해야 한다.

```text
Cloud Agent 가치
=
Independent Environment
+ Async Delegation
+ Parallel Execution
+ Lower Developer Blocking Time
```

이 관점이 다음 장의 Task Routing 기준이 된다.
