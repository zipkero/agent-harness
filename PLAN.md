# AI Agent Harness 커리큘럼

---

# 🎯 프로젝트
agent-harness (Go CLI)

---

# 🧠 구성 요소 (9개) → 챕터 매핑

| 컴포넌트 | 담당 챕터 | 역할 |
|---|---|---|
| Context | CH04 | context 구성 및 압축 |
| Flow | CH14 | plan → apply → verify 실행 흐름 orchestrator |
| Control | CH08 | abort / pause / override 정책 |
| Verify | CH07 | 검증 및 상태 전이 |
| Retry | CH07 | error 분류 및 재시도 |
| Cost | CH02(구조) + CH13(로직) | token budget, 모델 분리 |
| Parse | CH03 | 응답 파싱 |
| Plan Gate | CH05 | 실행 전 검증 및 위험 필터 |
| Observability | CH12 | 상태 추적 및 로깅 |

---

# 📚 CHAPTERS

---

## CH01. 하네스의 실체

- LLM ≠ 실행 주체
- 하네스 = 실행 시스템

### 테스트 전략 개요

이 프로젝트에서 테스트는 세 레이어로 접근한다:

| 레이어 | 대상 | 방법 |
|---|---|---|
| Unit | Parser, Gate, ErrorClassifier 등 순수 함수 | 실제 LLM 없이 고정 입력 → 고정 출력 검증 |
| Mock LLM | API client, Context Builder | 미리 준비한 응답 파일(golden file)로 LLM 대체 |
| Integration | Flow orchestrator, Apply Engine | 실제 파일 시스템 + 임시 디렉터리 사용 |

> 각 챕터 완료 기준은 어느 레이어로 검증하는지 위 표를 기준으로 판단한다.
> LLM을 실제로 호출하는 테스트는 CI에 포함하지 않는다 (비용 및 비결정성).

### 전체 아키텍처 흐름

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

### 데이터 흐름

    사용자 요청
      → (context 구성) → prompt
      → (API 호출) → raw 응답
      → (parsing) → structured output
      → (plan 생성) → plan.md
      → (gate 검증) → 통과/거부
      → (apply) → 파일 변경
      → (verify) → 성공/실패
      → (retry or done)

### 전역 Config 구조

모든 챕터의 설정값은 단일 config 파일을 기준으로 한다. 각 챕터는 자신의 섹션만 추가한다.

```yaml
api:
  key: ""                          # CH02
  model: claude-sonnet-4-6         # CH02
  max_tokens: 8192                 # CH02

gate:
  deny_paths: [go.mod, .github/**] # CH05
  deny_ops: [delete, rename]       # CH05
  on_violation: abort              # CH05

context:
  deny_context_paths: [.env, "*secret*", "*credential*"]  # CH04
  max_tokens: 4096                 # CH04

budget:
  max_tokens: 0                    # CH13 (0 = 제한 없음)
  on_exceed: abort                 # CH13
```

> 이 구조는 구현 없이 설계 기준으로만 존재한다.
> 각 챕터에서 자신의 섹션을 구현할 때 이 스키마를 참조한다.
> 설정값 위치가 분산되면 CH14 통합 시 충돌이 생기므로 이 파일이 유일한 anchor다.

### 인터페이스 설계 원칙

이 프로젝트 전체에서 반복되는 패턴을 CH01에서 미리 확립한다.

**Go 인터페이스의 역할**

인터페이스는 구현체를 교체 가능하게 만드는 유일한 수단이다. 테스트에서 Mock으로 교체하거나, 챕터 간 단계적 구현 교체(stub → 실제 구현)가 가능한 이유다. Go에서 인터페이스는 사용하는 쪽(caller)에서 정의한다.

**이 프로젝트에서 인터페이스가 적용되는 위치**

| 인터페이스 | 사용처 | 테스트 대체 | 운영 구현 | 챕터 |
|---|---|---|---|---|
| `HookRunner` | ApplyEngine | `NoopHook` | `PolicyHook` | CH06 stub → CH08 구현 |
| `LLMClient` | Flow | `MockClient` | `APIClient` | CH02 구현 → CH14 주입 |
| `Verifier` | Flow | `AlwaysPassVerifier` | `TestRunner` | CH07 구현 → CH14 주입 |
| `ContextBuilder` | Flow | `FixedContextBuilder` | `RealContextBuilder` | CH04 구현 → CH14 주입 |

**의존성 주입 패턴**

```go
// 인터페이스는 사용하는 쪽에서 정의
type HookRunner interface {
    Pre(ctx context.Context, plan Plan) error
    Post(ctx context.Context, result ApplyResult) error
}

// ApplyEngine은 구현체를 모른다
type ApplyEngine struct {
    hook HookRunner
}

func NewApplyEngine(hook HookRunner) *ApplyEngine { ... }

// 테스트: NoopHook 주입
engine := NewApplyEngine(&NoopHook{})

// 운영: PolicyHook 주입
engine := NewApplyEngine(NewPolicyHook(cfg))
```

> 각 챕터에서 stub을 먼저 정의하고 나중에 교체하는 이유가 바로 이 패턴이다.
> 인터페이스 없이 구체 타입을 직접 참조하면 교체 시 모든 호출부를 수정해야 한다.

### 에러 타입 설계 원칙

이 프로젝트에서 에러는 caller가 분기할 수 있도록 구조화한다. 에러 분류(TRANSIENT / PERMANENT / SYNTAX)가 CH03, CH07, CH10에 걸쳐 사용되므로 CH01에서 공통 타입을 확립한다.

```go
type ErrorClass string

const (
    ErrorClassTransient ErrorClass = "TRANSIENT" // 재시도 가능
    ErrorClassPermanent ErrorClass = "PERMANENT" // 즉시 abort
    ErrorClassSyntax    ErrorClass = "SYNTAX"    // retry + hint 추가
)

type HarnessError struct {
    Class   ErrorClass
    Stage   string // "parse", "apply", "verify" 등
    Wrapped error
}

func (e *HarnessError) Error() string {
    return fmt.Sprintf("[%s/%s] %v", e.Stage, e.Class, e.Wrapped)
}
func (e *HarnessError) Unwrap() error { return e.Wrapped }
```

**에러 분류 확인은 `errors.As` 사용**

```go
var he *HarnessError
if errors.As(err, &he) && he.Class == ErrorClassTransient {
    // retry
}
```

**규칙**
- sentinel error는 단순 상태 표현(EOF 등)에만 사용, 분기가 필요한 에러는 `HarnessError`로 래핑
- 에러 래핑 시 `fmt.Errorf("...: %w", err)` 사용 — `errors.Is` / `errors.As` 체인 유지
- 에러를 새로 생성하는 위치(발생 지점)에서만 `HarnessError`를 생성, 상위 레이어는 그대로 전파

> 이 타입은 CH03(에러 분류), CH07(상태 전이 조건), CH10(tool 응답 처리)에서 공통으로 사용한다.
> 각 챕터에서 별도 에러 타입을 정의하면 CH14 통합 시 분류 로직이 중복된다.

### Config 로드 및 검증 패턴

모든 챕터의 설정값은 단일 Config 구조체를 통해 로드된다. 각 챕터는 자신의 섹션에 `Validate()` 메서드를 추가하는 방식으로 검증 책임을 분산한다.

```go
type Config struct {
    API     APIConfig     `yaml:"api"`
    Gate    GateConfig    `yaml:"gate"`
    Context ContextConfig `yaml:"context"`
    Budget  BudgetConfig  `yaml:"budget"`
}

func LoadConfig(path string) (Config, error) {
    // 1. 파일 읽기
    // 2. YAML 파싱
    // 3. 각 섹션 Validate() 호출
    var cfg Config
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return Config{}, err
    }
    return cfg, cfg.Validate()
}

func (c Config) Validate() error {
    return errors.Join(
        c.API.Validate(),
        c.Gate.Validate(),
        c.Context.Validate(),
        c.Budget.Validate(),
    )
}

// 각 챕터가 자신의 섹션에 구현 — 예시
func (c APIConfig) Validate() error {
    if c.Model == "" {
        return errors.New("api.model is required")
    }
    return nil
}
```

> 각 챕터는 자신의 config 섹션이 추가될 때 `Validate()` 메서드를 함께 구현한다.
> 이 패턴을 CH01에서 확립하지 않으면 CH14 통합 시 검증 로직이 분산되거나 누락된다.

### 완료 기준
- 하네스 전체 구성요소 9개의 역할과 실행 순서를 설명할 수 있다
- 각 컴포넌트가 어느 챕터에서 구현되는지 매핑할 수 있다
- 인터페이스 설계 위치 테이블을 기준으로 각 컴포넌트의 교체 지점을 설명할 수 있다
- config 파일 로드 시 필수 필드 누락이 명확한 에러로 반환된다
- 각 챕터가 자신의 config 섹션에 `Validate()` 메서드를 추가하는 패턴이 확립된다

---

## CH02. Claude API 기본

- API 요청 / 응답 구조
- CallOption 설계 (Model, MaxTokens, BudgetTokens, CacheControl)
  - BudgetTokens는 기본값 0 = 제한 없음, CH13에서 로직 채움
- API Key 로드 순서
  1. 환경변수 우선
  2. config 파일 fallback
- HTTP 레벨 에러 처리 (4xx / 5xx 분류)

### context.Context 인터페이스 확립

이 챕터에서 모든 함수의 첫 인자를 `ctx context.Context`로 고정한다.

```go
func (c *Client) Call(ctx context.Context, opt CallOption, prompt string) (Response, error)
```

이후 모든 챕터는 이 인터페이스를 그대로 따른다. 나중에 추가하면 CH02~CH13의 인터페이스를 전부 수정해야 하므로 여기서 확립한다.

- `ctx`를 HTTP 요청에 전달 → 상위에서 cancel 시 요청 즉시 중단
- CH08 abort: `cancel()` 호출로 전파
- CH15 goroutine: 각 agent goroutine이 동일 ctx를 공유 → 하나가 실패하면 전체 취소 가능

> BudgetTokens 슬롯을 처음부터 포함하는 이유:
> CH13에서 retrofit하면 CallOption 구조 전체를 변경해야 하므로 슬롯만 먼저 확보.

### 테스트 인프라 구축

CH02부터 모든 챕터가 Mock LLM을 필요로 한다. 여기서 한 번 구축하고 이후 챕터에서 재사용한다.

**Golden File 구조**

```
testdata/
├── api/
│   ├── simple_response.json          // 정상 단일 응답
│   ├── 429_rate_limit.json           // Rate limit 에러
│   ├── 500_server_error.json         // 서버 에러
│   └── streaming/
│       ├── chunk_01.json             // SSE 청크 (CH03용)
│       └── chunk_02.json
├── diffs/
│   ├── valid_patch.diff              // 정상 unified diff (CH06용)
│   └── malformed_patch.diff          // 깨진 diff
└── plans/
    └── safe_plan.md                  // 정상 plan (CH05용)
```

**Mock HTTP 서버 패턴**

```go
// 테스트마다 서버를 띄우고 URL을 Client에 주입
func newMockServer(t *testing.T, fixture string) *httptest.Server {
    t.Helper()
    data, err := os.ReadFile(filepath.Join("testdata/api", fixture))
    require.NoError(t, err)

    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.Write(data)
    }))
    t.Cleanup(srv.Close)
    return srv
}

// 사용
srv := newMockServer(t, "simple_response.json")
client := NewClient(WithBaseURL(srv.URL))
```

**호출 횟수 카운팅 (retry 검증용)**

```go
type countingHandler struct {
    count    int
    response []byte
}

func (h *countingHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    h.count++
    w.WriteHeader(http.StatusServiceUnavailable)
    w.Write(h.response)
}
```

**임시 디렉터리 규칙**

- `t.TempDir()` 사용 — 테스트 종료 시 자동 정리
- 테스트 간 파일 공유 금지 (각 테스트가 독립된 디렉터리 사용)
- 실제 작업 디렉터리의 파일을 수정하는 테스트 작성 금지

> 연결 끊김 시뮬레이션: `net.Pipe()`로 연결 후 즉시 close → `io.ErrUnexpectedEOF` 발생
> 이 패턴은 CH03 스트리밍 중단 테스트에서 사용한다.

### 완료 기준
- API key 로드 → 단일 요청 → 응답 수신까지 동작한다
- 4xx / 5xx 에러를 구분해서 처리한다
- 모든 공개 함수가 `ctx context.Context`를 첫 인자로 받는다
- ctx가 cancel되면 진행 중인 HTTP 요청이 중단된다
- Mock HTTP 서버가 fixture 파일을 응답으로 반환한다
- 임시 디렉터리에서 파일 생성/수정이 테스트 종료 후 자동 정리된다
- 호출 횟수 카운팅으로 retry 동작을 검증할 수 있다

---

## CH03. Parsing + 호출 안정화

### Part 1. 응답 파싱 + 신뢰 경계

"LLM 출력을 어떻게 읽는가"에 집중한다.

- structured output 파싱
- fallback parsing (markdown strip, JSON fragment 추출)
- LLM 출력 신뢰 경계

#### 스트리밍 응답 처리

- SSE(Server-Sent Events) 청크 수신 루프
- 청크 단위 누적 파싱 (부분 JSON 조립)
- 스트리밍 중단(EOF / 네트워크 단절) 시 처리
- 스트리밍 vs 단일 응답 파싱 인터페이스 통일

> 스트리밍 청크는 결국 "불완전한 응답을 어떻게 파싱하는가" 문제이므로 파싱 챕터에서 다룬다.
> API 연결(CH02)은 연결 자체에 집중하고, 응답을 어떻게 소비하는지는 여기서 결정한다.

#### LLM 출력 신뢰 경계

LLM이 생성한 diff/JSON을 그대로 실행하면 안 된다.

- 파싱 단계에서 구조 검증 (스키마 일치 여부)
- 허용된 필드 외 unknown field 거부
- 출력 크기 상한 적용 (비정상적으로 큰 응답 차단)

> LLM 출력은 외부 입력과 동일한 신뢰 수준으로 처리한다.
> 파싱 성공 ≠ 출력 신뢰. 구조가 맞아도 내용은 별도 검증이 필요하다.

> CH02와 분리한 이유:
> API 연결 자체와 응답 처리는 독립적인 관심사다.
> CH02에서 둘을 묶으면 첫 동작 확인 전에 parsing 설계까지 결정해야 하는 부담이 생긴다.

#### 완료 기준 (Part 1)
- LLM이 markdown 코드 블록으로 감싼 JSON을 올바르게 추출한다
- structured output 파싱 실패 시 fallback이 동작한다
- 스트리밍 응답을 청크 단위로 누적해 완성된 출력을 반환한다
- 스트리밍 중단 시 이미 수신한 청크를 기준으로 에러를 반환한다
- 허용 스키마 외 필드가 포함된 LLM 출력을 거부한다

---

### Part 2. 호출 안정화

"API 연결을 어떻게 안정화하는가"에 집중한다.

- retry / backoff
- 에러 분류 (transient / permanent / syntax) → CH01의 `HarnessError` 사용
- 429 Retry-After 파싱 및 대기

#### 에러 분류 기준

| ErrorClass | 조건 예시 | 동작 |
|---|---|---|
| TRANSIENT | connection refused, timeout, 5xx | retry |
| SYNTAX | 파싱 실패, 스키마 불일치 | retry + hint 추가 |
| PERMANENT | 4xx (401/403), 요청 구조 오류 | abort |

> CH01에서 정의한 `HarnessError`를 그대로 사용한다.
> 이 챕터에서 새 에러 타입을 정의하지 않는다.

#### 완료 기준 (Part 2)
- retry/backoff가 TRANSIENT 에러에서만 동작한다
- PERMANENT 에러에서 retry 없이 즉시 에러를 반환한다
- 429 수신 시 Retry-After 헤더에 명시된 시간만큼 대기 후 재시도한다
- 에러가 `HarnessError`로 래핑되어 Stage와 Class가 포함된다

---

## CH04. Context Builder + Selective Context

- CLAUDE.md / AGENTS.md 로드
- rule priority
- conflict resolution
- selective file selection
- context compression

### 프롬프트 템플릿 설계

Context Builder가 구성하는 최종 prompt의 구조:

```
[system prompt]
  - 하네스 역할 정의
  - 출력 형식 명세 (unified diff / JSON schema)
  - 금지 행동 명세

[context block]
  - CLAUDE.md 규칙
  - 선택된 파일 내용

[task block]
  - 사용자 요청
  - plan (있을 경우)
  - 이전 실패 hint (retry 시)
```

- 블록별 템플릿 분리 (system / context / task)
- few-shot 예시 삽입 위치 및 조건
- retry 시 hint 블록 추가 방식
- 토큰 예산에 따른 블록 우선순위 (context 압축 순서)

### Credential 노출 방지

Context에 포함되는 파일이 민감 정보를 담고 있을 수 있다.

- deny_context_paths 설정: `.env`, `*secret*`, `*credential*` 등 패턴 매칭
- 파일 선택 단계에서 deny 목록 필터링 (LLM에 전달 전 차단)
- 선택된 파일에 고엔트로피 문자열(API key 패턴) 포함 시 경고

> Context가 LLM에 전송되는 순간 외부로 나간다.
> 파일 선택 단계가 마지막 방어선이다.

### 완료 기준
- CLAUDE.md를 로드해서 context에 포함한다
- 파일 목록에서 관련 파일만 선택해서 context 크기를 줄인다
- rule 충돌 시 우선순위 기준으로 하나가 선택된다
- system / context / task 블록이 분리된 템플릿으로 구성된다
- deny_context_paths에 매칭되는 파일은 context에 포함되지 않는다

---

## CH05. Plan 생성 + Plan Gate

- plan.md 구조
- target file 지정
- step 분해
- done criteria

### Plan Gate
- 파일 존재 검증
- 범위 검증
- 위험 작업 필터링 — 정책은 외부 설정 파일로 분리
  - deny_paths: go.mod, go.sum, .github/** 등
  - deny_ops: delete, rename 등
  - on_violation: abort 또는 confirm
- Gate 코드는 정책을 로드해 검증만 수행, 기준 변경 시 설정 파일만 수정

### Prompt Injection 방어

LLM이 생성한 plan이 하네스 자체의 동작을 변경하려는 내용을 포함할 수 있다.

- plan의 target file이 하네스 자신의 소스를 가리키는 경우 거부
- plan step에 시스템 명령 패턴(`rm -rf`, `curl | sh` 등) 포함 시 거부
- plan 출처가 사용자 요청이 아닌 LLM 자체 생성임을 명시적으로 표기

> plan.md는 LLM이 작성한 문서다. 사용자 의도와 다른 내용이 삽입될 수 있다.
> Gate는 plan의 내용을 신뢰하지 않는 전제로 검증한다.

### 완료 기준
- LLM 응답으로부터 plan.md를 생성한다
- deny_paths에 포함된 파일을 대상으로 한 plan은 gate에서 거부된다
- deny_ops에 해당하는 작업이 on_violation 정책에 따라 처리된다
- 하네스 자신의 소스 파일을 target으로 하는 plan이 거부된다
- plan step에 시스템 명령 패턴이 포함된 경우 gate에서 차단된다

---

## CH06. Patch Apply Engine ⭐

### 적용 방식: unified diff + AST 검증 분리

- LLM 출력 형식: unified diff
- Apply: go-gitdiff 라이브러리로 파싱 후 적용
- AST 역할: syntax 검증 전용 (파싱 성공 여부만 확인)
- 포매팅: gofmt 후처리
- rollback: apply 전 원본 파일 백업, 실패 시 복원
- FileMeta (EOL / encoding) 보존

> AST 기반 function replace는 사용하지 않는다.
> comment association 비결정성 및 포매팅 파괴 문제로 인해 unified diff 방식으로 대체.

### Hook 인터페이스 (이 챕터에서 stub 정의)

Apply Engine이 pre/post hook 슬롯을 가지되, 실제 구현은 CH08에서 교체.
CH06에서는 아무것도 하지 않는 NoopHook으로 연결.

### 파일 시스템 원자성

단순 백업(.bak) 방식은 apply 도중 crash 시 파일이 손상된 상태로 남는다.

**왜 단순 백업이 불충분한가**

```
1) file.go 읽기
2) file.go.bak 복사         ← crash → .bak만 남음
3) 패치 적용 후 덮어쓰기      ← crash → file.go 반쯤 쓰인 상태
4) 성공 → .bak 삭제
```

3번 중간에 crash가 나면 `file.go`는 복구 불가능한 상태가 된다.

**Atomic Replace 패턴**

```go
func atomicWrite(path string, content []byte) error {
    dir := filepath.Dir(path)
    // 반드시 같은 파티션에 임시 파일 생성 (cross-device rename 방지)
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil {
        return err
    }
    tmpPath := tmp.Name()

    if _, err := tmp.Write(content); err != nil {
        os.Remove(tmpPath)
        return err
    }
    if err := tmp.Close(); err != nil {
        os.Remove(tmpPath)
        return err
    }

    // os.Rename은 POSIX에서 atomic
    // 실패해도 원본 파일은 손상되지 않는다
    return os.Rename(tmpPath, path)
}
```

**Rollback 설계**

- apply 전 원본 내용을 메모리에 보관 (파일이 작은 경우)
- 파일이 큰 경우 임시 디렉터리에 스냅샷 저장 후 atomic write로 복원
- `.bak` 파일을 작업 디렉터리에 남기지 않는다
- rollback 자체도 atomic write를 사용한다

> cross-device rename이 발생하면 os.Rename이 에러를 반환한다.
> 임시 파일은 반드시 대상 파일과 동일한 파티션(같은 디렉터리)에 생성해야 한다.

### 완료 기준
- unified diff 형식의 LLM 출력을 파싱해서 파일에 적용한다
- AST syntax 검증 실패 시 apply를 거부한다
- apply 실패 시 원본 파일로 rollback된다
- NoopHook이 pre/post 슬롯에 연결되어 있다
- 패치 적용 도중 에러가 발생해도 원본 파일이 손상되지 않는다
- rollback 완료 후 임시 파일이 남지 않는다

---

## CH07. Verify + Retry State Machine

### 상태 전이

    [Apply 완료]
         │
         ▼
      [Verify]
      /      \
    PASS    FAIL
     │        │
    [Done]  [error 분류]
              /        \
        TRANSIENT    PERMANENT
            │              │
       [Retry N회]      [Abort]
            │
         [Verify]
            │
      retry >= limit
            │
          [Abort]

### Error 분류 기준

| ErrorClass | 조건 예시 | 동작 |
|---|---|---|
| TRANSIENT | connection refused, timeout | retry |
| SYNTAX | syntax error, parse fail | retry + hint 추가 |
| PERMANENT | undefined symbol, type mismatch | abort |

### Abort 조건 (명시)
- retry >= MaxRetry (기본 3회)
- 동일 error signature 연속 2회 → 무한루프 방지 abort

### Error Signature 비교

error signature는 에러 메시지에서 위치 정보(라인 번호, 파일명)를 제거한 정규화된 문자열이다. "같은 에러가 반복되는가"를 판단하기 위해 사용한다.

**왜 원본 메시지를 그대로 비교하면 안 되는가**

```
# retry 1 실패 메시지
./main.go:42: undefined: Foo

# retry 2 실패 메시지 (같은 에러지만 라인이 바뀜)
./main.go:47: undefined: Foo
```

원본 메시지를 비교하면 다른 에러로 판단해 무한 retry가 발생한다.

**Signature 추출 규칙**

```go
var (
    lineNumPattern  = regexp.MustCompile(`:\d+:`)   // :42: → :N:
    columnPattern   = regexp.MustCompile(`:\d+\s`) // :12 → :N
    filePathPattern = regexp.MustCompile(`\S+\.go`) // 파일명 제거
)

func extractSignature(errMsg string) string {
    s := lineNumPattern.ReplaceAllString(errMsg, ":N:")
    s = columnPattern.ReplaceAllString(s, ":N ")
    // 앞 200자만 사용 (비정상적으로 긴 메시지 대응)
    if len(s) > 200 {
        s = s[:200]
    }
    return strings.TrimSpace(s)
}
```

**비교 방식: prefix match**

전체 메시지가 아닌 앞부분(signature)만 비교한다. 에러 메시지 끝에 추가되는 hint나 stack trace는 비교에서 제외된다.

```go
type RetryState struct {
    lastSignature string
    consecutiveCount int
}

func (s *RetryState) record(err error) bool {
    sig := extractSignature(err.Error())
    if strings.HasPrefix(sig, s.lastSignature) || s.lastSignature == sig {
        s.consecutiveCount++
    } else {
        s.consecutiveCount = 1
        s.lastSignature = sig
    }
    return s.consecutiveCount >= 2 // true = abort
}
```

**예시**

| retry | 에러 메시지 | signature | 연속 횟수 | abort? |
|---|---|---|---|---|
| 1 | `./main.go:42: undefined: Foo` | `undefined: Foo` | 1 | no |
| 2 | `./main.go:47: undefined: Foo` | `undefined: Foo` | 2 | **yes** |
| 1 | `./main.go:42: undefined: Foo` | `undefined: Foo` | 1 | no |
| 2 | `./main.go:42: cannot use Bar` | `cannot use Bar` | 1 | no (다른 에러) |

> SYNTAX 에러의 경우 retry 시 prompt에 hint를 추가한다 (CH03). hint가 효과가 없으면 signature가 동일하게 유지되므로 2회 연속 후 자동 abort된다.

### 포함
- test / lint 실행
- failure classification
- error signature 추출 및 비교 (정규화 → prefix match)
- retry limit
- abort 조건

### 완료 기준
- verify 성공 시 Done으로 전이된다
- TRANSIENT 에러에서 retry가 동작하고 MaxRetry 초과 시 abort된다
- PERMANENT 에러에서 즉시 abort된다
- 동일 error signature 2회 연속 시 abort된다

---

## CH08. Hook / Policy (Control 컴포넌트)

CH06에서 정의한 Hook 인터페이스의 실제 구현체를 여기서 작성.

- pre hook 구현
- post hook 구현
- abort / pause / override 강제 실행 정책
- Control: 실행 흐름 중단 및 재개 조건

### PolicyHook 구현

```go
type PolicyHook struct {
    cfg HookConfig
}

type HookConfig struct {
    PreChecks  []CheckFunc  // apply 전 실행 순서대로
    PostChecks []CheckFunc  // apply 후 실행 순서대로 (성공/실패 무관)
    OnViolation string      // "abort" | "warn"
}

type CheckFunc func(ctx context.Context, plan Plan) error
```

pre hook은 체인으로 실행된다. 하나라도 에러를 반환하면 이후 체크는 실행하지 않고 즉시 전파한다.

```go
func (h *PolicyHook) Pre(ctx context.Context, plan Plan) error {
    for _, check := range h.cfg.PreChecks {
        if err := check(ctx, plan); err != nil {
            if h.cfg.OnViolation == "abort" {
                return &HarnessError{Class: ErrorClassPermanent, Stage: "pre-hook", Wrapped: err}
            }
            slog.Warn("hook.pre.violation", "err", err)
        }
    }
    return nil
}
```

post hook은 apply 성공/실패 무관하게 전부 실행한다. 개별 체크 에러는 로그에만 기록하고 전파하지 않는다 (cleanup 목적이므로).

### Abort / Pause / Override 정책

**Abort 조건 (즉시 중단)**

| 조건 | 발생 위치 | 설명 |
|---|---|---|
| pre hook PERMANENT 에러 | apply 전 | 정책 위반, 복구 불가 |
| Gate 거부 (CH05) | plan 이후 | deny_paths / deny_ops 위반 |
| budget 초과 (CH13) | API 호출 전 | `on_exceed: abort` 설정 시 |
| retry >= MaxRetry (CH07) | verify 이후 | 무한 루프 방지 |
| 동일 error signature 2회 연속 (CH07) | verify 이후 | 진전 없는 반복 방지 |

**Pause 조건 (대기 후 재개)**

| 조건 | 재개 트리거 |
|---|---|
| 429 rate limit (CH03) | Retry-After 대기 완료 |
| rate limiter 토큰 고갈 (CH13) | 토큰 보충 완료 |

> Pause는 ctx cancel로 중단 가능하다. SIGINT 수신 시 대기 중인 hook도 취소된다.

**Override 정책**

정책상 abort가 발생해야 하는 상황을 사용자가 강제로 통과시킬 수 있다.

```go
type OverridePolicy struct {
    AllowDenyPaths bool // deny_paths 무시
    AllowDenyOps   bool // deny_ops 무시
    MaxRetryBump   int  // MaxRetry를 이 값으로 임시 상향
}
```

- override는 CLI 플래그 `--override-policy`로만 활성화
- override 활성화 시 `slog.Warn`으로 반드시 기록
- 코드 레벨에서 override를 hardcode하지 않는다

### 완료 기준
- pre hook이 apply 전에 실행되고, 체인 중 에러 발생 시 이후 체크가 실행되지 않는다
- post hook이 apply 후 (성공/실패 무관) 전부 실행된다
- `OnViolation: abort` 설정 시 pre hook 에러가 `ErrorClassPermanent`로 래핑된다
- `OnViolation: warn` 설정 시 pre hook 에러가 로그에만 기록되고 실행이 계속된다
- abort 조건 5가지 각각이 해당 위반 상황에서 실행을 중단시킨다
- override 플래그 없이는 abort 조건이 우회되지 않는다
- override 활성화 시 slog.Warn이 기록된다

---

## CH09. Skill

재사용 가능한 작업 단위. LLM 호출 없이 실행 가능한 고정 로직을 캡슐화한다.

### Skill 구조

```
Skill
├── Name       string          // registry 조회 키
├── Input      Schema          // 입력 필드 정의 + 유효성 검증 규칙
├── Output     Schema          // 출력 필드 정의
└── Execute    func(Input) Output
```

### Skill Registry

- 이름 → 구현체 매핑 (map[string]Skill)
- 등록: `registry.Register(skill)`
- 조회: `registry.Get(name)` — 없으면 에러 반환
- 중복 등록 시 패닉 (초기화 시점에 발견되어야 함)

### 입출력 유효성 검증

- 호출 전: Input 스키마 검증 (필수 필드, 타입)
- 호출 후: Output 스키마 검증 (계약 위반 감지)
- 검증 실패는 실행 에러와 구분된 SkillValidationError로 분류

### CH16와 역할 분리

| | CH09 | CH16 |
|---|---|---|
| 관심사 | 단일 skill 실행 계약 | skill registry 운영 (등록/조회/목록) |
| 범위 | skill 인터페이스 + 검증 | 다수 skill 관리 및 Flow 연결 |

### Flow와의 연결

plan step에 `type` 필드를 추가해 LLM 호출 없이 skill을 직접 실행할 수 있다.

```go
type StepType string

const (
    StepTypeLLM   StepType = "llm"
    StepTypeSkill StepType = "skill"
)

type PlanStep struct {
    Type   StepType       // "llm" | "skill"
    Skill  string         // type=skill일 때 registry 조회 키
    Input  map[string]any // skill 입력
    Target string         // type=llm일 때 대상 파일
}
```

Flow(CH14)에서 step을 실행할 때:

```
[PlanStep 실행]
      │
      ├── type = "llm"   → Claude API 호출 → Apply Engine
      └── type = "skill" → registry.Get(name) → Execute(input)
```

> skill step은 LLM 호출과 Apply를 건너뛰므로 observability에서도 별도 분기로 기록한다.

### Execute 실패 분류

skill 실행 실패는 두 가지로 구분한다.

| 실패 유형 | 설명 | 에러 타입 |
|---|---|---|
| 검증 실패 | Input/Output 스키마 불일치 | `SkillValidationError` |
| 실행 실패 | Execute 내부 에러 | `HarnessError` (CH01) |

`SkillValidationError`는 skill 계약 위반이므로 PERMANENT로 분류 → retry 없이 즉시 abort.

```go
type SkillValidationError struct {
    SkillName string
    Field     string // 어떤 필드가 문제인지
    Reason    string
}

func (e *SkillValidationError) Error() string {
    return fmt.Sprintf("skill[%s].%s: %s", e.SkillName, e.Field, e.Reason)
}
```

### 멱등성 계약

skill은 멱등적이어야 한다. 같은 Input으로 두 번 호출해도 시스템 상태가 동일해야 한다. 멱등성을 보장할 수 없는 skill은 등록 시 `Idempotent: false`로 표시한다.

```go
type Skill struct {
    Name       string
    Idempotent bool       // false = 실패 시 retry 금지
    Input      Schema
    Output     Schema
    Execute    func(Input) Output
}
```

> `Idempotent: false` skill이 실패하면 CH10의 비멱등 tool 처리와 동일하게 PERMANENT 처리한다.

### 완료 기준
- skill을 정의하고 이름으로 registry에 등록할 수 있다
- 존재하지 않는 이름으로 조회 시 에러를 반환한다
- Input 스키마 검증 실패 시 실행 전에 `SkillValidationError`를 반환한다
- Output 스키마 검증 실패 시 `SkillValidationError`를 반환한다
- `SkillValidationError`는 PERMANENT로 분류되어 retry 없이 abort된다
- 동일 입력에 대해 동일 출력 계약이 보장된다 (멱등성)
- `Idempotent: false` skill이 실패하면 retry 없이 abort된다
- plan step의 type이 `skill`이면 LLM 호출 없이 registry에서 직접 실행된다
- skill step 실행이 observability(CH12)에 별도 stage로 기록된다

---

## CH10. MCP / Tool Use

- 외부 시스템 연결
- tool execution은 하네스가 담당
- credential: CH02의 API key 로드 패턴과 동일하게 통일

### Tool 응답 신뢰 경계

tool 응답은 LLM 출력과 동일한 신뢰 수준으로 처리한다. 외부 시스템이 악의적이거나 손상된 응답을 보낼 수 있다.

- tool 응답도 CH03과 동일하게 스키마 검증 (허용 필드 외 unknown field 거부)
- 응답 크기 상한 적용 (비정상적으로 큰 응답 차단)
- 파싱 성공 ≠ 응답 신뢰

### Tool 실행 옵션

```go
type ToolCallOption struct {
    Timeout         time.Duration // 0 = 기본값 30s
    MaxResponseSize int64         // 바이트 단위, 0 = 기본값 1MB
    Idempotent      bool          // true = 실패 시 retry 허용
}
```

- timeout 초과 시 context cancel → TRANSIENT 에러로 분류 (CH07 연결)
- `Idempotent: false`인 tool은 retry 없이 즉시 PERMANENT 처리
- 재시도 가능 여부는 tool 등록 시 명시

### Tool 실행 격리

- 파일 시스템 접근 tool은 CH05의 deny_paths 정책과 동일하게 적용
- 쉘 명령 실행 tool은 allow-list 기반 허용만 (deny-by-default)
- tool이 하네스 자신의 소스를 대상으로 접근하려 할 경우 거부 (CH05 Prompt Injection 방어와 동일 원칙)

### 완료 기준
- tool 호출 요청을 받아 실행하고 결과를 반환한다
- 외부 시스템 credential을 CH02와 동일한 방식으로 로드한다
- tool 응답이 허용 스키마를 벗어나면 거부된다
- tool 응답 크기가 상한을 초과하면 에러를 반환한다
- tool 실행이 timeout을 초과하면 취소되고 TRANSIENT로 분류된다
- 비멱등 tool은 실패 시 retry 없이 abort된다
- 파일 시스템 접근 tool이 deny_paths를 준수한다

---

## CH12. Observability

- stage timing
- token usage
- retry count
- failure tracking

### SpanRecorder

외부 라이브러리 없이 span 개념을 직접 구현한다. Flow의 각 stage를 `Record`로 감싸면 전체 실행 타임라인이 자동으로 수집된다.

```go
type Span struct {
    Stage     string
    StartedAt time.Time
    Duration  time.Duration
    TokensIn  int
    TokensOut int
    Err       error
}

type SpanRecorder struct {
    mu    sync.Mutex
    spans []Span
}

func (r *SpanRecorder) Record(stage string, fn func() error) error {
    start := time.Now()
    err := fn()
    r.mu.Lock()
    r.spans = append(r.spans, Span{
        Stage:     stage,
        StartedAt: start,
        Duration:  time.Since(start),
        Err:       err,
    })
    r.mu.Unlock()
    return err
}

func (r *SpanRecorder) Summary() []Span {
    r.mu.Lock()
    defer r.mu.Unlock()
    return slices.Clone(r.spans)
}
```

CH14 Flow에서의 사용:

```go
err = recorder.Record("apply", func() error {
    return applyEngine.Apply(ctx, patch)
})
err = recorder.Record("verify", func() error {
    return verifier.Run(ctx)
})
```

### 구조화 로깅 (slog)

```go
// stage 진입
slog.Info("stage.start", "stage", "apply", "request_id", reqID)

// stage 완료
slog.Info("stage.done",
    "stage", "apply",
    "duration_ms", span.Duration.Milliseconds(),
    "tokens_in", span.TokensIn,
    "tokens_out", span.TokensOut,
)

// stage 실패
slog.Error("stage.fail",
    "stage", "apply",
    "err", err,
    "retry", retryCount,
)
```

> slog 레벨(Debug/Info/Error)은 stage 결과에 따라 구분한다.
> SpanRecorder는 CH14에서 Flow에 주입되어 전 단계를 커버한다.

### 완료 기준
- 각 stage(context / plan / apply / verify)의 소요 시간을 출력한다
- 요청별 token usage를 기록한다
- retry 횟수와 실패 유형을 추적한다
- 각 stage가 SpanRecorder를 통해 timing을 기록한다
- 실행 종료 후 전체 span 요약(stage / duration / err)을 출력할 수 있다
- slog 레벨(Debug/Info/Error)이 stage 결과에 따라 구분된다

---

## CH13. Cost Control

CH02에서 확보한 BudgetTokens 슬롯에 실제 로직 추가.

- selective context
- token budget 적용 (BudgetTokens 초과 시 abort 또는 context 축소)
- cache
- retry 제한
- 모델 분리

### Rate Limiting

retry/backoff(CH03)는 에러 발생 후의 대응이다. Rate limiting은 에러 발생 전 속도 제어다. 두 레이어는 독립적으로 동작한다.

**두 가지 Rate Limit 구분**

| 종류 | 단위 | 제어 방법 | 설정 키 |
|---|---|---|---|
| Request/min | 요청 횟수 | token bucket | `budget.max_rpm` |
| Token/min | 입출력 토큰 합 | sliding window | `budget.max_tpm` |

config 추가:

```yaml
budget:
  max_tokens: 0       # CH13 (0 = 제한 없음)
  on_exceed: abort    # CH13
  max_rpm: 0          # 분당 최대 요청 수 (0 = 제한 없음)
  max_tpm: 0          # 분당 최대 토큰 수 (0 = 제한 없음)
```

**Token Bucket 구현**

```go
type RateLimiter struct {
    mu         sync.Mutex
    tokens     float64
    maxTokens  float64
    refillRate float64 // tokens/sec
    lastRefill time.Time
}

func (r *RateLimiter) Wait(ctx context.Context) error {
    for {
        r.mu.Lock()
        r.refill()
        if r.tokens >= 1 {
            r.tokens--
            r.mu.Unlock()
            return nil
        }
        wait := time.Duration((1-r.tokens)/r.refillRate * float64(time.Second))
        r.mu.Unlock()

        select {
        case <-ctx.Done():
            return ctx.Err() // ctx cancel 시 즉시 반환
        case <-time.After(wait):
        }
    }
}
```

API client 호출 전 `limiter.Wait(ctx)` 호출.

**429 응답 처리와의 연계**

- 429 수신 → `Retry-After` 헤더 파싱 → rate limiter의 refillRate를 해당 기간 동안 0으로 설정
- 이후 요청들이 자동으로 대기 (CH03 backoff와 이중 보호)

### 완료 기준
- BudgetTokens 초과 시 설정된 정책(abort / 축소)이 동작한다
- 저비용 모델과 고성능 모델이 작업 유형에 따라 분리 호출된다
- cache hit 시 API 호출이 발생하지 않는다
- 설정된 RPM 초과 시 요청이 대기한다
- 대기 중 ctx cancel 시 즉시 에러를 반환한다
- 429 수신 시 Retry-After 기간 동안 rate limiter가 일시적으로 중단된다

---

## CH14. 하네스 완성 (Flow 컴포넌트)

Flow = plan → apply → verify 전체 실행 흐름을 조율하는 orchestrator.

최소 구성:

- API client
- parser
- context builder
- plan gate
- apply engine
- verify / retry
- hook
- observability
- cost control

### CLI 인터페이스 설계

모든 컴포넌트가 완성된 시점에 CLI 진입점을 설계한다.

```
agent-harness run --request "..."     // 단일 요청 실행
agent-harness run --file request.txt  // 파일에서 요청 로드
agent-harness plan --request "..."    // plan 생성까지만 실행 (dry-run)
agent-harness apply --plan plan.md    // 기존 plan 적용
```

- 서브커맨드 구조 (cobra 또는 표준 flag)
- 공통 플래그: `--config`, `--dry-run`, `--verbose`
- 실행 결과는 stdout (JSON 또는 텍스트), 에러는 stderr
- exit code: 0 = 성공, 1 = 실행 실패, 2 = 설정 오류

> CLI는 Flow orchestrator의 얇은 래퍼다.
> 비즈니스 로직이 CLI 레이어에 들어오지 않도록 Flow가 모든 결정을 담당한다.

### Graceful Shutdown

CH02에서 확립한 `context.Context` + cancel 패턴을 OS 신호와 연결한다.

**신호 수신 → cancel 전파 흐름**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

go func() {
    <-sigCh
    cancel() // 전체 파이프라인에 전파
}()

err := flow.Run(ctx, request)
```

**각 단계별 중단 시점**

| 단계 | 중단 가능 여부 | 중단 시 동작 |
|---|---|---|
| Context 구성 | 즉시 가능 | 파일 읽기 중단, 상태 없음 |
| Plan 생성 | 즉시 가능 | 진행 중인 API 요청 취소 |
| Apply 실행 | **불가** | rollback 완료 후 종료 |
| Verify 실행 | 가능 | 실행 중인 프로세스 kill 후 종료 |

**Apply 중 Shutdown 처리**

Apply가 시작되면 context cancel을 무시하고 완료 또는 rollback까지 진행한다.

```go
// Apply는 부모 ctx가 cancel되어도 계속 실행
applyCtx := context.WithoutCancel(ctx) // Go 1.21+

err := applyEngine.Apply(applyCtx, patch)
if err != nil {
    // rollback은 반드시 완료 — Background context 사용
    _ = applyEngine.Rollback(context.Background())
}

// Apply 완료 후 부모 ctx 상태 확인
if ctx.Err() != nil {
    return ctx.Err() // 지연된 종료
}
```

**exit code 규칙**

| 상황 | exit code |
|---|---|
| 정상 완료 | 0 |
| 실행 실패 (apply 에러 등) | 1 |
| 설정 오류 | 2 |
| 신호 종료 (SIGINT/SIGTERM) | 130 |

### 완료 기준
- 사용자 요청 하나로 context 구성 → plan → gate → apply → verify 전 흐름이 단일 실행된다
- 각 단계 실패 시 정의된 정책(retry / abort / rollback)이 동작한다
- observability가 전 흐름에 걸쳐 기록된다
- `run` / `plan` / `apply` 서브커맨드가 동작한다
- `--dry-run` 플래그로 apply 없이 plan까지만 실행된다
- SIGTERM 수신 시 진행 중인 API 요청이 취소된다
- Apply 중 SIGTERM 수신 시 Apply가 완료(또는 rollback)된 후 종료된다
- exit code가 정상 종료(0)와 신호 종료(130)를 구분한다

---

## CH15. Multi-Agent

> **의존 관계**: CH14 Flow 완성 후 진행한다. 각 agent는 CH14 Flow의 특정 단계만 실행하는 방식으로 분리된다.

- explorer / planner / implementer / verifier
- 필요할 때만 사용

### Agent Spawn: goroutine + channel

각 agent를 goroutine으로 실행하고 결과를 channel로 수집한다.

```go
type AgentResult struct {
    Role   string
    Output any
    Err    error
}

ch := make(chan AgentResult, len(agents))

for _, agent := range agents {
    go func(a Agent) {
        out, err := a.Run(ctx, input)
        ch <- AgentResult{Role: a.Role(), Output: out, Err: err}
    }(agent)
}

// 결과 수집
for range agents {
    result := <-ch
    // 처리
}
```

- ctx는 CH02에서 확립한 인터페이스 그대로 전달 → 상위 cancel 시 모든 agent goroutine 종료
- 하나의 agent 실패 시 `cancel()` 호출로 나머지 agent도 중단 가능
- subprocess나 외부 프로세스 없이 Go 네이티브 패턴으로 구현

### 동시성 처리 (파일 단위 lock)

여러 agent가 동시에 파일을 수정하는 경우 Apply Engine에 file lock 적용.
path 기준 mutex를 획득하고 rollback은 lock 범위 안에서만 동작.

```go
type FileLockRegistry struct {
    mu    sync.Mutex
    locks map[string]*sync.Mutex
}

func (r *FileLockRegistry) Lock(path string) func() {
    // path별 mutex 획득, unlock 함수 반환
}
```

### 완료 기준
- 두 agent가 동일 파일을 동시에 수정하려 할 때 하나가 대기한다
- 각 agent 역할(explorer / planner / implementer / verifier)이 분리되어 동작한다
- 하나의 agent가 에러를 반환하면 나머지 agent가 ctx cancel로 중단된다
- 모든 agent 결과가 수집된 후 다음 단계로 진행된다

---

## CH16. 확장

### Skill Registry 완성 (CH09 연결)

CH09에서 정의한 단일 Skill 실행 계약을 registry로 운영한다.

- 등록: `registry.Register(skill)` — 중복 시 패닉
- 조회: `registry.Get(name)` — 없으면 에러 반환
- 목록: `registry.List()` — 등록된 전체 skill 이름 반환
- Flow(CH14)에서 `StepTypeSkill` step 실행 시 registry 경유

### PR 자동화

Verify 성공 후 변경 사항을 자동으로 PR로 제출한다.

```
[Verify 성공]
      │
      ▼
[git diff → branch 커밋]
      │
      ▼
[GitHub API → PR 생성]
      │
      ▼
[PR URL → Observability(CH12) 기록]
```

- credential 로드: CH02의 API key 로드 패턴과 동일 (`GITHUB_TOKEN` 환경변수 우선)
- PR 생성 실패는 PERMANENT 에러로 분류 (CH01 `HarnessError` 사용)
- `--dry-run` 플래그 시 PR 생성 없이 diff만 출력

### 리뷰 루프

PR 리뷰 코멘트를 새 요청으로 재입력해 수정 → PR 업데이트 사이클을 구성한다.

```
[PR 리뷰 코멘트 수신]
      │
      ▼
[코멘트 → 새 요청으로 변환]
      │
      ▼
[Flow(CH14) 재실행]
      │
      ▼
[Verify 성공 → PR 업데이트]
```

- 코멘트 파싱: GitHub API로 미해결 코멘트만 수집
- 코멘트가 없으면 루프 종료

### LLM Provider 추상화

현재 구현은 Claude API에 종속된다. Provider를 교체 가능하게 만들려면 CH02의 `LLMClient` 인터페이스가 이미 준비되어 있다. 이 챕터에서 실제 교체 가능성을 검증한다.

```go
// CH02에서 이미 정의된 인터페이스
type LLMClient interface {
    Call(ctx context.Context, opt CallOption, prompt string) (Response, error)
}

// 새 provider 추가 시 이 인터페이스만 구현하면 된다
type OpenAIClient struct { ... }
func (c *OpenAIClient) Call(...) (Response, error) { ... }
```

**Provider 선택 설계**

```yaml
api:
  provider: claude      # "claude" | "openai" | (추후 확장)
  model: claude-sonnet-4-6
```

```go
func NewLLMClient(cfg APIConfig) (LLMClient, error) {
    switch cfg.Provider {
    case "claude":
        return NewClaudeClient(cfg), nil
    case "openai":
        return NewOpenAIClient(cfg), nil
    default:
        return nil, fmt.Errorf("unknown provider: %s", cfg.Provider)
    }
}
```

- provider 전환 시 Flow, Parser, Context Builder는 변경 없음
- Parser(CH03)는 provider별 응답 포맷이 다르므로 provider별 파서를 별도로 구현
- `CallOption`에 provider 중립적인 필드만 남기고, provider 전용 옵션은 `Extra map[string]any`로 수용

### 언어 지원 확장 (Apply Engine + Verify)

현재 Apply Engine은 Go 전용이다 (gofmt, go vet). 다른 언어로 확장하려면 포매터와 검증기를 교체 가능하게 만들어야 한다.

```go
type LanguagePlugin interface {
    Extensions() []string        // ".py", ".ts" 등
    Format(src []byte) ([]byte, error)
    Validate(path string) error  // syntax 검증
    TestCommand() []string       // verify 시 실행할 명령
}

type PluginRegistry struct {
    plugins map[string]LanguagePlugin // ext → plugin
}

func (r *PluginRegistry) ForFile(path string) (LanguagePlugin, bool) {
    ext := filepath.Ext(path)
    p, ok := r.plugins[ext]
    return p, ok
}
```

- plugin이 없는 확장자는 포매팅 없이 그대로 적용 (Apply는 동작, Verify는 스킵)
- Go plugin: gofmt + go/parser AST 검증 (CH06 현재 구현)
- 신규 plugin 추가 시 Flow, Apply Engine 코드 변경 없음

> 이 패턴을 구현하면 CH06의 Go 전용 코드가 얼마나 잘 분리되어 있는지 자연스럽게 검증된다.

### 완료 기준
- skill registry에 skill을 등록하고 CH14 Flow에서 step으로 실행된다
- 존재하지 않는 skill 이름 조회 시 에러를 반환한다
- Verify 성공 후 자동으로 PR이 생성되고 URL이 기록된다
- 리뷰 코멘트를 입력으로 받아 수정 → PR 업데이트 사이클이 동작한다
- `--dry-run` 플래그 시 PR 생성 없이 diff만 출력된다
- `api.provider` 설정 변경만으로 LLM 구현체가 교체되고 Flow는 변경되지 않는다
- 새 LanguagePlugin을 등록하면 해당 확장자 파일에 포매팅과 검증이 적용된다
- plugin이 없는 확장자 파일에 Apply가 정상 동작한다

---

# 🔥 핵심 규칙 (이 구조 유지 이유)

이 순서는 "막히는 순서" 기준이다:

1. API 호출 안됨 → CH02
2. 응답 못 씀 → CH03 (Parsing 포함)
3. context 이상 → CH04
4. plan 잘못됨 → CH05
5. 코드 깨짐 → CH06
6. 테스트 실패 루프 → CH07
7. 위험 실행 → CH08
8. 반복 작업 → CH09
9. 외부 연결 → CH10
10. 상태 안 보임 → CH12
11. 비용 터짐 → CH13
12. 전체 통합 → CH14
13. 복잡도 증가 → CH15 (CH14 이후, Flow 완성 후 진행)

---

# 📌 결론

- Parsing은 CH03에 포함 (고정)
- CallOption.BudgetTokens는 CH02에서 구조 정의, CH13에서 로직 (고정)
- Plan Gate 정책은 외부 설정 파일로 분리 (고정)
- Apply Engine = unified diff + AST 검증 전용 (고정)
- Apply Engine = atomic write 기반 rollback (고정)
- Hook 인터페이스는 CH06에서 stub 정의, CH08에서 구현 (고정)
- Plan = CH05 (고정)
- Apply = CH06 (고정)
- Verify = CH07 (고정)
- Flow orchestrator = CH14 (고정)
- 인터페이스 설계 원칙은 CH01에서 확립, 전 챕터에서 적용 (고정)
- 테스트 인프라(Mock 서버, Golden File)는 CH02에서 구축, 전 챕터에서 재사용 (고정)
- Rate Limiting = CH13 (고정, budget 섹션과 통합)
- Graceful Shutdown = CH14 (고정, Flow 완성과 동시 구현)
- Tool 응답 신뢰 경계 = CH10 (고정, CH03 LLM 출력 신뢰 경계와 동일 원칙)
