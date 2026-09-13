# N+1 in Hibernate Is Not a Bug. It's a Default.

Fetch join, EntityGraph, batch size, projections: what to use, when — and how to prove it in a test. Updated for Hibernate 7.4, Jakarta Persistence 3.2 and Spring Boot 4.


![svg](<svgs/00-hero-n1-hibernate.svg>)

A `/orders` endpoint. Fifty rows on the page. Locally it returned in 40 ms and nobody looked twice.

In production it took 2.4 seconds and the connection pool was saturated at 60 requests per second. The endpoint was running **151 queries** per call: one for the orders, fifty for the customers, fifty for the line collections, and fifty more that nobody had predicted, fired from inside a Jackson serializer after the controller had already returned.

Nothing was broken. No one had written bad code. Hibernate did exactly what it had been told to do: load lazily, one owner at a time.

That's the thing about N+1 — it isn't a defect you introduce, it's the default you inherit. Every lazy association in your model is a candidate, and it stays invisible until the row count in production is two orders of magnitude bigger than the row count on your laptop.

This article covers: what N+1 actually is at the mechanical level, how to detect it (including a test that fails when it comes back), the four real fixes with the SQL each one generates, the traps in each, and a decision table so you stop guessing. Everything here is written against Hibernate ORM 7.4 (7.4.7.Final is the current stable series as of this writing), Jakarta Persistence 3.2, Spring Boot 4.1 / Spring Data JPA 4.1, and Java 25.

Two of the most-repeated pieces of N+1 advice on the internet are now wrong on this stack. I'll flag both.

## What N+1 actually is

The domain for everything below: an `Order` belongs to a `Customer` (many-to-one), holds a list of `OrderLine`s (one-to-many), and each line points at a `Product` (many-to-one).

**Code 1 — the model**

```java
@Entity
@Table(name = "orders")
public class Order {
 
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
 
    private Instant createdAt;
 
    @ManyToOne(fetch = FetchType.LAZY)   // EAGER is the JPA default here. It is
    @JoinColumn(name = "customer_id")
    private Customer customer;
 
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = tr
    private List<OrderLine> lines = new ArrayList<>();
}
 
@Entity
@Table(name = "order_line")
public class OrderLine {
 
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
 
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
 
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;
 
    private int quantity;
    private BigDecimal price;
}
```

Now the service. Read it and try to spot the problem — this is the point, you can't, not by reading.

**Code 2 — the innocent code**

```java
@Service
@Transactional(readOnly = true)
public class OrderReportService {
 
    private final OrderRepository orders;
 
    public List<OrderReportRow> report() {
        return orders.findAll().stream()                  // 1 query
            .map(o -> new OrderReportRow(
                o.getId(),
                o.getCustomer().getName(),                // +1 per order   (to-
                o.getLines().stream()                     // +1 per order   (col
                    .mapToInt(OrderLine::getQuantity).sum(),
                o.getLines().stream()
                    .map(l -> l.getProduct().getName())    // +1 per line   (nes
                    .toList()))
            .toList();
    }
}
```

Three separate N+1s stacked on top of each other. The log (Hibernate 7 alias format):

```sql
select o1_0.id,o1_0.created_at,o1_0.customer_id from orders o1_0
select c1_0.id,c1_0.email,c1_0.name from customer c1_0 where c1_0.id=?
select l1_0.order_id,l1_0.id,l1_0.price,l1_0.product_id,l1_0.quantity from order
select p1_0.id,p1_0.name,p1_0.sku from product p1_0 where p1_0.id=?
select p1_0.id,p1_0.name,p1_0.sku from product p1_0 where p1_0.id=?
select c1_0.id,c1_0.email,c1_0.name from customer c1_0 where c1_0.id=?
select l1_0.order_id,... from order_line l1_0 where l1_0.order_id=?
... and so on, forever
```

**Scheme A — the shape of it**

GET /orders?limit=100
1. `SELECT * FROM orders LIMIT 100` → orders
   for each order → order.getLines()
2. `SELECT * FROM order_line WHERE order_id = 1`
3. `SELECT * FROM order_line WHERE order_id = 2`
   ...
101. `SELECT * FROM order_line WHERE order_id = 100` → order_line
   for each line → line.getProduct()
   up to 1,000 more: `SELECT * FROM product WHERE id = ?` → product

![svg](<svgs/scheme-a-n1-shape.svg>)

```
Up to 1,000 more:   SELECT * FROM product WHERE id = ?   ← N+1 inside N+1
 
Total: 1 + 100 + (≤1000) round-trips. For one page.
```

The mechanics are worth stating precisely, because the fix follows from them. A lazy `@ManyToOne` is materialized as a **proxy**: an object with the right type and the right id and nothing else. A lazy `@OneToMany` is a `PersistentBag` — a `List` implementation that holds a reference to the session and a flag saying "not initialized yet". The first call to any real method on either one triggers `initialize()`, which issues a `SELECT` in the current persistence context. One owner, one select. Do it in a loop and you get a loop of selects.

Nobody wrote that loop. It is the sum of a lazy mapping and a `for`.

## Why you don't notice

Your dev database has five orders. Five orders means six queries. Six queries on localhost over a Unix socket is under a millisecond and shows up nowhere.

Production has five hundred orders on the page. Five hundred round-trips at 0.8 ms each — a realistic figure for a database in the same availability zone — is 400 ms of pure latency before the first byte of JSON. That's not the real cost, though. The real cost is that each of those 500 round-trips holds a pooled connection, so a HikariCP pool of 10 supports a small fraction of the throughput you sized it for, and the failure mode is pool-exhaustion timeouts on an unrelated endpoint at 3 a.m.

Two things conspire to hide it further.

**`spring.jpa.open-in-view` is still `true` by default in Spring Boot 4.** The persistence context stays open for the whole request, through the controller and through JSON serialization. So the lazy loads triggered by Jackson walking your entity graph succeed — silently, outside any transaction you wrote, after your service method has returned. Nothing throws. You just get slow. Turn it off and the same code throws `LazyInitializationException` at the exact line that caused the problem, which is what you want.

**And no, `FetchType.EAGER` does not fix it.** This is the single most common wrong fix. `EAGER` only guarantees the association is populated by the time the entity is returned; it says nothing about *how*. For `em.find()` Hibernate will usually manage a join. For a JPQL query, `select o from Order o` with an eager collection typically becomes exactly the same N+1 — Hibernate runs your query, then initializes each eager association per row. You've lost laziness and kept the N+1. And when it *does* join, eager collections on multiple associations give you a cartesian product on every single query, whether or not the caller needed the data.

Eager fetching moves the problem, it doesn't solve it. Map everything lazy, then fetch deliberately per use case. That's the whole philosophy of the four fixes below.

## Detecting it, properly

Intuition doesn't work here. Counting does.

Log the SQL with parameters. `spring.jpa.show-sql=true` is the wrong tool: it writes to stdout, bypasses your logging config, and doesn't show bound parameters. Use the logger categories instead — note that the bind-parameter category changed in Hibernate 6 and is `org.hibernate.orm.jdbc.bind`, not the `org.hibernate.type.descriptor.sql` you'll find in older articles.

**Code 3 — application.yml for a dev/test profile**

```yaml
spring:
  jpa:
    open-in-view: false              # make lazy loads fail where they happen
    properties:
      hibernate:
        generate_statistics: true    # dev/test only — it has a cost
        highlight_sql: true
        format_sql: false            # one query per line, so you can grep -c
 
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # bound parameters (Hibernate 6+ catego
    org.hibernate.stat: DEBUG            # per-session summary
```

With statistics on, every session logs a summary, and two counters in it name the problem directly:

```
Session Metrics {
    ... spent preparing 401 JDBC statements;
    100 collections fetched (lazy)
    300 entities fetched (lazy)
}
```

`collections fetched` and `entities fetched` count lazy initializations. Not "queries ran" — "something was loaded one-at-a-time because it wasn't there when you needed it". A read path fixed with a fetch join or an EntityGraph has both at zero. One fixed with batch fetching does not — the collections are still lazy, just loaded fifty at a time — which is a distinction the test section comes back to.

**Code 4 — reading it programmatically**

```java
Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
stats.clear();
 
service.report();
 
System.out.printf("statements=%d entityFetches=%d collectionFetches=%d%n",
    stats.getPrepareStatementCount(),
    stats.getEntityFetchCount(),        // lazy to-one loads
    stats.getCollectionFetchCount());   // lazy collection loads
```

If you don't want statistics enabled at all, a `StatementInspector` costs nothing and needs no extra dependency.

**Code 5 — counting selects with a StatementInspector**

```java
public class CountingStatementInspector implements StatementInspector {
 
    public static final AtomicInteger SELECTS = new AtomicInteger();
 
    @Override
    public String inspect(String sql) {
        if (sql.regionMatches(true, 0, "select", 0, 6)) SELECTS.incrementAndGet(
        return sql;
    }
}
```

```
spring.jpa.properties.hibernate.session_factory.statement_inspector: com.acme.sh
```

In production you're looking at different signals: query-count fan-out in your tracing backend (OpenTelemetry's JDBC instrumentation makes an N+1 visually unmistakable — a span with a hundred identical children), and repeated identical statements with different single bind values in `pg_stat_statements`. If you want a CI gate rather than a dashboard, Hypersistence Optimizer and QuickPerf both exist for exactly this, and IntelliJ / JPA Buddy will flag some cases statically before you run anything.

## Fix 1 — join fetch

The direct approach: tell the query to bring the associations along.

**Code 6 — JPQL**

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
 
    @Query("""
        select o
        from Order o
        join fetch o.customer
        left join fetch o.lines l
        left join fetch l.product
        where o.createdAt >= :since
        """)
    List<Order> findWithLinesSince(Instant since);
}
```

A side note on `:since`. Binding a named parameter to the method argument without `@Param` works only if the interface was compiled with `-parameters`. The Spring Boot Maven and Gradle plugins turn that flag on for you; a plain compiler configuration does not, and the query fails at startup with a "parameter name not available" error. Add `@Param("since")` if you're not sure what compiles your code.

One statement comes out:

```sql
select o1_0.id, o1_0.created_at, c1_0.id, c1_0.name, ...,
       l1_0.id, l1_0.quantity, ..., p1_0.id, p1_0.name, ...
from orders o1_0
join customer c1_0 on c1_0.id = o1_0.customer_id
left join order_line l1_0 on o1_0.id = l1_0.order_id
left join product p1_0 on p1_0.id = l1_0.product_id
where o1_0.created_at >= ?
```

**Scheme B**

1. `SELECT o.*, l.*, p.* FROM orders o LEFT JOIN order_line l ON l.order_id = o.id LEFT JOIN product p ON p.id = l.product_id`

orders (`o.*`) + order_line (`l.*`) + product (`p.*`) → one result set of rows 1, 2, 3 ... 1,000 with columns `o.*` | `l.*` | `p.*` — Order columns repeated 10× each.

**1 round-trip, ~1,000 rows.**

![svg](<svgs/scheme-b-fetch-join.svg>)

Two details people get wrong. `join fetch` **vs** `left join fetch`: an inner join silently drops every order that has no lines. If the collection can be empty and you still want the parent, it must be `left`. And **`distinct` is no longer needed** to de-duplicate the root — Hibernate 6 and later de-duplicate root entities automatically, and the `hibernate.query.passDistinctThrough` setting that used to keep `DISTINCT` out of the SQL is gone. Writing `select distinct o` today means you're asking the database to sort and de-duplicate a thousand wide rows for nothing.

**Code 7 — the same thing type-safely, via Criteria**

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> q = cb.createQuery(Order.class);
Root<Order> o = q.from(Order.class);
 
o.fetch(Order_.customer);
Fetch<Order, OrderLine> lines = o.fetch(Order_.lines, JoinType.LEFT);
lines.fetch(OrderLine_.product, JoinType.LEFT);
 
q.select(o).where(cb.greaterThanOrEqualTo(o.get(Order_.createdAt), since));
List<Order> result = em.createQuery(q).getResultList();
```

### The pagination trap — and why what you've read about it is out of date

This is the piece of N+1 folklore that changed in 2026, so it's worth being precise about versions.

For roughly a decade, combining `setMaxResults()` with a collection `join fetch` produced this:

```
WARN HHH90003004: firstResult/maxResults specified with collection fetch; applyi
```

The reason is structural: after the join, one order is many rows, so `LIMIT 20` in SQL would return a fraction of twenty orders. Hibernate's answer was to drop the limit from the SQL, fetch **the entire result set**, and paginate the list in the JVM. A `Page` of 20 that quietly loads 400,000 rows. On Hibernate 6 and 7.0–7.3 this is still the behavior, and the standard defense is to turn the warning into a failure:

```
spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch: 
```

**Hibernate 7.4 fixes it.** From 7.4, the limit is applied as part of the SQL query — Hibernate wraps the root selection in a subquery that carries the `limit`/`offset`, then joins the collection against it. The 7.4 migration guide puts it plainly: when pagination or a limit is used with a query that fetches a collection, the limit is now processed as part of the SQL query. It works on every supported database that allows limits and offsets in subqueries, which is all of them except Sybase ASE. And if you are on Spring Boot 4.1 you already have it: 4.1.1 manages Hibernate 7.4.5.Final and Jakarta Persistence 3.2.0, so the fix is in the box. If you need the old in-memory behavior back, there's a query hint, `org.hibernate.limitInMemory` — and you almost certainly don't.

So: on 7.4+, paginating a collection fetch join is safe. On anything earlier, it is not, and the ids-then-fetch pattern remains the correct workaround.

**Code 8 — the two-query pattern (still correct on ≤ 7.3, still useful on 7.4)**

```java
// 1. page the ids — one row per order, so LIMIT means what it says
List<Long> ids = em.createQuery(
        "select o.id from Order o order by o.createdAt desc", Long.class)
    .setFirstResult(page * size)
    .setMaxResults(size)
    .getResultList();
 
// 2. fetch the full graph for exactly those ids
List<Order> orders = em.createQuery("""
        select o from Order o
        left join fetch o.lines
        where o.id in :ids
        order by o.createdAt desc
        """, Order.class)
    .setParameter("ids", ids)
    .getResultList();
```

Two more traps that 7.4 did not remove.

**MultipleBagFetchException.** Fetch-join two `List` collections in one query and Hibernate refuses outright: it cannot tell which row belongs to which bag. The usual advice is "change them to `Set`", which works and is a lie by omission — you've traded an exception for an unbounded cartesian product. The honest fixes are `@OrderColumn` (if the order is genuinely persistent), or fetch-joining one collection and batching the other, which is the next section.

**Cartesian size.** 100 orders × 10 lines × 3 tags is 3,000 rows on the wire to reconstruct 100 objects. Fetch join is not free; it trades round-trips for bytes. When the collections are large, the trade goes the wrong way.

**Verdict:** best for to-one associations and a single collection. On 7.4+, pagination is no longer a reason to avoid it.

## Fix 2 — EntityGraph

Fetch join welds the fetch plan into the query. If two endpoints need the same rows in different shapes, you write the query twice. EntityGraph separates the two: what to select stays in the query, what to load becomes a separate, reusable, composable declaration.

**Scheme C**

Query: `find Order` — one repository method feeding three graphs:
- graph `"summary"` → `[customer]` → order → customer → **1 SQL, 1 join**
- graph `"with-lines"` → `[customer, lines]` → order → customer → lines → **1 SQL, 2 joins**
- graph `"full"` → `[customer, lines.product, lines.discounts]` → order → customer → lines → product, discounts → **1 SQL, 4 joins**

**One repository method. Three fetch plans. Zero duplicated JPQL.**

![svg](<svgs/scheme-c-entitygraph.svg>)

**Code 9 — a named graph on the entity**

```java
@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "Order.withLinesAndProducts",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "lines", subgraph = "lines")
    },
    subgraphs = @NamedSubgraph(
        name = "lines",
        attributeNodes = @NamedAttributeNode("product")))
public class Order { ... }
```

### The JPA 3.2 signature almost every article gets wrong

Jakarta Persistence 3.2 replaced the stringly-typed hint map with real API, and the shape of it is not what most people assume. EntityGraph is not passed as a `FindOption`. It gets its own overload, as the first argument:

```java
<T> T find(Class<T> entityClass, Object primaryKey, FindOption... options)
<T> T find(EntityGraph<T> entityGraph, Object primaryKey, FindOption... options)
```

**Code 10 — dynamic graph, JPA 3.2 style**

```java
EntityGraph<Order> graph = em.createEntityGraph(Order.class);
graph.addAttributeNodes("customer");
graph.addSubgraph("lines").addAttributeNodes("product");
 
// JPA 3.2 / Hibernate 7 — graph first, no hint strings
Order order = em.find(graph, id);
 
// combine freely with other FindOptions
Order fresh = em.find(graph, id, CacheRetrieveMode.BYPASS, LockModeType.OPTIMIST
 
// the pre-3.2 form, for readers still on Hibernate 6 / Spring Boot 3
Order legacy = em.find(Order.class, id,
        Map.of("jakarta.persistence.fetchgraph", graph));
```

Worth knowing: the `find(EntityGraph, ...)` overload interprets the graph as a **load graph**, not a fetch graph. The distinction — fetchgraph means "everything not listed is lazy", loadgraph means "everything not listed keeps its mapped fetch type" — still matters, and it's why the two legacy hint names existed. In practice Hibernate treats unlisted basic attributes as eager under both, so the difference only really bites on associations.

**Code 11 — graph + query, and the multi-id load**

```java
List<Order> orders = em.unwrap(Session.class)
    .createSelectionQuery("from Order o where o.createdAt >= :since", Order.clas
    .setParameter("since", since)
    .setEntityGraph(graph, GraphSemantic.FETCH)
    .getResultList();
 
// Hibernate 7: load a whole page of ids with a graph, in one statement
List<Order> page = em.unwrap(Session.class)
    .findMultiple(graph, ids);
```

That `findMultiple(EntityGraph<E>, List<?> ids, FindOption...)` overload is the clean modern form of step 2 in Code 8 — it batches the id list and applies the graph, without you writing the `where id in :ids` by hand.

**Code 12 — Spring Data JPA**

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
 
    @EntityGraph(attributePaths = {"customer", "lines", "lines.product"})
    List<Order> findByCreatedAtAfter(Instant since);
 
    @EntityGraph("Order.withLinesAndProducts")
    Optional<Order> findWithGraphById(Long id);
 
    @EntityGraph(attributePaths = "customer")   // to-one only
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);
}
```

The thing to internalize: under the hood, Hibernate implements entity graphs as joins. A graph is not a different loading strategy, it's a different way to spell a fetch join. Which means every caveat from Fix 1 applies unchanged — cartesian products, bag restrictions, and (on pre-7.4) in-memory pagination. People are routinely surprised by this; turn on SQL logging once and the surprise goes away.

Prefer a graph over a fetch join when: the same query serves several shapes; you're loading by id; or you're on a Spring Data derived query where there is no JPQL to add `join fetch` to.

## Fix 3 — batch fetching, the one you should turn on globally

Fixes 1 and 2 both change the shape of the main query. Batch fetching doesn't touch it at all — and that is precisely why it's the most useful of the four.

The mechanism: an association stays lazy, but when Hibernate is forced to initialize one proxy, it looks in the persistence context for other uninitialized proxies of the same type and loads up to N of them in a single statement. N+1 becomes ⌈N/batch⌉ + 1.

**Scheme D**

1. `SELECT * FROM orders LIMIT 100` → orders
   order.getLines() on the first order
2. `SELECT * FROM order_line WHERE order_id IN (1..50)`
3. `SELECT * FROM order_line WHERE order_id IN (51..100)` → order_line
   line.getProduct()
4–7. `SELECT * FROM product WHERE id IN (...50 ids...)` → product
8–9. `SELECT * FROM customer WHERE id IN (...50 ids...)` → customer

**1 + 2 (lines) + 4 (products) + 2 (customers) = 9 queries, not 401.**

![svg](<svgs/scheme-d-batch-fetching.svg>)

**Code 13 — per association or per entity**

```java
@Entity
@BatchSize(size = 50)             // every lazy proxy of Product batches
public class Product { ... }
 
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 50)         // this collection batches
    private List<OrderLine> lines = new ArrayList<>();
}
```

**Code 14 — or, better, globally**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50
```

That one line, with no change to the service in Code 2, turns 401 queries into nine:

```sql
select o1_0.id, ... from orders o1_0
select c1_0.id, ... from customer   c1_0 where c1_0.id       = any (?)
select l1_0.order_id, ... from order_line l1_0 where l1_0.order_id = any (?)
select l1_0.order_id, ... from order_line l1_0 where l1_0.order_id = any (?)
select p1_0.id, ... from product   p1_0 where p1_0.id       = any (?)
...
```

Whether you get `= any (?)` or a classic `in (?, ?, ?, …)` list is the dialect's call: Hibernate asks it through `Dialect.useArrayForMultiValuedParameters()`. When the answer is yes, the whole id batch travels as a single array parameter, and that is worth more than it looks — one SQL string instead of one per distinct batch length, so the statement cache and the planner both stop thrashing. On PostgreSQL, where the numbers below were taken, that makes the old Hibernate 5 "padded batch size" folklore moot. On dialects that answer no (MySQL, Oracle, SQL Server) it isn't quite: Hibernate still emits `in` lists of a few fixed lengths as the remaining batch shrinks, so distinct batch sizes still mean distinct statements in the cache. It's a smaller problem than it was, not a vanished one. Turn on SQL logging once and you will see which form your database gets.

Should you set it globally? Yes, for almost every application. It is the only one of the four fixes that is a safety net rather than a decision: it never changes results, never changes the main query, never breaks pagination, and it converts the worst case of any N+1 you haven't found yet from linear to linear-divided-by-fifty. The reason it isn't on by default is essentially historical — enabling it would change the SQL emitted by every existing application on upgrade, and Hibernate is conservative about that.

It composes with pagination precisely because it leaves the main query alone: `LIMIT` stays in SQL where it belongs, and the collections arrive afterwards in batches. This is the answer to "paginated list plus a collection" — on every Hibernate version, including the ones before 7.4 made fetch joins page correctly.

**Code 15 — the sibling strategy: subselect fetching**

```java
@OneToMany(mappedBy = "order")
@Fetch(FetchMode.SUBSELECT)
private List<OrderLine> lines = new ArrayList<>();
```

```sql
-- the original query
select o1_0.id, ... from orders o1_0 where o1_0.created_at >= ?
 
-- on first access to any order.lines: exactly one more query
select l1_0.order_id, ... from order_line l1_0
where l1_0.order_id in (select o1_0.id from orders o1_0 where o1_0.created_at >=
```

Two queries total, regardless of how many owners — better than batching when the result set is large and unpaginated. The cost is that it re-executes your original `where` clause as a subquery: expensive if that clause was expensive, and the two result sets can drift apart without a consistent snapshot. Great for "load this filtered set and all their lines". Not great with `LIMIT`.

The honest trade-off for all of Fix 3: it's still lazy. It still needs an open persistence context, and the query count still depends on data volume rather than being a constant. It reduces the damage; it doesn't make the fetch plan explicit.

## Fix 4 — stop loading entities

The fix people forget, and usually the best one.

A read-only endpoint needs five fields. Loading an object graph to produce them means paying for entity instantiation, the persistence context's identity map, dirty-checking snapshots, and every column of every table involved — to then throw all of it away after serialization.

**Code 16 — constructor projection into a record**

```java
public record OrderSummary(Long id, String customerName, long lineCount, BigDeci
 
public interface OrderRepository extends JpaRepository<Order, Long> {
 
    @Query(value = """
        select new com.acme.shop.OrderSummary(
            o.id, o.customer.name, count(l), coalesce(sum(l.price * l.quantity),
        from Order o
        left join o.lines l
        where o.createdAt >= :since
        group by o.id, o.customer.name, o.createdAt
        order by o.createdAt desc
        """,
       countQuery = "select count(o) from Order o where o.createdAt >= :since")
    Page<OrderSummary> summaries(Instant since, Pageable pageable);
}
```

One query. One row per order, so `Pageable` works the way you expect. No persistence context, no lazy anything, no N+1 possible — you cannot lazily load what you never mapped.

Two details in that code are not decoration. First, the explicit `countQuery`. Spring Data builds the count query for a `Page` by rewriting the select clause of your JPQL, and for a query that combines a constructor expression with `group by` that rewrite is unreliable: depending on the version it fails to parse or counts one row per group, so the `Page` reports a total that has nothing to do with the number of orders. Give every `Page`-returning `@Query` its own `countQuery`, and leave the `group by` out of it. Second, `o.createdAt` in the `group by`. PostgreSQL would accept the `order by` without it, because the column is functionally dependent on the primary key you're already grouping on; Oracle, SQL Server and H2 reject it. If you only ever run on Postgres you can drop it, but then say so in a comment.

**Code 17 — Spring Data interface projections, and their limit**

```java
public interface OrderView {
    Long getId();
    CustomerView getCustomer();          // joined into the same select
    interface CustomerView { String getName(); }
}
 
List<OrderView> findByCreatedAtAfter(Instant since);
```

Closed projections with to-one paths compile to a single select. Collection paths inside a projection do not — nested one-to-many in a DTO is still a fetch-shaped problem, and plain JPA has no good answer for it. That's the gap where Blaze-Persistence Entity Views and jOOQ's `MULTISET` earn their keep; both build genuinely nested result structures in one round-trip.

The rule is short: **entities for writes, projections for reads.** Most N+1 bugs live on read paths that never needed an entity in the first place.

## The numbers

Setup: Postgres 17 in Testcontainers, Hibernate 7.4, Spring Boot 4.1, Java 25. 1,000 orders, 10 lines each, one customer per order, 200 distinct products shared across the lines, database in the same network as the application. Workload: load 100 orders with their lines and each line's product.

These figures are representative, not measured on your hardware — they're here to show the shape of the difference, which is what generalizes. The ratios hold; the milliseconds won't. Run it yourself before quoting them.

```
Strategy                     Queries    Rows fetched    p50 (ms)
 ───────────────────────────────────────────────────────────────
 Naive lazy                    401       ~1,400          ~280
 Fetch join (lines+product)      1       ~1,000           ~28
 EntityGraph (lines.product)     1       ~1,000           ~28
 @BatchSize(50)                  9       ~1,400           ~26
 DTO projection                  1          100           ~11
 ───────────────────────────────────────────────────────────────
```

Read that table honestly. Fetch join and EntityGraph are the same row because they are the same thing — one emits the SQL the other spells differently. Batch fetching lands within noise of both here, and pulls clearly ahead the moment you add `LIMIT`, or a second collection, or run on Hibernate 7.3. And the DTO projection beats everything by more than a factor of two, because it's the only row that doesn't transfer data nobody asked for.

The gap that matters isn't between the fixes. It's between the first row and all the others.

## Making sure it never comes back

Every fix above is one refactor away from being undone. Someone adds a field to a response DTO, touches a lazy association, and the endpoint is back to 151 queries — with no failing test, because the output is still correct. It's only slow.

So assert on the query count.

**Code 18 — the regression test**

```java
@SpringBootTest
@Testcontainers
class OrderReportServiceQueryCountTest {
 
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");
 
    @Autowired OrderReportService service;
    @Autowired EntityManagerFactory emf;
    @Autowired TestDataFactory data;
 
    @Test
    void report_doesNotRunOneQueryPerOrder() {
        data.orders(100).withLinesEach(10).acrossProducts(200);          // ≥ 2×
 
        Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
        stats.clear();
 
        service.report();
 
        assertThat(stats.getPrepareStatementCount())
            .as("query count for 100 orders")
            .isLessThanOrEqualTo(12);                // 1 + 2 customers + 2 line
 
        // with a fetch join or an EntityGraph you can be stricter:
        // assertThat(stats.getCollectionFetchCount()).isZero();
    }
}
```

Pick the assertion that matches the fix. The statement count always works: it catches regressions in magnitude whatever strategy you chose. `getCollectionFetchCount() == 0` is stricter, but it is only valid when the collections arrive with the query — a fetch join or an EntityGraph. Under `default_batch_fetch_size` that counter is not zero and never will be: the collections are still loaded lazily, just fifty at a time. Assert zero there and you fail a correctly fixed endpoint.

The critical detail is the fixture size: seed more rows than your batch size. A test with 10 orders and `default_batch_fetch_size=50` passes identically whether the code is fixed or broken, because one batch covers everything. Use at least twice the batch size. And the to-one side needs the same treatment: pin the cardinality. The threshold of 12 above assumes about 200 distinct products, which is four product batches of 50. A factory that quietly creates a fresh product for every line gives you 1,000 products, 20 product batches, and a red test on code that is correctly fixed. Whatever your fixture API looks like, the number of distinct to-one targets is part of the assertion, so make it explicit in the test rather than a property of whoever wrote the factory.

If you prefer annotations to assertions, QuickPerf's `@ExpectSelect(n)` does the same job declaratively, and datasource-proxy's `QueryCountHolder` works if you'd rather count at the JDBC layer than trust Hibernate's own statistics.

One such test per list endpoint. That's the whole discipline.

## The decision table

```
SITUATION                      USE                       WATCH FOR
──────────────────────────────────────────────────────────────────────────
One entity by id + graph       em.find(graph, id)        nothing — easy case
List, to-one only              join fetch / graph        inner vs left join
List, 1 collection, no page    join fetch / graph        cartesian size
List, 1 collection, paged      7.4+: join fetch is ok    7.3 pages in memory
                               7.3-: batch fetching
2+ collections                 join fetch the biggest,   MultipleBagFetch-
                               batch the rest            Exception
Big unpaged filtered set       @Fetch(SUBSELECT)         re-runs your where
Read-only screen or API        DTO projection (record)   nested: Blaze/jOOQ
Legacy code, no refactor       batch_fetch_size=50       still lazy
"Is it actually fixed?"        Statistics + query count  seed >= 2x batch
──────────────────────────────────────────────────────────────────────────
```

**Scheme E — the same thing as a flow**

- Do you need entities at all — will you modify them? → **No** → DTO projection (record / interface projection). **Done.** ✓
- **Yes** ↓ Is the result paginated? → **Yes** → to-one: `fetch join / EntityGraph`; collections: `@BatchSize / default_batch_fetch_size` (or fetch join, if you're on Hibernate 7.4+)
- **No** ↓ How many collections do you need?
  - **0–1** → `join fetch` or `EntityGraph` — one SQL statement.
  - **2+** → fetch join the **biggest**, batch the rest. (never two `List` fetch joins → `MultipleBagFetchException`)

**Always:** `default_batch_fetch_size=50` as the safety net + one query-count assertion per list endpoint in CI.

![svg](<svgs/scheme-e-decision-flow.svg>)

## Takeaways

- **N+1 is the default, not a bug.** Every lazy association is a candidate.
- **`FetchType.EAGER` is not a fix.** It usually produces the same N+1, minus your ability to choose.
- **Measure with `Statistics` and query counts, not intuition.** `collections fetched` and `entities fetched` name the problem directly.
- **To-one plus a single collection → `join fetch` or `EntityGraph`.** They're the same mechanism; pick by reusability.
- **On Hibernate 7.4+, pagination with a collection fetch join is finally safe.** On 7.3 and earlier it silently pages in memory — keep `fail_on_pagination_over_collection_fetch=true` until you upgrade.
- **Paginated lists and multi-collection graphs → `default_batch_fetch_size`.** Set it to 50 globally in every project you own.
- **Read-only endpoints → DTO projections.** Stop loading entities you're going to throw away.
- **Turn `open-in-view` off,** so the problem surfaces at the line that caused it.
- **Put a query-count assertion in CI,** with a fixture bigger than your batch size and the to-one cardinality pinned.

## Further reading

- Hibernate ORM 7.4 User Guide — the Fetching chapter, and the 7.4 migration guide for the limits-and-fetch-joins change
- Jakarta Persistence 3.2 specification — the Entity Graphs chapter, and the new `find` / `findMultiple` overloads
- Vlad Mihalcea and Thorben Janssen on fetching strategies — still the deepest free material on the subject
- Blaze-Persistence Entity Views, and jOOQ's `MULTISET`, for nested projections JPA can't express

Link the 7.4 docs, not the 5.x ones. Half the pain in this topic comes from advice written for a version nobody runs any more.

---

Source: "N+1 in Hibernate Is Not a Bug. It's a Default." by Alex Klimenko, Medium (Sep 2026)
https://medium.com/@alxkm/n-1-in-hibernate-is-not-a-bug-its-a-default-17d3ef0cb047
