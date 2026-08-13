---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-06
---

# ICE 진단 라우터

> [!summary] 한 줄 요약
> v4 모바일 실패 원인을 특정하려고 브라우저↔서버 aiortc 연결만 격리 측정하는 진단 엔드포인트를 만들어 dev에 배포했고, 실기기 테스트로 TURN 부재 가설을 기각했다. 프로젝트: [[AI음성]]

## 무슨 작업

`/AI_voice/diag/ice/*` 임시 진단 엔드포인트를 만들었다. OpenAI·RAG·소음필터를 전부 배제하고 "브라우저 ↔ 서버 aiortc" WebRTC 연결 성립만 측정한다.

health(aiortc 가용성·실효 ICE 서버·has_turn), session(SDP 교환 + candidate 종류 집계), 세션별 ICE 상태 전이 기록, close 네 가지 경로로 구성했다. `ENABLE_ICE_DIAG` 게이트로 잠그고 aiortc 미설치 시 503 폴백을 넣었다.

## 왜

v4 롤백 사유가 "모바일에서 [음성] 버튼 무반응"이었다. 서버·브라우저 양쪽에 TURN이 없어 모바일 캐리어 NAT를 못 뚫는다는 가설을 세웠는데([[STUN과 NAT]] 참고), 검증에는 실기기 측정이 필요했다.

v4 전체(2,700줄)를 되살려 테스트하면 변수가 너무 많아 원인을 특정할 수 없으므로, 연결 성립 여부만 분리해 재는 최소 도구를 택했다. 연결 수립 흐름은 [[WebRTC 연결 과정]] 참고.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `7441d82` | 진단 라우터 신규 (+294) + voice_router 등록 (+3) | `ai_voice/infrastructure/api/router/ice_diag_router.py`, `ai_voice/voice_router.py` |
| `e1bac9a` | 활성 스위치를 공용 `.env`에서 모듈 내부 기본값으로 이동. PeerConnection 누수(pop만 하고 close 미호출) 수정, 세션 상한·TTL 도입. dev 다운 사고 후 기본 비활성 + 동시 생성 1건 + 타임아웃으로 재강화 | `ai_voice/infrastructure/api/router/ice_diag_router.py` |

dev 머지는 PR #728. ICE 가설이 기각되어 역할이 끝났으므로 main에는 보내지 않기로 결정했다.

## 검증

- 로컬 스모크 테스트 **14건 통과** — 운영과 동일한 fastapi 0.115.9, aiortc를 클라이언트로 SDP 교환~ICE 연결까지 ([[aiortc와 서버측 WebRTC]] 참고)
- 모바일 실기기(갤럭시 / Android / **4G 셀룰러**): 서버 직결 **성공** — 선택 경로 `클라이언트 srflx ↔ 서버 host (udp)`
- 결론: **TURN 가설 기각**. 대신 서버 응답이 매번 정확히 5초라는 단서를 확보해 [[AI음성 - 서버 ICE 수집 5초 지연 제거]]로 이어짐
