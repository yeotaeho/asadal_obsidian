---
tags: [아사달, 일일작업, AI음성]
created: 2026-08-26
---

# gpt-realtime-2.1 전 계정 배포 누락 규명과 대표 모델 반영

> [!summary] 한 줄 요약
> 음성 모델 gpt-realtime-2.1을 han 계정 **700여 개** DB에 배포할 때 일부 계정에 반영되지 않던 원인 세 가지를 규명하고, 카탈로그 중복 행 정리와 db_tool.py 계열 수정으로 대표 모델까지 반영을 완료했다. 서비스: [[AI음성]]

## 무슨 작업

배포용 스크립트 `ai_model_update.py`(카톡 공유본)를 검토해 누락 원인을 추적했다. 원인은 한 가지가 아니라 세 겹이었다.

첫째, 스크립트의 조회 쿼리에 `fication = 'basic'` 이 하드코딩돼 있어 **voice 행은 애초에 업데이트 대상이 아니었다**. fication 을 실행 시 선택(basic/voice/직접 입력)하도록 고친 `ai_model_update_v2.py` 를 만들었다.

둘째, 공통 카탈로그 `chatty.chatbot_ai_list` 에 gpt-realtime-2.1 이 **중복 등록**돼 있었다. soft-delete 된 basic 행(id 152)과 살아있는 voice 행이 공존했고, 삭제된 행을 DELETE 로 정리하자 사이트에서 모델이 정상 노출됐다.

셋째, 구조 규명이 핵심이었다. 사이트의 음성 대표 모델은 각 봇의 `chatbot_ai_model` **voice 행 `model` 컬럼**이고, 이 행은 관리 화면에서 음성 설정을 **저장할 때 비로소 INSERT** 된다(lazy 생성). 저장 이력이 없는 봇은 행 자체가 없어서, UPDATE 전용인 기존 스크립트로는 **몇 번을 돌려도 영구 누락**된다.

## 왜

gpt-realtime-2.1 추가 후 마이그레이션에서 han 계정 일부(광진구 등)에 모델이 반영되지 않았다. 초기에는 스크립트 문제로 보였지만, 실제로는 스크립트 필터·카탈로그 중복·행 lazy 생성 구조가 겹친 복합 원인이었다.

```
카탈로그(chatty.chatbot_ai_list)      ← 모델 등록 + 상태 플래그(defa=대표, disp, disa, type)
        ↓ 마이그레이션
계정별 chatbot_ai_model (voice 행)    ← 봇별 사용 모델(models) + 선택 모델(model)
        └ 행이 없으면 관리화면 저장 시 INSERT   ← UPDATE 전용 스크립트의 사각지대
```

최종적으로 서버 통합 도구 `db_tool.py`(jobs 구조, Plan→Apply) 관련 파일을 수정해 대표 모델 값 업데이트까지 반영을 완료했다.

## 어디

| 구분 | 내용 |
|---|---|
| 원인 스크립트 | `ai_model_update.py` — `fication='basic'` 하드코딩, deleted_at 미고려, UPDATE 전용 |
| 수정본 | `ai_model_update_v2.py` — fication 실행 시 선택 추가 |
| 최종 반영 | 서버 `db_tool.py` 및 관련 Job 파일 수정으로 대표 모델 업데이트 |
| 데이터 정리 | `chatty.chatbot_ai_list` id 152 (soft-delete 된 basic 중복 행) DELETE |
| 관련 테이블 | `chatbot_ai_list`(카탈로그·상태 플래그) · 계정별 `chatbot_ai_model`(봇별 모델) |

## 검증

- `ai_model_update_v2.py` — `python -m py_compile` 통과.
- DBeaver 로 구로·광진구 DB의 `chatbot_ai_model` voice 행 확인 — models 에 gpt-realtime-2.1 추가·대표(model) 반영 확인.
- 실제 사이트에서 리얼타임 2.1 이 대표 모델로 표시되는 것 확인.
- 대표 미설정 사이트에서 수동 저장 시 voice 행이 신규 INSERT 되는 것을 확인해 lazy 생성 구조를 실증.
