---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-07
---

# 세션 재사용 경합 수정

> [!summary] 한 줄 요약
> 같은 `session_id` 가 재사용될 때 이전 세션의 정리가 새 세션을 부수던 문제를 **세션 generation** 으로 막았다. 다만 리뷰가 지목한 메커니즘(예약된 콜백이 늦게 실행됨)은 **재현되지 않았고**, 실제로 열리는 창은 `close()` 가 **진행 도중** 새 세대를 가로지르는 것이었다. 프로젝트: [[AI음성]]

## 무슨 작업

[[AI음성 - v3·v4 orchestrator 분리]] 에서 범위 밖으로 미뤄뒀던 Codex Important 1 을 처리했다. 지목된 경로는 둘이었다.

1. ICE `failed`/`closed` 콜백이 `create_task(self.close(session_id))` 로 정리를 예약하면서, 캡처한 `pc` 가 현재 등록된 PC 인지 확인하지 않는다.
2. 데이터채널 `on_message` 가 `_orchestrator_for(session_id)` 로 **현재** 세션의 v3/v4 플래그를 읽는다.

고치기 전에 실제로 재현되는지부터 확인했다. 실제 `aiortc` 로 매니저를 끝까지 구동했다.

## 무엇이 드러났나

**리뷰가 서술한 메커니즘은 재현되지 않는다.** 예약된 stale close 가 `_pcs[session_id] = pc` 이후에 실행된다는 것이 지적의 전제인데, 그런 일은 일어나지 않는다. `create_answer` 자신의 `await self.close(session_id)` 가 등록보다 **먼저** yield 하고, 이벤트 루프가 그 지점에서 예약된 close 를 소진해버린다. 그때 `_pcs` 는 이미 비어 있어 무해한 no-op 이다. 순서를 찍어 확인했다.

```
01 --- 1st create_answer ---
02 _pcs[s] = pc#1   <-- REGISTER
03 --- 2nd create_answer (reuse) ---
04 close() ENTER   registered=pc#1
05 _pcs.pop(s) -> pc#1
06 close() ENTER   registered=None     <-- stale close 가 여기서 소진된다
07 close() EXIT
08 close() EXIT
09 _pcs[s] = pc#2   <-- REGISTER (이미 늦었다)
```

**진짜 창은 `close()` 가 "진행 중" 일 때다.** `close()` 는 `_pcs` 를 먼저 뽑아낸 뒤, 코드 스스로 상한을 둘 만큼 오래 걸릴 수 있는 구간을 지난다 — `replaceTrack` 은 `wait_for(timeout=2.0)`, `detach` 는 `await pc.close()`. 그 사이 `create_answer` 는 `_pcs` 에서 id 를 못 보므로 **자기 선행 정리를 건너뛰고** 완전히 살아있는 새 세션을 등록한다. 뒤늦게 깨어난 이전 세대의 close 가 `session_id` 만 보고 정리를 마저 진행하면 새 세션이 무너진다.

측정한 피해다.

| 대상 | 결과 |
|---|---|
| `_session_metadata` | 새 세대 것이 `None` 으로 지워짐 |
| `_session_use_v4_orchestrator` | 지워짐 → 그 세션이 **조용히 v3 orchestrator 로 떨어진다** |
| 브릿지 | 살아있는 새 세대가 detach 됨 |
| runtime | 살아있는 새 세대가 release 됨 |
| PeerConnection | `_pcs` 에는 남아 좀비가 됨 |

v4 플래그가 지워지는 것이 곧 지적 2번의 피해다. **두 경로는 별개가 아니라 같은 뿌리**였다. 지적 2번을 그 자체 방향(이전 세대의 늦은 이벤트가 새 세대 플래그로 라우팅)으로도 재현했다 — gen1 의 detach 가 진행 중일 때 gen1(v4) 이벤트가 v3 orchestrator 를 탔다.

## 왜

`session_id` 하나로는 "지금 살아있는 세션" 을 가리키지 못하는데, 정리 코드 전체가 `session_id` 만 보고 dict 을 건드리고 있었다. 진입 시점에 세대를 확인하는 것만으로는 부족하다 — stale close 는 그 검사를 **통과한 뒤에** 멈추기 때문이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `88ac1e0` | 세션 재사용 경합 — 이전 세대 콜백이 새 세션을 부수는 문제 | `server_webrtc.py`, `voice_tests/` |
| `a609a47` | 좀비 세션 스위퍼도 세대 스냅샷을 쓰도록 수정 | `server_webrtc.py`, `test_session_generation_guard.py` |

브랜치 `fix/v4-session-generation-guard` (`feat/v4-isolated-restore` 기준 2커밋). 지적 2번과 테스트 자산이 main 에 없어 v4 브랜치 위에 쌓았다.

수정의 뼈대는 넷이다.

- `create_answer` 가 세션마다 전역 단조 증가 generation 을 발급하고, ICE 콜백이 자기 세대를 함께 넘긴다.
- `close()` 가 **첫 `await` 전에** 이 세대의 자원을 동기적으로 전부 떼어낸 뒤 그 지역 변수만으로 teardown 한다. `await` 뒤에 dict 을 다시 읽지 않으므로 새 세대를 건드릴 수 없다.
- 브릿지도 같은 generation 을 받아 교체된 세대의 `detach` / 채널 이벤트(`open`·`message`·`close`)를 무시한다.
- 세대를 `_connect_realtime` 시작 시점에 **고정**한다. OpenAI 가 나중에 여는 역방향 데이터채널이 그때 다시 조회하면 새 세대 번호를 집어와 가드가 통째로 무력해진다.

스위퍼도 같은 결함이었다. `list(self._pcs.items())` 스냅샷을 돌며 세션마다 `await close()` 로 멈추는데, 앞 항목을 정리하는 사이 뒤 항목이 교체되면 스냅샷의 죽은 PC 를 근거로 판단해놓고 방금 등록된 새 세션을 닫는다. 세대를 스냅샷과 같은 컴프리헨션에서 원자적으로 함께 읽게 고쳤다.

`ai_voice/services/voice_chat/v3/` 와 v3 라우터는 건드리지 않았다.

## 검증

`pytest ai_voice/voice_tests/` **53 → 60 passed**. venv 는 `fastapi==0.115.9` / `starlette==0.45.3` 핀 유지.

소스 문자열 검사가 아니라 **실제로 콜백을 늦게 실행시켜** 검증한다. `replaceTrack` 과 `pc.close()` 를 `asyncio.Event` 로 멈춰 세대 교차 구간을 만들고, 그 안에서 새 세션을 등록한다.

변이 테스트 **5건**으로 각 가드가 시그니처가 아니라 동작을 잡는지 실측했다. `on_message` 가드 / `detach` 가드 / `close()` 의 claim 순서 / 역방향 채널 세대 고정 / 스위퍼 스냅샷 — 각각 대응 테스트가 정확히 하나씩 실패한다.

재현 스크립트로도 교차 확인했다. 수정 전에는 `metadata=None`·`use_v4=None`·detach 2회·release 2회, 수정 후에는 `metadata={'gen': 2}`·`use_v4=True`·detach 1회·release 0회.

> [!note] 동작 변화 하나
> `runtime.release_session` 은 `session_id` 하나로만 지정된다. 정리 도중 새 세대가 등록됐다면 건너뛴다 — 안 그러면 살아있는 세션의 대화 상태를 끊는다. 그래서 **재사용된 세션은 이전 세대의 LangGraph 대화 상태를 물려받는다**. 진짜 고아가 되면 `_reap_orphan_runtime_sessions` 가 회수한다.

## Codex 리뷰

Critical 0건. 지적 6건은 검토만 하고 **반영하지 않기로 했다**(사용자 판단). 각각 코드로 확인한 결과다.

| 지적 | 판정 |
|---|---|
| [Bug] 협상 실패 경로 `close()` 에 generation 미전달 (`:400`) | **타당.** `generation` 이 그 자리에 스코프로 있다. 한 줄이면 된다 |
| [Bug] `create_answer` 동시 호출 미직렬화 (`:280`) | **타당.** 세션별 `asyncio.Lock` 이 정답. 1차에서 범위 밖으로 뺀 그 지점이다 |
| [Risk] `release_session` 앞 세대 검사가 불충분 (`:483`) | **타당하나 서술보다 좁다.** `release_session` 은 `async with self._lock` 뒤 동기 `pop` 이라, 비경합 시 `acquire()` 는 yield 하지 않는다. 창은 **락 경합 시에만** 열린다. 완전히 닫으려면 `VoiceChatGraphRuntime` 이 generation 을 알아야 한다 |
| [Risk] orphan reaping 도 같은 패턴 (`:574`) | **타당하나 pre-existing.** 이번 두 커밋이 건드리지 않은 코드다 |
| [Nit] 테스트가 동시 `create_answer` 를 다루지 않음 | 맞다 |
| [Dead Code] 없음 | — |

후속 과제로 남긴다.
