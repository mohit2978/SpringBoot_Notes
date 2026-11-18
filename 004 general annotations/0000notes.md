# General Spring Boot Annotations

## Table of Contents

0. [Annotations in one minute](#0-annotations-in-one-minute)
1. [@SpringBootApplication](#1-springbootapplication)
2. [How a request reaches your controller method](#2-how-a-request-reaches-your-controller-method)
3. [@Controller](#3-controller)
4. [@RestController (and @Controller vs @RestController)](#4-restcontroller)
5. [@ResponseBody](#5-responsebody)
6. [@RequestMapping and the HTTP-method shortcuts](#6-requestmapping)
7. [@RequestParam (type conversion, @InitBinder, PropertyEditor)](#7-requestparam-type-conversion-initbinder--propertyeditor)
8. [@PathVariable (and @PathVariable vs @RequestParam)](#8-pathvariable-and-pathvariable-vs-requestparam)
9. [Other request annotations: @RequestHeader, @CookieValue, @ResponseStatus](#9-other-request-annotations)
10. [@RequestBody (JSON deserialization & @JsonProperty)](#10-requestbody-json-deserialization--jsonproperty)
11. [ResponseEntity (status, headers & body)](#11-responseentity-status-headers--body)
12. [@Qualifier & @Primary (bean disambiguation)](#12-qualifier--primary-bean-disambiguation)
13. [Jackson JSON exclusion annotations](#13-jackson-json-exclusion-annotations-jsonignore-jsonproperty-etc)
14. [Interview Questions](#14-interview-questions)

---

## 0. Annotations in one minute

An annotation (`@Something`) is **metadata** attached to a class, method, field or parameter. It does nothing by itself; Spring **reads it at startup or at request time (via reflection)** and acts on it:

- `@SpringBootApplication`, `@Component`, `@Service` → "create and manage this object as a bean"
- `@GetMapping("/users")` → "call this method for `GET /users`"
- `@RequestParam`, `@PathVariable`, `@RequestBody` → "fill this parameter from that part of the request"
- `@JsonIgnore`, `@JsonProperty` → "Jackson, treat this field specially when converting to/from JSON"

| Annotation | Where | What it does |
|---|---|---|
| `@SpringBootApplication` | Main class | Configuration + auto-configuration + component scan |
| `@Controller` / `@RestController` | Class | Handles HTTP requests (views / data) |
| `@RequestMapping`, `@GetMapping`, … | Class / method | Maps URL + HTTP method to a method |
| `@RequestParam` | Parameter | Query parameter `?name=value` |
| `@PathVariable` | Parameter | Part of the path `/users/{id}` |
| `@RequestBody` | Parameter | JSON body → Java object |
| `@ResponseBody` | Class / method | Return value → response body (JSON) |
| `ResponseEntity` | Return type | Full control of status + headers + body |
| `@Qualifier` / `@Primary` | Injection point / bean | Pick one bean when several match |
| `@JsonIgnore`, `@JsonProperty`, … | Field / class | Control JSON (de)serialization |

---

## 1. @SpringBootApplication

### Introduction
- `@SpringBootApplication` is the primary bootstrap annotation used in every Spring Boot application.
- It is placed directly on the main class (the class containing the entry point `public static void main(String[] args)`).
- It is a convenience meta-annotation combining three foundational Spring annotations:
  1. `@Configuration`
  2. `@EnableAutoConfiguration`
  3. `@ComponentScan`
- It triggers Spring Boot's auto-configuration engine, scans for components across the base package and its sub-packages, and allows developers to register additional beans.

![svg](<svgs/01-springbootapplication-equals-three.svg>)

> **Core formula:**
> ```
> @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
> ```

---

### Definition & meta-annotations
Inside the Spring Boot framework (`org.springframework.boot.autoconfigure`), `@SpringBootApplication` is declared as:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration          // a specialised @Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {

    @AliasFor(annotation = EnableAutoConfiguration.class)
    Class<?>[] exclude() default {};

    @AliasFor(annotation = EnableAutoConfiguration.class)
    String[] excludeName() default {};

    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};

    @AliasFor(annotation = Configuration.class, attribute = "proxyBeanMethods")
    boolean proxyBeanMethods() default true;
}
```

- `@Retention(RUNTIME)`: the annotation is kept in the `.class` file and is visible via reflection at runtime, which is how Spring reads it.
- `@AliasFor`: lets you set the attributes of the inner annotations (e.g. `exclude`) directly on `@SpringBootApplication`.

---

### What it does internally
When the application starts, Spring Boot performs these steps in order:
1. **Loads the configuration class:** reads the class annotated with `@SpringBootApplication` as the central bean configuration class.
2. **Scans packages & sub-packages:** scans the package of the main class and all its sub-packages for Spring components (`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, etc.).
3. **Enables auto-configuration:** inspects the runtime classpath, existing user-defined beans and application properties to configure beans automatically.
4. **Creates and refreshes the ApplicationContext:** instantiates the Spring IoC container and manages the bean lifecycle.
5. **Starts the embedded server:** boots the embedded servlet container (Tomcat by default; Jetty or Undertow optional).
6. **Prepares request handlers:** the application is ready to receive and process HTTP requests.

![svg](<svgs/02-spring-boot-startup-flow.svg>)

---

### The three core annotations

#### A) `@Configuration`
- Marks the class as a source of bean definitions for the Spring IoC container.
- The modern Java-based equivalent of legacy XML `<beans>` configuration.
- Methods inside the class can be annotated with `@Bean`, so Spring instantiates, configures and manages the objects they return.

```java
@Configuration
public class AppConfig {

    @Bean
    public RestClient restClient() {          // Spring calls this once and keeps the object as a singleton bean
        return RestClient.create("https://api.example.com");
    }
}
```

#### B) `@EnableAutoConfiguration`
- Enables Spring Boot's smart auto-configuration mechanism.
- Automatically configures beans based on:
  - **Libraries present on the classpath:** e.g. if `spring-webmvc` is present, it configures `DispatcherServlet`; if a MySQL driver is present, it prepares a `DataSource`.
  - **Beans already defined:** it never overrides custom beans explicitly defined by the developer; it **backs off**.
  - **Application properties:** reads `application.properties` / `application.yml`.
- Dramatically reduces boilerplate and removes manual XML/Java configuration.

![svg](<svgs/04-auto-configuration-conditions.svg>)

Each auto-configuration class is guarded by **conditions**:

```java
@AutoConfiguration
@ConditionalOnClass(ObjectMapper.class)            // only if Jackson is on the classpath
public class JacksonAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                      // only if YOU didn't define one
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        return builder.build();
    }
}
```

Run the app with `--debug` (or `debug=true` in properties) to print the **CONDITIONS EVALUATION REPORT**: which auto-configurations matched and why the others didn't.

#### C) `@ComponentScan`
- Tells Spring to scan the current package (where the annotated class is) and all its child packages.
- Automatically registers every class with a stereotype annotation (`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration`) as a bean.
- Equivalent to `<context:component-scan>` in traditional Spring XML configuration.

---

### Where should we put it?
- Always put `@SpringBootApplication` on the main application class in the **root package** (top-level package).
- All other packages (`controller`, `service`, `repository`, `entity`, `exception`, etc.) must be **sub-packages** of this root package.
- This way `@ComponentScan` discovers every component automatically, with no manual package paths.

![svg](<svgs/03-package-structure-component-scan.svg>)

**Example package hierarchy:**

```text
com.example.demo                     <-- Root Package
├── controller                       <-- Sub-package (Discovered automatically)
│   └── UserController.java
├── service                          <-- Sub-package
│   └── UserService.java
├── repository                       <-- Sub-package
│   └── UserRepository.java
├── entity                           <-- Sub-package
│   └── User.java
├── exception                        <-- Sub-package
│   └── GlobalExceptionHandler.java
└── DemoApplication.java            <-- Main Class with @SpringBootApplication
```

A class in `com.example.utils` (a *sibling*, not a child, of `com.example.demo`) is **not** scanned. Injecting it fails with `NoSuchBeanDefinitionException`.

---

### Code example

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        // Creates & starts the ApplicationContext and the embedded Tomcat server
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

**Explanation:**
- `@SpringBootApplication` boots the application by combining configuration, auto-configuration and component scanning.
- `SpringApplication.run(...)` launches the application, bootstraps Spring's `ApplicationContext`, and starts the embedded web server.

---

### Important attributes
1. **`exclude` / `excludeName`:**
   - Exclude specific auto-configuration classes you don't need.
   - Useful when you want to skip an auto-configured feature (e.g. the automatic database connection while the DB isn't ready yet).
2. **`scanBasePackages` / `scanBasePackageClasses`:**
   - Supply custom base packages for component scanning when some classes live outside the root package hierarchy.

```java
@SpringBootApplication(
    scanBasePackages = "com.example",                     // scan com.example.** (includes com.example.demo)
    exclude = { DataSourceAutoConfiguration.class }
)
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

> Setting `scanBasePackages` **replaces** the default (the main class's package). Include that package too, or your own controllers disappear.

---

### Summary table

| Annotation | Purpose | What it does | Equivalent (XML) |
|---|---|---|---|
| `@Configuration` | Marks class as a bean configuration source | Allows methods annotated with `@Bean` | `<beans>` |
| `@EnableAutoConfiguration` | Enables Spring Boot auto-configuration | Configures beans based on classpath, properties & existing beans | N/A (Spring Boot feature) |
| `@ComponentScan` | Scans components across packages | Finds and registers `@Component`, `@Service`, `@Repository`, `@Controller` | `<context:component-scan>` |
| `@SpringBootApplication` | Combination meta-annotation | Unifies `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` | N/A |

---

## 2. How a request reaches your controller method

Before looking at each web annotation, here's where each one plays its part:

![svg](<svgs/05-how-a-request-reaches-your-method.svg>)

1. The request arrives at **`DispatcherServlet`** (Spring MVC's front controller).
2. **`HandlerMapping`** finds the method from the table built at startup out of all `@RequestMapping` / `@GetMapping` … annotations in `@Controller` / `@RestController` beans.
3. **Argument resolvers** fill the method parameters: `@PathVariable` from the path, `@RequestParam` from the query string, `@RequestBody` from the JSON body (via Jackson), `@RequestHeader` from headers, …
4. Your method runs and returns a value.
5. **Return value handling:**
   - `@ResponseBody` (or `@RestController`, or `ResponseEntity`) → an `HttpMessageConverter` writes the value into the response body (JSON via Jackson, or plain text for a `String`).
   - Otherwise → the value is a **view name**, and a `ViewResolver` renders an HTML template.

---

## 3. @Controller

### Introduction
- `@Controller` is a core stereotype annotation in Spring MVC. It marks a class as a web controller responsible for handling incoming HTTP requests.
- In annotation-based Spring MVC, **request-mapping methods are only picked up from `@Controller` / `@RestController` beans**. (The other way to handle requests is functional `RouterFunction` routes.)
- It's meta-annotated with `@Component`, so it's auto-detected by component scanning.

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
    @AliasFor(annotation = Component.class)
    String value() default "";
}
```

---

### View-oriented behavior
- A traditional `@Controller` is designed for web applications that return **HTML views**, rendered with Thymeleaf, JSP, FreeMarker, etc.
- **Default behavior:** a `String` returned from a controller method is treated as a **view name**, resolved by a `ViewResolver`.
- If an `@Controller` method needs to return raw data (JSON/XML) instead of a view, annotate the method with `@ResponseBody`, or return a `ResponseEntity`.

```java
@Controller
@RequestMapping("/app")
public class HomeController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("msg", "Welcome to Spring MVC");
        return "home"; // Resolves to /templates/home.html via ViewResolver
    }
}
```

---

## 4. @RestController

### Introduction & definition
- `@RestController` is a specialized convenience annotation for building **RESTful web services and APIs**.
- Introduced in Spring 4.0, it combines `@Controller` and `@ResponseBody`:

> **Formula:**
> ```
> @RestController = @Controller + @ResponseBody
> ```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```

---

### Data-oriented behavior
- Because `@ResponseBody` applies at the class level, **every method in a `@RestController` writes its return value directly to the HTTP response body**.
- It never looks for a view.
- Returned objects (DTOs, Maps, Lists) are serialized into JSON (or XML) by the registered `HttpMessageConverter`s, primarily Jackson's `ObjectMapper`.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // Automatically converted to JSON in the HTTP response body
        return new User(id, "John", "john@example.com");
    }
}
```

---

### @Controller vs @RestController

![svg](<svgs/06-controller-vs-restcontroller.svg>)

| Feature | `@Controller` | `@RestController` |
|---|---|---|
| **Primary purpose** | Traditional web apps (server-side rendered views) | RESTful web APIs & microservices (raw data) |
| **Return value** | View name (`String`) resolved to an HTML page | Data object (`User`, `List<T>`, `Map`, etc.) |
| **Response handling** | Resolved by `ViewResolver` | Written directly to the HTTP response body |
| **Needs `@ResponseBody`?** | Yes (on each method that returns data) | No (inherited at class level) |
| **Default behavior** | Forward to a view template | Serialize the payload to JSON/XML via Jackson |
| **Typical use cases** | Thymeleaf, JSP, legacy MVC web pages | Microservices, SPAs (React, Angular), mobile apps |
| **Code example** | `return "home";` (renders `home.html`) | `return new User(...);` (returns JSON `{ "id": 1 }`) |

---

### Same endpoint, both styles

**1. Using `@Controller` (returns a view):**

```java
@Controller
public class ProductController {

    @GetMapping("/products")
    public String getProducts(Model model) {
        List<Product> list = productService.getAll();
        model.addAttribute("products", list);
        return "products"; // Resolves to /templates/products.html
    }
}
// Response: renders the products.html template and sends HTML to the client
```

**2. Using `@RestController` (returns data):**

```java
@RestController
public class ProductController {

    @GetMapping("/products")
    public List<Product> getProducts() {
        return productService.getAll(); // Serialized directly to JSON
    }
}
// Response: JSON array of products
// [ {"id": 1, "name": "Laptop", "price": 50000}, {"id": 2, "name": "Phone", "price": 20000} ]
```

> **Bottom line:**
> Both handle incoming HTTP requests.
> - `@Controller` is for **views** (HTML).
> - `@RestController` is for **data** (JSON/XML).

---

## 5. @ResponseBody

### Introduction & core rule
- `@ResponseBody` tells Spring that the value returned by a controller method should be serialized directly into the **HTTP response body**, instead of being treated as a view name.
- It skips the `ViewResolver` entirely and uses the registered `HttpMessageConverter`s (e.g. Jackson for JSON).

> **The fundamental rule:**
> - **Without `@ResponseBody`:** the return value is a **view name** (Spring tries to resolve an HTML template).
> - **With `@ResponseBody`:** the return value is **data / the HTTP response body** (Spring serializes the object to JSON/XML).

```java
@Controller
public class LegacyApiController {

    // Because @ResponseBody is explicitly added, this returns JSON data, NOT a view name!
    @ResponseBody
    @GetMapping("/api/status")
    public Map<String, String> getStatus() {
        return Collections.singletonMap("status", "UP");
    }
}
```

Which converter is used depends on the return type and the request's `Accept` header: a `String` is written as `text/plain` by `StringHttpMessageConverter`; objects, maps and lists are written as JSON by Jackson.

---

## 6. @RequestMapping

### Introduction
- `@RequestMapping` is the core routing annotation in Spring MVC. It maps web requests onto controller classes and handler methods.
- It can be applied at two levels:
  1. **Class level:** sets the shared base path for all endpoints in that controller.
  2. **Method level:** defines the specific endpoint path and HTTP method.
  - When used at both levels, the paths are **concatenated**:
    - Class: `@RequestMapping("/api")`
    - Method: `@RequestMapping("/fetchUser")`
    - Full endpoint: `/api/fetchUser`

---

### Definition & meta-annotations

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
@Reflective(ControllerMappingReflectiveProcessor.class)
public @interface RequestMapping {

    String name() default "";

    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};

    RequestMethod[] method() default {};

    String[] params() default {};

    String[] headers() default {};

    String[] consumes() default {};

    String[] produces() default {};
}
```

---

### Key attributes explained
1. **`path` / `value`:** aliases for each other (`@AliasFor`). You can write `path = "/fetchUser"`, `value = "/fetchUser"`, or simply `"/fetchUser"`. Both accept arrays: `{"/a", "/b"}`.
2. **`method`:** the HTTP method (`RequestMethod.GET`, `POST`, `PUT`, `PATCH`, `DELETE`, …). If omitted, the mapping matches **all** methods.
3. **`consumes`:** narrows the mapping by the request's `Content-Type` header (e.g. `consumes = "application/json"`). Mismatch → **415 Unsupported Media Type**.
4. **`produces`:** narrows the mapping by the request's `Accept` header, i.e. what the endpoint returns (e.g. `produces = "application/json"`). Mismatch → **406 Not Acceptable**.
5. **`headers`:** only matches requests with specific headers (e.g. `headers = "X-API-VERSION=1"`).
6. **`params`:** only matches requests with specific query parameters (e.g. `params = "action=create"`).

```java
@RequestMapping(
    path = "/fetchUser",
    method = RequestMethod.GET,
    consumes = "application/json",
    produces = "application/json"
)
```

---

### Internal working & reflection
- **`@Mapping`:** meta-annotation marking this as a web-mapping annotation.
- **Startup registration:** while the application context starts, `RequestMappingHandlerMapping` inspects every `@Controller` / `@RestController` bean, reads its mapping annotations **via Java reflection**, and builds the URL → method table that `DispatcherServlet` uses for every request. That's why mappings must live in controller beans.
- **`@Reflective(ControllerMappingReflectiveProcessor.class)`:** registers reflection hints for these classes during **AOT processing** (e.g. GraalVM native images, where reflection must be declared ahead of time).

---

### HTTP-method shortcut annotations
Spring provides composed annotations that wrap `@RequestMapping(method = ...)`:
- `@GetMapping(path = "...")` → `@RequestMapping(method = RequestMethod.GET)`
- `@PostMapping(path = "...")` → `@RequestMapping(method = RequestMethod.POST)`
- `@PutMapping(path = "...")` → `@RequestMapping(method = RequestMethod.PUT)`
- `@DeleteMapping(path = "...")` → `@RequestMapping(method = RequestMethod.DELETE)`
- `@PatchMapping(path = "...")` → `@RequestMapping(method = RequestMethod.PATCH)`

Typical style: `@RequestMapping("/api/users")` on the class, shortcut annotations on the methods. Calling an existing path with the wrong method gives **405 Method Not Allowed**.

---

## 7. @RequestParam (Type Conversion, @InitBinder & PropertyEditor)

### Introduction
- `@RequestParam` extracts **query parameters** from the request URL and binds them to controller method parameters.

![svg](<svgs/07-url-anatomy-pathvariable-requestparam.svg>)

**URL breakdown:**

```text
http://localhost:8080/api/fetchUser?firstName=SHRAYANSH&lastName=JAIN&age=32

```

- **Host:** protocol, domain and port (`http://localhost:8080`).
- **Path:** resource route (`/api/fetchUser`).
- **Query delimiter:** `?` starts the query parameters; they're separated by `&`.
- **Request parameters:** key-value pairs (`firstName=SHRAYANSH`, `lastName=JAIN`, `age=32`).

---

### Code example

```java
@RestController
@RequestMapping(value = "/api")
public class SampleController {

    @GetMapping(path = "/fetchUser")
    public String getUserDetails(
            @RequestParam(name = "firstName") String firstName,
            @RequestParam(name = "lastName", required = false) String lastName,
            @RequestParam(name = "age") int age) {

        return "fetching and returning user details based on first name = " + firstName +
               ", lastName = " + lastName + " and age is = " + age;
    }
}
```

`GET /api/fetchUser?firstName=SHRAYANSH&age=32` →

```text
fetching and returning user details based on first name = SHRAYANSH, lastName = null and age is = 32
```

### Attributes & rules
- **`name` / `value`:** name of the query parameter in the URL. Can be omitted if it matches the Java parameter name (requires compiling with `-parameters`, which the Spring Boot parent does).
- **`required`:** whether the parameter is mandatory.
  - **By default, `required = true`.**
  - If a mandatory parameter is missing, Spring returns **400 Bad Request** (`MissingServletRequestParameterException`).
  - Set `required = false` for optional parameters (the value is then `null`, so use a wrapper type like `Integer`, not `int`), or use `Optional<String>`.
- **`defaultValue`:** a fallback value if the parameter is missing (this also makes it optional).
- **Multiple values:** `?tag=java&tag=spring` → `@RequestParam List<String> tag`.
- **All params:** `@RequestParam Map<String, String> params`.

---

### Automatic type conversion
Query values always arrive as **text**. Spring converts them to the declared Java type:

![svg](<svgs/08-type-conversion-initbinder.svg>)

1. **Primitive types:** `int`, `long`, `float`, `double`, `boolean`, … (e.g. `?age=28` → `int age = 28`).
2. **Wrapper classes:** `Integer`, `Long`, `Float`, `Double`, `Boolean`, …
3. **String:** used as-is.
4. **Enums:** matching names bind to enum constants (`?status=ACTIVE` → `Status.ACTIVE`).
5. **Dates:** with `@DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from`.
6. **Custom types:** by registering a custom `PropertyEditor`, `Converter` or `Formatter`.

If conversion fails (`?age=abc` for an `int`), Spring returns **400 Bad Request** (`MethodArgumentTypeMismatchException`).

---

### Custom type conversion with @InitBinder & PropertyEditorSupport
When we need custom logic before a request value is assigned to a parameter (trimming whitespace, converting to lower case, parsing a custom date format like `23/9/1987`), we can register a custom `PropertyEditor`.

#### 1. Create a custom PropertyEditor
Extend `PropertyEditorSupport` and override `setAsText(String text)`:

```java
import java.beans.PropertyEditorSupport;

public class FirstNamePropertyEditor extends PropertyEditorSupport {

    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        // Custom logic: trim whitespace and convert to lower case
        setValue(text == null ? null : text.trim().toLowerCase());
    }
}
```

#### 2. Register it via `@InitBinder` in the controller

```java
@RestController
@RequestMapping(value = "/api")
public class SampleController {

    @InitBinder("firstName")                       // only for the binder of the "firstName" request param
    protected void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(String.class, new FirstNamePropertyEditor());
    }

    @GetMapping(path = "/fetchUser")
    public String getUserDetails(
            @RequestParam(name = "firstName") String firstName,
            @RequestParam(name = "lastName", required = false) String lastName,
            @RequestParam(name = "age") int age) {

        return "fetching and returning user details based on first name = " + firstName +
               ", lastName = " + lastName + " and age is = " + age;
    }
}
```

**How it works:**
- An `@InitBinder` method configures the `WebDataBinder` that Spring creates to convert request values in **this controller**.
- `@InitBinder("firstName")` limits it to the binder for the `firstName` parameter. Without a name, it applies to every parameter/model attribute of the controller.
- `binder.registerCustomEditor(String.class, editor)` says "use this editor for every `String` this binder converts".
- When a request with `?firstName=  JOHN ` arrives, Spring runs `FirstNamePropertyEditor.setAsText()`, which trims and lowercases the value, and `"john"` is passed into the controller method.

> **Correction to a common version of this example:** the three-argument form `binder.registerCustomEditor(String.class, "firstName", editor)` registers an editor for a **property path**, i.e. a field called `firstName` inside a bound object (`@ModelAttribute User user` with a `firstName` property). A plain `@RequestParam String firstName` isn't converted through a property path, so that form doesn't apply to it. Use `@InitBinder("firstName")` + the two-argument form instead, as shown above.

**Modern alternative:** for app-wide conversions, register a `Converter` / `Formatter` once instead of per-controller editors:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(String.class, Money.class, Money::parse);   // ?price=10.50 → Money
    }
}
```

Spring also ships ready-made editors, e.g. `binder.registerCustomEditor(String.class, new StringTrimmerEditor(true))` trims every String and turns empty strings into `null`.

---

## 8. @PathVariable (and @PathVariable vs @RequestParam)

### Introduction
- `@PathVariable` extracts values directly from the URI path template (e.g. `/api/fetchUser/{firstName}`).
- Use it when the value is part of the path itself and identifies a specific resource.

```java
@RestController
@RequestMapping(value = "/api")
public class SampleController {

    @GetMapping(path = "/fetchUser/{firstName}")
    public String getUserDetails(@PathVariable(value = "firstName") String firstName) {
        return "fetching and returning user details based on first name = " + firstName;
    }
}
```

`GET /api/fetchUser/SHRAYANSH` → `fetching and returning user details based on first name = SHRAYANSH`

You can have several: `@GetMapping("/users/{userId}/orders/{orderId}")` with `@PathVariable Long userId, @PathVariable Long orderId`.

---

### Comparison: @PathVariable vs @RequestParam

| Feature | `@PathVariable` | `@RequestParam` |
|---|---|---|
| **Where the value comes from** | URI path template | Query string (after `?`) |
| **Example URL** | `GET /users/101` | `GET /users?id=101` |
| **Primary use case** | Identifying a specific resource (`id`, `orderId`) | Filtering, searching, sorting, pagination, optional data |
| **Required by default** | Yes (part of the path) | Yes by default (`required = false` can be configured) |
| **Can be optional?** | Practically no (omitting it changes the path, giving a different endpoint or 404) | Yes (`required = false` or `defaultValue = "..."`) |
| **RESTful style** | Very RESTful (clean hierarchical URI) | Used for options on top of the resource |

---

### Combined usage example
A common RESTful pattern combines both: `@PathVariable` identifies the resource, `@RequestParam` handles optional filters.

```java
@RestController
@RequestMapping("/users")
public class UserController {

    // Example Request: GET /users/101?sort=name&page=2
    @GetMapping("/{userId}")
    public String getUser(
            @PathVariable Long userId,
            @RequestParam(required = false) String sort,
            @RequestParam(defaultValue = "0") int page) {

        return "userId=" + userId + ", sort=" + sort + ", page=" + page;
    }
}
```

`GET /users/101?sort=name&page=2` → `userId=101, sort=name, page=2`
`GET /users/101` → `userId=101, sort=null, page=0`

> **Rule of thumb:**
> - Use `@PathVariable` for **identifying** a specific resource.
> - Use `@RequestParam` for **filtering, searching, or optional** extras about that resource.

---

## 9. Other request annotations

| Annotation | Reads from | Example |
|---|---|---|
| `@RequestHeader` | HTTP header | `@RequestHeader("X-Request-Id") String requestId` |
| `@CookieValue` | Cookie | `@CookieValue("SESSION") String session` |
| `@ModelAttribute` | Form fields / query params → object | `@ModelAttribute UserFilter filter` (binds `?city=..&age=..`) |
| `@RequestPart` | Multipart file upload | `@RequestPart("file") MultipartFile file` |
| `@ResponseStatus` | — (sets the response status) | `@ResponseStatus(HttpStatus.CREATED)` on a POST method |

```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)                       // 201 instead of 200
public User create(@RequestBody @Valid User user,
                   @RequestHeader(value = "X-Request-Id", required = false) String requestId) {
    return userService.save(user);
}
```

---

## 10. @RequestBody (JSON Deserialization & @JsonProperty)

### Introduction
- `@RequestBody` binds the body of the incoming HTTP request (typically JSON) directly to a Java object.
- Spring uses an `HttpMessageConverter`, by default Jackson's `MappingJackson2HttpMessageConverter`, to deserialize the JSON text into the target Java object.

![svg](<svgs/09-requestbody-json-to-object.svg>)

---

### Code example

**Controller:**

```java
@RestController
@RequestMapping(value = "/api")
public class SampleController {

    @PostMapping(path = "/saveUser")
    public String saveUser(@RequestBody User user) {
        return "User created " + user.getUsername() + ":" + user.getEmail();
    }
}
```

**Java domain class:**

```java
public class User {

    @JsonProperty("user_name")
    private String username;

    private String email;

    public String getUsername() {
        return username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
    public String getEmail() {
        return email;
    }
    public void setEmail(String email) {
        this.email = email;
    }
}
```

**Sample cURL request:**

```bash
curl --location --request POST 'http://localhost:8080/api/saveUser' \
--header 'Content-Type: application/json' \
--data-raw '{
    "user_name": "Shrayansh",
    "email": "sjxyztest@gmail.com"
}'
```

**Response:**

```text
User created Shrayansh:sjxyztest@gmail.com
```

---

### Key mechanics & @JsonProperty
- **Direct mapping:** the JSON payload of the POST/PUT request is mapped into the Java class fields.
- **`@JsonProperty("user_name")`:** maps the JSON key `"user_name"` to the Java property `username`.
- **Exact name match:** if the JSON key matches the Java field name (`"email"` ↔ `email`), `@JsonProperty` isn't needed.
- **Under the hood:** Jackson's `ObjectMapper` creates the object (no-arg constructor + setters, or a record / `@JsonCreator` constructor) and fills the fields.
- **Unknown JSON fields** (keys with no matching field) are ignored by Spring Boot's default `ObjectMapper` (`FAIL_ON_UNKNOWN_PROPERTIES` is disabled).
- Only **one** `@RequestBody` per method: the body stream can be read only once.

**Validation:** add `@Valid` plus constraints on the class:

```java
public class User {
    @NotBlank @JsonProperty("user_name") private String username;
    @Email    private String email;
}

@PostMapping("/saveUser")
public String saveUser(@RequestBody @Valid User user) { ... }   // invalid → 400 (MethodArgumentNotValidException)
```

(Needs the `spring-boot-starter-validation` dependency.)

---

## 11. ResponseEntity (Status, Headers & Body)

### Introduction
- `ResponseEntity` represents the **entire HTTP response**:
  1. **HTTP status code** (e.g. `200 OK`, `201 CREATED`, `400 BAD_REQUEST`, `404 NOT_FOUND`)
  2. **HTTP headers** (e.g. `Content-Type`, `Cache-Control`, `Location`, custom headers)
  3. **Response body** (payload serialized into JSON, XML or text)

![svg](<svgs/10-responseentity-structure.svg>)

---

### Code example

```java
@RestController
@RequestMapping(value = "/api")
public class SampleController {

    @GetMapping(path = "/fetchUser")
    public ResponseEntity<String> getUserDetails(@RequestParam(value = "firstName") String firstName) {
        String output = "fetched User details of " + firstName;

        return ResponseEntity
                .status(HttpStatus.OK)
                .header("Custom-Header", "DemoValue")
                .body(output);
    }
}
```

More realistic uses:

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return userService.findById(id)
            .map(ResponseEntity::ok)                          // 200 + body
            .orElse(ResponseEntity.notFound().build());       // 404, no body
}

@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody User user) {
    User saved = userService.save(user);
    return ResponseEntity.created(URI.create("/users/" + saved.getId()))   // 201 + Location header
            .body(saved);
}
```

---

### ResponseEntity vs @ResponseBody
- **`ResponseEntity` is all 3:** full programmatic control over status code, headers and body.
- **`@ResponseBody` is only the body:** with `@ResponseBody` / `@RestController`, Spring serializes the returned object into the body and uses **200 OK** by default (change it with `@ResponseStatus` or by throwing exceptions handled by `@ExceptionHandler`).
- **Returning a `String`:**
  - In a **`@RestController`**, Spring writes the String into the body as `text/plain` with status `200 OK` (via `StringHttpMessageConverter`).
  - In a **`@Controller`**, a returned `String` is interpreted as a **view name**!

---

## 12. @Qualifier & @Primary (Bean Disambiguation)

### Introduction
- When several beans of the same type exist in the Spring context, injecting that type is ambiguous, and Spring throws `NoUniqueBeanDefinitionException`.
- `@Qualifier` and `@Primary` both resolve the ambiguity, in different ways.

![svg](<svgs/11-qualifier-vs-primary-resolution.svg>)

**Resolution order** when several beans match: `@Qualifier` at the injection point → the single `@Primary` bean → a bean whose **name matches the field/parameter name** → otherwise `NoUniqueBeanDefinitionException`.

---

### 1. @Qualifier (bean level + injection point level)
- **Where it's used:** mainly at the **injection point** (constructor argument, field, setter), but it can also be put on the bean class/method to give it a qualifier.
- **Purpose:** tells Spring **exactly** which bean (by name or custom qualifier) to inject.
- Gives fine-grained, explicit control.

```java
// 1. Define multiple beans of the same type
@Component("mySqlDataSource")
public class MySqlDataSource implements DataSource { }

@Component("postgresDataSource")
public class PostgreDataSource implements DataSource { }

// 2. Explicitly inject using @Qualifier
@Service
public class ReportService {

    private final DataSource dataSource;

    @Autowired
    public ReportService(@Qualifier("mySqlDataSource") DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

(The default bean name for `@Component` is the class name with a lower-case first letter, e.g. `mySqlDataSource` for `MySqlDataSource`.)

#### Custom qualifier annotations
Instead of hard-coding bean names as strings, define your own qualifier annotation:

```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface FastDB { }

// Apply to bean
@Component
@FastDB
public class MySqlDataSource implements DataSource { }

// Inject via custom qualifier
@Service
public class ReportService {

    private final DataSource dataSource;

    @Autowired
    public ReportService(@FastDB DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

---

### 2. @Primary (bean level)
- **Where it's used:** on the **bean definition** (`@Component` class or `@Bean` method).
- **Purpose:** marks a bean as the **default** candidate when several beans match and no qualifier is specified at the injection point.
- Only one bean of a given type should be `@Primary`. Two primaries → `NoUniqueBeanDefinitionException` ("more than one 'primary' bean found").

```java
@Component
@Primary // Default bean
public class MySqlDataSource implements DataSource { }

@Component
public class PostgreDataSource implements DataSource { }

@Service
public class ReportService {

    private final DataSource dataSource;

    @Autowired
    public ReportService(DataSource dataSource) {
        // Spring injects MySqlDataSource because it is marked @Primary
        this.dataSource = dataSource;
    }
}
```

Need **all** of them? Inject a collection: `List<DataSource> dataSources` or `Map<String, DataSource>` (bean name → bean).

---

### Quick comparison table

| Feature | `@Qualifier` | `@Primary` |
|---|---|---|
| **Purpose** | Choose a specific bean explicitly | Designate a default bean |
| **Where used** | At the injection point (constructor, parameter, field) | At the bean definition (`@Component`, `@Bean`) |
| **Syntax** | `@Qualifier("beanName")` or a custom annotation | `@Primary` |
| **When to use** | When you need exact control | When one bean is the preferred default |
| **Multiple allowed?** | Yes (different qualifiers for different beans) | No (only one `@Primary` per type) |
| **Precedence (both used)** | `@Qualifier` **wins** | Overridden by `@Qualifier` |

> **Precedence rule:**
> If both are present, **`@Qualifier` always wins!**
> `@Primary` is only considered when no `@Qualifier` is specified at the injection point.
>
> - Use **`@Qualifier`** when you want **exact control**.
> - Use **`@Primary`** when you want a **default**.

---

## 13. Jackson JSON Exclusion Annotations (@JsonIgnore, @JsonProperty, etc.)

### Introduction
- Spring Boot uses the **Jackson** library by default to convert Java objects to JSON (**serialization**) and JSON to Java objects (**deserialization**).
- When exposing REST APIs, some fields must be excluded or restricted (passwords, secrets, internal metadata, audit timestamps).
- Jackson annotations control which fields are included or excluded in each direction.

![svg](<svgs/12-jackson-field-visibility.svg>)

---

### 1. `@JsonIgnore`: complete field exclusion
- Placed on a field, getter or setter.
- The field is **completely ignored in both directions**:
  - never serialized to the JSON response;
  - never deserialized from the incoming JSON request.

```java
import com.fasterxml.jackson.annotation.JsonIgnore;

public class User {

    private Long id;
    private String username;

    @JsonIgnore
    private String password; // Completely excluded from JSON

    // Getters and Setters
}
```

**Serialized output JSON:**

```json
{
  "id": 1,
  "username": "john_doe"
}
```

---

### 2. `@JsonIgnoreProperties`: class-level batch exclusion
- Applied at the class level to exclude several fields at once.
- `ignoreUnknown = true` ignores unknown JSON fields (useful when the class is used outside Spring Boot's pre-configured `ObjectMapper`).
- Can also ignore fields inherited from a superclass.

```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties({"password", "internalNotes"})
public class User {

    private Long id;
    private String username;
    private String password;
    private String internalNotes;

    // Getters and Setters
}
```

---

### 3. `@JsonProperty(access = Access.WRITE_ONLY)`: best for passwords
- **Input (deserialization):** accepted from the client's JSON request.
- **Output (serialization):** hidden from the JSON response.
- **Perfect for passwords, API keys and sensitive credentials.**

```java
import com.fasterxml.jackson.annotation.JsonProperty;

public class User {

    private Long id;
    private String username;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String password;

    // Getters and Setters
}
```

---

### 4. `@JsonProperty(access = Access.READ_ONLY)`: best for server-generated data
- **Output (serialization):** shown in the JSON response.
- **Input (deserialization):** ignored if the client sends it in the request body.
- **Perfect for IDs, creation timestamps and calculated fields.**

```java
public class User {

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private Long id;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private LocalDateTime createdAt;
}
```

---

### 5. `@JsonIgnoreType`: ignore an entire type
- Applied on a class definition.
- Any field of this type is ignored automatically, in every class that references it.

```java
import com.fasterxml.jackson.annotation.JsonIgnoreType;

@JsonIgnoreType
public class InternalMetadata {
    private String ipAddress;
    private String deviceInfo;
    private Long requestId;
}

public class User {
    private Long id;
    private String username;
    private InternalMetadata meta; // Whole object ignored automatically
}
```

---

### Quick comparison table

| Annotation | Serialize (Java → JSON) | Deserialize (JSON → Java) | Primary use case |
|---|---|---|---|
| `@JsonIgnore` | ❌ Excluded | ❌ Ignored | Never include in either direction |
| `@JsonProperty(access = WRITE_ONLY)` | ❌ Hidden | ✔ Accepted | Passwords, sensitive credentials |
| `@JsonProperty(access = READ_ONLY)` | ✔ Shown | ❌ Ignored | Server-generated IDs, timestamps |
| `@JsonIgnoreProperties({"a", "b"})` | ❌ Hidden | ❌ Ignored | Batch exclusion at class level |
| `@JsonIgnoreType` | ❌ Hidden type | ❌ Ignored type | Exclude an entire type everywhere |

Other handy ones: `@JsonInclude(JsonInclude.Include.NON_NULL)` (skip null fields in the output), `@JsonFormat(pattern = "dd-MM-yyyy")` (date format).

---

### Spring Boot controller integration example
Jackson annotations are applied automatically by Spring Boot's `HttpMessageConverter` when reading requests and writing responses. No extra controller code is needed.

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
        // Sensitive fields like password are automatically excluded by Jackson
    }
}
```

> In real projects, many teams return a separate **DTO** (e.g. `UserResponse` without a password field) instead of the entity. Then nothing sensitive can leak even if someone forgets an annotation.

---

### All-in-one example

**Entity definition:**

```java
@JsonIgnoreProperties({"internalNotes"})
public class User {

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private Long id;

    private String username;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String password;

    private String email;

    // Getters and Setters
}
```

**Incoming request (HTTP POST `/users`):**

```json
{
  "id": 999,
  "username": "john_doe",
  "password": "mypassword",
  "email": "john@gmail.com",
  "internalNotes": "for admin only"
}
```

*Note: `id` is ignored on input (READ_ONLY); `internalNotes` is ignored on input (`@JsonIgnoreProperties`).*

**Outgoing response (HTTP GET `/users/1`):**

```json
{
  "id": 1,
  "username": "john_doe",
  "email": "john@gmail.com"
}
```

*Note: `password` is hidden in output (WRITE_ONLY); `internalNotes` is hidden in output (`@JsonIgnoreProperties`).*

---

### Summary takeaways
- Use **`@JsonIgnore`** for internal fields that should never be serialized or deserialized.
- Use **`@JsonProperty(access = WRITE_ONLY)`** for sensitive input-only data (like passwords).
- Use **`@JsonProperty(access = READ_ONLY)`** for server-generated output-only data (like IDs and timestamps).
- Use **`@JsonIgnoreProperties`** or **`@JsonIgnoreType`** for class-level or type-wide exclusion.

---

## 14. Interview Questions

**Q1. What does `@SpringBootApplication` do?**
It combines `@SpringBootConfiguration` (a `@Configuration`), `@EnableAutoConfiguration` and `@ComponentScan`. The main class becomes a configuration class, Spring Boot auto-configures beans based on the classpath and properties, and components in the main class's package and its sub-packages are registered.

**Q2. How does auto-configuration work?**
Spring Boot ships auto-configuration classes (listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`). Each is guarded by conditions such as `@ConditionalOnClass`, `@ConditionalOnProperty` and `@ConditionalOnMissingBean`, so it only applies when the required library is present and you haven't defined your own bean. `--debug` prints the conditions report.

**Q3. How do you disable a specific auto-configuration?**
`@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` or `spring.autoconfigure.exclude=...` in properties.

**Q4. Why should the main class be in the root package?**
`@ComponentScan` scans the main class's package and its sub-packages only. Components in sibling or parent packages aren't found (unless you set `scanBasePackages`).

**Q5. `@Controller` vs `@RestController`?**
`@Controller` returns view names resolved to HTML templates. `@RestController` = `@Controller` + `@ResponseBody`, so every method's return value is written to the response body (JSON).

**Q6. What does `@ResponseBody` do?**
It tells Spring to write the method's return value into the HTTP response body using an `HttpMessageConverter`, instead of treating it as a view name.

**Q7. `@RequestParam` vs `@PathVariable`?**
`@PathVariable` binds part of the URL path (`/users/{id}`) and identifies a resource. `@RequestParam` binds query parameters (`?page=2`) for filtering, sorting and paging, and can be optional via `required = false` / `defaultValue`.

**Q8. What happens if a required `@RequestParam` is missing, or has the wrong type?**
Missing → 400 (`MissingServletRequestParameterException`). Unconvertible (e.g. `abc` for `int`) → 400 (`MethodArgumentTypeMismatchException`).

**Q9. What is `@InitBinder` used for?**
To customize the `WebDataBinder` for a controller: register `PropertyEditor`s or formatters (trim strings, parse custom dates), or restrict allowed fields. `@InitBinder("name")` limits it to one parameter or model attribute. App-wide conversions are better done with a `Converter`/`Formatter` in `WebMvcConfigurer.addFormatters`.

**Q10. How does `@RequestBody` work? What errors can it produce?**
Jackson (`MappingJackson2HttpMessageConverter`) deserializes the JSON body into the parameter type. Malformed JSON → 400 (`HttpMessageNotReadableException`). Unsupported `Content-Type` → 415. Failed `@Valid` constraints → 400 (`MethodArgumentNotValidException`).

**Q11. What does `@JsonProperty("user_name")` do?**
It maps the JSON key `user_name` to the Java field `username`, in both directions.

**Q12. `ResponseEntity` vs `@ResponseBody`?**
`ResponseEntity` controls the status, headers and body. `@ResponseBody` controls only the body, with status 200 by default (change it with `@ResponseStatus`).

**Q13. What do `consumes` and `produces` do in `@RequestMapping`?**
`consumes` restricts the request `Content-Type` (mismatch → 415). `produces` declares the response media type matched against the `Accept` header (mismatch → 406).

**Q14. `@Qualifier` vs `@Primary`? Which wins?**
`@Primary` marks the default bean on the bean definition. `@Qualifier` picks an exact bean at the injection point. `@Qualifier` wins. Without either, Spring falls back to matching the field/parameter name with a bean name, and otherwise throws `NoUniqueBeanDefinitionException`.

**Q15. How do you hide a password in API responses but still accept it in requests?**
`@JsonProperty(access = JsonProperty.Access.WRITE_ONLY)`, or better, return a response DTO that doesn't have a password field at all.

**Q16. `@JsonIgnore` vs `@JsonIgnoreProperties` vs `@JsonIgnoreType`?**
`@JsonIgnore` works on one field. `@JsonIgnoreProperties` lists fields at class level (and can ignore unknown properties). `@JsonIgnoreType` ignores every field of a given type wherever it appears.

**Q17. How are request-mapping annotations discovered?**
At startup, `RequestMappingHandlerMapping` inspects all `@Controller` beans with reflection and registers each `@RequestMapping`-annotated method in a lookup table. `DispatcherServlet` consults that table for every request.
