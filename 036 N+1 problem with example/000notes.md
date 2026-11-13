# N+1 Problem in JPA / Hibernate (with Examples)

## Table of Contents

1. [What is the N+1 problem?](#1-what-is-the-n1-problem)
2. [The setup: two related entities](#2-the-setup-two-related-entities)
3. [The code that triggers N+1](#3-the-code-that-triggers-n1)
4. [Why it happens: lazy collections are proxies](#4-why-it-happens-lazy-collections-are-proxies)
5. [How to see it yourself](#5-how-to-see-it-yourself)
6. [Why EAGER is NOT the fix](#6-why-eager-is-not-the-fix)
7. [The fixes](#7-the-fixes)
8. [JOIN vs JOIN FETCH](#8-join-vs-join-fetch)
9. [Still N+1 after adding a join? Common issues](#9-still-n1-after-adding-a-join-common-issues)
10. [Interview-ready summaries](#10-interview-ready-summaries)
11. [Interview Questions](#11-interview-questions)

---

## 1. What is the N+1 problem?

You run **1 query** to load a list of parents (e.g. authors). Then, while looping over them, you touch a lazy association (e.g. `author.getBooks()`). Each touch fires **1 more query**. With **N** parents you end up with **1 + N** queries instead of 1.

- 1 initial query, plus N additional queries (one per row from the first result).
- **100 authors → 1 + 100 = 101 queries.**

![svg](<svgs/01-n-plus-1-vs-join-fetch.svg>)

Each query is a network round-trip to the database. Individually they are fast, but 101 of them add up, put load on the DB and the connection pool, and get worse as data grows. That is why N+1 is one of the most common causes of slow Spring Boot APIs.

---

## 2. The setup: two related entities

```java
@Entity
public class Author {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author")      // @OneToMany is LAZY by default
    private List<Book> books;
}

@Entity
public class Book {
    @Id
    @GeneratedValue
    private Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)   // @ManyToOne is EAGER by default, so we set LAZY explicitly
    @JoinColumn(name = "author_id")
    private Author author;
}
```

**Sample data used in the examples:**

| author.id | author.name |
|---|---|
| 1 | Rowling |
| 2 | Tolkien |
| 3 | Orwell |

| book.id | book.title | book.author_id |
|---|---|---|
| 1 | Harry Potter 1 | 1 |
| 2 | Harry Potter 2 | 1 |
| 3 | The Hobbit | 2 |
| 4 | The Lord of the Rings | 2 |
| 5 | 1984 | 3 |

**The repository looks completely innocent:**

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {
}
```

---

## 3. The code that triggers N+1

This is the trap:

```java
@Service
public class LibraryService {

    @Autowired
    private AuthorRepository authorRepository;

    @Transactional(readOnly = true)   // keeps the Hibernate session open so lazy loading works
    public void printAllBooksByAuthor() {
        List<Author> authors = authorRepository.findAll(); // Query #1

        for (Author author : authors) {
            System.out.println(author.getName() + " wrote:");
            // 'books' is lazy, so touching it here fires a SEPARATE query
            for (Book book : author.getBooks()) {   // Query #2, #3, #4... one PER author
                System.out.println("  - " + book.getTitle());
            }
        }
    }
}
```

> Without `@Transactional` (and outside a web request with open-in-view), the session is already closed when `getBooks()` runs, and you get a `LazyInitializationException` instead of the extra queries. See [section 4](#4-why-it-happens-lazy-collections-are-proxies).

**What actually happens on the DB, with Hibernate SQL logging turned on:**

```sql
-- Query #1: fetch all authors (the "1")
SELECT * FROM author;

-- Now for EACH author returned above, a separate query fires (the "N")
SELECT * FROM book WHERE author_id = 1;   -- for Author #1
SELECT * FROM book WHERE author_id = 2;   -- for Author #2
SELECT * FROM book WHERE author_id = 3;   -- for Author #3
-- ... and so on, for every single author
```

**Console output for the sample data.** The SQL lines appear *between* the printed lines, which is the tell-tale sign of lazy loading:

```text
Hibernate: select a1_0.id, a1_0.name from author a1_0
Rowling wrote:
Hibernate: select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id=?
  - Harry Potter 1
  - Harry Potter 2
Tolkien wrote:
Hibernate: select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id=?
  - The Hobbit
  - The Lord of the Rings
Orwell wrote:
Hibernate: select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id=?
  - 1984
```

3 authors → **4 queries**. 100 authors → **101 queries**. That's where the name "N+1" comes from.

---

## 4. Why it happens: lazy collections are proxies

When Hibernate loads an `Author`, it does **not** load the books. It puts a placeholder collection (a `PersistentBag` proxy) in the `books` field. The first time you read from it, the proxy goes back to the database.

![svg](<svgs/02-lazy-proxy-trigger.svg>)

- Inside a loop, "first time you read from it" happens **once per author**, so you get N extra queries.
- If the Hibernate session is already closed, the proxy can't query, and you get `LazyInitializationException: could not initialize proxy - no Session`.

---

## 5. How to see it yourself

### 5.1 Enable SQL logging and statistics

```yaml
# application.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true  # logs a summary at the end, e.g. "101 JDBC statements executed"
```

Better for real projects: log through the logger instead of `System.out`:

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG                 # the SQL statements
    org.hibernate.orm.jdbc.bind: TRACE       # the bound parameter values (Hibernate 6)
    org.hibernate.stat: DEBUG                # statistics summary
```

Run `printAllBooksByAuthor()` with this enabled. Your console prints all 101 separate `SELECT` statements. That's the smoking gun in a real codebase.

### 5.2 Catch it in a test (so it never comes back)

```java
@SpringBootTest
class LibraryServiceTest {

    @Autowired LibraryService libraryService;
    @Autowired EntityManagerFactory emf;

    @Test
    void printsAllBooksWithOneQuery() {
        Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
        stats.clear();

        libraryService.printAllBooksByAuthor();

        // needs hibernate.generate_statistics=true
        assertThat(stats.getPrepareStatementCount()).isEqualTo(1);
    }
}
```

Before the fix this test fails with `expected 1 but was 4`. After the fix it passes.

---

## 6. Why EAGER is NOT the fix

A common first reaction is "just make it `FetchType.EAGER`". It doesn't fix N+1, and it often makes things worse.

![svg](<svgs/03-fetch-type-defaults.svg>)

| Association | Default fetch type |
|---|---|
| `@ManyToOne` | EAGER |
| `@OneToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

**Example: EAGER causing N+1 by itself:**

```java
@Entity
public class Book {
    @ManyToOne                    // default = EAGER
    @JoinColumn(name = "author_id")
    private Author author;
}

bookRepository.findAll();         // you never even call getAuthor()!
```

```sql
SELECT * FROM book;                       -- 1
SELECT * FROM author WHERE id = 1;        -- + 1 per distinct author
SELECT * FROM author WHERE id = 2;
SELECT * FROM author WHERE id = 3;
```

JPQL / `findAll()` runs the query as written (no join). Hibernate then sees the EAGER association isn't loaded yet and loads it immediately, one query per distinct author.

**Rule:** make every association `LAZY`, then decide **per query** what to fetch (next section).

---

## 7. The fixes

![svg](<svgs/05-fix-comparison-query-count.svg>)

### Fix 1: `JOIN FETCH` (eager fetch in one query)

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a LEFT JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}
```

**Resulting SQL is now just ONE query:**

```sql
SELECT a.*, b.*
FROM author a
LEFT JOIN book b ON b.author_id = a.id;
```

Hibernate takes this single flat result set and **rebuilds** it into `Author` objects with their `books` list already filled. Calling `author.getBooks()` later fires no queries, because the collection is already loaded.

> - `JOIN FETCH` (inner) drops authors that have no books. `LEFT JOIN FETCH` keeps them.
> - Hibernate 6 removes the duplicate `Author` objects (one per book row) automatically. In Hibernate 5 you had to write `SELECT DISTINCT a ...`.

### Fix 2: `@EntityGraph` (same result, no JPQL)

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = "books")   // generates a LEFT OUTER JOIN on books
    List<Author> findAll();                  // overrides the default findAll()

    @EntityGraph(attributePaths = {"books", "publisher"})
    List<Author> findByNameStartingWith(String prefix); // works with derived queries too
}
```

Use it when you want to keep derived query methods and just say "also load these associations".

### Fix 3: Batch fetching (`@BatchSize` / `default_batch_fetch_size`)

Lazy loading stays, but Hibernate loads children for **many parents at once** using `IN (...)`:

```java
@OneToMany(mappedBy = "author")
@BatchSize(size = 25)
private List<Book> books;
```

or globally:

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

```sql
SELECT * FROM author;                                         -- 1
SELECT * FROM book WHERE author_id IN (1, 2, 3, ..., 25);     -- +1 per 25 authors
SELECT * FROM book WHERE author_id IN (26, 27, ..., 50);
...
```

100 authors → 1 + ⌈100 / 25⌉ = **5 queries**. A good safety net, and it works with pagination.

### Fix 4: `@Fetch(FetchMode.SUBSELECT)` (Hibernate-specific)

```java
@OneToMany(mappedBy = "author")
@Fetch(FetchMode.SUBSELECT)
private List<Book> books;
```

```sql
SELECT * FROM author;
SELECT * FROM book WHERE author_id IN (SELECT id FROM author);   -- children of ALL loaded authors
```

Always **2 queries**, no matter how many authors.

### Fix 5: DTO projection (when you only need a few columns)

```java
public record AuthorBookDto(String authorName, String bookTitle) {}

@Query("SELECT new com.example.dto.AuthorBookDto(a.name, b.title) " +
       "FROM Author a JOIN a.books b")
List<AuthorBookDto> findAuthorBookTitles();
```

One query that returns only the columns you need. No entities, no proxies, nothing to lazy-load. Here a plain `JOIN` is correct, because you select `b.title` explicitly.

### Which fix when?

| Fix | Queries (100 authors) | Use when |
|---|---|---|
| `JOIN FETCH` / `@EntityGraph` | 1 | You need the children for this use-case |
| `@BatchSize` / `default_batch_fetch_size` | 1 + ⌈N/size⌉ | Paging parents, multiple collections, or as a global safety net |
| `FetchMode.SUBSELECT` | 2 | Always need children for every loaded parent |
| DTO projection | 1 | Read-only views / APIs that need only some columns |

---

## 8. JOIN vs JOIN FETCH

![svg](<svgs/04-join-vs-join-fetch.svg>)

| | `JOIN a.books` | `JOIN FETCH a.books` |
|---|---|---|
| Purpose | Filtering / conditions (`WHERE b.title = ...`) | Loading the association into the objects |
| Columns selected | Only `author` columns | `author` + `book` columns |
| `author.getBooks()` afterwards | Still lazy, extra query per author | Already loaded, no query |
| Fixes N+1? | ❌ No | ✅ Yes |

---

## 9. Still N+1 after adding a join? Common issues

The short answer: **if you write the JOIN correctly, it DOES solve it.** If N+1 is still happening despite using a join, something specific is going wrong. A correctly written `JOIN FETCH` produces **exactly one SQL query** with a real SQL `JOIN`:

```java
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();
```

```sql
SELECT a.*, b.* FROM author a JOIN book b ON b.author_id = a.id;
```

So if N+1 is STILL happening even with a join, here are the actual culprits:

### Issue 1: You wrote a plain JOIN, not JOIN FETCH

```java
@Query("SELECT a FROM Author a JOIN a.books")  // ← missing FETCH!
List<Author> findAllWithBooks();
```

This is the #1 mistake. A plain `JOIN` (without `FETCH`) is used for **filtering/conditions** (e.g. "only authors who have books"). It does **NOT** tell Hibernate to fill the `books` collection in the returned `Author` objects. The join happens in SQL for filtering only, and `author.getBooks()` is still lazy, so it fires a separate query when accessed later. `FETCH` is the keyword that says "actually load this association into the object graph."

**Fix:** `SELECT a FROM Author a JOIN FETCH a.books`

### Issue 2: Multiple `JOIN FETCH` on multiple collections in one query

```java
@Query("SELECT a FROM Author a JOIN FETCH a.books JOIN FETCH a.awards")
```

![svg](<svgs/06-multiple-bag-cartesian-product.svg>)

Fetching **two `List` collections** in one query with JOIN FETCH creates a **Cartesian product**: rows multiply (3 books × 2 awards = 6 rows for one author). Hibernate throws `MultipleBagFetchException: cannot simultaneously fetch multiple bags`. This isn't N+1 exactly, but it's a related trap people hit while trying to "fix" N+1 by joining everything.

**Fix:** fetch one collection per query (Hibernate keeps both in the same session, so the second query fills the same objects), or let `@BatchSize` handle the second collection:

```java
@Query("SELECT a FROM Author a LEFT JOIN FETCH a.books")
List<Author> findAllWithBooks();

@Query("SELECT a FROM Author a LEFT JOIN FETCH a.awards WHERE a IN :authors")
List<Author> fetchAwards(@Param("authors") List<Author> authors);

// in a @Transactional service method:
List<Author> authors = repo.findAllWithBooks();
repo.fetchAwards(authors);   // same entities, now with awards loaded too → 2 queries total
```

> Changing `List` to `Set` makes the exception go away, but the SQL still returns the multiplied rows. It hides the problem rather than fixing it.

### Issue 3: You joined the WRONG association

If your loop accesses a *different* lazy field than the one you fetched:

```java
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// later...
for (Author a : authors) {
    System.out.println(a.getPublisher().getName()); // 'publisher' wasn't fetched! N+1 still happens here
}
```

You fixed `books`, but `publisher` (a separate `@ManyToOne`) is still lazy and untouched. The N+1 just moved to a different field.

**Fix:** fetch everything the loop reads:

```java
@Query("SELECT a FROM Author a LEFT JOIN FETCH a.books JOIN FETCH a.publisher")
List<Author> findAllWithBooksAndPublisher();
```

(Fetching **one** collection plus any number of `...ToOne` associations in a single query is fine.)

### Issue 4: Pagination combined with JOIN FETCH (a classic hidden trap)

```java
@Query("SELECT a FROM Author a JOIN FETCH a.books")
Page<Author> findAllWithBooks(Pageable pageable);  // combining JOIN FETCH + pagination
```

![svg](<svgs/07-join-fetch-pagination-trap.svg>)

Hibernate **cannot** apply SQL `LIMIT`/`OFFSET` correctly when a collection join multiplies rows (one author row becomes several rows, one per book). So it loads **everything into memory** and cuts the page in Java, logging:

```text
WARN HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

(Hibernate 5: `HHH000104`.) This is a well-known gotcha: JOIN FETCH + `Pageable` silently brings back performance problems, sometimes worse than N+1, because the whole table is loaded into memory. (Spring Data also needs a separate `countQuery` here, because it can't derive a count query from a fetch join.)

**Fix: two queries (page the IDs, then fetch):**

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query(value = "SELECT a.id FROM Author a",
           countQuery = "SELECT COUNT(a) FROM Author a")
    Page<Long> findAuthorIds(Pageable pageable);                 // real LIMIT/OFFSET in SQL

    @Query("SELECT a FROM Author a LEFT JOIN FETCH a.books WHERE a.id IN :ids ORDER BY a.id")
    List<Author> findWithBooksByIds(@Param("ids") List<Long> ids);
}

@Transactional(readOnly = true)
public Page<Author> getAuthorsPage(Pageable pageable) {
    Page<Long> idPage = authorRepository.findAuthorIds(pageable);              // query 1
    List<Author> authors = authorRepository.findWithBooksByIds(idPage.getContent()); // query 2
    return new PageImpl<>(authors, pageable, idPage.getTotalElements());
}
```

Alternative: page normally (no fetch join) and add `@BatchSize` on `books`.

Make it fail loudly in dev instead of silently paging in memory:

```properties
spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch=true
```

### Issue 5: Native SQL query instead of JPQL, without proper object mapping

If you're not using JPA/JPQL at all (raw JDBC or native `@Query(nativeQuery = true)`), "N+1" as a *JPA lazy-loading* concept doesn't apply the same way. But you can still create the same anti-pattern by hand, by looping and firing a new query per row:

```java
List<Author> authors = jdbcTemplate.query("SELECT * FROM author", authorMapper);
for (Author a : authors) {
    a.setBooks(jdbcTemplate.query("SELECT * FROM book WHERE author_id = ?", bookMapper, a.getId())); // N+1 by hand
}
```

The cause (one query per loop iteration) is identical. It just comes from your own code structure, not from lazy loading. **Fix:** one `JOIN` query (or one `WHERE author_id IN (...)` query) and group the rows in Java.

### How to verify which of these is your actual issue

Turn on `hibernate.generate_statistics: true` and check the query count in the logs. If it's still N+1 after adding a join, check:
- (a) did you use `FETCH`, not just `JOIN`?
- (b) are you accessing a field you didn't actually fetch?
- (c) is there a `Pageable` on a fetch-join query (look for `HHH90003004` in the logs)?

---

## 10. Interview-ready summaries

**What is N+1 and how do you fix it?**
"The N+1 problem happens because lazy-loaded collections trigger a new query the moment you access them inside a loop: one query to fetch the parent list, then one more per parent to fetch its children. You can see it by enabling `show-sql: true` and watching the console print one query per row. The fix is `JOIN FETCH` in a custom JPQL query (or `@EntityGraph`), which pulls parent and children together in a single SQL join instead of N separate round-trips."

**Why might N+1 persist after adding a join?**
"A properly written `JOIN FETCH` does eliminate N+1 by loading the association in the same SQL query. If N+1 persists despite a join, the usual causes are: using plain `JOIN` instead of `JOIN FETCH` (which only filters and doesn't fill the collection), accessing a different lazy field than the one you fetched, or combining `JOIN FETCH` with pagination, which Hibernate can't safely translate to SQL-level LIMIT and may fall back to in-memory pagination instead."

---

## 11. Interview Questions

**Q1. What is the N+1 select problem?**
1 query loads N parent rows, then accessing a lazy association on each parent fires one extra query per parent: 1 + N queries in total instead of 1 (or a few).

**Q2. Is N+1 caused only by LAZY loading?**
No. `EAGER` associations also cause it with JPQL / `findAll()`: the query runs without a join, then Hibernate loads each EAGER association with a separate query, even if you never use it. Lazy loading causes it when you *access* the association in a loop. EAGER causes it *always*.

**Q3. What are the default fetch types?**
`@ManyToOne` and `@OneToOne` are EAGER. `@OneToMany` and `@ManyToMany` are LAZY. Best practice: make all of them LAZY and fetch per use-case.

**Q4. How do you detect N+1?**
`spring.jpa.show-sql=true` / `logging.level.org.hibernate.SQL=DEBUG` (look for repeated identical `SELECT ... WHERE x_id=?`), `hibernate.generate_statistics=true`, an APM/tracing tool showing many tiny DB spans per request, or a test asserting the statement count with Hibernate `Statistics`.

**Q5. JOIN vs JOIN FETCH?**
`JOIN` is for filtering: the association is not loaded into the entity. `JOIN FETCH` selects the association's columns and fills it in the returned entities. Only `JOIN FETCH` fixes N+1.

**Q6. JOIN FETCH vs @EntityGraph?**
Both produce a join and load the association in one query. `JOIN FETCH` lives in JPQL (you control inner vs left join). `@EntityGraph` is declarative, always uses a LEFT OUTER JOIN, and works with derived query methods such as `findByName(...)` without writing JPQL.

**Q7. What is `@BatchSize` and when do you prefer it over JOIN FETCH?**
It keeps lazy loading but loads children for up to `size` parents per query using `IN (...)`: 1 + ⌈N/size⌉ queries. Prefer it with pagination, when there are several collections (to avoid Cartesian products), or as a global safety net via `default_batch_fetch_size`.

**Q8. What is `MultipleBagFetchException`?**
Hibernate throws it when you `JOIN FETCH` two or more `List` (bag) collections in one query. The join creates a Cartesian product, and Hibernate can't tell real duplicates apart from join duplicates. Fix: fetch one collection per query (or use `@BatchSize` for the others). Switching to `Set` hides the exception, but the row explosion stays.

**Q9. Why is JOIN FETCH + pagination a problem?**
A collection fetch returns one row per child, so SQL `LIMIT` would cut a parent's children in half. Hibernate therefore loads all rows and paginates in memory (`HHH90003004`). Fix: page the IDs first, then fetch those IDs with `JOIN FETCH`, or use `@BatchSize`.

**Q10. What is `LazyInitializationException` and how is it related?**
It's thrown when a lazy proxy is accessed after the session is closed. It's the "other side" of lazy loading. Bad fix: switching to EAGER or enabling `hibernate.enable_lazy_load_no_trans` (opens a new session per access, which is N+1 in disguise). Good fix: fetch what you need inside the transaction (JOIN FETCH / EntityGraph / DTO).

**Q11. What does `spring.jpa.open-in-view` have to do with N+1?**
It's `true` by default in Spring Boot (with a startup warning). It keeps the session open for the whole web request, so lazy loading also works in controllers and during JSON serialization. That hides N+1: Jackson serializing `author.books` for 100 authors silently fires 100 queries. Many teams set it to `false` so lazy-loading bugs show up early.

**Q12. Can returning entities directly from a REST controller cause N+1?**
Yes. Jackson calls every getter, including lazy collections, during serialization (with open-in-view on). Return DTOs built from a fetch-join or projection query instead.

**Q13. Do you need DISTINCT with JOIN FETCH?**
In Hibernate 5, yes: `SELECT DISTINCT a` was needed to avoid duplicate parent objects (one per child row). In Hibernate 6 (Spring Boot 3), duplicates are removed automatically for fetch joins, so `DISTINCT` is no longer needed.

**Q14. Does the second-level cache solve N+1?**
Not really. The L2 cache can turn N database hits into N cache hits for entities by ID. But the first time you still run N queries, collection caches need extra setup, and you still run N lookups. Fix the query shape first. Caching is an addition, not a replacement.

**Q15. How would you fix N+1 in a read-only listing API?**
Use a DTO projection (`SELECT new ...Dto(a.name, b.title) FROM Author a JOIN a.books b`) or an interface projection. It's a single query, selects only the needed columns, and has no entities or proxies to lazy-load.
