# @ConfigurationProperties

## Problem with @Value

Good for injecting a single property, but becomes problematic as configuration grows.

```java
@Component
public class DBConnection {
    @Value("${app.username}")
    private String username;

    @Value("${app.password}")
    private String password;

    @Value("${app.db.url}")
    private String dbUrl;
}
```

- Too much duplication of `@Value` annotation.Multiple times you need to add Value annotation.
- No way to validate, e.g.:
  - password must be present
  - password length `10 <= password <= 25`

## @ConfigurationProperties

- Maps configuration (from `application.properties`) into a Java object.
- Makes configuration structured, reusable, and validated.

`application.properties`:

```properties
user.name=test_username
user.age=27
user.active=true
```

binds to:

```java
@Component
@ConfigurationProperties(prefix = "user")
public class UserConfigurations {

    private String name;
    private int age;
    private boolean active;

    // getters and setters
}
```

We can simply inject `UserConfigurations` and access the properties just like Java objects:

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    UserConfigurations userConfigurations;

    @GetMapping("/")
    public void printUserConfig() {
        System.out.println("name: " + userConfigurations.getName());
        System.out.println("Age: " + userConfigurations.getAge());
        System.out.println("Is active: " + userConfigurations.isActive());
    }
}
```
Here first spring creates empty bean ,and then configurationBindings binds application.properties to or POJO using setter and getters.

## Handling Nested / Complex Configuration

```properties
user.name=test_username
user.age=27
user.active=true
user.address.city=myCityName
user.address.country=myCountryName
```

```java
@Component
@ConfigurationProperties(prefix = "user")
public class UserConfigurations {

    private String name;
    private int age;
    private boolean active;
    private AddressConfig address;

    public static class AddressConfig {
        private String city;
        private String country;
        // getters and setters
    }

    // getters and setters
}
```

**Why must the nested class be `static`?**

- Spring creates the `UserConfigurations` bean (because it's `@Component`).
- It sees a field `AddressConfig address`.
- The binder uses reflection and tries to instantiate `AddressConfig` using the **default no-arg constructor**.
- If the class is **non-static**, there is **no default no-arg constructor** — by default Java adds one constructor that takes the outer class as a parameter, like `InnerClass(OuterClass obj) { }`.
- Since the binder does not find a no-arg constructor, reflection fails → binding fails → the `address` field stays `null`.

That's why nested classes must be `static` for automatic binding.

## Configurations as List

```properties
user.name=test_username
user.age=27
user.active=true
user.address.city=myCityName
user.address.country=myCountryName

#List<String>
user.roles[0]=ADMIN
user.roles[1]=EDITOR

#List<Object>
user.courses[0].name=Java
user.courses[0].enrolled=true
user.courses[1].name=Springboot
user.courses[1].enrolled=false
```

```java
@Component
@ConfigurationProperties(prefix = "user")
public class UserConfigurations {

    private String name;
    private int age;
    private boolean active;
    private AddressConfig address;
    private List<String> roles;
    private List<Course> courses;

    public static class AddressConfig {
        private String city;
        private String country;
        // getter and setter
    }

    public static class Course {
        String name;
        boolean enrolled;
        // getter and setter
    }

    // getters and setters
}
```

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    UserConfigurations userConfigurations;

    @GetMapping("/")
    public void printUserConfig() {
        System.out.println("name: " + userConfigurations.getName());
        System.out.println("Age: " + userConfigurations.getAge());
        System.out.println("Is active: " + userConfigurations.isActive());
        System.out.println("Address: " + userConfigurations.getAddress().getCity() + ":"
                + userConfigurations.getAddress().getCountry());
        System.out.println("Roles: " + userConfigurations.getRoles().get(0) + ":" + userConfigurations.getRoles().get(1));
        System.out.println("Courses: " + userConfigurations.getCourses().get(0).getName() + ":"
                + userConfigurations.getCourses().get(1).getName());
    }
}
```

Output:

```
name: 27
Age: shrayanshjain
Is active: true
Address: myCityName:myCountryName
Roles: ADMIN:EDITOR
Courses: Java:Springboot
```

## Configurations as Map

Can be used as a nested class and Map too — key=value, or a Map of an object.

```properties
#Map<String, String> map prefeneces then key and after = we have value
user.preferences.theme=dark
user.preferences.language=en
user.preferences.timezone=IST
```

```java
@Component
@ConfigurationProperties(prefix = "user")
public class UserConfigurations {

    private String name;
    private int age;
    private boolean active;
    private AddressConfig address;
    private List<String> roles;
    private List<Course> courses;
    private Map<String, String> preferences;

    public static class AddressConfig {
        private String city;
        private String country;
        // getter and setter
    }

    public static class Course {
        String name;
        boolean enrolled;
        // getter and setter
    }

    // getters and setters
}
```

```java
System.out.println("Preferences: " + userConfigurations.getPreferences().get("theme"));
```

**Map of an Object** (`Map<String, AddressConfig>`):

```java
@Component
@ConfigurationProperties(prefix = "user")
public class UserConfigurations {

    private String name;
    private int age;
    private boolean active;
    private AddressConfig address;
    private List<String> roles;
    private List<Course> courses;
    private Map<String, String> preferences;
    private Map<String, AddressConfig> locations;

    public static class AddressConfig {
        private String city;
        private String country;
        // getter and setter
    }

    // getters and setters
}
```

```properties
# locations map then home object having city and country and then office object
user.locations.home.city=Delhi
user.locations.home.country=India
user.locations.office.city=Noida
user.locations.office.country=India
```

```java
System.out.println("Location: " + userConfigurations.getLocations().get("home").getCity());
```

## Validations

Add the dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

- Add `@Validated` on the `@ConfigurationProperties` class.
- If any validation fails, the application will fail to **startup**.
- Spring Boot automatically validates fields after binding, before the app fully starts.

```java
@Component
@ConfigurationProperties(prefix = "user")
@Validated
public class UserConfigurations {

    @NotBlank(message = "name must not be empty")
    private String name;

    @Min(value = 1, message = "Age can not be 0")
    private int age;

    // getters and setters
}
```

If validation fails (e.g. `user.age=0`), startup fails with a `ConfigurationPropertiesBindException` / binding validation error report.

### Validation Annotations

| Annotation | Description | Example |
|---|---|---|
| `@NotNull` | Value cannot be null | `@NotNull String name;` |
| `@NotBlank` | Not null or empty (for String) | `@NotBlank String username;` |
| `@NotEmpty` | Not null or empty (for List, Map, etc.) | `@NotEmpty List<String> courses;` |
| `@Min(value)` | Minimum value (numeric types) | `@Min(1) int age;` |
| `@Max(value)` | Maximum value | `@Max(100) int age;` |
| `@Positive` / `@PositiveOrZero` | Must be `> 0` / `>= 0` | `@Positive int retryCount;` |
| `@Negative` / `@NegativeOrZero` | Must be `< 0` / `<= 0` | `@Negative int loss;` |
| `@Email` | Must be valid email format | `@Email String email;` |
| `@Pattern(regexp)` | Must match regex | `@Pattern(regexp = "[A-Za-z]*") String userName;` |
| `@AssertTrue` / `@AssertFalse` | Must be true/false | `@AssertTrue boolean isActive;` |

## Constructor Binding (Immutability)

Automatic binding by default uses setter methods, so the class cannot be immutable. To make a `@ConfigurationProperties` class immutable, use **Constructor Binding**.

`@Component`  makes spring IoC to manage the bean so we do not want that ,we want `Configuration Binder` to do that.we tell by `@ConfigurationPropertiesScan`  , so we telling what ever id annotated with `@ConfigurationProperties` create its object and it creates object by constructor as it knows values from `application.properties`.

Earlier we were having no args constructor so IOC was working as no need of values and `Configuration Binder` was setting values by setters.

Key points:
- No `@Component` on the class — there's no default constructor, and Spring IoC wouldn't know what values to pass into the constructor parameters. 
- Bean creation responsibility is given to the **Configuration Binder** instead. It invokes the constructor with the proper property values, since there's no setter method in an immutable class.
- Earlier (with setters) it wasn't an issue because there's a default constructor — Spring IoC creates the empty bean, and the configuration binder later uses setters to update the fields.

```java
@ConfigurationProperties(prefix = "user")
public class ImmutableUserConfiguration {

    private final String name;
    private final int age;
    private final boolean active;

    ImmutableUserConfiguration(String name, int age, boolean active) {
        this.name = name;
        this.age = age;
        this.active = active;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public boolean isActive() {
        return active;
    }
}
```

Enable configuration properties scanning in the main application class:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class ConfigurationPropertiesApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigurationPropertiesApplication.class, args);
    }
}
```

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    ImmutableUserConfiguration immutableUserConfiguration;

    @GetMapping("/")
    public void printUserConfig() {
        // immutable
        System.out.println("immutable name: " + immutableUserConfiguration.getName());
        System.out.println("immutable Age: " + immutableUserConfiguration.getAge());
        System.out.println("immutable is active: " + immutableUserConfiguration.isActive());
    }
}
```

Output:

```
immutable name: shrayanshjain
immutable Age: 27
immutable is active: true
```

**Validation with constructor binding:** no difference — validation annotations work the same way on immutable (constructor-bound) classes.

```java
@ConfigurationProperties(prefix = "user")
@Validated
public class ImmutableUserConfiguration {

    private final String name;

    @Min(1)
    private final int age;

    private final boolean active;

    ImmutableUserConfiguration(String name, int age, boolean active) {
        this.name = name;
        this.age = age;
        this.active = active;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public boolean isActive() {
        return active;
    }
}
```
