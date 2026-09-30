---
tags: [여태호, 일일작업, 기술파악, gastown, 멀티에이전트]
created: 2026-09-23
---

# Gas Town 에이전트 협업 환경 설치

> [!summary] 한 줄 요약
> WSL2 Ubuntu 에 Gas Town(gt 1.2.1)을 설치하고 trend 저장소를 rig 로 붙였다. `gt doctor` 는 **92개 통과·실패 0** 이다. 남은 단계는 WSL 안에서 Claude 로그인 후 `gt up` 하는 것이다. 서비스: [[기술파악]]

## 무슨 작업

Gas Town 은 Claude Code 세션 여러 개와 Git 작업 공간을 관리하는 별도 프로그램이다. Mayor 가 작업을 나누고, Polecat 이 작업하고, Witness 가 작업자를 감시하고, Refinery 가 병합 대기열을 처리한다. 앱 코드는 바꾸지 않았고 저장소 밖 WSL 환경만 구성했다.

| 구성 요소 | 버전·위치 |
|---|---|
| Go | **1.27.1** `/usr/local/go` |
| Dolt | **2.3.5** `/usr/local/bin/dolt` |
| gt (Gas Town) | **1.2.1** `/usr/local/bin/gt` |
| bd (Beads) | **1.0.4** `/usr/local/bin/bd` (의도적으로 낮춤) |
| Claude Code (Linux) | **2.1.280** `/home/taeho/.local/bin/claude` |
| HQ | `/home/taeho/gt` |
| rig | `~/gt/trend` (beads prefix `tr`) |
| crew | `~/gt/trend/crew/taeho` |

## 왜 이렇게 구성했나

**일반 사용자 `taeho` 로 실행한다.** Gas Town 은 에이전트를 기본값 `claude --dangerously-skip-permissions` 로 띄운다. Claude Code 는 root 에서 이 플래그를 거부하고, 기존 WSL 은 root 만 있었다. sudo 없는 `taeho` 를 만들어 기본 사용자로 지정했고, 관리 작업은 `wsl -u root` 로 한다.

**bd 를 1.0.4 로 고정했다.** `@latest` 의 bd 1.3.0 은 새 DB 스키마를 커밋하지 않은 채 마이그레이션 가드에 걸려 `gt install` 이 실패했다(beads#4566, 9/12 에도 Gas Town rig 생성에서 재현 보고). gt 1.2.1 릴리스(6/6) 시점 최신인 1.0.4 로 맞추자 해결됐다.

**WSL interop 을 껐다.** WSL 안에서 Windows 쪽 `gh.exe`·`git.exe` 가 로그인된 자격 증명으로 실행되고 있었다. 권한 확인 없이 도는 에이전트가 main 에 push 하면 곧 운영 배포다. `/etc/wsl.conf` 에 `[interop] enabled=false` 를 넣어 Windows 실행 파일 호출을 막았다. `/mnt/c` 파일 접근은 유지된다.

**push 인증을 일부러 넣지 않았다.** Refinery 기본 병합 방식이 `direct` 이고, formula 에서 `git push origin <main>` 을 직접 실행한다. `auto_push=false` 만으로는 막을 수 없어서 WSL 에 GitHub 인증이 없는 상태를 차단 장치로 쓴다.

**비용 한도를 걸었다.** `gt config cost-tier economy` 로 상시 도는 patrol 역할(Deacon·Witness·Refinery)을 Haiku·Sonnet 으로 내렸고, `scheduler.max_polecats` 를 무제한(-1)에서 **2** 로 줄였다.

## 설치 중 막힌 곳

```
bd 1.3.0 + gt install  → schema migration dirty tables (beads#4566) → bd 1.0.4 로 교체
rig beads prefix       → DB 가 'trend', routes 가 'tr' → bd rename-prefix tr-
rig identity bead 누락  → bd create --id=tr-rig-trend --type=rig 수동 생성
HQ .claude/settings.json → git 추적 중이라 doctor 가 못 지움 → git rm 후 커밋
Git Bash 경로 변환       → wsl --cd /home/... 가 Windows 경로로 바뀜 → 스크립트 안에서 cd
```

## 남은 단계

- [ ] WSL 에서 `claude` 실행해 로그인과 첫 실행 온보딩 완료
- [ ] `cd ~/gt && gt up` 후 `gt mayor attach` 로 분석 전용 작업 1건 시험
- [ ] 코드 작업을 맡기기 전에 병합 정책 결정 (`merge_strategy: pr` 또는 별도 통합 브랜치)
- [ ] 테스트를 돌리게 하려면 WSL 에 `uv` 설치

## 검증

- `gt doctor` → **92 passed, 1 warning(daemon 미실행, gt up 때 기동), 0 failed**
- `whoami` → `taeho`, `which claude` → `~/.local/bin/claude`, PATH 의 `/mnt/c` 항목 **0개**
- `gh.exe --version` → `Exec format error` (interop 차단 확인)
- `gt scheduler status` → `Capacity: 2 free of 2`

## 추가 수정 — 대화형 wsl 이 root 로 뜨던 문제

직접 `wsl` 을 열자 `taeho` 가 아니라 root·HOME=`/`·`sh` 로 떨어졌다. Windows 레지스트리의 Ubuntu `DefaultUid` 가 **1000** 인데 Ubuntu 안에는 UID 1000 사용자가 없었다(`taeho` 는 1001). 레지스트리를 따르는 실행 경로는 사용자를 못 찾아 기본값으로 떨어진다. `wsl --manage Ubuntu --set-default-user taeho` 로 **1001** 로 맞췄다.

interop 을 끈 부작용도 있었다. Ubuntu Pro 연동 서비스 `wsl-pro.service` 가 Windows `cmd.exe` 를 못 불러 2초마다 재시작했고 재시작 횟수가 **803회**였다. 쓰지 않는 서비스라 `systemctl disable --now` 후 mask 했다.

## 추가 수정 — Refinery 가 폴더 신뢰 창에서 멈춤

`gt up` 에서 Refinery 만 `timeout waiting for runtime prompt` 로 실패했다. tmux 화면을 떠 보니 Claude Code 가 "Quick safety check" 폴더 신뢰 창에서 멈춰 있었다.

```
신뢰 판정: 현재 폴더 → 부모로 올라가며 찾되 git 저장소 루트에서 멈춤
~/gt (HQ 저장소, 신뢰함)
~/gt/trend/witness        → HQ 저장소 안 → 통과
~/gt/trend/refinery/rig   → trend 저장소 루트 → 신뢰 창
~/gt/trend/polecats/<이름>/trend, crew/taeho → 같은 문제
```

Gas Town 의 `AcceptWorkspaceTrustDialog` 는 "1번이 Yes" 라고 가정하고 Enter 만 보낸다. 그런데 Claude Code 2.1.280 은 **1번이 `No, exit`** 라서 자동 수락은 믿을 수 없다. `gt down` 뒤 `~/.claude.json` 에 refinery·mayor·crew 경로와 polecat 이름 50개 경로, 모두 **53개**를 신뢰 등록했다. 다시 `gt up` 하자 모든 서비스가 올라왔고 `gt doctor` 는 **93 passed, 0 warnings, 0 failed** 다.
