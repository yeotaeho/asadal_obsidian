---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-03
---

# 오디오 탭 복원과 업링크 무변경 확인

> [!summary] 한 줄 요약
> "못 알아듣는다 / 발화가 끝나도 발화 중으로 본다" 보고를 귀로 대조할 수 있게, 본체에서 제거됐던 구간별 오디오 탭을 이 레포 v4 경로에 **플래그 파일 내용 on/off** 방식으로 되살렸다. 이번 작업(재생 제어)이 업링크 게이트를 바꾸지 않았음을 diff 로 확인했다. 기능: [[음성채팅 안내문 온오프와 재생 제어]]

## 무슨 작업

### 업링크는 손대지 않았다

`origin/dev` 대비 이 브랜치의 `v4/audio_tracks.py` diff 에 `NoiseFiltered`·`Uplink`·게이트(`_gate`·speech·silence·barge·force_flush) 코드 변경은 **0건**이다. 기반 클래스에 들어간 것은 재생 카운터(`_played_frames`)와 일시정지 분기(`_paused`·`_resume_backlog`)뿐이며, 업링크 두 클래스는 이를 상속하지만 일시정지가 걸리지 않는 한 평시와 동일하게 동작한다. 오디오 경로에서 실제로 달라진 것은 다운링크 셋이다 — 프록시 클래스가 v3 사본 → v4 사본(프리버퍼 1.0초 → 0.2초, 60초 무프레임 좀비 가드, `PROXY-DOWN` 태그), 사용자 발화 감지 시 다운링크 버퍼 비우기, 재생 완료 통지.

### 오디오 탭

본체 커밋 `28fd33c`(구간별 WAV 덤프)·`5ed46ab`(/tmp 플래그)·`ca83b70`(활성 로그)이 `8cb886a` 에서 제거됐던 것을 이식했다. 달라진 점은 스위치와 경로다.

| 항목 | 본체 원본 | 이번 이식 |
|---|---|---|
| 켜기/끄기 | `touch /tmp/voice_audio_tap.on` / `rm` (파일 존재) | `echo on/off > data/logs/voice_audio_tap` (파일 **내용**) |
| 산출 경로 | `/tmp/voice_tap` (systemd 호스트) | `/data/logs/voice_tap` (컨테이너, 호스트 `./data/logs` 마운트) |
| 구간 | A 브라우저 원음 · B 게이트 방출 | A · B · **C OpenAI 수신**(다운링크 기반 클래스) |
| 기록 시점 | 지터버퍼에서 꺼낼 때 | **버퍼에 넣기 전**(Codex P2) — 상한 초과로 버려지는 프레임까지 담는다 |

`echo on` 뒤 세션 생성부터 적용되고(재시작 불필요), 활성 시 `[V4 TAP] 오디오 탭 활성` 경고 로그가 찍힌다. 기록기는 어떤 실패든 삼켜 통화를 죽이지 않고, 파일당 10분 상한이다.

```
data/logs/voice_tap/20260903_183012_fc865f99_A_from_browser.wav   ← 브라우저가 보낸 원음(폐기분 포함)
data/logs/voice_tap/20260903_183012_fc865f99_B_to_openai.wav      ← 게이트가 OpenAI 에 실제로 내보낸 것
data/logs/voice_tap/20260903_183012_fc865f99_C_from_openai.wav    ← OpenAI 가 보내온 것
```

## 왜

사용자 보고 "말을 잘 못 알아듣거나 발화가 끝나도 발화 중이라고 판단". 로그 수치로는 체감과 대조가 안 되고, 본체에서 같은 목적으로 만든 탭이 제거된 상태였다. 스위치를 파일 삭제가 아닌 on/off 로 해달라는 요청을 반영했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `0a261c8` | 탭 이식 + 테스트 9건 + DEPLOY.md 절 | `v4/audio_tracks.py` `test_v4_audio_tap.py` `docs/DEPLOY.md` |
| `c431fd5` | 소스 탭을 폐기 전에 기록 (Codex P2) | 〃 |

브랜치 `feat/voice-audio-tap` — `feat/voice-playback-control`(`abf7a21`) 위에 쌓았다. 같은 파일의 같은 함수를 고치므로 main 기준으로 따면 dev 머지에서 충돌한다.

## 검증

- TDD — 10건 실패 확인 후 구현. 전체 **278 passed**.
- Codex 리뷰 2회전 — P2 1건(폐기 전 기록) 반영 후 클린.
- dev 실측은 아직 — 머지 후 `echo on` 으로 켜고 "못 알아듣는" 통화를 재현해 A/B 를 들어야 한다.

## 실수 기록

탭 브랜치를 만들 때 셸 작업 디렉터리가 본체(Chatty_Project)에 남아 있어 본체에 `feat/voice-audio-tap` 브랜치를 잘못 만들었다. 커밋·푸시 전에 발견해 stash 로 사용자의 미커밋 `requirements.txt` 변경을 보존한 채 원래 브랜치로 되돌리고 삭제했다. 여러 레포를 오갈 때는 `cd` 를 명령마다 명시한다.
