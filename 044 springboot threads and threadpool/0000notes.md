# Spring Boot — Threads, Thread Pools and Concurrency

## The mental model

A Spring Boot app is **not** single-threaded, and it is **not** one big pool either. It runs **several independent thread pools**, each with its own settings. Tuning one does nothing to the others.

![thread pools in a spring boot app](<svgs/01-thread-pools-in-a-spring-boot-app.svg>)

| Pool | Thread names | Default size | Property |
|---|---|---|---|
| Tomcat HTTP workers | `http-nio-8080-exec-N` | **200** | `server.tomcat.threads.max` |
| `@Async` executor | `task-N` | **8** core | `spring.task.execution.pool.core-size` |
| `@Scheduled` scheduler | `scheduling-N` | **1** | `spring.task.scheduling.pool.size` |
| Kafka listeners | `kafka-listener-N` | 1 per consumer | `concurrency = "N"` |
| HikariCP (connections, not threads) | `HikariPool-1 housekeeper` | **10** connections | `spring.datasource.hikari.maximum-pool-size` |
| JVM internals | `GC`, `C2 CompilerThread`, `Finalizer`… | ~30–60 | not configurable |

The single most important consequence: **your controller code runs on a Tomcat thread that is shared with nothing, but your bean is shared with everyone.**

---

## How a request gets a thread (Tomcat)

![tomcat request pipeline](<svgs/02-tomcat-request-pipeline.svg>)

```
Client → OS accept queue → Acceptor/Poller → worker thread → your @RestController
         (accept-count)    (max-connections)  (threads.max)
```

**The three numbers and what each does:**

```properties
server.tomcat.threads.max=200        # concurrent requests being PROCESSED
server.tomcat.threads.min-spare=10   # idle threads kept warm
server.tomcat.max-connections=8192   # open sockets held (incl. idle keep-alive)
server.tomcat.accept-count=100       # OS backlog of not-yet-accepted connections
```

**Connections ≠ threads.** With NIO, Tomcat can hold 8192 open connections using only 200 threads, because an idle keep-alive socket is watched by the poller and consumes no worker. A thread is borrowed **only while a request is actively being processed**.

What happens when each fills:

| Full | Result |
|---|---|
| `threads.max` | Requests wait for a free worker — latency climbs, no error yet |
| `max-connections` | New connections go to the accept queue |
| `accept-count` | OS rejects the TCP handshake → **"Connection refused"** at the client |

Other servers:

```properties
# Jetty
server.jetty.threads.max=200
server.jetty.threads.min=8

# Undertow  (io = 1 per core, worker = io × 8)
server.undertow.threads.io=4
server.undertow.threads.worker=32
```

---

## What is the maximum number of threads?

This is the question everyone asks, and it has **four different answers** depending on which ceiling you hit first.

![max thread ceilings](<svgs/03-max-threads-ceilings.svg>)

### 1. Framework ceiling — what you configure

```properties
server.tomcat.threads.max=200     # default
```

This is a hard cap you choose. Sane range for a normal REST service: **100–400**.

### 2. JVM memory ceiling

Every **platform thread** gets its own stack in **native memory — not the heap**. Default `-Xss` is **1 MB** on 64-bit Linux.

```
max threads ≈ (total RAM − heap − metaspace − code cache) / -Xss
```

Worked example on an 8 GB box with `-Xmx4g`:

```
8 GB  − 4 GB heap − ~0.5 GB (metaspace, code cache, GC structures)
= ~3.5 GB native available
÷ 1 MB per thread stack
≈ 3500 threads   ← theoretical maximum
```

Exceed it and you get:

```
java.lang.OutOfMemoryError: unable to create native thread
```

Note this OOM is about **native** memory, so increasing `-Xmx` makes it *worse*, not better. You can trade stack size for thread count:

```
-Xss512k     → roughly doubles the number of threads you can create
```

(Do not go below ~256k — you risk `StackOverflowError` on deep frameworks stacks.)

### 3. OS ceiling

```bash
ulimit -u                          # max processes/threads per user
cat /proc/sys/kernel/threads-max   # system-wide (often 100,000+)
cat /proc/sys/vm/max_map_count     # each stack needs a mapping
```

In Docker/Kubernetes also check the cgroup `pids.max` limit.

### 4. Practical ceiling — the real answer

**A few hundred. Rarely useful beyond ~1000.**

Long before memory runs out, context switching destroys throughput. The CPU spends its time swapping between threads instead of running them, and every downstream resource (DB pool, remote API) becomes the real bottleneck.

> Going from 200 → 2000 threads almost always makes throughput **worse**, not better.

### So how do I pick the right number?

**Little's Law** — how many threads you need to sustain a target rate:

```
threads = target throughput (req/s) × average response time (s)

Example:  500 req/s × 0.2 s = 100 threads
```

**Brian Goetz's formula** — accounting for how much of the time is waiting:

```
threads = cores × target CPU utilisation × (1 + wait time / compute time)

8 cores, 90% utilisation, 90 ms waiting on DB + 10 ms computing:
= 8 × 0.9 × (1 + 90/10)
= 8 × 0.9 × 10
= 72 threads
```

Rules of thumb:

| Workload | Threads |
|---|---|
| CPU-bound (computation, parsing) | `cores` or `cores + 1` |
| I/O-bound (DB, HTTP calls) | much higher — use the formula above |
| Mixed REST service | start at 200, measure, adjust |

### The trap: threads vs DB connections

```properties
server.tomcat.threads.max=200
spring.datasource.hikari.maximum-pool-size=10     # default!
```

200 threads all fighting over **10** connections. 190 of them sit in `TIMED_WAITING` inside `HikariPool.getConnection()`. Raising `threads.max` here achieves nothing — the connection pool is the real limit.

HikariCP's own sizing advice is famously *small*:

```
connections = (cores × 2) + effective spindle count
```

A bigger connection pool is usually **slower**, because the database itself does not benefit from more concurrent sessions than it has cores/disks.

---

## `@Async` — running work on another thread

**Enable it:**

```java
@SpringBootApplication
@EnableAsync
public class DemoApplication { }
```

**Use it:**

```java
@Service
public class ReportService {

    @Async
    public void fireAndForget(Long id) {
        log.info("running on {}", Thread.currentThread().getName());
        // long job
    }

    @Async
    public CompletableFuture<Report> buildReport(Long id) {
        Report r = slowBuild(id);
        return CompletableFuture.completedFuture(r);
    }
}
```

**Calling it:**

```java
@GetMapping("/report/{id}")
public String trigger(@PathVariable Long id) {
    log.info("controller on {}", Thread.currentThread().getName());
    reportService.fireAndForget(id);
    return "accepted";
}
```

### Output

```
10:12:04.118 [http-nio-8080-exec-1] INFO  c.a.ReportController - controller on http-nio-8080-exec-1
10:12:04.121 [task-1]               INFO  c.a.ReportService    - running on task-1
```

The controller returned immediately on `exec-1`; the work continued on `task-1`.

### Running several async calls in parallel

```java
@GetMapping("/dashboard")
public Dashboard dashboard() throws Exception {
    CompletableFuture<User>          user   = userService.loadAsync();      // 100 ms
    CompletableFuture<List<Order>>   orders = orderService.loadAsync();     // 100 ms
    CompletableFuture<List<Product>> recos  = recoService.loadAsync();      // 100 ms

    CompletableFuture.allOf(user, orders, recos).join();   // wait for all

    return new Dashboard(user.get(), orders.get(), recos.get());
}
```

```
Sequential : 100 + 100 + 100 = ~300 ms
Parallel   : max(100,100,100) = ~105 ms
```

### Configuring the `@Async` pool

```properties
spring.task.execution.pool.core-size=8
spring.task.execution.pool.max-size=32
spring.task.execution.pool.queue-capacity=100
spring.task.execution.pool.keep-alive=60s
spring.task.execution.thread-name-prefix=async-
```

Or as a bean, when you want several pools:

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("reportExecutor")
    public Executor reportExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("report-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// then:
@Async("reportExecutor")
public void build() { ... }
```

### `@Async` gotchas

| Gotcha | Why |
|---|---|
| **Self-invocation does nothing** | `this.asyncMethod()` bypasses the proxy — no new thread. Call it from *another* bean. |
| Method must be `public` | Proxy cannot intercept private/protected methods |
| Return type must be `void`, `Future`, `CompletableFuture` | Returning a plain `Report` gives you `null` |
| Exceptions vanish on `void` methods | Use `AsyncUncaughtExceptionHandler`, or return a `CompletableFuture` |
| `ThreadLocal` does not carry over | Security context, MDC, request scope are lost — see below |

---

## `@Scheduled` — the one-thread trap

```java
@Component
public class Jobs {

    @Scheduled(fixedRate = 5000)
    public void jobA() { Thread.sleep(10_000); }   // takes 10s!

    @Scheduled(fixedRate = 1000)
    public void jobB() { log.info("tick"); }
}
```

### Output with defaults

```
10:00:00.001 [scheduling-1] jobA start
10:00:10.004 [scheduling-1] jobA end
10:00:10.005 [scheduling-1] tick      ← jobB starved for 10 seconds
```

**`spring.task.scheduling.pool.size` defaults to 1.** One slow job delays every other scheduled job in the whole application.

```properties
spring.task.scheduling.pool.size=5
```

```
10:00:00.001 [scheduling-1] jobA start
10:00:01.002 [scheduling-2] tick      ← now independent
10:00:02.001 [scheduling-3] tick
```

Also note: `fixedRate` vs `fixedDelay`

| | Meaning |
|---|---|
| `fixedRate = 5000` | Start every 5 s, regardless of how long the last run took |
| `fixedDelay = 5000` | Wait 5 s **after the previous run finishes** |
| `cron = "0 0 2 * * *"` | Cron expression (2 AM daily) |

> In a multi-instance deployment, `@Scheduled` runs on **every instance**. Use ShedLock or a DB lock if the job must run once cluster-wide.

---

## ThreadPoolTaskExecutor — the growth order everyone gets wrong

![thread pool growth order](<svgs/04-threadpool-growth-order.svg>)

When a task is submitted, the executor does this **in order**:

1. Threads < `core-size`? → **create a new thread** (even if others are idle)
2. Queue has room? → **enqueue the task** (pool does **not** grow)
3. Threads < `max-size`? → **now** create more threads
4. Otherwise → **reject** (`RejectedExecutionException`)

**The counter-intuitive part:** the queue fills up *before* the pool grows past core size.

### The Spring Boot default trap

```properties
spring.task.execution.pool.core-size=8
spring.task.execution.pool.max-size=<Integer.MAX_VALUE>       # default
spring.task.execution.pool.queue-capacity=<Integer.MAX_VALUE> # default
```

Because the queue is **unbounded**, step 2 never fails, so step 3 never runs. **You are stuck at 8 threads forever**, and the queue grows until you run out of heap.

**Fix:** always bound the queue.

```properties
spring.task.execution.pool.core-size=8
spring.task.execution.pool.max-size=32
spring.task.execution.pool.queue-capacity=100
```

### Rejection policies

| Policy | Behaviour |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | The submitting thread runs the task — natural backpressure, **usually the best choice** |
| `DiscardPolicy` | Silently drops the task |
| `DiscardOldestPolicy` | Drops the oldest queued task, then retries |

---

## Thread safety — the #1 Spring concurrency bug

Spring beans are **singletons by default**. One instance serves all 200 request threads at the same time.

![singleton bean thread safety](<svgs/05-singleton-bean-thread-safety.svg>)

```java
// BROKEN — works perfectly with 1 user, corrupts data with 200
@Service
public class OrderService {

    private int counter = 0;          // SHARED across all threads
    private User currentUser;         // SHARED — thread A overwrites thread B

    public void process(User u) {
        this.currentUser = u;         // race condition
        this.counter++;               // lost updates (not atomic)
        validate(this.currentUser);   // might be someone else's user!
    }
}
```

```java
// CORRECT — stateless; per-request data lives on the stack
@Service
public class OrderService {

    private final OrderRepository repo;                  // final, immutable ref
    private final AtomicLong counter = new AtomicLong(); // thread-safe

    public OrderService(OrderRepository repo) { this.repo = repo; }

    public void process(User u) {       // u is a LOCAL → one copy per thread
        counter.incrementAndGet();
        validate(u);
    }
}
```

| Safe as a bean field | Unsafe as a bean field |
|---|---|
| `final` injected dependencies | plain `int` / `long` counters |
| immutable config values | per-request objects (`User`, DTOs) |
| `AtomicInteger`, `AtomicLong`, `LongAdder` | `ArrayList`, `HashMap`, `StringBuilder` |
| `ConcurrentHashMap`, `CopyOnWriteArrayList` | `SimpleDateFormat` ← classic production bug |
| stateless helpers | any mutable object shared between calls |

> `SimpleDateFormat` is not thread-safe and produces silently wrong dates under load. Use `DateTimeFormatter` (immutable, thread-safe).

**If you genuinely need per-request state:**

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String traceId;   // safe — one instance per HTTP request
}
```

---

## ThreadLocal and why context gets lost

Spring keeps a lot of per-request data in `ThreadLocal`:

| Holder | What it stores |
|---|---|
| `RequestContextHolder` | current `HttpServletRequest` |
| `SecurityContextHolder` | authenticated user (Spring Security) |
| `TransactionSynchronizationManager` | the active transaction / EntityManager |
| `LocaleContextHolder` | request locale |
| SLF4J `MDC` | logging correlation id |

Because these live on the *thread*, **they do not follow work onto an `@Async` thread**:

```java
@Async
public void audit() {
    // SecurityContextHolder.getContext().getAuthentication() → null !
    // MDC.get("traceId") → null !
}
```

Fixes:

```java
// propagate Spring Security
SecurityContextHolder.setStrategyName(
        SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);

// or wrap the executor
executor.setTaskDecorator(task -> {
    var context  = RequestContextHolder.getRequestAttributes();
    var security = SecurityContextHolder.getContext();
    var mdc      = MDC.getCopyOfContextMap();
    return () -> {
        try {
            RequestContextHolder.setRequestAttributes(context);
            SecurityContextHolder.setContext(security);
            if (mdc != null) MDC.setContextMap(mdc);
            task.run();
        } finally {
            RequestContextHolder.resetRequestAttributes();
            SecurityContextHolder.clearContext();
            MDC.clear();
        }
    };
});
```

> **Leak warning:** pool threads are reused forever. If you `set()` a `ThreadLocal` and never `remove()` it, the value survives into the *next* user's request — a real data-leak bug, and a memory leak too.

---

## Concurrency tools you will actually use

```java
// 1. Atomic — lock-free counters
private final AtomicLong hits = new AtomicLong();
hits.incrementAndGet();

// LongAdder is faster under heavy contention
private final LongAdder requests = new LongAdder();
requests.increment();

// 2. Concurrent collections
private final Map<String, User> cache = new ConcurrentHashMap<>();
cache.computeIfAbsent(key, k -> loadUser(k));    // atomic

// 3. synchronized — simple but blocks everything
public synchronized void update() { ... }         // locks the whole object

// 4. ReentrantLock — more control, and virtual-thread friendly
private final ReentrantLock lock = new ReentrantLock();
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
}

// 5. ReadWriteLock — many readers, one writer
private final ReadWriteLock rw = new ReentrantReadWriteLock();

// 6. Semaphore — rate limiting / bulkhead
private final Semaphore permits = new Semaphore(10);
permits.acquire();
try { callFlakyApi(); } finally { permits.release(); }

// 7. CompletableFuture — composition
CompletableFuture.supplyAsync(() -> loadA(), executor)
        .thenCombine(CompletableFuture.supplyAsync(() -> loadB(), executor),
                     (a, b) -> merge(a, b))
        .thenApply(this::transform)
        .exceptionally(ex -> fallback());
```

**Prefer, in order:** immutability → atomics/concurrent collections → explicit locks → `synchronized`.

---

## Virtual threads (Java 21+, Spring Boot 3.2+)

![platform vs virtual threads](<svgs/06-platform-vs-virtual-threads.svg>)

```properties
spring.threads.virtual.enabled=true
```

That single line makes Tomcat use **one virtual thread per request**. Your blocking JDBC code is unchanged, but a thread parked on I/O no longer holds an OS thread hostage.

| | Platform thread | Virtual thread |
|---|---|---|
| Backed by | 1 OS thread | JVM object on the heap |
| Stack | ~1 MB native | few hundred bytes, grows on demand |
| Creation cost | ~1 ms | ~1 µs |
| Realistic max | a few thousand | **millions** |
| Blocking I/O | parks the OS thread | unmounts, frees the carrier |
| Should you pool them? | Yes | **No — create one per task** |

### Output difference

```
# without virtual threads
[http-nio-8080-exec-1] INFO - handling request

# with spring.threads.virtual.enabled=true
[tomcat-handler-0] INFO - handling request        ← a virtual thread
```

### Caveats

- **Pinning** — on Java 21–23, a `synchronized` block holds the carrier thread and blocks other virtual threads. Use `ReentrantLock` instead. (JDK 24 removed this limitation.)
- **No help for CPU-bound work** — you still have the same number of cores.
- **The DB connection pool is still the limit** — 10,000 virtual threads still queue for 10 Hikari connections.
- **No backpressure** — that remains WebFlux's advantage.
- **Do not pool them** — `Executors.newVirtualThreadPerTaskExecutor()`, never a fixed pool.

---

## Diagnosing thread problems

![thread states](<svgs/07-thread-states.svg>)

### Thread states

| State | Meaning |
|---|---|
| `NEW` | Created, not started |
| `RUNNABLE` | Running, **or blocked on socket I/O** (the JVM cannot tell) |
| `BLOCKED` | Waiting to enter a `synchronized` block — **contention** |
| `WAITING` | `wait()`, `join()`, `park()` with no timeout |
| `TIMED_WAITING` | `sleep(n)`, `poll(timeout)` — normal for idle pool threads |
| `TERMINATED` | Finished |

### Taking a thread dump

```bash
jcmd <pid> Thread.print > dump.txt
jstack <pid> > dump.txt
kill -3 <pid>            # goes to stdout
```

or, from the running app:

```
GET /actuator/threaddump
```

### What to look for

```
"http-nio-8080-exec-12" #39 daemon prio=5 BLOCKED on 0x000000076ab2
   at com.acme.OrderService.process(OrderService.java:42)
```
→ Lock contention in your own code. Many threads BLOCKED on the same monitor = a `synchronized` bottleneck.

```
"http-nio-8080-exec-3" #28 daemon TIMED_WAITING (parking)
   at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:...)
```
→ Connection-pool starvation. Raising `threads.max` will **not** help; fix the pool size or the slow queries.

```
Found one Java-level deadlock:
  "thread-1" is waiting to lock monitor owned by "thread-2"
  "thread-2" is waiting to lock monitor owned by "thread-1"
```
→ Classic deadlock. `jstack` detects and prints these automatically.

### Monitoring with Actuator

```properties
management.endpoints.web.exposure.include=health,metrics,threaddump
```

Useful metrics:

```
tomcat.threads.busy          threads currently serving requests
tomcat.threads.current       threads alive in the pool
tomcat.threads.config.max    the configured ceiling
executor.active              @Async threads working
executor.queued              tasks waiting in the @Async queue
executor.pool.size           current @Async pool size
hikaricp.connections.active  DB connections in use
hikaricp.connections.pending threads WAITING for a connection  ← watch this
jvm.threads.live             total JVM threads
jvm.threads.peak             high-water mark
```

**The alert that matters:** `tomcat.threads.busy / tomcat.threads.config.max > 0.8` sustained means you are close to saturating the pool.

---

## Configuration reference

```properties
# ---- Tomcat (HTTP) ----
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10
server.tomcat.max-connections=8192
server.tomcat.accept-count=100
server.tomcat.connection-timeout=20s

# ---- @Async executor ----
spring.task.execution.pool.core-size=8
spring.task.execution.pool.max-size=32
spring.task.execution.pool.queue-capacity=100
spring.task.execution.pool.keep-alive=60s
spring.task.execution.thread-name-prefix=async-
spring.task.execution.shutdown.await-termination=true
spring.task.execution.shutdown.await-termination-period=30s

# ---- @Scheduled ----
spring.task.scheduling.pool.size=5
spring.task.scheduling.thread-name-prefix=sched-

# ---- DB connections ----
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.leak-detection-threshold=60000

# ---- Virtual threads (Java 21+) ----
spring.threads.virtual.enabled=true

# ---- Graceful shutdown: let in-flight requests finish ----
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Mutable field in a singleton bean | Random wrong data under load only | Make beans stateless; pass state as parameters |
| `SimpleDateFormat` as a field | Corrupted/garbled dates | Use `DateTimeFormatter` |
| `@Async` called via `this.method()` | Runs synchronously, no new thread | Call from another bean |
| `@Scheduled` pool size 1 | One slow job starves all others | `spring.task.scheduling.pool.size=5` |
| Unbounded `@Async` queue | Pool never grows past 8; heap fills | Set `queue-capacity` |
| `threads.max` ≫ Hikari pool size | Threads pile up in `TIMED_WAITING` | Size the connection pool, not the thread pool |
| `ThreadLocal` never cleared | Data leaks into the next request | `remove()` in a `finally` block |
| Raising `threads.max` to "fix" slowness | Throughput gets worse | Find the real bottleneck first (DB, downstream API) |
| `synchronized` with virtual threads (JDK ≤ 23) | Carrier thread pinning, poor scaling | Use `ReentrantLock` |
| No graceful shutdown | In-flight requests killed on deploy | `server.shutdown=graceful` |

---

## Summary

- Spring Boot runs **several separate pools** — Tomcat (200), `@Async` (8), `@Scheduled` (1), plus JVM internals.
- **Connections ≠ threads.** Tomcat holds 8192 sockets with 200 workers; a thread is only borrowed during active processing.
- **Max threads has four ceilings:** your config → JVM native memory (`RAM / -Xss`) → OS limits → and the one that actually bites, **context switching at a few hundred threads**.
- Size pools with **Little's Law** (`throughput × latency`) or **Goetz's formula** (`cores × utilisation × (1 + wait/compute)`), then measure.
- **Your thread pool is rarely the bottleneck** — the DB connection pool usually is.
- `ThreadPoolTaskExecutor` fills the **queue before** growing past core size; an unbounded queue means the pool never grows.
- **Singleton beans are shared by every request thread** — keep them stateless. This is the #1 Spring concurrency bug.
- `ThreadLocal` context (security, MDC, transaction) **does not follow `@Async`** — propagate it explicitly and always clean it up.
- **Virtual threads** (Java 21 / Boot 3.2+, one property) remove the thread-count ceiling entirely while keeping ordinary blocking code.
- Diagnose with `jcmd Thread.print`, `/actuator/threaddump`, and watch `tomcat.threads.busy` and `hikaricp.connections.pending`.
