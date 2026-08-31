---
tags: [아사달, 일일작업, 챗봇]
created: 2026-08-28
---

# GLM-5.3 모델 추가

> [!summary] 한 줄 요약
> 챗봇이 고를 수 있는 LLM 목록에 `glm-5.3` 과 짧은 별칭 `glm5.3` 을 추가했다. 같은 날 [[챗봇 - Gemini 3.6 Flash 모델 추가]] 와 같은 패턴이다. 프로젝트: [[챗봇]]

## 무슨 작업

GLM 관련 코드를 전부 grep 해 등록 지점을 확인했다. 이번에도 `utils/whoami.py` 한 곳이었다.

| 위치 | 내용 | 조치 |
|---|---|---|
| `whoami.py:216` | `ChatGLM` 클래스 (Z.ai `GLM_API_KEY`, base_url) | 그대로 재사용 |
| `whoami.py:905` | `get_model_by_name` 분기 | **추가** |
| `whoami.py:43` | `glm-4.7-flash` → RunPod SLLM | 무관 (자체 서버 경로) |

기존 `glm-5.2` 분기와 같은 형태로 **2줄**을 추가했다.

```python
elif model_name in ["glm-5.3", "glm5.3"]:
    return ChatGLM(model="glm-5.3", temperature=temperature), "glm-5.3"
```

Gemini 때와 마찬가지로 `startswith("glm-")` 폴백이 이미 존재하므로 **긴 이름 `glm-5.3` 은 수정 없이도 통했다**. 실질은 짧은 별칭 `glm5.3` 과 5.2 분기와의 일관성이다.

`.claude/worktrees/epic-elbakyan-289c49/` 아래에도 같은 히트가 나오지만 오래된 워크트리 사본이라 건드리지 않았다.

## 왜

GLM 5.3 추가 요청을 받았다. GLM 5.2 도 과거에 같은 방식으로 추가된 이력이 있다.

이번엔 **커밋 전에** 모델 ID 를 확인했다. Gemini 때 문자열을 명명 패턴으로 추정하고 사후에 확인한 순서를 뒤집은 것이다. 공식 표기는 `glm-5.3` 이 맞고, 단가는 입력 **$1.40** / 출력 **$4.40** per 1M 으로 5.2 와 동일하다. 캐시 입력은 $0.26, 컨텍스트 1M, 출력 최대 128K 다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `e2ef47c` | GLM-5.3 모델 추가 | `utils/whoami.py` |

브랜치 `feat/glm-5-3`, origin/main(`1be533b`) 기준 1커밋이다. Gemini 브랜치와 독립된 별개 PR 로 간다.

## 검증

`python -m py_compile utils/whoami.py` 통과만 확인했다. 런타임 검증은 Gemini 때와 같은 이유로 불가능하다 — 로컬에 의존성과 `config.py` 가 없어 모듈 임포트가 안 된다. Codex 리뷰는 생략했다.

`requirements.txt` 가 수정된 상태로 남아 있었는데 무관한 변경이라 커밋에 넣지 않았다.
