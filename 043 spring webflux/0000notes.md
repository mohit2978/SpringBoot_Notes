# Spring WebFlux

## What is it

Spring WebFlux is the **reactive, non-blocking web framework** that Spring added in Spring 5 (Spring Boot 2). It sits next to Spring MVC as a completely separate stack — not a replacement, not an upgrade. You pick one per application.

Three things define it:

1. **Non-blocking** — a thread is never parked waiting for I/O.
2. **Asynchronous** — you return a *promise of data* (`Mono`/`Flux`), not the data itself.
3. **Backpressure-aware** — the consumer tells the producer how much it can take.

It is built on **Project Reactor**, which is an implementation of the **Reactive Streams** specification. By default it runs on **Netty** instead of Tomcat.

```
Spring MVC     →  1 request = 1 thread, thread blocks during I/O
Spring WebFlux →  1 thread   = many requests, thread never blocks
```

---

## Why is it needed — the problem it solves

### The thread-per-request model runs out of threads

In Spring MVC, every incoming request grabs a thread from the Tomcat pool (default **200**) and keeps it until the response is written. If the request calls a database or another microservice, that thread **sits there doing nothing** while it waits.

The CPU is idle. The thread is not. It is parked inside a blocking socket read, and no other request can use it.

Do the maths:

```
200 threads in the pool
each request waits 100 ms on the DB

throughput ceiling = 200 / 0.1s = ~2000 requests/sec   ← hard wall
request 2001 queues up and eventually times out
```

Each thread also costs roughly **1 MB of stack memory**, so 200 idle threads = 200 MB of RAM burnt on waiting. Raise the pool to 2000 threads and you get 2 GB of stacks plus a context-switching storm that makes things *slower*, not faster.

This is the classic **C10K problem** — how do you serve 10,000 concurrent connections on one box.

### The fix: stop tying a thread to a request

WebFlux uses a small pool of **event-loop threads** (default = number of CPU cores). When a handler issues an I/O call, the thread registers a callback and **immediately moves on to the next request**. When the database answers, the event loop picks the work back up.

Same hardware, same 100 ms database — but now a handful of threads serve tens of thousands of concurrent requests, because no thread is ever idle-waiting.

![thread per request vs event loop](<svgs/01-thread-per-request-vs-event-loop.svg>)

**The catch, stated up front:** this only works if *nothing* in the chain blocks. One JDBC call inside a WebFlux handler freezes every request sharing that event-loop thread. This is the single biggest source of WebFlux bugs in production.

---

## Reactive Streams — the specification underneath

Reactive Streams is a tiny spec (4 interfaces) that Reactor, RxJava, Akka Streams and Java 9's `java.util.concurrent.Flow` all implement.

| Interface | Role |
|---|---|
| `Publisher<T>` | The source. Has one method: `subscribe(Subscriber)` |
| `Subscriber<T>` | The consumer. Has `onSubscribe`, `onNext`, `onError`, `onComplete` |
| `Subscription` | The control handle. Has `request(long n)` and `cancel()` |
| `Processor<T,R>` | Both a Publisher and a Subscriber (used to build operators) |

The handshake between them is what makes backpressure possible:

![reactive streams protocol](<svgs/02-reactive-streams-protocol.svg>)

### Backpressure

Backpressure is the consumer saying *"send me only 2 more, that is all I can handle"*. Without it, a fast producer and a slow consumer means an unbounded queue and eventually `OutOfMemoryError`.

This is the key difference from plain callbacks or Java 8 Streams — the demand flows **upstream**.

![backpressure](<svgs/03-backpressure.svg>)

---

## Mono and Flux

Project Reactor gives you exactly two publisher types. You will use these everywhere.

| Type | Emits | Use for |
|---|---|---|
| `Mono<T>` | 0 or 1 item, then completes | one user, one save, one HTTP call, `void` |
| `Flux<T>` | 0 to N items (can be infinite) | lists, streams, SSE, paged data |

![mono and flux marble diagram](<svgs/04-mono-flux-marble.svg>)

### Nothing happens until you subscribe

This trips up everyone once. A `Mono`/`Flux` is a **recipe, not a running task**.

```java
Mono<User> mono = userRepository.save(user);   // NOTHING has happened yet
// no insert has been issued, the DB has not been touched
```

The chain only runs when something subscribes. In a controller, **Spring subscribes for you** when you return the publisher — which is why you must `return` it, never ignore it.

```java
// WRONG — silently does nothing
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

---

## The two stacks side by side

![webflux vs mvc stack](<svgs/05-webflux-vs-mvc-stack.svg>)

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Starter | `spring-boot-starter-web` | `spring-boot-starter-webflux` |
| Default server | Tomcat | Netty |
| Threads | ~200, one per request | ~cores, shared by all |
| Return type | `User`, `List<User>` | `Mono<User>`, `Flux<User>` |
| Data access | JPA / Hibernate / JDBC (blocking) | R2DBC / reactive Mongo, Redis, Cassandra |
| HTTP client | `RestClient` / `RestTemplate` | `WebClient` |
| Testing | `MockMvc` | `WebTestClient`, `StepVerifier` |
| Debugging | normal stack traces | fragmented stack traces |
| Learning curve | low | steep |

> You cannot put both `spring-boot-starter-web` and `spring-boot-starter-webflux` on the classpath and expect WebFlux — Spring Boot detects the servlet stack and starts Spring MVC. Remove the web starter.

---

## Setup

**pom.xml**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- reactive database driver — NOT spring-boot-starter-data-jpa -->
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

# see which thread everything runs on — very useful while learning
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{20} - %msg%n
```

Startup log confirms the stack:

```
Netty started on port 8080
Started DemoApplication in 1.412 seconds
```

(Spring MVC would say `Tomcat started on port(s): 8080`.)

---

## Code — approach 1: annotated controllers

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

**Repository** — extend `ReactiveCrudRepository`, not `JpaRepository`

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

### Output

`GET http://localhost:8080/api/users`

```json
[
  { "id": 1, "name": "Mohit", "email": "mohit@example.com" },
  { "id": 2, "name": "Rahul", "email": "rahul@example.com" }
]
```

A `Flux` serialises to a normal JSON array — the client sees no difference. What changed is *how* the server produced it.

`GET http://localhost:8080/api/users/99` (does not exist)

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

## Code — approach 2: functional endpoints

WebFlux's own style — routes are declared as beans instead of annotations. Useful when you want routing visible in one place.

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

Same output as the annotated version — just a different way of wiring it.

---

## Code — streaming with SSE (where WebFlux really shines)

This is something Spring MVC genuinely cannot do well. The client receives items **as they are produced**, not after the whole list is ready.

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

---

## Code — WebClient (calling other services)

`WebClient` replaces `RestTemplate` in reactive code. It is non-blocking end to end.

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

### Parallel calls — the real payoff

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

```
Blocking version (sequential):  100 + 100 + 100 = ~300 ms
Reactive version (zip):         max(100,100,100) = ~105 ms
```

All three requests are in flight at the same time, on the same thread, with no thread pool involved.

---

## Operators you will actually use

![operators marble diagram](<svgs/06-operators-marble.svg>)

| Operator | What it does |
|---|---|
| `map(fn)` | synchronous 1→1 transform |
| `flatMap(fn)` | returns a Publisher — **use for any async/DB/HTTP call**; order not guaranteed |
| `concatMap(fn)` | like `flatMap` but preserves order (slower, sequential) |
| `filter(pred)` | drops non-matching items |
| `zip(a, b)` | pairs two streams, runs them in parallel |
| `merge(a, b)` | interleaves two streams as items arrive |
| `concat(a, b)` | plays `a` fully, then `b` |
| `switchIfEmpty(alt)` | fallback when the stream is empty |
| `defaultIfEmpty(val)` | emit a default instead of nothing |
| `onErrorResume(fn)` | recover with another publisher |
| `onErrorReturn(val)` | recover with a fixed value |
| `retry(n)` / `retryWhen(...)` | re-subscribe on error |
| `timeout(duration)` | error out if too slow |
| `doOnNext(fn)` | side effect (logging) — does not change the stream |
| `delayElements(d)` | space items out in time |
| `take(n)` / `skip(n)` | limit the stream |
| `collectList()` | `Flux<T>` → `Mono<List<T>>` (careful: buffers everything) |

**`map` vs `flatMap` — the most common confusion:**

```java
// map: the lambda returns a plain value
Flux<String> names = users.map(User::getName);              // Flux<String>

// flatMap: the lambda returns a Publisher — flatMap unwraps it
Flux<Order> orders = users.flatMap(u -> orderRepo.findByUserId(u.getId()));

// if you used map here you would get Flux<Flux<Order>> — nested and useless
```

---

## Error handling

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

> `try/catch` does **not** work around a reactive chain — the exception happens later, on another thread, after your `try` block has already exited. You must use the `onError*` operators.

---

## Threading model and Schedulers

![schedulers and blocking calls](<svgs/08-schedulers-blocking-call.svg>)

If you are forced to call blocking code (a legacy JPA repository, a file read, a blocking SDK), **never** call it directly. Push it onto `boundedElastic`:

```java
public Mono<List<User>> legacyLookup() {
    return Mono.fromCallable(() -> jpaUserRepository.findAll())   // blocking
            .subscribeOn(Schedulers.boundedElastic())             // moved off the event loop
            .map(ArrayList::new);
}
```

**Detecting accidental blocking** — add BlockHound in tests and it throws the moment blocking code runs on a non-blocking thread:

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

## Testing

**WebTestClient** — for endpoints:

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

**StepVerifier** — for the streams themselves:

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

```
Process finished with exit code 0
BUILD SUCCESS
```

---

## Benefits

1. **Scales with concurrency, not with threads.** A handful of threads serves tens of thousands of concurrent connections. Same box, far more traffic.
2. **Much lower memory per connection.** No 1 MB stack per in-flight request — big deal for gateways holding many idle sockets.
3. **Real streaming.** SSE, WebSocket, chunked responses — the client gets data as it is produced instead of waiting for the full result set.
4. **Backpressure across the wire.** A slow consumer can throttle a fast producer; the system degrades gracefully instead of running out of memory.
5. **Parallel composition is easy.** `zip`, `merge`, `flatMap` make fan-out calls trivial, and total latency becomes the slowest call rather than the sum.
6. **Built-in resilience operators.** `timeout`, `retryWhen`, `onErrorResume` are first-class, not bolt-on.
7. **Efficient under slow clients.** Mobile clients on bad networks don't each pin a server thread.
8. **Choice of programming model.** Annotated controllers (familiar) or functional routing.

---

## Disadvantages

1. **Steep learning curve.** Everyone on the team must think in streams. Code review and onboarding get slower.
2. **Debugging is genuinely painful.** Stack traces are fragmented across operators and threads and often show Reactor internals rather than your code. (`Hooks.onOperatorDebug()` helps but costs performance.)
3. **It only pays off if the *whole* chain is non-blocking.** One JDBC/JPA call and you have all the complexity with none of the scalability — and a worse outcome than plain MVC, because now a single blocking call stalls thousands of requests instead of one.
4. **The reactive ecosystem is smaller.** No JPA/Hibernate. R2DBC has no lazy loading, no dirty checking, no `@OneToMany` graph handling — you write more SQL yourself. Many third-party SDKs are blocking-only.
5. **`ThreadLocal` breaks.** Anything built on it — MDC logging, `SecurityContextHolder`, transaction context — does not follow the request across threads. You must use Reactor Context / Micrometer context propagation instead.
6. **No performance win for CPU-bound work.** Reactive helps with *waiting*, not with *computing*. If your endpoint is doing heavy calculation, WebFlux gains you nothing.
7. **It does not make a single request faster.** One user hitting one endpoint sees roughly the same latency. The benefit is throughput and stability under load.
8. **Testing and tooling are more involved.** `StepVerifier`, virtual time, BlockHound — extra machinery to learn.
9. **Harder to hire for and maintain.** Plain Spring MVC is what most Java developers know.

---

## When to use / when not to

![decision flowchart](<svgs/07-when-to-use-decision.svg>)

**Good fit**

- API gateways and BFF layers that fan out to many downstream services
- Streaming endpoints — SSE, WebSocket, live feeds, chat, notifications
- High-concurrency, I/O-bound microservices with a reactive datastore (Mongo, Redis, Cassandra, R2DBC)
- IoT / telemetry ingestion — many long-lived connections, small messages
- Anywhere you're hitting connection-pool exhaustion rather than CPU limits

**Bad fit**

- Standard CRUD over a relational DB with JPA — the data layer is blocking, so there is nothing to gain
- CPU-bound work (image processing, report generation, heavy computation)
- Low or moderate traffic where the thread pool was never the bottleneck
- A team new to reactive with a tight deadline
- Legacy codebases full of blocking libraries you cannot replace

### Important: consider virtual threads first

Since Java 21 / Spring Boot 3.2 you can set:

```properties
spring.threads.virtual.enabled=true
```

on **plain Spring MVC**. Blocking calls no longer pin an OS thread, so the thread-per-request model suddenly scales to very high concurrency — while you keep ordinary imperative code, readable stack traces, working debuggers and `ThreadLocal`.

For a lot of applications this is now the better trade: most of the scalability benefit, almost none of the complexity cost.

**WebFlux still wins when you need:** true streaming semantics, backpressure propagated across the network, or a minimal memory footprint per idle connection. Virtual threads give you cheap threads — they do **not** give you backpressure.

---

## Common pitfalls

| Pitfall | What happens | Fix |
|---|---|---|
| Not returning / not subscribing | Code silently never runs | `return` the publisher from the controller |
| `block()` inside a handler | Deadlock or stalled event loop | Never call `block()` in WebFlux; compose with operators |
| Blocking JDBC / file I/O in a chain | Freezes every request on that thread | R2DBC, or `subscribeOn(Schedulers.boundedElastic())` |
| `map` where `flatMap` is needed | `Flux<Flux<T>>` / `Mono<Mono<T>>` | Use `flatMap` when the lambda returns a publisher |
| `try/catch` around a chain | Never catches anything | `onErrorResume` / `onErrorReturn` |
| `collectList()` on a huge/infinite Flux | Buffers everything → OOM | Stream it, or paginate |
| Using `ThreadLocal` / MDC | Values lost across threads | Reactor `Context`, context propagation |
| Mixing `spring-boot-starter-web` with webflux | App silently starts as Spring MVC | Remove the servlet web starter |

---

## Summary

- WebFlux exists because **thread-per-request wastes threads on waiting**, and threads are expensive.
- It replaces "one thread per request" with "a few event-loop threads, callbacks for I/O".
- `Mono` = 0–1 item, `Flux` = 0–N items; both are **lazy** — nothing runs until subscribed.
- Backpressure (`request(n)`) is what makes Reactive Streams more than just callbacks.
- The whole chain must be non-blocking, top to bottom, or the benefit disappears.
- It buys **throughput and streaming**, not lower single-request latency.
- Cost: complexity, debugging pain, a smaller ecosystem, no JPA.
- In 2026, evaluate **virtual threads on Spring MVC** before reaching for WebFlux — unless you specifically need streaming or backpressure.
