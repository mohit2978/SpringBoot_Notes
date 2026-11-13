# Filter vs Interceptor in Spring Boot

## Table of Contents

1. [In simple words](#1-in-simple-words)
2. [What is a servlet? Where does DispatcherServlet fit?](#2-what-is-a-servlet-where-does-dispatcherservlet-fit)
3. [Filters](#3-filters)
4. [Multiple filters and their ordering](#4-multiple-filters-and-their-ordering)
5. [Interceptors](#5-interceptors)
6. [Multiple interceptors and their ordering](#6-multiple-interceptors-and-their-ordering)
7. [Filters and interceptors together](#7-filters-and-interceptors-together)
8. [Filter vs Interceptor: key differences](#8-filter-vs-interceptor-key-differences)
9. [Common pitfalls](#9-common-pitfalls)
10. [Interview Questions](#10-interview-questions)

---

## 1. In simple words

- **Filter:** intercepts the HTTP request and response **before they reach any servlet** (including Spring's `DispatcherServlet`). It belongs to the **Servlet API** (`jakarta.servlet`), not to Spring.
- **Interceptor:** **specific to Spring MVC**. It intercepts the HTTP request and response **inside `DispatcherServlet`, just before they reach the controller** (and after the controller is done).

> **Analogy**
> - **Filter = security gate of the building.** It checks *everyone* before they enter.
> - **Interceptor = staff inside the building.** They check only people who have already entered and are going to a *specific room* (a controller).

![svg](<svgs/01-where-filters-and-interceptors-sit.svg>)

**One-line rule**
- Use a **Filter** when you need to work with the raw HTTP request/response **before Spring is involved** (security, CORS, encoding, compression, logging every request).
- Use an **Interceptor** when you need **Spring MVC information**: which controller/method handles the request, its annotations, the model.

---

## 2. What is a servlet? Where does DispatcherServlet fit?

A **servlet** is nothing but a Java class that accepts an incoming request, processes it, and returns a response.

We can create multiple servlets, for example:
- Servlet 1: configured to handle REST APIs
- Servlet 2: configured to handle SOAP APIs, etc.

**`DispatcherServlet`** is the servlet Spring provides. By default it is mapped to `/` and handles every request that reaches the application. It is the **front controller** of Spring MVC.

The request flow is: **Tomcat (servlet container) → filter chain → servlet (DispatcherServlet) → interceptors → controller**. Inside `DispatcherServlet` this happens:

![svg](<svgs/02-dispatcherservlet-request-flow.svg>)

1. **Choose the controller:** `HandlerMapping` finds which controller method matches the URL. It returns a `HandlerExecutionChain`, which is the handler **plus the interceptors** that apply to that URL.
2. **Create an instance:** the controller bean (with its dependencies) is created by the IoC container. This normally happens once, at startup, since controllers are singletons.
3. **Invoke the controller method:** interceptors' `preHandle()` runs first, then the method runs, then `postHandle()`.
4. **Response:** the view is rendered / JSON is written, `afterCompletion()` runs, and the response goes back through the filters to Tomcat and the client.

So:
- **Filter:** add logic that is **independent of the servlet** that will handle the request. You can have many filters and control their order.
- **Interceptor:** add logic that is **specific to a particular servlet** (Spring's `DispatcherServlet`) and its controllers. You can have many interceptors and control their order.

---

## 3. Filters

### 3.1 What is a filter?

- Part of the **Servlet API** (`jakarta.servlet.Filter`; `javax.servlet` before Spring Boot 3).
- Works at the **lowest level**: it intercepts every request **before** it reaches `DispatcherServlet`, and the response **after** it's generated.
- Typical uses: logging, authentication, CORS, encoding, compression, request IDs.

**Filter methods**

| Method | When it runs |
|---|---|
| `init(FilterConfig)` | Once, when the filter is created (startup) |
| `doFilter(request, response, chain)` | For **every request**, before **and** after the rest of the chain |
| `destroy()` | Once, when the filter is destroyed (shutdown) |

### 3.2 A filter is code AROUND `chain.doFilter()`

![svg](<svgs/03-filter-anatomy-around-chain.svg>)

- Code **before** `chain.doFilter(...)` runs on the way **in** (request).
- `chain.doFilter(...)` passes control to the next filter → servlet → interceptors → controller.
- Code **after** `chain.doFilter(...)` runs on the way **out** (response).
- If you **don't** call `chain.doFilter(...)`, the request stops in this filter. The controller is never reached.

### 3.3 Example: logging filter with timing

```java
@Component
public class LoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        System.out.println("[Filter] Before request");
        long start = System.currentTimeMillis();

        chain.doFilter(request, response);          // pass to next filter / servlet

        long end = System.currentTimeMillis();
        System.out.println("[Filter] After request - Time taken : " + (end - start) + " ms");
    }
}
```

**Registration with URL patterns and order:**

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<LoggingFilter> loggingFilter() {
        FilterRegistrationBean<LoggingFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new LoggingFilter());
        bean.addUrlPatterns("/*");   // apply to all requests
        bean.setOrder(1);            // priority (lower = runs first)
        return bean;
    }
}
```

> When you register with `FilterRegistrationBean`, **don't** also put `@Component` on the filter class. Otherwise Spring Boot registers it a second time (for `/*`) and it runs twice. Use one approach: `@Component` (applies to all URLs) **or** `FilterRegistrationBean` (URL patterns + order).

**Controller (used by every example in these notes):**

```java
@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() throws InterruptedException {
        Thread.sleep(100);   // simulate some work
        return "Hello from Controller";
    }
}
```

**Output for `GET /hello`:**

```text
[Filter] Before request
[Filter] After request - Time taken : 102 ms
```

Response: `Hello from Controller`

- The filter runs **before** the request reaches `DispatcherServlet`.
- The filter runs **after** the response is generated (even if the controller wasn't called because of an error).

### 3.4 Example: full filter anatomy (init, doFilter, destroy)

```java
@Component
public class RequestLoggingFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("[Filter] init() - once at startup");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest req = (HttpServletRequest) request;
        long start = System.currentTimeMillis();
        System.out.println("[Filter] BEFORE - " + req.getMethod() + " " + req.getRequestURI());

        chain.doFilter(request, response);   // MUST call to continue

        long duration = System.currentTimeMillis() - start;
        System.out.println("[Filter] AFTER - Time taken : " + duration + " ms");
    }

    @Override
    public void destroy() {
        System.out.println("[Filter] destroy() - once at shutdown");
    }
}
```

### 3.5 Example: a filter that blocks requests

```java
public class AuthFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse res = (HttpServletResponse) response;

        String token = req.getHeader("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            res.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Missing token"); // 401
            return;                       // chain.doFilter() NOT called → request stops here
        }
        chain.doFilter(request, response);
    }
}
```

Registering two filters with different URL patterns and order:

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<RequestLoggingFilter> loggingFilter() {
        FilterRegistrationBean<RequestLoggingFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new RequestLoggingFilter());
        bean.addUrlPatterns("/api/*");   // applies only to /api/**
        bean.setOrder(1);                // lower order = runs first
        return bean;
    }

    @Bean
    public FilterRegistrationBean<AuthFilter> authFilter() {
        FilterRegistrationBean<AuthFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new AuthFilter());
        bean.addUrlPatterns("/*");       // all URLs
        bean.setOrder(2);                // runs after the logging filter
        return bean;
    }
}
```

(Here the filter classes have no `@Component`, because they are registered through `FilterRegistrationBean`.)

### 3.6 Prefer `OncePerRequestFilter` in Spring apps

A plain `Filter` can run more than once for one request (e.g. on an error dispatch to `/error` or a forward). Spring's `OncePerRequestFilter` guarantees **one execution per request** and gives you typed `HttpServletRequest`/`HttpServletResponse`:

```java
@Component
public class RequestIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);              // appears in every log line of this request
        response.setHeader("X-Request-Id", requestId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("requestId");                  // threads are reused → always clean up
        }
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return request.getRequestURI().startsWith("/actuator");   // skip some URLs
    }
}
```

### 3.7 Three ways to register a filter

| Way | URL patterns | Order |
|---|---|---|
| `@Component` on the filter class | Always `/*` | `@Order(n)` on the class |
| `FilterRegistrationBean` in a `@Configuration` | `addUrlPatterns(...)` | `setOrder(n)` |
| `@WebFilter("/api/*")` + `@ServletComponentScan` on the main class | In the annotation | Not controllable (avoid when order matters) |

---

## 4. Multiple filters and their ordering

```java
import jakarta.servlet.*;
import java.io.IOException;

public class MyFilter1 implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        Filter.super.init(filterConfig);
    }

    @Override
    public void doFilter(ServletRequest servletRequest,
                         ServletResponse servletResponse,
                         FilterChain filterChain)
            throws IOException, ServletException {

        System.out.println("MyFilter1 inside");
        filterChain.doFilter(servletRequest, servletResponse);
        System.out.println("MyFilter1 completed");
    }

    @Override
    public void destroy() {
        Filter.super.destroy();
    }
}
```

```java
import jakarta.servlet.*;
import java.io.IOException;

public class MyFilter2 implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        Filter.super.init(filterConfig);
    }

    @Override
    public void doFilter(ServletRequest servletRequest,
                         ServletResponse servletResponse,
                         FilterChain filterChain)
            throws IOException, ServletException {

        System.out.println("MyFilter2 inside");
        filterChain.doFilter(servletRequest, servletResponse);
        System.out.println("MyFilter2 completed");
    }

    @Override
    public void destroy() {
        Filter.super.destroy();
    }
}
```

```java
@Configuration
public class AppConfig {

    @Bean
    public FilterRegistrationBean<MyFilter1> myFirstFilter() {
        FilterRegistrationBean<MyFilter1> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new MyFilter1());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setOrder(2);
        return filterRegistrationBean;
    }

    @Bean
    public FilterRegistrationBean<MyFilter2> mySecondFilter() {
        FilterRegistrationBean<MyFilter2> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new MyFilter2());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setOrder(1);
        return filterRegistrationBean;
    }
}
```

**Output.** A lower order value means higher priority, so `MyFilter2` (order 1) runs before `MyFilter1` (order 2):

```text
MyFilter2 inside
MyFilter1 inside
hitting db to get the userdata
MyFilter1 completed
MyFilter2 completed
```

![svg](<svgs/04-multiple-filters-stack-order.svg>)

### Why do filters run in reverse order on the way out?

- Filters form a **chain, like a stack**. The request goes through them in the order they were added: **first in**.
- Each filter calls `chain.doFilter()` and **waits** for it to return. The code after `chain.doFilter()` runs only when everything inside it has finished.
- So the **last** filter that saw the request is the **first** to see the response: **last out (LIFO)**.
- This keeps the "around" behavior correct (open → process → close), so each filter can clean up what it started.

**Memory trick:** filters are like a stack. **Push** on the way in, **pop** on the way out.

| Step | What happens | Direction |
|---|---|---|
| 1 | Client sends request | |
| 2 | F1.doFilter() (before chain) | ↓ forward |
| 3 | F2.doFilter() (before chain) | ↓ forward |
| 4 | F3.doFilter() (before chain) | ↓ forward |
| 5 | Controller (target resource) | |
| 6 | F3.doFilter() (after chain) | ↑ reverse |
| 7 | F2.doFilter() (after chain) | ↑ reverse |
| 8 | F1.doFilter() (after chain) | ↑ reverse |
| 9 | Response sent to client | |

---

## 5. Interceptors

### 5.1 What is an interceptor?

- Part of **Spring MVC** (`org.springframework.web.servlet.HandlerInterceptor`).
- Works **inside `DispatcherServlet`**.
- Intercepts requests **just before the controller method is called**, and again after it.
- Typical uses: logging, authentication/authorization, performance monitoring, modifying the request/response, audit.

(The previous topic, *019 custom interceptor*, shows creating one interceptor and a custom annotation in detail.)

### 5.2 The three hooks

![svg](<svgs/05-interceptor-three-hooks.svg>)

| Hook | When it runs | If it returns / what happens | Where to use |
|---|---|---|---|
| `preHandle()` | Before the controller | `true` → continue, `false` → block | Authentication, authorization, input validation, logging |
| `postHandle()` | After the controller, before view rendering (skipped if the controller throws) | Can modify `ModelAndView` (model data) | Add model attributes, modify response data (MVC views only) |
| `afterCompletion()` | After the response is completely written (always, like `finally`) | The exception (if any) is available here | Cleanup, logging the final status, releasing resources, metrics |

### 5.3 Example: logging interceptor with timing

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        System.out.println("[Interceptor] preHandle - Before controller");
        long start = System.currentTimeMillis();
        request.setAttribute("startTime", start);   // store start time for later
        return true;                                // return false to stop the request
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
                           ModelAndView modelAndView) throws Exception {
        long start = (long) request.getAttribute("startTime");
        long end = System.currentTimeMillis();
        System.out.println("[Interceptor] postHandle - After controller, Time taken : " + (end - start) + " ms");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
                                Exception ex) throws Exception {
        System.out.println("[Interceptor] afterCompletion - Request Completed");
    }
}
```

> Store per-request data in `request.setAttribute(...)`, **not** in a field. The interceptor is a singleton shared by all threads.

**Registration:**

```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {

    @Autowired
    private LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/**")                         // include all paths
                .excludePathPatterns("/login", "/public/**");   // except these
    }
}
```

**Output for `GET /hello`:**

```text
[Interceptor] preHandle - Before controller
[Interceptor] postHandle - After controller, Time taken : 101 ms
[Interceptor] afterCompletion - Request Completed
```

Response: `Hello from Controller`

- `preHandle()` runs before the controller.
- `postHandle()` runs after the controller but before the view is rendered.
- `afterCompletion()` runs after the request is completed.

### 5.4 Example: interceptor that uses Spring MVC information

This is something a filter **cannot** do: look at *which* controller method will handle the request and read its annotations.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AdminOnly { }
```

```java
@Component
public class AuditInterceptor implements HandlerInterceptor {

    // 1. BEFORE controller
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        if (!(handler instanceof HandlerMethod method)) {
            return true;                       // e.g. static resources: nothing to check
        }
        System.out.println("[Interceptor] preHandle - Before controller : "
                + method.getBeanType().getSimpleName() + "." + method.getMethod().getName());

        // check custom annotation on the controller method
        if (method.getMethodAnnotation(AdminOnly.class) != null && !isAdmin(request)) {
            response.sendError(403, "Access Denied");
            return false;                      // block: controller is not called
        }
        return true;                           // continue
    }

    // 2. AFTER controller, BEFORE view rendering
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
                           ModelAndView modelAndView) throws Exception {
        System.out.println("[Interceptor] postHandle - Before view");
        if (modelAndView != null) {            // null for @RestController / @ResponseBody
            modelAndView.addObject("appVersion", "1.0.5");
        }
    }

    // 3. AFTER everything (even if an exception occurred)
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
                                Exception ex) throws Exception {
        if (ex != null) {
            System.out.println("[Interceptor] afterCompletion - EXCEPTION : " + ex.getMessage());
        } else {
            System.out.println("[Interceptor] afterCompletion - Request Completed");
        }
    }

    private boolean isAdmin(HttpServletRequest request) {
        return "ADMIN".equals(request.getHeader("X-Role"));   // demo only
    }
}
```

```java
@RestController
@RequestMapping("/api")
public class AdminController {

    @AdminOnly
    @DeleteMapping("/users/{id}")
    public String deleteUser(@PathVariable Long id) {
        return "deleted " + id;
    }
}
```

- `DELETE /api/users/5` without `X-Role: ADMIN` → `403 Access Denied`; the controller never runs.
- With `X-Role: ADMIN` → `deleted 5`.

> `ex` in `afterCompletion()` is only set for exceptions that were **not** handled. If a `@ControllerAdvice` / `@ExceptionHandler` already handled the exception, `ex` is `null`.

---

## 6. Multiple interceptors and their ordering

The previous topic showed how to add one interceptor and how `preHandle`, `postHandle` and `afterCompletion` come into the picture. Here is how to add more than one.

```java
@Configuration
public class AppConfig implements WebMvcConfigurer {

    @Autowired
    MyCustomInterceptor1 myCustomInterceptor1;

    @Autowired
    MyCustomInterceptor2 myCustomInterceptor2;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        // below order is maintained while calling interceptors
        registry.addInterceptor(myCustomInterceptor1)
                .addPathPatterns("/api/*") // Apply to these URL patterns
                .excludePathPatterns("/api/updateUser", "/api/deleteUser"); // Exclude for these URL patterns

        registry.addInterceptor(myCustomInterceptor2)
                .addPathPatterns("/api/*") // Apply to these URL patterns
                .excludePathPatterns("/api/updateUser");
    }
}
```

**Output:**

```text
inside pre handle Method - MyCustomInterceptor1
inside pre handle Method - MyCustomInterceptor2
hitting db to get the userdata
inside post handle method- MyCustomInterceptor2
inside post handle method- MyCustomInterceptor1
inside after completion method - MyCustomInterceptor2
inside after completion method- MyCustomInterceptor1
```

![svg](<svgs/06-multiple-interceptors-order.svg>)

- `preHandle()` runs in **registration order** (1 → N).
- `postHandle()` and `afterCompletion()` run in **reverse order** (N → 1).
- **If `preHandle()` returns `false`, the next interceptors and the controller are not invoked.** `afterCompletion()` still runs for the interceptors whose `preHandle()` already returned `true`, so they can clean up.
- Order can also be set explicitly: `registry.addInterceptor(rateLimitInterceptor).order(1)`.

> **Path pattern gotcha:** `/api/*` matches one path segment only (`/api/getUser`), not `/api/v1/users`. Use `/api/**` to match everything below `/api`.

Example with explicit order:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired private AuditInterceptor auditInterceptor;
    @Autowired private RateLimitInterceptor rateLimitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(rateLimitInterceptor)     // runs first
                .addPathPatterns("/**")
                .order(1);

        registry.addInterceptor(auditInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/health")
                .order(2);
    }
}
```

### Why do interceptors run in reverse order on the way out?

It's the same idea as filters: think of interceptors as a **stack (LIFO)**.

- When the request comes in, it passes through all interceptors one by one (forward order).
- After the controller processes it, the response travels back along the same path.
- The **last** interceptor that saw the request must be the **first** one to handle the response.
- This keeps the flow consistent and lets each interceptor clean up what it added (like open & close pairs).

**Real-life analogy:** you walk through doors I1 → I2 → I3 to reach a room, and you come back out through I3 → I2 → I1.

| Step | Interceptors I1, I2, I3 |
|---|---|
| 1–3 | I1.preHandle() → I2.preHandle() → I3.preHandle() |
| 4 | Controller method |
| 5–7 | I3.postHandle() → I2.postHandle() → I1.postHandle() |
| 8–10 | I3.afterCompletion() → I2.afterCompletion() → I1.afterCompletion() |

**Memory trick:** "Last In, First Out", like a stack: **push** on request, **pop** on response.

---

## 7. Filters and interceptors together

When both filters and interceptors are used, the filters wrap everything, including all interceptors:

![svg](<svgs/07-filters-and-interceptors-full-order.svg>)

**Output (two filters + two interceptors):**

```text
MyFilter1 inside
MyFilter2 inside
inside pre handle Method - MyCustomInterceptor1
inside pre handle Method - MyCustomInterceptor2
hitting db to get the userdata
inside post handle method- MyCustomInterceptor2
inside post handle method- MyCustomInterceptor1
inside after completion method - MyCustomInterceptor2
inside after completion method- MyCustomInterceptor1
MyFilter2 completed
MyFilter1 completed
```

> In this run `MyFilter1` was given the lower order value, so it ran first. Whichever filter has the lower order runs first on the way in and last on the way out.

**Remember**
1. Always call `chain.doFilter()` in a filter, otherwise the request is blocked.
2. Return `true` from `preHandle()` to continue, `false` to block.
3. Put cleanup code in `afterCompletion()`; it always runs (for interceptors whose `preHandle()` returned `true`).
4. Together they give complete control over the whole request lifecycle.

---

## 8. Filter vs Interceptor: key differences

| Feature | Filter | Interceptor |
|---|---|---|
| Belongs to | Servlet API (`jakarta.servlet`) | Spring MVC |
| Works at | Before `DispatcherServlet` | Inside `DispatcherServlet` |
| Can access | Request, response | Request, response, **handler (controller method)**, `ModelAndView` |
| Configuration | `FilterRegistrationBean`, `@Component`, `@WebFilter` (web.xml in old apps) | `WebMvcConfigurer.addInterceptors()` |
| Use cases | Encoding, CORS, security, logging, compression | Business checks, logging, authorization by annotation |
| Runs for | Every request, including requests to other servlets | Only requests handled by `DispatcherServlet` |
| Can stop a request | Yes, by not calling `chain.doFilter()` | Yes, by returning `false` from `preHandle()` |
| Hooks | One method (`doFilter`) around the chain | Three: `preHandle`, `postHandle`, `afterCompletion` |

**What each can and cannot do**

| Capability | Filter | Interceptor |
|---|---|---|
| Replace / wrap the request or response object (e.g. to read or modify the body) | ✅ Yes | ❌ No (body may already be read) |
| Access raw `InputStream` / `OutputStream` | ✅ Yes | ❌ No |
| Know which controller method will handle the request | ❌ No | ✅ Yes (`handler` object) |
| Read annotations on the controller method | ❌ No | ✅ Yes |
| Access Spring `ModelAndView` | ❌ No | ✅ Yes (in `postHandle`) |
| Use Spring beans (`@Autowired`) | ✅ Yes, if it is a Spring bean | ✅ Yes (always) |
| Apply to requests for other (non-Spring) servlets | ✅ Yes | ❌ No (only `DispatcherServlet`) |
| Works without Spring | ✅ Yes (Java EE / Jakarta standard) | ❌ No (Spring only) |
| Exception info after the response | ❌ No (only via try/catch around the chain) | ✅ Yes (`afterCompletion`) |

### Which one should I use?

![svg](<svgs/08-filter-interceptor-or-aop.svg>)

**Summary**
- Use a **Filter** for low-level tasks like security, encoding, CORS, compression.
- Use an **Interceptor** for application-specific logic related to controllers.
- Both are powerful and can be used together.
- (For logic around service methods, not HTTP, use **Spring AOP**.)

---

## 9. Common pitfalls

| Pitfall | What happens | Fix |
|---|---|---|
| Forgetting `chain.doFilter()` | Request hangs / empty response, controller never called | Always call it (unless you're deliberately blocking and writing the response) |
| `@Component` **and** `FilterRegistrationBean` on the same filter | Filter registered twice and runs twice | Use only one of them |
| Plain `Filter` running twice for one request | Error dispatch (`/error`) or forward runs it again | Extend `OncePerRequestFilter` |
| Changing headers in `postHandle()` for a `@RestController` | No effect: JSON is already written, response committed | Use a filter, `ResponseBodyAdvice`, or set headers in `preHandle()` |
| Storing request data in an interceptor/filter **field** | Data leaks between concurrent requests (singleton bean) | Use `request.setAttribute(...)` or local variables |
| Reading the request body in a filter | Controller later gets an empty body (stream can be read only once) | Wrap with `ContentCachingRequestWrapper` |
| `MDC` / `ThreadLocal` not cleared | Values leak into the next request on the same pooled thread | Clear in `finally` (filter) or `afterCompletion()` |
| `/api/*` pattern | Doesn't match nested paths like `/api/v1/users` | Use `/api/**` |

---

## 10. Interview Questions

**Q1. What is the difference between a filter and an interceptor?**
A filter is part of the Servlet API and runs before the request reaches any servlet (including `DispatcherServlet`). An interceptor is part of Spring MVC and runs inside `DispatcherServlet`, around the controller method. Filters see only the raw request/response. Interceptors also know the handler (controller method) and the `ModelAndView`.

**Q2. In what order do filters and interceptors run?**
Filters (in order) → `DispatcherServlet` → interceptors' `preHandle` (registration order) → controller → `postHandle` (reverse) → view render → `afterCompletion` (reverse) → filters in reverse.

**Q3. Why do filters/interceptors run in reverse order on the way back?**
Because each one wraps the next (like a stack / Russian dolls). The last one to receive the request is the first to see the response (LIFO). This keeps "around" logic symmetric, so each one can clean up what it opened.

**Q4. How do you set the order of filters?**
`FilterRegistrationBean.setOrder(n)` or `@Order(n)` on a `@Component` filter. A lower value runs first. Spring Security's filter chain is itself registered as a filter with order `-100` by default (`spring.security.filter.order`).

**Q5. How do you set the order of interceptors?**
By registration order in `addInterceptors()`, or explicitly with `.order(n)` on the registration.

**Q6. What happens if `preHandle()` returns false?**
The controller and all remaining interceptors are skipped, and so are all `postHandle()` calls. `afterCompletion()` runs only for the interceptors whose `preHandle()` already returned `true`. You should write the response yourself (e.g. `response.sendError(403)`).

**Q7. When is `postHandle()` not called?**
When the controller throws an exception, or when a `preHandle()` returned `false`. `afterCompletion()` is still called (with the exception, if it wasn't handled).

**Q8. Why can't you modify the response in `postHandle()` for REST APIs?**
For `@ResponseBody` / `@RestController` methods, the message converter writes the JSON while the handler result is being processed, which happens before `postHandle()`. The response is already committed, so header/body changes are ignored. Use a filter or a `ResponseBodyAdvice` instead.

**Q9. How do you exclude certain URLs?**
Interceptor: `.excludePathPatterns("/login", "/public/**")`. Filter: register it only for the right patterns with `addUrlPatterns(...)`, or override `shouldNotFilter()` in `OncePerRequestFilter`.

**Q10. What is `OncePerRequestFilter` and why use it?**
A Spring base class that guarantees the filter runs only once per request, even across forwards and error dispatches. It also gives you typed `HttpServletRequest`/`HttpServletResponse` and a `shouldNotFilter()` hook.

**Q11. Is Spring Security implemented with filters or interceptors? Why?**
Filters. A `DelegatingFilterProxy` hands off to Spring's `FilterChainProxy`, which runs the `SecurityFilterChain`. Security must run before *anything* else, including requests that never reach a controller (static files, `/error`, other servlets). It also needs to block requests as early as possible.

**Q12. How do you read the request body in a filter without breaking the controller?**
The input stream can be read only once. Wrap the request in `ContentCachingRequestWrapper` (and the response in `ContentCachingResponseWrapper`, calling `copyBodyToResponse()` at the end), then pass the wrapper down the chain.

**Q13. Can you `@Autowired` beans into a filter?**
Yes, if the filter is a Spring bean (`@Component`, or a `@Bean` you pass to `FilterRegistrationBean`). A filter created with `new` inside the registration bean method can't use field injection, but you can pass the dependencies through its constructor.

**Q14. Interceptor vs AOP?**
An interceptor works at the HTTP/controller level, has access to the request and response, and only covers `DispatcherServlet` requests. AOP (`@Aspect`) works on any Spring bean method (services, repositories) regardless of HTTP, using proxies. Use interceptors for web concerns, AOP for business-method concerns (transactions, retries, method metrics).

**Q15. Do interceptors run for async requests (`Callable`, `DeferredResult`)?**
`preHandle` runs on the request thread. After the handler starts async processing, `postHandle`/`afterCompletion` are **not** called on that thread; `afterConcurrentHandlingStarted()` is called instead (implement `AsyncHandlerInterceptor`). The normal hooks run later, when the async result is dispatched back.
