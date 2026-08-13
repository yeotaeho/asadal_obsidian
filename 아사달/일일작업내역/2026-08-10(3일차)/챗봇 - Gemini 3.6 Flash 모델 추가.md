---
tags: [아사달, 일일작업, 챗봇]
created: 2026-08-10
---

# Gemini 3.6 Flash 모델 추가

> [!summary] 한 줄 요약
> 챗봇이 고를 수 있는 LLM 목록에 `gemini-3.6-flash` 와 짧은 별칭 `gemini-3.6` 을 추가했다. 프로젝트: [[챗봇]]

## 무슨 작업

모델 등록 지점을 먼저 찾았다. 레포 전역 grep 결과 `utils/whoami.py` 의 `get_model_by_name()` **한 곳**이었고, `claude-opus-5` · `gpt-5.6-sol` 같은 최신 모델명이 이 파일에만 존재했다. 별도 화이트리스트·가격표가 없어 다른 파일은 손댈 필요가 없었다.

기존 `gemini-3.5` 분기와 같은 형태로 **2줄**을 추가했다.

```python
elif model_name in ("gemini-3.6-flash", "gemini-3.6"):
    return ChatGoogleGenerativeAI(model="gemini-3.6-flash", temperature=temperature, disable_streaming=False), "gemini-3.6-flash"
```

한 가지 짚어둘 점이 있다. 같은 함수 아래쪽의 `startswith("gemini-")` 폴백이 이미 모든 `gemini-*` 이름을 그대로 넘기고 있어서, **긴 이름 `gemini-3.6-flash` 는 코드 수정 없이도 이미 동작했다**. 이번 변경이 실제로 얹은 것은 **짧은 별칭 `gemini-3.6`** 과 기존 분기와의 일관성이다.

## 왜

신규 모델 반영 업무를 받았다. "`utils/whoami.py` 에 추가하면 되는가" 라는 질문에 대한 확인 작업이 절반이었고, 결론은 맞다는 것이었다.

`utils/` 는 공용 모듈이라 원래 수정 전 확인이 필요한 범위지만, 이번엔 사용자가 파일을 직접 지정했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `859054c` | Gemini 3.6 Flash 모델 추가 | `utils/whoami.py` |

브랜치는 `feat/gemini-3-6-flash` 이고 **origin/main 기준**으로 땄다. 로컬 main 이 origin 보다 뒤처져 있어 `git fetch` 후 `origin/main` 에서 분기했다.

## 검증

`python -m py_compile utils/whoami.py` 통과만 확인했다. **런타임 검증은 못 했다.**

로컬에 `sqlalchemy` · `langchain_google_genai` 가 설치돼 있지 않고 `config.py` 도 없어서(gitignore 대상) 모듈 임포트 자체가 안 된다. `config.example.py` 로 shim 을 만들고 `ChatGoogleGenerativeAI` 를 가짜 클래스로 바꿔 분기만 확인하려 했지만, 의존성 부재로 임포트 단계에서 실패했다.

`whoami.py` 에 대한 기존 테스트 파일도 없다. Codex 리뷰는 이번 작업에서 생략했다.

남은 확인 사항은 두 가지다.

- 모델 ID 문자열 `gemini-3.6-flash` 가 Google 공식 표기와 일치하는지. 기존 `gemini-3.5-flash` 명명 패턴을 따른 것이라 `-preview` 접미사 여부 등을 대조해야 한다.
- 실환경에서 계정 설정 `model_name: gemini-3.6-flash` 로 실제 응답이 오는지.
