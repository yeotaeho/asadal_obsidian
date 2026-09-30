---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-14
---

# 음성채팅기록 음성 필드 배관과 목록 재생 UI

> [!summary] 한 줄 요약
> 2단계를 **"필드 먼저, 녹음 나중"** 으로 순서를 바꿔, `voice_audio` 컬럼과 통지 → 프론트 → PHP → 목록 ▶ 재생·삭제 배관을 녹음 없이 먼저 놓았다. 녹음은 나중에 통지에 `q` 를 붙이는 공급자로 끼어든다. 기능: [[음성채팅기록]]

## 무슨 작업

컬럼에 들어가는 값이 URL 문자열 하나라 녹음 없이도 배관을 끝까지 놓고 검증할 수 있다. 과장님 허락(dev·test 계정에 컬럼 추가 후 테스트, 운영은 완료 후 재논의)을 받았다.

| 조각 | 무엇 |
|---|---|
| DDL | `class.chatbot.php` `create_sql5` 에 `voice_audio text DEFAULT NULL` — 앞으로 만드는 봇 전부 |
| 기존 봇 | dev·test 계정 DB 의 `ASADAL_*_CHATING_PROCESS` 에 ALTER 스크립트 (사용자 실행) |
| 모듈 | `voice_chat.turn` 에 `q:{path,t}` 가 오면 `apiVoiceUrl` 로 절대 URL 을 만들어 `voice_audio` 로 넘김 |
| 프론트 저장 | `persistVoiceChatTurn` 이 `voice_audio` 가 있을 때만 `message_q` 에 키를 넣음 |
| 목록 PHP | `learn_chatprocess_list.php` 가 질문 행에 답변 본문 `answer_text` 를 서브쿼리로 동봉 (재생용) |
| 목록 화면 | `chatlog.htm` 에 음성 칸 — ▶ ⏸ ■, 질문 `<audio>` 재생 후 답변을 본체 TTS 로 이어 읽기, 삭제 전에 파일 `DELETE` |

답변 TTS 는 챗봇 화면의 `TTSPlayer`(`/chat/js/tts_module.js`)를 그대로 불러 쓴다. 관리자 페이지에는 원래 플레이어가 없었다.

## 왜

목록 컬럼은 `voice_audio IS NULL` 이면 `-`, 값이 있으면 ▶ 다. 텍스트 채팅·요약 행은 NULL 이라 지금과 같고, 녹음이 붙는 날부터 ▶ 가 자동으로 뜬다. 컬럼이 없으면 `process_add.php` 의 `GetInsertSQL` 이 키를 **소리 없이 버리므로** ALTER 가 먼저다.

설계서 하나를 고쳤다. compose 가 묶는 건 `/data` 가 아니라 **`/data/logs` 뿐**이라, 설계서의 `/data/voice_records` 는 컨테이너 안에만 남아 재빌드 때 사라진다. 녹음 착수 때 볼륨 한 줄을 더하거나 `/data/logs` 밑으로 둔다.

키워드(#)는 본체 `/ask` 의 부산물(LLM 추출 + 검색 매칭)이고 통계표 `chatbot_chatting_keyword_stat_daily` 로 그린다. 음성은 재료가 없어 **본체가 질문 행에서 재계산하는 확정 API** 를 제안했고 보류 중이다.

## 어디

| 파일 (본체 `dev/`, git 밖) | 내용 |
|---|---|
| `class.chatbot.php:308` | DDL 한 줄 |
| `webrtc_voice_module.js:1227` | `q` → 절대 URL |
| `index2.js:2711` | `voice_audio` 조건부 전달 |
| `learn_chatprocess_list.php:112, :128` | `answer_text` 서브쿼리 + 유니코드 복원 |
| `chatlog.htm` | 음성 칸·재생 컨트롤러·삭제 훅 (11 덩어리) |
| `dev/backup_20260914_before_voicefield/` | 5개 파일 수정 전 원본 |

## 검증

- `index2.js`·`webrtc_voice_module.js` `node --check` 통과
- `chatlog.htm` 인라인 스크립트 4블록을 PHP 조각 제거 후 `node --check` 통과
- 배포 전이라 라이브 확인은 남음 — 테스트 행 하나의 `voice_audio` 에 WAV 주소를 넣어 ▶ → 답변 TTS 이어짐, 일시정지·정지, 삭제 시 `DELETE` 호출(404 무시)을 볼 것
