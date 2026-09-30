---
tags: [아사달, 일일작업, 챗봇]
created: 2026-09-11
---

# Gemini 3.8 Flash·GPT-6 Astra 추가

> [!summary] 한 줄 요약
> 신규 모델 4종을 확인해 2종(Gemini 3.8 Flash, GPT-6 Astra)을 등록하고, DeepSeek 은 revert, Muse Spark 는 보류했다. 프로젝트: [[챗봇]]

## 무슨 작업

신규 모델 4건을 하나씩 확인하고 등록 여부를 판단했다. 결과는 **2건 등록, 1건 revert, 1건 보류** 다.

| 모델 | 확인 결과 | 조치 |
|---|---|---|
| Gemini 3.8 Flash | 정식 ID `gemini-3.8-flash` (2026-09-02 GA) | 등록 |
| GPT-6 Astra | API 개방, 현재 키로 호출 성공 | 등록 (temperature 처리 포함) |
| DeepSeek V4.1 Flash | 이미 V4.1 이 응답 중, 고칠 게 없음 | revert |
| Meta Muse Spark 1.1 | 미국 개발자 전용 preview | 보류 |

### Gemini 3.8 Flash

기존 3.7 분기와 같은 형태로 2줄을 추가했다. 아래쪽 `startswith("gemini-")` 폴백이 이미 긴 이름을 통과시키고 있었으므로, 이번 변경의 실효는 **짧은 별칭 `gemini-3.8`** 을 정식 ID 로 정규화하는 것뿐이다.

### GPT-6 Astra — 이번 작업의 핵심

`.env` 의 `OPENAI_API_KEY` 로 직접 호출해봤고 **접근 가능**했다. `GET /v1/models` 목록에 존재하고, chat/completions · responses · SSE 스트리밍 모두 200 이다.

그런데 그대로 두면 **매 호출 400** 이 날 상태였다.

```
temperature=0    -> 400 'temperature does not support 0 with this model. Only the default (1) value is supported.'
temperature=0.7  -> 400 (동일)
temperature 미전송 -> 200
```

`gpt-6-astra` 는 맨 아래 `startswith("gpt-")` 폴백으로 떨어지고, 거기서 기본값 `temperature=0` 을 그대로 실어 보낸다. 그래서 `gpt-5.3-chat-latest` 와 동일하게 `temperature=None` 전용 분기를 앞쪽에 넣었다.

```python
elif model_name in ("gpt-6-astra", "gpt-6", "gpt6-astra"):
    resolve_name = "gpt-6-astra"
    return ChatOpenAI(**_build_openai_chat_kwargs(model_name=resolve_name, temperature=None, ...)), resolve_name
```

`startswith("gpt-6")` 를 쓰지 않은 이유는 Astra Pro 출시 시 `gpt-6-astra-pro` 가 Astra 로 잘못 접히기 때문이다.

### DeepSeek V4.1 Flash — 만들었다가 되돌림

처음엔 `deepseek-v4.1-flash` 별칭 추가와 함께 디스패치 이름을 deprecated `deepseek-v4-flash` 에서 정식 `deepseek-flash` 로 바꿨다(`eb36149`). 확인 과정에서 **DeepSeek 이 모델명에서 버전 번호를 뺐고, 어느 이름을 보내든 이미 V4.1-Flash 가 응답한다**는 것이 드러났다. 당장 깨진 게 없어 revert 했다. 상세는 [[챗봇]] 의 결정 기록에 남겼다.

### Meta Muse Spark 1.1

미국 개발자 전용 public preview 라 한국에서는 Meta Model API 계정 생성 자체가 안 된다. `.env` 에 Meta 키도 없다 (`LLAMA_CLOUD_API_KEY` 는 LlamaIndex Cloud 용으로 무관). 테스트 불가, 코드 미반영이다.

## 왜

신규 모델 반영 업무 4건을 받았다. 절반은 "붙일 수 있는가" 를 확인하는 일이었고, 실제로 2건은 붙일 수 없거나 붙일 필요가 없다는 결론이 나왔다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `7399db7` | Gemini 3.8 Flash 모델 추가 (+2) | `utils/whoami.py` |
| `eb36149` | DeepSeek-V4.1-Flash 모델 추가 (+2/-2) | `utils/whoami.py` |
| `2411f1f` | 위 커밋 Revert | `utils/whoami.py` |
| `d267a68` | GPT-6 Astra 모델 추가 (+3) | `utils/whoami.py` |

브랜치는 `feat/gemini-3.8-flash` 이고 **main 기준**이다. 작업 시작 시점의 체크아웃은 `feat/voice-msg-toggle` 이었으나 무관한 변경이 섞이는 것을 피해 main 에서 새로 땄다.

## push 실패 — amend 사고

`d267a68` 은 처음 `dcac1cb` 로 커밋한 뒤 주석 제거 요청을 받아 **amend** 한 것이다. 그런데 그 사이에 `dcac1cb` 가 **이미 origin 에 push 돼 있었다**. amend 로 해시가 바뀌면서 로컬과 원격이 갈라졌고, 다음 push 가 거부됐다.

```
! [rejected]  feat/gemini-3.8-flash -> feat/gemini-3.8-flash (non-fast-forward)
```

로컬 1 ahead / 원격 1 behind, 공통 조상은 `2411f1f` 다. 두 tip 은 주석 한 줄 차이뿐이다.

**원인 판단이 틀렸던 지점** — `git branch -vv` 에 upstream 이 안 보여서 미push 로 단정했다. push 를 `-u` 없이 하면 추적 브랜치가 안 잡혀서 그렇게 보인다. **push 여부는 `git branch -vv` 가 아니라 `git ls-remote origin <브랜치>` 로 확인해야 한다.**

## 검증

**OpenAI 원시 API 는 직접 검증했다.** 키로 4회 호출했고 (`/v1/models`, chat/completions, responses, stream) 모두 의도대로 동작했다. temperature 400 도 실제로 재현해 확인했다.

**함수 단위 실행 검증은 못 했다.** 로컬에 venv 도 `config.py` 도 없어(둘 다 gitignore) `utils.whoami` 임포트 자체가 실패한다 — `ModuleNotFoundError: No module named 'config'`. `langchain_openai` 도 미설치다.

`temperature=None` 이 langchain 에서 필드 생략으로 이어지는지는 **같은 파일의 `gpt-5.3-chat-latest` 분기가 이미 같은 패턴으로 운영 중**이라는 점에 기댔다. 같은 400 을 같은 방법으로 피하고 있다.

pytest 는 돌리지 않았다. `whoami.py` 에 대한 기존 테스트가 없고, 임포트가 안 되는 환경이라 새로 짜도 실행되지 않는다. **배포 환경에서 두 모델로 한 번씩 호출하는 것이 실질 검증**이다.

Codex 리뷰(`--base main --scope branch`)는 커밋마다 돌렸고 **지적사항 없음** 이었다.
