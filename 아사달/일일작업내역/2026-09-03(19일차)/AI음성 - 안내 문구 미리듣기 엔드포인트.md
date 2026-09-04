---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-03
---

# 안내 문구 미리듣기 엔드포인트

> [!summary] 한 줄 요약
> 관리자 페이지 미리듣기를 OpenAI TTS API(`gpt-4o-mini-tts`) 단발 합성 엔드포인트로 구현했고, 그 과정에서 휴면 코드였던 `OpenAITTSService` 의 잘못된 요청 필드 3건을 실호출로 잡아 고쳤다. 기능: [[음성채팅 안내문 온오프와 재생 제어]]

## 무슨 작업

`POST /voice-chat-v4/voice-preview` 를 추가했다. 에디터 HTML 원문을 인사말 로더의 `strip_html` 로 정제하고, 음성은 기존 `RealtimeVoice` Literal(10종, 기본 `cedar`)로 검증한 뒤 mp3 를 `audio/mpeg` 로 돌려준다. 재생 제어(▶/⏸/⏹)는 브라우저 `<audio>` 기본 동작이라 서버 개입이 없다.

합성은 레포에 있던 휴면 코드 `OpenAITTSService` 를 그대로 썼다. Realtime WebSocket 으로 통화와 동일한 음성을 내는 안도 검토했으나 **미리듣기 한 번을 위한 세션 핸드셰이크는 과투자**라 기각했다(사용자 결정). 톤 차이가 크면 그때 WS 로 바꾼다.

## 휴면 코드의 결함 — 실호출로 확정

Codex 가 요청 필드명을 지적해 `.env` 키로 실제 OpenAI 를 호출해 봤다. 첫 mp3 호출은 성공했지만 그건 **mp3 가 기본값이라 잘못된 필드가 무시돼도 결과가 같았던 것**이었다.

| 결함 | 증상 | 수정 |
|---|---|---|
| 포맷 필드명 `format` | OpenAI 가 오류 없이 무시 — wav 요청에 mp3 응답 | `response_format` |
| 기본 포맷 `pcm16` | 필드명을 고치자 400 (지원값 mp3·aac·opus·flac·pcm·wav) | `pcm` (24kHz s16le) |
| `sample_rate` 파라미터 | API 에 없는 파라미터. 호출자 값을 결과에 그대로 적어 재생 속도가 틀어질 수 있음 | 인자·CLI 옵션 제거, 결과는 고정 24000 |
| `asyncio.TimeoutError` 미처리 | 엔드포인트가 500 으로 샘 | `TTSSynthesisError` 로 정규화 → 502 |

수정 후 실호출로 **wav 는 RIFF, mp3 는 audio/mpeg, pcm 은 audio/pcm** 을 확인했다.

## 왜

기획서 v2.0.4 슬라이드 1 의 미리듣기. 프론트 담당자 출근 전에 백엔드 계약을 준비해 두기 위해서다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `92d35e9` | 미리듣기 엔드포인트 + 테스트 5건 | `voice_chat_v4_router.py` `voice_tests/test_voice_preview.py` |
| `e1c07a9` | `format` → `response_format` (Codex P1) | `services/tts/openai_tts_service.py` `voice_tests/test_openai_tts_payload.py` |
| `0c2eebf` | 결과 metadata 도 `response_format` 에서 읽기 (Codex P2) | 〃 |
| `7fc657c` | 기본 포맷 `pcm16` → `pcm`, 타임아웃 정규화 (Codex P1·P2) | 〃 |
| `a552217` | `sample_rate` 인자 제거 (Codex P2) | `openai_tts_service.py` |

브랜치 `feat/voice-preview` (origin/main `8261d3f` 기준). 미푸시.

## 검증

- TDD — 엔드포인트·서비스 테스트 모두 실패를 먼저 확인. 전체 **256 passed**.
- 실호출 4회(mp3·wav·pcm16·pcm)로 필드명·포맷 지원값 확정. 시더 mp3 시험 음성을 사용자에게 전달해 통화 음성과 톤 비교를 요청.
- Codex 리뷰 5회전. 코드 지적 4건 전부 반영. 남은 1건은 **인증 부재(P1)** — 이 서비스는 세션 생성을 포함한 모든 엔드포인트가 무인증(프록시가 접근 통제 담당)이라 이 엔드포인트만 인증을 붙이는 건 일관성이 없고 프론트 계약도 없다. 사용자 결정 사항으로 넘김.

## 배운 것

외부 API 는 모르는 필드를 **오류 없이 버릴 수 있다.** 성공 응답이 왔다고 요청이 맞았다는 뜻이 아니다. 기본값과 다른 값을 요청해 응답이 실제로 바뀌는지 봐야 파라미터가 먹는 것이 증명된다. 이번엔 wav 요청 한 번이 그 증명이었다.
