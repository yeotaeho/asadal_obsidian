---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-10
---

# 음성채팅기록 dev 검증과 도구 호출 전 한마디 제외

> [!summary] 한 줄 요약
> dev 에서 첫 통화를 돌려 `voice_chat.turn` 이 브라우저까지 닿는 것을 확인했고, 그 실측이 **도구를 부르기 전 모델의 한마디가 답변으로 기록돼 짝이 한 칸씩 밀리는 결함**을 드러내 `response.done` 기준 기록으로 고쳤다. 기능: [[음성채팅기록]]

## 무슨 작업

프론트 두 파일에 추적용 `console.log` 를 붙여 dev 통화의 통지 4건을 눈으로 봤다. 첫 두 턴은 맞았고 **RAG 도구가 도는 3번 턴부터 답변이 한 칸씩 밀렸다.**

| 턴 | 질문 | 기록된 답변 | 실제 답변 |
|---|---|---|---|
| 3 | 여행지 하나 추천해줄래 | **좋아, 여행지 취향을 떠올리면서…** | 죄송하지만, 제공된 근거에서는… |
| 4 | 일본이나 하고 싶은데… | 죄송하지만, 제공된 근거에서는… | 도쿄, 교토, 오사카를… |

기록 시점을 전사 `.done` 에서 `response.done` 으로 옮기고, `response.output` 에 `function_call` 항목이 있으면 그 응답의 말은 안내말로 본다. 테스트 **347 passed** (345 + 2). Codex 리뷰 지적 없음.

## 왜

### 검증 명령이 두 번 틀렸다

처음 준 grep 은 나올 수 없는 문자열이었다. `voice_chat.turn` 은 DataChannel 페이로드 안에만 있고 로그에는 전송 *실패* 경고만 찍힌다. `Realtime 응답 텍스트` 는 `VOICE_CHAT_LOG_REALTIME_RESPONSE_TEXT` 가 켜져야 나온다.

다음으로 컨테이너 이름을 틀렸다. `docker exec ai_pro_voice` 가 `0` 을 돌려준 건 결함이 아니라 **그게 운영 컨테이너**여서다. dev 와 운영이 같은 서버에서 `ai_pro_voice_dev`/15252, `ai_pro_voice`/5252 로 돈다. dev 는 `dev` push 로 자동 배포되고 있었다 (`.gitea/workflows/deploy.yml`).

### 한마디의 정체

서버 안내말(`kind="tts"`)이 아니다. `_schedule_immediate_tts` 는 호출자가 0건이라 v4 GA 에서 안 돌고, `ImmediateRagResponseNode` 문구는 "~에 대해 한번 알아볼게요" 고정 템플릿이라 "좋아, 여행지 취향을 떠올리면서…" 가 나올 수 없다.

GA 모델이 `function_call` 을 내기 전에 **스스로 한마디** 한 것이다. 자동 응답이라 서버가 낸 `response.create` 가 없고 kind 도 없어 `"final"` 로 떨어졌다.

```
[턴 3]
  R1  자동 응답 ─ 전사 .done "좋아, 추천해볼게요"   ← 이 시점엔 도구 호출인지 모른다
      └ function_call_arguments.done              ← 그 뒤에 온다
      └ response.done  (output: [message, function_call])
  R2  서버 tool_continue ─ "죄송하지만…"
```

전사 `.done` 이 `function_call` 보다 먼저 오므로 `.done` 시점 기록으로는 못 거른다. `response.done` 은 응답 전체(`output`)를 실어 오니 거기서 판정한다. 끼어들기 경로는 이미 `response.done` 이 flush 하던 구조라 기록 경로가 하나로 합쳐졌다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `5061ebb` | 기록 시점을 `response.done` 으로, `output` 의 `function_call` 로 안내말 판정 | `ai_voice/services/voice_chat/server_webrtc.py` |
| 〃 | dev 실측 순서(질문 → 한마디+function_call → tool_continue × 2턴) 재현 테스트. 옛 코드에서 실패 확인 | `ai_voice/voice_tests/test_turn_log_end_to_end.py` |
| 〃 | "`.done` 만으로는 안 나가고 `response.done` 에서 나간다" 계약 테스트 + 기존 5개에 `response.done` 보강 | `ai_voice/voice_tests/test_server_webrtc_turn_log_hooks.py` |
| `30c6c2a` | dev 사본에 머지 (`15252`·`DEV-Ok` 유지) | `feat/voice-chat-log-dev` |

프론트 추적 로그(본체 `dev/`, git 밖) — `webrtc_voice_module.js:1227` `case 'voice_chat.turn'` + `CustomEvent('voice-chat-turn')`, `index2.js:1282` 리스너 등록 / `:1292` 해제. 다음 단계는 `index2.js` 의 `console.log` 를 `persistVoiceChatTurn(e.detail)` 로 바꾸는 것뿐이다.

## 검증

- `pytest ai_voice/voice_tests -q` **347 passed** — 원본·dev 사본 모두
- 새 테스트를 소스 수정 없이 돌려 **실패** 확인 후 수정본에서 통과
- `codex review --commit 5061ebb` — "response.output 으로 도구 호출 전 한마디를 걸러내며 정상·끼어들기 경로 모두 보존", 지적 없음
- dev 재검증은 `feat/voice-chat-log-dev` → `dev` PR 머지 뒤. 콘솔에서 턴 3·4 의 `a_text` 가 실제 답변으로 붙어야 한다

## 부수 발견

- `VOICE_CHAT_LOG_REALTIME_RESPONSE_TEXT` — `.env.example` 은 빈 값인데 코드는 키 부재에만 기본 `true`. 빈 값이면 꺼진다. 옆 키들은 `or` 로 흡수하는데 이것만 다르다. 본체 동일
- `_schedule_immediate_tts` → `_maybe_send_immediate_tts` → `ImmediateRagResponseNode` 사슬은 죽은 코드. 본체 동기화 원칙으로 미삭제
- 화면에서 답변 말풍선이 질문보다 위에 그려지는 건 사용자 전사가 늦게 도착해 생기는 기존 프론트 표시 순서 문제. 기록 짝과 무관
