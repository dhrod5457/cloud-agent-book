# Current Phase

Phase 5 - 장별 설계 진행 중

# Completed

- Phase 1 방향 정의
- Phase 2 전체 목차 설계
- Phase 3 목차 검증
- Phase 4 `campus-platform` 예제 프로젝트 설계
- Phase 5 1장 설계
- Phase 5 2장 설계
- Cloud Agent 실행 자원과 LLM 사용량 분리 관점 추가
- Agent Worker / Test Runner 역할 분리 설계
- Result Filter 설계 추가
- Cloud Session 병렬 Test Runner 패턴 추가

# Phase 4 Artifacts

- `examples/campus-platform/README.md`
- `examples/campus-platform/architecture.md`
- `examples/campus-platform/testing.md`
- `examples/campus-platform/evolution.md`
- `examples/campus-platform/cloud-test-runner.md`

# Phase 5 Artifacts

- `chapters/01/plan.md`
- `chapters/02/plan.md`
- `research/anthropic/claude-code-web-execution-resources.md`

# Decisions

## 책의 기본 방향

- 책의 상위 개념은 `Agent Ready Software Engineering`으로 정의한다.
- `Cloud-Agent Ready`는 하위 개념으로 다룬다.
- Java/Spring Boot는 주요 실전 예제이지만 책의 원칙은 언어와 제품에 종속되지 않는다.
- 특정 제품의 사용법보다 프로젝트 구조와 개발 프로세스 설계를 중심으로 다룬다.
- 프로젝트 파일을 Source of Truth로 사용한다.

## 예제 프로젝트

대학의 학생, 출결, 알림, 외부 연동을 최소 범위로 사용한다.

최종 목표 모듈:

- `auth`
- `student`
- `attendance`
- `notification`
- `integration`
- 최소 범위의 `common`

예제는 일반적인 단일 Spring Boot 프로젝트에서 시작하여 책의 진행에 따라 명시적인 모듈 구조로 진화한다.

기본 기술 스택:

- Java 21+
- Spring Boot 3.x
- Gradle Groovy DSL
- MyBatis
- PostgreSQL
- Redis
- Kafka
- Testcontainers
- Docker
- JUnit 5
- ArchUnit
- Mock HTTP Server
- GitHub Actions

정확한 제품 버전은 해당 장을 집필할 때 공식 문서를 확인하여 고정한다.

## 테스트와 외부 의존성

- PostgreSQL: Testcontainers
- Redis: Testcontainers 또는 테스트 목적에 따른 Fake
- Kafka: Testcontainers 또는 이벤트 Port Fake
- 외부 HTTP API: Mock Server / Fake Adapter
- HSM: 기본 환경에서는 Fake Adapter, 실제 장비 검증은 Local/Enterprise 환경
- Jenkins: 기업 내부망 적용 사례
- Agent 작업 완료 판단은 Full Verification과 Task Contract Acceptance Criteria를 연결한다.

## Cloud Agent 실행 자원 활용 원칙

2장 `Local Agent, Cloud Agent, Hybrid Agent`에 Cloud Agent를 독립 실행 노드로 활용하는 절을 둔다.

핵심 원칙:

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

Cloud compute resource와 LLM token/plan usage를 구분한다.

Build, Test, Docker Build, Static Analysis, Migration Validation 등의 결정론적 작업은 가능한 경우 Test Runner가 먼저 실행한다.

```text
Task
  ↓
Test Runner
  ├─ PASS → 종료
  └─ FAIL
       ↓
   Result Filter
       ↓
   Agent Worker
       ↓
      수정
       ↓
   Test Runner 재검증
```

Cloud Worker는 다음 두 역할로 구분한다.

### Agent Worker

- 코드 분석
- 설계 판단
- 구현
- 복잡한 오류 분석
- Review

### Test Runner

- Build
- Test
- Docker Build
- Lint
- Static Analysis
- Migration Validation
- E2E Test

Result Filter는 원본 로그 전체 대신 다음 정보를 LLM에 전달한다.

- exit code
- 성공/실패 건수
- 실패 테스트명
- root cause
- 핵심 stack trace
- 중요 warning
- 전체 로그 위치

원본 로그는 artifact로 보존하고 필요할 때 특정 부분만 추가 조회한다.

## Claude Code Web 사례 처리 원칙

Claude Code Web은 일반 원칙을 설명하기 위한 제품 사례로만 사용한다.

2026-09-16 기준 공식 문서에서 Cloud Session의 대략적인 resource limit은 다음과 같이 확인했다.

- 4 vCPU
- 16 GB RAM
- 30 GB disk

이 값은 변경 가능한 대략적 상한이므로 본문의 영구적인 전제로 사용하지 않는다.

공식 문서가 각 작업의 격리된 실행 환경과 병렬 작업을 설명하더라도, 각 세션의 CPU/RAM이 물리적으로 전용 보장된다고 표현하지 않는다.

여러 세션의 LLM 추론은 계정/조직 사용량 제한을 소비할 수 있으므로 병렬 compute와 병렬 reasoning을 구분한다.

관련 조사:

- `research/anthropic/claude-code-web-execution-resources.md`

## 프로젝트 진화

```text
일반 Spring Boot 프로젝트
→ 재현 가능한 환경
→ Agent Contract / Task Contract
→ 표준 실행 인터페이스
→ 테스트 격리 / 자동 검증
→ 모듈 경계 / Architecture Rule
→ 병렬 Agent 개발
→ Trust Boundary / CI Gate
→ Project Memory / Observability
→ Planner / Worker / Reviewer
→ PM Agent
→ Hybrid Enterprise
→ Agent Ready 성숙도 평가
```

## Snapshot 전략

실제 코드 구현 단계에서는 하나의 예제 Repository를 유지하고 Stage별 Git tag를 사용하는 방식을 우선한다.

코드를 복제한 여러 디렉터리를 유지하는 방식은 피한다.

# In Progress

Phase 5 장별 설계.

현재 1장과 2장의 `plan.md`가 작성된 상태다.

2장에는 다음 신규 주제를 반영했다.

- Compute Resource와 LLM Usage 분리
- Cloud Agent as Test Runner
- Agent Worker와 Test Runner 구분
- Result Filter
- 병렬 Cloud Session 기반 검증
- Context 최소화
- Claude Code Web의 현재 실행 환경 사례와 제한사항

# Next

Phase 5를 계속 진행한다.

다음 대상:

`chapters/03/plan.md` - Agent Ready 프로젝트의 기준

이후 각 장에 대해 다음을 순차적으로 설계한다.

- 장의 목표
- 문제 정의
- 핵심 주장
- 독자가 얻는 것
- 예제에서 사용할 Stage
- 필요한 구조/그림
- 필요한 코드 예제
- 필요한 공식 자료 조사
- 앞 장과 뒤 장의 연결
- 본문에서 의도적으로 다루지 않을 내용

장 설계가 확정되기 전에는 Phase 6 본문 초고를 시작하지 않는다.

# Open Questions

Phase 5 이후 실제 구현과 집필 과정에서 검증한다.

- MyBatis 예제가 특정 독자층에 지나치게 종속되지 않는지
- Redis/Kafka가 모든 장에 불필요한 복잡도를 만들지 않는지
- Stage별 Git tag가 독자의 실습 흐름에 가장 적절한지
- 기업 환경 장에서 Tibero/HSM/Jenkins 사례의 깊이를 어느 수준까지 둘지
- Result Filter를 shell 기반 helper로 시작할지 별도 도구로 만들지
- Test Runner의 결과 포맷을 JSON Schema로 고정할지
