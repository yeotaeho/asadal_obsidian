---
tags: [학습노트, moc, 아키텍처]
created: 2026-08-31
---

# 00 학습 지도 — 소프트웨어 아키텍처

레이어드 구조에서 출발해 DDD·헥사고날·MSA까지, 아키텍처 용어들의 **층위**를 정리한 지도.
2026-08-31 정리.

## 가장 중요한 것 — 이 용어들은 같은 층위가 아니다

Layered·Hexagonal·Clean·MSA는 비슷한 레벨의 **아키텍처 스타일**이지만, DDD는 **설계 방법론**, Adapter는 **디자인 패턴**이다. 실무에서 섞여 쓰이니 층위부터 갈라야 헷갈리지 않는다.

```
                    Software Architecture
                            │
      ┌──────────────┬──────┴───────┬──────────────────┐
  시스템 구조      내부 구조      설계 방법        통신·데이터 구조
      │              │              │                  │
  Monolith       Layered          DDD            Event Driven
  Modular Mono   Hexagonal                       CQRS
  MSA / SOA      Clean / Onion                   Event Sourcing
  Serverless     MVC / MVVM
```

| 층위 | 답하는 질문 |
|---|---|
| 시스템 구조 | 시스템 **전체를 어떻게 나눌** 것인가 |
| 내부 구조 | 애플리케이션 **하나의 안을 어떻게 구성**할 것인가 |
| 설계 방법 | 비즈니스 도메인을 **어떻게 모델링**할 것인가 |
| 통신·데이터 | 서비스와 데이터가 **어떻게 상호작용**할 것인가 |

## 서로 배타적이지 않다

층위가 다르니 **겹쳐 쓰는 게 정상**이다. 아래 조합이 실제로 가능하다.

```
MSA
 ├─ Order Service   : Hexagonal + DDD
 │        │ OrderCreated Event
 │        ↓  Kafka
 └─ Payment Service : Clean + DDD
```

## 읽는 순서 (추천)

1. [[레이어드 아키텍처와 MVC]] — Controller → Service → Repository. 가장 기본이자 Fat Service의 출발점
2. [[DDD — 도메인 중심 설계]] — Entity·Value Object·Aggregate·Bounded Context
3. [[헥사고날 — 포트와 어댑터]] — 도메인을 외부 기술에서 떼어내기. 어댑터 패턴 포함
4. [[클린·어니언 — 의존성은 안쪽으로]] — 같은 목표의 다른 표현
5. [[모놀리식에서 MSA까지]] — Monolith → Modular Monolith → MSA, SOA와 Serverless
6. [[이벤트 기반·CQRS·이벤트 소싱]] — 나눈 서비스들이 통신하는 법

백엔드 기준으로는 **Layered → DDD → Hexagonal → Modular Monolith → MSA** 흐름을 잡으면 큰 뼈대가 선다.

## 전체를 관통하는 문장

> **MSA는 "시스템을 어떻게 나누나", Hexagonal·Clean·Layered는 "앱 하나의 안을 어떻게 짜나", DDD는 "도메인을 어떻게 모델링하나"의 답이다.**
> 서로 다른 질문에 답하므로 골라 쓰는 게 아니라 겹쳐 쓴다.
