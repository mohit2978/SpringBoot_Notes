# Spring WebFlux

## Table of Contents

1. [What is it (in simple words)](#1-what-is-it-in-simple-words)
2. [Why it's needed: the problem it solves](#2-why-its-needed-the-problem-it-solves)
3. [How the event loop actually works (Netty)](#3-how-the-event-loop-actually-works-netty)
4. [The two stacks side by side](#4-the-two-stacks-side-by-side)
5. [Reactive Streams: the specification underneath](#5-reactive-streams-the-specification-underneath)
6. [Mono and Flux](#6-mono-and-flux)
7. [Setup](#7-setup)
8. [A request's path through WebFlux](#8-a-requests-path-through-webflux)
9. [Code: approach 1, annotated controllers](#9-code-approach-1-annotated-controllers)
10. [Code: approach 2, functional endpoints](#10-code-approach-2-functional-endpoints)
11. [Code: streaming with SSE](#11-code-streaming-with-sse-where-webflux-really-shines)
12. [Code: WebClient (calling other services)](#12-code-webclient-calling-other-services)
13. [Operators you will actually use](#13-operators-you-will-actually-use)
14. [Error handling](#14-error-handling)
15. [Threading model and Schedulers](#15-threading-model-and-schedulers)
16. [ThreadLocal breaks: use Reactor Context (and WebFilter)](#16-threadlocal-breaks-use-reactor-context-and-webfilter)
17. [Data access with R2DBC](#17-data-access-with-r2dbc)
18. [Security in WebFlux](#18-security-in-webflux)
19. [Testing](#19-testing)
20. [Debugging reactive code](#20-debugging-reactive-code)
21. [Benefits](#21-benefits)
22. [Disadvantages](#22-disadvantages)
23. [When to use / when not to (and virtual threads)](#23-when-to-use--when-not-to)
24. [Common pitfalls](#24-common-pitfalls)
25. [Summary](#25-summary)
26. [Interview Questions](#26-interview-questions)

---

## 1. What is it (in simple words)

Spring WebFlux is the **reactive, non-blocking web framework** Spring added in Spring 5 (Spring Boot 2). It sits **next to** Spring MVC as a completely separate stack. It is not a replacement and not an upgrade: you pick one per application.

Three things define it:

1. **Non-blocking:** a thread is never parked waiting for I/O.
2. **Asynchronous:** you return a *promise of data* (`Mono`/`Flux`), not the data itself.
3. **Backpressure-aware:** the consumer tells the producer how much it can take.

It's built on **Project Reactor**, an implementation of the **Reactive Streams** specification. By default it runs on **Netty** instead of Tomcat.

```
Spring MVC     →  1 request = 1 thread, the thread blocks during I/O
Spring WebFlux →  1 thread  = many requests, the thread never blocks
```

> **Restaurant analogy**
> - **Spring MVC** = one waiter per table. The waiter takes your order, walks to the kitchen and **stands there** until the food is ready. 200 waiters → at most 200 tables. Table 201 waits at the door.
> - **WebFlux** = a few waiters for the whole restaurant. A waiter takes an order, hands it to the kitchen, and **immediately serves other tables**. When the kitchen rings the bell, any free waiter brings the food. A few waiters handle hundreds of tables, **as long as no waiter ever stands and waits** (that's the "never block" rule).

---

## 2. Why it's needed: the problem it solves

### 2.1 The thread-per-request model runs out of threads

In Spring MVC, every incoming request takes a thread from the Tomcat pool (default **200**) and keeps it until the response is written. If the request calls a database or another microservice, that thread **sits there doing nothing** while it waits.

The CPU is idle, but the thread isn't free. It's parked inside a blocking socket read, and no other request can use it.

Do the maths:

```
200 threads in the pool
each request waits 100 ms on the DB

throughput ceiling = 200 / 0.1s = ~2000 requests/sec   ← hard wall
request 2001 queues up and eventually times out
```

Each thread also costs roughly **1 MB of stack memory**, so 200 idle threads = 200 MB of RAM spent on waiting. Raise the pool to 2000 threads and you get 2 GB of stacks plus heavy context switching, which makes things *slower*, not faster.

This is the classic **C10K problem**: how do you serve 10,000 concurrent connections on one machine?

### 2.2 The fix: stop tying a thread to a request

WebFlux uses a small pool of **event-loop threads** (by default about the number of CPU cores, minimum 4). When a handler starts an I/O call, the thread registers a callback and **immediately moves on to the next request**. When the database answers, the event loop picks the work back up.

Same hardware, same 100 ms database, but now a handful of threads serve tens of thousands of concurrent requests, because no thread is ever idle-waiting.

![svg](<svgs/01-thread-per-request-vs-event-loop.svg>)

**The catch, stated up front:** this only works if *nothing* in the chain blocks. One JDBC call inside a WebFlux handler freezes every request sharing that event-loop thread. This is the single biggest source of WebFlux bugs in production.

---

## 3. How the event loop actually works (Netty)

Netty uses the operating system's non-blocking I/O (epoll on Linux, kqueue on macOS). One event-loop thread watches **many sockets at once** and only does work when a socket has something ready:

![svg](<svgs/02-netty-event-loop.svg>)

1. **select():** ask the OS "which of my sockets have data ready?" (a new request, a DB reply, space to write a response).
2. **Run the handler code** for those events: parse the request, run your controller, apply operators. This must be **quick**.
3. **Start I/O and don't wait:** the R2DBC driver or `WebClient` writes the query/request to a non-blocking socket and registers a callback.
4. **When the reply arrives**, it shows up as a ready socket in step 1, and its callback continues the reactive chain.

You can see these threads in the logs: `reactor-http-nio-1`, `reactor-http-nio-2`, … (one per event loop).

```
10:14:02.115 [reactor-http-nio-3] INFO  UserController - fetching user 1
10:14:02.131 [reactor-http-nio-3] INFO  UserController - got user 1      ← same few threads, many requests
```

If step 2 contains `Thread.sleep()`, a JDBC call, a blocking file read or `.block()`, the loop is stuck there, and **all** connections assigned to that thread stop being served.

---

## 4. The two stacks side by side

![svg](<svgs/11-mvc-vs-webflux-stack.svg>)

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Starter | `spring-boot-starter-web` | `spring-boot-starter-webflux` |
| Default server | Tomcat | Netty |
| Threads | ~200, one per request | ~cores, shared by all |
| Front controller | `DispatcherServlet` | `DispatcherHandler` |
| Request object | `HttpServletRequest` | `ServerHttpRequest` / `ServerWebExchange` |
| Filters | Servlet `Filter`, `HandlerInterceptor` | `WebFilter` |
| Return type | `User`, `List<User>` | `Mono<User>`, `Flux<User>` |
| Data access | JPA / Hibernate / JDBC (blocking) | R2DBC / reactive Mongo, Redis, Cassandra |
| HTTP client | `RestClient` / `RestTemplate` | `WebClient` |
| Security | `SecurityFilterChain`, `SecurityContextHolder` | `SecurityWebFilterChain`, `ReactiveSecurityContextHolder` |
| Testing | `MockMvc` | `WebTestClient`, `StepVerifier` |
| Debugging | normal stack traces | fragmented stack traces |
| Learning curve | low | steep |

> If you put both `spring-boot-starter-web` and `spring-boot-starter-webflux` on the classpath, you don't get WebFlux: Spring Boot detects the servlet stack and starts Spring MVC. Remove the web starter. (Having WebFlux on the classpath *just to use `WebClient`* inside an MVC app is fine and common.)

---

## 5. Reactive Streams: the specification underneath

Reactive Streams is a tiny spec (4 interfaces) that Reactor, RxJava, Akka Streams and Java 9's `java.util.concurrent.Flow` all implement.

| Interface | Role |
|---|---|
| `Publisher<T>` | The source. Has one method: `subscribe(Subscriber)` |
| `Subscriber<T>` | The consumer. Has `onSubscribe`, `onNext`, `onError`, `onComplete` |
| `Subscription` | The control handle. Has `request(long n)` and `cancel()` |
| `Processor<T,R>` | Both a Publisher and a Subscriber (used to build operators) |

The handshake between them is what makes backpressure possible:

![svg](<svgs/04-reactive-streams-protocol.svg>)

1. The subscriber calls `publisher.subscribe(subscriber)`.
2. The publisher calls `subscriber.onSubscribe(subscription)`.
3. The subscriber asks for items: `subscription.request(n)`.
4. The publisher sends **at most n** items with `onNext(item)`.
5. Repeat 3–4 as needed. At the end it calls `onComplete()` (success) **or** `onError(e)` (failure). After either, nothing else is sent.

### 5.1 Backpressure

Backpressure is the consumer saying *"send me only 2 more, that's all I can handle"*. Without it, a fast producer and a slow consumer means an unbounded queue and eventually an `OutOfMemoryError`.

This is the key difference from plain callbacks or Java 8 Streams: the demand flows **upstream**.

![svg](<svgs/05-backpressure.svg>)

When the source **can't** slow down (sensor data, user clicks, a message broker pushing), choose a strategy:

```java
Flux<Reading> readings = sensor.readings();

readings.limitRate(100);                     // request in batches of 100 instead of "unbounded"
readings.onBackpressureBuffer(10_000);       // queue up to 10k, then signal an error
readings.onBackpressureDrop(r -> log.warn("dropped {}", r)); // throw away what can't be handled
readings.onBackpressureLatest();             // keep only the newest item (good for "current price")
```

Over HTTP, backpressure rides on TCP flow control: if a client reads a streaming response slowly, Netty's write buffer fills up and it stops requesting more items from your `Flux`.

---

## 6. Mono and Flux

Project Reactor gives you exactly two publisher types, and you'll use them everywhere.

| Type | Emits | Use for |
|---|---|---|
| `Mono<T>` | 0 or 1 item, then completes | one user, one save, one HTTP call, `void` (`Mono<Void>`) |
| `Flux<T>` | 0 to N items (can be infinite) | lists, streams, SSE, paged data |

![svg](<svgs/06-mono-flux-marble.svg>)

### 6.1 Creating them

| Factory | What it does |
|---|---|
| `Mono.just(value)` / `Flux.just(a, b, c)` | Emit values you already have |
| `Mono.empty()` / `Flux.empty()` | Complete without emitting |
| `Mono.error(ex)` | Fail immediately |
| `Mono.fromCallable(() -> ...)` | Run a (possibly blocking) call **lazily**, at subscription |
| `Mono.defer(() -> mono)` | Build the publisher lazily, per subscriber |
| `Flux.fromIterable(list)` | Emit the elements of a collection |
| `Flux.range(1, 5)` | 1, 2, 3, 4, 5 |
| `Flux.interval(Duration.ofSeconds(1))` | 0, 1, 2, … every second (infinite) |
| `Flux.create(sink -> ...)` / `Sinks.many()` | Bridge from callbacks / push events into a Flux |

```java
Mono<String> m1 = Mono.just(expensiveCall());              // ⚠️ expensiveCall() runs NOW, while building
Mono<String> m2 = Mono.fromCallable(() -> expensiveCall()); // runs only when someone subscribes
```

### 6.2 Nothing happens until you subscribe

This trips up everyone at least once. A `Mono`/`Flux` is a **recipe, not a running task**.

![svg](<svgs/07-lazy-cold-vs-hot.svg>)

```java
Mono<User> mono = userRepository.save(user);   // NOTHING has happened yet
// no insert has been issued, the DB has not been touched
```

The chain only runs when something subscribes. In a controller, **Spring subscribes for you** when you return the publisher, which is why you must `return` it and never ignore it.

```java
// WRONG: silently does nothing
@PostMapping("/users")
public void create(@RequestBody User u) {
    userRepository.save(u);        // no subscriber → never executes
}

// RIGHT
@PostMapping("/users")
public Mono<User> create(@RequestBody User u) {
    return userRepository.save(u); // Spring subscribes → it runs
}
```

**Cold vs hot publishers**
- **Cold (default):** each subscriber triggers the work again from scratch. Subscribing twice to `userRepository.findAll()` runs **two** queries.
- **Hot:** emits regardless of subscribers; late subscribers only see new items (a live price feed, chat messages, `Sinks.many().multicast()`). Use `.cache()` or `.share()` to turn a cold source into a shared one.

---

## 7. Setup

**pom.xml**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- reactive database driver: NOT spring-boot-starter-data-jpa -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

**application.properties**

```properties
spring.r2dbc.url=r2dbc:h2:mem:///userDB
spring.r2dbc.username=sa
spring.r2dbc.password=

# see which thread everything runs on: very useful while learning
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{20} - %msg%n
```

**src/main/resources/schema.sql.** R2DBC has **no** `ddl-auto`, so you create tables yourself:

```sql
CREATE TABLE IF NOT EXISTS users (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(100),
    email VARCHAR(100)
);
```

The startup log confirms the stack:

```
Netty started on port 8080
Started DemoApplication in 1.412 seconds
```

(Spring MVC would say `Tomcat started on port(s): 8080`.)

---

## 8. A request's path through WebFlux

![svg](<svgs/03-webflux-request-path.svg>)

1. **Netty** accepts the connection and reads the HTTP request on an event-loop thread.
2. **HttpHandler** adapts Netty's request/response into Spring's `ServerHttpRequest`/`ServerHttpResponse` (bundled as a `ServerWebExchange`).
3. The **WebFilter chain** runs (your filters, CORS, Spring Security's `SecurityWebFilterChain`).
4. **DispatcherHandler** (the front controller):
   - `HandlerMapping` finds the handler (`@GetMapping` method or `RouterFunction` route);
   - `HandlerAdapter` invokes it, and it returns a `Mono`/`Flux` almost immediately;
   - `HandlerResultHandler` **subscribes** to that publisher and writes items to the response as they arrive (JSON array, SSE events, …).

---

## 9. Code: approach 1, annotated controllers

This looks almost identical to Spring MVC. Only the return types change.

**Entity**

```java
@Table("users")
public class User {

    @Id
    private Long id;
    private String name;
    private String email;

    // constructors, getters and setters
}
```

**Repository:** extend `ReactiveCrudRepository`, not `JpaRepository`.

```java
@Repository
public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Flux<User> findByName(String name);          // many → Flux
    Mono<User> findByEmail(String email);        // one  → Mono
}
```

**Service**

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }

    public Mono<User> getUserById(Long id) {
        return userRepository.findById(id)
                .switchIfEmpty(Mono.error(
                        new ResponseStatusException(HttpStatus.NOT_FOUND, "User " + id + " not found")));
    }

    public Mono<User> createUser(User user) {
        return userRepository.save(user);
    }

    public Mono<Void> deleteUser(Long id) {
        return userRepository.deleteById(id);
    }
}
```

**Controller**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public Flux<User> getAll() {
        return userService.getAllUsers();
    }

    @GetMapping("/{id}")
    public Mono<User> getOne(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> create(@RequestBody User user) {
        return userService.createUser(user);
    }

    @DeleteMapping("/{id}")
    public Mono<Void> delete(@PathVariable Long id) {
        return userService.deleteUser(id);
    }
}
```

`@RequestBody` can also be a publisher (`@RequestBody Mono<User> user`), so even reading the body is non-blocking.

### Output

`GET http://localhost:8080/api/users`

```json
[
  { "id": 1, "name": "Mohit", "email": "mohit@example.com" },
  { "id": 2, "name": "Rahul", "email": "rahul@example.com" }
]
```

A `Flux` serialises to a normal JSON array, so the client sees no difference. What changed is *how* the server produced it.

`GET http://localhost:8080/api/users/99` (doesn't exist)

```json
{
  "timestamp": "2026-09-24T10:14:02.115+00:00",
  "path": "/api/users/99",
  "status": 404,
  "error": "Not Found",
  "message": "User 99 not found"
}
```

---

## 10. Code: approach 2, functional endpoints

WebFlux's own style: routes are declared as beans instead of annotations. Useful when you want all routing visible in one place.

**Handler**

```java
@Component
public class UserHandler {

    private final UserService userService;

    public UserHandler(UserService userService) {
        this.userService = userService;
    }

    public Mono<ServerResponse> getAll(ServerRequest request) {
        return ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(userService.getAllUsers(), User.class);
    }

    public Mono<ServerResponse> getOne(ServerRequest request) {
        Long id = Long.valueOf(request.pathVariable("id"));
        return userService.getUserById(id)
                .flatMap(user -> ServerResponse.ok().bodyValue(user))
                .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(User.class)
                .flatMap(userService::createUser)
                .flatMap(saved -> ServerResponse.status(HttpStatus.CREATED).bodyValue(saved));
    }
}
```

**Router**

```java
@Configuration
public class UserRouter {

    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler handler) {
        return RouterFunctions
                .route(GET("/fn/users"), handler::getAll)
                .andRoute(GET("/fn/users/{id}"), handler::getOne)
                .andRoute(POST("/fn/users"), handler::create);
    }
}
```

Same output as the annotated version, just a different way of wiring it.

---

## 11. Code: streaming with SSE (where WebFlux really shines)

This is something Spring MVC can't do nearly as well. The client receives items **as they're produced**, not after the whole list is ready.

```java
@GetMapping(value = "/api/users/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<User> streamUsers() {
    return userService.getAllUsers()
            .delayElements(Duration.ofSeconds(1));   // simulate a slow feed
}
```

### Output

`curl http://localhost:8080/api/users/stream`

```
data:{"id":1,"name":"Mohit","email":"mohit@example.com"}

data:{"id":2,"name":"Rahul","email":"rahul@example.com"}

data:{"id":3,"name":"Priya","email":"priya@example.com"}
```

Each line appears **one second apart**, streamed over a single open connection. The server holds no thread while waiting between items.

A live counter, for testing:

```java
@GetMapping(value = "/api/ticks", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> ticks() {
    return Flux.interval(Duration.ofSeconds(1))
            .map(i -> "tick " + i);
}
```

```
data:tick 0
data:tick 1
data:tick 2
...
```

### Broadcasting to many clients (a hot stream with `Sinks`)

A tiny chat: anything POSTed is pushed to every connected SSE client.

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    // multicast = every subscriber gets the messages emitted after it subscribed
    private final Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();

    @PostMapping
    public Mono<Void> send(@RequestBody String message) {
        sink.tryEmitNext(message);
        return Mono.empty();
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream() {
        return sink.asFlux();
    }
}
```

Open `curl localhost:8080/chat/stream` in two terminals, then `curl -X POST -d "hello" localhost:8080/chat`. Both terminals print `data:hello`.

---

## 12. Code: WebClient (calling other services)

`WebClient` replaces `RestTemplate` in reactive code and is non-blocking end to end.

```java
@Service
public class ProductService {

    private final WebClient webClient;

    public ProductService(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://localhost:9090").build();
    }

    public Mono<Product> getProduct(Long id) {
        return webClient.get()
                .uri("/products/{id}", id)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError,
                        resp -> Mono.error(new RuntimeException("Product not found")))
                .bodyToMono(Product.class)
                .timeout(Duration.ofSeconds(3))
                .retry(2);
    }
}
```

### Parallel calls: the real payoff

Three independent calls that each take 100 ms:

```java
public Mono<Dashboard> buildDashboard(Long userId) {
    Mono<User>           user   = userService.getUserById(userId);
    Mono<List<Order>>    orders = orderClient.getOrders(userId);
    Mono<List<Product>>  recos  = recoClient.getRecommendations(userId);

    return Mono.zip(user, orders, recos)
            .map(t -> new Dashboard(t.getT1(), t.getT2(), t.getT3()));
}
```

![svg](<svgs/12-parallel-calls-with-zip.svg>)

```
Blocking version (sequential):  100 + 100 + 100 = ~300 ms
Reactive version (zip):         max(100,100,100) = ~105 ms
```

All three requests are in flight at the same time, on the same thread, with no thread pool involved.

---

## 13. Operators you will actually use

![svg](<svgs/08-operators-marble.svg>)

| Operator | What it does |
|---|---|
| `map(fn)` | Synchronous 1→1 transform |
| `flatMap(fn)` | Returns a Publisher: **use it for any async/DB/HTTP call**. Order not guaranteed |
| `concatMap(fn)` | Like `flatMap` but keeps order (one at a time, slower) |
| `flatMapSequential(fn)` | Runs concurrently like `flatMap`, but emits results in the original order |
| `filter(pred)` | Drops non-matching items |
| `zip(a, b)` | Pairs two streams; the sources run in parallel |
| `merge(a, b)` | Interleaves two streams as items arrive |
| `concat(a, b)` | Plays `a` fully, then `b` |
| `switchIfEmpty(alt)` | Fallback when the stream is empty |
| `defaultIfEmpty(val)` | Emits a default instead of nothing |
| `onErrorResume(fn)` | Recovers with another publisher |
| `onErrorReturn(val)` | Recovers with a fixed value |
| `retry(n)` / `retryWhen(...)` | Re-subscribes on error |
| `timeout(duration)` | Errors out if too slow |
| `doOnNext(fn)` | Side effect (logging); doesn't change the stream |
| `then()` / `then(mono)` | Ignores the items, continues when the source completes |
| `delayElements(d)` | Spaces items out in time |
| `take(n)` / `skip(n)` | Limits the stream |
| `collectList()` | `Flux<T>` → `Mono<List<T>>` (careful: buffers everything) |

**`map` vs `flatMap`: the most common confusion**

```java
// map: the lambda returns a plain value
Flux<String> names = users.map(User::getName);              // Flux<String>

// flatMap: the lambda returns a Publisher, and flatMap unwraps it
Flux<Order> orders = users.flatMap(u -> orderRepo.findByUserId(u.getId()));

// if you used map here you would get Flux<Flux<Order>>: nested and useless
```

**Limiting concurrency.** `flatMap` subscribes to inner publishers eagerly (up to 256 at a time by default). To avoid hammering a downstream service, cap it:

```java
userIds.flatMap(id -> webClient.get().uri("/users/{id}", id).retrieve().bodyToMono(User.class),
                8);   // at most 8 HTTP calls in flight
```

---

## 14. Error handling

In a reactive chain an error is **a signal** (`onError`) that travels downstream, skipping `map`/`flatMap` steps until an `onError*` operator handles it.

```java
public Mono<User> getUserSafely(Long id) {
    return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
            .onErrorResume(UserNotFoundException.class,
                    ex -> Mono.just(User.guest()))              // fallback value
            .doOnError(ex -> log.error("lookup failed for {}", id, ex))
            .timeout(Duration.ofSeconds(2))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));
}
```

Global handler:

```java
@RestControllerAdvice
public class GlobalErrorHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public Mono<ResponseEntity<String>> handleNotFound(UserNotFoundException ex) {
        return Mono.just(ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage()));
    }
}
```

> `try/catch` does **not** work around a reactive chain. The exception happens later, on another thread, after your `try` block has already exited. You must use the `onError*` operators.

> **Order matters.** Operators apply to everything *above* them. `retryWhen` placed after `timeout` retries timeouts too; `onErrorResume` placed before `timeout` won't catch the timeout.

---

## 15. Threading model and Schedulers

By default, Reactor **doesn't switch threads**: each step runs on whichever thread emitted the signal (often a Netty event-loop thread, or the R2DBC driver's thread). Schedulers let you choose explicitly:

![svg](<svgs/09-schedulers-publishon-subscribeon.svg>)

| Scheduler | Use for |
|---|---|
| `Schedulers.parallel()` | CPU-bound work (threads = CPU cores) |
| `Schedulers.boundedElastic()` | **Blocking** I/O you can't avoid (up to 10 × cores threads, with a task queue) |
| `Schedulers.single()` | One reusable thread |
| `Schedulers.immediate()` | The current thread |

- **`subscribeOn(scheduler)`** decides on which thread the **source** starts (subscription). Its position in the chain doesn't matter.
- **`publishOn(scheduler)`** switches the thread for the operators **after** it.

```java
Flux.range(1, 3)
    .map(i -> { log.info("map a"); return i; })      // caller thread
    .publishOn(Schedulers.parallel())
    .map(i -> { log.info("map b"); return i; })      // parallel-1
    .subscribe();
```

```
[reactor-http-nio-2] map a
[reactor-http-nio-2] map a
[parallel-1] map b
...
```

### The blocking-call trap

![svg](<svgs/10-blocking-call-trap.svg>)

If you're forced to call blocking code (a legacy JPA repository, a file read, a blocking SDK), **never** call it directly. Push it onto `boundedElastic`:

```java
public Mono<List<User>> legacyLookup() {
    return Mono.fromCallable(() -> jpaUserRepository.findAll())   // blocking
            .subscribeOn(Schedulers.boundedElastic())             // moved off the event loop
            .map(ArrayList::new);
}
```

**Detecting accidental blocking:** add BlockHound to your tests. It throws the moment blocking code runs on a non-blocking thread:

```xml
<dependency>
    <groupId>io.projectreactor.tools</groupId>
    <artifactId>blockhound</artifactId>
    <scope>test</scope>
</dependency>
```

```java
static { BlockHound.install(); }
```

```
reactor.blockhound.BlockingOperationError: Blocking call!
    java.net.SocketInputStream#socketRead0
```

---

## 16. ThreadLocal breaks: use Reactor Context (and WebFilter)

A request can hop between threads at every async step, so anything stored in a `ThreadLocal` (MDC log fields, `SecurityContextHolder`, `@Transactional` state in MVC) gets "left behind" on the old thread.

![svg](<svgs/13-threadlocal-vs-reactor-context.svg>)

**Reactor Context** is an immutable key/value map attached to the **subscription**, so it travels with the chain no matter which thread runs each step. It's written at the **bottom** of the chain (closest to the subscriber) and read anywhere above it.

**WebFilter** is WebFlux's version of a servlet `Filter`. Here's one that adds a request id to the Context and measures time:

```java
@Component
public class RequestIdFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String requestId = UUID.randomUUID().toString();
        long start = System.currentTimeMillis();
        exchange.getResponse().getHeaders().add("X-Request-Id", requestId);

        return chain.filter(exchange)                                    // ← the rest of the request
                .contextWrite(Context.of("requestId", requestId))        // visible to everything upstream
                .doFinally(signal -> log.info("{} {} took {} ms",
                        exchange.getRequest().getMethod(), exchange.getRequest().getPath(),
                        System.currentTimeMillis() - start));
    }
}
```

Reading it anywhere in the chain:

```java
public Mono<User> getUser(Long id) {
    return Mono.deferContextual(ctx -> {
        log.info("[{}] loading user {}", ctx.get("requestId"), id);
        return userRepository.findById(id);
    });
}
```

For MDC and tracing (trace ids in logs), Spring Boot 3.2+ can copy Context values into ThreadLocals automatically around each operator:

```properties
spring.reactor.context-propagation=auto
```

---

## 17. Data access with R2DBC

R2DBC (Reactive Relational Database Connectivity) is the non-blocking replacement for JDBC. Spring Data R2DBC gives you repositories similar to Spring Data JPA, **but it's not an ORM**:

| JPA / Hibernate | Spring Data R2DBC |
|---|---|
| `@Entity`, `ddl-auto` creates tables | `@Table`, you write `schema.sql` |
| `@OneToMany`, `@ManyToOne`, lazy loading | **No relationships**; load related rows with a second query and combine them |
| Dirty checking, L1/L2 cache | None: call `save()` explicitly |
| JPQL | Derived queries, `@Query` with native SQL, `DatabaseClient` / `R2dbcEntityTemplate` |

Loading a user with their orders (no `@OneToMany`, so you compose it yourself):

```java
public Mono<UserWithOrders> userWithOrders(Long userId) {
    return userRepository.findById(userId)
            .zipWith(orderRepository.findByUserId(userId).collectList())
            .map(t -> new UserWithOrders(t.getT1(), t.getT2()));
}
```

**Transactions still work with `@Transactional`.** Spring Boot auto-configures an `R2dbcTransactionManager` (a `ReactiveTransactionManager`), and the transaction is bound to the Reactor Context instead of a ThreadLocal:

```java
@Transactional
public Mono<Void> transfer(Long fromId, Long toId, BigDecimal amount) {
    return accountRepository.findById(fromId)
            .flatMap(from -> accountRepository.save(from.debit(amount)))
            .then(accountRepository.findById(toId))
            .flatMap(to -> accountRepository.save(to.credit(amount)))
            .then();   // an error anywhere → both updates are rolled back
}
```

(The method must **return** the publisher; the transaction spans the subscription.)

---

## 18. Security in WebFlux

Spring Security has a reactive flavor built on `WebFilter`s:

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        return http
                .csrf(ServerHttpSecurity.CsrfSpec::disable)                 // stateless API
                .authorizeExchange(ex -> ex
                        .pathMatchers("/public/**").permitAll()
                        .pathMatchers("/admin/**").hasRole("ADMIN")
                        .anyExchange().authenticated())
                .httpBasic(Customizer.withDefaults())
                .build();
    }

    @Bean
    public MapReactiveUserDetailsService users(PasswordEncoder encoder) {
        return new MapReactiveUserDetailsService(
                User.withUsername("mohit").password(encoder.encode("1234")).roles("ADMIN").build());
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Getting the current user: there's no `SecurityContextHolder` (ThreadLocal). Use `ReactiveSecurityContextHolder` (backed by the Reactor Context) or inject the principal:

```java
@GetMapping("/me")
public Mono<String> me(Mono<Principal> principal) {
    return principal.map(Principal::getName);
}

// anywhere in the chain
Mono<String> username = ReactiveSecurityContextHolder.getContext()
        .map(ctx -> ctx.getAuthentication().getName());
```

---

## 19. Testing

**WebTestClient:** for endpoints.

```java
@WebFluxTest(UserController.class)
class UserControllerTest {

    @Autowired WebTestClient webTestClient;
    @MockBean  UserService userService;

    @Test
    void shouldReturnAllUsers() {
        when(userService.getAllUsers())
                .thenReturn(Flux.just(new User(1L, "Mohit", "mohit@example.com")));

        webTestClient.get().uri("/api/users")
                .exchange()
                .expectStatus().isOk()
                .expectBodyList(User.class)
                .hasSize(1)
                .contains(new User(1L, "Mohit", "mohit@example.com"));
    }
}
```

(In Spring Boot 3.4+, `@MockBean` is deprecated in favor of `@MockitoBean`.)

**StepVerifier:** for the streams themselves.

```java
@Test
void shouldEmitTwoUsersThenComplete() {
    Flux<User> users = userService.getAllUsers();

    StepVerifier.create(users)
            .expectNextMatches(u -> u.getName().equals("Mohit"))
            .expectNextMatches(u -> u.getName().equals("Rahul"))
            .verifyComplete();
}

@Test
void shouldErrorWhenMissing() {
    StepVerifier.create(userService.getUserById(999L))
            .expectError(UserNotFoundException.class)
            .verify();
}
```

**Virtual time:** test `delayElements` / `interval` without actually waiting.

```java
@Test
void ticksWithoutWaiting() {
    StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(1)).take(3))
            .thenAwait(Duration.ofHours(3))
            .expectNext(0L, 1L, 2L)
            .verifyComplete();          // finishes instantly
}
```

```
Process finished with exit code 0
BUILD SUCCESS
```

---

## 20. Debugging reactive code

Stack traces show Reactor internals instead of where you *built* the chain. Tools that help:

| Tool | What it does | Cost |
|---|---|---|
| `.log()` | Prints every signal (`onSubscribe`, `request(n)`, `onNext`, `onError`) with the thread name | Noisy, dev only |
| `.checkpoint("loading user")` | Adds a marker to the stack trace if an error passes through this point | Cheap, can stay in production |
| `Hooks.onOperatorDebug()` | Records where every operator was assembled, for full traces | Expensive, dev only |
| `ReactorDebugAgent` (`reactor-tools`) | Same info as above via bytecode instrumentation | Low, production-friendly |

```java
userRepository.findById(id)
        .log("user-lookup")                // [user-lookup] | onSubscribe / request(unbounded) / onNext(User...) / onComplete()
        .checkpoint("after findById")
        .flatMap(this::enrich);
```

---

## 21. Benefits

1. **Scales with concurrency, not with threads.** A handful of threads serves tens of thousands of concurrent connections. Same machine, far more traffic.
2. **Much lower memory per connection.** No 1 MB stack per in-flight request, which matters a lot for gateways holding many idle sockets.
3. **Real streaming.** SSE, WebSocket, chunked responses: the client gets data as it's produced instead of waiting for the full result set.
4. **Backpressure across the wire.** A slow consumer can throttle a fast producer, so the system degrades gracefully instead of running out of memory.
5. **Parallel composition is easy.** `zip`, `merge`, `flatMap` make fan-out calls simple, and total latency becomes the slowest call rather than the sum.
6. **Built-in resilience operators.** `timeout`, `retryWhen`, `onErrorResume` are first-class, not bolted on.
7. **Efficient under slow clients.** Mobile clients on bad networks don't each pin a server thread.
8. **Choice of programming model.** Annotated controllers (familiar) or functional routing.

---

## 22. Disadvantages

1. **Steep learning curve.** Everyone on the team must think in streams. Code review and onboarding get slower.
2. **Debugging is genuinely painful.** Stack traces are fragmented across operators and threads and often show Reactor internals rather than your code. (`Hooks.onOperatorDebug()` helps but costs performance.)
3. **It only pays off if the *whole* chain is non-blocking.** One JDBC/JPA call and you have all the complexity with none of the scalability. The outcome is worse than plain MVC, because a single blocking call now stalls thousands of requests instead of one.
4. **The reactive ecosystem is smaller.** No JPA/Hibernate. R2DBC has no lazy loading, no dirty checking, no `@OneToMany` graph handling, so you write more SQL yourself. Many third-party SDKs are blocking-only.
5. **`ThreadLocal` breaks.** Anything built on it (MDC logging, `SecurityContextHolder`, transaction context) doesn't follow the request across threads. You must use the Reactor Context / Micrometer context propagation instead.
6. **No performance win for CPU-bound work.** Reactive helps with *waiting*, not with *computing*. If your endpoint does heavy calculation, WebFlux gains you nothing.
7. **It doesn't make a single request faster.** One user hitting one endpoint sees roughly the same latency. The benefit is throughput and stability under load.
8. **Testing and tooling are more involved.** `StepVerifier`, virtual time, BlockHound: extra machinery to learn.
9. **Harder to hire for and maintain.** Plain Spring MVC is what most Java developers know.

---

## 23. When to use / when not to

![svg](<svgs/14-when-to-use-webflux.svg>)

**Good fit**

- API gateways and BFF layers that fan out to many downstream services (Spring Cloud Gateway itself is built on WebFlux)
- Streaming endpoints: SSE, WebSocket, live feeds, chat, notifications
- High-concurrency, I/O-bound microservices with a reactive datastore (Mongo, Redis, Cassandra, R2DBC)
- IoT / telemetry ingestion: many long-lived connections, small messages
- Anywhere you're hitting connection-pool exhaustion rather than CPU limits

**Bad fit**

- Standard CRUD over a relational DB with JPA: the data layer is blocking, so there's nothing to gain
- CPU-bound work (image processing, report generation, heavy computation)
- Low or moderate traffic where the thread pool was never the bottleneck
- A team new to reactive with a tight deadline
- Legacy codebases full of blocking libraries you can't replace

### Important: consider virtual threads first

Since Java 21 / Spring Boot 3.2 you can set:

```properties
spring.threads.virtual.enabled=true
```

on **plain Spring MVC**. Blocking calls no longer pin an OS thread, so the thread-per-request model suddenly scales to very high concurrency, while you keep ordinary imperative code, readable stack traces, working debuggers and `ThreadLocal`.

For many applications this is now the better trade: most of the scalability benefit, almost none of the complexity cost.

| | Spring MVC + virtual threads | Spring WebFlux |
|---|---|---|
| Code style | Plain imperative, blocking calls are fine | Reactive chains (`Mono`/`Flux`) |
| JPA / JDBC | ✅ Works | ❌ Needs R2DBC / reactive drivers |
| Debugging, stack traces | Normal | Hard |
| `ThreadLocal`, MDC, `SecurityContextHolder` | ✅ Work | ❌ Need Reactor Context |
| Backpressure | ❌ No | ✅ Yes |
| Streaming (SSE, WebSocket) | Possible but limited | ✅ First-class |
| Memory per idle connection | Small (virtual thread) | Smallest |

**WebFlux still wins when you need:** true streaming semantics, backpressure propagated across the network, or a minimal memory footprint per idle connection. Virtual threads give you cheap threads; they do **not** give you backpressure.

---

## 24. Common pitfalls

| Pitfall | What happens | Fix |
|---|---|---|
| Not returning / not subscribing | Code silently never runs | `return` the publisher from the controller |
| `block()` inside a handler | Exception ("block() is not supported in thread reactor-http-nio") or a stalled event loop | Never call `block()` in WebFlux; compose with operators |
| Blocking JDBC / file I/O in a chain | Freezes every request on that thread | R2DBC, or `subscribeOn(Schedulers.boundedElastic())` |
| `map` where `flatMap` is needed | `Flux<Flux<T>>` / `Mono<Mono<T>>` | Use `flatMap` when the lambda returns a publisher |
| `Mono.just(expensiveCall())` | The call runs immediately, even if nobody subscribes | `Mono.fromCallable(...)` / `Mono.defer(...)` |
| Subscribing twice to a cold publisher | The query / HTTP call runs twice | Reuse the result, or `.cache()` |
| `try/catch` around a chain | Never catches anything | `onErrorResume` / `onErrorReturn` |
| `collectList()` on a huge/infinite Flux | Buffers everything → OOM | Stream it, or paginate |
| Unbounded `flatMap` to a downstream service | Hundreds of parallel calls hammer it | `flatMap(fn, concurrency)` |
| Using `ThreadLocal` / MDC | Values lost across threads | Reactor `Context`, context propagation |
| Expecting JPA features from R2DBC | No relations, no lazy loading, no `ddl-auto` | Write `schema.sql`, compose queries yourself |
| Mixing `spring-boot-starter-web` with webflux | App silently starts as Spring MVC | Remove the servlet web starter |

---

## 25. Summary

- WebFlux exists because **thread-per-request wastes threads on waiting**, and threads are expensive.
- It replaces "one thread per request" with "a few event-loop threads plus callbacks for I/O" (Netty, `reactor-http-nio-*`).
- `Mono` = 0–1 item, `Flux` = 0–N items. Both are **lazy**: nothing runs until something subscribes.
- Backpressure (`request(n)`) is what makes Reactive Streams more than just callbacks.
- The whole chain must be non-blocking, top to bottom (R2DBC, WebClient, reactive drivers), or the benefit disappears.
- `ThreadLocal` doesn't follow the request, so use Reactor Context (`contextWrite` / `deferContextual`), `ReactiveSecurityContextHolder`, and reactive transactions.
- It buys **throughput and streaming**, not lower single-request latency.
- Cost: complexity, debugging pain, a smaller ecosystem, no JPA.
- In 2026, evaluate **virtual threads on Spring MVC** before reaching for WebFlux, unless you specifically need streaming or backpressure.

---

## 26. Interview Questions

**Q1. What is Spring WebFlux and how is it different from Spring MVC?**
A reactive, non-blocking web framework built on Project Reactor and (by default) Netty. MVC uses one thread per request that blocks during I/O. WebFlux uses a few event-loop threads that never wait: handlers return `Mono`/`Flux`, and the work continues in callbacks when I/O completes.

**Q2. What problem does WebFlux solve?**
Thread exhaustion under high concurrency with slow I/O (the C10K problem). With blocking I/O, throughput is capped by the thread count. Non-blocking I/O lets a few threads handle many thousands of concurrent connections.

**Q3. Does WebFlux make a single request faster?**
No. Single-request latency is about the same. It improves throughput and stability under load, and makes parallel fan-out (`zip`) easy.

**Q4. What is Reactive Streams? Name the four interfaces.**
A specification for asynchronous stream processing with non-blocking backpressure: `Publisher`, `Subscriber`, `Subscription`, `Processor`.

**Q5. What is backpressure? How do you handle a producer that can't slow down?**
The consumer signals demand with `request(n)`, and the producer never sends more than requested. For uncontrollable sources: `limitRate`, `onBackpressureBuffer`, `onBackpressureDrop`, `onBackpressureLatest`.

**Q6. `Mono` vs `Flux`?**
`Mono` emits 0 or 1 item, then completes or errors. `Flux` emits 0 to N items (possibly infinite).

**Q7. What does "nothing happens until you subscribe" mean? Who subscribes in a controller?**
Building a chain only describes the work. It executes on `subscribe()`. In WebFlux controllers, the framework subscribes to the returned publisher, so you must return it.

**Q8. `map` vs `flatMap` vs `concatMap`?**
`map`: sync transform returning a value. `flatMap`: the function returns a publisher; inner publishers run concurrently and results may interleave. `concatMap`: like `flatMap` but one at a time, preserving order. `flatMapSequential` runs concurrently but emits in order.

**Q9. Cold vs hot publishers?**
Cold publishers restart for every subscriber (a DB query runs per subscribe). Hot publishers emit regardless of subscribers, and late subscribers miss earlier items (`Sinks.many().multicast()`, `share()`).

**Q10. `Mono.just()` vs `Mono.fromCallable()` vs `Mono.defer()`?**
`just` captures an already-computed value (eager). `fromCallable` runs the callable lazily at subscription. `defer` creates a fresh publisher lazily for each subscriber.

**Q11. `subscribeOn` vs `publishOn`?**
`subscribeOn` chooses the thread where the source starts (position doesn't matter). `publishOn` switches the thread for the operators after it.

**Q12. How do you call blocking code (JDBC, a legacy SDK) from WebFlux?**
Ideally, don't: use a reactive driver. If you must, wrap it: `Mono.fromCallable(blockingCall).subscribeOn(Schedulers.boundedElastic())`. Detect mistakes with BlockHound.

**Q13. Why can't you call `block()` in a WebFlux handler?**
It parks an event-loop thread, the exact thing the model forbids. Reactor throws `IllegalStateException` when `block()` is called on a non-blocking thread, and even where it's allowed it kills scalability.

**Q14. How do you handle errors in a reactive chain? Why doesn't try/catch work?**
Errors are `onError` signals that happen asynchronously, after the `try` block has returned. Use `onErrorResume`, `onErrorReturn`, `onErrorMap`, `retryWhen`, `timeout`, and `@ExceptionHandler` methods returning `Mono`.

**Q15. Why doesn't `ThreadLocal` work in WebFlux? What replaces it?**
A request can run on different threads at each async step. Use the Reactor Context (`contextWrite` / `deferContextual`), `ReactiveSecurityContextHolder` for security, and `spring.reactor.context-propagation=auto` to bridge MDC/tracing.

**Q16. Can you use JPA/Hibernate with WebFlux? What about transactions?**
JPA is blocking, so it defeats the purpose (or must be wrapped with `boundedElastic`). Use R2DBC / Spring Data R2DBC instead: no relationships or lazy loading, but `@Transactional` works via `R2dbcTransactionManager`, with the transaction bound to the Reactor Context.

**Q17. What replaces `RestTemplate`, `Filter`, `HandlerInterceptor`, `DispatcherServlet` and `MockMvc` in WebFlux?**
`WebClient`, `WebFilter`, `WebFilter` (no interceptor equivalent), `DispatcherHandler`, and `WebTestClient` (plus `StepVerifier` for streams).

**Q18. Annotated controllers vs functional endpoints?**
Same engine. Annotated controllers look like MVC. Functional endpoints define routes as `RouterFunction` beans and handlers as `HandlerFunction`s returning `Mono<ServerResponse>`, which keeps routing explicit and in one place.

**Q19. How do you test reactive code?**
`WebTestClient` for endpoints, `StepVerifier` for publishers (`expectNext`, `expectError`, `verifyComplete`), virtual time for delays/intervals, BlockHound for blocking calls.

**Q20. WebFlux or Spring MVC with virtual threads?**
Virtual threads give MVC cheap blocking threads, so you keep simple code, JPA, ThreadLocal and normal stack traces. That's usually the better default. Choose WebFlux for streaming (SSE/WebSocket), end-to-end backpressure, gateways with huge numbers of idle connections, or an already fully reactive stack.
