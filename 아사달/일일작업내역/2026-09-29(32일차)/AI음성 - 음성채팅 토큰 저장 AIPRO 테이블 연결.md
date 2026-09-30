---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-29
---

# 음성채팅 토큰 저장 AIPRO 테이블 연결

> [!summary] 한 줄 요약
> 짝이 맺어진 v4 문답을 계정 MariaDB 의 AIPRO 메시지 1행과 응답별 사용량 행으로 파이썬이 직접 저장하게 했다. 테스트 main **442개**·dev **464개** 통과. 기능: [[음성채팅 토큰 저장]]

## 무슨 작업

DB 담당자의 스키마 정의서(`token_save.md`)를 검토했다. 본체는 영상 생성만 `feature/token-save` 로 구현돼 dev 에 머지돼 있고, 이미지는 넣었다가 되돌렸으며, 텍스트 채팅은 아직 프론트 → PHP 경로다. 영상 코드는 첫 저장 때 `CREATE TABLE IF NOT EXISTS` 를 스스로 실행한다.

음성은 새 모듈 `usage_log.py` 가 문답 1건을 한 트랜잭션으로 쓴다. 응답별 사용량은 `_emit_assistant_text` 에서 합치지 않고 목록으로 들고, `_send_turn_record` 가 브라우저로는 합계를, 저장소로는 목록을 넘긴다. 연결은 저장마다 열고, 테이블 없음(1146)은 조용히 건너뛴다.

## 왜

담당자가 "각자 로직에서 파이썬에서 토큰 저장" 을 요청했다. 사용자는 원래 기록되지 않던 과금(인사말·종료말·취소된 응답·입력 전사·VITO·임베딩)은 무시하라고 했다. 담당자는 `job_type` 값이 예시라며 `voice_chat` 을 허용했다.

규약상 USAGE 는 호출 1회당 1행이고 `total_tokens` 를 직접 더하면 안 된다. 그래서 브라우저 기록용 합계와 별개로 응답별 원문을 끝까지 운반했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `8782801` | 토큰 저장 (main 기준) | `v4/usage_log.py` · `server_webrtc.py` · `v4/turn_log.py` · `voice_chat_v4_router.py` · `config.py` · `requirements.txt` · `.env.example` · `docs/DEPLOY.md` |
| `918457a` | dev cherry-pick. 충돌은 dev 전용 화자 판정 줄과 나란히 추가된 것뿐이라 양쪽 유지 | `server_webrtc.py` · `test_session_generation_guard.py` |

## 검증

- 새 테스트 14개를 먼저 쓰고 실패를 확인했다(모듈 없음·`get_usage_log` 없음·`request_param_name` None).
- `test_usage_log.py` 9개: 행 값, 비완료 응답 매핑, 사용량 없는 턴, 사용량 실패 시 커밋 없음, 1146 무경고, 그 밖 오류 경고, 백그라운드 실행, 루프 밖 호출 무예외, HOST 없으면 꺼짐.
- `test_usage_log_wiring.py` 5개: 도구 턴 USAGE 2건과 브라우저 합계 유지, 인사말 제외, 봇 없는 세션 제외, FastAPI 의 Request 주입, `x-forwarded-for` 전달.
- `pytest ai_voice/voice_tests tests` main **442 passed**, dev **464 passed**.
- 실제 드라이버 escape 로 SQL 을 렌더링해 문법과 `executemany` 일괄 INSERT 경로를 확인했다. 실 DB 저장은 dev 배포 뒤 확인한다.
