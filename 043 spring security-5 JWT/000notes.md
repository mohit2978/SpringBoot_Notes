
## Why jwt disbales CSRF

CSRF protection defends against one specific attack, and a JWT sent in the `Authorization` header doesn't have that weakness. That's why people turn it off.

**How CSRF works:** With session-based auth, the browser **automatically attaches the session cookie** to every request to your domain. If you're logged in to `yourbank.com` and visit `evil.com`, a hidden form on that page can post to `yourbank.com/transfer`. The browser sends your cookie along, and the server thinks it's you. A CSRF token stops this because `evil.com` can't read or guess the token.

**Why JWT is different:** In a typical setup, the frontend stores the JWT (in memory or localStorage) and adds it manually:

```
Authorization: Bearer <token>
```

The browser never attaches this header on its own. A forged request from `evil.com` won't include it, so the request is unauthenticated and gets rejected. The attack has nothing to exploit, and a CSRF token adds no protection.

In Spring Security, you'd typically do:

```java
http
    .csrf(csrf -> csrf.disable())
    .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    ...
```

**Important caveat:** This only holds if the JWT is sent via the header. If you store the JWT in a **cookie** and the server reads it from there, the browser attaches it automatically, so CSRF is a risk again. Keep CSRF protection on in that case (or use `SameSite` cookies plus a CSRF token).

**Trade-off:** Storing JWTs in localStorage avoids CSRF but exposes them to **XSS**, since any injected script can read them. HttpOnly cookies protect against XSS but bring CSRF back. Which one you pick depends on which risk you handle better.

**Summary:**
- Header-based JWT, stateless: CSRF can be disabled safely.
- Cookie-based JWT or session: keep CSRF enabled.



# JWT (JSON Web Token)

## What it is

A JWT is a compact, URL-safe string that carries **claims** (data about a user or session) and is **digitally signed**, so the server can verify it hasn't been tampered with. It lets a server authenticate requests **without storing session state**.

## Structure

A JWT has three parts separated by dots:

```
xxxxx.yyyyy.zzzzz
Header.Payload.Signature
```

Each part is Base64URL-encoded.

**1. Header** says which algorithm signs the token:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**2. Payload** holds the claims:
```json
{
  "sub": "user123",
  "name": "Mohit",
  "role": "ADMIN",
  "iat": 1731145000,
  "exp": 1731148600
}
```
Common registered claims:
- `sub`: subject (usually the user ID)
- `iat`: issued at
- `exp`: expiry time
- `iss`: issuer
- `aud`: intended audience

**3. Signature** is computed as:
```
HMACSHA256(base64(header) + "." + base64(payload), secretKey)
```
If anyone changes the header or payload, the signature no longer matches and the token is rejected.

**Key point:** The payload is **encoded, not encrypted**. Anyone can decode and read it. Never put passwords or sensitive data in it.

## Authentication flow

1. The client sends credentials (`POST /login`).
2. The server validates them and creates a JWT signed with its secret or private key.
3. The server returns the token.
4. The client stores it and sends it with each request: `Authorization: Bearer <token>`.
5. The server verifies the signature and `exp`, reads the claims, and processes the request.

No database or session lookup is needed. That's what makes JWT **stateless** and a good fit for scaling across multiple servers or microservices.

## Signing algorithms

| Type | Algorithm | How it works |
|---|---|---|
| Symmetric | HS256 | One shared secret signs and verifies |
| Asymmetric | RS256, ES256 | Private key signs, public key verifies |

Asymmetric is better when many services must verify tokens but only one (the auth server) should issue them.

## Access and refresh tokens

JWTs usually can't be revoked before they expire, so the common pattern is:
- **Access token**: short-lived (5-15 min), sent on every request.
- **Refresh token**: long-lived (days), used only to get new access tokens. It is often stored server-side so it can be revoked.

## Spring Boot example (jjwt)

```java
// Generate
String token = Jwts.builder()
    .subject(user.getUsername())
    .claim("role", user.getRole())
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000))
    .signWith(secretKey)
    .compact();

// Validate and read
Claims claims = Jwts.parser()
    .verifyWith(secretKey)
    .build()
    .parseSignedClaims(token)
    .getPayload();
String username = claims.getSubject();
```

In Spring Security, you typically write a `OncePerRequestFilter` that extracts the token from the header, validates it, and sets the `Authentication` in the `SecurityContext`.

## Advantages

- Stateless and scalable
- Self-contained, so no DB hit per request
- Works well across services and domains
- Works with mobile and SPA clients

## Drawbacks and pitfalls

- **Hard to revoke.** A stolen token is valid until it expires. Mitigate with short expiry, a denylist, or refresh-token rotation.
- **Larger than session IDs**, since it is sent on every request.
- **Payload is readable**, so don't store secrets in it.
- **Storage dilemma.** localStorage is exposed to XSS. Cookies need CSRF protection. HttpOnly cookies with `SameSite` are often the safer choice.
- **Stale data.** If a user's role changes, the old token still carries the old role until it expires.

## Security best practices

- Always use HTTPS.
- Use strong secrets (256+ bits for HS256).
- Always check `exp`, and `iss`/`aud` where relevant.
- Pin the expected algorithm. Reject `alg: none` and algorithm-confusion attacks (this is a classic JWT vulnerability).
- Keep access tokens short-lived.
- Don't put sensitive data in the payload.

## JWT vs sessions

| | Session | JWT |
|---|---|---|
| State | Server-side | Client-side (stateless) |
| Revocation | Easy | Hard |
| Scaling | Needs shared store | Easy |
| Size | Small ID | Larger token |


---

Then the token is still rejected, because you can't produce a valid signature without the server's secret key.

## Why it fails

The signature isn't just a checksum. It's computed with a **secret key** that only the server knows:

```
signature = HMACSHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

If you change the payload (say `"role": "USER"` to `"role": "ADMIN"`) and also replace the signature, you have two options, and both fail:

1. **Recompute it yourself.** You'd need `SECRET_KEY`. Without it, any signature you generate is just a guess.
2. **Make one up.** When the server receives the token, it recomputes the signature from the header and payload using its real secret. The result won't match yours, so it returns `401 Unauthorized` (in jjwt, a `SignatureException`).

## Example

Original token, issued by the server:
```
header.{"role":"USER"}.signature_A
```

You change it to:
```
header.{"role":"ADMIN"}.signature_B   (signature_B is your fake)
```

The server computes the correct signature for `{"role":"ADMIN"}`, gets `signature_C`, sees that `signature_B ≠ signature_C`, and rejects the token.

## When forgery does become possible

The mechanism is sound, but implementation mistakes can break it:

- **Weak or guessable secret** (like `"secret"` or `"password123"`). Attackers can brute-force it offline from a single token, then forge any token they like. Use a long random secret (256+ bits).
- **`alg: none` attack.** If the library accepts unsigned tokens, an attacker sets the header to `"alg": "none"` and drops the signature. Modern libraries reject this by default, but always pin the expected algorithm.
- **Algorithm confusion (RS256 → HS256).** With asymmetric keys, an attacker may sign a token using the *public key* as an HMAC secret, and a badly configured server accepts it. Again, enforce the expected algorithm server-side.
- **Leaked secret key.** Anyone who has it can mint valid tokens.

## Takeaway

JWT security rests on one thing: **the secret (or private key) stays secret**. Anyone can read and edit the payload, but without the key they can't produce a signature the server will accept.


---

![alt text](image-1.png)

We provide refresh token and then we validate refresh token and then after validating we generate new access token!!

# Step1:User Creation(dynamically) (Very important part)

We have already seen how User Creation is done!!


## 1. UserRegisterEntity (the user table)

```java
@Entity
@Table(name = "user_register")
public class UserRegisterEntity implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private String role;

    public void setPassword(String password) { this.password = password; }

    public String getRole() { return role; }

    public void setRole(String role) { this.role = role; }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority(role));
    }

    @Override
    public String getPassword() { return password; }

    @Override
    public String getUsername() { return username; }
}
```

- `@Entity` and `@Table` map this class to a database table called `user_register`.
- `id` is the primary key, and the database auto-increments it (`IDENTITY`).
- `username` must be unique and can't be null. `password` can't be null.
- `implements UserDetails` is what matters for security. `UserDetails` is the interface Spring Security uses to represent a user, so your entity is both a DB row and a Spring Security user.
  - `getUsername()` and `getPassword()` tell Spring the login name and the stored (hashed) password.
  - `getAuthorities()` turns your `role` string into a `GrantedAuthority`, which Spring uses for permission checks like `hasRole(...)`.
  - The other `UserDetails` methods (`isEnabled()`, `isAccountNonLocked()`, etc.) have default implementations that return `true`, so you don't have to write them.

## 2. UserRegisterEntityRepository (DB access)

```java
@Repository
public interface UserRegisterEntityRepository extends JpaRepository<UserRegisterEntity, Long> {

    Optional<UserRegisterEntity> findByUsername(String username);
}
```

- Extending `JpaRepository<UserRegisterEntity, Long>` gives you `save`, `findById`, `findAll`, `delete`, etc. for free. `Long` is the type of the ID.
- `findByUsername` is a **derived query**. Spring reads the method name and generates `SELECT * FROM user_register WHERE username = ?` for you.
- It returns `Optional` because the user may not exist.

## 3. UserRegisterEntityService (loads users for Spring Security)

```java
@Service
public class UserRegisterEntityService implements UserDetailsService {

    @Autowired
    private UserRegisterEntityRepository userAuthEntityRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userAuthEntityRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("user not found"));
    }

    public UserDetails save(UserRegisterEntity userRegisterEntity) {
        return userAuthEntityRepository.save(userRegisterEntity);
    }
}
```

- `implements UserDetailsService` makes this the class Spring Security calls when it needs to look up a user by username during authentication.
- `loadUserByUsername` fetches the user from the DB. If they don't exist, it throws `UsernameNotFoundException`, which Spring treats as a failed login. It can return the entity directly because the entity implements `UserDetails`.
- `save` stores a new user. (IntelliJ underlines it only because the return value isn't used anywhere. That's a warning, not an error.)

## 4. UserController (the APIs)

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @Autowired
    UserRegisterEntityService userRegisterEntityService;

    @Autowired
    PasswordEncoder passwordEncoder;

    /*
    using this API to register the user into the system. username, password, role.
     */
    @PostMapping("/user-register")
    public ResponseEntity<String> register(@RequestBody UserRegisterEntity userRegisterDetails) {
        // Hash the password before saving
        userRegisterDetails.setPassword(passwordEncoder.encode(userRegisterDetails.getPassword()));

        userRegisterEntityService.save(userRegisterDetails);

        return ResponseEntity.ok("User registered successfully!");
    }

    @GetMapping("/users")
    public String getUsersDetails() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return "fetched user details successfully";
    }
}
```

- `@RestController` with `@RequestMapping("/api")` means every endpoint here starts with `/api` and returns data (not HTML pages).
- **`POST /api/user-register`**:
  1. `@RequestBody` converts the incoming JSON (`username`, `password`, `role`) into a `UserRegisterEntity`.
  2. The plain-text password is replaced with its BCrypt hash, so the real password is never stored.
  3. The user is saved and a success message is returned.
- **`GET /api/users`**: reads the currently logged-in user from `SecurityContextHolder`, which is where Spring keeps "who is making this request". Right now the `authentication` variable isn't used; the method just returns a fixed string.

## 5. SecurityConfig (security rules)

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

        http.authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/user-register").permitAll()
                        .anyRequest().authenticated())
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

- `passwordEncoder()` registers BCrypt as a bean. That's why the controller can `@Autowired` it. BCrypt adds a random salt, so the same password produces a different hash each time.
- `securityFilterChain` defines the rules:
  - `/api/user-register` is open to everyone (`permitAll`), because new users have no account yet.
  - Every other URL requires the user to be authenticated.
  - `STATELESS` tells Spring not to create HTTP sessions, so each request must prove who it is on its own. This is the setup you use with JWT.
  - `csrf.disable()` is safe here for the reason we covered earlier: no session cookies, so there's nothing for CSRF to exploit. (The yellow underline is only IntelliJ warning you that CSRF is off.)

## How it fits together right now

Registration works fully: JSON goes in, the password is hashed, and the user is saved to the `user_register` table.

But **`/api/users` will always return 401/403 at the moment.** It requires authentication, and this config has no way to log in: no `httpBasic()`, no form login, and no JWT filter yet. To make it work, either:
- add `.httpBasic(Customizer.withDefaults())` to the chain for quick testing with username and password, or
- add the JWT login endpoint and filter from my previous message.

![alt text](image-7.png)

![alt text](image-8.png)

Till now we have seen how user is generated dynamically!!

## Step-2 Token Generation(When we login)

For form-based and basic-auth we used to have implemenataion in SpringBoot but for JWT no implementation, so we need to provide it and every engineer implement in his own way!!

![alt text](image-9.png)

JWT we need to provide everything AuthenticationProvider ,Filters and everything .

We will try to stick to Security Framework only to implement the JWT functionality.So we not going to create any apis to generate token or apis for refresh token in controller!!We will put in filters only .

>Note:Remember filter creates Authentication Object. So here we will create Authentication object from filter.

>Note:AuthenticationManager gets the AUthentication object and delegate to proper AuthenticationProvider.Provider inteacts with Db through UserDetails Service.

>Note: when AuthenticationManager gives back Authentication object  to Filter ,Filter stores that Object to SecurityContext.

![alt text](image-10.png)

![alt text](image-11.png)

### Very Imp

![alt text](image-12.png)

Security filter creates `Authentication` object from UserName,password or token or sessionID!!Each filter creates it own Authentication object like UsrerName Password has different , Basic auth has different and so on!!

Till now `Authentication` is false!!

Then securityFilter pass this `Authentication` Object to ProviderManager!!

ProviderManager gives the request which has Authentication Object to one of AuthenticationProvider like DAOAuthenticationProvider and so on by testing each Provider which one can accept the request as ProvideManager has a list of Provider!!

### ProviderManager.java (framework code)

![alt text](image-13.png)

### DaoAuthenticationProvider.java (framework code)

![alt text](image-14.png)

### What we need to do
![alt text](image-15.png)

![alt text](image-16.png)



![alt text](image-17.png)

### Let us code 



```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId><!-- for json processing-->
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

need jwt dependency to generate token so use jjwt-api (it just provide api), jjwt-impl (implementation of jjwt to sign the key ,get the token)(can use other impl too),jjwt-jackson (payload has Key-value in json so for json processing using it)

---

Let us create a class first

![alt text](image-21.png)

## Custom Filter 

`OnePerRequestFilter` make sure that in one request any filter should not run twice!!

See here how Filter get apis!! We telling this filter only works for `/generate-token` end-point!!

![alt text](image-19.png)

UserNamePasswordAuthenticationToken we are getting as it is supported by DAOAuthentricationProvider!! 


Then we have passed to AuthneticationManager which will deligate to 
DAOAuthentricationProvider and which checks from DB whether credentials are valid or not !!If matches then it will put true in Isauthneticated()!!

If Authenticated we generating the token!!

This we getting from jjwt-impl all the below Apis of jwt is exposed by that




```java


@Component
public class JwtUtil {

    private final SecretKey key;
    private final long accessExpirationMs;

    public JwtUtil(@Value("${jwt.secret}") String secret,
                   @Value("${jwt.access-expiration-ms}") long accessExpirationMs) {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
        this.accessExpirationMs = accessExpirationMs;
    }

    // ---------- GENERATION ----------
    public String generateAccessToken(UserDetails user) {
        List<String> roles = user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList();

        Date now = new Date();
        return Jwts.builder()
                .subject(user.getUsername())                       // "sub"
                .claim("roles", roles)                             // custom claim
                .issuedAt(now)                                     // "iat"
                .expiration(new Date(now.getTime() + accessExpirationMs)) // "exp"
                .signWith(key)                                     // HS256 signature
                .compact();                                        // header.payload.signature
    }

    // ---------- VALIDATION ----------
    // Throws JwtException if the signature is invalid, the token is expired, or it is malformed
    public Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    public String extractUsername(String token) {
        return parseClaims(token).getSubject();
    }

    public boolean isTokenValid(String token, UserDetails user) {
        return extractUsername(token).equals(user.getUsername());
    }
}
```

Here JWT is created!!We using HMAC algo , so same key is used in encrption and decryption!!Here we have put key here only , but In prod that should not be case ,we must have key somewhere else also should use different keys!!



##### Config

DAOAuthenticationProvider also we need to provide!!So see below for that!

![alt text](image-22.png)

Now see below we creating object of filter as Filter was not annotated with @Bean so we  need to create its object ourself!!

Also we need to add AuthenticationManager's List of Provider we provide only DAO one!!

 Here we have created a new list but we can also get the AuthenticationMananger and add out provider to list!!


![alt text](image-23.png)

 see here we have `ProviderManager` which is implementation of `AuthenticationManager` and we giving it a list 


 Another way is get `ProviderManager` created by springboot and update list and add your `AuthenticationProvider`

addFilterBefore add our filter to filter we provided before `UserNamePasswordAuthneticationFilter`!!

![alt text](image-24.png)

![alt text](image-25.png)

In response Header we getting JWT token as we have configuered!!

So see we have not put `/generate-token` apu in controller ,It is in Filter part!!


## Step-3 Token Validation

User try to access any api ,so User first goes to SecurityFilter which validates token and then token is passed to Controller if token is valid! 



![alt text](image-26.png)

`/api/users` is protected API

So we need to create filter again!!
![alt text](image-27.png)



If token is not null then we ceating `JWtAuthenticationToken` which is child of `AbstractAuthenticationToken` which is child of `Authentication`!! And initially we have put `Authenticated` as false!!

Now to authenticate we need a provider that `AuthenticationManager` can delegate `authenticate()` request to.So we created `JwtAuthenicationProvider` as springboot doesnt have anything to understand JWt and added this `provider` as in list to manager in securityConfig.

So now Authentication manager goes to both of the `AuthenticationProvider` 1st to `DAOAuthenticationProvider` and asks do it supports `JwtAuthenticationTokem` object it says no and then to `JwtAuthenicationProvider` and asks do it support and it says yes.

![alt text](image-28.png)

Initially we set up isAuthenticated as false.

After validation of JWT token ,we storing Authentication object in SecurityContext.`filterChain.doFilter()` makes sure SecurityChain goes on and not end!!

Now `DAOAutheticatorProvider` do not understand JWT Authentication so we create another `AuthenticatorProvider`!!

![alt text](image-29.png)

here when we return `UserNamePasswordAuthenticationToken` itinternally set up isAuthhenticated to true

![alt text](image-30.png)

We added `JWTAuthenticationProvider` in List!!

![alt text](image-31.png)

This is where validation is done !and from here we returining userName which we have put in subject!!

And In AuthenticationProvider we are checking if this username is null that means Bad credentials!!


and then we returing `Authentication` Object of type `UserNamePasswordAutthenticationToken` which is further passed to controller!!

![alt text](image-32.png)

![alt text](image-35.png)

![alt text](image-33.png)

![alt text](image-34.png)

![alt text](image-36.png)

403 forbidden is given!!

## Step-4 Refresh Token

After 15 min again provide username-password ,so instead increase the time for JWT like 1 day or 2 day , then their might be case where JWT is compromised ,as no way to delete the token,so it is recommended JWT to be short-lived so insted we use `refresh-token`!!

Refresh token is used to get new token without putting credentails again and again so need to add filter here too!!

we want refresh token along with time we get token.

![alt text](image-37.png)


![alt text](image-38.png)

![alt text](image-39.png)

 we genrating one more token sam way but this refresh token is for 7 days.

We can put refreshToken in ResponseBody so user can store in localStorage. And another way is to put in cookie.

We putting RefreshToken in Cookie !!We put `setHttpOnly(true)` so no JS code can access it!! Also `setSecure(true)` so only Https can access it!! We also put `setPath()` so for only this path ,Cookie is sent else browser will send for all request!!


generally browser send cookie for all the request with `setPath()`,we telling browser to send cookie only for this path

---

Now we need to write our own refresh token

![alt text](image-40.png)

![alt text](image-41.png)

Now here we no need to reach controller so we do not do `filterChain.doFilter()` in else part

---

![alt text](image-42.png)

---

![alt text](image-43.png)

---

![alt text](image-44.png)

---

![alt text](image-45.png)

![alt text](image-46.png)

![alt text](image-47.png)

---

![alt text](image-48.png)

![alt text](image-49.png)

![alt text](image-50.png)

![alt text](image-51.png)

This is very simple!!

---

## Authorization

![alt text](image-52.png)


![alt text](image-53.png)

Api will be accessed by Specific role only!!

![alt text](image-54.png)


![alt text](image-55.png)


![alt text](image-56.png)

---


Below is a complete setup built on your existing code. Your `UserRegisterEntity`, `UserRegisterEntityRepository`, `UserRegisterEntityService`, and register API stay the same.

## The design

| | Access token | Refresh token |
|---|---|---|
| Format | JWT (signed) | Random string (not a JWT) |
| Lifetime | 15 minutes | 7 days |
| Stored on server? | No (stateless) | Yes, in the DB |
| Sent with | Every API request | Only to `/api/auth/refresh` |
| Can be revoked? | No (it expires quickly) | Yes (delete it from the DB) |

The refresh token is stored in the DB so you can **revoke** it on logout. A JWT can't be cancelled before it expires, but a DB row can be deleted. Making the refresh token a random string instead of a JWT keeps it simple: the server just looks it up.

**Files you'll add:**
```
config/   SecurityConfig.java (updated)
jwt/      JwtUtil.java, JwtAuthFilter.java
entity/   RefreshToken.java
repo/     RefreshTokenRepository.java
service/  RefreshTokenService.java
dto/      LoginRequest.java, RefreshRequest.java, AuthResponse.java
exception/InvalidRefreshTokenException.java
controller/AuthController.java
```

---

## 1. pom.xml

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

`jjwt-api` is what your code calls. `impl` and `jackson` do the actual signing and JSON work at runtime.

## 2. application.properties

```properties
# Generate with: openssl rand -base64 32   (never commit a real secret)
jwt.secret=PASTE_YOUR_BASE64_SECRET_HERE
jwt.access-expiration-ms=900000        # 15 minutes
jwt.refresh-expiration-ms=604800000    # 7 days
```

---

## 3. JwtUtil: access token generation and validation

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.List;

@Component
public class JwtUtil {

    private final SecretKey key;
    private final long accessExpirationMs;

    public JwtUtil(@Value("${jwt.secret}") String secret,
                   @Value("${jwt.access-expiration-ms}") long accessExpirationMs) {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
        this.accessExpirationMs = accessExpirationMs;
    }

    // ---------- GENERATION ----------
    public String generateAccessToken(UserDetails user) {
        List<String> roles = user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList();

        Date now = new Date();
        return Jwts.builder()
                .subject(user.getUsername())                       // "sub"
                .claim("roles", roles)                             // custom claim
                .issuedAt(now)                                     // "iat"
                .expiration(new Date(now.getTime() + accessExpirationMs)) // "exp"
                .signWith(key)                                     // HS256 signature
                .compact();                                        // header.payload.signature
    }

    // ---------- VALIDATION ----------
    // Throws JwtException if the signature is invalid, the token is expired, or it is malformed
    public Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    public String extractUsername(String token) {
        return parseClaims(token).getSubject();
    }

    public boolean isTokenValid(String token, UserDetails user) {
        return extractUsername(token).equals(user.getUsername());
    }
}
```

**How it works**

- **Constructor:** decodes your Base64 secret into a `SecretKey`. `hmacShaKeyFor` throws an error at startup if the key is shorter than 256 bits, so a weak key can't slip through.
- **`generateAccessToken`:** builds the three parts.
  - Header: `{"alg":"HS256"}`, added automatically because the key is an HMAC key.
  - Payload: `sub`, `roles`, `iat`, `exp`.
  - Signature: `signWith(key)` computes HMAC-SHA256 over the header and payload.
  - `compact()` joins them into `xxxxx.yyyyy.zzzzz`.
- **`parseClaims`:** this one call does all the validation:
  1. splits the token and decodes it,
  2. recomputes the signature with your key and compares it (a tampered payload throws `SignatureException`),
  3. checks `exp` (an expired token throws `ExpiredJwtException`),
  4. rejects unsigned `alg: none` tokens.

  If it returns without throwing, the token is genuine and still valid.
- **`isTokenValid`:** since signature and expiry are already checked during parsing, all that's left is confirming the token belongs to this user.

---

## 4. JwtAuthFilter: validates the token on every request

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserRegisterEntityService userService;

    public JwtAuthFilter(JwtUtil jwtUtil, UserRegisterEntityService userService) {
        this.jwtUtil = jwtUtil;
        this.userService = userService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        // 1. No Bearer token? Let the request continue unauthenticated.
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = header.substring(7); // remove "Bearer "

        try {
            // 2. Validate signature + expiry and read the username
            String username = jwtUtil.extractUsername(token);

            if (username != null
                    && SecurityContextHolder.getContext().getAuthentication() == null) {

                // 3. Load the user from the DB
                UserDetails user = userService.loadUserByUsername(username);

                if (jwtUtil.isTokenValid(token, user)) {
                    // 4. Mark this request as authenticated
                    UsernamePasswordAuthenticationToken auth =
                            new UsernamePasswordAuthenticationToken(
                                    user, null, user.getAuthorities());
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (JwtException | UsernameNotFoundException e) {
            // Invalid/expired token or deleted user: do nothing.
            // The context stays empty, so Spring returns 401 for protected URLs.
        }

        chain.doFilter(request, response);
    }
}
```

**How it works**

- `OncePerRequestFilter` guarantees the filter runs exactly once per request.
- `SecurityContextHolder` is how Spring knows who is calling. Putting an `Authentication` into it is the same as saying "this request is logged in as this user, with these roles". Your `/api/users` endpoint reads from this same place.
- The filter never rejects a request itself. It either sets the authentication or leaves it empty, and the rules in `SecurityConfig` decide what happens next. That's why the public endpoints (register, login, refresh) still work without a token.
- The authentication is built with `null` credentials because the password isn't needed after login.

---

## 5. RefreshToken entity

```java
@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 200)
    private String token;

    @ManyToOne(optional = false)
    @JoinColumn(name = "user_id")
    private UserRegisterEntity user;

    @Column(nullable = false)
    private Instant expiryDate;

    public Long getId() { return id; }
    public String getToken() { return token; }
    public void setToken(String token) { this.token = token; }
    public UserRegisterEntity getUser() { return user; }
    public void setUser(UserRegisterEntity user) { this.user = user; }
    public Instant getExpiryDate() { return expiryDate; }
    public void setExpiryDate(Instant expiryDate) { this.expiryDate = expiryDate; }
}
```

`@ManyToOne` means one user can have several refresh tokens, for example one per device (phone, laptop). Each login creates a new row.

## 6. RefreshTokenRepository

```java
@Repository
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    Optional<RefreshToken> findByToken(String token);

    void deleteByUser(UserRegisterEntity user);
}
```

`findByToken` looks up a token when the client sends it. `deleteByUser` removes all of a user's tokens, which gives you "log out from all devices".

## 7. InvalidRefreshTokenException

```java
@ResponseStatus(HttpStatus.UNAUTHORIZED)
public class InvalidRefreshTokenException extends RuntimeException {
    public InvalidRefreshTokenException(String message) {
        super(message);
    }
}
```

`@ResponseStatus` makes Spring automatically return **401** whenever this exception is thrown from a controller.

## 8. RefreshTokenService: generation, rotation, revocation

```java
@Service
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final long refreshExpirationMs;
    private final SecureRandom secureRandom = new SecureRandom();

    public RefreshTokenService(RefreshTokenRepository refreshTokenRepository,
                               @Value("${jwt.refresh-expiration-ms}") long refreshExpirationMs) {
        this.refreshTokenRepository = refreshTokenRepository;
        this.refreshExpirationMs = refreshExpirationMs;
    }

    // ---------- GENERATION ----------
    @Transactional
    public RefreshToken createRefreshToken(UserRegisterEntity user) {
        RefreshToken refreshToken = new RefreshToken();
        refreshToken.setUser(user);
        refreshToken.setToken(generateRandomToken());
        refreshToken.setExpiryDate(Instant.now().plusMillis(refreshExpirationMs));
        return refreshTokenRepository.save(refreshToken);
    }

    // ---------- VALIDATION + ROTATION ----------
    @Transactional(noRollbackFor = InvalidRefreshTokenException.class)
    public RefreshToken rotate(String oldTokenValue) {
        RefreshToken oldToken = refreshTokenRepository.findByToken(oldTokenValue)
                .orElseThrow(() -> new InvalidRefreshTokenException("Invalid refresh token"));

        // A refresh token can be used only once: delete it right away
        refreshTokenRepository.delete(oldToken);

        if (oldToken.getExpiryDate().isBefore(Instant.now())) {
            throw new InvalidRefreshTokenException("Refresh token expired, please log in again");
        }

        // Issue a brand-new refresh token for the same user
        return createRefreshToken(oldToken.getUser());
    }

    // ---------- REVOCATION ----------
    @Transactional
    public void revoke(String tokenValue) {
        refreshTokenRepository.findByToken(tokenValue)
                .ifPresent(refreshTokenRepository::delete);
    }

    @Transactional
    public void revokeAllForUser(UserRegisterEntity user) {
        refreshTokenRepository.deleteByUser(user);
    }

    private String generateRandomToken() {
        byte[] bytes = new byte[64];
        secureRandom.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

**How it works**

- **`generateRandomToken`:** 64 bytes from `SecureRandom` (cryptographically secure, unlike `Random`) give 512 bits of randomness, encoded as a URL-safe string. It can't be guessed.
- **`createRefreshToken`:** saves a row of token, user, and expiry date.
- **`rotate`** is the refresh step, and it's the most important method here:
  1. Look up the token. If it isn't found, it's fake or already used, so return 401.
  2. **Delete it immediately**, so it can never be used again.
  3. If it has expired, return 401 and the user must log in again.
  4. Create and return a **new** refresh token.

  This is called **refresh token rotation**. If an attacker steals a refresh token, it stops working as soon as either the attacker or the real user uses it once.
- `noRollbackFor`: by default, throwing a `RuntimeException` rolls back the transaction, which would undo the delete. This setting keeps the expired token deleted even though an exception is thrown.
- **`revoke`:** deletes one token (logout on this device). **`revokeAllForUser`** deletes all of them (logout everywhere, or after a password change).

---

## 9. DTOs

```java
public record LoginRequest(String username, String password) {}

public record RefreshRequest(String refreshToken) {}

public record AuthResponse(String accessToken, String refreshToken) {}
```

Records are short immutable classes. Spring converts JSON to and from them automatically.

## 10. AuthController: login, refresh, logout

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final RefreshTokenService refreshTokenService;

    public AuthController(AuthenticationManager authenticationManager,
                          JwtUtil jwtUtil,
                          RefreshTokenService refreshTokenService) {
        this.authenticationManager = authenticationManager;
        this.jwtUtil = jwtUtil;
        this.refreshTokenService = refreshTokenService;
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        try {
            // 1. Check username + password
            Authentication authentication = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(request.username(), request.password()));

            UserRegisterEntity user = (UserRegisterEntity) authentication.getPrincipal();

            // 2. Issue both tokens
            String accessToken = jwtUtil.generateAccessToken(user);
            RefreshToken refreshToken = refreshTokenService.createRefreshToken(user);

            return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken.getToken()));

        } catch (AuthenticationException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body("Invalid username or password");
        }
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
        // Validates + rotates; throws 401 if invalid/expired
        RefreshToken newRefreshToken = refreshTokenService.rotate(request.refreshToken());

        String newAccessToken = jwtUtil.generateAccessToken(newRefreshToken.getUser());

        return ResponseEntity.ok(new AuthResponse(newAccessToken, newRefreshToken.getToken()));
    }

    @PostMapping("/logout")
    public ResponseEntity<String> logout(@RequestBody RefreshRequest request) {
        refreshTokenService.revoke(request.refreshToken());
        return ResponseEntity.ok("Logged out");
    }
}
```

**How it works**

- **Login:** `authenticationManager.authenticate(...)` does the following for you:
  1. calls your `loadUserByUsername` to fetch the user,
  2. runs `passwordEncoder.matches(rawPassword, storedHash)` with BCrypt,
  3. throws `BadCredentialsException` (a subclass of `AuthenticationException`) on failure.

  On success, `getPrincipal()` returns your `UserRegisterEntity`, because that's what `loadUserByUsername` returned.
- **Refresh:** no password is needed. The refresh token proves the user logged in recently. You get back a new access token **and** a new refresh token, and the old refresh token is gone.
- **Logout:** deletes the refresh token. The current access token keeps working for up to 15 minutes, which is why access tokens are kept short. The client should also discard it.

---

## 11. SecurityConfig (updated)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/user-register", "/api/auth/**").permitAll()
                    .anyRequest().authenticated())
            .exceptionHandling(e -> e
                    .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)))
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

**What changed from your version**

- `"/api/auth/**"` is public, because login, refresh, and logout must work without an access token.
- The `AuthenticationManager` bean exposes Spring's manager so `AuthController` can use it. Spring Boot wires it automatically to your `UserDetailsService` (`UserRegisterEntityService`) and your `PasswordEncoder` bean.
- `HttpStatusEntryPoint(UNAUTHORIZED)`: without this, Spring returns **403** for missing or expired tokens. A **401** is the correct status, and the frontend uses it as the signal "call `/refresh` now".
- `addFilterBefore(...)` makes your JWT filter run before Spring's built-in login filter, so the request is already authenticated when it reaches authorization.

---

## The complete flow

**1. Register**
```
POST /api/user-register
{"username":"mohit","password":"1234","role":"ROLE_USER"}
```

**2. Log in**
```
POST /api/auth/login
{"username":"mohit","password":"1234"}

→ 200 {"accessToken":"eyJhbGciOiJIUzI1NiJ9...","refreshToken":"q8Zk3...xY"}
```

**3. Call a protected API**
```
GET /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
→ 200
```
`JwtAuthFilter` validates the token, sets the `SecurityContext`, and the controller runs.

**4. After 15 minutes the access token expires**
```
GET /api/users  →  401
```

**5. The client refreshes**
```
POST /api/auth/refresh
{"refreshToken":"q8Zk3...xY"}

→ 200 {"accessToken":"<new JWT>","refreshToken":"<new random token>"}
```
The old refresh token is now deleted. The client retries step 3 with the new access token. The user never notices any of this.

**6. Log out**
```
POST /api/auth/logout
{"refreshToken":"<current refresh token>"}
```

**7. After 7 days without use**, the refresh token has expired, `/refresh` returns 401, and the user has to log in again.

---

## Improvements for a real project

- **Hash refresh tokens in the DB.** Store a SHA-256 hash of the token instead of the raw value, so a database leak doesn't hand out working tokens.
- **Reuse detection.** If an already-used refresh token shows up again, it was probably stolen. Call `revokeAllForUser` to log that user out everywhere.
- **Clean up expired rows** with a `@Scheduled` job that deletes tokens past their `expiryDate`.
- **Refresh token storage on the frontend.** Putting it in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie protects it from XSS. If you do that, remember from your first question that CSRF protection matters again for the `/refresh` endpoint.
- **Lock down roles at registration.** Your register API still lets anyone send `"role":"ROLE_ADMIN"`. Set `ROLE_USER` on the server instead of trusting the request.

---

**A JWT stays valid until its `exp` time, even after logout.** The server doesn't store it anywhere, so there's nothing to delete. Anyone holding the token can keep using it until it expires.

That's the main trade-off of stateless JWT. To invalidate a token before it expires, the server has to keep *some* state again. Here are the options, from simplest to strongest.

---

## Option 1: Short access token + revocable refresh token (what you already have)

This is the setup from the last answer:
- The access token lives only **15 minutes**.
- Logout **deletes the refresh token** from the DB.
- The client deletes both tokens from its storage.

After logout, the old access token works for at most 15 more minutes, and then it can never be renewed. **For most apps this is good enough**, and it's what most companies do. Lower the expiry (say, 5 minutes) to shrink the window further.

Use options 2 and 3 when you need the token to stop working **immediately**.

---

## Option 2: Denylist (blacklist) of logged-out tokens

Give every JWT a unique ID (`jti`). On logout, save that ID. The filter rejects any token whose ID is on the list.

You only need to keep each entry **until the token would have expired anyway**, so the list stays small.

### Step 1: Add a unique ID to every token (JwtUtil)

```java
public String generateAccessToken(UserDetails user) {
    List<String> roles = user.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();

    Date now = new Date();
    return Jwts.builder()
            .id(UUID.randomUUID().toString())   // "jti": unique ID for this token
            .subject(user.getUsername())
            .claim("roles", roles)
            .issuedAt(now)
            .expiration(new Date(now.getTime() + accessExpirationMs))
            .signWith(key)
            .compact();
}
```

### Step 2: Table for revoked tokens

```java
@Entity
@Table(name = "revoked_tokens")
public class RevokedToken {

    @Id
    private String jti;            // the token's unique ID

    @Column(nullable = false)
    private Instant expiresAt;     // when the JWT would expire anyway

    protected RevokedToken() {}    // required by JPA

    public RevokedToken(String jti, Instant expiresAt) {
        this.jti = jti;
        this.expiresAt = expiresAt;
    }
}
```

```java
@Repository
public interface RevokedTokenRepository extends JpaRepository<RevokedToken, String> {

    void deleteByExpiresAtBefore(Instant time);
}
```

### Step 3: TokenBlacklistService

```java
@Service
public class TokenBlacklistService {

    private final RevokedTokenRepository repository;
    private final JwtUtil jwtUtil;

    public TokenBlacklistService(RevokedTokenRepository repository, JwtUtil jwtUtil) {
        this.repository = repository;
        this.jwtUtil = jwtUtil;
    }

    public void blacklist(String token) {
        try {
            Claims claims = jwtUtil.parseClaims(token);
            repository.save(new RevokedToken(
                    claims.getId(),
                    claims.getExpiration().toInstant()));
        } catch (JwtException e) {
            // Already expired or invalid: nothing to blacklist
        }
    }

    public boolean isRevoked(String jti) {
        return repository.existsById(jti);
    }

    // Runs every hour: removes entries for tokens that have expired anyway
    @Scheduled(fixedRate = 3600000)
    @Transactional
    public void cleanup() {
        repository.deleteByExpiresAtBefore(Instant.now());
    }
}
```

Add `@EnableScheduling` to your main application class so `cleanup()` runs.

### Step 4: Check the denylist in JwtAuthFilter

```java
try {
    Claims claims = jwtUtil.parseClaims(token);   // signature + expiry check

    // NEW: reject tokens that were logged out
    if (tokenBlacklistService.isRevoked(claims.getId())) {
        chain.doFilter(request, response);         // stays unauthenticated → 401
        return;
    }

    String username = claims.getSubject();
    if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
        UserDetails user = userService.loadUserByUsername(username);
        UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
        auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        SecurityContextHolder.getContext().setAuthentication(auth);
    }
} catch (JwtException | UsernameNotFoundException e) {
    // invalid token: leave unauthenticated
}
```

(Inject `TokenBlacklistService` into the filter's constructor.)

### Step 5: Logout revokes both tokens (AuthController)

```java
@PostMapping("/logout")
public ResponseEntity<String> logout(
        @RequestHeader(value = "Authorization", required = false) String authHeader,
        @RequestBody RefreshRequest request) {

    // 1. Kill the refresh token (no more new access tokens)
    refreshTokenService.revoke(request.refreshToken());

    // 2. Kill the current access token immediately
    if (authHeader != null && authHeader.startsWith("Bearer ")) {
        tokenBlacklistService.blacklist(authHeader.substring(7));
    }

    return ResponseEntity.ok("Logged out");
}
```

Now the logout request sends `Authorization: Bearer <accessToken>` plus `{"refreshToken": "..."}` in the body, and **both tokens stop working instantly**.

**Redis is a better fit than a DB table** for the denylist in production. You store `jti` with a TTL equal to the token's remaining lifetime, Redis deletes it automatically when it expires (no cleanup job), and lookups are much faster than a DB query.

---

## Option 3: Token version per user (logout from all devices)

The denylist handles one token at a time. To kill **every** token for a user at once (after "log out everywhere", a password change, or a ban), store a version number on the user.

### Add a field to UserRegisterEntity

```java
@Column(nullable = false)
private int tokenVersion = 0;

public int getTokenVersion() { return tokenVersion; }
public void incrementTokenVersion() { this.tokenVersion++; }
```

### Put the version into the JWT (JwtUtil)

```java
public String generateAccessToken(UserRegisterEntity user) {
    // ... same as before, plus:
    // .claim("ver", user.getTokenVersion())
}
```

### Compare it in JwtAuthFilter

```java
UserRegisterEntity user = (UserRegisterEntity) userService.loadUserByUsername(username);

Integer tokenVer = claims.get("ver", Integer.class);
if (tokenVer == null || tokenVer != user.getTokenVersion()) {
    chain.doFilter(request, response);   // old token → reject
    return;
}
```

### Logout from all devices

```java
@Transactional
public void logoutAllDevices(UserRegisterEntity user) {
    user.incrementTokenVersion();              // every existing JWT now has an old "ver"
    userRepository.save(user);
    refreshTokenService.revokeAllForUser(user); // and no refresh token can renew them
}
```

**How it works:** a token issued when the version was `3` carries `"ver": 3`. After the increment, the DB says `4`, so the filter rejects every old token at once. New logins get `"ver": 4`.

This costs nothing extra in your app, because your filter **already loads the user from the DB** on every request.

---

## Option 4: Change the secret key (emergency only)

If your secret key leaks, replace it. Every token ever issued becomes invalid instantly, and **every user is logged out**. Use this only for emergencies.

---

## Which one to use

| Need | Solution |
|---|---|
| Normal app, a few minutes of leftover validity is acceptable | Option 1: short expiry + delete refresh token |
| Logout must take effect instantly | Option 1 + Option 2 (denylist, ideally Redis) |
| "Log out all devices", password change, banning a user | Option 3 (token version) |
| Secret key leaked | Option 4 (rotate the key) |

**A good combination for your project:** 15-minute access tokens, refresh token rotation (already done), and token version (Option 3). It needs no extra table or Redis, and it covers password changes and logout-all.

One thing to keep in mind: options 2 and 3 make your API **partly stateful** again, since each request does a lookup. That's the price of instant revocation, and it's an accepted trade-off. Most real systems do some version of it.

---
## If a person do login ,logout 15 to 20 times then 15 tp 20 jwt token be genarated

Yes, every login creates a **new** JWT, so 20 logins produce 20 tokens. That's normal and not a problem. Here's why, and what does need handling.

## The JWTs themselves cost nothing

- **The server doesn't store JWTs.** A token exists only on the client that received it, so 20 tokens use zero database space.
- **Generating one is very cheap.** It's a single HMAC computation, a few microseconds.
- **Old tokens die on their own.** Each has a 15-minute `exp`, after which it's useless forever. You never have to delete anything.

So at any moment there are at most a handful of "still valid" tokens for that user, and only within the last 15 minutes.

## Are the old ones a security risk?

Not really. A logged-out token is still technically valid for up to 15 minutes, but only the person who received it has it. Someone would have to **steal** it in that window to misuse it.

How each option from the last answer handles 20 logins:

| Setup | What happens with 20 login/logout cycles |
|---|---|
| Option 1 (short expiry) | Old tokens expire within 15 minutes. Nothing is stored. |
| Option 2 (denylist) | Each logout adds 1 row, so at most ~20 rows, and the cleanup job removes each one once its token has expired. |
| Option 3 (token version) | Nothing is stored. If you increment the version on logout, every older token dies instantly. |

## What does pile up: refresh tokens

Refresh tokens **are** stored in the DB. Logout deletes the row, so login → logout → login is fine.

The problem is a user who logs in **without** logging out (closes the tab, clears the browser, logs in on many devices). Each login adds a row that lives for 7 days. Two fixes:

### Fix 1: Delete expired refresh tokens regularly

```java
// RefreshTokenRepository
void deleteByExpiryDateBefore(Instant time);
```

```java
// RefreshTokenService
@Scheduled(cron = "0 0 3 * * *")   // every day at 3 AM
@Transactional
public void deleteExpiredTokens() {
    refreshTokenRepository.deleteByExpiryDateBefore(Instant.now());
}
```

(Needs `@EnableScheduling` on your main class.)

### Fix 2: Limit active sessions per user

For example, allow at most 5 devices. On a new login, the oldest session is removed.

```java
// RefreshTokenRepository
List<RefreshToken> findByUserOrderByExpiryDateAsc(UserRegisterEntity user);
```

```java
// RefreshTokenService
private static final int MAX_SESSIONS = 5;

@Transactional
public RefreshToken createRefreshToken(UserRegisterEntity user) {
    List<RefreshToken> existing = refreshTokenRepository.findByUserOrderByExpiryDateAsc(user);

    // Remove the oldest ones so that, after adding the new one, there are at most MAX_SESSIONS
    int toRemove = existing.size() - (MAX_SESSIONS - 1);
    for (int i = 0; i < toRemove; i++) {
        refreshTokenRepository.delete(existing.get(i));
    }

    RefreshToken refreshToken = new RefreshToken();
    refreshToken.setUser(user);
    refreshToken.setToken(generateRandomToken());
    refreshToken.setExpiryDate(Instant.now().plusMillis(refreshExpirationMs));
    return refreshTokenRepository.save(refreshToken);
}
```

With `MAX_SESSIONS = 1`, you get "only one active login at a time", like some banking apps: logging in on a new device logs out the old one.

## One more thing: rate-limit the login endpoint

Logging in 20 times in a row is harmless for one honest user. But someone hammering `/api/auth/login` thousands of times is usually **password guessing**, and BCrypt checks are deliberately slow, so it also loads your server. Limit login attempts per IP or username (for example, 5 per minute) with a library like **Bucket4j**, or at your API gateway or Nginx.

## Summary

- Many JWTs being created is fine: they aren't stored and they expire by themselves.
- Refresh tokens are what accumulate, so clean up expired ones and optionally cap sessions per user.
- If you want logout to kill every old token instantly, use the token version from Option 3.