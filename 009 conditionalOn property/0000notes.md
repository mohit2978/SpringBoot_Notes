
![alt text](<009conditional On property_250202_213507_250716_002317_1.jpg>) ![alt text](<009conditional On property_250202_213507_250716_002317_2.jpg>) ![alt text](<009conditional On property_250202_213507_250716_002317_3.jpg>) ![alt text](<009conditional On property_250202_213507_250716_002317_4.jpg>)



**`@ConditionalOnMissingBean`**

A conditional annotation that tells Spring: **"only create this bean if no other bean of this type already exists in the application context."** It's the backbone of how Spring Boot's auto-configuration avoids overriding beans you've defined yourself.

**Core use case: auto-configuration provides sensible defaults, but lets you override them**

**Example — a simplified version of how Spring Boot's own auto-configuration works internally:**

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

**Scenario 1 — you define nothing yourself:**
```java
// No custom MessageService bean anywhere in your app
```
Result: Spring creates `DefaultMessageService` automatically, since no other `MessageService` bean exists — the condition (`@ConditionalOnMissingBean`) is satisfied.

**Scenario 2 — you define your own bean:**
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
Result: Spring sees a `MessageService` bean already exists (yours), so it **skips** creating `defaultMessageService()` entirely — your custom bean wins, no conflict, no duplicate bean error.

**Why this matters — this is EXACTLY how Spring Boot's real auto-configuration works**

When you add `spring-boot-starter-data-jpa`, Spring Boot auto-configures a `DataSource` bean for you with sensible defaults (reading `application.yml` properties). But if you define your own `DataSource` bean:
```java
@Bean
public DataSource myCustomDataSource() {
    return DataSourceBuilder.create()
        .url("jdbc:mysql://custom-host/mydb")
        .build();
}
```
Spring Boot's internal auto-configuration class (real example: `DataSourceAutoConfiguration`) is annotated with `@ConditionalOnMissingBean(DataSource.class)` — so the moment you define your own, Spring Boot's default is automatically skipped. **This is precisely why Spring Boot "just works" out of the box, but also lets you fully override any default without fighting the framework** — no need to explicitly disable auto-configuration, just define your own bean of the same type.

**Related conditionals worth knowing (same family, quick mention):**
- `@ConditionalOnBean` — opposite: only create this bean IF another specific bean already exists (used for beans that depend on optional features being present)
- `@ConditionalOnProperty` — only create the bean if a specific property is set to a certain value (e.g., `feature.new-cache.enabled=true`)
- `@ConditionalOnClass` — only create the bean if a specific class is on the classpath (this is how Spring Boot detects "oh, you have the MySQL driver on your classpath, let me configure a MySQL-flavored DataSource")

**One-line interview answer:**
"`@ConditionalOnMissingBean` tells Spring to only create a bean if none of that type already exists in the context — it's how Spring Boot's auto-configuration provides sensible defaults for things like `DataSource` or `ObjectMapper`, while letting you completely override them just by defining your own bean of the same type, without needing to explicitly disable anything."