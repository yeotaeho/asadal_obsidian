---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# 브라우저 이벤트 중계 복원

> [!summary] 한 줄 요약
> 2026-07-13 롤백이 지웠던 OpenAI→브라우저 DataChannel 중계를 되살리고, 같은 누락이 다시 생기지 않도록 회귀 테스트 11건을 붙였다. 프로젝트: [[AI음성]]

## 무슨 작업

v3는 브라우저가 OpenAI와 직접 붙어서 이벤트가 알아서 닿는다. 반면 v4는 서버가 중간에 끼므로, 서버가 넘겨주지 않으면 프론트에 아무것도 안 간다.

그 중계 코드가 통째로 빠져 있었다. 자막·바지인·토큰 사용량이 전부 죽는 상태였다. 오디오는 미디어 트랙으로 따로 흐르므로 통화 자체는 성립해서 **화면만 죽은 것처럼 보이는** 형태라 놓치기 쉬웠다.

복원한 것은 세 갈래다.

| 구성 | 하는 일 |
|---|---|
| `_browser_channels` | 브라우저가 만든 `"oai-events"` 채널을 세션별로 보관. 교체된 세대의 채널이 뒤늦게 열려 새 세대 항목을 덮어쓰지 않도록 세대를 확인한다 |
| `_relay_event_to_browser` | OpenAI 이벤트 원문을 가공 없이 전달. delta류는 로그만 건너뛰고 중계는 항상 한다 |
| `_extract_realtime_token_usage` | `response.done`이면 `token_usage`를 SSE 접두어 없이 순수 JSON으로 추가 전송 |

딕셔너리 공유를 생성자가 아니라 `attach_track`으로 넘긴 것이 이번 작업의 유일한 설계 판단이다.

```
매니저 --> v3/realtime_listener_bridge.py (파사드, kwargs 고정)
       --> _RealtimeListenerBridge.__init__   <-- 새 인자 못 넣음
매니저 --> attach_track(**kwargs)              <-- 이 경로로 우회
```

`v3/` 는 바이트 고정이라 파사드를 손댈 수 없다. `attach_track`은 `**kwargs`를 그대로 넘겨주므로 그 길을 썼다.

## 왜

롤백(`e70ab01`)이 이 코드를 지웠다. 그 뒤 v4 복원을 7개 태스크로 쪼개 진행했는데, 오디오 트랙·orchestrator·좀비 스위퍼·이벤트 루프는 챙기면서 **프론트와의 이벤트 계약만 목록에 없었다**.

롤백 이전 커밋(`48052bd`, `8d299df`, `abfa0f5`)에서 원본 구현을 찾아 그대로 되살렸다. 로그 수준만 다르게 뒀다 — 원본은 이벤트마다 warning을 찍어 운영 로그를 덮는다.

## 어디

| 커밋 | 내용 | 브랜치 |
|---|---|---|
| `65c199a` | 중계 복원 + 테스트 11건 | `fix/v4-session-generation-guard-dev` |
| `aa7cfcb` | 같은 내용 | `fix/v4-session-generation-guard` (main PR용) |

## 검증

`pytest ai_voice/voice_tests/` — **dev 브랜치 92 passed, main 기준 브랜치 86 passed** (차이는 dev에만 있는 v3 가드 6건).

변이 실측으로 테스트가 실제로 무언가를 지키는지 확인했다.

| 변이 | 결과 |
|---|---|
| 중계 호출 제거 | `test_openai_message_handler_actually_relays` 실패 |
| `attach_track` 공유 대입 제거 | `test_attach_track_shares_dict_by_reference` 실패 |
| delta 중계 차단 | `test_delta_events_are_still_relayed` 실패 |

첫 번째는 **처음에 안 잡혔다.** 나머지 테스트가 전부 `_relay_event_to_browser`를 직접 부르는 탓에 배선이 끊겨도 통과했다. OpenAI 메시지 핸들러를 실제로 구동하는 테스트를 따로 두고서야 잡혔다. 구현을 직접 부르는 테스트만으로는 "배선이 끊겼다"를 절대 못 잡는다는 게 이번의 교훈이다.

## Codex 리뷰

P1 하나가 올라왔다 — "브라우저 채널이 `connecting` 상태라 `session.created`를 버리고, 그래서 프론트가 멈춘다. 버퍼링해라."

**반려했다.** 두 전제가 모두 사실과 달랐다.

| 주장 | 실제 |
|---|---|
| 등록된 채널이 `connecting` | aiortc는 `_setReadyState("open")` **후에** `emit("datachannel")` 한다 (`rtcsctptransport.py:1790→1809`). `connecting`인 순간이 없다 |
| `session.created`를 놓치면 멈춘다 | `case 'session.created': break;` — no-op. 연결 상태는 프론트 자기 채널의 `onopen`에서 넘어간다 |

경합 자체는 실재한다. `on_track`이 `setRemoteDescription` 안에서 동기 발화하므로(aiortc `rtcpeerconnection.py:1054`) OpenAI 연결이 브라우저 채널 등록보다 먼저 끝날 수 있다. 다만 그 창에 떨어지는 건 프론트가 `break`로 흘리는 두 이벤트뿐이고, 실제로 처리되는 이벤트는 전부 사용자 발화 이후에 생긴다. 롤백 이전 원본도 같은 가드만 두고 버퍼는 없었다.

## 정정

리뷰를 검증하다가 **내 판단 오류를 찾았다.** "중계 누락이 음성이 아예 안 되는 근본 원인"이라고 했는데 과했다. 오디오는 미디어 트랙으로 흐르므로 통화 자체는 성립한다. 중계 누락으로 잃는 건 자막·바지인·토큰 표시다.

커밋 메시지와 코드 주석에 적었던 "프론트가 `session.created`를 기다리며 멈춘다"도 사실이 아니어서 함께 고쳤다. 근거가 틀린 주석은 다음 사람을 잘못 이끈다.

진짜 원인은 따로 있었고 같은 날 규명했다 — [[AI음성 - v4 무음 원인 규명]] 참고.
