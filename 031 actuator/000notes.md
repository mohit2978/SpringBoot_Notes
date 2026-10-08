

## What is Actuator

Provides production-ready endpoints to monitor and manage the Spring Boot application.

## Project Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

`application.properties`:

```properties
spring.application.name=order-service

# by-default path is /actuator this is optional
management.endpoints.web.base-path=/manage

# Expose all actuator endpoints, by-default '/actuator/health' & '/actuator/info' endpoint is exposed
# '*' expose all the endpoints
# use comma separated like: health, info, metrics, loggers etc. to expose selected endpoints
management.endpoints.web.exposure.include=*
```

## GET: /health

Provides health status of the application: `UP`, `DOWN`, `OUT_OF_SERVICE`, `UNKNOWN`.

```json
{
  "status": "UP"
}
```


`/health` can be extended to add additional checks beyond just application server status, e.g. DB health, Cache status.

Now we extended to Db health cheack!!

**DB health check:**

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean isDBUp = checkDBConnection();
        return isDBUp ? Health.up().withDetail("DB", "Available").build()
                      : Health.down().withDetail("DB", "Not-Available").build();
    }

    private boolean checkDBConnection() {
        // check DB is up or not
        return true;
    }
}
```

Cache helath check!!

**Cache health check:**

```java
@Component
public class CacheHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean isCacheUp = checkCacheStatus();
        return isCacheUp ? Health.up().withDetail("Cache", "Available").build()
                         : Health.down().withDetail("Cache", "Not-Available").build();
    }

    private boolean checkCacheStatus() {
        // check cache status
        return false;
    }
}
```

Response now shows an **aggregate status** with no other components detail 

```json
{
  "status": "UP"
}
```

By default, `/health` shows only the overall status. To see per-component details, add:

```properties
# by-default is 'never'
management.endpoint.health.show-details=always
```

Response now shows an **aggregate status** with all other components detail too — if 1 component is down, overall status is down:

```json
{
  "status": "DOWN",
  "components": {
    "cache": {
      "status": "DOWN",
      "details": { "Cache": "Not-Available" }
    },
    "database": {
      "status": "UP",
      "details": { "DB": "Available" }
    }
  }
}
```

## GET: /metrics and /metrics/{metric name}

`/metrics` lists all available metrics endpoints. Hitting a specific one, e.g. `/metrics/executor.pool.core`, returns details:

```json
{
  "name": "executor.pool.core",
  "description": "The core number of threads for the pool",
  "baseUnit": "threads",
  "measurements": [
    { "statistic": "VALUE", "value": 8 }
  ],
  "availableTags": [
    { "tag": "name", "values": ["applicationTaskExecutor"] }
  ]
}
```

### Important metrics

**JVM Memory Metrics**
| Metric | Represents | Example |
|---|---|---|
| `jvm.memory.used` | Memory currently in use by JVM | `{"statistic":"VALUE","value":99478616}` |
| `jvm.memory.max` | Max memory JVM can use | `{"statistic":"VALUE","value":10989076477}` |

**Garbage Collection Metrics** — `jvm.gc.pause`: time spent in GC.
- `COUNT`: total no. of GC events occurred
- `TOTAL_TIME`: total time spent in GC (usually seconds)
- `MAX`: longest single GC pause observed

```json
[
  { "statistic": "COUNT", "value": 12 },
  { "statistic": "TOTAL_TIME", "value": 2.305 },
  { "statistic": "MAX", "value": 0.9 }
]
```

**Threads**
| Metric | Represents | Example |
|---|---|---|
| `jvm.threads.live` | Number of live threads | `{"statistic":"VALUE","value":22}` |
| `jvm.threads.peak` | Peak live thread count since JVM started | `{"statistic":"VALUE","value":50}` |

**System Metrics**
- `system.cpu.usage`: CPU used by JVM (range 0.0 - 1.0). Value 0.10 -> 10%

**HTTP Server / Requests** — `http.server.requests`
- `COUNT`: total no. of HTTP requests received
- `TOTAL_TIME`: total time spent handling all requests (seconds)
- `MAX`: longest time taken to handle a single request

```json
{
  "measurements": [
    { "statistic": "COUNT", "value": 152 },
    { "statistic": "TOTAL_TIME", "value": 23.45 },
    { "statistic": "MAX", "value": 0.89 }
  ],
  "availableTags": [
    { "tag": "method", "values": ["GET", "POST"] },
    { "tag": "status", "values": ["200", "404", "500"] }
  ]
}
```

**Database / JDBC Metrics**
| Metric | Represents | Value |
|---|---|---|
| `jdbc.connections.active` | Connections currently in use | 3 |
| `jdbc.connections.idle` | Idle connections in pool | 7 |
| `jdbc.connections.max` | Max connections allowed | 10 |

## GET: /threaddump

Helps diagnose deadlocks or thread leaks. Shows:
- Which threads are active, blocked, or waiting
- Stack trace for each thread (what code it is executing)
- Thread name, id, priority, etc.

```json
[
  {
    "threadName": "main",
    "threadId": 1,
    "blockedTime": -1,
    "blockedCount": 0,
    "waitedTime": -1,
    "waitedCount": 0,
    "threadState": "RUNNABLE",
    "stackTrace": [
      "com.concepts.MyService.methodName(MyService.java:142)",
      "com.concepts.ActuatorApp.main(ActuatorApp.java:10)"
    ]
  },
  {
    "threadName": "thread2",
    "threadId": 2,
    "blockedTime": -2,
    "blockedCount": 0,
    "waitedTime": -1,
    "waitedCount": 0,
    "threadState": "WAITING",
    "stackTrace": [
      "com.concepts.ClassName.methodName(ClassName.java:12)",
      "com.concepts.ActuatorApp.main(ActuatorApp.java:10)"
    ]
  }
]
```

## Other Available Endpoints

Official docs: https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/heapdump` | GET | Downloads JVM heap as `.hprof` file |
| `/mappings` | GET | Lists all Spring MVC request mappings |
| `/beans` | GET | Displays complete list of all Spring beans in the application |
| `/configprops` | GET | Lists all `@ConfigurationProperties` beans |
| `/loggers` | GET | Lists all loggers and their current levels |
| `/shutdown` | POST | Lets the application be gracefully shutdown |
| `/env` | GET | Shows environment properties |
| `/actuator/env/{property}` | GET | Shows a specific environment property |

By default, access to all endpoints except `/shutdown` and `/heapdump` is unrestricted. These two are critical:
- `/shutdown` can stop the application
- `/heapdump` can expose sensitive info (tokens, passwords, etc.)

They're restricted by default; to unrestrict (accepting the risk):

```properties
# Make it require authentication
management.endpoint.shutdown.access=unrestricted
management.endpoint.heapdump.access=unrestricted
```

## Security

Actuator endpoints can be secured with Spring Security (see separate Spring Security notes, 9 parts).

`pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

`application.properties`:

```properties
spring.security.user.name=user
spring.security.user.password=pass
spring.security.user.roles=ADMIN
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/manage/health", "/manage/info").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults()); // basic auth for simplicity
        return http.build();
    }
}
```

- Accessing `/env` without authentication → `401 Unauthorized`
- Accessing after successful authentication → data returned


whichever urls you want to secure just add it here in config
## Custom Actuator Endpoint

Annotate a class with `@Endpoint(id = "custom-endpoint-name")`. The `id` forms the URL path: `/actuator/{id}`.

Supported operation types:
- `@ReadOperation` — equivalent to HTTP GET
- `@WriteOperation` — equivalent to HTTP POST
- `@DeleteOperation` — equivalent to HTTP DELETE

Return type can be any serializable object: Map, List, String, POJO, primitive, etc.

```java
// our endpoint becomes '/actuator/my-custom-stats'
@Component
@Endpoint(id = "my-custom-stats")
public class MyCustomStatsEndpoint {

    @ReadOperation
    public String readAll() {
        return "Hello, Spring Boot!";
    }

    @ReadOperation
    public String read(@Selector String name, @Selector String message) {
        return "Hello: " + name + " msg for you is: " + message;
    }
    // path: /my-custom-stats/{name}/{message} -- @Selector args follow URL path sequence

    @WriteOperation
    public String refresh() {
        // simulate say cache refresh
        return "refreshed";
    }

    @DeleteOperation
    public String remove(@Selector String key) {
        return "reset done for key: " + key;
    }
}
```

Notes:
- `@Selector` parameters follow the sequence they appear in the URL path.
- For POST and DELETE operations, authentication is required.

To expose the custom endpoint, add it to the exposure include list:

```properties
management.endpoints.web.exposure.include=my-custom-stats,health,info
```

Example calls:
- `GET /manage/my-custom-stats` → `Hello, Spring Boot!`
- `GET /manage/my-custom-stats/shrayansh/how are you` → `Hello: shrayansh msg for you is: how are you`
- `POST /manage/my-custom-stats` (basic auth) → `refreshed`
- `DELETE /manage/my-custom-stats/myKey` (basic auth) → `reset done for key: myKey`

## Pushing Metrics to Datadog

Metrics can be pushed to monitoring platforms like Datadog, Prometheus, CloudWatch, etc.

`application.properties`:

```properties
management.datadog.metrics.export.apiKey=<your-datadog-api-key>
# when true, it will try to push the metrics
management.datadog.metrics.export.enabled=true
# every 5s it will push the metrics to datadog
management.datadog.metrics.export.step=5s
```

`pom.xml`:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-datadog</artifactId>
</dependency>
```

All Spring Boot Actuator metrics are automatically pushed to Datadog once configured. In Datadog, choose the metric to monitor, e.g. `http.server.requests.count`, under Metrics > Summary.

---

## Production Issues: Which Actuator Endpoint to Check and Why?

In production incidents, raw application logs can be overwhelming or insufficient. Spring Boot Actuator endpoints provide direct introspection into JVM internals, thread states, connection pools, and runtime configurations.

### Quick Reference Matrix

| Production Issue / Symptom | Actuator Endpoint / Metric | Why Check It? |
|---|---|---|
| **High CPU Usage (100% CPU spike / freeze)** | `/threaddump`<br>`/metrics/system.cpu.usage`<br>`/metrics/process.cpu.usage` | Identifies which exact threads are in `RUNNABLE` state and the specific lines of code spinning (infinite loops, expensive serialization, regex catastrophic backtracking). |
| **Out of Memory (`OutOfMemoryError` / OOM)** | `/heapdump`<br>`/metrics/jvm.memory.used`<br>`/metrics/jvm.gc.pause` | Downloads memory snapshot (`.hprof`) to find memory leaks (dominator tree) and verifies if JVM is stuck in Stop-The-World Full GC cycles. |
| **Database Pool Starvation (`HikariPool - Connection timeout`)** | `/metrics/hikaricp.connections.*`<br>`/metrics/jdbc.connections.*`<br>`/health` | Shows active, idle, and pending threads waiting for connections. Pinpoints connection leaks, missing indexes, or unclosed transactions. |
| **High API Latency & Slow Response Times** | `/metrics/http.server.requests`<br>`/httpexchanges` (or `/httptrace`) | Allows filtering by endpoint (`uri`), HTTP method, and status code to pinpoint which API is slow, its max latency, and request volume. |
| **Downstream Dependency Outage (DB, Redis, MQ down)** | `/health` (with `show-details=always`) | Instantly pinpoints which external system (MySQL, Redis, RabbitMQ, Kafka) is unhealthy without grepping logs. |
| **Wrong Configuration / Profile Mismatch on Deploy** | `/env`<br>`/configprops` | Verifies active Spring profiles, precedence of environment variables vs properties files, and actual injected configuration values. |
| **Need Live Debugging Without Restarting Application** | `/loggers` (`GET` & `POST`) | Dynamically change log level (e.g., from `INFO` to `DEBUG` or `TRACE`) at runtime for a specific class/package without stopping or restarting the pod. |
| **404 Not Found / Missing Request Mapping** | `/mappings`<br>`/beans` | Shows all registered URL routes, controller methods, and HTTP verbs to diagnose routing bugs or missing beans. |
| **Worker / Async Thread Pool Exhaustion** | `/metrics/executor.pool.*`<br>`/metrics/tomcat.threads.*` | Shows busy vs idle Tomcat worker threads or queued `@Async` tasks to identify thread pool exhaustion and blocked workers. |
| **Kubernetes Pod CrashLoopBackOff or Unhealthy** | `/health/liveness`<br>`/health/readiness` | Distinguishes whether the JVM is deadlocked (liveness fails → pod restart) or temporarily busy/loading (readiness fails → stop routing traffic). |
| **Disk Space Exhaustion (`No space left on device`)** | `/health` (`diskSpace` component) | Shows free vs total disk threshold; triggers early warnings before writes to logs or temp files crash the process. |

---

### Detailed Production Scenarios & Triage Guide

---

#### 1. High CPU Usage / Application Unresponsive (100% CPU Spike)

- **Actuator Endpoints:**
  - `GET /manage/threaddump`
  - `GET /manage/metrics/process.cpu.usage`
  - `GET /manage/metrics/system.cpu.usage`
- **Why Check This?**
  - High CPU is almost always caused by either **busy-waiting loops**, **infinite loops**, **catastrophic regex backtracking**, or **intensive GC thrashing**.
  - `/threaddump` captures a snapshot of all active threads and their exact execution stack trace at that millisecond.
- **How to Investigate:**
  1. Take **3 thread dumps 5 to 10 seconds apart** (using `curl http://localhost:8080/manage/threaddump > td1.json`).
  2. Search for threads in state:
     - `"threadState": "RUNNABLE"`: If the same thread is on the exact same class and line of code across all 3 dumps, that method is stuck in an infinite loop or CPU-heavy spin.
     - `"threadState": "BLOCKED"` or `"WAITING"`: Indicates thread contention or deadlock (threads waiting for monitors held by other threads).
  3. Correlate with `/metrics/jvm.gc.pause` — if CPU is pegged at 100% but application threads are idle, CPU is being consumed by JVM GC threads (GC overhead limit exceeded).

---

#### 2. OutOfMemoryError (`java.lang.OutOfMemoryError: Java heap space`) & Memory Leaks

- **Actuator Endpoints:**
  - `GET /manage/heapdump`
  - `GET /manage/metrics/jvm.memory.used`
  - `GET /manage/metrics/jvm.memory.max`
  - `GET /manage/metrics/jvm.gc.pause`
- **Why Check This?**
  - Logs only show the symptom: `java.lang.OutOfMemoryError: Java heap space`, but logs do not reveal which objects are hogging the memory.
  - `/heapdump` creates an instant binary snapshot (`.hprof` file) of the live JVM memory.
- **How to Investigate:**
  1. Download the heap dump file:
     ```bash
     curl -O http://localhost:8080/manage/heapdump
     ```
  2. Open the file in an analysis tool such as **Eclipse Memory Analyzer (MAT)**, **VisualVM**, or **JProfiler**.
  3. Look at the **Dominator Tree** and **Leak Suspects Report**:
     - Common culprits: Unbounded in-memory caches, unclosed DB result sets, static collections (`static List` / `Map`), large file uploads buffered into byte arrays in memory.
  4. Check `/metrics/jvm.gc.pause`: if `TOTAL_TIME` and `COUNT` are surging while `jvm.memory.used` remains near `jvm.memory.max` after GC, the JVM is unable to reclaim memory (true memory leak).

---

#### 3. Database Connection Starvation (`HikariPool - Connection is not available`)

- **Actuator Endpoints:**
  - `GET /manage/metrics/hikaricp.connections.active`
  - `GET /manage/metrics/hikaricp.connections.idle`
  - `GET /manage/metrics/hikaricp.connections.pending`
  - `GET /manage/metrics/hikaricp.connections.timeout`
  - `GET /manage/health`
- **Why Check This?**
  - A common production outage occurs when all database connections in the Hikari pool are exhausted, causing incoming HTTP requests to block and time out with `500 Internal Server Error` or `SQLTransientConnectionException`.
- **How to Investigate:**
  1. Inspect the connection count metrics:
     - `hikaricp.connections.active`: Number of connections currently executing SQL queries.
     - `hikaricp.connections.idle`: Number of available connections.
     - `hikaricp.connections.pending`: Number of threads currently waiting to acquire a connection.
  2. **Diagnosis:**
     - If `active == max` and `pending > 0`: Connection pool is exhausted.
     - **Why is it exhausted?**
       - **Slow Queries / Missing Indexes:** Queries take 5–10 seconds instead of 10ms, tying up connections.
       - **Connection Leak:** Transaction not closing or third-party HTTP call performed inside `@Transactional` holding the connection open during network wait time.
       - Take a `/threaddump` to find threads stuck inside `org.postgresql.jdbc` or `com.mysql.cj.jdbc`.

---

#### 4. High API Latency, Slow Endpoints & HTTP 5xx Spikes

- **Actuator Endpoints:**
  - `GET /manage/metrics/http.server.requests`
  - `GET /manage/httpexchanges` (Spring Boot 3) / `GET /manage/httptrace` (Spring Boot 2)
- **Why Check This?**
  - Pinpoints exactly which API route is slow without needing to query distributed logs or tracing systems first.
- **How to Investigate:**
  1. Query metrics filtered by tags to isolate specific endpoints and error codes:
     ```http
     GET /manage/metrics/http.server.requests?tag=uri:/api/v1/checkout&tag=status:500
     ```
  2. The response provides:
     - `COUNT`: Total requests handled.
     - `TOTAL_TIME`: Cumulative time spent.
     - `MAX`: Peak response time (e.g., if MAX is 28.5 seconds, some request hit a downstream timeout).
     - `TOTAL_TIME / COUNT`: Average latency.
  3. Use `/httpexchanges` to see recent HTTP requests, including request headers, response status, and duration in milliseconds.

---

#### 5. Downstream Dependency Outage (Database, Redis, Kafka Down)

- **Actuator Endpoints:**
  - `GET /manage/health`
- **Why Check This?**
  - When customer requests suddenly fail with generic errors, `/health` immediately reports which downstream subsystem failed without having to inspect multiple dashboards.
- **How to Investigate:**
  - Ensure `management.endpoint.health.show-details=always` is enabled.
  - Hitting `/manage/health` reveals component-by-component status:
    ```json
    {
      "status": "DOWN",
      "components": {
        "db": {
          "status": "UP",
          "details": { "database": "PostgreSQL", "validationQuery": "isValid()" }
        },
        "redis": {
          "status": "DOWN",
          "details": { "error": "org.springframework.data.redis.RedisConnectionFailureException: Unable to connect" }
        },
        "diskSpace": {
          "status": "UP",
          "details": { "total": 104857600000, "free": 45000000000, "threshold": 10485760 }
        }
      }
    }
    ```
  - **Verdict:** Immediately isolates the issue to Redis connectivity; PostgreSQL and Disk are healthy.

---

#### 6. Misconfiguration / Wrong Environment Variables After a Deployment

- **Actuator Endpoints:**
  - `GET /manage/env`
  - `GET /manage/configprops`
- **Why Check This?**
  - Following a deployment, a service might behave unexpectedly (e.g. connecting to the wrong database URL, using incorrect feature flags, or loading the `dev` profile instead of `prod`).
- **How to Investigate:**
  1. `GET /manage/env`:
     - Shows all active profiles (`spring.profiles.active`).
     - Shows the exact hierarchy of configuration sources: system properties > OS environment variables > `application-prod.properties` > `application.properties`.
     - Identifies which source overrode a property.
  2. `GET /manage/configprops`:
     - Displays the actual bound values of all `@ConfigurationProperties` beans in the Spring container.
  3. Note: Sensitive values (passwords, secret keys) are masked by default (`******`) to prevent credential leakage.

---

#### 7. Live Debugging Without Restarting (Dynamic Log Level Change)

- **Actuator Endpoints:**
  - `GET /manage/loggers`
  - `POST /manage/loggers/{package-or-class}`
- **Why Check This?**
  - In production, log levels are set to `INFO` or `WARN` for performance and disk hygiene.
  - When an edge-case bug occurs in production, you cannot reproduce it in QA, and restarting the service with `DEBUG` logs will wipe the in-memory state or lose the affected instance.
  - Actuator allows you to dynamically change log levels at runtime for specific packages without application downtime.
- **How to Use:**
  1. Check current log level:
     ```bash
     curl http://localhost:8080/manage/loggers/com.example.service.OrderService
     ```
     Output: `{"configuredLevel":"INFO","effectiveLevel":"INFO"}`
  2. Change log level to `DEBUG` on the fly:
     ```bash
     curl -X POST http://localhost:8080/manage/loggers/com.example.service.OrderService \
          -H "Content-Type: application/json" \
          -d '{"configuredLevel":"DEBUG"}'
     ```
  3. Observe detailed debug logs in real time to capture the bug.
  4. Reset back to `INFO` once diagnosed to avoid excessive disk I/O:
     ```bash
     curl -X POST http://localhost:8080/manage/loggers/com.example.service.OrderService \
          -H "Content-Type: application/json" \
          -d '{"configuredLevel":"INFO"}'
     ```

---

#### 8. HTTP 404 Not Found / Missing Request Mapping

- **Actuator Endpoints:**
  - `GET /manage/mappings`
  - `GET /manage/beans`
- **Why Check This?**
  - When an API returns `404 Not Found` even though the controller code exists in the repository.
- **How to Investigate:**
  1. `GET /manage/mappings`:
     - Lists every registered Spring MVC / WebFlux handler mapping, including HTTP methods, URL patterns, consumes/produces types.
     - Confirms whether the path prefix (e.g. `/api/v1`) or path variables matched the expected pattern.
  2. If the URL is missing from `/mappings`, check `GET /manage/beans`:
     - Confirms whether the `@RestController` bean was actually scanned and instantiated by the Spring component scanner.
     - If missing, check package scanning (`@ComponentScan`) or missing annotations (`@Controller` vs `@Service`).

---

#### 9. Tomcat Worker Thread or Async Executor Starvation

- **Actuator Endpoints:**
  - `GET /manage/metrics/tomcat.threads.busy`
  - `GET /manage/metrics/tomcat.threads.current`
  - `GET /manage/metrics/executor.active`
  - `GET /manage/metrics/executor.queued`
- **Why Check This?**
  - When requests start hanging or timing out with gateway errors (`504 Gateway Timeout`), but CPU and Memory are low.
- **How to Investigate:**
  - Tomcat has a default pool limit (typically 200 threads).
  - If `tomcat.threads.busy == tomcat.threads.current == 200`, all worker threads are stuck waiting (e.g. for downstream web services or database locks). No new HTTP connections can be accepted.
  - For `@Async` tasks: if `executor.queued` is continuously increasing, the background thread pool is overwhelmed and tasks are piling up in memory.

---

#### 10. Kubernetes Pod Restarts & Health Check Failures

- **Actuator Endpoints:**
  - `GET /manage/health/liveness`
  - `GET /manage/health/readiness`
- **Why Check This?**
  - In containerized environments (Kubernetes, ECS), misconfigured health probes cause unnecessary pod restarts (CrashLoopBackOff) or dropping healthy pods from the load balancer.
- **Distinction:**
  - **Liveness Probe (`/health/liveness`):**
    - Checks if internal application state is corrupt or permanently deadlocked.
    - If status is `DOWN`, Kubernetes terminates and restarts the container.
  - **Readiness Probe (`/health/readiness`):**
    - Checks if the application is currently able to serve incoming traffic (e.g. caches initialized, DB connections available).
    - If status is `DOWN`, Kubernetes removes the pod from the Service endpoint so traffic stops routing to it, but does **not** kill the pod.
