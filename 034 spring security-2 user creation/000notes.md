# Notes 

![alt text](034_spring_security_2_250716_003027_1.jpg) ![alt text](034_spring_security_2_250716_003027_2.jpg) ![alt text](034_spring_security_2_250716_003027_3.jpg) ![alt text](034_spring_security_2_250716_003027_4.jpg) ![alt text](034_spring_security_2_250716_003027_5.jpg) ![alt text](034_spring_security_2_250716_003027_6.jpg) ![alt text](034_spring_security_2_250716_003027_7.jpg) ![alt text](034_spring_security_2_250716_003027_8.jpg) ![alt text](034_spring_security_2_250716_003027_9.jpg) ![alt text](034_spring_security_2_250716_003027_10.jpg) ![alt text](034_spring_security_2_250716_003027_11.jpg)



 ![alt text](034_spring_security_2_250716_003027_12.jpg)
  ![alt text](034_spring_security_2_250716_003027_13.jpg) 

Here's the complete code for all files shown across the 4 images:

---

## `UserAuthEntity.java`

```java
@Entity
@Table(name = "user_auth")
public class UserAuthEntity implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private String role;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority(role));
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return true; }

    // Getters and Setters
    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getRole() {
        return role;
    }

    public void setRole(String role) {
        this.role = role;
    }
}
```

---

## `UserAuthEntityRepository.java`

```java
@Repository
public interface UserAuthEntityRepository extends JpaRepository<UserAuthEntity, Long> {

    Optional<UserAuthEntity> findByUsername(String username);
}
```

---

## `UserAuthEntityService.java`

```java
@Service
public class UserAuthEntityService implements UserDetailsService {

    @Autowired
    private UserAuthEntityRepository userAuthEntityRepository;

    public UserAuthEntity save(UserAuthEntity userAuth) {
        return userAuthEntityRepository.save(userAuth);
    }

    @Override
    public UserAuthEntity loadUserByUsername(String username) throws UsernameNotFoundException {
        return userAuthEntityRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));
    }
}
```

---

## `UserAuthController.java`

```java
@RestController
@RequestMapping("/auth")
public class UserAuthController {

    @Autowired
    private UserAuthEntityService userAuthEntityService;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @PostMapping("/register")
    public ResponseEntity<String> register(@RequestBody UserAuthEntity userAuthDetails) {
        // Hash the password before saving
        userAuthDetails.setPassword(passwordEncoder.encode(userAuthDetails.getPassword()));

        // Save user
        userAuthEntityService.save(userAuthDetails);
        return ResponseEntity.ok("User registered successfully!");
    }
}
```

---

## `SecurityConfig.java`

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/register").permitAll()
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable())
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

---

## Postman request body (Image 4)

```json
{
    "username": "my_username",
    "password": "111abc",
    "role": "ROLE_USER"
}
```

**Response:**
```
User registered successfully!
```

**DB result (`SELECT * FROM USER_AUTH`):**

| ID | PASSWORD | ROLE | USERNAME |
|----|----------|------|----------|
| 1 | $2a$10$euzihUhyp4exMejDkyDb0eK2q49sqTcG8EShOnT2GmaL7lxvleOfO | ROLE_USER | my_username |

---

## Required imports for all files

```java
// UserAuthEntity.java
import jakarta.persistence.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import java.util.Collection;
import java.util.List;

// UserAuthEntityRepository.java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

// UserAuthEntityService.java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

// UserAuthController.java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

// SecurityConfig.java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
```

---

## Quick summary of how files connect

```
POST /auth/register
        │
        ▼
UserAuthController        → receives JSON body as UserAuthEntity
        │
        ├── passwordEncoder.encode()   → hashes "111abc" → "$2a$10$..."
        │
        └── userAuthEntityService.save()
                │
                └── userAuthEntityRepository.save()  → INSERT into user_auth table

POST /any-protected-endpoint  (with Basic Auth header)
        │
        ▼
DaoAuthenticationProvider
        │
        ├── userAuthEntityService.loadUserByUsername()
        │       └── userAuthEntityRepository.findByUsername()  → SELECT from DB
        │
        └── passwordEncoder.matches(rawPassword, storedHash)  → true/false
```

---

## What this code implements

This is the **3rd approach** to Spring Security user storage: storing username + BCrypt-hashed password in a real database (production-recommended).

---

## The 4 files and what each does

### 1. `UserAuthEntity.java` — the user table mapped as a Java class

This is a JPA entity that maps to the `user_auth` database table. The key design decision is that it **implements `UserDetails`** directly.

Why implement `UserDetails`? Because during authentication, Spring Security's `DaoAuthenticationProvider` calls `loadUserByUsername()` and expects a `UserDetails` object back. If your entity doesn't implement `UserDetails`, you'd have to manually convert your entity into a `UserDetails` object every time. By implementing it directly on the entity, you skip that extra mapping step.

The class has three database columns: `username` (unique, not null), `password` (not null), and `role`. It also implements all the boolean methods from `UserDetails` — `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, `isEnabled()` — all returning `true` for simplicity. The `getAuthorities()` method wraps the role string into a `SimpleGrantedAuthority` object, which is what Spring Security uses internally for authorization checks.

---

### 2. `UserAuthEntityRepository.java` — database access layer

This is a standard Spring Data JPA repository interface. It extends `JpaRepository<UserAuthEntity, Long>`, which gives you all the basic CRUD operations for free.

The only custom method added is `findByUsername(String username)`, which returns an `Optional<UserAuthEntity>`. Spring Data auto-generates the SQL query from the method name — no SQL needed. This method is what `UserDetailsService` calls internally to look up a user by their username during login.

---

### 3. `UserAuthEntityService.java` — the UserDetailsService implementation

This class **implements `UserDetailsService`**, which is the interface Spring Security requires to load user data during authentication.

Why implement `UserDetailsService`? Because you are storing users in a database. Spring Security doesn't know how to talk to your database on its own. By implementing this interface and overriding `loadUserByUsername()`, you tell Spring Security exactly how to fetch a user — in this case, via the repository's `findByUsername()` method.

The `loadUserByUsername()` method calls `userAuthEntityRepository.findByUsername(username)`. If no user is found, it throws `UsernameNotFoundException("User not found")`. If found, it returns the `UserAuthEntity` object directly — and since `UserAuthEntity` already implements `UserDetails`, Spring Security accepts it without any extra conversion.

The class also has a `save()` method that simply delegates to `userAuthEntityRepository.save(userAuth)`. This is used during user registration.

---

### 4. `UserAuthController.java` — the registration endpoint

This is a REST controller mapped to `/auth`. It has one endpoint: `POST /auth/register`.

When a registration request comes in, the controller does two things before saving:

First, it **encodes the password** — it calls `passwordEncoder.encode(userAuthDetails.getPassword())` and sets the result back on the entity. This converts the plain-text password (e.g. `"111abc"`) into a BCrypt hash before it ever touches the database.

Second, it **saves the user** by calling `userAuthEntityService.save(userAuthDetails)`, which persists the entity to the database.

This is why the `/auth/register` endpoint must be **publicly accessible** — you can't require a user to be logged in before they can register.

---

### 5. `SecurityConfig.java` — security configuration

This class does two important things:

It registers a `BCryptPasswordEncoder` bean. This bean is what gets autowired into the controller and also what `DaoAuthenticationProvider` uses internally to verify passwords during login.

It configures the `SecurityFilterChain` to permit unauthenticated access to `/auth/register` (via `.permitAll()`), while requiring authentication for every other endpoint (via `.anyRequest().authenticated()`). It also disables CSRF (common for REST APIs) and enables HTTP Basic authentication.

---

## The complete flow

**Registration flow:**

```
POST /auth/register
    → SecurityConfig permits this URL without auth
    → UserAuthController.register() receives the request body
    → passwordEncoder.encode() hashes the plain password (e.g. "111abc" → "$2a$10$...")
    → userAuthEntityService.save() stores the entity in the DB
    → DB now has: username="my_username", password="$2a$10$...(hash)", role="ROLE_USER"
```

**Login flow (subsequent requests):**

```
Any protected request with credentials
    → UsernamePasswordAuthenticationFilter intercepts
    → Calls AuthenticationManager → ProviderManager
    → DaoAuthenticationProvider takes over
    → Calls UserAuthEntityService.loadUserByUsername(username)
    → Calls userAuthEntityRepository.findByUsername(username) → fetches from DB
    → Returns UserAuthEntity (which is also a UserDetails)
    → DaoAuthenticationProvider calls passwordEncoder.matches(rawPassword, storedHash)
    → If match → Authentication object marked as authenticated
    → Stored in SecurityContext → request proceeds
```

---

## Why this design is production-recommended

- Passwords are **never stored as plain text** — BCrypt hashes are one-way and salted
- The entity directly implementing `UserDetails` **eliminates boilerplate** mapping code
- `UserDetailsService` gives Spring Security a clean hook to load users **from any source** (DB, LDAP, etc.) without changing the authentication machinery
- The `SecurityConfig` follows the **industry standard** of making only the registration endpoint public, keeping everything else locked down

  ![alt text](034_spring_security_2_250716_003027_14.jpg) ![alt text](034_spring_security_2_250716_003027_15.jpg) ![alt text](034_spring_security_2_250716_003027_16.jpg) ![alt text](034_spring_security_2_250716_003027_17.jpg)



### Concept: User Creation vs. Token Issuance

In Spring Boot, **saving a user** and **issuing a JWT** are two completely separate responsibilities handled by different parts of the application.

Spring Boot and Spring Data JPA **do not know anything about JWT by default**.
If you simply save a user using JPA, it will **only store the user in the database**.

---

# 1️⃣ The Default Behavior (Just Database Persistence)

If you only call the repository to save a user, Spring Boot will simply insert the record into the database.

Example:

```java
public User createUser(User user) {
    user.setPassword(passwordEncoder.encode(user.getPassword()));
    return userRepository.save(user);
}
```

### What Happens Here

1. The password is hashed using `PasswordEncoder`.
2. Spring Data JPA generates an SQL query.
3. An `INSERT` statement is executed in the database.
4. The saved `User` object is returned.

Example SQL generated:

```sql
INSERT INTO users (id, email, password) VALUES (...);
```

### Result

The user is stored in the database, but:

* No JWT is generated
* No login session is created
* The frontend receives only the saved user object

This means the user would still need to **go to the login page and authenticate** after signup.

---

# 2️⃣ The Modern Application Behavior (Signup + Auto Login)

In modern applications, we usually **automatically log the user in right after registration**.

This is done by adding **extra logic in the Controller**.

Example:

```java
@PostMapping("/signup")
public ResponseEntity<AuthResponse> createUserHandler(@RequestBody User user) {

    // STEP 1: Store the user in the database
    User createdUser = new User();
    // set fields here...
    userRepository.save(createdUser);

    // STEP 2: Manually create an Authentication object
    Authentication authentication =
        new UsernamePasswordAuthenticationToken(
            createdUser.getEmail(),
            createdUser.getPassword()
        );

    SecurityContextHolder.getContext().setAuthentication(authentication);

    // STEP 3: Generate the JWT
    String token = jwtTokenProvider.generateJwtToken(authentication);

    // STEP 4: Return the JWT to the client
    return new ResponseEntity<>(new AuthResponse(token, true), HttpStatus.ACCEPTED);
}
```

---

# What Happens in This Flow

```text
User Signup Request
        ↓
User stored in database
        ↓
Authentication object created
        ↓
JWT generated
        ↓
JWT returned to frontend
```

Response example:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

Now the frontend **already has the token** and can immediately access protected APIs.

---

# Why Do We Do This?

This pattern is called:

```text
Auto-Login on Registration
```

Benefits:

* Better user experience
* No extra login step
* Immediate access to the application

Example UX flow:

```text
Signup Form
     ↓
Account Created
     ↓
JWT Generated
     ↓
User enters dashboard instantly
```

---

# Architecture Separation

Spring Boot separates responsibilities into layers.

## 1️⃣ Persistence Layer (JPA)

Responsible for **database operations**.

Action:

```java
userRepository.save(user);
```

Responsibilities:

* Hash password
* Generate primary key
* Insert data into database

Does **NOT** handle:

* Authentication
* JWT tokens
* Sessions

---

## 2️⃣ Presentation Layer (Controller)

Responsible for **handling HTTP requests and responses**.

Action:

```java
jwtTokenProvider.generateJwtToken(authentication);
```

Responsibilities:

* Authenticate the user
* Generate JWT
* Return token to frontend

---

# Interview Explanation

If asked in an interview:

**"Does Spring Boot automatically generate JWT when a user is created?"**

You can answer:

> "No. Spring Boot and Spring Data JPA only handle persistence. When a user is saved using userRepository.save(), it simply inserts the record into the database. JWT generation is handled separately in the authentication layer, usually inside the controller or authentication service after successful login."

---

# Key Takeaway

Saving a user and generating a JWT are **two independent operations**.

```text
userRepository.save(user)
        ↓
Stores user in database

jwtTokenProvider.generateJwtToken(authentication)
        ↓
Creates authentication token for API access
```

A modern application typically **combines these two steps during signup** to enable **automatic login after registration**.
