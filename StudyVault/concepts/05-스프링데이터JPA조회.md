# 05-스프링데이터JPA조회 — 개념 트래커

| 개념 | 시도 | 정답 | 최근 테스트 | 상태 |
|------|------|------|-------------|------|
| Pageable → LIMIT + OFFSET 변환 | 1 | 0 | 2026-08-06 | 🔴 |
| COUNT 쿼리 실행 규칙 (findBy+List vs findAll(Spec)+List) | 1 | 0 | 2026-08-06 | 🔴 |
| Specification(스펙)의 목적과 실제 구현 방식 | 1 | 0 | 2026-08-06 | 🔴 |

### 오답 메모

**Pageable → LIMIT + OFFSET 변환**
- 혼동: `OFFSET 10`만 적었다 ("pageable 헷갈")
- 핵심: `PageRequest.of(page, size)` → **`LIMIT size OFFSET page*size`** — OFFSET만이 아니라 둘 다로 변환된다
- 함께: **페이지 번호는 0부터** 시작. `PageRequest.of(1, 10)`은 1페이지가 아니라 **2페이지(11~20번째)**
- 부속: `ordererId`는 주문자(회원) ID이므로 WHERE 절은 `orderer_id`(주문 자신의 ID가 아니다)

**COUNT 쿼리 실행 규칙 (findBy+List vs findAll(Spec)+List)**
- 혼동: 쿼리 형태만 답하고 COUNT 쿼리 실행 여부를 전혀 언급하지 못했다 — 이것이 문제의 핵심 포인트였다
- 핵심 표:
  - `findByXxx(..., Pageable)` + `Page` → COUNT ⭕
  - `findByXxx(..., Pageable)` + `List` → COUNT ❌
  - **`findAll(Specification, Pageable)` + `List`여도 → COUNT ⭕** ⚠️
- 함정: "Page면 COUNT, List면 안 함"으로만 외우면 틀린다. **스펙을 쓰는 `findAll`은 예외**
- 회피법: 스펙을 쓰면서 COUNT를 피하려면 **커스텀 리포지터리 기능으로 직접 구현**해야 한다(다른 방법 없음)

**Specification(스펙)의 목적과 실제 구현 방식**
- 혼동: "spec 가 뭐지.." — 개념 미학습
- 핵심: 목적은 **검색 조건 조합 폭발 방지**. 조건마다 `find` 메서드를 만들면 조건 A/B/C에 대해 7개(조합 2^n)로 폭발한다. 검색 조건 자체를 객체로 만들어 `findAll(spec)` 하나로 처리
- 인터페이스: `boolean isSatisfiedBy(T agg)` — 리포지터리에 쓰면 `agg`는 **애그리거트 루트**, DAO에 쓰면 **검색 결과로 리턴할 데이터 객체**
- ★ 반드시 알 함정: 책의 `isSatisfiedBy()` 메모리 필터링은 **개념 설명용이고 실제 구현이 아니다**. 모든 애그리거트를 메모리에 올릴 수 없고 조회 성능이 붕괴한다. **실제로는 `toPredicate()`로 DB 쿼리의 WHERE 절로 변환**되어 DB가 필터링한다
