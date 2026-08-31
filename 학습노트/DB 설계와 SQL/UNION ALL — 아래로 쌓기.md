---
tags: [학습노트, DB, SQL, UNION]
created: 2026-08-26
---

# UNION ALL — 아래로 쌓기

> [!summary] 한 줄 요약
> JOIN이 PK-FK 관계로 테이블을 **옆으로** 붙이는 결합 연산이라면, UNION ALL은 관계 없이 **규격(개수·타입·순서)만 맞으면 아래로** 쌓는 집합 연산이다. 기준은 컬럼명이 아니라 **순서**다.

## JOIN과는 아예 다른 카테고리

> JOIN은 '결혼' — 나와 상대의 정보가 합쳐져 한 행이 된다. UNION ALL은 '줄 세우기' — 1반 뒤에 2반을 그대로 세운다.

| 구분 | JOIN (결합·가로) | UNION ALL (집합·세로) |
|---|---|---|
| 핵심 조건 | 연결 고리 (`ON a.id = b.fk_id`) | **규격 일치** (개수·타입·순서) |
| 데이터 관계 | 부모-자식 관계 필요 | 관계 없어도 형태만 같으면 됨 |
| 실패 시 | 매칭 안 되면 행 누락 | 규격 다르면 쿼리 에러 |
| 쓰임 | 산정 근거 옆에 파일명 표시 | 올해 로그 아래에 작년 로그 이어 붙이기 |

## 기준은 컬럼명이 아니라 순서다

UNION ALL은 컬럼 이름을 비교하지 않는다. **첫 번째 쿼리가 만든 칸에 두 번째 쿼리 데이터를 순서대로 들이붓고**, 결과 헤더도 첫 번째 쿼리의 이름을 따른다. `SELECT 이름, 나이`와 `SELECT 성함, 연세`도 합쳐진다 — 순서가 어긋나면 이름 칸에 금액이 들어가는 사고가 나므로, 합치고 싶은 컬럼을 **양쪽 SELECT에 같은 순서로** 직접 나열해야 한다.

한쪽에만 있는 컬럼은 가짜 컬럼으로 규격을 맞춘다.

```sql
SELECT id, amount, note        FROM table_a
UNION ALL
SELECT id, amount, NULL AS note FROM table_b;  -- note가 없으니 NULL로 칸 채움
```

## UNION vs UNION ALL

- **UNION** — 합친 뒤 중복 행을 찾아 제거한다. 중복 탐색 연산 때문에 **느리다**.
- **UNION ALL** — 묻지도 따지지도 않고 그대로 쌓는다. 10건 + 5건 = 무조건 15건. **빠르다**.

중복이 나올 일이 없거나 있어도 무방하면 **무조건 UNION ALL부터** 고려한다.

## CTE와의 조합

연도별·성격별로 나뉜 테이블을 UNION ALL로 쌓아 [[CTE — 이름 붙인 가상 테이블|CTE]] 하나로 감싸면, 흩어진 데이터를 단일 테이블처럼 통계 낼 수 있다.

```sql
WITH TotalLogs AS (
    SELECT id, amount FROM table_2025
    UNION ALL
    SELECT id, amount FROM table_2026
)
SELECT SUM(amount) FROM TotalLogs;
```

## 관련

- [[JOIN은 왜 하는가]] — 옆으로 붙이는 반대 방향
- [[CTE — 이름 붙인 가상 테이블]] — 쌓은 결과를 담는 그릇
