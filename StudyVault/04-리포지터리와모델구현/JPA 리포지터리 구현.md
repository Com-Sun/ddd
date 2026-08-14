---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 4장 §4.1~4.1.2 (p.130~135)
keywords: jpa, repository, entitymanager, module-location, soft-delete
---

# JPA 리포지터리 구현 (★★★)

#ddd #repository #jpa #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 인터페이스 위치 | **도메인 영역** (애그리거트와 같이) |
| 구현 클래스 위치 | **인프라스트럭처 영역** |
| 인터페이스 작성 기준 | **애그리거트 루트** |
| 기본 기능 2가지 | **ID로 애그리거트 조회 / 애그리거트 저장** |
| 조회 메서드 이름 규칙 | **`findBy` + 프로퍼티이름(프로퍼티 값)** |
| 수정 반영 | **별도 메서드 불필요** — JPA가 트랜잭션 범위에서 자동 반영 |
| 삭제 | 실무에서는 **삭제 플래그(soft delete)** 를 쓰는 경우가 많다 |

## §4.1.1 모듈 위치

```
   com.myshop.order.domain              com.myshop.order.infra
   ┌──────────────────────────┐         ┌──────────────────────────┐
   │  Order (애그리거트 루트)  │         │                          │
   │  «interface»             │◀╌╌╌╌╌╌╌╌│  JpaOrderRepository      │
   │   OrderRepository        │  구현    │                          │
   └──────────────────────────┘         └──────────────────────────┘
```

> [!important]
> 리포지터리 **인터페이스는 애그리거트와 같이 도메인 영역**에 속하고,
> 리포지터리를 **구현한 클래스는 인프라스트럭처 영역**에 속한다.

> [!warning] 타협안에 주의
> 팀 표준에 따라 리포지터리 구현 클래스를 `com.myshop.order.domain.impl` 같은 패키지에 위치시킬 수도 있는데,
> 이것은 **인터페이스와 구현체를 분리하기 위한 타협안 같은 것이지 좋은 설계 원칙을 따르는 것은 아니다.**
> 가능하면 **구현 클래스를 인프라스트럭처 영역에 위치**시켜서 인프라스트럭처에 대한 의존을 낮춰야 한다.

## §4.1.2 리포지터리 기본 기능

리포지터리가 제공하는 기본 기능은 두 가지다.

- **ID로 애그리거트 조회하기**
- **애그리거트 저장하기**

```java
public interface OrderRepository {
    Order findById(OrderNo no);
    void save(Order order);
}
```

> [!important] 인터페이스는 애그리거트 루트를 기준으로 작성한다
> 주문 애그리거트는 `Order` 루트 엔티티를 비롯해 `OrderLine`, `Orderer`, `ShippingInfo` 등 다양한 객체를 포함하는데,
> 이 구성요소 중에서 **루트 엔티티인 `Order`를 기준으로** 리포지터리 인터페이스를 작성한다.

### 메서드 이름 규칙

> 애그리거트를 조회하는 기능의 이름에 특별한 규칙은 없지만, 널리 사용되는 규칙은 **`findBy` + 프로퍼티이름(프로퍼티 값)** 형식이다.

```java
Optional<Order> findById(OrderNo no);       // null 대신 Optional 사용 가능
```

`findById()`는 ID에 해당하는 애그리거트가 존재하면 `Order`를, 존재하지 않으면 **`null`을 리턴**한다. `null`을 사용하고 싶지 않다면 **`Optional`** 을 사용해도 된다.

### 구현 (리스트 4.1)

```java
package com.myshop.order.infra;

import com.myshop.order.domain.Order;
import com.myshop.order.domain.OrderNo;
import com.myshop.order.domain.OrderRepository;

import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;

@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public Order findById(OrderNo id) {
        return entityManager.find(Order.class, id);      // find로 ID 조회
    }

    @Override
    public void save(Order order) {
        entityManager.persist(order);                    // persist로 저장
    }
}
```

| 코드 | 설명 |
|------|------|
| `entityManager.find()` | `EntityManager`의 `find` 메서드로 **ID로 애그리거트를 검색** |
| `entityManager.persist()` | `EntityManager`의 `persist` 메서드로 **애그리거트를 저장** |

## ★ 수정 결과 저장 메서드는 필요 없다

> [!important]
> **애그리거트를 수정한 결과를 저장소에 반영하는 메서드를 추가할 필요는 없다.**
> JPA를 사용하면 **트랜잭션 범위에서 변경한 데이터를 자동으로 DB에 반영**하기 때문이다.

```java
public class ChangeOrderService {

    @Transactional
    public void changeShippingInfo(OrderNo no, ShippingInfo newShippingInfo) {
        Optional<Order> orderOpt = orderRepository.findById(no);
        Order order = orderOpt.orElseThrow(() -> new OrderNotFoundException());
        order.changeShippingInfo(newShippingInfo);
        // 별도 save() 호출 불필요 — 커밋 시점에 UPDATE 쿼리 실행
    }
}
```

```
   @Transactional 메서드 실행
        ▼
   order.changeShippingInfo(...)   ← 애그리거트 상태 변경
        ▼
   메서드 종료 → 트랜잭션 커밋
        ▼
   JPA가 변경된 객체를 감지하여 UPDATE 쿼리 실행  (더티 체킹)
```

## ID 외의 조건으로 조회

ID가 아닌 다른 조건으로 조회할 때는 `findBy` 뒤에 **조건 대상이 되는 프로퍼티 이름**을 붙인다.

```java
public interface OrderRepository {
    List<Order> findByOrdererId(String ordererId, int startRow, int size);
}
```

> 한 개 이상의 `Order` 객체를 리턴할 수 있으므로 컬렉션 타입 중 하나인 **`List`를 리턴 타입**으로 사용했다.

### JPQL 구현 (리스트 4.2)

```java
@Override
public List<Order> findByOrdererId(String ordererId, int startRow, int fetchSize) {
    TypedQuery<Order> query = entityManager.createQuery(
            "select o from Order o " +
            "where o.orderer.memberId.id = :ordererId " +
            "order by o.number.number desc",
            Order.class);
    query.setParameter("ordererId", ordererId);
    query.setFirstResult(startRow);
    query.setMaxResults(fetchSize);
    return query.getResultList();
}
```

## 삭제 기능

```java
public interface OrderRepository {
    public void delete(Order order);
}
```

```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public void delete(Order order) {
        entityManager.remove(order);
    }
}
```

> [!warning] ★ 실무에서는 실제로 삭제하지 않는다 (p.135)
> 삭제 요구사항이 있더라도 **데이터를 실제로 삭제하는 경우는 많지 않다.**
> - 관리자 기능에서 **삭제한 데이터까지 조회**해야 하는 경우가 있다.
> - **데이터 원복**을 위해 일정 기간 동안 보관해야 할 때도 있다.
>
> 이런 이유로 사용자가 삭제 기능을 실행할 때 데이터를 바로 삭제하기보다는
> **삭제 플래그를 사용해서 데이터를 화면에 보여줄지 여부를 결정하는 방식(soft delete)** 으로 구현한다.

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "리포지터리 인터페이스/구현체 위치" | **도메인 / 인프라스트럭처** |
| "구현체를 `domain.impl`에 두면?" | **타협안**일 뿐, 좋은 원칙 아님 |
| "인터페이스 작성 기준" | **애그리거트 루트** |
| "조회 메서드 이름 규칙" | **`findBy` + 프로퍼티이름** |
| "ID로 조회 / 저장 EntityManager 메서드" | **`find()` / `persist()`** |
| "수정 결과를 반영하는 메서드가 필요한가?" | **불필요.** 트랜잭션 커밋 시 자동 반영 |
| "삭제는 실제로 하는가?" | 실무는 대부분 **삭제 플래그** |
| "삭제 EntityManager 메서드" | **`remove()`** |

## Related Notes

- [[리포지터리 개요]] — 2장의 개념
- [[리포지터리와 애그리거트]] — 3장의 원칙
- [[스프링 데이터 JPA 리포지터리]] — 실무에서 주로 쓰는 방식
- [[엔티티와 밸류 기본 매핑]]
- [[모듈 구성]] — 2장의 패키지 구성
- [[도메인 구현과 DIP]] — 이 구현이 DIP를 어기는 지점
- [[연습문제 - 리포지터리와 모델 구현]]
