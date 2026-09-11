---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-11
---

# 음성채팅기록 토큰 사용량 전달

> [!summary] 한 줄 요약
> 채팅기록 목록에서 음성 행만 비어 있던 **입력·캐시·출력·합계** 네 칸을 채우고, 음성에서는 항상 404 로 끝나던 키워드 커밋 호출을 뺐다. 기능: [[음성채팅기록]]

## 무슨 작업

목록 화면에서 음성 행만 **키워드·입력·캐시·출력·합계 다섯 칸이 `-`** 였다. 두 무리는 경로가 전혀 달라 따로 추적했다.

토큰 네 칸은 `token_usage` 키 하나로 들어간다. 텍스트 채팅은 `/ask` SSE 의 마지막 청크로 받아 저장 본문에 실어 보내고, `process_add.php` 가 질문·답변 두 행에 같은 값을 복사한다. 음성은 그 키를 안 보내 네 칸이 모두 비었다.

키워드(#)는 채팅기록 컬럼이 아니라 **`chat_process_keyword_map` 별도 테이블**이다. 재료를 `/ask` 가 만들어 Redis 에 10분 임시 저장하고, 저장 뒤 프론트가 커밋해야 행이 생긴다.

## 왜

**토큰은 이미 브라우저까지 와 있었다.** 음성 서버는 `response.done` 마다 텍스트 채팅과 같은 모양의 `token_usage` 이벤트를 DataChannel 로 보내고 있는데, 프론트 모듈에 분기가 없어 `default` 로 버려졌다.

그 이벤트를 프론트에서 주워 쓰는 대신 **서버가 기록 통지에 실어 보내게** 했다. 어느 응답이 기록되는 답변인지는 서버만 안다 — 도구 호출 응답도 usage 를 내므로 프론트가 받으면 어느 값이 그 질문의 것인지 가릴 수 없다. `response.done` 은 기록을 확정하는 자리이자 usage 가 실려 오는 자리라, 인자 하나로 흘려보내면 끝이었다.

```
response.done ─ usage 추출 ─┐
              └ 전사 flush ─┴→ TurnLogger.on_assistant_text(tokens=…)
                              → voice_chat.turn { …, token_usage }
                              → index2 persistVoiceChatTurn
                              → process_add.php → 질문·답변 두 행
```

**키워드는 재료가 없다.** 음성은 `/ask` 를 타지 않아 임시 키워드가 없고, 커밋은 매 턴 **404** 로 돌아와 콘솔 경고만 남겼다. 저장은 그 전에 끝나 기록엔 영향이 없었지만 헛호출이라 뺐다. 음성 키워드를 채우려면 shim 의 pgvector 검색에서 생략한 형태소 키워드 추출과 `SEARCH_TOKEN_LIST` 매핑을 본체에서 가져와야 해 1단계 범위 밖이다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `b4c719d` | 답변 응답의 usage 를 기록 통지에 실음 | `v4/turn_log.py` · `server_webrtc.py` |
| (git 밖) | `token_usage` 전달, 키워드 커밋 호출 제거 | 본체 `dev/index2.js` `persistVoiceChatTurn` |

프론트 모듈은 손대지 않았다. `voice_chat.turn` 페이로드를 통째로 펼쳐 넘기고 있어 새 키가 그대로 따라간다.

`_relay_event_to_browser` 의 기존 `token_usage` 중계는 남겼다. 본체에도 같은 코드가 있어 **본체 동기화 원칙** 상 이 브랜치에서 지울 것이 아니다. 소비자가 없는 경로라는 점만 별건으로 남긴다.

## 검증

`pytest ai_voice/voice_tests` **351 passed** (기준선 349, 신규 2건). 페이로드 모양을 통째로 비교하던 기존 단언 2건에 `token_usage` 키를 더했다.

- 기록되는 답변 응답의 usage 가 실리고 **도구 호출 응답 것이 아님**을 도구 턴으로 고정
- usage 없는 응답은 `token_usage` 가 `None` — PHP `is_array` 검사에 걸려 네 칸을 건너뛴다

Codex 리뷰(`--commit b4c719d`) 지적 없음.
