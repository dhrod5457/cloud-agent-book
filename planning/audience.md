# Audience

## 주요 독자

이 책의 1차 독자는 중급 이상 소프트웨어 개발자다.

특히 다음 역할을 주요 독자로 본다.

- Java/Spring Boot 백엔드 개발자
- Tech Lead
- Software Architect
- 개발팀 리더
- CI/CD 및 개발환경 담당자

## 2차 독자

- AI Coding Agent를 개발팀에 도입하려는 조직
- AI 기반 개발 자동화를 설계하는 엔지니어
- AI Agent Platform을 만드는 개발자
- PM Agent 또는 Coding Agent orchestration을 설계하는 개발자

## 주요 독자가 아닌 대상

다음 독자를 주대상으로 삼지 않는다.

- 프로그래밍 입문자
- AI 코딩 도구의 기본 사용법만 원하는 독자
- Prompt Engineering 입문만을 원하는 독자
- 특정 AI 제품의 기능 설명서가 필요한 독자

## 선수지식

독자는 최소한 다음 개념을 알고 있다고 가정한다.

- Git 기본 사용
- Branch, Merge, Pull Request
- 일반적인 서버 애플리케이션 구조
- Build, Test, Deploy
- Docker 기본 개념
- Unit Test와 Integration Test의 차이
- REST API
- CI/CD 기본 개념

Java 실전 예제를 이해하려면 다음 지식이 있으면 좋다.

- Java
- Spring Boot
- Gradle 또는 Maven

반면 다음 개념은 책에서 설명한다.

- AI Coding Agent 구조
- Agent Sandbox
- Agent Context
- Agent Memory와 Project Memory
- Agent orchestration
- PM, Planner, Worker, Reviewer 역할
- Cloud Agent 실행환경
- Agent Contract
- Task Contract
- 자동 검증과 Definition of Done
- Trust Boundary

## 책을 읽은 후 할 수 있어야 하는 것

독자는 일반적인 소프트웨어 프로젝트를 Agent Ready 프로젝트로 전환할 수 있어야 한다.

구체적으로 다음을 설계할 수 있어야 한다.

- Agent가 저장소를 Clone한 뒤 최소한의 명령으로 실행 가능한 환경
- 재현 가능한 개발환경과 자동 Setup
- Agent Contract와 Task Contract
- 외부 시스템 격리 전략
- Testcontainers, Fake, Mock 기반 테스트 환경
- 통합 검증 명령
- Agent 병렬 작업을 고려한 모듈 경계
- Branch, Worktree, Sandbox 기반 병렬 Git 전략
- Local Agent와 Cloud Agent의 역할 분리
- Long Running Task의 상태 관리
- Agent 작업의 Definition of Done
- Planner, Worker, Reviewer 구조
- PM Agent 기반 작업 분배와 검증 구조
- Cloud Agent의 Secret, Network, Repository, Deployment 권한 경계
