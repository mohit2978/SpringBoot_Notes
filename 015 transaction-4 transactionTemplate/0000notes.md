# TransactionTemplate vs @Transactional



## 1. What is `@Transactional`?

`@Transactional` is a **declarative** way to manage transactions.
Spring uses **AOP (Aspect-Oriented Programming)** under the hood — it wraps your method in a **proxy** that:
1. Begins a transaction **before** the method
2. Commits on success
3. Rolls back on `RuntimeException` (by default)

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        // If any RuntimeException thrown here → auto rollback
    }
}
```

### How Spring handles it internally (AOP Proxy)

```
Client calls orderService.placeOrder()
        ↓
Spring AOP Proxy intercepts the call
        ↓
Proxy → BEGIN TRANSACTION
        ↓
Proxy → calls actual placeOrder() method
        ↓
Method executes
        ↓
No exception → COMMIT
Exception    → ROLLBACK
```

---

## 2. Limitations of `@Transactional` — WHY we need TransactionTemplate

### Problem 1: Works only on Spring-managed proxy calls

`@Transactional` relies on **Spring AOP proxy**. It does NOT work when:
- You call a `@Transactional` method **from within the same class** (self-invocation)
- The object is not a Spring Bean

```java
@Service
public class OrderService {

    // This transactional will NOT work!
    // called from within same class → proxy is bypassed
    public void process() {
        this.placeOrder(order); // calls directly, no proxy involved
    }

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
    }
}
```

### Problem 2: Cannot control transaction boundaries programmatically

With `@Transactional`, the transaction covers the **entire method**.
You **cannot** do this:

```
Do some non-transactional work (e.g., call an external API)
    ↓
Start transaction
    ↓
Save to DB
    ↓
Commit
    ↓
Do more non-transactional work (e.g., send email)
```

### Problem 3: Conditional transactions

You cannot conditionally start/skip a transaction based on runtime logic.

### Problem 4: Exception handling inside transaction

If you catch an exception inside the method and do not rethrow it, Spring will NOT rollback.

---

## 3. What is `TransactionTemplate`?

`TransactionTemplate` is the **programmatic** approach to transaction management.
It gives you **full control** over:
- When the transaction starts
- When it commits / rolls back
- What part of the code is transactional

```java
@Service
public class OrderService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public void placeOrder(Order order) {
        // Non-transactional work (validate input, call external API)
        validateOrder(order);

        // Only THIS block runs inside a transaction
        transactionTemplate.execute(status -> {
            orderRepository.save(order);
            inventoryRepository.reduce(order.getProductId(), order.getQuantity());
            return null;
        });

        // Non-transactional work (send confirmation email)
        emailService.sendConfirmation(order);
    }
}
```

---

## 4. Setting up `TransactionTemplate`

```java
@Configuration
public class TransactionConfig {

    @Bean
    public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
        TransactionTemplate template = new TransactionTemplate(transactionManager);
        template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        template.setTimeout(30); // 30 seconds
        return template;
    }
}
```

Or create it in the service constructor:

```java
@Service
public class PaymentService {

    private final TransactionTemplate transactionTemplate;

    public PaymentService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }
}
```

---

## 5. Detailed Code Examples

### Example 1: Basic Usage

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TransactionTemplate transactionTemplate;

    public User createUser(String name, String email) {
        // Step 1: Non-transactional — validate input
        if (name == null || email == null) {
            throw new IllegalArgumentException("Name and email are required");
        }

        // Step 2: Transactional — DB operations only
        User savedUser = transactionTemplate.execute(status -> {
            User user = new User();
            user.setName(name);
            user.setEmail(email);
            return userRepository.save(user);
        });

        // Step 3: Non-transactional — send welcome email (DB connection already released)
        emailService.sendWelcomeEmail(savedUser.getEmail());

        return savedUser;
    }
}
```

---

### Example 2: Manual Rollback using `TransactionStatus`

`execute()` gives you a `TransactionStatus` object.
Call `status.setRollbackOnly()` to manually mark for rollback without throwing an exception.

```java
@Service
public class OrderService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public String processOrder(Order order) {
        return transactionTemplate.execute(status -> {
            try {
                orderRepository.save(order);

                boolean inventoryAvailable = inventoryRepository.reduce(
                    order.getProductId(), order.getQuantity()
                );

                if (!inventoryAvailable) {
                    // Mark rollback WITHOUT throwing exception
                    status.setRollbackOnly();
                    return "FAILED: Insufficient inventory";
                }

                return "SUCCESS";

            } catch (Exception e) {
                // Explicit rollback on any unexpected error
                status.setRollbackOnly();
                return "FAILED: " + e.getMessage();
            }
        });
    }
}
```

---

### Example 3: Self-Invocation Problem SOLVED

```java
// @Transactional approach — BROKEN for self-invocation
@Service
public class ReportService_BAD {

    public void generateAll() {
        this.generateReport("sales");   // Proxy bypassed — no transaction!
        this.generateReport("finance"); // Same issue
    }

    @Transactional
    public void generateReport(String type) {
        // Transaction will NOT apply when called via this.*
    }
}

// TransactionTemplate approach — WORKS for self-invocation
@Service
public class ReportService_GOOD {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public void generateAll() {
        generateReport("sales");   // Works fine
        generateReport("finance"); // Works fine
    }

    public void generateReport(String type) {
        transactionTemplate.execute(status -> {
            reportRepository.save(new Report(type));
            return null;
        });
    }
}
```

---

### Example 4: Per-iteration transactions in a loop (Bulk Import)

```java
@Service
public class BulkImportService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public ImportResult importProducts(List<ProductDto> products) {
        int success = 0;
        int failed = 0;

        for (ProductDto dto : products) {
            // Each product gets its OWN transaction
            // If one fails, others are NOT affected
            try {
                transactionTemplate.execute(status -> {
                    Product product = mapToEntity(dto);
                    productRepository.save(product);
                    return null;
                });
                success++;
            } catch (Exception e) {
                failed++;
                log.warn("Failed to import product: " + dto.getName(), e);
            }
        }

        return new ImportResult(success, failed);
    }
}
```

NOTE: With @Transactional on the method — if ANY product fails, ALL are rolled back.
With TransactionTemplate in a loop — each product transaction is independent.

---

### Example 5: Configuring Propagation and Isolation

```java
@Service
public class PaymentService {

    private final TransactionTemplate requiredNewTemplate;
    private final TransactionTemplate readCommittedTemplate;

    public PaymentService(PlatformTransactionManager transactionManager) {
        // Template 1: Always starts a NEW transaction (REQUIRES_NEW propagation)
        this.requiredNewTemplate = new TransactionTemplate(transactionManager);
        this.requiredNewTemplate.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
        );

        // Template 2: READ_COMMITTED isolation for read-only operations
        this.readCommittedTemplate = new TransactionTemplate(transactionManager);
        this.readCommittedTemplate.setIsolationLevel(
            TransactionDefinition.ISOLATION_READ_COMMITTED
        );
        this.readCommittedTemplate.setReadOnly(true);
    }

    public void processPayment(Payment payment) {
        // Save audit log in its own transaction (even if outer tx fails)
        requiredNewTemplate.execute(status -> {
            auditRepository.save(new AuditLog("Payment attempt", payment.getId()));
            return null;
        });

        // Process actual payment in a new independent transaction
        requiredNewTemplate.execute(status -> {
            paymentRepository.save(payment);
            accountRepository.debit(payment.getAccountId(), payment.getAmount());
            return null;
        });
    }

    public PaymentSummary getPaymentSummary(Long id) {
        return readCommittedTemplate.execute(status ->
            paymentRepository.findSummaryById(id)
        );
    }
}
```

---

## 6. `executeWithoutResult()` — when you do not need a return value

```java
transactionTemplate.executeWithoutResult(status -> {
    orderRepository.save(order);
    inventoryRepository.update(order.getProductId());
    // no return needed — cleaner than returning null
});
```

---

## 7. Key Differences — Summary Table

| Feature                                | `@Transactional`         | `TransactionTemplate`                   |
|----------------------------------------|--------------------------|-----------------------------------------|
| Approach                               | Declarative (AOP proxy)  | Programmatic                            |
| Self-invocation                        | Does NOT work            | Works                                   |
| Partial method transaction             | Wraps entire method      | Any code block inside method            |
| Conditional transactions               | Not possible             | Full runtime control                    |
| Manual rollback without exception      | Not easy                 | `status.setRollbackOnly()`              |
| Per-iteration transactions (loops)     | All-or-nothing           | Independent per iteration               |
| Configuration                          | Annotation attributes    | Setter methods on template object       |
| Code verbosity                         | Low (clean annotations)  | Higher (explicit lambda)                |
| Best for                               | Simple CRUD services     | Complex transactional logic             |

---

## 8. When to use which?

### Use `@Transactional` when:
- Simple service methods (CRUD operations)
- The whole method should be in one transaction
- No complex conditional logic needed
- No self-invocation issue

### Use `TransactionTemplate` when:
- You need a transaction **only around a small block** inside a method
- You face **self-invocation** problems
- You need **per-iteration transactions** in a loop (bulk operations)
- You want **conditional rollback** using `status.setRollbackOnly()`
- You need **fine-grained exception handling** inside the transaction
- You want to keep **non-transactional work** (API calls, email sending) outside the DB
  transaction to avoid holding the connection too long

---

## 9. Internals — How TransactionTemplate works

```
transactionTemplate.execute(callback)
        ↓
PlatformTransactionManager.getTransaction()   → BEGIN TRANSACTION
        ↓
callback.doInTransaction(status)              → YOUR CODE runs here
        ↓
  No exception AND status.isRollbackOnly() == false
        → transactionManager.commit()

  Exception thrown OR status.isRollbackOnly() == true
        → transactionManager.rollback()
```

TransactionTemplate uses the same PlatformTransactionManager as @Transactional
— just without the AOP proxy layer.

---

## 10. Complete Real-World Example — Order Fulfillment

```java
@Service
@RequiredArgsConstructor
public class OrderFulfillmentService {

    private final TransactionTemplate transactionTemplate;
    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;  // external API — NOT in transaction
    private final EmailService emailService;       // external — NOT in transaction

    public FulfillmentResult fulfill(OrderRequest request) {

        // Step 1: Validate — non-transactional
        validateRequest(request);

        // Step 2: Reserve inventory — own transaction
        boolean reserved = transactionTemplate.execute(status -> {
            return inventoryRepository.reserve(request.getProductId(), request.getQty());
        });

        if (!reserved) {
            return FulfillmentResult.failure("Inventory not available");
        }

        // Step 3: Charge payment — external API call, NO DB transaction needed here
        // Keeping it outside avoids holding DB connection during slow network calls
        PaymentResult paymentResult = paymentGateway.charge(request.getPaymentInfo());

        if (!paymentResult.isSuccess()) {
            // Compensate: release reservation in a NEW transaction
            transactionTemplate.executeWithoutResult(status -> {
                inventoryRepository.releaseReservation(request.getProductId(), request.getQty());
            });
            return FulfillmentResult.failure("Payment failed");
        }

        // Step 4: Save order + confirm inventory — single transaction
        Order savedOrder = transactionTemplate.execute(status -> {
            try {
                Order order = Order.from(request, paymentResult.getTransactionId());
                Order saved = orderRepository.save(order);
                inventoryRepository.confirmReservation(request.getProductId(), request.getQty());
                return saved;
            } catch (Exception e) {
                status.setRollbackOnly();
                return null;
            }
        });

        if (savedOrder == null) {
            // Refund payment — non-transactional, after rollback
            paymentGateway.refund(paymentResult.getTransactionId());
            return FulfillmentResult.failure("Order creation failed");
        }

        // Step 5: Send email — non-transactional, after DB commit
        emailService.sendOrderConfirmation(savedOrder);

        return FulfillmentResult.success(savedOrder.getId());
    }
}
```

Key point: paymentGateway.charge() and emailService.sendOrderConfirmation() are kept
OUTSIDE the transaction. The DB connection is not held open during slow external API calls,
which reduces lock contention and prevents connection pool exhaustion.
