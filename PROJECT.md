# 프로젝트 목적

Cloud Agent를 실제 개발팀에서 어떻게 활용할지 정리하는 실전 Software Engineering 책을 작성한다.

이 책은 특정 AI 제품 사용 설명서가 아니다.

핵심 질문은 다음과 같다.

> 클라우드 에이전트를 왜 사용하고, 로컬 에이전트와 어떻게 조합하며, 어떤 작업을 맡기고, 토큰과 클라우드 컴퓨팅 자원을 어떻게 효율적으로 활용할 것인가?

독자는 책을 읽은 뒤 자신의 프로젝트에서 다음을 판단하고 구성할 수 있어야 한다.

- 이 Task는 Local에서 할지 Cloud로 보낼지
- Build/Test/E2E를 Cloud에 어떻게 분산할지
- 여러 Cloud Worker를 어떻게 병렬화할지
- Cloud Agent Context와 Tool Output을 어떻게 줄일지
- Prebuilt Environment와 Cache를 어떻게 활용할지
- Local + Cloud Hybrid Workflow를 어떻게 구성할지

## 작업명

최종 제목은 Phase 5 목차가 안정된 뒤 확정한다.

기존 가제:

**AI Agent Ready Software Engineering**

현재 책의 중심 주제:

**Cloud Agent 활용과 Local + Cloud 개발 Workflow**

## 핵심 원칙

1. 전체 책을 한 번에 작성하지 않는다.
2. Phase 단위로 진행한다.
3. 각 Phase 결과는 프로젝트 파일에 기록한다.
4. 프로젝트 파일을 Source of Truth로 사용한다.
5. 제품 기능과 일반적인 활용 원칙을 구분한다.
6. Java/Spring Boot를 주요 실전 예제로 사용한다.
7. 본문보다 먼저 개념, 범위, 목차, 예제 프로젝트, 장별 설계를 확정한다.
8. 제품 및 기술의 현재 기능은 공식 문서를 통해 검증한다.
9. 새로운 주제는 `Cloud Agent를 더 잘 사용하는 방법과 직접 관련이 있는가?`를 기준으로 본문 포함 여부를 판단한다.
10. Agent Platform 일반론은 현재 책의 핵심 범위로 확장하지 않는다.

## 책 전체에서 유지할 메시지

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

> Cloud Agent에게 Repository 전체를 반복해서 이해시키지 않는다.

> 작은 Task와 작은 Context를 전달한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

## 전체 Phase

1. 책의 방향 정의
2. 전체 목차 설계
3. 목차 검증
4. 예제 프로젝트 설계
5. 장별 설계
6. 초고 작성
7. 기술 검토
8. 실전 검증
9. 전체 일관성 검토
10. 최종 편집

## 집필 방식

각 장은 가능하면 다음 흐름을 따른다.

- 문제
- 판단 기준
- 구조
- 좋은 사례 / 나쁜 사례
- 실전 적용
- 비용/Token/Compute 관점
- 체크리스트

## 주요 실전 예제

Java/Spring Boot 기반 `campus-platform`을 사용한다.

핵심 Workflow 예:

```text
Local
→ 기능 설계 / Task 분리

Cloud #1
→ Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Docker Build

Cloud #4
→ Web E2E

실패
→ 필요한 경우 Cloud Agent 분석

성공
→ PR / Local 통합 / 최종 Review
```

예제 기술 후보:

- Java 21+
- Spring Boot
- Gradle
- MyBatis
- PostgreSQL / Testcontainers
- Docker
- Playwright 또는 동등한 E2E 도구
- GitHub Actions 또는 Jenkins 사례

Redis, Kafka, HSM, Tibero/Oracle 등은 Cloud/Local 경계를 설명하는 데 필요한 범위에서만 사용한다.
