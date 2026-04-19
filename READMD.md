# Agent Harness

## 1. Overview

Agent Harness는 LLM을 활용하여 코드 변경을 실제로 적용하고,
검증까지 완료하는 실행 시스템이다.

이 프로젝트는 다음을 자동화한다:

- Plan 생성
- Diff 생성
- Apply (파일 반영)
- Verify (테스트/검증)
- Retry (자동 복구)

핵심은 "코드 생성"이 아니라
**코드 변경을 실제로 적용하고 통과시키는 것**이다.

---

## 2. Execution Model

전체 흐름:

1. Request 입력
2. Plan 생성 (+ Gate 검증)
3. Step 실행
    - LLM → diff 생성
    - Skill → 내부 처리
4. Apply (파일 변경, step 단위 atomic)
5. Verify (테스트)
6. 실패 시 Retry
7. 성공 시 다음 Step

---

## 3. Core Data Model

### 3.1 Plan

Plan은 Step의 ordered list다.

구성:

- steps[]
    - id
    - type (llm | skill)
    - description
    - target_files
    - action (create | modify | delete)
    - inputs

---

### 3.2 StepResult

각 step 실행 결과:

- step_id
- diff (optional)
- applied_files[]
- logs
- error (optional)
- status (success | failed)

---

### 3.3 ExecutionState (핵심)

Flow 간 전달되는 상태 객체:

- current_step
- completed_steps[]
- file_snapshot
- last_diff
- last_error
- retry_count
- context (LLM 입력용 누적 정보)

→ 모든 레이어는 이 상태를 읽고 수정한다

---

### 3.4 Episode

전체 실행 단위:

- input
- plan
- execution_state
- trace
- final_result
- evaluation

---

## 4. Data Flow

Layer 간 데이터 흐름:

```
Request
→ Plan Layer (Plan 생성)
→ Gate (schema 검증 + 논리 검증)
→ Flow Layer (ExecutionState 생성)
→ Step 실행
```

Step 실행 흐름:

```
ExecutionState
→ Prompt Layer (LLM 입력 생성)
→ API Layer (LLM 호출)
→ Diff 생성
→ Apply Layer (파일 반영, step 단위 atomic)
→ Verify Layer (테스트 실행)
→ 결과 반영 (ExecutionState 업데이트)
→ 다음 Step 반복
```

---

## 5. Context Propagation

Step 간 context 전달 방식:

ExecutionState.context를 사용한다.

포함 내용:

- 이전 step 결과 요약
- 변경된 파일 목록
- 실패 로그 (최근 N개)
- 현재 목표 (current step description)

Context 구성 우선순위 (token budget 초과 시):

1. 실패 로그 (최근 N개 cap 적용)
2. 현재 step description
3. 이전 step 요약
4. 변경 파일 목록

→ 우선순위 낮은 항목부터 truncate

---

## 6. Plan Gate

Plan 생성 후 즉시 검증한다.

### 검증 단계

1. Schema 검증 — 구조/타입 오류
2. 논리 검증 — 파일 누락, 순서 오류, 의존 관계

### 실패 처리

- Schema 오류 → hint 포함 retry
- 논리 오류 → hint 포함 retry
- N회 반복 실패 → abort (기본값: 3회)

---

## 7. Error Classification

### 7.1 LLM Step

| 타입 | 조건 | 처리 |
|---|---|---|
| SYNTAX | 컴파일 실패, AST invalid | retry + error message 전달 |
| TRANSIENT | network 실패, flaky test | 동일 입력으로 retry |
| PERMANENT | 동일 오류 N회 반복, 논리 오류 | fallback 또는 abort |
| APPLY_ERROR | diff hunk mismatch | fuzz matching → fullfile fallback |

### 7.2 Skill Step

Skill은 deterministic한 내부 기능이므로 별도 분류를 적용한다.

- `retryable: bool` 플래그로 제어
- retryable=true → TRANSIENT 처리 (일시적 환경 문제)
- retryable=false → 즉시 abort

---

## 8. Retry State Machine

각 step의 상태:

```
INIT → RUNNING → SUCCESS → 다음 step
                ↓
              FAILED
                ↓
          retry_count < max?
           ↙          ↘
       RETRYING     ABORTED
          ↓
       RUNNING
```

전이 규칙:

- FAILED → RETRYING: retry_count < max, non-permanent error
- FAILED → ABORTED: permanent error 또는 retry 초과
- RETRYING → RUNNING: 재실행
- SUCCESS → 다음 step 진행

---

## 9. Diff Apply & Fuzz Matching

### 기본 전략

- unified diff 기반
- exact match 우선

### Fuzz Matching 허용 범위

허용:

- whitespace 차이
- line offset (±3~5 lines)
- indentation mismatch

불허:

- semantic mismatch
- function/block 구조 변경
- ambiguous match (2개 이상 candidate)

### 실패 조건

- target hunk 없음
- ambiguous match

→ fullfile fallback으로 전환

---

## 10. Fullfile Fallback

조건:

- diff apply 2회 이상 실패
- fuzz matching 실패

동작:

- 파일 전체 재생성 후 overwrite

검증:

- formatting → compile → test

---

## 11. Apply Atomicity

Apply는 **step 전체 단위**로 atomic하게 동작한다.

- step 내 모든 파일 변경이 전부 성공하거나 전부 rollback
- partial apply 상태는 허용하지 않음
- txlog의 before_snapshot은 파일별로 저장

근거: step은 논리적으로 하나의 변경 단위이며,
일부만 적용된 상태는 컴파일/테스트가 의미 없음.

---

## 12. TxLog & Recovery

### TxLog 단위

step 단위로 기록:

- step_id
- before_snapshot (파일별)
- diff
- after_snapshot
- status

### Checkpoint 시점

- step 시작 전
- apply 완료 후

### Recovery 전략

crash 발생 시:

- 마지막 성공 step 확인
- 미완료 step은 before_snapshot으로 rollback
- 해당 step부터 재실행

---

## 13. Provider Abstraction

### 공통 인터페이스

```
Generate(input, options) → output
SupportsToolUse() → bool
ParseToolCall(output) → ToolCall
```

### Tool Use 처리

provider별 차이를 내부 공통 구조로 normalize:

- OpenAI: function call
- Anthropic: tool_use
- 기타: text 기반 파싱

Flow Layer는 provider 차이를 알 필요 없음.

---

## 14. Safety Model

- deny_paths: 수정 금지 경로
- deny_ops: 금지 작업
- policy hook: 커스텀 정책
- atomic write + rollback: step 단위
- txlog 기반 crash recovery

LLM output은 항상 검증 대상이다.

---

## 15. Philosophy

- LLM은 generator가 아니라 executor다
- 실패는 정상 흐름이다
- 모든 실행은 추적 가능해야 한다
- 시스템은 재현 가능해야 한다

---

## 16. Diff Contract

LLM은 unified diff 형식으로 코드 변경을 생성해야 한다.

요구사항:

- 파일 단위 구분 명확
- path 포함
- hunk header 필수 (@@)
- 여러 파일 변경 가능

Validation:

- diff parser 통과 필수
- 실패 시 retry

---

## 17. Verify Criteria

Verify 단계는 다음 순서로 수행된다:

1. formatting
2. compile
3. test

---

### 판정 기준

#### 1. SYNTAX

- compile 실패
- parsing/AST 오류

→ 즉시 retry (error message 포함)

---

#### 2. TEST FAILURE (세분화 필요)

test 실패는 즉시 PERMANENT로 처리하지 않는다.

다음 기준으로 분류한다:

---

### 2.1 TRANSIENT (flaky test)

조건:

- 동일 테스트를 재실행 시 성공
- 또는 일부 테스트만 간헐적으로 실패

판정 방식:

- 동일 test 최대 N회 재실행 (기본값: 2~3)
- 1회라도 성공하면 TRANSIENT로 간주

처리:

- 동일 상태 유지 후 retry
- context에 flaky 가능성 힌트 추가 (optional)

---

### 2.2 PERMANENT

조건:

- 동일 테스트 N회 연속 실패
- 항상 동일 assertion 또는 동일 위치에서 실패

판정 방식:

- 동일 입력 + 동일 코드 기준 반복 실행
- 실패 결과가 일관됨

처리:

- LLM retry (코드 수정 유도)
- retry 초과 시 abort

---

### 2.3 NO TEST

조건:

- 테스트 없음 또는 실행 불가

처리:

- compile 성공 시 success 처리 (configurable)
- optional: warning 로그 기록

---

### 출력 구조

Verify 결과는 다음을 포함한다:

- status (success | failed)
- error_type (SYNTAX | TRANSIENT | PERMANENT)
- error_message
- failed_tests[]
- retry_attempts

---

## 18. File State Model

파일 상태는 다음 3계층으로 관리된다:

- Workspace (실제 파일)
- Snapshot (ExecutionState 기준)
- TxLog Backup (rollback 기준)

규칙:

- apply → Workspace
- 성공 시 Snapshot 갱신
- 실패 시 TxLog 기반 rollback
- context는 Snapshot 기준 사용