# 참고자료

기준일: 2026-09-16

이 책은 특정 제품의 기능 목록을 설명하는 책이 아니다. 참고자료는 Cloud Agent의 실행 모델, Local / Cloud 경계, 실행환경, Runner-first, Event-driven Workflow 같은 본문 원칙을 확인하는 데 사용한 공식 자료만 정리한다.

제품 UI, 가격, CPU / RAM, 동시 Session 제한처럼 변경 가능성이 높은 정보는 출판 직전에 다시 확인한다.

## GitHub Copilot cloud agent

- GitHub Docs, `About GitHub Copilot cloud agent`
  - https://docs.github.com/en/copilot/concepts/agents/cloud-agent/about-cloud-agent
- GitHub Docs, `Customize the development environment for Copilot cloud agent`
  - https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/customize-the-agent-environment
- GitHub Docs, `Get the best results with Copilot cloud agent`
  - https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results
- GitHub Docs, `About cloud and local sandboxes for GitHub Copilot`
  - https://docs.github.com/en/copilot/concepts/about-cloud-and-local-sandboxes
- GitHub Docs, `About the GitHub Copilot app`
  - https://docs.github.com/en/copilot/concepts/agents/github-copilot-app

본문에서 확인하는 범위:

- 독립된 Cloud 개발환경
- Repository 탐색과 코드 변경
- Test / Lint 실행
- 개발환경 사전 구성
- Issue / PR / Automation과 연결되는 비동기 작업
- Local / Cloud 실행 위치 구분

## OpenAI Codex

- OpenAI, `Addendum to OpenAI o3 and o4-mini system card: Codex`, 2025-05-16
  - https://openai.com/index/o3-o4-mini-codex-system-card-addendum/
- OpenAI, `Codex is now generally available`, 2025-10-06
  - https://openai.com/index/codex-now-generally-available/

본문에서 확인하는 범위:

- Cloud container / sandbox 기반 Coding Task
- Repository와 Development Environment 제공
- 파일 수정과 명령 실행
- Test / Lint / Type Check 실행
- Diff 검토와 Pull Request 연결

## Anthropic Engineering

- Anthropic Engineering, `Quantifying infrastructure noise in agentic coding evals`, 2026-02-05
  - https://www.anthropic.com/engineering/infrastructure-noise
- Anthropic Engineering, `Scaling Managed Agents: Decoupling the brain from the hands`, 2026-04-08
  - https://www.anthropic.com/engineering/managed-agents
- Anthropic Engineering, `Building a C compiler with a team of parallel Claudes`, 2026-02-05
  - https://www.anthropic.com/engineering/building-c-compiler

본문에서 확인하는 범위:

- 실행 자원이 Agent 성능에 영향을 줄 수 있다는 점
- Reasoning과 Execution Environment의 Lifecycle 분리
- Agent가 읽기 쉬운 Test Output과 Artifact 설계
- 독립 Task 병렬화와 Sequential Bottleneck의 차이

공개된 실험 수치는 해당 실험의 결과로만 사용하며 일반적인 성능 기대값으로 확대하지 않는다.

## GitHub Engineering / Continuous AI 사례

- GitHub Blog, `Continuous AI in practice: What developers can automate today with agentic CI`
  - https://github.blog/ai-and-ml/generative-ai/continuous-ai-in-practice-what-developers-can-automate-today-with-agentic-ci/
- GitHub Blog, `Automate repository tasks with GitHub Agentic Workflows`
  - https://github.blog/ai-and-ml/automate-repository-tasks-with-github-agentic-workflows/
- GitHub Next, `Continuous AI`
  - https://githubnext.com/projects/continuous-ai/
- GitHub Blog, `Improving token efficiency in GitHub Agentic Workflows`
  - https://github.blog/ai-and-ml/github-copilot/improving-token-efficiency-in-github-agentic-workflows/
- GitHub Changelog, `Schedule and automate tasks with Copilot cloud agent`, 2026-06-02
  - https://github.blog/changelog/2026-06-02-schedule-and-automate-tasks-with-copilot-cloud-agent/
- GitHub Changelog, `GitHub Agentic Workflows are now in technical preview`, 2026-02-13
  - https://github.blog/changelog/2026-02-13-github-agentic-workflows-are-now-in-technical-preview/

본문에서 확인하는 범위:

- Build / Test / Lint 같은 결정론적 작업과 Agent 판단의 분리
- 정상 경로에서 불필요한 LLM 호출을 제거하는 방식
- 작은 검증 가능한 Task와 PR 단위 반복
- CI Failure / Review / Schedule 같은 Event에서 필요할 때 Agent를 호출하는 방식

## Repository 내부 Research

출판 원고의 상세 근거와 재검증 기록은 다음 문서에서 관리한다.

- `research/chapter-01-cloud-worker-official-sources.md`
- `research/chapter-02-local-cloud-official-sources.md`
- `research/github/continuous-ai-runner-first.md`
- `research/anthropic/agent-native-development-environment.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/infrastructure-noise.md`

## 출판 시 확인 규칙

참고자료의 URL과 제품 기능은 출판 직전에 다시 확인한다. URL이 변경되더라도 본문의 일반 원칙과 특정 제품의 현재 기능을 같은 것으로 취급하지 않는다.

본문은 다음과 같은 구조적 원칙을 유지한다.

```text
Task Routing
→ Prepared Environment
→ Runner-first
→ 필요한 경우 Cloud Agent
→ Evidence
→ Local Review / Internal Validation
```

제품은 이 원칙을 설명하는 근거와 사례이며, 책의 정의 자체는 아니다.
