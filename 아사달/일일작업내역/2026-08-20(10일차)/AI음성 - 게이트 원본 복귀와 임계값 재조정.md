---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-20
---

# 게이트 원본 복귀와 임계값 재조정

> [!summary] 한 줄 요약
> 1.2초 상향 2건을 롤백해 원본(v2.6.7) 기준으로 되돌린 뒤, 사용자 결정에 따라 임계값을 다시 조정하고 cedar 음성과 barge-in 정산 동작까지 원본으로 복원했다. 프로젝트: [[AI음성]]

## 무슨 작업

전날 넣었던 1.2초 상향 2커밋을 revert 로 되돌리는 것에서 시작해, 게이트 기본값 전체를 원본 인수인계 문서(v2.6.7) 기준으로 맞춘 뒤 다시 사용자 지시에 따라 개별 조정했다. 커밋 **7건**(revert 2 + 조정 5).

조정 과정에서 문서 대조로 **cedar 음성 미복원**을 발견해 함께 복원했다. v4 전면 롤백(`e70ab01`)에 쓸려나간 뒤 게이트 복원(`4fd394a`)이 audio_tracks 범위만 되돌려서 `constants.py` 의 GA voice 가 marin 으로 남아 있었다.

**최종 기본값** — min_utterance **1200** · silence_frames **20(400ms)** · speech_rms **300** · min_speech **900** · ratio 0.30 · barge_in **1200** · force_flush 30000 · 드레인 4배속.

## 왜

전날 조정(min_utterance·barge_in 1200)이 끼어들기 오탐을 못 막은 채 짧은 대답만 잃는 상태로 보여, **원본 임계값에서 다시 출발해 하나씩 조정**하기로 했다. GA 경로에 수치를 덮어쓰는 요소가 있는지부터 검토했고 없음을 확인했다.

프론트(`webrtc_voice_module.js`)는 필터 7필드를 아예 전송하지 않아 서버 클래스 기본값이 그대로 적용되고, GA 모델이면 프론트의 `session.update`(server_vad 백업 설정) 전송도 스킵된다. 즉 **서버 기본값이 유일한 실효값**이다.

다만 수치가 지배하지 못하는 우회 경로 2개를 확인했다.

```
① 패스스루 창 — barge-in 통과 후 침묵이 올 때까지 무심사 통과
② OpenAI interrupt_response — 도달한 오디오면 길이 무관하게 응답 중단
   → 서버 수치는 "도달 여부"만 통제한다
```

이어진 문답에서 **barge-in 트리거가 실발화가 아니라 소리(voice) 누적**이라는 점, `barge_in == min_utterance` 인 한 **침묵 종료 경로가 항상 관문 ①에서 미달**해 비율 관문(30%)이 결정권을 갖지 못한다는 점도 정리했다. 원본 문서의 주의사항 3번이 같은 현상을 기록하고 있다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `6c97110` | Revert — min_utterance 1200 상향 | `v4/audio_tracks.py` |
| `dbf7041` | Revert — barge_in 1200 상향 | `v4/audio_tracks.py` |
| `7a9e8c6` | 원본(v2.6.7) 복귀 — 1000/3/300/600 | `v4/audio_tracks.py` |
| `150c3f0` | min_utterance·barge_in 1.2초 상향 | `v4/audio_tracks.py` |
| `afb7372` | **cedar 음성 복원** (롤백 유실분) | `constants.py` |
| `21f2442` | barge-in 관문 미달 시 정산(폐기·리셋) 복귀 | `v4/audio_tracks.py` |
| `307f5d6` | 침묵 종료 60ms → **400ms** | `v4/audio_tracks.py` |
| `73f3fee` | 실발화 누적 600 → **900ms** | `v4/audio_tracks.py` |

테스트는 매 커밋 동반 갱신했다(`test_v4_filter_gate_and_drain.py` — 시나리오 프레임 수, 기본값 고정 핀, 침묵 종료 프레임 수).

## 되돌린 것과 남긴 것

8월에 넣었던 P1 수정 중 **P1-1(barge-in 실패 시 누적 유지)은 되돌렸고**, P1-2(드레인 4배속)와 force_flush 관문 우회는 유지했다.

되돌린 결과 관문 미달 발화가 barge_in 주기(1.2초)로 계속 폐기되므로, **기본값에서는 force_flush(30초)에 도달하지 않는다**. 세션 주입으로 `force_flush < barge_in` 인 구성에서만 살아 있는 안전장치가 됐다.

`silence_frames_to_end` 는 원본 3(60ms)으로 갔다가 **400ms 로 재상향**했다. 60ms 는 어절 사이 쉼·숨 고르기마다 발화를 쪼개 1.2초 미만 조각을 만들고, 그 조각이 min_utterance 관문에서 통째 소실되는 8-14 실측 메커니즘 그대로였기 때문이다.

## 인지하고 있는 비용

| 항목 | 영향 |
|---|---|
| min_utterance 1200 | "네." 같은 1.2초 미만 대답 전부 폐기 |
| min_speech 900 | barge-in 시점 실질 비율 요구가 **75%** — 원본 실측 정상 질문 분포(66~95%) 중 하단이 잘린다 |
| barge-in 정산 복귀 | 저음량 연속 발화가 1.2초 주기로 반복 폐기 (force_flush 도달 불가) |
| speech_rms 300 | OpenAI VAD 실측 하한(120)보다 보수적 — VAD 가 살릴 발화를 게이트가 먼저 폐기 |

끼어들기 오탐 방지를 우선한 선택이며, dev 배포 후 재측정으로 비용을 확인한다.

## 문서 갱신

[[AI음성 - 런타임 플로우]] 를 현행 코드 기준으로 갱신했다.

- 다이어그램 **7개 전부를 카드 스타일로 재작성** — 각 노드에 제목·위치·2~3줄 설명을 넣어 다이어그램만으로 읽히게 했다.
- **게이트 파라미터 표 신설** — 10개 항목의 값·한글 이름·설명 + 판정 흐름 요약.
- **GA 세션 설정 표 신설** — 모델 2.1, cedar, semantic_vad·eagerness low·interrupt_response·create_response, far_field, 지시문 변경 2건.
- barge-in 서술 정정 — "실발화 1.2초" → **"소리(RMS>50) 누적 1.2초"** (트리거는 voice_duration).

## 검증

- `pytest ai_voice/voice_tests/` — 커밋마다 실행, **177 passed**. 실패 1건(`test_default_exposes_both_v3_and_v4`)은 로컬 fastapi 버전 불일치로 기존부터 있던 환경 문제다.
- GA 경로 실효값 검토 — 프론트 payload(`createSessionRequestPayload`), 라우터 주입(`_build_audio_filter_options`), 트랙 생성(`server_webrtc:1318`) 3구간을 읽어 덮어쓰기 없음 확인.
- 미검증 — 프론트 확인은 로컬 사본(`Documents/AI_pro/voice/js/`) 기준이라 Gitea 배포본과의 동일성은 확인하지 않았다.

## 남은 일

- [ ] dev 배포 후 A/B/C 재측정 — 특히 min_speech 900 의 정상 질문 차단률
- [ ] 원본 브랜치 main PR (게이트 커밋 전체가 아직 dev 사본에만 있다)
