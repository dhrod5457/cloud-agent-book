# 1장. Coding Agent에서 Cloud Worker로

소프트웨어 개발에서 AI를 사용할 때 먼저 구분해야 할 것은 `대화하는 AI`와 `작업하는 Agent`다.

Chat 형태의 LLM은 질문을 받고 답을 만든다. 코드 조각을 제안하거나 오류 메시지를 해석할 수 있다. Coding Agent는 여기서 한 단계 더 나아가 Repository를 읽고, 파일을 수정하고, 명령을 실행하며, Build와 Test 결과를 바탕으로 다시 작업한다.

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

오류를 재현하려면 테스트를 실행해야 하고, Mapper XML이나 설정 파일까지 확인해야 할 수도 있다.

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

테스트가 실패하면 Agent는 실패 내용을 확인하고 관련 파일을 수정한 뒤 다시 실행할 수 있다.

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

LLM이 얼마나 잘 판단하는지와, 어떤 환경에서 무엇을 실행할 수 있는지는 서로 다른 문제다. 3장에서 이 차이를 Compute와 Token 관점으로 분리한다.

---

## 4. Remote Developer보다 Remote Worker로 보는 편이 낫다

Cloud Agent를 `인터넷에 있는 AI 개발자`라고 표현하면 이해하기는 쉽다. 그러나 실제 Workflow를 설계할 때는 `Remote Worker` 관점이 더 유용하다.

개발자 한 명을 추가했다고 생각하면 다음과 같이 넓은 요청을 만들기 쉽다.

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

> Cloud Agent에게 넘기는 것은 Prompt보다 Task에 가깝다.

---

## 5. 입력과 출력 모두 작업 상태를 가진다

Cloud Task의 입력은 자연어 요청 하나로 끝나지 않는다.

보통 다음과 같은 작업 상태가 함께 필요하다.

```text
Task
Repository
Base Commit
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

> Cloud Agent에게 넘기는 것은 대화보다 Task에 가깝고, 결과는 검증 가능한 작업 상태여야 한다.

다음 장에서는 이 Remote Worker와 Local Agent를 비교해 실행 위치를 선택하는 기준을 만든다.

---

## 참고자료

제품별 기능은 변경될 수 있으므로 본문에서는 공통 실행 모델만 사용한다. 제품 사례의 세부 사실은 `research/chapter-01-cloud-worker-official-sources.md`에서 기준일과 출처를 관리한다.

- GitHub Docs, `About GitHub Copilot cloud agent`
- GitHub Docs, `Configure the development environment`
- OpenAI, Codex 관련 공식 자료
