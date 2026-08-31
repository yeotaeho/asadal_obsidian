---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-27
---

# 음성선택 전 구간 구현

> [!summary] 한 줄 요약
> 기획서 V1.1 의 모델 4종 × 음성 10종 + 관리자 대기시간을 백엔드·본체·프론트 6파일 전 구간에 구현했다 — 서브에이전트 9태스크, **217 passed**, 최종 리뷰 clean to merge. 기능: [[음성 선택과 대기시간 옵션]]

## 무슨 작업

브레인스토밍으로 설계를 확정(스펙 `2026-08-27-voice-selection-design.md`)하고 9태스크 계획을 서브에이전트 주도 개발(SDD)로 실행했다. 태스크마다 구현자·리뷰어를 분리 디스패치하고 픽스 라운드를 돌렸다.

| # | 내용 | 위치 | 결과 |
|---|---|---|---|
| 1 | 관리자 음성 설정 로더 (greeting 패턴) | `voice_admin_config.py` 신규 | 픽스 1라운드(죽은 코드 3건) 후 clean |
| 2 | model Literal 4종 + 미지정(None) 감지 | constants + v3·v4 라우터 | clean |
| 3 | 세션 캐스케이드(사용자>관리자>코드) + 대기시간 서버 파생 | v4 라우터 | clean |
| 4 | `GET /voice-chat-v4/voice-settings` 조회 | v4 라우터 | clean |
| 5 | 본체 chatbot_config 5컬럼 배선 | Chatty_Project | **worktree 분리 작업** — 사용자 미커밋 변경 보호 |
| 6 | `window.VoiceSettings` 저장소 + voice 유실 수복 | voice_runtime.js | clean |
| 7 | connect 재해석·dialect 확장·잠복버그 2건·GA 가드 | webrtc_voice_module.js | clean |
| 8 | 환경설정 메뉴(모델→음성 2단) + sys_voice | index2.js·chatting.htm | 픽스 1라운드(외부 ul 게이트) 후 clean |
| 9 | 인터페이스 계약 문서 + 검증 체크리스트 | `docs/VOICE_SETTINGS_INTERFACE.md` | clean |

## 왜

기획서 V1.1 확정에 따라 사용자 선택(모델·음성)과 관리자 초기값(모델·음성·대기시간·on/off)을 잇는 배관이 필요했다. 관리자 값 저장 평면이 본체 `chatbot_config`(admin 페이지가 `save_chatbot_config` 로 저장하는 것을 실확인)로 확정되면서, **대기시간 파생 규칙(T→3필드)을 서버 단일 소유**로 옮겨 프론트가 틀릴 여지를 구조적으로 제거했다.

## 어디

| 저장소 | 브랜치/위치 | 커밋 |
|---|---|---|
| AI_Pro_VOICE | `feat-voice-selection-api` | `ba3d87d`..`9e13c6e` (8커밋 — 스펙·계획 포함) |
| 본체 Chatty_Project | worktree `Chatty_Project-voice-config`, 브랜치 `feat/voice-selection-config` | `8c64dfc` |
| 프론트 | `카카오톡 받은 파일/` 6파일 수정 (`.bak` 백업 보존) | git 없음 — 파일질라 배포 예정 |

## 검증

- `pytest ai_voice/voice_tests tests` — **217 passed** (매 태스크 + 최종).
- 태스크별 스펙·품질 리뷰 9회 + 픽스 라운드 2회 + 최종 전체 브랜치 리뷰 — **clean to merge** (통합 seam 6종 전부 PASS, 이연 minor 5건 전부 accept 판정).
- Codex 리뷰 2회(백엔드 중간·최종 범위) — actionable regression 없음.
- 런타임 검증은 dev 배포 후 수동 체크리스트 8항목(`docs/VOICE_SETTINGS_INTERFACE.md`)이 담당.

## 특기 사항

- Task 8 구현자가 스스로 결함(음성 설정만 켠 봇에서 메뉴 미노출)을 발견해 픽스 라운드로 해소했다.
- 채팅 UI 는 프론트 담당이 인수할 수 있어 `voice-selection` 경계 주석으로 블록 분리했고, 인수 계약(`window.VoiceSettings` + 조회 API)을 문서에 남겼다.
