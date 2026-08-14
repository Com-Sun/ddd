---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 3장 §3.4 (p.114~118)
keywords: aggregate-reference, id-reference, coupling, lazy-loading, scalability
---

# ID를 이용한 애그리거트 참조 (★★★)

#ddd #aggregate #aggregate-reference #concept

## Overview Table (한눈에 비교)

| 구분 | 필드(객체) 직접 참조 | **ID 참조** |
|------|----------------------|-------------|
| 편의성 | 탐색이 쉽다 | 응용 서비스에서 별도 로딩 필요 |
| **문제 ①** | **편한 탐색 오용** — 다른 애그리거트 상태를 쉽게 변경 | 근원적으로 **방지됨** |
| **문제 ②** | **성능에 대한 고민** — 지연/즉시 로딩 전략 | 고민 불필요 |
| **문제 ③** | **확장의 어려움** — 단일 기술(JPA) 강제 | 저장소별 분리 **가능** |
| 결합도 | 높다 | **낮다** (응집도는 높아진다) |
| 모델 복잡도 | 높다 | **낮다** |
| 주의점 | — | **N+1 조회 문제** (→ [[ID 참조와 조회 성능]]) |

## 애그리거트 간 참조란

> [!important]
> 애그리거트 관리 주체는 애그리거트 루트이므로,
> **애그리거트에서 다른 애그리거트를 참조한다는 것은 다른 애그리거트의 루트를 참조한다는 것**과 같다.

### 필드를 통한 직접 참조 (그림 3.6)

```java
public class Order {
    private Orderer orderer;
}

public class Orderer {
    private Member member;      // ← 다른 애그리거트 루트를 직접 참조
    private String name;
}

public class Member {
    private MemberId id;
}
```

편리하다.

```java
order.getOrderer().getMember().getId()
```

JPA는 `@ManyToOne`, `@OneToOne` 같은 애너테이션으로 **연관된 객체를 로딩하는 기능을 제공**하므로 필드를 이용해 다른 애그리거트를 쉽게 참조할 수 있다.

## ★ 직접 참조의 세 가지 문제

### 문제 ① — 편한 탐색 오용

> [!danger] 가장 큰 문제
> 한 애그리거트 내부에서 다른 애그리거트 객체에 접근할 수 있으면 **다른 애그리거트의 상태를 쉽게 변경**할 수 있게 된다.
> **구현이 쉬워진다는 것 때문에 다른 애그리거트를 수정하고자 하는 유혹**에 빠지기 쉽다.

```java
public class Order {
    private Orderer orderer;

    public void changeShippingInfo(ShippingInfo newShippingInfo,
                                   boolean useNewShippingAddrAsMemberAddr) {
        ...
        if (useNewShippingAddrAsMemberAddr) {
            // 한 애그리거트 내부에서 다른 애그리거트에 접근할 수 있으면
            // 구현이 쉬워진다는 것 때문에 다른 애그리거트의 상태를 변경하는
            // 유혹에 빠지기 쉽다.
            orderer.getMember().changeAddress(newShippingInfo.getAddress());
        }
    }
}
```

→ 한 애그리거트에서 다른 애그리거트의 상태를 변경하는 것은 **애그리거트 간의 의존 결합도를 높여** 결과적으로 애그리거트의 변경을 어렵게 만든다. (→ [[트랜잭션 범위]])

### 문제 ② — 성능에 대한 고민

JPA를 사용하면 참조한 객체를 **지연(lazy) 로딩**과 **즉시(eager) 로딩** 두 방식으로 로딩할 수 있다.

| 상황 | 유리한 로딩 방식 |
|------|------------------|
| 단순히 연관된 객체의 데이터를 **함께 화면에 보여줘야** 하면 | **즉시 로딩** |
| 애그리거트의 **상태를 변경하는 기능**을 실행하면 (불필요한 객체 로딩 불필요) | **지연 로딩** |

> [!warning]
> 이런 다양한 경우의 수를 고려해서 **연관 매핑과 JPQL/Criteria 쿼리의 로딩 전략을 결정**해야 한다.

### 문제 ③ — 확장

```
[ 초기 ]  단일 서버 + 단일 DBMS

[ 트래픽 증가 ]
      ▼
  하위 도메인별로 시스템 분리
      ▼
  하위 도메인마다 서로 다른 DBMS
      ▼
  심지어 하위 도메인마다 다른 종류의 데이터 저장소
  (한쪽은 MariaDB, 다른 쪽은 몽고DB)
      ▼
  ★ 더 이상 JPA와 같은 단일 기술로
    다른 애그리거트 루트를 참조할 수 없다
```

## ⭕ 해법 — ID로 참조하기

```java
public class Order {
    private Orderer orderer;
}

public class Orderer {
    private MemberId memberId;      // ← 객체가 아닌 ID로 참조
    private String name;
}
```

DB 테이블에서 **외래키로 참조하는 것과 비슷하게**, ID를 이용한 참조는 **다른 애그리거트를 참조할 때 ID를 사용**한다.

> [!important] ID 참조의 효과
> - 모든 객체가 참조로 연결되지 않고 **한 애그리거트에 속한 객체들만 참조로 연결**된다.
> - **애그리거트의 경계를 명확히** 하고 애그리거트 간 **물리적인 연결을 제거**하기 때문에 **모델의 복잡도를 낮춰준다.**
> - 애그리거트 간의 **의존을 제거하므로 응집도를 높여주는** 효과도 있다.
> - **구현 복잡도가 낮아진다.** 지연 로딩/즉시 로딩을 고민하지 않아도 된다.
> - 참조하는 애그리거트가 필요하면 **응용 서비스에서 ID를 이용해서 로딩**하면 된다.

```java
public class ChangeOrderService {

    @Transactional
    public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo,
                                   boolean useNewShippingAddrAsMemberAddr) {
        Order order = orderRepository.findById(id);
        if (order == null) throw new OrderNotFoundException();
        order.changeShippingInfo(newShippingInfo);

        if (useNewShippingAddrAsMemberAddr) {
            // ID를 이용해서 참조하는 애그리거트를 구한다.
            Member member = memberRepository.findById(
                    order.getOrderer().getMemberId());
            member.changeAddress(newShippingInfo.getAddress());
        }
    }
}
```

> [!tip]
> 응용 서비스에서 필요한 애그리거트를 로딩하므로 **애그리거트 수준에서 지연 로딩을 하는 것과 동일한 결과**를 만든다.

### 세 문제가 모두 해결된다

| 문제 | ID 참조로 해결되는 방식 |
|------|-------------------------|
| ① 편한 탐색 오용 | **근원적으로 방지.** 외부 애그리거트를 직접 참조하지 않으므로 **애초에 다른 애그리거트의 상태를 변경할 수 없다** |
| ② 성능 고민 | 지연/즉시 로딩 고민 **불필요** |
| ③ 확장 | **애그리거트별로 다른 구현 기술 사용 가능** |

### 확장 예 (그림 3.8)

```
   «interface» OrderRepository        «interface» ProductRepository
              △                                    △
              ╎                                    ╎
      JpaOrderRepository              MongoProductRepository
       (RDBMS에 저장)                      (NoSQL에 저장)
```

- 중요한 데이터인 **주문 애그리거트는 RDBMS**에 저장
- 조회 성능이 중요한 **상품 애그리거트는 NoSQL**에 저장
- 또한 각 도메인을 **별도 프로세스로 서비스**하도록 구현할 수도 있다

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "애그리거트가 다른 애그리거트를 참조한다" | 다른 애그리거트의 **루트를 참조**한다 |
| "직접 참조의 세 가지 문제" | **편한 탐색 오용 / 성능 고민 / 확장의 어려움** |
| "가장 큰 문제는?" | **편리함의 오용** (다른 애그리거트 상태 변경 유혹) |
| "화면에 함께 보여줄 때 유리한 로딩" | **즉시 로딩** |
| "상태 변경 기능 실행 시 유리한 로딩" | **지연 로딩** |
| "ID 참조가 낮추는 것 / 높이는 것" | 복잡도·결합도 **↓** / **응집도 ↑** |
| "ID 참조 시 참조 애그리거트는 어디서 로딩?" | **응용 서비스** |
| "ID 참조는 무엇과 유사한가" | DB 테이블의 **외래키 참조** |
| "ID 참조의 부작용" | **N+1 조회 문제** |

## Related Notes

- [[애그리거트]] — 경계
- [[트랜잭션 범위]] — 다른 애그리거트를 변경하면 안 되는 이유
- [[ID 참조와 조회 성능]] — N+1 문제와 해법
- [[엔티티 식별자와 밸류 타입]] — `MemberId` 같은 식별자 타입
- [[애그리거트 로딩 전략]] — 4장
- [[애그리거트 간 집합 연관]]
- [[연습문제 - 애그리거트]]
