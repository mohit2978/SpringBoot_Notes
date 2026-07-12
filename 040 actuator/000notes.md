# Spring Boot Actuator

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
