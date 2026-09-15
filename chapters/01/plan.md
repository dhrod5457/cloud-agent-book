# 1장 설계 - Coding Agent에서 Cloud Worker로

## 장의 목표

이 장은 책 전체에서 사용할 `Cloud Agent`의 정의를 확정한다.

Cloud Agent를 단순히 클라우드에서 실행되는 LLM이나 원격 Chat UI로 설명하지 않는다. Repository를 읽고, 독립 실행환경에서 명령을 실행하고, 코드를 변경하고, 검증 가능한 결과를 Git과 Artifact로 돌려주는 `Remote Development Worker`로 정의한다.

핵심 질문은 다음과 같다.

> Cloud Agent는 일반 LLM이나 Local Coding Agent와 무엇이 다른가?

이 장에서는 아직 Local/Cloud의 세부 선택 기준이나 Token 최적화 방법까지 들어가지 않는다. 먼저 책 전체에서 사용할 작업 주체의 모델을 만든다.

---

## 핵심 정의

```text
Cloud Agent
= LLM
+ Repository
+ 독립 실행환경
+ CPU / RAM / Disk
+ Development Tools
```

책 후반부에서는 이를 다음 문장으로 다시 사용한다.

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

Cloud Agent의 가치는 LLM의 추론 성능에만 있지 않다.

- 개발자 PC와 분리된 작업공간
- 독립 Branch와 Repository 상태
- CPU/RAM/Disk를 사용하는 Build/Test 실행
- 장시간 작업의 비동기 위임
- 여러 Worker의 병렬 실행
- 결과를 Commit/PR/Artifact로 반환하는 작업 흐름

이 관점이 이후 모든 장의 기준이 된다.

---

## 문제 정의

Cloud Agent를 `원격 AI 개발자`라고만 이해하면 다음 요소가 빠진다.

- 실제 코드는 어디에서 checkout되는가?
- 어떤 JDK, Node, Docker, Browser를 사용할 수 있는가?
- Build/Test는 누구의 CPU와 RAM에서 실행되는가?
- 작업 상태는 어떻게 Local 개발자에게 돌아오는가?
- 여러 Cloud Agent가 동시에 일할 때 작업공간은 어떻게 격리되는가?
- Agent가 오래 실행되어도 개발자가 계속 기다려야 하는가?

따라서 Cloud Agent는 `모델 + 실행환경 + Repository + 도구`의 결합으로 설명해야 한다.

---

## 핵심 주장

> 클라우드 에이전트의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> Cloud Agent를 원격 AI 개발자가 아니라 필요할 때 생성할 수 있는 Remote Worker로 이해한다.

이 장에서는 이 두 문장을 정의 수준에서 제시하고, Compute와 Token의 구체적인 관계는 3장에서 설명한다.

---

## 독자가 얻는 것

- Chat LLM, Coding Agent, Cloud Agent의 차이를 설명할 수 있다.
- Cloud Agent를 Remote Development Worker로 정의할 수 있다.
- Cloud Agent의 가치에 LLM뿐 아니라 Repository와 실행환경이 포함되는 이유를 이해한다.
- Cloud Agent의 결과를 대화 응답이 아니라 Commit, Test Result, Artifact, PR까지 포함한 작업 결과로 볼 수 있다.
- 이후 Local/Cloud Task Routing과 병렬 Worker 설명에 필요한 공통 모델을 이해한다.

---

## 예제 Stage

`campus-platform`을 하나의 Remote Worker가 작업할 수 있는 Repository로 본다.

예제에서는 아직 자동화 환경을 구현하지 않는다.

Cloud Worker가 다음 과정을 수행한다고 가정한다.

```text
Task 전달
   ↓
Repository checkout
   ↓
독립 작업공간
   ↓
코드 변경
   ↓
Build / Test
   ↓
Commit / Artifact
   ↓
PR 또는 결과 반환
```

이 흐름을 기준으로 이후 장에서 환경, Context, Branch, Test, Result를 각각 구체화한다.

---

# 절 구성

## 1.1 Chat LLM과 Coding Agent는 무엇이 다른가

Chat LLM은 기본적으로 입력 Context를 읽고 응답을 생성한다.

Coding Agent는 여기에 다음 능력이 결합된다.

- Repository 탐색
- 파일 읽기/수정
- Shell/Tool 실행
- Build/Test 실행
- Git 작업
- 반복적인 수정과 재검증

이 장에서는 특정 제품의 기능 목록을 비교하지 않는다.

핵심은 `응답 생성`에서 `Repository 안에서 작업 수행`으로 역할이 바뀐다는 점이다.

## 1.2 Local Coding Agent에서 Cloud Agent로

Local Coding Agent는 개발자의 PC와 현재 Workspace를 직접 사용한다.

Cloud Agent는 별도의 원격 작업공간에서 Repository를 기반으로 작업한다.

여기서는 차이를 정의만 하고 세부 비교는 2장으로 넘긴다.

## 1.3 Cloud Agent는 실행환경을 가진 Worker다

Cloud Agent를 구성하는 요소를 그림으로 보여준다.

```text
                 Cloud Agent

        +-------------------------+
        |           LLM           |
        +-------------------------+
                    |
        +-------------------------+
        |       Repository        |
        +-------------------------+
                    |
        +-------------------------+
        |  Independent Workspace  |
        | CPU / RAM / Disk        |
        | JDK / Node / Docker     |
        | Browser / Build Tools   |
        +-------------------------+
```

이 구조 때문에 Cloud Agent는 단순 API 호출과 다른 비용/성능 특성을 갖는다.

## 1.4 Remote Worker는 대화가 아니라 Task를 받는다

Remote Worker 관점에서는 다음과 같이 생각한다.

```text
Developer / Local Agent
        ↓
      Task
        ↓
 Cloud Remote Worker
        ↓
   Work + Verify
        ↓
 Evidence Result
```

Task에는 이후 7장에서 다음 정보가 포함된다.

- Goal
- Scope
- Relevant Files
- Validation
- Forbidden Changes
- Expected Output

1장에서는 형식 자체를 자세히 설명하지 않는다.

## 1.5 결과는 답변이 아니라 작업 상태다

Cloud Agent 작업의 결과 후보:

- Commit SHA
- Changed Files
- Diff
- Build Result
- Test Result
- Screenshot / Video
- Log Reference
- PR

따라서 `완료했습니다`라는 자연어 응답만으로 작업 완료를 판단하지 않는다.

Evidence 기반 결과는 8장과 실전 프로젝트 장에서 상세히 다룬다.

## 1.6 Cloud Agent를 제품 이름과 분리해서 이해한다

Claude Code, Codex, GitHub Copilot Coding Agent, Cursor 등의 제품은 현재 구현 사례다.

책의 정의는 특정 제품에 종속되지 않는다.

제품마다 다음이 다를 수 있다.

- Session 생성 방식
- Repository 연결 방법
- CPU/RAM/Disk
- Tool 지원
- Network 정책
- 동시 실행 제한
- 가격/사용량 정책

변경 가능한 제품 사양은 `research/` 문서에서 기준일과 출처를 관리한다.

---

# 필요한 구조/그림

1. Chat LLM → Coding Agent → Cloud Remote Worker 변화
2. `LLM + Repository + Execution Environment + Compute + Tools`
3. Task → Remote Worker → Evidence Result
4. Local 개발자와 Remote Worker 사이의 기본 비동기 흐름

---

# 필요한 실제 예제

`campus-platform`에서 다음 정도만 사용한다.

```text
Task:
attendance 모듈 테스트 보강

Cloud Worker:
- Repository checkout
- branch 생성
- 테스트 추가
- ./gradlew test 실행
- commit
- 결과 반환
```

이 장에서는 아직 Branch 전략, Context Package, Cache, Runner 분리까지 상세히 설명하지 않는다.

---

# 필요한 공식 자료 조사

제품 사례를 사용할 경우 다음만 확인한다.

- Repository 기반 작업 여부
- 독립 실행환경 여부
- 비동기/병렬 작업 가능 여부
- Commit/PR 결과 반환 방식

제품 사양 숫자는 본문 핵심 정의와 분리한다.

기존 조사 문서:

- `research/anthropic/claude-code-web-execution-resources.md`
- `research/github/continuous-ai-runner-first.md`

---

# 다음 장과의 연결

2장에서는 같은 Task를 Local Agent와 Cloud Agent 중 어디에서 실행할지 판단한다.

핵심 질문은 다음으로 바뀐다.

> 이 작업은 Local에서 해야 하는가, Cloud로 보내야 하는가?

---

# 본문에서 의도적으로 다루지 않을 내용

- Agent Ready Software Engineering 전체 방법론
- Agent Platform Architecture
- Agent Memory
- Agent Security Platform
- PM Agent 일반론
- Runner / Result Gateway 상세 구조
- Token 최적화 상세
- Prebuilt Environment / Cache 상세
- 특정 제품 설치법

이 내용 중 Cloud Agent 활용에 필요한 부분만 뒤 장에서 다룬다.
