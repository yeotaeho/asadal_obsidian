---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-18
---

# ElevenLabs STT scribe_v2 전환

> [!summary] 한 줄 요약
> 2026-07-09 제거 공지된 `scribe_v1` 을 부르던 배치 STT·음성 학습 두 곳을 `scribe_v2` 로 바꿨다. 코드 **두 줄**, 응답 파싱은 무변경, 테스트 **307 + 8 passed**. 프로젝트: [[AI음성]]

## 무슨 작업

배치 STT 의 요청 `model_id` 와 음성 학습 클라이언트의 기본 `model_id` 를 `scribe_v2` 로 바꿨다. 두 파일은 함께 쓰는 모듈이 없어 공유 상수를 만들지 않고 리터럴 두 개만 고쳤다.

두 호출부가 보내는 `model_id` 를 고정하는 테스트를 먼저 썼고, 수정 전 `['scribe_v1', 'scribe_v1']` 로 실패하는 것을 확인했다. 본체 동기화로 v1 이 되돌아오면 이 테스트가 잡는다. 결정과 기각한 대안은 [[AI음성 - 결정 기록]] 아키텍처 표에 남겼다.

## 왜

ElevenLabs changelog 2026-06-08 이 `scribe_v1` 을 deprecated 로 두고 **2026-07-09 제거**를 공지했다. 오늘 기준으로 제거일이 두 달 넘게 지났다. 제거 뒤 요청이 에러인지 다른 모델로 넘어가는지는 6/8·7/20 changelog 와 모델 문서 어디에도 없다.

[[AI음성 - 잡음·본인 음성 학습·화자 구분 재검토]] 의 발견 9번에서 이어진 작업이다.

## 호출 경로

배치 STT 는 라우터와 파이프라인 두 곳의 `process_stt_audio` 가 모두 같은 함수로 모인다. 그래서 요청값 한 줄이 두 경로를 다 덮는다.

```
배치 STT
  stt_router.process_stt_audio ─┐
  stt/pipeline.process_stt_audio ┴→ voice_service_pool.get_stt_instance()
    → ElevenLabsSTT.process_audio_to_text_async → _call_elevenlabs_api   ← model_id 요청값

음성 학습
  learning_router → run_learning_stt_and_segments (engine == "elevenlabs")
    → transcribe_with_timestamps(diarize=True, timestamps_granularity="word", num_speakers=힌트)   ← model_id 기본값
    → extract_transcription_components → build_speaker_segments (raw_payload.words[])
```

## 호환성 대조

speech-to-text/convert 문서(9/18 조회)를 우리 코드가 보내고 읽는 필드와 대조했다. 모두 v2 에 그대로 있다.

| 구분 | 필드 | v2 |
|---|---|---|
| 요청 공통 | `model_id`, `language_code` | 그대로 |
| 요청 학습 | `timestamps_granularity=word`, `diarize`, `num_speakers`, `tag_audio_events` | 그대로. `num_speakers` 최대 32 |
| 응답 | `text`, `words[].text·type·start·end·speaker_id` | 그대로. `speaker_id` 예시는 `speaker_0` |
| v2 신규 | `diarization_threshold` | `diarize=true` 이고 `num_speakers` 가 없을 때만 설정 가능. 안 보내면 모델 기본값(약 0.22) |

한국어는 v2 지원 언어의 "Good" 등급(WER 10~20%)이다. 학습 경로는 `speaker_id` 를 `normalize_speaker_id` 로 받아 등장 순서대로 "화자N" 을 붙이므로 id 형식이 달라도 깨지지 않는다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `d2a1e0e` | 배치 STT 요청 `model_id` 를 `scribe_v2` 로 | `ai_voice/services/stt/elevenlabs_stt.py` |
| 〃 | 음성 학습 기본 `model_id` 를 `scribe_v2` 로 | `ai_voice/services/learning/elevenlabs_timestamped.py` |
| 〃 | 두 호출부의 요청 `model_id` 고정 테스트 | `ai_voice/voice_tests/test_elevenlabs_scribe_model.py` |

브랜치는 `fix-elevenlabs-scribe-v2`(main `6aa8263` 에서 분기)이고 아직 푸시하지 않았다. 본체 Chatty_Project `ai_voice/` 에는 같은 두 줄(`services/stt/elevenlabs_stt.py:87`, `services/learning/elevenlabs_timestamped.py:37`)이 `scribe_v1` 그대로 남아 있다.

## 검증

| 항목 | 결과 |
|---|---|
| 새 테스트, 수정 전 | 실패. `['scribe_v1', 'scribe_v1'] == ['scribe_v2', 'scribe_v2']` |
| 새 테스트, 수정 후 | 통과 |
| `pytest ai_voice/voice_tests -q` | **307 passed** (경고 18건은 기존 pydantic·FastAPI deprecation) |
| `pytest tests -q` | **8 passed** |
| venv 핀 | fastapi 0.115.9 · starlette 0.45.3 일치 |

## 남은 것

| 할 일 | 담당 | 이유 |
|---|---|---|
| 운영·dev 로그에서 7/9 이후 `API 호출 실패` 확인 | 사용자 (서버 접근) | 그동안 학습·배치 STT 가 실패하고 있었는지 판단 |
| dev 머지 후 음성 학습 업로드 1건 확인 | 사용자 | 화자 수 힌트가 없으면 v2 기본 임계값으로 화자 분리가 달라질 수 있음 |
| 본체 Chatty_Project 에 같은 두 줄 반영 | 미정 | 무수정 원칙상 두 저장소가 지금 다르다 |
| `elevenlabs_stt.py` 의 `"audio_events": False` | 별건 | API 에 없는 필드(`tag_audio_events` 가 맞음)라 태깅이 꺼지지 않는다. `clean_stt_text` 가 괄호 태그를 지워 영향은 작다 |
