---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-07
---

# v3 가드 브랜치 정리

> [!summary] 한 줄 요약
> `fix/v3-router-chat-id-guard-main` 에서 루트 `.gitignore` 수정을 원복하고 테스트 디렉터리를 `voice_tests` 로 옮겨, PR diff 에 공용 파일이 섞이지 않게 했다. 프로젝트: [[AI음성]]

## 무슨 작업

Gitea compare 화면에 루트 `.gitignore` 가 올라와 있는 걸 발견해 원인을 나눠 봤다. 두 갈래였다.

첫째는 정상이었다. dev 가 main 보다 **3커밋 뒤처져** 있어서(GLM 5.2 모델 추가 관련) `.env` 와 `utils/whoami.py` 가 diff 에 딸려 나온 것이다. main 기준 브랜치 워크플로의 구조적 결과라 브랜치 문제가 아니다.

둘째가 실제 문제였다. 이 브랜치는 `.gitignore` 정리 전 시점에서 cherry-pick 됐는데, 정리를 v4 브랜치에만 적용하고 여기엔 안 했다. 어제 v4 브랜치에 한 것과 같은 정리를 그대로 얹었다.

이 과정에서 커밋 방식도 바꿨다. 처음엔 `--amend` 로 넣었다가 강제 푸시가 필요해졌는데, 강제 푸시는 변경 크기 때문이 아니라 amend 라는 선택 때문에 생긴 것이라 새 커밋으로 되돌렸다. 최종 PR diff 는 두 방식이 동일하다.

## 왜

루트 `.gitignore` 의 bare `tests/` 패턴이 `ai_voice/tests/` 를 통째로 무시해버려서, 원래는 `!ai_voice/tests/` 네거션을 추가해야만 커밋이 됐다. 브랜치 사정으로 공용 파일을 건드린 셈이다. 디렉터리 이름을 바꾸면 그 패턴에 안 걸리므로 네거션 자체가 불필요해진다.

Codex 리뷰에서 로그 경로 지적도 나왔다. `LOG_DIR` 이 고정 경로라 `pytest-xdist` 워커들이 같은 `app.log` 를 동시에 열어 Windows 에서 공유 위반이 난다. `tempfile.mkdtemp()` 로 프로세스마다 고유 디렉터리를 받게 바꿨고, 0700 생성이라 리눅스 공유 `/tmp` 의 계정 간 충돌도 같이 사라진다. 같은 코드가 v4 브랜치에도 있어 **두 브랜치를 함께** 고쳤다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `1d6d54d` | 테스트 하네스를 `voice_tests` 로 옮기고 루트 `.gitignore` 원복 | `.gitignore`, `ai_voice/voice_tests/` |
| `70f74f8` | 테스트 로그 디렉터리를 프로세스마다 분리 (v4 브랜치) | `ai_voice/voice_tests/conftest.py` |

## 검증

`pytest ai_voice/voice_tests/` — v3 브랜치 **6 passed**, v4 브랜치 **43 passed**. 두 프로세스 동시 실행에서도 양쪽 통과.

`git diff origin/main -- .gitignore` 가 **0줄**이다. `git check-ignore -v` 로 `voice_tests` 두 파일이 정상 추적됨을 확인했다.

Codex 판정은 Critical 0건, Important 2건(로그 경로 — 수정, pytest 검증 불가 — 환경 정책 문제라 직접 실행으로 대체). Minor 중 "커밋 메시지가 diff 와 불일치" 지적은 amend 커밋을 delta 로 비교한 오독이라 반려했다.
