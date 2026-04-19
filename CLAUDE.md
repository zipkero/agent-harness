# agent-harness — Agent Instructions

에이전트가 이 저장소에서 작업할 때 따라야 할 규칙. 프로젝트 개요·아키텍처는 `READMD.md`, 검증 경계는 `PLAN.md`, 구현 결정값은 `IMPLEMENT.md`를 본다.

## Coding Rules

- 모든 공개 함수 첫 인자 `context.Context`
- 모든 에러는 `HarnessError(Class + Stage)` 형태. Class는 TRANSIENT / PERMANENT / SYNTAX 중 하나
- 에러는 발생 지점에서만 생성한다. 상위 레이어는 그대로 전파 — 재래핑 금지
- `errors.Is` / `errors.As` 체인을 지원하도록 wrap 한다
- 경로는 내부에서 `/` 기준으로 통일하고, 파일 접근 시점에만 OS 경로로 변환
- 로깅은 `slog`. `--verbose` 플래그가 켜지면 Debug 레벨

## Test Conventions

테스트 함수명/파일 주석에 다음 태그를 명시한다:

- `[U]` Unit — 외부 의존 없음
- `[M]` Mock — httptest 등 fake 의존
- `[I]` Integration — 실제 파일/프로세스 사용

## Dependency Policy

`IMPLEMENT.md`의 외부 의존성 목록에 등록된 패키지만 사용한다. 새 의존성을 추가하려면 사유를 `IMPLEMENT.md`에 기재한 뒤 등록한다.

## Document Chain

문서 책임 분리:

| 문서 | 역할 |
|---|---|
| `READMD.md` | 프로젝트 개요, 설계 철학, 불변 모델(state machine / 에러 분류 체계 등) |
| `PLAN.md` | 완성 판정의 검증 경계 (무엇이 동작해야 하고 어떻게 관측하는가) |
| `IMPLEMENT.md` | 구현 구조 + 결정값 (config 스키마, 임계값, 템플릿) |
| 코드 | 최종 진실 |

진행 순서: `READMD.md` 숙지 → `PLAN.md`의 Task 단위로 구현 → verifier 승인 시 해당 Task 앞에 `✓` 마커 prepend.

## Phase Progression

- `PLAN.md`의 Task 를 Phase 내에서 한 개씩 순서대로 진행
- 각 Phase 첫 Task("회귀 검증")가 통과해야 그 Phase의 나머지 Task 진행
- Phase 통합 검증이 `✓` 를 받기 전까지 다음 Phase 진입 금지
- verifier reject 시 해당 Task의 `✓` 를 제거 (있으면) 후 재작업

## Scope Discipline

- 요청된 파일·섹션 외 수정 금지
- 리팩토링·테스트 보강은 별도 요청 시에만
- 새로 만들기보다 기존 파일 수정 우선
- 필요해 보여도 요청 범위를 벗어난 변경은 제안만, 적용은 하지 않음
