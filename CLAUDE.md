# agent-harness

Module: `github.com/zipkero/agent-harness`

## 패키지 구조

| 패키지 | 역할 |
|---|---|
| `cmd/agent-harness/` | CLI 진입점 |
| `internal/config/` | config.yaml 로드 + 검증 |
| `internal/harness/` | HarnessError, 공용 타입 |
| `internal/api/` | LLMProvider 인터페이스 + 공통 타입 |
| `internal/api/claude/` | Claude API 구현체 |
| `internal/parser/` | LLM 응답에서 diff/fullfile 추출 |
| `internal/context/` | 프롬프트 조립 + 토큰 축소 |
| `internal/plan/` | Plan 생성 + PlanGate |
| `internal/apply/` | diff 적용 + atomic write + rollback |
| `internal/apply/atomic/` | atomic file write |
| `internal/verify/` | go test/vet + RetryState |
| `internal/hook/` | pre/post hook 정책 |
| `internal/skill/` | Skill 인터페이스 + Registry (file_read, shell_run) |
| `internal/tool/` | ToolExecutor (tool_use 블록 실행) |
| `internal/observe/` | Recorder (timing, span) |
| `internal/cost/` | SessionBudget + RateLimiter |
| `internal/token/` | 토큰 카운팅 (근사 함수, CJK 보정, Calibrator) |
| `internal/flow/` | 전체 파이프라인 오케스트레이션 |
| `testdata/` | 테스트용 프로젝트, config, API fixture, diff fixture |

## 에러 규칙

- 모든 에러는 `HarnessError(Class + Stage)` — Class: TRANSIENT/PERMANENT/SYNTAX
- 에러 생성은 발생 지점에서만, 상위 레이어는 그대로 전파 (재래핑 금지)
- `errors.Is/As` 체인 지원

## 공통 규칙

- 모든 공개 함수 첫 인자 `context.Context`
- 경로: 내부 `/` 통일, 파일 접근 시 OS 변환
- 로깅: `slog` 사용, `--verbose` → Debug 레벨
- `.harness/`: txlog, audit, lock, backup 저장 (프로젝트 루트 하위)
- 프로젝트 루트: `--root` 플래그 → `.git/` 상위 탐색 → CWD

## 테스트 태그

- `[U]` Unit — 외부 의존 없음
- `[M]` Mock — httptest 등 fake 의존
- `[I]` Integration — 실제 파일/프로세스

## 의존성 정책

PLAN.md 외부 의존성 테이블에 등록된 패키지만 사용. 새 의존성 추가 시 사유 필요.

## Phase 진행 규칙

- PLAN.md 체크리스트를 한 개씩 순서대로 진행
- 이전 체크리스트 항목 완료 전 다음 항목 진행 금지
- 각 Phase 마일스톤의 검증 기준 통과 후 다음 Phase 진행
