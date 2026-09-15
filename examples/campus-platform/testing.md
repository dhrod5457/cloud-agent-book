# campus-platform Testing Strategy

## 목표

테스트 전략의 핵심 목표는 Agent가 실제 운영 인프라에 접근하지 못하더라도 변경 결과를 스스로 검증할 수 있게 하는 것이다.

테스트는 단순한 품질 보증 수단이 아니라 Agent 실행 가능성을 보장하는 프로젝트 인터페이스로 본다.

## 테스트 계층

### Unit Test

대상:

- 도메인 정책
- 값 객체
- 입력 검증
- 순수 계산 로직

외부 시스템을 사용하지 않는다.

속도가 빠르고 Agent가 변경 직후 즉시 실행할 수 있어야 한다.

### Integration Test

대상:

- MyBatis Mapper
- Transaction
- DB schema
- Redis Adapter
- Kafka Adapter

기본적으로 Testcontainers를 사용한다.

로컬 개발자 PC에 별도 DB, Redis, Kafka가 설치되어 있다고 가정하지 않는다.

### Contract Test

대상:

- 외부 학사 API
- Push Provider
- 내부 API

Mock HTTP Server 또는 명시적인 Fake Adapter를 사용한다.

외부 서비스의 실제 응답 구조와 내부 Adapter 계약이 어긋나는지 검증한다.

### Architecture Test

대상:

- 모듈 의존성 방향
- 금지된 package 참조
- adapter 침투
- common 역의존

문서에만 존재하는 아키텍처 규칙 중 기계적으로 검증 가능한 항목을 코드로 옮긴다.

### E2E Test

전체 흐름을 검증하지만 범위를 최소화한다.

예:

```text
학생 조회
→ 출결 등록
→ 출결 이벤트 발행
→ 알림 요청 생성
```

모든 외부 시스템을 실제 운영 환경과 연결하는 테스트는 E2E의 기본 정의로 두지 않는다.

## 외부 시스템별 기본 전략

### PostgreSQL

- Testcontainers 사용
- migration 실제 적용
- Mapper/Transaction 검증

### Redis

- Redis Testcontainer 사용
- 단순 도메인 테스트에서는 Fake 또는 비활성화 가능

### Kafka

- Kafka Testcontainer 또는 테스트 목적에 맞는 대체 구현 사용
- Unit Test에서는 이벤트 발행 Port를 Fake로 교체

### HSM

기본 테스트에서는 실제 HSM을 요구하지 않는다.

```text
HsmPort
  ├─ FakeHsmAdapter
  └─ RealHsmAdapter
```

Fake는 성공, 실패, timeout, 잘못된 응답 등 주요 시나리오를 재현할 수 있어야 한다.

실제 HSM 검증은 Local Agent 또는 제한된 Integration 환경에서 별도로 수행한다.

### 외부 HTTP API

- Mock HTTP Server
- 고정된 contract fixture
- timeout, 4xx, 5xx, malformed response 테스트 포함

## Fake와 Testcontainers의 역할 분담

Testcontainers는 실제 제품의 프로토콜과 동작을 확인해야 할 때 사용한다.

예:

- SQL
- Redis command
- Kafka producer/consumer

Fake Adapter는 다음 상황에서 사용한다.

- 상용 장비가 필요한 경우
- 네트워크 접근이 제한된 경우
- 라이선스가 필요한 경우
- 테스트에 실제 시스템이 지나치게 느리거나 불안정한 경우
- 실패 시나리오를 의도적으로 제어해야 하는 경우

즉 모든 외부 의존성을 Container로 만들거나 모든 것을 Mock으로 처리하는 방식을 피한다.

## 테스트 실행 인터페이스

최종적으로 사람과 Agent 모두 동일한 진입점을 사용한다.

예정 인터페이스:

```text
./scripts/test.sh
./scripts/verify.sh
```

세부 구현은 이후 장에서 결정한다.

`verify`는 최소한 다음 결과를 하나의 종료 코드로 반환해야 한다.

```text
format/lint
compile
unit test
integration test
architecture test
secret/security check
change-scope validation
```

성공은 exit code `0`, 실패는 non-zero를 원칙으로 한다.

## 빠른 검증과 전체 검증

Agent가 작은 변경마다 모든 테스트를 수행하면 피드백 시간이 지나치게 길어질 수 있다.

따라서 두 단계 검증을 설계한다.

### Fast Verification

작업 중 반복 실행한다.

- compile
- affected unit tests
- affected architecture rules
- 필요한 소수 integration tests

### Full Verification

작업 완료 판단과 PR Gate에서 실행한다.

- 전체 unit tests
- 전체 integration tests
- architecture tests
- security/secret checks
- repository diff validation

작업 완료 조건에는 Full Verification 결과를 사용한다.

## Flaky Test 정책

Agent가 실패 원인을 잘못 판단하지 않도록 flaky test를 허용 상태로 방치하지 않는다.

원칙:

- 자동 재시도로 성공한 테스트도 기록한다.
- 동일 테스트 반복 실패는 별도 결함으로 취급한다.
- 외부 네트워크 의존 테스트는 기본 verify 경로에서 제외한다.
- 시간, 랜덤 값, 비동기 처리에는 결정 가능한 테스트 경계를 둔다.

## 테스트 데이터

테스트 데이터는 재현 가능해야 한다.

- 현재 운영 DB 데이터를 전제로 하지 않는다.
- 개인정보를 저장소에 포함하지 않는다.
- fixture 생성 규칙을 명확히 한다.
- 테스트 ID와 시간 값을 필요하면 고정한다.

## Agent 작업 완료 조건과 연결

기능 구현 작업은 다음 조건만으로 완료되지 않는다.

```text
코드 작성 완료
```

최소 완료 조건은 다음과 같다.

- Acceptance Criteria 충족
- 신규 또는 수정 테스트 포함
- Fast Verification 성공
- Full Verification 성공
- Architecture Rule 위반 없음
- Secret 포함 없음
- 허용 범위를 벗어난 파일 변경 없음

이 기준은 Task Contract의 `Verification` 항목과 직접 연결한다.
