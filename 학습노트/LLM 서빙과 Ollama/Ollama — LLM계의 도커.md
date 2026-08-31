---
tags: [학습노트, LLM, Ollama, 서빙]
created: 2026-08-27
---

# Ollama — LLM계의 도커

> [!summary] 한 줄 요약
> Ollama는 오픈소스 LLM을 로컬에서 쉽게 실행·서빙하게 해주는 **런타임 도구**다. 서빙(REST API) + 모델 허브 + 경량화(양자화)를 한 번에 제공해서 **'LLM계의 도커'** 라 불린다.

## 세 가지 역할이 합쳐진 도구

- **서빙 (Local API Serving)** — 실행하면 로컬에 REST API 엔드포인트(**기본 11434 포트**)가 자동 활성화된다. OpenAI API를 호출하듯 로컬 LLM을 FastAPI·Spring Boot·LangChain 등에서 호출할 수 있다.
- **모델 허브 (Ollama Library)** — 도커 허브에서 이미지를 받듯, `ollama run llama3` 한 줄이면 Llama 3·Gemma 2·Mistral·Phi 3 등을 자동 다운로드부터 실행까지 끝낸다.
- **경량화 (Quantization)** — 내부적으로 `llama.cpp` 엔진 기반. 허브의 모델들은 기본 **4비트 양자화(INT4)** 처리된 GGUF 형태라 수십 GB짜리 원본보다 대폭 가볍다. Apple Silicon 가속과 Nvidia GPU/CPU 분배도 알아서 처리한다.

## 허깅페이스와의 차이 — 도매시장 vs 밀키트

> 허깅페이스는 모델이 저장된 **거대한 백화점(GitHub)**, Ollama는 그 모델을 가져와 내 컴퓨터에서 바로 서빙하는 **밀키트 조리기(Docker)**.

| 구분 | 허깅페이스 | Ollama |
|---|---|---|
| 역할 | 모델·데이터셋 공유 **저장소(Hub)** | 로컬 실행·관리 **런타임/엔진** |
| 제공 형태 | 원본 모델(PyTorch·Safetensors) + 코드 | 로컬 최적화 가공본 (**GGUF**) |
| 실행 방식 | Python 코드로 직접 로드·GPU 설정 | CLI 한 줄로 다운로드~API 서버까지 자동 |

허깅페이스의 수많은 모델 중 **대중적인 것들을 골라 로컬용으로 패키징**해 주는 게 Ollama다.

## 개발자 관점 포인트

- **Modelfile** — Dockerfile과 유사한 문법으로 커스텀 모델을 빌드. 베이스 모델 + 시스템 프롬프트 + `temperature` 등을 고정해 패키징한다.
- **비용 0원** — 클라우드 API의 토큰 과금 없이 하드웨어 자원만으로 RAG·에이전트 아키텍처를 자유롭게 실험할 수 있다.

## 관련

- [[Ollama 실행 구조와 CLI]] — run 한 줄 뒤에서 일어나는 일
- [[클라우드에서의 Ollama]] — 로컬이 아니라 AWS에 띄울 때
