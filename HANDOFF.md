# HANDOFF — DDD 학습 세션 인계

작성: 2026-08-14 · 이전 세션 종료 사유: **어시스턴트 환각으로 신뢰 상실 → 컨텍스트 초기화**

---

## 0. 다음 세션이 가장 먼저 읽어야 할 것

이 문서는 **신뢰할 수 없는 세션이 남긴 인계문**이다. 아래 §4의 "검증 등급"을 반드시 확인하고, **B/C 등급 서술은 사용 전에 직접 재검증**할 것.

---

## 1. 무엇을 하고 있었나

**사용자**: 『도메인 주도 개발 시작하기』(최범균)로 DDD를 학습 중인 현업 개발자.
**도구**: `.claude/skills/tutor` 스킬 — 서술형 퀴즈 → 채점 → 개념 파일/대시보드 갱신.
**목표**: 정답 맞히기가 아니라 **"왜 그런지 설명할 수 있는 상태"** 도달.

### 사용자 특성 (중요)
- 개발 경험이 있어 **감으로 정답을 맞히지만 근거를 말하지 못하는** 상태가 반복된다. 그 근거를 채우는 것이 학습 목표.
- **모르는 것을 정확히 "모르겠다"고 말한다.** 이 정직함이 진단의 핵심 자산이므로 존중할 것.
- 설명이 틀리면 **직접 지적한다** (실제로 이번 세션 종료의 계기).

### 진행 규칙 (사용자가 요청해 확립됨)
1. **라운드당 1문항.** (4 → 2 → 1로 두 번 줄여달라고 요청받음)
2. **객관식 금지, 서술형만.** 객관식은 찍을 수 있어 실력 측정이 안 된다.
3. **틀린 문제는 이해될 때까지 다음으로 넘어가지 않는다.**
4. 사용자가 던진 질문은 `StudyVault/00-Dashboard/내 질문 노트.md`에 **최신순 누적**.
5. **사용자가 "끝났다"고 말하기 전까지 세션을 마무리하지 말 것.** (마무리를 시도했다가 제지당함)

---

## 2. 파일 위치

| 파일 | 용도 |
|---|---|
| `StudyVault/00-Dashboard/학습 대시보드.md` | 영역별 숙련도 + 미해결 개념 목록 |
| `StudyVault/00-Dashboard/내 질문 노트.md` | **사용자가 던진 질문 9개**만 최신순 (Q9→Q1) |
| `StudyVault/concepts/{01,03,04,05,10}-*.md` | 영역별 개념 트래커 + 오답 메모 |
| `StudyVault/00-Dashboard/Exam Traps.md` | **기존 자료** (내가 만든 것 아님, 신뢰 가능) |
| `StudyVault/{01~12}-*/` | **기존 학습 노트** (내가 만든 것 아님, 신뢰 가능) |
| `ddd-start2/` | 책의 실제 예제 코드 — **1차 검증 소스** |
| `.claude/skills/tutor/SKILL.md` | 튜터 스킬 (이번 세션에 질문 로그 단계 추가함) |

---

## 3. 현재 진도

### 대시보드 수치 (2026-08-14 기준)

| 영역 | 정답 | 오답 | 정답률 | 수준 |
|---|---|---|---|---|
| 01-도메인모델 | 6 | 10 | 38% | 🟥 |
| 03-애그리거트 | 7 | 3 | 70% | 🟩 |
| 04-리포지터리와모델구현 | 0 | 2 | 0% | 🟥 |
| 05-스프링데이터JPA조회 | 0 | 3 | 0% | 🟥 |
| 10-이벤트 | 0 | 2 | 0% | 🟥 |
| **전체** | **13** | **20** | **39%** | 🟥 |

02·06·07·08·09·11·12장은 **미측정**.

> 수치가 낮아 보이는 것은 실력 하락이 아니라 **개념 수가 8개 → 23개로 늘어난 결과**다. 진단이 정밀해진 정상 신호.

### 라운드 이력

| 라운드 | 내용 | 결과 |
|---|---|---|
| 1 | 전 영역 진단 4문항 | 10% — 1장부터 막힘 |
| 2 | 1장+3장 기초 | 엔티티/밸류, set 메서드 해결 |
| 3 | 애그리거트 경계 (설문/쿠폰) | **4기준 × 2쌍 = 8개 전부 정답** |
| 4 | 호텔 — 밸류 추출 | "밸류를 왜/언제 빼는지 개념이 없다" 확인 |
| 5 | 온라인 강의 — 두 축 분리 | 축1·축2 분리 성공 |
| + | 사용자 질문 심층 (Q7·Q8·Q9) | 진행 중 |

### 해결된 개념 (🟢 11개)
엔티티/밸류 판단 기준 · equals() 규칙 + 실무 판단 · set 메서드 안티패턴 · "A가 B를 갖는다" 함정 · 애그리거트 4체크리스트 · 라이프 사이클 · "함께 생성"의 정밀 기준 · 애그리거트 내부 엔티티 · 두 축 분리 · 밸류 추출 3신호

### 미해결 개념 (🔴 12개) — 다음 우선순위

**1순위 — 반복 오류**
- `cardinality(1:N)를 판정 기준으로 쓰는 습관` — **2회 반복** (1라운드, 5라운드)
- 밸류 "추출" vs 엔티티 "추적" 방향 차이

**2순위 — 설명만 하고 테스트 안 함**
- 객체↔관계형 대응 4패턴 (Q9에서 방금 설명, **미테스트**)
- 밸류 컬렉션 매핑, `@Embeddable` vs `@Entity` clear() 성능
- 03-애그리거트: 판정 결과의 코드 반영 / 변경 주체 = 역할이지 코드 진입점이 아님
- 01: DB 스키마 ≠ 도메인 모델 / private setter 예외

**3순위 — 0% 또는 미측정**
- 05-스프링데이터JPA조회, 10-이벤트
- 02·06·07·08·09·11·12장 — 한 번도 안 봄

---

## 4. ⚠️ 검증 등급 — 반드시 확인할 것

이전 세션이 vault에 쓴 서술을 신뢰도별로 분류한다.

### A등급 — 실제 코드/DDL을 직접 읽고 확인함 (신뢰 가능)

명령을 실행해 출력을 확인한 사실들:

- **`ddd-start2` 도메인 클래스 24개 중 `equals()` 구현은 7개** — `OrderNo`, `MemberId`, `ProductId`, `CategoryId`, `Money`, `Email`, `Orderer`
- **애그리거트 루트 5개(`Order`, `Member`, `Product`, `Article`, `Category`) 전부 `equals()` 없음**
- **`Address`, `Receiver`, `ShippingInfo`, `OrderLine`, `Password`도 `equals()` 없음**
- `OrderNo`/`MemberId`/`ProductId`/`CategoryId`는 `implements Serializable` + `@EmbeddedId`로 사용됨
- **`order_line` 테이블에 PK가 없음** — `create index order_line_idx ON order_line (order_number, line_idx)` 뿐
- `article_content`에 `id int not null primary key` 있음 / `image`에 `image_id auto_increment primary key` 있음
- `purchase_order` 컬럼 13개 (`order_number, version, orderer_id, orderer_name, total_amounts, shipping_zip_code, shipping_addr1, shipping_addr2, shipping_message, receiver_name, receiver_phone, state, order_date`)
- **`member` 테이블 컬럼은 `member_id, name, password, blocked, emails` 5개뿐** (주소 컬럼 없음)
- `Article`이 `@SecondaryTable(name="article_content", pkJoinColumns=@PrimaryKeyJoinColumn(name="id"))` 사용
- `ArticleContent`는 `@Embeddable`, 식별자 필드 없음
- `Order.orderLines`는 `@ElementCollection` + `@CollectionTable(name="order_line")` + `@OrderColumn(name="line_idx")`
- `Money`는 `MoneyConverter`로 `Order.totalAmounts`, `OrderLine.price`, `OrderLine.amounts`, `Product.price`, `ProductData.price`에 매핑
- **`Address`는 `ShippingInfo`에서만 사용됨** (`common/model/`에 위치하지만 다른 사용처 없음)
- `ShippingInfo`가 `@AttributeOverrides`로 `Address`를 `shipping_zip_code/addr1/addr2`에 매핑
- **`Address` 클래스에는 검증도 도메인 메서드도 없음** — 필드 3개 + `public` 기본 생성자 + getter 3개
- 주소 검증은 응용 계층 `OrderRequestValidator`가 수행
- `Password.match(String)` 존재 (`equals()` 대신)
- 도메인 `.equals()` 호출은 2곳뿐이며 둘 다 ID 문자열 비교: `order/ui/MyOrderController.java:38`, `order/infra/domain/SecurityCancelPolicy.java:23`

### B등급 — 기존 vault 노트에서 인용 (2차 출처, 원서 대조 권장)

`StudyVault/{01~12}-*/`와 `Exam Traps.md`는 **이전 세션이 만든 것이 아니라 원래 있던 사용자 자료**다. 여기서 인용한 내용:

- COUNT 쿼리 규칙 (`findAll(Spec, Pageable)` + `List`도 COUNT 실행) — `05-스프링데이터JPA조회/페이징 처리하기.md`
- `@Entity` clear() = SELECT 1 + DELETE N vs `@Embeddable` = DELETE 1 — `04-리포지터리와모델구현/밸류 컬렉션 매핑.md`
- 애그리거트 4체크리스트, "다수의 애그리거트가 엔티티 1개" 경험칙 — `03-애그리거트/애그리거트.md`
- 자동 증가 칼럼 offset 누락 문제 — `10-이벤트/이벤트 저장소를 이용한 비동기 처리.md`
- "매핑되는 테이블의 식별자를 구성요소의 식별자와 동일시하면 안 된다" — `Exam Traps.md`

이 노트들 자체는 신뢰도가 높아 보이나, **원서 페이지와 대조하면 더 확실**하다.

### C등급 — 어시스턴트의 일반 지식/추론 (재검증 필요)

- JPA 1차 캐시가 같은 영속성 컨텍스트에서 동일 인스턴스를 보장한다 — 일반 JPA 지식, 코드로 검증 안 함
- `@EmbeddedId` 클래스에 JPA 스펙이 `Serializable`+`equals`+`hashCode`를 요구한다 — 일반 지식 (관찰된 코드와 정합하지만 스펙 문서 대조 안 함)
- **`article_content`를 별도 테이블로 뺀 이유가 "본문이 크고 목록 조회 때 불필요해서"** — **추론**. 책이 그렇게 설명하는지 확인 안 함
- 밸류 추출 "3신호"(함께 변한다/이름이 붙는다/로직이 있다) — 책의 서술을 어시스턴트가 재구성한 프레임. **책의 원래 표현이 아닐 수 있음**
- 엔티티/밸류 "판별 질문 3개"(통째 교체/참조됨/개별 상태) — 마찬가지로 어시스턴트가 만든 프레임
- 호텔 예약, 온라인 강의, 설문/쿠폰 등 **모든 퀴즈 시나리오는 교육용 창작**이며 책에 없음

### ❌ 확인된 오류 2건 (이미 정정 완료)

1. **`member` 테이블에 주소 컬럼 3개가 있고 `Address`가 3개 테이블에 저장된다** → **완전한 허위.** 검증 없이 지어냄.
   - vault 파일 오염 여부 grep 확인 결과 **유입 없음** (채팅에만 존재)
2. **`Address`가 검증 로직과 `isSameRegion()`을 갖는다** → **허위.** 교육용으로 지어낸 이상적 버전을 실제 코드로 서술.
   - 오염 위치 2곳 발견 후 정정 완료: `concepts/04-리포지터리와모델구현.md`, `00-Dashboard/내 질문 노트.md`
   - `concepts/01-도메인모델.md`의 "얻는 것 4가지"에도 일반론임을 명시하는 주석 추가

**교훈**: 코드 예시를 들 때 **실제 파일을 읽은 것인지 지어낸 것인지 반드시 구분해 표시할 것.** "예를 들면 이렇게 만들 수 있다"와 "실제 코드가 이렇다"는 완전히 다르다.

---

## 5. 다음 세션 권장 행동

1. **먼저 `StudyVault/00-Dashboard/학습 대시보드.md`와 `내 질문 노트.md`를 읽을 것.**
2. 사용자에게 **§4의 C등급 프레임(3신호, 판별 질문 3개)을 계속 쓸지 확인**할 것. 유용하지만 책의 표현이 아님을 밝히고 동의를 받는 편이 안전하다.
3. 다음 문제는 **1순위 미해결 개념(cardinality 반복 오류)** 부터.
4. **코드/스키마를 인용할 때는 반드시 `ddd-start2/`를 직접 읽고, 읽지 않은 것은 "가상의 예"라고 명시할 것.**
5. 사용자가 명시적으로 종료 의사를 밝히기 전까지 세션을 마무리하지 말 것.

## 6. 진행 중이던 것

Q9(객체↔관계형 대응) 설명을 마치고 사용자 확인을 기다리던 중이었다. 다음 선택지를 제시한 상태:
- 이 개념 확인 문제 (`Orderer`/`ArticleContent` 같은 상황 새로 하나)
- 다른 주제로 환기
- 더 파기 (`@AttributeOverride`가 왜 필요한지, `Password`가 밸류인데 왜 컬럼 하나인지)

**Q9 내용은 §4 A등급 사실에 기반하지만, "왜 테이블을 나눴나"의 이유(C등급)는 재검증 대상이다.**
