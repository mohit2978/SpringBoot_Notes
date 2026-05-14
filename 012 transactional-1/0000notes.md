# Notes
![alt text](<012transactional 1_241231_132930_250716_002319_1.jpg>) ![alt text](<012transactional 1_241231_132930_250716_002319_2.jpg>) ![alt text](<012transactional 1_241231_132930_250716_002319_3.jpg>) ![alt text](<012transactional 1_241231_132930_250716_002319_4.jpg>) ![alt text](<012transactional 1_241231_132930_250716_002319_5.jpg>)


## `@Transactional` — Complete Explanation

---

### What it does in one line:

> *"Wraps the method in a DB transaction — if anything fails, everything rolls back. If success, everything commits."*

---

### Basic example:

```java
@Transactional
public void transferMoney(Long fromId, Long toId, Double amount) {
    accountRepo.debit(fromId, amount);   // Step 1
    accountRepo.credit(toId, amount);    // Step 2
}
```

- Both steps succeed → **commit** ✅
- Step 2 fails → Step 1 also **rolls back** ✅
- Without `@Transactional` → Step 1 committed, Step 2 failed → **money lost** ❌

---

### How Spring implements it internally

Spring creates a **proxy** around your class:

```
You call → Proxy (opens transaction) → Your method → Proxy (commit/rollback)
```

- Before method: `BEGIN TRANSACTION`
- Method runs
- After method: `COMMIT` or `ROLLBACK`

This is why **self-invocation doesn't work**:
```java
// THIS WON'T WORK
public void methodA() {
    this.methodB(); // calling directly, bypasses proxy
}

@Transactional
public void methodB() { ... }
```
Because `this.methodB()` skips the proxy — no transaction opened.

---

### Rollback rules — most asked interview question

```java
@Transactional
public void doSomething() {
    // throws RuntimeException → ROLLBACK ✅
    // throws CheckedException → NO ROLLBACK ❌ (by default)
}
```

| Exception type | Default behavior |
|---|---|
| `RuntimeException` (unchecked) | Rollback |
| `Error` | Rollback |
| `Exception` (checked) | **NO rollback** |

To force rollback on checked exception:
```java
@Transactional(rollbackFor = Exception.class)
public void doSomething() throws Exception { }
```

To prevent rollback on specific exception:
```java
@Transactional(noRollbackFor = IllegalArgumentException.class)
```

---

### Propagation — second most asked

Defines what happens when a `@Transactional` method calls **another** `@Transactional` method.

```java
methodA() {        // has transaction T1
    methodB();     // what happens here?
}
```

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Use existing T1, if none create new |
| `REQUIRES_NEW` | **Suspend T1, create new T2** — independent transaction |
| `NESTED` | Create savepoint inside T1 — can rollback partially |
| `MANDATORY` | Must have existing transaction, else exception |
| `NEVER` | Must NOT have transaction, else exception |
| `SUPPORTS` | Use if exists, proceed without if not |
| `NOT_SUPPORTED` | Suspend existing, run without transaction |

**Most important to know: REQUIRED vs REQUIRES_NEW**

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void sendNotification() {
    // runs in its OWN transaction
    // even if outer transaction rolls back
    // notification still gets saved
}
```

Real use case: **audit logging** — even if main transaction fails, you want the audit log committed.

---

### Isolation levels — third most asked

Defines what a transaction can **see from other concurrent transactions**.

| Isolation | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | ✅ possible | ✅ possible | ✅ possible |
| `READ_COMMITTED` | ❌ prevented | ✅ possible | ✅ possible |
| `REPEATABLE_READ` | ❌ | ❌ prevented | ✅ possible |
| `SERIALIZABLE` | ❌ | ❌ | ❌ prevented |

- **Dirty read** — reading uncommitted data from another transaction
- **Non-repeatable read** — same query gives different result within same transaction
- **Phantom read** — new rows appear between two reads in same transaction

Default in Spring = whatever the **DB default is** (MySQL = REPEATABLE_READ).

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void myMethod() { }
```

---

### Common interview gotchas:

**1. `@Transactional` on private method — does it work?**
→ ❌ No. Proxy can't intercept private methods. Must be public.

**2. Exception caught inside method — does it rollback?**
```java
@Transactactional
public void doSomething() {
    try {
        riskyOperation();
    } catch(Exception e) {
        // swallowed — NO ROLLBACK
        // Spring never sees the exception
    }
}
```
→ ❌ No rollback — exception was caught before proxy saw it.

**3. `@Transactional` on class vs method**
```java
@Transactional          // applies to ALL methods
public class MyService {

    @Transactional(propagation = REQUIRES_NEW)  // overrides for this method
    public void specificMethod() { }
}
```

---

### One line summary:

> *"`@Transactional` uses Spring AOP proxy to wrap methods in a DB transaction. Rolls back on RuntimeException by default. REQUIRED propagation reuses existing transaction, REQUIRES_NEW creates independent one. Private methods and self-invocation won't work because they bypass the proxy."*

---
