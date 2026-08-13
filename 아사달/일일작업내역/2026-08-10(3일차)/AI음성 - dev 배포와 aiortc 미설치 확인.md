---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-10
---

# dev 배포와 aiortc 미설치 확인

> [!summary] 한 줄 요약
> v4 를 dev 에 머지(PR #760)했으나 **엔드포인트가 뜨지 않았다.** 원인은 `aiortc` 미설치이고, 배포 워크플로에 패키지 설치 단계가 없어서다. sudo 권한이 없어 팀장 요청 대기 중이다. 프로젝트: [[AI음성]]

## 무슨 작업

**dev 전용 사본 브랜치를 만들어 머지했다.** 원본 `fix/v4-session-generation-guard` 는 main 기준을 지켜야 하는데, dev 에는 main 에 없는 `ice_diag_router` 가 있어 그대로 병합하면 main PR 에서 ImportError 가 난다. 그래서 사본 `fix/v4-session-generation-guard-dev` 를 따로 만들어 거기서만 dev 를 병합했다.

충돌 3건을 풀었다.

| 파일 | 해결 | 근거 |
|---|---|---|
| `server_webrtc.py` | 내 쪽 | dev 쪽은 빈 줄. dev 의 ICE 픽스는 내 브랜치에도 같은 내용으로 있어 잃는 것 없음 |
| `conftest.py` | 내 쪽 | 내 것이 상위집합. dev 의 v3 가드 테스트가 쓰는 `webrtc_api` 스텁 포함 |
| `voice_router.py` | 양쪽 다 | `ice_diag_router` 와 v4 게이트는 별개 라우터 |

**인수인계 문서도 만들었다.** 파일 지도, 노드 79개의 함수 단위 런타임 플로우, 손댈 때 규칙을 담은 웹 문서와, 같은 내용을 옵시디언용으로 옮긴 노트 2개다. [[AI음성 - 런타임 플로우]], [[AI음성 - 파일·함수 레퍼런스]] 참고.

## 무엇이 막혔나

머지 후 배포 서버를 확인했더니 **v4 가 하나도 안 떴다.**

```
curl -sk https://dev.chatbaram.com:9000/openapi.json
  voice-chat-v4 경로:  0 개   ← 기대값 4
  voice-chat-v3 경로:  3 개
  diag/ice 경로:       4 개
```

소스는 정상이었다. dev 에 v4 패키지 5개 파일과 등록 코드가 다 들어가 있었다. 런타임에서만 등록이 안 된 것이다.

원인은 `voice_router.py` 의 안전장치였다. v4 라우터 import 가 실패하면 예외를 잡아 **v4 만 조용히 비활성화**하고 앱은 정상 기동한다. 그래서 v3 는 멀쩡히 돌고 v4 만 사라진 상태였다.

**진짜 원인은 배포 워크플로다.** `.gitea/workflows/deploy-dev.yml` 을 보니 이게 전부였다.

```
cd /home/chat_bot/Chatty_Project_DEV
git pull
sudo systemctl restart kss_app.service
```

`pip install -r requirements.txt` 단계가 없다. 내가 `requirements.txt` 에 `aiortc==1.9.0` 을 추가했지만 서버에는 설치되지 않았다.

## 서버 정보 파악

로그를 찾다가 엉뚱한 서버에 접속했었다. `110.45.147.70` 은 프론트를 올리는 웹호스팅 장비(`han.kr`)이고, 거기서 돌던 건 `chatbot_api_set:app` 이라는 다른 앱이었다. DNS 와 배포 워크플로에서 실제 dev 를 찾았다.

| 항목 | 값 |
|---|---|
| 주소 | `dev.chatbaram.com` → 110.45.147.56 |
| 배포 경로 | `/home/chat_bot/Chatty_Project_DEV` |
| 가상환경 | `/home/py_env/chatbot_env` (**공용**) |
| Chat App | `kss_app.service` · uvicorn · 9010 |
| Learning App | `kss_main.service` · uvicorn · 9011 |

**가상환경이 프로젝트 밖에 있고 두 서비스가 공유한다.** `aiortc` 가 `cryptography`·`pyopenssl`·`cffi` 를 끌어오므로 Learning App 에도 영향이 갈 수 있다. 그래서 바로 설치하지 않고 `--dry-run` 으로 버전 변경 여부를 먼저 확인하도록 요청서를 썼다.

설치 시점 기준 현재 버전은 `cryptography 46.0.5`, `pyOpenSSL 25.3.0`, `cffi 2.0.0` 이다.

## 왜

담당 과제 1번의 실체가 롤백된 v4 를 되살리는 것이었고, 백엔드 작업은 끝났다. 이제 실제로 서버에 올려 동작을 봐야 하는 단계인데 그 입구에서 막혔다.

`devuser` 계정에 sudo 권한이 없어 `pip install` 이 거부됐다. 팀장 승인이 필요하다.

## 어디

| 커밋 · PR | 내용 |
|---|---|
| `9c02f50` | dev 사본 브랜치에 dev 병합, 충돌 3건 해결 |
| PR #760 | `fix/v4-session-generation-guard-dev` → dev 머지 완료 |
| `779ce31` | dev 머지 커밋 |

브랜치 상태.

- `fix/v4-session-generation-guard` (`05481dc`) — main 기준 원본. **main PR 용으로 보존**
- `fix/v4-session-generation-guard-dev` (`9c02f50`) — dev 테스트용 사본
- `fix/v3-router-chat-id-guard-main` — 이미 dev 머지됨 (PR #743)

## 검증

`pytest ai_voice/voice_tests/` — 사본 브랜치에서 **81 passed** (내 75건 + dev 의 v3 가드 6건).

라우트 등록 실측 — v3 3개, v4 4개, ICE 진단 4개 전부 정상. **로컬에서는 문제없이 뜬다.**

## 추후 작업

- [ ] 팀장 승인 후 dev 서버에 `aiortc==1.9.0` 설치 → 재시작 → `openapi.json` 재확인
- [ ] `deploy-dev.yml` 에 의존성 설치 단계 추가 검토. 지금 구조로는 패키지를 추가할 때마다 같은 일이 반복된다
- [ ] 프론트 4곳의 `voice-chat-v3` 를 v4 로 전환 (백엔드가 뜬 뒤에)
- [ ] 로컬 검증은 aiortc **1.15.0** 에서 했다. `requirements.txt` 는 1.9.0 핀이라 버전 차이가 있다. 설치 후 확인 필요

## 알아둘 것

**dev 는 uvicorn 단일 프로세스다.** gunicorn 멀티워커가 아니라서 내가 고친 이벤트 루프 문제(`_get_loop`)는 dev 에서 원래 재현되지 않는다. 운영에서만 터지는 종류라 dev 테스트로는 그 부분을 검증할 수 없다. main 배포 때 염두에 둬야 한다.
