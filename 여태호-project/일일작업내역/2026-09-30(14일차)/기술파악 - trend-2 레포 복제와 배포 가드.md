---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-30
---

# trend-2 레포 복제와 배포 가드

> [!summary] 한 줄 요약
> 최신 main 을 새 레포 `yeotaeho/trend-2` 에 푸시했다. 배포 잡에 원본 레포 가드를 넣어 trend-2 에서는 **check 만 돌고 deploy 는 건너뛴다**. 기능: [[배포·CI-CD]]

## 무슨 작업

`origin/main`(`ecafe28`, 모바일 앱 v0.1 머지 포함) 위에 가드 커밋 하나를 얹어 `https://github.com/yeotaeho/trend-2.git` 의 `main` 으로 푸시했다. 로컬 main 은 origin 보다 155 커밋 뒤라 쓰지 않았고 임시 워크트리에서 작업했다.

```yaml
deploy:
  if: github.ref == 'refs/heads/main' && github.repository == 'yeotaeho/trend'
```

## 왜

trend-2 는 빈 public 레포이고 Actions 가 켜져 있었다. 그대로 푸시하면 deploy 잡이 돈다.

CI 의 `IMAGE` 가 `ghcr.io/${{ github.repository_owner }}/tech-radar` 라 owner 가 같은 복제 레포도 **운영 이미지와 같은 이름**을 쓴다. `:latest` 를 덮으면 VM 의 다음 `docker compose pull` 이 복제 레포 이미지를 받는다. SSH 단계는 trend-2 에 시크릿이 없어 실패하겠지만, 이미지 푸시는 그 앞 단계다.

trend-2 의 Actions 를 끄는 안도 있었다. 이력은 원본과 같게 남지만 테스트도 안 돌고, 나중에 Actions 를 켜면 배포가 되살아난다. 사용자가 레포 가드를 골랐다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `c29eeb2` | 배포 잡을 원본 레포에서만 실행 | `.github/workflows/ci.yml` |

브랜치 `fix/deploy-repo-guard` (로컬). trend-2 `main` = `c29eeb2`. 원본 trend 에는 아직 반영하지 않아 **두 레포 이력이 1커밋 갈라져 있다**.

## 검증

- YAML 파싱 확인. `jobs.deploy.if` 가 의도한 식으로 읽힌다
- Codex 리뷰(`--base ecafe28 --scope branch`) 지적 없음
- trend-2 run `36649963491` — **check success · deploy skipped**
- 원본 trend 에는 새 실행 없음(마지막 run 은 09-29 `ecafe28`)
