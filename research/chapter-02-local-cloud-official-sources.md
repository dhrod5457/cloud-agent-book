# 2장 공식 근거 - Local Agent와 Cloud Agent

기준일: 2026-09-16

이 문서는 Local Agent와 Cloud Agent를 제품 이름이 아니라 실행 위치, 격리, Git/Repository 전달, 비동기 작업 관점에서 설명하기 위한 공식 근거를 정리한다.

## GitHub

공식 문서에서 확인되는 내용:

- Copilot cloud agent는 자체 ephemeral development environment에서 작업한다.
- Repository를 탐색하고 코드를 변경하며 automated tests와 linters를 실행할 수 있다.
- GitHub는 cloud sandbox와 local sandbox를 별도 실행 선택지로 설명한다.
- GitHub의 agent 경험은 Issues, Pull Requests, automation과 연결되어 비동기 작업을 수행할 수 있다.

본문에서 추출할 일반 원칙:

- Local과 Cloud의 차이는 모델 이름보다 실행 위치와 접근 범위에서 먼저 발생한다.
- Cloud는 독립 실행환경과 비동기 위임에 유리하다.
- Local은 현재 Workspace, 개발자 장비, 내부 자원과의 짧은 feedback loop에 유리할 수 있다.

공식 자료:

- GitHub Docs, `About GitHub Copilot cloud agent`
- GitHub Docs, `About cloud and local sandboxes for GitHub Copilot`
- GitHub Docs, `About the GitHub Copilot app`

## OpenAI Codex

공식 자료에서 확인되는 내용:

- Codex는 editor, terminal, cloud 등 여러 개발 surface에서 사용할 수 있는 방향으로 제공되어 왔다.
- Cloud coding task는 독립 cloud environment에서 Repository와 tool을 사용해 수행할 수 있다.

본문에서 추출할 일반 원칙:

- Local/Cloud는 대체 관계보다 실행 위치 선택 문제로 설명한다.
- 같은 종류의 coding agent라도 Task에 따라 Local interactive workflow와 Cloud delegated workflow가 달라질 수 있다.

공식 자료:

- OpenAI, `Codex is now generally available`, 2025-10-06

## 본문 사용 규칙

- 제품별 UI/메뉴 사용법을 설명하지 않는다.
- Local이 항상 빠르다거나 Cloud가 항상 강하다는 식으로 일반화하지 않는다.
- VPN, 내부 DB, HSM 접근 가능 여부는 조직/제품 환경에 따라 달라질 수 있으므로 책에서는 Routing 조건으로 설명한다.
- 제품별 구체적 CPU/RAM/동시 Session 제한은 본문에서 고정하지 않는다.
