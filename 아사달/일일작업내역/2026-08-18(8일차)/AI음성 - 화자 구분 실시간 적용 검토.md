---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-18
---

# 화자 구분 실시간 적용 검토

> [!summary] 한 줄 요약
> "사용자 외 타인의 음성 차단" 과제의 사전 검토 — **ElevenLabs 실시간 API 로는 불가**(Scribe v2 Realtime 에 diarization 없음, 공식 문서 확정). 배치 우회는 지연·화자 ID 비영속으로 부적합. 권장은 게이트 4번째 관문으로 **로컬 화자 임베딩 유사도**. 프로젝트: [[AI음성]]

## 배경

기존 화자 구별은 **업로드 배치 경로**에만 있다 (SOUND_SPEAKER_DIARIZATION_FLOW 문서 기준).

```
sound.htm → /AI_voice/voice-learning/transcribe-and-summarize
→ learning_router → stt_and_segments_service
→ elevenlabs_timestamped.py (scribe_v1 · diarize=true · 파일 업로드 배치)
→ 세그먼트 + LLM 화자명 추론 → SOUND_SPEAKERS/SEGMENTS 저장
```

실시간 음성 채팅(WebRTC v4)과는 완전히 별개 경로다. API 키·클라이언트 인프라(`ELEVENLABS_API_KEY`)만 재사용 가능하다.

## 검토 결과

### ① ElevenLabs 로 실시간 화자 구분 — 불가

| 모델 | diarization | 비고 |
|---|---|---|
| Scribe v2 (배치) | **지원** (최대 32화자) | 파일 업로드, 현 배치 경로가 쓰는 방식 |
| Scribe v2 Realtime (~150ms 스트리밍) | **미지원** | 공식 문서의 기능 목록에 diarization 부재. 저지연 우선으로 의도적 제외 |

### ② 배치 API 를 발화 단위로 우회 — 가능하나 부적합

게이트가 발화를 버퍼로 쥐고 있으므로 접목 지점 자체는 있다(방출 전 버퍼 PCM → WAV → 배치 diarize). 그러나 세 가지가 치명적이다.

- **지연** — 발화당 HTTP 왕복 + 배치 처리로 +수 초. 완화로 확보한 +1.0~1.8초가 다시 무너진다.
- **화자 ID 비영속** — speaker_0/1 라벨은 **요청 안에서만 유효**하다. 요청 간 동일 인물 매칭이 안 되므로 "누가 사용자인가"를 API 가 알려주지 못한다 (배치 경로도 그래서 LLM 이름 추론을 뒤에 붙인 것).
- **과금** — 발화마다 STT 1회.

### ③ 권장 — 게이트 4번째 관문: 로컬 화자 임베딩 유사도

복원된 게이트 구조가 정확히 맞는 자리다. 게이트는 이미 **발화 단위 채택/폐기**를 하고 있으므로, 화자 검사를 관문 하나로 추가하면 된다.

```
발화 버퍼 (PCM 확보됨)
→ 화자 임베딩 추출 (ECAPA-TDNN 등 로컬 모델, CPU 발화당 ~50-150ms)
→ 등록된 사용자 임베딩과 코사인 유사도
→ 미달이면 폐기 (기존 3관문과 같은 정산 경로)
```

- **사용자 등록** — 통화 첫 통과 발화(들)로 프로파일 생성. 세션 메타데이터로 임계값 주입 가능(기존 7필드 배관과 동일 패턴).
- **접목 소스 위치** — `v4/audio_tracks.py` `_process_utterance`(정산 시 버퍼 접근) 또는 `_gate_passes` 옆 신규 검사, 임계값 배관은 `voice_chat_v4_router.py` + `server_webrtc.py` metadata.audio_filter 경로 재사용.
- **의존성** — speechbrain(ECAPA) 또는 resemblyzer. GPU 불필요, 단 세션당 CPU 비용 실측 필요.

### ④ 한계 — 겹침(overlap)은 diarization 으로 못 푼다

위 방법이 막는 것은 **사용자가 말하지 않는 동안의 타인 발화**(독립 발화 단위)다. 사용자 발화 **도중에 겹쳐 들어온** 타인 음성은 같은 발화 버퍼에 섞여 있어, 발화 단위 채택/폐기로는 분리 불가 — 프레임에서 특정 화자만 지우는 것은 diarization 이 아니라 **음원 분리(target-speaker extraction)** 영역이고, 실시간 상용 품질은 별도 GPU 모델 과제다. 요구 범위를 "겹침 제거"까지 잡으면 난이도가 한 단계 다르다는 것을 과제 정의 때 명시해야 한다.

### 대안 벤더 참고

실시간 diarization 을 제공하는 STT(Deepgram 등)도 있으나 한국어 품질 이슈가 알려져 있고, 무엇보다 STT 라벨은 **텍스트**에 붙는다 — v4 는 오디오를 OpenAI Realtime 에 직접 보내는 구조라, 텍스트 라벨로 오디오를 막으려면 파이프라인을 STT→LLM→TTS(v2 형태)로 되돌려야 해서 실시간 음성 대 음성의 이점을 잃는다.

## 결론

1. **ElevenLabs 실시간 화자 구분: 불가** — 공식 문서 확정.
2. 구현 방향은 **로컬 임베딩 기반 게이트 관문**(독립 발화 차단) 우선, 겹침 제거는 별도 난이도의 후속 과제로 분리.
3. 착수 전 결정 필요: 사용자 등록 방식(첫 발화 자동 vs 명시 등록), 유사도 임계값, 오차단(가족 목소리 유사 등) 시 폴백 정책.

Sources: [ElevenLabs Speech-to-Text 공식 문서](https://elevenlabs.io/docs/overview/capabilities/speech-to-text), [Scribe v2 Realtime 소개](https://elevenlabs.io/realtime-speech-to-text)
