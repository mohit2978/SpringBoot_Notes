# Notes

- We have already covered different type of Authentications.
- And with that, we have also covered, how at Security Filter layer itself we can do authorization.

![alt text](image.png)

- But in large scale applications, where we might have 100s of APIs, in that scenarios managing the roles at Security Filter might become difficult and cause scalable issue.

That’s where, Annotation based "Role Based Authorization" comes into the picture. And these annotations are used within our Controller class.

![alt text](image-1.png)


![alt text](image-2.png)


##  Dynamic User creation(Already seen)

![alt text](image-3.png)

---

![alt text](image-4.png)

---

![alt text](image-5.png)

---
![alt text](image-6.png)

---

![alt text](image-7.png)

Now, we should be able to create user, and in this one User can have 1 role like ROLE_ADMIN, ROLE_USER etc.
But we can give many permissions like ORDER_READ, SALES_DELETE etc.…  (more granular level permissions)

![alt text](image-8.png)

![alt text](image-9.png)

![alt text](image-10.png)

Above we have created user, who has both the permissions ROLE_USER and ORDER_READ, so API is successfully executed.

![alt text](image-11.png)

Now, when we try to access /sales API, which above user do not have permission, request has thrown exception.

![alt text](image-12.png)

![alt text](image-13.png)


### SecurityExpressionRoot.java

![alt text](image-15.png)
![alt text](image-14.png)

Now, one question comes to mind is, how this Authorization methods invokes before invocation of the Controller method.

Its because of INTERCEPTORS

![alt text](image-16.png)

@PreAuthorize Annotation is intercepted by 

AuthorizationManagerBeforeMethodInterceptor

What and how Interceptors works and how they are different from filters? 
We have already covered both the topics in depth. 

![alt text](image-17.png)


![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

Let's see with an example

Created 2 Users

![alt text](image-21.png)

![alt text](image-22.png)

![alt text](image-23.png)

![alt text](image-24.png)

Now, try to invoke this api with "b_users" credentials

![alt text](image-25.png)

Now, try to invoke this api with "a_users" credentials

![alt text](image-26.png)





# Keycloak

## Keycloak + Spring Boot — Step by Step

---

### What Keycloak does in one line:
> *"Keycloak is an external auth server — it handles login, issues JWT tokens, Spring Boot just validates those tokens."*

---

### The flow first — understand this clearly:

```
User → Login request → Keycloak (validates credentials)
                     → issues JWT token
User → API call with JWT → Spring Boot (validates token with Keycloak)
                         → allows/denies based on roles
```

Spring Boot **never stores passwords** — Keycloak handles all of that.

---

### Step 1 — Setup Keycloak Server

Run via Docker:
```bash
docker run -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest start-dev
```

---

### Step 2 — Configure Keycloak Admin Console

Go to `http://localhost:8080`

1. **Create a Realm** — e.g., `salonease-realm`
   - Realm = isolated namespace (like a tenant)

2. **Create a Client** — e.g., `salonease-client`
   - Client = your Spring Boot app
   - Set `Client Authentication = ON` (confidential client)
   - Set Valid Redirect URIs = `http://localhost:8081/*`

3. **Create Roles** — e.g., `CUSTOMER`, `SALON_OWNER`, `ADMIN`

4. **Create Users** — assign roles to users

5. **Get Client Secret** — from Credentials tab of client
   - You'll need this in `application.properties`

---

### Step 3 — Add dependency in `pom.xml`

```xml
<!-- Spring OAuth2 Resource Server -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

---

### Step 4 — `application.properties`

```properties
# Tell Spring where Keycloak's public keys are to validate JWT
spring.security.oauth2.resourceserver.jwt.issuer-uri=\
  http://localhost:8080/realms/salonease-realm

# Spring auto-fetches JWKS (public keys) from:
# http://localhost:8080/realms/salonease-realm/.well-known/openid-configuration
```

That's all Spring needs to **validate tokens**.

---

### Step 5 — Security Config

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/salon/**").hasRole("SALON_OWNER")
                .requestMatchers("/booking/**").hasRole("CUSTOMER")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthConverter())
                )
            );

        return http.build();
    }
}
```

---

### Step 6 — JWT Converter (most important step)

**Problem:** Keycloak puts roles inside JWT like this:
```json
{
  "realm_access": {
    "roles": ["CUSTOMER", "SALON_OWNER"]
  }
}
```

But Spring Security expects roles as:
```
ROLE_CUSTOMER, ROLE_SALON_OWNER
```

So you need a **converter** to extract roles from Keycloak's JWT structure:

```java
@Component
public class JwtAuthConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    @Override
    public AbstractAuthenticationToken convert(Jwt jwt) {
        
        // Extract roles from realm_access claim
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        List<String> roles = (List<String>) realmAccess.get("roles");

        // Convert to Spring GrantedAuthority with ROLE_ prefix
        List<SimpleGrantedAuthority> authorities = roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());

        return new JwtAuthenticationToken(jwt, authorities);
    }
}
```

Register it in SecurityConfig:
```java
.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
```

---

### Step 7 — Use roles in controller

```java
@RestController
public class BookingController {

    @GetMapping("/booking/my")
    @PreAuthorize("hasRole('CUSTOMER')")
    public List<Booking> getMyBookings() { ... }

    @DeleteMapping("/salon/booking/{id}")
    @PreAuthorize("hasRole('SALON_OWNER') or hasRole('ADMIN')")
    public void cancelBooking(@PathVariable Long id) { ... }
}
```

Enable `@PreAuthorize` by adding on config class:
```java
@EnableMethodSecurity
```

---

### Step 8 — How login actually works

Your **frontend** calls Keycloak directly:
```
POST http://localhost:8080/realms/salonease-realm/protocol/openid-connect/token

Body:
  grant_type=password
  client_id=salonease-client
  client_secret=xxxxx
  username=mohit
  password=pass123
```

Keycloak returns:
```json
{
  "access_token": "eyJhbGci...",  ← JWT token
  "refresh_token": "eyJhbGci...",
  "expires_in": 300
}
```

Frontend sends `access_token` in every API call:
```
Authorization: Bearer eyJhbGci...
```

---

### Step 9 — What Spring Boot does on each request

```
Request with JWT
    ↓
JwtAuthenticationFilter (Spring auto-adds this)
    ↓
Fetches Keycloak public key (JWKS endpoint) — cached
    ↓
Validates JWT signature + expiry
    ↓
Calls your JwtAuthConverter → extracts roles
    ↓
Sets SecurityContext with user + roles
    ↓
Your controller runs (or 403 if role mismatch)
```

---

### Complete picture in one diagram:

```
[Frontend]
    │ POST /token (username+password)
    ▼
[Keycloak] ──── issues JWT ────→ [Frontend]
                                      │
                                      │ GET /booking/my
                                      │ Authorization: Bearer <JWT>
                                      ▼
                                [Spring Boot]
                                      │
                                  validates JWT
                                  (using Keycloak public key)
                                      │
                                  extracts roles
                                      │
                                  allows/denies
```

---

### Interview one liner:

> *"Spring Boot acts as a Resource Server. It validates JWT tokens issued by Keycloak using public keys fetched from Keycloak's JWKS endpoint. A custom JwtAuthConverter extracts roles from the `realm_access` claim and maps them to Spring's GrantedAuthority. No passwords or user data is stored in Spring Boot."*

---




