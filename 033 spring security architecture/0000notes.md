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

## Context

The word "context" is one of those terms that confuses a lot of people. Let me break it down from the ground up — what "context" means in general programming, then specifically in Spring Security.---

![alt text](image-1.png)

### What "context" actually means

In plain terms, a **context** is just a *container of information relevant to the current situation*. Think of it like a room — whoever is in that room can see everything inside it, without anyone outside needing to pass it around manually.

In Spring Security specifically, the context answers the question: **"Who is the person making this request right now, and what are they allowed to do?"**

---

### The three layers explained

`SecurityContextHolder` is the outermost shell — a global static class you can call from anywhere in your app. It uses `ThreadLocal` internally, which means every thread gets its own private slot of memory.

`SecurityContext` is what lives inside that slot — one per thread. It's like a backpack tied to the current request. It holds exactly one thing: the `Authentication` object.

`Authentication` is the ID card inside the backpack. It contains three key pieces: the **principal** (who the user is — usually a `UserDetails` object), their **authorities** (roles like `ROLE_USER`, `ROLE_ADMIN`), and a flag `isAuthenticated()` (true/false).

---

### Why ThreadLocal matters

When your server handles 100 simultaneous requests, each one runs on its own thread. `ThreadLocal` ensures thread 1 (Alice's request) and thread 2 (Bob's request) each get their own isolated context — they can never see each other's data. This is how Spring Security stays stateless and thread-safe without you passing the user around manually everywhere.

---

### The lifecycle in one sentence

> `JwtFilter` validates the token → creates an `Authentication` object → puts it in the `SecurityContext` → your controller can read it freely → after the response, Spring clears it automatically.

This is why you can call `SecurityContextHolder.getContext().getAuthentication()` from any service or controller and always get the currently logged-in user — no method parameters needed. The context acts as an invisible carrier for that request's lifetime.



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

## Q--> do one of these work at a time??

This is a fantastic question, and it is a very common misconception when first looking at the Spring Security architecture. 

The short answer is **no. They are not used one at a time.** They do not compete with each other. Instead, they are three gears in the exact same machine. They work **together in a strict sequence** to log a user in and keep them logged in. 

To understand how they connect, think of them like the **Security System of an MNC Corporate Office**.

---

### The Corporate Security Analogy

* **`UserDetailsService` (The HR Filing Cabinet):** This just holds the employee records. It knows your username, your hashed password, and your role (e.g., "Admin" or "Employee").
* **`AuthenticationManager` (The Security Desk Guard):** This is the active worker. When you walk into the building and hand over your ID, this guard takes your ID, walks over to the HR Filing Cabinet, and verifies that you are who you say you are.
* **`SecurityContextHolder` (The Lanyard / ID Badge):** Once the guard verifies you, they print a physical badge and hang it around your neck. As you walk around the building, any room you try to enter just looks at the badge on your neck to see if you have access.

---

### The Exact Workflow (How they trigger each other)

Here is the exact millisecond-by-millisecond breakdown of how these three components work together during a single login request:

**Step 1: The Request Arrives**
The user submits a JSON payload with `{"username": "mohit", "password": "123"}` to your backend.

**Step 2: The Guard Intercepts (`AuthenticationManager`)**
Spring Security catches this request before it ever reaches your Controller. It hands the raw username and password to the `AuthenticationManager`.
* *"Hey Manager, someone is trying to log in. Please verify them."*

**Step 3: Fetching the Record (`UserDetailsService`)**
The `AuthenticationManager` does not know how to talk to your MySQL database. So, it calls your `UserDetailsService`. 
* The `AuthenticationManager` asks: *"Hey UserDetailsService, go find the database record for the username 'mohit' and bring it back to me."*
* The `UserDetailsService` runs a `userRepository.findByUsername("mohit")` query and returns the official database record (including the hashed password and roles).

**Step 4: The Verification**
The `AuthenticationManager` looks at the raw password the user typed ("123"), hashes it, and compares it to the hashed password retrieved by the `UserDetailsService`. 
* If they don't match, it throws a `BadCredentialsException` (HTTP 401).
* If they DO match, the user is officially authenticated.

**Step 5: Printing the Badge (`SecurityContextHolder`)**
Now that the user is verified, the `AuthenticationManager` creates a successful `Authentication` object (the ID Badge). But where do we put it so the rest of the application can see it?
* It stuffs that object into the `SecurityContextHolder`. 

---

### Why the `SecurityContextHolder` is the ultimate goal

Steps 1 through 4 only happen **once** (during the actual login request). 

But Step 5 (`SecurityContextHolder`) is what powers your entire application afterward. Because the `SecurityContextHolder` is a `ThreadLocal` variable, it means that data is glued to the specific thread processing the user's request.

If the user tries to access a protected `@RestController` later on, Spring doesn't need to check the database again. It just runs this code:

```java
// Spring checks the "Lanyard" to see if you are allowed in!
Authentication auth = SecurityContextHolder.getContext().getAuthentication();

if (auth.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
    // Let the user in
}
```

**Summary:** The `AuthenticationManager` uses the `UserDetailsService` to verify a user, and if successful, saves the result in the `SecurityContextHolder`. They are a single, continuous pipeline!


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




