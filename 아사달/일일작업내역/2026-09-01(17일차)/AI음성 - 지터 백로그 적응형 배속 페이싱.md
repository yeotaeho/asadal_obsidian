---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-01
---

# 지터 백로그 적응형 배속 페이싱

> [!summary] 한 줄 요약
> 지터 뭉침 후 버퍼 수위가 래칫으로 고여 발화 앞부분이 잘리던 문제를 수위 기준 4배속 페이싱으로 수선했다 — 업링크 전용, **244 passed**, Codex 리뷰 클린. 기능: [[음성채팅 v4 복원]]

## 무슨 작업

`_pace_playout` 에 백로그 따라잡기를 넣었다. 수위가 목표(prebuffer 4프레임)를 넘으면 드레인과 같은 수법(샘플 수 1/4 전달)으로 4배속 페이싱해 빚을 갚는다. 기반 클래스 기본값 `_CATCHUP_SPEEDUP = 1`(비활성), 업링크 트랙 2종(`UplinkAudioProxyTrackV4`·`NoiseFilteredAudioProxyTrackV4`)만 4 로 켠다.

다운링크는 의도적으로 1배속 유지다. TTS 스무딩과 barge-in 시 절단 범위가 서버 버퍼 깊이에 걸려 있어, 빨리 흘리면 끊어도 브라우저에 남는 꼬리가 길어진다.

## 왜

모바일 지터 실측 **633~772ms** ([[AI음성 - 오디오 탭 실측과 문제 종합]]) — 언더플로 동안 침묵으로 때운 시간만큼 페이싱 시계에 빚이 생기는데, 리베이스(`-0.25` 탕감)는 그 빚을 장부에서 지울 뿐이라 뒤늦게 도착한 뭉치가 1배속으로만 빠진다. 결과는 두 단계다.

```
수위 L, 지터 갭 G → 결과 수위 = max(L, G)   ← 래칫: 한번 오르면 안 내려옴
수위가 상한(960ms, 48프레임) 도달 → popleft ← 발화 앞부분부터 폐기
```

상한 960 에서도 **만수위 47/48** 이 관측된 상태였다. NetEq 급 시간축 신축은 불필요하다고 판단했다 — 업링크 하류는 OpenAI 모델이라 사람 귀가 없고, pts 는 원래 간격으로 찍혀 배속 유입이 무해하다(드레인 4배속과 동일 근거, 실전 검증 완료). 상세 원리: [[지터버퍼와 재타이밍]].

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `7550cbb` | 수위 기준 4배속 페이싱 + 회귀 테스트 4건 | `ai_voice/services/voice_chat/v4/audio_tracks.py` · `voice_tests/test_v4_jitter_catchup.py` |

브랜치 `fix/v4-jitter-backlog-catchup` (main 기준, AI_Pro_VOICE).

## 검증

- 신규 `test_v4_jitter_catchup.py` 4건 — 백로그 시 4배속(업링크 2종 파라미터라이즈) / 목표 수위 이하 1배속 / 다운링크 1배속 고정
- `pytest ai_voice/voice_tests -q` — **244 passed** (기존 240 비파괴)
- `pytest tests -q` — 8 passed
- Codex 리뷰(범위: main..branch) — **지적 0건** ("No actionable correctness defect")
