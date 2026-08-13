---
tags: [아사달, 프로젝트, AI음성, 아키텍처]
created: 2026-08-10
updated: 2026-08-10
---

# AI음성 - 파일·함수 레퍼런스

> [!summary] 한 줄 요약
> **"어디를 고쳐야 하는가"를 찾을 때 보는 노트다.** 파일 지도, 파일별 심볼표, 손댈 때 규칙을 담았다. 실행 순서는 [[AI음성 - 런타임 플로우]], 진행 상황은 [[AI음성]] 참고.

## 기술 스택

| 계층 | 라이브러리 | 이 경로에서 하는 일 |
|---|---|---|
| WebRTC | `aiortc==1.9.0` | 서버가 **양쪽 피어**를 직접 붙든다. `MediaRelay` 로 한 트랙을 여러 소비자에게 분배 |
| 오디오 프레임 | `av` (PyAV) | `AudioFrame` 단위 조작. 샘플레이트·PTS 재계산 |
| 수치 | `numpy==1.26.4` | 프레임 RMS 계산 |
| LLM | `openai==1.70.0` | Realtime API 음성 대 음성 + 검색 결과 요약 |
| API | `fastapi==0.115.9` | 라우터·스키마. **버전 핀이 중요** |
| 런타임 | gunicorn + UvicornWorker | 4워커. 이 구조가 `_get_loop()` 가 필요한 이유 |
| 동시성 | `asyncio` | 세션당 태스크 6~8개가 동시에 돈다 |

`aiortc` 가 이 경로의 성격을 결정한다. v3 는 브라우저가 OpenAI 와 직접 붙지만 **v4 는 서버가 중간에 끼어서**(Server-In-Path) 오디오를 만질 수 있게 된다. 대가로 서버가 WebRTC 협상·ICE·RTP 를 직접 감당한다.

> [!warning] fastapi 핀을 풀지 말 것
> 0.141 에서 `include_router()` 가 `.path` 없는 lazy 객체를 만든다. 라우트 열거로 "v4 경로가 없어야 한다" 를 확인하던 테스트가 **라우트를 하나도 못 읽어서 공허하게 통과**한 적이 있다. 지금은 빈 목록이면 시끄럽게 실패하는 카나리아가 있지만, 핀을 유지하는 게 먼저다.

## 파일 지도

### v4 전용 — 이 폴더만 고치면 v3·v2 에 영향이 없다

| 파일 | 줄 | 역할 |
|---|---|---|
| `router/voice_chat_v4_router.py` | 334 | 엔드포인트 4개. 세션 생성 시 필터 트랙과 v4 플래그를 주입 |
| `v4/audio_tracks.py` | 608 | 지터 버퍼 + 노이즈 게이트 + 좀비 판정. 가장 복잡 |
| `v4/tool_call_orchestrator.py` | 503 | LLM 의 함수 호출을 받아 RAG 를 실행 |
| `v4/tool_result_policy.py` | 152 | 검색 결과 포맷·중복 제거·필터·재랭킹 |
| `v4/voice_context_summarizer.py` | 62 | 검색 결과를 음성 답변용으로 요약 |
| `v4/__init__.py` | 0 | 비어 있음. 죽은 재수출을 걷어냈다 |

### 공용 — v2 도 함께 탄다. 고칠 때 영향 확인 필수

| 파일 | 변경 | 내용 |
|---|---|---|
| `voice_chat/server_webrtc.py` | +552 | ICE 픽스 · 이벤트 루프 · 트랙 주입 · 좀비 정리 · orchestrator 분기 · 세대 가드 |
| `voice_router.py` | +38 | v4 등록 게이트. import 실패 시 조용히 폴백 |
| `workflow/realtime_speaker.py` | +24 | GA 모델은 `response.create` 에서 modalities·voice 제외 |
| `workflow/runtime.py` | +4 | `active_session_ids()`. 고아 세션 스위퍼가 쓴다 |
| `requirements.txt` | +1 | `aiortc==1.9.0` |

### 재사용하는 기존 자산 — 새로 만들지 말 것

v4 가 import 하는 프로젝트 모듈 13개는 전부 `main` 에 이미 있던 것이다.

| 자산 | 쓰는 곳 | 무엇을 |
|---|---|---|
| `utils/whoami.py` | orchestrator, 라우터 | `get_chat_id_from_db` — UUID → 8자리 chat_id |
| `ai_voice/constants.py` | 라우터 | 모델 ID · 세션 config 6개. 하드코딩 0건 |
| `rag/rag_adapter.py` | orchestrator, 라우터 | `vector_search` — FAISS 검색 |
| `rag/query_enrichment_service.py` | orchestrator, 라우터 | 후속질문 문맥 보강, 스코프 조립 |
| `rag/search_options.py` | orchestrator, 라우터 | 검색 옵션 조립 |
| `realtime_session_schemas.py` | 라우터 | GA · legacy 세션 shape 검증 |
| `realtime_instruction_policy.py` | 라우터 | 시스템 프롬프트 조립 |
| `policy_keywords.py` | tool_result_policy | 키워드 정책 |
| `voice_chat/voice_logging.py` | audio_tracks | 로거 생성 |

## 파일별 심볼

### `v4/audio_tracks.py`

클래스 3개. `NoiseFilteredAudioProxyTrackV4` 가 `AudioProxyTrack` 을 상속한다.

| 심볼 | 줄 | 역할 |
|---|---|---|
| `AudioProxyTrack` | 16 | 지터 버퍼 본체 |
| `_reader_loop` | 114 | 업스트림에서 계속 읽어 버퍼에 넣는 **별도 태스크** |
| `_wait_for_prebuffer` | 164 | 재생 전 최소량이 찰 때까지 대기 |
| `_build_silence_frame` | 204 | 언더플로 때 채울 무음 |
| `_get_next_frame` | 218 | 버퍼에서 꺼냄. 비면 무음 + 언더플로 카운트 + **좀비 판정** |
| `_pace_playout` | 282 | 실시간 속도 페이싱 |
| `_retime_frame` | 300 | `Fraction` 으로 PTS 재계산 |
| `stop` | 323 | 리더 태스크 취소 + 업스트림 종료 |
| `get_diagnostics` | 75 | 버퍼·드롭·언더플로 스냅샷 |
| `SilenceAudioStreamTrack` | 331 | 실제 오디오가 붙기 전 자리를 채우는 무음 소스 |
| `NoiseFilteredAudioProxyTrackV4` | 359 | 노이즈 게이트를 얹은 v4 전용 트랙 |
| `_calculate_rms` | 431 | 프레임 음량 |
| `_process_utterance` | 442 | 모은 발화를 내보낼지 버릴지 판정 |
| `recv` | 479 | 상태 기계 본체. 20ms 마다 호출 |

### `v4/tool_call_orchestrator.py`

| 심볼 | 줄 | 역할 |
|---|---|---|
| `_SAFE_CHAT_ID_RE` | 23 | SQL 테이블명에 들어가므로 안전한 식별자만 허용 |
| `_to_top_k` | 25 | top_k 파싱. `TypeError·ValueError·OverflowError` 전부 포획 |
| `clear_session` | 74 | 세션의 tool 태스크 취소. `pop` 기반이라 멱등 |
| `extract_function_tool_call` | 82 | 이벤트에서 tool call 추출. 타입 2종 |
| `_parse_tool_arguments` | 115 | JSON 파싱. 실패 시 원문을 query 로 간주 |
| `schedule_function_tool_call` | 130 | 중복 제거 후 **별도 태스크**로 스케줄 |
| `_build_tool_output_payload` | 167 | 검색 결과를 응답 형태로 |
| `_send_function_call_output` | 201 | 데이터채널로 결과 전송 |
| `_send_tool_continue_response` | 236 | 답변 재개 지시. 0건이면 자체 지식 유도 |
| `_send_tool_error` | 262 | 실패해도 같은 계약의 payload 로 끝낸다 |
| `_handle_function_tool_call` | 283 | 본체 |

### `v4/tool_result_policy.py`

| 심볼                                 | 줄   | 역할                        |
| ---------------------------------- | --- | ------------------------- |
| `format_tool_results`              | 33  | 상위 N 개 포맷 + 길이 제한 + 중복 제거 |
| `is_contact_query`                 | 72  | 연락처 질문인지 판정               |
| `query_tokens`                     | 77  | 질문을 토큰으로                  |
| `filter_results_by_query_keywords` | 83  | 질문 토큰과 겹치는 결과만 남김         |
| `truncate_for_log`                 | 111 | 로그용 자르기                   |
| `rerank_results_for_answering`     | 118 | 답변용 재정렬                   |

`v4/voice_context_summarizer.py` 는 `_build_context()`(24) 하나와 요약 진입점뿐이다. `openai` 를 직접 쓴다.

### `server_webrtc.py` — v2 와 공유

| 심볼 | 줄 | 역할 |
|---|---|---|
| `_get_loop` | 8 | 요청 시점 running loop. gunicorn 멀티워커 대응 |
| `_to_bool` | 92 | 기존 헬퍼. 새로 만들지 말 것 |
| `_build_pc` | 113 | STUN 없이 PC 생성. **5,004ms → 6ms** |
| `_apply_audio_codec_preferences` | 143 | 코덱 우선순위 |
| `create_answer` | 269 | 세션 생성 진입점. 세대 발급 |
| `close` | 407 | 세대 가드 + 동기 선점 정리 |
| `_schedule_disconnect_grace` | 506 | ICE 끊김 후 유예 타이머 |
| `_cancel_disconnect_grace` | 542 | 유예 타이머 취소 |
| `_ensure_sweeper` | 560 | 좀비 스위퍼 기동 (전역 1개) |
| `_reap_orphan_runtime_sessions` | 565 | PC 없이 남은 runtime 세션 회수 |
| `_sweep_zombie_sessions` | 585 | 60초 주기 전역 스위퍼 |
| `_orchestrator_for` | 1087 | **v3/v4 orchestrator 분기** |
| `attach_track` | 1126 | 브릿지에 트랙·플래그 전달 |
| `detach` | 1166 | 브릿지 정리. 양쪽 orchestrator 모두 clear |
| `_connect_realtime` | 1231 | OpenAI 와 두 번째 SDP |
| `_start_rtp_stats_monitor` | 1318 | 1초 주기 RTP 통계 |
| `_start_audio_monitor` | 2106 | 무음·품질 감시 |

### v4 라우터

| 엔드포인트 | 핸들러 | 줄 | 역할 |
|---|---|---|---|
| `POST /agent/session` | `create_v4_agent_session` | 179 | 세션 생성. 필터 트랙 + v4 플래그 주입 |
| `POST /agent/session/{id}/release` | `release_v4_agent_session` | 242 | 세션 종료 |
| `GET /agent/config` | `get_v4_agent_config` | 253 | 클라이언트용 설정 조회 |
| `POST /agent/tools/search-knowledge-base` | `run_v4_search_knowledge_tool` | 268 | HTTP 로 직접 RAG 검색 |

접두사는 `/AI_voice/voice-chat-v4` 다. 게이트는 `voice_router.py` 가 쥐고 있고 **기본 활성**이다. 끄려면 `ENABLE_VOICE_V4=off` 를 넣는다. 알 수 없는 값이면 조용히 꺼지지 않고 기본값을 유지한다.

## 손댈 때 규칙

### 절대 건드리지 말 것

`ai_voice/services/voice_chat/v3/` 와 `voice_chat_v3_router.py` 는 `origin/main` 과 **바이트 단위로 같아야** 한다. `test_v4_integration_smoke.py` 가 이걸 강제한다. v4 를 고치려고 v3 파일을 여는 순간 뭔가 잘못된 것이다.

### 어느 파일을 고쳐야 하는가

| 고치려는 것 | 여기를 | 영향 |
|---|---|---|
| v4 오디오 동작 | `v4/audio_tracks.py` | v4 only |
| v4 RAG 동작 | `v4/tool_call_orchestrator.py` | v4 only |
| v4 결과 가공 | `v4/tool_result_policy.py` | **v3 사본과 동일. 파리티 테스트 확인** |
| v4 요약 | `v4/voice_context_summarizer.py` | **v3 사본과 동일. 파리티 테스트 확인** |
| 세션 수명주기 | `server_webrtc.py` | **v2 도 영향** |
| v4 엔드포인트 | `voice_chat_v4_router.py` | v4 only |

### 파리티가 걸린 파일

`tool_result_policy.py`(152줄)와 `voice_context_summarizer.py`(62줄)는 v3 사본과 **지금 바이트 동일**하다. 한쪽만 고치면 테스트가 실패하며 diff 를 보여준다. 그때 선택은 둘 중 하나다.

- 같은 수정을 양쪽에 반영한다. 대부분 이쪽이다.
- v4 만 의도적으로 갈라진다면 `PARITY_COPIES` 목록에서 빼고 이유를 커밋 메시지에 남긴다.

`tool_call_orchestrator.py` 와 `audio_tracks.py` 는 이미 실질적으로 갈라져서 목록에 없다.

### 새 코드를 넣기 전에

- 공용 자산을 먼저 찾는다. `utils/`, `ai_voice/constants.py`, `ai_voice/infrastructure/utils/`, `rag/` 에 이미 있는 경우가 많다.
- 불린 파싱은 `server_webrtc._to_bool` 이 이미 있다.
- 정수 변환은 `_to_top_k` 를 참고한다. 공용 `safe_int` 는 `OverflowError` 를 안 잡는다.
- 성공 경로에 `logger.warning` 을 쓰지 않는다. 로그 폭탄으로 운영 로그를 못 읽게 만든 전례가 있다.
- 루프는 항상 `_get_loop()` 를 거친다. `self._loop.create_task` 직접 호출은 테스트가 막는다.

## 함정 세 가지

> [!danger] 1 · `v3/` 폴더 파일을 쓰는 건 v3 가 아니라 v2 다
> v3 라우터는 `webrtc_api.create_webrtc_session` 을 직접 부르고 `server_webrtc` 를 아예 안 거친다. `server_webrtc` 를 쓰는 건 **v2 와 v4** 뿐이다. 폴더 이름에 속지 말 것.

> [!danger] 2 · v3 와 v4 의 `AudioProxyTrack` 은 상속 관계가 없다
> 이름만 같은 완전히 별개 클래스다. v3 쪽을 고쳐도 v4 에는 아무 영향이 없다. 좀비 스위퍼를 v3 파일에 넣었다가 v4 에서 전혀 동작하지 않은 적이 있다. 지금은 `test_orchestrator_isolation.py` 가 상속 관계 부재를 명시적으로 검증한다.

> [!danger] 3 · tool call 의 모든 종료 경로는 응답을 보내야 한다
> `_handle_function_tool_call()` 은 태스크로 돌기 때문에 예외가 조용히 삼켜진다. 응답을 안 보내고 끝나면 **통화가 그대로 멈춘다.** 실제로 두 번 터졌다. 자세한 내용은 [[AI음성 - 런타임 플로우]] 의 3단계 참고.

## 테스트 지도 · 75건

| 파일 | 건수 | 잠그는 것 |
|---|---|---|
| `test_lost_bugfixes.py` | 19 | chat_id 가드, top_k 파싱, 요약 타임아웃, GA 모델 분기 |
| `test_audio_track_injection.py` | 11 | 트랙 주입 배관, 협상 실패 시 상태 누수 |
| `test_orchestrator_isolation.py` | 10 | v4 세션이 v4 orchestrator 를 타는지. 실제 호출로 확인 |
| `test_v4_integration_smoke.py` | 8 | v3 바이트 동일성, 사본 드리프트, 라우트 등록 |
| `test_zombie_session_sweeper.py` | 8 | 유예 타이머·스위퍼 완주, 재사용 id 오종료 방지 |
| `test_session_generation_guard.py` | 7 | 늦은 콜백 세대 가드. 실제 지연을 만들어 재현 |
| `test_v4_router_gate.py` | 6 | 게이트 on·off·기본값·미상값 |
| `test_event_loop_helper.py` | 3 | 직접 루프 호출 회귀 차단 |
| `test_v4_package_import.py` | 3 | 패키지 import |

전용 venv 가 필요하다. `conftest.py` 헤더에 필수 패키지와 버전 핀이 적혀 있다.

```bash
pytest ai_voice/voice_tests/ -q
```

> [!tip] 테스트를 믿을 수 있게 유지하는 법
> 이 스위트의 가드는 대부분 **변이 테스트로 실효성을 확인**했다. 새 가드를 추가하면 같은 절차를 밟는다 — 지키려는 코드를 일부러 망가뜨리고, 그 가드만 실패하는지 본다. 소스 문자열만 검사하는 테스트는 조건이 뒤집혀도 통과한 전례가 있다.

## 관련 노트

- [[AI음성]] — 프로젝트 진행 상황
- [[AI음성 - 런타임 플로우]] — 실행 순서와 함수 체인
- [[v3와 v4 아키텍처]] — v3 와의 구조적 차이
- [[운영에서만 터지는 문제들]] — 이벤트 루프 불일치 등
