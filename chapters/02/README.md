# Chapter 2 Planning Notes

현재 2장의 정식 설계는 `plan.md`다.

장 제목:

`Local Agent와 Cloud Agent`

`execution-platform.md`는 이전 Phase 5에서 작성한 Cloud 실행 플랫폼 확장 설계 자료다.

Cloud Agent 중심 목차 재정렬 이후 이 파일 전체를 2장 본문에 넣지 않는다.

중요:

- `execution-platform.md` 내부에 남아 있는 과거 장 번호나 선행/후속 장 참조는 현재 18장 목차의 정식 참조가 아니다.
- 장 번호와 범위 판단은 `planning/toc.md`와 각 `chapters/NN/plan.md`를 우선한다.
- Phase 6 본문에는 해당 주제의 현재 장으로 필요한 내용만 가져온다.

현재 교차 장 배치:

- Compute / Token 분리 → 3장
- 장시간 작업 / 병렬성 → 4장
- 작은 Task / Progressive Context → 7장
- Result Gateway / Budget → 8장
- Prepared Environment / Cache → 9장
- Runner-first / Agent-on-failure → 10장
- Isolation / Git Handoff → 11장
- 병렬 Worker / Fan-out / Context 중복 → 12장
- Local / Cloud Handoff → 13장
- Event-driven Agent → 14장

따라서 Phase 6 본문 작성 시 2장은 `plan.md`를 기준으로 하고, `execution-platform.md`에서는 해당 장에 필요한 부분만 참고한다.

Agent Platform 일반론으로 확장된 내용은 `planning/future-topics.md`의 범위 판단을 우선한다.
