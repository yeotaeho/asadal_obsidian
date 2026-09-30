---
tags: [여태호, 일일작업, 기술파악]
created: 2026-09-16
---

# CLAUDE.md 지침 분리

> [!summary] 한 줄 요약
> 한 파일에 몰려 있던 에이전트 지침을 **CLAUDE.md 지도 + 경로별 규칙 + docs 지식**으로 나눠, 세션마다 항상 읽히는 양을 **약 380줄 → 약 120줄**로 줄였다. 서비스: [[기술파악]]

## 무슨 작업

CLAUDE.md 에는 작업 원칙·컨벤션·명령·문서 지도만 남겼다. 모듈별 "어떻게 할지" 규칙은 `.claude/rules/` 로, 구조·파이프라인·테이블 같은 "무엇인지" 지식은 `docs/` 로 옮겼다.

규칙 파일은 frontmatter `paths` 로 적용 경로를 건다. 그 경로의 파일을 편집할 때만 Claude Code 가 규칙을 불러온다. `paths` 가 없는 규칙은 항상 로드되며, Codex 리뷰 게이트와 옵시디언 기록 두 개뿐이다.

```
CLAUDE.md  (지도, 83줄, 항상)
├─ .claude/rules/          어떻게 할지
│   ├─ core-files.md       엔트리·공용 파일 → 수정 전 확인
│   ├─ sources.md          app/sources/, config/sources.yaml
│   ├─ pipeline.md         app/pipeline/, config/rules.yaml
│   ├─ notify.md           app/notify/, api/telegram·discord, jobs/notify·feedback
│   ├─ database.md         app/db/, alembic.ini, tests/integration/
│   ├─ testing.md          tests/
│   ├─ codex-review.md     항상
│   └─ obsidian-worklog.md 항상 (양식은 templates/ 에서 기록할 때만 읽음)
└─ docs/                   무엇인지
    ├─ architecture.md     폴더 지도 · 코드 추적 순서 · 역인덱스 · 배포
    ├─ pipeline.md         v2 단계별 요점 · LLM 예산 · 상태 전이 · 발송 정책
    └─ database.md         테이블 · 임베딩 · Neon 연결 · 마이그레이션
```

## 왜

CLAUDE.md 가 옵시디언 양식·Codex 절차까지 품으면서 **약 380줄**이 됐고, 수집기 하나를 고칠 때도 발송·DB 규칙이 전부 컨텍스트에 올라왔다. `@import` 로 파일만 쪼개면 정리는 되지만 결국 세션 시작 시 다 로드되므로, 경로별 규칙으로 필요한 것만 읽히게 했다.

진행 중에 두 번 방향을 바꿨다. 처음에는 Claude Code 와 Codex 공용으로 `AGENTS.md` 를 두고 CLAUDE.md 가 import 했다. 사용자가 **Claude Code 만 쓰기로** 해서 `AGENTS.md` 는 없애고 내용을 CLAUDE.md 로 합쳤다. 이때 Codex 리뷰 규칙까지 같이 지웠다가, 사용자 요청으로 리뷰 게이트는 **원래대로 되돌렸다**. 작성자와 다른 모델이 보는 독립 리뷰라는 점이 Claude `/code-review` 대비 장점이라 최종 게이트로 유지한다.

## 어디

| 커밋 | 내용 | 파일 |
|---|---|---|
| `10ca292` | CLAUDE.md 를 지도로 줄이고 규칙·지식을 분리 | `CLAUDE.md`, `.claude/rules/*.md`, `.claude/templates/obsidian-notes.md`, `docs/*.md` |

브랜치 `refactor/claude-md-rules` → [PR #6](https://github.com/yeotaeho/trend/pull/6). `.claude/settings.json` 은 로컬 절대 경로(graphify 훅)가 있어 커밋하지 않았다.

## 검증

- 규칙 파일 8개의 frontmatter `paths` 를 YAML 로 파싱해 경로 목록을 확인했다.
- Codex 리뷰(`origin/main` 대비 브랜치) 결과 **막는 지적 없음**. 코드·런타임 설정 변경이 없음을 확인했다.
- PR #6 CI 는 **통과 1 · 건너뜀 1**, 병합 가능(CLEAN).
- 코드 변경이 없어 `pytest` 는 돌리지 않았다.

관련: [[기술파악 - 결정 기록]], [[기술파악 - 폴더·파일 역할 지도]]
