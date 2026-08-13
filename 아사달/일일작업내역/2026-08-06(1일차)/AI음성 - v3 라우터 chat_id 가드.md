---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-06
---

# v3 라우터 chat_id 가드

> [!summary] 한 줄 요약
> 운영 v3 음성채팅 RAG 경로의 UUID fall-through 결함(조회 실패 시 UUID가 SQL 테이블명에 보간 → 구문 오류)을 검색 진입 전 차단으로 수정했다. 프로젝트: [[AI음성]]

## 무슨 작업

`voice_chat_v3_router.py`의 `_resolve_rag_chat_id`를 고쳤다. chat_bot_id(UUID) → chat_id(정수) 조회가 실패하면 이전에는 UUID가 그대로 반환되어 `td_{chat_id}_training_data` 테이블명에 들어가 SQL 구문 오류가 났다.

이제 해석 실패나 식별자 검증 실패 시 `vector_search`에 도달하기 전에 명확한 에러 응답을 반환한다.

## 왜

v4 복원 Task 2 리뷰에서 동일 패턴이 Critical로 발견됐고, 구현 담당 에이전트가 "운영 v3 라우터에도 같은 결함이 있다"고 별도 신고했다.

v3는 현재 실사용 경로인데, v4 복원 작업에는 "v3 라우터 불가침" 전역 제약이 있어 그쪽 브랜치에서는 고칠 수 없었다. 별도 브랜치로 분리해 처리했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `da2c819` | dev 기준 최초 작업 (별도 워크트리 세션) | `ai_voice/infrastructure/api/router/voice_chat_v3_router.py` 외 |
| `36ec060` | main 기준 재생성 — 브랜치 `fix/v3-router-chat-id-guard-main`, push 완료 | 동일 + `ai_voice/tests/test_v3_router_chat_id_guard.py` |

v4 쪽 수정(`tool_call_orchestrator.py`, 커밋 `5c4f5b3`)과 에러 페이로드 형태를 통일했다. 총 5 files changed, +379.

## 검증

- `pytest` 행위 테스트 — `get_chat_id_from_db`를 monkeypatch로 None 반환시켜 `vector_search`가 호출되지 않음을 검증
- 다음 단계: dev PR → 테스트 통과 후 main PR
