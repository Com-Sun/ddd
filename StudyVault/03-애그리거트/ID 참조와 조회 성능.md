---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 3장 §3.4.1 (p.118~120)
keywords: n-plus-1, query-optimization, dao, jpql, cache
---

# ID 참조와 조회 성능 — N+1 문제 (★★★)

#ddd #aggregate #aggregate-reference #query #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 문제 | ID 참조는 **지연 로딩과 같은 효과** → 대표적 부작용이 **N+1 조회 문제** |
| N+1 정의 | 조회 대상이 **N개**일 때, N개를 읽는 **1번의 쿼리** + 연관 데이터를 읽는 **N번의 쿼리** |
| 결과 | 더 많은 쿼리 실행 → **전체 조회 속도가 느려진다** |
| 쉬운 해법(비추천) | ID 참조 → **객체 참조 + 즉시 로딩** (ID 참조를 되돌리는 셈) |
| **권장 해법** | **조회 전용 쿼리** (별도 DAO + 조인 한 번) |
| 저장소가 다를 때 | **캐시** 또는 **조회 전용 저장소** |

## N+1 조회 문제

주문 목록을 보여주려면 **주문 애그리거트 + 상품 애그리거트 + 회원 애그리거트**를 함께 읽어야 한다.

```java
Member member = memberRepository.findById(ordererId);
List<Order> orders = orderRepository.findByOrderer(ordererId);
List<OrderView> dtos = orders.stream()
        .map(order -> {
            ProductId prodId = order.getOrderLines().get(0).getProductId();
            // 각 주문마다 첫 번째 주문 상품 정보 로딩 위한 쿼리 실행
            Product product = productRepository.findById(prodId);
            return new OrderView(order, member, product);
        }).collect(toList());
```

```
   주문 10개인 경우
   ├─ 주문을 읽어오기 위한 쿼리 ................ 1번
   └─ 주문별 상품을 읽어오기 위한 쿼리 ......... 10번
                                        총 11번 = N + 1
```

> [!danger] 정의
> **"조회 대상이 N개일 때 N개를 읽어오는 한 번의 쿼리와, 연관된 데이터를 읽어오는 쿼리를 N번 실행한다"** 고 해서 이를 **N+1 조회 문제**라고 부른다.
>
> ID를 이용한 애그리거트 참조는 **지연 로딩과 같은 효과**를 만드는데, **지연 로딩과 관련된 대표적인 문제가 바로 N+1 조회 문제**다.

> [!warning]
> 한 DBMS에 데이터가 있다면 **조인을 이용해서 한 번에 모든 데이터를 가져올 수 있음에도 불구하고**, 주문마다 상품 정보를 읽어오는 쿼리를 실행하게 된다.

## 해법 ① — 객체 참조 + 즉시 로딩 (권장하지 않음)

조인을 사용하는 **가장 쉬운 방법**은 ID 참조 방식을 **객체 참조 방식으로 바꾸고 즉시 로딩을 사용**하도록 매핑 설정을 바꾸는 것이다.

> [!warning]
> 하지만 이 방식은 애그리거트 간 참조를 **ID 참조에서 객체 참조로 다시 되돌리는 것**이다.
> → [[ID를 이용한 애그리거트 참조]]의 세 가지 문제가 되살아난다.

## ⭕ 해법 ② — 조회 전용 쿼리

> [!important]
> ID 참조 방식을 사용하면서 N+1 조회 문제가 발생하지 않도록 하려면 **조회 전용 쿼리**를 사용하면 된다.
> 데이터 조회를 위한 **별도 DAO**를 만들고, DAO의 조회 메서드에서 **한 번의 쿼리로 필요한 데이터를 로딩**하면 된다.

```java
@Repository
public class JpaOrderViewDao implements OrderViewDao {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<OrderView> selectByOrderer(String ordererId) {
        String selectQuery =
            "select new com.myshop.order.application.dto.OrderView(o, m, p) " +
            "from Order o join o.orderLines ol, Member m, Product p " +
            "where o.orderer.memberId.id = :ordererId " +
            "and o.orderer.memberId = m.id " +
            "and index(ol) = 0 " +
            "and ol.productId = p.id " +
            "order by o.number.number desc";

        TypedQuery<OrderView> query =
                em.createQuery(selectQuery, OrderView.class);
        query.setParameter("ordererId", ordererId);
        return query.getResultList();
    }
}
```

> [!tip] 무엇이 달라졌나
> 이 JPQL은 **`Order` 애그리거트 + `Member` 애그리거트 + `Product` 애그리거트를 조인으로 조회하여 한 번의 쿼리로 로딩**한다.
> **즉시 로딩이나 지연 로딩 같은 로딩 전략을 고민할 필요 없이**, 조회 화면에서 필요한 애그리거트 데이터를 한 번의 쿼리로 로딩할 수 있다.

> [!note] 다른 기술을 써도 된다
> 쿼리가 복잡하거나 **SQL에 특화된 기능**을 사용해야 한다면, 조회를 위한 부분만 **마이바티스(MyBatis)** 같은 기술로 구현할 수도 있다.

> [!warning] JPA 초심자가 빠지는 함정 (p.120)
> 처음 JPA를 사용하면 각 객체 간 모든 연관을 **지연 로딩과 즉시 로딩으로 어떻게든 처리하고 싶은 욕구**에 사로잡힌다.
> **하지만 이것은 실용적이지 않다.** ID를 이용해 애그리거트를 참조해도 **한 번의 쿼리로 필요한 데이터를 로딩하는 것이 가능하다.**

## 해법 ③ — 캐시 / 조회 전용 저장소

애그리거트마다 **서로 다른 저장소**를 사용하면 한 번의 쿼리로 관련 애그리거트를 조회할 수 없다.

| 기법 | 단점 | 장점 |
|------|------|------|
| **캐시** 적용 | 코드가 복잡해진다 | 시스템의 **처리량을 높일 수 있다** |
| **조회 전용 저장소** 구성 | 코드가 복잡해진다 | 〃 |

> [!important]
> 특히 **한 대의 DB 장비로 대응할 수 없는 수준의 트래픽이 발생하는 경우**,
> 캐시나 조회 전용 저장소는 **필수로 선택해야 하는 기법**이다.

```
   트래픽 규모
     작음  ──▶  조회 전용 쿼리(조인)로 충분
     큼    ──▶  캐시 / 조회 전용 저장소 (필수)
```

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "주문 10개 조회 시 쿼리 11번" | **N+1 조회 문제** |
| "N+1의 정확한 정의" | N개를 읽는 **1번** + 연관 데이터 **N번** |
| "ID 참조는 무엇과 같은 효과?" | **지연 로딩** |
| "N+1의 가장 쉬운 해법과 그 문제" | 객체 참조+즉시 로딩 → **ID 참조를 되돌리는 셈** |
| "권장 해법" | **조회 전용 쿼리** (별도 DAO, 조인 1회) |
| "SQL 특화 기능이 필요하면?" | 조회 부분만 **마이바티스** 등으로 구현 |
| "저장소가 서로 다르면?" | **캐시** 또는 **조회 전용 저장소** |
| "대규모 트래픽에서 캐시는?" | **필수 선택 기법** |

## Related Notes

- [[ID를 이용한 애그리거트 참조]] — 이 문제의 배경
- [[애그리거트 로딩 전략]] — 4장의 로딩 전략
- [[동적 인스턴스를 이용한 조회]] — 5장의 조회 전용 모델
- [[조회 전용 기능과 응용 서비스]] — 6장
- [[단일 모델의 단점]] — 11장 CQRS로 이어지는 문제의식
- [[CQRS]] — 명령/조회 모델 분리
- [[연습문제 - 애그리거트]]
