# 17장 설계 - Cloud가 항상 정답은 아니다

## 장의 목표

Cloud Agent를 사용할 수 있다는 이유만으로 모든 Task를 Cloud로 보내지 않도록, Cloud의 준비 비용·Context 비용·보안/네트워크 제약·Human Steering 요구가 이점보다 큰 상황을 판단하는 기준을 만든다.

핵심 질문:

> 어떤 Task는 왜 Local에 남겨야 하며, Cloud에서 시작한 작업을 언제 Local로 되돌려야 하는가?

---

## 핵심 주장

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

> Cloud Agent는 Local Agent를 대체하지 않는다.

> Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.

---

## 독자가 얻는 것

- Cloud Agent를 사용하지 않아야 할 대표 조건을 설명할 수 있다.
- Internal Network, Large Context, Human Steering, Setup Overhead를 판단할 수 있다.
- Cloud에서 재현할 수 없는 문제를 Local로 되돌릴 수 있다.
- 보안상 Repository/Secret을 Cloud에 제공할 수 없는 경우를 분리할 수 있다.
- 작은 수정의 Cloud overhead를 판단할 수 있다.
- 동일 파일/Schema를 다수 Agent가 수정하는 경우 병렬화를 중단할 수 있다.
- Cloud Task 중단과 Local Fallback을 정상 Workflow로 설계할 수 있다.

---

# 절 구성

## 17.1 Cloud 사용 자체가 목표가 아니다

잘못된 목표:

```text
Cloud Agent 사용률 최대화
```

권장 목표:

```text
전체 개발 Lead Time 감소
Developer Blocking Time 감소
검증 품질 유지
비용 통제
```

Cloud가 이 목표에 도움이 되지 않으면 Local을 선택한다.

---

## 17.2 너무 작은 Task

예:

```text
문구 1줄 수정
검증 10초
```

Cloud Overhead:

- Worker start
- checkout
- Context load
- branch/commit/PR
- Review

Task Work보다 준비 비용이 더 크면 Local이 낫다.

절대적인 `몇 분 이하` 규칙은 두지 않고 프로젝트에서 측정한다.

---

## 17.3 매우 큰 Context가 필요한 Task

예:

```text
전체 인증/학생/권한/결제/DB 구조를 함께 이해하고
새 Architecture를 설계
```

특징:

- Repository 전역 탐색
- 사람과 반복 대화
- 요구사항 변경
- 설계 선택지 비교

이런 작업은 Local에서 Developer와 Agent가 함께 탐색하는 편이 적합하다.

설계가 결정된 뒤 작은 실행 Task만 Cloud로 보낼 수 있다.

---

## 17.4 Human Steering이 지속적으로 필요한 Task

예:

```text
UI 방향 비교
→ 구현
→ 사람 확인
→ 방향 변경
→ 다시 구현
```

Cloud 비동기 위임의 장점이 줄어든다.

판단 질문:

> 작업 중간에 개발자가 몇 번 개입해야 하는가?

개입이 많으면 Local에 남긴다.

---

## 17.5 Internal Network 의존

대표 사례:

- VPN
- Tibero/Oracle 실제 DB
- HSM
- Internal Jenkins
- 사내 API
- 내부 Git/Nexus

Cloud가 접근할 수 없다면:

```text
Local
```

또는:

```text
Cloud 가능한 부분
→ Local 최종 검증
```

으로 Hybrid화한다.

---

## 17.6 Repository를 Cloud에 제공할 수 없는 경우

보안/계약/정책상:

- Source upload 금지
- 외부 SaaS 금지
- Secret 처리 제한
- 특정 데이터 반출 금지

가 있을 수 있다.

이 경우 제품 기능과 관계없이 Cloud Agent 선택이 불가능할 수 있다.

책에서는 법률/보안 정책 일반론으로 확장하지 않고 Routing Hard Constraint로 다룬다.

---

## 17.7 Cloud에서 재현할 수 없는 문제

예:

- 특정 개발자 장비에서만 발생
- 실제 HSM firmware와 관련
- 내부 네트워크 latency
- Production-only race condition
- 로컬 파일/디바이스 의존

좋지 않은 방식:

```text
Cloud에서 계속 재현 시도
→ Token/Compute 반복 소비
```

권장:

```text
재현 불가 확인
→ Local Fallback
```

---

## 17.8 미커밋 Local State에 의존하는 작업

Cloud는 clean Git state에서 시작하는 것이 일반적이다.

예:

```text
IDE에만 있는 수정
untracked config
local DB state
```

이 상태를 먼저 명시적으로 정리하지 않으면 Cloud Handoff가 불완전하다.

선택:

- Commit/Push 후 Cloud
- 현재 작업은 Local 유지

---

## 17.9 실행환경 준비 비용이 큰 Task

특정 Task 하나를 위해:

- 거대한 image build
- 대규모 dependency download
- 특수 simulator 설치
- 외부 Dataset 준비

가 필요하고 작업 자체는 짧다면 Cloud 이점이 줄어든다.

Prepared Environment가 반복 사용될 수 있는지 함께 본다.

---

## 17.10 동일 파일/Schema 동시 수정

나쁜 병렬화:

```text
Agent A → UserService
Agent B → UserService
Agent C → UserService
```

또는:

```text
Agent A/B → same DB migration sequence
```

Cloud 환경은 격리되어 있어도 Integration Cost가 커진다.

이런 Task는 순차화하거나 재분해한다.

---

## 17.11 요구사항이 아직 결정되지 않은 Task

예:

```text
신규 학생 인증 UX를 어떤 방식으로 할지 정해줘.
```

구현 전에 사람의 결정이 필요한 탐색 Task다.

Local:

```text
Developer + Agent
→ option 비교
→ 결정
→ Task Split
```

그 뒤 Cloud로 구현/검증을 보낸다.

---

## 17.12 Architecture 변경

다음과 같은 Task는 기본적으로 Local 중심이다.

- module boundary 재설계
- DB architecture 변경
- 인증 구조 교체
- cross-system transaction 재설계

Cloud Agent를 조사/후보 생성에 사용할 수는 있지만 비동기 독립 Task처럼 취급하지 않는다.

결정 후 반복 실행 작업만 Cloud로 분리한다.

---

## 17.13 불확실한 Production Incident

예:

```text
"가끔 응답이 느려진다."
```

처음부터 Cloud Agent에게 `수정`을 맡기면 안 된다.

먼저 Local/운영 환경에서:

- 재현
- 로그/Metric 확인
- 범위 축소

후 명확한 Bug Task가 생기면 Cloud로 넘길 수 있다.

---

## 17.14 비용이 검증 가치보다 큰 경우

예:

```text
Agent 5개 Best-of-N
→ 작은 문서 오탈자 수정
```

Cloud Compute/Token/Review 비용이 작업 가치보다 크다.

Task의 위험도와 가치에 맞는 실행 수준을 선택한다.

---

## 17.15 Cloud Task 중단 기준

실행 중 다음 조건이 나오면 중단/재분류를 검토한다.

- 같은 Failure Fingerprint 반복
- Budget 소진
- 예상보다 Scope 크게 증가
- Internal dependency 발견
- Repository 추가 5개 필요
- 요구사항 불명확
- 사람 판단이 반복적으로 필요
- Cloud 재현 불가

---

## 17.16 Local Fallback Decision

```text
Cloud Task
   ↓
문제 발생
   ↓
Cloud에서 독립 해결 가능?
   ├─ YES → Context/Retry Budget 내 계속
   └─ NO
       ↓
     Local Fallback
```

Fallback 시 전달:

- current commit
- failed command
- failure summary
- artifacts
- attempted fixes

Cloud에서 한 작업을 버리지 않고 Evidence를 Local로 가져온다.

---

## 17.17 Routing 역판단표

Cloud에 불리한 신호:

```text
Internal Network Required
Large Cross-module Context
Frequent Human Steering
Non-reproducible Failure
Tiny Task
High Shared-file Conflict
Policy Restriction
```

한 항목이 Hard Constraint라면 다른 장점이 많아도 Local이 될 수 있다.

---

## 17.18 campus-platform 예제

### Local

```text
HSM 장애 분석
Tibero-specific behavior
새 인증 Architecture
한 줄 운영 설정 수정
```

### Cloud

```text
AuthService 재현 가능한 Bug
Unit/Integration/E2E
Docker Build
독립 Refactoring
```

### Hybrid

```text
Migration 작성/일반 검증 → Cloud
Tibero 실검증 → Local
```

---

# 좋은 사례와 나쁜 사례

## Cloud 사용률 목표

좋지 않은 방식:

```text
모든 Task Cloud로 보내기
```

권장:

```text
Task 특성에 맞게 Routing
```

## 재현 불가인데 Retry

좋지 않은 방식:

```text
Cloud에서 계속 Agent Retry
```

권장:

```text
Local Fallback
```

## Architecture Task를 비동기 위임

좋지 않은 방식:

```text
"전체 Architecture 개선해"
→ Cloud Agent
```

권장:

```text
Local 설계
→ 작은 Cloud 실행 Task
```

---

# 필요한 그림

1. Cloud 부적합 조건 Decision Tree
2. Cloud Task Stop / Local Fallback
3. Task Size vs Overhead
4. Large Context vs Small Cloud Task
5. Hybrid Fallback

---

# 체크리스트

Cloud 보내기 전:

- Scope가 명확한가?
- 독립 검증 가능한가?
- Internal Network가 필요한가?
- Context가 과도하게 큰가?
- Human Steering이 잦은가?
- Task보다 Cloud Overhead가 큰가?
- Git으로 입력/결과를 전달 가능한가?

---

# 앞뒤 장 연결

16장:
정상적인 Local+Cloud 성공 Workflow

17장:
그 Workflow를 사용하지 않거나 중단해야 하는 조건

18장:
현재 범위 이후의 발전 방향

---

# 의도적으로 다루지 않을 내용

- 보안 정책 세부 법률 해석
- Enterprise Network 구축
- 제품별 Repository 정책 비교
- Agent Security Platform

---

# 장의 결론 메시지

> Cloud Agent를 잘 사용하는 능력에는 Cloud를 쓰지 않을 때를 아는 것도 포함된다.

> Cloud에 보낼 수 있다는 것과 Cloud에 보내는 것이 유리하다는 것은 다르다.

> Local Fallback은 실패가 아니라 Routing의 일부다.
