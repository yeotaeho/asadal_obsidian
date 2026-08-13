---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# v4 무음 원인 규명

> [!summary] 한 줄 요약
> v4가 연결은 되는데 오디오도 이벤트도 안 오던 원인이 `_connect_realtime`의 `session_config` 누락임을 aiortc 실측으로 규명하고 한 줄로 고쳤다. 프로젝트: [[AI음성]]

## 무슨 작업

dev에 v4 라우트가 떠 있는데도 데스크톱·모바일 모두 통화가 안 됐다. 코드 읽기로는 결론이 안 나서 **aiortc로 브라우저와 똑같은 절차를 밟는 재현 클라이언트**를 만들어 dev에 직접 붙였다.

```
RTCPeerConnection 생성 --> 오디오 트랙 추가 --> createDataChannel("oai-events")
  --> createOffer / setLocalDescription
  --> POST /AI_voice/voice-chat-vN/agent/session
  --> setRemoteDescription --> 25~40초 관측
```

## 측정 결과

| 대상 | 세션 | ICE | DataChannel | 이벤트 | 오디오 |
|---|---|---|---|---|---|
| v4 + 기본 모델 | 200 | completed | open | 0건 | **0프레임** |
| v4 + `gpt-realtime-2` | 200 | completed | open | 0건 | **0프레임** |
| v3 + 기본 모델 | **400** | — | — | — | — |
| v3 + `gpt-realtime-2` | 200 | completed | open | `session.created` · `session.updated` | **수신됨** |

마지막 줄이 대조군이다. 같은 OpenAI, 같은 모델, 같은 클라이언트인데 **v4만 조용히 죽는다**.

## 원인 — `session_config` 누락

`_connect_realtime`이 `create_webrtc_session`을 부를 때 `session_config`를 안 넘겼다. GA 모델이면 GA 경로가 `{type, model, tools}`만 든 최소 세션을 만들고 스키마 검증에서 걸린다.

```
Invalid GA Realtime session shape: 3 validation errors
  output_modalities  Field required
  audio              Field required
  instructions       Field required
```

그 `ValueError`를 `_create_ga_session`의 `except`가 500으로 바꿔 던지고, `_connect_realtime`의 `try/except`가 삼킨 뒤 `pc.close()`하고 조용히 반환한다.

증상이 고약한 이유는 **`create_answer`가 OpenAI 연결을 기다리지 않고 answer를 즉시 반환**하기 때문이다. 브라우저는 HTTP 200을 받고 ICE도 completed까지 가고 DataChannel도 열린다. 겉보기엔 완전히 정상인데 아무 일도 일어나지 않는다.

고친 것은 한 줄이다. 라우터가 이미 `_build_session_ga`로 만들어 `metadata["session_config"]`에 담아둔 값을 꺼내 쓴다. 새로 만들지 않았다.

```python
session_config = (self._session_metadata.get(session_id) or {}).get("session_config")
```

v3가 멀쩡했던 이유도 이걸로 설명된다 — v3 라우터는 처음부터 `session_config`를 넘기고 있었다.

## 곁다리 발견 — Beta API 폐지

v3에 비-GA 모델로 요청했을 때 OpenAI가 돌려준 원문이다.

```
"The Realtime Beta API is no longer supported. Please use /v1/realtime for the GA API."
"code": "beta_api_shape_disabled"
```

`webrtc_api.py`는 GA 목록(`gpt-realtime-2`, `gpt-realtime-2.0`)에 없는 모델을 전부 legacy=Beta 경로로 보낸다. 그런데 서버 기본값 `OPENAI_REALTIME_MODEL_V2`(`gpt-realtime-1.5`)와 `OPENAI_DEFAULT_MODEL`(`gpt-4o-realtime-preview-2024-12-17`)이 그 목록 밖이다.

**지금 당장 아무것도 깨뜨리지 않는다.** 프론트가 `voice_runtime.js`에서 항상 `gpt-realtime-2`를 보내고, 없으면 `[VOICE_CFG_MISSING_MODEL]`로 아예 멈추기 때문이다. 어제 v3로 모바일 통화가 됐던 것이 그 증거다.

문제는 밟을 수 있는 길이 남아 있다는 것이다. 프론트에 특정 에러 시 `gpt-realtime-1.5`로 내려가는 폴백이 있는데(`webrtc_voice_module.js:1142`), 그 폴백은 이제 **죽은 API로 가는 길**이다. 별도 과제로 둔다.

## 어디

| 커밋 | 내용 | 브랜치 |
|---|---|---|
| `fb1f4b2` | GA 세션 설정 전달 + 테스트 4건 | `fix/v4-session-generation-guard-dev` |
| `a493464` | 같은 내용 | `fix/v4-session-generation-guard` (main PR용) |

핵심 파일은 `ai_voice/services/voice_chat/server_webrtc.py`의 `_connect_realtime`이다.

## 검증

`pytest ai_voice/voice_tests/` — **dev 브랜치 96 passed, main 기준 브랜치 90 passed**.

테스트 4건 추가. 최소 세션이 스키마에 거부된다는 사실 자체도 함께 못박았다 — 그 검증이 언젠가 느슨해지면 전달 테스트의 의미가 사라지기 때문이다. `close()`가 `_session_metadata`를 먼저 비운 뒤 이 코루틴이 늦게 도달하는 경우도 죽지 않는지 확인한다.

변이 실측 — `session_config`를 `None`으로 되돌리면 해당 테스트 하나가 실패한다.

Codex 리뷰는 지적 없이 통과했다.

## 정정

앞서 "브라우저 이벤트 중계 누락이 근본 원인"이라고 판단했던 것은 틀렸다([[AI음성 - 브라우저 이벤트 중계 복원]]). 중계는 자막·바지인·토큰 표시용이고, 통화가 통째로 죽는 원인은 이 건이다. 중계 복원 자체는 여전히 필요하지만 이걸 고치기 전에는 **중계할 이벤트가 애초에 없었다**.

코드만 읽고 내린 결론이 두 번 빗나갔고, 실제로 붙여보고 나서야 잡혔다. 재현 클라이언트를 먼저 만들었으면 훨씬 빨랐다.

## 추후 작업

- [ ] dev 배포 후 재현 클라이언트를 다시 돌려 오디오 프레임이 실제로 잡히는지 확인
- [ ] Beta API 폐지 대응 — 서버 기본값 정리 + 프론트 폴백 제거 (별도 과제)
