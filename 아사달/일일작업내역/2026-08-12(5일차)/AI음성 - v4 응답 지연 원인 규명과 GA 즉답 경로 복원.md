---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-12
---

# v4 응답 지연 원인 규명과 GA 즉답 경로 복원

> [!summary] 한 줄 요약
> 복원된 v4가 오리지널보다 답변 시작이 수 초 늦던 원인을 규명하고, 오리지널의 GA 즉답 경로(OpenAI 자동 응답 + LangGraph 스킵)를 복원했다. Codex 리뷰 4라운드에서 나온 경합 3건도 함께 잡았다. 프로젝트: [[AI음성]]

## 무슨 작업

오리지널 v4(backup/voice-v4)는 실시간 대화처럼 빨랐는데 복원본은 느리다는 문제의 원인을 파악하고 수정했다. 원인은 복원 과정에서 **GA 모델 LangGraph 스킵 가드가 유실**된 것이었다.

오리지널은 두 가지가 맞물려 빨랐다. `create_response: True`로 OpenAI가 발화 종료 즉시 응답·tool call을 직접 처리했고, `_flush_turn_buffer`의 GA 가드(`e12eeea`)가 서버 턴 파이프라인을 통째로 건너뛰었다. 롤백(`e70ab01`)이 가드를 지웠고 복원이 가드를 되살리지 않아 LangGraph 경로가 재활성화됐다. 그 결과 이중 응답(말풍선 2개)이 재현됐고, `67f51b1`은 이를 반대 방향 — 자동 응답을 끄는 쪽 — 으로 눌렀다. 남은 유일한 응답 경로가 느렸던 것이다.

```
느려진 경로 (복원본):
발화 종료 → 전사 대기(+0.5~1.5초) → 턴 버퍼 타이머(+1.5초)
→ LangGraph(Decision LLM + vector_search) → response.create
→ tool 라운드(vector_search 재실행 + summarize LLM) → 답변

복원한 경로 (오리지널):
발화 종료 → OpenAI 즉시 응답 → 인라인 tool call
→ vector_search + summarize → 답변
```

## 왜

오리지널 개발자도 같은 이중 응답 충돌을 겪었고 "서버 경로를 끈다"로 해결했었다. 복원 작업은 증상만 보고 "자동 응답을 끈다"로 해결해 빠른 경로를 죽였다. 이번 수정은 오리지널이 검증한 방향으로 되돌린 것이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `e1e9896` | GA 즉답 경로 복원 — create_response 오버라이드 제거 + GA 가드 재이식 | `voice_chat_v4_router.py`, `server_webrtc.py` |
| `127abd9` | 가드를 v4 세션으로 한정 (v2 GA 세션 무응답 방지) | `server_webrtc.py` |
| `3263003` | 자동 응답도 큐가 인지하도록 response.created에서 pending 표시 | `server_webrtc.py` |
| `41b6d49` | 끼어들기 판정을 상태 문자열 대신 실제 진행/대기 작업 기준으로 | `server_webrtc.py` |
| `ca73d10` | 새 발화 시작 시 이전 턴의 진행 중 tool 태스크 취소 | `server_webrtc.py`, `v4/tool_call_orchestrator.py` |

가드 조건은 `_session_use_v4_orchestrator + GA 모델` 쌍이다. 브릿지는 v2 서버 관리 세션과 공유되고 v2는 호출자가 모델을 고르므로, GA 모델만 보면 v2 GA 세션의 유일한 응답 경로(LangGraph)까지 끊긴다.

## Codex 리뷰

4라운드 진행, 라운드마다 Important 1건 → 검증 후 반영 → 재리뷰. 최종 판정 **Ship**.

1. 가드가 v2 GA 세션까지 잡음 → v4 마커 조건 추가
2. 자동 응답은 pending 표시가 없어 tool 재개가 얹힘 → response.created에서 표시
3. GA 세션은 ctx.state가 "responding"이 안 돼 끼어들기 시 큐 잔류 → 판정 조건 확장
4. RAG 실행 중 끼어들기는 큐도 pending도 비어 못 잡음 → speech_started에서 tool 태스크 무조건 취소

## 검증

- pytest ai_voice/voice_tests/ **153 passed** (잔여 1건은 수정 전에도 실패하는 로컬 FastAPI 버전 문제로 실측 확인)
- 변이 실측 4종 — GA 가드 제거 / pending 표시 제거 / 끼어들기 조건 원복 / 취소 로직 제거, 각각 대응 테스트가 실패
- v3/ 는 origin/main 대비 무변경
- 테스트 파일: `test_v4_auto_response_disabled.py`(전제가 뒤집힘)를 `test_v4_ga_direct_response.py`로 교체, `test_response_queue_contention.py`에 5건 추가
- **dev 배포 후 실기 응답 속도 재측정 필요** — 서버 코드 기준으로는 오리지널과 동일 경로
