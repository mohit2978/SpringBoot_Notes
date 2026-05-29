# Basic JWT structure

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

## So different filter for auth ,validation and refresh token ??

Yes! Exactly right. For JWT you need 3 different filters. Let me explain all 3 in complete depth.

---

## The 3 JWT Filters Overview

```
Request comes in
      ↓
Filter 1: JwtAuthenticationFilter    ← handles POST /auth/login
      ↓                                 validates credentials
Filter 2: JwtValidationFilter        ← handles ALL other requests
      ↓                                 validates JWT token
Filter 3: JwtRefreshTokenFilter      ← handles POST /auth/refresh
      ↓                                 validates refresh token
                                        issues new access token
Controller
```

---

## Complete Project Structure

```
src/main/java/com/example/
├── config/
│   └── SecurityConfig.java
├── controller/
│   └── AuthController.java
├── entity/
│   └── UserAuthEntity.java
├── filter/
│   ├── JwtAuthenticationFilter.java   ← Filter 1
│   ├── JwtValidationFilter.java       ← Filter 2
│   └── JwtRefreshTokenFilter.java     ← Filter 3
├── repository/
│   └── UserAuthEntityRepository.java
├── service/
│   ├── JwtService.java
│   └── UserAuthEntityService.java
└── model/
    └── RefreshToken.java
```

---

## JwtService — the utility class all 3 filters use

```java
// YOU write this
// Handles all JWT operations — creating, validating, extracting

@Service
public class JwtService {

    // Store these in application.properties — never hardcode
    @Value("${jwt.access.secret}")
    private String ACCESS_SECRET_KEY;

    @Value("${jwt.refresh.secret}")
    private String REFRESH_SECRET_KEY;

    @Value("${jwt.access.expiration}")
    private long ACCESS_EXPIRATION; // e.g. 900000 = 15 minutes

    @Value("${jwt.refresh.expiration}")
    private long REFRESH_EXPIRATION; // e.g. 604800000 = 7 days

    // ─── ACCESS TOKEN ───────────────────────────────────────

    public String generateAccessToken(UserDetails userDetails) {
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(
                System.currentTimeMillis() + ACCESS_EXPIRATION))
            .claim("authorities",
                userDetails.getAuthorities().stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.toList()))
            .signWith(getAccessSignKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public String extractUsernameFromAccessToken(String token) {
        return extractClaim(token, Claims::getSubject, getAccessSignKey());
    }

    public boolean isAccessTokenValid(String token, UserDetails userDetails) {
        String username = extractUsernameFromAccessToken(token);
        return username.equals(userDetails.getUsername())
            && !isTokenExpired(token, getAccessSignKey());
    }

    // ─── REFRESH TOKEN ──────────────────────────────────────

    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(
                System.currentTimeMillis() + REFRESH_EXPIRATION))
            .signWith(getRefreshSignKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public String extractUsernameFromRefreshToken(String token) {
        return extractClaim(token, Claims::getSubject, getRefreshSignKey());
    }

    public boolean isRefreshTokenValid(String token, UserDetails userDetails) {
        String username = extractUsernameFromRefreshToken(token);
        return username.equals(userDetails.getUsername())
            && !isTokenExpired(token, getRefreshSignKey());
    }

    // ─── HELPERS ────────────────────────────────────────────

    private <T> T extractClaim(String token,
            Function<Claims, T> claimsResolver, Key key) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)
            .getBody();
        return claimsResolver.apply(claims);
    }

    private boolean isTokenExpired(String token, Key key) {
        Date expiration = extractClaim(token, Claims::getExpiration, key);
        return expiration.before(new Date());
    }

    private Key getAccessSignKey() {
        byte[] keyBytes = Decoders.BASE64.decode(ACCESS_SECRET_KEY);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    private Key getRefreshSignKey() {
        byte[] keyBytes = Decoders.BASE64.decode(REFRESH_SECRET_KEY);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

---

## Filter 1: JwtAuthenticationFilter

**Purpose:** Handle `POST /auth/login` — validate credentials, issue access token + refresh token.

**Triggers on:** Only `POST /auth/login`.

**Does NOT call chain.doFilter()** after success — it handles the response itself.

```java
// YOU write this
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired private AuthenticationManager authenticationManager;
    @Autowired private JwtService jwtService;
    @Autowired private UserAuthEntityService userDetailsService;

    // This filter ONLY triggers on POST /auth/login
    private static final AntPathRequestMatcher LOGIN_MATCHER =
        new AntPathRequestMatcher("/auth/login", "POST");

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // Skip this filter for all routes EXCEPT POST /auth/login
        return !LOGIN_MATCHER.matches(request);
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        // Step 1: Read credentials from request body
        // { "username": "alice", "password": "abc123" }
        ObjectMapper mapper = new ObjectMapper();
        LoginRequest loginRequest =
            mapper.readValue(request.getInputStream(), LoginRequest.class);

        try {
            // Step 2: Create unauthenticated Authentication object
            // (2-arg constructor — authenticated=false)
            UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(
                    loginRequest.getUsername(),
                    loginRequest.getPassword()
                );

            // Step 3: Pass to AuthenticationManager
            // → calls DaoAuthenticationProvider
            // → calls UserDetailsService.loadUserByUsername()
            // → verifies password with PasswordEncoder
            // → returns fully authenticated Authentication object
            Authentication authenticated =
                authenticationManager.authenticate(authToken);

            // Step 4: Get the UserDetails from authenticated object
            UserDetails userDetails =
                (UserDetails) authenticated.getPrincipal();

            // Step 5: Generate both tokens
            String accessToken =
                jwtService.generateAccessToken(userDetails);
            String refreshToken =
                jwtService.generateRefreshToken(userDetails);

            // Step 6: Store refresh token in DB
            // (so we can validate/revoke it later)
            userDetailsService.saveRefreshToken(
                userDetails.getUsername(), refreshToken);

            // Step 7: Send both tokens in response
            response.setContentType("application/json");
            response.setStatus(HttpServletResponse.SC_OK);

            Map<String, String> tokens = new HashMap<>();
            tokens.put("accessToken", accessToken);
            tokens.put("refreshToken", refreshToken);
            tokens.put("tokenType", "Bearer");
            tokens.put("expiresIn", "900"); // 15 minutes in seconds

            mapper.writeValue(response.getOutputStream(), tokens);

            // NOTE: we do NOT call chain.doFilter() here
            // The request is fully handled — response sent directly

        } catch (AuthenticationException e) {
            // Wrong username or password
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json");

            Map<String, String> error = new HashMap<>();
            error.put("error", "Invalid credentials");
            new ObjectMapper().writeValue(response.getOutputStream(), error);
        }
    }

    // Simple DTO for reading login request body
    public static class LoginRequest {
        private String username;
        private String password;
        // getters and setters
        public String getUsername() { return username; }
        public String getPassword() { return password; }
    }
}
```

---

## Filter 2: JwtValidationFilter

**Purpose:** Validate JWT access token on every protected request.

**Triggers on:** Every request EXCEPT `/auth/login` and `/auth/refresh` and `/auth/register`.

**Calls chain.doFilter()** always — it just sets Authentication and passes on.

```java
// YOU write this
@Component
public class JwtValidationFilter extends OncePerRequestFilter {

    @Autowired private JwtService jwtService;
    @Autowired private UserDetailsService userDetailsService;

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getServletPath();
        // Skip this filter for auth endpoints
        // They don't need a valid token to access
        return path.equals("/auth/login")
            || path.equals("/auth/register")
            || path.equals("/auth/refresh");
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        // Step 1: Extract Authorization header
        String authHeader = request.getHeader("Authorization");

        // Step 2: Check if Bearer token is present
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            // No token — pass to next filter
            // ExceptionTranslationFilter will handle 401 later
            chain.doFilter(request, response);
            return;
        }

        // Step 3: Extract the token
        String accessToken = authHeader.substring(7);
        // "Bearer eyJhbGciOiJIUzI1NiJ9..." → "eyJhbGciOiJIUzI1NiJ9..."

        try {
            // Step 4: Extract username from token
            String username =
                jwtService.extractUsernameFromAccessToken(accessToken);

            // Step 5: Only validate if username found
            //         AND no authentication already set in context
            if (username != null &&
                    SecurityContextHolder.getContext()
                        .getAuthentication() == null) {

                // Step 6: Load user from DB
                // (needed to verify token + get authorities)
                UserDetails userDetails =
                    userDetailsService.loadUserByUsername(username);

                // Step 7: Validate token
                // checks: signature, expiry, username match
                if (jwtService.isAccessTokenValid(accessToken, userDetails)) {

                    // Step 8: Build authenticated Authentication object
                    // (3-arg constructor — authenticated=true)
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,                    // principal
                            null,                           // credentials = null
                            userDetails.getAuthorities()    // [ROLE_USER]
                        );

                    // Step 9: Add request details (IP, session ID)
                    authToken.setDetails(
                        new WebAuthenticationDetailsSource()
                            .buildDetails(request)
                    );

                    // Step 10: Set in SecurityContext
                    // Now Spring Security knows this user is authenticated
                    SecurityContextHolder.getContext()
                        .setAuthentication(authToken);
                }
            }

        } catch (ExpiredJwtException e) {
            // Token expired — send 401 with specific message
            sendError(response,
                HttpServletResponse.SC_UNAUTHORIZED,
                "Access token expired");
            return;

        } catch (JwtException e) {
            // Token invalid (tampered, wrong signature etc)
            sendError(response,
                HttpServletResponse.SC_UNAUTHORIZED,
                "Invalid access token");
            return;
        }

        // Step 11: Always pass to next filter
        // AuthorizationFilter will check if user has permission
        chain.doFilter(request, response);
    }

    private void sendError(HttpServletResponse response,
            int status, String message) throws IOException {
        response.setStatus(status);
        response.setContentType("application/json");
        Map<String, String> error = new HashMap<>();
        error.put("error", message);
        new ObjectMapper().writeValue(response.getOutputStream(), error);
    }
}
```

---

## Filter 3: JwtRefreshTokenFilter

**Purpose:** Handle `POST /auth/refresh` — validate refresh token, issue new access token.

**Triggers on:** Only `POST /auth/refresh`.

**Does NOT call chain.doFilter()** — handles response itself.

```java
// YOU write this
@Component
public class JwtRefreshTokenFilter extends OncePerRequestFilter {

    @Autowired private JwtService jwtService;
    @Autowired private UserAuthEntityService userDetailsService;

    // Only triggers on POST /auth/refresh
    private static final AntPathRequestMatcher REFRESH_MATCHER =
        new AntPathRequestMatcher("/auth/refresh", "POST");

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return !REFRESH_MATCHER.matches(request);
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        // Step 1: Extract Authorization header
        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            sendError(response,
                HttpServletResponse.SC_BAD_REQUEST,
                "Refresh token missing");
            return;
        }

        // Step 2: Extract refresh token
        String refreshToken = authHeader.substring(7);

        try {
            // Step 3: Extract username from REFRESH token
            // Uses DIFFERENT secret key than access token
            String username =
                jwtService.extractUsernameFromRefreshToken(refreshToken);

            // Step 4: Load user from DB
            UserDetails userDetails =
                userDetailsService.loadUserByUsername(username);

            // Step 5: Check if refresh token exists in DB
            // and belongs to this user
            // (lets us revoke tokens on logout)
            boolean tokenExistsInDb =
                userDetailsService.isRefreshTokenValid(
                    username, refreshToken);

            if (!tokenExistsInDb) {
                sendError(response,
                    HttpServletResponse.SC_UNAUTHORIZED,
                    "Refresh token revoked or not found");
                return;
            }

            // Step 6: Validate refresh token
            if (!jwtService.isRefreshTokenValid(refreshToken, userDetails)) {
                // Token expired or invalid
                // Delete from DB since it's no longer usable
                userDetailsService.deleteRefreshToken(username);

                sendError(response,
                    HttpServletResponse.SC_UNAUTHORIZED,
                    "Refresh token expired — please login again");
                return;
            }

            // Step 7: Generate new access token
            String newAccessToken =
                jwtService.generateAccessToken(userDetails);

            // Step 8: Optionally rotate refresh token
            // (recommended — each refresh gives a new refresh token)
            String newRefreshToken =
                jwtService.generateRefreshToken(userDetails);

            // Delete old refresh token, save new one
            userDetailsService.deleteRefreshToken(username);
            userDetailsService.saveRefreshToken(username, newRefreshToken);

            // Step 9: Send new tokens in response
            response.setContentType("application/json");
            response.setStatus(HttpServletResponse.SC_OK);

            Map<String, String> tokens = new HashMap<>();
            tokens.put("accessToken", newAccessToken);
            tokens.put("refreshToken", newRefreshToken);
            tokens.put("tokenType", "Bearer");
            tokens.put("expiresIn", "900");

            new ObjectMapper().writeValue(response.getOutputStream(), tokens);

        } catch (ExpiredJwtException e) {
            userDetailsService.deleteRefreshToken(
                e.getClaims().getSubject());
            sendError(response,
                HttpServletResponse.SC_UNAUTHORIZED,
                "Refresh token expired — please login again");

        } catch (JwtException e) {
            sendError(response,
                HttpServletResponse.SC_UNAUTHORIZED,
                "Invalid refresh token");
        }
    }

    private void sendError(HttpServletResponse response,
            int status, String message) throws IOException {
        response.setStatus(status);
        response.setContentType("application/json");
        Map<String, String> error = new HashMap<>();
        error.put("error", message);
        new ObjectMapper().writeValue(response.getOutputStream(), error);
    }
}
```

---

## SecurityConfig — registering all 3 filters

```java
// YOU write this
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired private JwtAuthenticationFilter jwtAuthenticationFilter;
    @Autowired private JwtValidationFilter jwtValidationFilter;
    @Autowired private JwtRefreshTokenFilter jwtRefreshTokenFilter;
    @Autowired private UserAuthEntityService userAuthEntityService;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http)
            throws Exception {
        http
            .csrf(csrf -> csrf.disable())

            // JWT is stateless — no HTTP session needed
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            .authorizeHttpRequests(auth -> auth
                // Public endpoints — no token needed
                .requestMatchers("/auth/register").permitAll()
                .requestMatchers("/auth/login").permitAll()
                .requestMatchers("/auth/refresh").permitAll()

                // Role-based access
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasRole("USER")

                // Everything else needs authentication
                .anyRequest().authenticated()
            )

            // Register filters in correct order
            // Filter 1: runs before UsernamePasswordAuthenticationFilter
            .addFilterBefore(
                jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class)

            // Filter 2: runs before UsernamePasswordAuthenticationFilter
            .addFilterBefore(
                jwtValidationFilter,
                UsernamePasswordAuthenticationFilter.class)

            // Filter 3: runs before UsernamePasswordAuthenticationFilter
            .addFilterBefore(
                jwtRefreshTokenFilter,
                UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## Complete flow summary

```
POST /auth/login
{ username: "alice", password: "abc123" }
        │
        ↓
JwtAuthenticationFilter (Filter 1)
        │ creates unauthenticated Authentication(username, password)
        │ passes to AuthenticationManager
        │ → DaoAuthenticationProvider
        │   → UserDetailsService.loadUserByUsername("alice")
        │   → passwordEncoder.matches("abc123", "$2a$10$hash")
        │   → returns authenticated Authentication
        │ generates accessToken (15 min)
        │ generates refreshToken (7 days)
        │ saves refreshToken to DB
        │ returns both tokens in response
        ↓
Response:
{
  "accessToken":  "eyJhbGci... (expires in 15 min)",
  "refreshToken": "eyJhbGci... (expires in 7 days)"
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

GET /api/data
Authorization: Bearer eyJhbGci... (accessToken)
        │
        ↓
JwtValidationFilter (Filter 2)
        │ extracts token from header
        │ extracts username from token
        │ loads UserDetails from DB
        │ validates token (signature + expiry)
        │ builds authenticated Authentication object
        │ sets in SecurityContextHolder
        │ calls chain.doFilter() → passes on
        ↓
AuthorizationFilter
        │ checks user has permission for GET /api/data
        ↓
Controller executes → returns data

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

POST /auth/refresh
Authorization: Bearer eyJhbGci... (refreshToken)
        │
        ↓
JwtRefreshTokenFilter (Filter 3)
        │ extracts refresh token from header
        │ extracts username from refresh token
        │ loads UserDetails from DB
        │ checks refresh token exists in DB
        │ validates refresh token (signature + expiry)
        │ generates new accessToken
        │ generates new refreshToken (rotation)
        │ deletes old refreshToken from DB
        │ saves new refreshToken to DB
        │ returns both new tokens
        ↓
Response:
{
  "accessToken":  "eyJhbGci... (new — expires in 15 min)",
  "refreshToken": "eyJhbGci... (new — expires in 7 days)"
}
```

---

## Summary table of all 3 filters

| | Filter 1 | Filter 2 | Filter 3 |
|---|---|---|---|
| **Name** | JwtAuthenticationFilter | JwtValidationFilter | JwtRefreshTokenFilter |
| **Triggers on** | POST /auth/login | Every other request | POST /auth/refresh |
| **Input** | username + password | Access token | Refresh token |
| **Calls UserDetailsService?** | Yes — via AuthenticationManager | Yes — to rebuild auth | Yes — to get UserDetails |
| **Calls chain.doFilter()?** | No — sends response directly | Yes — always | No — sends response directly |
| **Output** | Access token + Refresh token | Sets SecurityContext | New access token + new refresh token |
| **Token used** | None (credentials used) | Access token | Refresh token |
| **Secret key** | None | Access secret | Refresh secret |
| **Hits DB?** | Yes — via UserDetailsService | Yes — via UserDetailsService | Yes — checks refresh token in DB |





