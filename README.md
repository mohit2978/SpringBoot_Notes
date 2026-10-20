# 🍃 Spring Boot — Deep Dive Notes

Personal study notes covering **50 Spring Boot topics** with hands-on code examples.
Built while working as a Software Engineer at Samsung SDS to strengthen core framework internals.

---

## 📚 Topics Covered

### 🔧 Core Spring Boot
| # | Topic |
|---|-------|
| 001 | [Introduction to Spring Boot](./001%20intro) |
| 002 | [Getting Started — Project Setup](./002%20start%20with%20springboot) |
| 003 | [Maven — Dependency Management](./003%20maven) |
| 004 | [General Annotations](./004%20general%20annotations) |
| 005 | [Bean Lifecycle](./005%20bean%20lifecycle) |
| 006 | [Dependency Injection](./006%20dependency%20injection) |
| 007 | [Bean Scopes](./007%20scopes) |
| 008 | [Unsatisfied Dependency Problem](./008%20Unsatisified%20dependency%20problem) |
| 009 | [@ConditionalOnProperty](./009%20conditionalOn%20property) |
| 010 | [Profiles](./010%20profiles) |
| 046 | [@ConfigurationProperties](./046%20ConfigurationProperties) |

### 💾 Transactions
| # | Topic |
|---|-------|
| 012 | [Transactional — Part 1](./012%20transactional-1) |
| 013 | [Transactional — Part 2 (Propagation)](./013%20transactional-2%20propogation) |
| 014 | [Transactional — Part 3 (Isolation Levels)](./014%20transactional-3%20isolation%20levels) |
| 015 | [TransactionTemplate vs @Transactional (Programmatic vs Declarative)](./015%20transaction-4%20transactionTemplate) |

### ✂️ AOP, Async & Concurrency
| # | Topic |
|---|-------|
| 011 | [Aspect Oriented Programming (AOP)](./011%20Aspect%20Oriented%20Programming%20(AOP)) |
| 016 | [Async — Part 1](./016%20Async) |
| 017 | [Async — Part 2 (Async and Transactional)](./017%20Async%20and%20Transactional) |
| 018 | [Spring Boot Threads, Thread Pools & Concurrency Internals](./018%20springboot%20threads%20and%20threadpool) |

### 🌐 REST & Web
| # | Topic |
|---|-------|
| 019 | [Custom Interceptor](./019%20custom%20interceptor) |
| 020 | [Filter vs Interceptor](./020%20filter%20interceptor) |
| 021 | [ResponseEntity & HTTP Codes](./021%20responseEntity%20and%20codes) |
| 022 | [Exception Handling](./022%20exceptionHandling) |
| 049 | [REST APIs, Validations & Request Lifecycle](./049%20rest%20apis%20and%20validations) |
| 050 | [HATEOAS](./050%20HATEOAS) |

### 🗄️ JPA & Database (14 Parts)
| # | Topic |
|---|-------|
| 023 | [Introduction to JDBC & JPA](./023%20jdbc%20java%20and%20jpa%20intro) |
| 024 | [JPA — Part 1 Basic Definitions & Flows](./024%20jpa-1%20basic%20definitions%20and%20flows) |
| 025 | [JPA — Part 2 Basic Definitions & Flows](./025%20jpa-2%20basic%20definitions%20and%20flows) |
| 026 | [JPA — Basic Terms & Database Communication Internals](./026%20jpa%20basic%20terms) |
| 027 | [JPA — Part 3 First Level Caching](./027%20jpa-3%20first%20level%20caching) |
| 028 | [JPA — Part 4 Second Level (L2) Caching & Spring Boot Default Cache](./028%20jpa-4%20second%20level%20caching) |
| 029 | [Hibernate Caching Demystified — Level 1 (L1) vs Level 2 (L2) Cache](./029%20l1%20vs%20l2%20cache) |
| 030 | [JPA — Part 5 General Table Annotations & DTO Mapping](./030%20jpa-5%20general%20table%20annotations) |
| 031 | [JPA — Part 6 One-to-One Mapping & Cascade Type](./031%20jpa-6%20one%20to%20one%20mapping%20and%20cascade%20type) |
| 032 | [JPA — Part 7 One-to-Many & Many-to-Many](./032%20jpa-7%20One%20to%20many%20and%20many%20to%20many) |
| 033 | [Spring Data JPA Repositories, Pagination & Sorting](./033%20spring%20jpa%20Repository%20pagination) |
| 034 | [JPA — Part 8 JPQL & Derived Queries](./034%20jpa-8%20jpql%20derived%20query) |
| 035 | [JPA — Part 9 & 10 Native Query, Specification & Criteria API](./035%20jpa-9%20native%20query%20specification%20and%20criteria%20api) |
| 036 | [JPA — N+1 Problem with Practical Examples & Solutions](./036%20N+1%20problem%20with%20example) |

### 📊 Monitoring
| # | Topic |
|---|-------|
| 037 | [Actuator](./037%20actuator) |

### 🔐 Spring Security (7 Parts)
| # | Topic |
|---|-------|
| 038 | [Common Security Attacks](./038%20attacks) |
| 039 | [Spring Security Architecture](./039%20spring%20security-1%20architecture) |
| 040 | [Spring Security — Part 2 User Creation](./040%20spring%20security-2%20user%20creation) |
| 041 | [Spring Security — Part 3 Form Login](./041%20spring%20security-3%20form%20login) |
| 042 | [Spring Security — Part 4 Basic Auth](./042%20spring%20security%20-4%20basic%20auth) |
| 043 | [Spring Security — Part 5 JWT](./043%20spring%20security-5%20JWT) |
| 044 | [Spring Security — Part 6 OAuth2](./044%20spring%20security-6%20Oauth) |
| 045 | [Spring Security — Part 7 Method Security & Role-Based Auth](./045%20spring%20security-7%20Method%20security%20Role%20based%20auth) |

### ⚡ Reactive Programming
| # | Topic |
|---|-------|
| 047 | [Spring WebFlux (Reactive Streams, Mono & Flux)](./047%20spring%20webflux) |

### 💡 Production Scenarios & Architecture Patterns
| # | Topic |
|---|-------|
| 048 | [Real-World Architecture Scenarios & Performance Optimizations](./048%20scenarios) |

---

## 🛠️ Tech Stack
![Java](https://img.shields.io/badge/Java-ED8B00?style=flat&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat&logo=spring-boot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring_Security-6DB33F?style=flat&logo=spring&logoColor=white)
![Hibernate](https://img.shields.io/badge/Hibernate-59666C?style=flat&logo=hibernate&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)

---

## 👨‍💻 About Me
**Mohit Kumar** — Software Engineer @ Samsung SDS  
NIT Allahabad M.Tech | GATE 2020 — 98.68 percentile  

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://linkedin.com/in/mohit2978)
[![GitHub](https://img.shields.io/badge/GitHub-100000?style=flat&logo=github&logoColor=white)](https://github.com/mohit2978)

---
