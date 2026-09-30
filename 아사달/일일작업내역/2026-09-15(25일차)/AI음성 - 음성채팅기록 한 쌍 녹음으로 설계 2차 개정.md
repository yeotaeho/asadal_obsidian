---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-15
---

# 음성채팅기록 한 쌍 녹음으로 설계 2차 개정

> [!summary] 한 줄 요약
> 기획서를 다시 확인해 녹음 범위를 **질문만 → 질문과 답변 한 쌍**으로 바꿨다. 답변은 다운링크 프록시가 브라우저로 내보낸 프레임을 담아 통화 때 목소리 그대로 남기고, dev compose 에 NAS 볼륨을 직접 적었다. 기능: [[음성채팅기록]]

## 무슨 작업

**dev compose 수정.** `/NAS/data_dev/voice_records:/mnt/voice_records` 볼륨과 `VOICE_RECORD_DIR` 환경변수를 넣었다. 경로를 `.env` 로 빼는 1차 안 대신 compose 에 직접 적었다. compose 는 gitignore 대상이라 서버마다 따로다. 원본은 CRLF 에 줄 끝마다 탭이 있어 로컬 YAML 파서가 원본부터 거부하므로, 줄 끝 공백을 떼고 파싱해 구조를 확인했다.

**설계서 2차 개정.** 기획서 V1.6 1쪽 ④와 3쪽이 재생 대상을 "질문한 내용과 AI가 답변한 녹음본" 으로 적고 있었다. 1차의 "답변은 재생 시 본체 TTS" 는 이 문구와 맞지 않고, Google TTS 로는 채팅 음성 설정(cedar·marin 등)을 낼 수 없으며 재생마다 목소리가 무작위다.

## 왜 가능해졌나

1차에서 답변 녹음을 기각한 사유는 "사용자가 어디까지 들었는지 계산해야 한다" 와 "끼어들기로 잘린다" 였다. **녹음 지점을 OpenAI 수신이 아니라 다운링크 프록시의 출력으로 옮기면** 둘 다 사라진다.

```
끼어들기·멈춤  → 서버가 다운링크 버퍼를 비움 → 안 나간 꼬리는 녹음에도 없음
일시정지      → 프록시가 무음을 내보냄 → 그 분기에서 on_frame 을 안 부름
답변 끝       → OpenAI 트랙이 무음을 계속 보내 소리로는 모름 → 재생 완료(버퍼 드레인) 이벤트로 자름
```

재료는 서버에 이미 있었다. `output_audio_buffer.started`·재생 완료 판정(`voice_playback.done`), 일시정지 플래그, 멈춤·끼어들기의 `track.clear()` 다.

## 설계 요점

| 항목 | 내용 |
|---|---|
| 파일 | `{세션8자}_{item_id}_qa.wav` 하나 — 질문 + 0.3초 무음 + 답변, 모노 24kHz |
| 답변 범위 | 질문이 닫힌 뒤부터 다음 발화·종료말·세션 종료까지의 재생 창들. 도구 호출 전 한마디 포함, RAG 대기 무음 제외 |
| 기록 시점 | 짝 맺음 + 답변 재생 완료 둘 다 찼을 때, 순서 무관 |
| 훅 | `AudioProxyTrack` 의 선택 콜백 `on_frame` — 다운링크·게이트 없는 업링크는 기반 `recv`, 게이트 트랙은 자기 `recv` |
| 프론트 정리 | `chatlog.htm` TTS 분기 삭제 → `<audio>` 하나, 목록 PHP `answer_text` 되돌림 |
| 비밀키 | `VOICE_RECORD_SECRET` 은 서버 `.env` — compose `env_file` 이 넣음. `openssl rand -hex 32` 로 새로 만들고 바꾸지 않음 |

## 어디

| 대상 | 내용 |
|---|---|
| `dev/docker-compose.yml` | 볼륨·환경변수 추가, 백업 `dev/backup_20260915_before_voicerecord/` |
| 설계서 (로컬, gitignore) | `docs/superpowers/specs/2026-09-09-voice-record-playback-design.md` — §0 개정 이력, §5.2~5.4 프레임·질문·답변 구간, §5.8 비밀키·마운트 |
| 결정 기록 | 아키텍처 4행(TTS·answer_text 행 뒤집음), 인프라 2행, 보류 2행 — [[AI음성 - 결정 기록]] |

## 검증

- 기획서 텍스트를 슬라이드 XML 에서 UTF-8 로 추출해 재생 범위 문구 확인
- compose 구조를 줄 끝 공백 제거 후 YAML 파싱으로 확인. 서버에서는 `docker compose config --quiet` 로 재확인 필요
- 다운링크 근거는 현행 코드로 확인 — `server_webrtc.py:1028` 생성, `:1996` 재생 이벤트, `:3074` 버퍼 비우기, `audio_tracks.py:460` 일시정지
- 미검증 가정 두 개는 첫 dev 통화에서 확인: `audio_start_ms` 기준 일치, 도구 턴에서 짝 맺음이 답변 소리보다 먼저 오는 경우
