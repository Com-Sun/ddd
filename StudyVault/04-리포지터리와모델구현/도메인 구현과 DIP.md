---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 4장 §4.7 (p.170~172)
keywords: dip, tradeoff, pragmatism, testability, jpa-annotation
---

# 도메인 구현과 DIP (★★★)

#ddd #repository #dip #architecture #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 인정 | **이 장에서 구현한 리포지터리는 DIP 원칙을 어기고 있다** |
| 위반 ① | 엔티티가 **JPA에 특화된 애너테이션**(`@Entity`, `@Table`, `@Id`, `@Column`) 사용 |
| 위반 ② | 도메인 패키지의 리포지터리 인터페이스가 **스프링 데이터 JPA의 `Repository`를 상속** |
| 순수 구조 | 구현 클래스를 **인프라에 위치**시키고 도메인에서 JPA 애너테이션 제거 |
| **저자의 선택** | **타협** — 개발 편의성과 실용성을 택함 |
| 근거 ① | **리포지터리와 도메인 모델의 구현 기술은 거의 바뀌지 않는다** |
| 근거 ② | JPA 애너테이션을 써도 **단위 테스트에 문제가 없다** |

## 위반 ① — 엔티티의 JPA 애너테이션

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
    ...
}
```

> [!warning]
> DIP에 따르면 **`@Entity`, `@Table`은 구현 기술에 속하므로** `Article` 같은 도메인 모델은 구현 기술인 JPA에 의존하지 말아야 하는데,
> 이 코드는 **도메인 모델인 `Article`이 영속성 구현 기술인 JPA에 의존**하고 있다.

## 위반 ② — 리포지터리 인터페이스의 상속

```java
import org.springframework.data.repository.Repository;

public interface ArticleRepository extends Repository<Article, Long> {
    void save(Article article);
    Optional<Article> findById(Long id);
}
```

> [!warning]
> `ArticleRepository` 인터페이스는 **도메인 패키지에 위치하는데 구현 기술인 스프링 데이터 JPA의 `Repository` 인터페이스를 상속**하고 있다.
> 즉 **도메인이 인프라에 의존**하는 것이다.

## 순수한 도메인 모델을 원한다면 (그림 4.9)

```
   [ domain ]                          [ infra ]
   «interface» ArticleRepository ◀╌╌╌╌ JpaArticleRepository
              │                                  │
              │ 사용                             │ 사용
              ▼                                  ▼
          Article  (JPA 애너테이션 없음)  ╌╌▶  JpaArticle (매핑 전용)
```

**해야 할 일**

| 작업 |
|------|
| 스프링 데이터 JPA의 `Repository` 인터페이스를 **상속받지 않도록 수정** |
| `ArticleRepository` 인터페이스를 **구현한 클래스를 인프라에 위치** |
| `Article` 클래스에서 `@Entity`, `@Table` 같은 **JPA 특화 애너테이션을 모두 제거** |
| **인프라에 JPA를 연동하기 위한 클래스를 추가** |

> 특정 기술에 의존하지 않는 순수한 도메인 모델을 추구하는 개발자는 이 구조로 구현한다.
> 이 구조를 가지면 **구현 기술을 변경하더라도 도메인이 받는 영향을 최소화**할 수 있다.

## ★ 저자의 선택 — 타협

> [!important] 핵심 논거 (p.171~172)
> **DIP를 적용하는 주된 이유는 저수준 구현이 변경되더라도 고수준이 영향을 받지 않도록 하기 위함이다.**
> **하지만 리포지터리와 도메인 모델의 구현 기술은 거의 바뀌지 않는다.**
>
> 저자의 경험:
> - JPA로 구현한 리포지터리 구현 기술을 **마이바티스나 다른 기술로 변경한 적이 없다.**
> - **RDBMS를 사용하다 몽고DB로 변경한 적도 없다.**
>
> **이렇게 변경이 거의 없는 상황에서 변경을 미리 대비하는 것은 과하다.**
> 그래서 저자는 애그리거트, 리포지터리 등 도메인 모델을 구현할 때 **타협을 했다.**

### 타협해도 괜찮은 이유

| 우려 | 실제 |
|------|------|
| JPA 애너테이션 때문에 테스트가 어렵지 않을까? | **도메인 모델을 단위 테스트하는 데 문제는 없다** |
| JPA에 맞춰 도메인 모델을 구현해야 하지 않을까? | 그런 경우도 있지만 **이런 상황은 드물다** |
| `Repository` 상속이 테스트를 방해하지 않을까? | **리포지터리 자체는 인터페이스이고 테스트 가능성을 해치지 않는다** |

> [!tip] 결론
> **DIP를 완벽하게 지키면 좋겠지만, 개발 편의성과 실용성을 가져가면서 구조적인 유연함은 어느 정도 유지했다.**
> **복잡도를 높이지 않으면서(즉 JPA 애너테이션을 도메인 모델에 사용하면서) 기술에 따른 구현 제약이 낮다면 합리적인 선택**이라고 생각한다.

```
   완벽한 DIP                              실용적 타협 (저자의 선택)
   ├ 도메인에 JPA 애너테이션 없음           ├ 도메인에 JPA 애너테이션 사용
   ├ 인프라에 별도 매핑 클래스              ├ 리포지터리는 Repository 상속
   ├ 구현 기술 교체에 완벽 대응             ├ 단위 테스트 가능 ⭕
   └ 복잡도 ↑↑                             └ 복잡도 ↓, 구현 제약 낮음
        ▲                                        ▲
   "거의 바뀌지 않는 것"을 위한 비용        판단 기준: 실제로 바뀌는가?
```

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "4장 리포지터리는 DIP를 지키는가?" | **어기고 있다** |
| "위반 사례 두 가지" | 도메인의 **JPA 애너테이션** + 리포지터리의 **`Repository` 상속** |
| "순수 구조로 가려면?" | 구현 클래스를 **인프라에 위치**, 도메인에서 JPA 애너테이션 **제거** |
| "저자가 타협한 근거" | **리포지터리·도메인 모델의 구현 기술은 거의 바뀌지 않는다** |
| "타협해도 되는 이유" | **단위 테스트에 문제 없다** |
| "판단 기준" | **복잡도를 높이지 않고 구현 제약이 낮다면 합리적** |

## Related Notes

- [[DIP]] — 2장의 원칙
- [[DIP와 아키텍처]] — "DIP를 항상 적용할 필요는 없다"
- [[인프라스트럭처 개요]] — 2장의 같은 타협 논의
- [[JPA 리포지터리 구현]]
- [[스프링 데이터 JPA 리포지터리]]
- [[엔티티와 밸류 기본 매핑]]
- [[연습문제 - 리포지터리와 모델 구현]]
