# Claude Code Web 실행 자원과 LLM 사용량 조사

작성 기준일: 2026-09-16

이 문서는 책의 일반 원칙을 검증하기 위한 제품 사례 조사다. 제품 사양은 변경될 수 있으므로 본문의 일반 원칙과 분리한다.

## 확인된 사실

### 세션별 격리 실행 환경

Anthropic 공식 문서는 Claude Code on the web이 Anthropic 관리 클라우드 인프라에서 동작하며, 작업을 시작할 때 격리된 가상 머신에 Repository를 clone한다고 설명한다.

또한 여러 작업을 병렬로 실행할 수 있고, 각 작업은 자신의 격리된 환경에서 독립적으로 진행된다. 같은 Repository를 대상으로 여러 작업을 동시에 실행하는 것도 가능하다.

출처:

- https://support.claude.com/en/articles/12618689-claude-code-on-the-web
- https://code.claude.com/docs/en/web-quickstart

### 현재 클라우드 세션의 대략적 자원 상한

Claude Code 공식 문서 기준으로 클라우드 세션의 대략적인 자원 상한은 다음과 같다.

- 4 vCPU
- 16 GB RAM
- 30 GB disk

공식 문서는 이 값이 대략적인 상한이며 시간이 지나면서 바뀔 수 있다고 명시한다. 대형 빌드나 메모리 집약적 테스트는 한계를 넘으면 실패하거나 종료될 수 있다.

출처:

- https://code.claude.com/docs/ko/claude-code-on-the-web

따라서 본문에서는 이 숫자를 Claude Code Web의 영구적인 계약 사양으로 표현하지 않고, "2026년 9월 공식 문서에 공개된 대략적인 상한"으로 표현한다.

### 컴퓨팅 자원과 LLM 사용량은 별도 관점으로 봐야 한다

Claude Code 공식 비용 문서는 Claude Code의 사용 비용을 API token consumption 기준으로 설명하며, context 크기가 커질수록 token 사용량이 증가한다고 설명한다.

또한 `/usage` 예시는 API 처리 시간과 전체 wall-clock 시간을 별도로 표시한다. 이는 모델 호출에 소비되는 자원과 세션 전체 실행 시간이 동일한 측정값이 아님을 보여준다.

클라우드 VM의 CPU, RAM, disk는 코드와 명령을 실행하기 위한 실행 자원이고, LLM token은 모델에 전달되는 입력과 모델이 생성하는 출력에 관련된 사용량이다.

따라서 Gradle build나 테스트가 CPU를 오랫동안 사용한다고 해서 그 실행 시간 자체가 같은 비율의 token 사용으로 직접 변환되는 것은 아니다. 다만 실행 결과의 로그를 Claude가 읽거나, 테스트 중간에 추가 판단을 반복하거나, 여러 Agent 세션이 동시에 추론하면 token/plan usage는 증가할 수 있다.

출처:

- https://code.claude.com/docs/en/costs
- https://support.claude.com/en/articles/14552983-models-usage-and-limits-in-claude-code

### 병렬 실행과 사용량 제한

공식 문서는 여러 session/subagent를 동시에 실행하면 token usage가 증가한다고 명시한다. 또한 Claude Code의 subscription usage는 계정 또는 조직의 사용량 제한을 따른다.

따라서 여러 클라우드 세션을 실행할 수 있다는 사실을 "무료 병렬 추론 자원"으로 해석하면 안 된다.

구분해야 할 것은 다음 두 가지다.

1. 여러 격리 실행 환경에서 build/test 같은 컴퓨팅 작업을 병렬 실행하는 것
2. 여러 LLM 세션이 동시에 codebase를 읽고 추론하는 것

첫 번째는 클라우드 실행 노드의 병렬성이고, 두 번째는 LLM 사용량을 동시에 소비하는 병렬 추론이다.

출처:

- https://code.claude.com/docs/en/agents
- https://code.claude.com/docs/en/errors

## 책에서 사용할 일반 원칙

제품과 무관하게 다음 원칙으로 추상화한다.

> LLM은 판단하고, 실행 환경은 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

빌드, 테스트, 정적 분석, Docker build, migration validation과 같이 결정론적으로 실행할 수 있는 작업은 가능한 한 실행 노드에서 수행한다.

LLM에는 원본 로그 전체보다 다음과 같은 구조화된 결과를 반환한다.

- exit code
- 전체/성공/실패 건수
- 실패 테스트명
- root cause
- 핵심 stack trace
- 중요 warning
- 원본 로그 위치

## 책에서 피해야 할 표현

다음 주장은 공식 문서가 보장하지 않으므로 단정하지 않는다.

- 각 세션의 4 vCPU와 16 GB RAM이 물리적으로 전용 할당된다는 주장
- 병렬 세션 수가 무제한이라는 주장
- 클라우드 명령 실행에는 어떤 형태의 사용량 제한도 없다는 주장
- 여러 세션을 실행해도 Claude 사용량이 증가하지 않는다는 주장
- 현재 공개된 자원 상한이 향후에도 유지된다는 주장

## 본문 반영 위치

2장 `Local Agent, Cloud Agent, Hybrid Agent` 안에 다음 독립 절을 둔다.

`클라우드 에이전트를 AI 개발자가 아니라 실행 노드로 보기`

이 절에서는 다음을 다룬다.

- Compute Resource와 LLM Token의 분리
- CPU-heavy / context-light 작업
- Test Runner 패턴
- Result Filter
- Agent Worker와 Test Runner의 분리
- 여러 Cloud Session을 병렬 테스트 노드로 사용하는 방식
- Task Context 최소화
- Claude Code Web 사례와 제한사항

이후 17장 Observability에서는 token, execution time, retry, compute resource 사용량을 서로 다른 지표로 다시 연결한다.
