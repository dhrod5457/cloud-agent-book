# 3장. Cloud Session, Container, Compute와 Token

Cloud Agent를 실제 개발에 사용하기 시작하면 비용과 성능을 설명할 때 자주 섞이는 개념이 있다.

바로 **Cloud에서 코드를 실행하는 자원**과 **LLM이 읽고 판단하는 자원**이다.

예를 들어 Cloud Worker가 30분 동안 전체 테스트를 실행했다고 하자.

이때 CPU와 RAM은 계속 사용될 수 있다. Testcontainers가 여러 Container를 띄우고, Gradle이 수천 개의 테스트를 실행하며, Disk에는 로그와 테스트 결과가 쌓일 수 있다.

그렇다고 이 30분이 그대로 30분치 LLM 추론이나 일정한 비율의 Token 사용을 의미하는 것은 아니다.

이 책에서는 두 자원을 먼저 분리해서 본다.

```text
Reasoning Resource
- LLM input
- LLM output
- reasoning
- source/context 읽기
- tool result 분석

Execution Resource
- CPU
- RAM
- Disk
- Process
- Container / VM
- Browser
- Build / Test Tools
```

두 자원은 서로 연결되어 있지만 같은 것은 아니다.

이 차이를 이해해야 Cloud Agent를 `비싼 LLM을 오래 실행하는 도구`가 아니라 `LLM 판단과 원격 컴퓨팅 자원을 결합한 Remote Worker`로 볼 수 있다.

이 장에서 반복해서 사용할 원칙은 다음과 같다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

---

## 1. Cloud Session은 하나의 작업 공간이다

Cloud Agent 제품마다 Session, Task, Workspace, Environment 같은 이름을 다르게 사용할 수 있다.

이 책에서는 제품의 이름과 상관없이 다음 구조를 하나의 `Cloud Session`으로 생각한다.

```text
Cloud Session
├─ Repository / Branch
├─ Workspace
├─ CPU
├─ RAM
├─ Disk
├─ Development Tools
└─ Agent / LLM interaction
```

핵심은 `계정에 원격 컴퓨터 한 대가 붙어 있다`는 식으로 이해하지 않는 것이다.

실제 Cloud coding agent에서는 Task나 Session 단위로 별도 Workspace나 Sandbox가 만들어질 수 있고, 여러 작업이 서로 다른 실행환경에서 동시에 진행될 수도 있다.

따라서 개발자가 봐야 할 것은 다음 질문이다.

```text
이 Task의 Source는 어느 Git 상태인가?
어느 Workspace에서 실행되는가?
어떤 CPU/RAM/Disk를 사용하는가?
어떤 Tool이 설치되어 있는가?
이 실행환경은 다른 Task와 격리되어 있는가?
결과는 어떻게 반환되는가?
```

제품별 vCPU 수나 RAM 크기는 바뀔 수 있다.

이 책에서는 그 숫자를 Cloud Agent의 본질로 보지 않는다.

중요한 것은 **Task마다 독립적으로 실행 가능한 Compute와 Workspace가 존재한다는 구조**다.

---

## 2. 모델이 코드를 실행하는 것은 아니다

개발자가 Agent에게 다음과 같이 요청했다고 하자.

```text
전체 테스트를 실행하고 실패한 테스트의 원인을 찾아 수정해줘.
```

겉으로 보면 하나의 요청이지만 내부적으로는 성격이 다른 작업이 섞여 있다.

먼저 LLM이 무엇을 해야 하는지 판단한다.

```text
./gradlew test
```

그다음 실제 테스트는 Shell과 JVM, 운영체제, CPU, RAM에서 실행된다.

```text
Gradle
→ JVM
→ JUnit
→ Application Context
→ Testcontainers
→ DB Container
```

실패가 발생하면 다시 LLM이 결과를 읽고 다음 행동을 판단할 수 있다.

전체 흐름을 단순화하면 다음과 같다.

```text
LLM
명령 결정
   ↓
Cloud Execution Environment
명령 실행
   ↓
CPU / RAM / Disk
Build / Test / Browser
   ↓
Result
   ↓
LLM
결과 분석 / 다음 행동 판단
```

즉 Agent가 `테스트를 실행했다`고 표현하더라도 모든 계산을 LLM이 수행한 것은 아니다.

LLM은 무엇을 실행할지 결정하고, Tool을 호출하고, 결과를 해석한다.

실제 컴파일, 테스트, Browser 렌더링, Docker Build는 일반 프로그램이 수행한다.

이 구분은 이후 Runner-first 구조의 출발점이 된다.

---

## 3. Brain과 Hands는 설명을 위한 간단한 모델이다

이 차이를 쉽게 설명하기 위해 이 책에서는 제한적으로 `Brain / Hands`라는 표현을 사용한다.

```text
Brain
= LLM
= 계획 / 판단 / 분석 / 수정 방향 결정

Hands
= Execution Environment
= Shell / CPU / RAM / Disk / Build / Test / Browser
```

예를 들어 다음 Task를 생각해 보자.

```text
AuthServiceTest.expiredToken 실패 수정
```

Brain이 하는 일:

```text
실패 의미 확인
→ 관련 코드 판단
→ 수정 방향 결정
→ 어떤 테스트를 다시 실행할지 결정
```

Hands가 하는 일:

```text
파일 읽기/쓰기
→ javac / Gradle 실행
→ JUnit 실행
→ Testcontainers 실행
→ 결과 파일 생성
```

이 모델을 Agent Platform 전체 아키텍처로 확장하지 않는다.

이 책에서 Brain/Hands를 사용하는 이유는 하나다.

> LLM이 판단하는 비용과 Cloud 실행환경이 계산하는 비용을 분리해서 보기 위해서다.

---

## 4. Build가 오래 걸린다고 Token이 같은 비율로 늘어나는 것은 아니다

`campus-platform` 전체 Build가 20분 걸린다고 가정하자.

```bash
./gradlew clean build
```

이 시간 동안 다음이 실행될 수 있다.

```text
Java Compile
Test Compile
Unit Test
Integration Test
Code Generation
Static Analysis
Package
```

주로 사용하는 자원은 다음과 같다.

```text
CPU
RAM
Disk I/O
Network I/O
Container Runtime
```

LLM 사용은 다른 지점에서 발생한다.

예를 들면:

```text
Task 읽기
Repository 구조 확인
실행 명령 선택
Build 결과 읽기
오류 분석
수정 코드 생성
```

따라서 다음 두 상황은 구분해야 한다.

### 상황 A

```text
Agent
→ build 실행

Container
→ 20분 동안 build

Agent
→ 최종 결과 20줄 확인
```

### 상황 B

```text
Agent
→ build 실행
→ 10초마다 상태 확인
→ 로그 전체 반복 조회
→ 중간 출력 계속 분석
→ 다시 조회
```

두 경우 모두 Build는 20분이지만 LLM이 읽는 Context와 Tool Output의 양은 다르다.

즉 `Wall-clock Time`과 `LLM Usage`를 같은 지표로 보면 Cloud Agent 비용을 잘못 해석할 수 있다.

---

## 5. Token은 언제 많이 사용되는가

Cloud Agent에서 Token 사용을 줄이려면 먼저 어디에서 Context가 생기는지 봐야 한다.

대표적인 지점은 다음과 같다.

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

예를 들어 Agent가 처음 Repository를 탐색하면서 다음 파일을 읽을 수 있다.

```text
README.md
AGENTS.md
build.gradle
settings.gradle
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

이 정보는 모두 Agent가 판단하기 위한 Context가 된다.

테스트가 실패한 뒤 100MB의 로그를 그대로 읽는다면 Tool Output 역시 Context가 된다.

반대로 Container가 10,000개의 테스트를 실행했더라도 Agent가 다음 요약만 읽는다면 상황은 달라진다.

```text
Tests
total: 10,000
passed: 9,997
failed: 3

Failures
- AuthServiceTest.expiredToken
- UserServiceTest.deleteUser
- UserMapperTest.insert
```

따라서 Token을 줄이는 방법은 단순히 `Agent에게 짧게 답하라고 한다`가 아니다.

다음 두 방향이 더 중요하다.

```text
Agent가 처음 읽어야 하는 Context를 줄인다.

Agent가 Tool에서 돌려받는 Output을 줄인다.
```

첫 번째는 7장의 Task Contract에서, 두 번째는 8장의 Result Gateway에서 구체화한다.

---

## 6. 테스트 10,000개를 실행하는 것과 로그 10,000개를 읽는 것은 다르다

Cloud Agent의 컴퓨팅 자원을 이해하는 데 가장 중요한 예가 테스트다.

다음 상황을 가정하자.

```text
JUnit Tests: 10,000
Execution Time: 25분
Log: 100MB
Failures: 3
```

좋지 않은 구조는 다음과 같다.

```text
Cloud Container
→ 테스트 10,000개 실행
→ 100MB 로그 생성
      ↓
LLM
→ 100MB 전체 분석
```

Container가 이미 실패 위치를 구조화된 테스트 리포트로 제공할 수 있는데도 LLM에게 전체 로그를 읽혀 다시 실패를 찾게 하는 것이다.

더 나은 구조는 다음과 같다.

```text
Cloud Container
→ 테스트 10,000개 실행
→ CPU/RAM 사용
      ↓
JUnit Report / Result Filter
→ 실패 3건 추출
      ↓
LLM
→ 실패 3건 우선 분석
```

필요한 경우에만 특정 실패의 상세 Stack Trace를 추가로 읽는다.

```text
Summary
  ↓
Failure Detail
  ↓
Specific Log
  ↓
Full Raw Log
```

핵심은 실행량 자체를 줄이는 것이 아니다.

Cloud CPU에는 많은 일을 시킬 수 있다.

대신 LLM에게는 판단에 필요한 결과부터 보여준다.

---

## 7. Compute-heavy / Context-light 작업을 찾는다

Cloud에 특히 보내기 좋은 작업 중 하나는 다음 특성을 가진다.

```text
Compute는 큼
Context는 작음
완료 조건은 명확함
```

대표적으로 다음 작업이 있다.

```text
전체 Unit Test
Integration Test
Docker Build
Web E2E
Static Analysis
Migration Validation
```

전체 Unit Test를 예로 들어보자.

Agent가 알아야 하는 것은 반드시 Repository 전체 구조가 아니다.

이미 명령이 정해져 있다면 다음이면 충분할 수 있다.

```text
Base SHA
Validation Command
Environment
Expected Exit Code
Artifact Location
```

실제 계산은 Cloud Runner가 수행한다.

```text
./gradlew test
```

이런 작업에서 LLM을 계속 참여시키는 것은 오히려 비용을 늘릴 수 있다.

따라서 이후 장에서는 다음 질문을 반복한다.

> 이 Task는 LLM이 계속 판단해야 하는가, 아니면 Compute만 필요한가?

---

## 8. Cloud Runner와 Cloud Agent는 같은 것이 아니다

Cloud 환경에서 실행된다고 모든 작업이 Agent 작업은 아니다.

예를 들어 다음 명령을 보자.

```bash
./gradlew test
```

명령이 고정되어 있고 성공 여부를 exit code로 판정할 수 있다면 일반 Runner로 충분하다.

```text
Cloud Runner
→ ./gradlew test
→ exit 0
→ PASS
```

LLM이 필요한 것은 다음과 같은 상황이다.

```text
FAIL
→ 왜 실패했는가?
→ 어떤 코드를 바꿔야 하는가?
→ 변경 범위를 어디까지 잡아야 하는가?
```

즉 기본 구조는 다음으로 발전할 수 있다.

```text
Task
  ↓
Runner
  ↓
PASS → Done
FAIL
  ↓
Cloud Agent
  ↓
Analyze / Fix
  ↓
Runner
  ↓
Verification
```

이 구조는 10장에서 상세히 다룬다.

이 장에서는 `Cloud Compute`와 `LLM Reasoning`을 분리할 수 있다는 점만 기억하면 된다.

---

## 9. Agent가 기다리는 시간과 Token 사용을 구분한다

Cloud Agent가 외부 프로세스를 실행하고 기다리는 상황을 보자.

```text
Agent
→ docker build 실행

Docker
→ 12분 동안 Image Build

Agent
→ 최종 exit code와 오류 요약 확인
```

이때 12분이라는 시간 자체가 LLM에게 12분 동안 계속 Context가 공급되었다는 뜻은 아니다.

그러나 구현 방식에 따라 Agent가 다음 행동을 할 수도 있다.

```text
30초마다 로그 확인
→ 현재 상태 분석
→ 다시 확인
→ 로그 누적 읽기
```

이 경우에는 Tool 호출과 결과 읽기가 반복된다.

따라서 장시간 Task에서는 다음을 구분한다.

```text
Process Running Time
Agent Active Reasoning Time
Tool Result Volume
Polling Frequency
```

좋은 Cloud Workflow는 장시간 프로세스가 실행되는 동안 Agent가 불필요하게 같은 상태를 반복 읽지 않게 한다.

결과가 나왔을 때 필요한 Summary를 읽는 방식이 더 단순하다.

---

## 10. 여러 Session을 띄우는 목적도 구분해야 한다

Cloud Agent의 장점 중 하나는 여러 독립 작업을 동시에 실행할 수 있다는 점이다.

하지만 `Session을 여러 개 띄운다`는 표현 안에는 서로 다른 두 가지가 섞일 수 있다.

### Parallel Compute

```text
Worker A
→ Backend Unit Test

Worker B
→ Integration Test

Worker C
→ Web E2E

Worker D
→ Docker Build
```

각 Worker는 정해진 작업을 실행한다.

이 경우 늘어난 것은 주로 병렬 Compute다.

### Parallel Reasoning

```text
Agent A
→ Repository 전체 분석

Agent B
→ Repository 전체 분석

Agent C
→ Repository 전체 분석

Agent D
→ Repository 전체 분석
```

이 경우 각 Agent가 같은 Source와 문서를 다시 읽고 비슷한 판단을 반복할 수 있다.

즉 Agent 네 개를 띄웠다고 네 배의 유효 작업이 생기는 것은 아니다.

같은 Context를 네 번 읽는 비용만 늘 수도 있다.

따라서 병렬화의 기본 순서는 다음과 같다.

```text
먼저 Task를 독립적으로 나눈다.
      ↓
각 Task의 Context를 줄인다.
      ↓
필요한 Compute 또는 Agent를 붙인다.
```

여러 Worker의 Context 중복 비용은 12장에서 상세히 다룬다.

---

## 11. 비용을 하나의 숫자로 보면 안 된다

Cloud Agent의 비용을 Token 하나로만 표현하면 실제 개발 Workflow의 장단점을 놓치기 쉽다.

최소한 세 종류를 나눠서 보는 것이 좋다.

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

예를 들어 Cloud에서 CPU를 더 사용해 30분짜리 테스트를 별도 Worker로 보냈다고 하자.

Compute 사용은 늘 수 있다.

하지만 개발자가 자신의 Mac에서 테스트가 끝나기를 기다리지 않고 다른 Feature를 개발할 수 있다면 Human Cost는 줄 수 있다.

반대로 작은 문구 수정 하나를 위해 Cloud Worker를 띄우고 Agent가 Repository를 탐색하고 PR까지 만든다면 Compute와 LLM, Review 비용이 모두 더 커질 수 있다.

따라서 Cloud Agent 최적화의 목표는 한 종류의 비용만 최소화하는 것이 아니다.

> 전체 개발 Workflow에서 어떤 자원을 어디에 쓰는 것이 유리한지 판단하는 것이다.

---

## 12. 개발자 대기시간도 비용이다

2장에서 `Agent Execution Time`과 `Developer Blocking Time`을 구분했다.

이 개념을 Compute/Token 관점에 연결해 보자.

Local에서 전체 테스트를 실행하면 다음 구조가 될 수 있다.

```text
Developer Mac
→ Gradle Test
→ CPU/RAM 점유
→ Docker/Testcontainers 점유
→ 개발 작업 영향
```

Cloud로 보내면 다음처럼 분리할 수 있다.

```text
Developer Mac
→ 현재 Feature 개발

Cloud Runner
→ 전체 Test
```

이때 Cloud Compute를 더 사용하더라도 개발자의 Blocking Time과 Local Resource Occupancy가 줄 수 있다.

그래서 책에서는 다음 값들을 별도로 본다.

```text
Compute Time
LLM Usage
Developer Blocking Time
Review Time
Retry Time
```

4장에서는 이 시간 분리를 Cloud Agent의 주요 가치 중 하나로 확장한다.

---

## 13. campus-platform에서 자원을 나눠 보면

`campus-platform`에서 출결 기능을 변경했다고 가정하자.

검증해야 할 작업은 다음과 같다.

```text
Unit Test
Integration Test
Docker Build
Web E2E
```

이 작업을 하나의 Agent가 순서대로 계속 지켜보게 할 필요는 없다.

다음과 같이 나눌 수 있다.

```text
Git SHA: abc123
        |
        +-- Runner #1
        |    ./gradlew :attendance:test
        |
        +-- Runner #2
        |    integrationTest
        |
        +-- Runner #3
        |    docker build
        |
        +-- Runner #4
             Playwright E2E
```

각 Runner는 CPU/RAM을 사용해 작업을 실행한다.

최종 결과는 다음 정도로 모을 수 있다.

```text
Unit: PASS
Integration: PASS
Docker: PASS
E2E: FAIL 2
```

그때 Agent가 E2E 실패 두 건만 분석한다.

```text
E2E Failure
- expired-token-admin-view
- attendance-error-message
```

이 구조에서 LLM은 네 개의 작업 전체를 계속 읽고 있지 않는다.

Compute가 먼저 일을 하고 판단이 필요한 지점에서 LLM이 들어온다.

이것이 `CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다`는 문장의 실제 의미다.

---

## 14. 제품별 CPU와 RAM 숫자를 본문 원칙으로 만들지 않는다

실제 Cloud coding agent를 사용하다 보면 다음 정보를 알고 싶어진다.

```text
몇 vCPU인가?
RAM은 몇 GB인가?
Disk는 얼마인가?
동시에 몇 Session까지 가능한가?
Session은 얼마나 오래 유지되는가?
```

이 정보는 실제 운영과 비용 계산에 중요하다.

하지만 책의 핵심 원칙으로 고정하기에는 변동성이 크다.

제품 정책과 요금제에 따라 변경될 수 있기 때문이다.

따라서 이 책에서는 다음처럼 분리한다.

```text
본문
→ Compute와 Token을 분리하는 원칙
→ Task별 자원 활용 방법
→ 병렬화 판단 방법

Research / 참고자료
→ 기준일별 vCPU/RAM/Disk
→ 동시 Session 제한
→ 가격 / Rate Limit
```

독자가 제품을 바꾸더라도 본문의 판단 기준은 그대로 사용할 수 있어야 한다.

---

## 15. 이 장에서 기억할 구조

Cloud Agent를 볼 때 다음 한 덩어리로 생각하지 않는다.

```text
Cloud Agent Cost
```

대신 다음처럼 분리한다.

```text
Task
  ↓
LLM
계획 / 판단
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

그리고 비용도 다음처럼 본다.

```text
Compute
+
LLM
+
Human
```

이 구조를 이해하면 다음 질문에 답하기 쉬워진다.

```text
이 작업은 LLM이 해야 하는가?
CPU/RAM만 있으면 되는가?
Agent가 전체 로그를 읽어야 하는가?
여러 Session에서 병렬 Compute가 가능한가?
같은 Context를 여러 Agent가 반복해서 읽고 있지는 않은가?
```

다음 장에서는 이 독립 실행환경을 왜 사용하는지, 그리고 장시간 작업과 병렬성을 개발자의 실제 시간과 어떻게 연결할지 살펴본다.

---

## 참고자료

제품별 자원 사양과 제한은 본문 원칙과 분리해 조사 문서에서 관리한다.

- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/infrastructure-noise.md`

공식 제품 수치는 본문에 고정하지 않고 사용할 경우 기준일과 출처를 함께 기록한다.
