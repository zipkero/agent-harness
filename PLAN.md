# agent-harness

Go CLI. LLM이 생성한 코드 변경을 안전하게 적용·검증·반복하는 실행 시스템.
Claude API가 1순위 구현 대상이지만, LLMProvider 인터페이스를 통해 다른 provider(OpenAI 등)도 교체 가능한 구조.

---

## 프로젝트 구조

```
agent-harness/
├── cmd/agent-harness/main.go
├── internal/
│   ├── config/
│   ├── harness/
│   ├── api/           # LLMProvider 인터페이스 + 공통 타입
│   │   └── claude/    # Claude API 구현체 (SSE, wire 타입)
│   ├── parser/
│   ├── context/
│   ├── plan/
│   ├── apply/
│   │   └── atomic/
│   ├── verify/
│   ├── hook/
│   ├── skill/
│   ├── tool/
│   ├── observe/
│   ├── cost/
│   ├── token/         # 토큰 카운팅 공용 유틸 (근사 함수, CJK 보정, 보정 계수)
│   └── flow/
├── testdata/
├── go.mod
└── config.yaml
```

---

## 아키텍처 흐름

```
[사용자 요청]
     ▼
[Context Builder] → CLAUDE.md / 파일 선택 / 압축
     ▼
[Plan 생성] → Plan Gate (위험 필터)
     ▼
[Cost Control] → budget 확인 / rate limit
     ▼
[Claude API] → [Parser] → 구조화된 출력
     │                        │
     │  ┌ tool_use → [Tool 루프] → tool_result ─┐
     │  └───────────────────────────────────────┘
     ▼
[Hook Pre] → 정책 검사
     ▼
[Apply Engine] → diff 적용 / atomic write / rollback
     ▼
[Format] → gofmt
     ▼
[Verify] → test / lint
     ├─ PASS → [Hook Post] → 다음 step
     └─ FAIL → [Rollback] → retry 또는 abort
```

---

## 외부 의존성

| 패키지 | 용도 | 시점 |
|---|---|---|
| `gopkg.in/yaml.v3` | config YAML 파싱 | Phase 1 |
| `github.com/bmatcuk/doublestar/v4` | `**` glob 패턴 | Phase 1 |
| `github.com/bluekeyes/go-gitdiff` | unified diff 파싱 | Phase 2 |
| `golang.org/x/sys/windows` | Windows atomic rename fallback | Phase 2 |
| `github.com/sabhiram/go-gitignore` | `.gitignore` 패턴 매칭 (경량 단일 목적 라이브러리) | Phase 1 |
| `github.com/spf13/cobra` 또는 표준 `flag` | CLI | Phase 3 |

---

## 공통 규칙

- **에러**: HarnessError(Class + Stage + Wrapped). TRANSIENT(재시도) / PERMANENT(abort) / SYNTAX(retry+hint)
- **로깅**: `slog`. `--verbose` → Debug
- **경로**: 내부 `/` 통일. 파일 접근 시 OS 변환
- **ctx**: 모든 공개 함수 첫 인자 `context.Context`
- **`.harness/`**: txlog, audit, lock 저장
- **테스트 태그**: `[U]` Unit / `[M]` Mock(httptest 등 fake 의존) / `[I]` Integration(실제 파일/프로세스)
- **`.gitignore` 파싱**: `github.com/sabhiram/go-gitignore` 사용 (외부 의존성 테이블 참조)
- **프로젝트 루트**: 탐색 순서 — `--root` 플래그 → CWD에서 상위로 `.git/` 탐색 → CWD. `config.yaml`, `.harness/`, `context.scan_root` 모두 이 루트 기준 상대 경로

**`.harness/` 디렉터리 구조** (필수 파일. 구현 진행 시 추가)

```
.harness/
├── lock.json          # 실행 중 잠금. {"request_id": "...", "pid": ..., "started_at": "RFC3339"}
#                        request_id: UUID v4 (crypto/rand 기반). CLI 실행마다 생성
#                        staleness 판정: PID 생존 확인 + started_at가 config `flow.lock_stale_minutes`(기본 60분) 초과 → stale lock 삭제 후 재획득
#                        PID 확인: Linux/macOS → os.FindProcess+signal 0, Windows → windows.OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION) 성공 여부
├── txlog.jsonl        # 트랜잭션 로그. 한 줄 = 한 이벤트
├── backup/            # apply 전 원본 파일 백업. rollback 시 복원 원본. committed 후 삭제
│   └── {relative_path}  # 원본 파일의 프로젝트 루트 기준 상대 경로 유지
└── audit/             # audit.enabled=true 시 LLM 호출 기록
    └── {timestamp}.jsonl
```

txlog 이벤트 형식:
```json
{"event": "prepared", "files": ["a.go","b.go"], "ts": "RFC3339"}
{"event": "file_done", "file": "a.go", "backup": ".harness/backup/a.go", "ts": "RFC3339"}
{"event": "committed", "ts": "RFC3339"}
```

audit 레코드 형식:
```json
{"ts": "RFC3339", "model": "...", "input_tokens": 0, "output_tokens": 0, "request": {...}, "response": {...}}
```

---

# Phase 1: Core — plan 생성까지

**마일스톤**: Plan 생성 파이프라인 통합 테스트 통과 (CLI는 Phase 3-3에서 추가)

**검증 기준** (통합 테스트 `TestPlanGeneration`, mock HTTP 서버 사용):
- 입력: request `"Add a Hello function to pkg/greet/greet.go that returns string"` + `testdata/project/`
- mock 서버: `testdata/api/plan_response.json` fixture를 반환하도록 구성 (실제 API 호출 없음)
- 성공 조건:
  1. 반환된 Plan 구조체가 schema 준수 (`goal` + `steps[]` 존재)
  2. steps에 target_files로 `pkg/greet/greet.go` 포함
  3. error nil
- 실패 조건: API key 미설정 시 PERMANENT 에러 반환

---

## 1-1. 기반 구조

> 전 챕터에서 쓰는 에러 타입과 config를 먼저 만든다. 이게 없으면 아무것도 시작 못함.

**에러 시스템** (`internal/harness/`)

- [ ] HarnessError 구조체 + ErrorClass 상수 3종
  - 모든 에러를 Class(TRANSIENT/PERMANENT/SYNTAX) + Stage("parse", "apply", "verify" 등)로 분류
  - `errors.Is/As` 체인 지원해서 상위 레이어가 원본 에러도 꺼낼 수 있게
  - 에러 생성은 발생 지점에서만, 상위는 그대로 전파

```go
// ErrorClass 상수
const (
    TRANSIENT  ErrorClass = "TRANSIENT"  // 재시도 가능 (네트워크, I/O, 5xx)
    PERMANENT  ErrorClass = "PERMANENT"  // 즉시 abort (인증 실패, 파일 없음, 정책 위반)
    SYNTAX     ErrorClass = "SYNTAX"     // retry + hint (파싱 실패, AST 오류, diff 불일치)
)

// Stage 상수
const (
    StageConfig  = "config"
    StageAPI     = "api"
    StageParse   = "parse"
    StagePlan    = "plan"
    StageGate    = "gate"
    StageApply   = "apply"
    StageFormat  = "format"
    StageVerify  = "verify"
    StageHook    = "hook"
    StageTool    = "tool"
    StageBudget  = "budget"
)
```

**Config** (`internal/config/`)

- [ ] Config 구조체 + YAML 로드
  - `yaml.v3`로 파싱. 탐색 순서: `--config` 플래그 → 프로젝트 루트 `config.yaml` → 에러
  - 각 섹션(api, gate, context, verify, budget, skills 등)별 서브 구조체
**config.yaml 필수 필드**

```yaml
version: "1"                    # string, 필수. 설정 스키마 버전

api:
  provider: "claude"            # string, 기본 "claude". LLM provider 선택 ("claude" | "openai" 등. Phase 4-4에서 확장)
  model: ""                     # string, 필수. 기본 모델 (plan 생성 + diff_model 미설정 시 diff 생성에도 사용. e.g. "claude-sonnet-4-20250514")
  base_url: ""                  # string, 선택. API base URL (provider별 기본값: Claude="https://api.anthropic.com", OpenAI="https://api.openai.com"). 테스트/프록시용 오버라이드
  diff_model: ""                # string, 선택. diff 생성용 모델 (미설정 시 api.model 사용)
  diff_context_window: 0        # int, 선택. diff_model의 context window (미설정 시 api.context_window 사용)
  max_retries: 3                # int, 기본 3. API 재시도 횟수
  context_window: 200000        # int, 기본 200000. 모델 context window 크기 (토큰)
  timeout: 300                  # int(초), 기본 300. API HTTP 요청 타임아웃 (스트리밍 응답 포함. 0 = 무제한)

context:
  scan_root: "."                # string, 기본 ".". 파일 탐색 기준 경로
  deny_context_paths: []        # []string, 기본 []. context 제외 glob 패턴 (.env, *.key 등)

gate:
  deny_paths: []                # []string, 기본 []. plan 대상 거부 glob 패턴
  deny_ops: []                  # []string, 기본 []. 금지 작업 (delete, rename 등)
  on_violation: "abort"         # "abort" | "warn", 기본 "abort"

verify:
  timeout: 60                   # int(초), 기본 60. 테스트 타임아웃
  scope: "package"              # "package" | "all", 기본 "package"

budget:
  max_input_tokens: 0           # int, 기본 0(무제한). 입력 토큰 예산
  max_output_tokens: 0          # int, 기본 0(무제한). 출력 토큰 예산
  max_rpm: 0                    # int, 기본 0(무제한). 분당 최대 요청 수
  max_tpm: 0                    # int, 기본 0(무제한). 분당 최대 토큰 수
  on_exceed: "abort"            # "abort" | "shrink", 기본 "abort"

tool:
  max_result_size: 65536        # int(bytes), 기본 64KB. tool_result 최대 크기
  max_tool_rounds: 10           # int, 기본 10. 멀티턴 tool use 최대 라운드 수

skill:
  shell_allow_list: []          # []string, 기본 []. shell.run 허용 명령 prefix (e.g. ["go test", "go vet"]). 명령이 항목으로 시작하면 허용
  shell_timeout: 30             # int(초), 기본 30. shell_run 실행 타임아웃

audit:
  enabled: false                # bool, 기본 false. LLM 호출 프롬프트/응답 JSONL 저장

flow:
  dirty_workdir: "abort"        # "abort" | "stash" | "proceed", 기본 "abort". 미커밋 변경 존재 시 정책
  #                               stash 시 `git stash -u` (untracked 포함, .gitignore 대상 제외). pop conflict → stash 보존 + 에러
  #                               git repo가 아닌 경우: "stash" → PERMANENT 에러, "abort" → dirty 판정 불가이므로 "proceed"로 fallback (경고 로그)
  lock_stale_minutes: 60        # int(분), 기본 60. lock 파일 stale 판정 기준 시간. PID 생존 확인 + 이 시간 초과 시 stale로 삭제
```

- [ ] Validate + 기본값
  - `errors.Join`으로 섹션별 검증 결과 집계해서 한 번에 리포트
  - unknown field → strict 파싱으로 감지 → 경고 로그 후 관대하게 재파싱
  - 없는 선택 필드 → `applyDefaults()`로 기본값
- [ ] version 마이그레이션
  - `version: "1"` 현재 유일. 미래 version은 `migrate_1_to_2` 같은 체인 함수로 대응
  - 미래 version 감지 → PERMANENT 에러
- [ ] `.harness/` 디렉터리 초기화
  - 프로젝트 루트 하위에 생성. `.gitignore` 추가 권장 메시지

**테스트 인프라** (`testdata/`)

- [ ] 테스트용 Go 프로젝트 구성
  - `testdata/project/` 하위에 최소 Go 모듈:
    - `go.mod` (module testproject)
    - `pkg/greet/greet.go` (빈 패키지 또는 최소 함수)
    - `CLAUDE.md` (context 포함 테스트용)
  - `testdata/config/` 하위에 테스트용 config:
    - `valid.yaml` (전 필드 설정)
    - `minimal.yaml` (필수 필드만 — 기본값 적용 테스트)
  - `testdata/api/` 하위에 mock API 응답 fixture:
    - `plan_response.json` (Phase 1 통합 테스트용 plan tool_use 응답. Claude API 응답 형식: `{"id":"...","type":"message","role":"assistant","content":[{"type":"tool_use","id":"...","name":"create_plan","input":{...}}],"stop_reason":"tool_use","usage":{...}}`)
  - `testdata/diffs/` 하위에 diff fixture:
    - `valid_add_function.diff` (정상 함수 추가)
    - `fuzz_offset.diff` (±3줄 범위 fuzz 필요한 diff)
    - `invalid_syntax.diff` (AST 실패용)
    - `context_mismatch.diff` (context line 불일치용)
  - Phase 1-4 통합 테스트(파일 선택, CLAUDE.md 포함)부터 사용
  - Phase 2 이후 diff 적용/검증 테스트에서도 재활용

**테스트**: [U] 필수 필드 누락 에러 / unknown field 경고 / 기본값 적용

---

## 1-2. LLM Provider + Claude 구현

> LLM과 통신하는 유일한 창구. Provider 인터페이스를 여기서 정의해야 상위 레이어가 provider에 비종속.

**LLMProvider 인터페이스** (`internal/api/`)

- [ ] LLMProvider 인터페이스 정의
  - `Call(ctx, []Message, CallOption) → (Response, error)` — 동기 호출. 내부적으로 `stream: false`
  - `CallStream(ctx, []Message, CallOption) → (ResponseStream, error)` — 스트리밍. 내부적으로 `stream: true`. CallOption에 stream 필드 없음 (메서드 선택이 곧 스트림 제어)
  - 하네스 전체가 이 인터페이스에만 의존. provider 구현체는 생성자 주입

```go
// ResponseStream — 스트리밍 응답 소비 인터페이스
type ResponseStream interface {
    // Next: 다음 스트림 이벤트 반환. 스트림 종료 시 io.EOF
    Next() (StreamEvent, error)
    // Response: 스트림 완료 후 최종 Response 반환. Next가 io.EOF 반환 전 호출 시 에러
    Response() (Response, error)
    // Close: 스트림 리소스 해제. 중도 취소 시에도 반드시 호출
    Close() error
}

// StreamEvent — 스트림 이벤트 (provider가 SSE 등을 이 타입으로 변환)
type StreamEvent struct {
    Type string // "text_delta" | "input_json_delta" | "content_block_stop" | "message_delta" | "error"
    // text_delta
    Text string
    // input_json_delta
    PartialJSON string
    // content_block_stop: 확정된 ContentBlock
    Block *ContentBlock
    // message_delta: stop_reason + usage
    StopReason string
    Usage      *TokenUsage
    // error: API 에러 (overloaded, rate limit 등)
    Error      error
}
```

- [ ] 하네스 내부 공통 타입 (provider-agnostic)
  - `Message`, `ContentBlock`, `Response`, `TokenUsage` — 하네스 내부에서만 사용하는 중립적 구조
  - provider 구현체가 자신의 wire 타입 ↔ 공통 타입 변환 책임

```go
// Message — 하네스 내부 공통 메시지
type Message struct {
    Role    string         // "user" | "assistant"
    Content []ContentBlock
}

// ContentBlock — type에 따라 사용 필드가 달라짐
type ContentBlock struct {
    Type  string // "text" | "tool_use" | "tool_result"

    // text
    Text string

    // tool_use (assistant → harness)
    ID    string          // tool call ID
    Name  string          // tool 이름 (e.g. "file_read")
    Input json.RawMessage // tool 입력 JSON

    // tool_result (harness → assistant)
    ToolUseID string // 대응하는 tool_use의 ID
    Content   string // 실행 결과 텍스트
    IsError   bool   // 실행 실패 여부
}

// Response — 하네스 내부 공통 응답
type Response struct {
    ID         string
    Content    []ContentBlock
    StopReason string     // "end_turn" | "tool_use" | "max_tokens"
    Usage      TokenUsage
}

// TokenUsage — 토큰 사용량 (비용 계산 기준)
type TokenUsage struct {
    InputTokens        int
    OutputTokens       int
    CacheCreationTokens int // provider가 지원하는 경우에만 (e.g. Claude prompt caching)
    CacheReadTokens    int
}
```

- [ ] Conversation
  - LLM API가 stateless라 매 호출마다 전체 히스토리 전송해야 함
  - `System` 필드: `messages[]`와 별도로 관리. provider가 API 호출 시 자체 방식으로 전달 (Claude: `system` 파라미터, OpenAI: role "system" 메시지 등)
  - Append 메서드(User/Assistant/ToolResult), Shrink(히스토리 축소), EstimateTokens(토큰 근사 — `internal/token/` 공용 함수 사용)
  - **생명주기**: Flow의 llm step마다 새 Conversation 생성. tool use 멀티턴 루프 중에는 동일 Conversation에 누적. step 간에는 히스토리 미공유 (이전 step 변경 결과만 context로 전달)
- [ ] CallOption
  - Model, MaxTokens, Temperature, BudgetTokens, CacheControl, Tools, ProviderOpts
  - `MaxTokens`: 호출 유형별 기본값 — plan 생성 4096, diff 생성 `min(context_window × 0.4, 16384)`, fullfile fallback `EstimateTokens(원본파일) × 2` (상한 `context_window × 0.5`). Flow가 유형에 따라 설정
  - `BudgetTokens`: Claude extended thinking 용도. `thinking.budget_tokens` 파라미터로 매핑. 0이면 thinking 비활성화. 다른 provider는 ProviderOpts로 격리하거나 무시

**ClaudeProvider** (`internal/api/claude/`)

- [ ] HTTP 클라이언트 + Claude wire 타입
  - `POST /v1/messages`. base URL은 config `api.base_url` (미설정 시 `https://api.anthropic.com`). 타임아웃은 config `api.timeout`
  - `system`은 `messages[]`와 별도 파라미터로 전달 (Claude API 스펙)
  - Claude 전용 wire 타입(claude.Message, claude.Response 등) ↔ 공통 타입 변환
- [ ] SSE 스트리밍 처리
  - `stream: true` 설정. SSE 파싱은 provider 내부에서 완결
  - 이벤트 흐름: `message_start` → (`content_block_start` → `content_block_delta`* → `content_block_stop`)* → `message_delta` → `message_stop`. `error` 이벤트는 어느 시점에든 발생 가능
    - `content_block_delta`: delta type 분기 — `text_delta`(텍스트 누적) / `input_json_delta`(tool 입력 JSON 누적)
    - `content_block_stop`: 해당 블록 확정. `input_json_delta` 누적분은 이 시점에 한 번 JSON 파싱
    - `message_delta`: `stop_reason` + 최종 `usage.output_tokens` 수신. 이거 안 받으면 stop_reason 판단 불가
    - `message_stop`: 스트림 종료 신호
  - `input_json_delta`는 불완전 JSON이므로 문자열 연결만, `content_block_stop` 시 한 번 파싱
  - 청크 경계: `\n\n` 기준 버퍼링, 불완전 라인은 다음 청크와 결합
  - `error` 이벤트: `{"type":"error","error":{"type":"...","message":"..."}}`. overloaded → TRANSIENT(retry), invalid_request → PERMANENT. 수신 시 스트림 즉시 종료
  - 스트림 중단: EOF/timeout 시 완성된 text 블록은 반환, 미완성 tool_use는 폐기 + TRANSIENT 에러. SSE 재연결 없이 새 HTTP 요청으로 전체 재시도 (부분 응답 폐기. Claude API가 서버측 재개를 미지원하므로)
- [ ] API key 로드
  - provider별 환경변수: Claude → `ANTHROPIC_API_KEY`, OpenAI → `OPENAI_API_KEY`. config에 key를 저장하지 않음 (시크릿 유출 방지). 미설정 → PERMANENT 에러
  - provider 인터페이스에 `EnvKey() string` 메서드 포함 — 각 구현체가 자신의 환경변수 이름을 반환
- [ ] 에러 분류 + retry
  - 5xx, timeout → TRANSIENT → retry. 4xx(401, 403 등) → PERMANENT → 즉시 abort
  - 429 → `Retry-After` 헤더 파싱 → 해당 시간만큼 대기 후 재시도
  - exponential backoff: base 1s, max 30s, jitter ±20%. 최대 횟수는 config `api.max_retries`(기본 3)
- [ ] RateLimiter 인터페이스 정의 + NoopLimiter
  - `RateLimiter` 인터페이스는 여기(`internal/api/`)에서 정의: `Wait(ctx) error`, `OnThrottle(retryAfter time.Duration)`
  - 호출 전 `Wait(ctx)`, 429 수신 시 `OnThrottle(retryAfter)`. Phase 1~2는 NoopLimiter 주입
  - Phase 3-2에서 실제 RateLimiter 구현(`internal/cost/`) 시 생성자 주입으로 교체

**테스트 인프라**

- [ ] Mock HTTP 서버 + fixture
  - `httptest.Server` + `testdata/api/` 파일 기반 응답. 호출 횟수 카운팅으로 retry 검증

**테스트**: [M] 요청-응답 / 에러 분류 / retry 동작 / 429 대기 / ctx cancel 중단 / SSE 스트리밍 누적·중단

---

## 1-3. 응답 파싱

> LLM 응답(공통 Response)에서 diff 텍스트와 fullfile 콘텐츠를 추출하는 단계. SSE 등 전송 계층 처리는 provider가 완결하고, parser는 완성된 Response만 받음. LLM 출력은 외부 입력과 동일 신뢰 수준으로 취급.
> **Plan JSON 파싱은 여기서 하지 않음** — Plan은 tool_use 응답의 `ContentBlock.Input`에서 직접 추출 (Phase 1-5).

**ResponseParser** (`internal/parser/`)

- [ ] diff 텍스트 추출
  - 응답에서 unified diff 부분만 분리. 코드 블록(````diff ... ````) strip, 빈 diff 감지
  - 추출만 여기서. 실제 unified diff 파싱(`go-gitdiff`)은 Phase 2 ApplyEngine
- [ ] fullfile 텍스트 추출
  - fullfile fallback 응답에서 코드 블록 strip → 전체 파일 텍스트 추출
- [ ] 신뢰 경계
  - 크기 상한 초과 → 거부
  - 파싱 실패 → SYNTAX 에러 (Stage: "parse/diff" 또는 "parse/fullfile")

**테스트**: [U] diff 추출 / fullfile 추출 / 빈 diff 감지 / 크기 상한 초과 거부. Phase 1 마일스톤에서는 단위 테스트만 — 실제 diff 적용 통합 테스트는 Phase 2

---

## 1-4. Context Builder

> LLM에 보낼 프롬프트를 조립한다. 어떤 파일을 얼마나 넣을지, 토큰 예산 안에서 어떻게 줄일지가 핵심.

**ContextBuilder** (`internal/context/`)

- [ ] 프롬프트 3블록 조립
  - system: 하네스 역할, 출력 형식(unified diff, JSON schema), 금지 행동 → Conversation.System에 저장
  - context: CLAUDE.md 내용, 선택된 파일 내용 → user 메시지 앞부분
  - task: 사용자 요청, plan(있으면), retry hint(있으면) → user 메시지 뒷부분
  - **CLAUDE.md 탐색**: 프로젝트 루트 `CLAUDE.md` → 없으면 무시 (에러 아님). 존재 시 context 블록 최상단에 포함, 축소 대상에서 제외

**System Prompt 템플릿** (step 유형별 분기. 공통 base + 유형별 output 섹션)

```text
# Base (모든 step 공통)
You are a code modification agent. You MUST follow these rules:

## Constraints
- Never modify files outside the provided target list.
- Never execute destructive system commands (rm -rf, curl | sh, etc.).
- Never output secrets, API keys, or credentials.
- If a change is impossible, respond with: {"error": "<reason>"}

## Context
The following project files and instructions are provided for reference.
```

```text
# Plan 생성용 (Phase 1-5에서 사용. tool_use로 출력 강제하므로 Output Format 지시 최소화)
## Output Format
- Use the `create_plan` tool to output your plan. Do not output free text.
```

```text
# Diff 생성용 (Phase 3 llm step에서 사용. tool use 허용 — LLM이 file_read로 파일을 읽어 정확한 diff 생성 가능)
## Tools
- You may use `file_read` and `shell_run` tools to inspect the codebase before generating your diff.
## Output Format
- When you are ready, output ONLY as unified diff (```diff ... ```). No explanations outside diff blocks.
- One diff block per file. Include full file path in the diff header.
- Do NOT mix tool calls and diff output in the same response. Use tools first, then output the diff.
```

```text
# Fullfile Fallback용 (diff 2회 실패 시)
## Output Format
- Output the complete modified file content. No diff, no explanation, only the full file.
- Wrap in a single code block with the appropriate language tag.
```

**Retry Hint 형식** (에러 유형별 LLM에 전달하는 보조 프롬프트)

```text
# diff 적용 실패 (SYNTAX/apply)
Your previous diff failed to apply.
Error: {error_message}
The actual file content around the failing hunk (lines {start}-{end}):
```
{file_snippet}
```
Please regenerate the diff for this file.

# AST/포맷 실패 (SYNTAX/format)
Your previous diff produced invalid syntax.
Error: {error_message}
Please fix the syntax error and regenerate the diff.

# 테스트 실패 (SYNTAX/verify)
Your previous change failed verification.
Command: {test_command}
Output (last 50 lines):
```
{test_output_tail}
```
Please fix the failing test and regenerate the diff.
```
- [ ] 파일 선택
  - 탐색 범위: config `context.scan_root` 기준, `.gitignore` 존중 (`go-gitignore`로 패턴 매칭)
  - 선택 전략 (MVP): 명시적 지정(plan target) → 키워드 매칭(토큰 분리, 경로 매칭) → 초과 시 크기 작은 순
  - `deny_context_paths` 매칭 파일 제외 (.env, 시크릿 등)
- [ ] context 축소 (ContextBuilder 책임: context 블록 관리)
  - 전체 예산: `api.context_window` × 80% (safety margin 20%)
  - 블록별 할당: 고정 블록(system + CLAUDE.md + task) 토큰을 먼저 산출 → 잔여 예산을 context 블록(67%)과 conversation 히스토리(33%)로 배분
  - context 블록 축소 트리거: 할당량 초과 시. 순서: few-shot 예시 제거 → 관련성 낮은 파일 제거 → 남은 파일 앞 N줄만 (잘림 표시 필수)
  - diff_model 사용 시 `api.diff_context_window` 기준으로 동일 비율 적용
- [ ] Conversation 히스토리 축소 (Conversation.Shrink 메서드 — Conversation이 자체 히스토리만 축소)
  - 트리거: Flow가 API 호출 전 `Conversation.Shrink(maxTokens)` 호출. maxTokens는 ContextBuilder가 산출한 히스토리 할당량
  - 순서: 직전 응답 유지 → 오래된 tool 결과 요약 교체(LLM 미사용, 앞 3줄 + `... truncated N lines` 형태로 단순 잘라내기) → 이전 retry 실패 diff 제거 → oldest 삭제
  - **최종 검증**: API 호출 직전 system + context + history + task 합산이 전체 예산 초과 시, ContextBuilder가 context 블록을 추가 축소
- [ ] 토큰 카운팅 (`internal/token/` — 공용 패키지. Conversation, ContextBuilder, fullfile fallback 등이 공유)
  - `Estimate(text string) int`: 1차 근사 `len(bytes)/4`. CJK 비율(유니코드 U+3000–U+9FFF + U+AC00–U+D7AF 범위 rune 수 / 전체 rune 수) 30% 이상 → `/3` 보정
  - `Calibrator`: API 피드백 보정. 첫 호출 후 실측과 비교하여 보정 계수 산출 (세션 중 3회 갱신). Estimate 호출 시 보정 계수 자동 적용
  - safety margin: 예산의 20% 차감
- [ ] retry hint
  - 에러 유형별 hint 구성은 상단 "Retry Hint 형식" 템플릿 기반
  - SYNTAX/apply: 실패 hunk의 에러 메시지 + 실제 파일 해당 영역(±10줄)을 task에 추가
  - SYNTAX/format: AST/gofmt 에러 메시지를 task에 추가
  - SYNTAX/verify: 테스트 명령 + 출력 마지막 50줄을 task에 추가
  - TRANSIENT: hint 없이 동일 프롬프트

**테스트**: [I] CLAUDE.md 포함 / 파일 선택 / deny 차단 / 잘림 표시 / [U] 블록 분리 / 축소 순서 / hint 추가

---

## 1-5. Plan 생성 + Gate

> LLM한테 "뭘 할 건지" 먼저 물어보고, 위험한 계획은 Gate에서 걸러낸다.

**Plan** (`internal/plan/`)

- [ ] Plan 생성
  - LLM에 요청 → JSON 응답 → Plan 구조체. Step type: "llm"(diff 생성) / "skill"(registry 실행)
  - **structured output**: Plan schema를 tool의 `input_schema`로 정의하여 tool_use로 JSON 출력 강제 (Claude는 tool_use 지원. 다른 provider용 fallback은 Phase 4-4에서 구현)
  - Plan 생성 = 1회 LLM 호출(JSON). Diff 생성 = step마다 별도 LLM 호출(unified diff text) — 오케스트레이션은 Phase 3 Flow

**Plan 생성용 Tool 정의** (LLM에 tool_use로 Plan JSON 출력을 강제하기 위한 tool)

```json
{
  "name": "create_plan",
  "description": "Create an execution plan for the user's request. Output the plan as structured JSON.",
  "input_schema": <아래 Plan JSON Schema 그대로 사용>
}
```

- **tools 배열**: plan 생성 호출 시 `create_plan` tool만 전달 (`file_read`/`shell_run` 미포함). LLM이 다른 tool을 요청할 수 없게 강제
- LLM이 `create_plan` tool_use로 응답 → Input에서 Plan 구조체 추출 → 하네스가 `plan.Request = request` 주입
- stop_reason이 `tool_use`가 아니거나 `create_plan` 외 tool 호출 시 → SYNTAX 에러 + retry (최대 `api.max_retries`회)
- 복수 tool_use 블록 응답: `create_plan`이 1개만 있으면 그것 사용, 2개 이상이면 SYNTAX 에러 + retry
- plan 생성은 단발 호출로 종료 — tool_result를 돌려보내지 않음 (대화를 이어갈 필요 없으므로)

**Plan JSON Schema** (LLM에 전달하는 tool input_schema. `request`는 LLM이 생성하지 않으므로 스키마에 미포함 — 파싱 후 하네스가 주입)

```json
{
  "type": "object",
  "properties": {
    "goal": { "type": "string", "description": "사용자 요청을 한 문장으로 요약" },
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["llm", "skill"] },
          "description": { "type": "string" },
          "target_files": { "type": "array", "items": { "type": "string" } },
          "ops": { "type": "array", "items": { "type": "string", "enum": ["create", "modify", "delete", "rename"] }, "description": "이 step에서 수행하는 파일 작업 종류" },
          "skill_name": { "type": "string" },
          "skill_input": { "type": "object" }
        },
        "required": ["type", "description", "target_files", "ops"],
        "if": { "properties": { "type": { "const": "skill" } } },
        "then": { "required": ["skill_name"] }
      }
    }
  },
  "required": ["goal", "steps"]
}
```

```go
// Plan — LLM이 생성하는 실행 계획
type Plan struct {
    Goal    string // 사용자 요청 요약 (LLM이 작성)
    Request string // 원본 사용자 요청 텍스트 (하네스가 주입. apply --plan 시 LLM step의 task 블록에 사용)
    Steps   []Step
}

// Step — Plan의 단일 실행 단위
type Step struct {
    Type        string   // "llm" | "skill"
    Description string   // 이 step이 하는 일 (사람이 읽을 용도)
    TargetFiles []string // 변경/읽기 대상 파일 경로 (Gate 검사 대상)

    Ops         []string // "create" | "modify" | "delete" | "rename" — Gate가 deny_ops 검사에 사용

    // type: "skill" 전용
    SkillName  string          // registry에서 조회할 skill 이름 (e.g. "file_read")
    SkillInput json.RawMessage // skill에 전달할 입력 JSON
}
```
- [ ] PlanGate
  - deny_paths: 지정 경로/패턴 대상 plan → 거부
  - deny_ops: Step.Ops와 매칭하여 금지 작업(delete, rename 등) 감지 → on_violation(abort/warn) 정책. Phase 2에서 실제 diff 헤더(`deleted file mode`, `rename from/to`)와 교차 검증
  - 보안: 하네스 소스 파일 대상 거부, 시스템 명령 패턴(`rm -rf`, `curl | sh`) 차단
  - Gate는 plan 내용을 신뢰하지 않는 전제로 동작

**LLM diff 신뢰성 전략** (여기서 정의, 실행은 Phase 2/3):
1. fuzz matching ±3줄 허용 (Phase 2 ApplyEngine)
2. SYNTAX → retry + hint (Phase 2 Verify, Phase 3 Flow)
3. 2회 실패 → 전체 파일 출력 fallback (아래 프로토콜)

**Fullfile Fallback 프로토콜** (diff 2회 실패 시 실행. 실행 주체: Flow)

- 조건: `EstimateTokens(fileContent) < (api.context_window × 0.6 - 고정블록토큰)` 일 때만 시도. 고정블록토큰 = system prompt + task prompt 추정치. 초과 → abort (토큰 카운팅 함수 재사용으로 CJK 보정 포함)
- 프롬프트: 원본 파일 전체를 context에 포함 + "Output the complete modified file content. No diff, no explanation, only the full file."
- 파싱: Parser가 코드 블록 strip → 전체 텍스트 추출 (internal/parser/)
- 적용: ApplyEngine의 diff 경로를 거치지 않고 직접 atomic write (원본 백업 + txlog 기록은 동일하게 수행). AST 검증 + gofmt는 동일 적용
- 다중 파일 step: diff 실패 카운트는 파일 단위로 추적. 파일 A 성공 + 파일 B 2회 실패 → B만 fullfile 시도. fullfile 적용은 파일 1개 단위, 트랜잭션은 step 전체 (B fullfile 실패 → A 포함 전체 rollback)
- 실패 시: AST/gofmt/verify 어느 단계든 실패 → rollback → abort (추가 retry 없음. diff 2회 + fullfile 1회 = 총 3회 시도가 최대)

**테스트**: [I] plan 생성 / [U] deny_paths·deny_ops 거부 / 하네스 소스 거부 / 명령 패턴 차단

---

# Phase 2: Engine — 패치 적용 → 테스트 통과

**마일스톤**: 단일 파일에 LLM diff 적용 → `go test` 통과

**검증 기준** (LLM 호출 없음, fixture 기반):
- 입력: `testdata/project/pkg/greet/greet.go` (원본) + `testdata/diffs/` fixture
  - 정상 적용: `valid_add_function.diff`, fuzz 적용: `fuzz_offset.diff`, 실패: `context_mismatch.diff`, AST 거부: `invalid_syntax.diff`
- 성공 조건:
  1. diff 적용 후 파일이 변경됨 (원본과 다름)
  2. `go/parser`로 AST 유효성 통과
  3. `gofmt` 적용 후 내용 불변 (이미 포맷됨)
  4. `go test ./...` exit code 0
  5. 적용 실패 시 원본 파일 복원 확인 (rollback)
- 실패 조건: context line 불일치 diff → SYNTAX 에러 + 원본 유지

---

## 2-1. Patch Apply

> LLM이 생성한 unified diff를 실제 파일에 적용한다. 핵심은 원자성(전부 성공 아니면 전부 rollback)과 LLM diff의 부정확성 대응.

**ApplyEngine** (`internal/apply/`)

- [ ] diff 파싱 + hunk 적용
  - `go-gitdiff`로 텍스트 → 구조체 파싱
  - hunk 적용은 직접 구현: line 단위 적용, 이전 hunk로 밀린 offset 보정
  - fuzz matching: context line이 ±3줄 범위 내 매칭이면 허용 (경고 로그). 매칭 실패 → SYNTAX
- [ ] AST 검증 + 포매팅
  - `.go` 파일에 `go/parser`로 syntax check. 실패 → SYNTAX
  - Format 메서드: apply 성공 후 Flow가 별도 호출. `gofmt` 실행, 실패 → SYNTAX
- [ ] atomic write
  - 대상 파일과 같은 디렉터리에 임시 파일 생성 → rename (cross-device 방지)
  - Windows: `os.Rename` 시도 → 실패 시 `MoveFileEx` + `MOVEFILE_REPLACE_EXISTING` fallback
  - Windows 열린 핸들: 최대 3회 retry (200ms → 500ms → 1s)
- [ ] 다중 파일 트랜잭션
  - `.harness/txlog.jsonl`에 상태 기록 (prepared → 각 파일 완료 → committed → 삭제). 형식은 공통 규칙의 txlog 이벤트 참조
  - 일부 실패 → 완료된 파일 원본 복원 → txlog 삭제
  - crash 후 재실행 → txlog 존재하면 자동 rollback. 인메모리 상태 복원은 안 함 (clean restart)
- [ ] HookRunner 인터페이스 + NoopHook stub
  - Pre(정책 검사) / Post(cleanup) 슬롯 정의
  - 인터페이스 기반 DI: Phase 2-3에서 PolicyHook 구현 시 생성자 주입으로 교체. Phase 2-1~2-2는 NoopHook으로 동작

**에러 분류**: SYNTAX(diff 파싱/context 불일치/AST/gofmt) / PERMANENT(파일 없음/rollback 실패) / TRANSIENT(I/O)

**테스트**: [I] diff 파싱→적용 / AST 거부 / fuzz 매칭 / 실패 rollback / 다중 파일 부분 실패 / txlog 복구 / gofmt / Windows atomic

---

## 2-2. Verify + Retry

> apply 결과를 검증하고, 실패하면 에러 유형에 따라 retry할지 abort할지 결정하는 상태 머신.

**Verifier** (`internal/verify/`)

- [ ] TestRunner
  - `go test` + `go vet` 실행. `exec.Command`로 직접 실행 (shell 미경유, OS 무관)
  - 기본: 변경 파일이 속한 패키지만. 패키지 결정: 파일 경로의 디렉터리 부분을 Go 패키지 경로로 사용 (e.g. `pkg/greet/greet.go` → `./pkg/greet`). 새 파일(create op)도 동일 — 디렉터리가 존재하지 않으면 apply에서 이미 생성됨. config `verify.scope: all`이면 `./...`
  - 타임아웃: config `verify.timeout` (기본 60초)
- [ ] RetryState 상태 머신
  - 상태 전이: Apply → Verify → PASS(Done) / FAIL(에러 분류 → retry or abort)
  - abort 조건: retry >= MaxRetry(기본 3) 또는 동일 error signature 누적 2회 (연속이 아니어도 — A→B→A면 A 2회로 abort)
- [ ] error signature
  - 목적: "같은 에러가 반복되는가"를 판단. 파일 경로/라인/시간 같은 변동 부분을 제거해서 비교
  - 정규화: 경로+라인 제거 → trim → exact match (prefix match는 오탐 발생)
  - ErrorNormalizer 인터페이스로 분리 → GoErrorNormalizer 구현. Phase 4에서 언어별 확장

**테스트**: [U] PASS → Done / TRANSIENT → retry / SYNTAX → retry+hint / PERMANENT → abort / signature 2회 → abort

---

## 2-3. Hook / Policy

> apply 전후에 정책 검사를 끼워넣는다. 위험한 변경을 abort하거나 경고만 남길 수 있게.

**PolicyHook** (`internal/hook/`)

- [ ] pre hook
  - 체인 실행: 하나라도 에러 → 이후 hook 스킵, PERMANENT 에러로 래핑
- [ ] post hook
  - 성공/실패 무관 전부 실행 (cleanup 목적)
- [ ] on_violation 정책
  - `abort` → PERMANENT 래핑 / `warn` → 로그만
- [ ] override
  - `--override-policy` CLI 플래그로만 활성화. slog.Warn 기록

**테스트**: [U] pre 체인 에러 시 스킵 / post 전부 실행 / abort·warn / override 없이 우회 불가

---

# Phase 3: Integration — end-to-end

**마일스톤**: `agent-harness run --request "..."` 전체 실행

**검증 기준**:
- 입력: `agent-harness run --request "Add a Greet function to pkg/greet/greet.go that takes a name and returns a greeting string, with test"` (테스트용 Go 프로젝트 대상)
- 성공 조건:
  1. plan 생성 → diff 적용 → `go test` 통과 → exit code 0
  2. 생성된 파일에 요청한 함수 존재
  3. `--dry-run` 시 파일 변경 없이 diff만 stdout 출력
  4. budget 초과 시 on_exceed 정책대로 동작 (abort 또는 shrink)
  5. SIGINT(Ctrl+C) 시 진행 중 apply rollback 후 정상 종료 (exit code 130)
- 실패 조건: `deny_paths`에 걸린 파일 대상 요청 → PERMANENT 에러 + exit code 1

---

## 3-1. Skill + Tool Use

> Skill은 LLM 없이 하네스가 직접 실행하는 고정 로직. Tool Use는 LLM이 "이 skill 실행해줘"라고 요청하는 프로토콜.

**Skill** (`internal/skill/`)

- [ ] Skill 인터페이스 + Registry
  - Register/Get/Execute. 중복 등록 → 패닉
  - **이름 규칙**: 내부 registry는 underscore 통일 (`file_read`, `shell_run`). LLM에 전달하는 tool name과 동일하게 유지하여 매핑 불필요
  - Input/Output 스키마 검증. 실패 → PERMANENT (retry 없이 abort)
  - Idempotent: false인 skill 실패 → retry 금지
- [ ] `file_read`
  - 멱등. 파일 읽기. `deny_context_paths` 매칭 → PERMANENT. 미존재 → PERMANENT
- [ ] `shell_run`
  - 비멱등. allow-list: config `skill.shell_allow_list` 기반 (deny-by-default). 비허용 명령 → PERMANENT
  - 실행 방식 2경로:
    - **Direct exec**: 내장 도구 목록(`go`, `git`, `gofmt`, `goimports`)은 `exec.Command`로 직접 실행 — 쉘 안 거침, OS 무관, injection 위험 없음. 명령의 첫 토큰이 목록에 포함되면 direct exec
    - **Shell exec**: 내장 목록 외 명령은 `runtime.GOOS`에 따라 `sh -c`(macOS/Linux) / `cmd /c`(Windows) 분기
  - allow_list 매칭: **prefix match** — 명령을 공백 기준 토큰화 후, allow_list 항목이 명령의 prefix면 허용 (e.g. `"go test"` → `go test ./... -v` 허용). forward slash 통일 정규화 후 비교. Windows에서 case-insensitive
  - 환경변수 격리: 민감 키(ANTHROPIC_API_KEY 등) 제거. 타임아웃 config `skill.shell_timeout`(기본 30초). 출력 config `tool.max_result_size`(기본 64KB) 제한(뒷부분 우선)

**Tool Use** (`internal/tool/`)

- [ ] ToolExecutor
  - 단일 tool_use 블록 실행: ContentBlock에서 name 추출 → Skill Registry에서 실행 → tool_result 반환
  - tool_result 크기(config `tool.max_result_size`) / 스키마 검증
  - 파일 접근 tool → `deny_paths` + `deny_context_paths` 양쪽 적용

**Claude API에 보내는 tools 정의** (필수 필드만)

```json
[
  {
    "name": "file_read",
    "description": "Read file contents at the given path",
    "input_schema": {
      "type": "object",
      "properties": {
        "path": { "type": "string", "description": "File path relative to scan_root" }
      },
      "required": ["path"]
    }
  },
  {
    "name": "shell_run",
    "description": "Run a shell command from the allow-list",
    "input_schema": {
      "type": "object",
      "properties": {
        "command": { "type": "string", "description": "Shell command to execute" }
      },
      "required": ["command"]
    }
  }
]
```
- [ ] 멀티턴 루프 (Flow가 구동, ToolExecutor는 단일 실행만)
  - API 호출 → stop_reason 분기: end_turn(종료) / tool_use(실행→재호출) / max_tokens(TRUNCATED 에러)
  - 종료 조건: config `tool.max_tool_rounds`(기본 10) 초과 / 동일 tool+input 연속 2회 / ctx cancel
  - 매 API 호출마다 budget.Record

**테스트**: [U] 등록·조회 / 스키마 거부 / allow-list / deny_paths / [M] tool_use 루프 / abort 조건 / timeout

---

## 3-2. Observability + Cost Control

> 뭐가 얼마나 걸렸고, 토큰을 얼마나 썼고, 어디서 실패했는지 추적. 비용이 터지지 않게 예산 관리.

**Observability** (`internal/observe/`)

- [ ] Recorder 인터페이스
  - Record(stage, fn): fn 실행을 span으로 감싸서 timing/에러 기록. Summary: 전체 span 요약
  - OTel 호환 구조로 설계 → Phase 4에서 exporter 교체 가능
- [ ] Prompt Audit Trail
  - LLM 호출 전후 프롬프트/응답 원문을 JSONL 저장. 기본 비활성화 (config `audit.enabled`)
  - deny_context_paths에 걸린 내용은 요청 시점에서 이미 제외되므로 audit에도 자연적으로 미포함. LLM 응답에 해당 경로가 언급되는 경우는 redact하지 않음 (audit은 원문 보존 원칙)

**Cost Control** (`internal/cost/`)

- [ ] SessionBudget
  - PreCheck(예상 토큰): 잔여 예산 부족하면 에러. Record(usage): 사용량 차감
  - config `budget.max_input_tokens` / `budget.max_output_tokens` (0 = 무제한). on_exceed: abort / shrink(context 축소 후 재시도)
  - budget은 토큰 수 기준 (모델별 단가 차이는 추적하지 않음 — MVP 범위. 금액 기반 budget은 Phase 4 확장)
- [ ] RateLimiter
  - RPM: token bucket (1분당 max_rpm개, 1초 간격 리필). TPM: 60초 sliding window
  - 0 = 해당 제한 비활성화. 대기 중 ctx cancel → 즉시 에러
  - 429 피드백: APIClient가 429 수신 → `OnThrottle(retryAfter)` → 해당 기간 새 요청 대기
- [ ] 모델 분리
  - plan 생성 = config `api.model` + `api.context_window`, diff 생성 = config `api.diff_model` + `api.diff_context_window` (미설정 시 각각 api.model, api.context_window 사용)
- [ ] Prompt Caching
  - Claude API `cache_control: {"type": "ephemeral"}` 파라미터 사용. 캐시 적중 시 input_tokens 과금 절감. 하네스 자체 캐시 없음
  - breakpoint 배치 전략: (1) system prompt 끝 (2) context 블록 중 가장 큰 파일 내용 끝 — 최대 4개. step 간 변하지 않는 블록에 우선 배치하여 적중률 확보
  - provider별 캐시 메커니즘이 다르므로, CallOption.CacheControl로 breakpoint 위치를 전달하고 provider가 자체 방식으로 적용
- [ ] Observability/Cost 책임 분리: Observability는 관측(audit), Cost Control은 제어(budget 차감). 같은 usage 데이터를 각자 목적으로 소비

**테스트**: [I] timing 기록 / usage 추적 / audit 저장 / [M] budget 정책 / RPM·TPM 대기 / 모델 분리 / cache 반영

---

## 3-3. Flow 통합 + CLI

> 전부 연결한다. Config → Context → Plan → Apply → Verify 전체 파이프라인을 하나로 묶고 CLI를 씌운다.

**Flow** (`internal/flow/`)

- [ ] Phase 1: 초기화
  - dirty workdir 확인: abort(즉시 에러) / stash(`git stash -u` → 실행 → `git stash pop`, stash 실패 → abort, pop conflict → stash 보존 + 에러 메시지 "git stash pop 수동 실행 필요" + exit code 1) / proceed(경고 후 진행). non-git 디렉터리 동작은 config.yaml `flow.dirty_workdir` 주석 참조
  - txlog 존재 → 이전 crash 복구 (자동 rollback)
  - lock 파일 획득 (request_id 기반 중복 실행 방지)
- [ ] Phase 2: Plan 생성
  - ContextBuilder.Build(request) → Conversation 조립 → ShrinkIfNeeded
  - budget PreCheck → 에러 시 on_exceed 분기: `abort` → PERMANENT / `shrink` → ContextBuilder.Shrink 호출 후 PreCheck 재시도 (1회만, 재실패 → abort)
  - LLM 호출(api.model, temperature 0) → budget Record
  - ParsePlan → `plan.Request = request` 주입 (LLM이 아닌 하네스가 설정) → PlanGate.Check
- [ ] Phase 3: Step 루프
  - skill step → SkillRegistry.Execute 직접 호출. 에러 → abort
  - llm step → 아래 retry 루프. step마다 독립 Conversation (step 간 히스토리 미공유, retry 간에는 힌트 누적)
- [ ] LLM step retry 루프
  - Context 구성: target_files 원본 내용을 context 블록에 포함 (기본 전략) + 이전 step 변경 + retry hint 반영. LLM은 추가로 file_read tool로 다른 파일 참조 가능
  - LLM 호출: ShrinkIfNeeded → budget PreCheck (on_exceed `shrink` 시 ContextBuilder.Shrink → 재시도 1회) → toolUseLoop(diff_model). budget Record는 루프 내부 매 호출. retry 시에도 매회 PreCheck 수행 — budget 부족 시 on_exceed 정책 적용 (abort 또는 shrink 후 1회 재시도)
  - toolUseLoop: LLM이 tool_use(file_read/shell_run)로 응답하면 실행→재호출, end_turn(text에 diff 포함)이면 루프 종료 후 diff 파싱. LLM이 tool과 diff를 혼합 출력하면 diff 부분만 추출 시도
  - TRUNCATED(max_tokens 잘림) → Shrink 후 1회 재시도, 재실패 → abort
  - Diff 파싱 실패(SYNTAX) → retry. 2회째 → fullfile fallback
  - Hook Pre → Apply(실패 시 내부 rollback) → Format(실패 시 명시적 Rollback) → Verify
  - Verify PASS → Post hook → 변경 내용을 다음 step context에 누적
  - Verify FAIL → Rollback → Post hook → RetryState 기록 → 동일 signature 2회면 abort
  - max retry 초과 → abort
- [ ] Phase 4: 완료
  - Recorder.Summary → lock 해제

**CLI** (`cmd/agent-harness/`)

- [ ] 서브커맨드
  - `run --request "..."`: 전체 실행. `--request-file path` 또는 `--request -`(stdin)로 파일/파이프 입력 지원. `--request`/`--request-file`/`--request -` 중 하나만 허용 (복수 지정 → exit code 2)
  - `plan --request "..."`: Phase 2까지 (plan JSON 출력, LLM 1회). `--request-file`/`--request -` 동일 지원. 출력 JSON에 `request` 필드 포함
  - `apply --plan plan.json`: Phase 1 → plan 로드 → Phase 3~4. 입력 파일은 Plan JSON Schema와 동일한 JSON 파일 (`plan` 서브커맨드 출력을 그대로 사용). plan JSON의 `request` 필드를 LLM step의 task 블록에 사용. `request` 필드 누락 → exit code 2. plan의 `target_files` 경로는 프로젝트 루트(`--root`) 기준 상대 경로로 해석
- [ ] 공통 플래그: `--root`(프로젝트 루트 경로), `--config`, `--dry-run`(diff 출력만, 파일 쓰기 없음), `--verbose`, `--override-policy`, `--force`
  - `--dry-run`: Apply/Format/Verify 스킵, LLM 호출 + tool use 루프는 정상 실행. `file_read`는 정상 실행, `shell_run`도 정상 실행 (allow-list가 이미 안전 명령만 허용하므로 dry-run에서도 동일 정책). 최종 diff만 stdout 출력
  - `--force`: dirty workdir 정책을 무시하고 `proceed`로 강제 진행 + stale lock 파일 즉시 삭제
- [ ] Graceful Shutdown
  - `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)`. Windows에서 SIGTERM은 수신 불가 — `//go:build !windows` 파일에서 SIGTERM 추가, Windows 빌드는 `os.Interrupt`만 등록
  - Apply 중 → 완료/rollback 후 종료
  - exit code: 0(성공) / 1(실행 중 실패 — apply/verify/API 에러 등) / 2(실행 전 설정 오류 — config 파싱 실패, 필수 플래그 누락, API key 미설정) / 130(신호)

**테스트**: [I] 전체 흐름 / 실패 정책 / 서브커맨드 / dry-run / SIGTERM / txlog 복구 / dirty workdir / toolUseLoop 에러

---

# Phase 4: Extension (MVP 이후)

---

## 4-1. Multi-Agent
> 하나의 Flow를 4개 역할(explorer/planner/implementer/verifier)로 분리하여 goroutine 병렬 실행.

- [ ] Agent 역할 분리 + goroutine/channel 결과 수집
- [ ] FileLockRegistry: path 기준 mutex, 알파벳순 정렬로 deadlock 방지
- [ ] agent 간 데이터 전달은 타입 구조체로만 (LLM 원문 금지)
- [ ] fail-fast(plan/apply) vs best-effort(explore) 정책
- [ ] Implementer 간 동일 영역 충돌 감지

## 4-2. Skill 확장
> registry에 실용 skill 추가 + 의존성 관리.

- [ ] `registry.List()`, 의존 skill 존재 확인, 순환 의존 DFS 감지
- [ ] 추가: `file.glob`, `git.diff`, `git.commit`, `json.query`

## 4-3. GitHub 통합
> Verify 성공 후 PR 자동 생성, 리뷰 코멘트로 수정 사이클 자동화.

- [ ] feature branch 커밋 → GitHub API → PR 생성
- [ ] 리뷰 루프: 미해결 코멘트 → Flow 재실행 → PR 업데이트 (maxRounds 3)
- [ ] `GITHUB_TOKEN` 환경변수. `--dry-run` 시 diff만

## 4-4. 추가 Provider 구현
> Phase 1-2에서 정의한 LLMProvider 인터페이스에 대한 추가 구현체.

- [ ] OpenAIProvider 구현 (GPT-4o 등)
- [ ] config `api.provider` 팩토리 연결: Phase 1 config에 이미 정의된 `api.provider` 필드("claude" 기본) 기반으로 provider 생성자 분기
- [ ] provider별 wire 타입 변환 + 스트리밍 구현
- [ ] tool_use 미지원 provider용 Plan JSON fallback (프롬프트에 JSON schema + "respond ONLY with valid JSON" + few-shot 예시 1쌍)
- [ ] provider 전용 옵션은 CallOption.ProviderOpts로 격리

## 4-5. 언어 플러그인
> Go 외 언어 지원. 포매터/검증기/테스트 명령을 언어별로 교체.

- [ ] LanguagePlugin 인터페이스: Extensions/Format/Validate/TestCommand/ErrorNormalizer
- [ ] PluginRegistry: 확장자 → plugin. 미등록 → 포매팅 없이 적용
- [ ] 내장: GoPlugin(gofmt/go vet), TypeScriptPlugin(prettier/tsc), PythonPlugin(black/mypy)

## 4-6. Observability 완성
> 자체 SpanRecorder를 OTel exporter로 교체.

- [ ] OtelRecorder, config `observability.exporter`: stdout/otlp/none
- [ ] Prometheus 메트릭. Flow 코드 변경 없이 교체
