# Claude Code Web 실행 자원과 Event-driven Agent 조사

작성 기준일: 2026-09-16

이 문서는 책의 일반 원칙을 검증하기 위한 제품 사례 조사다. 제품 사양과 기능은 변경될 수 있으므로 본문의 일반 원칙과 분리한다.

## 1. 세션별 격리 실행 환경

Anthropic 공식 문서는 Claude Code on the web이 Anthropic 관리 클라우드 인프라에서 동작하며, 작업을 시작할 때 Repository를 격리된 VM에 clone한다고 설명한다.

여러 독립 작업을 동시에 실행할 수 있으며 각 작업은 별도 세션과 branch에서 진행될 수 있다.

출처:

- https://code.claude.com/docs/en/web-quickstart
- https://code.claude.com/docs/ko/claude-code-on-the-web

## 2. 현재 공개된 대략적 자원 상한

2026년 9월 공식 문서 기준 클라우드 세션의 대략적 자원 상한은 다음과 같이 안내된다.

- 4 vCPU
- 16 GB RAM
- 30 GB disk

이 값은 변경될 수 있는 대략적인 상한으로 취급한다.

책에서는 다음과 같이 표현한다.

`2026년 9월 공식 문서에 공개된 대략적인 세션 자원 상한`

다음처럼 표현하지 않는다.

- 세션마다 물리 CPU 4개가 전용 보장된다.
- 16GB RAM이 항상 독점 할당된다.
- 세션을 무제한 병렬 생성할 수 있다.

출처:

- https://code.claude.com/docs/ko/claude-code-on-the-web

## 3. 컴퓨팅 자원과 LLM 사용량

클라우드 VM의 CPU, RAM, disk는 code/build/test command를 실행하는 실행 자원이다.

LLM token/usage는 모델에 전달되는 입력, Context, tool result와 모델 출력에 관련된 별도 관점의 자원이다.

따라서 Gradle build나 테스트가 오래 실행된다는 사실만으로 동일 비율의 token 사용이 발생한다고 해석하지 않는다.

다만 다음 경우에는 LLM 사용량이 증가할 수 있다.

- Agent가 실행 상태를 반복 확인
- 대량 stdout/stderr를 Context로 읽음
- 같은 Repository를 여러 세션에서 반복 탐색
- 실패 후 여러 번 재추론
- 여러 Agent session을 동시에 실행

Anthropic 공식 문서는 Claude Code on the web의 여러 병렬 작업이 계정의 Claude/Claude Code rate limit을 공유하며, 여러 작업을 병렬로 실행하면 사용량을 더 많이 소비한다고 설명한다.

출처:

- https://code.claude.com/docs/ko/claude-code-on-the-web
- https://code.claude.com/docs/en/costs

## 4. PR Auto-fix와 Event-driven Agent

Claude Code on the web의 PR auto-fix는 Event-driven Agent의 사례로 사용할 수 있다.

공식 문서 기준으로 auto-fix가 활성화된 PR에서는 Claude가 GitHub PR 활동을 구독하고 다음 이벤트에 반응할 수 있다.

- CI check failure
- review comment

각 이벤트가 발생하면 Claude가 상황을 조사한다.

- 수정이 명확하다고 판단하면 변경을 수행하고 push할 수 있다.
- 요청이 모호하거나 중요한 아키텍처 판단이 필요한 경우 바로 수정하지 않고 확인을 요구할 수 있다.

출처:

- https://code.claude.com/docs/ko/claude-code-on-the-web
- https://code.claude.com/docs/ko/commands (`/autofix-pr`)

책에서는 이를 제품 기능 소개가 아니라 다음 일반 패턴으로 추상화한다.

```text
Repository Event
      ↓
 deterministic check
      ↓
     PASS → 종료
      ↓ FAIL
Agent 활성화
      ↓
판단 / 수정
      ↓
Runner 재검증
```

실제 제품 내부 구현이 항상 별도의 non-LLM Runner를 먼저 사용한다고 단정하지 않는다. `Runner-first`는 책에서 제안하는 아키텍처이며, Claude 사례는 CI failure/review event가 Agent를 활성화할 수 있다는 사실을 보여주는 사례로만 사용한다.

## 5. 병렬 세션을 실행 노드로 보는 관점

개념적으로 다음처럼 여러 세션을 독립 작업에 사용할 수 있다.

```text
Cloud Session A
- Backend task

Cloud Session B
- Integration task

Cloud Session C
- Docker/build task
```

하지만 여러 세션에서 각각 Repository 전체 분석과 긴 추론을 반복하면 LLM usage도 함께 커질 수 있다.

책에서는 다음을 구분한다.

1. 여러 격리 환경에서 CPU/RAM 작업을 병렬 실행하는 것
2. 여러 LLM Agent가 동시에 codebase를 읽고 추론하는 것

첫 번째는 compute parallelism이고 두 번째는 parallel inference다.

## 6. 책에서 사용할 일반 원칙

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

Claude Code Web의 제품 기능은 위 원칙을 설명하는 사례 중 하나일 뿐이다.

## 7. Result Filter와의 연결

Claude Code Web 자체의 특정 로그 parser를 책의 표준으로 삼지 않는다.

책에서 권장하는 구조는 제품 독립적으로 다음과 같다.

```text
Runner
 ↓
Raw Log / Artifact
 ↓
Result Filter
 ↓
Failure Summary
 ↓
Agent
```

LLM에는 기본적으로 다음 정보만 전달한다.

- exit code
- 실패 테스트
- root cause 후보
- 핵심 stack trace
- warning/error count
- artifact path

추가 정보가 필요할 때만 특정 로그나 source file로 Context를 확대한다.

## 8. 책에서 피해야 할 표현

- 각 세션 자원이 물리적으로 전용이라는 주장
- 병렬 세션 수가 무제한이라는 주장
- 클라우드 명령 실행에는 어떤 형태의 제한도 없다는 주장
- 여러 세션을 실행해도 Claude 사용량이 증가하지 않는다는 주장
- 현재 공개된 자원 상한이 향후에도 유지된다는 주장
- Claude의 PR auto-fix 내부 구조가 책의 Runner-first 구조와 동일하다는 주장

## 9. 본문 반영 위치

2장 `Local Agent, Cloud Agent, Hybrid Agent`에서 다음 내용을 다룬다.

- Compute Resource와 LLM Usage 분리
- Cloud Runner / Agent Worker 구분
- Runner-first / Agent-on-exception
- Result Filter
- Event-driven Agent
- 병렬 Cloud Session과 Context 최소화
- Claude Code Web 실행 환경 사례
- PR Auto-fix 사례

후속 장 연결:

- 8장: Runner 실행 인터페이스
- 10장: 자동 검증과 Definition of Done
- 13장: Cloud Sandbox와 Git 격리
- 17장: token / invocation / compute observability
- 19장: PM Agent scheduling
