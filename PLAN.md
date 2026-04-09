# AI Agent Harness 커리큘럼

---

# 프로젝트

agent-harness (Go CLI)

---

# 구성 요소 (9개) → 챕터 매핑

| 컴포넌트 | 담당 챕터 | 역할 |
|---|---|---|
| Context | CH04 | context 구성 및 압축 |
| Flow | CH13 | plan → apply → verify 실행 흐름 orchestrator |
| Control | CH08 | abort / pause / override 정책 |
| Verify | CH07 | 검증 및 상태 전이 |
| Retry | CH07 | error 분류 및 재시도 |
| Cost | CH02(구조) + CH12(로직) | token budget, 모델 분리 |
| Parse | CH03 | 응답 파싱 |
| Plan Gate | CH05 | 실행 전 검증 및 위험 필터 |
| Observability | CH11 | 상태 추적 및 로깅 |

---

# Phase 구분

| Phase | 챕터 | 목표 | 마일스톤 |
|---|---|---|---|
| Phase 1: Core | CH01 → CH02 → CH03 → CH04 → CH05 | API 연결 + 파싱 + context + plan 생성까지 동작 | plan.md 생성 확인 |
| Phase 2: Engine | CH06 → CH07 → CH08 | 코드 적용 + 검증 + 정책 실행 | 단일 파일 패치 적용 → 테스트 통과 |
| Phase 3: Integration | CH09 → CH10 → CH11 → CH12 → CH13 | 전체 파이프라인 통합 + CLI | `agent-harness run` 단일 요청 end-to-end 동작 |
| Phase 4: Extension | CH14 → CH15 | Multi-Agent + 확장 기능 | MVP 이후 |

---

# 전체 아키텍처 흐름

```
[사용자 요청]
     │
     ▼
[Context Builder] ──→ CLAUDE.md / 파일 선택 / 압축
     │
     ▼
[Plan 생성] ──→ Plan Gate (검증 / 위험 필터)
     │
     ▼
[Claude API 호출] ──→ [Parser] ──→ 구조화된 출력
     │
     ▼
[Apply Engine] ──→ unified diff 적용 / rollback
     │
     ▼
[Verify / Retry] ──→ test / lint / 상태 전이
     │
     ▼
[Hook] ──→ pre/post 정책 실행
     │
     ▼
[Observability] ──→ timing / token / retry 기록
     │
     ▼
[Cost Control] ──→ budget 초과 시 abort 또는 축소
```

---

# 테스트 전략

| 레이어 | 대상 | 방법 |
|---|---|---|
| Unit | Parser, Gate, ErrorClassifier 등 순수 함수 | 고정 입력 → 고정 출력, LLM 없이 |
| Mock LLM | API client, Context Builder | golden file로 LLM 응답 대체 |
| Integration | Flow orchestrator, Apply Engine | 실제 파일 시스템 + 임시 디렉터리 |

- LLM 실제 호출 테스트는 CI에 포함하지 않음 (비용 + 비결정성)
- 임시 디렉터리는 `t.TempDir()` 사용, 테스트 간 파일 공유 금지

---

# Go 버전 요구사항

Go 1.21 이상 필수

- `context.WithoutCancel` (CH13 Apply shutdown 처리)
- `slices.Clone`, `slices.Sorted` (CH11, CH14)
- `errors.Join` (CH01 Config.Validate)

---

# 전역 Config 구조

단일 config.yaml 파일 기준. 각 챕터는 자신의 섹션만 추가.

| 섹션 | 담당 챕터 | 주요 필드 |
|---|---|---|
| api | CH02 | key, model, max_tokens |
| gate | CH05 | deny_paths, deny_ops, on_violation |
| context | CH04 | deny_context_paths, max_tokens |
| budget | CH12 | max_tokens, on_exceed, max_rpm, max_tpm |

---

# 인터페이스 설계 원칙

- 인터페이스는 사용하는 쪽(caller)에서 정의
- stub을 먼저 정의하고 나중에 실제 구현으로 교체
- Go implicit interface 활용 — 구현체가 인터페이스 정의보다 먼저 존재해도 무방

| 인터페이스 | 정의 위치 | stub | 실제 구현 |
|---|---|---|---|
| HookRunner | apply 패키지 (CH06) | NoopHook (CH06) | PolicyHook (CH08) |
| LLMClient | flow 패키지 (CH13) | MockClient | APIClient (CH02) |
| Verifier | flow 패키지 (CH13) | AlwaysPassVerifier | TestRunner (CH07) |
| ContextBuilder | flow 패키지 (CH13) | FixedContextBuilder | RealContextBuilder (CH04) |
| ResponseParser | flow 패키지 (CH13) | — | ClaudeParser (CH03) |

---

# 에러 타입 설계

- `HarnessError` — 모든 에러의 공통 래퍼 (Class + Stage + Wrapped)
- ErrorClass: `TRANSIENT` (재시도) / `PERMANENT` (즉시 abort) / `SYNTAX` (retry + hint)
- 에러 생성은 발생 지점에서만, 상위 레이어는 그대로 전파
- CH03, CH07, CH10에서 공통 사용

---

# CHAPTERS

---

## CH01. 하네스의 실체

**상태**: `TODO`

**목표**: 프로젝트 기반 구조 확립 — 에러 타입, Config 로드/검증, 인터페이스 설계 원칙

**산출물**:
- `harness/` 패키지: HarnessError, ErrorClass 정의
- `config/` 패키지: Config 구조체, YAML 로드, Validate() 패턴
- 전역 에러 타입이 전 챕터에서 일관되게 사용 가능한 상태

**핵심 결정**:
- sentinel error는 단순 상태(EOF 등)에만, 분기 필요한 에러는 HarnessError로 래핑
- Config.Validate()는 errors.Join으로 각 섹션 검증을 집계
- 에러 래핑 시 `%w` 사용 — errors.Is/As 체인 유지

**완료 기준**:
- [ ] 9개 컴포넌트의 역할과 실행 순서를 설명할 수 있다
- [ ] config 로드 시 필수 필드 누락이 명확한 에러로 반환된다
- [ ] 각 config 섹션에 Validate() 메서드가 존재한다

---

## CH02. Claude API 기본

**상태**: `TODO`

**선행**: CH01

**목표**: Claude API 단일 요청/응답 동작 확인 + 테스트 인프라 구축

**산출물**:
- `api/` 패키지: Client, CallOption, Response 구조체
- `testdata/api/` 디렉터리: golden file fixtures
- Mock HTTP 서버 헬퍼 (전 챕터에서 재사용)

**핵심 결정**:
- 모든 공개 함수의 첫 인자는 `ctx context.Context` — 여기서 확립, 이후 전 챕터 적용
- CallOption에 BudgetTokens 슬롯을 미리 확보 (CH12에서 로직 채움)
- API key 로드 순서: 환경변수 우선 → config 파일 fallback

**완료 기준**:
- [ ] API key 로드 → 단일 요청 → 응답 수신 동작
- [ ] 4xx / 5xx 에러 구분 처리
- [ ] ctx cancel 시 진행 중인 HTTP 요청 중단
- [ ] Mock HTTP 서버가 fixture 파일 기반으로 응답 반환
- [ ] 호출 횟수 카운팅으로 retry 동작 검증 가능

---

## CH03. Parsing + 호출 안정화

**상태**: `TODO`

**선행**: CH02

**목표**: LLM 응답 파싱 + API 호출 안정성 확보

### Part 1. 응답 파싱 + 신뢰 경계

**산출물**:
- `parser/` 패키지: ClaudeParser (ResponseParser 인터페이스 구현)

**핵심 결정**:
- Parser를 인터페이스로 분리 — CH15에서 OpenAIParser 등 추가 시 Flow 변경 없음
- LLM 출력은 외부 입력과 동일한 신뢰 수준으로 처리
- 허용된 필드 외 unknown field 거부, 출력 크기 상한 적용

**포함 범위**:
- structured output 파싱
- fallback parsing (markdown strip, JSON fragment 추출)
- 스트리밍 응답(SSE) 청크 수신 + 누적 파싱
- 스트리밍 중단 시 처리

**완료 기준**:
- [ ] markdown 코드 블록으로 감싼 JSON을 올바르게 추출
- [ ] structured output 파싱 실패 시 fallback 동작
- [ ] 스트리밍 청크 누적 → 완성된 출력 반환
- [ ] 스트리밍 중단 시 수신한 청크 기준 에러 반환
- [ ] 허용 스키마 외 필드 포함된 출력 거부

### Part 2. 호출 안정화

**핵심 결정**:
- 에러 분류: TRANSIENT(5xx, timeout) → retry / PERMANENT(4xx) → abort / SYNTAX(파싱 실패) → retry+hint
- CH01의 HarnessError를 그대로 사용, 새 에러 타입 정의하지 않음

**완료 기준**:
- [ ] TRANSIENT 에러에서만 retry/backoff 동작
- [ ] PERMANENT 에러에서 retry 없이 즉시 에러 반환
- [ ] 429 수신 시 Retry-After 헤더 파싱 후 대기 재시도
- [ ] 에러가 HarnessError로 래핑 (Stage + Class 포함)

---

## CH04. Context Builder + Selective Context

**상태**: `TODO`

**선행**: CH03

**목표**: 프롬프트 구성 — CLAUDE.md 로드, 파일 선택, context 압축

**산출물**:
- `context/` 패키지: ContextBuilder, FileRelevance, 프롬프트 템플릿

**핵심 결정**:
- 프롬프트는 system / context / task 3개 블록으로 분리
- 압축은 truncation 기반 (LLM 요약 사용 안 함 — 비용/비결정성)
- 압축 우선순위: few-shot 예시 → 관련도 낮은 파일 → CLAUDE.md 규칙 뒷부분
- 파일 관련도: plan target(제거 불가) > 키워드 매칭 > import 관계 > 파일 크기(감점)
- deny_context_paths로 민감 파일 차단 (.env, *secret*, *credential*)
- temperature=0 고정 (diff/plan 생성은 결정론성 필요)

**완료 기준**:
- [ ] CLAUDE.md 로드 → context 포함
- [ ] 관련 파일만 선택하여 context 크기 축소
- [ ] rule 충돌 시 우선순위 기준 해결
- [ ] deny_context_paths 매칭 파일 context 제외
- [ ] retry 시 hint 블록이 task 블록에 추가
- [ ] temperature=0 고정, golden file 테스트로 동일 입출력 검증

---

## CH05. Plan 생성 + Plan Gate

**상태**: `TODO`

**선행**: CH04

**목표**: LLM 응답으로 plan.md 생성 + Gate로 위험 plan 필터링

**산출물**:
- `plan/` 패키지: Plan 구조체, PlanGate

**핵심 결정**:
- Gate 정책은 외부 설정 파일로 분리 (deny_paths, deny_ops, on_violation)
- Gate는 plan 내용을 신뢰하지 않는 전제로 검증
- Prompt Injection 방어: 하네스 소스 대상 plan 거부, 시스템 명령 패턴 차단

**포함 범위**:
- plan.md 구조 (target file, step 분해, done criteria)
- 파일 존재 검증
- 범위 검증
- 위험 작업 필터링

**완료 기준**:
- [ ] LLM 응답으로부터 plan.md 생성
- [ ] deny_paths 파일 대상 plan → gate 거부
- [ ] deny_ops 위반 → on_violation 정책 적용
- [ ] 하네스 소스 파일 대상 plan 거부
- [ ] 시스템 명령 패턴 포함 plan step 차단

---

## CH06. Patch Apply Engine

**상태**: `TODO`

**선행**: CH05

**목표**: unified diff를 실제 파일에 적용 + atomic write + rollback

**산출물**:
- `apply/` 패키지: ApplyEngine, HookRunner 인터페이스, NoopHook

**핵심 결정**:
- 적용 방식: unified diff (AST 기반 function replace 사용 안 함 — comment association 비결정성)
- diff 파싱: go-gitdiff 라이브러리
- 파일 쓰기: atomic replace (같은 디렉터리에 임시 파일 → os.Rename)
- 다중 파일 패치: all-or-nothing (일부 실패 시 전체 rollback)
- AST 역할: syntax 검증 전용, 포매팅은 gofmt 후처리
- Hook 인터페이스는 여기서 stub(NoopHook) 정의, CH08에서 실제 구현

**에러 분류**:
- SYNTAX: diff 파싱 실패, hunk 불일치, AST 검증 실패 → retry+hint
- PERMANENT: 대상 파일 없음, rollback 실패 → abort
- TRANSIENT: 파일 쓰기 실패 (일시적 I/O) → retry

**완료 기준**:
- [ ] unified diff 파싱 → 파일 적용
- [ ] AST syntax 검증 실패 시 apply 거부
- [ ] apply 실패 시 원본 파일 rollback
- [ ] 다중 파일 일부 실패 시 전체 rollback
- [ ] NoopHook이 pre/post 슬롯에 연결
- [ ] 패치 도중 에러 발생해도 원본 파일 무손상
- [ ] rollback 후 임시 파일 잔존 없음
- [ ] Apply 에러가 HarnessError로 래핑 (ErrorClass 포함)

---

## CH07. Verify + Retry State Machine

**상태**: `TODO`

**선행**: CH06

**목표**: apply 결과 검증 + 에러 분류 기반 retry/abort 상태 머신

**산출물**:
- `verify/` 패키지: Verifier, RetryState, error signature 추출

**핵심 결정**:
- 상태 전이: Apply 완료 → Verify → PASS(Done) / FAIL(에러 분류 → retry or abort)
- Abort 조건: retry >= MaxRetry(기본 3) 또는 동일 error signature 2회 연속
- error signature: 파일 경로/라인 번호를 정규화하여 "같은 에러 반복" 판단
- SYNTAX 에러 retry 시 prompt에 hint 추가 (CH03 연계)

**포함 범위**:
- test / lint 실행
- failure classification
- error signature 추출 (정규화 → prefix match)
- retry limit, abort 조건

**완료 기준**:
- [ ] verify 성공 시 Done 전이
- [ ] TRANSIENT 에러 → retry, MaxRetry 초과 시 abort
- [ ] PERMANENT 에러 → 즉시 abort
- [ ] 동일 error signature 2회 연속 → abort

---

## CH08. Hook / Policy (Control 컴포넌트)

**상태**: `TODO`

**선행**: CH06 (HookRunner 인터페이스), CH07

**목표**: CH06의 NoopHook을 실제 PolicyHook으로 교체 + abort/pause/override 정책

**산출물**:
- `hook/` 패키지: PolicyHook, HookConfig, OverridePolicy

**핵심 결정**:
- pre hook: 체인 실행, 하나라도 에러 → 이후 체크 스킵
- post hook: 성공/실패 무관 전부 실행, 에러는 로그만 (cleanup 목적)
- OnViolation: "abort" → PERMANENT 래핑 / "warn" → 로그만
- Override는 CLI 플래그(`--override-policy`)로만 활성화, 코드 하드코딩 금지

**Abort 조건 총정리**:
- pre hook PERMANENT 에러 (apply 전)
- Gate 거부 (CH05)
- budget 초과 (CH12)
- retry >= MaxRetry (CH07)
- 동일 error signature 2회 연속 (CH07)

**완료 기준**:
- [ ] pre hook 체인 실행, 에러 시 이후 체크 미실행
- [ ] post hook 전부 실행 (성공/실패 무관)
- [ ] OnViolation: abort → PERMANENT 래핑
- [ ] OnViolation: warn → 로그만
- [ ] abort 조건 5가지 각각 동작 확인
- [ ] override 플래그 없이는 abort 우회 불가
- [ ] override 활성화 시 slog.Warn 기록

---

## CH09. Skill

**상태**: `TODO`

**선행**: CH01 (HarnessError)

**목표**: LLM 호출 없이 실행 가능한 고정 로직 단위 + 최소 registry

**산출물**:
- `skill/` 패키지: Skill 구조체, Registry, SkillValidationError
- 샘플 skill 2개: `file.read` (멱등), `shell.run` (비멱등)

**핵심 결정**:
- Registry: Register/Get/Execute — 중복 등록 시 패닉
- 입출력 스키마 검증 (호출 전 Input, 호출 후 Output)
- SkillValidationError → PERMANENT (retry 없이 abort)
- Idempotent: false skill 실패 시 retry 금지
- shell.run은 allow-list 기반 (deny-by-default)

**CH09 / CH13 / CH15 역할 분리**:
- CH09: skill 인터페이스 + 검증 + 최소 registry
- CH13: Flow에서 StepTypeSkill dispatch
- CH15: registry 확장 (List() + 실용 skill 추가)

**완료 기준**:
- [ ] skill 등록 및 이름 조회
- [ ] 미존재 이름 조회 시 에러
- [ ] Input 스키마 검증 실패 → SkillValidationError
- [ ] Output 스키마 검증 실패 → SkillValidationError
- [ ] 멱등성 계약 보장 (동일 입력 → 동일 출력)
- [ ] Idempotent: false skill 실패 → retry 없이 abort
- [ ] file.read: 미존재 파일 → PERMANENT
- [ ] shell.run: allow-list 외 명령 → PERMANENT

---

## CH10. Tool Use (Claude API tool_use)

**상태**: `TODO`

**선행**: CH03 (파싱), CH05 (Gate 정책)

**목표**: Claude API tool_use 응답 처리 — 하네스가 tool 실행 후 tool_result 반환

**산출물**:
- `tool/` 패키지: ToolExecutor, ToolCallOption

**핵심 결정**:
- MCP 프로토콜은 범위 밖 — Claude API의 tool_use 블록만 처리
- tool 응답 신뢰 경계: CH03과 동일 원칙 (스키마 검증, 크기 상한)
- credential: CH02의 API key 로드 패턴과 통일
- Idempotent: false tool → 실패 시 PERMANENT
- 파일 접근 tool은 CH05 deny_paths 정책 적용

**완료 기준**:
- [ ] tool 호출 요청 → 실행 → 결과 반환
- [ ] credential 로드 (CH02 패턴 동일)
- [ ] 허용 스키마 벗어난 tool 응답 거부
- [ ] 응답 크기 상한 초과 시 에러
- [ ] timeout 초과 → 취소 + TRANSIENT 분류
- [ ] 비멱등 tool 실패 → abort
- [ ] 파일 접근 tool → deny_paths 준수

---

## CH11. Observability

**상태**: `TODO`

**선행**: CH07 (Verify 단계 필요)

**목표**: 실행 전 단계의 timing, token 사용량, retry, 실패를 추적

**산출물**:
- `observe/` 패키지: SpanRecorder, Span, TokenUsage

**핵심 결정**:
- 외부 라이브러리 없이 span 직접 구현
- Record() 함수로 각 stage를 감싸면 타임라인 자동 수집
- LLM 호출 단계는 TokenUsage 포함, non-LLM 단계는 영값
- slog 레벨: Debug/Info/Error를 stage 결과에 따라 구분
- CH13에서 Flow에 주입하여 전 단계 커버

**완료 기준**:
- [ ] 각 stage 소요 시간 출력
- [ ] 요청별 token usage 기록
- [ ] retry 횟수 + 실패 유형 추적
- [ ] 실행 종료 후 전체 span 요약 출력
- [ ] slog 레벨이 stage 결과에 따라 구분

---

## CH12. Cost Control

**상태**: `TODO`

**선행**: CH02 (BudgetTokens 슬롯), CH11 (TokenUsage 누적)

**목표**: 세션 토큰 예산 + rate limiting + 모델 분리

**산출물**:
- `cost/` 패키지: SessionBudget, RateLimiter

**핵심 결정**:
- 예산 체계: 세션 예산(budget.max_tokens, 전체 실행) vs 호출 예산(CallOption.BudgetTokens, 단일 호출)
- 세션 예산은 SpanRecorder의 TokenUsage 누적으로 추적
- 세션 예산 초과 시 on_exceed 정책: abort 또는 shrink(context 축소 후 재시도)
- Rate limiting: token bucket (RPM) + sliding window (TPM)
- 429 수신 시 Retry-After 기간 동안 rate limiter 일시 중단
- 대기 중 ctx cancel 시 즉시 에러 반환

**완료 기준**:
- [ ] BudgetTokens 초과 시 정책(abort/축소) 동작
- [ ] 작업 유형에 따라 저비용/고성능 모델 분리 호출
- [ ] cache hit 시 API 호출 미발생
- [ ] RPM 초과 시 요청 대기
- [ ] 대기 중 ctx cancel → 즉시 에러
- [ ] 429 수신 시 Retry-After 동안 rate limiter 중단

---

## CH13. 하네스 완성 (Flow 컴포넌트)

**상태**: `TODO`

**선행**: CH01~CH12 전체

**목표**: 전 컴포넌트를 통합하는 Flow orchestrator + CLI 진입점

**산출물**:
- `flow/` 패키지: Flow orchestrator, LLMClient/Verifier/ContextBuilder 인터페이스 정의
- `cmd/` 패키지: CLI 진입점 (run / plan / apply 서브커맨드)

**핵심 결정**:
- Flow = context 구성 → plan → gate → apply → verify 전체 조율
- CLI는 Flow의 얇은 래퍼 — 비즈니스 로직이 CLI에 들어오지 않음
- 서브커맨드: run(전체 실행), plan(plan까지만), apply(기존 plan 적용)
- 공통 플래그: --config, --dry-run, --verbose
- plan step type이 skill이면 LLM 호출 없이 CH09 registry에서 직접 실행

**Graceful Shutdown**:
- SIGTERM/SIGINT → context cancel 전파
- Apply 실행 중에는 cancel 무시 (context.WithoutCancel) → 완료 또는 rollback 후 종료
- Verify는 cancel 가능
- exit code: 0(성공), 1(실패), 2(설정 오류), 130(신호 종료)

**완료 기준**:
- [ ] 단일 요청으로 context → plan → gate → apply → verify 전 흐름 실행
- [ ] 각 단계 실패 시 정의된 정책 동작 (retry/abort/rollback)
- [ ] observability가 전 흐름에 걸쳐 기록
- [ ] skill step → LLM 호출 없이 registry 직접 실행
- [ ] run / plan / apply 서브커맨드 동작
- [ ] --dry-run으로 apply 없이 plan까지만 실행
- [ ] SIGTERM 시 API 요청 취소
- [ ] Apply 중 SIGTERM → Apply 완료/rollback 후 종료
- [ ] exit code 구분 (0/1/2/130)

---

## CH14. Multi-Agent

**상태**: `TODO`

**선행**: CH13 (Flow 완성 필수)

**목표**: 4개 agent 역할 분리 + goroutine 기반 병렬 실행 + 파일 단위 lock

**산출물**:
- `agent/` 패키지: Agent, AgentResult, FileLockRegistry, PipelineInput

**핵심 결정**:
- 4 역할: explorer / planner / implementer / verifier
- goroutine + channel로 결과 수집 (subprocess 없이 Go 네이티브)
- FileLockRegistry: path 기준 mutex, 여러 파일 동시 잠금 시 알파벳 순 정렬로 deadlock 방지
- 순차 파이프라인: Explorer → Planner → Implementer → Verifier
- 부분 실패 정책: fail-fast (plan/apply) vs best-effort (explore)

**완료 기준**:
- [ ] 동일 파일 동시 수정 시 하나가 대기
- [ ] 4개 agent 역할 분리 동작
- [ ] 하나의 agent 에러 → 나머지 ctx cancel 중단
- [ ] 전체 결과 수집 후 다음 단계 진행
- [ ] planner가 explorer 결과를 입력으로 사용
- [ ] fail-fast: 하나 실패 → 전체 중단
- [ ] best-effort: 일부 실패 → 성공 결과만 수집

---

## CH15. 확장

**상태**: `TODO`

**선행**: CH13

**목표**: registry 확장 + PR 자동화 + provider 추상화 + 언어 지원 + 평가 시스템

**산출물**:
- registry.List() 추가
- 실용 skill 등록 (git.commit, file.glob, json.query 등)
- PR 자동화 (Verify 성공 → branch 커밋 → GitHub API → PR 생성)
- 리뷰 루프 (PR 코멘트 → 새 요청 → Flow 재실행 → PR 업데이트)
- LLM Provider 추상화 (api.provider 설정으로 교체)
- LanguagePlugin 인터페이스 (언어별 포매터/검증기 교체)
- Harness Evaluation (task suite + 지표 리포트)

**핵심 결정**:
- Provider 전환 시 Flow, Parser, Context Builder 변경 없음
- provider 전용 옵션은 Extra map[string]any로 수용
- plugin 없는 확장자는 포매팅 없이 그대로 적용
- 평가 지표: Task 성공률, Retry율, Plan-Apply 일치율, Gate 거부율

**완료 기준**:
- [ ] skill registry 확장 + Flow에서 step 실행
- [ ] Verify 성공 후 PR 자동 생성
- [ ] 리뷰 코멘트 → 수정 → PR 업데이트 사이클
- [ ] api.provider 변경만으로 LLM 구현체 교체
- [ ] 새 LanguagePlugin 등록 → 해당 확장자에 포매팅/검증 적용
- [ ] --eval 플래그로 task suite 실행 + 지표 리포트

---

# 핵심 규칙 (챕터 순서 근거)

이 순서는 "막히는 순서" 기준:

1. API 호출 안됨 → CH02
2. 응답 못 씀 → CH03
3. context 이상 → CH04
4. plan 잘못됨 → CH05
5. 코드 깨짐 → CH06
6. 테스트 실패 루프 → CH07
7. 위험 실행 → CH08
8. 반복 작업 → CH09
9. 외부 연결 → CH10
10. 상태 안 보임 → CH11
11. 비용 터짐 → CH12
12. 전체 통합 → CH13
13. 복잡도 증가 → CH14 (CH13 이후)

---

# 결론 (고정 결정)

- Parsing → CH03
- CallOption.BudgetTokens → CH02 구조, CH12 로직
- Plan Gate 정책 → 외부 설정 파일
- Apply Engine → unified diff + AST 검증 전용 + atomic write rollback
- Hook 인터페이스 → CH06 stub, CH08 구현
- Plan → CH05, Apply → CH06, Verify → CH07
- Flow orchestrator → CH13
- 인터페이스 설계 원칙 → CH01 확립, 전 챕터 적용
- 테스트 인프라 → CH02 구축, 전 챕터 재사용
- Rate Limiting → CH12
- Graceful Shutdown → CH13
- Tool 응답 신뢰 경계 → CH10
