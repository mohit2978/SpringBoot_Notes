# Hibernate Caching: Level 1 (L1) vs Level 2 (L2) Cache

## Table of Contents

1. [In simple words: L1 vs L2](#1-in-simple-words-l1-vs-l2)
2. [Why caching matters](#2-why-caching-matters)
3. [L1 cache: Session / persistence context scope](#3-l1-cache-session--persistence-context-scope)
4. [L2 cache: SessionFactory / application scope](#4-l2-cache-sessionfactory--application-scope)
5. [Lookup order: L1 → L2 → Database](#5-lookup-order-l1--l2--database)
6. [What is stored: live object vs plain values](#6-what-is-stored-live-object-vs-plain-values)
7. [L1 vs L2: full comparison](#7-l1-vs-l2-full-comparison)
8. [The query cache (L2 companion)](#8-the-query-cache-l2-companion)
9. [Hands-on: Spring Boot configuration & example](#9-hands-on-spring-boot-configuration--example)
10. [Production best practices, traps & eviction](#10-production-best-practices-traps--eviction)
11. [Interview Questions](#11-interview-questions)

---

## 1. In simple words: L1 vs L2

> **L1 cache = your own notepad.** You keep it on your desk while doing *one task*. Nobody else can see it, and you throw it away when the task is done.
>
> **L2 cache = the office whiteboard.** Everyone in the office can read it, and it stays up until someone wipes it.

- **L1** belongs to **one Session / EntityManager**, which in Spring usually means **one `@Transactional` method / one request**. It is always on.
- **L2** belongs to the **whole application** (the `SessionFactory`) and is **shared by all requests and threads**. It is off until you turn it on.

![svg](<svgs/01-l1-vs-l2-big-picture.svg>)

| Question | L1 cache | L2 cache |
|---|---|---|
| Who can see it? | Only the current transaction | Every request / thread in the app |
| How long does it live? | Until the transaction ends | As long as the app runs (until evicted / TTL) |
| Is it on by default? | Yes, always (cannot be turned off) | No, you must configure it |
| What problem does it solve? | Same entity loaded twice **in one transaction** | Same entity loaded again **in a different request** |
| What does it store? | The live Java object | Plain values (a "dehydrated" copy) |

**One-line answer:** L1 avoids repeat queries *inside one transaction*. L2 avoids repeat queries *across transactions and users*.

---

## 2. Why caching matters

In enterprise Java applications, database calls are by far the slowest operations: network round-trip, SQL parsing, disk reads, row locking, and converting rows to objects.

Hibernate gives you a **two-tier cache**:
- **First-Level Cache (L1):** mandatory, built-in, and transactional. It removes duplicate queries within the same business transaction / HTTP request.
- **Second-Level Cache (L2):** optional, configurable, and shared. It removes duplicate queries across different users, sessions, and HTTP requests for as long as the application runs.

---

## 3. L1 cache: Session / persistence context scope

The **First-Level Cache** is bound directly to the Hibernate `Session` (JPA `EntityManager`). Think of it as an in-memory map `(entity type + id) → object` that holds every entity loaded in the current persistence context.

### Key characteristics

- **Always active:** enabled by default. It **cannot** be disabled.
- **Scope:** local to the current `Session` / `EntityManager`, which is usually tied to one `@Transactional` method or one HTTP request.
- **Thread safety:** bound to a single thread; **never shared** between concurrent requests.
- **Same object every time:** within a session, repeated lookups for the same ID return the exact same Java object (`u1 == u2` is `true`).
- **Lifecycle:** starts when the `Session` opens; discarded when the `Session` closes.
- **More than a cache:** the same persistence context also does **dirty checking** (changed entities are updated automatically at commit) and **write-behind** (SQL is sent at flush/commit, not immediately).

![svg](<svgs/02-l1-cache-in-action.svg>)

### How an L1 lookup works

1. Code calls `session.get(UserDetails.class, 101L)` or `repository.findById(101L)`. Hibernate looks in the session's internal map (`StatefulPersistenceContext.entitiesByKey`).
2. **First call (cache miss):** the entity isn't there. Hibernate runs `SELECT * FROM users WHERE id = 101`, turns the row into an entity, puts it into the L1 map, and returns it.
3. **Second call in the same session (cache hit):** Hibernate finds `UserDetails#101` in the map and returns the same Java object. **Zero SQL queries.**
4. **Different session / request:** Session 2's L1 cache starts empty. Because L1 caches are isolated, **Session 2 must run the SQL query again.**

### Example

```java
@Service
public class UserService {

    @Autowired private UserRepository userRepository;

    @Transactional
    public void l1Demo() {
        UserDetails u1 = userRepository.findById(101L).orElseThrow(); // SQL runs
        UserDetails u2 = userRepository.findById(101L).orElseThrow(); // L1 hit: no SQL
        System.out.println(u1 == u2);                                  // true

        u1.setName("Bob");   // no save() call needed:
    }                        // dirty checking sends UPDATE when the transaction commits
}
```

Console:

```text
Hibernate: select u1_0.id, u1_0.email, u1_0.name from user_details u1_0 where u1_0.id=?
true
Hibernate: update user_details set email=?, name=? where id=?
```

> **Common surprise:** remove `@Transactional` and you get **two** SELECTs. Without an outer transaction, each Spring Data repository call opens its own short transaction, so each call gets its own session and its own empty L1.

### Managing L1 and preventing OutOfMemoryError

Because L1 keeps every managed entity in memory until the session ends, processing thousands of records in one session can use up the heap. Clear or detach entities explicitly:

```java
// Evict a single specific entity from L1
session.evict(user);
// or JPA: entityManager.detach(user);

// Clear the entire L1 cache for the current session
session.clear();
// or JPA: entityManager.clear();

// Check if an entity is currently managed in L1
boolean isCached = session.contains(user);
```

---

## 4. L2 cache: SessionFactory / application scope

The **Second-Level Cache** sits above individual sessions, at the `SessionFactory` (JPA `EntityManagerFactory`) level. It lives for the whole application lifetime and is shared by all threads and requests.

### Key characteristics

- **Optional & configurable:** disabled by default. Needs a provider dependency (e.g. Ehcache, Infinispan, Redisson) and configuration.
- **Scope:** application-wide (one JVM, or a whole cluster with a distributed provider).
- **Thread safety:** thread-safe and shared across concurrent requests.
- **Opt-in per entity:** entities must be marked with `@Cacheable` (and Hibernate's `@Cache` to pick a strategy).
- **Lifecycle:** survives beyond individual transactions and sessions until evicted by TTL (time-to-live), an LRU/LFU size limit, or manual invalidation.
- **Used for lookups by ID:** `findById`, lazy loading of a `@ManyToOne`, and loading associations. A JPQL query (`findAll`, `findByName`, …) still goes to the database unless the [query cache](#8-the-query-cache-l2-companion) is enabled too.

![svg](<svgs/03-l2-cache-across-requests.svg>)

### L2 concurrency strategies (`@Cache`)

When many threads read and change cached data at the same time, Hibernate uses a concurrency strategy to keep the cache consistent:

![svg](<svgs/06-l2-concurrency-strategies.svg>)

| Concurrency Strategy | Best Used For | Behavior on Update / Concurrency |
| :--- | :--- | :--- |
| **`READ_ONLY`** | Static reference data (countries, currency codes, postal codes) | Throws an exception if you try to update a cached entity. Fastest. |
| **`READ_WRITE`** | Read-heavy data with occasional updates | Uses soft locks during the transaction (read-committed behavior). The cache key is locked until commit. |
| **`NONSTRICT_READ_WRITE`** | Rarely updated data where a short window of stale reads is acceptable | No locking. On update, the entry is evicted after the transaction commits. |
| **`TRANSACTIONAL`** | Strict JTA enterprise environments | Fully transactional cache (e.g. Infinispan with JTA / XA). |

---

## 5. Lookup order: L1 → L2 → Database

When your application asks for an entity by ID, Hibernate checks in a fixed order before going to the database.

![svg](<svgs/04-lookup-order-l1-l2-db.svg>)

### Lookup sequence explained

1. **Step 1: L1 check.** Hibernate looks in the current session's persistence context.
   - **HIT:** return the live entity immediately. **(~nanoseconds)**
   - **MISS:** go to step 2.
2. **Step 2: is L2 usable?** Is the second-level cache enabled, and is this entity `@Cacheable`?
   - **NO:** go straight to the database (step 4).
   - **YES:** go to step 3.
3. **Step 3: L2 region lookup.** Hibernate asks the cache provider (e.g. the Ehcache / Redis region).
   - **HIT:** the stored values are read from L2. Hibernate builds a brand-new entity object, puts it into the current session's L1 cache, and returns it. **(~µs to 1 ms, zero DB queries!)**
   - **MISS:** go to step 4.
4. **Step 4: database query (fallback).**
   - Hibernate runs the SQL `SELECT`.
   - If L2 is enabled, Hibernate stores the row's values in the L2 region.
   - Hibernate creates the entity, stores it in the current session's L1 cache, and returns it. **(~ms, depends on network and query)**

---

## 6. What is stored: live object vs plain values

One of the biggest differences between L1 and L2 is **what is actually kept in memory**.

![svg](<svgs/05-live-object-vs-dehydrated-data.svg>)

- **In L1 cache:**
  - Stores the **actual live Java entity object**.
  - `User u1 = session.get(User.class, 101);`
  - `User u2 = session.get(User.class, 101);`
  - `u1 == u2` is **`true`** (exact same object in memory).
- **In L2 cache:**
  - Stores a **"dehydrated" array of plain values and IDs** (e.g. `Object[] { 101L, "Alice", "alice@example.com", 1 }`), NOT the Java entity object.
  - **Why?** If L2 stored live objects, two threads could change the same object at the same time with no transaction boundaries, causing race conditions and dirty reads.
  - When Session B reads from L2, Hibernate **hydrates** (builds) a *new* entity object from those values.
  - `User uA = sessionA.get(User.class, 101);`
  - `User uB = sessionB.get(User.class, 101);`
  - `uA == uB` is **`false`** (different objects), but `uA.equals(uB)` is **`true`** (if `equals()` is implemented on the ID / business key).

---

## 7. L1 vs L2: full comparison

| Feature | Level 1 (L1) Cache | Level 2 (L2) Cache |
| :--- | :--- | :--- |
| **Scope** | `Session` / `EntityManager` | `SessionFactory` / `EntityManagerFactory` |
| **Default state** | **Enabled** (mandatory) | **Disabled** (optional) |
| **Can it be disabled?** | ❌ No | ✅ Yes |
| **Lifetime** | Lives and dies with the session / transaction | Application runtime, until evicted |
| **Thread safety** | Thread-confined (not shared) | Thread-safe (shared by all sessions & threads) |
| **What is stored** | Live Java entity objects on the heap | Dehydrated values (copies) |
| **Same object?** | `u1 == u2` is `true` | `uA == uB` is `false` (new object per session) |
| **Configuration** | None needed | Provider jar + properties + annotations |
| **Providers** | Built into Hibernate (`StatefulPersistenceContext`) | Ehcache, Infinispan, Hazelcast, Redis/Redisson |
| **Eviction APIs** | `session.evict(e)`, `session.clear()` | `sessionFactory.getCache().evict(User.class, id)` |
| **Query results** | Not cached as results (the entities end up in L1) | Can be cached with the query cache |
| **Clustered / distributed** | ❌ No | ✅ Yes (with distributed providers like Infinispan / Redis) |

---

## 8. The query cache (L2 companion)

L2 caches entities **by primary key** (`id`). It does **not** cache JPQL / HQL query results (e.g. `SELECT u FROM User u WHERE u.age > 25`) by default.

That's the job of the **query cache**:
- The query cache stores **only the IDs** of the entities that matched the query (keyed by query text + parameters), not the entity data.
- When the same query runs again, Hibernate takes the IDs from the query cache and then loads each entity from the **L2 cache**.
- **Important:** if the entity is not in L2 (L2 disabled, or the entry was evicted), each ID is loaded with its own SELECT: **N+1 queries** (1 lookup for the IDs + N queries for the entities). Hibernate 6.5+ can store full entity data in the query cache instead (`hibernate.cache.query_cache_layout`).
- The cached result is **invalidated whenever any row in the queried tables changes** through Hibernate, so it only pays off for rarely changing tables.

![svg](<svgs/08-query-cache-flow.svg>)

```properties
# Enable Hibernate Query Cache
spring.jpa.properties.hibernate.cache.use_query_cache=true
```

In code (the hint is needed on **each** query you want cached):

```java
Query query = session.createQuery("FROM UserDetails u WHERE u.status = :status");
query.setParameter("status", "ACTIVE");
query.setHint(org.hibernate.jpa.AvailableHints.HINT_CACHEABLE, true);
List<UserDetails> users = query.getResultList();
```

With Spring Data JPA:

```java
@QueryHints(@QueryHint(name = AvailableHints.HINT_CACHEABLE, value = "true"))
List<UserDetails> findByStatus(String status);
```

---

## 9. Hands-on: Spring Boot configuration & example

### Step 1: Add dependencies (`pom.xml`)

```xml
<!-- Hibernate JCache & Ehcache 3 -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

### Step 2: Configure properties (`application.properties`)

```properties
# Enable Second-Level Cache
spring.jpa.properties.hibernate.cache.use_second_level_cache=true

# Specify Cache Region Factory (JCache implementation)
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory

# Path to Ehcache XML config
spring.jpa.properties.hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
spring.jpa.properties.hibernate.javax.cache.uri=classpath:ehcache.xml

# Only entities marked @Cacheable go to L2 (this is the default)
spring.jpa.properties.jakarta.persistence.sharedCache.mode=ENABLE_SELECTIVE

# Enable SQL & statistics logging to observe cache hits vs misses
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.generate_statistics=true
```

### Step 3: Configure cache regions (`src/main/resources/ehcache.xml`)

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://www.ehcache.org/v3"
        xsi:schemaLocation="http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-107-ext-3.0.xsd">

    <cache alias="com.example.entity.UserDetails">
        <key-type>java.lang.Long</key-type>
        <value-type>java.lang.Object</value-type>
        <expiry>
            <ttl unit="minutes">60</ttl>
        </expiry>
        <resources>
            <heap unit="entries">2000</heap>
        </resources>
    </cache>
</config>
```

### Step 4: Annotate the entity with `@Cacheable` and `@Cache`

```java
package com.example.entity;

import jakarta.persistence.*;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Table(name = "user_details")
@Cacheable   // jakarta.persistence.Cacheable — NOT Spring's @Cacheable!
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "com.example.entity.UserDetails")
public class UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String email;

    public UserDetails() {}

    public UserDetails(String name, String email) {
        this.name = name;
        this.email = email;
    }

    // Getters and Setters...
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}
```

### Step 5: Verify it

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    // Call 1: New HTTP Request / Session 1
    @Transactional(readOnly = true)
    public UserDetails fetchInRequestA(Long id) {
        return userRepository.findById(id).orElse(null); // L1 miss, L2 miss → SQL, stored in L2
    }

    // Call 2: Different HTTP Request / Session 2
    @Transactional(readOnly = true)
    public UserDetails fetchInRequestB(Long id) {
        return userRepository.findById(id).orElse(null); // L1 miss, L2 HIT → ZERO SQL
    }
}
```

Call them from two separate HTTP requests (e.g. `GET /users/101` twice). With `generate_statistics=true`, the session summary in the logs shows the difference:

```text
# Request 1
    1 JDBC statements executed;  1 L2C puts;  0 L2C hits;  1 L2C misses
# Request 2
    0 JDBC statements executed;  0 L2C puts;  1 L2C hits;  0 L2C misses
```

> If both lookups happen inside **one** `@Transactional` method, the second one is an **L1** hit and never even reaches L2. To test L2 you need two different sessions.

---

## 10. Production best practices, traps & eviction

### ✅ When to use the L2 cache

1. **Read-heavy, rarely changed entities:** lookup tables, roles, configuration parameters, countries, categories.
2. **Frequent primary-key reads:** entities repeatedly loaded by `findById()` or through `@ManyToOne` associations.
3. **Controlled updates:** entities modified only through Hibernate (not bulk native SQL or other systems).

### ❌ When NOT to use the L2 cache

1. **Write-heavy entities:** frequent updates cause constant invalidations and lock contention.
2. **Direct database updates:** if external cron jobs, ETL pipelines, or other services change rows directly, the L2 cache becomes **stale**.
3. **Multiple app instances with a local cache:** if 5 instances each use their own in-JVM Ehcache, instance A won't know when instance B updates a row! Use a **distributed / clustered** provider (Redis, Infinispan, Hazelcast) for multi-node setups.

![svg](<svgs/07-how-l2-goes-stale.svg>)

### 🛑 Large batch processing and L1

```java
// Anti-pattern: Loading 50,000 entities in one session -> OutOfMemoryError!
// Solution: Periodic Flush & Clear
for (int i = 0; i < entities.size(); i++) {
    entityManager.persist(entities.get(i));
    if (i % 50 == 0) {
        entityManager.flush(); // Sends SQL INSERTs to DB
        entityManager.clear(); // Empties L1 cache to reclaim heap
    }
}
```

(Combine with `spring.jpa.properties.hibernate.jdbc.batch_size=50` so the INSERTs are sent in JDBC batches.)

### 🧹 Programmatic L2 eviction

```java
// Evict specific entity from L2
sessionFactory.getCache().evict(UserDetails.class, 101L);

// Evict all entities of a type
sessionFactory.getCache().evict(UserDetails.class);

// Evict entire L2 cache
sessionFactory.getCache().evictAllRegions();

// JPA-standard equivalent
entityManagerFactory.getCache().evict(UserDetails.class, 101L);
```

---

## 11. Interview Questions

### Q1: What is the difference between L1 and L2 cache, in one sentence?
> L1 is a private, always-on cache per Session/transaction that avoids loading the same entity twice in one transaction. L2 is an optional, shared, application-wide cache that avoids reloading the same entity across different transactions and users.

### Q2: Can we disable the Hibernate first-level cache?
> **No.** The L1 cache *is* the persistence context. Hibernate needs it to guarantee "one object per row per session", to do dirty checking, and to safely handle bidirectional/circular references. You can only empty it (`clear()`, `detach()`), or use a `StatelessSession` for bulk work, which has no persistence context at all.

### Q3: Is the L1 cache shared between two requests or threads?
> No. Each request/transaction gets its own Session and its own L1. Two requests loading the same user both hit the database (unless L2 is enabled).

### Q4: Why does `findById` run SQL twice in my code even though L1 exists?
> Probably there is no surrounding `@Transactional`. Each Spring Data repository call then runs in its own transaction with its own session, so the second call starts with an empty L1. Wrap both calls in one `@Transactional` method.

### Q5: Why does the L2 cache store dehydrated values instead of live Java objects?
> To avoid concurrency problems. If L2 stored live objects, threads from different sessions would change the same object without transactional isolation. Storing plain values lets Hibernate build a clean, separate entity object for each session.

### Q6: Does the L2 cache help `findAll()` or `findByName()`?
> Not on its own. L2 is used for lookups by ID. A JPQL query always runs SQL. To cache query results you also need the **query cache**, which stores the matching IDs and then loads the entities from L2.

### Q7: What happens to the L2 cache if I update data with a bulk JPQL or native query (`@Modifying`)?
> Bulk statements skip the normal entity lifecycle, so Hibernate can't update individual cache entries:
> - **JPQL bulk update/delete:** Hibernate evicts the whole L2 region of the affected entity.
> - **Native SQL update:** Hibernate can't tell which tables changed, so by default it evicts **all** L2 regions (narrow it with `addSynchronizedEntityClass(...)`).
> - **Changes made outside the app** (another service, DBA, ETL): Hibernate never sees them, and L2 stays **stale** until TTL or manual eviction.
>
> `@Modifying(clearAutomatically = true)` clears the **L1** cache (the `EntityManager`) so the current transaction doesn't keep using outdated objects. It doesn't manage L2.

### Q8: Which concurrency strategy would you choose for a `Country` table? For `Product` prices?
> `Country`: `READ_ONLY` (never changes, fastest). `Product` with occasional price updates: `READ_WRITE`. If a few seconds of stale data is acceptable and updates are rare: `NONSTRICT_READ_WRITE`.

### Q9: Are collections cached automatically when the entity is cached?
> No. Add `@Cache` on the collection field as well. The collection cache stores only the child IDs, so the child entity must also be `@Cacheable`. Otherwise each element is loaded from the DB (N+1).

### Q10: L2 cache with multiple app instances: what is the risk?
> With a local (in-JVM) provider, each instance has its own copy. An update on instance B doesn't reach instance A's cache, so A serves stale data. Use a clustered/distributed provider (Infinispan, Hazelcast, Redisson) or short TTLs, or cache only read-only data.

### Q11: `jakarta.persistence.@Cacheable` vs Spring's `@Cacheable`?
> `jakarta.persistence.Cacheable` goes on an **entity** and marks it for Hibernate's L2 cache. `org.springframework.cache.annotation.Cacheable` goes on a **method** and caches that method's return value (any object: DTOs, API responses) in a Spring Cache (Caffeine, Redis…) using `@EnableCaching`. They are unrelated and configured separately.

### Q12: What is the difference between Hibernate L2 cache and a Redis cache?
> - **Hibernate L2 cache:** ORM-specific. It transparently caches entities (and query results) and is automatically invalidated when entities change through Hibernate.
> - **Redis cache (e.g. via Spring Cache):** a standalone, external, distributed key-value store. It caches any data you choose (DTOs, JSON, session state) and can be shared across microservices. You decide what goes in and when it is evicted. (Redis can also *be* the L2 provider through Redisson.)

### Q13: How do you check that the L2 cache is actually being used?
> Enable `hibernate.generate_statistics=true` and look for `L2C hits / puts / misses` in the session summary log, or read `sessionFactory.getStatistics().getSecondLevelCacheHitCount()`. You can also watch whether the SELECT disappears from the SQL log on the second request.

### Q14: What is a `StatelessSession` and how does it relate to L1?
> A Hibernate session with **no** persistence context: no L1 cache, no dirty checking, no cascades, no lazy loading. Every operation goes straight to the database. It's useful for large batch inserts/updates where L1 would only waste memory.
