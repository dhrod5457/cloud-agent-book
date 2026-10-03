# 이 책을 읽는 방법

이 책은 1장부터 18장까지 순서대로 읽을 수 있도록 구성했다.

하지만 클라우드 에이전트를 이미 사용하고 있거나 특정 문제를 해결하려는 독자는 필요한 경로부터 읽어도 된다.

# 처음 읽는다면

처음 클라우드 에이전트 작업 흐름을 설계한다면 순서대로 읽는 것을 권장한다.

```text
Part I
Cloud Agent를 이해한다
        ↓
Part II
어떤 Task를 Cloud로 보낼 것인가
        ↓
Part III
Cloud 실행환경과 검증을 설계한다
        ↓
Part IV
Local과 Cloud를 연결한다
        ↓
Part V
실제 프로젝트에 적용한다
        ↓
Part VI
Cloud의 한계와 다음 단계를 정한다
```

앞 장에서 만든 개념을 뒤 장에서 다시 정의하지 않고 적용하는 구조이기 때문이다.

# Cloud 도입 여부를 먼저 판단하고 싶다면

다음 순서로 읽는다.

```text
1장
Cloud Agent 정의
↓
2장
Local / Cloud 차이
↓
4장
Cloud 실행의 가치
↓
5장
Task Routing
↓
6장
실제 개발 작업 분류
↓
17장
Cloud를 쓰지 말아야 할 조건
```

이 경로를 읽으면 특정 제품을 설치하기 전에 어떤 작업이 클라우드에 적합한지 먼저 판단할 수 있다.

# Token과 비용을 줄이는 것이 관심사라면

다음 장을 연결해서 읽는다.

```text
3장
Compute와 LLM 자원 분리
↓
7장
작은 Task / 작은 Context
↓
8장
작은 Output / Evidence
↓
9장
Startup Cost
↓
10장
Runner-first
↓
12장
병렬화와 중복 Context 비용
```

핵심은 `Agent Prompt를 짧게 쓰는 방법`만 찾는 것이 아니다.

```text
Compute에는 실행을 맡기고
LLM에는 필요한 판단만 맡긴다.
```

# CI/CD와 자동화가 관심사라면

다음 경로가 빠르다.

```text
10장
Runner-first / Agent-on-failure
↓
11장
작업 격리
↓
13장
Handoff
↓
14장
Event-driven Task
↓
15장
통합 운영 모델
↓
18장
Harness / Orchestration
```

CI 실패, PR 검토, 야간 정기 테스트 같은 이벤트가 어떻게 작업이 되고, 어떤 경우에만 에이전트를 호출할지 연결해서 볼 수 있다.

# 실제 적용 예를 먼저 보고 싶다면

15~16장을 먼저 읽어도 된다.

```text
15장
campus-platform 운영 모델
↓
16장
하나의 기능 End-to-End Timeline
```

모르는 개념이 나오면 다음처럼 앞 장으로 돌아간다.

```text
Task Routing → 5장
Task Contract → 7장
Evidence → 8장
Prepared Environment → 9장
Runner-first → 10장
Isolation → 11장
Parallel Cost → 12장
Handoff → 13장
```

# 내부망이 있는 프로젝트라면

다음 장을 함께 읽는다.

```text
2장
Internal Network가 Local / Cloud 선택에 미치는 영향

5장
Hard Constraint 기반 Routing

13장
Cloud 검증과 Local Internal Validation 분리

15장
Tibero / HSM / Internal API를 포함한 운영 모델

17장
Local Fallback과 재Routing
```

클라우드에서 내부망을 억지로 복제하는 방법보다 클라우드에서 끝낼 조건과 로컬에서 확인할 조건을 나누는 데 초점을 둔다.

# 코드블록과 수치 읽는 방법

본문에는 작업 흐름을 설명하기 위한 코드블록과 숫자가 자주 나온다.

예:

```text
Test 10,000개
Build 20분
Retry 2회
PR 20개 / day
```

이 값은 별도 언급이 없는 한 제품의 성능 기준이나 권장값이 아니다.

설명용 수치는 구조와 판단 방법을 보여주기 위한 예다. 실제 프로젝트에서는 빌드 시간, 최초 환경 준비 시간, 검토 처리 능력, 재시도 비율을 직접 측정해 기준을 잡는다.

# 제품 이름을 읽는 방법

GitHub, OpenAI, Anthropic 등의 제품 사례가 나오더라도 이 책은 특정 제품을 표준으로 두지 않는다.

제품 사례에서는 다음만 가져온다.

```text
Repository를 받을 수 있는가?
독립 실행환경이 있는가?
명령을 실행할 수 있는가?
코드를 변경할 수 있는가?
검증 결과를 반환할 수 있는가?
```

가격, CPU / RAM, 동시 세션 수처럼 변경 가능성이 높은 정보는 본문의 핵심 원칙과 분리한다.

# 반복해서 등장하는 AUTH-142 예제

여러 장에서 `AUTH-142 / expired token` 예제가 이어진다.

각 장에서 새로운 문제를 만드는 대신 같은 작업을 다음 관점으로 계속 본다.

```text
7장  Task Contract
8장  Evidence
10장 Runner 재검증
11장 Git / SHA 추적
13장 Handoff
15장 운영 모델
16장 End-to-End Timeline
```

따라서 같은 예제가 다시 나와도 처음부터 다시 설명하는 것으로 읽기보다 **하나의 작업이 작업 흐름의 다음 단계로 이동하는 과정**으로 보면 된다.

# 마지막에는 이 질문으로 돌아온다

책을 읽는 동안 다음 질문을 계속 기준으로 삼는다.

> 이 작업은 로컬에서 해야 하는가, 클라우드로 보내야 하는가?

그리고 클라우드로 보낸다면 한 단계 더 묻는다.

```text
Cloud Runner인가?
Cloud Agent인가?
어떤 Evidence를 받아야 하는가?
언제 Local로 돌아와야 하는가?
```

이 네 질문이 책 전체를 연결하는 읽기 기준이다.
