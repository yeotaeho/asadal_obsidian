---
tags: [학습노트, DB, SQL, CTE]
created: 2026-08-26
---

# CTE — 이름 붙인 가상 테이블

> [!summary] 한 줄 요약
> CTE(`WITH` 절)는 JOIN 같은 '연산'이 아니라 **쿼리 하나가 실행되는 동안만 존재하는 가상 테이블 '정의'** 다. 복잡한 로직을 먼저 그릇에 담아 이름을 붙이고, 그 그릇을 실제 테이블과 JOIN한다.

## 기본 구조 — 정의하고, 참조한다

```sql
WITH FileCounts AS (              -- 정의: "이 쿼리 안에서 FileCounts는 이 데이터다"
    SELECT basis_id, COUNT(*) AS file_cnt
    FROM evidence_files
    GROUP BY basis_id
    HAVING COUNT(*) >= 3
)
SELECT b.*, f.file_cnt            -- 사용: 방금 정의한 표를 진짜 테이블처럼 JOIN
FROM calculation_basis b
JOIN FileCounts f ON b.id = f.basis_id;
```

서브쿼리로 짰다면 FROM 절 안에 괄호가 중첩됐겠지만, CTE는 **"파일 개수부터 세고, 그다음 합친다"는 논리가 위에서 아래로** 읽힌다.

## JOIN·VIEW와의 위치 관계

- **JOIN** — 테이블을 옆으로 합치는 '행위'(연산자). **CTE** — 데이터를 쓰기 편한 그릇에 담는 '구조화'(정의). 보통 **CTE로 만든 그릇을 실제 테이블과 JOIN**해서 쓴다.

| 구분 | CTE (`WITH`) | VIEW (`CREATE VIEW`) |
|---|---|---|
| 저장 | 쿼리 실행 중에만 존재 | DB에 영구 저장 |
| 유효 범위 | **SQL 문장 하나** | 삭제 전까지 계속 |
| 용도 | 복잡한 쿼리의 가독성 | 자주 쓰는 JOIN의 공용화 |

## 핵심 패턴 — 미리 줄여놓고 JOIN한다

일반 JOIN은 "전체 대 전체"를 붙인 뒤 필터링하지만, CTE 패턴은 **필요한 데이터만 먼저 골라 작은 표를 만들고 가벼워진 상태로 연결**한다.

```sql
WITH CompletedBasis AS (
    SELECT id, name FROM calculation_basis WHERE status = 'COMPLETED'
)
SELECT cb.name, e.file_name
FROM CompletedBasis cb
JOIN evidence_files e ON cb.id = e.basis_id;
```

수백만 건 로그에서 조건에 맞는 ID 몇 개만 CTE로 추린 뒤 상세 테이블과 붙이면, **전체 로그를 통째로 JOIN하는 불상사**를 막는다. 쿼리 실행 중 메모리엔 원본 물리 테이블과 요약본 가상 테이블이 공존하는 셈이다.

그 외 쓰임 — **셀프 조인**(사원 ↔ 상사처럼 같은 테이블을 다시 붙일 때 한쪽을 CTE로 정의하면 명확), **재귀 CTE**(조직도·댓글-답글처럼 부모-자식이 끝없이 이어지는 계층 조회), [[UNION ALL — 아래로 쌓기|UNION ALL]] 결과를 CTE로 감싸 통계 내기.

## 관련

- [[JOIN은 왜 하는가]] — CTE가 덜어주는 JOIN의 비용
- [[UNION ALL — 아래로 쌓기]] — CTE와 자주 조합되는 수직 결합
