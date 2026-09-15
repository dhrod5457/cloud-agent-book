# GitHub Continuous AI와 Runner-first 패턴 조사

작성 기준일: 2026-09-16

이 문서는 2장 `Local Agent, Cloud Agent, Hybrid Agent`의 설계 원칙을 검증하기 위한 사례 조사다. GitHub 제품 기능을 책의 일반 원칙과 분리해서 기록한다.

## 1. Continuous AI와 기존 CI/CD의 역할 구분

GitHub는 Continuous AI를 기존 CI/CD의 대체재로 설명하지 않는다.

기존 CI/CD가 잘 처리하는 작업:

- build
- test
- formatting
- static analysis
- 명확한 규칙 기반 검증

AI가 필요한 영역:

- 문서와 구현의 의미 불일치 판단
- CI 실패 원인 분석
- 테스트 부족 영역 판단
- 코드 품질 개선 제안
- 규칙만으로 표현하기 어려운 repository maintenance

따라서 책에서는 다음 원칙으로 추상화한다.

```text
결정론적으로 표현 가능한 작업
→ Runner

판단이 필요한 작업
→ Agent
```

출처:

- https://github.blog/ai-and-ml/generative-ai/continuous-ai-in-practice-what-developers-can-automate-today-with-agentic-ci/
- https://github.blog/ai-and-ml/automate-repository-tasks-with-github-agentic-workflows/
- https://githubnext.com/projects/continuous-ai/

## 2. Continuous test improvement 실험

GitHub Blog이 공개한 GitHub Next 실험 사례에서는 장기간 작은 테스트 개선 작업을 반복했다.

공개 수치:

- 테스트 커버리지: 약 5%에서 거의 100%
- 생성 테스트: 1,400개 이상
- 기간: 45일
- 공개된 token 비용: 약 80달러
- 작은 PR을 매일 생성해 점진적으로 리뷰

출처:

- https://github.blog/ai-and-ml/generative-ai/continuous-ai-in-practice-what-developers-can-automate-today-with-agentic-ci/

주의:

이 수치는 특정 Repository, 당시 모델, 당시 가격, 당시 Workflow를 기반으로 한 공개 실험 결과다. 다른 프로젝트의 비용 예측값으로 사용하지 않는다.

책에서 사용할 원칙:

```text
Coverage 분석
→ 작은 미검증 영역 선택
→ 테스트 생성
→ Runner 검증
→ PASS
→ 작은 PR
→ 반복
```

한 번에 `프로젝트 전체 테스트를 작성하라`고 요청하는 대신 작고 검증 가능한 작업을 반복하는 구조가 핵심이다.

## 3. Token efficiency 사례

GitHub는 2026년 실제 Agentic Workflows의 token usage를 계측하고 여러 production workflow를 최적화한 결과를 공개했다.

중요한 관찰:

### 결정론적인 데이터 수집을 LLM 밖으로 이동

PR diff, 파일 내용, review comment 등 항상 필요한 데이터를 Agent가 MCP 도구 호출로 매번 판단해서 가져오게 하지 않고, 사전 CLI 단계에서 미리 다운로드하는 방식이 사용됐다.

이는 LLM의 tool-use round trip을 제거하고 데이터 수집을 결정론적 단계로 옮기는 패턴이다.

### Relevance gate

Security Guard 사례에서는 security-sensitive file을 건드리지 않는 PR이면 LLM 호출 자체를 건너뛰는 relevance gate를 추가했다.

GitHub Blog은 이 패턴을 요약하면서 `The cheapest LLM call is the one you don't make.`라고 표현한다.

책에서는 직접 인용을 반복하기보다 다음 원칙으로 번역한다.

> 정상 경로는 Runner가 처리하고, 판단이 필요한 예외 경로에서만 Agent를 호출한다.

### 공개된 최적화 결과 예

GitHub가 공개한 초기 결과에는 workflow별로 서로 다른 감소율이 나타났다.

- Auto-Triage Issues: 109회 post-fix run에서 약 62% 감소
- Security Guard: 약 43% 감소
- Smoke Claude: 약 59% 감소

이 수치는 특정 workflow의 Effective Tokens 지표에 대한 실험 결과이므로 책의 일반 성능 수치로 사용하지 않는다.

출처:

- https://github.blog/ai-and-ml/github-copilot/improving-token-efficiency-in-github-agentic-workflows/

## 4. Nightly Agent 패턴

GitHub는 Copilot cloud agent automation 사례로 다음 작업을 공개했다.

`Fix failing tests nightly`: 매일 밤 main branch의 failing test를 확인하고, 수정 시도 후 draft pull request를 연다.

출처:

- https://github.blog/changelog/2026-06-02-schedule-and-automate-tasks-with-copilot-cloud-agent/

책에서는 이를 다음처럼 분리해서 설명한다.

```text
Nightly Scheduler
      ↓
 deterministic test
      ↓
 +----+----+
 |         |
PASS      FAIL
 |         |
종료       ↓
        Agent
          ↓
      수정 시도
          ↓
      Draft PR
```

제품이 실제 내부에서 테스트 전에 항상 별도 비-LLM Runner를 사용하는지까지 일반화하지 않는다.

책의 아키텍처 제안은 기존 CI/Runner를 먼저 실행하고 실패 이벤트에 Agent를 연결하는 형태다.

## 5. Event-driven Agent

GitHub Agentic Workflows는 다음과 같은 trigger를 지원한다고 공개되어 있다.

- pull request
- push
- issue
- comment
- schedule
- manual dispatch 계열

이 사례는 Agent가 상시 실행될 필요 없이 repository event 또는 schedule에 의해 필요할 때 실행될 수 있음을 보여준다.

출처:

- https://github.blog/changelog/2026-02-13-github-agentic-workflows-are-now-in-technical-preview/

## 6. 기존 CI/CD와 Agentic workflow를 겹치지 않게 한다

GitHub는 Agentic Workflows가 build/test/release pipeline을 대체하지 않는다고 명시한다.

책에서는 이를 다음 구조로 사용한다.

```text
CI / Runner
- build
- test
- lint
- static analysis
- rule-based validation

Agent
- failure analysis
- ambiguous decision
- targeted code change
- intent-aware review
```

이 구분은 `Runner-first / Agent-on-exception` 원칙의 근거 중 하나다.

## 7. 책에 반영할 일반 원칙

1. 정상적인 deterministic path에는 LLM을 넣지 않는다.
2. PASS 결과는 Agent가 다시 읽지 않아도 되게 한다.
3. FAIL 또는 판단이 필요한 이벤트만 Agent로 승격한다.
4. Agent가 항상 필요로 하는 데이터는 가능한 한 pre-agentic step에서 준비한다.
5. Agent가 읽는 Context와 tool schema를 최소화한다.
6. 큰 목표는 작은 검증 가능한 작업으로 나누고 PR 단위로 반복한다.
7. Agent 결과는 기존 CI/Runner로 다시 검증한다.

핵심 문장:

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

## 8. 본문 작성 시 주의사항

- Continuous test improvement의 45일 / 1,400+ tests / 약 $80 수치는 사례로만 사용한다.
- 비용 절감률은 특정 workflow 결과로만 표기한다.
- GitHub Agentic Workflows 내부 구현을 책의 표준 아키텍처라고 표현하지 않는다.
- GitHub 사례에서 확인된 사실과 책이 제안하는 `Runner-first` 구조를 구분한다.
- 제품 기능과 가격은 장 집필 시 다시 공식 자료를 확인한다.
