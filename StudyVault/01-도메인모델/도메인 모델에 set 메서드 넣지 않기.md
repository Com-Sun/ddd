---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 1장 §1.6.5 (p.053~056)
keywords: setter, anti-pattern, constructor, dto, encapsulation
---

# 도메인 모델에 set 메서드 넣지 않기 (★★★)

#ddd #domain-model #anti-pattern #concept

## Overview Table (한눈에 비교)

| 문제 | 결과 |
|------|------|
| **① 도메인 의도가 사라진다** | `completePayment()` → `setOrderState()`가 되면 "결제 완료"라는 도메인 지식이 코드에서 증발 |
| **② 온전하지 않은 상태의 객체** | `new Order()` 직후 필수 값이 비어 있는 **불완전한 객체**가 존재하게 됨 |
| 해법 ① | 도메인 의도를 드러내는 **행위 메서드** 사용 |
| 해법 ② | **생성자에서 필요한 데이터를 모두 받고 검증** |
| 밸류 타입 | **불변**으로 만들면 자연스럽게 set 메서드가 없어진다 |
| 예외 | **DTO**는 상황에 따라 set 메서드를 가질 수 있다 |

> [!note] 왜 습관이 되었을까
> set/get 메서드를 습관적으로 만드는 가장 큰 이유는 **프로그래밍에 입문할 때 읽은 책의 예제 코드** 때문이다.
> 처음 배운 예제를 그대로 따라 하다 보니 **상황에 상관없이** set/get을 추가하게 된다.

## 문제 ① — 도메인의 핵심 개념과 의도가 사라진다

```java
// ❌ 나쁜 예 — set 메서드
public class Order {
    public void setShippingInfo(ShippingInfo newShipping) { ... }
    public void setOrderState(OrderState state) { ... }
}

// ✅ 좋은 예 — 도메인 행위 메서드
public class Order {
    public void changeShippingInfo(ShippingInfo newShipping) { ... }
    public void completePayment() { ... }
}
```

| 메서드 | 전달하는 의미 |
|--------|---------------|
| `changeShippingInfo()` | **배송지 정보를 새로 변경한다** |
| `setShippingInfo()` | 단순히 배송지 **값을 설정한다** |
| `completePayment()` | **결제를 완료했다** |
| `setOrderState()` | 단순히 주문 상태 **값을 설정한다** |

> [!important] 구현 관점에서의 차이
> - `completePayment()`는 결제 완료 처리 코드를 구현하니까, **결제 완료와 관련된 도메인 지식을 코드로 구현하는 것이 자연스럽다.**
> - `setOrderState()`는 단순히 상태 값만 변경할지, 상태값에 따라 다른 처리를 함께 구현할지 **애매하다.**
>
> 습관적으로 작성한 set 메서드는 **필드값만 변경하고 끝나기 때문에**, 상태 변경과 관련된 **도메인 지식이 코드에서 사라진다.**

## 문제 ② — 온전하지 않은 상태의 객체

```java
// ❌ set 메서드로 데이터를 전달하도록 구현하면
//    처음 Order를 생성하는 시점에 Order는 완전하지 않다.
Order order = new Order();

// set 메서드로 필요한 모든 값을 전달해야 한다
order.setOrderLine(lines);
order.setShippingInfo(shippingInfo);

// 주문자(orderer)를 설정하지 않은 상태에서 주문 완료 처리
order.setState(OrderState.PREPARING);
```

> [!danger] 무엇이 문제인가
> 주문자 정보를 담고 있는 필드 `orderer`가 `null`인 상태에서 **상품 준비 중 상태로 바뀌었다.**
> 그렇다고 `orderer`가 `null`인지 검사하는 코드를 `setState()` 메서드에 두는 것도 **맞지 않다.**

### 해법 — 생성자로 모두 받고, 생성자에서 검증한다

```java
Order order = new Order(orderer, lines, shippingInfo, OrderState.PREPARING);
```

```java
public class Order {
    public Order(Orderer orderer, List<OrderLine> orderLines,
                 ShippingInfo shippingInfo, OrderState state) {
        setOrderer(orderer);
        setOrderLines(orderLines);
        // ... 다른 값 설정
    }

    private void setOrderer(Orderer orderer) {
        if (orderer == null) throw new IllegalArgumentException("no orderer");
        this.orderer = orderer;
    }

    private void setOrderLines(List<OrderLine> orderLines) {
        verifyAtLeastOneOrMoreOrderLines(orderLines);
        this.orderLines = orderLines;
        calculateTotalAmounts();
    }

    private void verifyAtLeastOneOrMoreOrderLines(List<OrderLine> orderLines) {
        if (orderLines == null || orderLines.isEmpty()) {
            throw new IllegalArgumentException("no OrderLine");
        }
    }

    private void calculateTotalAmounts() {
        this.totalAmounts = orderLines.stream()
                                      .mapToInt(x -> x.getAmounts())
                                      .sum();
    }
}
```

> [!tip] 결정적 차이는 접근 범위다
> 이 코드에도 `setOrderer` 같은 set 메서드가 있지만, 앞서 언급한 set 메서드와 **중요한 차이**가 있다.
> 바로 **접근 범위가 `private`** 이라는 점이다.
> - **클래스 내부에서 데이터를 변경할 목적**으로만 사용된다.
> - `private`이므로 **외부에서 데이터를 변경할 목적으로 set 메서드를 사용할 수 없다.**

```
  public setter   →  외부 어디서든 상태를 깨뜨릴 수 있다      ❌
  private setter  →  생성자/도메인 메서드 안에서만 사용       ⭕
                     (검증 로직을 한곳에 모을 수 있다)
```

## 밸류 타입은 불변으로

> [!important]
> **불변 밸류 타입을 사용하면 자연스럽게 밸류 타입에는 set 메서드를 구현하지 않는다.**
> set 메서드를 구현해야 할 특별한 이유가 없다면, 불변 타입의 장점을 살릴 수 있도록 **밸류 타입은 불변으로 구현**한다.

## 예외 — DTO는 다르다

DTO(Data Transfer Object)는 **표현 영역과 응용 영역 사이에서 데이터를 주고받을 때 사용하는 일종의 구조체**다.

| 시대 | 상황 |
|------|------|
| 오래된 프레임워크 | 요청 파라미터나 DB 칼럼의 값을 설정할 때 set 메서드를 **필요로 했기 때문에** 어쩔 수 없이 DTO에 get/set을 구현해야 했다 |
| 최근 프레임워크 | **private 필드에 직접 값을 할당하는 기능**을 제공한다 → set 메서드를 만들 필요가 없다 |

> [!tip] 최신 권장
> 프레임워크가 제공하는 기능을 최대한 활용하면 **DTO도 불변 객체가 되어** 불변의 장점을 DTO까지 확장할 수 있다.
>
> DTO는 도메인 로직을 담고 있지 않기에, get/set 메서드를 제공해도 **도메인 객체의 데이터 일관성에 영향을 주지는 않는다.**

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "set 메서드의 가장 큰 문제" | 도메인의 **핵심 개념/의도가 코드에서 사라진다** |
| "`setOrderState()` vs `completePayment()`" | 후자가 **도메인 지식을 표현** |
| "`new Order()` 후 set으로 값 채우기의 문제" | **온전하지 않은(불완전한) 상태**의 객체가 존재 |
| "해법은?" | **생성자에서 필요한 데이터를 모두 받고 검증** |
| "그럼 코드의 private setter는 왜 괜찮은가?" | **접근 범위가 private** → 외부에서 변경 불가 |
| "밸류 타입에 set 메서드를 넣지 않는 이유" | **불변**으로 구현하기 때문 |
| "DTO에도 set을 넣으면 안 되나?" | **넣어도 된다.** 단, 최신 프레임워크는 불필요 |

## Related Notes

- [[밸류 타입]] — 불변 원칙
- [[엔티티와 식별자]]
- [[도메인 모델 도출]] — 생성자에서 제약 검증하기
- [[도메인 용어와 유비쿼터스 언어]] — 이름이 곧 도메인 지식
- [[애그리거트 루트]] — 상태 변경을 루트로 모으기
- [[연습문제 - 도메인 모델 시작하기]]
