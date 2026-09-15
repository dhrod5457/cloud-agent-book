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

Cloud Agent의 가치를 단순히 "원격에서 Claude를 실행한다"거나 "더 많은 AI 사용량을 확보한다"는 관점으로 설명하지 않는다.

이 장의 핵심 관점은 다음과 같다.

> 클라우드 에이전트의 중요한 장점 중 하나는 LLM 자체보다 독립된 실행 환경과 병렬 컴퓨팅 자원을 필요할 때 생성할 수 있다는 점이다.

## 핵심 주장

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

Cloud Agent를 항상 완전한 개발자 Agent로 사용할 필요는 없다. Build, Test, Docker Build, Migration Validation과 같이 명령으로 결정론적으로 실행할 수 있는 작업은 Cloud Test Runner에게 맡기고, LLM은 작업 계획과 실패 분석처럼 판단이 필요한 순간에만 개입하도록 설계할 수 있다.

## 독자가 얻는 것

- Local Agent, Cloud Agent, Hybrid Agent의 차이를 실행 환경 관점에서 설명할 수 있다.
- Cloud Agent의 컴퓨팅 자원과 LLM token/usage를 구분할 수 있다.
- Cloud Agent를 Test Runner 또는 Validation Node로 사용할 수 있다.
- 여러 Cloud Session을 병렬 테스트 노드로 배치할 수 있다.
- Result Filter를 사용해 LLM에 전달되는 로그와 Context를 줄일 수 있다.
- Agent Worker와 Test Runner를 구분할 수 있다.
- Cloud Worker에 Repository 전체가 아니라 작업 범위만 전달하는 방법을 설계할 수 있다.
- 내부망 작업은 Local Agent에 남기고 독립 검증 작업은 Cloud Agent로 분리하는 Hybrid 구조를 설계할 수 있다.

## 예제 Stage

Stage 1 - Local / Cloud 실행 위치를 분리하는 단계

아직 PM Agent orchestration 전체를 구현하지 않는다.

`campus-platform`에서 다음 작업을 구분한다.

- 로컬에서만 가능한 작업: Tibero, HSM, 내부 Jenkins, VPN 내부 API 검증
- 클라우드에서도 가능한 작업: Unit Test, Integration Test, Docker Build, 정적 분석, Repository 기반 검증

---

# 절 구성

## 2.1 Local Agent와 Cloud Agent의 차이는 모델이 아니라 실행 위치다

Local Agent와 Cloud Agent의 차이를 다음 관점으로 비교한다.

| 관점 | Local Agent | Cloud Agent |
| --- | --- | --- |
| 실행 위치 | 개발자 PC 또는 내부 서버 | 격리된 원격 실행 환경 |
| 로컬 파일 | 직접 접근 가능 | Repository 또는 업로드된 작업공간 중심 |
| VPN/내부망 | 접근 가능 | 일반적으로 제한될 수 있음 |
| 개발자 도구 | 기존 환경 활용 | 사전 구성 또는 setup 필요 |
| 병렬 실행 | 로컬 자원 한계 | 독립 세션을 통한 병렬화 가능 |
| 격리 | 별도 worktree/container 필요 | 세션별 격리 환경을 제공하는 제품이 많음 |

이 절에서는 21장의 기업 Hybrid Agent 구조를 미리 상세히 설명하지 않고, 실행 위치 차이만 정의한다.

## 2.2 클라우드 실행 자원과 LLM 사용량을 분리해서 생각한다

Cloud Agent의 VM/Container에서 사용하는 CPU, RAM, disk와 LLM token usage는 같은 자원이 아니다.

예를 들어 다음 명령이 클라우드 실행 환경에서 오래 실행될 수 있다.

- Gradle 전체 빌드
- JUnit 단위 테스트
- Spring Boot 통합 테스트
- Docker 이미지 빌드
- Testcontainers 실행
- npm build
- E2E 테스트
- 정적 분석
- DB Migration 검증

기본 흐름은 다음과 같이 설명한다.

```text
LLM이 실행할 명령을 결정
        ↓
Cloud VM / Container
CPU / RAM / Disk 사용
        ↓
Build / Test / Validation 실행
        ↓
실행 결과 생성
        ↓
필요한 결과만 LLM에 전달
        ↓
LLM이 실패 원인 또는 다음 작업 판단
```

LLM 사용량이 주로 발생하는 지점은 다음과 같이 정리한다.

- 작업 계획
- 소스코드와 문서 읽기
- 명령 실행 결과 읽기
- 로그 분석
- 수정 방향 판단
- 코드 생성과 리뷰

반대로 CPU 사용률이 높거나 테스트 프로세스가 오래 실행되는 시간 자체를 곧바로 LLM token 사용량으로 해석하지 않는다.

단, 실행 중 Claude가 반복적으로 상태를 확인하거나 대량 로그를 읽고 재추론하면 사용량은 증가할 수 있음을 명시한다.

## 2.3 핵심 원칙: CPU에는 일을 많이 시키고, LLM에는 결과를 적게 보여준다

이 절의 대표 예제로 대규모 테스트 로그를 사용한다.

### 좋지 않은 구조

```text
10,000개 테스트 실행
        ↓
100 MB stdout / stderr
        ↓
전체 로그를 LLM에 전달
        ↓
LLM 전체 로그 분석
        ↓
수정
        ↓
다시 전체 로그 전달
```

문제:

- 실패와 무관한 성공 로그까지 Context를 소비한다.
- 같은 로그의 반복 전달이 발생한다.
- Context가 커질수록 분석 비용과 오류 가능성이 증가한다.
- 병렬 Worker가 동일 패턴을 반복하면 사용량이 빠르게 증가한다.

### 권장 구조

```text
10,000개 테스트 실행
        ↓
exit code 확인
        ↓
실패 테스트 추출
        ↓
root cause 추출
        ↓
핵심 stack trace 추출
        ↓
구조화된 결과 생성
        ↓
LLM에 몇 KB 수준 결과 전달
```

예제 결과:

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

핵심 메시지:

> 테스트 실행량을 줄이는 것보다 LLM에게 반환되는 정보량을 줄이는 것이 token 절약에 더 중요할 수 있다.

테스트 10,000개를 CPU가 실행하는 것과 테스트 10,000개의 로그를 LLM이 읽는 것은 다른 비용 구조라는 점을 강조한다.

## 2.4 Cloud Agent를 Test Runner처럼 사용한다

Cloud Agent를 항상 다음 일을 모두 수행하는 개발자로 볼 필요는 없다.

```text
분석
→ 설계
→ 구현
→ 빌드
→ 테스트
→ 로그 분석
→ 수정
→ 재검증
```

일부 세션은 다음과 같은 실행 전용 역할로 제한할 수 있다.

- 전체 Build
- 전체 Test
- 모듈별 테스트 병렬 실행
- Integration Test
- E2E Test
- Docker Build
- Migration Validation
- Lint
- Static Analysis
- PR Validation

구조:

```text
Local Developer / PM Agent
            ↓
      구현 및 작업 분해
            ↓
     Cloud Test Runner
            ↓
 Build / Test / Validation
            ↓
       Result Filter
            ↓
      실패 결과만 반환
            ↓
 Local Agent / PM Agent 분석
```

핵심 관점:

> AI Agent가 모든 작업을 직접 수행하는 구조보다 AI는 판단하고 실행 환경은 명령을 수행하는 구조가 더 효율적인 경우가 있다.

## 2.5 Agent Worker와 Test Runner를 구분한다

Cloud Worker를 하나의 역할로 취급하지 않는다.

### Agent Worker

판단이 필요한 작업을 담당한다.

- 코드 분석
- 설계 판단
- 구현
- 복잡한 오류 분석
- 변경 범위 판단
- Review

### Test Runner

결정론적으로 실행 가능한 작업을 담당한다.

- Build
- Unit Test
- Integration Test
- Docker Build
- Lint
- Static Analysis
- Migration Validation
- E2E Test

기본 실행 정책:

```text
Task
  ↓
Test Runner 실행
  ├─ 성공 → 종료
  │
  └─ 실패
       ↓
   Result Filter
       ↓
   Agent Worker 호출
       ↓
      수정
       ↓
   Test Runner 재검증
```

이 구조의 목적은 단순히 비용 절감이 아니다.

- 결정론적인 작업과 추론 작업을 분리한다.
- 실패하지 않은 작업에 불필요한 LLM 분석을 사용하지 않는다.
- Agent Worker가 읽어야 하는 Context를 줄인다.
- 실행 노드를 쉽게 병렬화할 수 있다.

## 2.6 Result Filter

오케스트레이션 구조에 `Result Filter`를 명시적인 컴포넌트로 둔다.

```text
PM
 ↓
Task Scheduler
 ↓
Cloud Runner
 ↓
Raw Result
 ↓
Result Filter
 ↓
LLM
```

Result Filter의 책임:

- exit code 수집
- 성공/실패 건수 집계
- 실패 테스트명 추출
- root cause 후보 추출
- 핵심 stack trace 추출
- 중요 warning 추출
- 변경된 artifact 정보 수집
- 전체 로그 저장 위치 반환

원본 로그는 삭제하지 않는다.

LLM에는 축약된 결과를 전달하되, 추가 분석이 필요하면 특정 실패 로그만 다시 조회할 수 있게 한다.

```text
Summary
  ↓
LLM 판단
  ↓
추가 정보가 필요한가?
  ├─ 아니오 → 수정 또는 종료
  └─ 예
       ↓
   특정 테스트 로그만 조회
```

이는 이후 8장 실행 인터페이스, 10장 자동 검증, 17장 Observability와 연결한다.

## 2.7 여러 Cloud Session을 병렬 테스트 노드로 사용한다

예제 구조:

```text
Local PM Agent
|
+-- Cloud Session #1
|    Backend Unit Test
|
+-- Cloud Session #2
|    Integration Test
|
+-- Cloud Session #3
|    Frontend Test
|
+-- Cloud Session #4
|    Docker Build
|
+-- Cloud Session #5
     Migration Test
```

각 세션의 목적을 "프로젝트 전체를 이해하는 Agent"가 아니라 "주어진 검증 작업을 실행하는 노드"로 제한할 수 있다.

권장 방식:

```text
Task Contract
+ 실행 명령
+ 필요한 파일 범위
+ 완료 조건
        ↓
Cloud Session
        ↓
명령 실행
        ↓
Result Filter
        ↓
Summary 반환
```

피해야 할 방식:

```text
Cloud Session #1 → Repository 전체 분석
Cloud Session #2 → Repository 전체 분석
Cloud Session #3 → Repository 전체 분석
Cloud Session #4 → Repository 전체 분석
Cloud Session #5 → Repository 전체 분석
```

이 경우 세션마다 같은 Repository 탐색과 Context 구성이 반복되고, 병렬 컴퓨팅의 이점보다 LLM 사용량 증가가 커질 수 있다.

## 2.8 Context 최소화

Cloud Worker에게 Repository 전체 이해를 요구하기보다 Task Contract를 이용해 필요한 범위만 제공한다.

예:

```text
Task: 회원 탈퇴 API 테스트

관련 파일:
- UserController.java
- UserService.java
- UserRepository.java
- UserServiceTest.java

검증 명령:
./gradlew test --tests UserServiceTest

완료 조건:
- 정상 탈퇴 PASS
- 존재하지 않는 사용자 404
- 이미 탈퇴한 사용자 409

변경 금지:
- DB Schema
- 인증 모듈
- 공통 Exception 구조
```

이 예제는 6장 Task Contract를 선행해서 설명하지 않는다.

2장에서는 "작업 범위를 제한하면 Cloud Worker가 Repository 전체를 반복 탐색할 필요가 없다"는 원칙만 제시하고, 계약 형식 자체는 6장에서 상세히 다룬다고 연결한다.

## 2.9 Claude Code Web 사례

Claude Code Web은 제품 사례로만 사용한다.

2026년 9월 Anthropic 공식 문서에서 Cloud Session의 대략적인 resource limit은 다음과 같이 안내된다.

- 4 vCPU
- 16 GB RAM
- 30 GB disk

공식 문서는 이 값이 시간이 지나며 변경될 수 있는 대략적인 상한이라고 설명한다.

또한 각 작업이 격리된 환경에서 실행되며 여러 작업을 병렬로 실행할 수 있다고 설명한다.

개념적으로 다음과 같이 볼 수 있다.

```text
Claude 계정
|
+-- Cloud Session A
|    약 4 vCPU / 16 GB RAM
|    Backend Test
|
+-- Cloud Session B
|    약 4 vCPU / 16 GB RAM
|    Integration Test
|
+-- Cloud Session C
     약 4 vCPU / 16 GB RAM
     Docker Build
```

단 다음을 명확히 적는다.

- 위 수치는 현재 공개된 대략적인 session resource limit이며 고정 계약 사양이 아니다.
- 여러 세션이 존재한다고 해서 CPU/RAM이 물리적으로 전용 보장된다고 해석하지 않는다.
- 서비스 제공자의 동시 실행 제한과 스케줄링 정책이 있을 수 있다.
- Claude 사용량과 rate limit은 별도로 적용된다.
- 여러 세션에서 Claude가 동시에 Repository를 분석하고 추론하면 사용량을 더 빠르게 소비할 수 있다.
- Cloud compute resource와 LLM usage limit은 서로 다른 개념으로 취급한다.

제품 세부사항의 Source of Truth는 `research/anthropic/claude-code-web-execution-resources.md`로 둔다.

## 2.10 Local / Cloud / Hybrid 선택 기준

이 장의 기존 주제와 새 절을 다시 연결한다.

### Local Agent가 적합한 작업

- VPN 내부 시스템 접근
- 로컬에만 존재하는 개발환경 사용
- HSM 또는 사내 DB 검증
- uncommitted change를 포함한 빠른 상호작용

### Cloud Agent Worker가 적합한 작업

- 독립적으로 정의된 코드 변경
- Repository 기반 리팩터링
- 명확한 Acceptance Criteria가 있는 작업
- 장시간 비동기 작업

### Cloud Test Runner가 적합한 작업

- 전체 Build
- 전체 Test
- Module Test
- Docker Build
- Static Analysis
- Migration Validation
- PR Validation

### Hybrid가 적합한 작업

```text
Cloud
- 독립 구현
- CPU-heavy validation
- 병렬 테스트

Local
- VPN
- HSM
- Tibero / Oracle
- 내부 Jenkins

PM / Human
- 작업 분배
- 결과 통합
- 승인
```

상세한 기업 내부망 구조는 21장에서 다룬다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - 전체 테스트

### 나쁜 방식

```text
Cloud Agent 시작
→ Repository 전체 탐색
→ 전체 테스트 실행
→ 100MB 로그 전체 읽기
→ 테스트 성공 여부 판단
```

### 좋은 방식

```text
Cloud Test Runner 시작
→ ./gradlew test
→ Result Filter
→ BUILD PASS 또는 실패 요약만 반환
```

성공했다면 LLM 추가 분석 없이 종료한다.

## 사례 B - 모듈 병렬 검증

### 나쁜 방식

5개의 Agent가 각각 Repository 전체를 읽은 후 각자 테스트를 실행한다.

### 좋은 방식

PM이 먼저 작업을 분해하고 각 Runner에게 명령과 필요한 범위만 전달한다.

```text
runner-1 → :student:test
runner-2 → :attendance:test
runner-3 → :notification:test
runner-4 → integrationTest
runner-5 → migrationValidate
```

각 Runner는 결과 요약만 반환한다.

## 사례 C - 실패 시에만 추론 확대

```text
Fast Test
   ↓
PASS ─────────────→ 종료
   ↓ FAIL
Result Filter
   ↓
Agent Worker
   ↓
관련 코드 + 실패 로그 분석
   ↓
수정
   ↓
Test Runner
```

이를 `progressive reasoning` 패턴으로 설명할 수 있다. 처음부터 최대 Context를 사용하는 대신 실패가 발생한 지점에서만 추론 범위를 확대한다.

---

# 필요한 구조/그림

1. Local / Cloud / Hybrid 비교 그림
2. `LLM → Container → Result Filter → LLM` 흐름
3. Agent Worker / Test Runner 역할 분리
4. 5개 Cloud Session 병렬 Test Runner 구조
5. Bad: 전체 로그 반환 / Good: 실패 요약 반환 비교
6. Context 범위를 점진적으로 확장하는 흐름

---

# 필요한 코드/스크립트 예제

이 장의 본문에서는 애플리케이션 기능 코드보다 실행 인터페이스 예제를 사용한다.

후속 구현 단계에서 다음 예제를 고려한다.

- `scripts/test-unit.sh`
- `scripts/test-integration.sh`
- `scripts/validate-migration.sh`
- `scripts/filter-test-result.sh` 또는 동등한 Result Filter
- machine-readable JSON summary

구체적인 구현은 8장과 10장에서 다룬다.

예상 Result Filter 출력 구조:

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

이 JSON은 구조 설명용이며 이 단계에서는 실제 코드를 구현하지 않는다.

---

# 공식 자료 조사

현재 확인한 자료:

- Claude Code on the web: isolated VM, parallel task, GitHub repository workflow
- Claude Code cloud environment resource limits: 약 4 vCPU / 16 GB RAM / 30 GB disk
- Claude Code costs: token usage, context 관리
- Claude Code agents: parallel session/subagent 사용 시 usage 증가
- Claude Code usage limits: account/plan usage limit

조사 결과는 다음 파일에 보관한다.

`research/anthropic/claude-code-web-execution-resources.md`

제품 사양은 장 집필 시 다시 공식 문서를 확인한다.

---

# 다른 장과의 연결

## 1장

Agent Ready Software Engineering이 필요한 이유를 설명한 뒤 실행 위치의 차이로 확장한다.

## 6장 Task Contract

2장의 Context 최소화 예시를 정식 Task Contract 구조로 구체화한다.

## 8장 Agent 실행 인터페이스

Test Runner가 호출하는 `setup`, `test`, `verify` 명령을 표준화한다.

## 10장 자동 검증

Result Filter가 읽는 build/test/security 결과와 Definition of Done을 연결한다.

## 13장 Git / Worktree / Cloud Sandbox

병렬 Cloud Session의 Repository 격리 전략을 상세히 설명한다.

## 17장 Observability

다음 지표를 서로 분리해 관찰한다.

- LLM token/usage
- wall-clock execution time
- CPU/RAM resource usage
- retry count
- test result

## 19장 PM Agent

PM Agent가 Agent Worker와 Test Runner를 구분해서 스케줄링하는 구조로 확장한다.

## 21장 Hybrid Agent

내부망 검증을 Local Agent에 남기고 Cloud Runner 결과와 통합하는 기업 구조를 다룬다.

---

# 본문에서 의도적으로 다루지 않을 내용

- Claude 요금제별 상세 사용량 숫자 비교
- Cloud VM의 물리적 호스트 자원 보장 여부 추측
- 특정 제품의 동시 실행 session 수를 일반 원칙으로 고정
- CI 제품 사용법
- Result Filter의 구체적인 파서 구현
- PM Agent 스케줄러 구현

---

# 장의 결론 메시지

클라우드 에이전트의 가치를 "더 많은 AI를 사용하는 방법"으로 이해하면 동일한 Repository 분석과 추론을 여러 세션에서 반복하면서 사용량이 빠르게 증가할 수 있다.

반대로 클라우드 에이전트를 필요할 때 생성할 수 있는 독립된 실행 노드로 보면 활용 방식이 달라진다.

이 장은 다음 두 문장으로 정리한다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.
