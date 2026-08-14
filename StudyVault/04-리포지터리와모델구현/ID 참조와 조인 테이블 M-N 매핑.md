---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 4장 §4.3.10 (p.160~161)
keywords: many-to-many, join-table, id-reference, element-collection
---

# ID 참조와 조인 테이블을 이용한 단방향 M-N 매핑 (★★)

#ddd #repository #jpa-mapping #aggregate-reference #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 원칙(3장) | 애그리거트 간 **집합 연관은 성능상의 이유로 피해야 한다** |
| 그럼에도 유리하다면 | **ID 참조를 이용한 단방향 집합 연관**을 적용해 볼 수 있다 |
| 구현 | **`@ElementCollection` + `@CollectionTable`** (조인 테이블) |
| 저장되는 것 | 객체가 아니라 **식별자(`CategoryId`) 컬렉션** |

## 전제 — 3장의 원칙

> [!warning]
> 3장 [[애그리거트 간 집합 연관]]에서 애그리거트 간 집합 연관은 **성능상의 이유로 피해야 한다**고 했다.
> **그럼에도 불구하고 요구사항을 구현하는 데 집합 연관을 사용하는 것이 유리하다면,
> ID 참조를 이용한 단방향 집합 연관을 적용해 볼 수 있다.**

## 구현

```java
@Entity
@Table(name = "product")
public class Product {
    @EmbeddedId
    private ProductId id;

    @ElementCollection
    @CollectionTable(name = "product_category",
                     joinColumns = @JoinColumn(name = "product_id"))
    private Set<CategoryId> categoryIds;
}
```

```
   ┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
   │ product      │     │ product_category     │     │ category     │
   │  product_id  │◀────┤  product_id  (FK)    │     │  id          │
   │  name        │     │  category_id         │╌╌╌╌▶│  name        │
   └──────────────┘     └──────────────────────┘     └──────────────┘
                          (조인 테이블)                  ╌╌ = 객체 참조 없음
                                                          (ID로만 연결)
```

> [!important] 핵심 차이
> `@ElementCollection`에 담기는 것이 **`Category` 객체가 아니라 `CategoryId` 식별자**다.
> → **애그리거트 간 물리적 연결이 없다.** 조인 테이블은 있지만 **객체 참조는 없다.**

| 매핑 | 컬렉션 원소 | 애그리거트 간 결합 |
|------|-------------|--------------------|
| `@ManyToMany Set<Category>` | **`Category` 객체** | 강하게 결합 ❌ |
| `@ElementCollection Set<CategoryId>` | **`CategoryId` 식별자(밸류)** | 결합 없음 ⭕ |

## 조회 — `member of`

특정 카테고리에 속한 `Product` 목록을 구할 때는 JPQL의 `member of` 연산자를 사용한다.

```java
@Repository
public class JpaProductRepository implements ProductRepository {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Product> findByCategoryId(CategoryId catId, int page, int size) {
        TypedQuery<Product> query = entityManager.createQuery(
            "select p from Product p " +
            "where :catId member of p.categoryIds order by p.id.id desc",
            Product.class);
        query.setParameter("catId", catId);
        query.setFirstResult((page - 1) * size);
        query.setMaxResults(size);
        return query.getResultList();
    }
}
```

> [!tip]
> `:catId member of p.categoryIds`는 **`categoryIds` 컬렉션에 지정한 값이 존재하는지 검사**하는 조건이다.

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "애그리거트 간 집합 연관의 원칙" | **성능상 피해야 한다** |
| "그래도 필요하면?" | **ID 참조 단방향** 집합 연관 |
| "M-N 구현 애너테이션" | **`@ElementCollection` + `@CollectionTable`** |
| "컬렉션에 담기는 것" | **`CategoryId` 식별자** (객체 아님) |
| "컬렉션 원소 존재 검사 JPQL" | **`member of`** |
| "`@ManyToMany`와의 차이" | 객체 참조 없음 → **애그리거트 간 결합 제거** |

## Related Notes

- [[애그리거트 간 집합 연관]] — 3장의 원칙과 배경
- [[ID를 이용한 애그리거트 참조]]
- [[밸류 컬렉션 매핑]] — `@ElementCollection` 상세
- [[엔티티 식별자와 밸류 타입]] — `CategoryId` 같은 식별자 타입
- [[연습문제 - 리포지터리와 모델 구현]]
