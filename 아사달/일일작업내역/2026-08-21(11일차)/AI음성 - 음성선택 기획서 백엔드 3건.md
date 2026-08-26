---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-21
---

# 음성선택 기획서 백엔드 3건

> [!summary] 한 줄 요약
> 음성선택 기획서 V1.9 를 소스와 대조해 프론트 연동 전 백엔드에서 막아야 할 3건을 고쳤다. 기능: [[음성 선택과 대기시간 옵션]]

## 무슨 작업

기획서 4장을 `ai_voice` 소스와 대조해 과제 2(시간 옵션)·과제 3(음성 10종)의 준비 상태를 확인했다. 결론은 **두 기능 다 파라미터는 이미 뚫려 있고 프론트가 실어 보내기만 하면 된다**는 것이었지만, 그대로 넘기면 터질 함정 3개가 같이 나왔다.

세 건을 한 커밋으로 고쳤다. 대조 과정에서 확인한 프론트 요청 사양도 정리해 전달했다.

| 건 | 문제 | 조치 |
|---|---|---|
| 기본 모델 | 라우터 기본값 `gpt-realtime-1.5` 가 **2026-05-12 폐지된 Beta 평면 스키마** 경로다. 프론트가 `model` 을 생략하면 죽은 API 로 세션을 열러 갔다 | v3·v4 기본값을 `OPENAI_DEFAULT_MODEL`(`gpt-realtime-2.1`)로 |
| voice 검증 | 허용값 검증이 없어 오타가 그대로 OpenAI 로 갔다. 세션 생성이 실패하면 **통화 자체가 안 열린다** | 기획서 10종 `Literal` 로 받아 422 로 차단 |
| 대기시간 미사용 | 라디오의 "미사용" 을 표현할 스위치가 없었다 | `wait_time_enabled=False` 면 게이트 없는 트랙으로 전환 |

**음성 목록 API 는 만들지 않았다.** `Literal` 이 OpenAPI 스키마에 그대로 실려 프론트가 `/docs` 에서 읽어간다.

## 왜

기획서의 "대기시간" 이 코드의 어느 파라미터인지부터 확정해야 했다. 정의가 **"챗봇이 답변 중일 때 사용자 음성을 새 질문으로 인식하기까지의 시간"** 이라 `barge_in_ms` 와 정확히 일치했고, 기본값도 양쪽 다 1.2초로 같았다.

문제는 그 값 하나만 낮추면 안 된다는 것이었다. 방출 전에 3관문을 통과해야 하는데 기본값이 `min_utterance 1200` · `min_speech 600` 이라, 대기시간만 0.5초로 내리면 0.5초 시점에 정산이 돌고 관문 미달로 **발화가 폐기·리셋된다**. 무시되는 게 아니라 말을 잃는다.

```
대기시간 T 적용 시
  스케일 O  barge_in_ms = T · min_utterance_duration_ms = T · min_speech_duration_ms = T/2
  스케일 X  speech_ratio_threshold (비율은 무차원) · speech_rms_threshold (음량 축)
            silence_frames_to_end (발화 "종료" 판정, 다른 축) · force_flush_ms
```

미사용 분기는 새로 만들 게 없었다. `server_webrtc:1323` 이 `audio_track_cls is not None` 일 때만 게이트로 감싸므로, 게이트 없는 트랙을 넘기면 그대로 투명 릴레이가 된다.

## 되돌린 판단 — 기반 클래스 직접 사용

처음엔 미사용 경로에 기반 클래스 `AudioProxyTrack` 을 그대로 쓰려 했다. 두 가지 이유로 접었다.

v3 사본의 `max_buffer_ms` 는 **320** 이다. 미사용을 켜는 순간 `26f6560` 에서 960 으로 올린 모바일 지터 대응(실측 지터 **633~772ms**)이 통째로 되돌아간다. v4 사본은 960 이지만 이번엔 로그 꼬리표가 `PROXY-DOWN` 이라, 업링크 언더플로가 다운링크로 찍혀 방향 구분이 사라진다.

그래서 꼬리표만 바꾼 `UplinkAudioProxyTrackV4` 를 뒀다. 5줄이고, 지터버퍼·좀비 판정은 전부 상속받는다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `c5e754b` | GA 기본 모델 · voice 10종 Literal · 대기시간 미사용 트랙 | `constants.py`, `voice_chat_v3_router.py`, `voice_chat_v4_router.py`, `v4/audio_tracks.py` |

브랜치는 `feat-voice-selection-api` (`caf8066`..`c5e754b`).

## 검증

- `pytest ai_voice/voice_tests/` — **185 passed** (기존 180, 신규 5). 로컬 `.venv` 는 핀(fastapi 0.115.9)과 일치해 `test_default_exposes_both_v3_and_v4` 도 통과했다.
- OpenAPI 스키마 확인 — `VoiceChatV4AgentSessionRequest.voice` 가 10종 enum 으로, `model` 기본값이 `gpt-realtime-2.1` 로 나온다.
- Codex 리뷰 2회(`codex exec review --commit c5e754b`) — 두 번 다 **actionable regression 없음**. 단 두 번 다 `powershell.exe` 호출이 정책에 막혀 Codex 는 pytest·grep 을 못 돌렸고 파일 읽기만 했다.

## 깨진 테스트 하나

`test_v4_router_calls_manager_with_filter_track` 이 호출부를 정규식으로 핀하고 있어서, 주입 클래스가 변수가 되는 순간 깨졌다.

소스 정규식으로는 "어느 클래스가 실제로 갔는지" 를 표현할 수 없다. fake 를 세워 `create_answer` 가 받은 kwarg 를 직접 보는 방식으로 바꾸고, 미사용 모드까지 같이 검증하게 했다. 조각이 import 문·로그 문자열에 흩어져 있어도 통과하던 예전 검사보다 오히려 촘촘하다.
