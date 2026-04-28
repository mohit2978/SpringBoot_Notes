# Notes

## Spring Security 

Spring Security is a powerful authentication and authorization framework for Java/Spring applications. Here's a breakdown of the key concepts:

### What it does
Spring Security handles two core concerns:
- **Authentication** — *Who are you?* (login, identity verification)
- **Authorization** — *What can you do?* (permissions, roles, access control)

### How it works — The Filter Chain

Spring Security intercepts every HTTP request through a chain of filters before it reaches your application:

```
HTTP Request → [Filter 1] → [Filter 2] → ... → [Your Controller]
```

The most important filter is the **`SecurityFilterChain`**, which decides whether a request is allowed through.

### Core Concepts

**1. Authentication**
- Users provide credentials (username/password, token, etc.)
- Spring verifies them via an `AuthenticationManager`
- On success, a `SecurityContext` stores the authenticated user's details for the duration of the request

**2. UserDetailsService**
- You implement this interface to tell Spring *how to load a user* (from DB, LDAP, etc.)
```java
@Service
public class MyUserService implements UserDetailsService {
    public UserDetails loadUserByUsername(String username) {
        // fetch user from DB
    }
}
```

**3. Password Encoding**
- Passwords are never stored in plain text
- Spring uses `PasswordEncoder` (typically `BCryptPasswordEncoder`)

**4. Authorization / Access Control**
- Done via roles (`ROLE_ADMIN`, `ROLE_USER`) or authorities
- Can be configured at the URL level or method level:
```java
// URL-level
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/public/**").permitAll()
    .anyRequest().authenticated()
);

// Method-level
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }
```

### Common Authentication Mechanisms

| Mechanism | Use Case |
|---|---|
| Form Login | Traditional web apps |
| HTTP Basic | APIs, simple clients |
| JWT (JSON Web Token) | Stateless REST APIs |
| OAuth2 / OpenID Connect | Social login, SSO |
| LDAP | Enterprise/corporate environments |

### Minimal Configuration Example (Spring Boot)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .logout(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Key Classes to Know

| Class/Interface | Role |
|---|---|
| `SecurityFilterChain` | Defines security rules for HTTP |
| `AuthenticationManager` | Processes authentication requests |
| `UserDetailsService` | Loads user data |
| `GrantedAuthority` | Represents a permission/role |
| `SecurityContextHolder` | Stores the current user's auth info |

### Spring Boot Auto-configuration
If you add `spring-boot-starter-security` to your project, Spring Boot automatically:
- Secures all endpoints
- Generates a default login page
- Creates a default user with a random password (printed in logs)

You then override this by defining your own `SecurityFilterChain` bean.

---



![alt text](033_spring_security_architecture_250716_003028_1.jpg) ![alt text](033_spring_security_architecture_250716_003028_2.jpg) ![alt text](033_spring_security_architecture_250716_003028_3.jpg) ![alt text](033_spring_security_architecture_250716_003028_4.jpg)



![alt text](image.png)


Here's the general Spring Security architecture — no JWT-specific pieces, just the core framework components and how they relate to each other.Here's what each layer does:

---

### 1. `DelegatingFilterProxy`

A standard servlet filter registered with the servlet container (Tomcat). Its only job is to look up Spring's `FilterChainProxy` bean from the `ApplicationContext` and delegate to it. This is the seam between the Java EE world and the Spring world.

---

### 2. `FilterChainProxy`

The central entry point for Spring Security. It holds a list of `SecurityFilterChain` instances and picks the right one based on the request URL. For example, `/api/**` might use a stateless chain, while `/admin/**` uses a different one with stricter rules.

---

### 3. `SecurityFilterChain`

An ordered list of filters that every matched request passes through. The key built-in ones are:

- `SecurityContextPersistenceFilter` — restores or creates a `SecurityContext` at the start of the request, and saves it at the end
- `Authentication filters` — a family of filters depending on your mechanism: `UsernamePasswordAuthenticationFilter` for form login, `BasicAuthenticationFilter` for HTTP Basic, `OAuth2LoginAuthenticationFilter` for OAuth2, etc. Only one typically activates per request
- `ExceptionTranslationFilter` — catches `AuthenticationException` (→ 401) and `AccessDeniedException` (→ 403) thrown by downstream filters and translates them into proper HTTP responses
- `AuthorizationFilter` — the final check before your controller is reached; enforces `.authorizeHttpRequests()` rules

---

### 3 supporting components

`AuthenticationManager` — receives an `Authentication` token (e.g. username + password) and delegates to one or more `AuthenticationProvider` implementations that know how to verify it (against a DB, LDAP, in-memory, etc.).

`SecurityContextHolder` — a thread-local store that holds the `SecurityContext` for the current request. Any code in your app can call `SecurityContextHolder.getContext().getAuthentication()` to find out who's logged in.

`UserDetailsService` — an interface you implement to tell Spring *how to load a user* (typically from a database). The `AuthenticationProvider` calls it during credential verification.

---

### The key design insight

Every mechanism — form login, HTTP Basic, OAuth2, JWT — is just a different `AuthenticationFilter` plugged into the same chain. The rest of the architecture (`ExceptionTranslationFilter`, `AuthorizationFilter`, `SecurityContextHolder`) stays the same regardless of which authentication method you choose.

## UserDetailsService


 `UserDetailsService` exists because Spring Security needs to answer one question during authentication:

> **"Given a username, what do I know about this user?"**

---

### The core reason — separation of concerns

Spring Security knows *how* to authenticate (compare passwords, verify tokens, check roles) but it has **no idea where your users live**. Your users might be in:

- A PostgreSQL database
- An LDAP/Active Directory server
- An in-memory list (for testing)
- A remote REST API
- A flat file

So Spring Security defines a simple contract — `UserDetailsService` — and says:

> *"You tell me how to load a user, I'll handle everything else."*

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

That's the entire interface. One method. You implement it, Spring Security calls it.

---

### What happens under the hood

When a user tries to log in, here's exactly where `UserDetailsService` fits:

```
User submits credentials
        |
        ▼
AuthenticationFilter          ← intercepts the request
        |
        ▼
AuthenticationManager         ← orchestrates
        |
        ▼
DaoAuthenticationProvider     ← the default provider
        |
        ├── calls UserDetailsService.loadUserByUsername(username)
        |         └── returns a UserDetails object (your user from DB)
        |
        └── calls PasswordEncoder.matches(rawPassword, storedHash)
                  └── compares what user typed vs what DB has
```

`DaoAuthenticationProvider` is the glue — it uses `UserDetailsService` to fetch the user, then uses `PasswordEncoder` to verify the password. You never have to write that comparison logic yourself.

---

### What `UserDetails` carries back

Your `loadUserByUsername` must return a `UserDetails` object. Spring Security uses this to get:

```java
public interface UserDetails {
    String getUsername();
    String getPassword();           // the stored hash
    Collection<GrantedAuthority> getAuthorities();  // roles like ROLE_ADMIN
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

A typical implementation just wraps your `User` entity:

```java
@Service
public class MyUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        User user = userRepo.findByUsername(username)
            .orElseThrow(() ->
                new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())       // already BCrypt hashed
            .roles(user.getRole())              // e.g. "ADMIN", "USER"
            .accountLocked(!user.isActive())
            .build();
    }
}
```

---

### Why not just let you write the whole auth logic yourself?

Because then every developer would have to re-implement password comparison, account status checks, role loading, exception handling, and integration with the filter chain — all correctly and securely. `UserDetailsService` draws a clear line:

| Spring Security handles | You handle |
|---|---|
| Password comparison | Fetching the user |
| Role/authority checks | Mapping roles from your DB |
| Account lock/expiry logic | Defining what "locked" means in your system |
| Session/token management | Where users are stored |

---

### In JWT specifically — it plays a slightly different role

With JWT, there's no password comparison happening on each request (the token already proves identity). But `UserDetailsService` is still used to:

1. **Reload fresh user details** on every request (in case the user was disabled since the token was issued)
2. **Populate the `Authentication` object** that gets stored in `SecurityContextHolder`

```java
// Inside JwtAuthFilter
UserDetails userDetails = userDetailsService.loadUserByUsername(username);

if (jwtService.isTokenValid(token, userDetails)) {
    // Set authentication in context
    UsernamePasswordAuthenticationToken auth =
        new UsernamePasswordAuthenticationToken(
            userDetails, null, userDetails.getAuthorities()
        );
    SecurityContextHolder.getContext().setAuthentication(auth);
}
```

So even in a stateless JWT setup, `UserDetailsService` acts as a **real-time check** — it ensures the user still exists and is still active, even if their token hasn't expired yet.

---

### In one sentence

`UserDetailsService` is the plugin point that lets Spring Security stay completely storage-agnostic — you own the data layer, Spring Security owns the security logic.




