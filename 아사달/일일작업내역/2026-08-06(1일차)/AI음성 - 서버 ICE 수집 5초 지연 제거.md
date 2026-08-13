---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-06
---

# 서버 ICE 수집 5초 지연 제거

> [!summary] 한 줄 요약
> 서버측 RTCPeerConnection 생성마다 고정 소모되던 ICE 수집 지연을 STUN 제거로 **5,004ms → 6ms**로 줄였다. v4 롤백 사유("버튼 무반응")의 유력한 실체. 프로젝트: [[AI음성]]

## 무슨 작업

`server_webrtc.py`에 `_build_pc()` 헬퍼를 만들어 기본값을 `RTCConfiguration(iceServers=[])`(STUN 없음)로 바꿨다. `ENABLE_SERVER_STUN=on`으로 기존 동작 복귀가 가능하다.

적용 지점은 두 곳 — `create_answer()`(브라우저↔서버)와 `_connect_realtime()`(서버↔OpenAI). v4 세션은 두 연결을 모두 만들므로 세션당 **최대 10초 → 약 10ms**가 된다.

## 왜

진단 측정에서 dev 서버의 SDP answer 생성이 5042~5066ms, 편차 20ms 이내로 나왔다. 편차가 없다는 것은 지연이 아니라 **상수 타임아웃**이라는 뜻이다.

원인은 aioice 소스에서 확인했다.

```
RTCPeerConnection()  (인자 없음)
  → aiortc 기본 STUN 사용
  → aioice get_component_candidates(..., timeout=5)
      ifaddr로 모든 어댑터 열거 (운영 서버 11개)
      IPv4 주소마다 STUN 질의 태스크
      asyncio.wait(tasks, timeout=5)   ← 하나라도 미응답이면 5초 소진
```

인터넷에 못 나가는 인터페이스가 하나만 있어도 5초를 꽉 채운다 (dev는 host=11 / srflx=10, 최소 1개 미도달). aiortc는 비-trickle이라 그동안 `setLocalDescription()`이 블록되고, gunicorn 워커 4개가 차례로 묶여 **동시 4세션이면 앱 전체 응답 불능**이 된다 — 진단 중 실제 dev 다운 사고로 재현됐다. 배경은 [[gunicorn 멀티워커 구조]], [[운영에서만 터지는 문제들]] 참고.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `a011fca` | dev용 원본 (PR #736 머지) | `ai_voice/services/voice_chat/server_webrtc.py` (+41 −2) |
| `63ac694` | main 기준 재생성 — `feat/v4-isolated-restore` 첫 커밋으로 흡수 (v4의 선행 조건) | 동일 |

## 검증

STUN 제거의 안전 근거를 둘 다 실측했다.

- 서버는 공인 IP라 srflx == host. 모바일 4G 실기기에서 실제 선택 경로도 "클라이언트 srflx ↔ **서버 host**"였다 — 서버 srflx는 만들어놓고 쓰이지도 않았다 ([[STUN과 NAT]])
- 서버→OpenAI 구간: NAT 뒤 로컬 PC에서 srflx 없이(사설 host만) `gpt-realtime-2` 연결 **성공**. OpenAI가 공인 host 후보를 제공하고 우리 주소는 peer-reflexive로 학습한다 ([[aiortc와 서버측 WebRTC]])
- 검증 스크립트 **7건 통과** — 기본값 6ms / `ENABLE_SERVER_STUN=on` 시 5004ms + srflx 생성 / host 후보 수 동일(연결성 손실 없음)
