---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-18
---

# 종료말 간헐 누락 수정

> [!summary] 한 줄 요약
> 종료 버튼을 눌러도 가끔 종료말이 안 나오던 결함을, 종료 시 마이크 입력을 서버에서 끊고 취소 확인을 받은 뒤 종료말을 내도록 고쳐 없앴다. 기능: [[음성채팅 종료 안내말]]

## 무슨 작업

종료 버튼은 OpenAI **출력**(생성·재생)만 끊고 **입력**(마이크 → OpenAI VAD)은 살려 두고 있었다. 그 상태에서 종료말은 취소 확인을 기다리며 큐에 서 있었다. 이 대기 구간과 종료말 발화 구간에 들어온 소리가 종료말을 지웠다.

`fix/voice-end-farewell`(origin/main 기준) 에서 테스트 6건을 먼저 써서 실패를 확인한 뒤 고쳤다. **`pytest ai_voice/voice_tests` 311 passed, `pytest tests` 8 passed.**

## 왜

세 경로가 있었고 모두 타이밍 창에 걸리는 간헐 증상이었다.

| 경우 | 창 | 무엇이 사라지나 |
|---|---|---|
| ① 발화 감지가 `queued` 종료말을 지움 | 취소 요청 → `response.done`/`cleared` 수신 (수백 ms) | 끼어들기 가드가 `speaking`·`draining` 만 막아 `_handle_interruption` 이 큐와 상태를 비움. 종료말도 `voice_end.done` 도 없이 프론트 타임아웃 |
| ② OpenAI 가 종료말을 스스로 취소 | 종료말 재생 전체 (2~3초) | `interrupt_response=True` 라 우리 가드와 무관하게 취소. `cancelled` 로 끝나 소리 없이 `voice_end.done` |
| ③ 서버가 모르는 자동 응답과 충돌 | 사용자 발화 끝 → `response.created` 도착 | `pending` 이 비어 cancel 없이 종료말 발행, OpenAI 거부. 뒤늦은 자동 응답이 종료말 id 로 오인됨 |

게이트 트랙은 발화를 문장 끝까지 붙들었다가 무음 400ms 뒤 방출한다. 그래서 **프론트가 마이크를 꺼도 클릭 직전의 말이 클릭 뒤에 OpenAI 로 간다.** 입력 차단을 서버에서 한 이유다.

```
종료 클릭
 → 업링크 프록시 pause (게이트가 붙든 발화도 방출 안 함)
 → input_audio_buffer.clear           새 자동 응답 재료 차단
 → response.cancel(pending 무관, 항상) + output_audio_buffer.clear + 큐 폐기   state=cancelling
 → 결과 대기: 몰랐던 응답의 response.created(→ queued, 그 done 대기) 또는 response_cancel_not_active 에러
 → 종료말 발행 → speaking → draining → stopped → 드레인 → voice_end.done
```

OpenAI 는 클라이언트 이벤트를 순서대로 처리하므로 cancel 의 결과가 오면 앞서 생긴 자동 응답은 정리돼 있다. 발행 뒤 첫 `response.created` 는 종료말이 된다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `cde22e6` | 업링크 정지 · 항상 cancel 후 확인 대기(`cancelling`) · 대기 중 시작된 재생 비움 · 종료 중 끼어들기 무시 · 확인용 에러 중계 제외 · `_handle_interruption` 의 죽은 `_clear_farewell` 제거 | `ai_voice/services/voice_chat/server_webrtc.py` `ai_voice/services/voice_chat/v4/audio_tracks.py` |

## 검증

- 새 테스트: `queued` 종료말이 발화 감지에 살아남음 · 업링크 정지와 `input_audio_buffer.clear` 가 cancel 보다 먼저 · 몰랐던 자동 응답의 취소를 기다렸다 발행하고 그 id 를 종료말로 오인하지 않음 · 취소할 응답이 없으면 에러를 확인으로 삼음 · 그 에러는 브라우저에 안 감 · 멈춘 게이트는 붙든 발화를 방출 안 함.
- 기존 테스트 5건은 `__new__` 브릿지에 `_uplink_tracks` 추가, 재생 제어 테스트에 확인 이벤트 한 줄 추가로 맞춤.
- `test_barge_in_releases_farewell_so_button_works_again` 는 도달 불가 경로가 돼 새 ① 테스트로 대체.

**dev 확인 필요** — `speech_started` 가 이미 난 상태에서 `input_audio_buffer.clear` 뒤 자동 응답이 정말 안 생기는가, 취소할 응답이 없을 때 `response_cancel_not_active` 가 항상 오는가.
