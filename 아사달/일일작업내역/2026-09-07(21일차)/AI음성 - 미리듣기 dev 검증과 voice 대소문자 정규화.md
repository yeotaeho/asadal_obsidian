---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-07
---

# 미리듣기 dev 검증과 voice 대소문자 정규화

> [!summary] 한 줄 요약
> dev 에 배포된 Realtime 미리듣기가 올바른 요청에 **200 audio/wav** 를 돌려주는 것을 확인했고, 프론트의 422 원인이 `voice: "Cedar"`(대문자)임을 찾아 서버가 소문자로 정규화해 받도록 고쳤다. 기능: [[음성채팅 안내문 온오프와 재생 제어]]

## 무슨 작업

사용자가 `feat/voice-preview-realtime` 을 푸시·dev 머지·배포한 뒤 관리자 페이지에서 미리듣기를 눌렀더니 422 가 났다. 같은 주소 `https://api-aipro.chatbaram.com/voice/dev/voice-chat-v4/voice-preview` 를 UTF-8 JSON 으로 직접 두드려 배포 상태부터 확인했다.

| 요청 | 응답 |
|---|---|
| `{text, model: gpt-realtime-2.1, voice: cedar}` | **200 audio/wav, 2.85초 음성, 3.73초 소요** |
| `{text, voice: marin}` (model 생략) | 200, 기본 모델 적용 |
| `{text: "<p>&nbsp;</p>"}` | 422 `읽을 문구가 없습니다.` |
| `voice: "시더"` / `model: gpt-4o-mini-tts` / `text` 누락 / 2001자 | 422 `literal_error` / `literal_error` / `missing` / `string_too_long` |

서버는 정상이었고, 프론트 Payload 탭에서 `voice: "Cedar"` 가 원인으로 드러났다. `RealtimeVoice` Literal 은 대소문자를 구분한다.

## 왜

본체 저장 요청(`chatbot_config`)도 `voice_tts: "Cedar"` 를 보내고, 본체는 `_normalize_voice_name` 으로 소문자화해 저장한다. 프론트가 화면 표기를 그대로 보내는 것이 이미 관행이므로 미리듣기 요청도 같은 규칙으로 받는 게 맞다(사용자 결정). 화이트리스트 자체는 유지해 오타(`Ceder`)는 여전히 422 다.

Windows 셸 curl 로 한글 본문을 보내면 코드페이지 때문에 400 `error parsing the body` 가 나서 처음엔 오진할 뻔했다. 한글 JSON 검증은 파이썬 `httpx` 로 UTF-8 바이트를 직접 보내야 한다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `8251f72` | `VoicePreviewRequest.voice` 에 `field_validator(mode="before")` 로 strip·lower | `ai_voice/infrastructure/api/router/voice_chat_v4_router.py` · `ai_voice/voice_tests/test_voice_preview.py` |

브랜치 `feat/voice-preview-realtime`, 이 커밋은 **미푸시**.

## 검증

- `pytest ai_voice/voice_tests tests -q` **279 passed** (신규 `test_preview_request_accepts_capitalized_voice`).
- Codex 범위 리뷰(`--base 49a60d8`) 지적 없음.
- dev 실호출 200 WAV 를 사용자에게 전달.
