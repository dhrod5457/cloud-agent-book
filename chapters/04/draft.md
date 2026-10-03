# 4장. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

클라우드 에이전트를 사용하는 이유를 `AI가 코드를 대신 작성한다`로만 설명하면 클라우드의 장점이 잘 보이지 않는다.

로컬 에이전트도 저장소를 읽고 코드를 수정하며 테스트를 실행할 수 있다. 같은 에이전트라도 클라우드에서 실행할 때 달라지는 것은 **실행 위치와 작업 방식**이다.

이 장에서는 클라우드의 가치를 네 가지로 본다.

```text
독립 실행환경
장시간 작업의 비동기 위임
독립 Task의 병렬 실행
Agent Execution Time과 Developer Blocking Time의 분리
```

핵심은 다음 두 문장이다.

> 클라우드 에이전트의 핵심 가치는 더 많은 토큰이 아니라 독립 실행환경과 병렬성이다.

> 클라우드 에이전트가 오래 실행되는 것과 개발자가 오래 기다리는 것은 같은 의미가 아니다.

3장에서 실행 자원과 LLM을 분리했다면, 이번 장에서는 그 실행 자원을 개발 작업 흐름에서 어떻게 활용하는지 본다.

---

## 1. 독립 실행환경이 첫 번째 가치다

로컬 에이전트는 개발자의 현재 컴퓨터와 자원을 공유하는 경우가 많다.

```text
Developer Mac
├─ IDE
├─ Local Agent
├─ Docker
├─ Database
├─ Browser
├─ Emulator
└─ Gradle
```

여기에 전체 테스트, Testcontainers, Docker 빌드, 브라우저 E2E까지 동시에 실행하면 CPU, RAM, 디스크 I/O가 현재 개발 작업과 경쟁한다.

클라우드에서는 이 실행을 별도 작업자에게 맡길 수 있다.

```text
Developer Mac
├─ IDE
├─ Local Agent
└─ 현재 기능 개발

Cloud Worker
├─ Repository
├─ CPU / RAM / Disk
├─ Gradle
├─ Docker
├─ Browser
└─ Test Runtime
```

클라우드 환경이 항상 로컬보다 빠르다는 뜻은 아니다. 여기서 중요한 점은 **개발자의 현재 작업공간과 실행 자원을 분리할 수 있다는 것**이다.

이 분리는 다음 효과를 만든다.

- 무거운 빌드 / 테스트가 로컬 작업을 덜 방해한다.
- 작업별로 독립된 실행환경을 사용할 수 있다.
- 하나의 개발자 PC보다 많은 실행 작업을 동시에 수행할 수 있다.
- 작업이 끝날 때까지 터미널을 계속 지켜볼 필요가 줄어든다.

---

## 2. Cloud Worker는 독립 작업 노드다

1장에서 클라우드 에이전트를 다음처럼 정의했다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

실행 관점에서는 다음처럼 볼 수 있다.

```text
Cloud Worker
= Task
+ Repository State
+ Runtime
+ Compute
+ Tools
+ Result
```

클라우드 작업자가 가치 있으려면 로컬 개발자의 현재 화면을 계속 따라가는 것이 아니라, 일정한 기준점에서 독립적으로 작업할 수 있어야 한다.

예를 들어 다음 정도가 명확하다고 하자.

```text
Base SHA
abc123

Task
AuthService expired token 수정

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
HTTP 401
```

클라우드 작업자는 이 상태를 기준으로 작업할 수 있다. 그동안 개발자는 다른 브랜치나 다른 기능을 진행할 수 있다.

작업 입력 형식은 7장의 작업 명세에서 구체화한다. 여기서는 한 가지 원칙만 기억하면 된다.

> 클라우드의 비동기성은 작업이 현재 개발자의 작업공간과 분리될 수 있을 때 생긴다.

---

## 3. 장시간 작업은 Local에서 분리할 가치가 크다

클라우드에 보내기 좋은 대표 작업은 시간이 오래 걸리면서 중간에 사람의 판단이 거의 필요하지 않은 작업이다.

예:

```text
전체 Unit Test
Integration Test
Docker Build
Frontend E2E Regression
Static Analysis
Migration Validation
반복 Refactoring 후 전체 검증
```

예를 들어 통합 테스트가 오래 걸린다고 하자.

로컬에서는 다음 자원을 계속 점유할 수 있다.

```text
JVM
Docker
Testcontainers
Database Container
Disk I/O
Network
```

클라우드로 분리하면 구조가 달라진다.

```text
Developer
→ Cloud Task 위임
→ 다음 기능 진행

Cloud Worker
→ Integration Test
→ Evidence / Artifact 생성
```

여기서 `오래 걸리면 무조건 Cloud`라는 규칙을 만들지는 않는다.

장시간 작업이라도 범위가 불명확하거나 사람의 중간 판단과 방향 조정이 계속 필요하거나 내부망에서만 재현된다면 로컬이 더 적합할 수 있다.

어떤 작업을 실제로 클라우드로 보낼지는 5장에서 판단한다.

---

## 4. Agent Execution Time과 Developer Blocking Time은 다르다

클라우드 에이전트의 성능을 `몇 분 만에 끝냈는가`만으로 보면 비동기 작업의 장점을 놓친다.

아래 시간은 설명용 예다.

```text
10:00 Cloud Task 위임
10:01 Developer는 다음 기능 시작
10:40 Cloud Task 완료
11:20 Developer가 결과 Review
```

클라우드 작업의 실행시간은 40분이지만, 개발자가 그 40분 내내 기다린 것은 아니다. 따라서 다음 두 값을 구분해서 본다.

```text
Agent Execution Time
= Worker가 Task를 수행한 시간

Developer Blocking Time
= 해당 Task 때문에 Developer가 실제로 멈춘 시간
```

클라우드 작업이 로컬보다 조금 늦게 끝나더라도 개발자가 다른 일을 하지 못하고 기다리는 시간은 줄어들 수 있다.

그래서 클라우드 활용을 평가할 때 다음을 함께 본다.

```text
Agent Execution Time
Queue Time
Developer Blocking Time
Review Time
Retry Time
Local Resource Occupancy
```

이 값들은 서로 같은 방향으로 움직이지 않는다.

클라우드 실행 자원 사용량이 늘어도 개발자가 다른 일을 하지 못하고 기다리는 시간은 줄 수 있고, 에이전트가 많은 결과를 만들어도 검토 시간은 늘 수 있다.

---

## 5. Local Resource Occupancy도 비용이다

개발자의 컴퓨터가 충분히 빠르더라도 여러 작업이 동시에 실행되면 현재 개발 흐름에 영향을 줄 수 있다.

```text
IDE
Local Agent
Docker
PostgreSQL
Redis
Browser
Emulator
Gradle Test
Playwright
```

무거운 검증을 클라우드 작업자에게 맡기면 로컬에서는 다음 작업에 자원을 남길 수 있다.

```text
코드 탐색
Architecture 판단
빠른 수정 반복
내부망 작업
최종 Review
```

따라서 클라우드 비용을 토큰이나 클라우드 실행 자원만으로 보지 않는다.

개발자 장비의 점유와 작업 전환 비용도 전체 작업 흐름 비용에 포함한다.

---

## 6. 비동기 위임에 적합한 Task는 따로 있다

클라우드 작업자에 맡겨 두고 개발자가 다른 일을 하려면 작업이 일정 시간 독립적으로 진행될 수 있어야 한다.

비동기 위임에 유리한 신호:

```text
Scope가 명확함
완료 조건이 있음
중간 질문이 적음
독립 검증 가능
Git / Artifact로 결과 회수 가능
```

예:

```text
특정 Service Test 추가
Module 전체 Test
Docker Build
E2E Regression
Dependency Update 검증
```

반대로 다음 작업은 개발자의 개입이 계속 필요할 수 있다.

```text
새 인증 Architecture 방향 탐색
재현 조건이 없는 Production 장애 조사
여러 UI 안을 빠르게 비교하는 작업
```

이런 작업은 로컬에서 짧은 피드백 주기를 유지하는 편이 자연스럽다.

클라우드를 쓸 수 있는가보다 **사람이 없어도 이 작업이 의미 있게 전진할 수 있는가**를 먼저 본다.

---

## 7. 병렬화의 대상은 Agent가 아니라 독립 Task다

클라우드에서는 여러 작업자를 동시에 실행할 수 있다. 하지만 `Agent를 몇 개 띄울 수 있는가`가 첫 질문은 아니다.

먼저 묻는다.

> 서로 독립적으로 실행하고 검증할 수 있는 작업이 몇 개인가?

하나의 Git SHA를 다음처럼 검증한다고 하자.

```text
                  abc123
                     |
        +------------+------------+------------+
        |            |            |            |
      Unit       Integration     E2E        Docker
        |            |            |            |
        +------------+------------+------------+
                     |
                  Evidence
```

이 경우 네 개의 에이전트가 필요한 것은 아니다.

대부분은 네 개의 클라우드 실행기가 각각 독립적인 계산 작업을 실행하면 된다. 실패 분석이나 코드 수정이 필요할 때만 에이전트가 들어간다.

즉 클라우드 병렬성의 첫 사용처는 대개 **여러 모델의 병렬 추론이 아니라 계산 작업의 병렬 실행**이다.

여러 작업자를 실제로 얼마나 동시에 실행해야 하는지, 중복 맥락 정보와 검토 / 결과 통합 비용이 어디서 생기는지는 12장에서 다룬다.

---

## 8. 병렬화 전에는 독립성만 확인한다

이 장에서는 병렬화의 상세 비용 계산까지 들어가지 않는다. 다만 시작 조건 하나는 명확히 한다.

좋은 병렬 작업:

```text
Unit Test
Integration Test
E2E
Docker Build
```

같은 SHA를 읽고 각자 독립적으로 검증할 수 있다.

또 서로 다른 모듈의 제한된 변경도 병렬화 후보가 될 수 있다.

```text
Task A → attendance module
Task B → notification module
```

반대로 같은 핵심 파일이나 같은 DB 구조를 동시에 바꿔야 한다면 실제 독립 작업이 아니다.

```text
Task A → UserService.java
Task B → UserService.java
```

이런 경우의 병합 / 검토 비용과 병렬화 상한은 11~12장에서 자세히 다룬다.

4장에서 기억할 것은 하나다.

> 클라우드의 병렬성은 독립 작업이 있을 때만 가치가 생긴다.

---

## 9. Evidence가 있어야 비동기 위임이 끝난다

개발자가 클라우드 작업자의 실행 과정을 계속 지켜보지 않는다면 완료 시 검증 가능한 결과가 필요하다.

예:

```text
Git SHA: abc123
Unit: PASS
Integration: PASS
Docker: PASS
E2E: FAIL 1
Artifact: e2e-failure-screenshot.png
```

이런 결과가 있으면 개발자는 전체 로그를 처음부터 읽지 않고 실패한 지점부터 확인할 수 있다.

검증 근거의 구조와 도구 출력을 줄이는 방법은 8장에서 다룬다.

여기서는 다음 관계만 연결한다.

```text
비동기 위임
→ 작업 과정을 계속 관찰하지 않음
→ 완료 시 검증 가능한 Evidence 필요
```

---

## 10. campus-platform에 적용하면

출결 API를 수정한 뒤 다음 검증이 필요하다고 하자.

```text
Unit Test
Integration Test
Admin Web E2E
Docker Build
```

로컬에서 순차적으로 실행할 수도 있다.

또는 Base SHA를 기준으로 클라우드에 분리할 수 있다.

```text
Local
→ Attendance / Auth 수정
→ Base SHA abc123

Cloud
├─ Cloud Runner #1 :attendance:test
├─ Cloud Runner #2 integrationTest
├─ Cloud Runner #3 admin-web E2E
└─ Cloud Runner #4 Docker Build
```

개발자는 클라우드 검증이 진행되는 동안 다음 기능을 작업한다.

검증 결과가 돌아오면 실패 항목만 추가로 분석한다.

이 예제에서 클라우드의 가치는 `Test가 반드시 더 빨리 끝난다`는 것이 아니다.

```text
Local Compute 점유 감소
+
Developer Blocking Time 감소
+
독립 검증 병렬 실행
```

이 세 효과를 얻을 수 있다는 점이다.

---

## 11. 이 장에서 측정할 값

클라우드 에이전트를 도입한 뒤 단순히 `Agent가 몇 분 걸렸는가`만 측정하지 않는다.

최소한 다음을 나눠 본다.

| 항목 | 의미 |
| --- | --- |
| 에이전트 실행시간 | 클라우드 작업 자체 실행시간 |
| 대기시간 | 실행 시작 전 대기시간 |
| 개발자가 다른 일을 하지 못하고 기다리는 시간 | 개발자가 실제로 멈춘 시간 |
| 검토 시간 | 결과 확인과 승인에 사용한 시간 |
| 재시도 시간 | 실패 후 재실행에 사용한 시간 |
| 로컬 자원 점유 | 로컬 CPU / RAM / 디스크를 무거운 작업이 점유한 정도 |

모든 팀에 같은 목표값을 적용하지 않는다.

중요한 것은 클라우드 도입 전후에 **개발자의 전체 작업 흐름이 어떻게 바뀌었는가**다.

---

## 12. 이 장에서 기억할 구조

```text
Developer / Local Agent
        |
     Task Split
        |
        +------ Local 작업 계속
        |
     Cloud Worker
        |
 Build / Test / Independent Work
        |
      Evidence
        |
       Review
```

클라우드 에이전트의 가치는 에이전트 한 명의 응답시간만으로 평가할 수 없다.

독립 실행환경으로 로컬 자원을 분리하고, 사람이 기다리지 않아도 되는 작업을 비동기로 보내고, 독립 작업을 병렬 실행하는 데 의미가 있다.

다음 장에서는 이 가치를 실제 작업에 적용하기 위해 `Local / Cloud / Hybrid / Runner-first` 실행 위치 결정 기준을 만든다.