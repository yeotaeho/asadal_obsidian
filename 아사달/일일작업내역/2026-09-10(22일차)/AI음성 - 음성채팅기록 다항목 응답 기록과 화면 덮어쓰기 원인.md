---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-10
---

# 음성채팅기록 다항목 응답 기록과 화면 덮어쓰기 원인

> [!summary] 한 줄 요약
> dev 재검증에서 화면 답변이 서버 기록과 다른 현상을 추적해, **한 응답에 말 항목이 둘 실리는 경우** 서버는 마지막 항목만 남기고 프론트는 첫 항목으로 덮어쓰는 두 결함을 찾았다. 서버는 항목별 완성 전사를 모아 이어 붙이도록 고쳤고, 프론트는 원인 지점을 특정해 전달한다. 기능: [[음성채팅기록]]

## 무슨 작업

도구 호출 전 한마디 수정(`5061ebb`)을 dev 에 올려 다시 통화했다. 짝짓기는 정상이었는데, 화면 말풍선이 긴 답변을 다 그린 뒤 **서버가 한 번도 로그에 남기지 않은 짧은 문장**으로 바뀌었다.

| 질문 | 서버 기록 (`response.done`) | 화면 |
|---|---|---|
| 여행지 하나 추천해줄래 | 어디로 가느지 아직 정해지지 않았다면… (214자) | 좋아요, 여행 취향에 맞춰서 간단히 골라볼게요. |
| 일본으로 가고 싶어 | 교토를 추천해요… 홋카이도는… (224자) | 좋아요, 일본 여행 느낌을 몇 가지로 나눠서 바로 골라볼게요. |

서버 수정 후 **349 passed** (347 + 2). Codex 리뷰 지적 없음.

## 왜

### 전사 `.done` 은 응답 단위가 아니라 말 항목 단위다

Realtime 응답 하나는 출력 항목(message·function_call) 여럿을 실을 수 있다. `response.output_audio_transcript.done` 은 **항목마다** 오고, 응답의 끝은 `response.done` 하나다. 이번 두 질문에서 모델은 서두 한 문장과 긴 답변을 **한 응답에 항목 둘**로 냈다. 도구 호출은 없었다.

```
응답 R { output: [ ① "좋아요, …골라볼게요.",  ② "어디로 가느지 … 드릴게요." ] }

  ① delta… ① .done          서버 버퍼 = ①              화면 = ①
  ② delta… ② .done          서버 버퍼 = ② (① 덮음)      화면 = ② (스트리밍으로 긴 답변 표시)
  response.done               서버 flush → 기록 = ②       프론트가 output[0]=① 로 말풍선 덮어씀
```

서버 쪽은 직전 커밋에서 `.done` 이 버퍼를 교체하게 한 것의 한계였다. `ponytail:` 주석으로 "message 가 둘이면 앞 것을 잃는다" 고 적어둔 그 지점이 첫 재검증에서 바로 드러났다. 이번엔 서두가 앞이라 긴 답변이 남았지만, 순서가 반대였으면 긴 답변이 빠졌다.

### 프론트는 `response.done` 에서 첫 항목으로 되돌린다

`voice_chat_bridge.js:347` `handleResponseDone` 이 텍스트가 있으면 `onAssistantFinal` 을 한 번 더 부르고, 그 텍스트는 `voice_webrtc_handler.js:606` `extractRealtimeEventText` 가 `response.output` 을 돌다 **첫 번째** `transcript` 를 만나면 즉시 `return` 한 값이다. 항목별 `.done` 마다 누적을 지우므로(`:667`) 그 시점엔 누적이 비어 있어 이 추출값이 쓰인다. 프론트가 "응답 = 말 항목 하나" 를 전제한 것이다.

### 왜 항목이 둘인가 — 모델 행동이지 서버 지시가 아니다

서버는 응답을 하나만 요청하고(GA 자동 응답), 항목 수를 지시하지 않는다. 세션 지시문(`realtime_instruction_policy.py`)이 지식 질문에는 도구로 근거를 확인하라 하고, 잡담에만 "근거를 찾아보겠다는 안내도 하지 말라" 고 한다. 모델이 지식 질문으로 보고 **"골라볼게요" 서두를 먼저 낸 뒤, 도구를 부르지 않고 자체 지식으로 이어 답한** 결과다. 같은 통화의 "뭐 써야 하는지" 는 서두 뒤 실제로 도구를 불렀다. 둘 다 프로토콜상 정상이고, `.done` 이 둘인 것도 이상이 아니다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `7c8a7a0` | 항목별 완성 전사를 `_assistant_transcript_done` 에 모아 `response.done` 에서 순서대로 이어 붙임. 끊긴 항목 델타는 뒤에 | `ai_voice/services/voice_chat/server_webrtc.py` |
| 〃 | 두 항목 응답 기록 테스트 / 끊긴 둘째 항목 델타 이어붙임 테스트 / 대역에 dict 추가 | `test_turn_log_end_to_end.py` · `test_server_webrtc_turn_log_hooks.py` · `test_session_generation_guard.py` |
| `2458414` | dev 사본 머지 | `feat/voice-chat-log-dev` |

프론트 수정 지점(본체 `dev/`, 담당자) — `voice_webrtc_handler.js:606-611` 의 `return` 을 루프 밖으로 빼 항목 전부를 이어 붙이거나, `voice_chat_bridge.js:347` 이 이미 `.done` 으로 그린 응답은 다시 그리지 않게.

## 검증

- `pytest ai_voice/voice_tests -q` **349 passed** — 원본·dev 사본 모두
- `codex review --commit 7c8a7a0` — 지적 없음
- 실측 확인용 콘솔 한 줄 — 같은 `responseId` 로 `transcript.done` 이 두 번(itemId 다름), `response.done` 텍스트가 첫 항목이면 확정

```js
window.addEventListener('voice-webrtc-realtime-event', e => { const d = e.detail; if (/transcript\.done|response\.done/.test(d.type)) console.log('[trace]', d.type, d.responseId, d.itemId, (d.aggregatedText || d.text || '').slice(0, 40)); });
```
