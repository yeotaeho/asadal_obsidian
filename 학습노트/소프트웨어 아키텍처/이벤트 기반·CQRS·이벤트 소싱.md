---
tags: [학습노트, 아키텍처, EDA, CQRS, 이벤트소싱]
created: 2026-08-31
---

# 이벤트 기반·CQRS·이벤트 소싱

> [!summary] 한 줄 요약
> 셋 다 **데이터와 서비스 간 상호작용을 어떻게 설계할 것인가**에 대한 답이다. EDA는 호출 대신 이벤트로, CQRS는 읽기와 쓰기를 나눠서, Event Sourcing은 상태 대신 이력을 저장해서 푼다.

## Event-Driven Architecture — 호출하지 않고 알린다

서비스가 서로 직접 호출하는 대신 이벤트를 발행하고 소비한다.

```
Order Service → OrderCreated Event → Kafka
                                      ├→ 결제
                                      ├→ 배송
                                      └→ 알림
```

주문 서비스는 "결제 시작해! 배송 준비해! 이메일 보내!"라고 일일이 호출하지 않고 **`OrderCreated` 이벤트 하나만 발행**한다. 필요한 서비스가 알아서 소비한다. Kafka·RabbitMQ·AWS SQS/SNS를 주로 쓰며, [[모놀리식에서 MSA까지|MSA]]와 자주 함께 간다.

## CQRS — 쓰기와 읽기를 가른다

Command Query Responsibility Segregation. **Command(변경)** 와 **Query(조회)** 의 책임을 분리한다.

| | 일반 구조 | CQRS |
|---|---|---|
| 한 서비스 | createOrder, updateOrder, getOrder, getOrders | **Command Side** — CreateOrder, CancelOrder |
| | | **Query Side** — GetOrder, SearchOrder |

더 나아가 저장소도 나눌 수 있다 — **쓰기는 PostgreSQL, 읽기는 Elasticsearch**. 조회 요구와 쓰기 요구가 크게 다른 시스템에서 유용하다.

## Event Sourcing — 상태가 아니라 사건을 저장한다

일반 DB는 현재 상태(`balance = 70000`)를 저장하지만, Event Sourcing은 **일어난 사건**을 순서대로 쌓는다.

```
계좌 생성 → +100000 입금 → -20000 출금 → -10000 출금
현재 잔액 = 100000 - 20000 - 10000 = 70000  (계산으로 도출)
```

장점은 **모든 변경 이력을 추적할 수 있다**는 것이다. 금융·회계·주문·감사 시스템에서 유용하며 CQRS와 함께 쓰이는 경우가 많다.

## 관련

- [[모놀리식에서 MSA까지]] — 이 통신 방식이 필요해지는 시스템 구조
- [[DDD — 도메인 중심 설계]] — Domain Event가 EDA의 이벤트로 이어진다
