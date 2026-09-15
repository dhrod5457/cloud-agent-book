# 1장. Coding Agent에서 Cloud Worker로

소프트웨어 개발에서 AI를 사용할 때 가장 먼저 구분해야 할 것은 `대화하는 AI`와 `작업하는 Agent`다.

Chat 형태의 LLM은 질문을 받고 답을 만든다. 코드 조각을 제안하거나 오류 메시지를 해석할 수도 있다. 그러나 답을 생성하는 것과 실제 Repository에서 작업을 수행하는 것은 다른 문제다.

Coding Agent는 Repository를 읽고, 파일을 수정하고, 명령을 실행하고, Build와 Test를 수행한다. Cloud Agent는 여기에 독립된 원격 실행환경까지 결합한다.

이 책에서는 Cloud Agent를 다음과 같이 정의한다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

즉 Cloud Agent는 단순히 `클라우드에서 실행되는 LLM`이 아니다.

필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 **Remote Development Worker**다.

이 정의가 중요한 이유는 이후의 비용, 병렬성, 테스트, Git, 개발환경 설계를 모두 다르게 보게 만들기 때문이다.

---

## 1. Chat LLM과 Coding Agent는 무엇이 다른가

개발자가 일반적인 LLM에 다음과 같이 질문한다고 가정해 보자.

```text
이 Java 코드에서 NullPointerException이 발생할 가능성이 있는지 확인해줘.
```

LLM은 전달받은 코드를 읽고 가능성을 설명할 수 있다.

하지만 실제 프로젝트의 문제는 코드 한 조각만으로 끝나지 않는다.

```text
UserService.java
UserMapper.java
UserMapper.xml
UserServiceTest.java
application-test.yml
DB migration
```

오류를 재현하려면 테스트를 실행해야 할 수도 있고, Mapper XML이나 설정 파일까지 확인해야 할 수도 있다.

Coding Agent는 여기서 한 단계 더 나아간다.

```text
Repository 탐색
→ 관련 파일 확인
→ 코드 수정
→ Build/Test 실행
→ 실패 확인
→ 재수정
→ 검증 결과 반환
```

중요한 차이는 `더 긴 답변을 생성한다`가 아니다.

Agent가 **Repository와 Tool을 사용해 상태를 변경한다**는 점이다.

예를 들어 다음 명령을 Agent가 직접 실행할 수 있다고 하자.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

테스트가 실패하면 Agent는 실패 내용을 확인하고 관련 파일을 수정한 뒤 다시 실행할 수 있다.

이 순간 AI는 코드 설명 도구가 아니라 개발 Workflow에 참여하는 작업 주체가 된다.

---

## 2. Cloud Agent에는 컴퓨터가 필요하다

Coding Agent가 Repository를 실제로 수정하고 검증하려면 코드를 실행할 장소가 필요하다.

Java 프로젝트라면 적어도 다음이 필요할 수 있다.

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

Frontend까지 포함되면 Node와 Browser가 필요할 수 있다.

```text
Node
npm/pnpm
Chrome
Playwright
```

Docker Build나 Testcontainers를 실행하려면 Container Runtime도 필요하다.

따라서 Cloud Agent를 모델 하나로만 보면 실제 비용과 동작을 설명하기 어렵다.

다음 두 구조를 비교해 보자.

### 구조 A: LLM만 생각하는 경우

```text
Developer
   ↓
LLM
   ↓
Code Suggestion
```

### 구조 B: Cloud Agent를 Worker로 보는 경우

```text
Developer
   ↓
Task
   ↓
Cloud Worker
├─ LLM
├─ Repository
├─ CPU
├─ RAM
├─ Disk
├─ JDK / Node / Docker
└─ Build / Test Tools
   ↓
Commit / Test Result / Artifact / PR
```

실제 Cloud coding agent 제품들도 이러한 방향의 실행 모델을 사용한다.

GitHub의 Cloud coding agent는 별도의 ephemeral development environment에서 Repository를 탐색하고 코드를 변경하며 테스트와 lint를 실행할 수 있도록 설명되어 있다. OpenAI Codex 역시 Cloud 환경에서 Repository와 개발환경을 사용하고 파일을 수정하며 테스트, lint, type checking 같은 명령을 실행하는 형태로 설명되어 왔다.

제품마다 구현 방식과 제약은 다르다. 중요한 것은 특정 제품 기능이 아니라 공통 구조다.

```text
LLM
+
Code
+
Execution Environment
+
Tools
```

이 네 요소가 합쳐져야 `실제로 작업하는 Agent`가 된다.

---

## 3. Cloud Agent의 핵심 가치는 Token이 아니다

Cloud Agent를 처음 접하면 모델 성능이나 Token 양에 관심이 집중되기 쉽다.

하지만 Cloud 환경에서 얻는 실질적인 이점은 다른 곳에도 있다.

예를 들어 개발자의 Mac에서 다음 프로그램을 동시에 실행하고 있다고 가정하자.

```text
IDE
Local Agent
Docker
Database
Browser
Android Emulator
```

여기에 전체 Gradle Test와 Testcontainers, Docker Build까지 실행하면 개발환경 자체가 무거워질 수 있다.

이 검증 작업을 Cloud Worker로 보내면 구조가 달라진다.

```text
Developer Mac
├─ IDE
├─ Local Agent
└─ 현재 기능 개발

Cloud Worker
├─ Repository
├─ Gradle
├─ Docker
├─ Testcontainers
└─ 전체 Test 실행
```

Cloud Worker가 테스트를 수행하는 동안 개발자는 다른 작업을 계속할 수 있다.

또 서로 독립적인 작업이라면 여러 Worker를 동시에 실행할 수 있다.

```text
                  Git SHA
                     |
        +------------+------------+
        |            |            |
   Cloud #1      Cloud #2      Cloud #3
   Unit Test     E2E Test      Docker Build
        |            |            |
        +------------+------------+
                     |
                  Evidence
```

따라서 이 책에서 반복해서 사용할 문장은 다음과 같다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

물론 LLM 성능은 중요하다. 그러나 모델만 좋아져도 개발자 PC와 분리된 CPU, RAM, Browser, Docker 환경이 자동으로 생기는 것은 아니다.

Cloud Agent의 가치를 판단할 때는 `얼마나 잘 생각하는가`와 `어디에서 무엇을 실행할 수 있는가`를 함께 봐야 한다.

---

## 4. Remote Developer보다 Remote Worker에 가깝다

Cloud Agent를 `인터넷에 있는 AI 개발자`라고 표현하면 이해하기는 쉽다.

하지만 이 표현은 개발 Workflow를 설계할 때 몇 가지 오해를 만든다.

개발자 한 명을 추가한다고 생각하면 다음과 같은 지시를 만들기 쉽다.

```text
Repository 전체를 살펴보고 문제가 있으면 알아서 고쳐줘.
```

이런 Task는 범위가 넓다.

Agent가 Repository 전체를 탐색하고, 어떤 파일을 수정해야 하는지 추론하고, 어떤 테스트를 실행해야 하는지 다시 찾아야 한다.

반대로 Remote Worker라고 생각하면 입력과 출력이 달라진다.

```text
Task
AuthService expired token 처리 수정

Base Commit
abc123

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected
HTTP 401
```

Worker는 이 명확한 Task를 받아 독립환경에서 처리한다.

결과도 대화가 아니라 작업 결과로 받는다.

```text
Commit
abc789

Test
AuthServiceTest.expiredToken PASS

Changed Files
3
```

이 관점은 Cloud Agent를 덜 인간적으로 보는 것이 아니다.

개발 프로세스에서 **작업 단위와 검증 단위를 명확히 만든다**는 의미다.

Cloud Agent가 사람처럼 자유롭게 탐색할 수 있어도, 매 작업마다 전체 프로젝트를 다시 이해하게 만드는 방식은 비용과 재현성 측면에서 불리할 수 있다.

---

## 5. Cloud Agent의 입력은 Prompt만이 아니다

Cloud Task를 시작할 때 필요한 것은 자연어 Prompt 하나만이 아니다.

실제 작업에는 다음 정보가 함께 존재한다.

```text
Task
Repository
Base Commit
Branch
Environment
Context
Validation
Expected Result
```

예를 들어 다음 두 요청을 비교해 보자.

### 요청 A

```text
로그인 오류 고쳐줘.
```

### 요청 B

```text
Task:
Expired JWT 요청이 200이 아니라 401을 반환하도록 수정

Base SHA:
abc123

Relevant Files:
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Failure:
expected 401
actual 200

Validation:
./gradlew test --tests AuthServiceTest.expiredToken
```

두 요청 모두 자연어를 사용하지만 실제 작업 비용은 다르다.

A에서는 Agent가 문제 범위부터 찾아야 한다.

B에서는 이미 작업의 시작점과 검증 방법이 제공된다.

이 책의 후반에서는 이를 `Cloud Agent Task Contract`로 구체화한다.

1장에서는 한 가지 원칙만 기억하면 된다.

> Cloud Agent에게 넘기는 것은 Prompt가 아니라 Task다.

---

## 6. Cloud Agent의 출력도 답변만이 아니다

일반 Chat에서는 다음 응답이 충분할 수 있다.

```text
수정했습니다. 문제가 해결되었습니다.
```

하지만 실제 개발에서는 이 문장만으로 완료 여부를 판단할 수 없다.

Cloud Agent가 작업했다면 확인할 수 있는 결과가 필요하다.

예:

```text
Implementation
DONE

Commit
abc789

Build
PASS

Unit Test
314 / 314 PASS

Changed Files
- AuthService.java
- AuthServiceTest.java

Artifacts
- junit.xml
```

UI 변경이라면 Screenshot이나 E2E Video가 포함될 수도 있다.

```text
Build: PASS
E2E: PASS
Before Screenshot: before.png
After Screenshot: after.png
Video: e2e.webm
```

따라서 Cloud Agent의 결과는 다음과 같이 본다.

```text
Natural Language Summary
+
Commit / Diff
+
Test Result
+
Artifact
+
PR
```

모든 Task에 모든 Artifact가 필요한 것은 아니다.

중요한 것은 `Agent가 완료했다고 말했는가`와 `완료를 검증할 Evidence가 있는가`를 구분하는 것이다.

이 원칙은 8장에서 Result Gateway와 Evidence-based Result로 확장한다.

---

## 7. Git은 Remote Worker와 연결되는 경계다

Local Agent는 개발자의 현재 Working Directory를 직접 볼 수 있다.

Cloud Agent는 보통 별도의 작업환경에서 시작한다.

따라서 Local과 Cloud 사이에는 작업 상태를 넘기는 경계가 필요하다.

이 책에서는 Git을 그 핵심 경계로 사용한다.

```text
Local Developer
→ Code
→ Test
→ Commit
→ Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Branch
→ Work
→ Test
→ Commit
→ Push
→ PR
```

Git은 원래부터 코드 변경 이력을 관리하는 도구다.

Cloud Agent 환경에서는 여기에 역할이 하나 더 추가된다.

> 개발자와 Remote Worker 사이의 작업 상태 전달 프로토콜

예를 들어 Cloud Task는 다음 상태와 연결할 수 있다.

```text
Task ID: task-142
Base SHA: abc123
Branch: agent/task-142
Status: testing
Result: PASS
PR: #142
```

이 구조가 있으면 Worker가 여러 개여도 어느 Task가 어떤 Git 상태에서 작업 중인지 추적할 수 있다.

Git과 Branch, Worktree, Container 격리는 11장에서 상세히 다룬다.

---

## 8. Cloud Agent가 모든 개발을 맡는 것은 아니다

Cloud Agent의 실행환경이 독립적이라는 사실이 모든 Task를 Cloud로 보내야 한다는 뜻은 아니다.

다음 작업을 생각해 보자.

```text
새 인증 Architecture를 어떻게 설계할 것인가?
```

요구사항이 아직 바뀌고 있고 여러 모듈의 영향을 동시에 검토해야 한다면 개발자와 Local Agent가 짧게 질문과 수정을 반복하는 것이 더 나을 수 있다.

반면 다음 작업은 다르다.

```text
AuthServiceTest.expiredToken 실패 수정
```

실패가 재현되고 관련 파일과 검증 명령이 명확하다면 Cloud에 위임하기 쉽다.

또 다음 작업은 LLM 자체가 필요 없을 수도 있다.

```text
./gradlew test
```

명령과 판정 기준이 결정적이라면 일반 Cloud Runner가 처리하면 된다.

따라서 이 책의 기본 구조는 다음과 같다.

```text
Local
→ 설계 / 탐색 / 통합

Cloud Agent
→ 독립적인 판단 + 수정 Task

Cloud Runner
→ Build / Test / Validation
```

2장에서는 Local과 Cloud의 차이를, 5장에서는 실제 Routing 기준을, 10장에서는 Runner와 Agent의 분리를 다룬다.

---

## 9. campus-platform을 Remote Worker 관점으로 보면

이 책에서는 Java/Spring Boot 기반 `campus-platform`을 반복 예제로 사용한다.

초기 구조를 다음처럼 단순화해 보자.

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

개발자가 출결 인증 문제를 수정했다고 하자.

Local에서 계속 모든 검증을 실행할 수도 있다.

```text
Developer Mac
→ Unit Test
→ Integration Test
→ Docker Build
→ Web E2E
```

하지만 작업을 분리하면 다음처럼 볼 수 있다.

```text
Local Developer / Agent
→ 요구사항 확인
→ 코드 수정
→ Commit / Push
        |
        +-- Cloud Worker #1
        |    Unit Test
        |
        +-- Cloud Worker #2
        |    Integration Test
        |
        +-- Cloud Worker #3
        |    Docker Build
        |
        +-- Cloud Worker #4
             Web E2E
```

각 Worker는 동일한 Git 기준점에서 독립적으로 실행되고 결과를 Evidence로 반환한다.

개발자는 모든 검증이 끝날 때까지 터미널 앞에서 기다릴 필요가 없다.

이 구조가 바로 이 책에서 앞으로 다룰 Cloud Agent 활용의 출발점이다.

---

## 10. 책 전체에서 사용할 Cloud Agent 모델

지금까지의 내용을 하나의 그림으로 정리하면 다음과 같다.

```text
                 Developer / Local Agent
                           |
                        Task Split
                           |
                          Git
                           |
          +----------------+----------------+
          |                                 |
     Cloud Runner                      Cloud Agent
          |                                 |
 Build / Test / Validation        Analyze / Modify / Fix
          |                                 |
          +----------------+----------------+
                           |
                        Evidence
                           |
                           PR
                           |
                 Developer / Local Agent
                           |
                    Review / Integration
```

이 그림에는 책 전체의 핵심 주제가 대부분 들어 있다.

- Local과 Cloud의 역할 분리
- Cloud의 독립 실행환경
- Compute와 Token의 분리
- Runner와 Agent의 분리
- Git Handoff
- 작은 Task
- 자동 검증
- Evidence 기반 결과
- 병렬 Worker

이후 장에서는 이 요소를 하나씩 분해해 실제 개발 Workflow로 만든다.

---

## 11. 이 장에서 기억할 기준

Cloud Agent를 판단할 때 모델 이름보다 먼저 다음 질문을 한다.

```text
어떤 Repository에서 작업하는가?
어떤 실행환경을 사용하는가?
어떤 CPU/RAM/Disk 작업이 필요한가?
어떤 Tool을 실행할 수 있는가?
작업 상태를 어떻게 전달하는가?
어떻게 검증하는가?
어떤 Evidence를 반환하는가?
```

Cloud Agent를 이 기준으로 보면 `원격 AI 채팅`과 `Remote Development Worker`의 차이가 분명해진다.

이 책은 후자의 관점에서 출발한다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> Cloud Agent에게 넘기는 것은 Prompt가 아니라 Task다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

다음 장에서는 이 Remote Worker를 Local Agent와 비교하고, 어떤 작업을 어느 실행 위치에 두어야 하는지 기준을 만든다.

---

## 참고자료

제품별 기능은 변경될 수 있으므로 본문에서는 공통 실행 모델만 사용했다. 확인 기준일은 2026-09-16이다.

- GitHub Docs, `About GitHub Copilot cloud agent`
- GitHub Docs, `Configure the development environment`
- OpenAI, `Addendum to OpenAI o3 and o4-mini system card: Codex`, 2025-05-16
- OpenAI, `Codex is now generally available`, 2025-10-06

세부 조사 메모는 `research/chapter-01-cloud-worker-official-sources.md`에서 관리한다.
