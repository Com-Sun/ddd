---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 5장 §5.3~5.4 (p.178~182)
keywords: specification, criteria-api, static-metamodel, functional-interface
---

# 스프링 데이터 JPA 스펙 구현 (★★★)

#ddd #query #specification #spring #jpa #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 인터페이스 | `org.springframework.data.jpa.domain.Specification<T>` |
| 지네릭 `T` | **JPA 엔티티 타입** |
| 핵심 메서드 | `toPredicate(root, query, cb)` — JPA **크리테리아 API에서 조건을 표현하는 `Predicate` 생성** |
| **함수형 인터페이스** | ⭕ → **람다식**으로 생성 가능 |
| 권장 패턴 | 스펙 생성 기능을 **별도 클래스에 모은다** (`XxxSpecs`) |
| 정적 메타 모델 | `@StaticMetamodel` — 문자열 대신 **타입 안전한 프로퍼티 참조** |
| 사용 | 리포지터리/DAO의 **`findAll(Specification)`** |

## Specification 인터페이스 (리스트 5.1)

```java
package org.springframework.data.jpa.domain;

import java.io.Serializable;
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;
import org.springframework.lang.Nullable;

public interface Specification<T> extends Serializable {
    // not, where, and, or 메서드 생략

    @Nullable
    Predicate toPredicate(Root<T> root,
                          CriteriaQuery<?> query,
                          CriteriaBuilder cb);
}
```

> [!important]
> 지네릭 타입 파라미터 **`T`는 JPA 엔티티 타입**을 의미한다.
> `toPredicate()` 메서드는 **JPA 크리테리아 API에서 조건을 표현하는 `Predicate`를 생성**한다.

## 구현 예 (리스트 5.2)

> 조건: **① 엔티티 타입이 `OrderSummary`이고 ② `ordererId` 프로퍼티 값이 지정한 값과 동일하다.**

```java
package com.myshop.order.query.dao;

import com.myshop.order.query.dto.OrderSummary;
import com.myshop.order.query.dto.OrderSummary_;
import org.springframework.data.jpa.domain.Specification;

import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

public class OrdererIdSpec implements Specification<OrderSummary> {

    private String ordererId;

    public OrdererIdSpec(String ordererId) {
        this.ordererId = ordererId;
    }

    @Override
    public Predicate toPredicate(Root<OrderSummary> root,
                                 CriteriaQuery<?> query,
                                 CriteriaBuilder cb) {
        return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
    }
}
```

- `OrdererIdSpec`은 `Specification<OrderSummary>` 타입을 구현하므로 **`OrderSummary`에 대한 검색 조건**을 표현한다.
- `toPredicate()`는 **`ordererId` 프로퍼티 값이 생성자로 전달받은 값과 동일한지 비교하는 `Predicate`를 생성**한다.

## JPA 정적 메타 모델

```java
@StaticMetamodel(OrderSummary.class)
public class OrderSummary_ {
    public static volatile SingularAttribute<OrderSummary, String> number;
    public static volatile SingularAttribute<OrderSummary, Long> version;
    public static volatile SingularAttribute<OrderSummary, String> ordererId;
    public static volatile SingularAttribute<OrderSummary, String> ordererName;
    ... 생략
}
```

| 규칙 | 내용 |
|------|------|
| 애너테이션 | **`@StaticMetamodel`** 로 관련 모델을 지정 |
| 클래스 이름 | 모델 클래스 이름 뒤에 **`_`** 를 붙인 이름 |
| 필드 | 대상 모델의 **각 프로퍼티와 동일한 이름을 갖는 정적 필드** |
| 필드 타입 | 프로퍼티 타입에 따라 **`SingularAttribute`, `ListAttribute` 등** |

### 왜 문자열 대신 메타 모델인가

```java
// 문자열 방식도 가능하다
cb.equal(root.<String>get("ordererId"), ordererId)
```

> [!warning] 문자열 방식의 단점
> - **오타 가능성**이 있고 **실행하기 전까지는 오타를 놓치기 쉽다.**
> - **IDE의 코드 자동 완성 기능을 사용할 수 없어** 입력할 코드도 많아진다.
>
> 이런 이유로 **크리테리아를 사용할 때에는 정적 메타 모델 클래스를 사용하는 것이 코드 안정성이나 생산성 측면에서 유리하다.**

> [!tip]
> 정적 메타 모델 클래스를 직접 작성할 수 있지만, **하이버네이트 같은 JPA 프로바이더는 정적 메타 모델을 생성하는 도구**를 제공하므로 이를 사용하면 편리하다.

## ★ 스펙 생성 기능을 별도 클래스에 모으기 (리스트 5.3)

> 스펙 구현 클래스를 개별적으로 만들지 않고 **별도 클래스에 스펙 생성 기능을 모아**둘 수 있다.

```java
package com.myshop.order.query.dao;

import com.myshop.order.query.dto.OrderSummary;
import com.myshop.order.query.dto.OrderSummary_;
import org.springframework.data.jpa.domain.Specification;

import java.time.LocalDateTime;

public class OrderSummarySpecs {

    public static Specification<OrderSummary> ordererId(String ordererId) {
        return (root, query, cb) ->
                cb.equal(root.<String>get("ordererId"), ordererId);
    }

    public static Specification<OrderSummary> orderDateBetween(
            LocalDateTime from, LocalDateTime to) {
        return (root, query, cb) ->
                cb.between(root.get(OrderSummary_.orderDate), from, to);
    }
}
```

> [!important] 람다식이 가능한 이유
> **스펙 인터페이스는 함수형 인터페이스**이므로 **람다식을 이용해서 객체를 생성**할 수 있다.

```java
Specification<OrderSummary> betweenSpec =
        OrderSummarySpecs.orderDateBetween(from, to);
```

## §5.4 리포지터리/DAO에서 스펙 사용하기

> 스펙을 충족하는 엔티티를 검색하고 싶다면 **`findAll()` 메서드**를 사용하면 된다.
> `findAll()` 메서드는 **스펙 인터페이스를 파라미터로 갖는다.**

```java
public interface OrderSummaryDao
        extends Repository<OrderSummary, String> {
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```

```java
// 스펙 객체를 생성하고
Specification<OrderSummary> spec = new OrdererIdSpec("user1");
// findAll() 메서드를 이용해서 검색
List<OrderSummary> results = orderSummaryDao.findAll(spec);
```

> [!note] 버전 차이 주의
> 스프링 데이터 JPA는 **버전에 따라 사용법이 조금씩 다르다.**
> 읽는 시점에 따라 일부 코드는 다르게 동작할 수 있으니 **레퍼런스 문서를 함께 보면서** 학습할 것을 권한다.

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "스펙 인터페이스 패키지" | `org.springframework.data.jpa.domain` |
| "지네릭 `T`의 의미" | **JPA 엔티티 타입** |
| "핵심 메서드와 리턴 타입" | **`toPredicate()` → `Predicate`** |
| "람다로 만들 수 있는 이유" | **함수형 인터페이스**이기 때문 |
| "정적 메타 모델 애너테이션/이름 규칙" | **`@StaticMetamodel`** / 클래스명 + **`_`** |
| "문자열 프로퍼티 지정의 단점" | **오타·자동완성 불가** |
| "스펙으로 검색하는 메서드" | **`findAll(Specification)`** |
| "스펙 생성 기능 권장 배치" | **별도 클래스(`XxxSpecs`)에 정적 메서드로 모음** |

## Related Notes

- [[검색을 위한 스펙]] — 스펙의 개념
- [[스펙 조합]] — `and`/`or`/`not`/`where`
- [[스펙 빌더 클래스]]
- [[정렬 지정하기]]
- [[페이징 처리하기]]
- [[스프링 데이터 JPA 리포지터리]] — 4장
- [[연습문제 - 스프링 데이터 JPA 조회 기능]]
