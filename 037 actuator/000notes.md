

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
#### jvm.gc.pause

All Spring Boot Actuator metrics are automatically pushed to Datadog once configured. In Datadog, choose the metric to monitor, e.g. `http.server.requests.count`, under Metrics > Summary.

---
In Spring Boot Actuator (powered by Micrometer), jvm.gc.pause is a Timer metric that measures the frequency and duration of Garbage Collection (GC) pause events.
During a GC pause (a "Stop-The-World" event), application threads are halted while the JVM cleans up unreferenced memory. This metric helps track GC overhead, identify latency spikes, and detect memory pressure.

#### Key Information Provided

Because it is a Timer instrument, querying /actuator/metrics/jvm.gc.pause returns three core statistics:
 * COUNT: The total number of GC pause events that occurred since application startup.
 * TOTAL_TIME: The cumulative duration (usually in seconds) spent across all GC pause events.
 * MAX: The duration of the single longest GC pause in the current monitoring window.
Important Tags (Dimensions)
Micrometer enriches this metric with tags to help filter and isolate specific GC behaviors:

| Tag | Purpose | Common Values |
|---|---|---|
| action | The general classification of the collection. | end of minor GC, end of major GC |
| cause | Why the garbage collector was triggered. | Allocation Failure, G1 Evacuation Pause, System.gc(), Metadata GC Threshold |
| name | The specific collector running inside the JVM. | G1 Young Generation, G1 Old Generation, ZGC Pauses, Parallel GC |
Example Actuator Response
Calling GET /actuator/metrics/jvm.gc.pause:

```json
{
  "name": "jvm.gc.pause",
  "description": "Time spent in GC pause",
  "baseUnit": "seconds",
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 14.0
    },
    {
      "statistic": "TOTAL_TIME",
      "value": 0.412
    },
    {
      "statistic": "MAX",
      "value": 0.085
    }
  ],
  "availableTags": [
    {
      "tag": "action",
      "values": ["end of minor GC", "end of major GC"]
    },
    {
      "tag": "cause",
      "values": ["Allocation Failure", "Metadata GC Threshold"]
    },
    {
      "tag": "name",
      "values": ["G1 Young Generation", "G1 Old Generation"]
    }
  ]
}

```

You can drill down into specific tags by appending query parameters:
GET /actuator/metrics/jvm.gc.pause?tag=action:end of major GC

Practical Use Cases in Monitoring (e.g., Prometheus & Grafana)
 * Calculate GC Throughput Overhead:
   Using Prometheus PromQL, measure what percentage of CPU time is spent blocked by GC:
   rate(jvm_gc_pause_seconds_sum[5m])

   If this value exceeds 0.05 (meaning 5% or more of wall-clock time is paused), the JVM is experiencing significant allocation pressure.
 * Diagnose P99 Request Latency Spikes:
   Compare application request latency spikes with jvm.gc.pause.max. If high API response latencies align with spikes in jvm.gc.pause, the issue is garbage collection, not database or network latency.
 * Catch Accidental System.gc() Calls:
   Filter by cause="System.gc()" to identify unwanted manual GC triggers (often triggered by legacy libraries or RMI).

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

---

## How to Detect and Optimize a Slow API in Production

Detecting and optimizing slow APIs is a systematic end-to-end engineering workflow. In real-world enterprise applications, you **never guess what is slow** — you measure first, isolate the bottleneck layer, and apply targeted optimizations.

```
           [ STEP 1: DETECTION & PROFILING ]
Actuator Metrics / Percentiles ──> Distributed Tracing (APM) ──> Hikari / DB Logs
                                 │
                                 ▼
                     Where is time spent?
            ┌────────────────────┬────────────────────┐
            ▼                    ▼                    ▼
     [ Database Bottleneck ] [ Network / Downstream ] [ JVM / Application ]
            │                    │                    │
            ▼                    ▼                    ▼
   [ STEP 2: DATABASE FIX ] [ STEP 3: ASYNC / CACHE ] [ STEP 4: JVM / RUNTIME ]
   • Indexes                • CompletableFuture       • Virtual Threads
   • Fix N+1 queries        • Redis / Caffeine Cache  • GZIP Compression
   • DTO Projections        • Kafka / @Async Queues   • Low-latency GC (ZGC)
   • Keyset pagination      • HTTP Connection Pool    • Memory leak fixes
```

---

### Part 1: How to Detect a Slow API

Before applying any code changes, you must answer three questions:
1. *Which endpoint is slow?*
2. *Is it slow for all users (average) or only for the tail end (p95/p99 spikes)?*
3. *Which layer is burning the time (Application, DB, External REST call, or JVM GC pause)?*

---

#### 1. Actuator Metrics (`http.server.requests`) & Latency Percentiles

Spring Boot Actuator records latency for all incoming HTTP requests via Micrometer.

- **Check metric values:**
  ```http
  GET /manage/metrics/http.server.requests
  ```
- **Filter by URI to inspect a specific slow API:**
  ```http
  GET /manage/metrics/http.server.requests?tag=uri:/api/v1/orders
  ```

  Sample Response:
  ```json
  {
    "name": "http.server.requests",
    "measurements": [
      { "statistic": "COUNT", "value": 15000 },
      { "statistic": "TOTAL_TIME", "value": 7500.5 },
      { "statistic": "MAX", "value": 4.82 }
    ]
  }
  ```
  - **Average Latency:** `TOTAL_TIME / COUNT` = `7500.5 / 15000` = `0.5s (500ms)`.
  - **Max Latency:** `4.82s` indicates serious latency spikes for certain requests.

- **Enable Percentiles (p50, p95, p99) and Histograms:**
  Average latency can hide severe tail latency (where 1% of users wait 5+ seconds). Configure percentiles in `application.properties`:

  ```properties
  # Calculate p50, p90, p95, p99 latency percentiles
  management.metrics.distribution.percentiles[http.server.requests]=0.5,0.9,0.95,0.99

  # Enable histogram buckets to see distribution
  management.metrics.distribution.percentiles-histogram[http.server.requests]=true

  # Define SLA thresholds for alerting (e.g. 100ms, 500ms, 1s, 2s)
  management.metrics.distribution.slo[http.server.requests]=100ms,500ms,1s,2s
  ```

---

#### 2. Request Level Tracking (`/httpexchanges` or `/httptrace`)

Shows the last 100 HTTP requests in memory, including their exact URI, HTTP method, timestamp, and response duration in milliseconds.

- **Endpoint:**
  ```http
  GET /manage/httpexchanges
  ```
- **Spring Boot 3 Configuration:**
  ```java
  @Configuration
  public class HttpExchangeConfig {
      @Bean
      public HttpExchangeRepository httpExchangeRepository() {
          return new InMemoryHttpExchangeRepository();
      }
  }
  ```

---

#### 3. Distributed Tracing (APM / Micrometer Tracing / OpenTelemetry)

When an API calls multiple internal services and databases, logging timestamps is not enough. Distributed tracing assigns a unique `Trace ID` and `Span ID` to each request across microservices.

```
Request (Trace ID: a1b2c3d4)
 ├── [Span 1] API Gateway: 5ms
 └── [Span 2] Order Service: 1420ms
      ├── [Span 3] Auth Filter: 15ms
      ├── [Span 4] User Profile Service (HTTP): 80ms
      ├── [Span 5] Payment Service (HTTP): 1100ms   <=== BOTTLENECK FOUND!
      └── [Span 6] PostgreSQL Query: 225ms
```

- **Spring Boot 3 Tracing Setup:**
  ```xml
  <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-tracing-bridge-brave</artifactId>
  </dependency>
  <dependency>
      <groupId>io.zipkin.reporter2</groupId>
      <artifactId>zipkin-reporter-brave</artifactId>
  </dependency>
  ```
  ```properties
  # Sample 100% of requests in dev/staging (adjust in production, e.g. 0.1 for 10%)
  management.tracing.sampling.probability=1.0
  ```

---

#### 4. Database & Connection Pool Profiling

In over 70% of production issues, a slow API is caused by slow database operations or connection pool starvation.

- **HikariCP Connection Pool Metrics:**
  - `hikaricp.connections.acquire`: Time taken for an incoming request thread to get a connection from the pool. If this is high, threads are waiting because all pool connections are occupied.
  - `hikaricp.connections.pending`: Count of threads actively waiting for a free connection.
- **Database Slow Query Log:**
  - MySQL:
    ```sql
    SET GLOBAL slow_query_log = 'ON';
    SET GLOBAL long_query_time = 0.5; -- logs queries taking > 500ms
    ```
  - PostgreSQL:
    ```sql
    SET log_min_duration_statement = 500; -- logs queries taking > 500ms
    ```
- **Hibernate Statistics (Development & Staging):**
  ```properties
  spring.jpa.properties.hibernate.generate_statistics=true
  logging.level.org.hibernate.stat=DEBUG
  ```
  Reports total JDBC execution time, number of flushes, and total queries executed per transaction (instantly reveals N+1 query bugs).

---

#### 5. Custom AOP Timing Interceptor

For rapid detection without full APM infrastructure:

```java
@Aspect
@Component
@Slf4j
public class PerformanceMonitoringAspect {

    @Around("@annotation(org.springframework.web.bind.annotation.GetMapping) || " +
            "@annotation(org.springframework.web.bind.annotation.PostMapping)")
    public Object profileEndpoint(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            if (duration > 500) { // Alert on execution exceeding 500ms
                log.warn("SLOW API WARNING: Endpoint [{}] took {} ms",
                        joinPoint.getSignature().toShortString(), duration);
            }
        }
    }
}
```

---

### Part 2: How to Optimize a Slow API (Layer-by-Layer)

Once profiling identifies where time is spent, apply targeted optimizations systematically across 4 distinct layers:

---

#### Layer 1: Database & Persistence Optimization (~70–80% of Slow APIs)

##### 1. Indexing & Query Plan (`EXPLAIN ANALYZE`)
- **Problem:** Queries performing full table scans (`Seq Scan` / `ALL`) on millions of rows.
- **Detection:** Run `EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 120 AND status = 'COMPLETED';`
- **Solution:** Add single-column or composite B-tree index:
  ```sql
  -- Composite index matching filter criteria
  CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
  ```

##### 2. Fix the Hibernate N+1 Query Problem
- **Problem:** Querying 100 orders triggers 1 query for orders + 100 individual queries for associated customers.
- **Solution:**
  - **`JOIN FETCH` in JPQL:**
    ```java
    @Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.customer WHERE o.customer.id = :customerId")
    List<Order> findOrdersWithDetails(@Param("customerId") Long customerId);
    ```
  - **`@EntityGraph`:**
    ```java
    @EntityGraph(attributePaths = {"items", "customer"})
    List<Order> findByCustomerId(Long customerId);
    ```
  - **Batch Fetching (`default_batch_fetch_size`):**
    ```properties
    # Converts 100 individual queries into batch queries: WHERE id IN (?, ?, ?, ...)
    spring.jpa.properties.hibernate.default_batch_fetch_size=30
    ```

##### 3. Use DTO Projections Instead of Entire Entities
- **Problem:** `SELECT o FROM Order o` loads all 50 columns, manages entities in Hibernate L1 cache, and consumes massive heap memory.
- **Solution:** Fetch only required fields using Java Records:
  ```java
  public record OrderSummaryDTO(Long orderId, String orderNumber, BigDecimal totalAmount) {}

  public interface OrderRepository extends JpaRepository<Order, Long> {
      @Query("SELECT new com.example.dto.OrderSummaryDTO(o.id, o.orderNumber, o.totalAmount) " +
             "FROM Order o WHERE o.customer.id = :customerId")
      List<OrderSummaryDTO> findSummariesByCustomerId(@Param("customerId") Long customerId);
  }
  ```

##### 4. Keyset / Cursor Pagination Instead of Deep `OFFSET`
- **Problem:** `SELECT * FROM orders OFFSET 100000 LIMIT 20` reads 100,020 rows and discards the first 100,000.
- **Solution:** Use Keyset Pagination:
  ```sql
  -- Direct B-Tree lookup using index on ID
  SELECT * FROM orders 
  WHERE customer_id = :customerId AND id > :lastSeenId 
  ORDER BY id ASC 
  LIMIT 20;
  ```

##### 5. Avoid Holding Transactions Open During Network I/O
- **Problem:** Putting `@Transactional` on a method that makes an external HTTP request keeps a database connection checked out of Hikari pool for hundreds of milliseconds:
  ```java
  // BAD: Ties up a DB connection while waiting for 3rd-party HTTP call
  @Transactional
  public OrderResponse placeOrder(OrderRequest request) {
      Order order = orderRepo.save(createOrder(request));
      paymentClient.chargePayment(order); // External HTTP Call (900ms)
      return new OrderResponse(order);
  }
  ```
- **Solution:** Keep `@Transactional` tightly scoped to database operations only:
  ```java
  // GOOD: No DB connection held during HTTP call
  public OrderResponse placeOrder(OrderRequest request) {
      PaymentResult result = paymentClient.chargePayment(request); // Outside transaction
      return orderService.persistOrderWithPayment(request, result); // @Transactional inside
  }
  ```

---

#### Layer 2: Caching Strategy

##### 1. In-Memory Local Cache (Caffeine)
- Ideal for small, static, frequently read datasets (e.g. tax rates, config tables, countries):
  ```java
  @Cacheable(value = "countries", sync = true)
  public List<Country> getAllCountries() {
      return countryRepository.findAll();
  }
  ```

##### 2. Distributed Cache (Redis)
- Ideal for shared multi-pod caching of complex aggregated data:
  ```java
  @Cacheable(value = "userProfiles", key = "#userId", unless = "#result == null")
  public UserProfileDTO getUserProfile(Long userId) {
      return userRepository.findProfileById(userId);
  }
  ```
  - Always configure an appropriate **TTL (Time to Live)** to prevent stale data and Redis memory saturation.

##### 3. HTTP Level Caching & Conditional Requests (`ETag` / 304 Not Modified)
- Let client browsers and CDNs cache responses without repeatedly parsing or sending payload bytes:
  ```java
  @GetMapping("/products/{id}")
  public ResponseEntity<ProductDTO> getProduct(@PathVariable Long id, WebRequest request) {
      ProductDTO product = productService.findById(id);
      String etag = "\"" + product.version() + "\"";

      if (request.checkNotModified(etag)) {
          return null; // Automatically returns 304 Not Modified with empty body
      }

      return ResponseEntity.ok()
              .eTag(etag)
              .cacheControl(CacheControl.maxAge(300, TimeUnit.SECONDS))
              .body(product);
  }
  ```

---

#### Layer 3: Network & Downstream Dependencies

##### 1. Parallelize Independent Downstream Calls
- If an API calls 3 external services sequentially:
  - User Service (200ms) + Inventory Service (300ms) + Notification Service (250ms) = **750ms total**.
- Execute concurrently with `CompletableFuture.allOf()`:
  ```java
  public OrderSummaryAggregate getAggregatedOrderSummary(Long orderId) {
      CompletableFuture<OrderDetails> orderFuture = 
          CompletableFuture.supplyAsync(() -> orderService.getOrder(orderId));
      CompletableFuture<InventoryStatus> inventoryFuture = 
          CompletableFuture.supplyAsync(() -> inventoryService.getStatus(orderId));
      CompletableFuture<ShippingDetails> shippingFuture = 
          CompletableFuture.supplyAsync(() -> shippingService.getDetails(orderId));

      CompletableFuture.allOf(orderFuture, inventoryFuture, shippingFuture).join();

      return new OrderSummaryAggregate(
          orderFuture.join(),
          inventoryFuture.join(),
          shippingFuture.join()
      );
      // Latency becomes max(200, 300, 250) ≈ 300ms (more than 50% reduction!)
  }
  ```

##### 2. Offload Non-Critical Work Asynchronously
- Tasks such as sending emails, audit logging, push notifications, and analytics events should never block API response time:
  - Publish an event to Kafka / RabbitMQ or execute via `@Async`:
  ```java
  @PostMapping("/register")
  public ResponseEntity<UserResponse> register(@RequestBody UserRegistrationRequest request) {
      User user = userService.createUser(request);
      // Asynchronous event: API returns in 20ms instead of waiting 1.5s for SMTP email sending
      eventPublisher.publishEvent(new UserRegisteredEvent(user.getId()));
      return ResponseEntity.status(HttpStatus.CREATED).body(new UserResponse(user));
  }
  ```

##### 3. HTTP Client Connection Pooling & Timeouts
- Creating a new TCP connection on every HTTP call adds 50–150ms per request (TLS handshake overhead).
- Use a pooled `RestClient` or `RestTemplate` with strict connect and read timeouts:
  ```java
  @Bean
  public RestClient restClient() {
      PoolingHttpClientConnectionManager poolManager = new PoolingHttpClientConnectionManager();
      poolManager.setMaxTotal(200);
      poolManager.setDefaultMaxPerRoute(50);

      HttpClient httpClient = HttpClients.custom()
              .setConnectionManager(poolManager)
              .build();

      return RestClient.builder()
              .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
              .build();
  }
  ```

---

#### Layer 4: Application & JVM Runtime Optimization

##### 1. Virtual Threads (Java 21+ Project Loom / Spring Boot 3.2+)
- Traditional Spring MVC assigns 1 OS platform thread per incoming request (Tomcat default: 200 threads). When threads wait on DB or external HTTP I/O, the pool starves.
- Enable virtual threads:
  ```properties
  spring.threads.virtual.enabled=true
  ```
  - JVM creates lightweight virtual threads on demand (~1KB stack vs ~1MB OS thread stack).
  - Eliminates thread pool exhaustion on I/O-bound workloads.

##### 2. Enable GZIP Response Compression
- Compresses large JSON responses (e.g. 500KB JSON reduced to ~40KB):
  ```properties
  server.compression.enabled=true
  server.compression.mime-types=application/json,application/xml,text/html
  server.compression.min-response-size=2048
  ```

##### 3. JVM Memory & Low-Latency Garbage Collection
- Monitor `/metrics/jvm.gc.pause` to verify if Stop-The-World GC pauses are causing latency spikes.
- Switch to low-latency garbage collector (**ZGC** in Java 17+ or **G1GC**):
  ```bash
  # ZGC achieves sub-millisecond GC pauses even on large heaps
  java -XX:+UseZGC -XX:+ZGenerational -Xms4g -Xmx4g -jar app.jar
  ```

---

### Summary: Production Optimization Playbook

| Symptom / Observation | Detection Method | Root Cause | Optimization Fix |
|---|---|---|---|
| Latency spikes on specific endpoints under normal load | Actuator `/metrics/http.server.requests` | Missing DB Index or Full Table Scan | Run `EXPLAIN ANALYZE` and create composite indexes. |
| DB query count scales with number of returned rows | Hibernate stats (`generate_statistics`) | N+1 query problem | Use `JOIN FETCH`, `@EntityGraph`, or `default_batch_fetch_size`. |
| API latency increases as data grows | APM / Query logs | Deep `OFFSET` pagination or loading full entity trees | Switch to Keyset pagination and Record DTO Projections. |
| Requests queuing up while CPU & RAM are low | `hikaricp.connections.pending` > 0 | Connection pool starvation / Long transaction holding connection | Reduce connection hold time; remove external HTTP calls from `@Transactional`. |
| API calls multiple downstream services | Distributed Tracing (Zipkin / Jaeger) | Sequential blocking REST calls | Parallelize with `CompletableFuture.allOf()` or non-blocking `WebClient`. |
| Non-essential operations slowing down response | APM trace spans | Sending emails, audit logs, or PDF generation on main thread | Offload to Kafka / RabbitMQ or Spring `@Async`. |
| Tomcat worker thread pool exhaustion | `tomcat.threads.busy` = 200 | Blocking I/O holding platform threads | Enable Virtual Threads (`spring.threads.virtual.enabled=true`). |
| Periodic severe latency spikes across all endpoints | Actuator `/metrics/jvm.gc.pause` | Stop-the-world Full GC pauses | Tune JVM heap, eliminate memory leaks, switch to ZGC (`-XX:+UseZGC`). |

---

## Handling Distributed Failures in Microservices & Distributed Systems

In monolithic applications, in-memory method calls either succeed or throw an in-process exception. In distributed systems, communication happens over an unreliable network between independent services.

According to the **Fallacies of Distributed Computing**, networks are not reliable, latency is not zero, and downstream services will inevitably slow down, crash, or experience network partitions.

```
       [ CLIENT ]
           │
           ▼
     [ API Gateway ]
           │
           ├────────────────────────────┐
           ▼                            ▼
   [ Order Service ]             [ User Service ]
           │ (Slow / Down)              │ (Healthy)
           ▼                            ▼
  [ Payment Service ]            [ PostgreSQL ]
   (Takes 30s or hangs)
           │
           ▼
  ⚠️ CASCADING FAILURE:
  Order Service exhausts all 200 Tomcat threads waiting for Payment Service.
  Order Service crashes.
  API Gateway threads get stuck waiting for Order Service.
  Entire platform goes down!
```

---

### Core Philosophy: "Expect Failure, Design for Graceful Degradation"

When a downstream dependency fails or degrades, the system must:
1. **Fail fast** (never hang or hold resources indefinitely).
2. **Isolate the blast radius** (failure of recommendations must not crash checkout).
3. **Gracefully degrade** (return fallback data or partial response).
4. **Self-heal** (automatically resume normal operation when downstream recovers).
5. **Guarantee eventual consistency** (ensure distributed data doesn't corrupt on partial failures).

---

### Strategy 1: The Circuit Breaker Pattern (Resilience4j)

A Circuit Breaker prevents an application from repeatedly trying to execute an operation that is almost guaranteed to fail, saving thread resources and giving the failing service time to recover.

#### The Three Circuit Breaker States

```
        ┌──────────────────────────────────────────────────┐
        │                                                  │
        ▼                                                  │ Failure rate drops
  ┌───────────┐      Failure Rate > Threshold (e.g. 50%)    │ below threshold
  │  CLOSED   │ ───────────────────────────────────────► ┌──────┐
  │ (Normal)  │                                          │ OPEN │ (Fail Fast,
  └───────────┘ ◄─────────────────────────────────────── └──────┘  Calls Blocked)
        ▲        Canary calls succeed                       │
        │                                                  │ Wait duration expires
        │               ┌───────────────┐                  │ (e.g. 10s cooldown)
        └────────────── │   HALF-OPEN   │ ◄────────────────┘
                        │ (Trial Probe) │
                        └───────────────┘
```

1. **CLOSED (Normal State):** Requests flow to the downstream service. The circuit breaker monitors success and failure rates over a sliding window.
2. **OPEN (Tripped / Broken State):** When the failure rate (or slow call rate) exceeds the configured threshold (e.g., 50%), the breaker trips to `OPEN`. All incoming calls **fail immediately** without touching the network, throwing a `CallNotPermittedException` and routing straight to a fallback.
3. **HALF-OPEN (Testing Recovery):** After a cooldown wait duration (e.g., 10 seconds), the breaker transitions to `HALF-OPEN` and permits a small number of trial/canary requests. If they succeed, it returns to `CLOSED`. If they fail, it trips back to `OPEN`.

#### Implementation with Resilience4j & Spring Boot 3

`pom.xml`:
```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

`application.properties`:
```properties
# Circuit Breaker Configuration for paymentService
resilience4j.circuitbreaker.instances.paymentService.sliding-window-type=COUNT_BASED
resilience4j.circuitbreaker.instances.paymentService.sliding-window-size=10
resilience4j.circuitbreaker.instances.paymentService.minimum-number-of-calls=5
resilience4j.circuitbreaker.instances.paymentService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.paymentService.slow-call-rate-threshold=70
resilience4j.circuitbreaker.instances.paymentService.slow-call-duration-threshold=2s
resilience4j.circuitbreaker.instances.paymentService.wait-duration-in-open-state=10s
resilience4j.circuitbreaker.instances.paymentService.permitted-number-of-calls-in-half-open-state=3
resilience4j.circuitbreaker.instances.paymentService.automatic-transition-from-open-to-half-open-enabled=true

# Expose Circuit Breakers in Spring Boot Actuator /health
management.health.circuitbreakers.enabled=true
```

`PaymentClient.java`:
```java
@Service
@Slf4j
public class PaymentClient {

    private final RestClient restClient;

    public PaymentClient(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("https://payment-api.internal").build();
    }

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return restClient.post()
                .uri("/v1/charge")
                .body(request)
                .retrieve()
                .body(PaymentResponse.class);
    }

    // Fallback method MUST have matching parameter signature + Throwable parameter
    public PaymentResponse paymentFallback(PaymentRequest request, Throwable ex) {
        log.error("Payment Service unavailable or circuit open. Triggering fallback. Reason: {}", ex.getMessage());
        return new PaymentResponse(
            request.orderId(),
            PaymentStatus.PENDING_ASYNC_VERIFICATION,
            "Payment queued for background retry"
        );
    }
}
```

---

### Strategy 2: Timeouts (Fail Fast)

The single most dangerous failure mode in distributed systems is **slowness, not crashes**. A crashed server returns `503 Service Unavailable` in 2ms. A slow server hangs for 60 seconds, holding an upstream thread, database connection, and socket open.

**Rule:** Every external I/O call (HTTP, gRPC, JDBC, Redis) must have explicit, defensive timeouts.

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        // Time to establish TCP/TLS connection with server
        factory.setConnectTimeout(Duration.ofMillis(1000));
        // Max time to wait for downstream response packet
        factory.setReadTimeout(Duration.ofMillis(2500));
        return new RestTemplate(factory);
    }
}
```

With Resilience4j TimeLimiter:
```properties
resilience4j.timelimiter.instances.orderService.timeout-duration=2s
resilience4j.timelimiter.instances.orderService.cancel-running-future=true
```

---

### Strategy 3: Retries with Exponential Backoff and Jitter

Not all failures are permanent. Transient failures (temporary network blip, brief DNS glitch, pod restart) resolve in milliseconds.

#### The Golden Rules of Retrying
1. **Only retry idempotent operations:** Safe on `GET`, `PUT`, `DELETE`. Be extremely cautious retrying `POST /payments/charge` without an Idempotency Key!
2. **Only retry transient HTTP status codes:** Retry `503 Service Unavailable`, `504 Gateway Timeout`, or socket connection timeouts. **Never** retry `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, or `404 Not Found`.
3. **Always use Exponential Backoff + Random Jitter:**
   - Retrying immediately causes the **Thundering Herd Problem** (thousands of failed clients bombard a recovering service at the exact same millisecond, crashing it again).

$$\text{Wait Time} = \text{Initial Interval} \times (\text{Multiplier})^{\text{Attempt}} + \text{Random Jitter}$$

```properties
resilience4j.retry.instances.inventoryService.max-attempts=3
resilience4j.retry.instances.inventoryService.wait-duration=200ms
resilience4j.retry.instances.inventoryService.enable-exponential-backoff=true
resilience4j.retry.instances.inventoryService.exponential-backoff-multiplier=2
resilience4j.retry.instances.inventoryService.enable-randomized-wait=true
resilience4j.retry.instances.inventoryService.randomized-wait-factor=0.5
resilience4j.retry.instances.inventoryService.retry-exceptions=java.io.IOException,org.springframework.web.client.ResourceAccessException
resilience4j.retry.instances.inventoryService.ignore-exceptions=org.springframework.web.client.HttpClientErrorException
```

```java
@Service
public class InventoryClient {

    @Retry(name = "inventoryService", fallbackMethod = "inventoryFallback")
    public InventoryStatus checkStock(String productId) {
        // Will attempt up to 3 times: wait 200ms, then 400ms (+ jitter)
        return restClient.get()
                .uri("/inventory/{productId}", productId)
                .retrieve()
                .body(InventoryStatus.class);
    }

    public InventoryStatus inventoryFallback(String productId, Exception ex) {
        // Degrade gracefully: assume temporarily out of stock or return cached status
        return new InventoryStatus(productId, StockStatus.UNKNOWN_DEGRADED);
    }
}
```

---

### Strategy 4: Bulkhead Pattern (Isolation of Resources)

Inspired by the watertight compartments of a ship: if one compartment floods, the ship stays afloat.

In a Spring Boot app with 200 Tomcat threads:
- If `RecommendationsService` takes 10s and 200 users hit `/recommendations`, all 200 Tomcat threads get stuck.
- Now, critical `/checkout` and `/login` requests get rejected with `504 Gateway Timeout`!

#### Thread Pool Bulkhead vs Semaphore Bulkhead

| Type | How It Works | Best Used For |
|---|---|---|
| **Thread Pool Bulkhead** | Assigns a dedicated thread pool and bounded queue for a specific downstream client. Requests execute asynchronously. | Remote HTTP calls, microservice clients. |
| **Semaphore Bulkhead** | Limits the number of concurrent calls executing on the calling thread using a counter. | In-memory operations, low-overhead calls. |

```properties
# Reserve a maximum of 10 concurrent threads for recommendations
resilience4j.bulkhead.instances.recommendations.max-concurrent-calls=10
resilience4j.bulkhead.instances.recommendations.max-wait-duration=20ms

# Thread pool bulkhead
resilience4j.thread-pool-bulkhead.instances.recommendations.max-thread-pool-size=10
resilience4j.thread-pool-bulkhead.instances.recommendations.core-thread-pool-size=5
resilience4j.thread-pool-bulkhead.instances.recommendations.queue-capacity=20
```

```java
@Service
public class RecommendationService {

    @Bulkhead(name = "recommendations", fallbackMethod = "defaultRecommendations", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<List<Product>> getRecommendations(Long userId) {
        return CompletableFuture.supplyAsync(() -> remoteCall(userId));
    }

    public CompletableFuture<List<Product>> defaultRecommendations(Long userId, Throwable ex) {
        // If all 10 recommendation threads are busy, immediately return static top sellers
        return CompletableFuture.completedFuture(staticTopSellersList);
    }
}
```

---

### Strategy 5: Rate Limiting & Load Shedding

When downstream services or your own API is flooded with traffic (DDoS, viral traffic, unthrottled batch job):
- Reject excess traffic early with `HTTP 429 Too Many Requests` rather than letting servers run out of memory or crash.

```properties
# Allow maximum 50 requests per second
resilience4j.ratelimiter.instances.orderApi.limit-for-period=50
resilience4j.ratelimiter.instances.orderApi.limit-refresh-period=1s
resilience4j.ratelimiter.instances.orderApi.timeout-duration=10ms
```

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @PostMapping
    @RateLimiter(name = "orderApi", fallbackMethod = "rateLimitFallback")
    public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest req) {
        return ResponseEntity.ok(orderService.create(req));
    }

    public ResponseEntity<OrderResponse> rateLimitFallback(OrderRequest req, RequestNotPermitted ex) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .header("Retry-After", "5")
                .build();
    }
}
```

---

### Strategy 6: Distributed Transaction Failures — The Saga Pattern

In microservices, you cannot use `@Transactional` across multiple databases (distributed 2-Phase Commit / XA transactions are slow, block resources, and fail under network partitions).

To maintain consistency across distributed boundaries, use the **Saga Pattern**. A Saga is a sequence of local transactions. If any step fails, the Saga executes **compensating transactions** to reverse earlier steps.

```
Happy Path:
[ 1. Create Pending Order ] ──► [ 2. Reserve Payment ] ──► [ 3. Reserve Inventory ] ──► [ 4. Confirm Order ]

Failure Path (Inventory Out of Stock):
[ 1. Create Pending Order ] ──► [ 2. Reserve Payment ] ──► [ 3. Reserve Inventory FAILS! ]
                                         │
                                         ▼ COMPENSATING ACTION
                                [ Refund / Cancel Payment ]
                                         │
                                         ▼ COMPENSATING ACTION
                                [ Mark Order CANCELLED ]
```

#### Choreography vs Orchestration Sagas

| Feature | Choreography Saga | Orchestration Saga |
|---|---|---|
| **Coordination** | Event-driven (Services listen to Kafka/RabbitMQ events and publish next events). | Centralized Orchestrator state machine (e.g. Temporal, Camunda, or a custom Spring Service). |
| **Coupling** | Loosely coupled, no central point of failure. | Tightly coordinated, clear workflow definition. |
| **Best For** | Simple workflows (2–4 microservices). | Complex enterprise workflows (5+ services, loops, timeouts). |

---

### Strategy 7: The Dual-Write Problem — Transactional Outbox Pattern

A classic distributed failure bug:
```java
// BUG: Non-atomic dual write
@Transactional
public void placeOrder(OrderRequest req) {
    orderRepository.save(order);       // 1. Saved to Postgres DB
    kafkaTemplate.send("orders", req); // 2. Network blip or Kafka down -> THROWS EXCEPTION!
}
```
- If DB commits but Kafka write fails $\to$ Event lost forever.
- If Kafka write succeeds but DB commits fail $\to$ Ghost event sent downstream.

#### Solution: The Transactional Outbox Pattern
Write the business entity and an event record to an `outbox` table in the **same local database transaction**:

```
 ┌─────────────────────────────────────────────────────────────┐
 │                   Single Local DB Transaction               │
 │                                                             │
 │   INSERT INTO orders (...)                                  │
 │   INSERT INTO outbox_table (id, aggregate_type, payload...) │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                                ▼ Guaranteed by ACID
                        [ Database Disk ]
                                │
                                ▼ CDC Poller / Debezium
                        [ Apache Kafka ]
```

```java
@Transactional
public void placeOrder(OrderRequest req) {
    Order order = orderRepository.save(new Order(req));

    // Persist event in outbox table within the EXACT SAME local DB transaction
    OutboxEvent event = new OutboxEvent(
        "ORDER_CREATED",
        order.getId().toString(),
        objectMapper.writeValueAsString(order)
    );
    outboxRepository.save(event);
}
```
A background poller (or Debezium CDC reading WAL logs) reliably streams events from `outbox_table` to Kafka with **At-Least-Once Delivery**.

---

### Strategy 8: Safe Retries with Idempotency Keys

Because network timeouts leave the client unsure whether a request reached the server or failed on the way back, clients **will retry**.

If a client retries `POST /payments/charge`, how do you prevent charging the customer twice?

#### Implementation with Idempotency Keys:
1. Client generates a unique `Idempotency-Key` (e.g., UUID v4) in the HTTP Header:
   `Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`
2. Server checks Redis or database for this key before executing the business logic:
   - **Key not found:** Atomically insert key with status `IN_PROGRESS` (using Redis `SET key value NX EX 120`). Proceed with payment. Store final response.
   - **Key exists & `COMPLETED`:** Return the saved response immediately without re-processing.
   - **Key exists & `IN_PROGRESS`:** Reject concurrent duplicate with `HTTP 409 Conflict`.

```java
@PostMapping("/charge")
public ResponseEntity<?> charge(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent("idemp:" + idempotencyKey, "PROCESSING", Duration.ofMinutes(2));

    if (Boolean.FALSE.equals(acquired)) {
        Object existingResult = redisTemplate.opsForValue().get("idemp:res:" + idempotencyKey);
        if (existingResult != null) {
            return ResponseEntity.ok(existingResult); // Return original cached response
        }
        return ResponseEntity.status(HttpStatus.CONFLICT).body("Concurrent request in progress");
    }

    PaymentResult result = paymentService.executePayment(request);

    // Save final response in Redis
    redisTemplate.opsForValue().set("idemp:res:" + idempotencyKey, result, Duration.ofHours(24));
    return ResponseEntity.ok(result);
}
```

---

### Strategy 9: Dead Letter Queues (DLQ) & Poison Pill Handling

When consuming messages from Kafka or RabbitMQ, some messages fail repeatedly (e.g., malformed JSON, corrupted data payload, or persistent DB constraint violation).

If the consumer retries indefinitely, the entire partition/queue is blocked (**Poison Pill**).

#### Solution:
Retry transient errors with backoff (e.g. 3 attempts). If still failing, route the poisoned message to a **Dead Letter Topic / Queue (DLQ)** and commit the offset so other messages can proceed.

```java
@Configuration
public class KafkaRetryConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // Retry 3 times with 1-second backoff, then send to {original-topic}.DLT
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
        FixedBackOff backOff = new FixedBackOff(1000L, 3L);

        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);
        // Do NOT retry fatal deserialization or bad data exceptions
        errorHandler.addNotRetryableExceptions(
            DeserializationException.class, 
            MethodArgumentNotValidException.class
        );
        return errorHandler;
    }
}
```

---

### Strategy 10: Monitoring Distributed Resilience with Actuator

Spring Boot Actuator integrates directly with Resilience4j to give full visibility into distributed failure protection:

```properties
# Enable Actuator endpoints for Circuit Breakers and Retries
management.endpoints.web.exposure.include=health,metrics,prometheus
management.endpoint.health.show-details=always
management.health.circuitbreakers.enabled=true
management.health.ratelimiters.enabled=true
```

- **Health Endpoint (`GET /manage/health`):**
  ```json
  {
    "status": "UP",
    "components": {
      "circuitBreakers": {
        "status": "UP",
        "details": {
          "paymentService": {
            "status": "CIRCUIT_OPEN",
            "failureRate": "75.0%",
            "bufferedCalls": 10,
            "failedCalls": 8
          }
        }
      }
    }
  }
  ```
- **Prometheus / Micrometer Metrics:**
  - `resilience4j.circuitbreaker.state`: 0 (Closed), 1 (Open), 2 (Half-Open).
  - `resilience4j.circuitbreaker.calls`: Count of successful, failed, or permitted calls.
  - Set up alerting when `resilience4j.circuitbreaker.state{name="paymentService"} == 1` for > 3 minutes.

---

### Master Summary: Distributed Failure Defense Matrix

| Distributed Failure Scenario | Primary Mechanism | Why It Works |
|---|---|---|
| Downstream service is slow (hanging for 30s) | **Timeout + Circuit Breaker** | Fails fast in 2s; opens circuit to stop sending traffic and prevent upstream thread exhaustion. |
| Brief network glitch or pod restart | **Retry with Exponential Backoff + Jitter** | Recovers automatically from transient blips without thundering herd storm. |
| Non-critical service (e.g. reviews) dying | **Bulkhead + Fallback** | Isolates worker threads; returns cached/empty reviews while checkout stays 100% operational. |
| Flash sale traffic spike / DDoS | **Rate Limiter / Shed Load** | Rejects excess calls with `429 Too Many Requests` to keep the database and core system responsive. |
| Multi-service transaction failure | **Saga Pattern + Compensating Actions** | Automatically triggers refunds and inventory rollbacks to preserve eventual consistency. |
| Inconsistent DB write vs Message broker publish | **Transactional Outbox Pattern** | Guarantees atomic DB write + event publishing via single local ACID transaction + CDC. |
| Client retrying an uncertain HTTP payment call | **Idempotency Key (Redis/DB lock)** | Ensures operations execute exactly once, ignoring duplicate retry requests. |
| Malformed/unparseable message in Kafka/RabbitMQ | **Dead Letter Queue (DLQ)** | Drops poison message to DLQ after 3 retries, preventing consumer partition lag from freezing. |


