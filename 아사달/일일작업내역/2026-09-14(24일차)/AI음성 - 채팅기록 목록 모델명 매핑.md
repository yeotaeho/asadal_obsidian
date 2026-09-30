---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-14
---

# 채팅기록 목록 모델명 매핑

> [!summary] 한 줄 요약
> 음성 행이 "오픈AI 챗GPT-4.1" 로 보이던 원인은 목록 페이지의 변환 함수가 텍스트 카탈로그만 뒤지고 **하드코딩 폴백**을 돌려주던 것. 두 카탈로그를 합쳐 찾게 고쳤고 배포 후 정상 확인. 기능: [[음성채팅기록]]

## 무슨 작업

저장값은 정상이었다. `model_name` 컬럼이 실재하고, 브라우저 → 모듈 `normalizeRealtimeModel` → 서버 `resolved_model` → `TurnLogger` → `persistVoiceChatTurn` → PHP 로 리얼타임 ID 가 그대로 들어간다.

표시는 DB 를 안 읽는다. 카탈로그 두 표를 `file_put_model()` 이 합쳐 봇마다 정적 JS 를 써 두고, 페이지는 그걸 `<script>` 로 부른다.

| 표 | 위치 | 역할 |
|---|---|---|
| `chatty.chatbot_ai_list` | 공용 DB | 모델 카탈로그. `ai_id`·`nicnm`·`name`·`fication` |
| `chatbot_ai_model` | 계정 DB | 봇이 켠 모델(`models`)과 대표 모델(`model`), 분류별 한 행 |

`fication` 마다 다른 전역 배열로 갈린다. `basic` → `modelList`, `voice` → `modelListVoice`. 목록 페이지 `chatlog.htm` 의 `get_model()` 은 **`modelList` 만** 뒤지고 못 찾으면 `'오픈AI 챗GPT-4.1'` 리터럴을 돌려줬다. 필터 셀렉트도 같은 배열만 돌렸다.

두 곳 다 두 배열을 이어 붙여 찾게 했다. 폴백은 저장된 ID 원문, 빈 값은 `-`.

## 왜

로그인된 페이지에서 함수를 직접 호출해 확정했다. 카탈로그의 `gpt-4.1` 표시명은 **공백**("챗GPT 4.1")인데 화면은 **하이픈**("챗GPT-4.1")이었다. 잘못 매칭된 게 아니라 함수 안의 문자열이었다.

`modelListVoice` 는 페이지에 로드돼 있었고(리얼타임 3종, `nicnm == ai_id`) 참조만 0회였다. DB·생성 파일은 멀쩡했다.

## 어디

| 파일 (본체 `dev/`, git 밖) | 내용 |
|---|---|
| `chatlog.htm:480` | `get_model` — `modelList.concat(modelListVoice)` 에서 찾고, 폴백은 ID 원문 |
| `chatlog.htm:806` | 모델명 필터 — 같은 합친 카탈로그로 옵션 생성 |
| `dev/backup_20260911_before_modelmap/chatlog.htm` | 수정 전 원본 |

## 검증

- 패치한 함수를 살아 있는 페이지의 실제 카탈로그에 돌려 리얼타임 3종 이름이 나오고 `gpt-4.1` 은 기존 동작 유지. 필터 옵션 **19 → 22**
- 편집 조각 `node --check` 통과, 파일 차이는 두 덩어리뿐
- 배포 후 사용자 확인 — 음성 행 모델명 정상 표시

## 같이 발견한 것

- 카탈로그에 `gpt-realtime-2.1-mini` 가 없다. 기획서 4종 중 1종 미등록
- `learn_chatprocess_list.php` 모델 필터가 `LIKE '%id%'` 라 "리얼타임 2" 가 2.1 행까지 끌고 온다
- 같은 파일의 키워드 조회에 `chatbot_id` 조건이 없어 같은 계정 다른 봇의 키워드가 섞일 수 있다
