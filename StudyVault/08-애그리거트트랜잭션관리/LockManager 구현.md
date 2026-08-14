---
source_pdf: 도메인 주도 개발 시작하기.pdf
part: 8장 §8.4.1~8.4.2 (p.265~273)
keywords: lockmanager, lockid, jdbctemplate, unique-index, expiration
---

# LockManager 인터페이스와 DB 구현 (★★)

#ddd #transaction #offline-lock #concept

## Overview Table (한눈에 비교)

| 항목 | 핵심 내용 |
|------|-----------|
| 인터페이스 | **`LockManager`** — 네 개의 메서드 |
| `tryLock(type, id)` | **잠글 대상 타입과 식별자**를 받아 **`LockId`** 리턴 |
| `LockId` | 잠금을 식별하는 값. **해제·확인·연장에 사용**하므로 **어딘가에 보관해야 한다** |
| DB 테이블 | `locks(type, id, lockid, expiration_time)` |
| 주요키 | **`(type, id)`** — 동시에 두 사용자가 같은 대상을 잠그는 것을 방지 |
| 유니크 인덱스 | **`lockid`** — 각 잠금마다 새로운 LockId를 사용 |
| ★ 필수 순서 | 기능 실행 전에 **반드시 `checkLock()`으로 유효성 확인** |

## §8.4.1 `LockManager` 인터페이스 (리스트 8.1)

```java
package com.myshop.lock;

public interface LockManager {
    LockId tryLock(String type, String id) throws LockException;

    void checkLock(LockId lockId) throws LockException;

    void releaseLock(LockId lockId) throws LockException;

    void extendLockExpiration(LockId lockId, long inc) throws LockException;
}
```

| 메서드 | 역할 |
|--------|------|
| **`tryLock`** | **잠금 선점 시도**. `type`(잠글 대상 타입) + `id`(식별자) → `LockId` 리턴 |
| **`checkLock`** | **잠금이 유효한지 확인** |
| **`releaseLock`** | **잠금 해제** |
| **`extendLockExpiration`** | **잠금 유효 시간 연장** |

> 예: 식별자가 10인 `Article`에 대한 잠금을 구하려면
> `tryLock("domain.Article", "10")` 처럼 호출한다.

## `LockId` 클래스 (리스트 8.2)

```java
package com.myshop.lock;

public class LockId {
    private String value;

    public LockId(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
```

> [!important]
> `tryLock()`은 **잠금을 식별할 때 사용할 `LockId`를 리턴**한다. 각 잠금마다 **고유 식별자**를 갖는다.
> 일단 잠금을 구하면 **해제하거나, 유효한지 검사하거나, 유효 시간을 늘릴 때 `LockId`를 사용**한다.
> **`LockId`가 없으면 잠금을 해제할 수 없으므로 `LockId`를 어딘가에 보관해야 한다.**

## 사용 흐름

### 1) 잠금 선점 — 서비스가 LockId를 리턴

```java
// 서비스: 서비스는 잠금 ID를 리턴한다.
public DataAndLockId getDataWithLock(Long id) {
    // 1. 오프라인 선점 잠금 시도
    LockId lockId = lockManager.tryLock("data", id);
    // 2. 기능 실행
    Data data = someDao.select(id);
    return new DataAndLockId(data, lockId);
}
```

```java
// 컨트롤러: 서비스가 리턴한 잠금ID를 모델로 뷰에 전달한다.
@RequestMapping("/some/edit/{id}")
public String editForm(@PathVariable("id") Long id, ModelMap model) {
    DataAndLockId dl = dataService.getDataWithLock(id);
    model.addAttribute("data", dl.getData());
    // 3. 잠금 해제에 사용할 LockId를 모델에 추가
    model.addAttribute("lockId", dl.getLockId());
    return "editForm";
}
```

> [!warning]
> **잠금을 선점하는 데 실패하면 `LockException`이 발생**한다.
> 이때는 **"다른 사용자가 데이터를 수정 중이니 나중에 다시 시도하라"** 는 안내 화면을 보여주면 된다.

### 2) 폼에서 LockId 전송

```html
<form th:action="@{/some/edit/{id}(id=${data.id})}" method="post">
    <input type="hidden" name="lid" th:value="${lockId.value}">
    ...
</form>
```

### 3) 잠금 확인 → 기능 실행 → 해제

```java
// 서비스: 잠금을 해제한다.
public void edit(EditRequest editReq, LockId lockId) {
    // 1. 잠금 선점 확인
    lockManager.checkLock(lockId);
    // 2. 기능 실행
    ...
    // 3. 잠금 해제
    lockManager.releaseLock(lockId);
}
```

```java
// 컨트롤러: 서비스를 호출할 때 잠금ID를 함께 전달
@RequestMapping(value = "/some/edit/{id}", method = RequestMethod.POST)
public String edit(@PathVariable("id") Long id,
                   @ModelAttribute("editReq") EditRequest editReq,
                   @RequestParam("lid") String lockIdValue) {
    editReq.setId(id);
    someEditService.edit(editReq, new LockId(lockIdValue));
    model.addAttribute("data", data);
    return "editSuccess";
}
```

## ★ `checkLock()`을 반드시 먼저 실행하는 이유

> [!danger] (p.268)
> 서비스 코드를 보면 **`lockManager.checkLock()`을 가장 먼저 실행**하는데,
> 잠금을 선점한 이후에 실행하는 기능은 다음 상황을 고려하여 **반드시 주어진 `LockId`를 갖는 잠금이 유효한지 확인**해야 한다.
>
> - **잠금 유효 시간이 지났으면 이미 다른 사용자가 잠금을 선점**한다.
> - **잠금을 선점하지 않은 사용자가 기능을 실행했다면 기능 실행을 막아야 한다.**

## §8.4.2 DB를 이용한 구현

### 테이블 (리스트 8.3, MySQL)

```sql
create table locks (
    `type` varchar(255),
    id varchar(255),
    lockid varchar(255),
    expiration_time datetime,
    primary key (`type`, id)
) character set utf8;

create unique index locks_idx ON locks (lockid);
```

| 설계 | 이유 |
|------|------|
| **`(type, id)`를 주요키** | **동시에 두 사용자가 특정 타입 데이터에 대한 잠금을 구하는 것을 방지** |
| **`lockid`를 유니크 인덱스** | 각 잠금마다 **새로운 LockId를 사용**하므로 |
| **`expiration_time` 칼럼** | **잠금 유효 시간**을 보관 |

```sql
insert into locks values ('Order', '1', '생성한lockid', '2016-03-28 09:10:00');
```

### `LockData` 클래스 (리스트 8.4)

```java
package com.myshop.lock;

public class LockData {
    private String type;
    private String id;
    private String lockId;
    private long timestamp;

    public LockData(String type, String id, String lockId, long timestamp) {
        this.type = type;
        this.id = id;
        this.lockId = lockId;
        this.timestamp = timestamp;
    }

    public String getType() { return type; }
    public String getId() { return id; }
    public String getLockId() { return lockId; }
    public long getTimestamp() { return timestamp; }

    public boolean isExpired() {
        return timestamp < System.currentTimeMillis();
    }
}
```

> [!note] 필드명은 `timestamp`
> 책 리스트 8.4는 `expirationTime`으로 표기하지만 **실제 `ddd-start2/`는 `timestamp`** 다.
> 다만 **의미는 '만료 시각'** 이다 — `isExpired()`가 `timestamp < 현재시각`으로 판단하므로,
> 이 값에는 **잠금 생성 시각이 아니라 만료 시각**(`현재 + lockTimeout`)이 들어간다.
> 이름만 보면 오해하기 쉬운 지점이다. → `lock/LockData.java`

### `tryLock` 구현 (리스트 8.5, JdbcTemplate)

```java
@Component
public class SpringLockManager implements LockManager {
    private int lockTimeout = 5 * 60 * 1000;
    private JdbcTemplate jdbcTemplate;

    private RowMapper<LockData> lockDataRowMapper = (rs, rowNum) ->
            new LockData(rs.getString(1), rs.getString(2),
                         rs.getString(3), rs.getTimestamp(4).getTime());

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @Override
    public LockId tryLock(String type, String id) throws LockException {
        checkAlreadyLocked(type, id);
        LockId lockId = new LockId(UUID.randomUUID().toString());
        locking(type, id, lockId);
        return lockId;
    }

    private void checkAlreadyLocked(String type, String id) {
        List<LockData> locks = jdbcTemplate.query(
                "select * from locks where type = ? and id = ?",
                lockDataRowMapper, type, id);
        Optional<LockData> lockData = handleExpiration(locks);
        if (lockData.isPresent()) throw new AlreadyLockedException();
    }

    // 유효 시간이 지난 데이터는 삭제하고 빈 Optional을 리턴
    private Optional<LockData> handleExpiration(List<LockData> locks) {
        if (locks.isEmpty()) return Optional.empty();
        LockData lockData = locks.get(0);
        if (lockData.isExpired()) {
            jdbcTemplate.update(
                    "delete from locks where type = ? and id = ?",
                    lockData.getType(), lockData.getId());
            return Optional.empty();
        } else {
            return Optional.of(lockData);
        }
    }

    private void locking(String type, String id, LockId lockId) {
        try {
            int updatedCount = jdbcTemplate.update(
                    "insert into locks values (?, ?, ?, ?)",
                    type, id, lockId.getValue(),
                    new Timestamp(getExpirationTime()));
            if (updatedCount == 0) throw new LockingFailException();
        } catch (DuplicateKeyException e) {
            throw new LockingFailException(e);
        }
    }

    private long getExpirationTime() {
        return System.currentTimeMillis() + lockTimeout;
    }
}
```

> [!tip] 전체 흐름
> `tryLock()` → **`checkAlreadyLocked()`로 이미 선점됐는지 확인** → **`locking()`으로 잠금 선점**
>
> 동일한 주요 키나 `lockid`를 가진 데이터가 이미 존재해서 **`DuplicateKeyException`이 발생하면 `LockingFailException`** 을 발생시킨다.

### 나머지 구현 (리스트 8.6)

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
@Override
public void checkLock(LockId lockId) throws LockException {
    Optional<LockData> lockData = getLockData(lockId);
    if (!lockData.isPresent()) throw new NoLockException();
}

private Optional<LockData> getLockData(LockId lockId) {
    List<LockData> locks = jdbcTemplate.query(
            "select * from locks where lockid = ?",
            lockDataRowMapper, lockId.getValue());
    return handleExpiration(locks);
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
@Override
public void extendLockExpiration(LockId lockId, long inc) {
    Optional<LockData> lockDataOpt = getLockData(lockId);
    LockData lockData =
            lockDataOpt.orElseThrow(() -> new NoLockException());
    jdbcTemplate.update(
            "update locks set expiration_time = ? where type = ? AND id = ?",
            new Timestamp(lockData.getExpirationTime() + inc),
            lockData.getType(), lockData.getId());
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
@Override
public void releaseLock(LockId lockId) throws LockException {
    jdbcTemplate.update(
            "delete from locks where lockid = ?", lockId.getValue());
}
```

> [!note] `Propagation.REQUIRES_NEW`
> 잠금 관련 처리는 **호출한 트랜잭션과 분리된 새 트랜잭션**에서 실행되어야 하므로 `REQUIRES_NEW`를 사용한다.

## Exam/Test Patterns (시험 빈출 패턴)

| Scenario/Keyword | Answer |
|-------------------|--------|
| "`LockManager`의 네 메서드" | **`tryLock` / `checkLock` / `releaseLock` / `extendLockExpiration`** |
| "`tryLock`의 파라미터" | **잠글 대상 타입 + 식별자** |
| "`tryLock`의 리턴" | **`LockId`** |
| "`LockId`를 보관해야 하는 이유" | 없으면 **잠금을 해제할 수 없다** |
| "테이블의 주요키" | **`(type, id)`** — 동시 선점 방지 |
| "`lockid`에 거는 인덱스" | **유니크 인덱스** |
| "기능 실행 전에 반드시 할 일" | **`checkLock()`으로 유효성 확인** |
| "확인이 필요한 두 이유" | **유효 시간 만료 후 타인이 선점** / **선점하지 않은 사용자의 실행 차단** |
| "중복 키 발생 시" | `DuplicateKeyException` → **`LockingFailException`** |

## Related Notes

- [[오프라인 선점 잠금]] — 개념과 유효 시간
- [[선점 잠금]] / [[비선점 잠금]]
- [[애그리거트와 트랜잭션]]
- [[표현 영역에 의존하지 않기와 트랜잭션 처리]] — 6장
- [[연습문제 - 애그리거트 트랜잭션 관리]]
