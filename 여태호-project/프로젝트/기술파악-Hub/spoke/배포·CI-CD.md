---
tags: [여태호, 프로젝트, 기술파악, 배포]
created: 2026-09-13
updated: 2026-09-30
---

# 배포·CI/CD

> [!summary] 한 줄 요약
> Vultr 싱가포르 VM 한 대에 Docker Compose 로 올렸다. GitHub Actions 가 이미지를 빌드해 GHCR 에 올리고 VM 이 받아 띄우는 구조이며, **2026-09-13 첫 배포·HTTPS·디스코드 인터랙션 엔드포인트까지 완료**했다. GitHub 시크릿 3개도 등록했지만, GitHub 장애로 배포 재실행이 큐에 멈춰 있어 자동 배포가 되는지는 아직 확인하지 못했다. 서비스: [[기술파악]]

## 무엇을 했나

### 배포 대상

| 항목 | 값 |
|---|---|
| 제공자 | Vultr Cloud Compute (Shared CPU) |
| 인스턴스 | `vc2-1c-2gb` · 1 vCPU · 2GB · 55GB SSD · 2TB 전송 |
| 리전 | **Singapore, SG (APAC)** |
| IP | `149.28.146.228` |
| OS | Ubuntu 24.04.4 LTS x86_64 |
| 요금 | 월 $10.00 (자동 백업 해제로 $12 → $10) |
| 도메인 | `trend.yeotaeho.kr` (A 레코드 완료, Let's Encrypt 운영 인증서) |

리전 선택이 요금보다 중요했다. Neon 은 서울·도쿄 리전이 없고 아시아에서는 싱가포르와 시드니뿐이라 **DB 와 같은 리전**에 VM 을 뒀다. 한국에서 Neon 싱가포르까지 `session_scope` + 쿼리 1건이 커넥션이 이미 열려 있을 때 **590ms**, 커넥션을 새로 열 때 **1,512ms** 였다. 리전 안에서는 수십 ms 로 떨어져 디스코드 인터랙션 3초 예산과 파이프라인 처리량이 함께 좋아진다.

### CI/CD 구성

```
PR 푸시    ─▶ check (ruff · ruff format --check · mypy · pytest)

main 머지  ─▶ check ─▶ docker build ─▶ GHCR push (:latest + :<커밋SHA>)
                   └─▶ ssh VM ─▶ compose·Caddyfile 을 배포 SHA 에서 내려받기
                              ─▶ docker compose pull
                              ─▶ docker compose up -d --wait   (healthcheck 통과까지)
                              ─▶ caddy reload
                              ─▶ docker image prune -f
```

| 요소 | 내용 |
|---|---|
| 이미지 저장소 | `ghcr.io/yeotaeho/tech-radar`. 패키지가 **public** 이라 VM 에서 `docker login` 불필요 |
| 태그 | `:latest`(배포용) + `:<커밋SHA>`(롤백용) 두 개 |
| 롤백 | compose 참조가 `${IMAGE_TAG:-latest}`. VM 에서 `IMAGE_TAG=<SHA> docker compose pull && IMAGE_TAG=<SHA> docker compose up -d --wait` |
| VM 에 있는 파일 | `.env`(사람이 넣음) + `docker-compose.yml`·`Caddyfile`(배포마다 그 SHA 에서 내려받음). 코드는 없다 |
| 컨테이너 | `app`(시작 시 `alembic upgrade head` → uvicorn, FastAPI+APScheduler 한 프로세스) · `caddy`(80·443, Let's Encrypt 자동) |
| DB | 컨테이너 없음. Neon 외부, 같은 리전 |
| 성공 판정 | compose healthcheck(`python urlopen /health`) + `up -d --wait`. 실패 시 앱 로그 80줄 남기고 잡 실패 |
| 로그 | json-file `max-size 10m` × `max-file 3` |
| 동시 배포 | 워크플로 `concurrency` 로 앞 실행 취소 |

### GitHub 시크릿

| 이름 | 값 | 쓰는 곳 |
|---|---|---|
| `VM_HOST` | `149.28.146.228` | `Deploy to VM` |
| `VM_USER` | `root` | `Deploy to VM` |
| `VM_SSH_KEY` | 로컬 `~/.ssh/trend_deploy` 전체 | `Deploy to VM` |

세 개 모두 **2026-09-13 등록 완료**. `GITHUB_TOKEN` 은 Actions 가 자동으로 주고 GHCR 푸시에 쓴다. 앱 키는 VM `.env` 에만 있고 CI 는 알 필요가 없다.

### 첫 배포 (2026-09-13 03:28 KST)

| 단계 | 결과 |
|---|---|
| 인스턴스 생성 | 크롬 확장으로 Vultr 콘솔에서 직접. 사용자 승인 후 Deploy |
| Docker 설치 | 29.8.0 · Compose v5.5.1 |
| 방화벽 | ufw 가 기본 활성·22 만 허용이었다. **80·443 개방** |
| `.env` | 로컬에서 `TEST_DATABASE_URL` 만 빼고 전송, `GHCR_OWNER`·`DOMAIN` 추가, 권한 600 |
| CI | `check` 전부 통과, 이미지 빌드·GHCR 푸시 성공. `Deploy to VM` 은 시크릿 없어 실패 |
| 수동 배포 | CI 스크립트와 같은 명령을 SSH 로 실행 → **app · caddy 모두 Healthy** |
| 스케줄러 | 기동 직후 9개 소스 수집 확인 (09-11 수정 반영됨) |
| 로컬 정리 | 로컬 uvicorn 종료. 8000 포트 리스너 없음 |

### HTTPS 와 인터랙션 엔드포인트 (03:50 KST)

| 확인 | 결과 |
|---|---|
| DNS | `trend.yeotaeho.kr` → `149.28.146.228` (8.8.8.8 · 9.9.9.9 · KT) |
| 인증서 | **Let's Encrypt 운영** `O=Let's Encrypt, CN=YE2`, 2026-09-12 ~ 12-11 |
| `GET /health` | 200 `{"status":"ok"}` |
| `POST /webhook/discord` 서명 없음 | 401 |
| `POST /webhook/github` 서명 없음 | 401 |
| `http://` | 308 → `https://` |
| Interactions Endpoint URL | `https://trend.yeotaeho.kr/webhook/discord` **등록 완료** |
| 디스코드 검증 핸드셰이크 | 정상 서명 PING → **200**, 위조 서명 → **401** |
| 실제 버튼 클릭 | 👎 uv 0.12.12 (03:47:15) · 👍 LiteRAG (03:47:39) → `feedback` 2행, 로그 `webhook.discord_feedback` 200 |

DNS 가 없던 동안 ACME 가 반복 실패해 Caddy 가 **Let's Encrypt 스테이징으로 내려가** 재시도 간격이 5분까지 늘어나 있었다. 그대로면 신뢰되지 않는 인증서를 받게 되므로 `docker compose restart caddy` 로 상태를 초기화했고 곧바로 운영 인증서를 받았다.

API 로 엔드포인트를 등록할 때 `Python-urllib` 요청이 Cloudflare 에 차단됐다(오류 1010). 디스코드 API 는 `DiscordBot (<url>, <version>)` 형식 User-Agent 를 요구한다. `notify/discord.py` 가 `USER_AGENT` 를 명시하는 이유가 이것이다.

### 배포 재실행 (19:00 KST 기준 대기 중)

시크릿 등록 뒤 run `34710955914` 의 실패한 잡을 `gh run rerun --failed` 로 다시 돌렸다. **GitHub 장애**(Partial System Outage, Actions 성능 저하)로 1시간 넘게 `queued` 이고 러너가 잡을 받지 않았다. 대상 커밋 `014c2d3` 은 VM 에 이미 올라가 있어 서비스 영향은 없다. 이번 실행은 시크릿 내용이 맞는지 확인하는 용도다.

### 복제 레포 배포 가드 (2026-09-30)

같은 owner 의 복제 레포(`yeotaeho/trend-2`)에 푸시하면 `IMAGE` 가 운영과 같은 `ghcr.io/yeotaeho/tech-radar` 가 된다. deploy 잡 조건에 `github.repository == 'yeotaeho/trend'` 를 더해 **원본 레포에서만 배포**한다. 복제 레포에서는 check 만 돈다.

trend-2 run `36649963491` 에서 **check success · deploy skipped** 를 확인했다.

## 무엇이 달라졌나

**앱이 상시 가동된다.** 09-08~10 사흘, 09-11~13 29시간 동안 꺼져 있었다. 로컬 터미널 프로세스라 창을 닫으면 죽었다. 이제 `restart: unless-stopped` 로 컨테이너가 죽거나 VM 이 재부팅돼도 다시 뜬다.

**디스코드 버튼이 살아났다.** 지금까지 버튼은 "앱이 적시에 응답하지 않았습니다" 만 냈다. 앱의 `interactions_endpoint_url` 이 `None` 이고 앱이 127.0.0.1 만 듣고 있어 클릭이 도달조차 못 했다. 이제 두 경로(버튼·리액션)가 모두 `feedback` 으로 들어온다.

**DB 왕복이 짧아진다.** 같은 리전이라 파이프라인의 쿼리 수백 건이 네트워크에 묶이지 않는다. 인터랙션 핸들러가 응답 전에 DB 를 쓰는 구조를 손대지 않아도 3초 예산에 여유가 생겼다.

## 왜

개인화의 입력이 피드백인데 앱이 꺼져 있으면 라벨이 안 쌓인다. 2주 뒤 주간 리포트로 `kind_weights` 와 임계값을 조정하려면 그 사이 계속 돌아야 한다. 배포는 B2C 단계가 아니라 MVP 계획에 09-02 부터 있던 항목이다.

서버리스와 무료 PaaS 를 기각한 이유는 구조다. 스케줄러가 프로세스 안에 있고 429 게이트가 모듈 변수이며 설계서가 단일 프로세스를 전제한다. 유휴 시 0 으로 줄면 스케줄러가 죽는다. Oracle 무료 ARM 은 **7일간 CPU·네트워크·메모리가 모두 20% 미만이면 유휴로 보고 회수**하는 정책이 있고 우리 앱은 정의상 유휴다. 월 $10 로 그 위험을 없앴다.

## 어디

| 구분 | 내용 |
|---|---|
| 브랜치 | `main` (`014c2d3` 머지, PR #3) |
| 커밋 | `6126514` 파이프라인 보완 · `09cdafe` IMAGE_TAG·원자적 교체·reload · `1b18045` inode 유지 · `dd6789c` 트레이드오프 주석 |
| CI | `.github/workflows/ci.yml` |
| 복제 레포 | `yeotaeho/trend-2` main = `c29eeb2` (`fix/deploy-repo-guard`, 원본 미반영) |
| 컨테이너 | `docker-compose.yml` · `Dockerfile` · `Caddyfile` |
| 배포 키 | 로컬 `~/.ssh/trend_deploy`, Vultr 계정에 `gha-deploy-trend`, GitHub 시크릿 `VM_SSH_KEY` |

## 추후 작업

- [x] DNS A 레코드 `trend.yeotaeho.kr` → `149.28.146.228`
- [x] Let's Encrypt 운영 인증서 발급
- [x] 디스코드 Interactions Endpoint URL 등록 → 버튼 복구
- [x] GitHub 시크릿 3개 `VM_HOST`·`VM_USER`(root)·`VM_SSH_KEY` 등록
- [ ] **run `34710955914` 재실행 결과 확인.** GitHub 장애로 대기 중. `Deploy to VM` 이 초록이면 자동 배포 완성, 실패면 `VM_SSH_KEY` 내용(끝 줄바꿈 포함)부터 본다
- [ ] GitHub release 웹훅 `https://trend.yeotaeho.kr/webhook/github` 등록 (폴링 대기 없이 즉시 수집)
- [ ] SSH 연결이 간헐적으로 타임아웃했다(5회 중 4회 실패 후 성공). 로컬 회선인지 Vultr 쪽인지 관찰. CI 러너에서도 나면 재시도를 넣는다
- [ ] 공개 IP 라 스캐너가 `POST /graphql/api` 같은 요청을 보낸다. 404 로 끝나므로 조치 불필요, 로그 노이즈만 기억
- [ ] 배포 가드 커밋 `c29eeb2` 를 원본 trend 에도 PR 로 넣어 두 레포 이력 맞추기
- [ ] 관련도 0.8 개방(`threshold` 0.44 또는 `w_rel` 0.32), 버전 형제 억제 → [[개인화 1차]]

## 일일 기록

- [[기술파악 - trend-2 레포 복제와 배포 가드]]
- [[기술파악 - GitHub 시크릿 등록과 배포 재실행]]
- [[기술파악 - Vultr 배포와 CI-CD 구성]]
