---
tags: [여태호, 일일작업, 기술파악]
created: 2026-08-30
---

# MVP 파이프라인 구현

> [!summary] 한 줄 요약
> 문서만 있던 빈 레포에 0·1단계 전체(수집 → 관문 → 발송)를 구현하고, Codex 리뷰 3회로 Important 결함 10건을 잡았다. 기능: [[MVP 파이프라인]]

## 무슨 작업

기획서 v0.1 · 구현도 v0.2 를 읽고 로드맵의 **0단계(골격)** 와 **1단계(MVP)** 를 한 번에 만들었다. 소스 **31개 파일**, 테스트 **46개**, 커밋 **8개**.

수집기 3종(RSS · GitHub Releases · YouTube), 관문 5단계(정규화 · 중복제거 · 규칙 · 점수화 · LLM), 텔레그램 발송과 알림 피로 정책, GitHub·텔레그램 웹훅, APScheduler 잡, Docker/Caddy/CI 까지 포함한다.

구현 후 `codex exec` 로 리뷰를 **3회** 돌렸다. 1차에서 Important 7 · Minor 2, 2차에서 Important 3(부분 해결분), 3차에서 **결함 없음**.

## 왜

기획서·구현도가 이미 확정 단계라 설계를 더 논의할 게 없었다. 구현도 8장이 "0·1단계를 2주 안에 끝내는 것을 최우선"으로 잡고 있는데, 이 프로젝트의 가장 큰 리스크가 개인 프로젝트의 지속성이기 때문이다. 매일 쓰는 상태에 빨리 도달하는 게 우선이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `2024245` | 0단계 골격 — 설정·스키마·DB 모델·Alembic·컨테이너·CI | `app/config.py`, `app/db/`, `Dockerfile` |
| `47cc660` | 수집기 플러그인 3종 | `app/sources/` |
| `b9d3d5e` | 파이프라인 관문 5단계 | `app/pipeline/`, `config/rules.yaml` |
| `cef6774` | 텔레그램 발송 + 알림 피로 정책 | `app/notify/` |
| `7f116dd` | 스케줄러·웹훅·엔트리포인트 | `app/jobs/`, `app/api/`, `app/main.py` |
| `b12c6b3` | 문서·lockfile | `기획서.md`, `uv.lock` |
| `b764a0a` | Codex 1차 리뷰 반영 (8건) | `notify.py`, `normalize.py`, `github_release.py` 외 |
| `bd9ccf7` | Codex 2차 리뷰 반영 (3건) | `jobs/notify.py`, `jobs/pipeline.py` |

브랜치 `feat/mvp-pipeline` (`2024245`..`bd9ccf7`).

## 리뷰에서 잡은 것

1차 리뷰의 Important 7건 중 셋은 그냥 뒀으면 서비스를 망가뜨렸을 것이다.

**봇 토큰 노출** — `_call()` 이 봇 토큰을 URL 에 넣는데, httpx 의 `HTTPStatusError` 문자열에는 요청 URL 이 들어간다. 그 문자열이 `notifications.error` 컬럼과 docker 로그로 그대로 나갔다. 예외를 상태 코드만 담은 것으로 갈아끼웠다.

**영구 누락** — GitHub 폴링은 저장소 10개를 병렬로 돌면서 하나가 실패해도 나머지를 수집했는데, 이때 `last_polled_at` 은 전진했다. `since` 필터가 있으니 실패한 저장소의 그 사이 릴리즈는 다시는 안 보였다. `since` 를 아예 쓰지 않고 매번 최근 릴리즈를 다시 보게 바꿨다. 중복은 `url_hash` 유니크 제약이 막는다.

**decisions 무한 증식** — LLM 예산이 바닥나거나 API 가 죽으면, rule·score 판정만 기록된 채 항목이 `NEW` 로 남았다. 파이프라인 잡은 **120초**마다 도니까 같은 항목의 같은 판정이 계속 쌓인다. 예산 경로는 "항목에 손대기 전에 중단"으로, 예외 경로는 항목 단위 savepoint 로 막았다.

2차 리뷰는 1차 수정이 만든 **새 문제**를 짚었다. 발송 잡에서 20건을 한 번에 `FOR UPDATE` 로 잠근 뒤 루프 안에서 커밋하게 바꿨는데, 첫 커밋에서 나머지 19건의 락까지 풀린다는 지적이었다. 한 건씩 잠그고 커밋하도록 다시 고쳤다.

기각한 지적도 하나 있다. `notifications(item_id, channel)` 유니크 제약은 중복 *발송* 자체를 막지 못하고 중복 *기록*만 막는다. 실제 피해는 그대로 두면서 잡만 죽이므로 넣지 않았다. 경위는 [[기술파악 - 결정 기록]] 에 남겼다.

## 검증

```
uv run ruff check .        All checks passed!
uv run ruff format --check 49 files already formatted
uv run mypy app scripts    Success: no issues found in 36 source files
uv run pytest -q           46 passed
```

`app.openapi()` 로 라우트 등록도 확인했다 — `/health`, `/webhook/github`, `/webhook/telegram`.

리뷰 반영분에는 회귀 테스트 8개를 붙였다. 쿼리 파라미터 정렬 후 해시 동일성 1개, 재시도 판정(429·5xx 는 재시도 / 401·404 는 안 함) 4개, 텔레그램 예외에 토큰이 안 섞이는지 3개.

DB 가 붙는 경로(`dedupe`, `ingest`, 잡 3종)는 `pg_trgm`·`FOR UPDATE SKIP LOCKED`·`ON CONFLICT` 가 필요해 Postgres 없이는 테스트하지 못했다. Neon dev 브랜치를 붙인 뒤에 검증해야 한다.

## 다음

Neon 프로젝트를 만들어 `alembic upgrade head` 를 돌리고, 텔레그램 봇을 등록해 실제로 알림이 오는지 보는 게 다음 순서다. 그때까지는 파이프라인이 도는 것을 확인한 적이 없다.
