---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-13
---

# Tool-Only 휴면 분기 제거

> [!summary] 한 줄 요약
> flush 가드의 Tool-Only 분기(매 턴 검색 강제 모드)를 세팅처 0건 확인 후 v2 opt-in 계약째 제거했다 — **39줄 삭제, flush 가드 4종 → 3종**, 양 브랜치 커밋, Codex 클린 통과. 프로젝트: [[AI음성]]

## 무슨 작업

`_flush_turn_buffer` 의 Tool-Only 분기와 그 전용 메서드 2개(`_is_tool_only_mode`, `_build_tool_only_instructions`), 테스트의 스텁 1줄을 제거했다. 이 모드는 켜지면 검색이 필요 없는 발화에도 매 턴 `search_knowledge_base` 호출을 강제하는 실험 스위치였다.

제거 전 조사로 성격을 확정했다. 켜는 키(`v15_tool_only`·`tool_only_mode`·`realtime_tool_only`)를 세팅하는 코드가 저장소 전체에 **0건**, v4 라우터 metadata 에도 없음. 이론상 유일한 활성 경로는 v2 세션 요청의 클라이언트 metadata opt-in 뿐이었고, 유래는 2026-04-08 원본 덤프(`2f9e78b`)에 실려온 v1.5 실험기 스위치였다.

처음에는 v2 계약이라 별도 커밋으로 미루자고 제안했으나, 사용자가 v2 를 신경 쓰지 말고 제거하라고 결정해 계약째 걷어냈다.

## 왜

검색 필요 여부는 LLM 의 tool call 판단에 맡기는 것이 v4 의 설계다. 무조건 검색을 강제하는 분기는 그 설계와 충돌하고, 도달 불가능한 죽은 코드를 flush 경로 한가운데 남겨둘 이유가 없다 — 죽은 코드 즉시 제거 원칙.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `0ef439b` (-dev) / `3e43324` (원본) | Tool-Only 모드 제거 — 메서드 2개 + flush 분기 + 테스트 스텁, 39줄 삭제 | `ai_voice/services/voice_chat/server_webrtc.py`, `ai_voice/voice_tests/test_v4_ga_direct_response.py` |

양 브랜치(`fix/v4-session-generation-guard-dev` / `fix/v4-session-generation-guard`)에 동일 반영해 v4 소스 동등성을 유지했다. 푸시는 아직이다.

## 검증

- 잔여 참조 grep — `tool_only|tool-only|Tool-Only` 저장소 전체 **0건**.
- 임포트 생사 확인 — `WEBRTC_VOICE_CONVERSATION_CONFIG`·`build_mandatory_search_rule` 은 `_build_realtime_response_instructions` 가 계속 사용, 유지.
- `pytest ai_voice/voice_tests` — **173 passed, 1 failed**. 실패 1건은 기존 환경 이슈(agent_env fastapi 0.141.1 vs 핀 0.115.9 로 라우트 열거 공허 — 테스트 자체가 이 원인을 안내)로 본 변경과 무관.
- Codex 리뷰 (`--base HEAD~1 --scope branch`) — "No actionable regressions were identified" 클린 통과.
- [[AI음성 - 런타임 플로우]] 갱신 — flush 가드 4종 → 3종, Tool-Only 절을 제거 기록으로 교체, 삭제로 당겨진 `sw:` 라인 참조 전체 재계산(-19/-38줄).
