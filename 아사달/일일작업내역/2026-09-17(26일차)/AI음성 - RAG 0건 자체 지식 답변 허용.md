---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-17
---

# RAG 0건 자체 지식 답변 허용

> [!summary] 한 줄 요약
> 음성채팅에서 RAG 검색 결과가 0건이면 "찾지 못했다" 안내 대신, 학습 자료가 아니라고 밝힌 뒤 자체 지식으로 답하게 바꿨다. 서비스: [[AI음성]]

## 무슨 작업

2026-08-28 [[AI음성 - RAG 0건 폴백 미확인 안내 전환]] 에서 정한 "0건이면 미확인 안내" 를 **뒤집었다**. v4 오케스트레이터의 0건 지시 두 곳과, v3·v4 공용 세션 지시의 응답 정책 (C) 를 함께 고쳤다.

| 위치 | 이전 | 이후 |
|---|---|---|
| `v4/tool_call_orchestrator.py` `_send_tool_continue_response` | 자체 지식 금지, 찾지 못했다고 안내 | 일반 지식으로 짧게 답하되 학습된 자료가 아님을 밝힘, "근거에 따르면" 금지, 모르면 모른다고, 침묵 금지 |
| 같은 파일 도구 결과 `answer` | 찾지 못했다고 안내 | 일반 지식이라고 밝히고 답함 |
| `realtime_instruction_policy.py` 정책 (C) | 추측 금지, 정보 부족 안내 | 학습 자료가 아니라고 밝힌 뒤 일반 지식으로 답함 |

## 왜

사용자 요청이다. 찾지 못했다는 안내만으로는 질문이 해결되지 않는다.

강남구 구청장 건(근거인 양 단정한 오답) 재발을 줄이려고 **출처 고지와 "근거에 따르면" 표현 금지는 남겼다**. 세션 지시 (C) 를 같이 바꾼 것은 0건 지시와 충돌하면 예전 침묵 증상이 다시 나기 때문이다. (C) 는 v3·v4 공용이라 **v3 에도 적용된다**.

## 하지 않은 것

검색 결과는 있는데 답이 없는 경우의 요약기 규칙(`voice_context_summarizer.py` "찾지 못했다고 답하세요")은 요청 범위 밖이라 그대로 두었다. v3 오케스트레이터(바이트 고정)도 건드리지 않았다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `4353358` | RAG 0건 시 자체 지식 답변 허용 | `ai_voice/services/voice_chat/v4/tool_call_orchestrator.py` · `ai_voice/services/voice_chat/realtime_instruction_policy.py` · `ai_voice/voice_tests/test_rag_empty_result_policy.py` |

브랜치는 `main` 기준 `fix-rag-empty-self-knowledge` 다. 미푸시 상태다.

## 검증

- `pytest ai_voice/voice_tests -q` — main 기준 브랜치 **240 passed**, `feat/voice-record` 위 **420 passed**.
- `test_rag_empty_result_policy.py` 두 테스트를 새 계약(일반 지식·출처 고지·침묵 금지)으로 반전했다.
- 실제 통화 확인은 dev 배포 뒤 남아 있다.
