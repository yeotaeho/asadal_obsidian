---
tags: [아사달, 일일작업, 챗봇]
created: 2026-08-28
---

# Gemini 3.6 Flash 모델 추가

> [!summary] 한 줄 요약
> 챗봇이 고를 수 있는 LLM 목록에 `gemini-3.6-flash` 와 짧은 별칭 `gemini-3.6` 을 추가하고, 단가표 행까지 채웠다. 프로젝트: [[챗봇]]

## 무슨 작업

모델 등록 지점을 먼저 찾았다. 레포 전역 grep 결과 `utils/whoami.py` 의 `get_model_by_name()` **한 곳**이었고, `claude-opus-5` · `gpt-5.6-sol` 같은 최신 모델명이 이 파일에만 존재했다. 별도 화이트리스트가 없어 다른 파일은 손댈 필요가 없었다.

기존 `gemini-3.5` 분기와 같은 형태로 **2줄**을 추가했다.

```python
elif model_name in ("gemini-3.6-flash", "gemini-3.6"):
    return ChatGoogleGenerativeAI(model="gemini-3.6-flash", temperature=temperature, disable_streaming=False), "gemini-3.6-flash"
```

한 가지 짚어둘 점이 있다. 같은 함수 아래쪽의 `startswith("gemini-")` 폴백이 이미 모든 `gemini-*` 이름을 그대로 넘기고 있어서, **긴 이름 `gemini-3.6-flash` 는 코드 수정 없이도 이미 동작했다**. 이번 변경이 실제로 얹은 것은 **짧은 별칭 `gemini-3.6`** 과 기존 분기와의 일관성이다.

## 단가표

코드와 별개로 단가표(구글 시트)를 이어서 작성했다. 레포의 `utils/api_billing.py` 는 이미지 생성(Replicate·DALL·E·Recraft) 전용이라 텍스트 LLM 토큰 단가는 시트가 유일한 관리처다.

| 항목 | 값 |
|---|---|
| 입력 | **1.5** $/1M |
| 캐시 입력 | **0.15** $/1M |
| 출력 | **7.5** $/1M |
| 캐싱 지원 | O |

3.5 대비 입력·캐시 입력은 동일하고 출력만 **9 → 7.5 ($/1M, 약 17% 인하)** 다. 시트 25행의 한글 모델명이 `제미나이 3.5 플래시` 로 복사돼 있어 `제미나이 3.6 플래시` 로 고쳐야 한다고 전달했다.

공식 가격 페이지에서 기존 3.5 행의 값(1.5 / 0.15 / 9)이 정확히 재현돼 열 해석이 맞다는 것을 교차 확인했다.

## 왜

신규 모델 반영 업무를 받았다. "`utils/whoami.py` 에 추가하면 되는가" 라는 질문에 대한 확인 작업이 절반이었고, 결론은 맞다는 것이었다.

`utils/` 는 공용 모듈이라 원래 수정 전 확인이 필요한 범위지만, 이번엔 사용자가 파일을 직접 지정했다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `859054c` | Gemini 3.6 Flash 모델 추가 | `utils/whoami.py` |

브랜치는 `feat/gemini-3-6-flash` 이고 **origin/main 기준**으로 땄다. 로컬 main 이 origin 보다 뒤처져 있어 `git fetch` 후 `origin/main` 에서 분기했다. push 는 아직 안 했다.

## 검증

`python -m py_compile utils/whoami.py` 통과만 확인했다. **런타임 검증은 못 했다.**

로컬에 `sqlalchemy` · `langchain_google_genai` 가 설치돼 있지 않고 `config.py` 도 없어서(gitignore 대상) 모듈 임포트 자체가 안 된다. `config.example.py` 로 shim 을 만들고 `ChatGoogleGenerativeAI` 를 가짜 클래스로 바꿔 분기만 확인하려 했지만, 의존성 부재로 임포트 단계에서 실패했다.

`whoami.py` 에 대한 기존 테스트 파일도 없다. Codex 리뷰는 생략했다.

모델 ID 표기는 사후에 확인됐다. `gemini-3.6-flash` 가 공식 표기이고 `-preview` 접미사는 없다. 남은 것은 실환경에서 `model_name: gemini-3.6-flash` 로 실제 응답이 오는지다.
