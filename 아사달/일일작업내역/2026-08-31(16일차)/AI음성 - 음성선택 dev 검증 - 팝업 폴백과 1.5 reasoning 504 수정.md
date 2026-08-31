---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-31
---

# 음성선택 dev 검증 - 팝업 폴백과 1.5 reasoning 504 수정

> [!summary] 한 줄 요약
> dev 팝업에서 모델·음성 선택이 payload 에 안 실리던 원인 2건(캐시·VoiceSettings 우선 분기)을 잡아 **선택 반영 정상 동작을 확인**했고, 1.5 만 통화가 전멸하던 원인을 **reasoning 필드의 GA 504** 로 확정해 수정했다. 기능: [[음성 선택과 대기시간 옵션]]

## 무슨 작업

dev 팝업(`dev.han.kr/chat/popup/index.htm`)에서 프론트 담당의 메뉴가 쓴 `VOICE_RUNTIME_CONFIG` 값이 세션 payload 에 반영되는지 실검증했다. 안 실리던 원인은 두 겹이었다.

첫째는 **브라우저 캐시**. 모듈이 `Cache-Control: public, max-age=604800`(7일)로 서빙되는데 script 태그에 캐시 버스터가 없어, 파일을 갈아끼워도 브라우저가 구버전을 실행했다. DevTools Disable cache 로 해소하고 신버전 실행을 `connect.toString()` 검사로 확인했다.

둘째는 **모듈의 VoiceSettings 우선 분기**. 팝업에도 `VoiceSettings` 가 로드돼 있었고, `get()` 은 값이 비어도 항상 코드 기본값(2.1/cedar)을 돌려줘 `VOICE_RUNTIME_CONFIG` 를 영영 안 읽었다. `connect()` 를 존재 여부 분기에서 **값 유무 기준 병합**(`_user` > `VOICE_RUNTIME_CONFIG` > `get()` > 기존값)으로 교체했다. 이후 **모델·음성 선택이 payload 에 정상 반영**됨을 사용자가 확인했다.

이어서 **1.5 만 인사말·음성·음성채팅 전멸** 증상을 추적했다. 세션 config 필드를 이분 탐색으로 실호출한 결과, **`reasoning: {effort: "low"}` 하나만 넣어도 1.5 는 400 이 아니라 504 Gateway time-out** 으로 세션 생성이 죽었다(2.x 는 전부 201). `supports_realtime_reasoning` 판정을 constants 에 두고 v3·v4 세션 빌더가 1.5 에서만 reasoning 을 떼게 했다.

```
minimal            HTTP 201            +turn_detection    HTTP 201
full(배포본)        HTTP 504            +transcription     HTTP 201
full-no-tools      HTTP 504            +noise_reduction   HTTP 201
full-short-inst    HTTP 504            +reasoning         HTTP 504  ← 범인
```

## 왜

음성선택 기능의 dev 배포 후 수동 검증 단계다. 팝업 메뉴 계약(`VOICE_RUNTIME_CONFIG.voice_ai_model/voice_tts`)은 프론트 담당이 만든 것이라 모듈이 흡수해야 했고, 1.5 는 기획서 4모델에 포함돼 있어 통화 불능이면 기능 미완성이다. 8-21 실검증(1.5 GA 201)은 **최소 config** 기준이라 reasoning 이 든 운영 config 의 504 를 놓쳤다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `069eab4` | 1.5 세션에서 reasoning 필드 제거 (feat-voice-selection-api, 푸시 완료) | `ai_voice/constants.py` · `voice_chat_v{3,4}_router.py` · `voice_tests/test_reasoning_field_gate.py` |
| (무커밋) | connect() 병합 로직 교체 — 파일질라 배포 대상 | 프론트 `webrtc_voice_module.js` |

## 검증

- 회귀 스위트 **220 passed** (신규 reasoning 게이트 테스트 포함).
- 패치된 빌더 그대로 실세션 생성 — **1.5 · 2.1 둘 다 201 SDP answer**.
- 프론트 병합 로직 node 하네스 6케이스 통과(팝업 선택 / \_user 우선 / 기본 폴백 / 스토어 부재 / 숫자 id 가드 / 무설정 유지).
- Codex CLI 리뷰(커밋 069eab4) — 지적 없음, 승인.

## 남은 것

`feat-voice-selection-api` → dev 재머지(`app restart`) 후 팝업에서 1.5 통화 재확인. 프론트 담당에게 모듈 script 태그 캐시 버스터(`?v=<?=time();?>`) 요청.
