---
tags: [학습노트, 아키텍처, DDD, 도메인모델링]
created: 2026-08-31
---

# DDD — 도메인 중심 설계

> [!summary] 한 줄 요약
> DDD(Domain-Driven Design)는 아키텍처가 아니라 **설계 철학·방법론**이다. 프로그램 구조를 DB나 API 중심이 아니라 **실제 비즈니스 개념 중심**으로 설계하자는 것.

## 무엇이 다른가 — 계층이 아니라 도메인으로 본다

쇼핑몰을 만든다면 구조를 `Controller / Service / Repository`로만 보는 게 아니라, **회원·상품·주문·결제·배송**이라는 도메인을 중심으로 생각한다.

```
Order
 ├─ OrderItem
 ├─ OrderStatus
 ├─ Money
 └─ Address
```

## 핵심 용어

| 용어 | 의미 |
|---|---|
| Entity | 식별자를 갖고 생애 주기 동안 추적되는 객체 |
| Value Object | 식별자 없이 값 자체로 의미를 갖는 객체 (Money, Address) |
| Aggregate | 함께 변경되어야 하는 **일관성 단위** |
| Aggregate Root | 그 단위의 진입점. 외부는 루트를 통해서만 접근 |
| Repository | Aggregate를 저장·조회하는 창구 |
| Domain Service | 특정 Entity에 속하지 않는 도메인 로직 |
| Domain Event | 도메인에서 일어난 사건 |
| Bounded Context | 용어와 모델이 일관되게 통하는 경계 |

예를 들어 `Order`가 Aggregate Root이고, `OrderItem`·`ShippingAddress`·`PaymentInfo`가 하나의 일관성 단위를 이룬다. Aggregate 단위가 곧 [[ERD 부모-자식 관계|테이블의 부모-자식 경계]]와 겹치는 경우가 많다.

## 언제 쓰나

**복잡한 비즈니스 로직**이 있는 서비스에서 특히 유용하다. 단순 CRUD라면 [[레이어드 아키텍처와 MVC|레이어드]]로 충분하다.

DDD는 [[헥사고날 — 포트와 어댑터|Hexagonal]]·[[클린·어니언 — 의존성은 안쪽으로|Clean]]과 자주 함께 쓰인다. 그 구조들이 "도메인을 중심에 두라"고 말할 때, **그 도메인을 어떻게 모델링할지**를 채워주는 것이 DDD다.

## 관련

- [[헥사고날 — 포트와 어댑터]] — DDD의 도메인을 외부 기술로부터 지키는 구조
- [[ERD 부모-자식 관계]] — Aggregate 경계와 맞닿는 테이블 설계
