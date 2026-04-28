# Notes 

## Basic JWT structure

![alt text](image.png)

![alt text](image-57.png)

Any chnage in payload will make the Signature value chnaged ,but we have already avlues of signature ,so both will not match and then if it does not match it show token has been changed by   someone!! 

![alt text](image-1.png)

We provide refresh token and then we validate refresh token and then after validating we generate new access token!!

## Step1:User Creation(dynamically) (Very important part)

We have already seen how User Creation is done!!

![alt text](image-2.png)

![alt text](image-3.png)

![alt text](image-4.png)

![alt text](image-5.png)

![alt text](image-6.png)


![alt text](image-7.png)

![alt text](image-8.png)

Till now we have seen how user is generated dynamically!!

## Step-2 Token Generation

For form-based and basic-auth we used to have implemenattaion in SpringBoot but for JWT no implementation, so we need to provide it and every engineer implement in his own way!!

![alt text](image-9.png)

But we will try to stick to Security Framework only to implement the JWT functionality.So we not going to create any apis to generate token or apis for refresh token in controller!!

![alt text](image-10.png)

![alt text](image-11.png)

### Very Imp

![alt text](image-12.png)

Security filter creates `Authentication` object from UserName,password or token or sessionID!!Each filter creates it own Authentication object like UsrerName Password has different , Basic auth has different and so on!!

`Authentication` is interface. Till now `Authentication` is false!!

Then securityFilter pass this `Authentication` Object to ProviderManager!!

ProviderManager gives the request which has Authentication Object to one of AuthenticationProvider like DAOAuthenticationProvider and so on by testing each Provider which one can accept the request as ProvideManager has a list of Provider!!

### ProviderManager.java (framework code)

![alt text](image-13.png)

### DaoAuthenticationProvider.java (framework code)

![alt text](image-14.png)

![alt text](image-15.png)

![alt text](image-16.png)

### Let us code 

![alt text](image-17.png)

![alt text](image-18.png)

need jwt dependency to generate token so use jjwt-api (it just provide api), jjwt-impl (implementation of jjwt to sign the key ,get the token)(can use other impl too),jjwt-jackson (payload has Key-value so for json processing using it)

---

##### Custom Filter 

`OnePerRequestFilter` make sure that in one request any filter should not run twice!!

See here how Filter get apis!! We telling this filter only works for `generate-token`!!

![alt text](image-19.png)

UserNamePasswordAuthenticationToken we are getting as it is supported by DAOAuthentricationProvider!! Then we have passed to AuthneticationManager which will deligate to 
DAOAuthentricationProvider and which checks from DB whether credentials are valid or not !!If matches then it will put true in Isauthneticated()!!

If Authenticated we generating the token!!

We no need to go to Controllers, Filters are just before Controllers we know!

![alt text](image-20.png)

Here JWT is created!!We using HMAC algo , so same key is used in encrption and decryption!!Here we have put key here only , but In prod that should not be case ,we must have key somewhere else also should use different keys!!

![alt text](image-21.png)

##### Config

DAOAuthenticationProvider also we need to provide!!So see below for that!

![alt text](image-22.png)

Now see below we creating object of filter as Filter was not annotated with @Bean so we  need to create its object ourself!!

Also we need to add AuthenticationManager's List of Provider we provide only DAO one!!

 Here we have created a new list but we can also get the AuthenticationMananger and add out provider to list!!


![alt text](image-23.png)

addFilterBefore add our filter to filter we provided before `UserNamePasswordAuthneticationFilter`!!

![alt text](image-24.png)

![alt text](image-25.png)

In response Header we getting JWT token as we have configuered!!

So see we have not put `/generate-token` apu in controller ,It is in Filter part!!


## Step-3 Token Validation

User try to access any api ,so User first goes to SecurityFilter which validates token and then token is passed to Controller if token is valid! 

So we need to create filter again!!

![alt text](image-26.png)

![alt text](image-27.png)

After validation of JWT token ,we storing Authentication object in SecurityContext.`filterChain.doFilter()` makes sure SecurityChain goes on and not end!!

If token is not null then we ceating `JWtAuthenticationToken` which is child of `AbstractAuthenticationToken` which is child of `Authentication`!! And initially we have put `Authenticated` as false!!

![alt text](image-28.png)

Now `DAOAutheticatorProvider` do not understand JWT Authentication so we create another `AuthenticatorProvider`!!

![alt text](image-29.png)


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

After 15 min again provide username-password ,so instead increase the time for JWT like 1 day or 2 day , then their might be case where JWT is compromised ,so it is recommended JWT to be short-lived so insted we use `refresh-token`!!

Refresh token is used to get new token without putting credentails again and again so need to add filter here too!!


![alt text](image-37.png)


![alt text](image-38.png)

![alt text](image-39.png)

We putting RefreshToken in Cookie !!We put `setHttpOnly(true)` so no JS code can access it!! Also put `setSecure(true)` so only Https can access it!! We also put `setPath()` so for only this path ,Cookie is sent else browser will send for all request!!

---

![alt text](image-40.png)

![alt text](image-41.png)

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

So will not able to access!!


## JWT in Spring Security

### How JWT Works (the flow)

```
Client                          Server
  |                               |
  |-- POST /login {user, pass} -->|
  |<--------- JWT Token ----------|
  |                               |
  |-- GET /api/data               |
  |   Authorization: Bearer <JWT>-|
  |<-------- Response ------------|
```

JWT is **stateless** — the server doesn't store sessions. Instead, every request carries a self-contained token the server just verifies.

### JWT Structure

A JWT is three Base64-encoded parts separated by dots:

```
header.payload.signature
eyJhbGci...  .  eyJ1c2VyI...  .  SflKxwRJ...
```

- **Header** — algorithm used (e.g. HS256)
- **Payload** — claims: username, roles, expiry (`exp`), issued-at (`iat`)
- **Signature** — `HMAC(header + payload, secretKey)` — proves it wasn't tampered with

---

### Implementation Step-by-Step

#### 1. Add Dependencies (`pom.xml`)

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
</dependency>
```

---

#### 2. JwtService — Token Generation & Validation

```java
@Service
public class JwtService {

    private static final String SECRET_KEY = "your-256-bit-secret-here";

    // Generate token for a user
    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60)) // 1 hour
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    // Extract username from token
    public String extractUsername(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    // Validate token
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    private Date extractExpiration(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getExpiration();
    }

    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(SECRET_KEY);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

---

#### 3. JwtAuthFilter — Intercepts Every Request

This is the heart of JWT in Spring Security. It runs **before** the standard auth filter:

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired private JwtService jwtService;
    @Autowired private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // 1. Extract the Authorization header
        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response); // no token, skip
            return;
        }

        // 2. Pull the token out
        final String jwt = authHeader.substring(7);
        final String username = jwtService.extractUsername(jwt);

        // 3. If user not yet authenticated in this request
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            // 4. Validate token
            if (jwtService.isTokenValid(jwt, userDetails)) {

                // 5. Create auth token and set it in SecurityContext
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

---

#### 4. AuthController — Login Endpoint

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    @Autowired private AuthenticationManager authManager;
    @Autowired private JwtService jwtService;
    @Autowired private UserDetailsService userDetailsService;

    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request) {

        // Throws exception if credentials are wrong
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword()
            )
        );

        UserDetails user = userDetailsService.loadUserByUsername(request.getUsername());
        String token = jwtService.generateToken(user);

        return ResponseEntity.ok(token);
    }
}
```

---

#### 5. SecurityConfig — Wire Everything Together

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired private JwtAuthFilter jwtAuthFilter;
    @Autowired private UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())               // JWT is stateless, no CSRF needed
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // no sessions
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll() // login is public
                .anyRequest().authenticated()
            )
            // Add our JWT filter BEFORE the default username/password filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

### The Full Request Lifecycle

```
Request with JWT
       |
       ▼
 JwtAuthFilter
  ├── Extract token from "Authorization: Bearer <token>"
  ├── Decode & validate signature + expiry
  ├── Load UserDetails from DB
  └── Set Authentication in SecurityContextHolder
       |
       ▼
 Spring's AuthorizationFilter
  └── Checks roles/permissions against your rules
       |
       ▼
 Your Controller (if allowed)
```

---

### Key Security Considerations

| Concern | Best Practice |
|---|---|
| Secret key | Store in env variables, min 256-bit |
| Token expiry | Keep short (15–60 min), use refresh tokens |
| HTTPS | Always — JWT in plaintext is readable |
| Token storage (client) | `HttpOnly` cookie > `localStorage` (XSS safe) |
| Revocation | Maintain a token denylist in Redis/DB if needed |

---







