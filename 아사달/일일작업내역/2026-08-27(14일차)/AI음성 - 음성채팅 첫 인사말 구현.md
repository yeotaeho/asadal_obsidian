---
tags:
  - 아사달
  - 일일작업
  - AI음성
created: 2026-08-27
---

# 음성채팅 첫 인사말 구현

> [!summary] 한 줄 요약
> 관리자 설정 인사말을 v4 세션 시작 시 자동 발화 + 말풍선 표시하는 기능을 3개 레포에 걸쳐 구현하고 dev에서 **E2E 검증 완료**. 기능: [[음성채팅 첫 인사말]]

## 무슨 작업

관리자 페이지 [채팅화면설정 > 음성채팅 첫메시지] 문구가 마이크 클릭 즉시 AI 발화 + 봇 말풍선으로 나온다. 본체 저장(`/api/save_chatbot_config`) → 봇별 SQLite(`chatbot_config.db`) → 컨테이너 읽기 전용 마운트 → 세션 metadata → OpenAI 응답 큐 발화 + DataChannel 이벤트 순의 체인이다.

발화와 별개로 말풍선은 발화 트랜스크립트 경로 하나로 렌더한다 — 전용 이벤트와 이중 렌더되던 중복 말풍선을 브릿지 no-op으로 제거했다. 음성 시작 세션에서는 텍스트용 "채팅 첫 메시지"를 숨겼다(`startedByVoice`).

## 왜

팀장 지시 — 음성 채팅 진입 시 AI가 먼저 인사하는 UX. 문구는 봇마다 달라 관리자 설정으로 분리했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `f71b33d` (본체, PR #891) | voice_first_msg 저장·조회 배선 | `models/request_models.py` `utils/chatbot_config.py` |
| `fafa176` (AI_Pro_VOICE, PR #7) | 인사말 로더 + 세션 발화 | `greeting.py` `voice_chat_v4_router.py` `server_webrtc.py` |
| `5aa3307` (AI_Pro_VOICE, PR #9) | 말풍선용 voice_greeting.message 이벤트 | `server_webrtc.py` `test_voice_greeting.py` |
| 레포 미커밋 (서버 작업본) | 이벤트 수신·렌더 + 중복 제거 + 첫 메시지 분리 | `webrtc_voice_module.js` `voice_chat_bridge.js` `index2.js` `config.php` |

## 검증

- `pytest ai_voice/voice_tests/test_voice_greeting.py` — **8 passed** (HTML 정제·SQLite 경로·소스 계약).
- dev.han.kr 실측 — 마이크 클릭 → 인사말 발화 + 말풍선 1개, 서버 로그 `인사말 말풍선 이벤트 전송(N자)` 확인.
- 디버깅 확정 사항 — env 경로는 환경 페어(dev↔`/NAS/data_dev`, 운영↔`/NAS/data`)로 일치해야 하고, `.env` 반영은 컨테이너 재생성 필요. 상세는 [[리버스 프록시 프리픽스 라우팅과 dev·운영 컨테이너 분리]] 참고.
