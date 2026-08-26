---
tags:
  - 아사달
  - 프로젝트
  - AI음성
  - 아키텍처
created: 2026-08-10
updated: 2026-08-21
---

# AI음성 - 파일·함수 레퍼런스

> [!summary] 한 줄 요약
> **"어디를 고쳐야 하는가"를 찾을 때 보는 노트다.** 파일 지도, 파일별 심볼표, 호출 체인, 손댈 때 규칙을 담았다. 실행 흐름의 의미는 [[AI음성 - 런타임 플로우]], 진행 상황은 [[AI음성]] 참고.

> [!warning] 2026-08-18 갱신 — 게이트 복원으로 8-14 판의 "정정" 이 다시 뒤집혔다
> 8-14 판은 투명 탭 기준으로 "상속 없음 · 필터링 안 함" 을 정정 사항으로 못박았는데, **업무 지시로 게이트가 복원되면서 그 두 서술이 다시 반대가 됐다.**
>
> | 8-14 판 | 현재 (8-18, `336dcdc`) |
> |---|---|
> | `NoiseFilteredAudioProxyTrackV4` 는 상속 없음 | **`AudioProxyTrack` 을 상속한다**(at:375) — 지터버퍼·페이싱·좀비 가드를 물려받는다. `test_orchestrator_isolation.py` 가 상속 존재를 단언한다 |
> | 업링크 = 투명 탭 (관찰만) | **게이트다** — 발화를 쥐고 3관문(완화 기본값 200ms/120ms/30%)으로 방출·폐기를 결정한다. 경위는 [[AI음성 - 업링크 게이트 복원]] → [[AI음성 - 게이트 P1 수정]] → [[AI음성 - 게이트 임계값 완화]] |
>
> 라인 번호는 2026-08-18 `336dcdc` 기준으로 다시 계산했다.

## 기술 스택

| 계층 | 라이브러리 | 이 경로에서 하는 일 |
|---|---|---|
| WebRTC | `aiortc==1.9.0` | 서버가 **양쪽 피어**를 직접 붙든다. `MediaRelay` 로 한 트랙을 여러 소비자에게 분배 |
| 오디오 프레임 | `av` (PyAV) | `AudioFrame` 단위 조작. 다운링크에서 샘플레이트·PTS 재계산 |
| 수치 | `numpy==1.26.4` | 프레임 RMS 계산 (업링크 관찰용) |
| LLM | `openai==1.70.0` | Realtime API 음성 대 음성 + 검색 결과 요약 |
| API | `fastapi==0.115.9` | 라우터·스키마. **버전 핀이 중요** |
| 런타임 | uvicorn(dev) / gunicorn+UvicornWorker(운영) | 워커 수는 [[AI음성]] 의 실행 구조 정정 참고. `_get_loop()` 가 필요한 이유 |
| 동시성 | `asyncio` | 세션당 태스크 5~7개가 동시에 돈다 |

`aiortc` 가 이 경로의 성격을 결정한다. v3 는 브라우저가 OpenAI 와 직접 붙지만 **v4 는 서버가 중간에 끼어서**(Server-In-Path) 오디오와 이벤트를 만질 수 있게 된다. 대가로 서버가 WebRTC 협상·ICE·RTP 를 직접 감당한다.

> [!warning] fastapi 핀을 풀지 말 것
> 0.141 에서 `include_router()` 가 `.path` 없는 lazy 객체를 만든다. 라우트 열거로 확인하던 테스트가 **라우트를 하나도 못 읽어서 공허하게 통과**한 적이 있다. 지금은 빈 목록이면 시끄럽게 실패하는 카나리아가 있고, 로컬 `agent_env` 가 0.141 이라 그 카나리아 1건이 상시 실패 중이다.

## 파일 지도

### v4 전용 — 이 폴더만 고치면 v3·v2 에 영향이 없다

| 파일 | 줄 | 역할 |
|---|---|---|
| `router/voice_chat_v4_router.py` | 383 | 엔드포인트 4개. 세션 생성 시 게이트 트랙과 v4 플래그를 주입 (필터 7필드 주입 가능) |
| `v4/audio_tracks.py` | 673 | 업링크 게이트(지터버퍼 + 3관문 + barge-in + 드레인) + 다운링크 지터버퍼 + 좀비 판정 |
| `v4/tool_call_orchestrator.py` | 526 | LLM 의 함수 호출을 받아 RAG 를 실행 |
| `v4/tool_result_policy.py` | 152 | 검색 결과 포맷·중복 제거·필터·재랭킹 |
| `v4/voice_context_summarizer.py` | 62 | 검색 결과를 음성 답변용으로 요약 |
| `v4/__init__.py` | 0 | 비어 있음. 죽은 재수출을 걷어냈다 |

### 공용 — v2 도 함께 탄다. 고칠 때 영향 확인 필수

| 파일 | 줄 | 내용 |
|---|---|---|
| `voice_chat/server_webrtc.py` | 2,891 | 세션 매니저 + 리스너 브릿지. ICE 픽스 · 이벤트 루프 · 트랙 주입 · 좀비 정리 · orchestrator 분기 · 세대 가드 |
| `voice_chat/webrtc_api.py` | 235 | OpenAI SDP 협상. GA(multipart) / Legacy(application/sdp) 분기 |
| `voice_router.py` | 82 | v4 등록 게이트. import 실패 시 조용히 폴백 |
| `workflow/realtime_speaker.py` | — | `session.update` · `response.create` 송신 전담 |
| `workflow/runtime.py` | — | LangGraph 실행기. `active_session_ids()` 를 스위퍼가 쓴다 |
| `realtime_instruction_policy.py` | 75 | 세션 지시문 조립. **v3 라우터도 같은 함수를 쓴다** |

> [!danger] `realtime_instruction_policy.py` 는 v3 운영 동작을 바꾼다
> `build_v3_agent_instructions()` 를 v3·v4 라우터가 함께 호출한다. 여기 프롬프트를 고치면 **운영 중인 v3 의 답변 성향이 즉시 함께 바뀐다.** 2026-08-14 의 "잡담에도 검색 선언" 수정이 그 사례다 — 사용자 결정으로 v3 동시 반영을 택했다.

### 재사용하는 기존 자산 — 새로 만들지 말 것

v4 가 import 하는 프로젝트 모듈 13개는 전부 `main` 에 이미 있던 것이다.

| 자산 | 쓰는 곳 | 무엇을 |
|---|---|---|
| `utils/whoami.py` | orchestrator, 라우터 | `get_chat_id_from_db` — UUID → 8자리 chat_id |
| `ai_voice/constants.py` | 라우터, 브릿지 | 모델 ID · 세션 config 6개. 하드코딩 0건 |
| `rag/rag_adapter.py` | orchestrator, 라우터, retriever | `vector_search` — **pgvector** 하이브리드 검색 래퍼. `chat.faiss_process.pgvector_search` 를 부른다(모듈명만 FAISS, 실제 엔진은 pgvector) |
| `rag/query_enrichment_service.py` | orchestrator, 라우터, retriever | 후속질문 문맥 보강, 스코프 조립 |
| `rag/search_options.py` | orchestrator, 라우터, retriever | 검색 옵션 조립(top_k · chat_bot_id 만) |
| `realtime_session_schemas.py` | 라우터 | GA · legacy 세션 shape 검증 |
| `realtime_instruction_policy.py` | 라우터 | 시스템 프롬프트 조립 |
| `policy_keywords.py` | tool_result_policy, query_enrichment | 키워드 정책 |
| `voice_chat/voice_logging.py` | audio_tracks | 로거 생성 |

## 호출 체인 — 어느 함수가 어느 함수를 부르는가

의미 설명은 [[AI음성 - 런타임 플로우]] 에 있다. 여기서는 **호출 순서와 위치**만 압축한다. `sw:` = server_webrtc.py, `router:` = voice_chat_v4_router.py, `orch:` = v4/tool_call_orchestrator.py, `at:` = v4/audio_tracks.py.

### ① 세션 생성

```
create_v4_agent_session()            router:221
├ _build_v3_agent_session_config()   router:143  → _build_session_ga:177 / _legacy:157
│  └ build_v3_agent_instructions()   instruction_policy:48   ※ v3 공용
├ _build_audio_filter_options()      router:91
├ create_answer()                    sw:307
│  ├ close()                         sw:466      (같은 id 선행 정리)
│  ├ _build_pc()                     sw:142      (STUN 없음)
│  ├ _ensure_sweeper()               sw:620      (프로세스 전역 1개)
│  ├ @pc.on(ice/datachannel/track)   sw:337·363·391  ※ 세대 클로저 고정
│  ├ _apply_audio_codec_preferences() sw:172
│  └ except → close() → raise        sw:450
└ voice_chat_runtime.create_session() runtime:51
```

### ② 미디어 배관

```
on_track()                           sw:391
├ _find_transceiver_for_track()      sw:906
├ _initialize_speaker_sender()       sw:891      (트랙 없이 자리만 예약)
└ attach_track()                     sw:1182
   ├ configure_session()             realtime_speaker:53
   └ create_task(_connect_realtime)  sw:1299
      ├ _input_relay.subscribe()     sw:1317  (MediaRelay)
      ├ NoiseFilteredAudioProxyTrackV4()  at:375   ← metadata.audio_filter 를 kwargs 로
      │  ├ recv() 게이트 본체         at:529   (드레인 → 패스스루 → 버퍼링·방출 판정)
      │  │  ├ _gate_passes()         at:479   (3관문 순수 검사)
      │  │  └ _process_utterance()   at:491   (정산 · force=True 관문 우회)
      │  └ except → 원본 트랙 유지    sw:1328
      ├ _start_audio_monitor()       sw:1343 호출 · 정의 sw:2371
      ├ _setup_realtime_channel()    sw:2409     (oai-events 배선)
      ├ webrtc_api.create_webrtc_session()  webrtc_api:68 → _create_ga:165 / _legacy:125
      │  └ except → pc.close() → return  sw:1393
      └ @pc.on(track) → _on_remote_audio
         └ _attach_remote_audio_track()  sw:914
            └ _replace_sender_track()    sw:997   (wait_for 2초)
```

### ③ 이벤트 분기 — `on_message` 하나가 넷을 부른다

네 갈래는 **병렬이 아니라 같은 콜백 안에서 순차 실행**된다. tool call 만 `create_task` 로 띄워 그 이후가 비동기다.

```
on_message()                         sw:2435 (콜백 클로저)
├ 가드: decode / json.loads / _is_current_generation()   sw:2418 · 2453
├ _relay_event_to_browser()          sw:2316
│  ├ _is_tool_control_event()        sw:2297     (프론트 이중 실행 차단)
│  └ _extract_realtime_token_usage() sw:2268     (response.done 시)
├ _handle_realtime_event()           sw:1746
│  ├ _handle_speech_started()        sw:1919
│  ├ _handle_speech_stopped()        sw:1952
│  └ response.done → _send_next_queued_response()  sw:2635
├ _orchestrator_for()                sw:1162     ★ v3/v4 격리 실행 지점
│  ├ extract_function_tool_call()    orch:97
│  └ schedule_function_tool_call()   orch:145
└ _extract_transcript()              sw:1646
   ├ _capture_assistant_transcript() sw:1843     (AI 대본 → 로그만)
   └ if "input_audio_transcription"  sw:2519     ★ 자기응답 루프 차단
      └ _handle_transcript_text()    sw:1961
```

### ④ 턴 → 응답

```
_handle_transcript_text()            sw:1961
├ _normalize_transcript()            sw:2010
├ _handle_speech_started()           sw:1919  (idle 이면)
│  ├ _handle_interruption()          sw:2214
│  └ cancel_tool_tasks()             orch:82
├ _merge_final_transcript()          sw:2031  (final 일 때)
└ _schedule_turn_timeout()           sw:2048  (delta 0.5s / final 1.5s)
   └ _turn_timeout_fire()            sw:2080
      └ _flush_turn_buffer()         sw:2101
         ├ GA + v4 → return          sw:2125  ★ GA 통화는 여기서 끝
         ├ 중복 턴 / pending → skip   sw:2135·2143
         └ _process_user_text_queue() sw:2179  (FIFO)
            └ _deliver_text()        sw:1485
               └ runtime.submit_user_text()  runtime:75
                  └ graph: listener → turn_gate → intent_planner → retriever
                     └ _handle_graph_response()  sw:1508
                        ├ _should_emit_response()  sw:1730
                        ├ _build_realtime_response_instructions()  sw:1556
                        ├ _enqueue_response()      sw:2580
                        └ _send_next_queued_response()  sw:2635
                           └ _issue_response_create()   sw:2539
                              └ speaker.send_response / request_response
```

### ⑤ tool call

```
schedule_function_tool_call()        orch:145
├ 가드: tools 비활성 / call_id 없음 / 중복 call_id  orch:146·149·154
├ asyncio.get_running_loop()         orch:171   (gunicorn 대응)
└ _handle_function_tool_call()       orch:306
   ├ _parse_tool_arguments()         orch:130
   ├ 가드: 도구명 / query / rag 메타   orch:318·328·341
   ├ get_chat_id_from_db()           utils/whoami.py
   ├ _SAFE_CHAT_ID_RE.match()        orch:384  → _send_tool_error() orch:285
   ├ _to_top_k()                     orch:25   (OverflowError 포획)
   ├ build_effective_query()         rag/query_enrichment_service
   ├ wait_for(vector_search(), 5.0)  orch:426  ※ results=None 선초기화
   │  └ except Timeout/Exception     orch:493·505
   ├ _build_tool_output_payload()    orch:182
   ├ summarize_for_voice()           v4/voice_context_summarizer  (남은 예산만)
   ├ _send_function_call_output()    orch:216
   └ _send_tool_continue_response()  orch:251
      └ _response_dispatcher = _enqueue_tool_continue_response  sw:2617
```

### ⑥ 종료

```
release / ICE / 스위퍼 / reap / 무프레임  →  close()  sw:466
├ 세대 불일치 → return
├ (첫 await 전 동기 구간) pop 18 + reset 3 으로 분리   sw:492~513
│  └ _cancel_disconnect_grace()      sw:602   (자기 태스크면 cancel 안 함)
├ pc.close() / _release_speaker_sender()  sw:527·982
├ _listener_bridge.detach()          sw:1230  (pop 22건)
│  └ _tool_orchestrator[_v4].clear_session()  sw:1293  (판별 없이 양쪽)
├ 2차 세대 검사 → return             sw:543
└ wait_for(runtime.release_session(), 5.0)  runtime:160
```

## 파일별 심볼

### `v4/audio_tracks.py` — 클래스 4개, 업링크 둘이 다운링크를 **상속** (8-21 미사용 탭 추가)

업링크 트랙 둘 다 `AudioProxyTrack` 을 상속해 지터버퍼·페이싱·좀비 가드를 물려받는다 (투명 탭 시절의 "상속 없음" 은 더 이상 사실이 아니다). `SilenceAudioStreamTrack` 만 별개다.

| 심볼 | 줄 | 역할 |
|---|---|---|
| `AudioProxyTrack` | 16 | 지터버퍼 기반 클래스. 업링크 사본이라 상한 **960ms**(v3 사본은 320) |
| `_ensure_reader_started` · `_reader_loop` | 101 · 128 | 업스트림을 계속 읽어 버퍼에 넣는 별도 태스크 |
| `_wait_for_prebuffer` | 181 | 재생 전 최소량 대기 |
| `_build_silence_frame` | 233 | 언더플로 때 채울 무음 |
| `_get_next_frame` | 247 | 버퍼에서 꺼냄. 비면 무음 + 언더플로 카운트, **600회(≈60초)면 좀비 종료** |
| `_pace_playout` | 313 | 실시간 속도 페이싱 |
| `_retime_frame` | 331 | `Fraction` 으로 PTS 재계산 |
| `recv` | 342 | 기반 클래스 본체 |
| `SilenceAudioStreamTrack` | 362 | 자리를 채우는 무음 소스 |
| `UplinkAudioProxyTrackV4` | 390 | **게이트 없는 업링크 탭** — 대기시간 미사용 전용. 꼬리표만 `PROXY-UP` 으로 바꾼 5줄 |
| `NoiseFilteredAudioProxyTrackV4` | 402 | **업링크 게이트** — 발화를 쥐고 관문 판정으로 방출·폐기 |
| `_DRAIN_SPEEDUP` | 420 | 드레인 배속 **4** — 버스트(수신측 앞부분 폐기)와 1배속(라이브 유실)의 절충 |
| `_calculate_rms` | 504 | 프레임 음량 |
| `_gate_passes` | 515 | 3관문 순수 검사 — 부작용 없는 **단일 출처**. barge-in 사전 검사와 정산이 같은 조건을 본다 |
| `_process_utterance` | 527 | 정산 전담(카운터 · 채택/폐기 info 로그 · 리셋) |
| `recv` | 565 | 게이트 본체: 드레인 → 패스스루 → 버퍼링(침묵 대체 송출) → 방출 판정(barge-in / 침묵 정산 / force_flush) |
| `get_diagnostics` | 704 | 필터 카운터 포함 스냅샷 |

> [!note] 게이트 파라미터 (8-20 원본 복귀 기준)
> 발화 길이 **1200ms** · 실발화(RMS>**400**) **600ms** · 비율 **30%** · 발화 종료 침묵 **400ms** · barge-in **1200ms** · force_flush **30초**. 8-18 완화값(200/120/500/1초)은 8-20 에 원본 복귀 후 재조정됐다 — 최신 표는 [[AI음성 - 런타임 플로우]] 참고. 7필드 전부 세션 생성 요청으로 주입 가능. 소음 차단 이득은 0 으로 실측됐고(OpenAI VAD 가 담당), 게이트의 실익은 시간 옵션 과제의 실동작 파라미터다.

> [!warning] 대기시간을 낮출 땐 3필드를 함께 보낸다
> `barge_in_ms` 만 낮추면 그 시점에 정산이 돌고 3관문 미달로 **발화가 폐기·리셋된다**. `min_utterance_duration_ms` = T, `min_speech_duration_ms` = T/2 를 같이 보내야 한다. 비율(`speech_ratio_threshold`)은 무차원이라 스케일하지 않는다 — 같이 줄이면 관문이 사라진다. 상세는 [[음성 선택과 대기시간 옵션]].

### `v4/tool_call_orchestrator.py`

| 심볼 | 줄 | 역할 |
|---|---|---|
| `_SAFE_CHAT_ID_RE` | 22 | SQL 테이블명에 들어가므로 안전한 식별자만 허용 |
| `_to_top_k` | 25 | top_k 파싱. `TypeError·ValueError·OverflowError` 전부 포획 |
| `clear_session` | 78 | 세션 정리. `cancel_tool_tasks` 를 함께 부른다 |
| `cancel_tool_tasks` | 82 | 끼어들기용. 처리 이력(`_processed_tool_calls`)은 남긴다 |
| `extract_function_tool_call` | 97 | 이벤트에서 tool call 추출. 타입 2종 |
| `_parse_tool_arguments` | 130 | JSON 파싱. 실패 시 원문을 query 로 간주 |
| `schedule_function_tool_call` | 145 | 중복 제거 후 별도 태스크로 스케줄 |
| `_build_tool_output_payload` | 182 | 검색 결과를 응답 형태로. 본문 300자 컷 |
| `_send_function_call_output` | 216 | 데이터채널로 결과 전송 |
| `_send_tool_continue_response` | 251 | 답변 재개. `result_count==0` 이면 자체 지식 유도, `-1`(실패)은 일반 지시 |
| `_send_tool_error` | 285 | 실패해도 같은 계약의 payload 로 끝낸다 |
| `_handle_function_tool_call` | 306 | 본체 |

### `server_webrtc.py` — v2 와 공유, 2,891줄

**클래스 2개**가 한 파일에 있다. `ServerWebRTCSessionManager`(228~) 는 브라우저 쪽 연결·세대·정리를, `_RealtimeListenerBridge`(1041~) 는 OpenAI 쪽 연결·이벤트·턴·응답 큐를 맡는다.

| 심볼 | 줄 | 역할 |
|---|---|---|
| `_get_loop` | 8 | 요청 시점 running loop. gunicorn 대응 |
| `_to_bool` | 96 | 기존 헬퍼. 새로 만들지 말 것 |
| `_is_browser_control_message` | 122 | 브라우저 제어 JSON 판별. **원문 기준** 판정 |
| `_build_pc` | 142 | STUN 없이 PC 생성. **5,011ms → 2ms** |
| `_apply_audio_codec_preferences` | 172 | 코덱 우선순위 |
| `create_answer` | 307 | 세션 생성 진입점. 세대 발급 |
| `close` | 466 | 세대 가드 + 동기 선점 정리 |
| `_schedule_disconnect_grace` | 566 | ICE 끊김 후 30초 유예 |
| `_cancel_disconnect_grace` | 602 | 유예 취소. 자기 태스크면 dict 만 비움 |
| `_ensure_sweeper` | 620 | 좀비 스위퍼 기동 (전역 1개) |
| `_reap_orphan_runtime_sessions` | 625 | PC 없이 남은 runtime 세션 회수 |
| `_sweep_zombie_sessions` | 645 | 60초 주기 전역 스위퍼 |
| `_attach_remote_audio_track` | 914 | OpenAI 음성 → 지터버퍼 → sender |
| `_replace_sender_track` | 997 | replaceTrack. 2초 상한 |
| `_orchestrator_for` | 1162 | **v3/v4 orchestrator 분기** |
| `_is_function_tools_enabled` | 1171 | metadata 3키 확인 후 기본값 |
| `attach_track` | 1182 | 브릿지에 트랙·플래그 전달 |
| `detach` | 1230 | 브릿지 정리. 양쪽 orchestrator 모두 clear |
| `_connect_realtime` | 1299 | OpenAI 와 두 번째 SDP |
| `_deliver_text` | 1485 | LangGraph 제출. 예외는 로그만 |
| `_handle_graph_response` | 1508 | 그래프 결과 → 응답 큐 |
| `_build_realtime_response_instructions` | 1556 | 턴별 지시문 |
| `_extract_transcript` | 1646 | 사용자·AI 전사 양쪽 추출 |
| `_should_emit_response` | 1730 | (턴, 텍스트) 중복 차단 |
| `_handle_realtime_event` | 1746 | 상태 이벤트 처리 |
| `_capture_assistant_transcript` | 1843 | AI 대본 로그 |
| `_handle_speech_started` | 1919 | 새 턴 + 끼어들기 |
| `_handle_transcript_text` | 1961 | 턴 버퍼 축적 |
| `_merge_final_transcript` | 2031 | 중간본·최종본 이중 기록 제거 |
| `_schedule_turn_timeout` | 2048 | 0.5s / 1.5s 재예약 |
| `_flush_turn_buffer` | 2101 | 턴 확정. **가드 3종** |
| `_process_user_text_queue` | 2179 | FIFO 워커. 자기 태스크 확인 후 재스폰 |
| `_handle_interruption` | 2214 | 큐 clear + pending 해제 |
| `_is_tool_control_event` | 2297 | 중계 제외 판정 |
| `_relay_event_to_browser` | 2316 | 원문 중계. 예외 밖으로 안 냄 |
| `_start_audio_monitor` | 2371 | 입력 프레임 감시 |
| `_setup_realtime_channel` | 2409 | OpenAI 채널 배선 (세대 고정) |
| `_issue_response_create` | 2539 | 실제 응답 지시 |
| `_enqueue_response` | 2580 | 큐 적재 (merge) |
| `_enqueue_tool_continue_response` | 2617 | tool 재개를 같은 큐로 |
| `_send_next_queued_response` | 2635 | pending 이면 꺼내지 않음 |

### v4 라우터

| 엔드포인트 | 핸들러 | 줄 |
|---|---|---|
| `POST /agent/session` | `create_v4_agent_session` | 236 |
| `POST /agent/session/{id}/release` | `release_v4_agent_session` | 309 |
| `GET /agent/config` | `get_v4_agent_config` | 320 |
| `POST /agent/tools/search-knowledge-base` | `run_v4_search_knowledge_tool` | 335 |

접두사는 `/AI_voice/voice-chat-v4` 다. 게이트는 `voice_router.py` 가 쥐고 있고 **기본 활성**이다. 끄려면 `ENABLE_VOICE_V4=off`. 알 수 없는 값이면 조용히 꺼지지 않고 기본값을 유지한다.

###### 세션 요청 필드 (2026-08-21 `c5e754b`)

`VoiceChatV4AgentSessionRequest` 가 프론트와의 계약이다. 필터 7필드 외에 세 가지가 여기서 갈린다.

| 필드 | 기본값 | 비고 |
|---|---|---|
| `model` | `OPENAI_DEFAULT_MODEL` (`gpt-realtime-2.1`) | 구 기본값 `gpt-realtime-1.5` 는 **폐지된 Beta 평면 스키마** 경로였다 |
| `voice` | `None` (세션 설정의 `cedar`) | `RealtimeVoice` Literal 10종. 허용값 밖은 FastAPI 가 422 |
| `wait_time_enabled` | `True` | `False` 면 `UplinkAudioProxyTrackV4` 주입 + 필터 임계값 미주입 |

`voice` 목록은 OpenAPI 스키마에 그대로 실린다 — **별도 목록 API 는 없고, `/docs` 가 단일 출처다.**

## 손댈 때 규칙

### 절대 건드리지 말 것

`ai_voice/services/voice_chat/v3/` 는 `origin/main` 과 **바이트 단위로 같아야** 한다. `test_v4_integration_smoke.py` 가 이걸 강제한다(라우터는 별도 브랜치 수정이 있어 제외). v4 를 고치려고 v3 파일을 여는 순간 뭔가 잘못된 것이다.

### 어느 파일을 고쳐야 하는가

| 고치려는 것 | 여기를 | 영향 |
|---|---|---|
| v4 오디오 동작 | `v4/audio_tracks.py` | v4 only |
| v4 RAG 동작 | `v4/tool_call_orchestrator.py` | v4 only |
| v4 결과 가공 | `v4/tool_result_policy.py` | **v3 사본과 동일. 파리티 테스트 확인** |
| v4 요약 | `v4/voice_context_summarizer.py` | **v3 사본과 동일. 파리티 테스트 확인** |
| 세션 수명주기·턴·응답 큐 | `server_webrtc.py` | **v2 도 영향** |
| 답변 성향·프롬프트 | `realtime_instruction_policy.py` | **v3 운영도 함께 바뀜** |
| 세션 설정·모델·VAD | `ai_voice/constants.py` | **v1~v4 전부 영향** |
| v4 엔드포인트 | `voice_chat_v4_router.py` | v4 only |

### 파리티가 걸린 파일

`tool_result_policy.py`(152줄)와 `voice_context_summarizer.py`(62줄)는 v3 사본과 **지금 바이트 동일**하다. 한쪽만 고치면 테스트가 실패하며 diff 를 보여준다. 선택은 둘 중 하나다.

- 같은 수정을 양쪽에 반영한다. 대부분 이쪽이다.
- v4 만 의도적으로 갈라진다면 `PARITY_COPIES` 목록에서 빼고 이유를 커밋 메시지에 남긴다.

`tool_call_orchestrator.py` 와 `audio_tracks.py` 는 이미 실질적으로 갈라져서 목록에 없다.

### 새 코드를 넣기 전에

- 공용 자산을 먼저 찾는다. `utils/`, `ai_voice/constants.py`, `ai_voice/infrastructure/utils/`, `rag/` 에 이미 있는 경우가 많다.
- 불린 파싱은 `server_webrtc._to_bool` 이 이미 있다.
- 정수 변환은 `_to_top_k` 를 참고한다. 공용 `safe_int` 는 `OverflowError` 를 안 잡는다.
- 성공 경로에 `logger.warning` 을 쓰지 않는다. 로그 폭탄으로 운영 로그를 못 읽게 만든 전례가 있다.
- 루프는 항상 `_get_loop()` 를 거친다. `self._loop.create_task` 직접 호출은 테스트가 막는다.

## 함정 네 가지

> [!danger] 1 · `v3/` 폴더 파일을 쓰는 건 v3 가 아니라 v2 다
> v3 라우터는 `webrtc_api.create_webrtc_session` 을 직접 부르고 `server_webrtc` 를 아예 안 거친다. `server_webrtc` 를 쓰는 건 **v2 와 v4** 뿐이다. 폴더 이름에 속지 말 것.

> [!danger] 2 · 같은 이름의 클래스라도 파일이 다르면 별개다 — 상속 관계는 8-14 에 다시 생겼다
> v3 와 v4 의 `AudioProxyTrack` 은 이름만 같은 별개 클래스다. 좀비 스위퍼를 v3 파일에 넣었다가 v4 에서 전혀 동작하지 않은 적이 있다. 그리고 v4 파일 안의 상속 관계는 **시기별로 뒤집혔다** — 원본 게이트는 상속, 8-11 투명 탭은 비상속, 8-14 게이트 복원으로 **다시 상속**(`NoiseFilteredAudioProxyTrackV4` → `AudioProxyTrack`). 현재는 `test_orchestrator_isolation.py` 가 "v4 트랙이 같은 파일의 `AudioProxyTrack` 을 상속하고 v3 트랙과는 무관함" 을 단언한다. 옛 문서를 볼 때는 그 문서가 어느 시기 기준인지부터 확인할 것.

> [!danger] 3 · tool call 의 모든 종료 경로는 응답을 보내야 한다
> `_handle_function_tool_call()` 은 태스크로 돌기 때문에 예외가 조용히 삼켜진다. 응답을 안 보내고 끝나면 **통화가 그대로 멈춘다.** 실제로 두 번 터졌다(`results` 미바인딩, `top_k` OverflowError). 자세한 내용은 [[AI음성 - 런타임 플로우]] 의 6단계 참고.

> [!danger] 4 · GA 세션은 서버 턴 파이프라인을 타지 않는다
> `_flush_turn_buffer` 가 GA + v4 조건에서 즉시 반환한다. 턴 타이머·LangGraph·응답 큐에 코드를 넣어도 **현재 운영(GA) 경로에서는 실행되지 않는다.** 반대로 v2 GA 세션은 LangGraph 가 유일한 응답 경로라, 이 가드에서 v4 마커를 빼면 v2 가 무응답이 된다.

## 테스트 지도 · 175건

| 파일 | 건수 | 잠그는 것 |
|---|---|---|
| `test_lost_bugfixes.py` | 19 | chat_id 가드, top_k 파싱, 요약 타임아웃, GA 모델 분기 |
| `test_browser_event_relay.py` | 16 | 이벤트 원문 중계, tool 제어 필터, 토큰 사용량 |
| `test_response_queue_contention.py` | 14 | 응답 큐 경합, pending 유실 방지 |
| `test_ga_model_routing.py` | 12 | GA 판정 접두사, 세션 shape 분기 |
| `test_audio_track_injection.py` | 11 | 트랙 주입 배관, 협상 실패 시 상태 누수 |
| `test_orchestrator_isolation.py` | 10 | v4 세션이 v4 orchestrator 를 타는지 + 게이트 상속 관계. 실제 호출로 확인 |
| `test_zombie_session_sweeper.py` | 8 | 유예 타이머·스위퍼 완주, 재사용 id 오종료 방지 |
| `test_v4_integration_smoke.py` | 8 | v3 바이트 동일성, 사본 드리프트, 라우트 등록 |
| `test_v4_ga_direct_response.py` | 8 | GA 즉답 경로, LangGraph 스킵 가드 |
| `test_browser_control_and_text_fifo.py` | 8 | 제어 JSON 차단, FIFO 직렬화·재스폰 |
| `test_v4_filter_tuning.py` | 7 | 필터 임계값 세션 주입 |
| `test_session_generation_guard.py` | 7 | 늦은 콜백 세대 가드. 실제 지연을 만들어 재현 |
| `test_v4_filter_gate_and_drain.py` | 6 | 게이트 P1 회귀(누적 유지·force 우회·barge-in·침묵 정산·드레인 4배속) + 완화 기본값 고정 |
| `test_v4_router_gate.py` | 6 | 게이트 on·off·기본값·미상값 |
| `test_v4_audio_track_logging.py` | 6 | 업/다운링크 태그 구분 |
| `test_v3_router_chat_id_guard.py` | 6 | v3 라우터 chat_id 검증 |
| 나머지 6파일 | 23 | 로거 핀, 필터 판정 로그(force 우회 포함), GA session.update 스킵, 패키지 import, 이벤트 루프 |

```bash
pytest ai_voice/voice_tests/ -q
```

현재 **174 passed / 1 failed** 이며, 실패 1건은 로컬 `agent_env` 의 fastapi 가 핀(0.115.9)과 어긋나(0.141.1) 라우트 열거가 공허해지는 환경 이슈다 — 소스 문제가 아니다. 투명 탭 전용이던 `test_v4_uplink_transparent_tap.py` 는 게이트 복원과 함께 삭제됐다.

> [!tip] 테스트를 믿을 수 있게 유지하는 법
> 이 스위트의 가드는 대부분 **변이 테스트로 실효성을 확인**했다. 새 가드를 추가하면 같은 절차를 밟는다 — 지키려는 코드를 일부러 망가뜨리고, 그 가드만 실패하는지 본다. 소스 문자열만 검사하는 테스트는 조건이 뒤집혀도 통과한 전례가 있다.

## 관련 노트

- [[AI음성]] — 프로젝트 진행 상황
- [[AI음성 - 폴더·파일 역할 지도]] — ai_voice **전체**(STT·학습·회의록 포함)의 폴더·파일 역할. 이 노트는 v4 심층 전용이다
- [[AI음성 - 런타임 플로우]] — 실행 흐름의 의미와 예외 경로
- [[v3와 v4 아키텍처]] — v3 와의 구조적 차이
- [[운영에서만 터지는 문제들]] — 이벤트 루프 불일치 등
