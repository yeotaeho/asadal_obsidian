---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-06
---

# v4 격리 복원 (Task 1~7)

> [!summary] 한 줄 요약
> 2026-07-13 전면 롤백된 음성채팅 v4를 7개 태스크로 격리 복원했다. 태스크마다 이중 리뷰(Claude + Codex)와 수정 라운드를 거쳐 **41/41 테스트 통과**, 최종 전체 브랜치 리뷰까지 클린. 프로젝트: [[AI음성]]

## 무슨 작업

브랜치 `feat/v4-isolated-restore`(origin/main 기준)에 18커밋으로 v4를 복원했다. 진행 방식은 서브에이전트 구동 개발 — 태스크별로 구현 에이전트 → 이중 리뷰(superpowers + Codex) → 수정 라운드 → 범위 한정 재리뷰를 반복하고, 마지막에 전체 브랜치 리뷰(최상위 모델)로 마감했다.

수정 라운드가 잡은 Critical은 총 **4건**, 그중 하나는 원본 PR #590에도 있던 미수정 잠복 버그였다. 전체 경위는 [[v4 복원 작업 현황]] 참고.

## 왜

과거 롤백의 근본 원인이 "v4가 v3 운영 라우터를 하이재킹"한 것이었다. 그래서 이번 복원은 네 가지 전역 제약을 강제했다.

- v3 라우터 불가침 (byte-identical, 테스트로 고정)
- 공용 voice 설정(marin) 불변
- 성공 경로 `logger.warning` 금지 (과거 로그 폭탄 사고 재발 방지)
- 기본 비활성 — `ENABLE_VOICE_V4` 게이트 뒤에만 등록

롤백에 휩쓸려 유실된 유효 수정 4건(chat_id 변환, GA 모델 대응, gunicorn event loop, 좀비 스위퍼)도 복원 대상이었다. 관련 배경은 [[이벤트 루프]], [[gunicorn 멀티워커 구조]], [[aiortc와 서버측 WebRTC]] 참고.

## 어디

### Task 1 — v4 패키지 복원 (미등록)

| 커밋 | 내용 | 파일 |
|---|---|---|
| `63ac694` | ICE 수집 5초 지연 제거 (선행 조건, 브랜치 맨 앞 흡수) — 상세 [[AI음성 - 서버 ICE 수집 5초 지연 제거]] | `server_webrtc.py` |
| `a8e7ea3` | `backup/voice-v4`에서 v4 패키지 10개(1,909줄) + 라우터(333줄) 복원. 어디에도 미등록 = 동작 무변화. warning 5곳 debug 강등. `aiortc==1.9.0` requirements 명시 | `ai_voice/services/voice_chat/v4/`, `voice_chat_v4_router.py`, `requirements.txt` |
| `498c4b4` | 루트 `.gitignore`의 bare `tests/` 패턴이 모든 tests 디렉터리를 무시하던 함정 해소 | `.gitignore` |
| `4924ec4` | 리뷰 수정 — `v4/audio_tracks.py` 정상 경로 warning 7곳(발화 통과, 2초 주기 상태 등) → debug 6 + info 1. 이상 징후 5곳은 warning 유지 | `ai_voice/services/voice_chat/v4/audio_tracks.py` |

### Task 2 — 유실된 버그 수정 복원

| 커밋 | 내용 | 파일 |
|---|---|---|
| `3fc2f5f` | chat_id UUID→정수 변환 + GA 모델 `response.create` 분기(modalities/voice/output_audio_format 미지원 대응) 복원 | `v3/tool_call_orchestrator.py`, `workflow/realtime_speaker.py` |
| `5c4f5b3` | 리뷰 **Critical 2건** 수정 — ① 조회 None 시 UUID가 테이블명에 보간되던 fall-through 차단 ② `_SAFE_CHAT_ID_RE` 식별자 검증. Important — `summarize_for_voice`(LLM 호출)를 타임아웃 예산 안으로. hasattr뿐이던 테스트를 행위 테스트 6건으로 교체 | `v3/tool_call_orchestrator.py`, `ai_voice/tests/test_lost_bugfixes.py` |

### Task 3 — gunicorn event loop 픽스 복원

| 커밋 | 내용 | 파일 |
|---|---|---|
| `2a5cf6e` | `_get_loop()` 헬퍼 + 스케줄링 호출부 16곳 전부 경유 (원 커밋 `dd67c19` 유실분). 임포트 시점 idle 루프에 create_task가 걸려 실행 안 되던 문제 | `server_webrtc.py` |
| `c7919e3` | conftest의 전체 모듈 페이크 제거, 실제 leaf 의존성(openai, audio_normalizer)만 스텁 | `ai_voice/tests/conftest.py` |
| `c738e83` | 직접 루프 호출 회귀를 잡는 소스 검사 가드 (위반 줄 번호 보고) | `ai_voice/tests/test_event_loop_helper.py` |

### Task 4 — 오디오 트랙 주입 배관

| 커밋 | 내용 | 파일 |
|---|---|---|
| `fbe413d` | `create_answer(..., audio_track_cls=None)` → 브리지 → `_connect_realtime` 세션별 주입. None이면 bit-identical. v4 원본의 전역 주입 대신 세션별 설계 | `server_webrtc.py` |
| `018fd0e` | 리뷰 수정 — ① SDP 협상 실패 시 세션 상태 영구 잔존(기존 v2 누수 포함) → except에서 `close()` 후 re-raise ② 필터 생성자 예외 시 세션 사망 → 무필터 강등 | `server_webrtc.py`, `ai_voice/tests/test_audio_track_injection.py` |

### Task 5 — 좀비 세션 스위퍼 복원 (모바일 직결)

| 커밋 | 내용 | 파일 |
|---|---|---|
| `0b2a6fe` | PR #590 유실분 복원 — 30초 disconnect 유예 타이머, 60초 지연 시작 스위퍼, 고아 runtime 리퍼, `_MAX_SILENT_UNDERFLOW=600`(원본의 카운터 리셋 누락을 처음부터 반영) | `server_webrtc.py`, `workflow/runtime.py`, `v3/audio_tracks.py` |
| `4c16424` | 리뷰 **Critical 2건** 수정 — ① 유예 타이머가 close() 호출 시 자기 자신을 cancel해 정리가 중단되던 잠복 버그(원본 PR에도 존재, asyncio 재현으로 입증) → `current_task()` 가드 ② session_id 재사용 시 산 세션 오살 → pc 동일성 확인. Important — 언더플로 임계 도달 시 `stop()` 호출, `release_session` 5초 타임아웃 | `server_webrtc.py`, `v3/audio_tracks.py`, `ai_voice/tests/test_zombie_session_sweeper.py` |

### Task 6 — v4 라우터 조건부 등록

| 커밋 | 내용 | 파일 |
|---|---|---|
| `41b3703` | `ENABLE_VOICE_V4` 게이트 뒤 등록 (기본 꺼짐). v3 라우터 byte-identical 유지 | `ai_voice/voice_router.py`, `ai_voice/tests/test_v4_router_gate.py` |
| `fda7f7d` | 리뷰 수정 — 무조건 import가 aiortc 폴백 설계를 우회해 미설치 환경에서 앱이 죽던 문제. 배포 워크플로에 pip install 단계가 없어 requirements 핀만으로 방어 불가로 판정 → try/except + None 게이트 | `ai_voice/voice_router.py` |

### Task 7 — 통합 스모크 테스트

| 커밋 | 내용 | 파일 |
|---|---|---|
| `23c2eef` | 아키텍처 가드 6종 — v4→필터 배선, v4가 v3 미참조(롤백 원인 재발 방지), v3 브릿지리스, v3 byte-identical(git diff), 로그 레벨, 게이트 ON 시 양쪽 경로 노출 | `ai_voice/tests/test_v4_integration_smoke.py` |
| `c5483b9` | 리뷰 수정 — 리로드 위생, 호출 지점 정규식(false-pass 차단), 마커 3종 각각 + 별칭 로거, 경로 정확 일치 | 동일 |
| `5a38f5d` | 최종 리뷰 must-fix — 테스트 venv의 fastapi 드리프트(0.115.9→0.141.1)로 "기본 꺼짐" 증명 테스트가 빈 리스트로 공허 통과하던 문제. `app.openapi()["paths"]` + 빈 결과 카나리아, 0.141.1에서 시끄럽게 실패함을 트립 검증. conftest에 환경 핀 문서화 | `test_v4_router_gate.py`, `test_v4_integration_smoke.py`, `conftest.py`, `voice_router.py`(로그 문구 1단어) |

## 검증

- `pytest ai_voice/tests/` — **41/41 통과** (핀 버전 fastapi==0.115.9)
- 태스크별 이중 리뷰(superpowers + Codex) 7회 + 수정 라운드 8회 — 전부 클린 종료
- 최종 전체 브랜치 리뷰 — **READY-WITH-NOTES → 노트 해소 완료**
- 5개 불변식 검증 — v3 라우터·constants blob 동일, 게이트 이중 조건, warning 26개 전부 이상 경로, `_get_loop` 19곳 / 직접 호출 0곳
- 교차 태스크 합성 검증 — Task 4 teardown × Task 5 스위퍼 정리 순서, double-close 멱등, 협상 중 세션 오살 불가
- 미해결 — push 인증 만료로 마지막 9커밋 로컬 대기. 이후 dev PR(`app restart`) → `ENABLE_VOICE_V4=on` → 모바일 실기기 검증 순
