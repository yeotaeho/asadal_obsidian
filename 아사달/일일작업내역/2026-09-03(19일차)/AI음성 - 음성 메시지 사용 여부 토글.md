---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-03
---

# 음성 메시지 사용 여부 토글

> [!summary] 한 줄 요약
> 기획서 v2.0.4 의 "음성 메시지 설정 사용/미사용"을 첫 인사말·종료 안내말 **각각의 컬럼 2개**로 본체·음성 서비스에 배선했다. 프론트 키 이름만 담당자 출근(다음주 월요일) 후 별칭으로 붙이면 된다. 기능: [[음성채팅 안내문 온오프와 재생 제어]]

## 무슨 작업

기획서 3장(안내문 온오프·미리듣기·일시정지/멈춤/종료 버튼) 중 가장 작은 온오프부터 했다. 인사말·종료말 때 만든 배관 위에 플래그 컬럼만 얹는 구조라 두 레포 합쳐 **본문 변경 10줄**이다.

본체는 `chatbot_config` 에 `use_voice_first_msg`·`use_voice_end_msg`(`TEXT DEFAULT 'Y'`)를 추가하고, 요청 모델 필드 2개를 기존 `use_voice_wait` 와 같은 `_normalize_yn` 검증기(on/off → Y/N)에 걸었다. 조회 SELECT 목록에도 넣었다. 기존 봇 DB 는 자동 `ALTER TABLE` 이라 마이그레이션이 없다.

음성 서비스는 `load_voice_messages` 가 이미 `SELECT *` 로 행을 읽으므로, 플래그가 `'N'` 이면 빈 문자열을 돌려주는 `_message()` 헬퍼 하나를 넣었다. v4 라우터는 원래 `if greeting:` 조건으로 metadata 를 주입하므로 **발화와 말풍선(트랜스크립트)이 자동으로 함께 빠진다.** 종료말이 비면 `voice_end.request` 에 곧바로 `voice_end.done` 을 보내는 기존 경로를 탄다.

플래그를 하나로 묶을지 둘로 나눌지는 사용자가 **2개(첫·종료 각각)** 로 결정했다. 기획서 시안에 라디오가 두 메시지에 각각 그려져 있다.

## 왜

기획서 v2.0.4(음성채팅 안내문 온오프) 수령. 프론트 담당자가 다음주 월요일에 출근하므로 그 전에 백엔드가 받을 값을 준비해 두기로 했다. 키 이름은 `sys_vioce` 사례처럼 프론트가 정한 이름을 본체 `AliasChoices` 로 흡수하면 된다.

## 어디

| 커밋 | 레포 | 내용 | 파일 |
|---|---|---|---|
| `ae1e60a` | Chatty_Project `feat/voice-msg-toggle` | 컬럼 2개 + 요청 필드 2개 + SELECT | `utils/chatbot_config.py` `models/request_models.py` `utils/test_voice_msg_toggle.py` |
| `28eb71a` | 〃 | 테스트 import 스텁을 적재 후 복원 (Codex P2) | `utils/test_voice_msg_toggle.py` |
| `ddda054` | AI_Pro_VOICE `feat/voice-msg-toggle` | 미사용 플래그면 빈 문구 반환 | `ai_voice/services/voice_chat/greeting.py` `voice_tests/test_voice_greeting.py` |
| `a17dc42` | 〃 | 종료말 미사용 봇도 종료 시 진행 중 답변 즉시 중단 (`_stop_current_speech` 한 줄) | `ai_voice/services/voice_chat/server_webrtc.py` `voice_tests/test_voice_greeting.py` |

둘 다 main 기준 브랜치이고 아직 푸시하지 않았다.

## 검증

- TDD — 테스트 먼저 작성해 실패(AttributeError / no such column)를 확인한 뒤 구현.
- AI_Pro_VOICE `pytest ai_voice/voice_tests tests` — **252 passed** (신규 4건: 첫 인사말만 미사용 · 종료말만 미사용 · 컬럼 부재·NULL 은 사용 · 미사용이면 기본 문구 폴백도 안 함).
- 본체 `utils/test_voice_msg_toggle.py` — **2 passed**. 본체는 로컬에 `config.py`·DB 드라이버가 없어 대상 모듈 2개를 파일 경로로 적재하고 외부 의존을 적재 동안만 스텁으로 대체한다. 배포 직전 스키마(플래그만 없는 실제 봇 DB 형태)에서 자동 ALTER 후 기존 값 보존과 기본값 `'Y'` 를 확인.
- Codex 리뷰 — 음성 클린. 본체 P2 1건("`db` 패키지를 sys.modules 에서 영구 교체해 다른 테스트의 lazy import 오염") 반영 후 재리뷰. 재리뷰 2회 모두 "actionable regression 없음"으로 통과.
- 후속 `a17dc42` — 미사용 봇에서 답변 도중 종료 시 `response.cancel`·`output_audio_buffer.clear`·큐 비움 후 `voice_end.done` 을 고정하는 테스트 추가. 전체 **253 passed**, Codex 리뷰 클린. 미사용일 때 종료 흐름은 클릭 즉시 마이크 off → 서버 `voice_end.done` 즉시 응답 → 프론트 800ms 유예 뒤 disconnect 다.

## 배운 것

SQLite 는 `ALTER TABLE ADD COLUMN` 에 PRIMARY KEY 나 `CURRENT_TIMESTAMP` 같은 비상수 기본값을 못 붙인다. `check_and_add_missing_columns` 는 예외를 잡고 루프를 통째로 멈추므로, 테스트 픽스처를 "컬럼 하나짜리 테이블"로 만들면 `id` 에서 죽어 아무 컬럼도 안 붙는다. 실제 봇 DB 처럼 **이번 배포 직전 스키마 전체**를 만들어야 정직한 테스트가 된다.
