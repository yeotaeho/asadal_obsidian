---
tags: [학습노트, python, httpx, tenacity, 비동기]
created: 2026-08-31
---

# httpx — 요청을 보내는 쪽

> [!summary] 한 줄 요약
> 이 폴더의 다른 노트들이 **들어오는 요청**(uvicorn·gunicorn)을 다뤘다면, httpx는 정반대다. **FastAPI는 외부 → 우리 서버, httpx는 우리 서버 → 외부**를 담당한다. 처음 볼 때 둘이 가장 헷갈린다.

## 방향이 반대다

```
FastAPI + Uvicorn      외부 ──→ 우리 서버      (수신)
httpx                  우리 서버 ──→ 외부      (송신)
```

서버 애플리케이션은 이 둘을 **동시에** 갖는다. 웹훅을 받아서(FastAPI) 부족한 데이터를 외부 API에서 더 긁어오는(httpx) 흐름이 전형적이다.

```
GitHub ──POST webhook──→ Uvicorn → FastAPI → /webhook/github
                                                  │ 추가 데이터 필요
                                                  ▼
                                                httpx ──→ GitHub API
```

```python
import httpx

async with httpx.AsyncClient() as client:
    response = await client.get("https://hn.algolia.com/api/v1/search")
data = response.json()
```

## 왜 async httpx인가

외부 소스를 여러 곳 수집하는 구조라면 순차 요청은 시간이 그대로 더해진다.

| 방식 | 소요 |
|---|---|
| 순차 — GitHub 1초 → HN 1초 → 블로그 1초 → YouTube 1초 | 약 **4초** |
| 비동기 — 넷을 동시에 대기 | 약 **1초** |

원리는 이 폴더의 핵심 문장과 같다. **네트워크 응답을 기다리는 동안 이벤트 루프가 다른 요청을 진행**시킨다 → [[이벤트 루프]]. 반대로 **동기 HTTP 클라이언트를 async 함수 안에서 쓰면 루프가 통째로 멈춘다** — 요리사가 냄비 하나를 붙잡는 바로 그 상황이다 → [[동시성 층위 — 워커·스레드·작업]].

```
                     ┌→ GitHub API
                     ├→ Hacker News API
Collector ── httpx ──┼→ 외부 웹페이지
                     ├→ 기타 REST API
                     └→ RSS 관련 HTTP 요청
```

## tenacity — 실패했을 때의 정책

외부 API는 늘 정상 응답하지 않는다. Timeout, `429 Too Many Requests`, `500`, `503`이 일상이다. 바로 실패시키지 않고 간격을 늘려가며 재시도하는 역할을 tenacity가 맡는다.

```
1차 요청 → 실패 → 1초 대기
2차 요청 → 실패 → 2초 대기
3차 요청 → 성공
```

역할 분담이 명확하다.

| 라이브러리 | 책임 |
|---|---|
| **httpx** | HTTP 요청을 보내는 것 |
| **tenacity** | 그 요청이 실패했을 때 재시도 정책 |

## 곁다리 — 가져온 뒤의 처리

수집 파이프라인에서 httpx는 **HTML을 내려받는 데까지**만 책임진다. 본문만 골라내는 것은 별도 라이브러리(trafilatura 등)의 일이다.

```
httpx        → HTML 다운로드
trafilatura  → HTML에서 실제 기사 본문 추출
```

점수 기준은 통과했는데 본문이 없거나 너무 짧은 항목에만 원문 URL을 다시 긁는 식으로, **필요할 때만 요청을 보내는** 설계가 일반적이다.

## 관련 노트

- [[uvicorn과 gunicorn]] — 반대 방향, 요청을 받는 쪽
- [[이벤트 루프]] — async가 이득을 보는 원리
- [[동시성 층위 — 워커·스레드·작업]] — 루프를 막으면 어떻게 되는가
- [[uv — 의존성과 환경 재현]] — 이 라이브러리들을 설치·고정하는 도구
