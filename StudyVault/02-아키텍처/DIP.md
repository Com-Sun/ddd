---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 2장 §2.3~2.3.1 (p.070~077)
keywords: dip, dependency-inversion, high-level-module, abstraction, test-double
---

# DIP — 의존 역전 원칙 (★★★)

#ddd #architecture #dip #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| **고수준 모듈** | **의미 있는 단일 기능**을 제공하는 모듈 (예: 가격 할인 계산) |
| **저수준 모듈** | 고수준 모듈의 **하위 기능을 실제로 구현**한 것 (JPA로 고객 조회, Drools로 룰 실행) |
| 문제 | 고수준이 저수준에 의존하면 **구현 교체·테스트가 어렵다** |
| **DIP의 해법** | **저수준 모듈이 고수준 모듈에 의존하도록 뒤집는다** |
| 비밀 | **추상화한 인터페이스** |
| **★ 핵심 주의사항** | 추상화한 인터페이스는 **고수준 모듈 관점에서 도출**하고 **고수준 모듈에 위치**시킨다 |
| 얻는 것 | ① 구현 기술 교체 용이 ② **대역(Test Double)으로 테스트 가능** |

## 고수준 모듈과 저수준 모듈

```
  [ 고수준 ]  CalculateDiscountService   ← 의미 있는 단일 기능: "가격 할인 계산"
                        │
        ┌───────────────┴───────────────┐
        ▼                               ▼
  고객 정보를 구한다                 룰을 실행한다        ← 하위 기능
        │                               │
  [ 저수준 ]  JPA 모듈              Drools 모듈          ← 하위 기능의 실제 구현
```

- **고수준 모듈**: 의미 있는 단일 기능을 제공하는 모듈. `CalculateDiscountService`는 **가격 할인 계산** 기능을 구현한다.
- 고수준 기능을 구현하려면 **하위 기능**이 필요하다. 여기서는 ① 고객 정보 조회 ② 룰 실행.
- **저수준 모듈**: 하위 기능을 실제로 구현한 것. (JPA로 고객 정보를 읽는 모듈, Drools로 룰을 실행하는 모듈)

> [!warning] 고수준이 저수준을 직접 사용하면
> [[계층 구조 아키텍처]]에서 언급한 두 문제, 즉 **구현 변경과 테스트가 어렵다**는 문제가 발생한다.

## DIP — 의존을 뒤집는다

> [!important] 정의
> 고수준 모듈을 구현하려면 저수준 모듈을 사용해야 하는데,
> 반대로 **저수준 모듈이 고수준 모듈에 의존**하도록 만드는 것을
> **DIP(Dependency Inversion Principle, 의존 역전 원칙)** 이라고 한다.
>
> 비밀은 **추상화한 인터페이스**에 있다.

### 1단계 — 고수준 관점에서 추상화

`CalculateDiscountService` 입장에서는 룰 적용을 **Drools로 구현했는지 자바로 직접 구현했는지는 중요하지 않다.**
**'고객 정보와 구매 정보에 룰을 적용해서 할인 금액을 구한다'** 는 것만 중요하다.

```java
public interface RuleDiscounter {
    Money applyRules(Customer customer, List<OrderLine> orderLines);
}
```

### 2단계 — 고수준 모듈이 인터페이스에 의존

```java
public class CalculateDiscountService {
    private RuleDiscounter ruleDiscounter;

    public CalculateDiscountService(RuleDiscounter ruleDiscounter) {
        this.ruleDiscounter = ruleDiscounter;      // 생성자로 주입
    }

    public Money calculateDiscount(List<OrderLine> orderLines, String customerId) {
        Customer customer = findCustomer(customerId);
        return ruleDiscounter.applyRules(customer, orderLines);
    }
}
```

> [!tip]
> `CalculateDiscountService`에는 **Drools에 의존하는 코드가 없다.**
> 단지 `RuleDiscounter`가 룰을 적용한다는 사실만 알 뿐이다.
> 실제 구현 객체는 **생성자를 통해서 전달받는다.**

### 3단계 — 저수준 모듈이 인터페이스를 구현

```java
public class DroolsRuleDiscounter implements RuleDiscounter {
    private KieContainer kContainer;

    public DroolsRuleDiscounter() {
        KieServices ks = KieServices.Factory.get();
        kContainer = ks.getKieClasspathContainer();
    }

    @Override
    public Money applyRules(Customer customer, List<OrderLine> orderLines) {
        KieSession kSession = kContainer.newKieSession("discountSession");
        try {
            // ... 코드 생략
            kSession.fireAllRules();
        } finally {
            kSession.dispose();
        }
        return money.toImmutableMoney();
    }
}
```

### 구조 변화

```
[ Before ]                          [ After — DIP 적용 ]

CalculateDiscountService            CalculateDiscountService  (고수준)
         │                                    │
         ▼ 직접 의존                          ▼ 의존
  DroolsRuleEngine                    «interface» RuleDiscounter  (고수준에 위치!)
   (인프라/저수준)                              △
                                               ╎ 상속(= 의존의 다른 형태)
                                      DroolsRuleDiscounter  (저수준)
```

> [!important] 왜 "역전"인가
> 원래는 고수준 모듈이 저수준 모듈에 의존해야 하는데, 반대로 **저수준 모듈이 고수준 모듈에 의존**하게 된다.
> `RuleDiscounter` 인터페이스는 **고수준 모듈에 속하고**, `DroolsRuleDiscounter`는 그것을 구현했으므로 **저수준 모듈에 속한다.**

## 이점 ① — 구현 기술 교체가 쉽다

```java
// 사용할 저수준 객체 생성
RuleDiscounter ruleDiscounter = new DroolsRuleDiscounter();
// 생성자 방식으로 주입
CalculateDiscountService disService = new CalculateDiscountService(ruleDiscounter);
```

구현 기술을 변경하더라도 `CalculateDiscountService`를 **수정할 필요가 없다.** 생성 코드만 바꾸면 된다.

```java
// 사용할 저수준 구현 객체 변경
RuleDiscounter ruleDiscounter = new SimpleRuleDiscounter();
// 저수준 모듈을 변경해도 고수준 모듈은 수정 불필요
CalculateDiscountService disService = new CalculateDiscountService(ruleDiscounter);
```

## 이점 ② — 대역(Test Double)으로 테스트 가능

고객 조회도 마찬가지로 고수준 인터페이스를 만든다.

```java
public class CalculateDiscountService {
    private CustomerRepository customerRepository;
    private RuleDiscounter ruleDiscounter;

    public CalculateDiscountService(CustomerRepository customerRepository,
                                    RuleDiscounter ruleDiscounter) {
        this.customerRepository = customerRepository;
        this.ruleDiscounter = ruleDiscounter;
    }

    public Money calculateDiscount(List<OrderLine> orderLines, String customerId) {
        Customer customer = findCustomer(customerId);
        return ruleDiscounter.applyRules(customer, orderLines);
    }

    private Customer findCustomer(String customerId) {
        Customer customer = customerRepository.findById(customerId);
        if (customer == null) throw new NoCustomerException();
        return customer;
    }
}
```

```java
public class CalculateDiscountServiceTest {

    @Test
    public void noCustomer_thenExceptionShouldBeThrown() {
        // 테스트 목적의 대역 객체
        CustomerRepository stubRepo = mock(CustomerRepository.class);
        when(stubRepo.findById("noCustId")).thenReturn(null);

        RuleDiscounter stubRule = (cust, lines) -> null;

        // 대역 객체를 주입 받아 테스트 진행
        CalculateDiscountService calDisSvc =
                new CalculateDiscountService(stubRepo, stubRule);

        assertThrows(NoCustomerException.class,
                () -> calDisSvc.calculateDiscount(someLines, "noCustId"));
    }
}
```

| 대역 | 생성 방법 |
|------|-----------|
| `stubRepo` | **Mockito** 같은 Mock 프레임워크로 생성 |
| `stubRule` | 메서드가 한 개여서 **람다식**으로 생성 |

> [!tip] 왜 테스트가 가능해졌는가
> **DIP를 적용해서 고수준 모듈이 저수준 모듈에 의존하지 않도록 했기 때문이다.**
> JPA를 이용한 `CustomerRepository` 구현 클래스와 Drools를 이용한 `RuleDiscounter` 구현 클래스가 **없어도**
> 스텁이나 모의 객체 같은 **테스트 대역**으로 거의 모든 상황을 테스트할 수 있다.

## ★ DIP 주의사항 (§2.3.1)

> [!danger] DIP는 "인터페이스와 구현 클래스 분리"가 아니다
> DIP를 잘못 생각하면 단순히 인터페이스와 구현 클래스를 분리하는 정도로 받아들일 수 있다.
> DIP의 핵심은 **고수준 모듈이 저수준 모듈에 의존하지 않도록 하기 위함**이다.

### 잘못된 적용 (그림 2.10)

```
   [ 도메인 ]                        [ 인프라 ]
   CalculateDiscountService ──────▶ «interface» RuleEngine    ❌
                                            △
                                            ╎
                                      DroolsRuleEngine
```

| 무엇이 잘못됐나 | 설명 |
|-----------------|------|
| 의존 방향 | 도메인 영역이 **여전히 인프라스트럭처 영역에 의존**하고 있다 |
| 인터페이스 도출 관점 | `RuleEngine`은 고수준(도메인) 관점이 아니라 **룰 엔진이라는 저수준 모듈 관점**에서 도출됐다 |

### 올바른 적용 (그림 2.11)

```
   [ 도메인 ]
   CalculateDiscountService ──────▶ «interface» RuleDiscounter   ⭕
                                            △        (고수준에 위치)
   [ 인프라 ]                               ╎
                                      DroolsRuleDiscounter
```

> [!important] 규칙
> DIP를 적용할 때 하위 기능을 추상화한 인터페이스는 **고수준 모듈 관점에서 도출**한다.
> `CalculateDiscountService` 입장에서는 할인 금액을 구하기 위해 **룰 엔진을 쓰는지 직접 연산하는지는 중요하지 않다.**
> 단지 **"규칙에 따라 할인 금액을 계산한다"** 는 것만 중요하다.
> 즉 **'할인 금액 계산'을 추상화한 인터페이스는 저수준이 아닌 고수준 모듈에 위치한다.**

**저수준 관점 vs 고수준 관점 인터페이스 이름 비교**

| 관점 | 인터페이스 이름 | 판정 |
|------|-----------------|------|
| 저수준(룰 엔진) 관점 | `RuleEngine` — "룰 엔진을 쓴다" | ❌ |
| 고수준(도메인) 관점 | `RuleDiscounter` — "규칙으로 할인을 계산한다" | ⭕ |

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "의미 있는 단일 기능을 제공하는 모듈" | **고수준 모듈** |
| "하위 기능을 실제 구현한 모듈" | **저수준 모듈** |
| "DIP의 핵심 목적" | 고수준이 **저수준에 의존하지 않게** 하기 |
| "DIP를 가능하게 하는 비밀" | **추상화한 인터페이스** |
| "인터페이스를 어디서 도출하나?" | **고수준 모듈 관점** |
| "인터페이스는 어느 영역에 위치?" | **고수준 모듈(도메인/응용)** |
| "`RuleEngine` vs `RuleDiscounter`" | `RuleEngine`은 **저수준 관점 → 잘못된 DIP** |
| "DIP = 인터페이스와 구현 분리?" | **거짓.** 의존 방향 역전이 핵심 |
| "DIP의 두 가지 이점" | **구현 교체 용이 + 테스트 대역 사용 가능** |
| "상속은 의존인가?" | **의존의 다른 형태다** |

## Related Notes

- [[계층 구조 아키텍처]] — DIP가 해결하는 문제
- [[DIP와 아키텍처]] — 아키텍처 수준의 DIP 적용
- [[네 개의 영역]]
- [[리포지터리 개요]] — DIP의 대표적 적용 사례
- [[도메인 구현과 DIP]] — 4장의 현실적 타협
- [[도메인 서비스의 패키지 위치와 인터페이스]] — 7장의 DIP 적용
- [[연습문제 - 아키텍처 개요]]
