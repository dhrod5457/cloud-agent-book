# 용어집

이 용어집은 책 전체에서 반복해서 사용하는 용어의 의미를 고정하기 위한 것이다. 특정 제품의 명칭이나 구현 세부사항은 포함하지 않는다.

## Agent-on-failure

Cloud Runner가 먼저 검증을 실행하고, 재현 가능한 실패 중 코드 판단이 필요한 경우에만 Cloud Agent를 호출하는 방식.

## Artifact

Test Report, Log, Screenshot, Video, Trace처럼 실행 결과를 보존한 원본 또는 상세 결과물. Agent가 처음 읽는 작은 Evidence와 구분한다.

## Base SHA

Cloud Task가 시작한 Git 기준점. 어떤 Source 상태에서 작업을 시작했는지 식별한다.

## Cloud Agent

Repository와 독립 실행환경을 사용해 Task를 수행하는 Remote Development Worker. 이 책에서는 판단과 제한된 코드 수정이 필요한 구간에 사용한다.

## Cloud Runner

Build, Test, E2E, Docker Build처럼 명령과 판정 기준이 정해진 작업을 실행하는 Cloud 실행 주체. LLM 판단이 필요하지 않은 결정론적 경로를 우선 담당한다.

## Cloud Session

Repository, Workspace, 실행환경, Tool, Agent Interaction이 결합된 하나의 Cloud 작업 실행 단위. 제품별 Session 명칭과 정확히 일치한다는 의미는 아니다.

## Context Duplication

여러 Agent가 같은 Repository, 문서, 공통 Source를 반복해서 읽으면서 생기는 중복 Context와 탐색 비용.

## Developer Blocking Time

Cloud Task의 총 실행시간이 아니라 개발자가 해당 Task 때문에 다른 일을 진행하지 못하고 기다린 시간.

## Evidence

작업 결과를 검증할 수 있도록 정리한 작은 구조화 결과. Result SHA, Validation Result, 실패 요약, Artifact Reference 등이 포함될 수 있다.

## Failure Fingerprint

동일한 실패가 반복되는지 식별하기 위해 Test, Error Type, 핵심 Message, 위치 같은 정보를 조합한 실패 식별 정보.

## Fan-out

서로 독립적인 Task나 검증을 여러 Runner 또는 Worker로 나누어 동시에 실행하는 것.

## Fan-in

병렬 실행된 결과를 다시 합치고 Review, Merge, Regression, Integration Validation을 수행하는 단계.

## Handoff

Task와 Source 상태를 한 실행 위치에서 다른 실행 위치로 넘기는 과정. 이 책의 기본 경계는 Git, Task Contract, Evidence다.

## Harness

Agent가 반복해서 탐색하거나 추론해야 했던 개발 규칙을 발견 가능하고 실행 가능한 형태로 제공하는 구성. 예를 들어 표준 Build/Test Script, AGENTS.md, Task Contract, Result Gateway가 포함될 수 있다.

## Human Steering

작업 도중 사람이 방향을 자주 확인하고 수정해야 하는 정도. Human Steering이 높을수록 비동기 Cloud 위임의 이점이 줄어들 수 있다.

## Local Agent

개발자의 현재 Workspace와 짧은 Feedback Loop 안에서 사용하는 Agent. 요구사항 탐색, Architecture, 내부 자원 접근, Human Steering이 많은 작업에 유리할 수 있다.

## Local Fallback

Cloud Task를 계속 유지하는 이점이 사라졌을 때 현재 결과와 Evidence를 보존한 채 Local 또는 Hybrid Workflow로 이동하는 것. 실패가 아니라 재Routing의 한 형태다.

## Orchestration

Task Classification, Environment Selection, Runner / Agent Selection, Retry, Validation, Evidence 연결처럼 반복되는 실행 결정을 Workflow로 묶는 것.

## Prepared Environment

Task가 시작될 때 Runtime, Tool, Dependency, Cache가 이미 준비되어 있어 Agent가 개발환경 설치부터 반복하지 않도록 만든 Cloud 실행환경.

## Progressive Context

처음에는 Relevant Files 같은 작은 Context만 제공하고, Task 수행에 실제로 필요한 경우에만 Direct Dependency, 관련 문서, 넓은 Module Context 순으로 확장하는 방식.

## Result Gateway

대형 Tool Output과 Artifact를 그대로 Agent에게 전달하지 않고, 필요한 결과를 작은 구조화 Evidence로 변환하거나 필요한 상세 결과를 선택적으로 조회하게 하는 경계.

## Result SHA

Cloud Agent 또는 작업 결과로 만들어진 Git Commit의 SHA. Validation Result와 Evidence는 가능한 한 이 Source 상태와 연결한다.

## Runner-first

검증 가능한 작업은 먼저 Cloud Runner나 기존 CI가 실행하고, LLM 판단은 필요한 예외 경로에만 사용하는 원칙.

## Task

Cloud 또는 Local 실행 위치에 위임할 수 있도록 범위와 완료 조건을 가진 작업 단위.

## Task Candidate

Issue, CI Failure, Review Comment, Schedule 같은 Event에서 만들어진 작업 후보. Event가 발생했다고 즉시 Agent Task가 되는 것은 아니며 Dedup, Classification, Routing을 거친다.

## Task Contract

Cloud Worker가 독립적으로 작업을 시작할 수 있도록 Goal, Scope, Relevant Files, Forbidden Changes, Validation, Expected Result, Output / Evidence 등을 정의한 입력 경계.

## Task Routing

Task의 특성과 제약을 기준으로 Local, Cloud Runner, Cloud Agent, Hybrid 중 실행 위치와 실행 주체를 결정하는 과정.

## Validation Result

특정 Source 상태에서 실행한 Build/Test/E2E 등의 검증 결과. PASS / FAIL뿐 아니라 어떤 명령과 범위를 실행했는지 추적할 수 있어야 한다.

## Hybrid Workflow

하나의 Task나 기능을 단계별로 Local과 Cloud에 나누어 수행하는 방식. 예를 들어 Cloud에서 일반 검증을 끝낸 뒤 Tibero, HSM, VPN-only API 같은 내부 자원 검증을 Local에서 이어갈 수 있다.

## 이 책에서 유지하는 기본 관계

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ Evidence / Artifact
→ PR
```

실행 주체는 다음처럼 구분한다.

```text
Local / Local Agent
→ 요구사항 / Architecture / Human Steering / Internal Validation / Review

Cloud Runner
→ Build / Test / E2E / Docker / 결정론적 검증

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정
```
