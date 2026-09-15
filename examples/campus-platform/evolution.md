# campus-platform Evolution Plan

## 목적

예제 프로젝트는 최종 형태를 처음부터 보여주지 않는다.

독자가 기존 프로젝트의 문제를 먼저 경험하고, 각 장의 설계 원칙을 적용하면서 Agent Ready 프로젝트로 변하는 과정을 따라가게 한다.

각 Stage는 책의 장과 연결되며, 필요한 경우 Git tag 또는 별도 snapshot으로 재현할 수 있게 한다.

## Stage 0. 일반적인 Spring Boot 프로젝트

### 상태

- 단일 Spring Boot 프로젝트
- 수동 환경 설정
- 로컬 DB 연결 정보 필요
- 실행 방법이 README 일부 또는 개발자 지식에 의존
- 외부 API와 DB 호출이 코드에 강하게 결합
- 테스트 일부만 존재
- CI와 로컬 명령이 다를 수 있음

### 보여줄 문제

- Agent가 프로젝트 실행 방법을 추론함
- 운영/개발 인프라 없이는 테스트가 중단됨
- 변경 범위가 불명확함
- 완료 조건이 사람 판단에 의존함

### 관련 장

1~3장

---

## Stage 1. 재현 가능한 실행환경

### 추가

- Java/Gradle 버전 고정
- 환경변수 목록 정의
- Docker 기반 로컬 의존성
- 자동 bootstrap 절차

### 목표

새로운 개발자, CI, Agent가 같은 방식으로 프로젝트를 실행할 수 있게 한다.

### 관련 장

4장, 7장

---

## Stage 2. Agent Contract와 Task Contract

### 추가

- README 정리
- AGENTS.md
- architecture 문서
- development/testing 문서
- 금지 규칙
- Task Contract 템플릿

### 목표

Agent가 대화에서 프로젝트 규칙을 다시 설명받지 않아도 Repository에서 필요한 정보를 찾게 한다.

### 관련 장

5장, 6장

---

## Stage 3. 표준 실행 인터페이스

### 추가

```text
setup
build
test
verify
```

역할을 갖는 단일 진입점 계층을 만든다.

실제 명령은 `scripts/`, Gradle task 또는 Makefile 중 구현 시 결정하되, 사람과 Agent가 동일한 명령을 사용한다.

### 목표

Agent가 프로젝트별 빌드·테스트 명령을 추론하지 않게 한다.

### 관련 장

8장

---

## Stage 4. 테스트 격리와 자동 검증

### 추가

- PostgreSQL Testcontainer
- Redis/Kafka Testcontainer 또는 적절한 테스트 대역
- Fake HSM
- Mock External API
- Architecture Test
- Secret/Security Check
- Full Verification

### 목표

운영 인프라 없이 작업 결과의 대부분을 기계적으로 검증할 수 있게 한다.

### 관련 장

9~11장

---

## Stage 5. 병렬 변경 가능한 모듈 구조

### 변경

단일 애플리케이션 구조에서 명시적 도메인 경계로 리팩터링한다.

목표 구조:

```text
modules/
├─ auth/
├─ student/
├─ attendance/
├─ notification/
└─ integration/
```

### 추가

- module ownership
- dependency rules
- migration ownership
- common 최소화
- change locality 검증

### 목표

서로 다른 Agent가 동시에 작업할 때 같은 파일을 수정해야 하는 빈도를 줄인다.

### 관련 장

12장

---

## Stage 6. 병렬 Agent 실행

### 추가

- task branch
- git worktree
- independent clone/cloud sandbox
- Agent별 Task Contract
- Integrator 역할

### 목표

두 개 이상의 Agent가 독립 작업 후 결과를 통합할 수 있게 한다.

### 관련 장

13장

---

## Stage 7. Trust Boundary와 CI Gate

### 추가

- Agent별 Repository 권한
- Secret 전달 정책
- Network 접근 정책
- PR/Merge 권한 분리
- CI Gate
- Deployment Gate
- 승인 지점
- 감사 로그 기준

### 목표

Agent에게 작업 능력을 제공하면서도 운영 시스템에 대한 권한을 최소화한다.

### 관련 장

14~15장

---

## Stage 8. Project Memory와 Observability

### 추가

```text
docs/
├─ decisions/
├─ tasks/
├─ progress/
└─ knowledge/
```

그리고 Agent 실행 상태를 기록한다.

예:

```text
queued
running
blocked
failed
verifying
completed
```

추가 관찰 항목:

- 시작/종료 시각
- 실행 시간
- 재시도 횟수
- 변경 파일
- 검증 결과
- 실패 이유
- 토큰/비용 또는 컴퓨팅 사용량을 측정할 수 있는 경우 해당 값

### 목표

대화 컨텍스트가 사라져도 다음 Agent가 프로젝트와 작업 상태를 이어받을 수 있게 한다.

### 관련 장

16~17장

---

## Stage 9. 역할 분리 Agent

### 추가

작업 복잡도에 따라 다음 역할 중 필요한 것만 사용한다.

- Planner
- Worker
- Tester
- Reviewer
- Security Reviewer
- Integrator

### 목표

하나의 Agent가 계획, 구현, 검증을 모두 수행할 때 발생하는 자기검증 문제를 줄인다.

### 관련 장

18장

---

## Stage 10. PM Agent

### 추가

PM Agent가 다음을 수행한다.

```text
Project State 확인
→ 다음 작업 결정
→ Task 분해
→ Dependency 분석
→ 필요한 Agent 수 결정
→ Agent 배정
→ 상태 수집
→ 검증 결과 확인
→ 실패 재할당
→ 통합 판단
```

### 목표

사람이 모든 Worker에게 세부 작업을 직접 전달하지 않아도 프로젝트 단위 작업 흐름을 운영할 수 있게 한다.

### 관련 장

19~20장

---

## Stage 11. Hybrid Enterprise

### 추가

Cloud Agent와 Local Agent의 역할을 분리한다.

```text
Cloud Agent
- 독립 코드 개발
- Unit/Integration Test
- Refactoring
- Documentation
- Review

Local Agent
- VPN
- 내부 Git/Nexus
- Tibero/Oracle
- 실제 HSM
- Jenkins
- 내부 API
```

PM Agent 또는 사람이 두 실행 환경의 결과를 통합한다.

### 목표

Cloud Agent가 기업 내부망에 직접 접근하지 못해도 Agent 기반 개발 프로세스를 유지할 수 있게 한다.

### 관련 장

21장

---

## Stage 12. Agent Ready 성숙도 평가

마지막에는 프로젝트가 어느 수준까지 Agent Ready인지 평가한다.

평가 축:

- Discoverability
- Reproducibility
- Executability
- Testability
- Verifiability
- Isolation
- Parallelizability
- Security
- Observability
- Recoverability
- Governance

### 목표

`Agent를 사용한다/사용하지 않는다`라는 이분법 대신 프로젝트의 준비 수준을 단계적으로 평가한다.

### 관련 장

22장

---

# 구현 snapshot 전략

본문 집필과 실제 예제 코드 검증 단계에서 다음 방식 중 하나를 사용한다.

우선순위는 다음과 같다.

1. 하나의 예제 Repository + Stage별 Git tag
2. 장별 비교가 필요한 경우 특정 Stage snapshot
3. 별도 브랜치는 병렬 개발 실습 등 Git 자체가 주제일 때만 사용

복제된 디렉터리를 여러 개 두어 코드가 서로 달라지는 방식은 피한다.

예상 tag 예:

```text
stage-00-baseline
stage-01-reproducible
stage-02-agent-contract
stage-03-execution-interface
stage-04-verifiable
stage-05-modular
stage-06-parallel
stage-07-trust-boundary
stage-08-project-memory
stage-09-role-separated
stage-10-pm-agent
stage-11-hybrid
```

실제 tag 이름은 코드 구현이 시작될 때 확정한다.
