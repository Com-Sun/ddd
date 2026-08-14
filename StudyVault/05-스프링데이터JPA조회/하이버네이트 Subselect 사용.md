---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 5장 §5.10 (p.195~198)
keywords: subselect, immutable, synchronize, hibernate, view
---

# 하이버네이트 @Subselect 사용 (★★★)

#ddd #query #subselect #jpa #concept

## Overview Table (세 애너테이션)

| 애너테이션 | 역할 |
|-----------|------|
| **`@Subselect`** | **조회 쿼리를 값으로 갖는다.** 하이버네이트가 이 쿼리 결과를 **매핑할 테이블처럼 사용** |
| **`@Immutable`** | 해당 엔티티의 **매핑 필드/프로퍼티가 변경되어도 DB에 반영하지 않고 무시** |
| **`@Synchronize`** | 해당 엔티티와 관련된 **테이블 목록을 명시.** 로딩 전에 변경이 있으면 **먼저 플러시** |

| 항목 | 핵심 |
|------|------|
| 정체 | 하이버네이트의 **JPA 확장 기능** — **쿼리 결과를 `@Entity`로 매핑** |
| 비유 | RDBMS의 **뷰(view)** |
| ★ 장점 | 일반 `@Entity`와 같으므로 **`findById()`, JPQL, Criteria, 스펙** 모두 사용 가능 |
| ★ 주의 | `@Subselect`의 쿼리가 **`from` 절의 서브 쿼리**로 들어간다 |

## 사용 예 (리스트 5.8)

```java
import org.hibernate.annotations.Immutable;
import org.hibernate.annotations.Subselect;
import org.hibernate.annotations.Synchronize;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import java.time.LocalDateTime;

@Entity
@Immutable
@Subselect(
    """
    select o.order_number as number,
    o.version, o.orderer_id, o.orderer_name,
    o.total_amounts, o.receiver_name, o.state, o.order_date,
    p.product_id, p.name as product_name
    from purchase_order o inner join order_line ol
        on o.order_number = ol.order_number
        cross join product p
    where
    ol.line_idx = 0
    and ol.product_id = p.product_id
    """
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
    @Id
    private String number;
    private long version;

    @Column(name = "orderer_id")
    private String ordererId;

    @Column(name = "orderer_name")
    private String ordererName;

    ... 생략

    protected OrderSummary() {
    }
}
```

> [!important] `@Subselect`의 원리
> `@Subselect`는 **조회 쿼리를 값으로 갖는다.** 하이버네이트는 이 `select` 쿼리의 결과를 **매핑할 테이블처럼 사용**한다.
>
> RDBMS가 **여러 테이블을 조인해서 조회한 결과를 한 테이블처럼 보여주기 위한 용도로 뷰를 사용하는 것처럼**,
> `@Subselect`를 사용하면 **쿼리 실행 결과를 매핑할 테이블처럼 사용**한다.

## `@Immutable` — 수정 불가

> [!warning] 왜 필요한가
> **뷰를 수정할 수 없듯이 `@Subselect`로 조회한 `@Entity` 역시 수정할 수 없다.**
> 실수로 매핑 필드를 수정하면 하이버네이트는 변경을 반영하는 `UPDATE` 쿼리를 실행할 것이다.
> 그런데 **매핑한 테이블이 없으므로 에러가 발생**한다.
>
> **`@Immutable`을 사용하면 해당 엔티티의 매핑 필드/프로퍼티가 변경되어도 DB에 반영하지 않고 무시한다.**

## ★ `@Synchronize` — 플러시 타이밍 문제

### 문제 상황

```java
// purchase_order 테이블에서 조회
Order order = orderRepository.findById(orderNumber);
order.changeShippingInfo(newInfo);       // 상태 변경

// 변경 내역이 DB에 반영되지 않았는데 purchase_order 테이블에서 조회
List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
```

> [!danger] 무엇이 문제인가
> 특별한 이유가 없으면 하이버네이트는 **트랜잭션을 커밋하는 시점에 변경사항을 DB에 반영**한다.
> 따라서 `Order`의 변경 내역을 **아직 `purchase_order` 테이블에 반영하지 않은 상태**에서,
> 그 테이블을 사용하는 `OrderSummary`를 조회하게 된다.
> → **`OrderSummary`에는 최신 값이 아닌 이전 값이 담기게 된다.**

```
   order.changeShippingInfo(...)   ← 영속성 컨텍스트에만 반영
              │
              │  (아직 DB에 UPDATE 안 됨)
              ▼
   OrderSummary 조회               ← purchase_order를 읽음
              ▼
        ❌ 이전 값 조회
```

### 해결

> [!important]
> **`@Synchronize`는 해당 엔티티와 관련된 테이블 목록을 명시한다.**
> 하이버네이트는 **엔티티를 로딩하기 전에 지정한 테이블에 변경이 발생하면 플러시(flush)를 먼저** 한다.
>
> `OrderSummary`의 `@Synchronize`가 `purchase_order` 테이블을 지정하고 있으므로,
> `OrderSummary`를 로딩하기 전에 `purchase_order`에 변경이 발생하면 **관련 내역을 먼저 플러시**한다.
> 따라서 **로딩하는 시점에는 변경 내역이 반영된다.**

```
   order.changeShippingInfo(...)
              │
              ▼
   OrderSummary 조회 시도
              │
              ├──▶ @Synchronize 확인 → purchase_order 변경 있음 → FLUSH
              ▼
        ⭕ 최신 값 조회
```

## ★ 장점 — 일반 @Entity처럼 쓸 수 있다

> [!tip]
> `@Subselect`를 사용해도 **일반 `@Entity`와 같기 때문에 `find()`, JPQL, Criteria를 사용해서 조회할 수 있다**는 것이 장점이다.
> **이것은 [[스펙 조합|스펙]]을 사용할 수 있다는 것도 포함한다.**

```java
// @Subselect를 적용한 @Entity는 일반 @Entity와 동일한 방법으로 조회할 수 있다.
Specification<OrderSummary> spec = OrderSummarySpecs.orderDateBetween(from, to);
Pageable pageable = PageRequest.of(1, 10);
List<OrderSummary> results = orderSummaryDao.findAll(spec, pageable);
```

## ★ 주의 — 서브 쿼리 형태가 된다

> [!warning]
> `@Subselect`는 이름처럼 **`@Subselect`의 값으로 지정한 쿼리를 `from` 절의 서브 쿼리로 사용**한다.

```sql
select osm.number as number1_0_, ... 생략
from (
    select o.order_number as number,
        o.version,
        p.name as product_name
    from purchase_order o inner join order_line ol
        on o.order_number = ol.order_number
        cross join product p
    where
    ol.line_idx = 0
    and ol.product_id = p.product_id
) osm
where osm.orderer_id = ? order by osm.number desc
```

> [!important]
> `@Subselect`를 사용할 때는 **쿼리가 이러한 형태를 갖는다는 점을 유념**해야 한다.
> **서브 쿼리를 사용하고 싶지 않다면 네이티브 SQL 쿼리를 사용하거나
> 마이바티스와 같은 별도 매퍼를 사용해서 조회 기능을 구현**해야 한다.

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "쿼리 결과를 `@Entity`로 매핑" | **`@Subselect`** (하이버네이트 확장) |
| "`@Subselect`는 무엇과 비슷한가" | RDBMS의 **뷰(view)** |
| "수정 시 UPDATE로 에러가 나는 것을 막음" | **`@Immutable`** |
| "로딩 전 플러시를 유발" | **`@Synchronize`** |
| "`@Synchronize`가 없으면?" | **이전 값(stale)** 이 조회됨 |
| "`@Subselect` 엔티티에 스펙을 쓸 수 있나?" | **가능하다** (일반 `@Entity`와 동일) |
| "실행 쿼리의 형태" | `@Subselect` 쿼리가 **`from` 절 서브 쿼리** |
| "서브 쿼리를 피하려면" | **네이티브 SQL** 또는 **마이바티스** 등 별도 매퍼 |

## Related Notes

- [[동적 인스턴스를 이용한 조회]] — 다른 조회 전용 기법
- [[스펙 조합]] / [[스프링 데이터 JPA 스펙 구현]] — `@Subselect`에도 적용 가능
- [[페이징 처리하기]]
- [[검색을 위한 스펙]]
- [[CQRS]] — 11장의 조회 전용 모델
- [[연습문제 - 스프링 데이터 JPA 조회 기능]]
