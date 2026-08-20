---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-18
---

# 화자 관문 Phase 1 구현

> [!summary] 한 줄 요약
> 게이트 4번째 관문(발화 단위 화자 임베딩 대조)을 브레인스토밍 → 스펙 → 계획 → TDD 로 구현했다 — 커밋 7개, **189 passed**, Codex 리뷰 3라운드 전부 해소. 남은 것은 dev 서버 수동 설치·배포·2화자 실측. 프로젝트: [[AI음성]]

## 무슨 작업

[[AI음성 - 화자 구분 실시간 적용 검토]]의 권장안을 스펙으로 확정하고(사용자 결정 4건: 독립 발화만 / 첫 통과 발화 자동 등록 / 관찰→차단 2단계 / sherpa-onnx) Phase 1(관찰 모드)을 구현했다. 스펙·계획은 `docs/superpowers/specs·plans/2026-08-18-speaker-gate*.md`.

```
v4/speaker_gate.py (신규 168줄)
├─ SpeakerProfile        : 등록·이동평균 갱신(유사 발화만)·코사인 유사도
├─ SpeakerEmbedder       : sherpa-onnx lazy 싱글턴, SPEAKER_MODEL_PATH env
├─ channel_count_of      : av 12 는 .channels 부재 → layout.channels 도출
└─ mono_from_frame_array : planar/packed(인터리브) 양쪽 모노화

audio_tracks.py 접목
├─ _release_utterance()  : 세 방출 지점(barge-in·침묵·force_flush) 일원화
└─ _speaker_stage()      : 등록→대조, observe(백그라운드)/enforce(방출 전 대기),
                           락 직렬화, 전 경로 fail-open

voice_chat_v4_router.py  : 주입 필드 3종 (mode/threshold/min_ms)
```

## 모델 벤치 (Heami=사용자, Zira=타인 TTS)

| 모델 | 본인쌍 | 타인 교차쌍 | 추론 |
|---|---|---|---|
| **3dspeaker eres2net (선정, 37.8MB)** | **0.876** | **0.400~0.482** | 137ms |
| wespeaker resnet34 (탈락) | 0.901 | 0.701~0.788 | 133ms |

분리 폭 ~0.4 로 기본 임계값 0.6 이 정확히 본인/타인 사이. 짧은 발화(0.2초) 유사도 0.563 은 `speaker_check_min_ms=1000` short-skip 설계를 실측으로 뒷받침.

## Codex 리뷰 3라운드

| 지적 | 조치 |
|---|---|
| P1 packed 스테레오 인터리브를 모노로 오인 (aiortc opus 디코드 실형태) | `mono_from_frame_array(arr, channels)` 신설 (`21c9272`) |
| P2 observe 백그라운드 태스크 등록 순서 경합 | 트랙별 asyncio.Lock 직렬화 (`21c9272`) |
| P1 av 12 는 `.channels` 부재 → channels=1 폴백으로 수정 무력화 | `channel_count_of()` — layout.channels 도출, 실 av 프레임 테스트 (`8f763b3`) |
| P1 sherpa-onnx 의존성 미선언 | 사용자 승인 후 requirements.txt 추가 (`7a4ba0a`) |

## 어디

커밋 `13e1e4d`..`7a4ba0a` (7개), 브랜치 `fix/v4-session-generation-guard-dev`. 신규 테스트 15건 (`test_v4_speaker_gate.py`).

## 검증

- `pytest ai_voice/voice_tests/` — **189 passed**, 1 failed (기존 로컬 fastapi 환경, 무관). 기존 게이트 테스트 비파괴.
- 실모델 스모크: eres2net 로딩 + 512차원 임베딩 산출 확인.
- 기본값 `off` — 배포돼도 동작 변화 0. fail-open 6경로 테스트 고정.

## 남은 절차 (Phase 1 마무리)

- [ ] dev 서버 수동 설치: venv 에 `pip install sherpa-onnx`, eres2net 모델 업로드, `.env` 에 `SPEAKER_MODEL_PATH` (+선택 `VOICE_SPEAKER_GATE_MODE=observe`)
- [ ] 푸시 + dev PR 머지(app restart) — 로컬 푸시 인증 문제로 사용자 진행
- [ ] 2화자 관찰 실측 (Heami+Zira 스케줄, `speaker_gate_mode=observe` 주입) → 유사도 분포 → Phase 2 임계값 확정
