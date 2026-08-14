---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 4장 §4.2 (p.136~138)
keywords: spring-data-jpa, repository-interface, naming-convention, crud
---

# 스프링 데이터 JPA 리포지터리 (★★★)

#ddd #repository #spring #jpa #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 핵심 | **인터페이스만 정의하면** 구현 객체를 자동 생성해 **스프링 빈으로 등록** |
| 상속 대상 | `org.springframework.data.repository.Repository<T, ID>` |
| 타입 파라미터 | `T` = **엔티티 타입**, `ID` = **식별자 타입** |
| 저장 | `save(T)` / `saveAll(...)` |
| ID 조회 | `findById(ID)` — 없으면 `null` 또는 `Optional.empty()` |
| 프로퍼티 조회 | **`findBy` + 프로퍼티이름** (중첩 프로퍼티 가능) |
| 삭제 | `delete(T)` / `deleteById(ID)` |

> [!important] 실무의 기본 선택
> 필자를 포함한 다수의 개발자는 **스프링과 JPA로 구현할 때 스프링 데이터 JPA를 사용**한다.
> 리포지터리 인터페이스만 정의하면 나머지 **구현 객체는 스프링 데이터 JPA가 알아서 만들어준다.**
> 그래서 실질적으로 **리포지터리 인터페이스를 구현한 클래스를 직접 작성할 일은 거의 없다.**

## 등록 규칙

스프링 데이터 JPA는 다음 규칙에 따라 작성한 인터페이스를 찾아서 **구현한 스프링 빈 객체를 자동으로 등록**한다.

- `org.springframework.data.repository.Repository<T, ID>` 인터페이스 **상속**
- **`T`는 엔티티 타입**을 지정하고 **`ID`는 식별자 타입**을 지정

### 엔티티 예

```java
@Entity
@Table(name = "purchase_order")
@Access(AccessType.FIELD)
public class Order {
    @EmbeddedId
    private OrderNo number;      // OrderNo가 식별자 타입
    ...
}
```

### 리포지터리 인터페이스 (리스트 4.4)

```java
import org.springframework.data.repository.Repository;

import java.util.Optional;

public interface OrderRepository extends Repository<Order, OrderNo> {
    Optional<Order> findById(OrderNo id);

    void save(Order order);
}
```

### 사용

```java
@Service
public class CancelOrderService {

    private OrderRepository orderRepository;

    public CancelOrderService(OrderRepository orderRepository, ...생략) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public void cancel(OrderNo orderNo, Canceller canceller) {
        Order order = orderRepository.findById(orderNo)
                .orElseThrow(() -> new NoOrderException());
        if (!cancelPolicy.hasCancellationPermission(order, canceller)) {
            throw new NoCancellablePermission();
        }
        order.cancel();
    }
}
```

## 메서드 작성 규칙

### ① 저장

| 메서드 | 설명 |
|--------|------|
| `Order save(Order entity)` | 저장 후 엔티티 반환 |
| `void save(Order entity)` | 저장만 |

### ② 식별자로 조회

| 메서드 | 엔티티가 없을 때 |
|--------|------------------|
| `Order findById(OrderNo id)` | **`null`** 리턴 |
| `Optional<Order> findById(OrderNo id)` | **값이 없는 `Optional`** 리턴 |

### ③ 프로퍼티로 조회

**`findBy` + 프로퍼티이름(프로퍼티 값)** 형식을 사용한다.

```java
List<Order> findByOrderer(Orderer orderer);
```

**중첩 프로퍼티도 가능하다.**

```java
// Orderer 객체의 memberId 프로퍼티가 파라미터와 같은 Order 목록을 조회
List<Order> findByOrdererMemberId(MemberId memberId);
```

```
   findBy  +  Orderer  +  MemberId
             └─────┬────────┬────┘
              중첩 프로퍼티 경로: orderer.memberId
```

### ④ 삭제

| 메서드 | 설명 |
|--------|------|
| `void delete(Order order)` | **삭제할 엔티티를 전달** |
| `void deleteById(OrderNo id)` | **식별자로** 해당 엔티티를 삭제 |

> [!note]
> 조회를 위한 다양한 기능이 더 있으며, 이에 대한 내용은 5장 [[검색을 위한 스펙]] 이후에서 다룬다.

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "스프링 데이터 JPA의 핵심 이점" | **인터페이스만 정의**하면 구현체 자동 생성·빈 등록 |
| "상속해야 할 인터페이스" | **`Repository<T, ID>`** |
| "`T`와 `ID`는?" | **엔티티 타입 / 식별자 타입** |
| "`findById`가 없을 때 리턴" | **`null`** 또는 **빈 `Optional`** |
| "`findByOrdererMemberId`" | **중첩 프로퍼티** 조회 (`orderer.memberId`) |
| "삭제 메서드 두 형태" | **`delete(엔티티)` / `deleteById(식별자)`** |
| "리포지터리 구현 클래스를 직접 작성하나?" | **거의 없다** |

## Related Notes

- [[JPA 리포지터리 구현]] — 직접 구현하는 방식
- [[리포지터리 개요]] — 2장 개념
- [[검색을 위한 스펙]] — 5장의 조회 확장
- [[스프링 데이터 JPA 스펙 구현]]
- [[도메인 구현과 DIP]] — `Repository` 상속이 DIP를 어기는 지점
- [[연습문제 - 리포지터리와 모델 구현]]
