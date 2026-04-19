# PLAN

Agent Harness 의 완성 판정 기준. 각 Phase 는 "사용자 관점에서 무엇이 동작하는가"로 정의한다. 구현 구조·순서 결정은 Decision Point 섹션에 모으고, Task 는 "어떤 상태가 동작하면 끝났다고 판단할지"만 기술한다. `READMD.md` 가 설계 불변 모델(상태 기계, 에러 분류, 데이터 모델)을 정의하며, 이 문서는 그 위에서 "어떤 경계가 동작해야 완성인가"를 선언한다.

Phase 규칙:
- Phase 의 Decision Point 가 미결정 상태이면 해당 Phase 의 Task 진입은 사용자 승인이 필요하다.
- 각 Phase 의 첫 Task 는 직전 Phase 의 Exit Criteria 회귀 검증이다.
- Phase 통합 검증이 `✓` 되기 전에는 다음 Phase 진입 금지.

---

## Phase 1 — 단일 Step 요청이 적용·검증까지 완료된다

사용자가 단일 파일 수정 요청을 넣으면 시스템이 LLM 호출 → diff 생성 → Apply → Verify 까지 수행하고 구조화된 Episode 결과를 반환한다. 이 Phase 는 전체 파이프라인의 최소 기동을 증명한다.

### Tasks

- 단일 Step 요청이 Episode 단위로 성공/실패 결과를 반환한다
  - Request → Plan → Step 실행 → Verify → final_result 로 이어지는 종단 경로가 처음으로 맞물리는 지점이다. 이 경계가 무너지면 이후 Phase 의 모든 검증이 의미를 잃는다.
  - Purpose: 전체 레이어(Plan/Flow/Prompt/API/Apply/Verify)가 하나의 요청에서 실제로 연결되어 작동함을 확인한다.
  - Input: 1개 파일만 수정하는 최소 요청 (예: 주석 한 줄 추가)
  - Exit Criteria: CLI 실행이 종료 코드와 함께 Episode 결과(input + plan + final_result)를 반환하고, final_result 에 status / applied_files / diff 가 포함된다.

- Step Apply 가 전부-적용 또는 전부-롤백 중 하나로 귀결된다
  - 여러 파일을 동시 수정하는 Step 에서 중간 실패가 발생해도 워크스페이스가 일관된 상태로 보이는지 검증한다. README §11 의 Apply Atomicity 를 직접 관측 가능한 형태로 확인한다.
  - Purpose: partial apply 상태가 외부에서 관측되지 않음을 보장한다.
  - Input: 2개 이상 파일 변경을 포함하는 Step, 그 중 한 파일의 쓰기가 실패하도록 구성된 상황
  - Exit Criteria:
    - 실패 시 Step 내 모든 파일이 Step 시작 전 상태로 복원된다.
    - 성공 시 모든 파일이 동시에 최종 상태로 전환되며 중간 상태가 관측되지 않는다.

- Verify 결과가 구조화된 형태로 상위 레이어에 전달된다
  - Verify 산출물이 재시도/분류의 입력으로 실제 사용 가능한지 확인한다. 문자열 로그만 내보내는 형태로는 이후 Phase 의 에러 분류가 불가능하다.
  - Purpose: README §17 출력 구조가 상위 레이어가 의사 결정에 쓸 수 있는 형태로 제공됨을 검증한다.
  - Input: compile 실패를 유발하는 diff, test 실패를 유발하는 diff
  - Exit Criteria:
    - compile 실패 → status=failed, error_type=SYNTAX, error_message 포함.
    - test 실패 → status=failed, failed_tests[] 와 retry_attempts 포함 (Phase 1 시점의 error_type 세분도는 DP-1.3 에 따름).

### Decision Points

**DP-1.1 — Phase 1 에서 Skill Step 지원 여부**
- 옵션 A: Phase 1 은 LLM Step 타입만 지원. Skill 은 이후 Phase 로 이동.
  - trade-off: Phase 1 범위 최소화, Step type 분기 부담 연기. Skill 도입 시점에 Flow 인터페이스 확장이 필요.
- 옵션 B: LLM + Skill 모두 지원.
  - trade-off: Step type routing 을 초반부터 강제해 후속 리팩토링 부담 감소, 대신 Phase 1 구현량 증가.
- 상태: **결정 — 옵션 A (LLM Step 전용, Skill 은 이후 Phase)**

**DP-1.2 — Provider 선정 순서**
- 옵션 A: Anthropic 우선 (native tool_use, diff 생성 품질 관측 용이).
- 옵션 B: OpenAI 우선 (function call, 샘플·레퍼런스 풍부).
- 옵션 C: 두 provider 동시 도입해 abstraction 을 즉시 검증.
- trade-off: A/B 는 Phase 1 최단 경로, C 는 §13 Provider Abstraction 설계를 조기에 강제 검증.
- 상태: **결정 — 옵션 B (OpenAI 우선)**

**DP-1.3 — Phase 1 시점의 error_type 분류 범위**
- 옵션 A: Phase 1 에서는 SYNTAX / OTHER 2종만. 세분화는 Phase 3 에서 도입.
- 옵션 B: 처음부터 README §7 의 4종(SYNTAX/TRANSIENT/PERMANENT/APPLY_ERROR)을 모두 인식.
- trade-off: A 는 Phase 1 를 얇게 유지, B 는 이후 Phase 에서 분류기만 정교화하면 됨.
- 상태: **결정 — 옵션 B (4종 전체 인식)**

---

## Phase 2 — 복수 Step Plan 이 Gate 를 통과하고 순차 실행된다

여러 파일/단계에 걸친 요청이 multi-step Plan 으로 생성되고, Gate 로 schema·논리 검증 후, Step 간 context 전파를 포함해 순차 실행된다.

### Tasks

- Phase 1 회귀: 단일 Step 요청이 여전히 Episode 결과를 반환한다
  - 다중 Step 파이프라인 도입 이후에도 최소 경로가 깨지지 않음을 확인하는 구조적 회귀 검증이다.
  - Purpose: Phase 1 Exit Criteria 가 Phase 2 변경 후에도 동일하게 유지됨을 확인한다.
  - Input: Phase 1 에서 사용한 minimal 요청
  - Exit Criteria: Phase 1 의 모든 Exit Criteria 가 그대로 통과한다.

- Plan Gate 가 잘못된 Plan 을 거부하고 hint 포함하여 재시도한다
  - Plan 의 schema/논리 오류가 실행 단계로 전파되지 않는 경계를 검증한다. Gate 는 실패를 LLM 에게 다시 읽히는 지점이며, hint 없이 retry 하면 동일 실패가 반복된다.
  - Purpose: README §6 의 Gate 재시도·abort 의미론 검증.
  - Input: schema 위반 Plan 응답, 파일 참조 누락 Plan 응답
  - Exit Criteria:
    - schema/논리 오류 발생 시 LLM 재요청 프롬프트에 오류 지점을 식별 가능한 hint 가 포함된다.
    - 재시도 한도 초과 시 Episode 가 abort 상태로 종료되고 사유가 기록된다.

- Step 간 context 가 다음 Step 의 LLM 입력에 반영된다
  - README §5 가 규정한 우선순위 기반 context 전파를 관측 가능한 형태로 검증한다. 토큰 예산 초과 시 낮은 우선순위 항목부터 절단되어야 한다.
  - Purpose: ExecutionState.context 가 Step 경계를 넘어 유효한 정보를 전달하는지 검증한다.
  - Input: 2개 이상 Step 으로 구성된 Plan, 그리고 token budget 을 초과하도록 큰 변경 파일 목록
  - Exit Criteria:
    - N 번째 Step 의 프롬프트에 N−1 까지의 "변경된 파일 목록" 및 "이전 step 결과 요약" 이 포함된다.
    - Budget 초과 시 변경 파일 목록이 가장 먼저 축소되고, 실패 로그는 유지된다.

- Multi-step 요청이 순차 실행되고 중간 실패 시 이후 Step 이 실행되지 않는다
  - Step 순서와 실패 시 중단 의미론을 직접 관측한다. 실패 이후 Step 을 무시하지 않고 임의 실행하면 부분 결과가 남아 워크스페이스 상태가 흐려진다.
  - Purpose: Step sequencing 과 실패 전파의 경계를 확인한다.
  - Input: 3-step Plan 에서 2번째 Step 이 실패하도록 구성된 입력
  - Exit Criteria:
    - 1번째 Step 은 success 로 Episode 에 기록된다.
    - 2번째 Step 은 failed 로 기록되고 3번째 Step 은 실행되지 않는다.

### Decision Points

**DP-2.1 — Gate retry 카운터와 Step retry 의 관계**
- 옵션 A: 공유된 단일 retry budget.
  - trade-off: config 단순, 대신 Gate 반복이 Step retry 예산을 잠식.
- 옵션 B: 분리 (plan_gate.max_retry, step.max_retry).
  - trade-off: 각 단계 독립 튜닝 가능, config 표면 증가.
- 상태: **미결정**

**DP-2.2 — Context truncation 수행 주체**
- 옵션 A: Prompt Layer 내부 (provider 별 토큰 한도를 Prompt 가 인지).
  - trade-off: provider 특성 반영 쉬움, 다중 provider 지원 시 중복 증가.
- 옵션 B: 중앙 Context manager (provider-agnostic 한 우선순위 기반 절단 후 Prompt 로 전달).
  - trade-off: §5 의 우선순위 체계를 한 곳에 집중, 토큰 계산이 provider 별 근사로 남음.
- 상태: **미결정**

---

## Phase 3 — 실패 분류와 복구가 동작한다

Step 실패가 에러 타입별로 분류되고, TRANSIENT 는 재시도, PERMANENT 는 retry 한도에서 abort, APPLY_ERROR 는 fuzz matching → fullfile fallback 경로로 복구된다. 이 Phase 가 통과하면 "실패는 정상 흐름"이라는 철학(README §15)이 실제로 구현된 것이다.

### Tasks

- Phase 2 회귀: Multi-step Plan 과 Gate 가 여전히 동작한다
  - 에러 분류·복구 도입 후에도 정상 경로가 깨지지 않음을 구조적으로 검증한다.
  - Purpose: Phase 2 Exit Criteria 보존.
  - Input: Phase 2 의 성공 케이스 + Gate 검증 실패 후 통과 케이스
  - Exit Criteria: Phase 2 의 모든 Exit Criteria 가 그대로 통과한다.

- 각 실패 유형이 정의된 error_type 으로 분류된다
  - README §7 의 4종 분류가 실제 분기 의사결정으로 이어지는지 확인한다. 분류가 불완전하면 retry 경로가 잘못 선택되어 무한 retry 또는 조기 abort 가 발생한다.
  - Purpose: 분류 결과가 후속 처리 경로(재시도/중단/fallback)에 결정적으로 영향을 미침을 검증.
  - Input: compile 실패(SYNTAX), 네트워크 일시 실패(TRANSIENT), diff hunk 미일치(APPLY_ERROR), 동일 오류 N회 반복(PERMANENT)
  - Exit Criteria:
    - 각 입력이 해당 error_type 으로 태깅된 StepResult 를 생성한다.
    - 분류 결과가 retry / abort / fallback 중 정확히 한 경로로 분기한다.

- Retry 상태 기계가 max 와 PERMANENT 조건을 존중한다
  - README §8 의 전이 규칙(FAILED → RETRYING/ABORTED)을 직접 관측한다. max 초과·PERMANENT 처리 누락은 무한 루프 또는 조기 종료로 직결된다.
  - Purpose: 상태 기계의 전이 규칙이 강제됨을 검증.
  - Input: TRANSIENT 실패가 max 미만일 때, max 도달 시, PERMANENT 판정 시
  - Exit Criteria:
    - max 미만 TRANSIENT → RETRYING → RUNNING 경로로 재실행된다.
    - max 도달 시 ABORTED 로 전이하고 사유가 Episode 에 기록된다.
    - PERMANENT 판정 시 retry_count 와 무관하게 ABORTED 로 전이한다.

- Flaky test 가 재실행 판정으로 TRANSIENT 처리된다
  - README §17.2.1 의 flaky 분류 경계를 관측한다. 이를 PERMANENT 로 오분류하면 정상 흐름임에도 abort 가 발생한다.
  - Purpose: intermittent 실패가 코드 수정 없이 최종 success 로 귀결되는지 검증.
  - Input: 1회 실패, 재실행 시 성공하는 테스트가 섞인 verify 결과
  - Exit Criteria: 해당 테스트는 error_type=TRANSIENT 로 기록되고 Step 은 최종 success 로 확정된다.

- Diff apply 실패가 fuzz matching → fullfile fallback 순으로 복구된다
  - README §9, §10 의 두 단계 폴백이 실제 구동되는지 검증한다. exact match 만으로는 LLM 출력의 whitespace·offset 차이에 대응할 수 없고, fullfile fallback 이 없으면 ambiguous match 에서 진행 불가 상태에 빠진다.
  - Purpose: diff 실패가 적절한 복구 경로로 처리됨을 검증.
  - Input: whitespace 차이 diff, line offset 차이 diff, ambiguous match diff
  - Exit Criteria:
    - whitespace/offset 만 다른 경우 fuzz matching 으로 apply 성공한다.
    - ambiguous / hunk 없음 인 경우 fullfile fallback 이 호출되어 파일 overwrite 후 verify 를 통과한다.

### Decision Points

**DP-3.1 — Fullfile fallback 의 적용 범위**
- 옵션 A: 실패한 파일만 regenerate.
- 옵션 B: Step 내 모든 파일 regenerate (Apply Atomicity 와 동일 단위).
- trade-off: A 는 변경 폭 최소화, B 는 §11 Apply Atomicity 원칙과 정합.
- 상태: **미결정**

**DP-3.2 — Fuzz matching 허용 offset 상한**
- 옵션 A: ±3 lines (README 허용 하한).
- 옵션 B: ±5 lines (README 허용 상한).
- trade-off: A 는 false positive 위험 낮음, B 는 LLM 출력 관용도 높음.
- 상태: **미결정**

**DP-3.3 — Retry 기본 한도 구성**
- 옵션 A: 전체 에러 공통 max=3.
- 옵션 B: 에러 타입별 분리 (TRANSIENT=3, APPLY_ERROR=2, flaky 재실행=2~3).
- trade-off: A 는 config 단순, B 는 각 실패 특성에 맞춤.
- 상태: **미결정**

---

## Phase 4 — Crash recovery · Safety · Provider 추상화가 동작한다

TxLog 기반으로 crash 이후 재실행이 가능하고, deny_paths / deny_ops 정책이 LLM output 을 차단하며, Provider abstraction 이 최소 2개 provider 에서 동일 Flow 로 동작한다. 이 Phase 가 끝나면 "재현 가능한 실행"(README §15) 이 보장된다.

### Tasks

- Phase 3 회귀: 에러 분류·fallback·retry 가 여전히 동작한다
  - Recovery/Safety 추가가 기존 복구 경로를 깨뜨리지 않음을 확인한다.
  - Purpose: Phase 3 Exit Criteria 보존.
  - Input: Phase 3 의 SYNTAX / TRANSIENT / APPLY_ERROR / PERMANENT 케이스
  - Exit Criteria: Phase 3 의 모든 Exit Criteria 가 그대로 통과한다.

- Step 단위 TxLog 가 before/after snapshot 과 함께 기록된다
  - README §12 의 체크포인트 시점과 TxLog 필드가 실제 파일에 남는지 확인한다. 이 로그가 없으면 Recovery 가 불가능하다.
  - Purpose: step 경계마다 복구에 필요한 상태가 관측·읽기 가능한 형태로 저장됨을 검증.
  - Input: 2-step 요청의 정상 실행
  - Exit Criteria:
    - 각 step 의 before_snapshot 이 파일별로 기록된다.
    - apply 완료 후 after_snapshot 과 status 가 기록된다.
    - 체크포인트가 "step 시작 전" 과 "apply 완료 후" 2지점에 모두 존재한다.

- Crash 이후 재실행이 마지막 성공 step 부터 이어진다
  - README §12 의 복구 전략을 end-to-end 로 검증한다. 중간 실패 상태가 워크스페이스에 남은 채 재실행되면 분석 대상 상태가 오염된다.
  - Purpose: crash 발생 시 일관 상태로 돌아가 이어서 실행됨을 검증.
  - Input: 3-step 실행 중 2번째 step apply 중 강제 종료된 상황
  - Exit Criteria:
    - 재시작 시 1번째 step 결과는 보존된다.
    - 2번째 step 은 before_snapshot 으로 rollback 후 재실행된다.
    - 3번째 step 은 정상 진행된다.

- 정책으로 지정된 deny_paths / deny_ops 변경이 Apply 전에 차단된다
  - README §14 Safety Model 의 경계를 검증한다. LLM output 은 항상 검증 대상이라는 원칙이 실제 실행 경로에서 적용됨을 확인한다.
  - Purpose: 정책 hook 이 Apply 이전에 실행되어 어떠한 변경도 파일 시스템에 남기지 않음을 검증.
  - Input: deny_paths 에 포함된 경로를 수정하는 LLM diff 응답
  - Exit Criteria:
    - Step 이 failed 로 종료되고 에러 분류는 DP-4.2 에 따른다.
    - 파일 시스템에 해당 diff 의 어떤 변경도 반영되지 않는다.

- 최소 2개 provider 에서 동일 요청이 동등한 Flow 로 실행된다
  - README §13 Provider Abstraction 이 Flow Layer 에서 투명한지 관측 가능한 형태로 검증한다.
  - Purpose: provider 교체가 Flow/Apply/Verify 레이어 변경 없이 가능함을 확인.
  - Input: 동일 Plan 을 OpenAI / Anthropic provider 각각으로 실행
  - Exit Criteria:
    - 두 provider 모두 동일 step sequence 를 수행하며 Episode 결과 구조가 동일하다.
    - Flow Layer 코드에 provider 분기가 존재하지 않는다 (교체 시 provider 어댑터 외 코드 변경 없음).

### Decision Points

**DP-4.1 — TxLog 저장 포맷**
- 옵션 A: Episode 당 단일 JSON 파일.
- 옵션 B: Episode 디렉터리 + step 별 JSONL append.
- 옵션 C: SQLite.
- trade-off: A 는 단순하나 동시 쓰기 위험, B 는 crash 안전성 높음, C 는 본 규모에 과다.
- 상태: **미결정**

**DP-4.2 — 정책 차단의 error_type 분류**
- 옵션 A: PERMANENT 로 통합.
- 옵션 B: POLICY 전용 분류 추가.
- trade-off: A 는 분류 체계 최소, B 는 로그·관측이 명확.
- 상태: **미결정**

---

## Global Decision Points

모든 Phase 에 걸쳐 영향을 주는 결정값. Phase 1 진입 이전에 해소되어야 한다.

**DP-G.1 — ExecutionState 의 persistence 단위**
- 옵션 A: 메모리 전용. Crash 복구는 TxLog 재생(replay)으로 수행.
- 옵션 B: Step 경계마다 JSON checkpoint 를 별도 저장.
- trade-off: A 는 단일 진실 원천(TxLog)을 유지, B 는 복구 경로가 단순하나 TxLog 와 이중 관리 부담.
- 영향 Phase: Phase 1 (데이터 모델), Phase 4 (복구)
- 상태: **결정 — 옵션 A (메모리 전용 + TxLog replay 복구)**

**DP-G.2 — 실행 대상 언어 범위**
- 옵션 A: Go 전용. Formatter/compiler/test 모두 go 도구 체인에 고정.
- 옵션 B: Language adapter 인터페이스만 열어두고 Phase 1 은 Go 만 구현.
- trade-off: A 는 Verify 레이어 구현량 최소, B 는 이후 확장 여지 확보하나 인터페이스 설계 부담.
- 영향 Phase: Phase 1 Verify 경계
- 상태: **결정 — 옵션 A (Go 전용, go 도구 체인 고정)**
