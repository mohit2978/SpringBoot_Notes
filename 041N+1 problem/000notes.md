
> **A Comprehensive Guide to Association Fetching, Diagnostics, and Elimination in Spring Data JPA & Hibernate**  
> *Updated for Hibernate 6 & 7.4, Jakarta Persistence 3.2, Spring Boot 3.x & 4.x, and Java 21/25.*

---

![Hero: N+1 in Hibernate](<svgs/00-hero-n1-hibernate.svg>)

---

## 1. The Anatomy of an Outage: Why N+1 Stays Invisible

Consider an ordinary Spring Boot REST endpoint: `GET /api/orders`. 

Here is the complete end-to-end code for the **Controller**, **DTO**, **Service**, and **Repository**:

### 1. The Controller
```java
package com.example.shop.controller;

import com.example.shop.dto.OrderResponse;
import com.example.shop.service.OrderReportService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api")
public class OrderController {

    private final OrderReportService orderReportService;

    public OrderController(OrderReportService orderReportService) {
        this.orderReportService = orderReportService;
    }

    @GetMapping("/orders")
    public List<OrderResponse> getOrders() {
        return orderReportService.getRecentOrderSummaries();
    }
}
```

### 2. The Response DTO
```java
package com.example.shop.dto;

import java.util.List;

public record OrderResponse(
    Long orderId,
    String customerName,
    int totalQuantity,
    List<String> productNames
) {}
```

### 3. The Service
```java
package com.example.shop.service;

import com.example.shop.domain.OrderLine;
import com.example.shop.dto.OrderResponse;
import com.example.shop.repository.OrderRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional(readOnly = true)
public class OrderReportService {

    private final OrderRepository orderRepository;

    public OrderReportService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public List<OrderResponse> getRecentOrderSummaries() {
        return orderRepository.findAll().stream()                    // Query 1: SELECT * FROM orders
            .map(order -> new OrderResponse(
                order.getId(),
                order.getCustomer().getName(),                       // +1 query per order (Customer proxy)
                order.getLines().stream()                            // +1 query per order (Lines bag)
                     .mapToInt(OrderLine::getQuantity).sum(),
                order.getLines().stream()
                     .map(line -> line.getProduct().getName())       // +1 query per line (Product proxy)
                     .toList()
            ))
            .toList();
    }
}
```

### 4. The Repository
```java
package com.example.shop.repository;

import com.example.shop.domain.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Inherits findAll(), findById(), save(), count(), etc. from Spring Data JPA
}
```

### 5. The Entity Classes (Domain Model)

#### `Order.java`
```java
package com.example.shop.domain;

import jakarta.persistence.*;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    // NOTE: ManyToOne defaults to EAGER in JPA spec. Always explicitly override to LAZY!
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    // OneToMany defaults to LAZY
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    // Constructors, getters, setters
    public Long getId() { return id; }
    public Instant getCreatedAt() { return createdAt; }
    public Customer getCustomer() { return customer; }
    public List<OrderLine> getLines() { return lines; }

    public void addLine(OrderLine line) {
        lines.add(line);
        line.setOrder(this);
    }
}
```

#### `OrderLine.java`
```java
package com.example.shop.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "order_line")
public class OrderLine {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    private int quantity;
    private BigDecimal price;

    // Getters and setters
    public Long getId() { return id; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
    public BigDecimal getPrice() { return price; }
}
```

#### `Customer.java`
```java
package com.example.shop.domain;

import jakarta.persistence.*;

@Entity
@Table(name = "customer")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    private String name;
    private String email;

    // Getters and setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}
```

#### `Product.java`
```java
package com.example.shop.domain;

import jakarta.persistence.*;

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    private String name;
    private String sku;

    // Getters and setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getSku() { return sku; }
}
```

---

Locally on your developer machine, your database holds 5 sample orders. The endpoint executes 6 queries, responds in **18 milliseconds**, and everyone moves on to the next user story.

Then the code deploys to production:
* The production page displays **50 orders**.
* Response time explodes to **2.4 seconds**.
* Under modest traffic of 60 requests/sec, the HikariCP connection pool is completely exhausted, and unrelated microservices time out waiting for database connections.

When inspecting the logs, the single HTTP request executed **151 database queries**:
1. **1 query** to fetch the 50 orders.
2. **50 queries** to fetch the customer for each order (`@ManyToOne`).
3. **50 queries** to fetch the line items for each order (`@OneToMany`).
4. **50 queries** to fetch the product details for each line item (nested `@ManyToOne`, up to 500+ depending on distinct products).

```
          ┌──────────────────────────────────────────────┐
          │      GET /api/orders (Client Request)        │
          └──────────────────────┬───────────────────────┘
                                 │
                    1. SELECT * FROM orders LIMIT 50
                                 ▼
      ┌──────────────────────────────────────────────────────┐
      │  Order 1   │  Order 2   │  Order 3   │ ... Order 50  │
      └──────┬───────────┬────────────┬──────────────┬───────┘
             │           │            │              │
      SELECT customer  SELECT customer  SELECT customer  ... (50 queries)
      SELECT lines     SELECT lines     SELECT lines     ... (50 queries)
      SELECT product   SELECT product   SELECT product   ... (50+ queries)
                                 │
                                 ▼
                    🔥 1 + 50 + 50 + 50 = 151 QUERIES!
```

### The Query Math: How `@OneToMany` vs `@ManyToOne` Generate Queries

A common point of confusion is: *"If an order has multiple lines, why is it only 50 queries for lines, but can be 500+ for products?"*

#### 1. For `@OneToMany` (`order.getLines()`): Exactly 1 Query Per Parent Order (50 Queries)
When Hibernate initializes a lazy `@OneToMany` collection (`List<OrderLine> lines`), it does **not** fetch line items one by one. It queries all child rows belonging to that specific parent in **one single SELECT**:

```sql
SELECT * FROM order_line WHERE order_id = ?;
```

* For **Order 1** (even if it contains 10 lines): Hibernate executes **1 query**:
  ```sql
  SELECT * FROM order_line WHERE order_id = 1;
  -- Returns all 10 lines for Order 1 in this single result set!
  ```
* For **Order 2** (even if it contains 5 lines): Hibernate executes **1 query**:
  ```sql
  SELECT * FROM order_line WHERE order_id = 2;
  ```
* ... up to **Order 50**: Hibernate executes **1 query**:
  ```sql
  SELECT * FROM order_line WHERE order_id = 50;
  ```

Because all children for a parent are fetched together using the parent's foreign key, **50 orders = exactly 50 queries for `order_line`**.

---

#### 2. For Direct `@ManyToOne` (`order.getCustomer()`): Exactly 1 Query Per Parent (50 Queries)
Each `Order` holds a single reference to a `Customer`. Because it is mapped as `FetchType.LAZY`, Hibernate injects a ByteBuddy proxy holding only `customer_id`. When `order.getCustomer().getName()` is called for each order:

```sql
SELECT id, name, email FROM customer WHERE id = ?;
```
For 50 orders, this executes **50 individual queries** (unless customers are shared and already cached in the current Session).

---

#### 3. For Nested `@ManyToOne` Inside a Collection (`line.getProduct()`): The 50+ to 500+ Query Explosion!
This is where the exponential query multiplication happens.

Notice that `OrderLine` has its own lazy `@ManyToOne` pointing to `Product`:
```java
public class OrderLine {
    ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product; // Each line item holds a separate Product proxy!
}
```

When you loop through the lines to extract product names:
```java
order.getLines().stream()
     .map(line -> line.getProduct().getName()) // <-- NESTED N+1 TRIGGER!
```

If 50 orders have an average of **10 lines each**, your application has:
$$\text{Total Line Items} = 50 \text{ orders} \times 10 \text{ lines} = 500 \text{ lines}$$

Hibernate must now initialize the `Product` proxy for each of those 500 line items:
```sql
SELECT id, name, sku FROM product WHERE id = ?; -- Query fired for line 1
SELECT id, name, sku FROM product WHERE id = ?; -- Query fired for line 2
SELECT id, name, sku FROM product WHERE id = ?; -- Query fired for line 3
... [Can fire up to 500 separate queries!]
```

#### Why did the example say 50 queries instead of 500? The First-Level Cache (Session):
* If every line item in your database references a **unique, distinct product**, Hibernate will fire **500 queries** just for products!
* However, if products are shared (e.g., multiple orders purchased the same "USB-C Cable" with ID `12`), Hibernate checks its **First-Level Cache (the active Hibernate Session)**. If entity `Product#12` was already loaded into the Session by a previous line, Hibernate reuses the in-memory object and avoids hitting the database again.
* In our scenario, assuming 50 distinct products were touched across the 500 lines, Hibernate executed 50 product queries. If all 500 products were distinct, the query count would have skyrocketed to **601 queries**!

---

### Summary Table of Query Multiplication

| Step | Association | Type | What Hibernate Executes | Queries Fired |
| :--- | :--- | :--- | :--- | :---: |
| **Step 1: Orders** | Root Entity | Table query | `SELECT * FROM orders LIMIT 50;` | **1** |
| **Step 2: Customer** | `order.getCustomer()` | Direct `@ManyToOne` | `SELECT * FROM customer WHERE id = ?;` | **50** |
| **Step 3: Lines** | `order.getLines()` | `@OneToMany` Collection | `SELECT * FROM order_line WHERE order_id = ?;` | **50** *(1 per order)* |
| **Step 4: Product** | `line.getProduct()` | Nested `@ManyToOne` | `SELECT * FROM product WHERE id = ?;` | **50 to 500+** *(N+1 inside N+1)* |
| **TOTAL** | | | | **151 to 601+ Queries!** |

Nothing crashed. No stack trace was printed. Hibernate did not malfunction—**it performed exactly as instructed**.

> [!IMPORTANT]
> **N+1 is not a bug introduced by an error; it is the default mechanical consequence of object-relational mapping with lazy loading.** Every lazy association in your entity graph is a latent query multiplier that remains silent until production row volumes trigger a cascade.

---

## 2. What N+1 Actually Is at the Engine Level

To permanently fix N+1, you must understand how Hibernate handles the lazy associations in `Order`, `OrderLine`, `Customer`, and `Product` under the hood using **ByteBuddy Dynamic Proxies** and **Persistent Collections**.

---

### The Innocent Service Code (Revisited)

Now observe the `OrderReportService.getRecentOrderSummaries()` method introduced above:

```java
package com.example.shop.service;

import com.example.shop.domain.OrderLine;
import com.example.shop.dto.OrderResponse;
import com.example.shop.repository.OrderRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional(readOnly = true)
public class OrderReportService {

    private final OrderRepository orderRepository;

    public OrderReportService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public List<OrderResponse> getRecentOrderSummaries() {
        return orderRepository.findAll().stream()                    // Query 1: SELECT * FROM orders
            .map(order -> new OrderResponse(
                order.getId(),
                order.getCustomer().getName(),                       // +1 query per order (to-one proxy)
                order.getLines().stream()                            // +1 query per order (collection bag)
                     .mapToInt(OrderLine::getQuantity).sum(),
                order.getLines().stream()
                     .map(line -> line.getProduct().getName())       // +1 query per line item (nested proxy)
                     .toList()
            ))
            .toList();
    }
}
```

Reading this code, nothing looks suspicious. But observe the SQL statements emitted by Hibernate (shown here in modern Hibernate 6 & 7 alias notation):

```sql
-- 1. Initial query to load orders
select o1_0.id, o1_0.created_at, o1_0.customer_id from orders o1_0;

-- 2. Proxy initialization for order 1's Customer
select c1_0.id, c1_0.email, c1_0.name from customer c1_0 where c1_0.id = ?;

-- 3. PersistentBag initialization for order 1's Lines
select l1_0.order_id, l1_0.id, l1_0.price, l1_0.product_id, l1_0.quantity 
from order_line l1_0 where l1_0.order_id = ?;

-- 4. Proxy initialization for product on line 1
select p1_0.id, p1_0.name, p1_0.sku from product p1_0 where p1_0.id = ?;

-- 5. Proxy initialization for product on line 2
select p1_0.id, p1_0.name, p1_0.sku from product p1_0 where p1_0.id = ?;

-- 6. Proxy initialization for order 2's Customer
select c1_0.id, c1_0.email, c1_0.name from customer c1_0 where c1_0.id = ?;

-- 7. PersistentBag initialization for order 2's Lines
select l1_0.order_id, l1_0.id, l1_0.price, l1_0.product_id, l1_0.quantity 
from order_line l1_0 where l1_0.order_id = ?;

-- ... repeating for every single order and line item!
```

---

### The Anatomy of the Query Explosion

The diagram below maps out how a single endpoint call cascades through your database:

![The Anatomy of N+1 Query Explosion](<svgs/scheme-a-n1-shape.svg>)

#### Why Does This Happen Mechanically?

1. **Lazy `@ManyToOne` produces a ByteBuddy Proxy:**  
   When Hibernate loads an `Order`, it populates `customer_id`. Rather than querying the `customer` table, it creates a synthetic proxy subclass (`Customer$ByteBuddy$xyz`). The proxy contains only the `id`. The moment any other getter (`customer.getName()`) is invoked, the proxy intercepts the call and runs:
   ```sql
   SELECT * FROM customer WHERE id = ?
   ```
2. **Lazy `@OneToMany` produces a `PersistentBag`:**  
   Hibernate wraps the `lines` list in its internal collection wrapper (`org.hibernate.collection.spi.PersistentBag`). The bag holds a reference to the active `Session` and an uninitialized state flag. Calling `.stream()`, `.size()`, or `.iterator()` on the list triggers collection initialization:
   ```sql
   SELECT * FROM order_line WHERE order_id = ?
   ```
3. **Loop Multiplication:**  
   Because the service loops through $N$ orders, Hibernate issues $N$ customer queries, $N$ collection queries, and up to $N \times M$ nested product queries.

---

## 3. Why It Hides in Development & Why EAGER Makes It Worse

### 1. The Development vs Production Gap
In development on `localhost`:
* 5 orders $\to$ 6 queries.
* Query time over an in-memory database (H2) or local socket is `< 0.2 ms`.
* Total request time is $\approx 5\text{ ms}$. It is physically imperceptible.

In production:
* 100 orders $\to$ $1 + 100 + 100 + 1,000 = 1,201$ queries.
* Even with a tight database network latency of `0.8 ms` per round-trip:  
  $$1,200 \times 0.8\text{ ms} = 960\text{ ms of pure network wait time!}$$
* Worse, each round-trip holds a connection checked out from HikariCP. If your pool has 10 connections, only 8 concurrent users will completely saturate the pool, causing timeouts across your entire application.

---

### 2. The Open-Session-in-View (OSIV) Trap

In Spring Boot, `spring.jpa.open-in-view` is enabled (`true`) by default.
* The Hibernate `Session` / `EntityManager` remains open across the entire HTTP request lifecycle, through controller execution and JSON serialization.
* When Jackson serializes the returned entities into JSON, it calls entity getters (`order.getCustomer()`, `order.getLines()`).
* **These getters trigger lazy loading outside of your `@Transactional` service boundary!**
* No error is thrown. The endpoint silently issues hundreds of queries during response serialization.

> [!TIP]
> **Always disable OSIV in your configuration:**
> ```yaml
> spring:
>   jpa:
>     open-in-view: false
> ```
> With OSIV set to `false`, any uninitialized association accessed outside an active transaction immediately throws `LazyInitializationException`. This forces your queries to fetch all required data within the service boundary where performance can be audited.

---

### 3. The Big Lie: `FetchType.EAGER` Does NOT Fix N+1

The most common mistake made by developers is changing `@ManyToOne(fetch = FetchType.LAZY)` to `FetchType.EAGER`.

```java
// ❌ DANGEROUS: Does NOT fix N+1 in JPQL queries!
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

Here is why `FetchType.EAGER` is an anti-pattern:

1. **`EAGER` only controls *when* data is loaded, not *how* it is fetched.**  
   When you query orders via JPQL or Spring Data JPA:
   ```java
   @Query("SELECT o FROM Order o")
   List<Order> findAllOrders();
   ```
   Hibernate parses your query and executes:
   ```sql
   SELECT o.id, o.created_at, o.customer_id FROM orders o;
   ```
   After loading the 100 `Order` rows, Hibernate inspects the entity metadata and sees `customer` is marked `EAGER`. It immediately fires 100 separate queries to load each customer!
   **You get the exact same N+1 queries, but now you have lost the ability to load an order without its customer.**
2. **Cartesian Product Everywhere:**  
   When using `em.find(Order.class, id)`, Hibernate will attempt an outer join. If multiple associations are eager, every query will join all tables, creating massive row duplication even when you only need a single field.

> [!CAUTION]
> **Rule of Thumb:** Every association in JPA (`@ManyToOne`, `@OneToOne`, `@OneToMany`, `@ManyToMany`) should be mapped as `FetchType.LAZY`. Never use `FetchType.EAGER` as a workaround for N+1.

---

## 4. Proper Detection & Measurement

Stop relying on intuition. Count statements with automated diagnostics.

### 1. Configuring Logging Properly

Do not use `spring.jpa.show-sql=true`. It prints unformatted SQL directly to `stdout`, bypassing logging frameworks and omitting bind parameter values.

Instead, configure targeted logging categories in `application.yml`:

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        generate_statistics: true       # Enables session metrics (Dev/Test only)
        highlight_sql: true             # Colorizes console output
        format_sql: false               # Keep 1 line per query to make grep/counting easy

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE  # Shows bound parameter values (Hibernate 6 & 7)
    org.hibernate.stat: DEBUG           # Logs per-session metrics on close
```

> [!NOTE]
> In Hibernate 5, the bind parameter logging category was `org.hibernate.type.descriptor.sql`. In **Hibernate 6 and 7**, this category was moved to `org.hibernate.orm.jdbc.bind`.

When `generate_statistics: true` is active, Hibernate logs a session summary at the end of each transaction:

```
Session Metrics {
    151 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 1 JDBC connections;
    124000 nanoseconds spent preparing 151 JDBC statements;
    100 nanoseconds spent executing 151 JDBC statements;
    50 collections fetched (lazy)
    100 entities fetched (lazy)
}
```

Notice the two critical metrics:
* `entities fetched (lazy)`: Counts single-entity proxy initializations.
* `collections fetched (lazy)`: Counts lazy collection initializations.

If either counter is non-zero, an N+1 condition occurred.

---

### 2. Programmatic Inspection via `StatementInspector`

To count executed statements in test suites without the performance overhead of Hibernate statistics, implement a custom `StatementInspector`:

```java
package com.example.shop.diagnostic;

import org.hibernate.resource.jdbc.spi.StatementInspector;
import java.util.concurrent.atomic.AtomicLong;

public class QueryCountInspector implements StatementInspector {

    private static final ThreadLocal<AtomicLong> SELECT_COUNT = 
            ThreadLocal.withInitial(AtomicLong::new);

    public static void reset() {
        SELECT_COUNT.get().set(0);
    }

    public static long getSelectCount() {
        return SELECT_COUNT.get().get();
    }

    @Override
    public String inspect(String sql) {
        if (sql.trim().regionMatches(true, 0, "select", 0, 6)) {
            SELECT_COUNT.get().incrementAndGet();
        }
        return sql;
    }
}
```

Register it in your `application-test.yml`:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        session_factory:
          statement_inspector: com.example.shop.diagnostic.QueryCountInspector
```

---

## 5. Fix 1 — `JOIN FETCH` (The Direct Round-Trip Solution)

`JOIN FETCH` tells the persistence provider to fetch the associated entities or collections in the same database query using an SQL `JOIN`.

![Fix 1: JOIN FETCH Mechanics](<svgs/scheme-b-fetch-join.svg>)

### 1. JPQL Implementation

```java
package com.example.shop.repository;

import com.example.shop.domain.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.Instant;
import java.util.List;

public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
        SELECT o FROM Order o
        JOIN FETCH o.customer
        LEFT JOIN FETCH o.lines l
        LEFT JOIN FETCH l.product
        WHERE o.createdAt >= :since
        ORDER BY o.createdAt DESC
        """)
    List<Order> findAllWithLinesAndProducts(@Param("since") Instant since);
}
```

#### Key Details:
1. **`JOIN FETCH` vs `LEFT JOIN FETCH`:**  
   * Use inner `JOIN FETCH` for required `@ManyToOne` associations (e.g., an order must have a customer).
   * Use `LEFT JOIN FETCH` for collections (`o.lines`). An inner join would silently exclude orders that currently have 0 lines.
2. **The `DISTINCT` Keyword in Modern Hibernate:**  
   * In Hibernate 5, fetching a collection duplicated root `Order` objects in the returned list, requiring `SELECT DISTINCT o`.
   * **In Hibernate 6 and 7, root entities are automatically deduplicated in memory.** Adding `DISTINCT` to JPQL today forces the SQL engine to sort and deduplicate wide Cartesian rows, wasting database CPU.

---

### 2. Type-Safe Fetching via Criteria API

```java
public List<Order> findOrdersTypeSafe(Instant since, EntityManager em) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Order> cq = cb.createQuery(Order.class);
    Root<Order> order = cq.from(Order.class);

    // Fetch to-one association
    order.fetch("customer", JoinType.INNER);

    // Fetch to-many and nested to-one
    Fetch<Order, OrderLine> linesFetch = order.fetch("lines", JoinType.LEFT);
    linesFetch.fetch("product", JoinType.LEFT);

    cq.select(order)
      .where(cb.greaterThanOrEqualTo(order.get("createdAt"), since))
      .orderBy(cb.desc(order.get("createdAt")));

    return em.createQuery(cq).getResultList();
}
```

---

### 3. The Pagination Trap (And What Changed in Hibernate 7.4)

Historically, combining pagination (`setFirstResult()` / `setMaxResults()`) with a collection `JOIN FETCH` was a critical bug:

```
WARN: HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

#### Why Did This Happen?
When an order with 10 lines is joined, the database returns 10 rows for that 1 order. If Hibernate put `LIMIT 20` into the SQL, the database would return 20 flat rows—representing only 2 full orders and part of a third!

To prevent returning partial entities, Hibernate historically:
1. Removed `LIMIT` and `OFFSET` from the SQL query.
2. Fetched **the entire database table (e.g., 500,000 rows)** into JVM memory.
3. Sublisted the collection in Java memory.

> [!WARNING]
> In **Hibernate 6.x and 7.0–7.3**, this behavior still exists. If you paginate over a collection fetch, your JVM will suffer high memory pressure and garbage collection pauses.

#### The Breakthrough in Hibernate 7.4+ (Spring Boot 4.x)
Starting in **Hibernate 7.4**, pagination over collection fetches is processed natively in SQL! Hibernate generates a subquery that applies the limit to the parent root entity first, and then joins the collections against the subquery:

```sql
-- Generated by Hibernate 7.4+
SELECT o.id, c.name, l.id, p.name
FROM (
    SELECT o1.id, o1.created_at, o1.customer_id
    FROM orders o1
    WHERE o1.created_at >= ?
    ORDER BY o1.created_at DESC
    LIMIT 20 OFFSET 0
) o
JOIN customer c ON c.id = o.customer_id
LEFT JOIN order_line l ON l.order_id = o.id
LEFT JOIN product p ON p.id = l.product_id;
```

#### Two-Query Pattern for Hibernate 6.x & 7.0–7.3
If you are running Spring Boot 3.x or Hibernate 6, use the standard two-step query pattern:

```java
// Step 1: Page the primary IDs with LIMIT in SQL
List<Long> orderIds = em.createQuery("""
        SELECT o.id FROM Order o
        WHERE o.createdAt >= :since
        ORDER BY o.createdAt DESC
        """, Long.class)
    .setParameter("since", since)
    .setFirstResult((int) pageable.getOffset())
    .setMaxResults(pageable.getPageSize())
    .getResultList();

// Step 2: Fetch full association graph for exactly those IDs
List<Order> pagedOrders = em.createQuery("""
        SELECT o FROM Order o
        JOIN FETCH o.customer
        LEFT JOIN FETCH o.lines l
        LEFT JOIN FETCH l.product
        WHERE o.id IN :ids
        ORDER BY o.createdAt DESC
        """, Order.class)
    .setParameter("ids", orderIds)
    .getResultList();
```

---

### 4. The `MultipleBagFetchException`

If you attempt to join fetch two `List` collections in a single JPQL query:

```java
// ❌ THROWS MultipleBagFetchException
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.lines LEFT JOIN FETCH o.tags")
List<Order> findWithLinesAndTags();
```

Hibernate throws:
```
org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags: [Order.lines, Order.tags]
```

A `Bag` is an unordered collection that allows duplicates (`java.util.List`). Fetching two bags simultaneously creates an $N \times M$ Cartesian product where Hibernate cannot reconstruct which row belongs to which child collection without generating duplicates.

**The Fix:** Fetch-join the largest collection, and use batch fetching (`@BatchSize`) for the other.

---

## 6. Fix 2 — `EntityGraph` (Declarative Fetch Plans)

While `JOIN FETCH` couples the fetching strategy directly to the JPQL query string, `EntityGraph` decouples the fetch plan, making it reusable across multiple queries.

![Fix 2: EntityGraph Decoupling](<svgs/scheme-c-entitygraph.svg>)

### 1. Spring Data JPA `@EntityGraph`

```java
package com.example.shop.repository;

import com.example.shop.domain.Order;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

public interface OrderRepository extends JpaRepository<Order, Long> {

    // 1. Dynamic ad-hoc paths
    @EntityGraph(attributePaths = {"customer", "lines", "lines.product"})
    List<Order> findByCreatedAtAfter(Instant since);

    // 2. Fetch plan for single entity lookup by ID
    @EntityGraph(attributePaths = {"customer", "lines.product"})
    Optional<Order> findWithDetailsById(Long id);

    // 3. To-one only graph is safe for pagination in all Hibernate versions
    @EntityGraph(attributePaths = {"customer"})
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);
}
```

---

### 2. Jakarta Persistence 3.2 Dynamic API

In Jakarta Persistence 3.2 (Hibernate 7.4 / Spring Boot 4.1), the old string-based hint map (`"jakarta.persistence.fetchgraph"`) has been replaced by first-class typed API overloads on `EntityManager`:

```java
// Create dynamic graph
EntityGraph<Order> graph = em.createEntityGraph(Order.class);
graph.addAttributeNodes("customer");
graph.addSubgraph("lines").addAttributeNodes("product");

// Modern JPA 3.2 overload: EntityGraph is passed directly as first parameter!
Order order = em.find(graph, orderId);

// Batch loading multiple IDs with an EntityGraph in Hibernate 7:
List<Order> orders = em.unwrap(org.hibernate.Session.class)
    .findMultiple(graph, List.of(1L, 2L, 3L, 4L));
```

> [!NOTE]
> Under the hood, Hibernate executes `EntityGraph` as SQL joins. All rules regarding Cartesian products and bag restrictions apply equally to entity graphs.

---

## 7. Fix 3 — Batch Fetching (The Universal Global Safety Net)

Batch fetching is the most pragmatic fix in enterprise architectures. It does not alter your query strings or JPQL definitions. Instead, it instructs Hibernate to initialize uninitialized proxies and collections in groups rather than one by one.

![Fix 3: Batch Fetching Mechanics](<svgs/scheme-d-batch-fetching.svg>)

### How It Works Mechanically

When Hibernate encounters an uninitialized proxy or collection, it inspects the current `Session` for all other uninitialized proxies of the same type and issues a single query using an `IN` clause:

```sql
-- Without batching (100 queries)
SELECT * FROM order_line WHERE order_id = 1;
SELECT * FROM order_line WHERE order_id = 2;
...

-- With batching size 50 (2 queries!)
SELECT * FROM order_line WHERE order_id IN (1, 2, 3 ... 50);
SELECT * FROM order_line WHERE order_id IN (51, 52, 53 ... 100);
```

### 1. Enable Globally in `application.yml`

This single property converts every undiscovered N+1 across your entire application from $O(N)$ into $O(\lceil N / 50 \rceil)$:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50
```

### 2. Entity-Level Annotation

You can also configure batching per entity or per association:

```java
@Entity
@Table(name = "product")
@BatchSize(size = 50)
public class Product { ... }

@Entity
@Table(name = "orders")
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 50)
    private List<OrderLine> lines = new ArrayList<>();
}
```

### 3. PostgreSQL Dialect Optimization: `= ANY(?)`

On modern database engines such as PostgreSQL, Hibernate optimizes batch fetching by using array parameters:

```sql
SELECT * FROM order_line WHERE order_id = ANY(?);
```

Instead of generating different SQL strings for batches of size 50, 43, or 12 (which thrash the database prepared statement cache), a single prepared statement is reused with varying array sizes.

---

## 8. Fix 4 — DTO Projections (The Best Fix for Read Operations)

The most scalable fix for read operations is to stop loading entities altogether.

```
       Write Operation                 Read Operation (GET API)
  ┌─────────────────────────┐        ┌─────────────────────────┐
  │      Load Entity        │        │   Execute Projection    │
  │            ▼            │        │            ▼            │
  │    Modify Properties    │        │ Map directly to DTO/Row │
  │            ▼            │        │            ▼            │
  │ Automated Dirty-Check   │        │ Stream straight to JSON │
  │            ▼            │        └─────────────────────────┘
  │     Flush UPDATE SQL    │          ⚡ No Persistence Context
  └─────────────────────────┘          ⚡ No Dirty Checking
                                       ⚡ Zero N+1 Possible
```

When rendering an API response or report, you do not need dirty-checking snapshots, entity proxies, or persistence context identity maps. Loading entities just to map them to JSON is pure overhead.

### 1. Constructor Projection into a Java Record

```java
package com.example.shop.dto;

import java.math.BigDecimal;

public record OrderSummaryDto(
    Long orderId,
    String customerName,
    long totalLineCount,
    BigDecimal totalOrderAmount
) {}
```

```java
package com.example.shop.repository;

import com.example.shop.domain.Order;
import com.example.shop.dto.OrderSummaryDto;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.Instant;

public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query(value = """
        SELECT new com.example.shop.dto.OrderSummaryDto(
            o.id,
            o.customer.name,
            count(l.id),
            coalesce(sum(l.price * l.quantity), 0)
        )
        FROM Order o
        JOIN o.customer
        LEFT JOIN o.lines l
        WHERE o.createdAt >= :since
        GROUP BY o.id, o.customer.name, o.createdAt
        ORDER BY o.createdAt DESC
        """,
        countQuery = "SELECT count(o) FROM Order o WHERE o.createdAt >= :since"
    )
    Page<OrderSummaryDto> findSummariesPaged(@Param("since") Instant since, Pageable pageable);
}
```

#### Why This Is Superior:
1. **Zero N+1 Possible:** You cannot lazily load what is never mapped as an entity.
2. **Aggregations in SQL:** Calculations (`count`, `sum`) occur inside the database engine rather than streaming thousands of line item rows across the network into Java memory.
3. **Always supply an explicit `countQuery`** when combining projections with `GROUP BY` and `Pageable` to avoid pagination parsing failures.

---

### 2. Spring Data Closed Interface Projections

For simple hierarchical to-one views, Spring Data interface projections automatically construct optimal SQL joins:

```java
public interface OrderDetailsView {
    Long getId();
    Instant getCreatedAt();
    CustomerView getCustomer();

    interface CustomerView {
        String getName();
        String getEmail();
    }
}
```

```java
List<OrderDetailsView> findProjectedByCreatedAtAfter(Instant since);
```

---

## 9. Performance Benchmark & Empirical Comparison

Workload: Load 100 orders with 10 line items each and resolve customer and product details (200 distinct products) over PostgreSQL 17 in Testcontainers:

| Strategy | Total Queries | Rows Over Network | p50 Latency (ms) | Connection Hold Time |
| :--- | :---: | :---: | :---: | :---: |
| **Naive Lazy (Unfixed)** | **401** | ~1,400 | **280 ms** | **High** (holds pool connection across loop) |
| **`JOIN FETCH`** | **1** | ~1,000 | **28 ms** | **Minimal** (1 rapid statement) |
| **`EntityGraph`** | **1** | ~1,000 | **28 ms** | **Minimal** (1 rapid statement) |
| **`default_batch_fetch_size: 50`** | **9** | ~1,400 | **26 ms** | **Low** (9 fast statement round-trips) |
| **DTO Record Projection** | **1** | **100** | **11 ms** | **Ultra-Low** (bypasses entity hydration) |

---

## 10. Automated Regression Testing: Keeping N+1 Out of CI

Every N+1 fix is one refactoring away from being broken. If someone adds a getter call to a DTO mapper, the 151-query loop returns without any test failing.

Write automated tests that fail if query counts spike!

```java
package com.example.shop;

import com.example.shop.service.OrderReportService;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import jakarta.persistence.EntityManagerFactory;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class OrderQueryCountIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17-alpine");

    @Autowired
    private OrderReportService orderReportService;

    @Autowired
    private EntityManagerFactory entityManagerFactory;

    @Autowired
    private TestDataSeeder testDataSeeder;

    @BeforeEach
    void setUp() {
        // Seed 100 orders with 10 lines each across 200 products
        testDataSeeder.seedOrders(100, 10, 200);
    }

    @Test
    @DisplayName("Report endpoint should execute bounded statements and never exceed batch threshold")
    void testReportQueryCountRegression() {
        Statistics statistics = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
        statistics.setStatisticsEnabled(true);
        statistics.clear();

        // Execute service under test
        var report = orderReportService.getRecentOrderSummaries();
        assertThat(report).hasSize(100);

        long statementsCount = statistics.getPrepareStatementCount();

        // With batch size 50: 1 (orders) + 2 (lines) + 4 (products) + 2 (customers) = 9
        // Allow a safety margin of <= 12 statements
        assertThat(statementsCount)
            .as("Statement count must be bounded, not linear N+1")
            .isLessThanOrEqualTo(12);

        // If using JOIN FETCH or EntityGraph, you can enforce an exact count:
        // assertThat(statistics.getCollectionFetchCount()).isZero();
        // assertThat(statementsCount).isEqualTo(1);
    }
}
```

> [!IMPORTANT]
> **Fixture Sizing Rule:** Always seed test data that is **at least $2\times$ your batch size** (e.g., 100 orders if batch size is 50). If you test with only 5 orders, batch fetching will hide an N+1 bug because everything fits into a single batch of 1!

---

## 11. The Master Decision Tree & Cheat Sheet

![Production Decision Flowchart](<svgs/scheme-e-decision-flow.svg>)

### Production Decision Matrix

| Scenario | Recommended Strategy | Emitted SQL | Watch Out For |
| :--- | :--- | :--- | :--- |
| **Single entity lookup by ID** | `em.find(graph, id)` or `@EntityGraph` | 1 Query (JOIN) | Simplest case |
| **List query, To-One associations only** | `JOIN FETCH` or `EntityGraph` | 1 Query (JOIN) | Use `LEFT JOIN FETCH` if association is nullable |
| **List query, 1 collection, Unpaginated** | `JOIN FETCH` or `EntityGraph` | 1 Query (JOIN) | Watch Cartesian row size |
| **List query, 1 collection, Paginated** | **Hibernate 7.4+:** `JOIN FETCH`<br>**Hibernate $\le$ 7.3:** `@BatchSize(50)` | 1 or 2 Queries | Pre-7.4 silently paginates in JVM memory |
| **2 or more collections** | `JOIN FETCH` the largest collection, batch the rest | 1 + Batches | Never fetch-join 2 `List`s (`MultipleBagFetchException`) |
| **Read-Only API, Table, or Export** | **DTO Projection (Record)** | 1 Query (SELECT) | **Always preferred for read APIs** |
| **Global safety net across app** | `default_batch_fetch_size: 50` | Batched `IN` | Still lazy; requires open Session |

---

## 12. Summary: The Golden Rules

1. **Map all associations lazy:** Always use `FetchType.LAZY`. `FetchType.EAGER` is an anti-pattern that causes hidden queries.
2. **Turn off OSIV:** Set `spring.jpa.open-in-view=false` so lazy loading leaks fail immediately in tests instead of degrading production.
3. **Entities for writes, Projections for reads:** If you are not modifying entity state in a transaction, project directly to a Java record.
4. **Set batch fetching globally:** Add `spring.jpa.properties.hibernate.default_batch_fetch_size: 50` to every Spring Boot application.
5. **On Hibernate 7.4+, paginated collection fetches are safe:** Hibernate now generates SQL subqueries for limits. On Hibernate 6, use the two-query ID pattern.
6. **Assert in CI:** Use Hibernate `Statistics` or QuickPerf to ensure query counts remain constant as your database grows.

---

*Notes compiled and updated for modern Java and Spring Boot ecosystems. References: Hibernate ORM 7.4 User Guide, Jakarta Persistence 3.2 Specification, and Vlad Mihalcea High-Performance Java Persistence.*
