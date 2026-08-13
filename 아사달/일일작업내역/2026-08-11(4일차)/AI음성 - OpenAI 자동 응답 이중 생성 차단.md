---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# OpenAI 자동 응답 이중 생성 차단

> [!summary] 한 줄 요약
> "다른 창/기기" 오류의 진짜 원인 — 한 발화에 OpenAI 자동 응답과 서버 LangGraph 응답이 둘 다 생기고 있었다. v4 세션 빌더의 사본에서만 자동 응답을 껐다. 프로젝트: [[AI음성]]

## 무슨 작업

[[AI음성 - tool 제어 이벤트 필터]] 배포 후에도 증상이 그대로라 다시 조사했는데, 답은 화면 스크린샷에 이미 있었다. **어시스턴트 말풍선이 사용자 발화 하나당 두 개**였다.

```
"아."  →  "안녕하세요! 혹시 놀라셨거나…"        ← OpenAI 자동 응답 (RAG 없음, 부실)
       →  "잠깐만요, 관련 근거부터 확인해볼게요."  ← 서버 LangGraph 응답
```

GA `turn_detection`의 `create_response: True` 때문에 OpenAI가 발화 종료를 감지하면 **스스로 응답을 만든다.** 그런데 v4는 서버가 전사를 받아 LangGraph로 RAG를 붙인 답을 만들고 `response.create`를 또 보낸다. 겹치면 늦은 쪽이 거부당하고, 그게 `ACTIVE_RESPONSE_IN_PROGRESS`다.

서버의 `_should_emit_response` 가드는 `(turn_id, response_text)` 중복만 보므로 OpenAI가 스스로 만든 응답은 감지하지 못한다. 설정에서 꺼야 했다.

## 어떻게

v4 빌더 `_build_session_ga`가 `deepcopy`한 자기 사본에서만 끈다.

```python
session_cfg["audio"]["input"]["turn_detection"]["create_response"] = False
```

끼어들기(`interrupt_response`)와 VAD 감도는 그대로라 바지인은 유지된다.

**상수는 손대지 않았다.** `voice_chat_v3_router.py:153`이 같은 상수(`WEBRTC_VOICE_CONVERSATION_CONFIG_GA`)를 쓰는데, v3는 브릿지리스라 서버가 응답을 만들지 않는다 — 거기서 끄면 **응답이 아예 사라진다.** 그 사고를 막는 보호 테스트를 따로 뒀다.

## 어디

| 커밋 | 내용 | 브랜치 |
|---|---|---|
| `67f51b1` | v4 GA 세션에서 `create_response=False` + 테스트 4건 | `fix/v4-session-generation-guard-dev` |
| `203fbab` | 같은 내용 | `fix/v4-session-generation-guard` (main PR용) |

## 검증

`pytest ai_voice/voice_tests/` — **109 passed**.

| 변이 | 결과 |
|---|---|
| 덮어쓰기 제거 | v4 비활성 테스트 실패 |
| 상수를 직접 수정 (v3 오염) | v3 보호 테스트 **2건** 실패 |

Codex 리뷰 지적 없음. 배포 후 실기 확인 — **말풍선이 발화당 하나**로 정리됐고, 그 하나가 RAG를 반영한 답이다.
