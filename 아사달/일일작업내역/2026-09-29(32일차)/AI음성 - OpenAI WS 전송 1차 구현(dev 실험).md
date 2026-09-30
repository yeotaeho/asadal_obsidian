---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-29
---

# OpenAI WS 전송 1차 구현(dev 실험)

> [!summary] 한 줄 요약
> 플래그를 켠 세션만 서버↔OpenAI 구간을 WebRTC 대신 WebSocket 으로 붙이는 1차를 main·dev 브랜치로 올렸다. 전송 방식만 바꾸고 게이트·재생 제어·녹음·화자 판정은 같은 코드를 탄다. 실제 API 로 한 바퀴를 확인했다. 기능: [[OpenAI WS 전송]]

## 무슨 작업

WS 연결을 WebRTC 데이터 채널처럼 보이게 감싸는 `OpenAIWsChannel` 과, 받은 답변 소리를 20ms 프레임으로 실시간 박자에 맞춰 내는 `OpenAIWsAudioTrack` 을 새로 두었다. 브릿지는 플래그가 켜진 세션에서 이 둘을 WebRTC 연결 자리에 끼운다. 나머지 배선은 그대로다.

```
브라우저 ─WebRTC─ 서버 업링크 트랙(게이트·지터버퍼 그대로) ─ 채널이 20ms 마다 당김
                                  → input_audio_buffer.append ─WS─▶ OpenAI
OpenAI ─WS─▶ 이벤트 → 데이터 채널처럼 open·message·close (브라우저 중계·턴 처리 그대로)
          └ response.output_audio.delta → OpenAIWsAudioTrack(없으면 무음) → 다운링크 프록시 → 브라우저
합성 이벤트  output_audio_buffer.started / stopped / cleared  ← 우리가 실제로 내보낸 소리 기준
끼어들기     speech_started(interrupt_response) → 쌓인 답변 버림 + conversation.item.truncate
```

스위치는 오디오 탭·화자 판정과 같은 플래그 파일이다. `data/logs/voice_openai_transport` 에 `ws` 를 쓰면 새 세션부터 WS 로 붙고, 없거나 다른 값이면 WebRTC 다. 사투리 모드처럼 OpenAI 가 소리를 듣지 않는 세션은 제외했다.

## 왜

화자 판정 1·2단계(타인 발화에 대한 응답만 막기, TV 소리에 AI 말이 끊기지 않기)는 응답별 소리를 서버가 쥐어야 한다. WebRTC 는 답변 소리가 응답 구분 없이 RTP 한 줄로 오고 출력 버퍼가 OpenAI 쪽에 있다. dev 첫 통화에서 판정이 첫 답변 소리보다 0~230ms 늦게 나온 것도 계기였다.

1차는 전송 방식만 바꿨다. 업링크 내용·시점이 WebRTC 와 같아야 두 방식을 공정하게 비교할 수 있다. 지터버퍼 우회와 응답별 붙들기·버리기는 2차다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `8436d88` (dev `355dfbd`) | WS 채널·답변 트랙, 브릿지 분기, 테스트 9건 | `v4/openai_ws.py` · `server_webrtc.py` · `test_openai_ws.py` |

브랜치는 `feat-openai-ws`(main 기준)와 `feat-openai-ws-dev`(dev 기준 cherry-pick)이고 둘 다 푸시했다. dev 쪽은 `server_webrtc.py` 에서 충돌이 났다. import, 사투리 분기·화자 판정 생성, 사투리 메서드 자리를 dev 코드를 지우지 않는 쪽으로 풀고 커밋 메시지에 적었다. 같은 작업 트리를 토큰 저장 작업이 쓰고 있어서 별도 worktree(`AI_Pro_VOICE-ws`)에서 작업했다.

## 검증

- `pytest ai_voice/voice_tests` 는 main 기준 **429 passed**, dev 기준 **465 passed** 였다. `pytest tests` 는 8 passed.
- 새 테스트 9건: 답변 트랙의 20ms 프레임·무음·시작/끝 표시, 비우기와 늦은 조각 무시, 세션 설정 선전송과 답변 소리 비중계, 업링크 48kHz→24kHz, `output_audio_buffer.clear` 로컬 처리와 자르기, 끼어들기 켜짐/꺼짐, 세션 설정 모양, 플래그, 브릿지 분기.
- 실제 API 로 합성 질문(미리듣기 모듈로 만든 목소리)을 흘렸다. `session.update` 는 오류 없이 받아들여졌고, 말 감지·전사·응답·답변 재생이 WebRTC 와 같은 순서로 나왔다.
- 말 끝에서 첫 답변 소리까지 **0.71초**(dev WebRTC 실측 0.60~0.74초)였다. 답변 5.1초가 **0.7초 만에** 도착했고(실시간의 약 7배), `output_audio_buffer.stopped` 는 실제로 다 재생한 시점에 나왔다.
- AI Hub 음성은 국외 반출 금지 조건이라 이 시험에 쓰지 않았다.
