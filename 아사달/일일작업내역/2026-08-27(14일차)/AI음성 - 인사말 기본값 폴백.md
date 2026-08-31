---
tags:
  - 아사달
  - 일일작업
  - AI음성
created: 2026-08-27
---

# 인사말 기본값 폴백

> [!summary] 한 줄 요약
> test.han.kr 모바일에서 인사말이 안 나오던 원인이 미저장 봇임을 규명하고, greeting.py에 프론트 기본값과 같은 문구의 폴백을 넣어 간극을 제거했다. 기능: [[음성채팅 첫 인사말]]

## 무슨 작업

test.han.kr에서 "데스크톱 O·모바일 X" 증상을 추적했다. 프론트 JS 13종·CSS 10종을 test↔dev md5 대조해 **전부 바이트 동일**임을 확인, 파일 차이를 소거한 끝에 원인은 **기기별 쿠키로 선택된 봇이 서로 달랐고 그 봇이 설정 미저장**이었던 것으로 확정됐다.

이 과정에서 설정 구조를 규명했다 — 저장버튼이 프론트 MySQL(`chatbot_setup_detail`)과 본체 SQLite(`chatbot_config.db`) **양쪽에 따로 쓰는 병렬 이중 저장**이고, 서로 폴백하지 않는다. 그런데 프론트 `chatbotConfig.php`는 미저장 봇에도 기본 인사말을 화면에 표시해서, "보이는데 발화는 안 되는" 구조적 간극이 있었다.

팀장 승인 후 `load_voice_greeting()`이 저장값 부재 시(파일·컬럼·값·식별자 전부) `DEFAULT_GREETING`("안녕하세요. 무엇을 도와드릴까요?")을 반환하게 바꿨다. 문구는 chatbotConfig.php 기본값과 동일하고, 일치 여부를 지키는 테스트를 추가했다.

## 왜

미저장 봇은 침묵하는 것이 기존 설계였는데, 관리자 화면이 기본 인사말을 보여주고 있어 사용자 기대와 어긋났다. 이번 test.han.kr 혼선이 그 간극의 실제 사례다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `82ab59a` | 기본 인사말 폴백 + 테스트 갱신 | `ai_voice/services/voice_chat/greeting.py` `test_voice_greeting.py` |
| `2b0775e` | DEPLOY.md "조용히 비활성" 서술 정정 | `docs/DEPLOY.md` |

브랜치 `feat/voice-greeting-fallback` (main 기준) — dev 머지 테스트 후 main PR 예정.

## 검증

- `pytest ai_voice/voice_tests` — **194 passed** (폴백 4경로 + 프론트 문구 일치 테스트 신규).
- Codex CLI 리뷰(`codex exec review --commit 82ab59a`) — "No actionable correctness regression". 리뷰가 짚은 DEPLOY.md 드리프트는 `2b0775e`로 정정.
