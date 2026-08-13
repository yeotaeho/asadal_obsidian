---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-06
---

# v4 기본 활성 전환과 테스트 경로 정리

> [!summary] 한 줄 요약
> v4를 환경변수 없이도 쓰도록 기본 활성으로 뒤집고, 루트 `.gitignore` 수정을 되돌리기 위해 테스트 디렉터리를 `voice_tests`로 옮겼다. 프로젝트: [[AI음성]]

## 무슨 작업

두 가지를 처리했다.

**① 테스트 경로 정리** — 루트 `.gitignore`의 bare `tests/` 패턴을 피하려고 `!ai_voice/tests/` 부정 패턴을 넣었던 것을 되돌렸다. 디렉터리 이름을 `ai_voice/voice_tests/`로 바꾸니 공용 설정을 전혀 건드리지 않고 같은 목적을 달성한다. 기능 옆에 `test_*.py`를 두는 이 저장소의 기존 관례와도 맞다.

**② v4 기본 활성** — `ENABLE_VOICE_V4=on`을 설정해야 등록되던 게이트를 뒤집어, 환경변수 없이 기본으로 v4를 쓰도록 바꿨다. 테스트는 **41 → 43건**으로 늘었다.

## 왜

`.gitignore`는 공용 설정이라 한 기능을 위해 손대는 게 부담스럽다. 확인해보니 이 저장소는 이미 `chat/private_rag/test_*.py` 등 테스트를 추적하고 있었고, `tests/` 무시 패턴은 정책이라기보다 우연히 걸린 것이었다. 이름만 바꾸면 되는 문제였다.

v4 기본 활성은 사용자 지시다. 다만 v4는 2026-07-13에 롤백된 이력이 있어 **끄는 스위치는 남겼다**. 문제가 생기면 코드 배포 없이 `ENABLE_VOICE_V4=off`만으로 즉시 되돌릴 수 있어야 한다.

알 수 없는 값(오타 등)은 기본값을 유지하도록 했다. `ENABLE_VOICE_V4=ON ` 같은 실수 하나로 v4가 **조용히 꺼지는** 편이 더 위험하기 때문이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `1e708e6` | `ai_voice/tests` → `ai_voice/voice_tests` rename, 루트 `.gitignore` origin/main 상태로 원복, 중첩 `.gitignore` 제거, conftest 테스트 로그를 저장소 안(`_tmp_logs`)에서 시스템 임시 폴더로 이동 | `.gitignore`, `ai_voice/voice_tests/*` |
| `a9e7add` | v4 기본 활성 전환 — `VOICE_V4_DEFAULT_ENABLED` 상수 + `_voice_v4_enabled()` 헬퍼, 게이트 테스트 5종 재작성 | `ai_voice/voice_router.py`, `voice_tests/test_v4_router_gate.py`, `test_v4_integration_smoke.py`, `conftest.py` |

`1e708e6`으로 이전 커밋 `498c4b4`(gitignore 예외 추가)는 무효화됐다.

### 게이트 동작

| 환경변수 | 결과 |
|---|---|
| 미설정 | **v4 활성** (기본값) |
| `off` `0` `false` `no` | 비활성 |
| `on` `1` `true` `yes` | 활성 |
| 오타 등 알 수 없는 값 | **활성** (기본값 유지) |

`aiortc` import 실패 시 `None` 폴백은 그대로이며 게이트보다 우선한다.

### 갱신한 테스트

```
test_v4_hidden_by_default            → test_v4_exposed_by_default
test_gate_on_exposes_both_v3_and_v4  → test_default_exposes_both_v3_and_v4
+ test_v4_hidden_when_explicitly_disabled   (off/0/false/no + 대소문자·공백)
+ test_v4_exposed_when_explicitly_enabled   (on/1/true/yes + 대소문자·공백)
+ test_unknown_value_falls_back_to_default
```

## 검증

- `pytest ai_voice/voice_tests/` — **43 passed** (fastapi==0.115.9)
- 환경변수 없이 실제 마운트 확인 — `VOICE_V4_DEFAULT_ENABLED = True`, `/AI_voice/voice-chat-v4/agent/session` 노출
- 불변식 유지 — `.gitignore` · `voice_chat_v3_router.py` · `constants.py` 전부 origin/main과 diff **0**
- 옛 경로(`ai_voice/tests`) 참조 잔존 없음
