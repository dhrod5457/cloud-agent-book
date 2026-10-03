# 15장. campus-platform Cloud Agent Workflow 설계

지금까지는 클라우드 에이전트 활용 원칙을 각각 나눠서 설명했다.

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

새로운 에이전트 플랫폼을 설계하기보다는 **실제 Java/Spring Boot 프로젝트에서 로컬 에이전트, 클라우드 실행기, 클라우드 에이전트를 어느 단계에 배치할지** 연결해 본다.

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

이 작업 흐름의 중심은 에이전트가 아니다.

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

책에서 사용하는 `campus-platform`은 클라우드 에이전트 작업 흐름을 설명하기 위한 축약 프로젝트다.

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

이 내부 자원을 클라우드에 모두 복제하는 것을 목표로 하지 않는다.

클라우드에서 재현 가능한 범위와 내부 환경에서만 검증 가능한 범위를 분리한다.

---

## 2. 실행 주체를 세 가지로 나눈다

이 프로젝트에서는 역할을 다음처럼 나눈다.

| 실행 주체 | 기본 역할 |
| --- | --- |
| 로컬 에이전트 / 개발자 | 요구사항, 시스템 구조, 작업 분해, 내부망, 최종 통합 |
| 클라우드 실행기 | 빌드, 테스트, E2E, Docker, 마이그레이션 검증 |
| 클라우드 에이전트 | 재현 가능한 실패 분석, 작은 오류 수정, 제한된 리팩터링 |

핵심은 클라우드 에이전트가 모든 개발 작업의 기본 실행 주체가 아니라는 점이다.

```text
판단과 상호작용이 많은 작업
→ Local

결정론적 실행
→ Cloud Runner

독립적인 판단 + 코드 수정
→ Cloud Agent
```

5~6장의 실행 위치 결정과 작업 목록을 프로젝트 수준에서 적용한 결과다.

---

## 3. Cloud Environment는 Task Type과 연결한다

9장에서 만든 미리 준비한 실행환경을 프로젝트 운영 단위로 연결한다.

예:

```text
backend-test
→ Java 21 / Gradle / Docker / Testcontainers

frontend-e2e
→ Node / Playwright / Browser

migration-test
→ Java / Migration Tool / Disposable DB
```

운영 시 중요한 것은 설치 절차를 매 작업마다 설명하는 것이 아니다.

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

필요한 도구가 반복해서 빠진다면 프롬프트가 아니라 실행환경을 수정한다.

---

## 4. 반복 작업은 Task Catalog로 고정한다

팀이 매번 새로운 프롬프트부터 만들지 않도록 반복 작업의 실행 방식을 정리한다.

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

작업 목록의 목적은 프롬프트 양식을 늘리는 것이 아니다.

다음 세 가지를 매번 다시 결정하지 않게 하는 것이다.

```text
누가 실행하는가?
어떤 Environment를 쓰는가?
무엇으로 검증하는가?
```

---

## 5. Task 상태는 Git 기준점과 연결한다

클라우드 작업이 여러 개 실행될수록 대화창보다 Git 상태가 중요해진다.

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

에이전트의 대화 기록을 보지 않아도 어느 코드가 어느 검증을 통과했는지 알 수 있어야 한다.

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

다른 SHA의 결과를 하나의 검증 근거 묶음으로 사용하지 않는다.

```text
Unit        → def456
Integration → def456
Docker      → def456
E2E         → def789
```

위 상태라면 E2E를 다시 맞춰야 한다.

---

## 7. Result Gateway는 공통 반환 인터페이스가 된다

실행기마다 로그 형식이 달라도 개발자와 에이전트가 처음 읽는 결과는 작게 맞출 수 있다.

작업 결과물 예:

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

실패가 있을 때만 8장의 필요할 때 결과의 세부 내용을 단계적으로 더 확인하는 방식을 사용한다.

```text
Summary
→ Failure Detail
→ Specific Artifact
```

운영 모델에서는 `대형 로그 저장`과 `Agent에게 전달할 결과`를 분리한다.

---

## 8. Failure는 코드 Agent와 Environment 경로로 나눈다

실행기가 FAIL했다고 모두 클라우드 에이전트 작업이 되지는 않는다.

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

10장의 Agent-on-failure를 실제 운영 흐름에 배치한 것이다.

---

## 9. Internal Validation을 별도 Stage로 둔다

클라우드에서 끝낼 수 없는 검증은 작업 흐름 밖의 예외가 아니다.

명시적인 단계로 둔다.

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

마이그레이션:

```text
Cloud
→ 일반 Migration Validation

Internal
→ Tibero 최종 검증
```

HSM 관련 작업:

```text
Cloud
→ Pure Logic / Mock / Contract Test

Internal
→ Real HSM Integration
```

이렇게 하면 내부망 제약이 클라우드 작업 흐름 전체를 막지 않는다.

---

## 10. PR과 Event는 같은 운영 모델에 연결된다

수동 작업과 이벤트로 시작하는 작업을 별도 시스템으로 만들 필요는 없다.

사람이 만든 작업:

```text
Developer
→ Task Contract
→ Queue / Worker
```

이벤트가 만든 작업:

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

14장의 이벤트로 시작하는 구조는 이 운영 모델의 **작업 입력 채널 하나**로 들어온다.

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

클라우드 에이전트 작업 흐름이 좋아졌는지는 에이전트 세션 수로 판단하지 않는다.

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

에이전트 호출 수가 줄더라도 검토를 기다리는 작업이 쌓이면, 작업을 시작해서 마칠 때까지 걸리는 전체 시간은 줄지 않을 수 있다.

---

## 12. 처음부터 운영 플랫폼을 만들 필요는 없다

초기에는 다음 정보만으로도 작업 흐름을 운영할 수 있다.

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

이를 Git, CI 메타데이터, PR, 결과물 저장소로 관리할 수 있다.

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

> 작업 명세로 작업을 넘기고, 실행기가 가능한 일을 먼저 실행하며, 판단이 필요한 실패에만 에이전트를 사용하고, 검증 근거로 결과를 다시 로컬에 돌려준다.

다음 장에서는 이 정적인 운영 모델을 `학생 출결 API 인증 변경` 하나에 적용해 요구사항부터 병합까지 시간 순서대로 따라간다.
