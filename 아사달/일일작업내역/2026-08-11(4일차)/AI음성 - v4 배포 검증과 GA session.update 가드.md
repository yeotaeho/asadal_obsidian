---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# v4 배포 검증과 GA session.update 가드

> [!summary] 한 줄 요약
> dev 배포 후 재현 클라이언트로 v4 부활을 실측 확인하고, 그 과정에서 드러난 매 통화마다 뜨는 에러 토스트의 원인을 찾아 함께 제거했다. 프로젝트: [[AI음성]]

## 무슨 작업

[[AI음성 - v4 무음 원인 규명]]에서 고친 `session_config` 전달이 실제로 동작하는지 dev 배포 후 aiortc 재현 클라이언트로 확인했다. 확인하는 김에 남아 있던 `error` 이벤트 하나를 추적해 함께 잡았다.

## 3단계 실측

| 단계 | DataChannel 이벤트 | 인바운드 오디오 |
|---|---|---|
| 수정 전 | **0건** | **0프레임** |
| GA 세션 설정 전달 후 | 3건 — `session.created` · `session.updated` · **`error`** | 1,123프레임 |
| session.update 가드 후 | **2건** | 1,107프레임 |

오디오는 초당 약 47프레임으로 끊김 없이 들어온다. 20ms 프레임 기준 정상 속도다.

## 남은 error 의 정체

`session.updated` 직후 서버가 보내는 두 번째 `session.update`가 거부당하고 있었다.

```
"code": "missing_required_parameter"
"message": "Missing required parameter: 'session.type'."
```

`send_session_update`가 만드는 payload가 legacy **평면** 스키마였다. `modalities`·`input_audio_format`·`output_audio_format`·`voice`가 최상위에 있고 `session.type`이 없다. 그 함수 안 주석도 스스로 "WebRTC 전용 평면 스키마"라고 밝히고 있었다. GA는 이를 `session.audio.*` 아래 중첩으로 받는다.

**기능 손실은 없었다.** GA 세션은 최초 POST에 설정을 전부 싣고 들어간다.

| 항목 | GA 초기 설정에 포함 |
|---|---|
| VAD | `semantic_vad`, eagerness low, interrupt_response |
| 전사 | `gpt-4o-transcribe`, ko |
| 노이즈 감소 | far_field |
| 음성·지침·tools | 포함 (`_build_session_ga`) |

**문제는 화면이었다.** 브라우저 이벤트 중계를 복원한 뒤로 이 error가 프론트까지 닿는데, 메시지가 `handleError`의 어느 분기에도 안 맞아 기본 문구로 떨어진다 — "잠시 음성 서비스 이용이 원활하지 않습니다". 오디오는 멀쩡히 나오는데 접속할 때마다 이 토스트가 떴다.

## 왜 서버만 몰랐나

프론트는 같은 함정을 이미 알고 막아뒀다.

```javascript
// webrtc_voice_module.js:1428
// session.update를 재전송하지 않는다. flat 포맷 전송 시 GA API가 거부한다.
if (!this.isGaRealtimeModel()) { ... }
```

프론트 팀이 먼저 부딪혀 가드를 넣었는데 서버에는 반영되지 않은 상태였다. 같은 처리를 payload를 만드는 지점 하나에 뒀다 — 호출부마다 막는 것보다 작고, 나중에 다른 호출자가 생겨도 자동으로 보호된다.

## 어디

| 커밋 | 내용 | 브랜치 |
|---|---|---|
| `50d6c84` | GA 세션 평면 `session.update` 가드 + 테스트 4건 | `fix/v4-session-generation-guard-dev` |
| `c218828` | 같은 내용 | `fix/v4-session-generation-guard` (main PR용) |

핵심 파일은 `ai_voice/services/voice_chat/workflow/realtime_speaker.py`의 `send_session_update`다.

## 검증

`pytest ai_voice/voice_tests/` — **dev 브랜치 100 passed, main 기준 브랜치 94 passed**.

변이 실측을 **양방향**으로 했다.

| 변이 | 결과 |
|---|---|
| 가드 제거 | `test_ga_model_skips_session_update` 실패 |
| 모든 모델을 건너뛰게 | legacy 전송 테스트 **3건** 실패 |

payload가 언젠가 GA 형태로 바뀌면 `test_payload_is_flat_schema_which_ga_rejects`가 실패해 가드를 재검토할 계기가 된다. 건너뛸 이유가 사라졌는데 계속 건너뛰는 상태를 막는 장치다.

Codex 리뷰는 지적 없이 통과했다.

배포 후 재현 클라이언트로 `error` 이벤트가 실제로 사라진 것까지 확인했다 — 3건 → 2건.

## 배운 것

코드만 읽고 내린 결론이 이번 건에서 두 번 빗나갔다. **재현 클라이언트를 만든 뒤로는 한 번에 잡혔다.** aiortc로 브라우저 절차를 그대로 밟는 스크립트는 30줄 남짓인데, 그걸 먼저 만들었으면 훨씬 빨랐다. 서버 로그가 `level=logging.WARNING`으로 막혀 있어 더 그랬다.

## 추후 작업

- [ ] 모바일 실기기 검증 (v4 경로로 실제 통화)
- [ ] 원본 브랜치(`fix/v4-session-generation-guard`)로 main PR
