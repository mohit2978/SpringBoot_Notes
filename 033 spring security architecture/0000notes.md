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