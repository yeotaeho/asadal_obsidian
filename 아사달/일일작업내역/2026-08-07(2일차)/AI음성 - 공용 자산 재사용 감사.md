---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-07
---

# 공용 자산 재사용 감사

> [!summary] 한 줄 요약
> v4 브랜치가 기존 공용 자산을 제대로 가져다 썼는지 34개 에이전트로 전수 조사해, 유일한 재발명 **423줄을 삭제**하고 사본 드리프트 가드를 넣었다. 교체 과정에서 넣은 회귀 1건은 Codex 가 잡아 되돌렸다. 프로젝트: [[AI음성]]

## 무슨 작업

"이미 만들어둔 모델·함수를 가져다 썼는지, 아니면 불필요하게 새로 만들었는지" 를 확인하는 감사였다. 인벤토리 4갈래 → 대조 → 적대적 검증 → 종합 순으로 워크플로를 돌렸다. 후보 14건 중 4건이 검증을 통과했다.

**결론은 대체로 잘 썼다는 것이다.** v4 가 import 하는 프로젝트 모듈 13개가 전부 `main` 에 이미 있던 것이다. RAG 검색(`rag_adapter.vector_search`), 챗봇 ID 조회(`utils.whoami`), 프롬프트 정책, 세션 스키마 검증, 상수 6개를 그대로 쓴다. 모델명·타임아웃 하드코딩 **0건**, 새 환경변수 **2개**뿐이다. STUN 게이트는 `main` 에 있던 `_to_bool` 을, 이벤트 루프는 stdlib `asyncio.get_running_loop()` 를 쓴다.

## 무엇이 걸렸나

진짜 재발명은 **1건**이었다. `v4/rtp_stats_utils.py` 423줄인데 v4 가 쓰는 건 `to_int` 하나뿐이고 나머지 12개 함수는 참조가 0회였다.

더 나쁜 건 이게 **함정**이라는 점이다. v4 세션의 RTP 통계는 실제로 v3 코드를 탄다 — 공용 `server_webrtc.py:67` 이 `v3.rtp_stats_utils` 에서 심볼을 가져오고 그게 v4 트랙에도 적용된다. 나중에 지터·손실 지표 버그를 잡는 사람이 `v4/rtp_stats_utils.py` 를 고치면 **그 수정은 조용히 아무 효과도 내지 못한다.**

덤으로 `v4/__init__.py` 의 재수출 4개는 패키지 레벨 소비자가 0건이었다. v3 것도 똑같이 죽어 있어 죽은 코드까지 복사한 상태였다.

## 회귀를 넣었다가 되돌린 일

`to_int` 를 공용 `utils/retriever_utils.safe_int` 로 갈아탔는데, Codex 가 반례를 찾았다. `safe_int` 는 `int(float(v))` 인데 잡는 예외가 `TypeError`/`ValueError` 뿐이라 float 변환 자체가 넘치는 값에서 `OverflowError` 가 샌다.

```
to_int(10**400)  -> top_k=10        safe_int(10**400)  -> OverflowError
to_int("1e309")  -> top_k=3         safe_int("1e309")  -> OverflowError
```

이 값은 OpenAI 가 보낸 tool 인자를 `json.loads` 한 결과다. `json.loads` 는 `Infinity` 와 자릿수 제한 없는 정수를 그대로 만들어주므로 도달 가능하다. 파싱 지점이 검색 `try` 블록 **바깥**이라 예외가 `_handle_function_tool_call` 밖으로 나가고, 이 코루틴은 태스크로 돌아 예외가 조용히 삼켜진다. 결과는 tool 응답 미전송 = **realtime 세션 행**이다. 앞서 고친 `results` 미바인딩과 같은 실패 유형이다.

내 검증 매트릭스는 23개 입력을 돌렸지만 거대값 구간이 없었다. 유한값에서만 확인하고 "완화이지 계약 변경이 아니다" 라고 단정한 게 잘못이었다.

공용 `safe_int` 를 고치면 근본 해결이지만 공용 모듈이라 범위 밖으로 두고, v4 안에 세 예외를 모두 잡는 `_to_top_k` 를 뒀다. 삭제된 `to_int` 에도 있던 `float('inf')` 구멍까지 함께 막혔다.

## 남긴 가드

`tool_result_policy.py`(152줄)와 `voice_context_summarizer.py`(62줄)는 v3 사본과 바이트 동일한데 드리프트를 잡는 장치가 없었다. v4 세션은 v4 사본을, v2 세션은 v3 사본을 타므로 한쪽만 고치면 증상이 한쪽 경로에서만 나타난다. 바이트 동일성을 검사하고 실패 시 unified diff 를 붙이는 테스트를 넣었다. 의도적으로 갈라뜨릴 땐 `PARITY_COPIES` 에서 빼면 된다.

`tool_call_orchestrator.py`(487)와 `audio_tracks.py`(608), 라우터(334) 사본은 **그대로 둔다.** v4 가 실질적으로 갈라졌고, 공유로 되돌리면 v4 수정이 v2 를 건드려 롤백 원인을 재현한다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `252f961` | rtp_stats_utils 사본 제거, 공용 safe_int 재사용 | `v4/rtp_stats_utils.py` 삭제, `v4/__init__.py`, `v4/tool_call_orchestrator.py` |
| `56ed06c` | 갈라지지 않은 v3/v4 사본 드리프트 가드 | `test_v4_integration_smoke.py` |
| `05481dc` | top_k 파싱 OverflowError 회귀 수정 | `v4/tool_call_orchestrator.py`, `test_lost_bugfixes.py` |

순 **439줄 삭제**.

## 검증

`pytest ai_voice/voice_tests/` **75 passed**.

변이 테스트로 새 가드의 실효성을 실측했다.

| 변이 | 실패하는 테스트 |
|---|---|
| `except` 에서 `OverflowError` 제거 | 거대값 3건만 |
| v4 사본에만 한 줄 추가 | 해당 파리티 파라미터만 |
| top_k clamp 제거 | `[99-10]` 만 |
| 기본값 fallback 제거 | `[0-3]` 등 |

Codex 판정은 Critical 0건, Important 1건(위 회귀 — 수정), Minor 0건이다.

## 남은 것

`utils/retriever_utils.py:71` 의 `safe_int` 는 여전히 `OverflowError` 를 안 잡는다. 내부 호출자 1곳(`:428`, `_rag_filter_match_strength`)이 노출돼 있지만 0~4 범위 값이라 실제 위험은 낮다. 선재 결함이고 공용 모듈이라 손대지 않았다.
