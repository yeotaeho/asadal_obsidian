---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-13
---

# GitHub 시크릿 등록과 배포 재실행

> [!summary] 한 줄 요약
> 자동 배포에 필요한 GitHub 시크릿 3개를 등록하고 실패했던 배포 잡을 다시 돌렸다. 그런데 GitHub 장애로 실행이 **큐에서 멈춰 있어** 시크릿이 실제로 동작하는지는 아직 확인하지 못했다. 기능: [[배포·CI-CD]]

## 무슨 작업

첫 배포 때 `Deploy to VM` 단계가 시크릿이 없어 실패했고, 그 아래 단계는 같은 명령을 SSH 로 직접 실행해 대신했다. 이번에 사용자가 시크릿 3개를 등록했다. 개인키는 Claude 가 입력하지 않고 사용자가 직접 붙여 넣었다.

| 시크릿 | 값 | 등록 시각 (UTC) |
|---|---|---|
| `VM_HOST` | `149.28.146.228` | 09:11:12 |
| `VM_USER` | `root` | 09:11:50 |
| `VM_SSH_KEY` | 로컬 `~/.ssh/trend_deploy` 전체 (ed25519, 공개키 주석 `gha-deploy-trend`) | 09:18:11 |

`GITHUB_TOKEN` 은 Actions 가 자동으로 주고 GHCR 푸시에만 쓴다. `GITHUB_` 접두사는 저장소 시크릿 이름으로 쓸 수도 없다. 앱 키(`ANTHROPIC_API_KEY`·`DATABASE_URL`·`DISCORD_*`)는 VM 의 `~/tech-radar/.env` 에만 있다. CI 는 이미지를 빌드하고 SSH 로 명령만 보내므로 이 값들을 알 필요가 없다.

## 배포 재실행이 멈춘 이유

`gh run rerun 34710955914 --failed` 로 실패한 잡을 다시 돌렸다. 대상 커밋은 `014c2d3`(PR #3 머지)이다. 그런데 **1시간 넘게 `queued`** 인 채로 러너가 잡을 하나도 받지 않았다.

```
09:18 UTC  VM_SSH_KEY 등록
09:19      재실행 요청 → queued
09:36      githubstatus: "Incident with several GitHub Services" 조사 시작
10:00      여전히 queued, 잡 0개 배정
           Partial System Outage — Actions degraded · Pull Requests major outage
```

우리 쪽 문제는 아니다. 앞서 `gh secret list` 가 먼저 **HTTP 500** 을 낸 것도 같은 장애 때문이다. 재시도하자 목록이 정상으로 나왔다.

**서비스에는 영향이 없다.** 재실행이 배포하려는 커밋은 VM 에 이미 수동으로 올려 둔 커밋과 같다. 실행이 돌면 같은 이미지를 한 번 더 올리는 셈이라, 이번 실행은 시크릿이 맞게 들어갔는지 확인하는 용도다. 실행은 큐에 그대로 두었다.

## 어디

| 구분 | 내용 |
|---|---|
| 워크플로 | `.github/workflows/ci.yml` 의 `deploy` 잡 `Deploy to VM` 단계 (`appleboy/ssh-action@v1`) |
| 실행 | run `34710955914` (push main, `014c2d3`) |
| 시크릿 위치 | `github.com/yeotaeho/trend/settings/secrets/actions` |

코드 변경은 없다.

## 검증

- `gh secret list` 에 세 이름이 모두 있다 (첫 호출 HTTP 500, 재시도로 성공).
- 로컬에서 `trend_deploy` 키로 VM SSH 접속이 되고, 파일 첫 줄이 `-----BEGIN OPENSSH PRIVATE KEY-----` 다. 붙여 넣은 시크릿 **내용**이 맞는지는 Actions 가 돌아야 알 수 있다.
- `https://trend.yeotaeho.kr/health` → **200** (0.38초).
- 앞서 디스코드 버튼 👎(uv 0.12.12, 03:47:15)·👍(LiteRAG, 03:47:39)가 `feedback` 에 들어오고 앱 로그에 `webhook.discord_feedback` 가 200 으로 남은 것을 확인했다. 👎 가 붙은 uv 0.12.12 는 0.12.13 을 먼저 보낸 뒤 나간 버전 형제 중복이다. 억제 규칙이 필요하다는 근거가 실제 라벨로 남았다.
- 배포 잡 결과는 **미확인**. GitHub 가 복구돼 실행이 끝나면 확인한다.
