---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# 모바일 정상 동작 확인

> [!summary] 한 줄 요약
> [[AI음성 - 모바일 ICE 원인 확정]]의 프론트 패치를 dev 에 반영하고 **폰 실기기에서 v4 음성 대화가 정상 동작하는 것을 확인**했다 — 과제 1번(AI음성 모바일 오류 수정)의 목표 달성. 프로젝트: [[AI음성]]

## 무슨 작업

`webrtc_voice_module.js` 의 `connect()` 에 ICE 후보 수집 대기를 넣고, 세션 POST 를 수집 전의 `offer.sdp` 대신 **후보가 포함된 `pc.localDescription.sdp`** 로 바꿨다. 서버(aiortc)가 trickle ICE 를 지원하지 않으므로 오퍼에 후보가 실려 있어야 서버가 먼저 연결 체크를 시작할 수 있다.

```
기존:  setLocalDescription(offer) → 즉시 POST(offer.sdp)      ← 후보 0개
수정:  setLocalDescription(offer) → 수집 완료 대기(상한 2초)
       → POST(pc.localDescription.sdp)                        ← 후보 포함
```

폴백 상한 2초를 둬서 STUN 이 늦어도 진행은 된다. 수집은 구글 STUN 기준 보통 0.1~0.5초라 체감 지연이 없다.

## 배포 절차

레포 커밋이 아니라 dev 프론트 미러 직반영이다.

| 단계 | 내용 |
|---|---|
| 사전 대조 | 로컬 미러(`AI_pro/voice`)와 배포본을 HTTP 로 받아 **sha256 완전 일치** 확인 (55,714B) |
| 패치 | `connect()` 수정, `node --check` 통과, 55,714B → 57,040B |
| 업로드 | FileZilla 로 `/home/dev.han.kr/www/chat/voice/js/webrtc_voice_module.js` 교체 |

## 검증

**폰(안드로이드) 실기기에서 v4 음성 대화 정상 동작.** 이 프로젝트에서 모바일이 동작한 최초의 순간이다. 데스크톱·사무실 망 회귀 없음(같은 코드 경로가 더 견고해질 뿐).

## 남은 정리

- [ ] **Gitea `Ji-Hwan/frontend` 정식 PR** — dev 미러 반영은 테스트용. 같은 패치를 저장소에 올려야 운영에 간다
- [ ] JS 참조부의 `?v=` 캐시 파라미터 증가 — 실사용자 브라우저가 옛 파일을 캐시하고 있을 수 있다
- [ ] 백엔드 main PR (`fix/v4-session-generation-guard`)
- [ ] (선택·보강) 인프라 — 외부발 신규 인바운드 UDP 허용. 대칭형 NAT 통신사 대비
