# REST APIs, Validations & Request Lifecycle in Spring Boot

## Table of Contents

1. [Validations in Spring Boot](#1-validations-in-spring-boot)
2. [Why aren't frontend validations enough?](#2-why-arent-frontend-validations-enough)
3. [How an API request travels through Spring Boot](#3-how-an-api-request-travels-through-spring-boot-complete-lifecycle)
4. [What makes an API "production-ready"?](#4-what-makes-an-api-production-ready)
5. [Complete production-ready implementation](#5-complete-production-ready-spring-boot-implementation)
6. [How APIs are tested before production](#6-how-apis-are-tested-before-production)
7. [How to test an API for a specific user](#7-how-to-test-an-api-for-a-specific-user)
8. [How to load test an API](#8-how-to-load-test-an-api)
9. [How API testing is done in production](#9-how-api-testing-is-done-in-production)
10. [Interview Questions](#10-interview-questions)

---

## 1. Validations in Spring Boot

In any enterprise web application, never trust user input. Validation is the first line of defense to ensure data conforms to required business rules, formats, and integrity constraints before entering the application core.

### 1.1 Maven Dependency

Spring Boot provides validation through the **Jakarta Bean Validation (JSR-380)** specification, implemented by **Hibernate Validator**:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

> **Note:** Prior to Spring Boot 2.3, validation was bundled inside `spring-boot-starter-web`. Since Spring Boot 2.3+, it must be added explicitly.

---

### 1.2 Core Built-in Validation Annotations

| Annotation | Description | Applicable Types | Example |
|---|---|---|---|
| `@NotNull` | Value cannot be `null`. (Empty strings `""` or whitespace `"   "` are allowed). | Any Object | `@NotNull Long id;` |
| `@NotEmpty` | Value cannot be `null` and length/size must be `> 0`. (Whitespace `" "` is allowed). | `CharSequence`, `Collection`, `Map`, Array | `@NotEmpty List<String> tags;` |
| `@NotBlank` | Value cannot be `null` and trimmed length must be `> 0`. (No empty string, no whitespace). | `CharSequence` / `String` | `@NotBlank String name;` |
| `@Size(min, max)` | Size/length must be between `min` and `max` (inclusive). | `String`, `Collection`, `Map`, Array | `@Size(min = 3, max = 50) String title;` |
| `@Min(value)` | Number must be greater than or equal to `value`. | Numeric types, `BigDecimal`, `BigInteger` | `@Min(18) int age;` |
| `@Max(value)` | Number must be less than or equal to `value`. | Numeric types, `BigDecimal`, `BigInteger` | `@Max(100) int percentage;` |
| `@Positive` | Number must be strictly `> 0`. | Numeric types | `@Positive BigDecimal price;` |
| `@PositiveOrZero` | Number must be `>= 0`. | Numeric types | `@PositiveOrZero int stock;` |
| `@Negative` | Number must be strictly `< 0`. | Numeric types | `@Negative int temperature;` |
| `@Email` | Must be a well-formed email address. | `CharSequence` | `@Email String email;` |
| `@Pattern(regexp = ...)` | Must match the specified regular expression. | `CharSequence` | `@Pattern(regexp = "^[0-9]{10}$") String phone;` |
| `@Past` | Date/time must be strictly in the past. | Date, Time, `LocalDate`, `Instant`, etc. | `@Past LocalDate dateOfBirth;` |
| `@Future` | Date/time must be strictly in the future. | Date, Time, `LocalDate`, `Instant`, etc. | `@Future LocalDateTime eventDate;` |
| `@AssertTrue` | Boolean must be `true`. | `boolean`, `Boolean` | `@AssertTrue boolean termsAccepted;` |
| `@AssertFalse` | Boolean must be `false`. | `boolean`, `Boolean` | `@AssertFalse boolean isDeleted;` |

#### Interview Gold: `@NotNull` vs `@NotEmpty` vs `@NotBlank`

![svg](<svgs/02-notnull-notempty-notblank.svg>)

| Input Value | `@NotNull` | `@NotEmpty` | `@NotBlank` |
|---|:---:|:---:|:---:|
| `null` | ❌ Invalid | ❌ Invalid | ❌ Invalid |
| `""` (empty string) | ✅ Valid | ❌ Invalid | ❌ Invalid |
| `"   "` (whitespace only) | ✅ Valid | ✅ Valid | ❌ Invalid |
| `"John"` (valid string) | ✅ Valid | ✅ Valid | ✅ Valid |

> **Rule of Thumb:** Always use `@NotBlank` for String text fields (names, emails, titles) to prevent strings containing only spaces.

---

### 1.3 How to Trigger Validation in Controllers

#### 1. Validating Request Body (`@Valid` with `@RequestBody`)
Put `@Valid` before the `@RequestBody` argument. If any field fails validation, Spring automatically aborts method execution and throws `MethodArgumentNotValidException`:

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        // Only reached if all validation checks pass!
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

#### 2. Validating Path Variables and Query Parameters (`@Validated`)
For validating parameters directly on controller methods (`@PathVariable`, `@RequestParam`), annotate the **controller class** with `@Validated`. If a parameter fails validation, Spring throws `ConstraintViolationException`.

> **Spring Framework 6.1+ / Spring Boot 3.2+:** Spring MVC has **built-in method validation**. If a controller method has constraint annotations directly on its parameters (`@Min`, `@NotBlank`…), they're validated even **without** class-level `@Validated`, and a failure throws **`HandlerMethodValidationException`** (→ 400). With class-level `@Validated`, the older AOP-proxy validation runs instead and throws `ConstraintViolationException`. Whichever style you use, handle the matching exception in your `@RestControllerAdvice` (see 5.5).

```java
@RestController
@RequestMapping("/api/v1/products")
@Validated // Required at class-level for method parameter validation!
public class ProductController {

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProductById(
            @PathVariable @Min(value = 1, message = "Product ID must be greater than 0") Long id) {
        return ResponseEntity.ok(productService.findById(id));
    }

    @GetMapping
    public ResponseEntity<List<ProductDto>> searchProducts(
            @RequestParam @NotBlank @Size(min = 2, max = 50) String keyword,
            @RequestParam(defaultValue = "1") @Min(1) int page,
            @RequestParam(defaultValue = "20") @Max(100) int size) {
        return ResponseEntity.ok(productService.search(keyword, page, size));
    }
}
```

#### 3. Validating Nested Objects & Collections
To validate objects inside objects or lists, you **must** place `@Valid` on the nested field:

```java
public class OrderRequest {

    @NotNull(message = "Customer ID is required")
    private Long customerId;

    @Valid // REQUIRED: tells validator to inspect fields inside ShippingAddress!
    @NotNull(message = "Shipping address is required")
    private ShippingAddress address;

    @Valid // REQUIRED: tells validator to inspect each item in the list!
    @NotEmpty(message = "Order must contain at least one item")
    private List<OrderItemRequest> items;
}
```

---

### 1.4 `@Valid` vs `@Validated`

| Feature | `@Valid` | `@Validated` |
|---|---|---|
| **Origin** | Standard Java / Jakarta (`jakarta.validation.Valid`) | Spring Framework (`org.springframework.validation.annotation.Validated`) |
| **Scope** | Can be placed on fields, methods, constructors, parameters | Can be placed on classes, methods, parameters |
| **Validation Groups** | Does **NOT** support groups | **Supports validation groups** (`@Validated(OnCreate.class)`) |
| **Method Parameter Validation** | Does not trigger AOP validation on `@PathVariable` / `@RequestParam` (Spring 6.1+ validates constrained parameters natively anyway) | Triggers AOP validation on method arguments when on the class (`ConstraintViolationException`) |
| **Cascading** | Marks nested fields / list elements for validation (`@Valid private Address address;`) | Not used for cascading |

---

### 1.5 Validation Groups

Validation Groups allow applying different validation rules to the same DTO depending on the operation (e.g. creating vs updating).

![svg](<svgs/05-validation-groups.svg>)

**1. Define Marker Interfaces:**
```java
public interface OnCreate {}
public interface OnUpdate {}
```

**2. Annotate DTO Fields with Groups:**
```java
public class UserDto {

    @Null(groups = OnCreate.class, message = "ID must be null when creating")
    @NotNull(groups = OnUpdate.class, message = "ID is required when updating")
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class}, message = "Name is required")
    private String name;

    @NotBlank(groups = OnCreate.class, message = "Password is required on registration")
    @Null(groups = OnUpdate.class, message = "Use separate endpoint to update password")
    private String password;
}
```

**3. Specify Group in Controller with `@Validated`:**
```java
@PostMapping
public ResponseEntity<UserDto> create(@Validated(OnCreate.class) @RequestBody UserDto dto) {
    return ResponseEntity.ok(userService.create(dto));
}

@PutMapping("/{id}")
public ResponseEntity<UserDto> update(@Validated(OnUpdate.class) @RequestBody UserDto dto) {
    return ResponseEntity.ok(userService.update(dto));
}
```

---

### 1.6 Custom Annotations and Custom Validators

When built-in annotations are not enough (e.g. validating phone numbers, checking enum values, password strength, cross-field password matching):

#### Step 1: Create the Custom Annotation
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneNumberValidator.class)
@Documented
public @interface ValidPhoneNumber {

    String message() default "Invalid phone number. Must start with '+' followed by 10 to 15 digits";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

#### Step 2: Implement `ConstraintValidator`
```java
public class PhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {

    // Regex for E.164 international phone number format: e.g. +919876543210
    private static final Pattern PHONE_PATTERN = Pattern.compile("^\\+[1-9]\\d{9,14}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isBlank()) {
            return true; // Let @NotBlank handle null/blank checks if required
        }
        return PHONE_PATTERN.matcher(value).matches();
    }
}
```

#### Step 3: Use it in DTO
```java
public class RegisterRequest {

    @NotBlank(message = "Phone number is required")
    @ValidPhoneNumber
    private String phoneNumber;
}
```

---

## 2. Why Aren't Frontend Validations Enough?

A common question in engineering interviews and design reviews:
> *"If the React/Angular/Vue frontend already checks email format, password length, and empty fields, why do we need backend validations?"*

![svg](<svgs/01-why-backend-validation.svg>)

### 1. Zero Trust & Total Client Bypass
The frontend executes on the user's client machine (browser, mobile device). Anyone can bypass client-side validation completely in seconds:
- **cURL / Postman / Insomnia:** Direct HTTP requests can be crafted without touching the frontend UI.
- **Browser DevTools:** Disabling JavaScript, modifying DOM elements, altering form inputs in the console, or tampering with network requests before dispatch.
- **Interception Proxies (Burp Suite, OWASP ZAP):** Malicious actors intercept legitimate requests, modify fields (e.g. change `price: 100` to `price: -1` or inject SQL scripts), and forward them to your server.

```
+-------------------------------------------------------------+
|                      ATTACKER VECTOR                        |
|                                                             |
|   Attacker  --->  cURL / Postman  --->  [ Bypasses UI ]     |
|                                              |              |
|                                              v              |
|                                     Spring Boot Server      |
|                                (If no backend validation:   |
|                                  DATABASE IS COMPROMISED!)  |
+-------------------------------------------------------------+
```

### 2. Diverse and Multiple Clients
Modern backend APIs rarely serve just one web frontend. A typical system serves:
- Web App (React / Angular / Vue)
- iOS App (Swift)
- Android App (Kotlin)
- Third-party Partner B2B Integrations
- Internal Admin Dashboards & Cron Workers

Relying on frontend validation requires duplicating business validation rules across 5 different codebases in 5 different languages. If one client has a bug, bad data enters the database.

### 3. Mobile "Version Drift"
When you update validation rules on the web, users get the latest JavaScript bundle immediately. But mobile users may not update their iOS/Android app for months or years. If validation is only on the frontend, users on legacy app versions can send invalid payloads indefinitely.

### 4. Database Integrity and Downstream Failure
The database is the system of record. If invalid data slips through:
- Corrupted database state causes foreign key conflicts, negative inventory, or calculation failures.
- Batch processing jobs, billing calculations, and analytics pipelines will crash unexpectedly.

### Summary: Complementary Responsibilities

| Dimension | Frontend Validation | Backend Validation |
|---|---|---|
| **Primary Goal** | **User Experience (UX)** | **Security & Data Integrity** |
| **Speed** | Instant visual feedback (no network round-trip) | Network round-trip required |
| **Trust Level** | **Untrusted** (can be bypassed) | **Authoritative & Enforced** |
| **Role** | Friendly guidance for legitimate users | Impenetrable barrier against invalid/malicious input |

---

## 3. How an API Request Travels Through Spring Boot (Complete Lifecycle)

When a client sends an HTTP request (e.g., `POST /api/v1/orders`), it travels through a well-defined sequence of architectural layers before reaching business code and returning.

### 3.1 Architectural Flow Diagram

![svg](<svgs/03-api-request-lifecycle.svg>)

<details>
<summary>Detailed sequence diagram (Mermaid)</summary>

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant OS as Network / Embedded Tomcat
    participant Filter as Filter Chain (Security, CORS, Logging)
    participant DS as DispatcherServlet (Front Controller)
    participant HM as HandlerMapping
    participant Interceptor as HandlerInterceptor (preHandle)
    participant HA as HandlerAdapter (ArgumentResolvers & MessageConverter)
    participant Val as Bean Validator (Hibernate Validator)
    participant Ctrl as @RestController
    participant Svc as @Service Layer
    participant Repo as @Repository / DB
    participant EH as @RestControllerAdvice (Exception Handler)

    Client->>OS: TCP Request (POST /api/v1/orders)
    OS->>Filter: HttpServletRequest
    note over Filter: Spring Security, CORS, MDC Trace ID
    Filter->>DS: Process Request

    DS->>HM: Find Handler for POST /api/v1/orders
    HM-->>DS: Returns HandlerExecutionChain (Controller + Interceptors)

    DS->>Interceptor: preHandle()
    alt Interceptor returns false
        Interceptor-->>Client: Request Aborted (e.g., 401/403)
    end

    DS->>HA: handle(request, response, handler)
    HA->>HA: HttpMessageConverter (Jackson JSON -> DTO)
    HA->>Val: Validate DTO (@Valid)

    alt Validation Fails
        Val-->>DS: Throws MethodArgumentNotValidException
        DS->>EH: Handle Exception via ExceptionHandlerExceptionResolver
        EH-->>DS: Formats Error Response (400 Bad Request)
        DS-->>Filter: Error Response
        Filter-->>Client: Return JSON Error Response
    else Validation Passes
        HA->>Ctrl: Invoke Controller Method
        Ctrl->>Svc: Call Business Logic
        Svc->>Repo: Query / Persist Entity
        Repo-->>Svc: Entity Result
        Svc-->>Ctrl: DTO Result
        Ctrl-->>HA: Return ResponseEntity<T>
        HA->>HA: HttpMessageConverter (Jackson DTO -> JSON)
        HA-->>DS: ModelAndView / Resolved Response
        DS->>Interceptor: postHandle() & afterCompletion()
        DS-->>Filter: Outgoing HttpServletResponse
        Filter-->>OS: Response Headers & Body
        OS-->>Client: HTTP 201 Created (JSON Response)
    end
```

</details>

---

### 3.2 Step-by-Step Breakdown of Every Layer

#### Layer 1: Network & Embedded Web Server (Tomcat / Jetty / Undertow)
1. The client establishes a TCP connection to port `8080`.
2. The web server's acceptor thread accepts the connection and hands it to a worker thread (`http-nio-8080-exec-*`).
3. The server parses raw HTTP bytes (method, URI, headers, body) into standard Servlet API objects: `HttpServletRequest` and `HttpServletResponse`.

#### Layer 2: Servlet Filter Chain (`jakarta.servlet.Filter` / `OncePerRequestFilter`)
Before reaching Spring MVC, the request passes through the **Servlet Filter Chain**:
- **Logging / Tracing Filter:** Generates a unique `X-Correlation-ID` or `traceId` and puts it into SLF4J's `MDC` (Mapped Diagnostic Context).
- **CORS Filter:** Checks `Origin` headers against allowed origins.
- **Spring Security Filter Chain (`SecurityFilterChain`):** Authenticates JWT tokens, parses session cookies, extracts user roles, and populates `SecurityContextHolder`. If unauthenticated/unauthorized, it halts here and returns `401 Unauthorized` or `403 Forbidden`.

#### Layer 3: DispatcherServlet (The Front Controller)
- The central brain of Spring MVC.
- Every HTTP request mapped to `/` is received by `DispatcherServlet.doDispatch(request, response)`.

#### Layer 4: HandlerMapping
- `DispatcherServlet` queries registered `HandlerMapping` implementations (primarily `RequestMappingHandlerMapping`).
- Evaluates HTTP method (`GET`, `POST`), URL path (`/api/v1/orders`), headers, and consumes/produces clauses to find the matching `@RestController` method.
- Returns a **`HandlerExecutionChain`**, which packages:
  - The target controller method (`HandlerMethod`).
  - All matching `HandlerInterceptor` instances.

#### Layer 5: HandlerInterceptor (`preHandle`)
- Executes pre-processing hooks (`preHandle()`) on registered interceptors (e.g. rate limiters, session verification, custom performance timers).
- If any interceptor returns `false`, execution immediately aborts and does not proceed to the controller.

#### Layer 6: HandlerAdapter & Argument Resolution
- `DispatcherServlet` calls `RequestMappingHandlerAdapter.handle()`.
- **Argument Resolvers (`HandlerMethodArgumentResolver`):** Resolves method arguments:
  - `@PathVariable` $\rightarrow$ `PathVariableMethodArgumentResolver`
  - `@RequestParam` $\rightarrow$ `RequestParamMethodArgumentResolver`
  - `@RequestBody` $\rightarrow$ `RequestResponseBodyMethodProcessor`
- **HttpMessageConverter (Jackson):** Reads the incoming JSON stream from `HttpServletRequest.getInputStream()` and deserializes it into the Java DTO.

#### Layer 7: Validation Engine (Hibernate Validator)
- If `@Valid` or `@Validated` is present on the argument, Spring invokes the `Validator`.
- It executes constraint checks against all annotations (`@NotBlank`, `@Min`, `@ValidPhoneNumber`, etc.).
- **If validation fails:** It does **NOT** call the controller method. Instead, it throws **`MethodArgumentNotValidException`**, which contains a `BindingResult` listing all violated fields and error messages.

#### Layer 8: Controller (`@RestController`)
- The controller method executes cleanly because the incoming DTO is guaranteed to be syntactically and structurally valid.
- The controller orchestrates request handling and delegates business processing to the Service layer.

#### Layer 9: Service Layer (`@Service`)
- Contains pure business logic, calculations, domain validation, and transaction boundaries (`@Transactional`).
- Coordinates multiple repositories or external integrations.

#### Layer 10: Repository Layer (`@Repository`)
- Uses Spring Data JPA / Hibernate to execute SQL queries or updates against the Database.

#### Layer 11: Return Path & Serialization
1. The Controller returns a `ResponseEntity<T>` or a raw DTO.
2. **`ResponseBodyAdvice` / `HttpMessageConverter`:** Jackson serializes the Java response object into JSON and sets `Content-Type: application/json`.
3. **`HandlerInterceptor.postHandle()`:** Runs after controller execution (skipped if an exception occurred).
4. **`HandlerInterceptor.afterCompletion()`:** Always runs, ideal for cleanup (e.g., stopping request timers).
5. The response bubbles back through the Servlet Filters (which can add security headers like `X-Content-Type-Options: nosniff`).
6. Tomcat writes the HTTP response bytes back to the socket.

---

### 3.3 What Happens When an Exception Occurs?

![svg](<svgs/04-exception-resolution-flow.svg>)

<details>
<summary>Flowchart (Mermaid)</summary>

```mermaid
flowchart TD
    A[Exception Thrown in Controller / Service / DB / Validation] --> B[DispatcherServlet catches Throwable]
    B --> C[HandlerExceptionResolverComposite]
    C --> D{ExceptionHandlerExceptionResolver}
    D -- Found @ExceptionHandler in @RestControllerAdvice? --> E[Execute Exception Handler Method]
    E --> F[Construct Error DTO & HTTP Status]
    F --> G[Serialize to JSON via HttpMessageConverter]
    G --> H[Return Response to Client]
    D -- Not Found? --> I{ResponseStatusExceptionResolver}
    I -- Found @ResponseStatus? --> J[Apply Status & Return]
    I -- Not Found? --> K[DefaultHandlerExceptionResolver]
    K --> L[Spring Standard Error Response via /error]
```

</details>

1. **Detection:** Any uncaught exception thrown in the Controller, Service, Repository, or Validation layer bubbles up to `DispatcherServlet.doDispatch()`.
2. **Delegation:** `DispatcherServlet` invokes `HandlerExceptionResolverComposite`.
3. **`ExceptionHandlerExceptionResolver`:**
   - Scans for beans annotated with `@ControllerAdvice` or `@RestControllerAdvice`.
   - Matches the thrown exception type against methods annotated with `@ExceptionHandler(SpecificException.class)`.
4. **Execution:** The matching handler method executes, formats a custom error response object (e.g. `ApiErrorResponse`), and specifies the HTTP status (e.g., `400 Bad Request`, `404 Not Found`, `409 Conflict`).
5. **Fallbacks:** if no `@ExceptionHandler` matches, `ResponseStatusExceptionResolver` applies a `@ResponseStatus` on the exception class, and `DefaultHandlerExceptionResolver` maps Spring's own exceptions (missing parameter → 400, wrong method → 405, unsupported media type → 415…). Anything still unresolved goes to the servlet container, which forwards to Spring Boot's `/error` endpoint (`BasicErrorController`) → 500.
6. **No Stack Traces Leaked:** The internal stack trace is logged to server logs (with MDC trace ID) for engineers, but a safe, standardized error payload is sent to the client.

> **Standard error format (RFC 9457, formerly RFC 7807):** Spring 6 has `ProblemDetail`. Return it from your handlers, or set `spring.mvc.problemdetails.enabled=true` so Spring's built-in errors use it too:
> ```json
> { "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "Invalid request content.", "instance": "/api/v1/users" }
> ```

---

## 4. What Makes an API "Production-Ready"?

Writing an API that works on `localhost:8080` is easy; writing an API that survives 10,000 requests per second in production with zero downtime requires meeting enterprise production standards.

### The 10 Pillars of Production-Ready APIs

| Pillar | Requirement | How It's Implemented in Spring Boot |
|---|---|---|
| **1. Strict Validation** | Reject invalid requests at the boundary; provide clear, field-level error messages. | `spring-boot-starter-validation`, `@Valid`, `@NotBlank`, custom validators. |
| **2. Standardized Error Contract** | Uniform error structure across all endpoints. Never leak Java stack traces to clients! | `@RestControllerAdvice`, RFC 7807 `ProblemDetail` or custom `ApiErrorResponse`. |
| **3. Security & Auth** | Authentication, Role-Based Access Control (RBAC), CORS, CSRF, security headers. | Spring Security (`SecurityFilterChain`, `@PreAuthorize("hasRole('ADMIN')")`). |
| **4. Observability & Tracing** | Every log line must correlate to a single request; monitor JVM and traffic metrics. | SLF4J `MDC` with `X-Correlation-ID`, Spring Boot Actuator, Micrometer. |
| **5. Pagination & Limits** | **Never** return unbounded lists (`List<User>`); enforce maximum page size. | `Pageable`, `Page<T>`, `Slice<T>`, `@PageableDefault(size = 20)`. |
| **6. Proper HTTP Semantics** | Accurate HTTP verbs and status codes (`200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`, `422`, `500`). | `ResponseEntity.status(HttpStatus.CREATED).body(...)`. |
| **7. API Versioning** | Allow APIs to evolve without breaking existing mobile or web clients. | URI path versioning: `/api/v1/users`, `/api/v2/users`. |
| **8. Documentation** | Interactive, auto-generated API specifications for consumers. | `springdoc-openapi-starter-webmvc-ui` (OpenAPI 3 / Swagger UI). |
| **9. Resiliency & Timeouts** | Prevent cascading failures when downstream databases or microservices slow down. | HikariCP pool limits, HTTP client connection timeouts, Resilience4j Circuit Breakers. |
| **10. Graceful Shutdown** | Allow in-flight requests to complete before pods terminate during deployments. | `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`. |

---

## 5. Complete, Production-Ready Spring Boot Implementation

Below is a complete, real-world reference implementation illustrating all concepts: **validation, custom constraints, global exception handling, correlation ID tracking, and standardized response structures.**

### 5.1 Standardized API Responses

#### Success Wrapper: `ApiResponse.java`
```java
package com.example.common.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Getter;

import java.time.Instant;

@Getter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    @Builder.Default
    private boolean success = true;

    private String message;

    private T data;

    @Builder.Default
    private Instant timestamp = Instant.now();

    public static <T> ApiResponse<T> ok(T data, String message) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .build();
    }

    public static <T> ApiResponse<T> created(T data, String message) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .build();
    }
}
```

#### Error Contract: `ApiErrorResponse.java`
```java
package com.example.common.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Getter;

import java.time.Instant;
import java.util.Map;

@Getter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiErrorResponse {

    private boolean success;
    private int status;
    private String error;
    private String message;
    private String path;
    private String traceId;
    private Instant timestamp;
    private Map<String, String> validationErrors; // Field -> Error Message
}
```

---

### 5.2 Custom Validation Annotation & Validator

#### Custom Annotation: `@PasswordMatches.java`
```java
package com.example.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchesValidator.class)
@Documented
public @interface PasswordMatches {

    String message() default "Passwords do not match";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

#### Custom Validator: `PasswordMatchesValidator.java`
```java
package com.example.validation;

import com.example.dto.CreateUserRequest;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class PasswordMatchesValidator implements ConstraintValidator<PasswordMatches, CreateUserRequest> {

    @Override
    public boolean isValid(CreateUserRequest request, ConstraintValidatorContext context) {
        if (request.getPassword() == null || request.getConfirmPassword() == null) {
            return false;
        }

        boolean matches = request.getPassword().equals(request.getConfirmPassword());
        if (!matches) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("Confirm password must match password")
                   .addPropertyNode("confirmPassword")
                   .addConstraintViolation();
        }
        return matches;
    }
}
```

---

### 5.3 Validated Request DTO: `CreateUserRequest.java`

```java
package com.example.dto;

import com.example.validation.PasswordMatches;
import jakarta.validation.constraints.*;
import lombok.Data;

@Data
@PasswordMatches // Custom class-level validation
public class CreateUserRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 30, message = "Username must be between 3 and 30 characters")
    @Pattern(regexp = "^[a-zA-Z0-9._-]+$", message = "Username can only contain alphanumeric characters, dots, and underscores")
    private String username;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be a valid email address")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be at least 8 characters long")
    @Pattern(
        regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=!]).*$",
        message = "Password must contain at least one digit, one uppercase letter, one lowercase letter, and one special character"
    )
    private String password;

    @NotBlank(message = "Confirm password is required")
    private String confirmPassword;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "User must be at least 18 years old")
    @Max(value = 120, message = "Age cannot exceed 120")
    private Integer age;
}
```

---

### 5.4 Correlation ID / Tracing Filter: `CorrelationIdFilter.java`

Ensures every incoming request gets an `X-Correlation-ID` attached to logs via SLF4J MDC:

```java
package com.example.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter implements Filter {

    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    public static final String MDC_KEY = "traceId";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String correlationId = httpRequest.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put(MDC_KEY, correlationId);
        httpResponse.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

---

### 5.5 Global Exception Handler: `GlobalExceptionHandler.java`

Handles validation errors, business exceptions, and unexpected crashes gracefully:

```java
package com.example.exception;

import com.example.common.response.ApiErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.HandlerMethodValidationException;
import jakarta.validation.ConstraintViolationException;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 1. Handles @Valid / @Validated DTO validation failures (HTTP 400)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiErrorResponse> handleValidationException(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> errors = new HashMap<>();
        for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {
            errors.put(fieldError.getField(), fieldError.getDefaultMessage());
        }

        log.warn("Validation failed for path [{}]: {}", request.getRequestURI(), errors);

        ApiErrorResponse response = ApiErrorResponse.builder()
                .success(false)
                .status(HttpStatus.BAD_REQUEST.value())
                .error(HttpStatus.BAD_REQUEST.getReasonPhrase())
                .message("Input validation failed")
                .path(request.getRequestURI())
                .traceId(MDC.get("traceId"))
                .timestamp(Instant.now())
                .validationErrors(errors)
                .build();

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    // 2. Handles custom business exceptions, e.g. ResourceNotFoundException (HTTP 404)
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {

        log.warn("Resource not found: {}", ex.getMessage());

        ApiErrorResponse response = ApiErrorResponse.builder()
                .success(false)
                .status(HttpStatus.NOT_FOUND.value())
                .error(HttpStatus.NOT_FOUND.getReasonPhrase())
                .message(ex.getMessage())
                .path(request.getRequestURI())
                .traceId(MDC.get("traceId"))
                .timestamp(Instant.now())
                .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }

    // 3. @PathVariable / @RequestParam validation failures (HTTP 400)
    //    HandlerMethodValidationException → Spring 6.1+ built-in method validation
    //    ConstraintViolationException     → class-level @Validated (AOP) validation
    @ExceptionHandler({HandlerMethodValidationException.class, ConstraintViolationException.class})
    public ResponseEntity<ApiErrorResponse> handleParameterValidation(
            Exception ex, HttpServletRequest request) {

        log.warn("Parameter validation failed for path [{}]: {}", request.getRequestURI(), ex.getMessage());

        ApiErrorResponse response = ApiErrorResponse.builder()
                .success(false)
                .status(HttpStatus.BAD_REQUEST.value())
                .error(HttpStatus.BAD_REQUEST.getReasonPhrase())
                .message("Invalid request parameters")
                .path(request.getRequestURI())
                .traceId(MDC.get("traceId"))
                .timestamp(Instant.now())
                .build();

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    // 4. Fallback for unhandled 500 errors (never leak stack trace to client!)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiErrorResponse> handleGlobalException(
            Exception ex, HttpServletRequest request) {

        log.error("Unhandled internal server error on [{}]: ", request.getRequestURI(), ex);

        ApiErrorResponse response = ApiErrorResponse.builder()
                .success(false)
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .error(HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase())
                .message("An unexpected error occurred. Please contact support quoting traceId: " + MDC.get("traceId"))
                .path(request.getRequestURI())
                .traceId(MDC.get("traceId"))
                .timestamp(Instant.now())
                .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}
```

---

### 5.6 Production-Ready Controller: `UserController.java`

```java
package com.example.controller;

import com.example.common.response.ApiResponse;
import com.example.dto.CreateUserRequest;
import com.example.dto.UserResponse;
import com.example.service.UserService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
@Tag(name = "User Management", description = "Endpoints for managing users")
public class UserController {

    private final UserService userService;

    @PostMapping
    @Operation(summary = "Register a new user")
    public ResponseEntity<ApiResponse<UserResponse>> createUser(
            @Valid @RequestBody CreateUserRequest request) {

        UserResponse createdUser = userService.createUser(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.created(createdUser, "User registered successfully"));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get user by ID")
    public ResponseEntity<ApiResponse<UserResponse>> getUserById(
            @PathVariable @Min(value = 1, message = "ID must be a positive number") Long id) {

        UserResponse user = userService.getUserById(id);
        return ResponseEntity.ok(ApiResponse.ok(user, "User fetched successfully"));
    }

    @GetMapping
    @Operation(summary = "Get paginated users list")
    public ResponseEntity<ApiResponse<Page<UserResponse>>> getAllUsers(
            @PageableDefault(size = 20, sort = "createdAt") Pageable pageable) {

        Page<UserResponse> users = userService.getAllUsers(pageable);
        return ResponseEntity.ok(ApiResponse.ok(users, "Users retrieved successfully"));
    }
}
```

---

### 5.7 Sample Client Request and Validation Response

#### Bad Request Payload:
```json
POST /api/v1/users
Content-Type: application/json

{
  "username": "a",
  "email": "not-an-email",
  "password": "secret",
  "confirmPassword": "mismatch",
  "age": 15
}
```

#### Production-Grade Standardized Error Response (`400 Bad Request`):
```json
{
  "success": false,
  "status": 400,
  "error": "Bad Request",
  "message": "Input validation failed",
  "path": "/api/v1/users",
  "traceId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "timestamp": "2026-10-09T03:55:00.123Z",
  "validationErrors": {
    "username": "Username must be between 3 and 30 characters",
    "email": "Email must be a valid email address",
    "password": "Password must contain at least one digit, one uppercase letter, one lowercase letter, and one special character",
    "confirmPassword": "Confirm password must match password",
    "age": "User must be at least 18 years old"
  }
}
```

---

## 6. How APIs Are Tested Before Production

![svg](<svgs/06-testing-pyramid.svg>)

| Level | What it checks | Spring Boot tools | Speed |
|---|---|---|---|
| **Unit** | One class in isolation (service logic, validators) | JUnit 5, Mockito, AssertJ | ms |
| **Slice** | One layer with real Spring wiring: controller + validation + JSON, or repository + DB | `@WebMvcTest` + `MockMvc`, `@DataJpaTest` | ~1 s |
| **Integration** | The whole app with real infrastructure | `@SpringBootTest(webEnvironment = RANDOM_PORT)`, `TestRestTemplate` / `WebTestClient`, **Testcontainers** (real Postgres/Kafka in Docker) | seconds |
| **Contract** | Producer and consumer agree on the API shape | Spring Cloud Contract, Pact | seconds |
| **End-to-end** | A real user journey across services/UI | Postman/Newman, Playwright, REST Assured against staging | minutes |

### 6.1 Testing the validation of this API with MockMvc

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean                     // Spring Boot 3.4+ (older versions: @MockBean)
    private UserService userService;

    @Test
    @WithMockUser                    // needed if Spring Security is on the classpath
    void createUser_withInvalidBody_returns400WithFieldErrors() throws Exception {
        String body = """
            { "username": "a", "email": "not-an-email", "password": "secret",
              "confirmPassword": "mismatch", "age": 15 }
            """;

        mockMvc.perform(post("/api/v1/users")
                        .with(csrf())                         // POST with Spring Security's CSRF on
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.validationErrors.email").value("Email must be a valid email address"))
                .andExpect(jsonPath("$.validationErrors.age").value("User must be at least 18 years old"));

        verifyNoInteractions(userService);    // controller logic was never reached
    }
}
```

### 6.2 Integration test against a real database (Testcontainers)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserApiIntegrationTest {

    @Container
    @ServiceConnection                                  // Boot 3.1+: wires spring.datasource.* automatically
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired
    private TestRestTemplate rest;

    @Test
    void createAndFetchUser() {
        var request = Map.of("username", "mohit", "email", "mohit@example.com",
                "password", "Secret@123", "confirmPassword", "Secret@123", "age", 30);

        ResponseEntity<String> created = rest.postForEntity("/api/v1/users", request, String.class);

        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

---

## 7. How to Test an API for a Specific User

"Test for a specific user" usually means one of three things:
1. **Authorization:** does user A get *their own* data, and get 403 for user B's data?
2. **Reproducing a bug** that only one user reports.
3. **Rolling out** a change to one user (or a few) first.

![svg](<svgs/07-test-api-as-a-specific-user.svg>)

### 7.1 In automated tests (spring-security-test)

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// a) Fake logged-in user with given roles (no real login needed)
@Test
@WithMockUser(username = "user-42", roles = "USER")
void userSeesOwnOrders() throws Exception {
    mockMvc.perform(get("/api/v1/orders/mine"))
           .andExpect(status().isOk());
}

// b) Load a REAL user from your UserDetailsService (their real roles/authorities)
@Test
@WithUserDetails("mohit@example.com")
void adminCanListAllUsers() throws Exception {
    mockMvc.perform(get("/api/v1/users")).andExpect(status().isOk());
}

// c) JWT-secured API: build a token for a specific subject + authorities
@Test
void userCannotReadSomeoneElsesOrder() throws Exception {
    mockMvc.perform(get("/api/v1/orders/7")                        // order 7 belongs to user-99
                    .with(jwt().jwt(j -> j.subject("user-42"))
                               .authorities(new SimpleGrantedAuthority("ROLE_USER"))))
           .andExpect(status().isForbidden());                     // ownership check must reject it
}
```

> Test (c) is the most important one: **IDOR (Insecure Direct Object Reference)** bugs, where changing an id in the URL returns another user's data, are among the most common API vulnerabilities. Always test "user A asks for user B's resource → 403/404".

### 7.2 Manually, on dev / staging

Log in as a **test user**, take its token, and call the API with it:

```bash
TOKEN=$(curl -s -X POST https://staging.example.com/auth/login \
          -H "Content-Type: application/json" \
          -d '{"username":"test-user-42","password":"<test-password>"}' | jq -r .accessToken)

curl -s https://staging.example.com/api/v1/orders/mine \
     -H "Authorization: Bearer $TOKEN" | jq
```

- In **Postman**, create an **environment per test user** (`user42`, `admin`) holding its credentials. A pre-request script or the collection's auth settings fetches the token, so switching users is one dropdown.
- Keep a few seeded test users per role (`USER`, `ADMIN`, `SUPPORT`), plus users with edge-case data (no orders, 10k orders, suspended account).

### 7.3 In production (safely)

You **never** log in with a real customer's password. Instead:

- **Find their requests in logs and traces:** put the user id into the MDC after authentication, so every log line carries it:
  ```java
  @Component
  public class UserIdMdcFilter extends OncePerRequestFilter {
      @Override
      protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
              throws ServletException, IOException {
          Authentication auth = SecurityContextHolder.getContext().getAuthentication();
          if (auth != null && auth.isAuthenticated()) {
              MDC.put("userId", auth.getName());
          }
          try {
              chain.doFilter(req, res);
          } finally {
              MDC.remove("userId");
          }
      }
  }
  ```
  (Register it **after** the Spring Security filter chain so the user is known, e.g. with `http.addFilterAfter(new UserIdMdcFilter(), AuthorizationFilter.class)` instead of `@Component`.) Then search `userId:42 AND status:500` in Kibana/Loki and open the `traceId` in Zipkin/Tempo to see exactly what happened.
- **Feature flag targeted at a user id:** enable the new code path only for that user, an internal staff account, or a beta group:
  ```java
  if (featureFlags.isEnabled("new-checkout", userId)) { return newCheckout(cart); }
  return oldCheckout(cart);
  ```
- **Impersonation ("act as user")** for support staff: an admin-only, fully **audited** feature (Spring Security's `SwitchUserFilter` exists for this). It shows what the user sees without knowing their password.
- **Dedicated production test accounts:** flagged (`is_test_account = true`) so they're excluded from billing, emails and analytics.

---

## 8. How to Load Test an API

A **load test** answers: *how many requests per second can this API handle, at what latency, before errors or slowdowns start?*

![svg](<svgs/08-load-test-types.svg>)

| Type | Load pattern | Question it answers |
|---|---|---|
| **Load test** | Ramp up to the expected peak, hold | Do we meet our SLO (e.g. p95 < 300 ms, errors < 1%) at normal peak? |
| **Stress test** | Keep increasing until it breaks | What's the breaking point, and how does it fail (slowly or crash)? |
| **Spike test** | Sudden jump (10× in seconds) | Does it survive a flash sale / viral moment, and recover afterwards? |
| **Soak (endurance) test** | Normal load for hours | Memory leaks, connection-pool leaks, disk filling up over time? |

### 8.1 Step by step

1. **Define the goal / SLO:** e.g. "500 req/s on `GET /api/v1/users`, p95 < 300 ms, error rate < 1%".
2. **Use a prod-like environment** (staging with the same instance sizes, DB size and config). Load-testing a laptop tells you little.
3. **Use realistic data and traffic:** many different users (not one token for everything, because caches would make it unrealistically fast), a realistic mix of endpoints, and think time between calls.
4. **Warm up first:** the JVM's JIT compiler and the caches need a minute or two before numbers are meaningful.
5. **Ramp up gradually** while **watching the metrics** (Grafana: latency, errors, CPU, heap, GC, Hikari/Tomcat pools, DB CPU).
6. **Find the bottleneck** when latency or errors rise: thread pool exhausted? DB connections pending? slow query? GC pauses? CPU at 100%? (See the *037 actuator* notes for how to read these.)
7. **Fix, re-run, compare.** Keep the script in Git and run it in CI before big releases to catch regressions.

![svg](<svgs/09-load-test-setup.svg>)

### 8.2 Example with k6

```javascript
// load-test.js   →   k6 run -e TOKEN=$TOKEN load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up to 100 virtual users
    { duration: '5m', target: 100 },   // hold
    { duration: '1m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<800'],   // ms, the test FAILS if exceeded
    http_req_failed: ['rate<0.01'],                  // < 1% errors
  },
};

export default function () {
  const res = http.get('https://staging.example.com/api/v1/users?page=0&size=20', {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);                                            // think time
}
```

k6 prints a summary at the end. The lines to read are request rate, latency percentiles, error rate, and whether the thresholds passed:

```text
     http_req_duration..............: avg=84ms  min=12ms med=61ms max=1.2s p(90)=160ms p(95)=210ms
     http_req_failed................: 0.12%  ✓ 41   ✗ 33871
     http_reqs......................: 33912  94.2/s
     ✓ http_req_duration p(95)<300   ✓ http_req_failed rate<0.01
```

**Other tools:** **Gatling** (Java/Scala DSL, great HTML reports), **JMeter** (GUI, widely used), **Locust** (Python). For a quick check on one endpoint:

```bash
hey -n 10000 -c 100 http://localhost:8080/api/v1/users/1      # 10k requests, 100 concurrent
```

### 8.3 Reading the results

- **Throughput flattens while latency climbs** → you've hit a bottleneck (pool, DB, CPU).
- **Errors appear suddenly** → timeouts, pool exhaustion (`HikariPool - Connection is not available`), or rate limits.
- **Latency slowly grows during a soak test** → memory or connection leak.
- **Little's law** for sizing: `concurrent requests = throughput × latency`. 500 req/s × 0.2 s = 100 requests in flight, so pools and threads need at least that headroom.

> Don't load-test **production** without a plan: it competes with real users and can trigger alerts, autoscaling costs or rate limits at third parties. If you must (e.g. before a big sale), do it in a scheduled window, against isolated test data, with everyone informed.

---

## 9. How API Testing Is Done in Production

Pre-production tests can't catch everything (real traffic, real data, real infrastructure). Production testing focuses on **releasing safely and detecting problems fast**, without hurting real users.

![svg](<svgs/10-testing-in-production.svg>)

### 9.1 Smoke tests right after a deploy

A small script of **read-only** checks against the new version:

```bash
#!/usr/bin/env bash
set -e
BASE=https://api.example.com
curl -fsS $BASE/actuator/health | grep -q '"status":"UP"'
curl -fsS -H "Authorization: Bearer $SMOKE_TEST_TOKEN" "$BASE/api/v1/users/me" > /dev/null
curl -fsS "$BASE/api/v1/products?page=0&size=1" > /dev/null
echo "smoke tests passed"
```

If it fails, the pipeline rolls back automatically.

### 9.2 Canary releases and blue-green deployments

- **Canary:** send 1–5% of real traffic to the new version (Kubernetes + Argo Rollouts / Flagger, or the load balancer). Compare its error rate and p99 latency with the old version. Promote step by step (5% → 25% → 100%) or roll back automatically.
- **Blue-green:** run the new version ("green") next to the old one ("blue"), test green, then switch the load balancer. Switching back is instant.

### 9.3 Feature flags (dark launches)

Deploy the code switched **off**, then enable it for staff, a beta group, one user id, or 1% of users, all **without a new deploy**. If something breaks, flip it off. Tools: Unleash, LaunchDarkly, Togglz, Flagsmith (or a simple DB-backed flag).

### 9.4 Synthetic monitoring

Scripted "robot users" call key endpoints every minute **from outside** (Datadog Synthetics, Checkly, Grafana k6 Cloud, UptimeRobot): health, login, search, place a test order with a test account. They alert you before customers notice, including at night when there's no real traffic.

### 9.5 Shadow (mirrored) traffic

Copy real production requests to the new version (Istio / Envoy / nginx `mirror`), compare its responses with the current version, and **throw its responses away**. Great for rewrites and migrations. Make sure mirrored requests can't cause side effects (no emails, no payments, separate DB writes).

### 9.6 Observability and alerting

Production testing only works if you can *see* problems: RED metrics (Rate, Errors, Duration) per endpoint, logs with `traceId`/`userId`, distributed traces, and alerts on error rate and p99 latency (see the *037 actuator* notes and the monitoring section of *003 maven*).

### 9.7 Chaos engineering

Deliberately inject failures (kill a pod, add latency to the DB, drop a dependency) with tools like Chaos Monkey for Spring Boot, Chaos Mesh, or AWS FIS. This checks that timeouts, retries, circuit breakers and autoscaling really work. Start in staging, then move to production in small, controlled experiments.

### 9.8 Rules for testing in production

- Use **dedicated test accounts / test tenants**, flagged and excluded from billing, emails and analytics.
- Prefer **read-only** checks. Write tests must be **idempotent** and **clean up** after themselves.
- **Never** use or modify real customer data, and never use a real user's credentials.
- Rate-limit your own probes, and tell the on-call team about planned tests.
- Always have a **fast rollback** (canary auto-rollback, blue-green switch, feature flag off).

---

## 10. Interview Questions

**Q1. `@NotNull` vs `@NotEmpty` vs `@NotBlank`?**
`@NotNull` rejects only `null`. `@NotEmpty` also rejects empty strings/collections. `@NotBlank` (strings only) also rejects whitespace-only strings.

**Q2. `@Valid` vs `@Validated`?**
`@Valid` is standard Jakarta: it triggers validation of a `@RequestBody` and cascades into nested objects/lists. `@Validated` is Spring's: it supports validation groups and, on a class, enables method-level validation of parameters via AOP.

**Q3. Which exceptions are thrown when validation fails?**
`@Valid @RequestBody` → `MethodArgumentNotValidException`. Constrained `@PathVariable`/`@RequestParam` → `HandlerMethodValidationException` (Spring 6.1+ built-in) or `ConstraintViolationException` (class-level `@Validated`). All should become 400 in a `@RestControllerAdvice`.

**Q4. How do you validate nested objects and lists?**
Put `@Valid` on the nested field (`@Valid private Address address;`, `@Valid private List<Item> items;`).

**Q5. How do you write a custom validator, and a cross-field one?**
Create an annotation with `@Constraint(validatedBy = X.class)` and implement `ConstraintValidator<Ann, Type>`. For cross-field checks (password == confirmPassword), put the annotation on the class and validate the whole object.

**Q6. Why is frontend validation not enough?**
Clients can be bypassed (curl, Postman, proxies), there are many clients (web, mobile, partners), old mobile versions linger, and only the backend can protect data integrity. Frontend validation is for UX, backend validation is for security.

**Q7. Walk through the lifecycle of a request in Spring Boot.**
Tomcat thread → filters (security, CORS, logging) → `DispatcherServlet` → `HandlerMapping` → interceptors `preHandle` → argument resolvers + Jackson → validation → controller → service → repository → Jackson serializes the response → interceptors → filters → client. Exceptions go to `HandlerExceptionResolver`s / `@RestControllerAdvice`.

**Q8. What makes an API production-ready?**
Validation, a consistent error contract (no stack traces), security, observability (trace ids, metrics), pagination, correct HTTP semantics, versioning, docs (OpenAPI), resilience (timeouts, circuit breakers), and graceful shutdown.

**Q9. How do you test a controller's validation without starting the whole app?**
`@WebMvcTest(Controller.class)` with `MockMvc` and a mocked service. Send an invalid body, assert 400 and the field errors, and verify the service was never called.

**Q10. How do you test an API as a specific user?**
In tests: `@WithMockUser`, `@WithUserDetails`, or `.with(jwt().jwt(j -> j.subject("user-42")))`, including negative tests that user A can't read user B's data (IDOR). Manually: log in as a test user and call the API with its token (Postman environment per user). In production: find the user's requests by `userId`/`traceId` in logs and traces, use feature flags targeted at that user, or audited impersonation. Never use the customer's credentials.

**Q11. How do you load test an API? What do you measure?**
Define an SLO, script realistic traffic with k6/Gatling/JMeter against a prod-like environment, warm up, ramp up, and watch throughput, p95/p99 latency, error rate, CPU, heap/GC and pool usage. Find the bottleneck, fix it, re-run.

**Q12. Load vs stress vs spike vs soak test?**
Load = expected peak (meets SLO?). Stress = beyond the limit (where/how does it break?). Spike = sudden burst (survive and recover?). Soak = long duration (leaks?).

**Q13. How is API testing done in production?**
Smoke tests after deploy, canary or blue-green releases with automatic rollback, feature flags for gradual rollout, synthetic monitoring, shadow traffic, strong observability and alerting, and chaos experiments, always with test accounts and without touching real customer data.

**Q14. What's a canary release?**
Sending a small percentage of real traffic to the new version, comparing its error rate and latency with the old one, then gradually increasing the share or rolling back automatically.
