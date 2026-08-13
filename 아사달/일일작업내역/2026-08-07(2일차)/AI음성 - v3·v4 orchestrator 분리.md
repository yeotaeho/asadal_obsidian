---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-07
---

# v3·v4 orchestrator 분리

> [!summary] 한 줄 요약
> v4 세션이 공용 `server_webrtc` 를 거쳐 v3 코드를 타고 있던 구조를 세션별 분기로 바꿔, v4 를 고치려고 v3 를 건드릴 일이 없어졌다. 그 과정에서 v4 사본에만 있던 **세션 행(hang) 유발 결함 2건**이 새로 드러나 함께 고쳤다. 프로젝트: [[AI음성]]

## 무슨 작업

사용자가 "왜 v4 복원 브랜치가 `v3/tool_call_orchestrator.py` 와 `v3/audio_tracks.py` 를 수정했나" 를 물어 추적한 데서 시작했다. 두 파일은 서로 다른 경우였다.

`tool_call_orchestrator` 는 v4 가 실제로 그 코드를 탄다. v4 라우터는 자체 `v4/tool_call_orchestrator.py` 를 갖고 있으면서도 **아무도 그걸 import 하지 않았고**, 실제 v4 세션은 공용 `server_webrtc` 를 거쳐 v3 것을 탔다.

```
v4 라우터 → server_webrtc_manager.create_answer()
          → server_webrtc.py:72  from ...v3.tool_call_orchestrator import ...
          → :970  self._tool_orchestrator = RealtimeToolCallOrchestrator(...)
          → :2065 extract_function_tool_call() → schedule_function_tool_call()
```

`audio_tracks` 는 정반대였다. v4 는 `v4/audio_tracks.py` 의 독립 사본을 쓰고, 두 `AudioProxyTrack` 은 **상속 관계가 없다**(실측 확인). 그래서 v3 에 넣은 좀비 스위퍼는 v4 에 아무 효과가 없는 순수 v3 수정이었다.

`server_webrtc` 가 세션이 어느 경로로 만들어졌는지 기억했다가 orchestrator 를 골라주게 바꾸고, v3 두 파일은 `origin/main` 으로 원복했다. 플래그는 매니저와 브릿지가 세션 dict 을 공유하지 않아 네 단계로 전달한다.

```
create_answer(use_v4_orchestrator=True)
  → ServerWebRTCSessionManager._session_use_v4_orchestrator
  → attach_track(use_v4_orchestrator=...)
  → _RealtimeListenerBridge._session_use_v4_orchestrator
  → _orchestrator_for(session_id)
```

## 무엇이 드러났나

두 사본이 서로 다른 방향으로 갈라져 있어 단순 교체가 아니라 병합이었다. 이식하는 과정에서 **v4 사본에만 있던 결함 2건**이 새로 나왔다.

| 파일 | 결함 | 영향 |
|---|---|---|
| `v4/tool_call_orchestrator.py` | `results` 가 `try` 안에서만 바인딩되는데 함수 끝에서 참조 → 타임아웃·예외 경로에서 `UnboundLocalError` | 태스크로 돌아 예외가 삼켜짐 → tool 응답이 영영 안 감 → **세션 행** |
| `v4/audio_tracks.py` | 좀비 스위퍼가 `self.stop()` 을 안 부름 | 상태만 `ended` 로 바뀌고 리더 태스크·업스트림 소스는 계속 살아있음 → **스위퍼가 아예 동작 안 함** |

두 번째는 v3 대상이던 테스트를 v4 로 돌리자마자 드러났다. 테스트를 옳은 대상에 걸면 결함이 스스로 나온다는 걸 보여준 사례다.

이식한 것은 Codex 가 Critical 로 잡았던 chat_id fall-through 가드와 `summarize_for_voice` 타임아웃 예산 2건이다.

## 재검토에서 더 나온 것

공용 파일 3개가 v3 동작도 바꾸지만, 이건 "v4 코드가 v3 에 샌 것" 과는 다른 범주라 그대로 뒀다. `server_webrtc.py`(ICE 픽스·이벤트 루프 픽스·SDP 실패 정리), `realtime_speaker.py`(GA 모델 분기 — v3 기본 모델 `gpt-realtime-1.5` 는 안 걸림), `runtime.py`(순수 추가).

같은 유형 3건은 고쳤다. v4 라우터·orchestrator 가 자기 사본을 두고도 v3 의 `voice_context_summarizer` / `tool_result_policy` 를 import 하던 것을 v4 사본으로 돌렸다. 이제 **v4 소스에 `voice_chat.v3` 참조가 하나도 없다**. `_build_proxy_marker_snapshot` 이 v3 클래스로만 `isinstance` 검사해 v4 세션은 항상 `proxy=no-track` 으로 찍히던 것도 고쳤다. 참조 없는 v4 사본 4개(**270줄**)는 삭제했다.

dev 머지 시 터질 문제 하나를 미리 잡았다. 확장한 v3 바이트 동일성 가드가 v3 라우터까지 보고 있었는데, 이건 변경 출처를 못 가린다. v3 브랜치의 라우터를 얹어 실측하니 실제로 실패했다. 라우터는 성질 기준 가드 둘이 이미 덮고 있어 바이트 동일성은 v3 서비스 패키지에만 남겼다.

## 왜

과거 v4 롤백(2026-07-13)의 원인이 v4 가 v3 경로에 배선돼 들어간 것이었다. 이번 복원은 그걸 피하려 했는데, 정작 orchestrator 배선이 그대로 남아 있어서 v4 를 고치는 변경이 운영 중인 v3 동작까지 바꾸는 상태였다. 같은 실수가 다른 층위에서 반복된 셈이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `a196e0b` | v4 tool orchestrator 를 v3 에서 분리하고 v4 자체 사본에 배선 | `server_webrtc.py`, `v4/`, `v3/` 원복 |
| `006361b` | v4 패키지 자립 + 죽은 사본 4개 제거 | `v4/`, `voice_chat_v4_router.py` |
| `83ac9c9` | v3 바이트 동일성 가드에서 라우터 제외 | `test_v4_integration_smoke.py` |
| `a64b00a` | orchestrator 분기를 실제 호출로 검증 | `test_orchestrator_isolation.py` |

브랜치 `feat/v4-isolated-restore` 는 main 기준 **25커밋**이다.

## 검증

`pytest ai_voice/voice_tests/` **53 passed**. 두 브랜치가 합쳐진 상태를 흉내낸 뒤에도 통과.

변이 테스트 4건으로 가드 실효성을 실측했다. 라우터 kwarg 제거 / 매니저 전달 제거 / **브릿지 저장 제거** / detach 정리 제거 각각에 대해 대응 가드가 정확히 하나씩 실패한다.

`ai_voice/services/voice_chat/v3/` 는 `origin/main` 과 바이트 동일하다.

## Codex 리뷰

Critical 0건.

**Important 2 는 실제 구멍이라 고쳤다.** 기존 격리 가드가 전부 소스 인스펙션이라 `attach_track` 안의 저장문 한 줄만 지워도 52건이 그대로 통과했다. 그 상태에서 브릿지 dict 은 항상 비어 모든 v4 세션이 조용히 v3 로 떨어진다 — 이 리팩터가 막으려던 실패 그 자체다. 실제로 `attach_track` 을 태워 확인하는 행동 테스트를 추가했다. 내 앞선 변이 테스트 3건은 변이가 두 줄을 동시에 지워 다른 가드가 먼저 걸리는 바람에 이 경로를 못 짚었다.

**Important 1(세션 재사용 경합)은 범위 밖으로 판단**해 별도 작업으로 등록했다. ICE 콜백의 PC identity 미검사는 `origin/main` 부터 있던 문제이고, 채널 이벤트 라우팅은 내 변경이 한 겹 얹은 게 맞지만 근본 문제는 pre-existing 이다. v4 라우터는 매번 새 UUID 를 쓰므로 재사용은 방어 코드 경로다. 제대로 고치려면 세션 generation 카운터로 세션 수명주기 전반을 손봐야 한다.

Minor(v3 행동 커버리지 감소)는 의도한 것이다. v3 가 `origin/main` 과 고정돼 있어 여기서 검증하면 main 코드를 테스트하는 셈이다.
