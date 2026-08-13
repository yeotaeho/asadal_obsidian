---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# 필터 임계값 주입과 응답 큐 경합 수정

> [!summary] 한 줄 요약
> 모바일 진단을 위한 필터 판정 로그, 과제 2번의 기반이 되는 임계값 세션 주입, RAG 중 발화 충돌을 막는 응답 큐 수정 — 세 커밋을 한 배치로 처리했다. 프로젝트: [[AI음성]]

## 무슨 작업

### `0d88b9d` — 필터 판정 로그를 info로

`NoiseFilteredAudioProxyTrackV4`는 관문 세 개(발화 길이·실발화 시간·실발화 비율)에 걸린 발화를 **통째로 버리는데**, 그 판정이 debug라 운영에서 안 보였다. 폐기는 무반응의 직접 원인이 될 수 있으므로 info로 올렸다. 통과도 같이 올렸다 — 폐기 로그만 있으면 "로그가 없다"가 오디오 미도달인지 전부 통과인지 구분되지 않는다.

당시에는 몰랐지만 이 변경은 **로거 핀 때문에 무효**였다 — [[AI음성 - 오디오 로거 핀 제거와 방향 태그]] 참고.

### `ada3c8e` — 필터 임계값을 세션별로 주입 가능하게

하드코딩돼 있던 관문 임계값 6종을 세션 생성 요청으로 받게 했다.

| 필드 | 범위 |
|---|---|
| `min_utterance_duration_ms` / `barge_in_ms` | **100~3000ms** — 과제 2번(0.1~3초) 구간 그대로 |
| `min_speech_duration_ms` | 0~3000ms |
| `speech_rms_threshold` / `silence_threshold_rms` | 0~32767 |
| `speech_ratio_threshold` | **0~1 비율** — 30 같은 퍼센트 값이 오면 전 발화 폐기라 검증으로 차단 |

배관은 `session_config` 때와 같은 경로다 — 라우터가 `metadata["audio_filter"]`에 실으면 브릿지가 트랙 생성 시 kwargs로 편다. 안 보낸 값은 dict에 넣지 않아 기본값이 유지된다. **모바일 프리셋 값은 실측 없이 정하지 않았다** — 근거 없이 낮추면 노이즈 오작동만 는다.

### `a8d2d35` — 응답 큐 경합

RAG 조회 중에 사용자가 말하면 `ACTIVE_RESPONSE_IN_PROGRESS`가 났다. 결함이 둘이었다.

1. `_send_next_queued_response`가 pending 검사 **전에** `popleft()` 한다. 검사에 걸리면 항목이 이미 큐에서 빠진 뒤라 **그대로 증발한다.** LangGraph 응답이 조용히 사라질 수 있었다.
2. v4 orchestrator의 `_send_tool_continue_response`가 큐를 우회해 `speaker.request_response`를 직접 불렀다. 진행 중인 응답 위에 그대로 얹힌다.

1번은 꺼내기 전 검사로, 2번은 매니저가 주입한 콜백으로 **같은 큐**에 태우도록 고쳤다. 콜백을 생성자로 못 넘기는 이유는 `orchestrator_kwargs`를 v3 orchestrator와 공유하는데 `v3/`가 바이트 고정이라 시그니처를 못 늘려서다 — `browser_channels`를 `attach_track`으로 우회한 것과 같은 제약이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `0d88b9d` | 판정 로그 info + 테스트 4건 | `v4/audio_tracks.py` |
| `ada3c8e` | 임계값 주입 배관 + 테스트 8건 | `voice_chat_v4_router.py`, `server_webrtc.py` |
| `a8d2d35` | 큐 드롭 방지 + tool 재개 큐 경유 + 테스트 9건 | `server_webrtc.py`, `v4/tool_call_orchestrator.py` |

main PR용 원본 브랜치에도 체리픽 완료.

## 검증

**113 → 120 → 129 passed.** 변이 실측은 커밋마다 돌렸다.

특기할 것은 `a8d2d35`의 변이 3번 — "매니저가 주입 안 함"이 처음에 안 잡혔다. 테스트가 dispatcher를 직접 주입해버려 배선이 끊겨도 통과했다. 실제 인스턴스를 만들어 확인하는 테스트를 따로 두고서야 잡혔다. 구현을 직접 부르는 테스트는 배선 절단을 못 잡는다는 교훈이 이번 브랜치에서 세 번째다.

Codex 리뷰(3커밋 범위) 지적 없음.
