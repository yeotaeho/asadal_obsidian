---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-28
---

# 화자 판정 shadow 구현과 dev 브랜치 반영

> [!summary] 한 줄 요약
> 통화 첫 발화로 주인 목소리를 잡고 이후 발화마다 점수를 로그로만 남기는 화자 판정(shadow)을 v4 에 붙여 main·dev 브랜치로 올렸다. 모델은 eres2netv2 하나를 이미지에 넣는다. 기능: [[화자 판정]]

## 무슨 작업

화자 판정을 소리를 막는 관문이 아니라 곁가지로 붙였다. 업링크 프레임과 OpenAI 발화 경계를 녹음기(PairRecorder)와 똑같이 받아 발화를 자르고, 앞 3초를 eres2netv2 로 임베딩해 작업 스레드에서 판정한다. 결과는 `[SPK]` 로그로만 남고 통화에는 손대지 않는다.

기준 목소리는 통화 첫 1초 이상 발화 3개의 평균이다. 통화 메모리에만 두고 통화가 끝나면 버린다. 1초 미만은 보류하고, 임계는 AI Hub 로 정한 임시값 0.5 다. 스위치는 오디오 탭과 같은 플래그 파일(`data/logs/voice_speaker_gate` 내용 `shadow`)이다.

```
브라우저 → 서버 업링크 트랙 ─────────────────▶ OpenAI (소리 그대로)
               └ on_frame ─┬▶ PairRecorder
                           └▶ SpeakerGate (24kHz 링버퍼 5초)
OpenAI speech_started / speech_stopped ─▶ 구간 자르기(앞 3초) → 작업 스레드 2개에서 임베딩
    → 등록 1~3 / 점수·판정 / 계산 ms / 발화끝후 ms
OpenAI output_audio_buffer.started ─▶ 첫 답변 소리 발화끝후 ms (판정이 먼저 끝났는지 대조)
```

모델은 Dockerfile 이 빌드 때 sherpa-onnx 공식 릴리스에서 받아 sha256 을 확인하고 `/opt/speaker_models` 에 둔다. requirements 에는 `sherpa-onnx==1.13.5` 와, 직접 쓰게 된 `scipy==1.17.1` 을 적었다.

## 왜

AI Hub 측정에서는 v2 의 오류율이 base 의 절반이었지만, 실제 통화 경로(브라우저 잡음 제거·AGC·Opus)의 v2 점수는 없었다. 거부 동작을 바로 켜면 임계값을 근거 없이 정하게 되므로 판정만 기록하는 단계를 먼저 둔다. Voice Focus 는 CER 악화로 필터 사용을 접었으니, 다른 사람·TV 목소리 대응은 목소리 지문이 맡는다.

모델 두 개를 서버에 올려 비교하는 안은 접었다. 오디오 탭이 실제 경로의 소리를 남기므로 모델 비교는 오프라인으로 하면 된다. 서버 볼륨 대신 이미지를 고른 이유는 배포가 이미지 빌드라 서버 수작업이 없고, 운영 릴리스 태그 하나에 코드와 모델이 묶이기 때문이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `7e22c6d` (dev `e313640`) | 화자 판정 shadow 모듈·연결·테스트 | `v4/speaker_gate.py` · `server_webrtc.py` · `test_speaker_gate.py` · `test_session_generation_guard.py` |
| `bfc92af` (dev `ace2a4e`) | sherpa-onnx·scipy 핀, 모델 이미지 포함 | `requirements.txt` · `Dockerfile` |
| `659ea2b` (dev `0891c27`) | 배포 문서에 스위치·모델·로그 읽는 법 | `docs/DEPLOY.md` |

브랜치는 `feat-speaker-gate`(main 기준)와 `feat-speaker-gate-dev`(dev 기준 cherry-pick, 충돌 없음)이고 둘 다 푸시했다. dev PR 은 사용자가 열어 머지한다.

## 검증

- `pytest ai_voice/voice_tests` 는 main 기준 **426 passed**, dev 기준 **442 passed** 였다. `pytest tests` 는 8 passed.
- 새 테스트는 6건이다. OpenAI 오프셋대로 자르기와 3초 상한, 1초 미만 보류, 등록 3개 뒤 본인·타인 판정과 16kHz 변환, 모델이 없을 때 경고 1회, 플래그 파일 스위치, 브릿지 배선을 본다.
- 실제 모델로 끝까지 돌렸다. 로컬 venv 에 sherpa-onnx 1.13.5 를 설치해도 numpy 는 1.26.4 그대로였다. AI Hub 같은 성별 두 화자를 48kHz 프레임부터 흘리니 판정 1회가 **약 280ms**, 모델 로드를 포함한 첫 판정이 890ms 였다.
- 그 시험에서 본인은 0.745·0.613, 타인은 0.527·0.466 이 나왔다. **타인 한 건이 임시 임계 0.5 를 넘었다.** 임계를 실통화 점수로 다시 정해야 하는 근거다.
- Dockerfile 의 해시 줄은 로컬 모델로 통과, 틀린 해시로 실패를 확인했다. 다운로드 주소는 302 뒤 200 이고 크기가 71,441,526바이트로 로컬 파일과 같다. Docker 빌드 자체는 로컬에 Docker 가 없어 dev 배포에서 확인한다.
- 일일 폴더 번호를 바로잡았다. 9-23 이 30일차라 오늘 폴더는 31일차다. `2026-09-28(30일차)` 에 두었던 Voice Focus 노트를 이 폴더로 옮겼다.
