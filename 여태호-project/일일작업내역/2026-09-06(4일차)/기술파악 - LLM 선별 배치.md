---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-06
---

# LLM 선별 배치와 3국면 파이프라인 (v2 2단계)

> [!summary] 한 줄 요약
> 키워드 관문을 지우고 정책 문장을 읽는 LLM 선별 배치로 바꿨다. 파이프라인을 관문→선별→점수·판정 3국면으로 재구성하고, 모든 LLM 호출을 자문 잠금 예약으로 상한 안에 묶었다. 기능: [[검증 파이프라인 v2]]

## 무슨 작업

구현서 2단계 Task 2.1~2.7 을 순서대로.

**정책 설정 재구조화.** `rules.yaml` 을 `policy`(관심·비관심 문장·주목 저장소·스택)·`exclude`·`dedupe`·`triage`·`scoring`·`notify` 로 바꾸고 모든 모델에 `extra="forbid"`. 옛 키(`include_keywords`·`always_pass_sources` 등)가 남아 있으면 기동이 실패한다. `rules.py` 는 exclude 판정만 남기고, 점수의 `kw` 항을 선별 관련도 `rel` 로 교체, 전역 `is_stale` 가드 추가.

**호출 예약.** `db/budget.py` 의 `reserve_call` 이 단일 고정 키 `pg_advisory_xact_lock` 안에서 오늘 행 수를 세고 상한 미만이면 `llm_calls` 에 예약 행을 커밋한다. 판정 예산을 선별·탐색이 공유하므로 키를 하나로 뒀다. "오늘" 은 트랜잭션이 시각을 한 번 읽어 범위와 `called_at` 에 같이 쓴다.

**선별 배치.** `pipeline/triage.py` 가 25건을 한 호출에 넣어 관련도 0~1 + 이유를 구조화 출력으로 받는다. 시스템 블록에 정책과 캐시 제어. 실패를 기반·배치·항목 3층위로 나눠 항목 실패만 폐기 카운트에 센다.

**3국면.** `pipeline.py` 를 재작성. 1국면(항목별 세이브포인트: stale→중복→exclude), 2국면(25건 배치 선별), 3국면(항목별 세이브포인트: 점수→보강→판정). 배치 잠금을 세 국면과 외부 호출 동안 유지한다(단일 프로세스 전제). 판정 본문 보강은 `llm.body_for_judge` 로 분리해 탐색 슬롯과 공유할 준비.

## 무엇이 달라졌나

실배치로 확인한 통과 흐름이다. 어제라면 키워드 없어 `no_keyword` 로 죽었을 항목들이 관련도로 통과한다.

| 항목 | 관련도 | 점수 | 판정 |
|---|---|---|---|
| Claude Code v2.1.261 (에이전트 시스템 프롬프트 파일) | 0.60 | 0.46 통과 | importance 4 SCORED |
| anthropic-sdk-python v1.4.0 | 0.80 | 0.51 통과 | importance 3 SCORED |
| uv 0.12.10 | 0.60 | 0.46 통과 | importance 3 SCORED |
| Next.js v16.4.0-canary.16~19 | 0.30~0.40 | 미달 | 캐너리라 탈락 |
| 조코딩 "중학생도 AI" | 0.10 | 0.21 | 입문 홍보로 탈락 |

선별이 캐너리 릴리즈는 낮게(0.3), 실제 기능 추가 릴리즈는 높게(0.6~0.8) 갈랐다. **arXiv 는 전량 stale** 로 선별 앞에서 걸러져 호출을 쓰지 않았다.

## 왜

관심 정책은 이미 `rules.yaml` 문장으로 있었는데 그것을 읽는 관문이 맨 끝 판정 LLM 뿐이었다. 선별을 앞에 둬서 정책을 두 번 읽게 하고, 키워드 목록·화이트리스트 우회를 지웠다. 결정과 기각 대안은 [[기술파악 - 결정 기록]] 의 v2 절.

## 어디

| 커밋 | 내용 |
|---|---|
| `ff8ab69` | 정책 설정 재구조화 (exclude 전용·rel 항·stale·extra=forbid) |
| `a945466` | `reserve_call` 자문 잠금 예약 |
| `76e0070` | LLM 선별 배치 |
| `f60bbd7` | 파이프라인 3국면, 관문 삭제 |
| `4baf922` | CLAUDE.md 갱신 |
| `b50f94d` `64a4300` | Codex 반영: SDK 재시도 차단, 운영 DB 가드, 3국면 통합 테스트 |

핵심 파일: `app/db/budget.py`, `app/pipeline/triage.py`, `app/jobs/pipeline.py`, `app/pipeline/rules.py`, `app/pipeline/scoring.py`, `config/rules.yaml`.

## 검증

- `ruff` · `mypy` 통과, 단위 **107 passed, 1 skipped**.
- 통합 테스트 **6 passed** (Neon `dev` 브랜치 `br-crimson-pond-azpw4eyr`, `.env` 의 `TEST_DATABASE_URL`). 벡터 중복 쿼리 A, 예약 원자성, 3국면 실패 처리 검증.
- Codex 리뷰 2회. 1차 Critical 2(SDK 재시도·운영 DB 보호)·Important 1(3국면 회귀 테스트) 반영, 2차 "없음".
- 통합 테스트 디버깅에서 `_load` 헬퍼가 세션을 `async with` 로 닫아 반환하던 버그를 찾아 고침. 파이프라인 로직 자체는 정상.
