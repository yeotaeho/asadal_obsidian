---
tags: [여태호, 프로젝트, 기술파악, MVP파이프라인]
created: 2026-08-30
updated: 2026-08-30
---

# MVP 파이프라인

> [!summary] 한 줄 요약
> 빈 레포에서 시작해 수집 → 관문 → 발송이 이어지는 0·1단계 전체를 구현했다. 아직 실제 알림을 받아본 상태는 아니고, Neon 연결과 봇 등록이 남았다. 서비스: [[기술파악]]

## 무엇을 했나

구현도 로드맵의 **0단계(골격)** 와 **1단계(MVP)** 를 한 번에 만들었다. 소스 파일 **31개**, 테스트 **46개**.

| 계층 | 만든 것 |
|---|---|
| 골격 | `config.py`(pydantic-settings + YAML), `schemas.py`, `db/`(모델 6개 + Alembic), Dockerfile, Compose, Caddy, CI |
| 수집기 | `sources/base.py`(프로토콜 + 레지스트리), `rss.py`, `github_release.py`, `youtube.py` |
| 파이프라인 | `normalize.py`, `ingest.py`, `dedupe.py`, `rules.py`, `scoring.py`, `llm.py` |
| 발송 | `notify/base.py`, `policy.py`, `telegram.py` |
| 잡·API | `jobs/`(scheduler, collect, pipeline, notify), `api/`(health, github, telegram), `main.py` |
| 설정 | `config/sources.yaml`(소스 9개), `config/rules.yaml`(키워드 32개 + 가중치 + 발송 정책) |

## 무엇이 달라졌나

문서만 있던 레포에 **돌아가는 파이프라인**이 생겼다. 소스 하나 추가가 파일 하나 + YAML 항목 하나로 끝나고, 알림 피로 규칙은 `notify/policy.py` 한 곳에만 있다.

Codex 리뷰를 **3회** 돌려 Important 지적 **10건**을 반영했다. 특히 다음 셋은 실제로 서비스를 망가뜨릴 결함이었다.

- **봇 토큰 노출** — httpx 예외 문자열이 URL(토큰 포함)을 담아 `notifications.error` 와 docker 로그로 새어나갔다.
- **영구 누락** — GitHub 저장소 하나가 실패해도 `last_polled_at` 이 전진해서 그 사이 릴리즈를 다시는 못 봤다.
- **decisions 무한 증식** — LLM 예산이 바닥나거나 API 가 죽으면, 판정만 기록하고 `NEW` 로 남은 항목을 **2분마다** 다시 판정했다.

## 왜

기획서·구현도가 v0.1/v0.2 까지 확정돼 있었고, 구현도 8장 로드맵이 "0·1단계를 2주 안에 끝내는 것을 최우선"으로 잡고 있었다. 설계 논의를 더 하는 것보다 매일 쓰는 상태에 먼저 도달하는 게 이 프로젝트의 최대 리스크(지속성)를 줄인다.

## 어디

| 구분 | 내용 |
|---|---|
| 브랜치 | `feat/mvp-pipeline` (`2024245`..`bd9ccf7`, 8커밋) |
| 엔트리 | `app/main.py` |
| 관문 | `app/pipeline/` |
| 알림 규칙 | `app/notify/policy.py` |
| 설정 | `config/sources.yaml`, `config/rules.yaml` |

## 문서와 다르게 간 것

| 문서 | 실제 | 왜 |
|---|---|---|
| python-telegram-bot 21.x | httpx 직접 호출 | 쓰는 API 가 `sendMessage`·`answerCallbackQuery` 둘뿐이라 의존성 하나를 줄였다 |
| APScheduler Postgres jobstore | 기본 MemoryJobStore | interval 잡을 기동 시 매번 재등록하므로 저장할 상태가 없다 |
| `user_prefs` 테이블 | `config/rules.yaml` | 구현도가 "MVP 는 YAML 로 대체 가능"이라 명시 |
| `Notifier.send(item, summary, level)` | `+ source_name` 인자 추가 | 메시지의 "출처: …" 줄에 소스 이름이 필요한데 async 지연 로딩은 피했다 |

## 추후 작업

- [ ] Neon 프로젝트 생성 + `dev`/`main` 브랜치 분리, `alembic upgrade head`
- [ ] `.env` 채우고 텔레그램 봇 생성 → `setWebhook` 등록
- [ ] GitHub 저장소에 release 웹훅 등록
- [ ] VM 배포 후 2주 사용 → 기획서 9장 성공 기준 점검
- [ ] 임계값·가중치 튜닝 (`decisions` 로그 근거)
- [ ] 주간 튜닝 리포트 스크립트 (구현도 7.3)
- [ ] 발송을 예약-후-전송(outbox)으로 바꿀 때 `notifications(item_id, channel)` 유니크 제약 함께 도입
- [ ] GitHub 폴링을 repo 별 커서로 (지금은 첫 페이지 20건만 봄)

## 일일 기록

- [[기술파악 - MVP 파이프라인 구현]]
