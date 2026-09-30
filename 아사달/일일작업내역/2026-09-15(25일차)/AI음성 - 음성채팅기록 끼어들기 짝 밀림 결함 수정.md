---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-15
---

# 음성채팅기록 끼어들기 짝 밀림 결함 수정

> [!summary] 한 줄 요약
> 답이 영영 오지 않는 질문이 대기열에 남아 이후 기록의 짝이 통화 끝까지 한 칸씩 밀리던 결함을, **답이 사라지는 순간 그 질문을 `item_id` 로 지목해 버리는** 방식으로 고쳤다. Codex 지적 두 번을 반영해 **363 passed**, 재리뷰 지적 없음. 기능: [[음성채팅기록]]

## 무슨 작업

처음 계획은 "끼어들 때 이전 턴 질문을 `turn_id` 로 버리기" 였다. 코드를 읽으니 답이 사라지는 경로가 셋이었다.

| 경로 | 예 |
|---|---|
| 도구 태스크 실제 취소 | RAG 조회 중 끼어들기·멈춤 |
| 대기 중 도구 재개 비움 | 한마디 응답이 끝나기 전에 RAG 가 끝나 재개가 대기열에 있을 때 끼어들기 |
| 전사 없는 응답 종료 | 응답 시작 직후 끼어들기, 실패. **도구와 무관** |

`turn_id` 는 늦게 온 전사가 새 턴 값을 받아 거르지 못한다. 그래서 응답이 시작될 때 직전 사용자 발화(`speech_stopped` 의 `item_id`)를 그 응답의 대상으로 적어 두고, 위 세 곳에서 `TurnLogger.on_answer_abandoned(item_id)` 로 그 질문만 지목하게 했다.

```
speech_stopped(item)            last_user
response.created                response = last_user, tool 표시 해제
response.done (function_call)   tool = response
답이 사라짐                       on_answer_abandoned(tool 또는 response)
  → 대기 중 질문이면 삭제 + 폐기 표시(set) / 아직이면 폐기 표시만
on_user_text                    폐기 표시된 item 의 전사는 받지 않음
```

도착 순서 대기열에 "빈 답변 자리" 를 넣는 안도 검토했다. 전사가 끝내 오지 않는 잡음 발화면 빈 자리가 남아 오히려 밀림을 만들어 기각했다.

## 왜

GA 세션의 답변은 도구 후속 답변까지 `turn_id` 가 없다. 오케스트레이터가 넘기는 값을 읽는 시점에는 이미 비어 있다. 그래서 짝이 사실상 도착 순서로 맺어진다. 테스트 도우미가 `turn_id` 를 직접 넣어 이 점을 가리고 있었다.

```
기대 [(질문2, 답변2), (질문3, 답변3)]
수정 전 [(질문1, 답변2), (질문2, 답변3)]
```

2단계 녹음은 질문 `item_id` 로 붙으므로, 고치지 않았다면 "질문1 + 한마디" 녹음이 답변2 행에 붙었을 것이다.

**함께 확인한 별도 결함(미수정).** 상태가 `idle` 일 때 질문 전사가 도착하면 `_handle_speech_started` 가 불려 **같은 턴의 RAG 를 취소하고 재생 중인 한마디를 끊는다.** 임시 테스트로 확인했다. 도구 턴 한마디 응답의 done 이 `idle` 로 돌린 뒤 전사가 늦으면 생긴다. 기록은 이번 수정으로 보호되지만 답이 사라지는 체감 결함이라, dev 로그로 빈도를 보고 결정한다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `57139ec` | 답이 사라진 질문을 `item_id` 로 지목해 폐기, `cancel_tool_tasks` bool 반환, `_is_announcement` 공용화 | `server_webrtc.py`, `v4/turn_log.py`, `v4/tool_call_orchestrator.py`, 테스트 4개 |
| `4541a8a` | 대기 중에 지운 질문도 폐기 표시 유지 (Codex P2 — 재전송 전사로 되살아남) | `v4/turn_log.py`, `test_turn_log.py` |
| `023c83a` | 폐기 표시를 상한 없는 `set` 으로 (Codex 재리뷰 P2 — 밀려난 id 의 늦은 전사) | `v4/turn_log.py`, `test_turn_log.py` |

브랜치 `feat/voice-chat-log`. 설계 문서는 `docs/superpowers/specs/2026-09-15-voice-chat-log-tool-turn-improvements.md` §3.4·§4.

## 검증

- `pytest ai_voice/voice_tests` **351 → 363 passed**, `pytest tests` 8 passed, 경고 18건 동일
- 새 테스트: TurnLogger 단위 6(지목 폐기, 늦은 도착, 전사 안 오는 발화가 남 안 건드림, id 없음, 재전송, 폐기 6개 누적), 브릿지 e2e 5(도구 조회 중 끼어들기, 도구 호출 응답 진행 중 끼어들기, 전사 없는 취소, 늦게 도착한 전사, 정상 도구 턴), orchestrator 반환값 1
- 작업 트리 CRLF 보존 확인 — 저장소는 LF, `core.autocrlf=true`, diff 는 의도한 줄만
- Codex: `57139ec` P2 → `4541a8a`, 범위 재리뷰 P2 → `023c83a`, 범위 재리뷰 지적 없음
- dev 통화 검증은 남음
