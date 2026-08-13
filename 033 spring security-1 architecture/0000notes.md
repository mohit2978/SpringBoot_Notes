
![alt text](image.png)

![alt text](image-2.png)

![alt text](image-6.png)

![alt text](image-7.png)



![alt text](image-1.png)

---

## SpringBoot Security - 1

There are various types of attacks:
- CSRF (Cross-Site Request Forgery)
- CORS (Cross-Origin Resource Sharing)
- SQL Injection
- XSS (Cross-Site Scripting)

And we need to protect our resources from these attacks, and for that we need proper:
- **Authentication**: Verify who you are
- **Authorization**: Checks what you are allowed to do

That's where Spring boot Security comes into the picture.

### Architecture of Spring Boot Security

In video no #18, we have already seen, what are filters and where exactly they fit.

![alt text](033-video-ref-filters-vs-interceptors.png)

![alt text](033-filters-interceptors-recap-diagram.png)

Now, lets enhance it for understanding Spring Security:

![alt text](033-filters-chain-with-security-filter-chain.png)

Within Security Filter Chain, the flow is:

![alt text](033-security-filter-chain-architecture.png)
![alt text](033_spring_security_architecture_250716_003028_2.jpg) ![alt text](033_spring_security_architecture_250716_003028_3.jpg) 
If spring boot project is already present, add below dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <!--
    Provides core feature like:
    - Authentication
    - Authorization
    - Security filters etc.
    -->
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <!--
    Enable persistent session management
    Using relational DB
    -->
    <artifactId>spring-session-jdbc</artifactId>
</dependency>
```

If setting up new Spring boot project:

Go to spring initializer i.e. "start.spring.io"

![alt text](033-spring-initializer-setup.png)

And if we want to persist the session in relational DB, then we need to add below dependency in pom.xml

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-jdbc</artifactId>
</dependency>
```

Now, lets understand the end to end flow with an example for each individual Authentication and Authorization mechanism:

1. Form Login (Stateful)
2. Basic Authentication (Stateless)
3. JWT (Stateless)
4. OAuth2
   - a. Authorization Code (Stateful or Stateless)
   - b. Client Credentials (Stateless)
   - c. Password Grant (Stateless)
5. API Key Authentication (Stateless)

Etc..


