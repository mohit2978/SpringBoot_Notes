# Oauth-2

>Note: First Authentication happens and then authorization!!

![alt text](image.png)

![alt text](image-1.png)

Already seen in Oauth in HLD!!In 8th step we can have JWT or Opaque token or any other type of token!!

## Entities 

![alt text](image-54.png)

 Let me clear this up completely — this is where most people get confused.Now the most important question you asked — **is the Resource Server part of the Client App?** Let me show both real scenarios.
 
 ![alt text](image-55.png)
 
 Now let me answer each of your questions directly.

---

### Your question 1: "Auth Server — do we not write it?"

Correct, in 99% of projects you do NOT write the Auth Server. You use one of these:

**Use theirs (social login):** Google, GitHub, Facebook — you just register your app on their developer console, get a `client_id` and `client_secret`, and you're done. They run the entire Auth Server.

**Host your own (enterprise):** Tools like Keycloak, Auth0, or Okta — you deploy and configure them, but you do not write the code from scratch. Keycloak for example is a full Auth Server you install like a Docker container.

**Write your own (rare):** Spring has a library called Spring Authorization Server for this. Only large companies building identity platforms do this.

---

### Your question 2: "Is the Resource Server part of the Client App?"

It depends on your architecture, and both are valid:

**Same app** — your Spring Boot app handles login (Client role) AND protects `/api/*` endpoints (Resource Server role). You add both `oauth2-client` and `oauth2-resource-server` dependencies. Common in small projects.

**Separate apps** — your frontend or BFF handles login and gets the token (Client). Then it sends that token to your backend microservices. Each microservice is a standalone Resource Server — it just validates the token on every request and does not care about login at all. This is the standard in microservices architecture.

---

### The one-line rule to remember each role

| Entity | One job | You write it? |
|---|---|---|
| Resource Owner | owns the data, approves access | no — it's the user |
| Client App | gets the token | yes |
| Auth Server | issues the token | no — Google/Keycloak etc. |
| Resource Server | validates the token | yes — your API |

Think of it as: the Client App is the **front door** of your building that checks people in. The Resource Server is the **room** that checks key cards. The Auth Server is the **government** that issued the key cards. You build the building — the government is already there.


Perfect question. Let me explain all 4 roles in OAuth2 with complete clarity.

---

## The Real World Analogy First

Think of **logging into a hotel with your driving licence**:

```
You          → Resource Owner  (you own your identity data)
Hotel        → Client          (wants to verify who you are)
Govt/DVLA    → Authorization Server (issued your licence, confirms identity)
Hotel System → Resource Server (holds your room, services — protected resources)
```

---

## The 4 OAuth2 Roles

### 1. Resource Owner

```
The USER who owns the data and grants permission to access it.

Example:
- You logging into "Spotify" using Google
- YOU are the resource owner
- You own your Google profile data
- You GRANT permission to Spotify to read it

Real code example — this is just the person clicking
"Allow" on the consent screen. No code represents them.
They are a human being.
```

### 2. Authorization Server

```
Issues tokens. Verifies identity. NOT your code.

Examples of Authorization Servers:
- Google        (accounts.google.com)
- GitHub        (github.com/login/oauth)
- Facebook      (facebook.com/dialog/oauth)
- Okta
- Auth0
- Keycloak      ← open source, you can HOST this yourself
- AWS Cognito

What it does:
1. Shows login page to user
2. Verifies username/password
3. Shows consent screen ("Allow Spotify to read your profile?")
4. Issues Access Token + Refresh Token
5. Provides public keys so Resource Server can verify tokens

YOU DO NOT WRITE THIS.
```

### 3. Client

```
YES — your frontend IS the client.
But also your backend can be the client in some flows.

The CLIENT is whatever application WANTS to access
protected resources on behalf of the user.

Examples:
┌─────────────────────────────────────────────────────┐
│  Your React/Angular frontend  ← Client (most common)│
│  Your mobile app              ← Client              │
│  Your Spring Boot backend     ← Client (server-side)│
│  Postman                      ← Client (testing)    │
└─────────────────────────────────────────────────────┘

The Client:
1. Redirects user to Authorization Server login page
2. Receives Authorization Code after login
3. Exchanges code for Access Token
4. Uses Access Token to call Resource Server

In React (YOU write this):
const loginWithGoogle = () => {
    // Redirect to Google's authorization server
    window.location.href =
        "https://accounts.google.com/o/oauth2/auth" +
        "?client_id=YOUR_CLIENT_ID" +
        "&redirect_uri=http://localhost:3000/callback" +
        "&response_type=code" +
        "&scope=openid profile email";
};
```

### 4. Resource Server

```
YES — your Spring Boot backend IS the Resource Server.

The Resource Server:
- Holds the PROTECTED RESOURCES (your API endpoints, data)
- Accepts requests WITH Access Tokens
- Validates the Access Token
- Returns data if token is valid

THIS IS YOUR SPRING BOOT CODE.
```

---

## Visual of all 4 roles together

```
┌──────────────────────────────────────────────────────────────┐
│                     OAUTH2 FLOW                              │
│                                                              │
│  ┌─────────────┐          ┌──────────────────────────────┐  │
│  │  Resource   │          │    Authorization Server      │  │
│  │   Owner     │─────────▶│  (Google / GitHub / Keycloak)│  │
│  │  (User)     │  logs in │                              │  │
│  └─────────────┘          │  1. verifies identity        │  │
│         │                 │  2. shows consent screen     │  │
│         │                 │  3. issues Access Token      │  │
│         │                 │  4. issues Refresh Token     │  │
│         │                 └──────────────────────────────┘  │
│         │                             │                      │
│         │                    Access Token issued             │
│         │                             │                      │
│         ▼                             ▼                      │
│  ┌─────────────┐          ┌──────────────────────────────┐  │
│  │   Client    │          │      Resource Server         │  │
│  │  (React /   │─────────▶│   (YOUR Spring Boot App)     │  │
│  │  Frontend)  │  calls   │                              │  │
│  │             │  with    │  1. receives request         │  │
│  └─────────────┘  token   │  2. validates Access Token   │  │
│                            │  3. returns protected data   │  │
│                            └──────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Concrete Example — Login with Google

```
Step 1: User clicks "Login with Google" on your React app
─────────────────────────────────────────────────────────
CLIENT (React) redirects user to:
https://accounts.google.com/o/oauth2/auth
    ?client_id=abc123
    &redirect_uri=http://localhost:3000/callback
    &response_type=code
    &scope=openid profile email

Step 2: User logs into Google
─────────────────────────────────────────────────────────
AUTHORIZATION SERVER (Google) shows login page
RESOURCE OWNER (User) enters Google credentials
Google verifies them internally

Step 3: User grants permission
─────────────────────────────────────────────────────────
AUTHORIZATION SERVER shows consent screen:
"Your App wants to access your profile and email. Allow?"
RESOURCE OWNER clicks Allow

Step 4: Authorization Server redirects back with code
─────────────────────────────────────────────────────────
AUTHORIZATION SERVER redirects to:
http://localhost:3000/callback?code=AUTHORIZATION_CODE

Step 5: Client exchanges code for token
─────────────────────────────────────────────────────────
CLIENT (React or Spring Boot) sends POST to Google:
POST https://oauth2.googleapis.com/token
{
    code: "AUTHORIZATION_CODE",
    client_id: "abc123",
    client_secret: "secret",
    redirect_uri: "http://localhost:3000/callback",
    grant_type: "authorization_code"
}

AUTHORIZATION SERVER responds:
{
    "access_token": "ya29.a0AfB...",
    "refresh_token": "1//0eXYZ...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "id_token": "eyJhbGci..."  ← contains user info
}

Step 6: Client calls Resource Server with token
─────────────────────────────────────────────────────────
CLIENT (React) calls YOUR Spring Boot API:
GET http://localhost:8080/api/profile
Authorization: Bearer ya29.a0AfB...

Step 7: Resource Server validates token
─────────────────────────────────────────────────────────
YOUR SPRING BOOT APP:
1. Receives the request
2. Extracts Bearer token
3. Validates token with Google's public keys
4. Extracts user info from token
5. Returns protected data
```

---

## YOUR Spring Boot as Resource Server — the code

```java
// ════════════════════════════════════════
// pom.xml — add this dependency
// ════════════════════════════════════════

// <dependency>
//     <groupId>org.springframework.boot</groupId>
//     <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
// </dependency>


// ════════════════════════════════════════
// application.properties
// ════════════════════════════════════════

// Tell Spring where to get Google's public keys
// Spring uses these to VALIDATE incoming tokens
// spring.security.oauth2.resourceserver.jwt.issuer-uri=
//     https://accounts.google.com


// ════════════════════════════════════════
// SecurityConfig.java — YOU write this
// Makes your Spring Boot a Resource Server
// ════════════════════════════════════════
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http)
            throws Exception {
        http
            .csrf(csrf -> csrf.disable())

            // Tell Spring this app is a Resource Server
            // It will automatically validate JWT tokens
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(
                        jwtAuthenticationConverter())
                )
            )

            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }

    // Converts JWT claims into Spring Security roles
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter =
            new JwtGrantedAuthoritiesConverter();

        // Google puts roles in "authorities" claim
        authoritiesConverter.setAuthoritiesClaimName("authorities");
        authoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter =
            new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}


// ════════════════════════════════════════
// Your Controller — protected resource
// ════════════════════════════════════════
@RestController
public class ProfileController {

    // This endpoint IS the protected resource
    // Resource Server protects it
    @GetMapping("/api/profile")
    public Map<String, Object> getProfile(
            @AuthenticationPrincipal Jwt jwt) {

        // jwt contains claims from the token
        // Spring Resource Server already validated the token
        // by this point — if invalid, request never reaches here

        Map<String, Object> profile = new HashMap<>();
        profile.put("username", jwt.getSubject());
        profile.put("email", jwt.getClaim("email"));
        profile.put("name", jwt.getClaim("name"));
        return profile;
    }

    // Role-based access
    @GetMapping("/admin/dashboard")
    @PreAuthorize("hasRole('ADMIN')")
    public String adminOnly() {
        return "Admin dashboard data";
    }
}
```

---

## What Resource Server does internally

```
Incoming Request:
GET /api/profile
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
        │
        ↓
BearerTokenAuthenticationFilter  (Spring provides automatically)
        │
        │  extracts token from header
        │
        ↓
JwtDecoder  (Spring provides automatically)
        │
        │  fetches Google's public keys from:
        │  https://accounts.google.com/.well-known/openid-configuration
        │
        │  validates:
        │  ✓ signature (was this token really issued by Google?)
        │  ✓ expiry    (is it still valid?)
        │  ✓ issuer    (did it come from accounts.google.com?)
        │  ✓ audience  (was it issued for YOUR app?)
        │
        ↓
JwtAuthenticationConverter
        │
        │  converts JWT claims into Authentication object:
        │  principal   = Jwt object (contains all claims)
        │  authorities = [ROLE_USER] or [ROLE_ADMIN]
        │  authenticated = true
        │
        ↓
SecurityContextHolder.setAuthentication(auth)
        │
        ↓
AuthorizationFilter
        │  checks user has permission for /api/profile
        │
        ↓
Your Controller executes
        │
        ↓
Returns protected data to Client
```

---

## Your Spring Boot can be BOTH Client AND Resource Server

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Scenario 1: Frontend is Client                            │
│  ─────────────────────────────                             │
│  React App          →  Client                              │
│  Google             →  Authorization Server                │
│  Spring Boot API    →  Resource Server                     │
│                                                            │
│  Scenario 2: Spring Boot is Client (server-side flow)      │
│  ─────────────────────────────────────────────────────     │
│  Spring Boot App    →  Client                              │
│  Google             →  Authorization Server                │
│  Google APIs        →  Resource Server                     │
│  (Calendar/Gmail)                                          │
│                                                            │
│  Scenario 3: Spring Boot is BOTH                           │
│  ───────────────────────────────                           │
│  Spring Boot        →  Client (calls another service)      │
│  Spring Boot        →  Resource Server (serves your API)   │
│  Keycloak           →  Authorization Server                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Summary — your exact questions answered

| Role | Is it your code? | What is it? |
|---|---|---|
| **Resource Owner** | No — human user | The person who owns the data and grants access |
| **Client** | YES — your React frontend | The app that requests access on behalf of user |
| **Authorization Server** | NO — Google/GitHub/Keycloak | Issues tokens, verifies identity |
| **Resource Server** | YES — your Spring Boot API | Holds protected data, validates tokens |

```
Resource Owner  = User who clicks "Login with Google"
Client          = Your React app (requests the token)
Auth Server     = Google (issues the token) ← NOT your code
Resource Server = Your Spring Boot API      ← YES your code
                  protects endpoints
                  validates tokens
                  returns protected data
```

![alt text](image-2.png)

OAuth is used to acess protected data!! OIDC can be used to veryify the user too!!

OpenID Connect is built on top of OAuth 2.0 and adds an identity layer. OAuth 2.0 is primarily for authorization, providing access tokens to access resources. OpenID Connect adds an identity token that contains user authentication information, allowing client apps to verify the user's identity. So, OAuth 2.0 handles access control, while OpenID Connect handles user authentication and identity.

## OIDC
OAuth 2.0 is designed only for authorization. It is used for granting access to data and features from one application to another. In OAuth, the client is given a token which it uses to access the data on the resource server, but it doesn’t get to know anything about the user.

OAuth 2.0 works by having a client application request permission from a user to access resources on a resource server. Once authorized, the client receives an access token, which it uses to make API calls to the resource server. This token represents the granted permissions but does not include user identity details, ensuring the client accesses data without knowing who the user is. This separation enhances security and privacy by limiting the client’s knowledge to authorization scope only.

You may have seen that when you try to login to an app, then the app can prompt you to authenticate using your Facebook or Google account. In this case, the app is probably using OpenID Connect. OpenID Connect allows a range of clients, including web-based, mobile, and JavaScript clients, to request and receive information about authenticated sessions and end-users.


### Local user authentication
Let’s suppose we are not using OpenID Connect or any Identity Provider (discussed in the next section) for authentication. In that case, our application will have to maintain a local database in which we will store user information and credentials. This seems like an easy solution but there are a few problems:

- If our organization has lots of applications, then maintaining a database for each application is a tedious task. It requires resources and staff to manage.
- Users find the task of signing up as cumbersome. First, they need to remember so many passwords, and second, it is likely that they’ll use the same password everywhere. If that password gets compromised, all the applications that the user uses with that password will also become compromised

### Identity Providers
To solve the problem of local authentication, Identity Providers came in the picture. As the name suggests, Identity Providers take care of your authentication needs while you focus on your main business. They provide the identity of the user so that organizations can directly onboard the user without asking for any details.

It is a win-win situation for both the user and the organization. The organization is not required to store the personal data of its user, and the users are saved from creating an account each time they need to use some application.

![alt text](image-42.png)

## OpenId Connect Terminologies

1. Identity token
While discussing OAuth, we discussed the authorization code and access token. In the case of OpenId Connect, there is one more token that we can request. This token is called the identity token, which encodes the user’s authentication information.

    In contrast to access tokens, which are only intended to be understood by the resource server, ID tokens are intended to be understood by the client application. The ID token contains the user information in JSON format. The JSON is wrapped into a JWT.

    When a client receives the identity token, it should validate it first. The client must validate the following fields:

    - iss - Client must validate that the issuer of this token is the Authorization Server.
    - aud - Client must validate that the token is meant for the client itself.
    - exp - Client must validate that the token is not expired.

ex:

```json
{
  "iss": "https://server.example.com",
  "sub": "24400320",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1311281970,
  "iat": 1311280970,
  "auth_time": 1311280969,
  "acr": "urn:mace:incommon:iap:silver"
}
```
Let’s look at what these values mean:

- iat - The iat claim identifies the time at which the JWT was issued. This claim can be used to determine the age of the JWT. Its value must be a number containing a NumericDate value.

- auth_time - Time when the End-User authentication occurred. Its value is a JSON number representing the number of seconds from 1970–01–01T0:0:0Z as measured in UTC until the date/time.
- nonce - ID token requests may come with a nonce request parameter to protect from replay attacks. When the request parameter is included, the server will embed a nonce claim in the issued ID token with the same value of the request parameter.

>note: The Identity token contains only basic information about the user. To get the complete user information, the client must send the access token (please note access token should be sent not identity token) to UserInfo endpoint.

![alt text](image-43.png)

![alt text](image-44.png)

![alt text](image-45.png)

### Authorization Code Flow for Authentication

The Authorization code flow for OpenID Connect is similar to the Authorization Code Flow that we discussed in the OAuth 2.0 . The only difference is the change in the value of the scope field. It must contain openid as one of the values, followed by other scope values based on what type of user data the client wants.


- What would happen if the client does not provide an openid in the scope field while sending a request to the authorization server?

    The answer is that in this case, the flow will work as a normal authorization flow. The client app will not get access to the user information as it will not receive the identity token.

- Can user information be fetched from the UserInfo endpoint by sending the access token in the request even if openid was not provided in the scope field when an access token was requested?

    The answer is NO. When we send a request to the token endpoint to fetch the access token, then we must send openid in the scope field. We must also send other scope values like email or address if we want to get this information. The access token that is returned is based on the scope values that were sent with the request. When we hit the UserInfo endpoint, then only that user information is returned which the access token is authorized to get.

    Let’s say that while sending a request to the token endpoint, the scope value is “openid email”. The client sends this request and gets an access token. If the client sends this access token to the UserInfo endpoint, it will get only email information. It will not get an address or any other information.

![alt text](image-46.png)

![alt text](image-47.png)

If the scope does not include 'openid', the flow works as a normal OAuth 2.0 authorization flow. In this case, the client app will not receive an identity token and cannot access user information through the UserInfo endpoint.
### Implicit Code Flow for Authentication
This flow is also similar to the Implicit grant type discussed in the OAuth chapter. This flow is used for single-page JavaScript apps or those apps which do not have a backend.

In Implicit flow, the response_type field can either take token or id_token or token_id_token as value. This leads to some interesting cases depending upon what is provided in the scope field.

![alt text](image-48.png)

![alt text](image-49.png)

![alt text](image-50.png)

### Hybrid Code Flow for Authentication

In Authorization flow, we first get authorization token from authorization endpoint and then get the access token and identity token from the token endpoint. This takes some time as two server calls are needed.

In the implicit flow, we get the access token and identity token from the authorization endpoint. This is faster but is not secure.

In the hybrid flow, the client gets immediate access to the identity token from the authorization endpoint itself. The client also gets the authorization code from the authorization endpoint. Later, it fetches the access token from the token endpoint which can be used to get further user info.

#### Hybrid flow type#
In hybrid flows, the response_type field should contain code as one of the values and token or id_token or token+id_token as the other value.

![alt text](image-51.png)

![alt text](image-52.png)

![alt text](image-53.png)

In the hybrid code flow, tokens can be issued by both the authorization endpoint and the token endpoint because they serve different purposes and have different security considerations. The authorization endpoint issues tokens immediately to the client for faster access, but these tokens may have fewer claims or different security properties. The token endpoint issues tokens after the client exchanges the authorization code, allowing for more secure and complete tokens. This separation helps balance speed and security. For example, the access token from the authorization endpoint and the token endpoint may differ due to different lifetimes or security scopes.


OpenID Connect (OIDC) builds on OAuth 2.0 by adding authentication on top of authorization. While OAuth 2.0 handles authorization (granting access to resources), OIDC adds the identity layer, allowing the client to authenticate the user and obtain user information via the identity token and UserInfo endpoint.

OAuth 2.0 was designed primarily for authorization, meaning it grants access to resources without verifying the user's identity. It focuses on allowing third-party applications to access user data with permission. Authentication, which is about verifying who the user is, was not part of OAuth's original purpose. OpenID Connect was introduced on top of OAuth 2.0 to add this authentication layer, enabling clients to confirm the user's identity.

![alt text](image-3.png)

![alt text](image-4.png)

### Auth0 Registration:

![alt text](image-5.png)

![alt text](image-6.png)


![alt text](image-7.png)

After registration, we get Client id and Secret

![alt text](image-8.png)

![alt text](image-9.png)

![alt text](image-10.png)

### SecurityConfig.java

![alt text](image-11.png)

Basic Controller class, just for testing

![alt text](image-12.png)

Notice one thing that, I have not created any User in our Springboot app.

![alt text](image-13.png)

![alt text](image-14.png)


![alt text](image-15.png)


![alt text](image-16.png)


What? With just pom.xml, application.properties and SecurityConfig.java changes, we able to run the complete OAuth2 flow.

Answer is Yes, Springboot Security framework provides the compete functionality of OAUTH2 protocol, we don’t have to code anything.


But here is the twist:

When I tried to access the "/users" API through postman (not through browser), it takes me back to Login page, why?

![alt text](image-17.png)

Because, Springboot assume that, Oauth2 login will be done on a browser, so by-default it creates SESSION.

![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

just rightmost image 

![alt text](image-21.png)

![alt text](image-22.png)

![alt text](image-23.png)

![alt text](image-24.png)

![alt text](image-25.png)


![alt text](image-26.png)

![alt text](image-27.png)

![alt text](image-28.png)


![alt text](image-29.png)


![alt text](image-30.png)


![alt text](image-31.png)


![alt text](image-32.png)

OAuth2LoginAuthenticationFilter parent class has one method "onAuthenticationSuccess()" at last,  which by default do some cleanup task once authentication process completes, I have overwrite that method and set the ID_TOKEN in the response body.

![alt text](image-33.png)

![alt text](image-34.png)

Start the application

![alt text](image-35.png)


After Authorizing from the Authorization server:

![alt text](image-36.png)


Now, only 1 task left, now Client will pass this TOKEN with every request and we have to verify it.


![alt text](image-37.png)

Created New Filter to Validate the token

![alt text](image-38.png)

Utility Class, just to validate the token

![alt text](image-39.png)

![alt text](image-40.png)

![alt text](image-41.png)



 OAuth2 is often misunderstood. Let me build it up from the concept, then show the full flow with code.Now let's look at the actual Spring Boot code for both sides — the Client and the Resource Server.

 ![alt text](image-56.png)

---

### The key mental model before diving into code

OAuth2 is a **delegation protocol** — you are letting your app act on your behalf with another service, without giving it your password. JWT is just the *format* the token might use. OAuth2 itself doesn't care — the token could be opaque (a random string) or a JWT. Google and GitHub use JWTs.

---

### Part 1 — Spring Boot as OAuth2 Client (Login with Google)

**`pom.xml`**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

**`application.yml`**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: YOUR_GOOGLE_CLIENT_ID
            client-secret: YOUR_GOOGLE_CLIENT_SECRET
            scope: openid, email, profile
```

That's it for the client side — Spring auto-configures the entire redirect → consent → callback → token exchange flow.

**`SecurityConfig.java`**
```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("/dashboard", true)
            );
        return http.build();
    }
}
```

**Reading user info after login**
```java
@GetMapping("/dashboard")
public String dashboard(@AuthenticationPrincipal OAuth2User principal) {
    String email = principal.getAttribute("email");
    String name  = principal.getAttribute("name");
    return "Welcome " + name + " (" + email + ")";
}
```

---

### Part 2 — Spring Boot as OAuth2 Resource Server (protecting your API)

This is when *your own API* accepts tokens issued by an Auth Server (like Keycloak, Auth0, or Google).

**`pom.xml`**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**`application.yml`**
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Spring fetches the public key automatically from here
          issuer-uri: https://accounts.google.com
          # OR provide the JWKS URI directly:
          # jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs
```

**`SecurityConfig.java`**
```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults()) // validates JWT automatically
            );
        return http.build();
    }
}
```

**Accessing claims in controller**
```java
@GetMapping("/api/profile")
public Map<String, Object> profile(@AuthenticationPrincipal Jwt jwt) {
    return Map.of(
        "subject",  jwt.getSubject(),
        "email",    jwt.getClaimAsString("email"),
        "roles",    jwt.getClaimAsStringList("roles"),
        "expires",  jwt.getExpiresAt()
    );
}
```

---

### How validation actually works (the JWKS magic)

When a request hits your Resource Server with `Bearer <token>`:

1. Spring's `JwtDecoder` decodes the token header to find which key ID (`kid`) was used to sign it
2. It calls the Auth Server's `/.well-known/jwks.json` endpoint **once** to fetch the public key, then **caches it**
3. It verifies the signature locally using that public key — no Auth Server call on every request
4. It checks `exp` (expiry), `iss` (issuer), `aud` (audience)
5. It builds a `JwtAuthenticationToken` and puts it in the `SecurityContextHolder`

```
Token header:  { "alg": "RS256", "kid": "key-id-123" }
               ↓
Spring fetches: GET https://accounts.google.com/.well-known/jwks.json
               ↓
Finds the public key matching kid="key-id-123"
               ↓
Verifies RSA signature locally → sets Authentication
```

---

### OAuth2 vs plain JWT — the one-line distinction

| | Plain JWT (custom) | OAuth2 + JWT |
|---|---|---|
| Who issues the token | Your own app | A dedicated Auth Server (Google, Keycloak) |
| Key management | You manage the secret | Auth Server publishes public key via JWKS |
| User consent flow | None | Full redirect + consent screen |
| Token types | Just access token | access + refresh + id_token |
| Standard | No | Yes (RFC 6749) |

OAuth2 is the **protocol**. JWT is one possible **token format** within it. You can have OAuth2 without JWT, and JWT without OAuth2 — but together they're the industry standard.











