# Current Phase

Phase 3 - 목차 검증 완료

# Completed

- Phase 1 방향 정의
- 책이 해결할 핵심 문제 정의
- 핵심 주장 정의
- 주요 독자 정의
- 선수지식 정의
- 독자의 최종 역량 정의
- 포함 범위와 제외 범위 정의
- Phase 2 전체 목차 초안 작성
- Phase 3 목차 검증
- 장 간 중복 검토
- 빠진 핵심 개념 검토
- 장 순서와 난이도 흐름 검토
- 특정 AI 제품 편향 검토
- Java/Spring Boot 편향 검토
- 실전성과 이론의 균형 검토
- Security, Audit, Human Escalation 범위 보완
- `review/toc-review.md` 작성
- `planning/toc.md`를 7개 Part, 22개 장으로 수정

# In Progress

없음. Phase 3 완료 상태이며 다음 Phase 시작 전 사용자 검토를 기다린다.

# Next

Phase 4 - 예제 프로젝트 설계

Phase 4에서는 아직 전체 구현을 시작하지 않고 다음을 정의한다.

- 예제 시스템 도메인
- 모듈 구조
- 기술 스택
- 외부 의존성
- 테스트 전략
- Agent Ready 단계별 진화 과정
- 일반 원칙과 Java/Spring Boot 구현 예의 경계

예정 산출물은 `examples/campus-platform/`의 설계 문서이며, 실제 애플리케이션 구현은 이후 검증에 필요한 범위에서 진행한다.

# Decisions

- 책의 상위 개념은 `Agent Ready Software Engineering`으로 정의한다.
- `Cloud-Agent Ready`는 하위 개념으로 다룬다.
- Java/Spring Boot는 주요 실전 예제이지만 책의 원칙은 언어와 제품에 종속되지 않는다.
- 특정 제품의 사용법보다 프로젝트 구조와 개발 프로세스 설계를 중심으로 다룬다.
- 프로젝트 파일을 Source of Truth로 사용한다.
- 하나의 `campus-platform` 예제를 책 전체의 기본 축으로 사용한다.
- 전체 목차는 7개 Part, 22개 장으로 구성한다.
- 4장은 Repository 탐색성과 context 구조에 집중한다.
- 12장은 병렬 변경과 모듈 충돌 제어에 집중한다.
- `Task Contract`와 `Trust Boundary`는 독립 장으로 유지한다.
- Project Memory와 Long Running Task는 하나의 장으로 통합한다.
- Agent Lifecycle과 Planner/Worker/Tester/Reviewer 역할 분리는 하나의 장으로 통합한다.
- CI/CD Gate는 기업 내부망 문제가 아니라 일반 Trust Boundary의 연장선으로 다룬다.
- Trust Boundary에 Untrusted Input, Prompt Injection, Secret Exfiltration, Tool/Command Boundary를 포함한다.
- Agent Observability와 Quality Evaluation은 하나의 장으로 묶고 실행 시간, 재시도, 비용과 자원 사용량을 포함한다.
- PM Agent는 Repository, Verification, Parallel Development, Memory, Observability를 설명한 이후 배치한다.
- 기업 내부망과 Hybrid Agent는 일반 원칙 이후 적용 사례로 다룬다.
- 제품별 기능 설명은 사례 또는 부록으로 분리한다.

# Open Questions

Phase 4에서 다음 사항을 결정한다.

- `campus-platform`의 구체적인 도메인 범위
- 모듈 수와 각 모듈의 책임 경계
- JPA와 MyBatis 중 예제 기본 선택 또는 병행 방식
- Gradle과 Maven 중 기본 빌드 도구
- Redis, Kafka, DB, HSM, 외부 API를 예제에서 어느 수준까지 실제 구성할지
- Testcontainers와 Fake Adapter의 역할 분담
- 단일 모듈에서 시작해 멀티모듈로 진화시킬지, 처음부터 멀티모듈로 제시할지
