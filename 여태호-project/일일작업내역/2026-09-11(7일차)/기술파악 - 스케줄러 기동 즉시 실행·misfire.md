---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-11
---

# 스케줄러 기동 즉시 실행·misfire

> [!summary] 한 줄 요약
> "앱이 적시에 실행되지 않는다" 를 조사해 APScheduler interval 트리거의 첫 실행이 등록 + 한 주기라는 걸 확인하고, 모든 잡을 기동 직후 1회 즉시 실행·밀린 회차 합치기·grace 무제한으로 바꿨다. 같은 날 오전에는 실배치로 카드 6장을 보내고 관련도 0.8 이 0.01 차로 막히는 걸 확인했다. 기능: [[MVP 파이프라인]], [[개인화 1차]]

## 무슨 작업

**오전. 실배치 실행.** `run_job.py collect pipeline notify feedback` 을 돌렸다. 첫 배치는 `scored=0`. arXiv 최고점이 **0.440** 으로 임계값 0.45 에 0.01 모자랐다. 관련도 0.8 → `0.20×0.5 + 0.30×0.8 + 0.10 = 0.44`. 통과선 0.8333 인데 선별 LLM 은 0.1 단위로 답하므로 실질은 0.9 만 통과한다. 어제 가중치 이동은 0.425 를 0.44 로 올린 데서 그쳤다. 배치를 세 번 더 돌려 GitHub 릴리즈 5건(anthropic-sdk-python v1.5.0, claude-code v2.1.268, ruff 0.16.7, uv 0.12.13, next.js canary.26)이 판정을 통과했고 발송 잡이 **push 2 · silent 3** 을 보냈다. 시드 리액션 10개가 전부 204. 직후 피드백 폴링은 `reacted=0` 으로 봇 시드를 사용자 반응으로 세지 않는 게 실데이터로 확인됐다.

**오후. 스케줄러 진단.** 사용자가 16:44:53 에 `uv run uvicorn app.main:app` 을 띄운 상태였다. 헬스체크 200, 파이프라인은 16:46:53 에, 발송은 16:47:53 에 첫 실행돼 탐색 슬롯이 🧪 카드(REVA) 를 보냈다. 스케줄러는 살아 있었다. 문제는 interval 트리거가 첫 실행을 등록 + 한 주기로 잡는 것. 기동 뒤 arXiv 는 60분, GitHub 릴리즈는 30분, 나머지 소스는 15분 동안 아무 수집도 없었다.

```
잡                     주기     기동 후 첫 실행
pipeline               120s     +2분
notify                 180s     +3분
feedback               600s     +10분
anthropic·openai·hf·yt 900s     +15분
vercel·github         1800s     +30분
arxiv ×2              3600s     +60분
```

`decisions` 활동 시간대를 보면 09-07 은 여섯 시간대에 흩어져 있고(스케줄러 가동) 09-08 이후는 하루 한 시간대뿐(수동 실행만)이라, 사흘간 프로세스가 꺼져 있었던 것도 확인됐다.

**수정.** `_interval(seconds)` 한 함수에 공통 옵션을 모았다. `next_run_time=now` 로 기동 직후 1회, `coalesce=True` + `misfire_grace_time=None` 으로 절전 복귀 시 밀린 회차를 한 번으로 합쳐 늦게라도 돈다. 기본 grace 1초는 밀린 회차를 통째로 버렸다.

## 왜

개인화의 입력이 피드백인데 앱이 꺼져 있으면 라벨이 안 쌓인다. 켜 둬도 첫 한 시간은 수집이 없어 "안 돈다" 로 보였다. 배포로는 프로세스 생존만 해결되고 첫 실행 지연은 컨테이너 재시작마다 반복되므로 코드에서 고쳐야 했다.

grace 를 숫자가 아니라 무제한으로 둔 이유는 이 앱의 잡이 전부 멱등이고(수집은 url_hash, 파이프라인은 NEW 만, 발송은 무음·상한 가드) 늦게 도는 게 안 도는 것보다 낫기 때문이다. 절전 직전 실행이 아직 안 끝났으면 `max_instances=1` 에 걸려 그 회차는 건너뛰고 다음 주기에 돈다. 동시 실행을 허용하는 것보다 한 주기 늦는 쪽을 택했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `f887616` | `_interval()` 공통 옵션, 즉시 실행·coalesce·grace 무제한 | `app/jobs/scheduler.py` `tests/test_scheduler.py` |
| `a27b251` | 실제 `start()` 로 네 잡 즉시 호출 검증, max_instances 한계 docstring (Codex 반영) | 같음 |
| `b0f02d2` | 고정 sleep → `asyncio.Event` 대기 (Codex 반영) | `tests/test_scheduler.py` |

브랜치 `feat/kind-weights-feedback`, PR #2 에 포함. 실행 중인 uvicorn 은 `--reload` 없이 떠 있어 재시작해야 반영된다.

## 검증

- 단위 **131 passed**. 새 테스트 2건은 실패를 먼저 보고 구현했다. 첫 테스트는 진짜 스케줄러를 띄워 네 잡이 즉시 불리는지 본다.
- Codex 리뷰 2회. Important 2건(밀린 회차와 max_instances 상호작용, 테스트가 pending 속성만 검사) 반영 후 재리뷰 Important 없음. Minor(고정 sleep) 반영.
- 실배치: 오늘 LLM 호출 선별 10 · 판정 5 · 탐색 3(상한). 카드 6장 발송, 시드 10개 204, 폴링 reacted 0.

## 남긴 것

- 관련도 0.8 을 열려면 `threshold` 0.44 또는 `w_rel` 0.32. 사용자 결정 대기.
- claude-code v2.1.267, next.js canary.23~25 가 아직 NEW 라 다음 배치에서 형제 중복 발송이 재현될 것. 버전 형제 억제 근거.
- YouTube 두 소스가 404 (어제는 정상). `fail_count` 1.
- 배포 권고: 작은 VM + Docker Compose (CI 가 이미 그 형태). Oracle 무료 ARM 이면 buildx 에 `linux/arm64` 추가.
