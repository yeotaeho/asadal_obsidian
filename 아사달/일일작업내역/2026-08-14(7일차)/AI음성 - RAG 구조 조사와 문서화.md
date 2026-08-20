---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-14
---

# RAG 구조 조사와 문서화

> [!summary] 한 줄 요약
> 음성 RAG 전 구간을 어댑터부터 SQL 까지 읽어 정리하면서 **엔진이 FAISS 가 아니라 pgvector** 임을 확인하고(기존 문서 오기 정정), 답변 품질에 직접 영향을 주는 구조 문제 3건을 찾았다. 프로젝트: [[AI음성]]

## 무슨 작업

"RAG 시스템이 어떻게 되어 있나" 질문에서 출발해 `rag_adapter` → `pgvector_search` SQL → 후처리 → 경로별 분기까지 전 구간을 읽고 [[AI음성 - 런타임 플로우]] 6단계에 정리했다. 이어서 "검색 중 사용자가 끼어들면 어떻게 되나" 도 코드로 확인해 같은 절에 덧붙였다.

**엔진 오기 정정.** 그동안 문서와 설명에서 "FAISS 검색" 이라고 써 왔는데 실제로는 **pgvector** 다. 호출 경로가 `chat/faiss_process.py` 라 오해하기 쉽지만, 음성이 부르는 함수는 그 파일 안의 `pgvector_search` 이고 `rag_adapter.py` docstring 도 "pgvector 단일 경로를 사용" 이라고 명시하고 있다. 두 구조 노트의 FAISS 표기를 모두 고쳤다.

## 무엇을 알아냈나

검색은 **벡터 + TRGM 하이브리드**다. 진입은 OR 조건(`거리 < 0.2` 또는 키워드 ILIKE)이라 벡터가 멀어도 단어가 겹치면 후보가 되고, 후보는 벡터 거리순 **100개**로 잘린 뒤 `0.7 × (1 − 거리) + 0.3 × TRGM` 으로 재정렬돼 돌아온다.

문제로 보이는 것이 셋이다.

| # | 발견 | 왜 문제인가 |
|---|---|---|
| 1 | `_rank_voice_results` 가 **SQL 의 혼합 점수를 버리고** `(hit_score, related_hit, retrieval_score)` 로 재정렬 | DB 는 벡터 70% 가중으로 잘 정렬해 주는데, 파이썬이 **키워드 규칙 우선**으로 뒤집는다. 벡터 유사도가 3순위 타이브레이커로 밀린다 |
| 2 | `v4/tool_result_policy.py`(152줄)가 **GA 실경로에서 미실행** | orchestrator 가 이 모듈을 import 조차 안 한다. 재랭킹(연락처 질문 시 숫자 포함 +3)과 키워드 필터가 실제 통화에 안 걸린다. HTTP 엔드포인트에서만 쓰이는데 v4 는 서버가 내부 실행하므로 그 경로를 안 탄다 |
| 3 | 쿼리 보강 스코프가 **GA 만 세션 구분 없음**(`db\|chat_bot_id`) | 같은 챗봇을 쓰는 **서로 다른 사용자**의 후속질문 맥락이 섞일 수 있다. TTL 30분, 같은 워커에 있을 때 발생 |

부수적으로 상수 `0.2` 하나가 SQL 에서는 거리 임계값, 파이썬에서는 유사도 하한으로 **두 의미**로 쓰인다. 튜닝하면 서로 다른 두 기준이 동시에 움직인다.

## 검색 중 끼어들기

태스크가 겹쳐 돌지 않는다. `_handle_speech_started` 가 `cancel_tool_tasks()` 를 **조건 없이** 부르기 때문이다. RAG 실행 중에는 pending 도 큐도 비어 있어 끼어들기 판정으로는 못 잡는 것을 별도로 끊는 구조다.

"B 에 답하다가 A 의 검색이 끝나면 A 에 답한다" 는 과거 실제 버그였고 이 코드가 그것을 막는다. `CancelledError` 가 `BaseException` 이라 `except Exception` 에 안 걸려 태스크가 조용히 죽고, 결과도 재개 지시도 나가지 않는다. OpenAI 쪽은 `interrupt_response: true` 로 스스로 응답을 버려 양쪽이 함께 포기한다.

단 **v3·v2 에는 `cancel_tool_tasks` 가 없다**(v3 바이트 고정). 그쪽은 늦게 끝난 RAG 가 취소된 턴의 답변을 되살릴 수 있다.

## 어디

코드 수정 없음 — 읽기 전용 조사와 문서화다.

| 노트 | 변경 |
|---|---|
| `AI음성 - 런타임 플로우` | 6단계에 "검색 엔진과 후처리" · "경로별 후처리" · "검색 도중 끼어들면" 절 신설 |
| `AI음성 - 파일·함수 레퍼런스` | 재사용 자산표의 FAISS 표기를 pgvector 로 정정 |

## 검증

코드 실측으로 확인했다.

- 엔진: `rag_adapter.py:8` 의 `from chat.faiss_process import get_embeddings, pgvector_search`
- 진입 OR 조건·후보 상한: `faiss_process.py:1494`(`max(k*10, 100)`), `:1513~1524`(ILIKE EXISTS)
- 혼합 점수: `:1539` 의 `0.7 * (1 - v.vector_score) + 0.3 * COALESCE(s.text_score, 0)`
- 파이썬 재정렬: `rag_adapter.py:35~51`
- `tool_result_policy` 미사용: 저장소 grep 으로 import 처가 v3·v4 **라우터뿐**임을 확인
- 스코프 차이: `build_query_scope(` 호출 4곳 전수 대조
- 끼어들기: `server_webrtc.py:1940~1944` + `tool_call_orchestrator.py:82`, v3 에 동명 메서드 부재 확인

## 추후

- [ ] 정렬 정책 결정 — 파이썬 재정렬을 걷어내고 SQL 혼합 점수를 살릴지, 키워드 우선을 유지할지. A/B 실측 필요
- [ ] `tool_result_policy` 를 GA 경로에 연결할지 삭제할지 결정
- [ ] 쿼리 보강 스코프에 `session_id` 추가 검토 (v4 는 session_id 를 알고 있다)
