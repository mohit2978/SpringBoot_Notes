# Notes 

|Feature|	Encoding	|Encryption|
|-------|---------------|--------------|
Purpose	|Data format transformation (for readability or compatibility)	|Data security (protecting confidentiality)
Goal	|Make data understandable to other systems (not secure)	|Make data unreadable to unauthorized parties (secure)
Key	|No key involved; uses standard algorithms (e.g., Base64, URL Encoding)	|Uses a key (secret or public/private) to encrypt and decrypt data
Reversibility	|Easily reversible by anyone who knows the encoding method	|Reversible only by someone with the correct key
Use Cases	|Data transmission, URLs, file formats (e.g., Base64 for images)	|Protecting sensitive data like passwords, credit cards, communication
Example	|Base64("hello") = aGVsbG8=	|AES("hello", secret_key) = garbled_text

### TL;DR:

Encoding is for data readability (not secure).

Encryption is for data confidentiality (secure).

### Encoding (Base64):
Original Message: hello

Encoded (Base64): aGVsbG8=

Anyone can decode aGVsbG8= back to hello using any Base64 decoder.

### Encryption (using a password/key):
Original Message: hello

Encrypted (AES with key = "abc123"): b94d27b9934d3e08a52e52d7da7dabfa (just an example of scrambled text)

Without the correct key, no one can understand or decrypt this back to hello.

Simple Difference:
Encoding = wrapping the message in a readable way.

Encryption = locking the message with a key.


![alt text](<036 spring security-4 basic auth_250716_001652_1.jpg>) 


## Basic Auth in Spring Security — Complete ArchitectureI'll walk through the complete Basic Auth architecture in three diagrams — first the overall request flow, then the internal filter chain detail, and finally what happens inside the authentication pipeline.

### Diagram 1 — The full request lifecycleThe request either flows all the way through to your controller, or gets rejected at one of two gates — missing header or bad credentials. Now let's zoom into the internal component chain.

![alt text](image.png)

### Diagram 2 — Inside the authentication pipelineNow the full code behind each component above.

![alt text](image-1.png)
---

### What the header actually looks like

Basic Auth encodes credentials in every single request — there is no session or token:

```
Authorization: Basic dXNlcjpwYXNzd29yZA==
                      ↑
               Base64("user:password")
```

The `BasicAuthenticationFilter` simply calls `Base64.decode()` and splits on `:` to get the username and password. That's all — no crypto, no signing.

---

### Full Spring Boot implementation

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
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // no sessions
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults()); // enables BasicAuthenticationFilter
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

`.httpBasic()` registers `BasicAuthenticationFilter` into the filter chain. That single line is what makes Basic Auth work.

---

#### 2. `UserDetailsService` — loads user from DB

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
            .password(user.getPassword())        // BCrypt hash from DB
            .roles(user.getRole())               // e.g. "ADMIN", "USER"
            .accountLocked(!user.isActive())
            .build();
    }
}
```

---

#### 3. `User` entity and repository

```java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;     // stored as BCrypt hash
    private String role;
    private boolean active;
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

---

#### 4. A protected controller

```java
@RestController
@RequestMapping("/api")
public class SecuredController {

    @GetMapping("/hello")
    public String hello() {
        // get the currently authenticated user
        Authentication auth = SecurityContextHolder
            .getContext()
            .getAuthentication();
        return "Hello, " + auth.getName();
    }

    @GetMapping("/admin")
    @PreAuthorize("hasRole('ADMIN')")
    public String adminOnly() {
        return "Admin area";
    }
}
```

---

### What happens per request — the full step sequence

```
GET /api/hello
Authorization: Basic dXNlcjpwYXNzd29yZA==

1. BasicAuthenticationFilter runs
   └── reads Authorization header
   └── Base64.decode("dXNlcjpwYXNzd29yZA==") → "user:password"
   └── splits → username="user", password="password"
   └── creates UsernamePasswordAuthenticationToken (unauthenticated)

2. ProviderManager.authenticate(token)
   └── loops through providers
   └── DaoAuthenticationProvider supports this token type → delegates

3. DaoAuthenticationProvider
   ├── calls UserDetailsService.loadUserByUsername("user")
   │     └── hits DB, returns UserDetails with stored BCrypt hash
   └── calls BCryptPasswordEncoder.matches("password", "$2a$10$...")
         └── TRUE → authentication succeeds

4. SecurityContextHolder.getContext().setAuthentication(authToken)
   └── stores authenticated token for this thread/request

5. AuthorizationFilter
   └── checks user's roles against the required permissions
   └── allowed → request reaches @RestController
```

---

### What happens on failure

```
// Missing header
→ BasicAuthenticationFilter skips (no header found)
→ ExceptionTranslationFilter catches AccessDeniedException
→ BasicAuthenticationEntryPoint sends:
     HTTP/1.1 401 Unauthorized
     WWW-Authenticate: Basic realm="Realm"

// Wrong password
→ DaoAuthenticationProvider throws BadCredentialsException
→ ExceptionTranslationFilter catches it
→ Same 401 response sent back
```

The `WWW-Authenticate: Basic realm="Realm"` header in the response is what tells the browser to pop up a login dialog.

---

### Key characteristics of Basic Auth

| Characteristic | Detail |
|---|---|
| Stateless | Credentials sent on every request — no session |
| Encoding | Base64 only — NOT encryption |
| Security | Must use HTTPS — credentials readable if intercepted |
| Best for | Internal APIs, service-to-service, simple tools |
| Avoid for | Public-facing apps, mobile apps, anything needing logout |

The critical point: Basic Auth is not secure by itself. Base64 is trivially reversible — `dXNlcjpwYXNzd29yZA==` decodes to `user:password` instantly. It is only safe over HTTPS where the entire request is encrypted.



![alt text](<036 spring security-4 basic auth_250716_001652_2.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_3.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_4.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_5.jpg>)
 ![alt text](<036 spring security-4 basic auth_250716_001652_6.jpg>) 
 ![alt text](<036 spring security-4 basic auth_250716_001652_7.jpg>)