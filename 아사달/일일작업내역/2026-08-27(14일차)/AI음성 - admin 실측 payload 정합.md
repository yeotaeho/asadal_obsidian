---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-27
---

# admin 실측 payload 정합

> [!summary] 한 줄 요약
> admin 프론트가 실제 보내는 payload 필드명·값이 계약 문서와 전부 달라, 본체가 **별칭+정규화로 흡수**하도록 고치고 메뉴 노출을 조회 API 단일 출처로 확정했다. 기능: [[음성 선택과 대기시간 옵션]]

## 무슨 작업

프론트 담당의 실전송 payload(DevTools 스크린샷)를 대조하니 5필드 중 4개가 다른 이름이었고 값 형식도 달랐다 — `voice_ai_model`, `voice_tts: "Cedar"`(대문자), `voice_wait_use: "on"`, `sys_vioce: "on"`(**오타**). 프론트 수정 요청 대신 본체 `ChatbotConfigRequest` 에 `AliasChoices` 별칭과 `field_validator` 정규화를 넣어 표준값으로 저장되게 했다.

```
voice_ai_model → voice_model   ("gpt-realtime-2.0" 은 "gpt-realtime-2" 로 교정)
voice_tts      → voice_name    ("Cedar" → "cedar")
voice_wait_use → use_voice_wait ("on"/"off" → "Y"/"N", 빈 문자열 wait_time → 미저장)
sys_vioce·sys_voice → use_voice_config
```

부수 확정 — config_chat.htm 이 on/off 를 payload 로 보내는 것이 실측되면서, **봇별 메뉴 노출의 단일 출처가 chatbot_config(use_voice_config)** 로 정해졌다. index2.js 의 `loadVoiceSettings` 가 조회 응답의 `use_voice_config==='N'` 이면 메뉴를 숨기도록 `voiceMenuEnabled` 플래그를 연결했다(4 hunks). PHP `$sys_voice` 변수는 선택 사항으로 강등.

## 왜

Pydantic 은 미지식 필드를 조용히 버리므로, 정합 없이는 관리자가 저장해도 **모든 음성 설정이 증발**했을 것이다. 인사말(voice_first_msg)이 겪었던 것과 같은 유형의 사일런트 유실이다.

## 어디

| 커밋 | 내용 | 저장소 |
|---|---|---|
| `3face66` | 별칭 5종 + 정규화 4종 | 본체 worktree `feat/voice-selection-config` |
| `f85b979` | Y/N 관용 정책(이상값 미저장) 사유 주석 — Codex P2 기각 판정 박제 | 〃 |
| `59146ca` | 계약 문서에 실측 별칭 수용표·노출 단일 출처 반영 | AI_Pro_VOICE `feat-voice-selection-api` |
| — | index2.js `voiceMenuEnabled` 배선 (git 없음, .bak 보존) | 프론트 |

## 검증

- 실측 payload 3종(음성설정 페이지·채팅화면설정 페이지·표준명) 시나리오로 pydantic 검증 — **전부 통과** ("Cedar"→cedar, "on"→Y, "2.0"→2, ""→미저장 확인).
- `node --check index2.js` 클린. AI_Pro_VOICE 전체 스위트 **217 passed**.
- Codex 리뷰 P2 1건("이상값을 422 로 거부하라") — **기각**: 공용 혼합 payload 엔드포인트의 관용 계약과 충돌, 읽기측 화이트리스트가 이중 방어. 사유를 코드 주석으로 남김.
