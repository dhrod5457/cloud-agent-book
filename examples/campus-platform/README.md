# campus-platform

`campus-platform`은 책 전체에서 사용하는 Java/Spring Boot 예제 시스템이다.

목적은 대학 시스템 자체를 완성하는 것이 아니라, 일반적인 업무 시스템이 책의 진행에 따라 어떻게 `Agent Ready` 프로젝트로 변화하는지 보여주는 데 있다.

## 예제 시스템이 선택한 도메인

대학의 학생·출결·알림·외부 연동을 최소 범위로 사용한다.

핵심 유스케이스는 다음과 같다.

- 학생 조회
- 수강 정보 조회
- 출결 기록
- 출결 상태 조회
- 알림 발행
- 외부 학사 시스템에서 학생 정보 동기화

인증, 학생, 출결, 알림, 외부 연동이라는 서로 다른 변경 축이 존재하므로 모듈 경계, 외부 의존성, 병렬 Agent 작업, 이벤트 처리, 테스트 격리를 설명하기에 적합하다.

## 의도적으로 구현하지 않는 범위

- 실제 대학 ERP 전체 기능
- 실제 금융·학생증 발급
- 실제 HSM 명령 구현
- 실제 모바일 앱
- LMS 전체 기능
- 복잡한 조직·권한 모델
- 운영 수준의 대용량 데이터 모델

필요한 경우 인터페이스와 Fake Adapter만 제공한다.

## 기본 기술 선택

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
- WireMock 또는 동등한 Mock HTTP Server
- GitHub Actions를 기본 CI 예제로 사용
- Jenkins는 기업 환경 적용 사례에서 추가

버전 번호는 집필 시점의 지원 상태를 확인한 후 장별로 고정한다.

## 데이터베이스 선택 원칙

책의 기본 실행환경은 개발자가 별도 상용 DB를 준비하지 않아도 재현 가능해야 한다.

따라서 기본 예제 DB는 PostgreSQL을 사용한다.

Tibero/Oracle과 같은 상용 DB는 기업 환경 장에서 다음 문제를 설명하는 사례로 사용한다.

- Cloud Agent가 라이선스 또는 내부망 DB에 접근하지 못하는 경우
- DB별 SQL 차이
- Local Agent를 통한 실제 통합 검증
- CI에서 상용 DB를 사용할 수 없는 경우의 테스트 전략

## Persistence 기술 선택

기본 예제는 MyBatis를 사용한다.

이 선택은 SQL 중심 업무 시스템에서 Repository/Mapper 경계, 외부 DB 차이, 테스트 대역을 명시적으로 보여주기 쉽기 때문이다.

다만 책의 설계 원칙은 MyBatis에 종속되지 않는다. 필요한 장에서는 JPA를 사용하는 경우의 차이를 별도 주석이나 보조 예제로 설명한다.

## 프로젝트 진화 원칙

예제는 완성된 Agent Ready 프로젝트로 시작하지 않는다.

책의 흐름에 맞춰 다음과 같이 변화한다.

```text
Stage 0  일반 Spring Boot 프로젝트
Stage 1  실행환경 재현
Stage 2  Agent Contract / Task Contract
Stage 3  표준 실행 인터페이스
Stage 4  테스트 격리와 자동 검증
Stage 5  모듈 경계와 Architecture Rule
Stage 6  병렬 Agent 개발
Stage 7  Trust Boundary / CI Gate
Stage 8  Project Memory / Observability
Stage 9  Planner / Worker / Reviewer
Stage 10 PM Agent / Hybrid Enterprise
```

각 단계에서는 이전 단계의 문제를 먼저 보여준 뒤 필요한 구조를 추가한다.

## 예제 사용 규칙

- 예제 코드는 책의 주장을 검증하기 위한 최소 범위로 유지한다.
- 특정 기술의 사용법 자체가 본문을 지배하지 않게 한다.
- Local, CI, Cloud Agent에서 동일한 검증 명령을 사용할 수 있어야 한다.
- 외부 시스템이 없어도 핵심 기능 대부분을 검증할 수 있어야 한다.
- 실제 운영 시스템 접근은 기본 전제로 두지 않는다.
- 각 단계의 변경 이유가 설명되지 않는 구조 변경은 하지 않는다.
