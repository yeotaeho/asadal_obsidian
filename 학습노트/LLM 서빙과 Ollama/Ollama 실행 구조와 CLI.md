---
tags: [학습노트, LLM, Ollama, CLI]
created: 2026-08-27
---

# Ollama 실행 구조와 CLI

> [!summary] 한 줄 요약
> Ollama는 **클라이언트-서버 구조**다. `ollama run`은 백그라운드 서버(11434)에 모델을 로드시키고 터미널을 채팅 클라이언트로 바꾸는 것뿐이라, **터미널을 `/exit`로 나가도 API 서버는 계속 살아 있다.**

## `ollama run <모델명>`이 하는 일

```
1. 백그라운드 올라마 서버(데몬, 11434 포트) 확인
2. 모델 파일을 디스크 → VRAM(또는 RAM)에 로드
3. http://localhost:11434/api/... REST API 개방
4. 터미널을 대화형 클라이언트로 전환
```

핵심 — 터미널에서 채팅하는 그 순간에도 백그라운드 11434 포트는 **외부 백엔드의 API 요청을 받을 준비가 끝난 상태**다.

## 핵심 CLI 명령어

| 명령어 | 역할 |
|---|---|
| `ollama run <모델>` | 다운로드 + 메모리 로드 + API 개방 + 대화 프롬프트 진입 |
| `ollama serve` | 백그라운드 서버만 수동 기동 (보통 부팅 시 자동이라 드묾) |
| `ollama list` | 로컬에 받아둔 모델 목록 |
| `ollama pull <모델>` | 대화 없이 다운로드만 미리 |

## 종료와 메모리 관리 — /exit ≠ 서버 종료

- `/exit`로 대화창을 나가도 **서버 엔진은 계속 동작**한다.
- 대신 **약 5분간 API 요청이 없으면** 모델을 VRAM에서 자동 언로드(Unload)한다.
- 이후 요청이 오면 알아서 다시 로드한다 — 그래서 **첫 응답만 로드 시간만큼 느릴 수 있다.**

## API 확인과 호출

- 상태 확인 — `GET http://localhost:11434/` → `Ollama is running`.
- 생성 호출 — `POST /api/generate` 또는 `/api/chat`.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma2",
  "messages": [{ "role": "user", "content": "왜 하늘은 파란색이야?" }],
  "stream": false
}'
```

```python
import ollama
response = ollama.chat(model='gemma2', messages=[
    {'role': 'user', 'content': '로컬 서버 서빙 테스트 중이야.'},
])
print(response['message']['content'])
```

## 관련

- [[Ollama — LLM계의 도커]] — 이 도구의 정체
- [[클라우드에서의 Ollama]] — 같은 구조를 AWS에 올리면
