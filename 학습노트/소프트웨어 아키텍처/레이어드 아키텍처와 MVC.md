---
tags: [학습노트, 아키텍처, 레이어드, MVC, SpringBoot]
created: 2026-08-31
---

# 레이어드 아키텍처와 MVC

> [!summary] 한 줄 요약
> 레이어드 아키텍처는 애플리케이션을 **역할별 계층으로 나누는** 구조이고, Spring Boot의 Controller → Service → Repository 패턴이 그 대표적인 예다. 단순하고 배우기 쉽지만 규모가 커지면 **Fat Service** 문제가 생긴다.

## 계층과 의존 방향

```
Client
  ↓
Controller     ← 요청/응답 처리
  ↓
Service        ← 비즈니스 로직
  ↓
Repository     ← DB 접근
  ↓
Database
```

| Layer | 역할 | 예시 |
|---|---|---|
| Presentation | 외부 요청/응답 처리 | Controller, DTO |
| Business | 핵심 비즈니스 로직 | Service |
| Persistence | 데이터 접근 | Repository, DAO |
| Database | 실제 데이터 저장 | PostgreSQL, MySQL |

핵심은 **책임 분리**다. Controller는 "HTTP 요청은 내가 담당", Service는 "비즈니스 규칙은 내가 담당", Repository는 "DB 접근은 내가 담당"이라고 선을 긋는다. 의존 방향은 위에서 아래로 흐른다.

```java
@RestController
public class ProductController {
    private final ProductService productService;   // Repository를 직접 부르지 않는다

    @GetMapping("/products/{id}")
    public ProductDto getProduct(@PathVariable Long id) {
        return productService.getProduct(id);
    }
}
```

Controller가 Repository를 바로 호출하는 것도 작은 프로젝트에선 가능하지만, 비즈니스 로직이 커질수록 **Controller에 로직이 섞이기 쉬워** 권장되지 않는다.

## 장점 — 버그의 위치가 명확하다

가장 큰 장점은 구조가 단순해 **어디를 봐야 하는지 바로 안다**는 것이다.

```
API 요청 문제   → Controller 확인
계산/검증 문제  → Service 확인
DB 조회 문제    → Repository 확인
```

그래서 CRUD 중심의 Spring Boot 프로젝트에서 매우 흔하게 쓰인다.

## 단점 — Fat Service

프로젝트가 커지면 `ProductService`·`OrderService`·`PaymentService`가 각각 비대해지고, 레이어 간 결합도 강해진다. 이 지점에서 [[헥사고날 — 포트와 어댑터|Hexagonal]]·[[클린·어니언 — 의존성은 안쪽으로|Clean]]·[[DDD — 도메인 중심 설계|DDD]]와 비교하게 된다.

## MVC와 MVVM

같은 "내부 구조" 계열의 분리 방식이다.

- **MVC** (Model / View / Controller) — 웹 애플리케이션의 고전. `사용자 → Controller → Service·Model → View`. Spring MVC(`@Controller` + Model + Thymeleaf)가 대표적이다.
- **MVVM** (Model / View / ViewModel) — View와 ViewModel이 양방향으로 묶이는 구조(`View ↕ ViewModel ↕ Model`). 프론트엔드와 모바일(Android, WPF)에서 주로 쓴다.

## 관련

- [[헥사고날 — 포트와 어댑터]] — 레이어드의 결합 문제를 푸는 다음 단계
- [[00 학습 지도 (MOC)]] — 아키텍처 용어들의 층위 구분
