# Current Phase

Phase 2 - 전체 목차 설계 완료, 사용자 검토 대기

# Completed

- Phase 1 방향 정의
- 책이 해결할 핵심 문제 정의
- 핵심 주장 정의
- 주요 독자 정의
- 선수지식 정의
- 독자의 최종 역량 정의
- 포함 범위와 제외 범위 정의
- `planning/toc.md` 초안 작성
- 전체 7개 Part, 24개 장 구성
- 각 장의 목적 정의
- 각 장에서 독자가 얻는 것 정의
- 각 장의 핵심 개념 정의
- 선행 장 정의
- `campus-platform` 기반 실전 예제 정의
- 장 의존성 초안 작성

# In Progress

- Phase 2 목차 사용자 검토

# Next

사용자 승인 후 Phase 3 목차 검증을 진행한다.

Phase 3 검증 항목:

- 장 수가 과도하지 않은지
- 장 간 중복
- 빠진 핵심 개념
- 장 순서와 난이도 흐름
- 특정 AI 제품 편향
- Java/Spring Boot 편향
- 실전성과 이론의 균형
- Security, Audit, Human Escalation 범위

# Decisions

- 책의 상위 개념은 `Agent Ready Software Engineering`으로 정의한다.
- `Cloud-Agent Ready`는 하위 개념으로 다룬다.
- Java/Spring Boot는 주요 실전 예제이지만 책의 원칙은 언어와 제품에 종속되지 않는다.
- 특정 제품의 사용법보다 프로젝트 구조와 개발 프로세스 설계를 중심으로 다룬다.
- 프로젝트 파일을 Source of Truth로 사용한다.
- 하나의 `campus-platform` 예제를 책 전체의 기본 축으로 사용한다.
- 필요한 경우 작은 보조 예제를 사용할 수 있지만 별도의 대형 예제 시스템은 추가하지 않는다.
- `Task Contract`는 독립 장으로 둔다.
- `Trust Boundary`는 독립 장으로 둔다.
- `Agent Observability`와 `Agent Evaluation`은 하나의 장으로 묶는다.
- PM Agent는 Repository, Verification, Parallel Development, Memory를 설명한 이후 배치한다.
- 기업 내부망과 Hybrid Agent는 일반 원칙 이후 별도 Part에서 다룬다.
- 제품별 기능 설명은 사례 또는 부록으로 분리한다.

# Open Questions

Phase 3에서 다음 사항을 검증한다.

- 현재 24개 장을 유지할지 일부를 통합할지
- 4장 `Agent-friendly Repository`와 12장 `병렬 Agent 개발을 위한 모듈 경계`를 분리할지
- 10장 `자동 검증`과 17장 `Observability/Evaluation`의 경계가 충분히 명확한지
- 15장 `Project Memory`와 16장 `Long Running Agent`를 분리할지
- 18장 `Agent Lifecycle`과 20장 `PM Agent`의 책임 경계가 적절한지
- 22장 `내부망 프로젝트`와 23장 `CI/CD와 Agent 권한`을 분리할지
