# Scope

## 반드시 다룰 범위

### Agent Ready Project Structure

- Agent가 이해하기 쉬운 저장소 구조
- 모듈 경계와 의존성 방향
- 공통 모듈 남용 방지
- Generated file, migration, shared file의 충돌 관리
- Agent-friendly Repository 설계

### Agent Contract

- README.md
- AGENTS.md
- CLAUDE.md
- architecture.md
- development.md
- testing.md
- ADR
- 프로젝트 목적, 기술 스택, 빌드·테스트·검증 방법, 금지 규칙, Git 정책

### Task Contract

작업 단위마다 최소한 다음 정보를 명시하는 방식을 다룬다.

- Goal
- Scope
- Allowed Files
- Forbidden Changes
- Dependencies
- Acceptance Criteria
- Verification

### 재현 가능한 실행 환경

- 자동 Setup
- 로컬과 CI 명령 통일
- Build, Test, Verify 표준화
- Container 또는 Sandbox 기반 환경 격리

### 테스트 가능성

- Unit Test
- Integration Test
- Contract Test
- E2E Test
- Testcontainers
- Mock Server
- Fake Adapter
- 외부 시스템이 없어도 검증 가능한 구조

### 자동 검증과 Definition of Done

- Format
- Lint
- Compile
- Unit Test
- Integration Test
- Architecture Rule
- Security Check
- Secret Detection
- Git Diff 검토
- PASS/FAIL 기반 완료 판단

### 병렬 Agent 개발

- Branch per Agent
- Git Worktree
- Independent Clone
- Cloud Sandbox
- 충돌 최소화를 위한 모듈 설계
- Migration 및 Shared File 충돌 관리

### Agent Lifecycle

- 프로젝트 상태 확인
- 작업 결정
- 작업 분해
- Agent 수 결정
- Agent 생성
- 작업 할당
- 실행
- 테스트
- 검증
- Merge
- Agent 종료

### Agent Orchestration

- Planner
- Worker
- Tester
- Reviewer
- Security Reviewer
- Integrator
- PM Agent
- 작업 복잡도에 따른 역할 선택

### Agent Memory와 Project Memory

- 세션 컨텍스트와 지속 상태의 분리
- task list
- progress
- decisions
- failed attempts
- test result
- next task
- Git과 프로젝트 문서를 Project Memory로 사용하는 방법

### Local / Cloud / Hybrid Agent

- Local Agent의 내부망 접근
- Cloud Agent의 격리된 코드 작업
- Hybrid 구조
- VPN, 내부 Git, Nexus, Tibero/Oracle, Jenkins, Redis, Kafka, HSM, 내부 API가 존재하는 기업 환경

### Trust Boundary와 Security

- Secret 전달 범위
- Network 접근 범위
- Repository 쓰기 권한
- Production 접근
- PR 및 Merge 권한
- Deployment 권한
- Agent별 최소 권한

### Agent Observability와 Evaluation

- queued
- running
- blocked
- failed
- verifying
- completed
- 테스트 성공과 작업 품질 평가의 차이
- 변경 범위, 불필요한 수정, 요구사항 충족도 평가

## 다루지만 깊게 들어가지 않는 범위

다음 기술은 책의 원칙을 설명하기 위한 수단으로 사용한다.

- Docker
- Testcontainers
- GitHub Actions
- Jenkins
- Redis
- Kafka
- MCP
- Agent SDK/API

각 기술의 입문서 수준 사용법까지 설명하지 않는다.

## 의도적으로 제외할 범위

- Claude Code 사용 설명서
- Codex 사용 설명서
- GitHub Copilot 사용 설명서
- Devin 사용 설명서
- Prompt Engineering 입문
- LLM 또는 Transformer 원리
- 특정 Agent Framework 튜토리얼
- MCP 서버 개발 입문서
- AI 모델 성능 비교
- IDE별 사용법
- 일반적인 Spring Boot 입문
- 일반적인 Docker 입문
- 일반적인 CI/CD 입문

## Java/Spring Boot 예제의 역할

Java/Spring Boot는 추상적인 원칙을 실제 프로젝트에서 검증하기 위한 주요 예제다.

예제 자체가 책의 목적은 아니다. 특정 프레임워크 기능보다 다음 변화 과정을 보여주는 데 사용한다.

일반 프로젝트 → 재현 가능한 환경 → Agent Contract → 자동 검증 → 테스트 격리 → 모듈 분리 → Local Agent → Cloud Agent → Parallel Agent → PM Agent → Full Agentic Development

## 특정 AI 제품의 사용 범위

Claude Code, Codex, GitHub Copilot Coding Agent, Devin 등은 다음 용도로 사용한다.

- 현재 Agent 기능의 실제 사례
- Local/Cloud 실행 모델 비교
- Sandbox와 Repository 접근 방식 비교
- 책의 설계 원칙이 실제 제품에서도 적용 가능한지 검증

제품별 UI나 명령 사용법 자체를 책의 중심 내용으로 삼지 않는다.
