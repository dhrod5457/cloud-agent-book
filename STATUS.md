# Current Phase

Phase 4 - 예제 프로젝트 설계 완료

# Completed

- Phase 1 방향 정의
- Phase 2 전체 목차 설계
- Phase 3 목차 검증
- Phase 4 `campus-platform` 예제 프로젝트 설계
- 예제 도메인 범위 확정
- 모듈 책임 경계 확정
- 기본 기술 스택 결정
- 외부 의존성 처리 원칙 결정
- 테스트 전략 정의
- Agent Ready 단계별 진화 계획 정의
- 일반 원칙과 Java/Spring Boot 구현 예의 경계 정의

# Phase 4 Artifacts

- `examples/campus-platform/README.md`
- `examples/campus-platform/architecture.md`
- `examples/campus-platform/testing.md`
- `examples/campus-platform/evolution.md`

# Decisions

## 예제 도메인

대학의 학생, 출결, 알림, 외부 연동을 최소 범위로 사용한다.

핵심 유스케이스:

- 학생 조회
- 수강 정보 조회
- 출결 기록 및 조회
- 알림 발행
- 외부 학사 시스템 동기화

## 모듈

최종 목표 모듈은 다음과 같다.

- `auth`
- `student`
- `attendance`
- `notification`
- `integration`
- 최소 범위의 `common`

예제는 처음부터 완성형 멀티모듈로 시작하지 않는다. 일반적인 단일 Spring Boot 프로젝트에서 시작하여 책의 진행에 따라 명시적인 모듈 구조로 진화한다.

## 기술 스택

기본 예제:

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

정확한 제품 버전은 해당 장을 집필할 때 지원 상태와 공식 문서를 확인하여 고정한다.

## 데이터베이스

기본 로컬/CI DB는 PostgreSQL을 사용한다.

Tibero/Oracle은 기업 환경 장의 상용 DB 사례로 다루며 Cloud Agent가 직접 접근할 수 없는 상황과 Local Agent 기반 실제 통합 검증을 설명하는 데 사용한다.

## Persistence

기본 구현 예는 MyBatis를 사용한다.

설계 원칙은 Persistence Framework에 종속시키지 않으며 필요한 곳에서 JPA 적용 시 차이를 설명한다.

## 외부 의존성

- PostgreSQL: Testcontainers
- Redis: Testcontainers 또는 테스트 목적에 따른 Fake
- Kafka: Testcontainers 또는 이벤트 Port Fake
- 외부 HTTP API: Mock Server / Fake Adapter
- HSM: 기본 환경에서는 Fake Adapter, 실제 장비 검증은 Local/Enterprise 환경
- Jenkins: 기업 내부망 적용 사례

## 테스트 전략

- Unit Test
- Integration Test
- Contract Test
- Architecture Test
- 최소 범위 E2E Test
- Fast Verification
- Full Verification

Agent 작업 완료 판단은 Full Verification 결과와 Task Contract의 Acceptance Criteria를 연결한다.

## 프로젝트 진화

예제는 다음 흐름으로 발전한다.

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

없음. Phase 4 완료 상태이며 다음 Phase 시작 전 사용자 검토를 기다린다.

# Next

Phase 5 - 장별 설계

각 장을 작성하기 전에 `chapters/<장번호>/plan.md`를 작성한다.

장 설계 문서에는 다음을 포함한다.

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

Phase 5는 1장부터 순차적으로 진행한다. 장 설계가 승인되기 전에는 해당 장의 본문 초고를 작성하지 않는다.

# Open Questions

Phase 5 이후 실제 구현과 집필 과정에서 검증한다.

- MyBatis 예제가 특정 독자층에 지나치게 종속되지 않는지
- Redis/Kafka가 모든 장에 불필요한 복잡도를 만들지 않는지
- Stage별 Git tag가 독자의 실습 흐름에 가장 적절한지
- 기업 환경 장에서 Tibero/HSM/Jenkins 사례의 깊이를 어느 수준까지 둘지
