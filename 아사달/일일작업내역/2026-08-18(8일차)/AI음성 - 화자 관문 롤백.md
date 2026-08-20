---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-18
---

# 화자 관문 롤백

> [!summary] 한 줄 요약
> 화자 관문 Phase 1 커밋 7개를 브랜치에서 걷어내고 **계획·백업·데모만 남겼다** — 도입 여부는 회의에서 결정하기로. 코드는 `backup/speaker-gate-phase1` 브랜치에 온전히 보존. 프로젝트: [[AI음성]]

## 무슨 작업

[[AI음성 - 화자 관문 Phase 1 구현]] 커밋 7개(`13e1e4d`..`7a4ba0a`)가 **미푸시 로컬 상태**임을 확인하고 `git reset` 으로 걷어냈다. 브랜치는 게이트 임계값 완화(`336dcdc`, dev 배포본과 동일)로 복귀. 기존 미커밋 변경(aiortc 1.14 핀)은 스태시로 보존 후 복원했다.

## 남긴 것 (회의 자료)

| 자산 | 위치 |
|---|---|
| **코드 전체 (커밋 7개)** | 로컬 브랜치 `backup/speaker-gate-phase1` — 승인 시 머지로 즉시 복원 |
| 설계 스펙 | `docs/superpowers/specs/2026-08-18-speaker-gate-design.md` (untracked) |
| 구현 계획 + 모델 벤치 결과 | `docs/superpowers/plans/2026-08-18-speaker-gate.md` (untracked) |
| **실연 데모** | `Downloads/speaker_gate_live_test.py` + `speaker_gate.py` 사본 + eres2net 모델 — 저장소와 무관하게 동작, 회의에서 두 사람 대화 시연 가능 |
| 검토·실측 기록 | [[AI음성 - 화자 구분 실시간 적용 검토]] · [[AI음성 - 화자 관문 Phase 1 구현]] |

## 회의에 가져갈 판단 재료

- **동작 확인됨**: 실사용 테스트에서 교대 발화는 정상 구분. 감정 실험의 본인 최저 유사도 **0.4** → 임계값 단독으론 부족, 등록 강화(2~3발화 평균) 병행 필요.
- **한계 확정**: 겹침(동시 발화)은 이 방식으로 못 가름 — ElevenLabs 포함 diarization 공통 한계. 제거는 음원 분리(TSE, PoC 1~2주 급, GPU 가능성) 별도 과제.
- **리소스**: GPU 불필요, RAM ~180MB/프로세스, CPU 발화당 40~120ms, 모델 37.8MB(Apache 2.0).

## 검증

- 롤백 후 `pytest ai_voice/voice_tests/` — **174 passed** (화자 관문 이전과 동일, 환경 실패 1건 별개)
- 브랜치 HEAD = `336dcdc` = origin 및 dev 배포본과 동일
- 데모 스크립트는 Downloads 사본 참조로 전환해 롤백 후에도 동작
