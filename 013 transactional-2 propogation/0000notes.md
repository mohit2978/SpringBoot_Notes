# Notes
![alt text](<013 transaction-2_250716_001828_1.jpg>) ![alt text](<013 transaction-2_250716_001828_2.jpg>) ![alt text](<013 transaction-2_250716_001828_3.jpg>) ![alt text](<013 transaction-2_250716_001828_4.jpg>) ![alt text](<013 transaction-2_250716_001828_5.jpg>) ![alt text](<013 transaction-2_250716_001828_6.jpg>) ![alt text](<013 transaction-2_250716_001828_7.jpg>) ![alt text](<013 transaction-2_250716_001828_8.jpg>) ![alt text](<013 transaction-2_250716_001828_9.jpg>) 

![alt text](<013 transaction-2_250716_001828_10.jpg>) 


Great question! Let me break it down with a real example.

Imagine you have two methods calling each other:

```java
@Transactional                          // Method A
public void placeOrder() {
    orderRepo.save(order);              // Step 1
    sendNotification();                 // calls Method B
    stockRepo.deduct();                 // Step 3
}

@Transactional(propagation = ???)       // Method B
public void sendNotification() {
    notificationRepo.save(notification); // Step 2
}
```

When `placeOrder()` calls `sendNotification()` — **what should Method B do with the transaction?** That's exactly what propagation decides.

---

## Visualising each propagation

```
placeOrder()       sendNotification()
──────────────     ──────────────────
```

**REQUIRED** *(default)* — B joins A's transaction. One single TX.
```
[──────────── TX1 ────────────────────]
  save order   save notification   deduct stock
```
If notification fails → **everything** rolls back including the order.

---

**REQUIRES_NEW** — B suspends A, opens its own TX, then A resumes.
```
[────── TX1 ──── PAUSED ──── RESUMED ──]
  save order             deduct stock

              [── TX2 ──]
               save notif   ← commits/rollbacks independently
```
If notification fails → **only TX2 rolls back**. Order and stock are unaffected.

---

**NESTED** — B runs inside A but with a save-point.
```
[──────────────── TX1 ──────────────────]
  save order  │SAVEPOINT│  save notif  │RELEASE│  deduct stock
```
If notification fails → rolls back **only to the save-point**. Order is still saved. If outer TX fails → everything rolls back including the notification.

---

**SUPPORTS** — B uses TX if one exists, otherwise runs without one.
```
Called from placeOrder()  →  joins TX1  (same as REQUIRED)
Called standalone         →  no TX at all
```

---

**NOT_SUPPORTED** — B always runs outside any TX.
```
[────── TX1 ──── PAUSED ──── RESUMED ──]
  save order             deduct stock

              [no TX]
               save notif   ← runs raw, no rollback possible
```

---

**MANDATORY** — B demands a TX must already exist, otherwise throws.
```
Called from placeOrder()  →  joins TX1  ✓
Called standalone         →  💥 IllegalTransactionStateException
```
Use this to enforce that a method should **never** be called without a transaction.

---

**NEVER** — B demands **no** TX should exist, otherwise throws.
```
Called from placeOrder()  →  💥 IllegalTransactionStateException
Called standalone         →  runs fine, no TX
```
Use this for operations that must **never** run inside a transaction (e.g. sending an email — you don't want it rolled back).

---

## The key mental model

Think of it as: **"When Method B is called, what should it do with the existing transaction from Method A?"**

| Propagation | Simple way to remember |
|---|---|
| `REQUIRED` | Share the TX — we're all in this together |
| `REQUIRES_NEW` | I need my own TX — don't involve me in yours |
| `NESTED` | I'll work inside your TX but with an escape hatch |
| `SUPPORTS` | TX or no TX — I'm fine either way |
| `NOT_SUPPORTED` | Keep me out of any TX |
| `MANDATORY` | You MUST have a TX before calling me |
| `NEVER` | You must NOT have a TX when calling me |

![alt text](<013 transaction-2_250716_001828_11.jpg>) ![alt text](<013 transaction-2_250716_001828_12.jpg>) ![alt text](<013 transaction-2_250716_001828_13.jpg>) ![alt text](<013 transaction-2_250716_001828_14.jpg>) ![alt text](<013 transaction-2_250716_001828_15.jpg>) ![alt text](<013 transaction-2_250716_001828_16.jpg>) ![alt text](<013 transaction-2_250716_001828_17.jpg>) ![alt text](<013 transaction-2_250716_001828_18.jpg>) ![alt text](<013 transaction-2_250716_001828_19.jpg>) ![alt text](<013 transaction-2_250716_001828_20.jpg>) ![alt text](<013 transaction-2_250716_001828_21.jpg>) ![alt text](<013 transaction-2_250716_001828_22.jpg>) ![alt text](<013 transaction-2_250716_001828_23.jpg>) ![alt text](<013 transaction-2_250716_001828_24.jpg>)