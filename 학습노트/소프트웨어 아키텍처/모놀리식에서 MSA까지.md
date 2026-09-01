---
tags: [학습노트, 아키텍처, MSA, 모놀리식, 인프라]
created: 2026-08-31
---

# 모놀리식에서 MSA까지

> [!summary] 한 줄 요약
> **시스템 전체를 어떻게 나눌 것인가**에 대한 스펙트럼이다. 하나로 뭉친 Monolith → 내부만 모듈로 쪼갠 Modular Monolith → 실행 단위까지 쪼갠 MSA. 작은 프로젝트에서 무조건 MSA는 좋지 않다.

## Monolithic — 하나의 애플리케이션

```
Frontend → Backend (User·Product·Order·Payment·Delivery) → Database
```

`shopping.jar` 하나에 모든 기능이 들어간다. **초기 개발이 빠르지만** 규모가 커지면 코드베이스 관리가 어려워진다.

## Modular Monolith — 실행은 하나, 내부는 분리

최근 많이 이야기되는 절충안. Spring Boot 서버 하나로 뜨지만 내부는 도메인 모듈로 강하게 나눈다.

```
Application
├── user     ├── domain ├── application └── infrastructure
├── order    ├── domain ├── application └── infrastructure
└── payment  ├── domain ├── application └── infrastructure
```

나중에 모듈 단위로 떼어내 MSA로 넘어가기도 좋아서, **개인 프로젝트나 스타트업에는 MSA보다 현실적인 선택**인 경우가 많다.

## MSA — 독립 실행 단위로 분리

```
Client → API Gateway → [User Service | Order Service | Payment Service]
                              ↓             ↓              ↓
                          User DB       Order DB      Payment DB
```

DB도 서비스별로 분리하는 것이 흔하다.

| 장점 | 단점 |
|---|---|
| 독립 배포 | 네트워크 통신 복잡 |
| 독립 확장 | 데이터 일관성 어려움 |
| 장애 격리 | 운영 복잡도 증가 |
| 팀 단위 개발 | 장애 추적 어려움 |

## SOA — MSA의 선배

MSA 이전부터 있던 서비스 중심 구조로, 여러 서비스를 **ESB(Enterprise Service Bus)** 로 연결한다. 초점이 다르다 — SOA는 **서비스 재사용과 기업 시스템 통합**, MSA는 **서비스 독립성과 독립 배포**.

## Serverless — 실행 환경을 클라우드에 맡기기

```
Client → API Gateway → AWS Lambda → DynamoDB
```

서버를 직접 운영하지 않고 함수 단위 코드에만 집중한다. AWS Lambda, Azure Functions, Google Cloud Functions가 대표적이다.

## 관련

- [[이벤트 기반·CQRS·이벤트 소싱]] — MSA와 짝을 이루는 서비스 간 통신 방식
- [[리버스 프록시 프리픽스 라우팅과 dev·운영 컨테이너 분리]] — 서비스 분리를 실제 인프라에 얹은 경험
- [[클라우드에서의 Ollama]] — 컨테이너·로드밸런서 배치의 실제 사례
