# 3장 설계 - Cloud Session, Container, Compute와 Token

## 장의 목표

Cloud Agent의 실행 자원과 LLM 사용량을 분리해서 이해하게 한다.

이 장의 핵심 질문은 다음과 같다.

> Cloud Agent가 오래 Build/Test를 실행하는 시간과 LLM Token 사용량은 같은 비용인가?

답은 아니다.

Cloud Agent를 Remote Worker로 보면 최소한 두 종류의 자원이 존재한다.

```text
Reasoning Resource
- LLM input/output/reasoning

Execution Resource
- CPU
- RAM
- Disk
- Process
- Container / VM
- Development Tools
```

두 자원은 상호작용하지만 같은 개념이 아니다.

---

## 핵심 주장

> 클라우드 에이전트의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

이 장에서는 이 네 문장을 Cloud Agent 비용과 실행 구조의 기초로 사용한다.

---

## 독자가 얻는 것

- Cloud Session과 실행 Container/VM의 역할을 설명할 수 있다.
- CPU/RAM/Disk 사용과 LLM Token 사용을 구분할 수 있다.
- Build/Test가 오래 걸려도 그 시간 자체가 Token 소비를 뜻하지 않는 이유를 이해한다.
- 어떤 시점에 Token이 주로 사용되는지 설명할 수 있다.
- 대량 실행 결과를 LLM Context와 분리해야 하는 이유를 이해한다.
- Cloud Session 여러 개를 병렬 실행 노드로 보는 기초를 이해한다.

---

# 개념 모델

## Brain / Hands의 간단한 모델

이 책에서 Brain/Hands는 Agent Platform 아키텍처로 확장하지 않는다.

Compute와 Token을 구분하기 위한 설명 모델로만 사용한다.

```text
Brain
= LLM
= 계획 / 판단 / 분석 / 수정 방향 결정

Hands
= Cloud Execution Environment
= Shell / CPU / RAM / Disk / Build / Test / Browser
```

기본 흐름:

```text
LLM
명령과 작업 범위 결정
        ↓
Cloud Container / VM
CPU / RAM / Disk 사용
        ↓
Build / Test / Validation 실행
        ↓
실행 결과 생성
        ↓
필요한 결과만 LLM에 전달
        ↓
LLM
실패 원인 또는 다음 행동 판단
```

핵심은 `Hands가 계산하는 시간`과 `Brain이 읽고 판단하는 양`을 분리하는 것이다.

---

# 절 구성

## 3.1 Cloud Session은 독립 작업 단위다

Cloud Session은 제품마다 구현 방식이 다르지만 책에서는 다음 추상화로 설명한다.

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

중요한 것은 계정 전체에 하나의 컴퓨터가 있다는 식으로 단순화하지 않는 것이다.

제품에 따라 작업별 격리 VM/Container/Workspace가 만들어질 수 있고 여러 Session을 동시에 실행할 수 있다.

구체적인 CPU/RAM/Disk 수치와 동시 실행 제한은 변경 가능하므로 본문 핵심 논리와 분리한다.

관련 제품 사양은 `research/`에서 기준일과 출처를 관리한다.

## 3.2 Compute와 Token은 다른 자원이다

예를 들어 Cloud Container에서 다음 작업이 실행될 수 있다.

- `./gradlew build`
- JUnit 전체 테스트
- Spring Boot Integration Test
- Testcontainers
- Docker Build
- npm build
- Playwright E2E
- Static Analysis
- Migration Validation

이 작업은 주로 CPU/RAM/Disk와 실행시간을 사용한다.

LLM Token은 주로 다음 시점에 사용된다.

- Agent가 Task를 읽을 때
- Repository와 문서를 읽을 때
- 실행할 명령을 계획할 때
- Tool Result를 읽을 때
- 실패 로그를 분석할 때
- 수정 방향을 판단할 때
- 코드를 생성하거나 Review할 때

따라서 다음 두 상황을 구분한다.

```text
CPU 100%
테스트 30분 실행
```

과

```text
LLM이 30분 동안 계속 대량 Context를 읽고 추론
```

은 같은 비용 구조가 아니다.

## 3.3 테스트 10,000개를 실행하는 것과 10,000개 로그를 읽는 것은 다르다

예:

```text
Cloud Container
→ 10,000 tests 실행
→ CPU/RAM 사용
→ 100MB log 생성
```

좋지 않은 구조:

```text
100MB log
→ LLM 전체 전달
→ 전체 로그 분석
```

권장 개념:

```text
100MB log / test report
→ 프로그램이 실패 항목 추출
→ 실패 3건 요약
→ LLM은 실패 3건부터 분석
```

이 장에서는 원칙만 설명한다.

Result Filter / Result Gateway 구현은 8장에서 다룬다.

## 3.4 LLM이 기다리는 것과 Token을 소비하는 것을 구분한다

외부 프로세스가 실행되는 동안 Agent가 아무 추가 추론도 하지 않는다면 wall-clock time은 흐르지만 그 시간 자체가 Token과 1:1로 증가하는 것은 아니다.

반대로 다음 행동은 Token 사용을 늘릴 수 있다.

- 상태를 짧은 간격으로 반복 조회
- 전체 로그를 반복해서 읽음
- 같은 Repository를 여러 Session에서 다시 분석
- 실패할 때마다 대량 Context를 재전송
- 여러 Agent가 동시에 추론

따라서 Cloud Agent 최적화에서는 다음을 함께 본다.

```text
Compute Time
LLM Usage
Tool Output Size
Context Size
Agent Invocation Count
```

## 3.5 Compute-heavy / Context-light 작업을 찾는다

Cloud에 특히 보내기 좋은 형태 중 하나는 다음이다.

```text
Compute는 큼
Context는 작음
판정은 명확함
```

예:

- 전체 Unit Test
- Integration Test
- Docker Build
- E2E 반복 실행
- Static Analysis
- Migration Validation

이런 작업은 Cloud CPU/RAM을 적극적으로 사용하면서 LLM에게는 결과 요약만 전달할 수 있다.

구체적인 Cloud Task 선택 기준은 5장과 6장에서 다룬다.

## 3.6 여러 Session의 병렬 Compute와 병렬 Reasoning을 구분한다

다음 두 구조는 다르다.

### 병렬 Compute

```text
Session A → Backend Test
Session B → Integration Test
Session C → E2E
Session D → Docker Build
```

각 Session이 정해진 명령을 실행하고 요약만 반환한다.

### 병렬 Reasoning

```text
Agent A → Repository 전체 분석
Agent B → Repository 전체 분석
Agent C → Repository 전체 분석
Agent D → Repository 전체 분석
```

두 번째 구조는 같은 Context 탐색과 LLM 추론이 반복될 수 있다.

따라서 Cloud 병렬화의 목적을 단순히 Agent 수 증가로 설명하지 않는다.

병렬 Worker와 중복 Context 비용은 12장에서 상세히 다룬다.

## 3.7 비용을 하나의 숫자로 보지 않는다

Cloud Agent 활용 비용은 최소 세 관점으로 나눌 수 있다.

### Compute

- CPU
- RAM
- Disk
- Container/VM runtime

### LLM

- input token
- output token
- reasoning/usage
- Tool Result 처리

### Human

- 기다리는 시간
- 오류 재현 시간
- Review
- Context Switching

책의 목표는 Token만 최소화하는 것이 아니다.

예를 들어 Cloud CPU를 더 사용해서 개발자가 30분짜리 테스트를 기다리지 않아도 된다면 전체 개발 비용은 낮아질 수 있다.

이 장에서는 비용 구조를 정의하고 실제 Task 선택은 뒤 장에서 다룬다.

## 3.8 제품별 Compute 사양은 사례로만 사용한다

Claude Code Web 등 실제 Cloud Coding Agent의 공개 사양은 개념을 설명하는 사례로 사용할 수 있다.

다만 다음은 핵심 원칙에서 분리한다.

- vCPU 수
- RAM 크기
- Disk 크기
- Session 동시 실행 제한
- 가격
- Rate Limit

이 값들은 변경될 수 있기 때문이다.

본문에서는 기준일을 명시한 박스 또는 각주로만 사용한다.

기존 조사:

- `research/anthropic/claude-code-web-execution-resources.md`

---

# 실전 예제 - campus-platform

## 예제 A - 전체 테스트

```text
Cloud Worker
→ ./gradlew test
→ CPU/RAM 사용
→ 8,214 tests 실행
→ 3 failures

Agent
→ 실패 3건만 우선 분석
```

## 예제 B - 병렬 검증

```text
Cloud #1
→ :student:test

Cloud #2
→ :attendance:test

Cloud #3
→ integrationTest

Cloud #4
→ Docker Build
```

각 Worker가 Repository 전체를 다시 분석하지 않도록 뒤 장에서 Task/Context를 줄인다.

## 예제 C - E2E

```text
Playwright
→ 1,200 scenarios
→ Browser/CPU/RAM 사용

Result
→ passed 1,187
→ failed 13

LLM
→ 13건 중 필요한 failure부터 분석
```

---

# 필요한 구조/그림

1. Cloud Agent = Brain + Hands 간단 모델
2. Compute Resource와 LLM Usage 분리
3. LLM → Container → Result → LLM 흐름
4. 10,000 Test / 3 Failure 예시
5. Parallel Compute vs Parallel Reasoning
6. Compute / LLM / Human Cost 구분

---

# 다음 장과의 연결

4장에서는 독립 실행환경을 실제로 왜 사용하는지 설명한다.

- 개발자 PC와 분리
- 장시간 작업 위임
- Developer Blocking Time 감소
- 병렬 실행

8장에서는 Tool Output과 Result Gateway를 이용해 LLM에게 전달하는 정보를 줄인다.

9장에서는 Prebuilt Environment와 Cache로 Cloud Compute의 시작 비용을 줄인다.

10장에서는 일반 Runner와 Cloud Agent를 분리한다.

---

# 본문에서 의도적으로 다루지 않을 내용

- Agent Platform 일반 아키텍처
- Brain/Hands lifecycle 분리
- Agent Memory
- Agent-native Observability
- PM Agent scheduling
- 제품별 가격 비교
- 특정 제품의 자원 수치를 일반 기준으로 고정하는 것

Brain/Hands는 Compute와 Token을 설명하기 위한 단순 모델로만 사용한다.
