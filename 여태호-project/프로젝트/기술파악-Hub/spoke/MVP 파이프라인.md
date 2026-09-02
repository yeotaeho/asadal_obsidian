---
tags: [여태호, 프로젝트, 기술파악, MVP파이프라인]
created: 2026-08-30
updated: 2026-09-03
---

# MVP 파이프라인

> [!summary] 한 줄 요약
> 빈 레포에서 시작해 수집 → 관문 → 발송 전체를 구현하고, Neon 에 붙여 첫 수집 4,656건과 파이프라인 첫 실행까지 실전 확인했다. 발송 채널은 텔레그램에서 **디스코드 봇**으로 바꿨다. 봇 서버 초대와 채널 ID 교정 뒤 테스트 발송을 하면 알림이 가기 시작한다. 서비스: [[기술파악]]

## 무엇을 했나

### 1일차 — 구현 (2026-08-30)

구현도 로드맵의 **0단계(골격)** 와 **1단계(MVP)** 를 한 번에 만들었다.

| 계층 | 만든 것 |
|---|---|
| 골격 | `config.py`(pydantic-settings + YAML), `schemas.py`, `db/`(모델 6개 + Alembic), Dockerfile, Compose, Caddy, CI |
| 수집기 | `sources/base.py`(프로토콜 + 레지스트리), `rss.py`, `github_release.py`, `youtube.py` |
| 파이프라인 | `normalize.py`, `ingest.py`, `dedupe.py`, `rules.py`, `scoring.py`, `llm.py` |
| 발송 | `notify/base.py`, `policy.py`, `telegram.py` |
| 잡·API | `jobs/`(scheduler, collect, pipeline, notify), `api/`(health, github, telegram), `main.py` |
| 설정 | `config/sources.yaml`(소스 9개), `config/rules.yaml`(키워드 32개 + 가중치 + 발송 정책) |

### 2일차 — Neon 연결·첫 수집·파이프라인 첫 실행 (2026-09-02)

| 한 것 | 결과 |
|---|---|
| `.env` 작성, Neon pooled 연결 | asyncpg TLS 버그(홈 경로 한글) 수정 후 **PostgreSQL 18.6** 연결 |
| `alembic upgrade head` | `0001` 초기 스키마 + `0002` 외부 문자열 컬럼 확장 |
| 첫 수집 | 두 번째 시도에서 9개 소스 전부 성공, **4,656건** 적재 |
| 파이프라인 첫 실행 | 3회 실행. 1차 크래시 → 수정 → 2·3차 배치 완주 |
| 실행이 드러낸 결함 | 화이트리스트 점수 탈락, dedupe 기준, savepoint 만료 |

### 3일차 — 디스코드 전환 (2026-09-03)

| 한 것 | 결과 |
|---|---|
| 발급 가이드 + `.env` 키 3개 | `DISCORD_BOT_TOKEN` · `DISCORD_CHANNEL_ID` · `DISCORD_PUBLIC_KEY` |
| `notify/discord.py` | 채널 메시지 + 👍/👎 버튼, `@silent` 플래그, 마크다운 이스케이프, 멘션 차단 |
| `api/discord.py` | Ed25519 서명 검증 → PING 응답 / 버튼 → `feedback` INSERT → ephemeral 답장 |
| 재시도·rate limit | 429 는 `retry_after` 존중, **프로세스 전역 게이트**, 상한(30초) 초과·3회 소진은 `RateLimited` 로 배치 중단 |
| URL 스킴 가드 | `store_items` 에서 http(s) 아니면 적재 거부 (모든 수집 경로 공통) |
| 키 검증 | 토큰 유효(봇 `trend`). 채널 ID 는 봇 자신의 ID 가 들어가 404, 봇은 아직 어느 서버에도 미초대 |

## 무엇이 달라졌나

문서만 있던 레포에 **돌아가는 파이프라인**이 생겼다. 소스 하나 추가가 파일 하나 + YAML 항목 하나로 끝나고, 알림 피로 규칙은 `notify/policy.py` 한 곳에만 있다. 소스 파일 **34개**, 테스트 **73개**.

발송 계층이 `Notifier` 프로토콜 덕에 채널 교체 비용이 낮았다 — 디스코드 어댑터 하나와 인터랙션 라우터 하나를 붙이고 발송 잡의 기본값만 바꿨다. 텔레그램 어댑터는 대안 구현으로 남아 있다.

Codex 리뷰를 **16회** 돌려 Important 지적 **26건**을 반영했다. 그냥 뒀으면 서비스를 망가뜨렸을 것들.

- **봇 토큰 노출** — httpx 예외 문자열이 URL(토큰 포함)을 담아 DB·로그로 새어나갔다.
- **영구 누락** — GitHub 저장소 하나가 실패해도 `last_polled_at` 이 전진했다.
- **decisions 무한 증식** — LLM 이 죽으면 2분마다 같은 판정이 쌓였다.
- **화이트리스트가 점수 관문을 못 넘음** — kw=0 이라 GitHub 릴리즈·YouTube 가 전부 탈락했다.
- **dedupe 가 알림된 적 없는 항목을 기준으로 삼음** — 같은 이슈의 재등장이 죽었다.
- **디스코드 rate limit** — `retry_after` 를 잘라 조기 재시도했고, 긴 429 뒤에도 다음 요청을 바로 보냈다. 프로세스 전역 게이트로 막았다.
- **링크 버튼 URL 미검증** — 피드가 준 `javascript:` 스킴이 버튼에 실릴 수 있었다. 적재 단계에서 막는다.

## 왜

기획서·구현도가 v0.1/v0.2 까지 확정돼 있었고, 로드맵이 "0·1단계를 2주 안에"를 최우선으로 잡았다. 매일 쓰는 상태에 먼저 도달하는 게 이 프로젝트의 최대 리스크(지속성)를 줄인다.

디스코드로 바꾼 건 사용자 결정이다. 웹훅 URL 하나로도 보낼 수는 있지만 웹훅 메시지에는 버튼이 없어 피드백(4단계 개인화의 입력)이 빠지므로 봇 방식을 택했다.

## 어디

| 구분 | 내용 |
|---|---|
| 브랜치 | `main` (`2024245`..`6d2cec8`, 27커밋 — 도구 산출물 `63ee367` 포함) |
| 엔트리 | `app/main.py` |
| 관문 | `app/pipeline/`, 오케스트레이션은 `app/jobs/pipeline.py` |
| 발송 | `app/notify/discord.py`, 정책은 `app/notify/policy.py` |
| 인터랙션 | `app/api/discord.py` |
| 설정 | `config/sources.yaml`, `config/rules.yaml`, `.env`(커밋 금지) |

## 문서와 다르게 간 것

| 문서 | 실제 | 왜 |
|---|---|---|
| 텔레그램 봇 선행 (구현도 전제) | **디스코드 봇** (텔레그램은 대안 어댑터) | 사용자 결정. 어댑터 추상화 덕에 교체 비용이 낮았다 |
| python-telegram-bot 21.x | httpx 직접 호출 | 쓰는 API 가 둘뿐 |
| APScheduler Postgres jobstore | 기본 MemoryJobStore | 저장할 상태가 없다 |
| `user_prefs` 테이블 | `config/rules.yaml` | 구현도가 YAML 대체 허용 |
| `Notifier.send(item, summary, level)` | `+ source_name` | "출처" 줄에 필요 |
| Anthropic 공식 RSS | 커뮤니티 미러 + `allowed_hosts` | 공식 RSS 없음 |
| 화이트리스트도 점수화 | 점수 관문 우회 (48h 이내) | 한국어 제목은 kw=0 |
| 중복은 `cluster_id` + 대표의 `mention_count` | dedupe 기준은 살아남은 항목만, `mention_count` 는 직접 계산 | 판정 전 항목이 기준이 되면 통과할 항목이 죽는다 |

## 추후 작업

- [x] Neon 연결 + 마이그레이션 + 첫 수집 + 파이프라인 첫 실행
- [x] `ANTHROPIC_API_KEY` 등록
- [x] 디스코드 어댑터·인터랙션 엔드포인트
- [ ] **봇을 서버에 초대** (OAuth2 URL, 권한 19456) + **채널 ID 교정** (개발자 모드 → 채널 우클릭)
- [ ] 채널이 잡히면 테스트 발송 1건 (사용자 허락 후)
- [ ] Interactions Endpoint URL 등록 (배포 후, `https://<도메인>/webhook/discord`)
- [ ] Neon `dev`/`main` 브랜치 분리
- [ ] GitHub 저장소에 release 웹훅 등록
- [ ] VM 배포 후 2주 사용 → 기획서 9장 성공 기준 점검
- [ ] 임계값·가중치 튜닝 (`decisions` 로그 근거)
- [ ] 주간 튜닝 리포트 스크립트 (구현도 7.3)
- [ ] 디스코드 429 게이트는 프로세스 단위 — 워커 분리 시 공유 저장소로 (리뷰 후속)
- [ ] `mention_count` trigram 비용 — 데이터가 커지면 `EXPLAIN ANALYZE` (리뷰 후속)
- [ ] 다중 워커 시 A·B 동시 SCORED 경합 (리뷰 후속)
- [ ] 본문 보강의 리다이렉트 최종 호스트 재검증 (리뷰 후속)
- [ ] outbox 전환 시 `notifications(item_id, channel)` 유니크 제약
- [ ] GitHub 폴링을 repo 별 커서로
- [ ] `.ua/` 를 `.gitignore` 에

## 일일 기록

- [[기술파악 - 디스코드 전환]]
- [[기술파악 - 파이프라인 첫 실행]]
- [[기술파악 - Neon 연결과 첫 수집]]
- [[기술파악 - MVP 파이프라인 구현]]
