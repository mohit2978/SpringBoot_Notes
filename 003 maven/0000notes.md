# Maven Notes

> **Maven is a Project Management Tool!!**

## Table of Contents

1. [What is Maven?](#1-what-is-maven)
2. [Why Maven, when we can run Java with commands?](#2-why-maven-when-we-can-run-java-with-commands)
3. [What is the use of Maven?](#3-what-is-the-use-of-maven)
4. [Installing Maven and the Maven Wrapper](#4-installing-maven-and-the-maven-wrapper)
5. [Maven project structure](#5-maven-project-structure)
6. [pom.xml: Project Object Model](#6-pomxml-project-object-model)
7. [Dependencies: scopes, transitive deps and conflicts](#7-dependencies-scopes-transitive-deps-and-conflicts)
8. [Maven repositories](#8-maven-repositories)
9. [Maven build lifecycle](#9-maven-build-lifecycle)
10. [Lifecycle phases in detail](#10-lifecycle-phases-in-detail)
11. [Lifecycle reference and other lifecycles](#11-lifecycle-reference-and-other-lifecycles)
12. [Common commands and options](#12-common-commands-and-options)
13. [Spring Boot + Maven](#13-spring-boot--maven)
14. [How to run a Java / Spring Boot app on a Linux server](#14-how-to-run-a-java--spring-boot-app-on-a-linux-server)
15. [Profiles and multi-module projects](#15-profiles-and-multi-module-projects)
16. [Key takeaways](#16-key-takeaways)
17. [Interview Questions](#17-interview-questions)
18. [Server / system questions (running in production)](#18-server--system-questions-running-in-production)

---

## 1. What is Maven?

- It's a **project management and build tool**. It helps developers with:
  - Build generation (compile → test → package)
  - Dependency resolution (downloading the libraries your code needs)
  - Documentation, reports, etc.

- Maven uses a **POM** (Project Object Model), the `pom.xml` file, to achieve this.

- When a `mvn` command is given, Maven looks for `pom.xml` in the **current directory** and reads the configuration it needs from there.

> Previously we had **ANT**. With Ant we had to tell it *what to do AND how to do it* (write every step: compile this folder, copy that file, zip this…).
> With Maven we just tell it **what** to do with a command (`mvn package`), and it already knows **how**, by convention.

---

## 2. Why Maven, when we can run Java with commands?

This is a very common question: Java already gives us `javac` (compile) and `java` (run). Why do we need another tool?

![svg](<svgs/01-why-maven-vs-plain-java.svg>)

### 2.1 For one file, plain commands are enough

```bash
# Hello.java
javac Hello.java        # creates Hello.class
java Hello              # runs it → Hello World

# Java 11+ can even compile & run a single file in one step
java Hello.java
```

### 2.2 For a real project, it quickly becomes painful

A small Spring Boot app has dozens of classes in packages, and `spring-boot-starter-web` alone pulls in **~30 jars** (Spring, Tomcat, Jackson, logging…). Doing it by hand looks like this:

```bash
# 1. Download every jar AND every jar THOSE jars need, in compatible versions → put them in lib/
#    (spring-web-6.1.4.jar, spring-core-6.1.4.jar, tomcat-embed-core-10.1.19.jar, jackson-databind-2.15.4.jar, …)

# 2. Compile all sources with all jars on the classpath
javac -d out -cp "lib/*" $(find src/main/java -name "*.java")

# 3. Copy resources (application.properties …)
cp -r src/main/resources/* out/

# 4. Compile and run tests (also needs junit + its dependencies on the classpath)
javac -d test-out -cp "out:lib/*:test-lib/*" $(find src/test/java -name "*.java")
java -cp "test-out:out:lib/*:test-lib/*" org.junit.platform.console.ConsoleLauncher --scan-classpath

# 5. Package
jar cfe app.jar com.conceptandcoding.learningspringboot.Application -C out .

# 6. Run: you must ship lib/ next to the jar, with the exact classpath
java -cp "app.jar:lib/*" com.conceptandcoding.learningspringboot.Application
```

(On Windows the classpath separator is `;` instead of `:`.)

| Problem with plain commands | How Maven solves it |
|---|---|
| Find & download jars manually | You declare them in `pom.xml`; Maven downloads them |
| Jars need other jars (transitive dependencies) | Maven resolves the whole dependency tree automatically |
| Version conflicts between libraries | Version management (`<dependencyManagement>`, BOMs, the Spring Boot parent) |
| Every project organised differently | One standard folder structure, so any Java dev understands any Maven project |
| Long `javac`/`jar` commands with classpaths | `mvn compile`, `mvn package` |
| Running tests by hand | `mvn test` runs every test, and the build fails if a test fails |
| "Works on my machine" | Same `pom.xml` → same build on every laptop and CI server |
| Shipping a jar plus a `lib/` folder | Spring Boot plugin builds **one executable fat jar** |
| Sharing your library with other teams | `mvn install` / `mvn deploy` publishes it to a repository |

**In one line:** `javac`/`java` are the engine; Maven is the driver that runs them in the right order, with the right libraries, the same way every time.

---

## 3. What is the use of Maven?

![svg](<svgs/02-what-maven-does.svg>)

1. **Dependency management:** download libraries and their dependencies, cache them, and resolve versions.
2. **Standard project structure:** convention over configuration.
3. **Build lifecycle:** validate → compile → test → package → verify → install → deploy, all with one command.
4. **Plugins:** extra tasks hooked into the build (code style checks, static analysis, coverage, Docker images, Spring Boot packaging).
5. **Repositories:** a local cache (`~/.m2`), Maven Central, and company repositories (Nexus/Artifactory) to share artifacts.
6. **Reproducible builds:** CI/CD servers (Jenkins, GitHub Actions) just run `mvn clean verify` or `mvn clean package`.
7. **Documentation & reports:** `mvn site`, test reports, dependency reports.

**Maven vs Ant vs Gradle**

| | Ant | Maven | Gradle |
|---|---|---|---|
| Style | Imperative: you write every step | Declarative: conventions + XML `pom.xml` | Declarative + scriptable: Groovy/Kotlin DSL |
| Dependency management | No (needs Ivy) | Yes | Yes |
| Standard structure | No | Yes | Yes (same as Maven) |
| Build speed | — | Good | Faster (incremental builds, build cache) |
| Typical use | Legacy projects | Most Spring Boot / enterprise projects | Android, large multi-module builds |

---

## 4. Installing Maven and the Maven Wrapper

```bash
# Install (needs a JDK)
sudo apt install maven          # Ubuntu / Debian
brew install maven              # macOS
# Windows: download the zip from maven.apache.org, add its bin/ to PATH

mvn -version                    # shows Maven version + which Java it uses
```

**Maven Wrapper (`mvnw`).** Projects from start.spring.io include `mvnw`, `mvnw.cmd` and a `.mvn/` folder. They download the **exact Maven version** the project expects, so nobody needs Maven installed:

```bash
./mvnw clean package            # Linux / macOS
mvnw.cmd clean package          # Windows
```

Use the wrapper on CI servers and build machines: everyone builds with the same Maven version.

---

## 5. Maven project structure

![svg](<svgs/03-maven-project-structure.svg>)

```
learningspringboot (your app name)
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── conceptandcoding (company name)
    │   │           └── learningspringboot (app name)
    │   │               └── Application.java
    │   └── resources
    │       └── application.properties
    └── test
        └── java
            └── com
                └── conceptandcoding (company name)
                    └── learningspringboot (app name)
                        └── ApplicationTest.java   ← Unit Test Cases live here (Test class)
```

Because every Maven project uses this layout, Maven knows where the code, resources and tests are without any configuration (these defaults come from the Super POM).

---

## 6. pom.xml: Project Object Model

`pom.xml` is the heart of every Maven project. It contains the project configuration.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.3</version>
        <relativePath/>
    </parent>

    <groupId>com.conceptandcoding</groupId>
    <artifactId>learningspringboot</artifactId>     <!-- Unique identifier of your project -->
    <version>0.0.1-SNAPSHOT</version>
    <name>springboot application</name>

    <properties>
        <java.version>17</java.version>             <!-- Key-value pair for configuration. Can be referenced as ${java.version} -->
    </properties>

    <repositories>
        <repository>
            <id>central</id>
            <url>https://repo.maven.apache.org/maven2</url>   <!-- Where Maven looks for dependencies and downloads artifacts (jars) -->
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>                               <!-- The dependencies our project relies on -->
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### pom.xml parts explained

**① XML Schema declaration:**
- Points to the XML schema that defines what a `pom.xml` may contain. You **can't** invent your own tags: `<parent>` exists because the schema defines it, but you can't write `<xyz>`. Everything has to match the schema.

**② `<parent>` block:**
- Says this project is a **child** of the project in `<parent>`. (**groupId** + **artifactId** + version uniquely identify a project.) Here `spring-boot-starter-parent` is the parent of our project, and `spring-boot-starter-parent` itself (through `spring-boot-dependencies`) is ultimately a child of the **Super POM**.
- If `<parent>` is NOT specified, Maven inherits from the **Super POM** by default. Every `pom.xml` is a child of the Super POM.
- Super POM: https://maven.apache.org/ref/3.6.3/maven-model-builder/super-pom.html

**③ `<groupId>`, `<artifactId>`, `<version>`:**
- Together (**GAV coordinates**) they uniquely identify a project, in your company and in Maven Central.
- `groupId` = organisation (reversed domain), `artifactId` = project name, `version` = release.

**④ `<properties>`:**
- Key-value pairs you can reference anywhere in the POM as `${key}`. For example, the Spring Boot parent reads `${java.version}` to configure the compiler.

**⑤ `<repositories>`:**
- The Central repo link is **NOT** needed here. It's shown just to tell you where downloads come from; it's already defined in the Super POM.

**⑥ `<build>`:** plugins and extra build tasks (see the [build lifecycle](#9-maven-build-lifecycle)).

### POM inheritance and the effective POM

![svg](<svgs/04-pom-inheritance-and-gav.svg>)

```bash
mvn help:effective-pom      # prints the final POM after merging Super POM + parent + yours
```

- **`-SNAPSHOT` version** (e.g. `0.0.1-SNAPSHOT`): work in progress. Maven may download a newer build of the same snapshot. Never release with snapshot dependencies.
- **Release version** (e.g. `1.0.0`): fixed forever, and can't be overwritten in a repository.

---

## 7. Dependencies: scopes, transitive deps and conflicts

### 7.1 Scopes

![svg](<svgs/08-dependency-scopes.svg>)

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>     <!-- not needed to compile, needed to run -->
</dependency>
```

| Scope | Compile main code | Run tests | In the final jar | Example |
|---|---|---|---|---|
| `compile` (default) | ✅ | ✅ | ✅ | `spring-boot-starter-web` |
| `provided` | ✅ | ✅ | ❌ (server provides it) | `jakarta.servlet-api` for WAR, Lombok |
| `runtime` | ❌ | ✅ | ✅ | JDBC drivers |
| `test` | ❌ (test code only) | ✅ | ❌ | JUnit, `spring-boot-starter-test` |

### 7.2 Transitive dependencies and version conflicts

You declare `spring-boot-starter-web`, and Maven also brings in what **it** needs (Spring MVC, Tomcat, Jackson…), recursively. See it with:

```bash
mvn dependency:tree
```

```text
[INFO] com.conceptandcoding:learningspringboot:jar:0.0.1-SNAPSHOT
[INFO] +- org.springframework.boot:spring-boot-starter-web:jar:3.2.3:compile
[INFO] |  +- org.springframework.boot:spring-boot-starter-json:jar:3.2.3:compile
[INFO] |  |  \- com.fasterxml.jackson.core:jackson-databind:jar:2.15.4:compile
[INFO] |  +- org.springframework.boot:spring-boot-starter-tomcat:jar:3.2.3:compile
[INFO] |  |  \- org.apache.tomcat.embed:tomcat-embed-core:jar:10.1.19:compile
[INFO] |  \- org.springframework:spring-webmvc:jar:6.1.4:compile
...
```

If two paths bring **different versions** of the same library, Maven picks the **nearest** one in the tree ("nearest wins"; if equally near, the first declared wins). To control it:

```xml
<!-- 1. Pin a version for the whole project (doesn't add the dependency, only sets its version) -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.17.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 2. Remove an unwanted transitive dependency -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>   <!-- e.g. to use Jetty instead -->
        </exclusion>
    </exclusions>
</dependency>
```

> This is why Spring Boot dependencies have **no `<version>`**: `spring-boot-starter-parent` imports `spring-boot-dependencies`, a **BOM** (Bill of Materials) that lists tested, compatible versions of hundreds of libraries in its `<dependencyManagement>`.

---

## 8. Maven repositories

![svg](<svgs/05-dependency-resolution-and-repos.svg>)

- **Local repository:** on your PC (`~/.m2/repository`). A **cache**: once a jar is downloaded, Maven never downloads it again.
- **Maven Central:** the public remote repository (no authentication needed).
- **Company remote repository** (Nexus / Artifactory): in the company network. Whatever jar your team uploads goes there, and it usually also proxies/caches Maven Central.

Lookup order: **local `.m2` → remote repositories** (company repo / Central) → save into `.m2`.

**`settings.xml`** configures Maven for *you / this machine* (not for one project):
- `~/.m2/settings.xml`: user settings. **Not created automatically**; create it when you need it.
- `$MAVEN_HOME/conf/settings.xml`: global settings that ship with Maven.
- The `~/.m2/repository` folder **is** created automatically the first time Maven downloads something.

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              http://maven.apache.org/xsd/settings-1.0.0.xsd">

  <!-- Local Repository: Location where Maven stores downloaded artifacts -->
  <localRepository>${user.home}/.m2/repository</localRepository>

  <!-- Send ALL downloads through the company repository instead of going to Central directly -->
  <mirrors>
    <mirror>
      <id>company-nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.mycompany.com/repository/maven-public/</url>
    </mirror>
  </mirrors>

  <!-- some other configurations go here -->

</settings>
```

> **`pom.xml` vs `settings.xml`**: `pom.xml` belongs to the **project** and is committed to Git (dependencies, plugins, *where* to deploy). `settings.xml` belongs to the **machine/user** and is never committed (local repo path, mirrors, **passwords**).

---

## 9. Maven build lifecycle

![svg](<svgs/06-maven-build-lifecycle.svg>)

| # | Phase | Description |
|---|-------|-------------|
| 1 | **validate** project structure | Validates the project is correct |
| 2 | **compile** source code | Compiles Java source |
| 3 | **test** the code (unit tests) | Runs unit tests |
| 4 | **package** compiled code (JAR or WAR) | Packages to JAR/WAR |
| 5 | **verify** the integrity of the package | Runs integration checks |
| 6 | **install** the package in the local repository | Installs to `~/.m2/` |
| 7 | **deploy** the package to a remote repository | Deploys to a remote repo |

- Phases run in this **fixed order**. If you run the `package` phase, **all its previous phases run first**.
- If you run a specific goal of a particular phase, all the goals of the previous phases run first, then the goals of the current phase up to the one you defined.

### Phases, goals and plugins

![svg](<svgs/07-phases-goals-plugins.svg>)

- A **phase** is a *step name* in the lifecycle (`compile`, `test`, `package`…). It does nothing by itself.
- A **goal** is the actual *task*, provided by a **plugin** (`compiler:compile`, `surefire:test`, `jar:jar`).
- Maven already binds default goals to the phases. If we want to add **more** tasks to a phase, we use the `<build>` element and attach a plugin goal to that phase.

Suppose at the 6th phase there are 3 tasks (T1, T2, T3), and you want to add one more task, T4. You add it with `<build>` → `<plugin>` → `<execution>` with `<phase>` + `<goal>`.

---

## 10. Lifecycle phases in detail

### Step 1: Validate (`mvn validate`)

`<build>` lets you add a new task inside one of the phases:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-checkstyle-plugin</artifactId>
      <version>3.1.2</version>
      <executions>
        <execution>
          <id>validate-checkstyle</id>
          <phase>validate</phase>    <!-- added to this phase -->
          <goals>
            <goal>check</goal>       <!-- 1 Task -->
          </goals>
        </execution>
      </executions>
      <configuration>
        <configLocation>myCodeStyle.xml</configLocation>
      </configuration>
    </plugin>
  </plugins>
</build>
```

> **Plugin:** Maven supports adding new plugins. The `<configuration>` block gives the plugin its settings.

`check` checks the **code style**, which varies across companies. We configure the rules in `myCodeStyle.xml`.

---

### Step 2: Compile (`mvn compile`)

**Run the command:** `mvn compile`

- It will **validate** and **compile** your code, and put the output under `${project.basedir}/target/classes`.
- Creates the **bytecode** (`.class` files) from the Java code.
- Internally, `maven-compiler-plugin` calls the Java compiler (the same one `javac` uses) with the full classpath built from your dependencies.
- The `target` folder is generated in this phase.

> You won't see any validation work happening in the `validate` phase by default. It has no tasks unless you add one (like checkstyle above).

---

### Step 3: Test (`mvn test`)

- It will **validate**, **compile**, compile the tests, and then run the **test cases** in your project (with `maven-surefire-plugin`).
- If any test fails, the build fails and later phases (package…) don't run.

Unit test class (`SpringbootApplicationTests.java`):

```java
package com.conceptandcoding.learningspringboot;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class SpringbootApplicationTests {

    @Test
    void contextLoads() {
        System.out.println("CONCEPT && CODING : TEST CASE RUNNING");
    }
}
```

> The `System.out.println` output shows up in the test output, and reports are written to `target/surefire-reports/`.

---

### Step 4: Package (`mvn package`)

Creates the **JAR** file.

- First completes the validate, compile and test phases, then runs the package phase, which **generates the `.jar` or `.war` file**.
- **The JAR is put here:** `target/learningspringboot-0.0.1-SNAPSHOT.jar`
- This JAR can be used by others as a dependency, or deployed and run.
- With Spring Boot, `spring-boot-maven-plugin` (goal `repackage`) turns it into an **executable fat jar** (see [section 13](#13-spring-boot--maven)).

---

### Step 5: Verify (`mvn verify`)

Nothing is forced here: there's no task in this phase by default.

- It can perform additional checks beyond unit tests, like:
  - **Integration tests** (with `maven-failsafe-plugin`)
  - **Static code analysis**
  - **Checksum verification**, code coverage thresholds (JaCoCo), etc.

**Static code analysis** with the PMD plugin (in the `verify` phase):

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-pmd-plugin</artifactId>
  <version>3.21.2</version>
  <executions>
    <execution>
      <id>pmd-analysis</id>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>     <!-- 'pmd' only writes a report; 'check' also FAILS the build on violations -->
      </goals>
    </execution>
  </executions>
</plugin>
```

**PMD is a source code analyzer.** It finds:
- unused variables
- unused imports
- empty catch blocks
- objects that are never used
- duplicate code, etc.

It reports these as warnings (or build failures with `check`).

---

### Step 6: Install (`mvn install`)

Installs the package into the **local repository**.

- It copies the `.jar` into your local Maven repository, which is typically in your user home directory: `~/.m2/repository` on Mac/Linux, `C:\Users\<you>\.m2\repository` on Windows.
- Path inside: `~/.m2/repository/com/conceptandcoding/learningspringboot/0.0.1-SNAPSHOT/learningspringboot-0.0.1-SNAPSHOT.jar`
- Now **other projects on your machine** can use it as a dependency with the same GAV.

---

### Step 7: Deploy (`mvn deploy`)

Deploys the `.jar` to a **remote repository** (company repo or Maven Central), so other developers/machines can use it.

**It needs configuration in both `pom.xml` and `settings.xml`.**

**`pom.xml`:**

```xml
<project>
  <!-- ... other project configurations ... -->

  <distributionManagement>
    <repository>
      <id>remote repository id</id>
      <url>https://remote-repository-url</url>    <!-- URL where you want to deploy -->
    </repository>
  </distributionManagement>

  <!-- ... other pom.xml configurations ... -->
</project>
```

**`settings.xml`:**

```xml
<settings>
  <!-- other settings configurations -->

  <servers>
    <server>
      <id>remote repository id</id>                      <!-- must match the <id> in pom.xml -->
      <username>remote-repository-username</username>    <!-- Username & Password go here -->
      <password>remote-repository-password</password>
    </server>
  </servers>

  <!-- other settings configurations -->
</settings>
```

> **URL** where you want to deploy → put it in `pom.xml` (committed, shared).
> **Username & password** → put them in `settings.xml` (private, never committed).

If we don't define the remote repository, `mvn deploy` fails with:

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:3.1.1:deploy
(default-deploy) on project learningspringboot: Deployment failed: repository element
was not specified in the POM inside distributionManagement element ...
```

> Deploying to Maven Central (https://repo.maven.apache.org/maven2) means publishing through Sonatype's publishing portal. Downloading from Central needs no credentials, but **publishing** to it needs an account, a verified groupId and signed artifacts.

---

## 11. Lifecycle reference and other lifecycles

### Default (build) lifecycle: complete table

| Phase | Description | Plugin goal (generally) |
|-------|-------------|------------------------|
| validate | Validate the project is correct and all necessary information is available | – |
| initialize | Initialize build state | – |
| generate-sources | Generate any source code | – |
| process-sources | Process the source code | – |
| generate-resources | Generate resources | – |
| process-resources | Copy and process resources into `target/classes` | resources:resources |
| compile | Compile the source code of the project | compiler:compile |
| process-classes | Post-process the generated classes | – |
| generate-test-sources | Generate any test source code | – |
| process-test-sources | Process test source code | – |
| generate-test-resources | Generate test resources | – |
| process-test-resources | Copy and process test resources | resources:testResources |
| test-compile | Compile the test source code | compiler:testCompile |
| test | Run tests using a suitable unit testing framework | surefire:test |
| prepare-package | Perform any operations needed before packaging | – |
| package | Package the compiled code in its distributable format (jar/war) | jar:jar or war:war |
| verify | Run any checks to verify the package is valid | – |
| install | Install the package into the local repository (.m2) | install:install |
| deploy | Copy the final package to the remote repository | deploy:deploy |

### Clean lifecycle

Used to clean the project.
- `pre-clean`
- `clean` → deletes the `target/` directory
- `post-clean`

### Site lifecycle

Used to generate project documentation.
- `pre-site`
- `site` → generates the site in `target/site`
- `post-site`
- `site-deploy`

### Examples of lifecycle execution

- If we run a phase, all previous phases are executed:

```
mvn test     → validate … compile … test-compile, test
mvn package  → runs everything up to package
mvn install  → validate → compile → test → package → verify → install
mvn deploy   → runs the entire default lifecycle
```

`mvn clean install` runs the **clean** lifecycle first, then the default lifecycle up to `install`.

### Folders created by a Maven build

After `mvn package` (jar project):

```
project-root/
├── src/
├── target/
│   ├── classes/            (compiled classes + resources)
│   ├── test-classes/       (compiled test classes)
│   ├── surefire-reports/   (test results)
│   ├── app.jar             (executable jar)
│   └── ...
└── pom.xml
```

---

## 12. Common commands and options

| Command | Description |
|---------|-------------|
| `mvn clean` | Cleans the project (deletes `target/`) |
| `mvn compile` | Compiles main source code |
| `mvn test` | Compiles and runs tests |
| `mvn package` | Packages the code (jar/war) |
| `mvn verify` | Package + integration tests / quality checks (common in CI) |
| `mvn install` | Installs the package to the local repo (`.m2`) |
| `mvn deploy` | Deploys the package to the remote repo |
| `mvn clean install` | Clean + build + test + install |
| `mvn clean package` | Clean + build + test + package |
| `mvn clean install -DskipTests` | Build and install, but skip running tests |
| `mvn dependency:tree` | Shows the dependency tree |
| `mvn help:effective-pom` | Shows the final POM after inheritance |
| `mvn help:effective-settings` | Shows the effective settings |
| `mvn spring-boot:run` | Runs a Spring Boot app directly from source |
| `mvn -pl module-a -am package` | Builds one module (and what it needs) in a multi-module project |

| Option | Description |
|--------|-------------|
| `-DskipTests` | Skip test execution (but still compiles tests) |
| `-Dmaven.test.skip=true` | Skip test compilation **and** execution |
| `-X` | Enable debug output |
| `-e` | Show the full stack trace of errors |
| `-U` | Force update of snapshots / re-check missing releases |
| `-o` | Offline mode (use only `.m2`) |
| `-P<profile-id>` | Activate a specific profile |
| `-q` | Quiet mode (fewer logs) |
| `-T 4` | Build modules in parallel with 4 threads |
| `-version` | Show the Maven version |

---

## 13. Spring Boot + Maven

### Typical flow

1. Create a Spring Boot project (Spring Initializr, start.spring.io).
2. `pom.xml` contains `spring-boot-starter-parent` (manages versions).
3. Add dependencies (starters).
4. Write code.
5. Run:
   - `mvn spring-boot:run` → run the application from source (development)
   - `mvn clean package` → creates the executable jar
   - `java -jar target/app.jar` → run the jar (this is what you do on servers)

### The executable ("fat") jar

![svg](<svgs/09-spring-boot-executable-jar.svg>)

A normal jar contains only **your** classes, so you'd need all dependency jars on the classpath to run it. The `spring-boot-maven-plugin` `repackage` goal (run automatically in `package`) builds a **fat jar** containing your code **and** every dependency, plus a small launcher:

```bash
mvn clean package
ls target/
# learningspringboot-0.0.1-SNAPSHOT.jar            ← fat, executable (~20 MB)
# learningspringboot-0.0.1-SNAPSHOT.jar.original   ← plain jar (your classes only)

java -jar target/learningspringboot-0.0.1-SNAPSHOT.jar
```

Tip: give the jar a fixed name, so deployment scripts don't change with every version:

```xml
<build>
    <finalName>app</finalName>      <!-- produces target/app.jar -->
    ...
</build>
```

### Important Maven plugins in Spring Boot

| Plugin | Purpose |
|--------|---------|
| `spring-boot-maven-plugin` | Builds the executable jar/war (`repackage`), runs the app (`spring-boot:run`), builds Docker images (`build-image`) |
| `maven-compiler-plugin` | Compiles Java code |
| `maven-surefire-plugin` | Runs unit tests (JUnit) |
| `maven-failsafe-plugin` | Runs integration tests (`*IT.java`) in `verify` |
| `maven-jar-plugin` | Creates the JAR file |
| `maven-war-plugin` | Creates the WAR file |
| `maven-resources-plugin` | Copies `src/main/resources` into `target/classes` |

---

## 14. How to run a Java / Spring Boot app on a Linux server

![svg](<svgs/10-run-spring-boot-on-linux-server.svg>)

### 14.1 Build the jar (on your laptop or in CI)

```bash
./mvnw clean package            # runs tests too; add -DskipTests only if they already ran in CI
ls target/app.jar               # with <finalName>app</finalName>
```

> The server only needs a **JRE** to *run* the jar. It needs a **JDK + Maven** only if you build on the server itself (`git clone` + `./mvnw package`).

### 14.2 Prepare the server (once)

```bash
ssh user@your-server-ip

# Install Java (pick the version your app is compiled for)
sudo apt update
sudo apt install -y openjdk-17-jre-headless        # Ubuntu / Debian
# sudo dnf install -y java-17-openjdk-headless     # RHEL / Amazon Linux / Fedora
java -version

# A dedicated user and folder for the app (don't run apps as root)
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp
sudo mkdir -p /opt/myapp
sudo chown myapp:myapp /opt/myapp
```

### 14.3 Copy the jar to the server

```bash
# from your laptop
scp target/app.jar user@your-server-ip:/tmp/app.jar
ssh user@your-server-ip "sudo mv /tmp/app.jar /opt/myapp/app.jar && sudo chown myapp:myapp /opt/myapp/app.jar"
```

### 14.4 Run it: the quick way (testing only)

```bash
cd /opt/myapp
java -jar app.jar                                          # foreground: stops when you close SSH / Ctrl+C

java -Xms256m -Xmx512m -jar app.jar \
     --spring.profiles.active=prod --server.port=8080      # JVM options BEFORE -jar, Spring args AFTER

nohup java -jar app.jar > app.log 2>&1 &                   # background: survives logout, NOT reboots/crashes
tail -f app.log                                            # watch the logs
ps -ef | grep app.jar                                      # find the process id
kill <pid>                                                 # graceful stop (SIGTERM)
```

`nohup` is fine for a quick test, but nothing restarts the app after a crash or a reboot. For real use, let **systemd** manage it.

### 14.5 Run it properly: a systemd service

Create `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Spring Boot App
After=network.target

[Service]
User=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=/opt/myapp/myapp.env
ExecStart=/usr/bin/java -Xms256m -Xmx512m -jar /opt/myapp/app.jar --spring.profiles.active=prod
SuccessExitStatus=143
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

- `Restart=on-failure`: restart automatically if the app crashes.
- `WantedBy=multi-user.target` + `enable`: start automatically when the server boots.
- `SuccessExitStatus=143`: a JVM stopped with SIGTERM exits with 143. This tells systemd that's a normal stop.

```bash
sudo systemctl daemon-reload            # reload after creating/editing the file
sudo systemctl enable --now myapp       # start now AND on every boot
sudo systemctl status myapp             # is it running?
sudo systemctl restart myapp            # after copying a new jar
sudo systemctl stop myapp
journalctl -u myapp -f                  # live logs (stdout of the app)
journalctl -u myapp --since "10 min ago"
```

### 14.6 Configuration and secrets on the server

Don't bake production passwords into the jar. Spring Boot reads configuration from outside the jar:

```bash
# /opt/myapp/myapp.env  (chmod 600, owned by myapp): loaded by EnvironmentFile=
SPRING_DATASOURCE_URL=jdbc:mysql://db-host:3306/appdb
SPRING_DATASOURCE_USERNAME=app
SPRING_DATASOURCE_PASSWORD=super-secret
```

Environment variables map to properties (`SPRING_DATASOURCE_PASSWORD` → `spring.datasource.password`). You can also put an `application-prod.properties` file next to the jar (`/opt/myapp/config/`); Spring Boot picks up `./config/` from the working directory automatically.

### 14.7 Check that it works

```bash
curl http://localhost:8080/actuator/health        # {"status":"UP"} (needs spring-boot-starter-actuator)
ss -tlnp | grep 8080                              # is something listening on 8080?
```

### 14.8 Expose it with nginx (reverse proxy + HTTPS)

Users shouldn't hit port 8080 directly. Put nginx in front on 80/443:

```nginx
# /etc/nginx/sites-available/myapp   (then symlink into sites-enabled/)
server {
    listen 80;
    server_name myapp.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo apt install -y nginx
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d myapp.com          # free HTTPS certificate (Let's Encrypt)
```

In the app, set `server.forward-headers-strategy=native` so Spring knows the original request was HTTPS.

### 14.9 Deploying a new version

```bash
scp target/app.jar user@server:/tmp/app.jar
ssh user@server "sudo cp /opt/myapp/app.jar /opt/myapp/app.jar.bak \
  && sudo mv /tmp/app.jar /opt/myapp/app.jar \
  && sudo systemctl restart myapp"
# rollback: move app.jar.bak back and restart
```

In real projects a CI/CD pipeline (Jenkins, GitHub Actions) runs these steps on every merge.

### 14.10 Running a plain (non-Spring) Java app on a server

Without Spring Boot's repackaging, a normal jar has no dependencies inside it. Three options:

```bash
# a) No dependencies: jar with a Main-Class in the manifest
jar cfe app.jar com.example.Main -C out .
java -jar app.jar

# b) Dependencies as separate jars: list them on the classpath (':' on Linux)
java -cp "app.jar:lib/*" com.example.Main
```

```xml
<!-- c) Build ONE "uber jar" containing all dependencies with the Shade plugin -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.example.Main</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Then `mvn package` → `java -jar target/myapp-1.0.jar`, and the same systemd setup works.

### 14.11 Alternatives: Docker, and WAR on an external Tomcat

**Docker:** the server only needs Docker, no Java.

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -t myapp:1.0 .
docker run -d --name myapp -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod --restart unless-stopped myapp:1.0
# or, without a Dockerfile:  ./mvnw spring-boot:build-image
```

**WAR on an existing Tomcat** (older setups): set `<packaging>war</packaging>`, mark `spring-boot-starter-tomcat` as `provided`, extend `SpringBootServletInitializer`, then copy `target/app.war` into `$TOMCAT_HOME/webapps/`.

---

## 15. Profiles and multi-module projects

**Maven profiles** change the build per environment (not to be confused with Spring profiles):

```xml
<profiles>
    <profile>
        <id>ci</id>
        <properties>
            <skipITs>false</skipITs>
        </properties>
    </profile>
</profiles>
```

```bash
mvn clean verify -Pci
```

**Multi-module project:** one parent POM (`<packaging>pom</packaging>`) listing `<modules>`. Building the parent builds all modules in dependency order:

```
shop-parent/
├── pom.xml            (<modules> common, order-service, payment-service </modules>)
├── common/pom.xml
├── order-service/pom.xml      (depends on common)
└── payment-service/pom.xml    (depends on common)
```

---

## 16. Key takeaways

- `javac`/`java` compile and run code. **Maven organises the whole build**: dependencies, structure, compile, test, package, publish, the same way on every machine.
- Maven follows a **lifecycle-based build**. Running a phase runs **all previous phases** automatically.
- Phases are steps; **plugin goals** do the work; `<build>` attaches extra goals to phases.
- Maven manages **dependencies automatically**, including transitive ones, cached in `~/.m2`.
- `pom.xml` = project config (in Git). `settings.xml` = machine config and passwords (not in Git).
- In Spring Boot, `spring-boot-maven-plugin` builds one **executable fat jar**. On a Linux server you need just a JRE: `java -jar app.jar`, managed by **systemd**, behind **nginx**.

---

## 17. Interview Questions

**Q1. What is Maven?**
A build and project management tool. Driven by `pom.xml`, it manages dependencies, follows a standard project structure and build lifecycle, and runs plugins to compile, test, package and publish Java projects.

**Q2. Why use Maven if `javac` and `java` can compile and run code?**
Plain commands don't download libraries, resolve transitive dependencies or versions, run tests, package with dependencies, or give a standard layout and reproducible builds. Maven does all of that with one command (`mvn package`), identically on every machine and CI server.

**Q3. What are GAV coordinates?**
`groupId:artifactId:version`, the unique identity of a project/artifact in any Maven repository.

**Q4. What is the Super POM? The effective POM?**
The Super POM is Maven's built-in default parent of every POM (Central repo, default directories, default plugin bindings). The effective POM is the result of merging Super POM + parents + your POM (`mvn help:effective-pom`).

**Q5. Phase vs goal vs plugin?**
A plugin is a collection of goals (e.g. `maven-compiler-plugin`). A goal is one task (`compiler:compile`). A phase is a lifecycle step (`compile`) that goals are bound to. Running a phase runs all goals bound to it and all earlier phases.

**Q6. What does `mvn install` do vs `mvn package` vs `mvn deploy`?**
`package` creates the jar in `target/`. `install` also copies it into the local `~/.m2` repository (for other local projects). `deploy` also uploads it to a remote repository (for other machines/teams).

**Q7. Local vs central vs remote repository?**
Local = `~/.m2/repository` cache on your machine. Central = public Maven Central. Remote/company = Nexus/Artifactory for internal artifacts, often also proxying Central. Maven checks local first, then remotes.

**Q8. `pom.xml` vs `settings.xml`?**
`pom.xml` is per project, in Git (dependencies, plugins, distribution URL). `settings.xml` is per user/machine, not in Git (local repo path, mirrors, proxies, credentials).

**Q9. What are dependency scopes?**
`compile` (default, everywhere), `provided` (compile but supplied at runtime by the container), `runtime` (not for compiling, needed to run), `test` (test code only), plus `system` and `import` (BOMs in `dependencyManagement`).

**Q10. What are transitive dependencies, and how are conflicts resolved?**
Dependencies of your dependencies, pulled in automatically. On a version conflict the nearest declaration in the tree wins (first declared on a tie). Control it with `<dependencyManagement>`, explicit versions or `<exclusions>`. Inspect with `mvn dependency:tree`.

**Q11. `<dependencyManagement>` vs `<dependencies>`?**
`<dependencies>` actually adds libraries to the project. `<dependencyManagement>` only fixes versions (and scopes) for when they're used, by this project or its children. A BOM is a POM made of `dependencyManagement` entries that you import.

**Q12. Why don't Spring Boot dependencies need a `<version>`?**
`spring-boot-starter-parent` imports the `spring-boot-dependencies` BOM, which manages tested, compatible versions for hundreds of libraries.

**Q13. What is a SNAPSHOT version?**
A development version (`1.0-SNAPSHOT`) that can change. Maven may fetch newer builds of it. Releases (`1.0`) are immutable.

**Q14. `-DskipTests` vs `-Dmaven.test.skip=true`?**
`-DskipTests` compiles tests but doesn't run them. `-Dmaven.test.skip=true` skips compiling and running them.

**Q15. What is the Maven Wrapper?**
`mvnw`/`mvnw.cmd` + `.mvn/` scripts in the project that download and use a fixed Maven version, so builds don't depend on what's installed on the machine.

**Q16. Maven vs Gradle?**
Both use the same repositories and directory conventions. Maven is declarative XML with a fixed lifecycle. Gradle uses a Groovy/Kotlin DSL, is more flexible, and is usually faster thanks to incremental builds and a build cache.

**Q17. What is a fat/uber jar? How does Spring Boot create one?**
A jar that contains your classes plus all dependencies, so `java -jar` just works. `spring-boot-maven-plugin`'s `repackage` goal puts classes in `BOOT-INF/classes`, libraries in `BOOT-INF/lib`, and sets `Main-Class` to Spring's `JarLauncher` and `Start-Class` to your main class. For non-Spring apps, use the Shade or Assembly plugin.

**Q18. How do you run a Spring Boot application on a Linux server?**
Build `app.jar` with `mvn clean package`, copy it to the server (`scp`), install a JRE, then run `java -jar app.jar --spring.profiles.active=prod`. For production, create a systemd service (auto start on boot, restart on failure, logs via `journalctl`), keep secrets in environment variables or external config, and put nginx in front for HTTPS. Alternatively, ship it as a Docker image.

**Q19. Why not use `mvn spring-boot:run` in production?**
It needs the source code, Maven and a JDK on the server, recompiles on start, and keeps Maven as the parent process. Production should run the tested, packaged artifact with `java -jar` (or a container image) for reproducibility and faster startup.

**Q20. What is `nohup java -jar app.jar &` missing compared to systemd?**
It survives logout, but not crashes or reboots, has no automatic restart, no managed logs and no clean start/stop/status commands. systemd provides all of these.

---

## 18. Server / system questions (running in production)

Once the jar runs on a Linux server, these are the questions you'll face (and get asked in interviews).

### 18.1 How do you check disk usage and free up space on a Linux server?

**Step 1: which filesystem is full?**

```bash
df -h                 # space per filesystem (look at Use% and "Mounted on")
df -i                 # inodes: "No space left on device" can also mean too many small files
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       30G   29G  1.0G  97% /
```

**Step 2: which folders/files are big?**

```bash
sudo du -h --max-depth=1 / 2>/dev/null | sort -h | tail -15      # biggest top-level folders
sudo du -sh /var/log/* /opt/myapp/* 2>/dev/null | sort -h         # drill down
sudo find / -xdev -type f -size +500M -exec ls -lh {} \; 2>/dev/null   # files > 500 MB
sudo ncdu /            # interactive explorer (sudo apt install ncdu)
```

**Step 3: free space safely.** Usual suspects for a Java server:

| What | How to clean |
|---|---|
| App logs (`app.log`, `nohup.out`) | `truncate -s 0 /opt/myapp/app.log` (empties the file even while the app writes to it) |
| systemd journal | `sudo journalctl --vacuum-size=500M` or `--vacuum-time=7d` |
| Old rotated logs | `sudo find /var/log -name "*.gz" -mtime +14 -delete` |
| Heap dumps (`*.hprof`, often GBs) | `find / -xdev -name "*.hprof" 2>/dev/null`, delete after analysing |
| Old jars / backups | remove old `app.jar.bak`, old releases |
| Package cache | `sudo apt clean && sudo apt autoremove` |
| Docker | `docker system df`, then `docker system prune -a` (removes unused images/containers) |
| `/tmp` | old upload / temp files |

> ⚠️ **Deleted but still using space?** If you `rm` a log file that a running process still has open, the space is **not** freed until the process closes it. Find such files with `sudo lsof +L1` (or `lsof | grep deleted`), then restart the app or truncate the file. That's why `truncate` is safer than `rm` for live logs.

**Step 4: stop it from happening again.**

```properties
# application.properties: Spring Boot's rolling log files
logging.file.name=/opt/myapp/logs/app.log
logging.logback.rollingpolicy.max-file-size=10MB
logging.logback.rollingpolicy.max-history=7
logging.logback.rollingpolicy.total-size-cap=1GB
```

- Use `logrotate` for any other log files, and cap the journal (`SystemMaxUse=1G` in `/etc/systemd/journald.conf`).
- Alert at ~80% disk usage (Prometheus node exporter / CloudWatch). The Actuator `/health` `diskSpace` component turns DOWN below a threshold (`management.health.diskspace.threshold=1GB`).

---

### 18.2 A service is slow, but CPU and memory look normal. What do you check next?

If CPU is low and memory is fine, the threads are **not working, they're waiting**: for a pool, the DB, another service, the disk/network, a lock, or the GC.

![svg](<svgs/11-slow-service-cpu-memory-normal.svg>)

**A systematic order**

1. **Where is the time spent?** Check the latency metrics and traces for the slow endpoint: `/actuator/metrics/http.server.requests?tag=uri:/api/orders`, and Zipkin/APM spans (DB span? HTTP-call span?).
2. **Thread pools exhausted?**
   - `tomcat.threads.busy` == `server.tomcat.threads.max` (200) → every worker thread is busy waiting.
   - `hikaricp.connections.active` == max and `hikaricp.connections.pending` > 0 → requests are queueing for a DB connection.
3. **Take thread dumps (the single most useful step).** Take 3 of them, ~5 s apart:
   ```bash
   jcmd <pid> Thread.print > td1.txt      # or: jstack <pid>, or GET /actuator/threaddump
   ```
   Look at what most threads are doing:
   - `java.net.SocketInputStream.socketRead0` → waiting on a slow DB/HTTP call (probably no timeout set)
   - `HikariPool.getConnection` → DB pool exhausted
   - `BLOCKED (on object monitor)` → lock contention or a deadlock (see 18.3)
4. **Database:** slow-query log, row locks (`SHOW PROCESSLIST` / `pg_stat_activity`), missing indexes (`EXPLAIN ANALYZE`), long transactions holding connections.
5. **Downstream services:** their latency/error rate; are retries multiplying the load? Are timeouts configured?
6. **Disk / network I/O:**
   ```bash
   top                 # %wa (I/O wait) high → disk is the bottleneck; load average high while CPU is idle
   vmstat 1            # 'wa' column, 'b' = processes blocked on I/O
   iostat -xz 1        # %util close to 100 → disk saturated
   ss -s               # socket summary; thousands in TIME_WAIT?
   ```
   Also check DNS resolution time and packet loss (`mtr <host>`). `curl -w` timing splits a request into DNS / connect / TLS / first byte:
   ```bash
   curl -o /dev/null -s -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" https://api.example.com/health
   ```
7. **GC pauses:** memory can look fine while long stop-the-world pauses freeze everything: check `jvm.gc.pause` max / GC logs (`-Xlog:gc*`).
8. **OS limits:** `Too many open files` (`ulimit -n`, `lsof -p <pid> | wc -l`), running out of ephemeral ports, connection limits on a load balancer.

**Typical fixes:** add timeouts everywhere (HTTP clients, DB), size the pools correctly, fix slow queries/indexes, add circuit breakers, cache, scale out. See the *037 actuator* notes for detailed scenarios.

---

### 18.3 What is a thread deadlock in Java, and how do you detect one?

A **deadlock** happens when two (or more) threads each hold a lock the other one needs, and each waits for the other **forever**. Nothing crashes: the threads are just stuck in `BLOCKED` state, CPU stays low, and requests using them hang.

![svg](<svgs/12-java-thread-deadlock.svg>)

**Example: two locks taken in opposite order**

```java
public class DeadlockDemo {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {                 // holds A
                sleep(100);
                synchronized (lockB) {             // waits for B (held by t2)
                    System.out.println("t1 done");
                }
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {                 // holds B
                sleep(100);
                synchronized (lockA) {             // waits for A (held by t1)
                    System.out.println("t2 done");
                }
            }
        }, "Thread-2");

        t1.start();
        t2.start();                                // nothing is ever printed
    }

    private static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

A deadlock needs all four conditions at once: **mutual exclusion** (a lock has one owner), **hold and wait** (hold one, wait for another), **no preemption** (a lock can't be taken away), and **circular wait** (A waits for B, B waits for A). Break any one and it can't happen.

**How to detect it**

```bash
jps -l                       # find the Java process id
jstack <pid>                 # or: jcmd <pid> Thread.print
```

The JVM detects monitor deadlocks for you and prints them at the end of the dump:

```text
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f...b8 (object 0x000000071...b0, a java.lang.Object),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f...a8 (object 0x000000071...a0, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
"Thread-1":
        at DeadlockDemo.lambda$main$0(DeadlockDemo.java:11)
        - waiting to lock <0x000000071...b0> (a java.lang.Object)
        - locked <0x000000071...a0> (a java.lang.Object)
```

Other ways to detect it:
- **Spring Boot Actuator:** `GET /actuator/threaddump` → look for threads in `BLOCKED` with `lockOwnerName` pointing at each other.
- **GUI tools:** JConsole / VisualVM / JMC have a "Detect Deadlock" button.
- **In code / monitoring:** check for deadlocks periodically and alert:
  ```java
  ThreadMXBean bean = ManagementFactory.getThreadMXBean();
  long[] ids = bean.findDeadlockedThreads();      // monitors AND ReentrantLock etc.
  if (ids != null) {
      for (ThreadInfo info : bean.getThreadInfo(ids, true, true)) {
          log.error("DEADLOCK: {}", info);
      }
  }
  ```

**How to prevent it**
- Always acquire multiple locks in the **same global order** (e.g. always lock the account with the smaller id first).
- Use `ReentrantLock.tryLock(timeout)` and back off instead of waiting forever.
- Keep synchronized blocks small, and never call external code (DB, HTTP, callbacks) while holding a lock.
- Prefer higher-level concurrency utilities (`ConcurrentHashMap`, `java.util.concurrent` queues, immutable objects) over hand-written locking.

> **Database deadlocks are different.** Two transactions locking rows in opposite order are detected by the database itself, which kills one of them (`DeadlockLoserDataAccessException` / "Deadlock found when trying to get lock"). The fix is the same idea: update rows in a consistent order, keep transactions short, and retry the losing transaction.

---

### 18.4 How do you monitor a Spring Boot application in production?

Use the three pillars of observability: **metrics** (numbers over time), **logs** (what happened), and **traces** (the path of one request across services). Add **alerts** on top.

![svg](<svgs/13-monitoring-spring-boot-in-production.svg>)

**1. Actuator: health and metrics endpoints**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.probes.enabled=true          # /actuator/health/liveness and /readiness
management.metrics.tags.application=${spring.application.name}
management.metrics.distribution.percentiles-histogram.http.server.requests=true   # p95/p99 latency
```

- `/actuator/health`: is the app up, and are the DB / disk / Redis OK? Used by load balancers and Kubernetes probes.
- `/actuator/prometheus`: every Micrometer metric (HTTP, JVM, Hikari, Tomcat, your custom ones) in Prometheus format.

**2. Metrics → Prometheus → Grafana**

Prometheus scrapes `/actuator/prometheus` every 15 s, and Grafana shows dashboards (the "JVM (Micrometer)" and "Spring Boot" community dashboards are a good start). What to watch:

| Area | Metric (Prometheus name) | Why |
|---|---|---|
| Traffic | `http_server_requests_seconds_count` (rate) | Requests per second |
| Errors | same, `status=~"5.."` | Error rate users feel |
| Latency | `http_server_requests_seconds_bucket` → p95/p99 | Slowness (averages hide it) |
| JVM memory | `jvm_memory_used_bytes{area="heap"}` | Memory leaks, OOM risk |
| GC | `jvm_gc_pause_seconds` | Pauses that freeze requests |
| Threads | `tomcat_threads_busy_threads`, `jvm_threads_states_threads{state="blocked"}` | Pool exhaustion, deadlocks |
| DB pool | `hikaricp_connections_pending` | Waiting for DB connections |
| Host | CPU, disk, network (node exporter) | Server health |

**3. Alerts.** Alert on what users feel, plus a few leading indicators:

```yaml
# Prometheus alert rules (examples)
- alert: HighErrorRate
  expr: sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
        / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
  for: 5m
- alert: SlowP99
  expr: histogram_quantile(0.99, sum by (le) (rate(http_server_requests_seconds_bucket[5m]))) > 1
  for: 10m
- alert: AppDown
  expr: up{job="myapp"} == 0
  for: 1m
```

Alertmanager routes these to Slack / email / PagerDuty.

**4. Centralised, structured logs**

- Ship logs from all instances to one place (ELK / OpenSearch, Grafana Loki, CloudWatch, Datadog) instead of SSH-ing into servers.
- Log in JSON so fields are searchable. Spring Boot 3.4+ has built-in structured logging:
  ```properties
  logging.structured.format.console=ecs     # or logstash / gelf
  ```
- Include a request/trace id in every log line (automatic with Micrometer Tracing via MDC), so you can find all logs for one failing request.

**5. Distributed tracing**

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```properties
management.tracing.sampling.probability=0.1     # trace 10% of requests in production
```

Zipkin / Tempo / Jaeger then show, for one slow request, which service and which DB/HTTP call took the time.

**6. Extras**
- An external **uptime check** (pings `/actuator/health` from outside, so it also catches network/DNS problems).
- **APM tools** (Datadog, New Relic, Dynatrace, Elastic APM) bundle metrics + traces + logs with a Java agent.
- **Secure** the actuator endpoints: expose only what's needed, put them on a separate management port (`management.server.port=9090`) or behind authentication.
- When something goes wrong: `/actuator/threaddump`, `/actuator/heapdump` (carefully), and change log levels live with `/actuator/loggers` (see the *037 actuator* notes).

**Interview one-liner:** "Actuator + Micrometer expose health and metrics. Prometheus scrapes them, Grafana visualises them, Alertmanager alerts on error rate, latency and saturation. Logs are structured JSON shipped to ELK/Loki with a trace id. Micrometer Tracing with OpenTelemetry sends traces to Zipkin/Tempo. Health endpoints feed the load balancer / Kubernetes probes."
