# AI Agent Harness — Extensions (CH14~CH19)

> CH13 완성 이후 진행하는 확장 챕터입니다.
> 핵심 파이프라인은 [PLAN-core.md](PLAN-core.md)를 참조하세요.

---

## CH14. Multi-Agent

> **의존 관계**: CH13 Flow 완성 후 진행한다. 각 agent는 CH13 Flow의 특정 단계만 실행하는 방식으로 분리된다.

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

### Agent 간 결과 합성

각 agent는 독립 실행이지만, planner → implementer처럼 순서가 있는 경우
이전 agent의 출력을 다음 agent의 입력으로 주입한다.

**순차 파이프라인 패턴**

agent가 역할별로 순서가 정해진 경우:

    [Explorer] → files []string
         │
         ▼
    [Planner]  → plan Plan       ← Explorer 결과를 context에 주입
         │
         ▼
    [Implementer] → patch Diff   ← Plan을 입력으로 사용
         │
         ▼
    [Verifier]    → VerifyResult

각 단계의 출력 타입을 명시하고, 다음 agent 생성 시 생성자에 주입:

```go
type PipelineInput struct {
    Request string
    Files   []string  // Explorer가 채움
    Plan    *Plan     // Planner가 채움
    Patch   *Diff     // Implementer가 채움
}
```

**병렬 실행 가능 조건**

동일 파이프라인 단계에서만 병렬화한다.
파일 lock(FileLockRegistry)이 같은 파일에 대한 경쟁을 직렬화한다.

### 부분 실패 처리

agent 하나가 실패했을 때 전체를 abort할지, 나머지 결과를 사용할지를 명시.

```go
type AgentPolicy string

const (
    AgentPolicyFailFast   AgentPolicy = "fail-fast"   // 하나 실패 → 전체 cancel
    AgentPolicyBestEffort AgentPolicy = "best-effort" // 나머지 결과 사용
)
```

- plan / apply 단계는 `fail-fast` (부분 적용은 더 위험)
- explore 단계는 `best-effort` 허용 (일부 파일 분석 실패해도 진행 가능)
- 선택된 정책은 Observability(CH11)에 기록

### Agent Role Contract (역할 계약)

각 agent가 무엇을 입력받고 무엇을 출력해야 하는지를 타입으로 강제한다. "역할"이 인터페이스가 되지 않으면 pipeline 조합이 불가능하다.

```go
type ExplorerOutput struct {
    RelevantFiles []string
    Summaries     map[string]string // file → one-line summary
}

type PlannerInput struct {
    Request string
    Files   ExplorerOutput // Explorer 결과를 그대로 받음
}

type PlannerOutput struct {
    Plan Plan
}
```

agent 간 데이터 계약이 느슨하면(`map[string]any` 등) pipeline 중간에 필드 누락이 런타임에야 발견된다.

### 부분 실패 상태 관리

Explorer 3개 중 2개가 성공했을 때, Planner가 받는 입력은 어떻게 처리하는가?

- **fail-fast**: Explorer 하나 실패 → 전체 취소 → Planner 실행 안 함
- **best-effort**: 성공한 2개만 가지고 Planner 실행

best-effort에서 Planner는 입력이 불완전하다는 것을 알아야 한다:

```go
type AgentResult[T any] struct {
    Value   T
    Err     error
    Partial bool // true = 일부 입력 누락 상태에서 생성된 결과
}
```

이 설계가 없으면 불완전한 context로 생성한 plan이 완전한 plan인 척 Apply까지 진행된다.

### Implementer 간 Patch 충돌 감지

여러 Implementer가 서로 다른 파일을 수정하면 문제 없다. 같은 파일의 같은 영역을 수정하면 conflict가 발생한다.

```go
type PatchConflict struct {
    File  string
    Hunk1 Hunk // Implementer A의 변경
    Hunk2 Hunk // Implementer B의 변경
}

func detectConflicts(patches []Patch) []PatchConflict {
    // 같은 파일, 겹치는 line range → conflict
}
```

conflict 발생 시 정책:
- `serial`: 하나씩 순서대로 적용 (나중 것이 앞 것을 덮어씀)
- `abort`: conflict 시 전체 취소

> `FileLockRegistry`는 "동시에 write하지 않도록" 보호한다.
> patch 충돌은 그보다 앞 단계 — "같은 영역을 수정하려는 의도"의 충돌이다.

### Agent 실행 비용 배분

여러 agent가 각자 LLM을 호출한다. CH12의 token budget을 어떻게 배분하는가?

```go
type AgentBudget struct {
    Total    int            // 전체 가능 토큰
    PerAgent map[string]int // role → 할당량
    Overflow string         // "steal" | "abort"
}
```

CH12에서 설계한 `RateLimiter`를 agent별로 인스턴스화하는 방식으로 구현한다. 이 연결이 없으면 CH12와 CH14가 단절된 챕터로 남는다.

```go
// CH12 RateLimiter를 agent role별로 인스턴스화
func buildAgentLimiters(budget AgentBudget) map[string]*RateLimiter {
    limiters := make(map[string]*RateLimiter, len(budget.PerAgent))
    for role, tokens := range budget.PerAgent {
        limiters[role] = NewRateLimiter(
            float64(tokens),          // bucket 크기 = 할당 토큰
            float64(tokens)/60.0,     // refill = 분당 할당량 균등 분배
        )
    }
    return limiters
}

// Flow 초기화 시 agent별 limiter 주입
limiters := buildAgentLimiters(cfg.AgentBudget)
agents := []Agent{
    NewExplorerAgent(ctx, flow, limiters["explorer"]),
    NewPlannerAgent(ctx, flow, limiters["planner"]),
    NewImplementerAgent(ctx, flow, limiters["implementer"]),
}
```

**Overflow 정책 처리**

```go
func (a *Agent) checkBudget(ctx context.Context, tokensNeeded int) error {
    if a.limiter == nil {
        return nil // 제한 없음
    }
    for i := 0; i < tokensNeeded; i++ {
        if err := a.limiter.Wait(ctx); err != nil {
            if a.budget.Overflow == "steal" {
                // 전체 budget pool에서 차감 시도 (구현 생략)
                return nil
            }
            return &HarnessError{
                Class:   ErrorClassPermanent,
                Stage:   "agent/budget",
                Wrapped: fmt.Errorf("agent[%s] token budget exhausted", a.role),
            }
        }
    }
    return nil
}
```

> agent별 limiter는 CH12의 `RateLimiter` 구조체를 그대로 재사용한다.
> `NewRateLimiter`의 시그니처는 CH12 구현 시 확정된다.

### 완료 기준
- [I] 두 agent가 동일 파일을 동시에 수정하려 할 때 하나가 대기한다
- [I] 각 agent 역할(explorer / planner / implementer / verifier)이 분리되어 동작한다
- [I] 하나의 agent가 에러를 반환하면 나머지 agent가 ctx cancel로 중단된다
- [I] 모든 agent 결과가 수집된 후 다음 단계로 진행된다
- [I] planner agent가 explorer 결과를 입력으로 받아 plan을 생성한다
- [I] `fail-fast` 정책에서 하나의 agent 실패 시 나머지가 ctx cancel로 중단된다
- [I] `best-effort` 정책에서 일부 agent 실패 시 성공한 결과만 수집해 다음 단계로 진행된다
- [I] 병렬 agent가 동일 파일을 동시에 수정하려 할 때 하나가 lock을 대기한다
- [I] Explorer 일부 실패 시 best-effort/fail-fast 정책에 따라 Planner 입력이 결정된다
- [U] Implementer 간 동일 파일 동일 영역 수정 충돌이 apply 전에 감지된다
- [I] 각 agent의 token 사용량이 CH11 observability에 agent별로 분리 기록된다
- [U] 부분 성공으로 생성된 결과에 `Partial: true` 플래그가 포함된다
- [I] ctx cancel 후 모든 agent goroutine이 누수 없이 종료된다 (`goleak` 또는 `t.Cleanup`으로 검증)

---

## CH15. Skill Registry 운영

> **전제**: CH09 (Skill 인터페이스), CH13 (Flow에서 skill step 실행)

CH09에서 정의한 단일 Skill 실행 계약을 registry로 운영한다.

### CH09와의 역할 분리

| | CH09 | CH15 |
|---|---|---|
| 관심사 | 단일 Skill 실행 계약 | Skill 생태계 운영 |
| 핵심 문제 | Execute + 검증 | 등록 / 조회 / 의존성 |

### Skill Registry 완성

- 등록: `registry.Register(skill)` — 중복 시 패닉
- 조회: `registry.Get(name)` — 없으면 에러 반환
- 목록: `registry.List()` — 등록된 전체 skill 이름 반환
- Flow(CH13)에서 `StepTypeSkill` step 실행 시 registry 경유

### Skill 의존성 선언

skill이 다른 skill을 호출해야 할 때 순환 의존성을 막아야 한다.

```go
type Skill struct {
    Name       string
    Idempotent bool
    Depends    []string // 이 skill이 의존하는 skill 이름 목록
    Input      Schema
    Output     Schema
    Execute    func(ctx context.Context, input Input) (Output, error)
}
```

등록 시 의존 skill이 이미 등록되어 있는지 확인한다. 순환 의존성 감지: DFS로 사이클 탐지 → 패닉 (초기화 시점 오류).

### 추가 Skill 구현

CH09의 2개에 이어 실용적인 skill을 추가하면서 인터페이스 설계를 검증한다:

| Skill | Idempotent | 설명 |
|---|---|---|
| `file.glob` | true | 패턴 매칭 파일 목록 반환 |
| `git.diff` | true | 현재 staged diff 반환 |
| `git.commit` | false | 변경사항 커밋 |
| `json.query` | true | JQ-style JSON 필드 추출 |

### 완료 기준
- [U] 의존 skill이 없는 skill 등록 시도 시 에러를 반환한다
- [U] 순환 의존 관계가 있는 skill 등록 시 초기화 단계에서 패닉이 발생한다
- [U] `registry.List()`가 등록된 skill 전체 이름을 반환한다
- [I] `file.glob` skill이 glob 패턴으로 파일 목록을 반환한다
- [U] `git.commit` skill이 Idempotent=false로 등록되어 실패 시 retry 없이 abort된다

---

## CH16. GitHub 통합

> **전제**: CH13 (Flow 완성), CH07 (Verify 완료 기준)

Verify 성공 후 변경 사항을 자동으로 PR로 제출하고, 리뷰 코멘트를 새 요청으로 재입력하는 수정 사이클을 구성한다.

### PR 자동화 흐름

```
[Verify 성공]
      │
      ▼
[git diff → feature branch 커밋]
  - branch명: harness/{request-hash}
  - commit message: LLM이 생성한 요약
      │
      ▼
[GitHub API: PR 생성]
  - base: main
  - head: harness/{request-hash}
  - title / body: LLM이 생성한 요약 사용
      │
      ▼
[PR URL → CH11 Observability 기록]
```

### PR 생성 실패 분류

| 상황 | ErrorClass |
|---|---|
| GITHUB_TOKEN 없음 | PERMANENT |
| 이미 같은 branch의 PR 존재 | PERMANENT |
| GitHub API 5xx | TRANSIENT |
| 네트워크 단절 | TRANSIENT |

### 리뷰 루프

PR 리뷰 코멘트를 새 요청으로 재입력해 수정 → PR 업데이트 사이클을 구성한다.

```go
type ReviewLoop struct {
    ghClient  GitHubClient
    flow      *Flow
    maxRounds int // 기본 3
}

func (l *ReviewLoop) Run(ctx context.Context, prNumber int) error {
    for round := 0; round < l.maxRounds; round++ {
        comments, err := l.ghClient.UnresolvedComments(ctx, prNumber)
        if err != nil {
            return err
        }
        if len(comments) == 0 {
            return nil // 모든 코멘트 해결됨
        }

        request := commentsToRequest(comments)
        if err := l.flow.Run(ctx, request); err != nil {
            return err
        }
        // 동일 branch에 force push → PR 자동 업데이트
    }
    return fmt.Errorf("review loop exhausted after %d rounds", l.maxRounds)
}
```

- credential 로드: `GITHUB_TOKEN` 환경변수 우선 → config 파일 fallback (CH02 패턴 동일)
- `--dry-run` 플래그: PR 생성 없이 diff만 stdout 출력

### 완료 기준
- [I] Verify 성공 후 feature branch가 생성되고 PR이 올라간다
- [U] 같은 branch의 PR이 이미 존재하면 PERMANENT 에러를 반환한다
- [M] GitHub API 5xx 시 TRANSIENT로 분류해 retry한다
- [M] 미해결 리뷰 코멘트를 수집해 Flow를 재실행한다
- [U] 코멘트가 없으면 루프를 종료한다
- [U] `maxRounds` 초과 시 PERMANENT 에러를 반환한다
- [I] `--dry-run` 시 PR 생성 없이 diff만 출력한다

---

## CH17. LLM Provider 추상화

> **전제**: CH02 (LLMClient 인터페이스), CH03 (Parser), CH13 (Flow 통합)

현재 구현은 Claude API에 종속된다. CH02의 `LLMClient` 인터페이스를 활용해 provider를 교체 가능하게 만들고 실제 교체 가능성을 검증한다.

### Provider별 차이 처리

| 차이 | 처리 방법 |
|---|---|
| 응답 JSON 구조 | provider별 Parser 구현 |
| tool_use 블록 포맷 | provider별 ToolFormatter |
| streaming 포맷 | provider별 StreamReader |
| 에러 코드 의미 | provider별 ErrorClassifier |

### LLMProvider 인터페이스

```go
type LLMProvider interface {
    LLMClient
    ParseResponse(raw []byte) (Response, error)
    FormatToolUse(blocks []ToolUseBlock) []byte
    ClassifyError(statusCode int, body []byte) ErrorClass
}

func NewLLMProvider(cfg APIConfig) (LLMProvider, error) {
    switch cfg.Provider {
    case "claude":
        return NewClaudeProvider(cfg), nil
    case "openai":
        return NewOpenAIProvider(cfg), nil
    default:
        return nil, fmt.Errorf("unknown provider: %s", cfg.Provider)
    }
}
```

### CallOption 중립화

provider 전용 옵션을 `Extra`로 격리한다:

```go
type CallOption struct {
    Model       string
    MaxTokens   int
    Temperature float64
    // provider 중립 필드만 여기에

    Extra map[string]any
    // Claude: {"thinking": {"type": "enabled"}}
    // OpenAI: {"response_format": {"type": "json_object"}}
}
```

config에 provider 선택 추가:

```yaml
api:
  provider: claude   # "claude" | "openai"
  model: claude-sonnet-4-6
```

### 완료 기준
- [M] config의 `api.provider: openai`로 변경만으로 OpenAI로 전환된다
- [U] Flow, ContextBuilder, PlanGate 코드를 수정하지 않아도 동작한다
- [U] provider별 응답 포맷 차이가 각 provider의 ParseResponse에서 흡수된다
- [U] Extra 필드의 unknown key는 경고 로그 후 무시된다
- [U] 지원하지 않는 provider 값 입력 시 PERMANENT 에러를 반환한다

---

## CH18. 언어 플러그인 시스템

> **전제**: CH06 (Apply Engine), CH07 (Verify / TestCommand)

현재 Apply Engine은 Go 전용이다(`gofmt`, `go vet`). 다른 언어로 확장하려면 포매터와 검증기를 교체 가능하게 설계해야 한다.

### LanguagePlugin 인터페이스

```go
type LanguagePlugin interface {
    Extensions() []string              // [".go"], [".py", ".pyw"]
    Format(src []byte) ([]byte, error) // 포매터 (gofmt, black 등)
    Validate(path string) error        // syntax 검증 (go vet, mypy 등)
    TestCommand() []string             // ["go", "test", "./..."]
}

type PluginRegistry struct {
    plugins map[string]LanguagePlugin  // ".go" → GoPlugin
}

func (r *PluginRegistry) ForFile(path string) (LanguagePlugin, bool) {
    ext := filepath.Ext(path)
    p, ok := r.plugins[ext]
    return p, ok
}
```

### Apply Engine 연동

```go
func (e *ApplyEngine) Apply(ctx context.Context, patch Patch) error {
    content, err := e.applyDiff(patch)
    if err != nil {
        return err
    }

    if plugin, ok := e.plugins.ForFile(patch.Path); ok {
        content, err = plugin.Format(content)
        if err != nil {
            return &HarnessError{Class: ErrorClassSyntax, Stage: "format", Wrapped: err}
        }
        if err := plugin.Validate(patch.Path); err != nil {
            return &HarnessError{Class: ErrorClassSyntax, Stage: "validate", Wrapped: err}
        }
    }
    // plugin 없는 확장자: 포매팅/검증 없이 그대로 적용

    return atomicWrite(patch.Path, content)
}
```

### 내장 플러그인

| 플러그인 | Format | Validate | TestCommand |
|---|---|---|---|
| GoPlugin | `gofmt` | `go vet` | `go test ./...` |
| TypeScriptPlugin | `prettier` | `tsc --noEmit` | `npm test` |
| PythonPlugin | `black` | `mypy` | `pytest` |

### 완료 기준
- [I] `.go` 파일에는 GoPlugin이 자동으로 적용된다
- [I] 등록된 플러그인이 없는 확장자의 파일은 포매팅 없이 그대로 적용된다
- [I] Format 실패 시 ErrorClassSyntax로 분류되어 CH07 retry 경로를 탄다
- [U] 외부 포매터(gofmt 등)가 없는 환경에서 플러그인 초기화 시 에러를 반환한다
- [I] CH07의 TestCommand가 플러그인에서 동적으로 조회된다

---

## CH19. Observability 완성 (OTel 연동)

> **전제**: CH11 (SpanRecorder + Recorder 인터페이스), CH13 (Flow에 Recorder 주입)

CH11에서 SpanRecorder로 내부 관측을 구현했다. 이 챕터에서 외부 시스템으로 내보내는 경로를 추가한다. CH11에서 OTel 호환 구조로 설계했으므로 코드 변경이 최소화된다.

### OtelRecorder 구현

CH11의 `Recorder` 인터페이스를 OTel Tracer로 구현한다:

```go
type OtelRecorder struct {
    tracer trace.Tracer
}

func (r *OtelRecorder) Record(ctx context.Context, stage string, fn func(ctx context.Context) error) error {
    ctx, span := r.tracer.Start(ctx, stage)
    defer span.End()

    err := fn(ctx)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
    }
    return err
}
```

### Exporter 선택 (config)

```yaml
observability:
  exporter: "stdout"   # "stdout" | "otlp" | "none"
  otlp_endpoint: ""    # exporter=otlp 시 필수
```

### Flow 초기화에서 Recorder 선택

```go
var recorder Recorder
switch cfg.Observability.Exporter {
case "stdout":
    recorder = NewSpanRecorder()         // CH11 구현체
case "otlp":
    recorder = NewOtelRecorder(cfg.Observability.OtlpEndpoint)
default:
    recorder = &NoopRecorder{}
}

flow := NewFlow(..., recorder)
```

Flow 코드는 변경 없음 — `Recorder` 인터페이스 교체만으로 동작한다.

### 메트릭 추가 (Prometheus)

타이밍(trace) 외에 카운터 메트릭을 추가한다:

```go
var (
    stageTotal  = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "harness_stage_total"},
        []string{"stage"},
    )
    stageErrors = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "harness_stage_errors_total"},
        []string{"stage", "class"},
    )
    tokensUsed  = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{Name: "harness_tokens_used"},
        []string{"direction"}, // "in" | "out"
    )
)
```

### 완료 기준
- [I] `exporter: stdout` 설정 시 각 stage의 trace가 stdout에 출력된다
- [I] `exporter: otlp` 설정 시 Jaeger UI에서 전체 파이프라인 trace를 볼 수 있다
- [U] Flow 코드를 변경하지 않고 exporter 교체가 가능하다
- [I] 각 stage의 실행 횟수와 에러 횟수가 Prometheus 메트릭으로 노출된다
- `exporter: none` 설정 시 observability 오버헤드가 없다

---
