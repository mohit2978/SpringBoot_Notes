

## 1. Spring Data Repository Hierarchy Overview

In Spring Data JPA, you do not write DAO implementations manually. Instead, you define an **interface** that extends one of Spring Data's built-in repository interfaces, and Spring automatically generates a proxy implementation (`SimpleJpaRepository`) at runtime.

### The Core Hierarchy (Conceptual / Classic Spring Boot 2.x)

```
                  Repository<T, ID>              (Marker interface — no methods)
                         ↑
                CrudRepository<T, ID>            (Basic CRUD operations)
                         ↑
         PagingAndSortingRepository<T, ID>       (Adds Pageable & Sort methods)
                         ↑
                JpaRepository<T, ID>             (Adds JPA-specific operations: flush, batch delete, List returns)
                         ↑
           YourCustomRepository (e.g., ProductRepository)
```

### Updated Hierarchy in Spring Boot 3.x (Spring Data 3.0+)

In **Spring Data 3.0 (Spring Boot 3+)**, `PagingAndSortingRepository` was decoupled from `CrudRepository` so developers can create read-only paging repositories without exposing write/delete methods, and `List*` interfaces were introduced to return `List<T>` instead of `Iterable<T>`:

```
                          Repository<T, ID>
                         ↗                 ↖
        CrudRepository<T, ID>           PagingAndSortingRepository<T, ID>
                 ↑                                       ↑
      ListCrudRepository<T, ID>         ListPagingAndSortingRepository<T, ID>
                         ↖                 ↗
                        JpaRepository<T, ID>   (Also extends QueryByExampleExecutor<T>)
```

> **Note:** For day-to-day usage, `JpaRepository<T, ID>` still inherits **all** methods from `CrudRepository` and `PagingAndSortingRepository`.

---

## 2. `CrudRepository` — Basic CRUD Operations

`CrudRepository<T, ID>` is part of the **Spring Data Commons** module (not specific to JPA — also used by Spring Data MongoDB, Redis, JDBC, etc.). It provides standard **Create, Read, Update, and Delete** functionality.

- **`T`**: The Entity type managed by the repository (e.g., `Product`).
- **`ID`**: The data type of the Entity's primary key (`@Id`, e.g., `Long`).

### Built-in Methods of `CrudRepository<T, ID>`

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {

    // CREATE / UPDATE
    <S extends T> S save(S entity);
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);

    // READ
    Optional<T> findById(ID id);
    boolean existsById(ID id);
    Iterable<T> findAll();
    Iterable<T> findAllById(Iterable<ID> ids);
    long count();

    // DELETE
    void deleteById(ID id);
    void delete(T entity);
    void deleteAllById(Iterable<? extends ID> ids);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();
}
```

### Code Example Using `CrudRepository`

```java
public interface CustomerCrudRepository extends CrudRepository<Customer, Long> {
    // Only basic CRUD methods are available out of the box
}

@Service
public class CustomerService {

    @Autowired
    private CustomerCrudRepository customerRepository;

    public void demoCrudOperations() {
        // 1. Save a single entity (INSERT if id is null, UPDATE if id exists)
        Customer c1 = new Customer("Rahul", "rahul@example.com");
        Customer saved = customerRepository.save(c1);

        // 2. Find by ID
        Optional<Customer> found = customerRepository.findById(saved.getId());

        // 3. Find all — NOTE: returns Iterable<Customer>, NOT List<Customer>!
        Iterable<Customer> allCustomers = customerRepository.findAll();
        List<Customer> list = new ArrayList<>();
        allCustomers.forEach(list::add); // Extra conversion needed if you want a List

        // 4. Count total rows
        long total = customerRepository.count();

        // 5. Delete by ID
        customerRepository.deleteById(saved.getId());
    }
}
```

### Limitations of `CrudRepository`:
1. `findAll()`, `findAllById()`, and `saveAll()` return `Iterable<T>` instead of `List<T>` (unless using `ListCrudRepository` in Spring Boot 3+).
2. **No built-in Pagination or Sorting** (`findAll(Pageable)` or `findAll(Sort)` do not exist here).
3. **No JPA-specific methods** like `flush()`, `saveAndFlush()`, or `deleteAllInBatch()`.

---

## 3. `PagingAndSortingRepository` — Pagination & Sorting Abstraction

`PagingAndSortingRepository<T, ID>` is also part of **Spring Data Commons**. Its sole responsibility is to add **pagination** and **sorting** abstractions on top of repository queries.

### Built-in Methods of `PagingAndSortingRepository<T, ID>`

```java
public interface PagingAndSortingRepository<T, ID> extends Repository<T, ID> {

    // Fetch all entities sorted by the given Sort object
    Iterable<T> findAll(Sort sort);

    // Fetch a specific page of entities matching the Pageable restriction
    Page<T> findAll(Pageable pageable);
}
```

*(In Spring Boot 2.x, `PagingAndSortingRepository` also inherited all `CrudRepository` methods. In Spring Boot 3.x, it only declares the 2 methods above unless combined with `CrudRepository` or `JpaRepository`.)*

### Code Example Using `PagingAndSortingRepository`

```java
public interface ProductPagingRepository extends PagingAndSortingRepository<Product, Long> {
}

@Service
public class ProductReadService {

    @Autowired
    private ProductPagingRepository productPagingRepository;

    public Page<Product> getProductsPage(int pageNo, int pageSize) {
        // Page index starts at 0!
        Pageable pageable = PageRequest.of(pageNo, pageSize, Sort.by("price").ascending());
        return productPagingRepository.findAll(pageable);
    }
}
```

### Limitations of `PagingAndSortingRepository`:
1. `findAll(Sort)` returns `Iterable<T>` (in `PagingAndSortingRepository`), requiring conversion if you need `List<T>` methods.
2. Does **not** provide JPA-specific persistence context operations (`flush()`, `saveAndFlush()`, `deleteAllInBatch()`, `getReferenceById()`).

---

## 4. `JpaRepository` — Full-Featured JPA Repository

`JpaRepository<T, ID>` belongs to the **Spring Data JPA** module (`org.springframework.data.jpa.repository.JpaRepository`).
It combines:
- All CRUD methods from `CrudRepository` (via `ListCrudRepository`)
- All Pagination and Sorting methods from `PagingAndSortingRepository` (via `ListPagingAndSortingRepository`)
- Query By Example methods from `QueryByExampleExecutor`
- **JPA-specific methods** for controlling the Hibernate Persistence Context (L1 cache flushing, batch deletes, lazy reference proxies).

### Built-in Methods Added / Overridden by `JpaRepository<T, ID>`

```java
public interface JpaRepository<T, ID>
        extends ListCrudRepository<T, ID>,
                ListPagingAndSortingRepository<T, ID>,
                QueryByExampleExecutor<T> {

    // 1. Overridden to return List<T> instead of Iterable<T>
    List<T> findAll();
    List<T> findAll(Sort sort);
    List<T> findAllById(Iterable<ID> ids);
    <S extends T> List<S> saveAll(Iterable<S> entities);

    // 2. Persistence Context (L1 Cache) Flushing
    void flush();
    <S extends T> S saveAndFlush(S entity);
    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);

    // 3. Batch Deletion (Executes a single SQL DELETE query)
    void deleteAllInBatch(Iterable<T> entities);
    void deleteAllByIdInBatch(Iterable<ID> ids);
    void deleteAllInBatch();

    // 4. Lazy Proxy Fetching (No immediate SQL SELECT)
    T getReferenceById(ID id); // Replaces deprecated getOne(id) and getById(id)
}
```

### Code Example Using `JpaRepository`

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
}

@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Transactional
    public void demoJpaFeatures() {
        // 1. Returns List<Product> directly — no Iterable conversion needed!
        List<Product> products = productRepository.findAll();

        // 2. saveAndFlush() — immediately synchronizes with DB instead of waiting for transaction commit
        Product p = new Product("MacBook Pro", 1999.99);
        productRepository.saveAndFlush(p);

        // 3. getReferenceById() — returns a lazy proxy without hitting the DB immediately
        // Useful for setting foreign key relationships without an extra SELECT query!
        Product proxyProduct = productRepository.getReferenceById(10L);

        // 4. deleteAllInBatch() — deletes all rows using ONE SQL query: DELETE FROM products
        productRepository.deleteAllInBatch();
    }
}
```

---

## 5. Clear Differences: `CrudRepository` vs `PagingAndSortingRepository` vs `JpaRepository`

### Side-by-Side Comparison Matrix

| Feature / Aspect | `CrudRepository<T, ID>` | `PagingAndSortingRepository<T, ID>` | `JpaRepository<T, ID>` |
| :--- | :--- | :--- | :--- |
| **Module / Package** | `spring-data-commons` | `spring-data-commons` | `spring-data-jpa` |
| **Technology Specific?** | Store-agnostic (works with JPA, Mongo, Redis, JDBC) | Store-agnostic | **JPA-specific** (Hibernate / EclipseLink) |
| **Basic CRUD (`save`, `findById`, `delete`)** | Yes | Yes (in SB 2.x) / Separate in SB 3.x | **Yes** |
| **Pagination & Sorting (`Pageable`, `Sort`)** | No | **Yes** | **Yes** |
| **Return Type of `findAll()`** | `Iterable<T>` | `Iterable<T>` (for `findAll(Sort)`) | **`List<T>`** |
| **Persistence Context `flush()` / `saveAndFlush()`** | No | No | **Yes** |
| **Batch Deletion (`deleteAllInBatch()`)** | No (only row-by-row `deleteAll()`) | No | **Yes** (single SQL `DELETE`) |
| **Lazy Proxy (`getReferenceById()`)** | No (only eager `findById()`) | No | **Yes** |
| **Query By Example (`Example<T>`)** | No | No | **Yes** |
| **When to Use?** | Simple NoSQL/JDBC apps needing only basic CRUD | When you only need pagination/sorting abstraction | **Default choice for almost all Spring Boot JPA apps** |

---

## 6. Deep-Dive: Critical Behavioral Differences Under the Hood

Interviewers frequently ask **not just what methods exist**, but **how their SQL execution differs**. Here are the 4 most important differences:

### Difference 1: `deleteAll()` (`CrudRepository`) vs `deleteAllInBatch()` (`JpaRepository`)

Suppose the `products` table has **1,000 rows**:

```java
// 1. CrudRepository's deleteAll()
productRepository.deleteAll();
```
**What happens in SQL (`deleteAll()`):**
1. Executes `SELECT * FROM products` to load all 1,000 entities into the L1 cache.
2. Loops over each entity and fires **1,000 individual `DELETE` queries**:
   ```sql
   SELECT id, name, price FROM products;
   DELETE FROM products WHERE id = 1;
   DELETE FROM products WHERE id = 2;
   ...
   DELETE FROM products WHERE id = 1000;
   ```
- **Pros:** Triggers JPA lifecycle callbacks (`@PreRemove`, `@PostRemove`) and cascades (`CascadeType.REMOVE`).
- **Cons:** Extremely slow for large tables (N+1 delete problem) and can cause `OutOfMemoryError`.

```java
// 2. JpaRepository's deleteAllInBatch()
productRepository.deleteAllInBatch();
```
**What happens in SQL (`deleteAllInBatch()`):**
1. Executes **a single bulk SQL query** directly against the database:
   ```sql
   DELETE FROM products;
   ```
- **Pros:** Massively faster — 1 query instead of 1,001 queries.
- **Cons:** Bypasses the L1 persistence context and does **not** trigger `@PreRemove` lifecycle hooks.

---

### Difference 2: `save()` (`CrudRepository`) vs `saveAndFlush()` (`JpaRepository`)

- **`save(entity)`**: Persists the entity into the **L1 Persistence Context (in-memory)**, and Hibernate delays executing the actual SQL `INSERT`/`UPDATE` until the transaction commits or before a query requires synchronization (Write-Behind optimization).
- **`saveAndFlush(entity)`**: Persists the entity AND **immediately forces Hibernate to execute the SQL `INSERT`/`UPDATE` statement** to the database right at that line of code (though it is still uncommitted until the transaction ends).

---

### Difference 3: `findById(id)` (`CrudRepository`) vs `getReferenceById(id)` (`JpaRepository`)

- **`findById(id)`**: Immediately checks L1/L2 cache and hits the database with `SELECT * FROM table WHERE id = ?`. Returns `Optional<T>`.
- **`getReferenceById(id)`**: Does **NOT** hit the database! It returns a lightweight **Hibernate proxy object** containing only the `id`. A `SELECT` query is fired only if you call a getter for a non-ID field (e.g., `proxy.getName()`).
  - **Best Use Case for `getReferenceById()`:** Setting a `@ManyToOne` foreign key when saving a child entity without wasting a `SELECT` query on the parent table:

```java
@Transactional
public Order createOrder(Long customerId, BigDecimal amount) {
    // NO SELECT query on customers table! Just creates a proxy reference with id = customerId
    Customer customerProxy = customerRepository.getReferenceById(customerId);

    Order order = new Order();
    order.setAmount(amount);
    order.setCustomer(customerProxy); // Sets customer_id foreign key on Order

    return orderRepository.save(order); // Only 1 INSERT into orders!
}
```

---

### Difference 4: `Iterable<T>` (`CrudRepository`) vs `List<T>` (`JpaRepository`)

- **`CrudRepository.findAll()`** returns `Iterable<T>`. You cannot directly call `.size()`, `.get(0)`, or `.stream()` without converting it to a `List` or using `StreamSupport.stream(iterable.spliterator(), false)`.
- **`JpaRepository.findAll()`** returns `List<T>` directly, making it ready for `.stream()`, indexing, and JSON serialization without boilerplate conversion.

---

## 7. Sorting in Spring Data JPA (With Clear Examples)

Sorting allows you to order query results dynamically using `org.springframework.data.domain.Sort` without hardcoding `ORDER BY` clauses in multiple repository methods.

### Setup: Sample `Product` Entity & Repository

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String category;
    private Double price;
    private Double rating;
    private LocalDateTime createdAt;
    // Constructors, Getters, Setters
}

public interface ProductRepository extends JpaRepository<Product, Long> {
    // Derived query accepting dynamic Sort
    List<Product> findByCategory(String category, Sort sort);

    // Static sorting embedded in method name
    List<Product> findByCategoryOrderByPriceDesc(String category);

    // Custom JPQL query with dynamic Sort
    @Query("SELECT p FROM Product p WHERE p.price >= :minPrice")
    List<Product> findExpensiveProducts(@Param("minPrice") Double minPrice, Sort sort);
}
```

---

### Example 1: Single-Field Sorting (Ascending & Descending)

```java
// 1A. Sort by price Ascending (Default direction is ASC)
Sort sortByPriceAsc = Sort.by("price").ascending();
List<Product> cheapestFirst = productRepository.findAll(sortByPriceAsc);
// Generated SQL: SELECT * FROM products ORDER BY price ASC

// 1B. Sort by price Descending
Sort sortByPriceDesc = Sort.by(Sort.Direction.DESC, "price");
// Or: Sort.by("price").descending();
List<Product> expensiveFirst = productRepository.findAll(sortByPriceDesc);
// Generated SQL: SELECT * FROM products ORDER BY price DESC
```

---

### Example 2: Multi-Field Sorting

#### Case A: Multiple fields in the SAME direction
```java
// Sort by category ASC, and if category is same, sort by price ASC
Sort sort = Sort.by(Sort.Direction.ASC, "category", "price");
List<Product> products = productRepository.findAll(sort);
// Generated SQL: SELECT * FROM products ORDER BY category ASC, price ASC
```

#### Case B: Multiple fields in DIFFERENT directions
Suppose you want `rating` **DESC** (highest rated first), and for products with the same rating, `price` **ASC** (cheapest first):

```java
// Approach 1: Chaining with .and()
Sort sort1 = Sort.by("rating").descending()
                 .and(Sort.by("price").ascending());

// Approach 2: Using Sort.Order objects
Sort sort2 = Sort.by(
    Sort.Order.desc("rating"),
    Sort.Order.asc("price")
);

List<Product> products = productRepository.findAll(sort1);
// Generated SQL: SELECT * FROM products ORDER BY rating DESC, price ASC
```

---

### Example 3: Case-Insensitive Sorting & Null Handling

By default, database sorting is case-sensitive (`'Zebra'` comes before `'apple'` in ASCII) and null ordering depends on the database vendor. You can control both with `Sort.Order`:

```java
Sort.Order nameOrder = Sort.Order.by("name")
        .with(Sort.Direction.ASC)
        .ignoreCase()   // Uses LOWER(name) in SQL
        .nullsLast();   // Puts NULL values at the bottom

Sort sort = Sort.by(nameOrder);
List<Product> products = productRepository.findAll(sort);
// Generated SQL: SELECT * FROM products ORDER BY LOWER(name) ASC NULLS LAST
```

---

### Example 4: Type-Safe Sorting with `Sort.TypedSort<T>` (No Hardcoded Strings!)

If you rename a field in `Product` (e.g., `price` → `unitPrice`), hardcoded strings like `Sort.by("price")` only fail at runtime. Use `Sort.sort(Class)` with Java method references for **compile-time safety**:

```java
Sort.TypedSort<Product> productSort = Sort.sort(Product.class);

Sort safeSort = productSort.by(Product::getRating).descending()
        .and(productSort.by(Product::getPrice).ascending());

List<Product> products = productRepository.findAll(safeSort);
// Generated SQL: SELECT * FROM products ORDER BY rating DESC, price ASC
```

---

### Example 5: Dynamic Sorting via REST Controller Parameter

```java
@GetMapping("/products/sorted")
public List<Product> getSortedProducts(
        @RequestParam(defaultValue = "price") String sortBy,
        @RequestParam(defaultValue = "asc") String direction) {

    Sort sort = direction.equalsIgnoreCase("desc")
            ? Sort.by(sortBy).descending()
            : Sort.by(sortBy).ascending();

    return productRepository.findAll(sort);
}
// Request: GET /products/sorted?sortBy=rating&direction=desc
```

---

## 8. Pagination in Spring Data JPA (With Clear Examples)

### 8.1 Advantages of Pagination

Imagine an `orders` or `products` table containing **5,000,000 rows**. If your API simply executes `orderRepository.findAll()` without pagination:
- ❌ **Application Crash (`OutOfMemoryError`):** Hibernate tries to hydrate 5,000,000 Java entity objects into heap memory (and the 1st-level persistence context cache), instantly exhausting JVM memory.
- ❌ **Database Starvation:** The database performs a massive full-table scan, thrashing the buffer pool, locking pages, and causing severe CPU and disk I/O spikes.
- ❌ **Network Saturation:** Serializing and transmitting a 500MB+ JSON payload across the network blocks client connections and consumes massive bandwidth.
- ❌ **Browser / Mobile Freeze:** Client-side DOMs or mobile UI renderers crash when attempting to parse and render millions of elements.

#### Key Advantages of Implementing Pagination:
1. 🛡️ **Memory Protection & Scalability:** Only a controlled subset (e.g., 10–50 entities) is kept in JVM memory at any given time.
2. ⚡ **Low Database Latency:** Queries scan and return only the requested slice of rows using index-backed `LIMIT` / `OFFSET`, keeping query response times in low milliseconds.
3. 🚀 **Reduced Network Bandwidth:** Payloads drop from hundreds of megabytes to a few kilobytes (e.g., ~15KB per page), ensuring instant network transfers.
4. 📱 **Superior User Experience (UX):** Enables modern UI patterns such as numbered pagers, "Load More" buttons, and smooth Infinite Scroll on web and mobile apps.
5. 🔒 **DoS Attack Prevention:** Prevents malicious actors or buggy clients from bringing down your application cluster by repeatedly issuing unbounded `SELECT *` requests.

---

### 8.2 What SQL Statement is Generated for Pagination?

When you invoke a Spring Data JPA repository method returning `Page<T>`:

```java
Pageable pageable = PageRequest.of(1, 10, Sort.by("price").descending());
Page<Product> page = productRepository.findAll(pageable);
```

Spring Data JPA and Hibernate execute **TWO SQL queries**:

#### 1. The Data Pagination Query
Fetches the actual rows for the requested page slice, applying sorting, `LIMIT`, and `OFFSET`:

```sql
-- MySQL / PostgreSQL / SQLite / MariaDB / H2:
SELECT id, name, category, price, rating, created_at
FROM products
ORDER BY price DESC
LIMIT 10 OFFSET 10;
```

#### 2. The Total Count Query
Executed automatically when returning `Page<T>` so Spring can calculate `totalElements` and `totalPages`:

```sql
SELECT COUNT(id) FROM products;
```

#### How Dialects Handle the Data Query Across Different Databases:

Hibernate automatically translates `Pageable` into the dialect-specific SQL syntax:

| Database | Generated SQL Clause | Example |
|:---|:---|:---|
| **MySQL / Postgres / SQLite / H2** | `LIMIT ? OFFSET ?` | `LIMIT 10 OFFSET 20` |
| **Oracle 12c+ / SQL Server 2012+** | ANSI SQL Standard: `OFFSET ? ROWS FETCH NEXT ? ROWS ONLY` | `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY` |
| **Legacy Oracle (pre-12c)** | Nested subquery with `ROWNUM` pseudo-column | `SELECT * FROM (SELECT a.*, ROWNUM rnum FROM (...) a WHERE ROWNUM <= 30) WHERE rnum > 20` |
| **Legacy SQL Server (pre-2012)** | Common Table Expression (CTE) with `ROW_NUMBER() OVER()` | `WITH CTE AS (SELECT ..., ROW_NUMBER() OVER(ORDER BY price) AS RowNum FROM products) SELECT * FROM CTE WHERE RowNum BETWEEN 21 AND 30` |

> 💡 **Notice with `Slice<T>`:** If your repository method returns `Slice<Product>` instead of `Page<Product>`, Hibernate executes **only ONE query**:
> ```sql
> SELECT id, name, category, price, rating, created_at
> FROM products
> ORDER BY price DESC
> LIMIT 11 OFFSET 10;   -- Notice LIMIT is (pageSize + 1) = 11!
> ```
> Hibernate fetches `11` rows instead of `10`. If an 11th row is returned, Hibernate knows `hasNext() == true` without ever executing an expensive `SELECT COUNT(*)` query!

---

### 8.3 Do We Need a Separate Call for the Next Page?

### **YES! Absolutely.**

Each page requires an **independent, separate request**. Here is why, and how it works:

#### Why Can't One Call Fetch All Pages?
1. **HTTP is Stateless:** The HTTP request-response cycle completes and closes immediately after sending Page 0. The server does not keep a channel or state open.
2. **Database Connections Close Immediately:** Spring Data JPA opens a database connection from the connection pool (HikariCP), executes the `LIMIT/OFFSET` SQL query, maps results to entities, closes the transaction, and returns the DB connection back to the pool. **No database cursor is held open across requests.**
3. **Fetching everything in one call defeats pagination:** If the server fetched all pages in advance, you would still suffer the exact same memory and performance problems as `findAll()`.

---

#### 1. In Client-Server / REST APIs (Frontend & Backend)

The frontend and backend interact through separate HTTP calls:

```
[ Frontend / React / Mobile ]                       [ Spring Boot REST API ]                     [ Database ]
              │                                                │                                       │
              │──── 1. GET /products?page=0&size=10 ──────────>│                                       │
              │                                                │─── SELECT ... LIMIT 10 OFFSET 0 ─────>│
              │                                                │<── Returns rows 1 to 10 ──────────────│
              │<─── 2. JSON: items 1-10 + hasNext: true ───────│                                       │
              │    (User browses or scrolls to bottom)         │                                       │
              │                                                │                                       │
              │──── 3. GET /products?page=1&size=10 ──────────>│ (NEW SEPARATE HTTP CALL)              │
              │                                                │─── SELECT ... LIMIT 10 OFFSET 10 ────>│
              │                                                │<── Returns rows 11 to 20 ─────────────│
              │<─── 4. JSON: items 11-20 + hasNext: true ──────│                                       │
```

---

#### 2. In Backend Batch Processing / Java Code

If your Spring Boot service needs to process millions of records in chunks (e.g., sending email notifications, generating reports, or syncing data), you must also make **separate repository calls in a loop**.

Spring Data JPA provides **`page.hasNext()`** and **`page.nextPageable()`** to make this clean:

```java
@Service
public class OrderMigrationService {

    @Autowired
    private OrderRepository orderRepository;

    public void processAllOrdersInBatches() {
        int pageSize = 100;
        Pageable pageable = PageRequest.of(0, pageSize);
        Page<Order> currentPage;

        do {
            // SEPARATE DATABASE CALL for each page
            currentPage = orderRepository.findAll(pageable);

            // Process current batch of 100 orders
            for (Order order : currentPage.getContent()) {
                processOrder(order);
            }

            // page.nextPageable() creates a new Pageable with pageNumber incremented by 1
            pageable = currentPage.nextPageable();

        } while (currentPage.hasNext()); // Stops when last page is reached
    }

    private void processOrder(Order order) {
        // Business logic here
    }
}
```

> **Key takeaway on `page.nextPageable()`:**
> Calling `currentPage.nextPageable()` does **NOT** fetch the data from the database. It only creates a new `Pageable` value object pointing to `pageNumber + 1`. You must still pass that `Pageable` into a **new `repository.findAll(pageable)` call** to execute the next SQL query!

---

### 8.4 How Pagination Works Under the Hood (LIMIT & OFFSET Math)

When you request **Page `p`** (0-indexed) with **Page Size `s`**:
```
OFFSET = p * s
LIMIT  = s
```

| Requested Page (`pageNo`) | Page Size (`pageSize`) | SQL `LIMIT` | SQL `OFFSET` | Records Returned |
| :---: | :---: | :---: | :---: | :--- |
| **`0`** (1st Page) | `10` | `10` | `0 * 10 = 0` | Rows 1 to 10 |
| **`1`** (2nd Page) | `10` | `10` | `1 * 10 = 10` | Rows 11 to 20 |
| **`2`** (3rd Page) | `10` | `10` | `2 * 10 = 20` | Rows 21 to 30 |

> **CRITICAL RULE:** Spring Data JPA page numbers are **0-based**!
> `PageRequest.of(0, 10)` fetches the **first page** (first 10 records).

---

### 8.5 Core Classes & Interfaces for Pagination

1. **`Pageable` (Interface):** Holds pagination information (page number, page size, and optional `Sort`).
2. **`PageRequest` (Class):** Concrete implementation of `Pageable`, created via static factory methods:
   - `PageRequest.of(int pageNumber, int pageSize)`
   - `PageRequest.of(int pageNumber, int pageSize, Sort sort)`
   - `PageRequest.of(int pageNumber, int pageSize, Sort.Direction direction, String... properties)`
3. **`Page<T>` (Interface):** Represents a single page of results **plus total metadata** (total elements, total pages, current page number, etc.).

---

### Example 1: Basic Pagination (Without Sorting)

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    public void fetchFirstPage() {
        // Page 0 (first page), 5 items per page
        Pageable pageable = PageRequest.of(0, 5);

        Page<Product> page = productRepository.findAll(pageable);

        // Extract actual list of 5 products on this page
        List<Product> productsOnPage = page.getContent();

        // Useful metadata provided by Page<T>
        long totalElements = page.getTotalElements();      // e.g., 47 total rows in DB
        int totalPages     = page.getTotalPages();         // e.g., ceil(47 / 5.0) = 10 pages
        int currentPageNo  = page.getNumber();             // 0
        int pageSize       = page.getSize();               // 5
        int itemsInThisPage= page.getNumberOfElements();   // 5 (or 2 on the last page)
        boolean isFirst    = page.isFirst();               // true
        boolean isLast     = page.isLast();                // false
        boolean hasNext    = page.hasNext();               // true
        boolean hasPrev    = page.hasPrevious();           // false
    }
}
```

**What SQL queries are executed by `productRepository.findAll(pageable)`?**
1. **Data Query:**
   ```sql
   SELECT id, name, category, price, rating, created_at
   FROM products
   LIMIT 5 OFFSET 0;
   ```
2. **Count Query** (executed automatically when returning `Page<T>` to calculate `totalElements` and `totalPages`):
   ```sql
   SELECT COUNT(id) FROM products;
   ```

---

### Example 2: Pagination + Sorting Combined

In real applications, you should **always combine Pagination with Sorting** — without an `ORDER BY` clause, databases do not guarantee a deterministic row order across pages!

```java
// Fetch Page 1 (2nd page), 10 items per page, sorted by price DESC, then name ASC
Sort sort = Sort.by(Sort.Order.desc("price"), Sort.Order.asc("name"));
Pageable pageable = PageRequest.of(1, 10, sort);

Page<Product> productPage = productRepository.findAll(pageable);
```
**Generated SQL:**
```sql
SELECT id, name, category, price, rating, created_at
FROM products
ORDER BY price DESC, name ASC
LIMIT 10 OFFSET 10;

SELECT COUNT(id) FROM products;
```

---

### Example 3: Pagination with Custom Derived Queries & `@Query`

You can pass `Pageable` as the last parameter to **any** derived query method, JPQL `@Query`, or Native SQL `@Query`:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // 1. Derived Query with Pageable
    Page<Product> findByCategoryAndPriceLessThan(String category, Double maxPrice, Pageable pageable);

    // 2. Custom JPQL @Query with Pageable (Spring automatically generates the count query!)
    @Query("SELECT p FROM Product p WHERE LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Page<Product> searchByKeyword(@Param("keyword") String keyword, Pageable pageable);

    // 3. Native SQL @Query with Pageable (Specify countQuery explicitly for complex native SQL)
    @Query(
        value = "SELECT * FROM products WHERE category = :category",
        countQuery = "SELECT COUNT(*) FROM products WHERE category = :category",
        nativeQuery = true
    )
    Page<Product> findByCategoryNative(@Param("category") String category, Pageable pageable);
}
```

**Calling the custom paginated method:**
```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("rating").descending());
Page<Product> electronicsPage = productRepository.findByCategoryAndPriceLessThan(
    "Electronics", 50000.0, pageable
);
```

---

### Example 4: Looping Through All Pages (Batch Processing Pattern)

When processing a huge table in background jobs, never call `findAll()`. Instead, iterate page-by-page using `page.hasNext()` and `page.nextPageable()`:

```java
public void processAllProductsInBatches() {
    Pageable pageable = PageRequest.of(0, 100, Sort.by("id").ascending());
    Page<Product> page;

    do {
        page = productRepository.findAll(pageable);
        List<Product> batch = page.getContent();

        // Process current batch of 100 products
        batch.forEach(this::sendPriceAlertIfApplicable);

        // Move pointer to the next page (pageNumber + 1)
        pageable = page.nextPageable();

    } while (page.hasNext());
}
```

---

### Example 5: Transforming Entities to DTOs with `Page.map()`

`Page<T>` has a built-in `.map(Function)` method that converts `Page<Product>` to `Page<ProductDto>` while **preserving all pagination metadata** (`totalElements`, `totalPages`, `number`, etc.):

```java
public Page<ProductDto> getProductsAsDto(int pageNo, int pageSize) {
    Pageable pageable = PageRequest.of(pageNo, pageSize, Sort.by("createdAt").descending());
    Page<Product> entityPage = productRepository.findAll(pageable);

    // Clean 1-liner conversion from Page<Product> -> Page<ProductDto>
    return entityPage.map(product -> new ProductDto(
            product.getId(),
            product.getName(),
            product.getPrice()
    ));
}
```

---

## 9. `Page<T>` vs `Slice<T>` vs `List<T>` with `Pageable`

When you pass a `Pageable` parameter to a repository method, you can choose between **three return types** depending on your performance and UI needs:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Option 1: Returns Page<Product> (Runs SELECT + SELECT COUNT(*))
    Page<Product> findByCategory(String category, Pageable pageable);

    // Option 2: Returns Slice<Product> (Runs ONLY SELECT with LIMIT pageSize + 1, NO COUNT query!)
    Slice<Product> findByBrand(String brand, Pageable pageable);

    // Option 3: Returns List<Product> (Runs ONLY SELECT with LIMIT & OFFSET, no metadata)
    List<Product> findByRatingGreaterThan(Double rating, Pageable pageable);
}
```

### How `Slice<T>` Works Without a `COUNT(*)` Query

If you ask for `PageRequest.of(0, 10)` on a method returning `Slice<T>`:
- Hibernate secretly executes `SELECT ... LIMIT 11 OFFSET 0` (`pageSize + 1`).
- If the database returns **11 rows**, Spring returns the first **10 rows** to you and sets `hasNext() = true`!
- If the database returns **10 or fewer rows**, Spring sets `hasNext() = false`!
- **Result:** Zero `SELECT COUNT(*)` overhead! On a 100-million-row table, skipping `COUNT(*)` can make queries **10x faster**.

### Comparison: `Page<T>` vs `Slice<T>` vs `List<T>`

| Feature | `Page<T>` (extends `Slice<T>`) | `Slice<T>` | `List<T>` |
| :--- | :--- | :--- | :--- |
| **SQL Queries Fired** | **2 Queries** (`SELECT` + `SELECT COUNT(*)`) | **1 Query** (`SELECT` with `LIMIT size + 1`) | **1 Query** (`SELECT` with `LIMIT size`) |
| **Knows `hasNext()` / `hasPrevious()`?** | Yes | **Yes** | No |
| **Knows `totalElements` & `totalPages`?** | **Yes** | No | No |
| **Performance on Huge Tables** | Slower (`COUNT(*)` scans matching rows) | **Fast** (no count query) | **Fastest** |
| **Best UI Use Case** | Numbered pagination (`Page 1 of 42`) | Infinite Scroll / "Load More" button (Instagram, X) | Simple top-N / widget lists |

---

## 10. Complete End-to-End Spring Boot REST API Example

### 1. Repository (`ProductRepository.java`)

```java
package com.example.shop.repository;

import com.example.shop.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByCategoryIgnoreCase(String category, Pageable pageable);
}
```

### 2. Service (`ProductService.java`)

```java
package com.example.shop.service;

import com.example.shop.dto.ProductResponseDto;
import com.example.shop.entity.Product;
import com.example.shop.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    public Page<ProductResponseDto> getProducts(
            String category,
            int pageNo,
            int pageSize,
            String sortBy,
            String sortDir) {

        Sort sort = sortDir.equalsIgnoreCase(Sort.Direction.ASC.name())
                ? Sort.by(sortBy).ascending()
                : Sort.by(sortBy).descending();

        Pageable pageable = PageRequest.of(pageNo, pageSize, sort);

        Page<Product> productPage = (category != null && !category.isBlank())
                ? productRepository.findByCategoryIgnoreCase(category, pageable)
                : productRepository.findAll(pageable);

        return productPage.map(p -> new ProductResponseDto(
                p.getId(), p.getName(), p.getCategory(), p.getPrice(), p.getRating()
        ));
    }
}
```

### 3. Controller (`ProductController.java`) — Two Approaches

```java
package com.example.shop.controller;

import com.example.shop.dto.ProductResponseDto;
import com.example.shop.repository.ProductRepository;
import com.example.shop.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;
    private final ProductRepository productRepository;

    // Approach 1: Explicit @RequestParam (Full control over parameter names)
    // Example URL: GET /api/products?category=Electronics&pageNo=0&pageSize=5&sortBy=price&sortDir=desc
    @GetMapping
    public ResponseEntity<Page<ProductResponseDto>> getProducts(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") int pageNo,
            @RequestParam(defaultValue = "10") int pageSize,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDir) {

        return ResponseEntity.ok(
                productService.getProducts(category, pageNo, pageSize, sortBy, sortDir)
        );
    }

    // Approach 2: Automatic Spring Web Pageable Binding with @PageableDefault
    // Spring automatically binds ?page=0&size=5&sort=price,desc&sort=name,asc into a Pageable object!
    @GetMapping("/auto-page")
    public ResponseEntity<Page<ProductResponseDto>> getProductsAuto(
            @PageableDefault(page = 0, size = 10, sort = "createdAt", direction = Sort.Direction.DESC)
            Pageable pageable) {

        Page<ProductResponseDto> response = productRepository.findAll(pageable)
                .map(p -> new ProductResponseDto(
                        p.getId(), p.getName(), p.getCategory(), p.getPrice(), p.getRating()
                ));
        return ResponseEntity.ok(response);
    }
}
```

### 4. Sample JSON Output Returned by `Page<ProductResponseDto>`

When calling `GET /api/products?category=Electronics&pageNo=0&pageSize=2&sortBy=price&sortDir=desc`:

```json
{
  "content": [
    {
      "id": 104,
      "name": "MacBook Pro 16",
      "category": "Electronics",
      "price": 2499.0,
      "rating": 4.9
    },
    {
      "id": 101,
      "name": "Sony WH-1000XM5",
      "category": "Electronics",
      "price": 399.0,
      "rating": 4.8
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 2,
    "sort": {
      "empty": false,
      "sorted": true,
      "unsorted": false
    },
    "offset": 0,
    "paged": true,
    "unpaged": false
  },
  "totalElements": 15,
  "totalPages": 8,
  "last": false,
  "size": 2,
  "number": 0,
  "sort": {
    "empty": false,
    "sorted": true,
    "unsorted": false
  },
  "numberOfElements": 2,
  "first": true,
  "empty": false
}
```

---

## 11. Top Interview Questions & Rapid Answers

### Q1: If `JpaRepository` has everything, why do `CrudRepository` and `PagingAndSortingRepository` exist?
> **Answer:** Because of the **Interface Segregation Principle (ISP)** and **multi-store abstraction**.
> 1. `CrudRepository` and `PagingAndSortingRepository` live in `spring-data-commons` and work across non-relational databases (Redis, MongoDB, Cassandra) where JPA concepts like `flush()` or `getReferenceById()` do not exist.
> 2. If you want to restrict a repository so developers **cannot** delete or mutate data, or only expose basic operations, you can extend a narrower interface or `Repository<T, ID>`.

### Q2: Did `PagingAndSortingRepository` change in Spring Boot 3 (Spring Data 3.0)?
> **Answer:** **Yes.** In Spring Data 2.x, `PagingAndSortingRepository` extended `CrudRepository`. In Spring Data 3.0+, `PagingAndSortingRepository` extends `Repository<T, ID>` directly (no longer extends `CrudRepository`). Additionally, `ListCrudRepository` and `ListPagingAndSortingRepository` were introduced so `findAll()` returns `List<T>` instead of `Iterable<T>`. `JpaRepository` extends both `ListCrudRepository` and `ListPagingAndSortingRepository`.

### Q3: What happens if you paginate a `@OneToMany` query using `JOIN FETCH` in Spring Data JPA?
> **Answer:** Hibernate logs a dangerous warning:
> `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!`
> Because a SQL `JOIN` with a child collection multiplies rows, Hibernate cannot safely apply SQL `LIMIT/OFFSET` at the database level. Instead, it **fetches the entire table into memory** and paginates in Java Heap, which can cause an `OutOfMemoryError`!
> **Fix:** Either use `@BatchSize` on the collection, or use a two-step query (first paginate IDs with `Pageable`, then fetch entities with `JOIN FETCH WHERE id IN (:ids)`).

### Q4: When should you use `Slice<T>` instead of `Page<T>`?
> **Answer:** Use `Slice<T>` when building **Infinite Scroll** or **"Load More"** UIs on large datasets where you do not need to display total page numbers (`Page 1 of 500`). `Slice<T>` avoids the expensive `SELECT COUNT(*)` query by fetching `pageSize + 1` rows to check if a next slice exists.

### Q5: Can I pass `Pageable.unpaged()` if a caller occasionally wants all records without writing a second repository method?
> **Answer:** **Yes!** Passing `Pageable.unpaged()` (or `Pageable.unpaged(Sort.by("name"))` in Spring Data 3.2+) to any `findAll(Pageable)` method disables `LIMIT/OFFSET` and returns all matching rows in a single `Page<T>`.

---

## 12. `flush()` vs `save()` in Spring Data JPA

### 12.1 What is `save()`?

`save()` is provided by `CrudRepository` (and inherited by `JpaRepository`). When you call:

```java
employeeRepository.save(employee);
```

Internally it does:
- If the entity is **new** (no ID or ID not found in persistence context) → calls `entityManager.persist(entity)`
- If the entity **already exists** → calls `entityManager.merge(entity)`

> **Key Point:** `save()` does **NOT** immediately execute an `INSERT` or `UPDATE` SQL statement against the database. It only transitions the entity to the **managed** state inside Hibernate's **Persistence Context** (first-level cache). The actual SQL is deferred.

### 12.2 What is `flush()`?

`flush()` forces Hibernate to **synchronize** the persistence context with the database **immediately**. It pushes all pending `INSERT`, `UPDATE`, and `DELETE` SQL statements to the database **right now**, without waiting for the transaction to commit.

```java
entityManager.flush();
```

> **Important:** `flush()` does **NOT** commit the transaction. The SQL is sent to the database, but can still be **rolled back** if the transaction fails later.

### 12.3 `save()` vs `saveAndFlush()` vs `flush()`

| Aspect | `save(entity)` | `saveAndFlush(entity)` | `flush()` |
|---|---|---|---|
| **Defined in** | `CrudRepository` | `JpaRepository` | `JpaRepository` |
| **What it does** | Persists/merges entity into Persistence Context | `save()` + immediate `flush()` | Flushes **all** pending changes to DB |
| **SQL execution** | **Deferred** (until flush or commit) | **Immediate** | **Immediate** |
| **Returns** | Managed entity | Managed entity | `void` |
| **Transaction committed?** | No | No | No |

### 12.4 When does SQL actually fire if you only call `save()`?

Hibernate flushes automatically in these scenarios:

1. **Transaction commit** — At the end of a `@Transactional` method, Spring commits the transaction, which triggers a flush.
2. **Before a JPQL/HQL query** — Hibernate auto-flushes so the query sees consistent data.
3. **Explicit `flush()` call** — You manually trigger it.

```java
@Transactional
public void example() {
    Employee emp = new Employee("Mohit", "IT");
    employeeRepository.save(emp);        // ← No SQL yet, entity is in Persistence Context

    // SQL INSERT fires here automatically when transaction commits at method end
}
```

### 12.5 When should you use `saveAndFlush()` or `flush()`?

#### ✅ Use `saveAndFlush()` when:

1. **You need the database-generated ID immediately**
```java
Employee saved = employeeRepository.saveAndFlush(emp);
System.out.println(saved.getId());  // Guaranteed to have DB-generated ID
```

2. **You want to catch database constraint violations early** (e.g., unique constraint) instead of getting the exception at commit time:
```java
try {
    employeeRepository.saveAndFlush(emp);
} catch (DataIntegrityViolationException e) {
    // Handle duplicate immediately
}
```

3. **Subsequent native SQL queries** need to see the saved data (native queries bypass the persistence context).

#### ✅ Use `flush()` when:

You have **multiple entities** saved via `save()` and want to push them all to the database at once before the transaction commits:

```java
@Transactional
public void bulkInsert(List<Employee> employees) {
    for (int i = 0; i < employees.size(); i++) {
        employeeRepository.save(employees.get(i));
        if (i % 50 == 0) {
            employeeRepository.flush();         // Push batch of 50 to DB
            entityManager.clear();              // Free memory in Persistence Context
        }
    }
}
```

#### ❌ Don't use `flush()` / `saveAndFlush()` when:

- You have no specific reason — unnecessary flushing **hurts performance** because Hibernate loses the ability to batch and reorder SQL statements.
- You are inside a `@Transactional` method and the default auto-flush at commit is sufficient.

### 12.6 Visual Flow

```
save(entity)
   │
   ▼
┌──────────────────────────┐
│   Persistence Context    │   (In-memory, 1st level cache)
│   Entity is MANAGED      │
│   No SQL executed yet    │
└──────────┬───────────────┘
           │
           │  flush() / saveAndFlush() / auto-flush at commit
           ▼
┌──────────────────────────┐
│      DATABASE            │   SQL INSERT/UPDATE executed
│   (but NOT committed)    │
└──────────┬───────────────┘
           │
           │  Transaction.commit()
           ▼
┌──────────────────────────┐
│   COMMITTED TO DISK      │   Data is permanently saved
└──────────────────────────┘
```

### 12.7 Quick Interview Answer

> **`save()`** stages the entity in Hibernate's persistence context (1st-level cache) — SQL is deferred until flush or commit.  
> **`flush()`** forces all pending SQL to the database immediately but does **not** commit the transaction.  
> **`saveAndFlush()`** = `save()` + `flush()` in one call.  
> Use `saveAndFlush()` when you need the generated ID immediately or want early constraint validation. In most cases, plain `save()` inside a `@Transactional` method is sufficient because Hibernate auto-flushes at commit.
