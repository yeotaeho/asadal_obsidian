---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-03
---

# compose 추적 제외와 브랜치 정리

> [!summary] 한 줄 요약
> dev PR 의 `docker-compose.yml` 충돌을 계기로 팀장 결정에 따라 compose 파일을 **git 추적에서 완전히 제외**(`docker-compose*.yml` gitignore, 참조본은 `docs/DEPLOY.md`)했고, 첫 배포에서 예고된 파일 삭제 실패를 복구했으며, 머지·기각이 끝난 브랜치 7개를 정리했다. 서비스: [[AI음성]]

## 무슨 작업

### 충돌의 원인

`feat/voice-playback-control` 을 origin/main 최신에서 땄더니 dev PR 에 팀장 배포 커밋 4개(`88e6623`~`8261d3f`)가 딸려 들어갔고, 그 안의 `docker-compose.yml` 수정이 dev 전용 compose(`ai_pro_voice_dev`/15252)와 부딪혔다. Chatty_Project 는 포트를 `config.py`·`.env`(gitignore)에서 읽어 추적 파일이 환경마다 같지만, 이 레포는 compose 안에 컨테이너명·포트가 직접 적혀 있어 dev·main 이 다른 내용으로 유지되는 구조였다.

우선 브랜치를 팀장 커밋 이전 main(`80d0f7b`)으로 리베이스해 PR 을 살렸고(`feat/voice-preview` 도 동일), 근본 해법으로 `.env` 치환안과 gitignore 안을 놓고 팀장이 **gitignore** 를 택했다.

### 추적 제외 3단계

| 커밋 | 브랜치 | 내용 |
|---|---|---|
| `b650ed5` (#33) | `chore/untrack-compose` (main) | `.gitignore` + `git rm --cached`, `docker-compose.example.yml` 신설, DEPLOY.md 문단 |
| `bec2148` (#34) | `chore/drop-compose-example` (main) | 예시 파일도 제거, 패턴을 `docker-compose*.yml` 로, 참조본은 DEPLOY.md 코드블록 |
| `7e3a0dc` | `chore/untrack-compose-dev` (dev) | 같은 변경을 dev 기준으로. DEPLOY.md·.gitignore 를 main 과 **바이트 동일**하게 맞춰 이후 main→dev 머지 충돌 방지 |

### 예고된 첫 배포 실패와 복구

#33 이 릴리즈 태그 `v1.1.3-app` 으로 운영 배포되자 `git checkout` 이 추적 해제된 compose 를 서버 작업 트리에서 지웠고, `[4/5] compose config` 가 `no such file` 로 멈췄다(빌드·up 전이라 컨테이너는 생존). 태그의 `docker-compose.example.yml` 이 삭제 직전 운영 compose(`743204a`)와 바이트 동일함을 확인하고, 운영 서버 `/home/chat_bot/Ai_Pro_Voice` 에 `docker-compose.yml` 을 되살린 뒤 재배포했다. dev 서버는 dev 값이어야 하므로 `git show d6cb4ae:docker-compose.yml` 로 복원하는 절차를 남겼다.

### 브랜치 정리

| 지운 브랜치 | 이유 |
|---|---|
| `chore/untrack-compose`, `chore/drop-compose-example`, `chore/untrack-compose-dev`, `chore/drop-dev` | 머지 완료, 고유 커밋 0 |
| `fix/v4-jitter-backlog-catchup`, `merge/v4-jitter-catchup-dev`, `merge/v4-jitter-revert-dev` | 적응형 배속 페이싱을 dev 에 넣었다 되돌린 이력. 기각된 기능, main 미반영 |

남긴 것 — `feat/voice-msg-toggle`·`feat/voice-playback-control`(dev 만 머지, main PR 대기), `feat/voice-preview`(로컬, 보류). 배속 페이싱 되돌림의 "미채택 사유" 주석(`1db9789`)은 dev 에만 있고 기능 차이가 없어 별도 브랜치를 만들지 않았다.

## 왜

한 파일이 환경마다 달라야 하는데 git 이 추적하면 브랜치를 오갈 때마다 충돌한다. `.env` 치환이 관리 부담은 더 작지만(구조 변경이 git 으로 흐름), 팀장이 서버 로컬 파일 방식을 택했다. 대가는 compose 구조가 바뀔 때 dev·운영 서버에 손으로 반영해야 하는 것이며, DEPLOY.md 의 참조본이 그 기준이다.

## 어디

| 구분 | 내용 |
|---|---|
| 레포 | AI_Pro_VOICE main `9324f9f`(#33) → `8a8b2fc`(#34), dev `7e3a0dc` |
| 파일 | `.gitignore` · `docs/DEPLOY.md` · (삭제) `docker-compose.yml`, `docker-compose.example.yml` |
| 서버 | `/home/chat_bot/Ai_Pro_Voice/docker-compose.yml`(운영 5252) · `/home/chat_bot/Ai_Pro_Voice_DEV/docker-compose.yml`(dev 15252) — 이제 서버 로컬 관리 |

## 검증

- 리베이스 후 `git merge-tree` 로 dev·main 충돌 마커 0 확인, 스위트 265 passed.
- dev 기준 브랜치의 `.gitignore`·`DEPLOY.md` 가 main 쪽과 `git diff` 빈 출력.
- 예시 파일 ↔ 삭제 직전 운영 compose `git diff` 빈 출력(복구 근거).
- 브랜치 삭제 전 `merge-base --is-ancestor` 와 `rev-list --count` 로 고유 커밋 0 확인.

## 배운 것

`git rm --cached` 로 추적을 풀면 그 커밋이 서버에 닿는 **첫 `reset --hard`/`checkout` 이 서버 파일을 지운다.** 추적 파일이 커밋에서 사라졌으니 작업 트리에서도 제거하는 게 정상 동작이다. 전환 커밋을 배포하기 전에 서버에서 백업하거나 첫 배포를 수동으로 하는 절차가 필요하다. 이번엔 예고했는데도 실제로 겪었으니, 다음에 같은 패턴을 쓸 때는 절차를 먼저 실행하고 머지한다.
