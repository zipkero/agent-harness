# agent-harness IMPLEMENT

PLAN.md의 Phase 1~3 Exit Criteria를 달성하기 위한 실행 전략 + 진행 추적 문서.
Phase 4(Extension)는 MVP 범위 외이므로 본 문서에서 제외한다.

진행 규칙: 구현 완료 → 본 문서 체크. PLAN 체크는 Phase 검증 기준 통과 후에만.

---

## 아키텍처

```
cmd/agent-harness            # CLI entrypoint, flag 파싱
└─ internal/flow             # 파이프라인 오케스트레이터 (유일한 조립점)
   ├─ internal/config        # YAML 로드 + 검증
   ├─ internal/harness       # HarnessError, Stage 상수
   ├─ internal/context       # 프롬프트 조립 + 축소
   │  └─ internal/token      # 토큰 근사 + Calibrator (공용)
   ├─ internal/api           # LLMProvider 인터페이스 + 공통 타입
   │  └─ internal/api/claude # Claude 구현 (SSE, wire 변환)
   ├─ internal/parser        # 응답에서 diff/fullfile 추출
   ├─ internal/plan          # Plan 생성 + PlanGate
   ├─ internal/apply         # diff 적용 + 트랜잭션
   │  └─ internal/apply/atomic # atomic write
   ├─ internal/verify        # go test/vet + RetryState
   ├─ internal/hook          # pre/post 정책
   ├─ internal/skill         # Skill Registry + file_read, shell_run
   ├─ internal/tool          # ToolExecutor (tool_use 블록 실행)
   ├─ internal/observe       # Recorder, audit
   └─ internal/cost          # SessionBudget + RateLimiter
```

**경계 원칙**
- flow 외 패키지는 서로를 직접 조립하지 않는다. 모든 DI는 flow에서.
- harness(error)는 bottom layer. 어떤 패키지도 재래핑 금지.
- token은 context/cost/parser 공용 유틸. 상태 없음(Calibrator는 세션 스코프).
- provider 교체 가능성은 api 인터페이스로만 담보. 상위 레이어는 claude를 모른다.

## 실행 흐름

```
CLI(run) → Config.Load → lock 획득 → dirty workdir 체크 → txlog 복구
        → ContextBuilder.Build(request) → Budget.PreCheck
        → Provider.Call(plan tool) → PlanParser → Plan.Request 주입 → PlanGate
        → for step in Plan.Steps:
             skill  → SkillRegistry.Execute
             llm    → toolUseLoop(diff_model):
                        Provider.Call → end_turn?
                          no(tool_use)  → ToolExecutor → append result → 재호출
                          yes            → Parser.ExtractDiff
                      → Hook.Pre → Apply → Format → Verify
                          PASS → Hook.Post → 다음 step
                          FAIL → Rollback → RetryState → retry or abort
        → Recorder.Summary → lock 해제 → exit
```

**주요 분기**
- Budget 초과 → `on_exceed`: abort / shrink(1회 재시도)
- Verify FAIL → error signature 2회 누적 또는 MaxRetry 초과 시 abort
- diff 2회 실패 + 토큰 여유 → fullfile fallback (파일 단위)
- SIGINT/SIGTERM → 진행 중 apply 완료 또는 rollback 후 exit 130

## 상태 모델

| 상태 | 저장 위치 | 수명 | 변화 규칙 |
|---|---|---|---|
| lock | `.harness/lock.json` | 실행 중 | 획득→해제. stale(PID 죽음 + >60분) 시 재획득 |
| txlog | `.harness/txlog.jsonl` | apply 시작~commit/rollback | prepared → file_done* → committed. crash 시 재실행이 자동 rollback |
| backup | `.harness/backup/{path}` | txlog prepared~committed | apply 전 원본 복사. committed 후 삭제 |
| audit | `.harness/audit/{ts}.jsonl` | 영구 | audit.enabled 시 append only |
| Conversation | 메모리 | llm step 1회 | step 시작 시 생성, tool 루프 간 누적, step 종료 시 폐기 |
| RetryState | 메모리 | llm step 1회 | Attempt/SigCount 기록, abort 판정 |
| SessionBudget | 메모리 | CLI 실행 1회 | PreCheck/Record 누적 |
| Calibrator 계수 | 메모리 | CLI 실행 1회 | API usage 실측과 근사 비교하여 세션 중 3회까지 갱신 |

---

## 구현 순서

의존성 기준. 각 섹션 앞에 선행 조건 1줄.

### Phase 1 — Plan 생성 파이프라인
> 선행: 없음. 이 Phase의 Exit은 PLAN `TestPlanGeneration` (mock HTTP) 통과.

- [ ] **1. 에러 + 공통 타입 기반 (harness)**
  - 목적: 모든 레이어가 공유하는 에러 분류를 먼저 확정해야 이후 패키지가 일관 에러를 반환할 수 있음 → PLAN: Phase 1-1 > 에러 시스템
  - 책임: 입력 `(Class, Stage, wrapped)` → 출력 `HarnessError`. 경계: 에러 생성은 발생 지점 전용, 상위 재래핑 금지
  - 설계: `HarnessError{Class, Stage, Err}` + `Unwrap/Is/As`. Stage/Class 상수 테이블.
  - 선택 이유: Class 3종(TRANSIENT/PERMANENT/SYNTAX)이 retry 상태 머신의 입력. 문자열 매칭 대신 타입으로 분류해야 Verify/Flow가 단일 기준으로 분기 가능.
  - 실패/예외: wrap 대상이 이미 HarnessError인 경우 Class 덮어쓰지 않고 그대로 전파.

- [ ] **2. Config 로드 + .harness 초기화 + testdata 인프라 (config, testdata/)**
  - 목적: 모든 후속 패키지가 Config 구조체 의존. .harness/ 부재 시 lock/txlog 동작 불가. Phase 1 통합 테스트 fixture도 같은 경계에서 완비되어야 이후 단위가 독립 검증 가능 → PLAN: Phase 1-1 > Config / `.harness/` 초기화 / 테스트 인프라
  - 책임: 입력 `--config`/`--root` → 출력 검증된 `Config` + `.harness/` 디렉터리 + `testdata/{project,config,api,diffs}/` fixture. 경계: 시크릿(API key) 미포함.
  - 설계:
    - Config: yaml.v3 strict → unknown field 경고 → 관대 재파싱 → `applyDefaults` → `errors.Join`으로 섹션 검증 집계. version 필드 == "1" 외 PERMANENT.
    - testdata: `project/`(최소 Go 모듈 + CLAUDE.md), `config/`(valid.yaml, minimal.yaml), `api/plan_response.json`(create_plan tool_use fixture), `diffs/`(valid_add_function, fuzz_offset, invalid_syntax, context_mismatch 4종).
  - 선택 이유: strict + 관대 재파싱 조합은 사용자 오타 감지와 forward-compat을 동시 확보. testdata를 config와 동일 경계에 묶는 이유는 Phase 1 Exit(`TestPlanGeneration`)이 fixture 없이 불가능하고, Phase 2/3의 diff 테스트도 같은 testdata를 재활용하기 때문.
  - 실패/예외: 필수 필드 누락/타입 불일치 → PERMANENT. 미래 version → PERMANENT. 디렉터리 생성 실패 → PERMANENT.

- [ ] **3. 토큰 카운팅 공용 유틸 (token)**
  - 목적: Context/Cost/Parser가 동일 근사 함수를 공유해야 예산 계산이 일관됨 → PLAN: Phase 1-4 > 토큰 카운팅
  - 책임: 입력 `string` → 출력 근사 토큰 수 + Calibrator(세션 보정 계수). 경계: 상태는 Calibrator에만.
  - 설계: `Estimate(s)`: `len(bytes)/4`, CJK rune 비율 ≥30%면 `/3` 보정. `Calibrator.Update(estimated, actual)` 세션 3회 제한.
  - 선택 이유: tokenizer 외부 의존 없음으로 빌드 단순. 근사값이기에 safety margin 20%로 오차 흡수. CJK 보정은 한국어 프로젝트 특성상 필수.
  - 실패/예외: 보정 계수 범위 벗어나면(예: <0.3 / >3.0) 무시하고 이전 값 유지.

- [ ] **4. LLMProvider 인터페이스 + 공통 타입 (api)**
  - 목적: 하네스 상위가 provider-agnostic이려면 변환 경계를 먼저 고정해야 함. NoopLimiter도 여기서 공급 → PLAN: Phase 1-2 > LLMProvider 인터페이스 / RateLimiter 인터페이스
  - 책임: 입력 `[]Message + CallOption` → 출력 `Response | ResponseStream`. 경계: wire 포맷 미노출, CallOption.stream 필드 없음(메서드 선택으로 제어).
  - 설계: `LLMProvider{Call, CallStream, EnvKey}`. `Conversation{System, Messages, Append*, Shrink, EstimateTokens}`. `RateLimiter{Wait, OnThrottle}` + NoopLimiter. Conversation.Shrink 메서드 **시그니처만 정의**, 실제 축소 알고리즘은 DP-2 확정 후 본 단위에 반영.
  - 선택 이유: stream 여부를 메서드로 나눠야 타입 시스템이 ResponseStream 사용을 강제. System을 Messages 밖에 두면 Claude(system 파라미터)/OpenAI(role system) 양쪽 매핑 단순.
  - 실패/예외: Conversation.Append는 role 교차 검증 없음(LLM 측 허용). Shrink 실패 시 원본 유지.

- [ ] **5. Claude Provider 구현 + Mock HTTP 인프라 (api/claude)**
  - 목적: Phase 1 통합 테스트의 실제 호출 경로. SSE·retry·에러 분류가 여기서 끝나야 상위 레이어가 단순화됨. Mock HTTP 서버는 본 단위의 테스트 수단이자 이후 상위 레이어(Plan/Flow) 테스트의 공통 fake → PLAN: Phase 1-2 > ClaudeProvider / 테스트 인프라
  - 책임: 입력 공통 Message/CallOption → 출력 공통 Response/Stream. 경계: SSE 파싱, wire 변환, API key 로드, retry, `httptest.Server` 기반 Mock(호출 횟수 카운팅, fixture 응답, 에러 시나리오 주입)은 모두 이 패키지.
  - 설계:
    - HTTP client + `POST /v1/messages`. base_url/timeout config 기반.
    - SSE 이벤트 흐름: message_start → (content_block_*) → message_delta → message_stop. input_json_delta는 content_block_stop 시점 1회 파싱.
    - 에러 분류: 5xx/timeout → TRANSIENT; 429 → `Retry-After` 기반 대기 후 TRANSIENT; 4xx → PERMANENT.
    - Retry: exp backoff 1s→30s, jitter ±20%, `api.max_retries`.
    - API key: `ANTHROPIC_API_KEY` env. 미설정 → PERMANENT.
  - 선택 이유: 스트림 재개 대신 전체 재시도 — Anthropic API에 서버측 재개 없음, 부분 응답 폐기가 단순·안전. allow-list 방식의 에러 분류는 신규 에러 코드를 TRANSIENT로 오분류할 위험 제거.
  - 실패/예외: overloaded error 이벤트 → TRANSIENT. invalid_request → PERMANENT. ctx cancel → 즉시 리소스 반환.

- [ ] **6. 응답 Parser (parser)**
  - 목적: LLM 출력은 외부 입력 신뢰 수준. 추출 로직을 단일 지점에 고정해야 안전 경계가 성립 → PLAN: Phase 1-3 > ResponseParser
  - 책임: 입력 `Response` → 출력 diff 텍스트 또는 fullfile 텍스트. 경계: 실제 diff 문법 파싱은 Apply 담당.
  - 설계: 코드블록 strip(` ```diff / ``` `) → 크기 상한 체크 → 빈 결과 거부. fullfile도 동일 strip.
  - 선택 이유: 추출과 문법 파싱을 분리해야 Phase 2 go-gitdiff 교체/업그레이드 영향을 parser에 미치지 않음.
  - 실패/예외: 크기 초과/빈 diff/코드블록 부재 → SYNTAX(Stage: "parse/diff" 또는 "parse/fullfile").

- [ ] **7. ContextBuilder + 프롬프트 조립 (context)**
  - 목적: 토큰 예산 내 system/context/task 3블록을 조립하고 축소 순서를 단일화 → PLAN: Phase 1-4 > ContextBuilder
  - 책임: 입력 `request + ScanRoot + Config` → 출력 Conversation(System 포함) + retry hint 누적. 경계: CLAUDE.md 탐색/제외 파일 적용/토큰 예산 분배.
  - 설계:
    - 블록: system(역할·출력형식·금지), context(CLAUDE.md + 선택 파일), task(request + plan + hint).
    - 파일 선택: plan target → 키워드 매칭 → 크기 작은 순. `.gitignore`/`deny_context_paths` 존중.
    - 예산: `context_window * 0.8`. 고정블록 산출 후 잔여를 context(67%)/history(33%) 배분. diff_context_window 있으면 그 값 기준.
    - 축소 순서: few-shot 제거 → 관련 낮은 파일 제거 → 앞 N줄 + 잘림 표시. Conversation.Shrink는 자체 히스토리만.
    - System prompt 분기: plan/diff/fullfile 3종. retry hint 템플릿 3종(apply/format/verify).
  - 선택 이유: ContextBuilder가 context만, Conversation이 history만 책임지면 축소 로직이 서로 침범하지 않음. 파일 선택 MVP를 휴리스틱으로 한정해 초기 복잡도 억제.
  - 실패/예외: CLAUDE.md 부재 → 무시. 고정블록만으로 예산 초과 → PERMANENT(설정 오류).

- [ ] **8. Plan 생성 + PlanGate (plan)**
  - 목적: Phase 1 검증 기준의 최종 출력. LLM의 tool_use 응답을 Plan으로 신뢰 경계 통과시키는 마지막 단계 → PLAN: Phase 1-5 > Plan / PlanGate
  - 책임: 입력 Conversation + tools=[create_plan] → 출력 검증된 `Plan`. 경계: request 주입은 하네스, Gate 통과 전 Plan 사용 금지.
  - 설계:
    - tool_use(create_plan) 강제. stop_reason 검증, 복수 create_plan 블록 → SYNTAX retry.
    - 스키마 검증(Step.Type/Ops/skill_name 조건부). `plan.Request = request` 주입.
    - PlanGate: deny_paths(glob), deny_ops 교차, 하네스 소스 차단, 명령 패턴 차단(`rm -rf`, `curl | sh`).
    - on_violation abort→PERMANENT, warn→로그만.
  - 선택 이유: structured output을 tool_use로 강제해야 free text JSON 파싱 실패 루프를 회피. Gate 분리로 "plan 내용 신뢰 0" 경계 유지.
  - 실패/예외: SYNTAX retry는 `api.max_retries`까지. Gate 위반 abort 시 Plan 반환하지 않음.

**Phase 1 통과 확인**: `TestPlanGeneration` (mock HTTP, fixture 기반) 통과 → PLAN Phase 1 체크.

---

### Phase 2 — 단일 파일 diff 적용 + 검증
> 선행: Phase 1 통과 확인 블록 전체 회귀(`TestPlanGeneration` 재실행 + Phase 1 단위 테스트 전수) 통과. Exit은 PLAN의 fixture 기반 통합 테스트(LLM 호출 없음).

- [ ] **9. Atomic write + ApplyEngine + txlog (apply)**
  - 목적: LLM diff 부정확성 + 원자성 보장을 단일 경계로 묶음 → PLAN: Phase 2-1 > ApplyEngine
  - 책임: 입력 diff text + target files → 출력 변경 완료 or 완전 rollback. 경계: gofmt/AST 검증 포함, Hook 호출은 flow 책임.
  - 설계:
    - Parse: go-gitdiff → hunk 구조.
    - Apply: line 단위, 이전 hunk offset 보정, context line fuzz ±3 허용(경고 로그).
    - AST: `.go` 파일 go/parser 검증. gofmt Format.
    - Atomic: 같은 디렉터리 temp → rename. Windows: Rename 실패 시 MoveFileEx fallback, open handle 200ms→500ms→1s 재시도.
    - 트랜잭션: txlog prepared → file_done*(파일별 backup 경로 기록) → committed → 파일/로그 삭제. 중도 실패 → file_done 된 파일을 backup에서 복원.
    - crash 재실행 진입점: txlog 존재 시 rollback만 수행하고 종료 or 계속(flow 결정).
    - HookRunner 인터페이스 + NoopHook stub(Phase 2-3에서 교체).
  - 선택 이유: fuzz matching은 LLM의 line 오차 수용의 최소 타협. AST 검증을 apply 내부에 둬야 잘못된 구문이 Verify까지 도달하지 않음.
  - 실패/예외: context 불일치/AST/gofmt → SYNTAX. 파일 없음/rollback 실패 → PERMANENT. I/O 일시 오류 → TRANSIENT.

- [ ] **10. Verify + RetryState (verify)**
  - 목적: apply 결과 판정과 retry/abort 결정을 단일 상태 머신에 집중 → PLAN: Phase 2-2 > Verifier
  - 책임: 입력 변경 파일 리스트 + Config → 출력 PASS/FAIL + 재시도 판정. 경계: Apply/Flow에 retry 결정 미위임.
  - 설계:
    - TestRunner: `exec.Command`로 go test/vet. scope=package면 변경 파일 디렉터리, all이면 `./...`. timeout=config.
    - RetryState: (Attempt, SigCount map). FAIL → error signature 생성 → SigCount++ → abort 판정(MaxRetry 또는 sig 2회).
    - ErrorNormalizer 인터페이스 + GoErrorNormalizer: 경로/라인 제거 → trim → exact match.
  - 선택 이유: sig exact match 선택은 prefix match의 오탐(부분 메시지 충돌) 회피. Normalizer 인터페이스화로 Phase 4 언어 확장 대비.
  - 실패/예외: timeout → TRANSIENT. vet 실패 → SYNTAX. test 실패 → SYNTAX(retry hint 대상).

- [ ] **11. Hook / Policy (hook)**
  - 목적: apply 전후 정책 검사 슬롯. Apply 내부 NoopHook을 실제 구현으로 교체 → PLAN: Phase 2-3 > PolicyHook
  - 책임: 입력 대상 파일 + Config → 출력 정책 판정. 경계: pre 실패는 apply 차단, post는 cleanup만.
  - 설계: Pre chain(하나라도 에러 → 이후 skip + PERMANENT wrap). Post always-run. on_violation abort/warn. `--override-policy` 플래그 시 warn 강등 + slog.Warn 기록.
  - 선택 이유: chain short-circuit을 pre에 한정해야 cleanup 누락 방지. override는 단일 플래그로만 허용해 audit 가능.
  - 실패/예외: override 없이 우회 불가. post hook 실패는 로그만.

**Phase 2 통과 확인**: fixture 기반 통합 테스트(valid/fuzz/mismatch/invalid) 통과 → PLAN Phase 2 체크.

---

### Phase 3 — End-to-end `agent-harness run`
> 선행: Phase 1·2 통과 확인 블록 전체 회귀(`TestPlanGeneration` + Phase 2 fixture 통합 테스트 재실행) 통과 **+ DP-3/DP-4 승인 완료**. Exit은 PLAN의 `run --request "..."` 성공 + dry-run + budget 분기 + SIGINT rollback.

- [ ] **12. Skill Registry + file_read + shell_run (skill)**
  - 목적: tool use의 실행 본체. allow-list 기반 안전 경계 확정 → PLAN: Phase 3-1 > Skill
  - 책임: 입력 name + JSON input → 출력 result + idempotent 플래그. 경계: tool 스키마 검증, deny 경로 차단.
  - 설계:
    - Registry: Register(중복 패닉)/Get/Execute. 이름은 underscore 통일.
    - file_read: 멱등. deny_context_paths 매칭 → PERMANENT.
    - shell_run: 비멱등. allow-list prefix match. Direct exec(go/git/gofmt/goimports) vs Shell exec(sh -c / cmd /c). env 시크릿 제거. timeout/출력 상한.
  - 선택 이유: direct exec 경로가 있으면 allow-list 매칭 실패 시에도 injection risk 제거. prefix match가 단일 명령+옵션 조합 허용과 양립 가능.
  - 실패/예외: 비허용 명령/스키마 실패 → PERMANENT. 비멱등 실패 재시도 금지.

- [ ] **13. ToolExecutor + 멀티턴 루프 경계 (tool)**
  - 목적: LLM tool_use 블록 1개 단위의 실행 책임. 멀티턴 루프 자체는 Flow가 구동 → PLAN: Phase 3-1 > Tool Use
  - 책임: 입력 ContentBlock(tool_use) → 출력 ContentBlock(tool_result). 경계: 루프 제어/종료 판정은 Flow.
  - 설계: name → Registry.Execute → result 크기 제한(`tool.max_result_size`) → IsError 세팅. deny_paths/deny_context_paths 양쪽 적용.
  - 선택 이유: 단일 실행과 루프 제어 분리해야 Flow가 round 카운트/종료 조건을 단일 지점에서 관리 가능.
  - 실패/예외: 스키마 실패 → IsError=true + result에 에러 메시지. 결과 크기 초과 → 뒷부분 우선 truncate.

- [ ] **14. Observability Recorder + Audit (observe)**
  - 목적: timing/usage/audit을 단일 API로 수집해 Flow 코드를 깨끗하게 유지 → PLAN: Phase 3-2 > Observability
  - 책임: 입력 stage+fn → 출력 span 기록. 경계: Cost Control은 별도(같은 usage를 각자 소비).
  - 설계: Recorder 인터페이스(Record/Summary) + 기본 SpanRecorder(메모리). Audit은 LLM 호출 전후 원문 JSONL append(`audit.enabled` 시). deny는 context 시점에 이미 제외됨.
  - 선택 이유: OTel 호환 구조로 설계해 Phase 4에서 exporter만 교체. 관측-제어 분리로 Cost 변경이 Observe 건드리지 않음.
  - 실패/예외: 파일 쓰기 실패 시 로그만(감사는 best-effort).

- [ ] **15. SessionBudget + RateLimiter 실구현 (cost)**
  - 목적: Phase 1-2에서 주입한 NoopLimiter 자리를 실구현으로 교체, budget 정책 강제 → PLAN: Phase 3-2 > Cost Control
  - 책임: 입력 CallOption+예상토큰, API usage → 출력 허용/거부, 대기. 경계: 캐싱 파라미터는 provider에 위임.
  - 설계:
    - SessionBudget: PreCheck(잔여 확인)/Record(차감). on_exceed abort/shrink.
    - RPM: token bucket 1초 리필. TPM: 60초 sliding window. 0이면 비활성.
    - 429 OnThrottle(retryAfter)로 스로틀 구간 설정.
    - Prompt Caching: CallOption.CacheControl breakpoint 4개(system 끝 + 대형 context 파일 끝). Claude provider가 ephemeral 적용.
  - 선택 이유: 토큰 기준 budget(금액 미추적)으로 MVP 단순화. sliding window는 TPM 버스트 방어에 token bucket보다 적합.
  - 실패/예외: 대기 중 ctx cancel → 즉시 에러. PreCheck 실패 시 on_exceed 분기(abort or shrink 1회).

- [ ] **16. Flow 파이프라인 + toolUseLoop (flow)**
  - 목적: 모든 패키지를 조립해 end-to-end 실행. retry/rollback/shrink 분기의 유일한 조율자 → PLAN: Phase 3-3 > Flow
  - 책임: 입력 Config+request → 출력 exit code. 경계: 하위 패키지 순서·에러 분류·상태 머신 재구현 금지.
  - 설계:
    - Phase A 초기화: dirty workdir 정책(abort/stash/proceed, non-git fallback), txlog 자동 rollback(DP-4 확정 정책), lock 획득.
    - Phase B Plan 생성: ContextBuilder → Budget.PreCheck(shrink 분기) → Provider.Call(**CallOption.Model = `api.model`**, tools=[create_plan]) → Budget.Record → PlanParser + request 주입 → PlanGate.
    - Phase C Step 루프: skill/llm 분기. llm step은 독립 Conversation. **llm step 호출 시 CallOption.Model = `api.diff_model`(미설정 시 `api.model`), context 예산 = `api.diff_context_window`(미설정 시 `api.context_window`)**.
    - toolUseLoop: tool_use → ToolExecutor → append tool_result → 재호출. 종료: end_turn, max_tool_rounds, 동일 tool+input 연속 2회, ctx cancel.
    - Diff 파싱 실패: SYNTAX retry. 2회째 → fullfile fallback. 토큰 조건 산출(`EstimateTokens(file) < context_window*0.6 - 고정블록토큰`)은 **DP-3 확정 주체**가 담당.
    - Hook.Pre → Apply → Format → Verify → Post. FAIL → Rollback → RetryState.
    - TRUNCATED(max_tokens) → Shrink + 1회 재시도.
    - Phase D 완료: Recorder.Summary → lock 해제.
  - 선택 이유: 조립·상태 머신 단일 지점. Phase 1/2 마일스톤이 Flow 부재 상태에서도 테스트 가능하도록 유지됐기에 Flow 도입을 마지막 순서에 배치.
  - 실패/예외: lock 실패 → PERMANENT(stale 아님 경우). stash pop conflict → 사용자 수동 조치 안내 + exit 1. 각 Phase의 에러 Class 그대로 전파.

- [ ] **17. CLI entrypoint + graceful shutdown (cmd/agent-harness)**
  - 목적: 사용자 접점. 플래그 파싱과 시그널 처리만, 로직은 Flow 위임 → PLAN: Phase 3-3 > CLI
  - 책임: 입력 argv/stdin → 출력 exit code 0/1/2/130. 경계: 비즈니스 로직 없음.
  - 설계:
    - 서브커맨드: `run`/`plan`/`apply`. `--request`/`--request-file`/`--request -` 상호배타.
    - 공통 플래그: `--root`, `--config`, `--dry-run`, `--verbose`, `--override-policy`, `--force`.
    - dry-run: Apply/Format/Verify 스킵, tool 루프 정상 실행, 최종 diff만 stdout.
    - signal.NotifyContext(SIGINT, SIGTERM). Windows는 SIGTERM 미등록(빌드 태그 분리). SIGINT 시 apply 완료/rollback 후 exit 130.
    - exit: 0 성공 / 1 실행 중 실패 / 2 실행 전 설정 오류 / 130 신호.
  - 선택 이유: CLI를 마지막에 배치하면 Flow 테스트가 CLI 의존 없이 가능. 서브커맨드 3종은 plan/apply 분리 실행 지원 최소 요건.
  - 실패/예외: 필수 플래그 누락/상호배타 위반 → exit 2. API key 미설정 → exit 2.

**Phase 3 통과 확인**: PLAN Phase 3 검증 기준 5종(성공 실행, 함수 존재, dry-run, budget 분기, SIGINT rollback) + 실패 기준(deny_paths) 전부 통과 → PLAN Phase 3 체크.

---

## Decision Points (승인 필요)

다음 항목은 선택지만 남겨둔다. 구현 Task는 승인 후 진입.

### DP-1. 파일 선택 알고리즘 (Phase 1-4)
PLAN은 "plan target → 키워드 매칭 → 크기 작은 순"을 MVP로 명시. 키워드 매칭 세부(토크나이저·경로 가중치·상한 N)는 미확정.
- 옵션 A: 단순 whitespace 토큰화 + 경로 substring 매칭, 상한 20 파일.
- 옵션 B: camelCase/snake_case 분리 + AST identifier 매칭.
- Trade-off: A는 구현 1일, 재현성 높음. B는 정확도↑이지만 Phase 1 범위 초과 위험 + go/parser 의존.
- **제안**: A(단순). Phase 4 확장 여지 남김.

### DP-2. Conversation 히스토리 축소 상세 (Phase 1-4)
PLAN은 "오래된 tool 결과를 앞 3줄 + `... truncated N lines`로 교체"를 명시. 교체 시점 경계가 미확정.
- 옵션 A: 매 Shrink 호출마다 가장 오래된 tool_result부터 순차 교체.
- 옵션 B: 토큰 초과분 충당될 때까지만 교체, 나머지 유지.
- Trade-off: A는 예측 가능·정보 손실 큼. B는 손실 작음·구현 복잡.
- **제안**: B. 성능 차 무시할 수준, 멀티턴 품질이 더 중요.

### DP-3. fullfile fallback 토큰 조건 해석 (Phase 1-5)
PLAN 조건: `EstimateTokens(file) < (context_window * 0.6 - 고정블록토큰)`. "고정블록토큰" 산출 주체가 미확정.
- 옵션 A: ContextBuilder가 마지막 산출한 값을 Flow에 노출.
- 옵션 B: Flow가 fallback 시점에 ContextBuilder를 재호출하여 계산.
- Trade-off: A는 상태 노출, 경계 오염. B는 호출 1회 추가, 경계 깨끗.
- **제안**: B. Flow가 유일한 조율자 원칙과 일치.

### DP-4. txlog 복구 진입점 (Phase 2-1 ↔ Phase 3-3)
crash 재실행 시 txlog rollback을 "즉시 실행 후 계속" 또는 "rollback 후 종료하고 사용자에게 재실행 요구" 중 선택.
- 옵션 A: 자동 rollback + 계속. UX 부드러움, 상태 오염 위험.
- 옵션 B: rollback + 종료. 명시적, 1회 추가 수동 실행 필요.
- Trade-off: A는 편의성, 재crash 루프 가능성. B는 안전, 추가 조치 필요.
- **제안**: B. PLAN "clean restart" 문구와 일치.

### DP-5. Hook/Policy 실제 검사 규칙 (Phase 2-3)
PLAN은 체인/정책/override만 정의하고 기본 hook 목록은 비어 있음.
- 옵션 A: MVP에 hook 0개 등록(인프라만).
- 옵션 B: `deny_paths` 재검사 + 파일 크기 상한 hook 기본 등록.
- Trade-off: A는 범위 축소. B는 Gate 중복 가능.
- **제안**: A. Gate가 이미 deny_paths 책임, 중복 제거.

---

## PLAN 매핑 점검

| PLAN 항목 | IMPLEMENT 단위 |
|---|---|
| 1-1 에러 시스템 | 1 |
| 1-1 Config / .harness 초기화 / testdata | 2 |
| 1-2 LLMProvider 인터페이스 / Conversation / CallOption | 4 |
| 1-2 ClaudeProvider (HTTP/SSE/key/retry) | 5 |
| 1-2 RateLimiter 인터페이스 + NoopLimiter | 4 (인터페이스) / 15 (실구현) |
| 1-3 ResponseParser | 6 |
| 1-4 ContextBuilder (블록/선택/축소/hint) | 7 |
| 1-4 토큰 카운팅 | 3 |
| 1-5 Plan 생성 + PlanGate | 8 |
| 2-1 ApplyEngine (파싱/AST/atomic/tx/HookRunner stub) | 9 |
| 2-2 Verifier / RetryState / Normalizer | 10 |
| 2-3 PolicyHook | 11 |
| 3-1 Skill + file_read + shell_run | 12 |
| 3-1 ToolExecutor | 13 |
| 3-2 Observability / Audit | 14 |
| 3-2 SessionBudget / RateLimiter / Caching | 15 |
| 3-3 Flow 전체 | 16 |
| 3-3 CLI + graceful shutdown | 17 |

**PLAN 누락 없음** (Phase 1~3). Phase 4는 본 문서 범위 외.
