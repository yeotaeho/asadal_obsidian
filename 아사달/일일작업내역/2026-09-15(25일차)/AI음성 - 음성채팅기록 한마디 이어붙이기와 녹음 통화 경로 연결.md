---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-15
---

# 음성채팅기록 한마디 이어붙이기와 녹음 통화 경로 연결

> [!summary] 한 줄 요약
> 도구 호출 전 한마디를 같은 질문의 답변 앞에 붙이고(개선 1), 녹음기를 통화 경로에 연결해 **질문 + 한마디 + 답변이 `_qa.wav` 한 파일**로 남게 했다. 테스트 **411 passed**, dev 사본 커밋까지(푸시 전). 기능: [[음성채팅기록]]

## 무슨 작업

사용자가 dev 머지·dev·test 계정 컬럼 추가·프론트 배포·서버 compose 볼륨과 `VOICE_RECORD_SECRET` 반영을 마친 뒤 두 가지를 진행했다. 1단계 브랜치에는 개선 1을, 녹음 브랜치에는 통화 경로 연결을 얹었다.

**개선 1.** 도구를 부른 응답의 말을 버리지 않고, 그 응답이 대상으로 삼은 사용자 발화 `item_id` 와 함께 보관한다. 다음 일반 답변의 대상 발화가 같으면 앞에 붙이고 토큰도 합친다. 연속 도구 호출은 순서대로 모두 붙는다.

**녹음 연결.** `feat/voice-record` 를 `feat/voice-chat-log` 위로 리베이스하고 브릿지에 훅을 걸었다.

```
업링크 트랙 recv ─ on_frame ─┐
다운링크 트랙 recv ─ on_frame ┤ (일시정지 침묵은 안 넘김)
speech_started/stopped ───────┤ item_id + audio_start_ms/audio_end_ms
_turn_log_user_text ──────────┤ set_turn (전사 도착 시점의 턴)
output_audio_buffer.started ──┤ 재생 창 열림
_send_after_drain ────────────┤ 재생 창 닫힘 (서버 버퍼가 빈 뒤)
_stop_current_speech ─────────┤ end_answer (멈춤·종료말)
질문 폐기 / _drop_turn_log ───┘ abandon / close
                              → PairRecorder
TurnLogger 짝 맺음 → _send_turn_record → resolve(q_item_id) → voice_chat.turn 에 q:{path,t}
```

마지막으로 사용자 허락을 받아 `.env.example` 에 `VOICE_RECORD_SECRET`·`VOICE_RECORD`, `docs/DEPLOY.md` 에 compose 볼륨과 녹음 절(비밀키 불변·루트 미생성·반영 확인 명령)을 넣었다.

## 왜

녹음은 사용자가 들은 소리라 한마디가 들어간다. 글에서만 빠지면 목록에서 글과 소리가 어긋나므로 개선 1을 녹음 연결보다 먼저 넣었다.

개선안 초안은 "사용자가 다시 말하면 머리말을 비운다" 였다. 그러면 끼어든 뒤 **취소된 한마디 응답의 `response.done` 이 늦게 와서** 빈 머리말에 새로 들어가고, 다음 질문의 답변 앞에 붙는다. 대상 발화로 묶어 이 경로를 테스트로 막았다.

**연결하다 녹음기 결함을 찾았다.** 녹음기는 발화 시작 때의 턴 id 로 같은 턴 질문을 합쳤는데, 턴 id 는 발화마다 새로 발급되고 기록기는 전사 도착 시점의 턴으로 합친다. 시작 시점 턴으로는 합칠 일이 없었다. `set_turn` 으로 바꾸고, 합치기를 짝 맺는 item 앞의 아직 짝 없는 질문까지로 좁혔다(처음 구현은 뒤 발화까지 흡수해 테스트가 잡았다).

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `b9f5139` | 한마디를 같은 발화의 답변 앞에 이어 붙이고 토큰 합산 | `server_webrtc.py` · `test_turn_log_end_to_end.py` |
| `e738725` | (리베이스, 구 `53a701f`) 저장소·재생·삭제 엔드포인트 | `records.py` · `voice_chat_v4_router.py` |
| `b17d685` | (리베이스, 구 `a236d5f`) 녹음기 `PairRecorder` | `v4/pair_recorder.py` |
| `03a818b` | 통화 경로 연결, `set_turn`·`end_answer`·프레임 입력, 통합 테스트 | `server_webrtc.py` · `v4/audio_tracks.py` · `v4/pair_recorder.py` · `test_pair_recording_wiring.py` |
| `7ee975b` | `.env.example`·`docs/DEPLOY.md` 녹음 설정 | `.env.example` · `docs/DEPLOY.md` |

dev 사본 `feat/voice-chat-log-dev` 는 `origin/dev`(`94be736`)로 fast-forward 후 `378987c`·`0fda341`·`4c5267e`·`2ef3d20`·`ac38b1b` 로 cherry-pick 했다. 푸시하지 않았다.

## 검증

- `pytest ai_voice/voice_tests` **411 passed**(이전 365), `pytest tests` 8 passed. dev 사본 브랜치에서도 411 passed.
- 통합 테스트 5건: 도구 턴 한 파일(질문 + 한마디 + 답변, RAG 대기 무음 제외, `q` 가 쓴 경로와 일치), 도구 조회 중 끼어들기(새 질문만 기록), 멈춤 버튼(누른 데까지), 세션 종료(들은 데까지), 저장소 꺼짐(`q` 없음).
- 녹음기 단위 테스트 추가: 프레임 입력(s16·스테레오), 예상 밖 표본 형식이면 녹음만 끔, 질문 길이 상한, 대기 쌍 상한, 같은 턴 합치기 범위.
- Codex `b9f5139` 지적 없음. `03a818b` P2 1건 "파일이 써지기 전에 URL 통지" 는 설계 §5.5 에서 받아들인 틈이라 미반영([[AI음성 - 결정 기록]]).
