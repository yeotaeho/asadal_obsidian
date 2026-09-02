---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-02
---

# Neon 연결과 첫 수집

> [!summary] 한 줄 요약
> `.env` 를 만들어 Neon 을 붙이고 마이그레이션·소스 동기화·첫 수집까지 돌렸다. 연결과 수집에서 버그 셋이 나와 고쳤고 결과는 **9개 소스 4,656건**. 기능: [[MVP 파이프라인]]

## 무슨 작업

프로젝트 루트에 `.env` 를 만들고 Neon pooled 연결 문자열을 넣었다. `alembic upgrade head` 로 스키마를 올리고 `sources.yaml` 을 DB 에 동기화한 뒤 `scripts/run_job.py collect` 로 전 소스를 수집했다.

첫 시도는 연결 자체가 죽었고, 두 번째 시도는 arXiv 적재에서 죽었다. 셋을 고친 세 번째 시도에서 **9개 소스 전부 성공, 1,099건 신규**(이전 시도의 3,557건과 합쳐 **4,656건**). `arxiv-cs-cl` 의 304→164 는 cs.AI 와 교차 등록된 논문 **140건**을 `url_hash` 가 걸러낸 것이다 — 중복제거가 실전에서 동작한 첫 사례.

## 왜

MVP 코드는 DB 없이 테스트한 순수 함수까지만 검증됐고, `pg_trgm`·`FOR UPDATE SKIP LOCKED`·`ON CONFLICT` 가 Neon 에서 실제로 도는지는 확인한 적이 없었다. 1일차 노트에 "그때까지는 파이프라인이 도는 것을 확인한 적이 없다"고 적어 둔 그 확인이다.

## 버그 셋

**연결 — asyncpg TLS 와 한글 홈 경로.** `sslmode=require` 를 asyncpg 에 넘기면 `~/.postgresql/root.crt` 를 읽는다. 이 PC 는 홈이 `C:\Users\여태호\` 라 경로가 깨져(`C:/Users/����ȣ/`) `ssl.load_verify_locations` 가 `OSError: [Errno 42] Illegal byte sequence` 를 낸다. asyncpg 는 `FileNotFoundError`·`NotADirectoryError` 만 잡는다(`connect_utils.py:713`). `ssl.create_default_context()` 를 `connect_args` 로 직접 넘겨 그 경로를 아예 타지 않게 했다. 덤으로 인증서 검증이 켜진다.

**적재 — `varchar(300)` 을 넘는 arXiv 저자 목록.** `items.author` 가 300자 제한이었다. 저자 수십 명인 논문에서 `StringDataRightTruncationError`. `author`·`external_id` 를 `text` 로 바꿨다(migration `0002`). Postgres 에서 `varchar(n)` 은 성능 이득이 없다.

**격리 — 적재 실패가 실패로 기록되지 않음.** 더 나쁜 건 그 예외가 `run_source` 의 `try` 밖으로 샜다는 것이다. `try` 가 `fetch()` 만 감쌌고 `store_items()` 는 밖에 있었다. 결과: `fail_count` 안 오름, `session_scope` 롤백, `run_all_sources` 중단 → 뒤 5개 소스 미실행. `store_items` 를 savepoint 로 감싸 실패 경로에 넣었다.

```
수정 전                              수정 후
try:                                 try:
    items = fetch()                      items = fetch()
except: 실패 기록                        savepoint 시작
inserted = store(items)   ← 여기서 죽음     try: inserted = store(items)
                                         except: savepoint 롤백; raise
                                         savepoint 커밋
                                     except: 실패 기록 (세션 살아 있음)
```

**덤 — Anthropic 공식 RSS 없음.** `/news/rss.xml` 404. 후보 4개 다 404. 커뮤니티 미러 `Olshansk/rss-feeds` 가 유일했다(260건, 최신 9월 1일, `anthropic.com` 원문 링크). 서드파티라 Codex 가 지적한 대로 `allowed_hosts` 검증을 붙여 외부 링크는 버린다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `dfbdef0` | asyncpg 에 SSL 컨텍스트 직접 전달 | `app/db/session.py`, `tests/test_session_url.py` |
| `f0bc18e` | 컬럼 `text` 화·적재 실패 격리·Anthropic 미러 | `app/db/models.py`, `alembic/versions/…_0002_…`, `app/jobs/collect.py`, `config/sources.yaml` |
| `de0616c` | `allowed_hosts` 검증 (리뷰 반영) | `app/sources/rss.py`, `config/sources.yaml` |

`.env` 는 커밋되지 않았다 (`git log --all -- .env` 비어 있음).

## 검증

```
PostgreSQL 18.6  neondb / neondb_owner
alembic current  0002 (head)
pg_trgm          설치됨, similarity() 동작
sync             9개 소스
collect          9/9 성공, 4,656건 NEW
pytest           56 passed  (연결 문자열 8개 + allowed_hosts 2개 추가)
ruff / mypy      통과
```

Codex 리뷰 **2회**. 1차에서 Important 1건(미러 신뢰) 반영, 2차에서 원 지적 닫힘 확인 + 중간 1건(리다이렉트 호스트)은 2단계 공격이라 후속으로 트리아지. 경위는 [[기술파악 - 결정 기록]].

## 다음

`ANTHROPIC_API_KEY` 와 텔레그램 봇 토큰이 비어 있어 아직 알림은 안 간다. 넣기 전에 **arXiv 튜닝**을 먼저 해야 한다 — trust 0.5 로 계산하면 fresh 논문이 0.475 > 0.45 라 하루 600건이 LLM 상한 300건을 잠식한다.

사용자 커밋 `63ee367` 에 `.ua/.trash-*/` JSON 1만 줄이 들어갔다. 도구 휴지통이라 `.gitignore` 에 `.ua/` 를 넣는 게 좋겠다.
