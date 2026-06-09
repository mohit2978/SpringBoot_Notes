# Notes
![alt text](016_async_2_250716_002345_1.jpg) ![alt text](016_async_2_250716_002345_2.jpg) ![alt text](016_async_2_250716_002345_3.jpg)



## The core problem

`@Transactional` and `@Async` fight each other because of one fundamental reason:

```
@Transactional  →  binds the transaction to the CURRENT THREAD
@Async          →  moves execution to a DIFFERENT THREAD

Transaction from Thread A cannot travel to Thread B.
```

So if you do this naively:

```java
@Service
public class OrderService {

    @Async
    @Transactional        // ← YOU THINK this wraps the async method
    public void processOrder(Order order) {
        orderRepo.save(order);
        paymentRepo.save(payment);
    }
}
```

What actually happens:

```
Main Thread                    Async Thread (new)
-----------                    ------------------
calls processOrder()
@Async fires →                 processOrder() runs here
                               @Transactional tries to bind to THIS thread
                               → new transaction opens on async thread ✓
                               → but caller has NO idea if it committed or failed
                               → caller cannot rollback based on async result
```

So `@Async` + `@Transactional` on the same method does technically work — but the transaction runs on the async thread, completely disconnected from the caller. The caller cannot participate in the transaction at all. This surprises most developers.

---

## Problem 1 — Caller cannot react to async transaction failure

```java
@Service
public class OrderService {

    @Async
    @Transactional
    public void processOrder(Order order) {
        orderRepo.save(order);
        throw new RuntimeException("Payment failed!"); // rolls back async TX
    }
}

@Service
public class CheckoutService {

    @Transactional          // TX1 on main thread
    public void checkout(Order order) {
        cartRepo.clear();          // clears cart in TX1
        orderService.processOrder(order);  // fires async, returns immediately
        // checkout has NO idea processOrder will fail
        // cart is cleared, but order never saved → inconsistent!
    }
}
```

Output:
```
Main thread:   cartRepo.clear()       → cart cleared in TX1
Main thread:   processOrder() fired   → returns immediately (Future)
Main thread:   TX1 COMMIT             → cart is cleared in DB ✓

Async thread:  orderRepo.save()       → saved
Async thread:  RuntimeException!
Async thread:  TX2 ROLLBACK           → order NOT saved

Final DB state:
  cart    → empty   ✓
  order   → missing ✗   ← inconsistent!
```

The caller committed already. It has no way to rollback its own TX based on what the async thread does later.

---

## Problem 2 — LazyInitializationException in async thread

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items;   // lazy loaded
}

@Async
@Transactional
public void processOrder(Order order) {
    // Order was loaded in CALLER's transaction on main thread
    // That transaction is already closed by the time async thread runs
    order.getItems().size();  // ← BOOM! No session on this thread
}
```

Output:
```
org.hibernate.LazyInitializationException:
  failed to lazily initialize a collection of role: Order.items,
  could not initialize proxy - no Session
```

Because the Hibernate session that loaded `order` belonged to the caller's thread. The async thread has no session for that object.

---

## The correct patterns

---

### Pattern 1 — Keep them fully separate (simplest and safest)

Do the transactional work first, commit it, then fire async for non-critical work.

```java
@Service
public class OrderService {

    // Step 1 — Transactional, runs on main thread, commits fully
    @Transactional
    public Order saveOrder(OrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        paymentRepo.save(new Payment(order));
        return order;
        // TX commits here — order is safely in DB
    }

    // Step 2 — Async, runs on separate thread, no transaction needed
    @Async
    public void sendConfirmationEmail(Long orderId) {
        // Load fresh from DB — has its own session
        Order order = orderRepo.findById(orderId).orElseThrow();
        emailService.send(order.getEmail(), "Order confirmed: " + orderId);
        System.out.println("Email sent for order " + orderId);
    }
}

@Service
public class CheckoutService {

    public void checkout(OrderRequest req) {
        // 1. Save synchronously — waits for commit
        Order order = orderService.saveOrder(req);

        // 2. Fire async AFTER commit — safe!
        orderService.sendConfirmationEmail(order.getId());
    }
}
```

Output:
```
Main thread:   TX1 started
Main thread:   order saved
Main thread:   payment saved
Main thread:   TX1 COMMITTED  ← fully safe in DB before async fires

Async thread:  loaded order 42 from DB fresh
Async thread:  email sent to user@example.com
```

This is the recommended pattern for 90% of cases.

---

### Pattern 2 — Async method has its own independent `@Transactional`

When the async work itself needs DB writes and should be transactional on its own.

```java
@Service
public class NotificationService {

    // This has its OWN transaction — independent of caller
    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAndNotify(Long orderId) {
        // Load fresh — don't accept entity from caller
        Order order = orderRepo.findById(orderId).orElseThrow();

        // Write to DB — in its own TX
        AuditLog log = new AuditLog(orderId, "Order processed");
        auditRepo.save(log);

        emailService.send(order.getEmail(), "Your order is confirmed");
        System.out.println("Audit log saved, email sent");
    }
}

@Service
public class OrderService {

    @Transactional
    public void processOrder(OrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        // TX1 commits here

        // Pass only the ID — not the entity object!
        notificationService.logAndNotify(order.getId());
    }
}
```

Key rule — always pass the **ID**, never the entity object to an async method:

```java
// WRONG — passing entity causes LazyInitializationException
notificationService.logAndNotify(order);

// CORRECT — pass ID, let async thread load it fresh
notificationService.logAndNotify(order.getId());
```

Output:
```
Main thread:   TX1 started
Main thread:   order saved (id=42)
Main thread:   TX1 COMMITTED
Main thread:   logAndNotify(42) fired → returns immediately

Async thread:  TX2 started (REQUIRES_NEW, independent)
Async thread:  loaded order 42 fresh from DB
Async thread:  audit log saved
Async thread:  email sent
Async thread:  TX2 COMMITTED
```

---

### Pattern 3 — Using `CompletableFuture` with `@Async`

When you need the result of the async operation back.

```java
@Service
public class ReportService {

    @Async
    @Transactional(readOnly = true)   // read-only TX on async thread
    public CompletableFuture<List<Order>> fetchOrdersAsync(Long userId) {
        List<Order> orders = orderRepo.findByUserId(userId);
        System.out.println("Fetched " + orders.size() + " orders on async thread");
        return CompletableFuture.completedFuture(orders);
    }
}

@Service
public class DashboardService {

    public void buildDashboard(Long userId) throws Exception {
        System.out.println("Firing async fetch...");

        CompletableFuture<List<Order>> future =
            reportService.fetchOrdersAsync(userId);

        // do other work while async runs...
        System.out.println("Doing other work on main thread...");

        // wait for result
        List<Order> orders = future.get();
        System.out.println("Got " + orders.size() + " orders back");
    }
}
```

Output:
```
Main thread:   Firing async fetch...
Main thread:   Doing other work on main thread...
Async thread:  Fetched 5 orders on async thread   ← runs in parallel
Main thread:   Got 5 orders back
```

---

## The golden rules — summary

```
Rule 1 — Never pass an entity object to an async method
         Always pass the ID and reload inside the async method

Rule 2 — Commit your transaction BEFORE firing async
         Never rely on async result to rollback main TX

Rule 3 — Async method should use REQUIRES_NEW if it needs its own TX
         It gets a completely fresh transaction and session

Rule 4 — Never expect async failures to affect main thread TX
         They are on different threads — different transactions

Rule 5 — @Async on same class does not work (same as @Transactional)
         Must be on a different Spring bean to be intercepted by proxy
```

---

## Quick decision chart

```
Do you need result of async work in your TX?
    YES → Don't use @Async. Use synchronous @Transactional only.
    NO  → Use @Async safely after your TX commits

Does async work need DB writes?
    YES → @Async + @Transactional(REQUIRES_NEW) + pass only ID
    NO  → @Async alone, no transaction needed

Is async work critical (failure = rollback main TX)?
    YES → Don't use @Async. Keep it synchronous.
    NO  → @Async is fine (emails, notifications, logging)
```

The bottom line: `@Async` is for fire-and-forget work that does not affect the correctness of your main transaction — things like emails, push notifications, audit logs, and report generation.


 ![alt text](016_async_2_250716_002345_4.jpg) ![alt text](016_async_2_250716_002345_5.jpg) ![alt text](016_async_2_250716_002345_6.jpg) ![alt text](016_async_2_250716_002345_7.jpg) ![alt text](016_async_2_250716_002345_8.jpg) ![alt text](016_async_2_250716_002345_9.jpg) ![alt text](016_async_2_250716_002345_10.jpg) ![alt text](016_async_2_250716_002345_11.jpg) ![alt text](016_async_2_250716_002345_12.jpg)