---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-13
---

# Vultr 배포와 CI/CD 구성

> [!summary] 한 줄 요약
> "앱이 적시에 실행되지 않는다" 가 스케줄러가 아니라 **디스코드 버튼 인터랙션 에러**임을 밝히고, 그 해결책인 배포를 실제로 마쳤다. Vultr 싱가포르 VM 에 첫 배포 성공, 두 컨테이너 Healthy. 기능: [[배포·CI-CD]]

## 무슨 작업

**진단부터 뒤집혔다.** 이틀 전 같은 문구를 스케줄러 문제로 보고 `_interval()` 을 고쳤는데, 사용자가 "디스코드에서 피드백을 줄 때 나오는 에러" 라고 알려줬다. 확인해 보니 완전히 다른 경로였다.

| 확인 | 결과 |
|---|---|
| 앱의 `interactions_endpoint_url` | **None (미등록)** |
| 봇의 게이트웨이 연결 | 없음 (REST 만 씀) |
| 앱이 듣는 주소 | 127.0.0.1:8000 (외부 도달 불가) |
| 카드의 버튼 | 정상 부착 (`fb:useful:21592`) |
| 카드의 리액션 | 👍 1(봇만), 👎 **2**(봇 + 사용자) |

디스코드가 버튼 클릭을 전달할 경로가 아예 없어서 3초를 기다리다 "앱이 적시에 응답하지 않았습니다" 를 띄운 것이다. 요청은 앱에 도달조차 하지 않았다. 반면 같은 카드의 👎 **리액션은 정상 기록**됐다. 09-10 에 리액션 경로를 넣은 이유가 정확히 이것이었다.

**배포 조사.** Oracle 무료 ARM 은 유휴 회수 정책 때문에 기각하고 Vultr 싱가포르를 골랐다. 리전은 요금이 아니라 **Neon 과 같은 리전** 기준으로 정했다. 한국에서 Neon 싱가포르까지 왕복이 따뜻할 때 **590ms**, 콜드 **1,512ms** 였다.

**CI/CD 보완.** `check` 와 GHCR 푸시는 이미 동작했고 막힌 곳은 SSH 단계였다. SHA 태그·`IMAGE_TAG` 롤백·compose 를 배포 SHA 에서 내려받기·`up -d --wait` 헬스체크·Caddyfile reload·로그 회전·`concurrency` 를 넣었다.

**크롬 확장으로 인스턴스 생성.** Vultr 콘솔을 직접 조작했다. 구성만 맞추고 멈춰 사용자 승인을 받은 뒤 Deploy 를 눌렀다. 진행 중 두 가지가 기본값과 달랐다. OS 기본이 Ubuntu 26.04 라 24.04 LTS 로 바꿨고, 자동 백업이 켜져 월 $12 였던 것을 껐다.

**첫 배포.** Docker 29.8.0 설치, ufw 가 22 만 열려 있어 80·443 개방, `.env` 전송(600), main 머지(`014c2d3`) 로 이미지 빌드. CI 의 SSH 단계는 시크릿이 없어 실패했으므로 **같은 명령을 수동으로 실행**해 띄웠다. `app`·`caddy` 모두 Healthy.

## 왜

앱이 꺼져 있으면 피드백이 쌓이지 않고, 피드백이 없으면 2주 뒤 리포트로 임계값을 조정할 근거가 생기지 않는다. 그리고 버튼 문제는 코드로 못 고친다. 공개 HTTPS 엔드포인트가 있어야 디스코드가 클릭을 배달한다.

`misfire_grace_time` 을 숫자가 아니라 무제한으로 둔 것과 같은 판단을 여기서도 했다. `alembic` 실패 시 재시작 루프를 Codex 가 지적했지만 고치지 않았다. 일시적 Neon 장애에서 스스로 복구되는 쪽이 단일 사용자 상시 서비스에 맞고, 영구 실패는 `IMAGE_TAG` 롤백으로 되돌린다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `6126514` | SHA 태그·compose 동기화·헬스체크 대기·로그 회전·concurrency | `.github/workflows/ci.yml` `docker-compose.yml` `.env.example` `.gitignore` |
| `09cdafe` | `IMAGE_TAG` 롤백, compose 원자적 교체, Caddyfile reload (Codex 반영) | 위 둘 |
| `1b18045` | Caddyfile 은 inode 유지로 덮어쓰기 (Codex 반영) | `ci.yml` |
| `dd6789c` | 비원자성 트레이드오프 주석 (Codex 트리아지) | `ci.yml` |
| `014c2d3` | PR #3 머지 → 첫 이미지 빌드 | `main` |

## 검증

- `ruff check .` · `ruff format --check .` · `mypy app` 통과, 단위 **131 passed**
- CI: `check` 8단계 전부 success, `docker build-push` success, `Deploy to VM` 만 실패(시크릿 없음)
- 배포: `docker compose up -d --wait` → `app Healthy` · `caddy Healthy`
- 외부: `curl -H "Host: trend.yeotaeho.kr" http://149.28.146.228/health` → **308 Permanent Redirect**, `Server: Caddy`. HTTPS 는 DNS 없어 아직 인증서 미발급
- 스케줄러: 컨테이너 기동 직후 9개 소스 `collect.done` 확인
- 로컬 uvicorn 종료 확인 (8000 리스너 없음). 두 스케줄러 동시 가동으로 인한 중복 발송 방지
- Codex 리뷰 5회. Important 6건 반영, 1건 트리아지

## 남긴 것

- DNS A 레코드와 GitHub 시크릿 3개는 사용자 작업이다. 시크릿이 없으면 `main` 푸시마다 SSH 단계가 계속 빨갛다
- SSH 연결이 간헐적으로 타임아웃했다. 5회 시도 중 4회 실패 후 성공. 원인 미확정
- 자기 자신을 매칭하는 프로세스 필터로 PowerShell 세션을 한 번 죽였다. `Get-CimInstance` 필터에 명령문 문자열이 걸리는 함정


## 후속 — HTTPS 와 버튼 복구 (03:50 KST)

사용자가 DNS A 레코드를 넣은 뒤 인증서를 확인했다. 두 군데서 막혔다.

**첫째, Caddy 가 스테이징으로 내려가 있었다.** DNS 가 없던 동안 ACME 가 반복 실패하면서 Let's Encrypt **스테이징** 으로 폴백하고 재시도 간격이 120초 → 300초로 늘어나 있었다. 그대로 두면 브라우저와 디스코드가 신뢰하지 않는 인증서를 받는다. `docker compose restart caddy` 로 상태를 초기화하니 즉시 운영 인증서를 받았다. 발급자 `O=Let's Encrypt, CN=YE2`, 유효기간 **2026-09-12 ~ 12-11**.

**둘째, Cloudflare 가 API 요청을 차단했다.** 엔드포인트 등록을 `urllib` 로 PATCH 했더니 **HTTP 403 오류 1010** 이 났다. 디스코드 API 는 `DiscordBot (<url>, <version>)` 형식 User-Agent 를 요구한다. `notify/discord.py` 가 `USER_AGENT` 를 명시하는 이유를 여기서 다시 확인했다. curl 로 UA 를 주자 바로 성공했다.

검증 결과다.

| 확인 | 결과 |
|---|---|
| DNS | 8.8.8.8 · 9.9.9.9 · KT 모두 `149.28.146.228` |
| `GET /health` | 200 `{"status":"ok"}` |
| `POST /webhook/discord` 서명 없음 | 401 |
| `POST /webhook/github` 서명 없음 | 401 |
| `http://` | 308 → `https://` |
| Interactions Endpoint | 등록 완료. 정상 PING **200**, 위조 서명 **401** |

디스코드는 URL 을 저장하기 전에 정상 서명 PING 과 위조 서명 요청을 함께 보내 **둘 다 올바르게 처리해야** 저장한다. 앱 로그에 `POST /webhook/discord 200` 과 `401` 이 연달아 찍혀 핸드셰이크가 그대로 확인됐다.

배포 직후 상태는 NEW 0건, QUEUED 1건(importance 4, 무음 시간이라 08:00 대기), 최근 1시간 LLM 호출 0건이다. 백로그가 다 빠지고 새 항목이 없는 정상 유휴다.
