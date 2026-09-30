---
tags: [아사달, 일일작업, AI음성]
created: 2026-09-14
---

# 음성채팅기록 녹음 저장소 NAS 확정과 설계 개정

> [!summary] 한 줄 요약
> 질문 녹음 저장소를 **NAS** 로 확정하고 dev 공유 쓰기를 실측했다. NAS 마운트가 `hard` 라는 사실이 설계를 바꿔, 녹음은 **메모리에 모았다가 짝이 맺어질 때 전용 스레드가 기록**하도록 설계서를 개정했다. 기능: [[음성채팅기록]]

## 무슨 작업

과장님이 녹음 저장 위치를 "스토리지" 로 답해 무엇을 가리키는지 추적했다. 플랫폼에 객체 저장소는 없고 NFS NAS 하나다. 본체 `DATA_HOME=/NAS/data_dev` 아래에 `account_settings`·`uploads`·`FAISS_INDEX` 등이 있고, 본체는 컨테이너가 아니라 systemd 프로세스라 NAS 를 직접 쓴다.

| 서비스 | 구동 | 저장소 |
|---|---|---|
| 본체 app·edu | systemd (dev 9000·9001, 운영 7000·7001) | 호스트에서 `/NAS/...` 직접 |
| 음성 서버 | 이 서버 유일한 docker compose | `./data/logs` 쓰기, `account_settings` 읽기 전용 |
| GPU `ffmpeg_api`·`cosine_sim` | 다른 서버, `docker run` | `/home/...` 로컬 경로 |

**컨테이너가 NAS 에 쓰는 첫 사례**라 권한을 실측했다.

```
mount | grep -i nas
  10.20.20.11:/volume1/data_dev on /NAS/data_dev type nfs4 (rw,...,hard,...,sec=sys)
음성 이미지 임시 컨테이너 -v /NAS/data_dev:/nas
  id -u → 0 / touch → 소유 0 0 / rm → rc=0
```

root 강등이 없어 쓰기·삭제가 된다. 그 대신 공유 전체를 물리면 컨테이너가 공유 전체에 root 권한을 가지므로 `voice_records` 하위 폴더만 붙인다.

## 왜 설계를 바꿨나

**`hard` 마운트.** NAS 가 응답하지 않으면 입출력이 실패하지 않고 복구될 때까지 멈춘다. 초안은 업링크 `recv` 에서 WAV 를 바로 쓰게 했는데, `recv` 는 이벤트 루프 위라 NAS 가 한 번 멈추면 **단일 프로세스의 모든 통화가 얼어붙는다.**

```
통화 중        B 프레임 → 모노 s16 → 메모리 링버퍼 (NAS 안 만짐)
발화 시작·종료  item_id 구간을 메모리에서 자름
짝이 맺어짐     전용 실행기에 쓰기를 넘기고 경로·토큰을 통지 (대기열 64 초과면 포기)
전용 스레드     24kHz 변환 → .part 로 쓰기 → 이름 확정
```

실행기를 따로 두는 이유는 asyncio 기본 실행기가 DNS 조회에도 쓰여, NAS 대기로 차면 새 통화의 OpenAI 연결까지 막히기 때문이다. GET 의 경로 검증 `resolve()` 도 NAS 에 `lstat` 을 해 같은 실행기로 보낸다.

코드 재확인에서 둘을 더 잡았다.

- 대기시간 미사용 세션은 게이트 없는 `UplinkAudioProxyTrackV4` 를 타는데 **B 탭이 없다.** 게이트 트랙에만 훅을 걸면 그 세션은 녹음 0건이라 두 트랙 모두 건다.
- 초안의 "오디오 탭 `/data` 마운트 그대로" 는 오류였다. compose 가 묶는 건 `/data/logs` 뿐이다.

## 어디

| 대상 | 내용 |
|---|---|
| 설계서 (로컬, gitignore) | `docs/superpowers/specs/2026-09-09-voice-record-playback-design.md` — §0 개정 요약, §2 전체 흐름(완료·대기 표시), §5.3 NAS 가 멈출 때, §5.7 마운트·실측, §5.9 프론트 완료분 |
| 결정 기록 | 인프라 3행, 아키텍처 4행, 보류 2행 추가 — [[AI음성 - 결정 기록]] |

## 검증

- NAS 마운트 옵션과 dev 공유 쓰기·삭제를 서버에서 실측(위 코드블록)
- 코드 변경은 없다. 설계 근거 줄 번호를 현행 코드로 전부 갱신했다
- 남은 미검증 가정: OpenAI `audio_start_ms` 가 B 누적 ms 와 같다는 전제. 첫 dev 통화에서 오디오 탭 B 파일과 대조한다
