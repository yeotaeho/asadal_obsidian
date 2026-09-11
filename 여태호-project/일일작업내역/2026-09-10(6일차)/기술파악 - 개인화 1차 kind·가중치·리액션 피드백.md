---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-10
---

# 개인화 1차 kind·가중치·리액션 피드백

> [!summary] 한 줄 요약
> `v2-다중사용자-1차-구현서.md` 를 실행 데이터로 검토해 "개인화가 아니라 전역 규칙이고 kind 가점은 arXiv 홍수를 연다" 고 판단했고, 사용자가 고른 네 가지(kind 감점·가중치 이동·판정 논문 눈금·리액션 피드백)를 7커밋으로 적용했다. 기능: [[개인화 1차]]

## 무슨 작업

먼저 Neon `main` 의 09-06~08 데이터로 1차 구현서를 검증했다.

| 확인한 사실 | 수치 |
|---|---|
| `feedback` 행 | **0건**. 버튼은 공개 URL 이 없어 한 번도 도착 못 함 |
| 선별 449건 중 arXiv | 380건. 관련도 0.5 이상이 40% |
| score 가 [0.35, 0.45) 인 arXiv | **154건** / 2.5일. `technique +0.10` 이면 전부 판정으로 감 |
| score 가 [0.425, 0.45) 인 arXiv | 28건. 관련도 0.8 대 (BeaconKV 사례) |
| 09-07 push 5건 | 2쌍이 버전 형제 (claude-code 2.1.257/261, MCP SDK 1.30.0/2.2.0) |
| 판정의 arXiv | 2/2 를 importance 4 로 통과, 그중 SDLC 서베이가 오탐 |

그 뒤 사용자 결정으로 네 가지를 적용했다. `kind` 는 선별 구조화 출력의 필수 필드로 넣고 점수식에 덧셈 항으로 붙였다. 감점만 둔다(survey −0.15, tutorial −0.05, promo −0.30). `w_src/w_rel` 은 0.20/0.30 으로 옮겨 arXiv 통과선을 관련도 **0.9 → 0.83** 으로 내렸다. 판정 프롬프트에 논문 눈금을 넣고 "강하게" 를 지웠다. 디스코드는 발송 직후 👍/👎 를 시드하고 `run_feedback` 잡이 10분마다 최근 100개 메시지를 읽어 반응을 `feedback` 에 쓴다.

실배치를 한 번 돌려 kind 가 실제로 돌아오는지 봤다. 50건 중 **technique 37 · survey 5 · other 6 · news 1 · promo 1**, 스키마 오류 0. 피드백 폴링은 HTTP 200 으로 채널 읽기 권한이 확인됐다.

## 왜

1차 구현서의 Task 1(users 테이블)·topics 는 사용자가 한 명인 동안 품질에 영향이 없고, kind 가중치·클러스터 상한은 모든 사용자에게 같은 전역 규칙이라 개인화가 아니다. 개인화의 입력은 policy 문장과 피드백뿐인데 피드백이 0건이었다. 그래서 순서를 "피드백부터 흐르게" 로 잡았다.

`technique` 가점을 뺀 이유는 점수식 때문이다. arXiv 는 `0.25×0.5 + 0.25×rel + 0.1` 이라 관련도 0.9 가 필요한데, +0.10 이면 0.5 부터 넘는다. 그 구간이 실측 154건이었다. 대신 신뢰도 비중을 관련도로 옮기면 0.83 부터 넘고 그 구간은 28건이다. `release_patch −0.15` 도 뺐다. 신뢰도 1.0 소스에서 관련도 1.0 을 요구해 GitHub 패치 릴리즈가 전멸한다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `41711a4` | 선별 출력에 kind, 점수 관문에 kind_weights (감점만) | `app/schemas.py` `app/config.py` `app/pipeline/triage.py` `app/pipeline/scoring.py` `app/jobs/pipeline.py` `config/rules.yaml` |
| `aaabe7f` | w_src 0.20 / w_rel 0.30, 통과 관련도 표 테스트 갱신 | `app/config.py` `config/rules.yaml` `tests/test_scoring.py` |
| `8a97fc7` | 판정 프롬프트 논문 눈금, "강하게" 삭제 | `app/pipeline/llm.py` `tests/test_prompts.py` |
| `6c654b8` | 통합 테스트 픽스처에 kind | `tests/integration/test_pipeline_db.py` |
| `bd300a1` | 리액션 시드·폴링, `run_feedback` 잡 | `app/notify/discord.py` `app/jobs/feedback.py` `app/jobs/scheduler.py` `scripts/run_job.py` |
| `075f090` | pydantic ValidationError → TriageBatchError (Codex 리뷰 반영) | `app/pipeline/triage.py` |
| `6ad4c22` | CLAUDE.md 에 feedback 잡·kind 항 | `CLAUDE.md` |

브랜치 `feat/kind-weights-feedback`. `main`(5e2ee0c) 에서 분기, 푸시·머지는 아직.

## 검증

- `uv run ruff check app tests scripts` · `mypy app` 통과. `ruff check .` 은 `main` 의 `.ua/` 쓰레기 커밋 때문에 깨져 있어 범위를 좁혔다(같은 날 정리).
- 단위 **129 passed**, 통합 **9 passed** (Neon dev, `sync_feedback` 테스트 추가). 모든 테스트는 실패를 먼저 보고 구현했다.
- Codex 리뷰 1회: Important 1건(ValidationError 층위) 반영 후 재리뷰 "없음".
- 실배치 `run_job.py collect pipeline` 1회, `run_job.py feedback` 1회. 부수 발견: `GITHUB_TOKEN` 401 로 github_release 수집 0건, Voyage 429 는 기존 알려진 제한.


## 후속 (같은 날 저녁)

- `GITHUB_TOKEN` 401 은 토큰 자체 문제였다. 17:43 수집 때는 옛 토큰, 18:00 에 `.env` 가 갱신됐고(`/user` 200, 잔여 4,997/5,000), 재수집은 **fetched 200 · inserted 5**, 저장소 실패 0.
- `.ua/` 104개 파일 추적 해제 + `.gitignore` 추가 (`f3080be`). `.gitignore` 마지막 줄에 개행이 없어 `.serena/.ua/` 로 붙는 바람에 둘 다 무시되지 않던 것을 고쳤다 (`9a9b96d`). 이제 `uv run ruff check .` 통과.
- 옵시디언이 꺼져 있어 기록이 한 번 실패했고, 켠 뒤 허브·기능 노트·결정 기록·이 노트를 썼다.
