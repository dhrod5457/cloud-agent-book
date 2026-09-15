# 2장 설계 - Local Agent, Cloud Agent, Hybrid Agent

## 장의 목표

Agent의 실행 위치에 따라 가능한 작업과 제한이 어떻게 달라지는지 설명한다.

이 장은 Local Agent와 Cloud Agent를 제품 이름으로 구분하지 않고 다음 기준으로 비교한다.

- 코드가 어디에서 실행되는가
- 어떤 네트워크에 접근할 수 있는가
- 어떤 컴퓨팅 자원을 사용할 수 있는가
- 작업 환경이 얼마나 격리되는가
- 여러 작업을 얼마나 쉽게 병렬화할 수 있는가
- 어떤 Context와 Secret을 전달해야 하는가
- 결정론적 실행과 LLM 판단을 어디서 분리할 것인가

Cloud Agent의 가치를 단순히 원격에서 LLM을 실행하거나 여러 AI 개발자를 동시에 띄우는 것으로 설명하지 않는다.

핵심 관점은 다음과 같다.

> 클라우드 에이전트의 중요한 장점 중 하나는 독립된 실행 환경과 병렬 컴퓨팅 자원을 필요할 때 생성할 수 있다는 점이다.

## 핵심 주장

이 장의 세 문장을 이후 장에서도 반복해서 사용하는 설계 원칙으로 정의한다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

따라서 목표는 AI를 최대한 많이 사용하는 시스템이 아니다.

> AI가 반드시 필요한 순간에만 호출되는 시스템을 설계한다.

Build, Test, Lint, Docker Build, Migration Validation처럼 결과가 명확한 작업은 Runner가 처리한다. PASS라면 작업을 종료한다. 실패하거나 사람이 정한 규칙만으로 다음 행동을 결정할 수 없을 때만 Agent Worker를 호출한다.

## 독자가 얻는 것

- Local Agent, Cloud Agent, Hybrid Agent의 차이를 실행 환경 관점에서 설명할 수 있다.
- Cloud compute resource와 LLM token/usage를 구분할 수 있다.
- Cloud Runner와 Agent Worker의 책임을 분리할 수 있다.
- 정상 경로에서는 LLM을 호출하지 않는 실행 구조를 설계할 수 있다.
- Result Filter로 Raw Result를 축약하고 필요한 실패 정보만 Agent에게 전달할 수 있다.
- Event-driven Agent와 Nightly Agent 패턴을 적용할 수 있다.
- 독립 작업을 fan-out하여 여러 Runner에서 병렬 실행할 수 있다.
- UI/E2E처럼 CPU와 시간이 많이 드는 검증을 Agent Context와 분리할 수 있다.
- Agent Harness를 Repository에 설계할 수 있다.
- Cloud Agent 호출 시 Repository 전체가 아니라 필요한 Context만 제공할 수 있다.

## 예제 Stage

Stage 1 - Local / Cloud 실행 위치를 분리하는 단계

아직 PM Agent orchestration 전체를 구현하지 않는다. 다만 후속 장에서 PM Agent가 사용할 수 있도록 Runner와 Agent Worker 사이의 인터페이스를 먼저 정의한다.

`campus-platform`에서 다음 작업을 구분한다.

- 로컬 중심: Tibero, HSM, 내부 Jenkins, VPN 내부 API 검증
- Cloud Runner: Unit Test, Integration Test, E2E, Docker Build, Lint, Static Analysis, Migration Validation
- Agent Worker: 실패 원인 분석, 코드 수정, 설계 판단, 복잡한 충돌 해결

---

# 절 구성

## 2.1 Local Agent와 Cloud Agent의 차이는 모델이 아니라 실행 위치다

Local Agent와 Cloud Agent를 실행 위치와 접근 범위로 비교한다.

| 관점 | Local Agent | Cloud Agent |
| --- | --- | --- |
| 실행 위치 | 개발자 PC 또는 내부 서버 | 격리된 원격 실행 환경 |
| 로컬 파일 | 직접 접근 가능 | Repository 또는 작업공간 중심 |
| VPN/내부망 | 접근 가능 | 일반적으로 제한될 수 있음 |
| 개발자 도구 | 기존 환경 활용 | 자동 setup 필요 |
| 병렬 실행 | 로컬 자원 한계 | 독립 실행 환경으로 fan-out 가능 |
| 격리 | worktree/container 등을 별도 구성 | 세션/작업별 격리 환경을 제공하는 제품이 많음 |

21장의 기업 Hybrid Agent 구조를 여기서 반복하지 않는다. 2장에서는 실행 위치의 차이와 Runner/Agent 분리 원칙까지만 설명한다.

## 2.2 클라우드 실행 자원과 LLM 사용량을 분리한다

Cloud VM/Container의 CPU, RAM, disk와 LLM token usage는 서로 다른 자원이다.

다음 작업이 클라우드 환경에서 오래 실행될 수 있다.

- Gradle 전체 빌드
- JUnit 단위 테스트
- Spring Boot 통합 테스트
- Docker 이미지 빌드
- Testcontainers 실행
- npm build
- E2E 테스트
- 정적 분석
- DB Migration 검증

기본 흐름:

```text
LLM이 필요한 명령 또는 작업 범위를 결정
        ↓
Cloud VM / Container
CPU / RAM / Disk 사용
        ↓
Build / Test / Validation
        ↓
실행 결과 생성
        ↓
Result Filter
        ↓
필요한 결과만 LLM에 전달
        ↓
LLM이 실패 원인 또는 다음 작업 판단
```

LLM 사용량이 주로 발생하는 시점:

- 작업 계획
- 소스코드와 문서 읽기
- 명령 결과 읽기
- 로그 분석
- 수정 방향 판단
- 코드 생성과 리뷰

CPU가 100% 사용되거나 테스트가 수십 분 실행되는 시간 자체를 곧바로 token 사용량으로 해석하지 않는다. 반대로 실행 중 Agent가 반복적으로 상태를 조회하거나 대량 로그를 읽으면 LLM 사용량은 증가할 수 있다.

## 2.3 Runner-first: 정상 경로에서는 Agent를 호출하지 않는다

이 장의 핵심 아키텍처다.

```text
Local Developer / PM Agent
            ↓
       Task Scheduler
            ↓
        Cloud Runner
            ↓
 Build / Test / Lint / Validation
            ↓
       +----+----+
       |         |
      PASS      FAIL
       |         |
      종료   Result Filter
                 ↓
           실패 정보 요약
                 ↓
            Agent Worker
                 ↓
          원인 분석 / 수정
                 ↓
            Cloud Runner
                 ↓
              재검증
                 ↓
                PR
```

Agent는 항상 실행되는 기본 경로가 아니다.

- PASS: Agent 호출 없이 종료
- FAIL: Result Filter로 원인을 축약한 뒤 Agent Worker 호출
- 판단 불가: 필요한 Context만 추가 조회
- 수정 후: 반드시 Runner가 다시 검증

이를 `Runner-first / Agent-on-exception` 패턴으로 정의한다.

## 2.4 토큰 절약은 성공 경로에서 Agent를 없애는 것부터 시작한다

가정:

```text
테스트 실행: 100회
성공: 95회
실패: 5회
```

비효율적 구조:

```text
100회 테스트
×
100회 Agent 개입
```

권장 구조:

```text
100회 Cloud Runner 실행

95회 PASS
→ Agent 호출 없음

5회 FAIL
→ Result Filter
→ 실패 5건만 Agent 호출
```

이 패턴은 성공 경로의 LLM 추론을 제거한다.

GitHub의 2026년 Agentic Workflows 토큰 최적화 사례에서도 결정론적인 데이터 수집을 사전 CLI 단계로 이동하고, 관련성이 없는 입력은 relevance gate에서 LLM 호출 자체를 건너뛰는 방식이 사용되었다. 본문에서는 특정 제품 최적화 기법 자체보다 다음 일반 원칙을 추출한다.

> 가장 저렴한 LLM 호출은 하지 않는 호출이다.

출처는 `research/github/continuous-ai-runner-first.md`에 분리한다.

## 2.5 CPU에는 일을 많이 시키고, LLM에는 결과를 적게 보여준다

예를 들어 테스트 10,000건이 100MB 로그를 만든다고 가정한다.

### 좋지 않은 구조

```text
테스트 실행
→ 전체 stdout/stderr 전달
→ LLM이 전체 로그 분석
→ 수정
→ 다시 전체 로그 전달
```

### 권장 구조

```text
테스트 실행
→ exit code 확인
→ 실패 테스트 추출
→ root cause 추출
→ 핵심 stack trace 추출
→ 몇 KB 수준의 구조화 결과 생성
→ LLM에 전달
```

예제:

```text
BUILD: FAIL

Tests:
- total: 8,214
- passed: 8,211
- failed: 3

Failures:
1. UserServiceTest.deleteUser
   UserService.java:142
   NullPointerException
2. AuthServiceTest.expiredToken
   expected: 401
   actual: 200
3. UserMapperTest.insert
   duplicate key
```

핵심은 테스트 실행량 자체보다 LLM에 반환되는 정보량과 LLM 호출 횟수를 먼저 줄이는 것이다.

## 2.6 Result Filter는 Agent가 아니라 일반 프로그램으로 구현한다

권장 구조:

```text
Cloud Runner
     ↓
 Raw Result
     ↓
Result Filter
     ├─ exit code
     ├─ failed tests
     ├─ root cause 후보
     ├─ 핵심 stack trace
     ├─ error count
     ├─ warning count
     └─ artifact path
     ↓
 Agent Worker
```

Result Filter의 기본 구현은 shell, Java/Python utility, test-report parser, CI script처럼 결정론적인 프로그램을 사용한다.

원본 로그가 100MB여도 Agent에게 전달되는 기본 결과는 몇 KB 수준을 목표로 한다. 원본 로그는 artifact로 보존하고 필요할 때만 특정 구간을 추가 조회한다.

Context 확대 순서:

```text
Summary
  ↓
Failure Detail
  ↓
Specific Test Log
  ↓
Relevant Source Files
  ↓
Full Raw Log
```

## 2.7 Cloud Runner와 Agent Worker를 분리한다

### Cloud Runner

역할:

- build
- unit test
- integration test
- E2E
- Docker build
- lint
- static analysis
- migration validation
- security scan

특징:

- CPU/RAM 중심
- LLM 추론 불필요
- 정해진 명령 실행
- exit code와 artifact 생성
- 병렬화하기 쉬움

### Agent Worker

역할:

- 코드 분석
- 원인 판단
- 설계 결정
- 수정안 생성
- 복잡한 오류 분석

특징:

- LLM 사용량 발생
- Context 크기가 중요
- 호출 빈도와 탐색 범위를 통제해야 함

기본 정책:

```text
가능하면 Runner
→ Runner가 해결할 수 없는 실패 또는 판단 문제
→ Agent Worker
```

## 2.8 GitHub Next Continuous AI: 작은 작업을 지속적으로 반복한다

GitHub가 공개한 Continuous AI 실험 중 테스트 개선 사례를 설계 사례로 사용한다.

공개된 실험 결과:

- 약 45일
- 1,400개 이상 테스트 생성
- 테스트 커버리지 약 5%에서 거의 100% 수준으로 증가
- 공개된 실험 기준 약 80달러의 token 비용
- 큰 변경 하나가 아니라 작은 PR을 지속적으로 생성

이 수치는 특정 프로젝트와 당시 모델/가격을 기준으로 한 사례이며 일반적인 비용 예측값으로 사용하지 않는다.

이 사례에서 추출할 원칙:

```text
Coverage 분석
      ↓
작은 미검증 영역 선택
      ↓
Agent가 테스트 생성
      ↓
Runner에서 테스트
      ↓
PASS
      ↓
작은 PR
      ↓
다음 영역 반복
```

예:

```text
1일차: UserService 미검증 branch
2일차: AuthService 미검증 branch
3일차: PaymentService 미검증 branch
```

핵심 메시지:

> 거대한 Agent 작업 하나보다 작고 검증 가능한 자동화 작업을 지속적으로 반복하는 편이 실패 범위와 리뷰 비용을 줄일 수 있다.

GitHub는 Continuous AI를 기존 CI/CD의 대체재가 아니라, 결정론적 CI/CD로 표현하기 어려운 판단 작업을 보완하는 구조로 설명한다.

## 2.9 Nightly Agent: 이벤트가 있을 때만 Agent를 깨운다

GitHub의 Copilot cloud agent 자동화 사례에는 nightly failing-test fix 패턴이 공개되어 있다.

```text
Nightly Scheduler
       ↓
   Test Runner
       ↓
   +---+---+
   |       |
  PASS    FAIL
   |       |
  종료      ↓
       Cloud Agent
           ↓
       수정 시도
           ↓
       Draft PR
```

책에서는 이를 특정 제품 기능이 아니라 `Event-driven Agent` 패턴으로 추상화한다.

Agent는 24시간 계속 실행될 필요가 없다.

- schedule
- CI failure
- review comment
- security alert
- 특정 repository event

같은 조건이 발생했을 때만 생성하거나 활성화할 수 있다.

## 2.10 Claude Code Web PR Auto-fix: CI 실패를 Agent 활성화 이벤트로 사용한다

Anthropic 공식 문서의 PR auto-fix는 이벤트 기반 Agent의 실제 사례로 사용한다.

```text
GitHub PR
   ↓
   CI
   ↓
+--+--+
|     |
PASS FAIL
|     |
종료   ↓
   Claude Agent
       ↓
   원인 분석
       ↓
    코드 수정
       ↓
      Push
```

공식 문서 기준 auto-fix가 활성화된 PR에서 CI check failure 또는 review comment가 발생하면 Claude가 이벤트를 받아 조사하고, 명확한 수정이 있다고 판단하면 변경을 push할 수 있다.

이 사례에서 제품 기능보다 다음 원칙을 강조한다.

> Agent를 상시 실행 프로세스로 만들지 말고, 판단이 필요한 이벤트에 반응하는 실행 단위로 설계할 수 있다.

제품별 세부 동작과 제한은 `research/anthropic/claude-code-web-execution-resources.md`에 둔다.

## 2.11 Fan-out: 독립 작업을 병렬 실행 노드로 분할한다

Java 17에서 Java 21로 여러 서비스 또는 모듈을 마이그레이션한다고 가정한다.

```text
Migration Controller
        |
+-------+-------+-------+-------+
|       |       |       |       |
A       B       C       D     ...
|       |       |       |
Runner  Runner  Runner  Runner
|       |       |       |
Build   Build   Build   Build
Test    Test    Test    Test
|       |       |       |
PR      PR      PR      PR
```

작업이 독립적이면 순차 실행보다 fan-out이 유리할 수 있다.

단, 각 Worker가 Repository 전체를 다시 분석하게 만들면 같은 Context를 반복 소비한다. Scheduler가 모듈, 명령, 관련 파일, 완료 조건을 미리 제한해야 한다.

Fan-out의 목적은 Agent 수를 늘리는 것이 아니라 독립적인 컴퓨팅 작업을 병렬화하는 것이다.

## 2.12 UI/E2E 반복 검증은 Container에 맡긴다

CPU/RAM과 실행 시간이 많이 필요한 예로 UI/E2E 테스트를 사용한다.

```text
Cloud Container
→ Playwright
→ 회원가입 반복 테스트
→ 로그인 시나리오
→ 잘못된 입력
→ timeout
→ retry
→ accessibility
→ multi-step form
```

예를 들어 1,200개 시나리오를 실행했다면 브라우저 로그 전체를 Agent에게 전달하지 않는다.

```text
총 시나리오: 1,200
성공: 1,187
실패: 13

공통 실패:
- timeout: 8
- validation mismatch: 3
- accessibility: 2
```

필요하면 실패 13건 중 특정 artifact, screenshot, trace만 단계적으로 조회한다.

## 2.13 Context 최소화

Agent 호출 시 Repository 전체 이해를 요구하지 않는다.

예:

```text
Task:
AuthServiceTest.expiredToken 실패 수정

관련 파일:
- AuthService.java
- AuthServiceTest.java
- JwtTokenProvider.java

실패:
expected: 401
actual: 200

관련 stack:
AuthServiceTest.java:94

검증:
./gradlew test --tests AuthServiceTest.expiredToken

변경 금지:
- DB schema
- 전체 JWT 구조
- 공통 Exception API
```

2장에서는 Context 최소화의 원칙만 설명하고, Task Contract 형식은 6장에서 상세히 다룬다.

## 2.14 Agent Harness: 프롬프트보다 실행 환경을 먼저 개선한다

Agent가 실패할 때마다 더 긴 프롬프트를 제공하는 방식에는 한계가 있다.

Repository 안에 Agent와 Runner가 사용할 수 있는 반복 가능한 실행 장치를 준비한다.

Agent Harness 후보:

- 명확한 build 명령
- test 명령
- lint 명령
- validation script
- architecture 문서
- coding convention
- 작은 Task 단위
- Result Filter/parser
- machine-readable report
- PR template
- artifact 규칙

Agent Harness는 특정 제품의 instruction 파일 하나를 의미하지 않는다. Agent가 프로젝트를 이해하고 실행하고 검증하는 데 필요한 Repository 수준의 지원 구조를 묶어서 설명하기 위한 개념으로 사용한다.

4장 Repository as Interface, 5장 Agent Contract, 8장 실행 인터페이스, 10장 자동 검증에서 각각 구체화한다.

## 2.15 Claude Code Web 실행 환경 사례

Claude Code Web은 클라우드 실행 환경 사례로만 사용한다.

2026년 9월 Anthropic 공식 문서에서 Cloud Session의 대략적인 resource limit은 다음과 같이 안내된다.

- 4 vCPU
- 16 GB RAM
- 30 GB disk

공식 문서는 이 값이 변경될 수 있는 대략적인 상한이라고 설명한다. 또한 작업은 격리된 VM에서 수행되고 여러 독립 작업을 병렬로 실행할 수 있다.

주의사항:

- 세션별 자원 수치는 물리적 전용 CPU/RAM 보장을 뜻하지 않는다.
- 동시 실행과 스케줄링 정책은 서비스 제공자에 따라 달라질 수 있다.
- Claude 사용량/rate limit은 별도로 적용된다.
- 여러 세션에서 동시에 LLM 추론을 수행하면 사용량 제한을 더 빠르게 소비할 수 있다.
- Cloud compute와 LLM usage는 서로 다른 자원으로 본다.

본문의 핵심 논리는 이 수치에 의존하지 않는다. 제품 사양은 research 문서에서 기준일과 함께 관리한다.

## 2.16 최종 권장 아키텍처

```text
                 PM / Orchestrator
                        |
                 Task Scheduler
                        |
           +------------+------------+
           |            |            |
      Runner A      Runner B      Runner C
      Unit Test      E2E Test      Build
           |            |            |
           +------------+------------+
                        |
                     Result
                        |
                +-------+-------+
                |               |
               PASS            FAIL
                |               |
               종료        Result Filter
                                |
                                v
                          Agent Worker
                                |
                           코드 수정
                                |
                                v
                          Cloud Runner
                                |
                             재검증
                                |
                                v
                               PR
```

최적화 목표:

1. Agent 호출 횟수 최소화
2. Agent에게 전달하는 Context 최소화
3. CPU/RAM 작업을 Runner로 최대한 이전

이 구조의 목적은 Agent 수를 늘리는 것이 아니다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - 전체 테스트

나쁜 방식:

```text
Cloud Agent 시작
→ Repository 전체 탐색
→ 전체 테스트
→ 전체 로그 분석
→ 성공 여부 판단
```

좋은 방식:

```text
Cloud Runner
→ ./gradlew test
→ Result Filter
→ PASS면 종료
→ FAIL일 때만 Agent 호출
```

## 사례 B - 모듈 병렬 검증

나쁜 방식:

```text
Agent A → Repository 전체 분석 → student test
Agent B → Repository 전체 분석 → attendance test
Agent C → Repository 전체 분석 → notification test
```

좋은 방식:

```text
runner-1 → :student:test
runner-2 → :attendance:test
runner-3 → :notification:test
runner-4 → integrationTest
runner-5 → migrationValidate
```

각 Runner는 summary와 artifact 위치만 반환한다.

## 사례 C - 실패 시에만 추론 확대

```text
Fast Verification
      ↓
     PASS ─────────→ 종료
      |
     FAIL
      ↓
Result Filter
      ↓
Agent Worker
      ↓
관련 코드 + 실패 정보 분석
      ↓
수정
      ↓
Runner 재검증
```

## 사례 D - Nightly regression

```text
매일 밤 main
→ Runner 전체 테스트
→ PASS: 종료
→ FAIL: 실패 요약
→ Agent Worker
→ 수정 가능한 경우 Draft PR
```

---

# 필요한 구조/그림

1. Local / Cloud / Hybrid 비교
2. Compute Resource와 LLM Usage 분리
3. Runner-first / Agent-on-exception 흐름
4. Raw Result → Result Filter → Agent
5. 100회 테스트 중 실패 5건만 Agent 호출하는 비용 구조
6. Nightly Event-driven Agent
7. PR CI failure → Agent Auto-fix
8. Fan-out migration
9. UI/E2E 대량 실행과 결과 축약
10. 최종 PM / Runner / Agent Worker 아키텍처

---

# 필요한 코드/스크립트 예제

본문에서는 애플리케이션 기능 코드보다 실행 인터페이스와 결과 포맷을 사용한다.

후속 구현 후보:

```text
scripts/
├─ verify-fast.sh
├─ verify-full.sh
├─ test-unit.sh
├─ test-integration.sh
├─ test-e2e.sh
├─ validate-migration.sh
└─ filter-result.sh
```

구체적인 구현은 8장과 10장에서 다룬다.

예상 Result Filter 출력:

```json
{
  "status": "FAIL",
  "exitCode": 1,
  "tests": {
    "total": 8214,
    "passed": 8211,
    "failed": 3
  },
  "failures": [
    {
      "test": "UserServiceTest.deleteUser",
      "location": "UserService.java:142",
      "cause": "NullPointerException"
    }
  ],
  "rawLog": "artifacts/test/full.log"
}
```

이 JSON은 구조 설명용이며 Phase 5에서는 구현하지 않는다.

---

# 공식 자료 조사

## GitHub

- Continuous AI: deterministic CI/CD와 reasoning task의 구분
- Continuous test improvement: 약 45일, 1,400+ tests, coverage 약 5% → near 100%, 약 $80 token 사례
- Agentic Workflows token optimization: deterministic data gathering을 pre-agentic step으로 이동, relevance gate로 불필요한 LLM 호출 제거
- Copilot cloud agent automation: nightly failing tests → fix attempt → draft PR
- Agentic Workflows: schedule/repository event 기반 trigger

자료:

- https://github.blog/ai-and-ml/generative-ai/continuous-ai-in-practice-what-developers-can-automate-today-with-agentic-ci/
- https://githubnext.com/projects/continuous-ai/
- https://github.blog/ai-and-ml/github-copilot/improving-token-efficiency-in-github-agentic-workflows/
- https://github.blog/changelog/2026-06-02-schedule-and-automate-tasks-with-copilot-cloud-agent/
- https://github.blog/changelog/2026-02-13-github-agentic-workflows-are-now-in-technical-preview/

상세 정리는 `research/github/continuous-ai-runner-first.md`에 둔다.

## Anthropic

- Claude Code on the web isolated VM / parallel session
- 현재 공개된 대략적 resource limit
- usage/rate limit
- PR auto-fix: CI failure/review comment event에 반응

자료:

- https://code.claude.com/docs/ko/claude-code-on-the-web
- https://code.claude.com/docs/en/web-quickstart
- https://code.claude.com/docs/en/costs

상세 정리는 `research/anthropic/claude-code-web-execution-resources.md`에 둔다.

---

# 다른 장과의 연결

## 4장 Repository as Interface

Agent Harness 중 탐색성과 canonical source를 구체화한다.

## 5장 Agent Contract

Agent에게 전달해야 할 프로젝트 규칙과 instruction precedence를 정의한다.

## 6장 Task Contract

2장의 Context 최소화 예시를 정식 작업 계약으로 구체화한다.

## 8장 Agent 실행 인터페이스

Runner가 호출하는 `setup`, `test`, `verify` 명령을 표준화한다.

## 10장 자동 검증

Result Filter가 수집하는 결과와 Definition of Done을 연결한다.

## 13장 Git / Worktree / Cloud Sandbox

fan-out된 작업의 Repository 격리 전략을 다룬다.

## 17장 Observability

다음 지표를 분리해 관찰한다.

- LLM token/usage
- Agent invocation count
- wall-clock execution time
- CPU/RAM resource usage
- retry count
- raw/filtered result size
- test result

## 19장 PM Agent

PM Agent가 Runner와 Agent Worker를 별도 자원으로 스케줄링하는 구조로 확장한다.

## 21장 Hybrid Agent

내부망 검증을 Local Runner/Agent에 남기고 Cloud Runner 결과와 통합한다.

---

# 본문에서 의도적으로 다루지 않을 내용

- 특정 제품 요금제 비교
- 현재 가격을 장기적인 비용 기준으로 사용하는 것
- Cloud VM 자원이 물리적으로 전용이라는 추측
- 특정 제품의 동시 실행 세션 수를 일반 원칙으로 고정하는 것
- CI 제품 사용법 자체
- Result Filter 구현 세부 코드
- PM Agent Scheduler 구현

---

# 장의 결론 메시지

클라우드 에이전트의 장점을 단순히 여러 AI 개발자를 동시에 실행할 수 있다는 의미로 이해하면 동일한 Repository 분석과 추론을 반복하면서 사용량이 증가할 수 있다.

더 효율적인 구조에서는 클라우드 환경을 필요할 때 생성하는 실행 노드로 보고, 결정론적 작업은 Runner에 맡긴다. 정상 경로에서는 Agent를 호출하지 않고, 실패하거나 판단이 필요한 예외 경로에서만 Agent를 호출한다.

이 장은 다음 세 문장으로 정리한다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.
