# ddd

『도메인 주도 개발 시작하기』(최범균) 학습 저장소.

## 구성

| 경로 | 내용 |
|---|---|
| [`StudyVault/`](StudyVault) | 장별 노트 · 개념 트래커 · 질문 기록 |
| [`ddd-start2/`](ddd-start2) | 책 예제 코드 |
| [`.agents/skills/tutor/`](.agents/skills/tutor) | 서술형 퀴즈를 출제·채점하고 노트를 갱신하는 스킬 |

Obsidian 볼트로 열면 위키 링크가 연결된다.

## 노트 규칙

이해한 내용이 아니라 틀린 지점을 기준으로 쌓는다.

- **개념 트래커** — [`StudyVault/concepts/`](StudyVault/concepts). 개념 단위로 시도·정답·상태(🟢🔴)를 기록하고, 오답은 결론이 아니라 무엇을 어떻게 착각했는지를 남긴다.
- **질문 기록** — [내 질문 노트](StudyVault/00-Dashboard/내%20질문%20노트.md). 막혔던 지점만 최신순으로 모은다.
- **정정** — 틀린 서술은 덮어쓰지 않고 주석을 달아 남긴다.

진행 상황은 [학습 대시보드](StudyVault/00-Dashboard/학습%20대시보드.md).

## 근거 등급

서술마다 출처 등급을 붙인다.

| 등급 | 기준 |
|---|---|
| **A** | 실제 코드 · DDL · 1차 문서를 직접 확인 |
| **B** | 쪽수가 있는 책 기반 노트에서 인용 |
| **C** | 일반 지식 · 추론. 재검증 대상 |

**정정 사례** — `@SecondaryTable`로 `article_content`를 분리한 이유를 "본문이 커서 목록 조회에서 제외하려고"라고 적어두었다(C등급). 대조 결과 두 가지가 어긋났다.

- 스키마가 반대다. 분리된 `article_content.content`는 `varchar(255)`이고, 같은 테이블에 남은 `product.detail`이 `text`다.
- `@SecondaryTable`은 목록 조회에서도 항상 조인한다. 책은 이를 해당 매핑의 한계로 다루며 해법으로 조회 전용 기능(5장·11장)을 제시한다.

정정 주석은 질문 노트 Q9 · Q11 · Q12에 남아 있다.

## ddd-start2

책 저자의 예제 코드. [madvirus/ddd-start2](https://github.com/madvirus/ddd-start2) `d46b6cb` 스냅샷.

Java 17 · Spring Boot 2.6.1 · MySQL 8. 통합 테스트(`*IT`)는 이름을 명시해야 실행되고, `src/sql/ddl.sql`로 만든 `shop` 스키마가 필요하다.

```bash
cd ddd-start2
./mvnw test -Dtest=OrderRepositoryIT
```
