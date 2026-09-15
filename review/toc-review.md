# Phase 3 목차 검증

## 검증 목적

Phase 2에서 작성한 7개 Part, 24개 장의 목차를 다음 기준으로 검토한다.

- 중복
- 빠진 개념
- 장 순서
- 난이도 흐름
- 특정 AI 제품 편향
- Java/Spring Boot 편향
- 실전성과 이론의 균형

## 결론

목차의 큰 흐름은 유지한다. 다만 24개 장 중 내용 경계가 가까운 장을 통합하여 22개 장으로 압축한다.

최종 흐름은 다음과 같다.

```text
개발 방식 변화
→ Agent Ready 기준
→ Repository와 Contract
→ 재현 가능한 실행환경
→ 테스트와 자동 검증
→ 병렬 개발
→ Trust Boundary와 CI/CD Gate
→ Project Memory와 장기 작업
→ Observability와 Evaluation
→ Agent Lifecycle과 역할 분리
→ PM Agent
→ 실패 처리와 통합
→ 기업 Hybrid 환경
→ 성숙도와 Governance
```

## 1. 장 수 검토

### 판단

24개 장은 절대적으로 과도한 수는 아니지만, 현재 구성에서는 독립 장으로 유지할 필요가 약한 구간이 있다.

### 통합

- 기존 15장 `Agent Memory와 Project Memory`
- 기존 16장 `Long Running Agent와 중단 가능한 작업`

두 장을 `Project Memory와 Long Running Task`로 통합한다.

Project Memory는 장기 작업을 이어가기 위한 기반이고, Long Running Task는 그 구조의 대표적인 적용 사례이므로 한 장 안에서 개념과 적용을 이어가는 편이 자연스럽다.

또한 다음 두 장을 통합한다.

- 기존 18장 `Agent Lifecycle: 생성에서 종료까지`
- 기존 19장 `Planner, Worker, Tester, Reviewer`

이를 `Agent Lifecycle과 역할 분리`로 통합한다.

Agent 생성 수, 역할 선택, 실행, 검증, 종료는 하나의 작업 단위에서 함께 결정되므로 별도 장으로 나누면 같은 실행 흐름을 반복 설명할 가능성이 높다.

결과적으로 24장에서 22장으로 줄인다.

## 2. Repository 설계와 병렬 모듈 경계의 중복

### 기존 문제

기존 4장과 12장 모두 모듈 경계, 공통 모듈, migration, shared file을 다룬다.

### 조정

두 장은 유지하되 책임을 명확히 분리한다.

4장은 `Repository as Interface` 관점에 집중한다.

- 탐색 가능성
- 디렉터리 구조
- 명명
- canonical source
- 문서 위치
- generated file 식별
- Agent의 context 탐색 비용

12장은 `Parallelizability` 관점에 집중한다.

- change locality
- bounded context
- module ownership
- shared state
- common module
- migration conflict
- 여러 Agent의 동시 변경 충돌

따라서 4장은 '이해하기 쉬운 저장소', 12장은 '동시에 변경하기 쉬운 저장소'를 다룬다.

## 3. 자동 검증과 Evaluation의 경계

### 판단

기존 10장과 17장은 모두 검증이라는 표현을 사용하지만 다른 문제를 다룬다.

10장은 기계적 완료 조건을 다룬다.

```text
compile
unit test
integration test
architecture test
secret scan
exit code
```

17장은 실행 중인 Agent와 결과물의 품질을 평가한다.

```text
queued/running/blocked
requirement coverage
change scope
unnecessary changes
retry count
execution time
cost
```

따라서 두 장은 유지한다. 17장의 이름과 설명에서 '검증'보다 `Observability`와 `Quality Evaluation`을 강조한다.

## 4. CI/CD 장의 위치

### 기존 문제

CI/CD와 Agent 권한이 기업 내부망 Part에 배치되어 있어, 일반적인 Agent 권한 모델이 특정 기업 인프라 문제처럼 보일 수 있다.

### 조정

기존 23장 `CI/CD와 Agent 권한 연결`을 Trust Boundary 바로 다음으로 이동한다.

흐름은 다음과 같이 변경한다.

```text
병렬 Git 작업
→ Trust Boundary
→ CI/CD Gate
→ Project Memory
```

CI/CD Gate는 기업 내부망 여부와 관계없이 모든 Agent Ready 프로젝트에서 필요한 일반 원칙으로 취급한다.

## 5. Local / Cloud / Hybrid의 반복

2장은 실행 모델을 정의하는 개념 장으로 유지한다.

기업 환경 장에서는 개념을 다시 설명하지 않고 다음 현실적인 제약에만 집중한다.

- VPN
- 사내 Git/Nexus
- 내부 DB
- Redis/Kafka
- HSM
- Jenkins
- 내부 API
- Cloud Agent가 접근할 수 없는 자원
- Local Agent와 Cloud Agent 사이의 handoff

## 6. 빠진 개념

### Context 관리

Agent가 Repository를 이해하는 과정에는 context 비용이 발생한다. 이를 별도 장으로 늘리지 않고 4장과 5장에 포함한다.

추가 항목:

- Progressive Disclosure
- Canonical Source
- Instruction Precedence
- Context Budget
- 중복 문서와 충돌하는 규칙 제거

### Untrusted Input과 Prompt Injection

기존 Trust Boundary는 Secret과 권한 중심이었다. Agent는 저장소 파일, Issue, 외부 문서, 웹 콘텐츠 등 비신뢰 입력을 명령처럼 해석할 수 있으므로 다음을 14장에 추가한다.

- Untrusted Input
- Prompt Injection
- Secret Exfiltration
- Tool Allowlist
- Command Execution Boundary
- Dependency/Script 실행 위험

### 비용과 자원 예산

Agent 시스템은 정확성뿐 아니라 실행 시간, 재시도 횟수, 모델 비용, 병렬 Agent 수를 관리해야 한다.

이를 17장 Observability와 22장 Governance에 포함한다.

### 제품 독립성

AGENTS.md, CLAUDE.md 등 제품별 파일이 Source of Truth가 되지 않도록 5장에서 canonical project contract와 product adapter 문서를 구분한다.

## 7. Java/Spring Boot 편향 검토

Java/Spring Boot는 계속 단일 실전 예제의 구현체로 사용한다.

다만 각 장은 다음 순서를 따른다.

1. 제품·언어 독립적인 문제
2. 일반 설계 원칙
3. 추상 구조
4. Java/Spring Boot 적용 예

따라서 Java, Gradle, Spring Boot, Testcontainers, ArchUnit 등의 구현 기술이 원칙보다 먼저 등장하지 않도록 한다.

## 8. 제품 편향 검토

Claude Code, Codex, GitHub Copilot Coding Agent, Devin은 본문 구조를 결정하는 기준으로 사용하지 않는다.

제품명은 다음 위치에서만 제한적으로 사용한다.

- 현재 구현 사례
- 비교 표
- Research 문서
- 부록

제품 기능이 변해도 본문의 핵심 논리가 유지되어야 한다.

## 9. 난이도 흐름 검토

현재 흐름은 적절하다.

초반에는 한 Agent가 하나의 Repository에서 작업할 수 있게 만드는 데 집중한다.

중반부터 여러 Agent의 병렬 작업, 권한, 상태 관리로 확장한다.

후반에서 orchestration, PM Agent, 기업 Hybrid 환경으로 확장한다.

PM Agent를 초반에 배치하지 않는다. PM Agent를 설명하기 전에 Task Contract, 검증, 병렬 작업, 상태 관리가 먼저 정의되어야 하기 때문이다.

## 10. Phase 3 결정 사항

- 24개 장을 22개 장으로 줄인다.
- Repository 탐색성과 병렬 모듈 경계를 별도 문제로 유지한다.
- Memory와 Long Running Task를 한 장으로 통합한다.
- Lifecycle과 역할 분리를 한 장으로 통합한다.
- CI/CD Gate를 Trust Boundary 다음으로 이동한다.
- Context 관리 원칙을 Repository/Agent Contract에 추가한다.
- Trust Boundary에 Prompt Injection과 Untrusted Input을 추가한다.
- Observability에 실행 시간, 재시도, 비용과 자원 사용 관점을 추가한다.
- Java/Spring Boot는 원칙을 설명한 뒤 적용하는 예제로만 사용한다.
- 특정 AI 제품은 목차의 구조를 결정하지 않는다.

## Phase 3 완료 기준

다음 조건을 충족하면 Phase 3를 완료한 것으로 본다.

- 중복 장의 통합 여부 결정
- 각 장의 책임 경계 명확화
- 빠진 핵심 주제 보완
- 장 순서와 의존성 검증
- 제품 및 언어 편향 검토
- 수정된 `planning/toc.md` 반영
- 다음 Phase가 예제 프로젝트 설계임을 `STATUS.md`에 기록
