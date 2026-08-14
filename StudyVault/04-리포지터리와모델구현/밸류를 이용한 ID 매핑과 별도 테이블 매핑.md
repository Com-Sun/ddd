---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 4장 §4.3.7~4.3.8 (p.151~155)
keywords: embeddedid, serializable, secondary-table, value-vs-entity
---

# 밸류를 이용한 ID 매핑과 별도 테이블 매핑 (★★★)

#ddd #repository #jpa-mapping #value-type #concept

## Overview Table (한눈에 비교)

| 주제 | 핵심 |
|------|------|
| **밸류 ID 매핑** (§4.3.7) | **`@EmbeddedId`** + 식별자 클래스는 **`Serializable` 구현** |
| 필수 구현 | JPA가 내부적으로 엔티티 비교에 사용하므로 **`equals()`/`hashCode()`** 를 알맞게 구현 |
| 이점 | 식별자에 **기능(도메인 로직) 추가** 가능 |
| **별도 테이블 밸류** (§4.3.8) | **`@SecondaryTable`** + **`@AttributeOverride`** |
| ★ 핵심 판단 | **별도 테이블에 저장한다고 해서 엔티티인 것은 아니다** |
| `@SecondaryTable` 주의 | 조회 시 **항상 조인**해서 읽어온다 → 목록 화면에서 불필요 |

## §4.3.7 밸류를 이용한 ID 매핑

```java
@Entity
@Table(name = "purchase_order")
public class Order {
    @EmbeddedId
    private OrderNo number;
}

@Embeddable
public class OrderNo implements Serializable {
    @Column(name = "order_number")
    private String number;

    public boolean is2ndGeneration() {
        return number.startsWith("N");
    }
}
```

> [!important] 두 가지 필수 조건
> ① 식별자 밸류 클래스는 **`Serializable` 인터페이스를 구현**해야 한다.
> ② JPA는 **내부적으로 엔티티를 비교할 목적으로 `equals()`와 `hashCode()`를 사용**하므로,
> 식별자로 사용할 밸류 타입은 **이 두 메서드를 알맞게 구현**해야 한다.

### 이점 — 식별자에 도메인 로직을 담을 수 있다

시스템 세대 구분이 필요한 코드는 `OrderNo`가 제공하는 기능으로 구분하면 된다.

```java
if (order.getNumber().is2ndGeneration()) {
    ...
}
```

> [!tip]
> 식별자를 단순 `String`으로 두면 이런 판단 로직이 **여기저기 흩어진다.**
> 밸류 식별자를 쓰면 **판단 로직이 식별자 타입 안에 응집**된다. (→ [[엔티티 식별자와 밸류 타입]])

## §4.3.8 별도 테이블에 저장하는 밸류 매핑

### ★ 밸류인가 엔티티인가 — 판단 순서

> [!important] 판단 절차 (p.152~153)
> **① 애그리거트에서 루트 엔티티를 뺀 나머지 구성요소는 대부분 밸류다.**
> 루트 엔티티 외에 또 다른 엔티티가 있다면 **진짜 엔티티인지 의심**해 봐야 한다.
>
> **② 단지 별도 테이블에 데이터를 저장한다고 해서 엔티티인 것은 아니다.**
> 주문 애그리거트도 `OrderLine`을 별도 테이블에 저장하지만 **`OrderLine` 자체는 엔티티가 아니라 밸류**다.
>
> **③ 밸류가 아니라 엔티티가 확실하다면, 그 엔티티가 다른 애그리거트는 아닌지 확인**해야 한다.
> 특히 **자신만의 독자적인 라이프 사이클**을 갖는다면 **구분되는 애그리거트일 가능성이 높다.**

```
   구성요소 발견
        ▼
   고유 식별자를 갖는가? ──아니오──▶ 밸류
        │ 예
        ▼
   독자적인 라이프 사이클을 갖는가? ──예──▶ 별도 애그리거트
        │ 아니오
        ▼
   같은 애그리거트에 속한 엔티티
```

**Product / Review 예**: `Review`는 엔티티가 맞지만 **리뷰 애그리거트에 속한 엔티티이지 상품 애그리거트에 속한 엔티티는 아니다.** (함께 생성/변경되지 않고 변경 주체도 다르다. → [[애그리거트]])

> [!danger] 흔한 착각
> 애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 **고유 식별자를 갖는지** 확인하는 것이다.
> **하지만 매핑되는 테이블의 식별자를 구성요소의 식별자와 동일한 것으로 착각하면 안 된다.**
> **별도 테이블로 저장하고 테이블에 PK가 있다고 해서, 그 테이블과 매핑되는 애그리거트 구성요소가 항상 고유 식별자를 갖는 것은 아니다.**

### 잘못된 매핑 (그림 4.5) vs 올바른 매핑 (그림 4.6)

게시글 데이터를 `ARTICLE` 테이블과 `ARTICLE_CONTENT` 테이블로 나눠 저장한다고 하자.

| 접근 | 모델 | 판정 |
|------|------|------|
| ❌ | `Article` ──1:1──▶ `ArticleContent` **(엔티티)** | `ARTICLE_CONTENT`의 `ID`가 식별자처럼 보여서 엔티티로 오해 |
| ⭕ | `Article` ──▶ `ArticleContent` **(밸류)** | `ID`는 **`ARTICLE` 테이블과 연결하기 위한 것**이지 `ArticleContent`를 위한 별도 식별자가 필요해서가 아니다 |

> [!important]
> 이것은 **게시글의 특정 프로퍼티를 별도 테이블에 보관한 것**으로 접근해야 한다.

### 구현 — `@SecondaryTable` (리스트 4.6)

```java
@Entity
@Table(name = "article")
@SecondaryTable(
    name = "article_content",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class Article {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @AttributeOverrides({
        @AttributeOverride(
            name = "content",
            column = @Column(table = "article_content", name = "content")),
        @AttributeOverride(
            name = "contentType",
            column = @Column(table = "article_content", name = "content_type"))
    })
    @Embedded
    private ArticleContent content;
}
```

| 속성 | 역할 |
|------|------|
| `@SecondaryTable`의 **`name`** | 밸류를 저장할 **테이블**을 지정 |
| `@SecondaryTable`의 **`pkJoinColumns`** | 밸류 테이블에서 엔티티 테이블로 **조인할 때 사용할 칼럼**을 지정 |
| `@AttributeOverride`의 **`column`의 `table`** | 해당 밸류 데이터가 저장된 **테이블 이름**을 지정 |

```java
// @SecondaryTable로 매핑된 article_content 테이블을 조인
Article article = entityManager.find(Article.class, 1L);
```

## ★ `@SecondaryTable`의 한계

> [!danger] 목록 화면에서 불필요한 조인
> 게시글 **목록**을 보여주는 화면은 `article` 테이블의 데이터만 필요하지 `article_content` 테이블의 데이터는 필요하지 않다.
> 그런데 `@SecondaryTable`을 사용하면 목록 화면에 보여줄 `Article`을 조회할 때
> **`article_content` 테이블까지 조인해서 데이터를 읽어오는데 이것은 원하는 결과가 아니다.**

**두 가지 대응**

| 방법 | 평가 |
|------|------|
| `ArticleContent`를 **엔티티로 매핑**하고 **지연 로딩**으로 설정 | ❌ **밸류인 모델을 엔티티로 만드는 것이므로 좋은 방법이 아니다** |
| **조회 전용 기능을 구현** | ⭕ **권장** (5장 조회 전용 쿼리, 11장 CQRS) |

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "밸류를 식별자로 매핑" | **`@EmbeddedId`** |
| "식별자 밸류 클래스의 필수 인터페이스" | **`Serializable`** |
| "식별자 밸류가 구현해야 할 두 메서드" | **`equals()` / `hashCode()`** (JPA가 엔티티 비교에 사용) |
| "별도 테이블에 저장하면 엔티티인가?" | **아니다.** `OrderLine`도 별도 테이블이지만 밸류 |
| "테이블에 PK가 있으면 엔티티인가?" | **아니다.** 연결용 PK일 수 있다 |
| "독자적 라이프 사이클을 가지면?" | **별도 애그리거트**일 가능성 높음 |
| "별도 테이블 밸류 매핑 애너테이션" | **`@SecondaryTable`** (+ `pkJoinColumns`) |
| "`@SecondaryTable`의 문제" | 목록 조회 시에도 **항상 조인** |
| "그 해법" | **조회 전용 기능 구현** (엔티티로 바꾸지 말 것) |

## Related Notes

- [[엔티티 식별자와 밸류 타입]] — 1장의 배경
- [[엔티티와 밸류 기본 매핑]]
- [[밸류 컬렉션 매핑]]
- [[애그리거트]] — 밸류/엔티티/애그리거트 구분
- [[동적 인스턴스를 이용한 조회]] — 5장의 조회 전용 기능
- [[CQRS]] — 11장
- [[연습문제 - 리포지터리와 모델 구현]]
