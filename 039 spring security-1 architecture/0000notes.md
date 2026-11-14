# Spring Security - 1: Architecture

## Table of Contents

1. [Why Spring Security?](#1-why-spring-security)
2. [Project setup and what you get by default](#2-project-setup-and-what-you-get-by-default)
3. [Where Spring Security sits: filters](#3-where-spring-security-sits-filters)
4. [The Security Filter Chain](#4-the-security-filter-chain)
5. [Authentication filters: "is this request for me?"](#5-authentication-filters-is-this-request-for-me)
6. [Authentication architecture: who does what](#6-authentication-architecture-who-does-what)
7. [The Authentication object](#7-the-authentication-object)
8. [AuthenticationManager and AuthenticationProvider](#8-authenticationmanager-and-authenticationprovider)
9. [UserDetailsService, UserDetails and PasswordEncoder](#9-userdetailsservice-userdetails-and-passwordencoder)
10. [SecurityContext and SecurityContextHolder](#10-securitycontext-and-securitycontextholder)
11. [After authentication: authorization and errors](#11-after-authentication-authorization-and-errors)
12. [Stateful vs stateless authentication](#12-stateful-vs-stateless-authentication)
13. [Writing a custom filter](#13-writing-a-custom-filter)
14. [Debugging Spring Security](#14-debugging-spring-security)
15. [What's next: authentication mechanisms](#15-whats-next-authentication-mechanisms)
16. [Interview Questions](#16-interview-questions)

---

## 1. Why Spring Security?

There are various types of attacks (see the previous topic, *038 attacks*):
- CSRF (Cross-Site Request Forgery)
- CORS (Cross-Origin Resource Sharing) misconfiguration
- SQL Injection
- XSS (Cross-Site Scripting)

To protect our resources from these attacks we need proper:
- **Authentication:** verifies **who you are**.
- **Authorization:** checks **what you are allowed to do**.

That's where Spring Security comes in. It is:
- a powerful, highly customizable **authentication and authorization** framework;
- built on top of **Servlet filters** (and Spring AOP for method security);
- able to secure both traditional web apps (MVC) and REST APIs;
- protection against common attacks out of the box: CSRF, session fixation, clickjacking, etc.

**Key idea:** Spring Security works as a **chain of filters**. It first **authenticates** the user, then **authorizes** the request, and only then lets the request reach your controller.

---

## 2. Project setup and what you get by default

**Existing Spring Boot project:** add the dependencies.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <!--
    Provides core features like:
    - Authentication
    - Authorization
    - Security filters etc.
    -->
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <!--
    Enables persistent session management.
    Only needed if you want to store sessions in a relational DB.
    -->
    <artifactId>spring-session-jdbc</artifactId>
</dependency>
```

**New project:** go to [start.spring.io](https://start.spring.io) and choose:

| Setting | Value used in these notes |
|---|---|
| Project / Language | Maven / Java |
| Spring Boot | 3.2.x |
| Group / Artifact | `com.conceptandcoding` / `learningspringboot` |
| Packaging / Java | Jar / 17 |
| Dependencies | **Spring Web**, **Spring Security** |

If you want to persist the session in a relational DB, also add `spring-session-jdbc` (above) and let Spring create its tables:

```properties
spring.session.jdbc.initialize-schema=always
```

Sessions are then stored in the `SPRING_SESSION` tables (they survive restarts and are shared by all app instances), and the cookie is named `SESSION` instead of `JSESSIONID`.

### What happens the moment you add the starter

Without writing any configuration:
- **Every endpoint requires authentication.**
- A default user `user` is created, with a random password printed at startup:
  ```text
  Using generated security password: 8e557245-73e2-4286-969a-ff57fe326336
  This generated password is for development use only. Your security configuration must be updated before running your application in production.
  ```
- A generated **login page** (`/login`) and **logout** (`/logout`) are available.
- **HTTP Basic** also works (`curl -u user:<password> http://localhost:8080/`).
- CSRF protection, security headers (`X-Frame-Options: DENY`, …) and session-fixation protection are on.

You can set a fixed user for quick tests:

```properties
spring.security.user.name=user
spring.security.user.password=pass
spring.security.user.roles=ADMIN
```

As soon as you define your own `SecurityFilterChain` bean, Spring Boot's default chain backs off and yours is used:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                              // for REST APIs (stateless)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/auth/**").permitAll()  // public endpoints
                .anyRequest().authenticated())                         // everything else secured
            .formLogin(form -> form.loginPage("/login").permitAll())
            .httpBasic(Customizer.withDefaults())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)); // no session
        return http.build();
    }
}
```

(This mixes options just to show the API. A real app picks either stateless + no CSRF, or session + CSRF.)

---

## 3. Where Spring Security sits: filters

In topic *020 filter interceptor* we saw what filters are and where they fit: **Tomcat → filter chain → DispatcherServlet → interceptors → controller**.

Spring Security plugs into that **servlet filter chain** as one special filter:

![svg](<svgs/01-where-spring-security-sits.svg>)

| Piece | What it is |
|---|---|
| **DelegatingFilterProxy** | A plain Servlet filter registered with Tomcat. It's the **bridge** between the Servlet container and Spring: it looks up a Spring bean named `springSecurityFilterChain` and delegates to it. (Registered in `web.xml` in old apps, automatically in Spring Boot.) |
| **FilterChainProxy** | That Spring bean. It holds all your `SecurityFilterChain`s and picks the right one for each request. |
| **SecurityFilterChain** | An ordered list of security filters plus a matcher saying which requests it applies to. |

Request flow:
1. The request comes into the application.
2. It goes to `DelegatingFilterProxy`.
3. It passes to `FilterChainProxy` (the `springSecurityFilterChain` bean).
4. `FilterChainProxy` selects the matching `SecurityFilterChain`, and each security filter in it is invoked in order.
5. If authentication succeeds, the `SecurityContext` is populated.
6. If any filter rejects the request, the chain stops and an error/redirect is returned.
7. If all filters pass, the request reaches the controller.

**Standard filter vs Spring Security filter**

| Standard filter | Spring Security filter |
|---|---|
| Registered via `@Component` or `FilterRegistrationBean` | Managed inside `springSecurityFilterChain` (`FilterChainProxy`) |
| You define the order with `@Order` / `setOrder()` | Order defined internally by Spring Security |
| General purpose (logging, CORS, etc.) | Security (authentication, authorization, session, etc.) |
| Runs before / after the security chain | Runs inside the security filter chain |

---

## 4. The Security Filter Chain

- Every incoming HTTP request passes through the chain **before** reaching the controller.
- Each filter has **one responsibility**, and filters run in a **fixed order** defined by Spring Security.
- `SecurityContextHolder` holds the `Authentication` for the current request while it travels through.

### 4.1 Multiple chains: first match wins

An app can have several `SecurityFilterChain`s, each with its own matcher (`/api/**`, `/admin/**`, …). `FilterChainProxy` finds the **first** matching chain and runs only its filters. If no chain matches, the request goes through without Spring Security.

![svg](<svgs/02-multiple-security-filter-chains.svg>)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")                       // only /api/** requests
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain webChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth                // no matcher → everything else
                .requestMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

### 4.2 The important filters, in order

![svg](<svgs/03-default-security-filter-order.svg>)

| # | Filter | Job |
|---|---|---|
| 1 | `DisableEncodeUrlFilter` | Stops the session id from being encoded into URLs |
| 2 | `WebAsyncManagerIntegrationFilter` | Makes the `SecurityContext` available to Spring async processing |
| 3 | `SecurityContextHolderFilter` | Loads the `SecurityContext` (e.g. from the session) for this request |
| 4 | `HeaderWriterFilter` | Adds security headers (`X-Frame-Options`, `X-Content-Type-Options`, …) |
| 5 | `CorsFilter` | Handles CORS / preflight requests |
| 6 | `CsrfFilter` | Protects against CSRF |
| 7 | `LogoutFilter` | Handles logout requests |
| 8 | `UsernamePasswordAuthenticationFilter` | Handles form login (`POST /login`) |
| 9 | `BearerTokenAuthenticationFilter` | Handles JWT / OAuth2 bearer tokens |
| 10 | `BasicAuthenticationFilter` | Handles HTTP Basic |
| 11 | `RequestCacheAwareFilter` | Restores the original request saved before redirecting to login |
| 12 | `SecurityContextHolderAwareRequestFilter` | Wraps the request so `request.isUserInRole()` / `getRemoteUser()` work |
| 13 | `AnonymousAuthenticationFilter` | Sets an anonymous user if nobody authenticated |
| 14 | `SessionManagementFilter` | Session fixation / concurrency control |
| 15 | `ExceptionTranslationFilter` | Turns security exceptions into 401 / 403 responses |
| 16 | `AuthorizationFilter` | Final authorization decision |

> **Spring Security 6 renames** (older notes and tutorials use the old names):
> - `SecurityContextPersistenceFilter` → **`SecurityContextHolderFilter`**. It only *loads* the context; saving it is done explicitly by the authentication filters.
> - `FilterSecurityInterceptor` → **`AuthorizationFilter`**.
>
> Not every filter is always active: only the filters for features you configure are added, and the exact list varies a little by version. The idea stays the same.

### 4.3 Does every request pass through every filter?

**Every request enters the chain, but filters may choose to do nothing** and just pass it on:
- For `/public/**` with `permitAll()`, the authentication filters find nothing to authenticate and simply continue.
- If CSRF is disabled, `CsrfFilter` isn't even added.
- If a filter's condition isn't matched, it simply calls `filterChain.doFilter(request, response)`.

Each filter does: some work **before** → `filterChain.doFilter(...)` → some work **after**. If a filter decides to send a response itself (like 401/403), it does **not** call `filterChain.doFilter()`, and the chain stops there. After completion, the response comes back through the filters in **reverse order** (first in, last out).

---

## 5. Authentication filters: "is this request for me?"

For each authentication method there is a **different filter**: form login uses one, JWT uses another, and so on. A request passes through the chain, and **only if a filter supports that request** does it try to authenticate. Otherwise it just continues.

| Authentication method | Filter used |
|---|---|
| Form login | `UsernamePasswordAuthenticationFilter` |
| HTTP Basic | `BasicAuthenticationFilter` |
| JWT / OAuth2 bearer token | `BearerTokenAuthenticationFilter` |
| OAuth2 login (client) | `OAuth2LoginAuthenticationFilter` |
| Remember-me | `RememberMeAuthenticationFilter` |
| Logout | `LogoutFilter` |

![svg](<svgs/04-each-auth-filter-decides.svg>)

**Example 1: form login request** (`POST /login`, `username=mohit`, `password=1234`)
- `UsernamePasswordAuthenticationFilter`: does this request match the login processing URL? **Yes** (`/login` by default) → extract username & password → create `UsernamePasswordAuthenticationToken` → `AuthenticationManager.authenticate(...)` → on success, the `SecurityContext` is set → continue.
- `BearerTokenAuthenticationFilter`: reached in the chain, but there's no Bearer token in the `Authorization` header, so it does nothing.

**Example 2: JWT request** (`GET /api/orders`, `Authorization: Bearer eyJhbGciOi...`)
- `UsernamePasswordAuthenticationFilter`: not a form-login request → continue.
- `BearerTokenAuthenticationFilter`: Bearer token found → extract it → create `BearerTokenAuthenticationToken` → `AuthenticationManager.authenticate(token)` → validate the JWT (signature, expiry, …) → on success, the `SecurityContext` is set → continue.
- Token missing/invalid on a protected URL → authentication fails → **401 Unauthorized**.

**Interview memory rule**
- **SecurityFilterChain** (big picture) decides: *"which group of filters applies to this request?"*
- **An individual authentication filter** decides: *"should I attempt MY authentication mechanism on this request?"* **Yes** → authenticate. **No** → continue the chain.

---

## 6. Authentication architecture: who does what

![svg](<svgs/05-authentication-architecture.svg>)

**Flow, step by step**
1. A request comes in and goes through the Security Filter Chain.
2. An authentication filter (e.g. `UsernamePasswordAuthenticationFilter`, or a JWT filter) creates an **unauthenticated** `Authentication` object with the credentials.
3. It calls `authenticationManager.authenticate(authentication)`.
4. `AuthenticationManager` (implemented by `ProviderManager`) delegates to the appropriate `AuthenticationProvider`.
5. The provider (e.g. `DaoAuthenticationProvider`) uses `UserDetailsService` to load user details from the DB / an external source, and `PasswordEncoder` to check the password.
6. If the credentials are valid, the provider returns an **authenticated** `Authentication` object.
7. The filter stores it in `SecurityContextHolder` (inside a `SecurityContext`).
8. Later filters (authorization) and your application use this `Authentication`.
9. Finally the request reaches the controller, if it's authorized.

> **Common question: "the filter creates an Authentication object and passes it to the AuthenticationManager, which returns the same object marked authenticated. True?"**
>
> **Almost true.** The key correction: `AuthenticationManager` is **not guaranteed to return the same object instance**. It receives an unauthenticated `Authentication` and, after successful authentication, usually returns a **new, richer** authenticated `Authentication` object.
>
> The authentication filter creates an unauthenticated `Authentication` containing the credentials and passes it to `AuthenticationManager`. `AuthenticationManager` delegates to an appropriate `AuthenticationProvider`. If verification succeeds, it returns an authenticated `Authentication`, usually containing the `UserDetails` as principal plus the authorities. The filter then stores that **returned** object in the `SecurityContext`.

**Key interfaces**

| Interface / class | Role |
|---|---|
| `SecurityFilterChain` | Defines the filters and security rules |
| `HttpSecurity` | Fluent API to configure a `SecurityFilterChain` |
| `Authentication` | Represents the authentication request / the authenticated user |
| `AuthenticationManager` | Authenticates the user (entry point) |
| `AuthenticationProvider` | Performs the actual authentication for one type of credentials |
| `UserDetailsService` | Loads user-specific data |
| `UserDetails` | User information model |
| `GrantedAuthority` | One role / permission |
| `PasswordEncoder` | Password hashing (BCrypt etc.) |
| `SecurityContextHolder` | Holds the `SecurityContext` (ThreadLocal) |
| `SecurityContext` | Holds the `Authentication` |
| `AuthenticationEntryPoint` | Handles 401 (not authenticated) |
| `AccessDeniedHandler` | Handles 403 (not authorized) |

---

## 7. The Authentication object

`Authentication` represents **"who is trying to access the application, and what do we know about them"**. It's an interface:

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();
    Object getDetails();
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated);
}
```

Spring uses the same type **before** authentication (as a request) and **after** authentication (as the authenticated user):

![svg](<svgs/06-authentication-before-after.svg>)

```java
// what the form-login filter builds (BEFORE)
UsernamePasswordAuthenticationToken auth =
        UsernamePasswordAuthenticationToken.unauthenticated("mohit", "abc123");
// "This person CLAIMS to be mohit." Not verified yet.
```

| Property | Question it answers | Before | After |
|---|---|---|---|
| `getPrincipal()` | **Who** is the user? | `"mohit"` (username) | Usually the `UserDetails` object |
| `getCredentials()` | **How** do they prove it? | `"abc123"` (password) | `null` (erased to reduce risk) |
| `getAuthorities()` | **What** are they allowed to do? | `[]` | `[ROLE_ADMIN]`; used later by e.g. `@PreAuthorize("hasRole('ADMIN')")` |
| `isAuthenticated()` | **Has** this been verified? | `false` | `true` |
| `getDetails()` | **Extra** info | Remote IP, session id (`WebAuthenticationDetails`) | same |

> An `Authentication` object existing ≠ the user being authenticated. It can be just an authentication **request** (`authenticated = false`).

Different mechanisms create different `Authentication` types (`UsernamePasswordAuthenticationToken`, `BearerTokenAuthenticationToken`, `JwtAuthenticationToken`, …), but all implement this interface.

**Memory rule:** Authentication = **WHO** (principal) + **PROOF** (credentials) + **PERMISSIONS** (authorities) + **STATUS** (authenticated) + **EXTRA** (details).

---

## 8. AuthenticationManager and AuthenticationProvider

### 8.1 AuthenticationManager

The core interface Spring Security uses to authenticate a user. It takes an `Authentication` as input and returns a fully authenticated `Authentication`, delegating the actual work to one or more `AuthenticationProvider`s:

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

Its default implementation is **`ProviderManager`**:

![svg](<svgs/07-provider-manager-loop.svg>)

- `authenticate()` is called by the filter.
- It loops through the configured `AuthenticationProvider`s.
- The first provider that `supports()` the token type tries to authenticate it.
- If authentication succeeds, a fully authenticated `Authentication` is returned.
- If all providers fail → an `AuthenticationException` is thrown (e.g. `BadCredentialsException`).

Spring Boot auto-configures an `AuthenticationManager`. If you need it yourself (e.g. in a JWT login endpoint), expose it as a bean:

```java
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

### 8.2 AuthenticationProvider

It performs the actual authentication for one kind of credential:

| Provider | Handles |
|---|---|
| `DaoAuthenticationProvider` | Username/password. Uses `UserDetailsService` + `PasswordEncoder`, and does extra checks (account locked, expired, …) |
| `JwtAuthenticationProvider` | Validates a JWT token, extracts the user & authorities |
| OAuth2 login providers | Validate OAuth2 tokens, load user info |
| Your custom provider | Custom logic (OTP, API key, legacy system, …) |

```java
@Component
public class ApiKeyAuthenticationProvider implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String apiKey = (String) authentication.getCredentials();
        if (!"secret-key-123".equals(apiKey)) {
            throw new BadCredentialsException("Invalid API key");
        }
        return new ApiKeyAuthenticationToken(apiKey, List.of(new SimpleGrantedAuthority("ROLE_SERVICE")));
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);  // only my token type
    }
}
```

---

## 9. UserDetailsService, UserDetails and PasswordEncoder

### 9.1 UserDetailsService: loads the user

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

Built-in implementations:
- `InMemoryUserDetailsManager`: manages username/password **in memory** (demos, tests).
- `JdbcUserDetailsManager`: manages username/password in a **DB** using Spring Security's default tables.
- Your own implementation (e.g. reading your JPA `User` entity), which is the most common.

```java
// In-memory users + password encoder
@Bean
public UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails user = User.withUsername("mohit")
            .password(encoder.encode("1234"))
            .roles("USER")
            .build();
    return new InMemoryUserDetailsManager(user);
}

@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

```java
// Your own, backed by JPA
@Service
public class AppUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public AppUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        return new CustomUserDetails(user);
    }
}
```

If there's exactly one `UserDetailsService` bean and one `PasswordEncoder` bean, Spring Boot wires them into a `DaoAuthenticationProvider` automatically.

### 9.2 UserDetails: the user model Spring Security understands

```java
public interface UserDetails extends Serializable {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();

    boolean isAccountNonExpired();      // account expired?
    boolean isAccountNonLocked();       // account locked?
    boolean isCredentialsNonExpired();  // password expired?
    boolean isEnabled();                // user enabled?
}
```

Default implementation `org.springframework.security.core.userdetails.User`:

```java
User user = new User(
        "mohit",
        passwordEncoder.encode("1234"),
        List.of(new SimpleGrantedAuthority("ROLE_USER"))
);
```

Custom implementation wrapping your own entity:

```java
public class CustomUserDetails implements UserDetails {

    private final User user;   // your entity

    public CustomUserDetails(User user) { this.user = user; }

    @Override public String getUsername() { return user.getEmail(); }
    @Override public String getPassword() { return user.getPassword(); }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority(role.getName()))
                .collect(Collectors.toList());
    }

    // return true for all below (or your custom logic)
    @Override public boolean isAccountNonExpired()     { return true; }
    @Override public boolean isAccountNonLocked()      { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled()               { return user.isEnabled(); }
}
```

**Role of `UserDetails`:** it encapsulates the user information: credentials (password), authorities (roles/permissions) and status flags. `AuthenticationProvider` uses it to verify the user, and the status flags are checked during authentication (locked account → `LockedException`, disabled → `DisabledException`).

**`UserDetails` vs `Authentication`**
- `UserDetails` = information **about the user**, loaded from the DB (username, password, authorities, enabled, accountNonLocked, …).
- `Authentication` = the **current security state** (principal, credentials, authorities, details, authenticated). After login, its **principal is often the `UserDetails`**.

### 9.3 PasswordEncoder

The raw password is hashed during registration. During login, `DaoAuthenticationProvider` calls `passwordEncoder.matches(rawPassword, storedHash)`: the entered password is hashed the same way and compared with the stored hash (`$2a$10$...` for BCrypt). Passwords are **never** stored or compared in plain text.

---

## 10. SecurityContext and SecurityContextHolder

After successful authentication, the `Authentication` object is put into the **SecurityContext**, so the rest of the request (authorization filters, controllers, services) can use it.

![svg](<svgs/08-security-context-holder.svg>)

```java
public interface SecurityContext extends Serializable {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
// default implementation: SecurityContextImpl

public class SecurityContextHolder {
    private static SecurityContextHolderStrategy strategy;   // ThreadLocal by default
    public static SecurityContext getContext() { ... }
    public static void setContext(SecurityContext context) { ... }
    public static void clearContext() { ... }
}
```

**In simple words:** `SecurityContextHolder` holds the `SecurityContext`, the `SecurityContext` holds the `Authentication`, and the `Authentication` holds information about the user.

**Memory tip:** SecurityContext = the **container** · Authentication = **identity + permissions** · SecurityContextHolder = **where it's kept**.

### 10.1 How it's populated (during authentication)

HTTP request → authentication filter → `AuthenticationManager` → `AuthenticationProvider` → successful authentication → the filter creates an empty `SecurityContext`, sets the `Authentication` in it, and calls `SecurityContextHolder.setContext(context)`.

If you authenticate **manually** (e.g. a custom login endpoint), do the same. In Spring Security 6 you also have to **save** the context explicitly if it should survive into the next request (session):

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authenticatedAuth);
SecurityContextHolder.setContext(context);
securityContextRepository.saveContext(context, request, response);  // e.g. HttpSessionSecurityContextRepository
```

### 10.2 Where it's stored, and the holder modes

- By default it's **ThreadLocal**: each thread (= each request) has **its own** `SecurityContext`, so two users' requests never mix.
- **Stateful** (form login): after login the context is saved in the `HttpSession`. On every following request, `SecurityContextHolderFilter` loads it from the session and puts it into the holder.
- **Stateless** (JWT): no session. For every request, the filter validates the token, builds a new `Authentication` and sets it in the holder.
- It's **cleared after every request** (threads are reused from the pool), and also after logout and when the session expires.

| `SecurityContextHolder` mode | Meaning |
|---|---|
| `MODE_THREADLOCAL` (default) | One context per thread |
| `MODE_INHERITABLETHREADLOCAL` | Child threads inherit the parent's context |
| `MODE_GLOBAL` | One global context for the entire application (rare; e.g. desktop apps) |

> Never store the `SecurityContext` in your own static fields; always go through `SecurityContextHolder`. For `@Async` or your own thread pools, use `DelegatingSecurityContextExecutor` so the context reaches the worker thread.

### 10.3 Using it: the controller can access the authenticated user

The `SecurityContext` is available for the whole request, so the controller (and services) can read it:

```java
// 1. Anywhere: from the holder
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Object principal = auth.getPrincipal();
if (auth != null && auth.isAuthenticated()) {
    // user is authenticated
}

// 2. In a controller: Authentication as a method parameter
@GetMapping("/profile")
public String profile(Authentication authentication) {
    return authentication.getName();           // or principal, authorities, etc.
}

// 3. In a controller: @AuthenticationPrincipal (= authentication.getPrincipal())
@GetMapping("/me")
public String me(@AuthenticationPrincipal UserDetails user) {
    return user.getUsername();
}
```

It's also what method security reads: `@PreAuthorize("hasRole('ADMIN')")` checks `auth.getAuthorities()`.

---

## 11. After authentication: authorization and errors

![svg](<svgs/10-authorization-and-errors.svg>)

**Request lifecycle**
1. The client sends a request to the application.
2. The request enters the Spring Security filter chain.
3. Authentication filters try to authenticate the user (form login / basic / JWT / OAuth2 …).
4. If authentication succeeds → the `Authentication` object is stored in `SecurityContextHolder`.
5. `AuthorizationFilter` checks whether the user has the required roles/permissions.
6. If authorized → the request goes to the controller.
7. If not authorized → `AccessDeniedException` → **403 Forbidden** (`AccessDeniedHandler`).
8. If not authenticated → `AuthenticationEntryPoint` → **401** (REST) or **redirect to login** (form login).

`ExceptionTranslationFilter` makes the 401 vs 403 decision: anonymous user → entry point (401/login). Logged-in user without permission → access denied handler (403).

---

## 12. Stateful vs stateless authentication

![svg](<svgs/09-stateful-vs-stateless.svg>)

**Stateful (session-based):** the server **stores** the user's session (`HttpSession`). After login, a session id (`JSESSIONID`) is sent to the client in a cookie. The client sends it on every request, and the server looks up the session to authenticate the user.
1. User submits the login form.
2. Server authenticates.
3. Server creates a session and stores it in the session store.
4. Server sends `JSESSIONID` (cookie).
5. *For every request:* the client sends `JSESSIONID` → the server finds the session in the store → if valid, the user is authenticated → response.

**Stateless (token-based):** the server does **not** store a session. After login it issues a token (usually a JWT). The client stores the token and sends it on every request in the `Authorization` header. The server validates the token every time; there's no session lookup.
1. User logs in.
2. Server validates the credentials.
3. Server generates a signed JWT.
4. Server returns the JWT to the client.
5. *For every request:* the client sends `Authorization: Bearer <JWT>` → the server validates the JWT (signature, expiry, claims) → if valid, the user is authenticated → response.

| Aspect | Stateful | Stateless |
|---|---|---|
| Session storage | On the server (HttpSession / DB / Redis) | Not on the server (client keeps the token) |
| Client storage | Cookie (`JSESSIONID`) | Token (memory / cookie / localStorage) |
| Server load | Higher (session lookup) | Lower (token validation only) |
| Scalability | Harder (sticky sessions / shared store) | Easier (horizontally scalable) |
| Logout | Easy (invalidate the session) | Harder (token expiry / deny-list) |
| CSRF | Enabled by default (needed) | Disabled (not needed if no cookies) |
| Examples | Form login (Spring Security default), HTTP Basic with session, remember-me | JWT, OAuth2 bearer tokens, API keys |
| Use cases | Traditional / monolithic web apps (MVC), server-side session control | REST APIs, microservices, mobile, SPAs, cross-domain apps |

**Drawbacks.** Stateful: scaling needs sticky sessions or a shared store like Redis, and it uses more server memory. Stateless: you can't log out before the token expires (unless you use a deny-list or short expiry), tokens make requests bigger, and they're more complex to manage securely.

**Memory tip:** Stateful → the **server** remembers. Stateless → the **token** remembers.

---

## 13. Writing a custom filter

Extend `OncePerRequestFilter` (recommended: it runs once per request) and register it at a specific position:

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // 1. Read JWT from header
        // 2. Validate token
        // 3. If valid, set Authentication in SecurityContextHolder
        // 4. Continue the chain
        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return request.getRequestURI().startsWith("/public/");   // skip some paths
    }
}
```

Register it in `SecurityConfig`:

```java
http.addFilterBefore(new JwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
// or addFilterAfter(filter, X.class), or addFilterAt(filter, X.class) (same position as X)
```

> Don't also annotate a security filter with `@Component`: Spring Boot would register it a second time as a normal servlet filter (outside the security chain). If you need it as a bean, disable that extra registration with a `FilterRegistrationBean` whose `setEnabled(false)`.

---

## 14. Debugging Spring Security

- **See the filter chain and every security decision:**
  ```properties
  logging.level.org.springframework.security=DEBUG     # TRACE for even more detail
  ```
  At startup this prints the filters in each chain. Per request it logs which chain matched, which filter authenticated, and why access was denied.
- **Per-request debug output** (development only, never in production):
  ```java
  @EnableWebSecurity(debug = true)
  ```
- **Check who is logged in:** add a temporary endpoint that returns `SecurityContextHolder.getContext().getAuthentication()`.
- **401 vs 403:** 401 = not authenticated (entry point). 403 = authenticated but not allowed, **or** a CSRF failure on POST.

---

## 15. What's next: authentication mechanisms

Now let's understand the end-to-end flow with an example for each authentication and authorization mechanism:

1. Form Login (Stateful)
2. Basic Authentication (Stateless)
3. JWT (Stateless)
4. OAuth2
   - a. Authorization Code (Stateful or Stateless)
   - b. Client Credentials (Stateless)
   - c. Password Grant (Stateless; deprecated in OAuth 2.1)
5. API Key Authentication (Stateless)

etc.

---

## 16. Interview Questions

**Q1. What is Spring Security and how does it work internally?**
An authentication and authorization framework built on Servlet filters. Tomcat calls `DelegatingFilterProxy`, which delegates to the `FilterChainProxy` bean. That bean picks the matching `SecurityFilterChain` and runs its filters: authentication filters build an `Authentication` and call `AuthenticationManager`, the result is stored in `SecurityContextHolder`, and `AuthorizationFilter` decides access before the controller runs.

**Q2. Difference between authentication and authorization?**
Authentication = who are you (verifying identity: password, token). Authorization = what are you allowed to do (roles/permissions on URLs or methods). Authentication happens first; authorization uses its result.

**Q3. What is the Security Filter Chain? What are DelegatingFilterProxy and FilterChainProxy?**
A `SecurityFilterChain` is an ordered list of security filters plus a request matcher. `DelegatingFilterProxy` is the Servlet filter Tomcat knows about; it forwards to the Spring bean `springSecurityFilterChain`, which is a `FilterChainProxy`. That proxy holds all chains, selects the first one that matches the request, and runs it.

**Q4. Does every request pass through every filter?**
Every request goes through the selected chain, but each filter decides whether to act. Filters that don't apply (e.g. form-login filter on a JWT request) just call `filterChain.doFilter()`.

**Q5. Which are the important filters, and why does the order matter?**
`SecurityContextHolderFilter`, `HeaderWriterFilter`, `CorsFilter`, `CsrfFilter`, `LogoutFilter`, the authentication filters (`UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`, `BasicAuthenticationFilter`), `AnonymousAuthenticationFilter`, `ExceptionTranslationFilter`, `AuthorizationFilter`. Order matters because each filter depends on the previous ones: e.g. authorization needs authentication to have run, and CORS must run before authentication so preflights aren't rejected.

**Q6. What are `AuthenticationManager` and `AuthenticationProvider`?**
`AuthenticationManager` is the single entry point: `authenticate(Authentication)`. Its default implementation, `ProviderManager`, loops over `AuthenticationProvider`s, and the first one whose `supports()` returns true does the actual check (e.g. `DaoAuthenticationProvider` using `UserDetailsService` + `PasswordEncoder`).

**Q7. Does `AuthenticationManager` return the same `Authentication` object?**
Not necessarily. It receives an unauthenticated token and usually returns a **new**, authenticated one (principal = `UserDetails`, authorities filled, credentials erased). The filter stores the returned object.

**Q8. What is `UserDetailsService`? `UserDetails`?**
`UserDetailsService.loadUserByUsername()` loads a user from your store. `UserDetails` is the user model Spring Security understands: username, password hash, authorities, and account status flags.

**Q9. What are `SecurityContext` and `SecurityContextHolder`?**
`SecurityContext` holds the current `Authentication`. `SecurityContextHolder` stores the context, per thread (ThreadLocal) by default, so it's available anywhere during the request.

**Q10. How is the `SecurityContext` kept between requests in stateful vs stateless apps?**
Stateful: saved in the `HttpSession` and reloaded by `SecurityContextHolderFilter` on each request. Stateless: rebuilt on every request from the token, and cleared at the end.

**Q11. Stateful vs stateless in Spring Security? When would you use each?**
Stateful uses a session + `JSESSIONID` cookie, needs CSRF protection, and suits server-rendered web apps. Stateless uses a token (JWT) in the `Authorization` header with `SessionCreationPolicy.STATELESS` and CSRF disabled, and suits REST APIs, SPAs, mobile and microservices.

**Q12. What's the difference between 401 and 403 in Spring Security?**
401: not authenticated, handled by `AuthenticationEntryPoint` (form login redirects to `/login` instead). 403: authenticated but lacking permission, handled by `AccessDeniedHandler`. `ExceptionTranslationFilter` chooses between them.

**Q13. How do you add a custom filter? `addFilterBefore` vs `addFilterAfter` vs `addFilterAt`?**
Extend `OncePerRequestFilter` and register it with `http.addFilterBefore(f, X.class)` (runs just before X), `addFilterAfter(f, X.class)` (just after X), or `addFilterAt(f, X.class)` (same position as X; it doesn't replace X).

**Q14. Where is the filter chain defined in Spring Boot?**
In `@Bean SecurityFilterChain` methods built from `HttpSecurity` (usually in a `@Configuration` + `@EnableWebSecurity` class). Without such a bean, Spring Boot's auto-configuration provides a default chain (everything authenticated, form login + basic).

**Q15. How does JWT authentication work in Spring Security?**
The client logs in once and receives a signed JWT. On every request it sends `Authorization: Bearer <jwt>`. `BearerTokenAuthenticationFilter` (resource server) or your custom filter validates the signature/expiry, builds an `Authentication` from the claims, and sets it in `SecurityContextHolder`. No session is used.

**Q16. How do you get the logged-in user in a controller?**
`SecurityContextHolder.getContext().getAuthentication()`, an `Authentication`/`Principal` method parameter, or `@AuthenticationPrincipal UserDetails user`.

**Q17. Why must the `SecurityContext` be cleared after each request, and what about `@Async`?**
Servlet threads are pooled. A leftover context would leak one user's identity into another user's request. `@Async` threads don't inherit a ThreadLocal, so use `DelegatingSecurityContextExecutor` (or `MODE_INHERITABLETHREADLOCAL`, carefully) to pass it on.
