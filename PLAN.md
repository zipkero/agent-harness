# AI Agent Harness 커리큘럼

---

## 프로젝트

agent-harness (Go CLI)

---

## 구성 요소 (9개) → 챕터 매핑

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

## 챕터 목록

### Core (CH01~CH13) → [PLAN-core.md](PLAN-core.md)

| 챕터 | 제목 | 핵심 주제 |
|---|---|---|
| CH01 | 하네스의 실체 | 아키텍처, 인터페이스 설계, 에러 타입, Config |
| CH02 | Claude API 기본 | 요청/응답, CallOption, context.Context, Mock 인프라 |
| CH03 | Parsing + 호출 안정화 | 구조화 파싱, 스트리밍, 신뢰 경계, retry/backoff |
| CH04 | Context Builder | CLAUDE.md 로드, 파일 선택, 압축, 프롬프트 설계 |
| CH05 | Plan 생성 + Plan Gate | plan.md 구조, 위험 필터, Prompt Injection 방어 |
| CH06 | Patch Apply Engine ⭐ | unified diff, atomic write, Transaction rollback |
| CH07 | Verify + Retry State Machine | 상태 전이, error signature, abort 조건 |
| CH08 | Hook / Policy | pre/post hook, abort/pause/override 정책 |
| CH09 | Skill | Skill 인터페이스, registry, 멱등성, 보안 |
| CH10 | MCP / Tool Use | 외부 시스템, tool_use 루프, 신뢰 경계 |
| CH11 | Observability | SpanRecorder, slog, AuditLog, OTel 호환 설계 |
| CH12 | Cost Control | token budget, rate limiting, cache, 모델 분리 |
| CH13 | 하네스 완성 (Flow) | 전체 파이프라인, CLI, graceful shutdown, 멱등성 |

### Extensions (CH14~CH19) → [PLAN-ext.md](PLAN-ext.md)

> CH13 완성 이후 진행한다.

| 챕터 | 제목 | 핵심 주제 |
|---|---|---|
| CH14 | Multi-Agent | goroutine, channel, 충돌 감지, 비용 배분 |
| CH15 | Skill Registry 운영 | 의존성 관리, 추가 Skill 구현 |
| CH16 | GitHub 통합 | PR 자동화, 리뷰 루프 |
| CH17 | LLM Provider 추상화 | multi-provider, CallOption 중립화 |
| CH18 | 언어 플러그인 시스템 | Go/TS/Python 포매터·검증기 교체 |
| CH19 | Observability 완성 | OTel 연동, Prometheus 메트릭 |

---

## 테스트 레이어 범례

완료 기준 항목에 사용되는 레이어 표기:

| 표기 | 레이어 | 설명 |
|---|---|---|
| `[U]` | Unit | 순수 함수, 에러 분류, 파싱 — 외부 의존 없음 |
| `[M]` | Mock LLM | httptest.Server / golden file — 실제 파일시스템 불필요 |
| `[I]` | Integration | 실제 파일시스템(t.TempDir), 프로세스 실행, OS 신호 |

> LLM을 실제로 호출하는 테스트는 CI에 포함하지 않는다 (비용 및 비결정성).
