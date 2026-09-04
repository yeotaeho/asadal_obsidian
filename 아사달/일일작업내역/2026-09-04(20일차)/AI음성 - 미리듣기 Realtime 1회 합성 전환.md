---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-04
---

# 미리듣기 Realtime 1회 합성 전환

> [!summary] 한 줄 요약
> 관리자 미리듣기를 TTS 에서 **OpenAI Realtime WebSocket 1회 합성**으로 바꿔 통화와 같은 모델·목소리로 읽게 했다. 브랜치 `feat/voice-preview-realtime` 에 커밋만 하고 푸시는 하지 않았다. 기능: [[음성채팅 안내문 온오프와 재생 제어]]

## 무슨 작업

회장님이 제안한 "저장 시 1회 TTS 로 녹음해 서버에 파일로 두고 재생" 방식을 검토해 회신 초안을 정리했고, 그 과정에서 TTS 판과 Realtime 판을 같은 문장으로 실측·청취 비교했다. 사용자가 두 파일에서 **목소리 차이를 들었고** Realtime 판을 택했다. 저장은 하지 않는다.

브레인스토밍으로 설계를 확정한 뒤 계획을 쓰고 서브에이전트 5 태스크로 구현했다. 새 패키지 `ai_voice/services/voice_chat/preview/` 에 벤더 분기 `speak_once` 와 OpenAI 모듈(WS 원시부 `RealtimeWsSession`·`connect` + 합성부 `synthesize`)을 두고, 라우터의 `POST /voice-preview` 를 TTS 호출에서 `speak_once` 호출로 바꿨다. 응답은 `audio/wav`.

```
요청  POST /voice-preview  { text (HTML 원문 ≤2000자), model: RealtimeModel, voice: RealtimeVoice }
      model·voice 는 v4 세션 생성과 같은 값·같은 화이트리스트
세션  session.update  { type: realtime, output_modalities: [audio], audio.output: {format pcm 24kHz, voice} }
응답  response.create { instructions: "다음 문장을 다른 말을 덧붙이지 말고 그대로 자연스럽게 말하라: …" }
수집  response.output_audio.delta 이어붙임 → response.done(completed) → PCM16 → WAV
오류  422 빈 문장 · 400 미지원 모델 · 502 합성 실패(연결·error 이벤트·status≠completed·오디오 0바이트·타임아웃·기타 예외) · 503 키 없음
```

타임아웃은 **15초 + 0.03초 × 문장 길이**(2000자 → 75초). 생성은 실시간의 약 9배 속도라 문장 길이에 비례해야 긴 문장이 502 로 죽지 않는다(Codex P2). 소켓은 성공·오류·타임아웃 모든 경로에서 닫힌다. 리뷰에서 `connect()` 중 타임아웃 취소가 `except Exception` 을 지나쳐 aiohttp 세션이 새는 결함을 잡아 `BaseException` 으로 고쳤다.

## 실측

같은 문장("안녕하세요. 무엇을 도와드릴까요?", 음성 2.7초) 3회.

| 회차 | gpt-4o-mini-tts (cedar) | gpt-realtime-2.1 (cedar) |
|---|---|---|
| 1 | 3.75s (첫 연결) | 2.59s |
| 2 | 1.62s | 5.02s |
| 3 | 1.72s | 1.73s |

Realtime 은 WS 연결 0.5~2.0초, 첫 오디오까지 0.7~2.7초, 이후 완료까지 0.3초다. 편차는 연결·응답 시작 구간이라 스트리밍으로도 줄지 않는다. **TTS 도 cedar 목소리 이름을 받는다**는 사실도 확인했다(400 이 날 줄 알았으나 정상). 구현 후 실제 키로 `speak_once` 를 호출해 **2.7초 음성, 2.96초 소요**를 확인하고 WAV 를 사용자에게 전달했다.

## 왜

회장님 방안의 전제("사용자 페이지가 매번 TTS 를 쓴다")가 실제와 달랐다. 통화의 첫·끝 인사말은 Realtime 이 `response.create` 지시문으로 직접 읽으므로 절약할 TTS 비용이 없고, 파일 방식은 TTS 목소리와 Realtime 목소리 차이를 미리듣기에서 실제 통화 첫 문장으로 옮겨 오며, 미저장 문구를 들을 수 없어 "듣고 고친다" 흐름이 깨진다. 저장 용량은 문장당 30KB 안팎이라 논거가 아니었다.

향후 Gemini Live 같은 다른 벤더 WS 음성 모델이 붙을 수 있어(사용자), `speak_once(text, model, voice, api_key)` 시그니처만 고정하고 `gpt-` 접두사로 OpenAI 모듈에 분기한다. 추상 클래스는 두지 않았다.

## 어디

브랜치 `feat/voice-preview-realtime` (`feat/voice-preview` 의 `21ab9ef` 에서 분기, **미푸시**).

| 커밋 | 내용 | 파일 |
|---|---|---|
| `fbe1a99` | OpenAI Realtime WS 1회 합성 모듈 — 원시부·합성부·WAV 변환 | `ai_voice/services/voice_chat/preview/openai_realtime.py` · `voice_tests/test_preview_openai_realtime.py` |
| `78647a4` | connect() 타임아웃 취소 시에도 aiohttp 세션을 닫는다 (리뷰 Important) | 〃 |
| `e9bf452` | 벤더 분기 `speak_once` | `preview/__init__.py` · `voice_tests/test_preview_speak_once.py` |
| `7576e0e` | 엔드포인트를 TTS 에서 Realtime 1회 합성으로, wav | `voice_chat_v4_router.py` · `voice_tests/test_voice_preview.py` |
| `49a60d8` | 타임아웃 문장 길이 비례, 모든 실패를 `PreviewSynthesisError` 로, 원시부 테스트 (Codex P2·리뷰) | `openai_realtime.py` · `test_preview_openai_realtime.py` |

설계·계획 문서는 `docs/superpowers/specs/2026-09-04-voice-preview-realtime-design.md`, `…-tts-design.md`, `docs/superpowers/plans/2026-09-04-voice-preview-realtime.md` 로 남겼고 커밋하지 않았다(사용자 지시).

## 검증

- `pytest ai_voice/voice_tests tests -q` **278 passed**. 신규 21개(합성부 18, 분기 3), 라우터 테스트 6개 교체. TTS 서비스 테스트(`test_openai_tts_payload.py`)는 그대로 통과.
- 태스크별 리뷰 3회(Important 1건 → 수정·재리뷰 통과), 브랜치 전체 리뷰 "머지 가능", Codex 범위 리뷰 P2 2건 → 수정 후 Codex 재리뷰 지적 없음.
- 실제 키 호출 1회로 소리 확인.

## 남은 일

프론트 계약 전달 — 요청에 `model`·`voice` 를 세션 생성과 같은 변수로 싣고, 응답 blob 을 `<audio>` 로 재생하며, 버튼에 로딩 표시(1.7~5.0초). 푸시·PR 은 사용자 결정. 통화용 시스템 프롬프트(말투) 주입은 최소 설정으로 시작했으니 들어 보고 다르면 추가.
