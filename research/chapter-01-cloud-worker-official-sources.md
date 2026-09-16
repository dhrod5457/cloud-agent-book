# 1장 공식 근거 - Cloud Worker 실행 모델

기준일: 2026-09-16
공식 자료 재검증: 2026-09-16

이 문서는 1장 본문에서 Cloud coding agent의 공통 실행 모델을 설명할 때 사용할 공식 근거를 정리한다.

제품 사용법이나 사양 비교가 목적이 아니다.

## GitHub Copilot cloud agent

공식 문서에서 확인되는 내용:

- Task 수행 중 자체 ephemeral development environment를 사용한다.
- 해당 환경에서 Repository를 탐색하고 코드를 변경할 수 있다.
- automated tests와 linters를 실행할 수 있다.
- 개발환경에 tools/dependencies를 사전 설치하도록 구성할 수 있다.
- Issue/PR/automation과 연결해 비동기 작업을 수행할 수 있다.

공식 자료:

- GitHub Docs, `About GitHub Copilot cloud agent`
  - https://docs.github.com/en/copilot/concepts/agents/cloud-agent/about-cloud-agent
- GitHub Docs, `Configure the development environment`
  - https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/customize-the-agent-environment
- GitHub Docs, `Best practices for using GitHub Copilot to work on tasks`
  - https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results

본문에서 사용할 일반 원칙:

> Cloud coding agent는 LLM 응답만 생성하는 것이 아니라 Repository와 실행환경을 이용해 코드를 수정하고 검증할 수 있다.

> 사전 준비된 dependency/tool 환경은 Agent가 매번 설치 방법을 추론하는 비용을 줄일 수 있다.

## OpenAI Codex

공식 자료에서 확인되는 내용:

- Codex는 cloud-based coding/software engineering agent로 소개되었다.
- Task는 cloud container/sandbox에서 실행될 수 있다.
- Repository와 사용자 정의 development environment를 작업환경에 제공할 수 있다.
- 해당 환경에서 파일을 읽고 수정하며 tests, linters, type checkers 같은 command를 실행할 수 있다.
- 작업 후 결과와 diff를 검토하거나 pull request 형태로 연결할 수 있다.

공식 자료:

- OpenAI, `Addendum to OpenAI o3 and o4-mini system card: Codex`, 2025-05-16
  - https://openai.com/index/o3-o4-mini-codex-system-card-addendum/
- OpenAI, `Codex is now generally available`, 2025-10-06
  - https://openai.com/index/codex-now-generally-available/

본문에서 사용할 일반 원칙:

> Cloud coding agent의 작업 단위에는 모델뿐 아니라 Repository, execution environment, tools가 함께 포함된다.

## 본문 사용 규칙

- 특정 제품을 `Cloud Agent의 정의`로 일반화하지 않는다.
- 제품별 CPU/RAM/동시 실행 한도는 본문 핵심 근거로 사용하지 않는다.
- 제품 UI/설정 방법은 설명하지 않는다.
- 여러 제품에서 공통으로 확인되는 `Repository + isolated/ephemeral environment + command execution + code changes + verification/result` 패턴만 사용한다.
- 기능이 변경될 수 있는 제품 세부사항은 research 문서에 남기고 본문에서는 원칙으로 추상화한다.
