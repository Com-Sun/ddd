---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 4장 §4.3.4 (p.145~147)
keywords: attribute-converter, single-column, autoapply, money
---

# AttributeConverter를 이용한 밸류 매핑 (★★)

#ddd #repository #jpa-mapping #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 문제 | **두 개 이상의 프로퍼티를 가진 밸류를 한 개 칼럼**에 매핑해야 할 때 |
| `@Embeddable`로 가능? | **불가능** |
| 해법 | **`AttributeConverter<X, Y>`** 인터페이스 구현 |
| 메서드 2개 | `convertToDatabaseColumn` (밸류→칼럼) / `convertToEntityAttribute` (칼럼→밸류) |
| 적용 | 구현 클래스에 **`@Converter`** 애너테이션 |
| `autoApply = true` | 모델의 **모든 해당 타입 프로퍼티에 자동 적용** |
| `autoApply = false` (기본값) | 프로퍼티마다 **`@Convert(converter = ...)`** 로 직접 지정 |

## 문제 — 한 개 칼럼에 매핑해야 할 때

`int`, `long`, `String`, `LocalDate` 같은 타입은 DB 테이블의 **한 개 칼럼에 매핑**된다.
비슷하게 **밸류 타입의 프로퍼티를 한 개 칼럼에 매핑해야 할 때**도 있다.

```
   [ 도메인 모델 ]                    [ DB 테이블 ]
   class Length {
       private int value;      ┐
       private String unit;    ┘  ──▶  WIDTH VARCHAR(20)
   }                                    "1000mm" 형식으로 저장
```

> [!danger]
> **두 개 이상의 프로퍼티를 가진 밸류 타입을 한 개 칼럼에 매핑하려면 `@Embeddable` 애너테이션으로는 처리할 수 없다.**
> 이럴 때 사용할 수 있는 것이 **`AttributeConverter`** 다.

## AttributeConverter 인터페이스

```java
package javax.persistence;

public interface AttributeConverter<X, Y> {
    public Y convertToDatabaseColumn(X attribute);
    public X convertToEntityAttribute(Y dbData);
}
```

| 메서드 | 역할 |
|--------|------|
| `convertToDatabaseColumn` | **밸류 타입 → DB 칼럼 값**으로 변환 |
| `convertToEntityAttribute` | **DB 칼럼 값 → 밸류**로 변환 |

## 구현 예 — Money (리스트 4.5)

```java
package com.myshop.common.jpa;

import com.myshop.common.model.Money;

import javax.persistence.AttributeConverter;
import javax.persistence.Converter;

@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {

    @Override
    public Integer convertToDatabaseColumn(Money money) {
        return money == null ? null : money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value) {
        return value == null ? null : new Money(value);
    }
}
```

## `autoApply` 속성

### `autoApply = true` — 자동 적용

> [!tip]
> 이 속성을 `true`로 지정하면 **모델에 출현하는 모든 `Money` 타입의 프로퍼티에 대해 `MoneyConverter`를 자동으로 적용**한다.

```java
@Entity
@Table(name = "purchase_order")
public class Order {

    @Column(name = "total_amounts")
    private Money totalAmounts;      // MoneyConverter를 자동 적용
}
```

### `autoApply = false` (기본값) — 직접 지정

> `@Converter`의 `autoApply` 속성을 `false`로 지정하면(이 속성의 **기본값이 `false`**),
> 프로퍼티 값을 변환할 때 사용할 **컨버터를 직접 지정**해야 한다.

```java
import javax.persistence.Convert;

public class Order {

    @Column(name = "total_amounts")
    @Convert(converter = MoneyConverter.class)
    private Money totalAmounts;
}
```

```
   @Converter(autoApply = true)   →  해당 타입 전부 자동 적용
   @Converter(autoApply = false)  →  @Convert(converter = X.class)로 개별 지정
                (기본값)
```

> [!important] ★ 실제 예제 코드는 `@Converter`를 아예 붙이지 않는다
> `ddd-start2/`의 `MoneyConverter`에는 **`@Converter` 애너테이션이 없다.**
> ```java
> // common/jpa/MoneyConverter.java — 실제 코드
> public class MoneyConverter implements AttributeConverter<Money, Integer> {
>     @Override public Integer convertToDatabaseColumn(Money money) { ... }
>     @Override public Money convertToEntityAttribute(Integer value) { ... }
> }
> ```
> 대신 **사용하는 필드마다 `@Convert`로 명시**한다.
> ```java
> // order/command/domain/Order.java
> @Convert(converter = MoneyConverter.class)
> @Column(name = "total_amounts")
> private Money totalAmounts;
> ```
> **왜 이렇게 했을까**: `autoApply = true`로 두면 **모델의 모든 `Money` 필드에 자동 적용**되는데,
> 어떤 필드가 변환되는지 **선언 지점에서 보이지 않는다.** 명시적 `@Convert`는 코드를 읽을 때 변환 여부가 드러난다.
> → [[코드 대조 검증 보고서]]

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "두 프로퍼티를 한 칼럼에 매핑" | **`AttributeConverter`** |
| "`@Embeddable`로 가능한가?" | **불가능** |
| "밸류 → 칼럼 변환 메서드" | **`convertToDatabaseColumn`** |
| "칼럼 → 밸류 변환 메서드" | **`convertToEntityAttribute`** |
| "구현 클래스에 붙이는 애너테이션" | **`@Converter`** |
| "`autoApply`의 기본값" | **`false`** |
| "`autoApply = false`일 때" | **`@Convert(converter = ...)`** 로 직접 지정 |

## Related Notes

- [[엔티티와 밸류 기본 매핑]] — `@Embeddable` 기본 매핑
- [[밸류 컬렉션 매핑]] — 컬렉션을 한 칼럼에 저장할 때도 사용
- [[밸류 타입]] — `Money` 밸류의 배경
- [[연습문제 - 리포지터리와 모델 구현]]
