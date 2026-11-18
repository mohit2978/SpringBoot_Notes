# @ConditionalOnProperty (and Conditional Beans)

## Table of Contents

1. [What is @ConditionalOnProperty?](#1-what-is-conditionalonproperty)
2. [The problem: every component is always created](#2-the-problem-every-component-is-always-created)
3. [The solution: @ConditionalOnProperty](#3-the-solution-conditionalonproperty)
4. [How the decision is made (prefix, value, havingValue, matchIfMissing)](#4-how-the-decision-is-made)
5. [Injecting a bean that might not exist (@Autowired required = false)](#5-injecting-a-bean-that-might-not-exist)
6. [Use case 2: one codebase, different deployments](#6-use-case-2-one-codebase-different-deployments)
7. [Advantages and disadvantages](#7-advantages-and-disadvantages)
8. [@ConditionalOnMissingBean](#8-conditionalonmissingbean)
9. [The @Conditional family](#9-the-conditional-family)
10. [Interview Questions](#10-interview-questions)

---

## 1. What is @ConditionalOnProperty?

**A bean is created conditionally: the bean may or may not be created.**

`@ConditionalOnProperty` lets a **property in `application.properties`** decide whether Spring creates a bean. Put it on a `@Component` class or a `@Bean` method:

- property matches → the bean is created
- property doesn't match → Spring **skips** the bean entirely (it never exists in the application context)

**We are controlling the creation of beans.** It's mainly used for **feature toggles** and for choosing between implementations per environment.

---

## 2. The problem: every component is always created

We already know the behavior of the program below: every `@Component` found by component scanning is created at startup.

![svg](<svgs/01-problem-all-beans-created.svg>)

```java
@Component
public class DBConnection {

    @Autowired
    MySQLConnection mySQLConnection;

    @Autowired
    NoSQLConnection noSQLConnection;

    @PostConstruct
    public void init() {
        System.out.println("DB Connection Bean Created with dependencies below:");
        System.out.println("is MySQLConnection object Null: " + Objects.isNull(mySQLConnection));
        System.out.println("is NoSQLConnection object Null: " + Objects.isNull(noSQLConnection));
    }
}
```

```java
@Component
public class MySQLConnection {

    MySQLConnection() {
        System.out.println("initialization of MySQLConnection Bean");
    }
}

@Component
public class NoSQLConnection {

    NoSQLConnection() {
        System.out.println("initialization of NoSQLConnection Bean");
    }
}
```

**Output: all beans are created**

```text
2024-05-25T10:55:57.207+05:30  INFO 6700 --- [main] w.s.c.ServletWebServerApplicationContext
initialization of MySQLConnection Bean
initialization of NoSQLConnection Bean
DB Connection Bean Created with dependencies below:
is MySQLConnection object Null: false
is NoSQLConnection object Null: false
2024-05-25T10:55:57.381+05:30  INFO 6700 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer
2024-05-25T10:55:57.387+05:30  INFO 6700 --- [main] c.c.l.SpringbootApplication
```

**But how would you handle these use cases?**

- **Use case 1:** we want to create only **one** bean, either `MySQLConnection` or `NoSQLConnection`.
- **Use case 2:** two components (services) share the same codebase, but one needs `MySQLConnection` and the other needs `NoSQLConnection`.

---

## 3. The solution: @ConditionalOnProperty

```java
@Component
@ConditionalOnProperty(prefix = "sqlconnection", value = "enabled", havingValue = "true", matchIfMissing = false)
public class MySQLConnection {

    MySQLConnection() {
        System.out.println("initialization of MySQLConnection Bean");
    }
}

@Component
@ConditionalOnProperty(prefix = "nosqlconnection", value = "enabled", havingValue = "true", matchIfMissing = false)
public class NoSQLConnection {

    NoSQLConnection() {
        System.out.println("initialization of NoSQLConnection Bean");
    }
}
```

```java
@Component
public class DBConnection {

    @Autowired(required = false)          // the bean may not exist (see section 5)
    MySQLConnection mySQLConnection;

    @Autowired(required = false)
    NoSQLConnection noSQLConnection;

    @PostConstruct
    public void init() {
        System.out.println("DB Connection Bean Created with dependencies below:");
        System.out.println("is MySQLConnection object Null: " + Objects.isNull(mySQLConnection));
        System.out.println("is NoSQLConnection object Null: " + Objects.isNull(noSQLConnection));
    }
}
```

**application.properties**

```properties
sqlconnection.enabled=true
```

**Output: only `MySQLConnection` is created**

```text
2024-05-25T11:32:03.951+05:30  INFO 8352 --- [main] w.s.c.ServletWebServerApplicationContext
initialization of MySQLConnection Bean
DB Connection Bean Created with dependencies below:
is MySQLConnection object Null: false
is NoSQLConnection object Null: true
2024-05-25T11:32:04.102+05:30  INFO 8352 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer
2024-05-25T11:32:04.106+05:30  INFO 8352 --- [main] c.c.l.SpringbootApplication
```

- `NoSQLConnection` was **not** created: `nosqlconnection.enabled` isn't in `application.properties`, and `matchIfMissing = false`.
- To create it, we'd need to put **`nosqlconnection.enabled=true`** in `application.properties`.

---

## 4. How the decision is made

**Rule:** if **`prefix.value` = `havingValue`** in `application.properties`, only then is the bean created.

![svg](<svgs/02-how-conditionalonproperty-decides.svg>)

| Attribute | Meaning | In our example |
|---|---|---|
| `prefix` | First part of the property key | `sqlconnection` |
| `value` / `name` | Rest of the key (aliases of each other) | `enabled` → key = `sqlconnection.enabled` |
| `havingValue` | The value the property must have | `"true"` |
| `matchIfMissing` | What to do if the property is **not written at all** | `false` |

**`matchIfMissing`** tells Spring what to do when the property (`prefix.value`) is **missing** from `application.properties`:
- **`matchIfMissing = true`:** behave as if the property was set to `havingValue`. You don't need to put it in properties; the bean is created automatically. Use this for features that should be **on by default**.
- **`matchIfMissing = false`** (the default): you **must** mention the property in `application.properties` to create the bean, or it won't be created. Use this for features that should be **off unless explicitly enabled**.

![svg](<svgs/03-matchifmissing-truth-table.svg>)

Details worth knowing:
- If you **omit `havingValue`**, the condition matches when the property exists and its value is anything **except `false`**.
- Comparison is case-insensitive for the value (`TRUE` works too).
- Short form: `@ConditionalOnProperty(name = "sqlconnection.enabled", havingValue = "true")`.
- Several names: `@ConditionalOnProperty(prefix = "app", name = {"cache", "metrics"})` matches only if **all** of them match.
- Relaxed binding works: `sqlconnection.enabled` can also come from the env var `SQLCONNECTION_ENABLED=true` or the command line `--sqlconnection.enabled=true`.

---

## 5. Injecting a bean that might not exist

On `@Autowired` you also need to put **`required = false`**, because by default **`required = true`**. Sometimes the bean will **not be created** because of the condition, and a required injection of a missing bean makes startup fail.

![svg](<svgs/05-injecting-an-optional-bean.svg>)

```text
***************************
APPLICATION FAILED TO START
***************************
Description:
Field noSQLConnection in com.example.DBConnection required a bean of type
'com.example.NoSQLConnection' that could not be found.
```

**Cleaner designs** (instead of null checks everywhere):

```java
// 1. One interface, two implementations: inject the interface.
//    Only the enabled implementation exists, so there's exactly one candidate.
public interface DBConnection { void connect(); }

@Component
@ConditionalOnProperty(name = "db.type", havingValue = "mysql")
public class MySQLConnection implements DBConnection { public void connect() { /* … */ } }

@Component
@ConditionalOnProperty(name = "db.type", havingValue = "nosql")
public class NoSQLConnection implements DBConnection { public void connect() { /* … */ } }

@Service
public class ReportService {
    private final DBConnection connection;          // MySQL or NoSQL, decided by db.type
    public ReportService(DBConnection connection) { this.connection = connection; }
}
```

```properties
db.type=mysql
```

A single property with different values (`db.type=mysql|nosql`) also avoids the risk of enabling both by mistake.

```java
// 2. Optional / ObjectProvider when the bean really is optional
public ReportService(Optional<NoSQLConnection> noSql) { … }              // empty if skipped
public ReportService(ObjectProvider<NoSQLConnection> provider) {
    NoSQLConnection conn = provider.getIfAvailable();                    // null if skipped
}
```

---

## 6. Use case 2: one codebase, different deployments

Two services are built from the **same code**. Each deployment just sets a different property:

![svg](<svgs/04-same-codebase-two-deployments.svg>)

```properties
# Component A (orders):     application-orders.properties
sqlconnection.enabled=true

# Component B (analytics):  application-analytics.properties
nosqlconnection.enabled=true
```

```bash
java -jar app.jar --spring.profiles.active=orders       # creates MySQLConnection only
java -jar app.jar --spring.profiles.active=analytics    # creates NoSQLConnection only
```

The same approach works for **feature toggles**: ship a new feature switched off (`newcache.enabled=false`), then turn it on per environment without changing code.

---

## 7. Advantages and disadvantages

**Advantages**

1. **Toggling of features:** turn functionality on/off with configuration only.
2. **Avoid cluttering the application context with unnecessary beans.**
3. **Save memory:** beans that aren't needed are never created.
4. **Reduce application startup time:** fewer beans to create and initialise.

**Disadvantages**

1. **Misconfiguration can happen:** a typo in the property name (`sqlconection.enabled`) silently skips the bean, with no error at startup until something needs it.
2. **Code complexity when overused:** too many conditions make it hard to know which beans actually exist.
3. **Multiple bean creation with the same configuration brings confusion:** if we put the **same prefix & value on multiple beans**, enabling one property creates several beans at once and it's unclear which one is used. That should be avoided.
4. **Complexity in managing:** you need to manage `application.properties` **as well as** the beans, both together.

**How to see what was decided:** run with `--debug` (or `debug=true`). The **CONDITIONS EVALUATION REPORT** lists every conditional bean and why it matched or not:

```text
Negative matches:
-----------------
   NoSQLConnection:
      Did not match:
         - @ConditionalOnProperty (nosqlconnection.enabled=true) did not find property 'enabled' (OnPropertyCondition)
```

The same report is available at runtime via `/actuator/conditions` (see the *037 actuator* notes).

---

## 8. @ConditionalOnMissingBean

A conditional annotation that tells Spring: **"only create this bean if no other bean of this type already exists in the application context."** It's the backbone of how Spring Boot's auto-configuration avoids overriding beans you've defined yourself.

**Core use case:** auto-configuration provides sensible defaults but lets you override them.

![svg](<svgs/06-conditionalonmissingbean-default-vs-custom.svg>)

**Example: a simplified version of how Spring Boot's own auto-configuration works internally**

```java
@Configuration
public class MessagingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(MessageService.class)
    public MessageService defaultMessageService() {
        System.out.println("Creating DEFAULT MessageService");
        return new DefaultMessageService();
    }
}
```

**Scenario 1: you define nothing yourself**

```java
// No custom MessageService bean anywhere in your app
```

Result: Spring creates `DefaultMessageService` automatically. No other `MessageService` bean exists, so the condition (`@ConditionalOnMissingBean`) is satisfied.

**Scenario 2: you define your own bean**

```java
@Configuration
public class MyAppConfig {
    @Bean
    public MessageService customMessageService() {
        System.out.println("Creating CUSTOM MessageService");
        return new CustomMessageService();
    }
}
```

Result: Spring sees that a `MessageService` bean already exists (yours), so it **skips** `defaultMessageService()` entirely. Your custom bean wins, with no conflict and no duplicate-bean error.

**Why this matters: this is exactly how Spring Boot's real auto-configuration works**

When you add `spring-boot-starter-data-jpa`, Spring Boot auto-configures a `DataSource` bean with sensible defaults (reading `spring.datasource.*` properties). But if you define your own `DataSource` bean:

```java
@Bean
public DataSource myCustomDataSource() {
    return DataSourceBuilder.create()
        .url("jdbc:mysql://custom-host/mydb")
        .build();
}
```

Spring Boot's internal auto-configuration (`DataSourceAutoConfiguration`) creates its pooled `DataSource` only under `@ConditionalOnMissingBean(DataSource.class)`, so the moment you define your own, Spring Boot's default is skipped. **This is why Spring Boot "just works" out of the box but still lets you override any default without fighting the framework.** There's no need to explicitly disable auto-configuration: just define your own bean of the same type.

> **Use it in auto-configuration / `@Configuration` classes, not as a trick on ordinary `@Component`s.** The condition is evaluated while bean definitions are being registered, so it only sees beans registered **before** it. Spring Boot guarantees auto-configurations are processed **after** your own beans, which is why it works reliably there. Between two of your own component-scanned classes, the order isn't guaranteed.

**One-line interview answer:**
"`@ConditionalOnMissingBean` tells Spring to only create a bean if none of that type already exists in the context. It's how Spring Boot's auto-configuration provides sensible defaults for things like `DataSource` or `ObjectMapper` while letting you override them completely just by defining your own bean of the same type, without explicitly disabling anything."

---

## 9. The @Conditional family

![svg](<svgs/07-conditional-annotations-family.svg>)

| Annotation | Creates the bean only if … | Typical use |
|---|---|---|
| `@ConditionalOnProperty` | a property has a specific value | Feature toggles, choosing an implementation |
| `@ConditionalOnMissingBean` | no bean of this type exists yet | Overridable defaults in auto-configuration |
| `@ConditionalOnBean` | another specific bean **does** exist | Beans that depend on an optional feature |
| `@ConditionalOnClass` | a class is on the classpath | "MySQL driver present → configure MySQL DataSource" |
| `@ConditionalOnMissingClass` | a class is **not** on the classpath | Fallback implementations |
| `@ConditionalOnWebApplication` | the app is a web app (servlet/reactive) | Web-only beans |
| `@ConditionalOnExpression` | a SpEL expression is true | Combining properties: `"${a.on:false} and ${b.on:false}"` |
| `@Profile("prod")` | a Spring profile is active | Environment-specific beans |
| `@Conditional(MyCondition.class)` | your own `Condition.matches(...)` returns true | Anything custom |

**Writing your own condition:**

```java
public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().getProperty("os.name", "").toLowerCase().contains("linux");
    }
}

@Bean
@Conditional(OnLinuxCondition.class)
public FileWatcher linuxFileWatcher() { return new InotifyFileWatcher(); }
```

**`@ConditionalOnProperty` vs `@Profile`:** `@Profile` switches whole groups of beans by environment name (`dev`, `prod`). `@ConditionalOnProperty` switches individual beans by a specific property, so it's finer-grained and better suited to feature flags. They can be combined: a profile's properties file sets the flags.

---

## 10. Interview Questions

**Q1. What is `@ConditionalOnProperty`?**
A Spring Boot condition that creates a bean (component class or `@Bean` method) only if a property in the environment has a given value. It's evaluated at startup, before the bean is created.

**Q2. Explain `prefix`, `value`/`name`, `havingValue` and `matchIfMissing`.**
`prefix` + `name` form the property key (`sqlconnection.enabled`). `havingValue` is the required value. `matchIfMissing` decides the outcome when the property isn't set at all: `true` = create the bean (on by default), `false` (default) = don't create it.

**Q3. What happens if `havingValue` is not specified?**
The condition matches when the property is present and not equal to `false`.

**Q4. A bean guarded by `@ConditionalOnProperty` is injected with `@Autowired` and the property is off. What happens?**
Startup fails ("required a bean of type … that could not be found"), because `@Autowired` is required by default. Use `@Autowired(required = false)`, `Optional<T>`, `ObjectProvider<T>`, or (better) inject a common interface that always has exactly one enabled implementation.

**Q5. How would you build a feature toggle in Spring Boot?**
`@ConditionalOnProperty(name = "feature.x.enabled", havingValue = "true")` on the feature's beans, with the property set per environment (profile files, env vars, config server). For toggles that must change **at runtime without a restart**, use a feature-flag library/service (e.g. Unleash, LaunchDarkly, Togglz) instead, because conditions are only evaluated at startup.

**Q6. How do you choose between two implementations of an interface at startup?**
Put `@ConditionalOnProperty(name = "db.type", havingValue = "mysql")` on one and `havingValue = "nosql"` on the other, and inject the interface. Only the matching implementation exists.

**Q7. What are the disadvantages of `@ConditionalOnProperty`?**
Silent misconfiguration (typos just skip the bean), complexity when overused, confusion when several beans share the same property, and having to keep properties and beans in sync.

**Q8. How do you find out why a conditional bean was or wasn't created?**
Run with `--debug` to print the conditions evaluation report, or call `/actuator/conditions`.

**Q9. What is `@ConditionalOnMissingBean`? Where should it be used?**
It creates the bean only if no bean of that type exists. It's meant for auto-configuration / library configuration classes, so users can override defaults by declaring their own bean. It's unreliable between ordinary component-scanned classes because registration order isn't guaranteed.

**Q10. `@ConditionalOnBean` vs `@ConditionalOnMissingBean`?**
Opposites: `OnBean` requires another bean to exist, `OnMissingBean` requires that none exists.

**Q11. `@ConditionalOnClass` — how does Spring Boot use it?**
Auto-configurations check whether a library class is on the classpath (e.g. `DataSource`, `ObjectMapper`, `DispatcherServlet`) and configure the related beans only if it is. That's how adding a starter dependency "switches on" features.

**Q12. `@ConditionalOnProperty` vs `@Profile`?**
`@Profile` activates beans by environment name. `@ConditionalOnProperty` activates them by a specific property value: more granular, ideal for feature flags.

**Q13. How do you write a custom condition?**
Implement `org.springframework.context.annotation.Condition` (`matches(ConditionContext, AnnotatedTypeMetadata)`) and use `@Conditional(MyCondition.class)` on the bean.
