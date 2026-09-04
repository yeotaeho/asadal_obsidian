---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-03
---

# 일시정지·멈춤 재생 제어 구현

> [!summary] 한 줄 요약
> 음성채팅 답변의 일시정지·재개·멈춤을 **서버 다운링크 지터버퍼 버퍼링**으로 구현했다. 브라우저는 DataChannel 메시지 3종만 보내고, 재생이 실제로 끝나면 서버가 `voice_playback.done` 을 보낸다. Codex 3회전에서 잡힌 결함 4건(재개 백로그 폐기·소스 종료 시 조기 완료·종료말 조기 완료·인플라이트 프레임)을 반영했다. 기능: [[음성채팅 안내문 온오프와 재생 제어]]

## 무슨 작업

OpenAI Realtime 에도 WebRTC 에도 일시정지가 없다. 그래서 OpenAI→브라우저 사이에 앉은 서버 프록시 트랙(`AudioProxyTrack`, 지터버퍼)이 프레임을 쥐고 침묵을 내보내는 방식으로 "진짜 일시정지"를 만들었다. 재개하면 멈춘 20ms 프레임부터 그대로 이어진다.

```
브라우저 ──voice_playback.pause──▶ 매니저 on_message ──▶ 브릿지.control_playback
                                                        └─▶ 프록시.pause()  : recv() 가 버퍼를 안 꺼내고 침묵 송출
                                                                              리더는 상한 960ms 대신 3분 상한으로 적재
브라우저 ──voice_playback.resume─▶ …               └─▶ 프록시.resume() : 멈춘 프레임부터 1배속
브라우저 ──voice_playback.stop───▶ …               └─▶ RAG 취소 + _stop_current_speech
                                                        (response.cancel · output_audio_buffer.clear · 큐 폐기)
                                                        + 프록시.clear()+resume()
서버 ──voice_playback.done──▶ 브라우저 : OpenAI stopped/cleared 뒤 프록시 버퍼가 빈 시점
```

## 결함으로 확인된 것 (Codex 3회전)

| 지적 | 왜 문제였나 | 수정 |
|---|---|---|
| 재개 직후 백로그 폐기 (P1) | 재개하면 평시 상한(960ms)이 다시 적용돼, OpenAI 가 아직 보내는 중이면 새 프레임 하나에 붙들린 프레임 하나가 버려져 **답변 절반이 듬성듬성 사라짐** | 백로그가 상한 아래로 내려갈 때까지 폐기를 멈추는 `_resume_backlog` 플래그 |
| 소스 종료 시 조기 완료 (P2) | `wait_drained` 가 리더 종료를 완료 조건으로 봐서 붙들린 프레임이 남아도 done 전송 | 소스 종료는 완료 조건에서 제외 |
| 종료말 조기 완료 (P1) | `voice_end.done` 을 OpenAI `stopped` 직후에 보내 서버 버퍼 잔여분이 잘림 — 프론트 800ms 유예에 기대던 부분 | 종료말 완료도 `_send_after_drain` 경유 |
| 인플라이트 프레임 (P2) | `stopped` 는 데이터채널, 마지막 프레임은 RTP 로 따로 와서 버퍼가 순간 비어 보일 수 있음 | 빈 상태 100ms 뒤 재확인(디바운스) |

끼어들기와 종료말 시작 시에도 서버 버퍼 꼬리를 비우게 했다. 기존에는 OpenAI 만 자기 버퍼를 비워 서버 꼬리 최대 ~1초가 이어서 들렸다.

## 왜

기획서 v2.0.4 슬라이드 3. "일시정지 후 재개하면 잘릴 수 있다"고 보고했던 제약은 상한을 일시정지 중 풀면서 사라졌다. 클릭 후 잔여음은 브라우저 지터버퍼 0.1~0.3초뿐이고 프론트가 `audio.muted` 로 지울 수 있다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `145f5c6` | 프록시 pause/resume/clear/wait_drained + 브릿지 control_playback·done 통지 + 매니저 라우팅·등록 | `v4/audio_tracks.py` `server_webrtc.py` `test_downlink_pause.py` `test_voice_playback_control.py` |
| `42024af` | 재개 백로그 보존 · 소스 종료 후 드레인 대기 · 종료 중 done 억제 | 〃 + `__new__` 브릿지 테스트 2곳 필드 추가 |
| `76f6d5f` | 종료말 완료 드레인 대기 · 디바운스 | 〃 |
| `bd3a53c` | 사용자 발화 감지 시 서버 다운링크 버퍼를 **조건 없이** 비움 — 재개 백로그 재생 중 발화하면 옛 답변 뒤에 새 답변이 붙어 밀리던 결함(사용자 지적) | `server_webrtc.py` `test_voice_playback_control.py` |
| `c6e6584` | 멈춤 시 걸려 있던 드레인 대기를 거둬 done 중복 방지 (Codex P2) | 〃 |
| `26d0b11` | 재생 중(일시정지 없이) 멈춤 경로 테스트 명시 | `test_voice_playback_control.py` |
| `c6a38f8` | **dev 실증 결함** — `wait_drained` 가 "버퍼 공백"을 기다려 영영 안 끝남(OpenAI 트랙은 응답 뒤에도 침묵 프레임을 계속 보내 지터버퍼가 늘 차 있음) → `voice_end.done`·`voice_playback.done` 미전송, 종료가 6초 타임아웃. 재생 카운터 기반 "호출 시점 백로그 재생 완료"로 교정 | `v4/audio_tracks.py` `test_downlink_pause.py` |
| `f523d06` · `71580fb` | 유예 중 도착한 늦은 프레임은 한 번만 더 기다림, 카운터 초과분 오판 제거 (Codex P1 2건) | 〃 |
| `abf7a21` | **dev 로그로 확정한 진짜 원인** — 매니저가 다운링크 프록시를 **v3 사본** `AudioProxyTrack` 으로 만들고 있었고 재생 제어 API 는 v4 사본에만 있었다. `_flush_downlink` 의 `clear()` 가 AttributeError → 사용자 발화 처리·종료 안내말·멈춤 전부 중단. 임포트를 v4 사본으로 교체(v3 파일 무수정) + 프록시 클래스 계약 테스트 | `server_webrtc.py` `test_voice_playback_control.py` |

브랜치 `feat/voice-playback-control`. 처음엔 origin/main 최신(`8261d3f`)에서 땄다가 dev PR 에서 `docker-compose.yml` 충돌이 나서 **팀장 배포 커밋 4개 이전의 main(`80d0f7b`)으로 리베이스**해 다시 푸시했다(해시가 모두 바뀜). dev PR 열림, 프론트 연결·실기기 검증이 남았다.

## 검증

- TDD — 트랙 7건·브릿지 9건 모두 실패 확인 후 구현. 전체 **262 passed**.
- Codex 리뷰 3회전 — P1 2건·P2 2건 전부 반영, 최종 "actionable regression 없음".
- 첫 커밋은 전체 스위트 8건 실패 상태로 들어갔다(체인 커맨드가 결과를 안 보고 커밋). 이후 커밋부터 실패 시 커밋을 막는 가드를 넣었다.
- 사용자 지적("재개 후 재생 중 말하면 오디오가 밀리지 않나")이 맞았다. 재개 백로그는 OpenAI 쪽 생성·송신이 끝난 오디오라 `interrupt_response` 도 서버 끼어들기 판정도 잡지 못했다. `speech_started` 에서 무조건 비우도록 고치고 테스트로 고정했다. 최종 **264 passed**, Codex 5회전 통과.

## 배운 것

일시정지의 정밀도는 서버 프레임 경계(20ms)이고, 통제 못 하는 건 이미 브라우저로 넘어간 0.1~0.3초뿐이다. 서버 버퍼의 960ms 는 손실이 아니라 재개 때 이어질 구간이다. 그리고 "완료" 신호는 층마다 다르다 — OpenAI 생성 완료(`response.done`) → OpenAI 송신 완료(`output_audio_buffer.stopped`) → 서버 버퍼 소진 → 브라우저 재생 완료. 프론트에 알려야 하는 건 세 번째다.

**dev 첫 테스트에서 드러난 것(추가).** 단위 테스트의 가짜 소스는 프레임을 보내다 멈추지만, 실제 OpenAI 다운링크는 응답이 끝나도 침묵 프레임을 실시간으로 계속 보낸다. 그래서 "버퍼가 빈다"는 조건은 실서버에서 영영 참이 되지 않았고 종료가 6초 타임아웃까지 걸렸다. 지터버퍼 완료 판정은 공백이 아니라 **재생 카운터**(호출 시점 백로그만큼 재생됨)로 해야 한다. 소스 모델이 실제와 다르면 테스트가 통과해도 틀린다 — 지속 소스 테스트(`_ContinuousSource`)를 추가해 고정했다. 전체 **267 passed**. 사용자가 보고한 "처음부터 마이크 음소거·발화 무반응"은 코드에서 원인을 못 찾아 서버 로그·브라우저 콘솔 대기 중.

**로그가 답을 줬다(추가).** 사용자 보고 "발화 무반응·종료 안 됨·안내말 없음"의 진짜 원인은 `docker logs` 의 한 줄 — `replaceTrack ... track=<...v3.audio_tracks.AudioProxyTrack>` 와 `AttributeError: 'AudioProxyTrack' object has no attribute 'wait_drained'` 였다. 런타임 플로우 문서는 다운링크가 v4 기반 클래스를 쓴다고 서술했지만 실제 임포트는 v3 사본이었다. 브릿지 테스트는 가짜 트랙, 트랙 테스트는 v4 클래스를 직접 써서 이 어긋남을 못 봤다. **"매니저가 만드는 그 객체가 그 API 를 갖는가"** 를 묻는 테스트가 빠져 있었고 이제 추가했다. 문서와 코드가 어긋날 수 있으니 실제 생성 지점의 클래스를 확인해야 한다. 전체 **268 passed**, Codex 클린.
