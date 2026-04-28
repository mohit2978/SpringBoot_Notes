# Notes 

![alt text](<035 spring security -3_250716_003026_1.jpg>) ![alt text](<035 spring security -3_250716_003026_2.jpg>) ![alt text](<035 spring security -3_250716_003026_3.jpg>) ![alt text](<035 spring security -3_250716_003026_4.jpg>) ![alt text](<035 spring security -3_250716_003026_5.jpg>) ![alt text](<035 spring security -3_250716_003026_6.jpg>) ![alt text](<035 spring security -3_250716_003026_7.jpg>) ![alt text](<035 spring security -3_250716_003026_8.jpg>) ![alt text](<035 spring security -3_250716_003026_9.jpg>) ![alt text](<035 spring security -3_250716_003026_10.jpg>) ![alt text](<035 spring security -3_250716_003026_11.jpg>) ![alt text](<035 spring security -3_250716_003026_12.jpg>) ![alt text](<035 spring security -3_250716_003026_13.jpg>) ![alt text](<035 spring security -3_250716_003026_14.jpg>) ![alt text](<035 spring security -3_250716_003026_15.jpg>) ![alt text](<035 spring security -3_250716_003026_16.jpg>) ![alt text](<035 spring security -3_250716_003026_17.jpg>) ![alt text](<035 spring security -3_250716_003026_18.jpg>) ![alt text](<035 spring security -3_250716_003026_19.jpg>) ![alt text](<035 spring security -3_250716_003026_20.jpg>) ![alt text](<035 spring security -3_250716_003026_21.jpg>)



## Form-Based Login Architecture in Spring SecurityI'll break this into three diagrams — the big picture login flow, the internal component pipeline, and the session-based subsequent request flow.

### Diagram 1 — Login request flow (first visit)The key thing form login adds that Basic Auth doesn't have is the **redirect dance** — save where you were trying to go, redirect to `/login`, then after success send you back to the original URL. That's `SavedRequestAwareAuthenticationSuccessHandler` doing its job.

![alt text](image.png)

Now let's look inside the authentication pipeline — what happens between "POST /login received" and "user is authenticated."

### Diagram 2 — Internal authentication pipelineAfter a successful login, Spring creates a session and stores the `SecurityContext` in it. Every subsequent request uses that session — no re-authentication needed. Here's how that looks.

![alt text](image-1.png)

### Diagram 3 — Subsequent request flow (already logged in)No DB call, no password check — just a session lookup. This is the fundamental difference between form login and Basic Auth: form login is **stateful**, Basic Auth is **stateless**.

![alt text](image-2.png)

Great question — this is a really common point of confusion. Let me clear it up precisely.

---

### The confusion: "stateless" doesn't mean "no DB call"

You're right that Basic Auth **does** hit the DB on every request. The word **stateless** refers to something different — it refers to **where the server stores knowledge about you between requests.**

![alt text](image-3.png)
---

### What "state" actually means here

Think of it this way:

> **Stateful** = the server remembers you between requests (via a session)
> **Stateless** = the server remembers nothing — you prove yourself fresh every single time


---

### Full implementation

#### 1. `SecurityConfig`

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")                      // custom login page URL
                .loginProcessingUrl("/login")             // POST target Spring intercepts
                .defaultSuccessUrl("/dashboard", false)   // false = use SavedRequest if exists
                .failureUrl("/login?error=true")
                .usernameParameter("username")            // matches your HTML <input name="">
                .passwordParameter("password")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout=true")
                .invalidateHttpSession(true)              // kills the session
                .deleteCookies("JSESSIONID")
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)                       // one session per user
                .maxSessionsPreventsLogin(false)          // new login kicks out old session
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

#### 2. Login page controller + Thymeleaf template

```java
@Controller
public class LoginController {

    @GetMapping("/login")
    public String loginPage() {
        return "login";   // renders login.html
    }
}
```

```html
<!-- templates/login.html -->
<form th:action="@{/login}" method="post">
    <input type="hidden" th:name="${_csrf.parameterName}"
                         th:value="${_csrf.token}"/>

    <input type="text"     name="username" placeholder="Username"/>
    <input type="password" name="password" placeholder="Password"/>

    <div th:if="${param.error}">
        Invalid username or password.
    </div>
    <div th:if="${param.logout}">
        You have been logged out.
    </div>

    <button type="submit">Log In</button>
</form>
```

The CSRF token is critical — Spring Security blocks any POST without it by default.

---

#### 3. `UserDetailsService`

```java
@Service
public class MyUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepo.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())   // BCrypt hash
            .roles(user.getRole())
            .build();
    }
}
```

---

#### 4. Protected controller

```java
@Controller
public class DashboardController {

    @GetMapping("/dashboard")
    public String dashboard(Model model) {
        Authentication auth = SecurityContextHolder
            .getContext().getAuthentication();
        model.addAttribute("user", auth.getName());
        return "dashboard";
    }

    @GetMapping("/admin")
    @PreAuthorize("hasRole('ADMIN')")
    public String admin() {
        return "admin";
    }
}
```

---

### The complete step-by-step sequence

```
1. GET /dashboard (no session)
   └── ExceptionTranslationFilter detects unauthenticated
   └── LoginUrlAuthenticationEntryPoint → 302 redirect /login
   └── RequestCache saves /dashboard as the target URL

2. GET /login
   └── Spring serves your login.html template

3. POST /login {username, password, _csrf}
   └── UsernamePasswordAuthenticationFilter intercepts
   └── Creates UsernamePasswordAuthenticationToken (unauthenticated)
   └── Calls ProviderManager.authenticate(token)
       └── DaoAuthenticationProvider
           ├── UserDetailsService.loadUserByUsername("alice")  → DB lookup
           └── BCrypt.matches("raw_pass", "$2a$10$...")        → true/false

4a. Failure:
    └── AuthenticationFailureHandler → redirect /login?error

4b. Success:
    └── SecurityContextHolder.getContext().setAuthentication(authToken)
    └── HttpSession created, SecurityContext saved as SPRING_SECURITY_CONTEXT
    └── JSESSIONID cookie sent to browser
    └── SavedRequestAwareAuthenticationSuccessHandler → redirect /dashboard

5. GET /dashboard (with JSESSIONID cookie)
   └── SecurityContextPersistenceFilter restores SecurityContext from session
   └── No DB call, no password check
   └── AuthorizationFilter checks roles → allowed
   └── Controller executes
   └── At end of request: SecurityContext saved back to session
```

---

### Key components and their responsibilities

| Component | What it does |
|---|---|
| `UsernamePasswordAuthenticationFilter` | Intercepts `POST /login`, extracts credentials |
| `LoginUrlAuthenticationEntryPoint` | Redirects unauthenticated users to `/login` |
| `RequestCache` | Remembers the original URL before the redirect |
| `DaoAuthenticationProvider` | Verifies credentials using `UserDetailsService` + `PasswordEncoder` |
| `AuthenticationSuccessHandler` | Redirects to original URL after success |
| `AuthenticationFailureHandler` | Redirects to `/login?error` on failure |
| `SecurityContextPersistenceFilter` | Loads/saves `SecurityContext` from/to `HttpSession` |
| `SessionManagementFilter` | Handles concurrent sessions, session fixation protection |
| `LogoutFilter` | Intercepts `/logout`, invalidates session, clears context |

---

### Form login vs Basic Auth — key differences

| | Form login | Basic Auth |
|---|---|---|
| State | Stateful — uses `HttpSession` | Stateless — credentials every request |
| Credentials sent | Once (POST /login) | Every single request |
| Token | `JSESSIONID` cookie | `Authorization: Basic` header |
| Login page | Custom HTML form | Browser native popup |
| Logout | Explicit `/logout` endpoint | No real logout mechanism |
| CSRF protection | Needed (POST form) | Not needed (no session) |
| Best for | Web apps with browsers | APIs, service-to-service |

