# Anthropic 공식 근거 - Infrastructure Noise와 실행 자원

기준일: 2026-09-16

이 문서는 3장 `Cloud Session, Container, Compute와 Token`에서 실행 자원과 LLM 사용량을 분리해 설명할 때 사용하는 공식 근거를 정리한다.

## 공식 자료

Anthropic Engineering, `Quantifying infrastructure noise in agentic coding evals`, 2026-02-05

- https://www.anthropic.com/engineering/infrastructure-noise

## 공식 자료에서 확인되는 내용

Anthropic은 Terminal-Bench 2.0 기반 내부 실험에서 동일한 모델, harness, task set을 사용하더라도 runtime infrastructure configuration에 따라 agentic coding 결과가 달라질 수 있음을 공개했다.

공개된 실험에서는 가장 제한적인 구성과 가장 여유 있는 구성 사이에 성공률 6 percentage points 차이가 발생했다.

또 resource ceiling을 완화했을 때 infrastructure error rate가 5.8%에서 2.1%로 감소한 사례를 공개했다.

이 수치는 Anthropic의 특정 benchmark와 infrastructure 구성에서 나온 실험 결과다. 일반 Cloud Agent의 성능 기대값으로 사용하지 않는다.

## 본문에서 사용할 원칙

본문에서는 다음 수준으로만 일반화한다.

> Agentic coding에서 실행환경은 수동적인 배경이 아니라 작업 성공 여부에 영향을 줄 수 있는 실행 조건이다.

> CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.

> 모델 성능과 실행 자원은 분리해서 관찰해야 한다.

## 사용하지 않을 일반화

다음은 본문 규칙으로 고정하지 않는다.

- 특정 CPU/RAM 값이 모든 Cloud Agent Task에 적합하다는 주장
- 특정 resource multiplier를 일반적인 최적값으로 사용하는 것
- 6 percentage points 차이를 다른 제품이나 실제 개발 Task에 그대로 적용하는 것
- 5.8% → 2.1% infrastructure error 변화를 일반적인 개선율로 사용하는 것

제품별 vCPU, RAM, Disk, Session 제한과 가격은 별도 Research에서 기준일과 함께 관리한다.
