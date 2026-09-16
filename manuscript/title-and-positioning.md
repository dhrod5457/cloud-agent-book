# 책 제목과 포지셔닝

## 최종 제목

**클라우드 코딩 에이전트 실전**

## 부제

**Local과 Cloud를 나누고 Task를 위임하는 개발 워크플로 설계**

## 표지용 한 줄 소개

> 더 많은 Agent보다 더 나은 Task Routing, 실행환경, 검증을 설계하는 법

## 짧은 소개 문구

Coding Agent가 코드를 작성할 수 있게 된 뒤의 문제를 다룬다.

어떤 Task를 Local에 남기고, 어떤 Task를 Cloud Runner나 Cloud Agent에 맡길 것인가. Cloud에서 작업을 시작하기 위해 무엇을 준비하고, 어떤 Evidence를 받아야 하며, 언제 Local로 돌아와야 하는가.

이 책은 특정 제품의 사용법보다 이 판단을 반복 가능한 개발 Workflow로 만드는 방법에 집중한다.

## 제목 선택 이유

### 클라우드

이 책의 중심은 LLM 자체가 아니라 독립 실행환경, 비동기 작업 위임, 병렬 Compute, Local과 Cloud 사이의 Handoff다.

### 코딩 에이전트

범용 AI Agent가 아니라 Repository를 읽고 수정하고 Build/Test를 실행하는 개발 Agent가 대상임을 분명하게 한다.

### 실전

제품 기능 소개가 아니라 Task Routing, Task Contract, Prepared Environment, Runner-first, Evidence, Git Handoff, Local Fallback을 실제 개발 Workflow에 적용한다.

## 부제가 설명하는 범위

`Local과 Cloud를 나눈다`는 말은 책의 최종 질문과 연결된다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
```

`Task를 위임한다`는 말은 단순 원격 실행보다 넓은 의미를 가진다.

```text
Task Contract
→ Git Handoff
→ Prepared Environment
→ Runner / Agent
→ Evidence
→ Local Review / Internal Validation
```

## 사용하지 않는 제목 방향

다음 표현은 현재 책의 범위를 넓게 오해하게 만들 수 있으므로 제목에서 사용하지 않는다.

- Agent Platform
- Agent OS
- Multi-Agent System
- Agent-Native Software Engineering
- Autonomous Development Platform

이 주제는 일부 후속 개념과 연결될 수 있지만 현재 책의 중심은 Cloud coding agent를 실제 개발 Workflow에 배치하는 방법이다.

## 표지 / 상세페이지 메시지 우선순위

```text
1. Local인가 Cloud인가
2. Runner인가 Agent인가
3. 작은 Task와 작은 Context
4. 검증 가능한 Evidence
5. Local → Cloud → Local Handoff
6. Cloud가 이득이 아니면 Local로 돌아온다
```

표지와 소개 문구에서도 Agent 수나 자동화 수준보다 이 판단 구조를 우선한다.
