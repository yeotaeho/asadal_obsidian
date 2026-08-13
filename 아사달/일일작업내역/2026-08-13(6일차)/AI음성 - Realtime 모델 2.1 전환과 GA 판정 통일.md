---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-13
---

# Realtime 모델 2.1 전환과 GA 판정 통일

> [!summary] 한 줄 요약
> 8/12 회의 결정(Realtime 모델 2.1 전환)을 백엔드에 반영했다. 그 과정에서 **모델만 바꾸면 조용히 죽는 잠복 버그**를 발견해 GA 판정을 프론트와 같은 접두사 규칙으로 통일했다. 프로젝트: [[AI음성]]

## 무슨 작업

회의에서 `gpt-realtime-2.1` 전환이 결정돼 소스를 검토했는데, **상수만 바꾸면 통화가 안 되는 상태**였다.

```
프론트:  'gpt-realtime-2.1'.startsWith('gpt-realtime-2')          → true  → GA
백엔드:  'gpt-realtime-2.1' in {gpt-realtime-2, gpt-realtime-2.0} → False → Legacy
```

Legacy 경로(`POST /v1/realtime?model=...`)는 OpenAI가 **2026-05-12로 폐지한 Beta API shape**이라 요청 자체가 실패한다. 프론트는 자기가 GA라고 믿고 있으니 로그만 봐서는 원인을 찾기 어렵다.

판정을 프론트와 같은 접두사 규칙으로 통일하고 호출부 **8곳**을 교체했다. `OPENAI_REALTIME_GA_MODELS` 집합은 제거했다.

```python
OPENAI_REALTIME_MODEL_GA = "gpt-realtime-2.1"
OPENAI_REALTIME_GA_MODEL_PREFIX = "gpt-realtime-2"

def is_ga_realtime_model(model: str) -> bool:
    return (model or "").strip().startswith(OPENAI_REALTIME_GA_MODEL_PREFIX)
```

죽은 기본값도 정리했다. `OPENAI_REALTIME_MODEL_V1`(`gpt-4o-realtime-preview-2024-12-17`)은 **2026-05-07 폐지 모델**이고 별칭 `OPENAI_DEFAULT_MODEL` 하나에만 쓰이고 있어 제거하고 기본값을 살아 있는 GA 모델로 돌렸다. `OPENAI_REALTIME_MODEL_V2`(`gpt-realtime-1.5`)는 Legacy 평면 설정의 정체성이라 남기고, 경로가 폐지됐다는 사실만 주석으로 명시했다.

## 왜

집합 방식이 근본 원인이다. 새 버전이 나올 때마다 판정에서 빠져 폐지된 경로로 조용히 떨어지는데, **프론트는 접두사라 자동으로 통과**하므로 두 시스템이 어긋난 채로 배포된다. 기준을 프론트에 맞추면 다음 버전(2.2)에서도 같은 사고가 나지 않는다.

## 전환 전 형식 호환 확인

모델을 바꾸기 전에 **2.1이 현재 v4 세션 설정을 그대로 받는지 OpenAI에 직접 물었다.** 4일차에 겪은 "연결은 되는데 아무 일도 안 일어남"(`session_config` 누락)과 같은 실패 모드를 배제하기 위해서다.

운영 코드의 `WEBRTC_VOICE_CONVERSATION_CONFIG_GA`와 `validate_ga_session_shape`를 파일 경로로 직접 로드해 사본이 아닌 실제 설정을 썼다. v4 전용 `create_response=False` 덮어쓰기와 `search_knowledge_base` tool까지 포함했다.

| 모델 | 결과 |
|---|---|
| `gpt-realtime-2` (대조군) | **HTTP 201** — SDP answer 1,541 bytes |
| `gpt-realtime-2.1` | **HTTP 201** — SDP answer 1,541 bytes |

대조군을 함께 돌려 키·네트워크·요청 형식 자체가 정상임을 고정했다. 검증 스크립트는 `scratchpad/probe_ga_21.py`에 남겼다.

**다만 세션 생성 시점의 형식 수락만 확인한 것이다.** 런타임 이벤트 형식, 오디오 왕복, VAD 동작 변화는 dev 배포 후 실측이 필요하다. 2.1은 "침묵·소음 처리와 끼어들기 동작 개선"을 명시하고 있어 **감지 지연 수치가 달라질 수 있다.**

## 어디

| 커밋 | 브랜치 | 내용 |
|---|---|---|
| `41fe01b` | `fix/v4-session-generation-guard-dev` | 원본 |
| `0b08c91` | `fix/v4-session-generation-guard` | 체리픽 (main PR용) |

교체한 호출부 8곳.

| 파일 | 지점 |
|---|---|
| `webrtc_api.py` | GA/Legacy 경로 분기 |
| `voice_chat_v4_router.py` | 세션 config 빌드 2곳 |
| `voice_chat_v3_router.py` | 동일 2곳 |
| `realtime_speaker.py` | `session.update` 생략, `response.create` 필드 |
| `server_webrtc.py` | v4 GA LangGraph 스킵 가드 |

## 검증

`pytest ai_voice/voice_tests/` — **dev 사본 165 passed, main 기준 159 passed.** (차이 6건은 dev에만 있는 v3 가드 테스트)

실패 1건(`test_default_exposes_both_v3_and_v4`)은 **이 변경과 무관하다.** stash로 되돌린 상태에서도 동일하게 실패한다. 로컬 `agent_env`의 fastapi 0.141.1이 핀(0.115.9)과 어긋나 라우트 열거가 비는 기지 이슈로, conftest의 카나리아가 의도대로 잡아낸 것이다.

회귀 테스트 `test_ga_model_routing.py` **5건 신규**. 핵심은 시그니처가 아니라 **실제 호출로 검증**하는 것이다.

- 판정을 고정 집합으로 되돌리면 → `gpt-realtime-2.1`이 Legacy를 타서 실패
- 접두사를 너무 넓게 잡으면 → `gpt-realtime-1.5`가 GA를 타서 실패

Codex 리뷰 **지적 없음**.

## 프론트 수정

백엔드 dev 머지 후 프론트 미러(`AI_pro/voice/js/`)를 이어서 수정했다. **업로드 전 상태**다.

| 파일 | 변경 |
|---|---|
| `voice_runtime.js:3` | `VOICE_WEBRTC_MODEL` → `gpt-realtime-2.1` |
| `webrtc_voice_module.js:14` | 정규화 `gpt-realtime` → `gpt-realtime-2.1` |
| `voice_input_plugin.js:28` | 동일 |
| `modes/voice_webrtc_handler.js:22` | 동일 |
| `webrtc_voice_module.js:1160~1172` | **죽은 폴백 제거** |

정규화가 **3파일에 중복**돼 있어 모델명을 바꾸면 3곳을 동시에 고쳐야 한다. 이번에도 그랬다. 대상값이 `gpt-realtime-1.5` 였는데 이는 백엔드에서 Legacy 경로로 라우팅되고 그 경로는 폐지됐다 — 즉 정규화가 **죽은 경로로 가는 매핑**이었다. OpenAI 가 `gpt-realtime` 의 대체로 지정한 2.1 로 돌렸다.

폴백은 제거했다. GA 세션에서 `session.audio.output.format.rate` 오류가 나면 `gpt-realtime-1.5` 로 재접속하는 코드였는데, 그 모델이 폐지된 경로를 타므로 **회복 가능한 설정 오류를 확정 실패로 바꾸는** 동작이었다. 이제 오류를 그대로 보고한다.

`webrtc_voice_widget.js:100` 의 `gpt-4o-realtime-preview-2024-12-17`(2026-05-07 폐지)은 **건드리지 않았다.** 미러 안에서 참조가 없어 사용 여부가 불확실하고, 이번 전환과 별개 사안이다.

### 검증

`node --check` 4파일 전부 통과.

| 파일 | 크기 |
|---|---|
| `voice_runtime.js` | 3,998 bytes |
| `webrtc_voice_module.js` | 56,754 bytes |
| `voice_input_plugin.js` | 17,152 bytes |
| `modes/voice_webrtc_handler.js` | 29,505 bytes |

### dev 실측 — 통화 성립

백엔드 dev 머지 + 프론트 업로드 후 재현 클라이언트로 확인했다. **2.1 을 보내는 것 자체가 배포 검증**이다 — 배포가 안 됐다면 폐지된 Legacy 경로로 가서 실패한다.

```
POST /agent/session   → HTTP 200,  model=gpt-realtime-2.1 (요청 그대로 반향)
ICE                   → checking → completed (0.1초)
DataChannel           → open
수신 이벤트           → 2건 [session.created, session.updated]
error 이벤트          → 0건
인바운드 오디오       → 871프레임 (약 46 f/s)
```

4일차 기준선과 동일하다. 이벤트 2건은 GA 세션의 정상 패턴으로 **평면 `session.update` 가드가 여전히 작동**한다는 뜻이고, 프레임 레이트도 그때(1,107프레임/25초)와 같다. 에러 토스트의 원인이던 error 이벤트도 0건이다.

| 지표 | 4일차 (`gpt-realtime-2`) | 지금 (`gpt-realtime-2.1`) |
|---|---|---|
| DataChannel 이벤트 | 2건 | 2건 |
| error 이벤트 | 0 | 0 |
| 프레임 레이트 | 약 46 f/s | 약 46 f/s |

ICE 가 0.1초에 끝나 5초 지연 제거도 살아 있음을 재확인했다.

#### 확인하지 못한 것

**이 테스트는 프론트를 거치지 않는다.** 재현 클라이언트가 자기 모델 문자열을 직접 보내므로, 업로드한 프론트가 실제로 2.1 을 보내는지는 **브라우저로 확인해야** 안다. `?v=` 부재로 구버전 JS 가 캐시돼 있을 수 있어 시크릿 창 또는 강력 새로고침이 필요하다.

**VAD 동작은 측정하지 못했다.** 무음을 보내는 테스트라 발화 감지·전사·응답 생성 경로가 돌지 않았다. 2.1 이 침묵·소음·끼어들기 처리 개선을 명시하고 있어 **감지 지연이 4일차의 +0.3~0.5초와 달라졌을 수 있다.** 이 수치가 과제 2번(시간 옵션) 설계 근거이므로 TTS 주입 측정이 남았다.

스크립트는 `scratchpad/v4_ga21.py` 에 남겼다.
