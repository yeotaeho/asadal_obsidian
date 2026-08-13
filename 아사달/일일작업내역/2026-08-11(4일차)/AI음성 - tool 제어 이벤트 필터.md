---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-11
---

# tool 제어 이벤트 필터

> [!summary] 한 줄 요약
> 서버와 프론트가 같은 함수 호출에 이중으로 응답하던 것을 중계 필터로 차단했다. 다만 "다른 창/기기" 오류의 진짜 원인은 따로 있었다 — [[AI음성 - OpenAI 자동 응답 이중 생성 차단]]. 프로젝트: [[AI음성]]

## 무슨 작업

데스크톱 실기 콘솔에서 이 오류가 반복됐다.

```
Conversation already has an active response in progress: resp_...
→ 프론트: "다른 창/기기에서 이미 음성 대화가 실행 중입니다"
```

창 하나 기기 하나인데 그렇게 보였다. 추적해 보니 **양쪽이 같은 함수 호출에 응답**하고 있었다.

| 주체 | 트리거 | 보내는 것 |
|---|---|---|
| 프론트 `executeFunctionCallIfNeeded` | `response.output_item.done` (function_call) | `function_call_output` + `response.create` |
| 서버 v4 orchestrator | 같은 이벤트 | `conversation.item.create` + `response.create` |

[[AI음성 - 브라우저 이벤트 중계 복원]] 이후 프론트가 이 이벤트를 처음 보게 되면서 드러난 충돌이다. 중계 이전에는 프론트가 못 봐서 서버만 처리했다. v4는 서버가 tool을 소유하는 설계이므로, tool **제어** 이벤트만 중계에서 빼고 자막·오디오·`response.done`은 그대로 뒀다 (`_is_tool_control_event`).

## Codex 리뷰 반영

P2 — orchestrator의 `extract_function_tool_call`은 `payload.get("item") or payload.get("output_item")` 로 읽는데 필터는 `item`만 봤다. "지금 중복 실행이 일어난다"는 리뷰 주장은 과했지만(프론트는 `data?.item?.type`만 읽어 그 형태에 반응 안 함), 필터가 "서버가 처리하는 것은 넘기지 않는다"는 자기 계약을 지키려면 서버와 인식 범위가 같아야 하므로 반영했다.

## 오진 기록

배포 후에도 증상이 그대로였다. 이 필터가 막는 중복은 실재하지만 **그 오류의 주원인이 아니었다.** 진짜 원인은 다음 노트에서 잡았고, 필터는 응답 개수가 정리된 뒤 드러날 실제 중복을 막으므로 유지한다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `a34e521` | tool 제어 이벤트 중계 제외 필터 | `server_webrtc.py`, `test_browser_event_relay.py` |
| `54ddb79` | `output_item` 형태도 인식 (Codex P2) | 동일 |

main PR용 원본 브랜치(`fix/v4-session-generation-guard`)에도 체리픽 완료.

## 검증

`pytest ai_voice/voice_tests/` — **105 passed**.

| 변이 | 결과 |
|---|---|
| 필터 무력화 | 차단 테스트 2건 실패 |
| item.type 무시하고 전부 차단 | message 아이템 중계 테스트 실패 |
| `output_item` 다시 무시 | 해당 테스트 실패 |

Codex 재리뷰 통과.
