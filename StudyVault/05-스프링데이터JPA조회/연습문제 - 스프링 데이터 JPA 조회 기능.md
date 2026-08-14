---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 5장 (p.173~198)
keywords: practice, specification, paging, sort, subselect, dynamic-instance
---

# 스프링 데이터 JPA 조회 기능 Practice (14문제)

#practice #ddd #query

## Related Concepts

- [[검색을 위한 스펙]]
- [[스프링 데이터 JPA 스펙 구현]]
- [[스펙 조합]]
- [[정렬 지정하기]]
- [[페이징 처리하기]]
- [[스펙 빌더 클래스]]
- [[동적 인스턴스를 이용한 조회]]
- [[하이버네이트 Subselect 사용]]

> [!hint]- 핵심 패턴 (클릭하여 보기)
> | Keyword | Answer |
> |---------|--------|
> | 상태 변경 | 명령 모델 |
> | 목록·상세 조회 | 조회 모델 |
> | 조건 충족 검사 인터페이스 | 스펙 |
> | 스펙 핵심 메서드 | `toPredicate()` |
> | 람다 가능 이유 | 함수형 인터페이스 |
> | 문자열 대신 | 정적 메타 모델 `Xxx_` |
> | and/or | 디폴트 메서드 |
> | not/where | 정적 메서드 |
> | null 안전 | `Specification.where()` |
> | 페이지 번호 | **0부터** |
> | `Page` 리턴 | COUNT 실행 |
> | `findAll(Spec, Pageable)` | List여도 COUNT 실행 |
> | 임의 객체 생성 | JPQL `new` |
> | 쿼리를 엔티티로 | `@Subselect` |
> | stale 방지 | `@Synchronize` |

---

## Question 1 - 명령 모델과 조회 모델 [recall]
> 명령 모델과 조회 모델은 각각 어떤 기능에 사용되는가? 5장에서 다루는 정렬·페이징·검색 조건은 어느 쪽인가?

> [!answer]- 정답 보기
> - **명령 모델**: **상태(데이터)를 변경**하는 기능 (회원 가입, 암호 변경, 주문 취소, 배송지 변경)
> - **조회 모델**: **데이터를 보여주는** 기능 (주문 목록, 주문 상세)
> 정렬·페이징·검색 조건은 **조회 모델**을 구현할 때 주로 사용한다. 엔티티·애그리거트·리포지터리는 주로 **명령 모델**로 사용된다.

---

## Question 2 - 스펙이 필요한 이유 [recall]
> 검색 조건 조합마다 `find` 메서드를 정의하는 방식의 문제는 무엇이며, 대안은?

> [!answer]- 정답 보기
> **조합이 증가할수록 정의해야 할 `find` 메서드도 함께 증가**한다.
> 대안은 **스펙(Specification)** 이다. 스펙은 **애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스**로, `boolean isSatisfiedBy(T agg)`를 갖는다.

---

## Question 3 - `agg` 파라미터 [recall]
> `isSatisfiedBy(agg)`의 `agg`는 무엇인가? 사용 위치에 따라 어떻게 달라지는가?

> [!answer]- 정답 보기
> `agg`는 **검사 대상이 되는 객체**다.
> - 스펙을 **리포지터리**에 사용하면 → **애그리거트 루트**
> - 스펙을 **DAO**에 사용하면 → **검색 결과로 리턴할 데이터 객체**

---

## Question 4 - 스펙 인터페이스 [recall]
> 스프링 데이터 JPA의 `Specification<T>`에서 `T`는 무엇을 의미하며, 핵심 메서드와 그 리턴 타입은?

> [!answer]- 정답 보기
> `T`는 **JPA 엔티티 타입**을 의미한다.
> 핵심 메서드는 **`toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb)`** 이고, **JPA 크리테리아 API에서 조건을 표현하는 `Predicate`** 를 생성해 리턴한다.

---

## Question 5 - 정적 메타 모델 [recall]
> 정적 메타 모델 클래스의 애너테이션, 이름 규칙, 그리고 문자열 방식 대비 장점은?

> [!answer]- 정답 보기
> - 애너테이션: **`@StaticMetamodel(대상.class)`**
> - 이름 규칙: 모델 클래스 이름 뒤에 **`_`** 를 붙임 (`OrderSummary_`)
> - 장점: 문자열은 **오타 가능성이 있고 실행 전까지 놓치기 쉬우며 IDE 자동 완성을 쓸 수 없다.** 메타 모델은 **코드 안정성과 생산성** 측면에서 유리하다.

---

## Question 6 - 스펙 조합 메서드 [recall]
> `and`, `or`, `not`, `where` 중 디폴트 메서드와 정적 메서드를 구분하고, 각각의 역할을 답하라.

> [!answer]- 정답 보기
> - **디폴트 메서드**: `and()`(두 스펙 **모두 충족**), `or()`(**하나 이상 충족**)
> - **정적 메서드**: `not()`(조건을 **반대로**), `where()`(**`null`이면 아무 조건도 만들지 않고**, 아니면 그대로 리턴)

---

## Question 7 - 정렬 두 방법 [recall]
> 정렬을 지정하는 두 가지 방법과, 메서드 이름 방식의 단점 두 가지는?

> [!answer]- 정답 보기
> ① **메서드 이름에 `OrderBy` 사용** ② **`Sort` 타입을 인자로 전달**
> 메서드 이름 방식의 단점: ① 정렬 기준 프로퍼티가 두 개 이상이면 **메서드 이름이 길어진다** ② 메서드 이름으로 순서가 정해지므로 **상황에 따라 정렬 순서를 변경할 수 없다.**

---

## Question 8 - 페이지 번호 [recall]
> `PageRequest.of(1, 10)`은 몇 번째부터 몇 번째까지의 데이터를 조회하는가?

> [!answer]- 정답 보기
> **11번째부터 20번째까지**다. **페이지 번호는 0번부터 시작**하므로 `of(1, 10)`은 한 페이지에 10개씩일 때 **두 번째 페이지**를 의미한다.

---

## Question 9 - Subselect 3종 [recall]
> `@Subselect`, `@Immutable`, `@Synchronize`의 역할을 각각 설명하라.

> [!answer]- 정답 보기
> - **`@Subselect`**: **조회 쿼리를 값으로 갖고**, 하이버네이트가 그 결과를 **매핑할 테이블처럼 사용**한다(뷰와 유사).
> - **`@Immutable`**: 매핑 필드/프로퍼티가 **변경되어도 DB에 반영하지 않고 무시**한다(매핑 테이블이 없어 UPDATE 시 에러 발생 방지).
> - **`@Synchronize`**: 관련 **테이블 목록을 명시**하고, 엔티티 **로딩 전에 그 테이블에 변경이 있으면 먼저 플러시**한다.

---

## Question 10 - 스펙 구현 [application]
> `OrderSummary`의 `ordererId`가 지정한 값과 같은지 검사하는 스펙을, ① 클래스로 구현하는 방식과 ② 별도 클래스에 모으는 람다 방식 두 가지로 작성하라.

> [!answer]- 정답 보기
> **① 클래스 방식**
> ```java
> public class OrdererIdSpec implements Specification<OrderSummary> {
>     private String ordererId;
>     public OrdererIdSpec(String ordererId) { this.ordererId = ordererId; }
>
>     @Override
>     public Predicate toPredicate(Root<OrderSummary> root,
>                                  CriteriaQuery<?> query, CriteriaBuilder cb) {
>         return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
>     }
> }
> ```
> **② 람다 방식** (스펙 인터페이스는 **함수형 인터페이스**이므로 가능)
> ```java
> public class OrderSummarySpecs {
>     public static Specification<OrderSummary> ordererId(String ordererId) {
>         return (root, query, cb) -> cb.equal(root.get("ordererId"), ordererId);
>     }
> }
> ```

---

## Question 11 - null 안전 조합 [application]
> 아래 코드를 `where()`를 사용해 간단히 고쳐라.
> ```java
> Specification<OrderSummary> nullableSpec = createNullableSpec();  // null일 수 있음
> Specification<OrderSummary> otherSpec = createOtherSpec();
> Specification<OrderSummary> spec =
>         nullableSpec == null ? otherSpec : nullableSpec.and(otherSpec);
> ```

> [!answer]- 정답 보기
> ```java
> Specification<OrderSummary> spec =
>         Specification.where(createNullableSpec()).and(createOtherSpec());
> ```
> `where()`는 정적 메서드로 **`null`을 전달하면 아무 조건도 생성하지 않는 스펙을 리턴**하고, `null`이 아니면 인자를 그대로 리턴하므로 **NPE 검사가 불필요**해진다.

---

## Question 12 - 조회 전용 DTO [application]
> 주문 목록 화면에 주문번호·상태·회원명·상품명을 보여줘야 한다. 지연/즉시 로딩 고민 없이 한 번의 쿼리로 필요한 데이터만 조회하는 방법과 코드를 제시하라.

> [!answer]- 정답 보기
> **동적 인스턴스**를 사용한다. JPQL `select` 절의 **`new` 키워드** 뒤에 **완전한 클래스 이름**과 **생성자 인자**를 지정한다.
> ```java
> @Query("""
>     select new com.myshop.order.query.dto.OrderView(
>         o.number, o.state, m.name, m.id, p.name)
>     from Order o join o.orderLines ol, Member m, Product p
>     where o.orderer.memberId.id = :ordererId
>     and o.orderer.memberId.id = m.id
>     and index(ol) = 0
>     and ol.productId.id = p.id
>     order by o.number.number desc
>     """)
> List<OrderView> findOrderView(String ordererId);
> ```
> 장점: **JPQL을 그대로 사용해 객체 기준으로 쿼리를 작성**하면서 **로딩 전략 고민 없이** 원하는 형태로 조회한다. 표현 영역이 다루기 쉽도록 생성자에서 **기본 타입으로 변환**해두면 편리하다.

---

## Question 13 - COUNT 쿼리 최적화 [analysis]
> 페이징 처리 시 COUNT 쿼리가 실행되는 조건을 세 경우로 나누어 정리하고, 불필요한 COUNT 쿼리를 피하는 방법을 제시하라.

> [!answer]- 정답 보기
> | 메서드 형태 | 리턴 타입 | COUNT |
> |---|---|---|
> | `findByXxx(..., Pageable)` | `Page` | **실행** |
> | `findByXxx(..., Pageable)` | `List` | **실행 안 함** |
> | `findAll(Specification, Pageable)` | **`List`여도** | **실행함** ⚠️ |
>
> **정리**: 프로퍼티를 비교하는 `findBy` 형식은 `Pageable`을 써도 **리턴 타입이 `List`면 COUNT를 실행하지 않는다.** 반면 **스펙을 사용하는 `findAll`은 리턴 타입이 `Page`가 아니어도 COUNT를 실행한다.**
> **회피 방법**: ① 페이징 메타 정보가 필요 없으면 **`Page` 대신 `List` 리턴** ② 스펙을 쓰면서 COUNT를 피하려면 **스프링 데이터 JPA의 커스텀 리포지터리 기능**으로 직접 구현한다.

---

## Question 14 - Subselect의 트레이드오프 [analysis]
> `@Subselect`로 조회 전용 모델을 만들 때 얻는 것과 주의할 점을 종합하고, `@Synchronize`가 없으면 어떤 버그가 생기는지 시나리오로 설명하라.

> [!answer]- 정답 보기
> **얻는 것**
> - **쿼리 결과를 `@Entity`로 매핑**할 수 있다(뷰와 유사).
> - 일반 `@Entity`와 같으므로 **`find()`, JPQL, Criteria, 그리고 스펙·페이징**을 모두 사용할 수 있다. 이것이 가장 큰 장점이다.
>
> **주의할 점**
> - **수정할 수 없다.** 실수로 매핑 필드를 수정하면 하이버네이트가 `UPDATE`를 실행하는데 **매핑한 테이블이 없어 에러**가 난다 → **`@Immutable`** 로 무시하게 한다.
> - `@Subselect`의 쿼리가 **`from` 절의 서브 쿼리**로 들어간다. 서브 쿼리를 원치 않으면 **네이티브 SQL이나 마이바티스** 같은 별도 매퍼를 써야 한다.
>
> **`@Synchronize`가 없을 때의 버그 시나리오**
> ```java
> Order order = orderRepository.findById(orderNumber);
> order.changeShippingInfo(newInfo);   // 영속성 컨텍스트에만 반영
> List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
> ```
> 하이버네이트는 **트랜잭션 커밋 시점에 변경을 DB에 반영**하므로, `purchase_order` 테이블에 아직 반영되지 않은 상태에서 그 테이블을 읽는 `OrderSummary`를 조회하게 된다 → **최신 값이 아닌 이전 값(stale)이 담긴다.**
> `@Synchronize({"purchase_order", ...})`를 지정하면 하이버네이트가 **로딩 전에 해당 테이블 변경을 먼저 플러시**하므로 최신 값이 조회된다.

---

> [!summary]- 패턴 요약 (클릭하여 보기)
> | Keyword | Answer |
> |---------|--------|
> | 5장의 모델 | 조회 모델 |
> | 리포지터리 vs DAO | 이 장에서는 혼용 |
> | 조건 조합 폭발 | 스펙으로 해결 |
> | 메모리 필터링 | 실제로는 안 씀 |
> | 스펙 `T` | JPA 엔티티 타입 |
> | 스펙 메서드 | `toPredicate` → `Predicate` |
> | 메타 모델 | `@StaticMetamodel`, `Xxx_` |
> | and/or | 디폴트 |
> | not/where | 정적 |
> | 정렬 | `OrderBy` / `Sort` |
> | `Sort` 결합 | `.and()` |
> | 페이징 | `Pageable` / `PageRequest` |
> | 페이지 번호 시작 | 0 |
> | COUNT 실행 | `Page` 리턴, 또는 스펙 `findAll` |
> | 처음 N개 | `findFirst3By` / `findTop3By` |
> | 빌더 메서드 | `and`, `ifTrue`, `ifHasText` |
> | 동적 인스턴스 | JPQL `new` + 완전한 클래스명 |
> | 쿼리 → 엔티티 | `@Subselect` |
> | 수정 무시 | `@Immutable` |
> | 플러시 유발 | `@Synchronize` |
> | 실행 쿼리 형태 | `from` 절 서브 쿼리 |
