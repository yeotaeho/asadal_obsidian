---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-21
---

# 1.5 GA 실검증과 legacy 경로 제거

> [!summary] 한 줄 요약
> 기획서 V1.1 이 확정한 모델 4종 중 유일한 미지수였던 `gpt-realtime-1.5` 를 GA 엔드포인트에 실호출로 검증(**201 성공**)하고, GA 판정을 계열 전체로 넓히면서 폐지된 legacy 경로를 코드에서 걷어냈다. 기능: [[음성 선택과 대기시간 옵션]]

## 무슨 작업

aiortc 로 진짜 SDP offer 를 만들어 OpenAI 에 직접 세션 생성을 시도하는 프로브를 돌렸다. 세션만 열고 즉시 닫아 비용은 무시 수준이다.

| 대상 | 결과 |
|---|---|
| GA `/v1/realtime/calls` × 2.1 / 2.1-mini / 2 / **1.5** | 전부 **HTTP 201 + SDP answer** |
| GA × `gpt-realtime-2.0` | 404 model_not_found — 기획서 "2.0" 표기의 실제 ID 는 **`gpt-realtime-2`** |
| Legacy `/v1/realtime?model=` (conversation·transcription) | 모델 무관 **400** "The Realtime Beta API is no longer supported" |

검증 결과대로 GA 판정 접두사를 `gpt-realtime-2` → **`gpt-realtime`** 으로 넓히고, 성공할 수 없게 된 legacy 코드를 제거했다 — `webrtc_api._create_legacy_session`, v3·v4 라우터의 `_build_session_legacy`, `LegacyRealtimeSessionShape`/`validate_legacy_session_shape`. 세션 생성은 이제 모델 무관 GA config 를 항상 빌드한다.

## 왜

기획서 V1.1 이 모델 선택지 4종(2.1/2.1-mini/2.0/1.5)을 확정했는데, 구 판정으로는 사용자가 1.5 를 고르는 순간 폐지된 Beta 경로로 빠져 **통화 자체가 안 열렸다**. 갈림길은 "1.5 를 GA 로 태울 수 있는가 검증" vs "기획에서 1.5 제외 요청"이었고, 검증이 세션 1콜이면 끝나는 일이라 검증부터 했다.

판정 확장은 라우팅만의 문제가 아니었다. `server_webrtc` 의 LangGraph 스킵과 `realtime_speaker` 의 평면 session.update 생략 가드가 같은 판정 함수를 쓰므로, 1.5 가 GA 로 서빙되는데 판정이 legacy 면 **이중 응답 + 평면 update 거부 에러**가 났을 것이다. 판정 기준은 "모델 버전"이 아니라 "어느 API 로 서빙되는가"여야 한다.

## 브랜치 정리 — 잘못된 베이스 발견

작업 중 워킹트리가 `feat/voice-greeting`(인사말 브랜치)에 있는 걸 발견했다. 편집분을 patch 로 떠서 `feat-voice-selection-api` 로 이동, origin/main(인사말·요약기 가드 머지로 전진)을 브랜치에 머지했다. v4 라우터에서 대기시간 트랙 분기(HEAD)와 인사말 주입(main)이 충돌 — 양쪽을 다 살렸다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `709fada` | origin/main 머지 (인사말 + 요약기 가드 + 배포 설정) | — |
| `04205e5` | GA 판정 계열 확장 + legacy 분기 4종 제거 + 테스트 전환 | `constants.py`, `webrtc_api.py`, `voice_chat_v{3,4}_router.py`, `realtime_session_schemas.py`, 테스트 4파일 |

브랜치 `feat-voice-selection-api` (`caf8066`..`04205e5`).

## 검증

- 실검증 프로브 — GA 5모델 + legacy 2모드, 위 표.
- `pytest ai_voice/voice_tests tests` — **205 passed** (197 + shim 8). legacy 를 고정하던 테스트 9건은 계열 밖 표본(`gpt-4o-realtime-preview`)으로 전환하거나 새 계약(무설정 폴백 = 평면 update 생략)으로 갱신했다.
- Codex 리뷰(`codex exec review --commit 04205e5`) — P1 1건: "session_config 없는 v1/STT 호출자가 GA 검증에서 깨진다". **회귀 아님으로 판정** — 그 경로는 변경 전에도 ① 기본 모델(2.1)이면 같은 검증에서, ② 1.5 명시면 Beta 400 에서 이미 실패했다(위 실측). v1 소생은 기획 밖 별건이라 반영하지 않았다.

## 프론트 전달

`getRealtimeDialect` 의 `startsWith('gpt-realtime-2')` 를 서버와 같게 `'gpt-realtime'` 으로 넓혀야 한다. 어긋나면 프론트가 1.5 를 legacy 방언으로 처리해 헛수고를 한다.
