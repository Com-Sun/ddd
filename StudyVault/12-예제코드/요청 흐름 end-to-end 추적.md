---
source_pdf: 도메인 주도 개발 시작하기.pdf
source_code: ddd-start2/
part: 예제 프로젝트 (2·6·10장 대응)
keywords: request-flow, trace, end-to-end, event-chain
---

# 요청 흐름 end-to-end 추적 (★★★)

#ddd #example-code #request-flow #concept

## Overview Table (두 가지 흐름)

| 흐름 | 진입점 | 관통하는 개념 |
|------|--------|---------------|
| **A. 주문 생성** | `POST /orders/order` | 표현→응용→도메인→리포지터리, **값 검증**, 식별자 생성, 도메인 서비스 |
| **B. 주문 취소** | `GET /my/orders/{orderNo}/cancel` | **권한 검사**, 애그리거트 규칙, **이벤트 체인**(동기→비동기) |

> [!tip] 왜 이 두 개인가
> **A는 "쓰기 경로 전체"**, **B는 "이벤트가 실제로 어떻게 흐르는지"** 를 보여준다.
> B는 10장 전체(강결합 → 이벤트 → 비동기 → 트랜잭션 연계)를 **한 번에 관통**한다.

---

## A. 주문 생성 흐름

### 전체 그림

```
 [브라우저]
    │ POST /orders/order  (orderReq 폼)
    ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ order/ui/OrderController#order()                    [표현]    │
 │   ① SecurityContextHolder에서 로그인 사용자 추출              │
 │   ② orderRequest.setOrdererMemberId(MemberId.of(username))   │
 └──────────────────────────────────────────────────────────────┘
    │ placeOrderService.placeOrder(orderRequest)
    ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ order/command/application/PlaceOrderService     [응용] @Transactional
 │   ③ OrderRequestValidator.validate()  → 에러 모아서 한 번에   │
 │   ④ ProductRepository.findById() × N  → OrderLine 생성       │
 │   ⑤ orderRepository.nextOrderNo()     → 식별자 생성          │
 │   ⑥ ordererService.createOrderer()    → 도메인 서비스        │
 │   ⑦ new Order(...)                    → 애그리거트 생성      │
 │   ⑧ orderRepository.save(order)                              │
 │   ⑨ return orderNo                    → 애그리거트 아닌 값!  │
 └──────────────────────────────────────────────────────────────┘
    │
    ▼ (⑦ 내부)
 ┌──────────────────────────────────────────────────────────────┐
 │ order/command/domain/Order 생성자                   [도메인]  │
 │   setNumber/setOrderer/setOrderLines/setShippingInfo (private)│
 │   → verifyAtLeastOneOrMoreOrderLines()  제약 검증             │
 │   → calculateTotalAmounts()             총액 계산             │
 │   → Events.raise(new OrderPlacedEvent(...))                  │
 └──────────────────────────────────────────────────────────────┘
    │
    ▼ 트랜잭션 커밋 → INSERT purchase_order + order_line
 [브라우저] "order/orderComplete" + orderNo
```

### 단계별 코드와 개념

| # | 코드 | 개념 노트 |
|---|------|-----------|
| ① ② | `OrderController.java` — `SecurityContextHolder`에서 사용자 추출 후 요청 객체에 주입 | [[표현 영역]] — **표현 영역이 세션/인증을 다룬다** |
| ③ | `OrderRequestValidator.validate()` → `List<ValidationError>` 반환, 비어 있지 않으면 `ValidationErrorException` | [[값 검증]] — ★ **에러를 모아 하나의 익셉션** |
| ④ | `productRepository.findById(new ProductId(...))` | [[ID를 이용한 애그리거트 참조]] |
| ⑤ | `orderRepository.nextOrderNo()` — 리포지터리 `default` 메서드 | [[식별자 생성 기능]] |
| ⑥ | `ordererService.createOrderer(memberId)` — 인터페이스는 도메인, 구현은 인프라 | [[도메인 서비스]] · [[DIP]] |
| ⑦ | `new Order(...)` — private setter가 제약 검증 | [[도메인 모델에 set 메서드 넣지 않기]] · [[애그리거트 루트]] |
| ⑧ | `orderRepository.save(order)` — `OrderLine`까지 함께 저장 | [[리포지터리와 애그리거트]] · [[애그리거트의 영속성 전파]] |
| ⑨ | `return orderNo` — **`Order`를 리턴하지 않는다** | [[메서드 파라미터와 값 리턴]] |

### ★ 관찰 1 — 검증 에러를 표현 영역이 변환한다

```java
// OrderController#order()
} catch (ValidationErrorException e) {
    e.getErrors().forEach(err -> {
        if (err.hasName()) {
            bindingResult.rejectValue(err.getName(), err.getCode());
        } else {
            bindingResult.reject(err.getCode());
        }
    });
    populateProductsAndTotalAmountsModel(orderRequest, modelMap);
    return "order/confirm";
}
```

> 6장 [[값 검증]]의 **해법 A(에러를 모아 하나의 익셉션)** 가 그대로 구현되어 있다.
> 응용 서비스는 `ValidationError` 목록을 담아 던지고, **표현 영역이 `BindingResult`로 변환**한다.
> 덕분에 사용자는 **모든 입력 오류를 한 번에** 본다.

### ★ 관찰 2 — 도메인 서비스가 컨텍스트 경계를 넘는다

```java
// order/command/domain/OrdererService.java  ← 도메인 영역 (인터페이스)
public interface OrdererService {
    Orderer createOrderer(MemberId ordererMemberId);
}

// order/infra/OrdererServiceImpl.java        ← 인프라 영역 (구현)
public Orderer createOrderer(MemberId ordererMemberId) {
    MemberData memberData = memberQueryService.getMemberData(ordererMemberId.getId());
    return new Orderer(MemberId.of(memberData.getId()), memberData.getName());
}
```

> [!important] 한 클래스에 세 개념이 겹쳐 있다
> - **[[DIP]]** — 인터페이스가 고수준(도메인)에, 구현이 저수준(인프라)에
> - **[[도메인 서비스]]** — 상태 없이 로직만
> - **[[바운디드 컨텍스트 간 관계]]의 ACL** — `member` 컨텍스트의 `MemberData`를 **`order` 컨텍스트의 `Orderer`로 변환**
>
> 즉 `order`는 `MemberData`를 **모른 채** 자기 모델(`Orderer`)만 다룬다.

---

## B. 주문 취소 흐름 ★ 이벤트 체인

### 전체 그림

```
 [브라우저]
    │ GET /my/orders/{orderNo}/cancel
    ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ order/ui/CancelOrderController                      [표현]    │
 │   new Canceller(user.getUsername())                          │
 └──────────────────────────────────────────────────────────────┘
    │ cancelOrderService.cancel(orderNo, canceller)
    ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ order/command/application/CancelOrderService    [응용] @Transactional
 │   ① orderRepository.findById() → 없으면 NoOrderException     │
 │   ② cancelPolicy.hasCancellationPermission(order, canceller) │
 │      → false면 NoCancellablePermission                       │
 │   ③ order.cancel()                                           │
 └──────────────────────────────────────────────────────────────┘
    │                                    ▲
    │                                    └── SecurityCancelPolicy (인프라)
    │                                        본인이거나 ROLE_ADMIN
    ▼ (③ 내부)
 ┌──────────────────────────────────────────────────────────────┐
 │ order/command/domain/Order#cancel()                 [도메인]  │
 │   verifyNotYetShipped()  → 이미 배송이면 AlreadyShippedException
 │   state = CANCELED                                           │
 │   Events.raise(new OrderCanceledEvent(number))               │
 └──────────────────────────────────────────────────────────────┘
    │ Events.raise → ApplicationEventPublisher.publishEvent
    │
    ├─────────────[ 동기 · 같은 트랜잭션 ]─────────────┐
    │                                                  ▼
    │                          ┌────────────────────────────────────┐
    │                          │ common/event/EventStoreHandler     │
    │                          │ @EventListener(Event.class)        │
    │                          │ → eventStore.save(event)           │
    │                          │ → INSERT evententry (JSON)         │
    │                          └────────────────────────────────────┘
    │
    ▼ 트랜잭션 커밋 (UPDATE purchase_order SET state='CANCELED')
    │
    └─────────────[ 커밋 후 · 별도 스레드 ]────────────┐
                                                       ▼
                            ┌──────────────────────────────────────┐
                            │ order/infra/OrderCanceledEventHandler│
                            │ @Async                               │
                            │ @TransactionalEventListener(         │
                            │     phase = AFTER_COMMIT)            │
                            │ → refundService.refund(orderNumber)  │
                            └──────────────────────────────────────┘
                                                       ▼
                            ┌──────────────────────────────────────┐
                            │ order/infra/paygate/                 │
                            │   ExternalRefundService              │
                            │ → 외부 결제 시스템 호출 (예제는 로그) │
                            └──────────────────────────────────────┘
```

### ★ 관찰 3 — 두 애너테이션이 함께 붙어 있다

```java
// order/infra/OrderCanceledEventHandler.java
@Async
@TransactionalEventListener(
        classes = OrderCanceledEvent.class,
        phase = TransactionPhase.AFTER_COMMIT
)
public void handle(OrderCanceledEvent event) {
    refundService.refund(event.getOrderNumber());
}
```

> [!important] 책은 따로 설명했지만 실제로는 **함께 쓴다**
> | 애너테이션 | 해결하는 문제 | 책 절 |
> |-----------|---------------|-------|
> | **`@Async`** | 외부 환불이 느려도 **주문 취소가 느려지지 않는다** | 10.5.1 [[비동기 이벤트 처리]] |
> | **`@TransactionalEventListener(AFTER_COMMIT)`** | **트랜잭션이 롤백되면 환불이 실행되지 않는다** | 10.6.1 [[이벤트 적용 시 고려사항]] |
>
> 둘 중 하나만 쓰면 반쪽이다.
> - `@Async`만 → 롤백돼도 환불이 나갈 수 있다 ❌
> - `@TransactionalEventListener`만 → 커밋 후지만 **같은 스레드라 여전히 느려진다** ❌
>
> **두 개를 함께 써야 성능과 정합성이 모두 해결된다.** 이것이 실제 코드에서만 보이는 통합 지점이다.

### ★ 관찰 4 — 이벤트 저장은 동기, 환불은 비동기

같은 `OrderCanceledEvent` 하나에 **두 개의 핸들러**가 붙는다.

| 핸들러 | 애너테이션 | 실행 시점 | 이유 |
|--------|-----------|-----------|------|
| `EventStoreHandler` | `@EventListener(Event.class)` | **동기, 같은 트랜잭션** | 이벤트 저장이 **트랜잭션과 함께 커밋**되어야 유실이 없다 |
| `OrderCanceledEventHandler` | `@Async` + `@TransactionalEventListener` | **커밋 후, 별도 스레드** | 외부 연동은 **늦어도 되고 실패해도 재시도** 가능 |

> [!tip] 이것이 10장의 결론이다
> [[이벤트 적용 시 고려사항]]에서 **"이벤트 발생 코드와 이벤트 저장 처리를 한 트랜잭션으로 처리하면 트랜잭션이 성공할 때만 이벤트가 DB에 저장된다"** 고 했다.
> `EventStoreHandler`가 **동기**인 이유가 정확히 이것이다.

### ★ 관찰 5 — `Event` 상위 클래스의 진짜 용도

```java
// common/event/EventStoreHandler.java
@EventListener(Event.class)   // ← Event를 상속한 모든 이벤트를 받는다
public void handle(Event event) {
    eventStore.save(event);
}
```

> [!important]
> 책 10.3.1은 `Event` 추상 클래스를 **"공통 프로퍼티(발생 시간)가 필요하면"** 만든다고 설명했다.
> 하지만 실제 용도는 더 크다 — **`@EventListener(Event.class)` 하나로 모든 이벤트를 이벤트 저장소에 일괄 저장**하기 위한 **타입 앵커**다.
>
> 그래서 이 프로젝트의 **모든 이벤트 클래스는 `Event`를 상속**한다:
> `OrderPlacedEvent`, `OrderCanceledEvent`, `ShippingInfoChangedEvent`, `ShippingStartedEvent`, `PasswordChangedEvent`, `MemberBlockedEvent`, `MemberUnblockedEvent`

### 권한 검사 — 도메인 객체 단위

```java
// order/command/domain/CancelPolicy.java        ← 도메인 (인터페이스)
public interface CancelPolicy {
    boolean hasCancellationPermission(Order order, Canceller canceller);
}

// order/infra/domain/SecurityCancelPolicy.java   ← 인프라 (구현)
public boolean hasCancellationPermission(Order order, Canceller canceller) {
    return isCancellerOrderer(order, canceller) || isCurrentUserAdminRole();
}
```

> 6장 [[권한 검사]]의 **③ 도메인 객체 단위 권한 검사**가 그대로 구현되어 있다.
> **"게시글 삭제는 본인 또는 관리자만"** 예시와 구조가 동일하다 —
> **애그리거트를 먼저 로딩해야 판단할 수 있으므로** URL이나 메서드 수준에서는 불가능하다.

---

## 이벤트 저장소 이후 — 포워더 / API

```
   evententry 테이블
        │
        ├──[ 포워더 방식 ]──▶ integration/EventForwarder  @Scheduled(1초)
        │                        │ offsetStore.get()
        │                        │ eventStore.get(offset, 100)
        │                        │ eventSender.send(entry)
        │                        ▼
        │                     SysoutEventSender (예제는 콘솔 출력)
        │
        └──[ API 방식 ]────▶ eventstore/ui/EventApi
                                 GET /api/events?offset=0&limit=5
```

> [!warning] ⚠️ 포워더는 실제로 동작하지 않는다
> `EventForwarder`에 `@Scheduled`가 붙어 있지만, **프로젝트 전체에 `@EnableScheduling`이 없다.**
> 스프링은 `@EnableScheduling` 없이는 `@Scheduled`를 처리하지 않으므로 **포워더는 실행되지 않는다.**
>
> 확인:
> ```bash
> grep -rn "EnableScheduling" ddd-start2/src/    # 결과 없음
> ```
> 데모 목적상 1초마다 콘솔 출력을 막으려는 의도일 수도 있다.
> **직접 돌려보려면 `ShopApplication`에 `@EnableScheduling`을 추가**하면 된다.
> → [[코드 대조 검증 보고서]]

---

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "주문 생성에서 식별자는 어디서 생성?" | **`orderRepository.nextOrderNo()`** (default 메서드) |
| "응용 서비스가 리턴하는 것" | **`OrderNo`** — 애그리거트 아님 |
| "검증 에러를 `BindingResult`로 바꾸는 주체" | **표현 영역(컨트롤러)** |
| "`OrdererServiceImpl`이 겹쳐 구현한 세 개념" | **DIP + 도메인 서비스 + ACL** |
| "`@Async`와 `@TransactionalEventListener`를 함께 쓰는 이유" | **성능 분리 + 롤백 시 미실행** 둘 다 필요 |
| "`OrderCanceledEvent`의 핸들러 수" | **2개** — 저장(동기) / 환불(비동기) |
| "`EventStoreHandler`가 동기인 이유" | 이벤트 저장이 **트랜잭션과 함께 커밋**되어야 유실 없음 |
| "`Event` 추상 클래스의 실제 용도" | **`@EventListener(Event.class)` 일괄 수신용 타입 앵커** |
| "취소 권한 검사가 도메인 객체 단위인 이유" | **애그리거트를 로딩해야 본인 여부 판단 가능** |
| "포워더가 동작하지 않는 이유" | **`@EnableScheduling` 부재** |

## Related Notes

- [[패키지 구조와 모듈 지도]]
- [[개념→코드 매핑표]]
- [[코드 대조 검증 보고서]]
- [[코드 리딩 연습문제]]
- [[요청 처리 흐름]] — 2장의 흐름 개념
- [[값 검증]] · [[권한 검사]] — 6장
- [[비동기 이벤트 처리]] · [[이벤트 적용 시 고려사항]] — 10장
- [[도메인 서비스]] — 7장
