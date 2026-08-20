---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-19
---

# search_web 미등록 도구 지시 제거

> [!summary] 한 줄 요약
> "추천해 볼게요"라고 선언만 하고 무응답/5초 지연되는 증상을 조사해, **등록되지 않은 도구(search_web) 호출을 지시하던 죽은 지시문**이 원인임을 확인·제거했다. 프로젝트: [[AI음성]]

## 무슨 작업

사용자 보고 이미지(음성 채팅에서 "오늘 같은 날씨에 어울리는 활동 추천"에 선언만 하고 무응답/지연)를 근거로 조사했다.

dev 세션의 실제 설정(`/agent/config`)을 직접 조회한 결과, 등록된 tool 은 `search_knowledge_base` 하나뿐인데 지시문(`V3_WEB_SEARCH_INSTRUCTION_SUFFIX`)은 "최신성이 중요한 질문(날씨 등)일 때 **search_web** 를 호출하라"고 지시하고 있었다. `search_web` 는 저장소 전역에 구현이 0건이고(`/agent/tools/search-web` → 404), `git log -S` 로 추적하니 2026-04-08 원본 덤프(`2f9e78b`)에서 짝이 되는 실행 코드 없이 지시문만 들어온 유물이었다 — 8-13 에 제거한 Tool-Only 모드와 같은 계보.

모델은 실행할 수 없는 도구 사용을 스스로 선언(예: 날씨 추천)한 뒤, 지시문의 흐름 규칙("첫 부족 응답에서 확인 질문과 함께 턴을 마무리하세요")을 따라 그대로 턴을 끝낸다. 이것이 이미지의 무응답 증상 직접 원인이다.

## 함께 확인한 것 (별건, 미조치)

- **5초 지연**: `search_knowledge_base` 왕복 예산이 `VOICE_CHAT_FUNCTION_TOOL_TIMEOUT_SECONDS` 기본 5.0초. 지식DB에 없는 잡담/일반상식 질문도 검색을 한 바퀴 돌고 0건 유도 메시지를 받은 뒤에야 답하는 구조.
- **연쇄 악화 가설**: 사용자가 기다리다 재발화하면 진행 중 검색 태스크가 무조건 취소되고, OpenAI 대화 기록에 결과 없는 function_call 이 남는다(런타임 문서에 기록된 미확인 항목) — 이미지의 문장 파편 반복과 부합.

## 조치

`realtime_instruction_policy.py` 에서 `V3_WEB_SEARCH_INSTRUCTION_SUFFIX` 와 `enable_web_search` 파라미터를 통째로 제거했다(죽은 코드 즉시 제거 원칙). 두 라우터(v3·v4) 모두 이 함수를 kwargs 없이 호출해 `enable_web_search` 는 항상 기본값 `True` 로만 쓰였다 — 꺼본 적 없는 죽은 유연성.

이 파일은 v3 라우터도 공유하므로 **v3 운영 응답에도 함께 반영된다**(사용자 확인 후 진행. 8-14 "잡담 검색 선언 완화"와 동일 전례).

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `526091d` | search_web 지시 제거 | `realtime_instruction_policy.py`, 테스트 신설 |

## 검증

- `pytest ai_voice/voice_tests/` — **176 passed**, 1 failed(기존 환경 무관)
- 신규 회귀 테스트 2건 — 지시문에 search_web 부재 + search_knowledge_base 존재 확인
- Codex 리뷰 클린

## 남은 작업

- [ ] 잡담·일반상식 질문에 RAG 호출 자체를 건너뛰는 지시문 추가 완화 (5초 지연 원인)
- [ ] dev 로그의 `tool 호출 완료(... elapsed_ms=...)` 로 실제 검색 소요 계측
- [ ] 취소된 tool call 이 이후 턴에 남기는 영향 실측 (미확인 항목)
