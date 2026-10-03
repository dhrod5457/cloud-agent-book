# 3장. Cloud Session, Container, Compute와 Token

클라우드 에이전트를 실제 개발에 사용하면 비용과 성능을 설명할 때 자주 섞이는 개념이 있다.

바로 **클라우드에서 코드를 실행하는 자원**과 **LLM이 읽고 판단하는 자원**이다.

예를 들어 클라우드 환경에서 전체 테스트를 30분 동안 실행했다고 하자. 그동안 CPU와 RAM이 사용되고, Testcontainers가 컨테이너를 띄우며, 디스크에는 로그와 테스트 결과가 쌓일 수 있다. 하지만 실행에 걸린 30분을 그대로 LLM이 30분 동안 추론했다거나 일정한 비율의 토큰을 사용했다는 뜻으로 해석할 수는 없다.

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

> CPU와 RAM 사용량은 LLM 토큰 사용량과 직접적으로 같은 개념이 아니다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

---

## 1. Cloud Session은 하나의 작업 실행 단위다

제품마다 세션(Session), 작업(Task), 작업공간(Workspace), 실행환경(Environment) 같은 이름을 다르게 사용할 수 있다.

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

제품별 vCPU, RAM, 디스크 수치는 바뀔 수 있다. 이 장에서는 특정 수치보다 **작업이 독립 실행환경을 사용한다는 구조**에 집중한다.

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

실제 테스트는 셸(Shell), JVM, 운영체제, CPU, RAM에서 실행된다.

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

에이전트가 `Test를 실행했다`고 표현하더라도 컴파일, 테스트, 브라우저 렌더링, Docker 빌드 자체는 일반 프로그램이 수행한다.

이 구분이 후반부의 클라우드 실행기 설계로 이어진다.

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

판단 역할인 Brain이 하는 일:

```text
실패 의미 확인
→ 관련 코드 판단
→ 수정 방향 결정
→ 어떤 검증을 다시 실행할지 결정
```

실행 역할인 Hands가 하는 일:

```text
파일 읽기/쓰기
→ Gradle 실행
→ JUnit 실행
→ Testcontainers 실행
→ 결과 파일 생성
```

이 모델을 별도의 에이전트 플랫폼 아키텍처로 확장하지 않는다.

목적은 하나다.

> LLM이 판단하는 비용과 실행환경이 계산하는 비용을 분리해서 보기 위해서다.

---

## 4. Build 시간과 LLM 사용량은 다른 지표다

다음 수치는 설명용 예다. 전체 빌드가 20분 걸린다고 하자.

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

주로 사용하는 자원은 CPU, RAM, 디스크 I/O, 네트워크 입출력, 컨테이너 실행환경이다.

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

두 경우 빌드 시간은 같아도 LLM이 읽는 도구 출력과 맥락 정보의 양은 다르다. 따라서 실제로 흐른 시간인 `Wall-clock Time`과 LLM 사용량인 `LLM Usage`를 같은 지표로 보면 비용을 잘못 해석할 수 있다.

---

## 5. Token은 Source뿐 아니라 Tool Output에서도 사용된다

에이전트가 판단하기 위해 읽는 것은 소스 코드만이 아니다.

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

이 정보가 에이전트의 판단에 쓰이는 맥락 정보가 된다.

예를 들어 컨테이너가 테스트 10,000개를 실행했다고 하자. 이 숫자는 설명용 예다.

결과가 다음과 같을 수 있다.

```text
Tests
total: 10,000
passed: 9,997
failed: 3
```

실행된 테스트 수가 많다고 해서 에이전트가 10,000개 테스트의 모든 출력을 읽어야 하는 것은 아니다.

중요한 것은 **실행량과 에이전트가 읽는 정보량을 분리하는 것**이다.

7장에서는 에이전트가 처음 읽는 소스 코드 / 맥락 정보를 줄이고, 8장에서는 도구 출력과 검증 근거를 다룬다.

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

테스트 보고서가 실패 3건을 구조화해서 제공할 수 있다면 LLM에게 전체 정상 로그를 다시 읽힐 필요는 없다.

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

원본 로그는 보존할 수 있다. 다만 에이전트에게 처음 보여주는 결과를 작게 만든다.

이 원칙은 8장의 Result Gateway에서 구체화한다.

---

## 7. Compute-heavy / Context-light 작업을 찾는다

클라우드에 보내기 쉬운 작업 중에는 다음 특성을 가진 것이 있다.

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

이미 실행 명령이 정해진 검증이라면 에이전트가 저장소 전체를 이해할 필요가 없을 수 있다.

```text
Base SHA
Command
Environment
Expected Exit Code
Artifact Location
```

실제 계산은 실행환경이 담당한다.

이때 중요한 질문은 다음이다.

> 이 작업은 계속 LLM의 판단이 필요한가, 아니면 대부분 실행 자원만 필요한가?

구체적인 Runner-first 구조는 10장에서 다룬다.

---

## 8. Parallel Compute와 Parallel Reasoning은 다르다

여러 클라우드 세션을 동시에 사용한다고 하자.

다음은 계산 작업의 병렬 실행이다.

```text
Worker A → Unit Test
Worker B → Integration Test
Worker C → Web E2E
Worker D → Docker Build
```

서로 다른 실행환경이 정해진 작업을 동시에 수행한다.

반면 다음은 여러 모델의 병렬 추론이다.

```text
Agent A → Repository 전체 분석
Agent B → Repository 전체 분석
Agent C → Repository 전체 분석
Agent D → Repository 전체 분석
```

후자는 각 에이전트가 같은 소스 코드와 문서를 반복해서 읽을 수 있다.

따라서 세션 수가 늘었다고 유효 작업량이 같은 비율로 증가하는 것은 아니다.

기본 순서는 다음과 같다.

```text
Task를 독립적으로 나눈다
      ↓
각 Task의 Context를 줄인다
      ↓
필요한 Compute 또는 Agent를 배치한다
```

병렬화의 이득과 맥락 정보를 중복해서 읽는 비용은 12장에서 자세히 다룬다.

---

## 9. 비용은 Compute, LLM, Human으로 나눠 본다

클라우드 에이전트 비용을 토큰 하나로만 표현하면 실제 작업 흐름의 병목을 놓치기 쉽다.

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

예를 들어 클라우드에서 장시간 테스트를 별도 실행하면 실행 자원 사용은 늘 수 있다. 하지만 개발자가 로컬 CPU / RAM 점유에서 벗어나 다음 작업을 진행할 수 있다면 사람이 들이는 비용은 줄 수 있다.

반대로 작은 문구 수정 하나를 위해 클라우드 실행환경을 준비하고 에이전트가 저장소를 탐색하고 PR까지 만든다면 전체 비용이 더 커질 수 있다.

따라서 최적화 목표는 토큰 하나를 최소화하는 것이 아니다.

> 전체 개발 작업 흐름에서 실행 자원, LLM, 사람이 들이는 비용을 어디에 쓰는 것이 유리한지 판단하는 것이다.

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

클라우드 에이전트 비용을 하나의 숫자로 보지 않는다.

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
