# Java JDBC & Spring Boot JPA — From Ground Up

---

## 1. What is JDBC?

**JDBC** stands for **Java Database Connectivity**. It is a **standard Java API** (part of `java.sql` package) that allows Java programs to talk to **any relational database** — MySQL, PostgreSQL, Oracle, H2, etc.

Think of JDBC as a **universal translator** between your Java code and the database.

```
┌──────────────┐        JDBC API        ┌──────────────┐
│  Java Code   │ ◄────────────────────► │   Database   │
│  (your app)  │    (java.sql.*)        │  (MySQL etc) │
└──────────────┘                        └──────────────┘
```

### Why do we need it?

- Every database has its own protocol (MySQL speaks differently from PostgreSQL).
- Without JDBC you would have to write **database-specific code** for each database.
- JDBC gives a **single set of interfaces** — the database vendor provides the **implementation** (called a **Driver**).

---

## 2. Key JDBC Components

| Component | Class / Interface | Purpose |
|---|---|---|
| **Driver** | `java.sql.Driver` | Translates JDBC calls into the database's native protocol |
| **DriverManager** | `java.sql.DriverManager` | Factory that creates `Connection` objects using the right driver |
| **Connection** | `java.sql.Connection` | A **live session** with the database (like a phone call) |
| **Statement** | `java.sql.Statement` | Used to send a **raw SQL** query to the database |
| **PreparedStatement** | `java.sql.PreparedStatement` | Pre-compiled SQL — **safer** and **faster** than Statement |
| **ResultSet** | `java.sql.ResultSet` | Holds the **rows returned** by a SELECT query |

---

## 3. What is a Connection?

A **Connection** (`java.sql.Connection`) is an **active, open session** between your Java program and the database server.

### What happens behind the scenes when you open a connection?

```
Step 1:  Your code calls DriverManager.getConnection(url, user, password)
Step 2:  DriverManager looks at the URL and finds the matching Driver
         (e.g. "jdbc:mysql://..." → MySQL Driver)
Step 3:  The Driver opens a TCP/IP socket to the database server
Step 4:  Authentication happens (username + password)
Step 5:  A Connection object is returned — you now have a live link
```

> **Analogy**: A Connection is like a **phone call**. You dial (getConnection), the other side picks up (authentication), and now you can talk (send SQL). When you are done, you hang up (connection.close()).

### Important things about Connection

- **It is expensive** — opening a connection involves network calls, authentication, memory allocation on both sides.
- **It must be closed** — if you forget to close, you get **connection leaks** and the database runs out of connections.
- **It is NOT thread-safe** — don't share one connection across multiple threads.
- In real apps we use a **Connection Pool** (like HikariCP) to reuse connections instead of opening/closing every time.

---

## 4. How a Query Works — Step by Step

### For a SELECT query:

```
1. Get a Connection
2. Create a Statement or PreparedStatement from the Connection
3. Execute the query → database processes it and returns rows
4. Iterate over the ResultSet (cursor-based, row by row)
5. Close ResultSet, Statement, Connection (in reverse order)
```

### For INSERT / UPDATE / DELETE:

```
1. Get a Connection
2. Create a PreparedStatement
3. Call executeUpdate() → returns an int (number of rows affected)
4. Close Statement, Connection
```

---

## 5. Complete JDBC Program (MySQL Example)

Below is a **complete, working** JDBC program that:
1. Connects to a MySQL database
2. Creates a table
3. Inserts a row
4. Queries and prints the data
5. Properly closes all resources

### 5.1 Dependencies (Maven)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>
```

### 5.2 The Full Program

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class JdbcDemo {

    // ──────────────────────────────────────────────
    //  STEP 1 — Connection Details
    // ──────────────────────────────────────────────
    // The JDBC URL tells the driver:
    //   - Protocol  → jdbc:mysql
    //   - Host      → localhost
    //   - Port      → 3306
    //   - Database  → mydb
    private static final String URL      = "jdbc:mysql://localhost:3306/mydb";
    private static final String USER     = "root";
    private static final String PASSWORD = "root";

    public static void main(String[] args) {

        // try-with-resources ensures Connection is auto-closed
        // even if an exception occurs
        try (Connection connection = DriverManager.getConnection(URL, USER, PASSWORD)) {

            System.out.println("✅ Connection established!");
            System.out.println("   Database: " + connection.getCatalog());
            System.out.println("   Auto-commit: " + connection.getAutoCommit());

            // ──────────────────────────────────────────────
            //  STEP 2 — Create Table
            // ──────────────────────────────────────────────
            String createTableSQL = """
                CREATE TABLE IF NOT EXISTS students (
                    id    INT AUTO_INCREMENT PRIMARY KEY,
                    name  VARCHAR(100) NOT NULL,
                    email VARCHAR(100) NOT NULL,
                    marks INT
                )
            """;

            try (Statement stmt = connection.createStatement()) {
                stmt.execute(createTableSQL);
                System.out.println("✅ Table 'students' ready.");
            }

            // ──────────────────────────────────────────────
            //  STEP 3 — Insert Data using PreparedStatement
            // ──────────────────────────────────────────────
            // '?' are placeholders — prevents SQL injection!
            String insertSQL = "INSERT INTO students (name, email, marks) VALUES (?, ?, ?)";

            try (PreparedStatement pstmt = connection.prepareStatement(insertSQL)) {
                // Set parameter values by index (1-based)
                pstmt.setString(1, "Rahul Sharma");
                pstmt.setString(2, "rahul@example.com");
                pstmt.setInt(3, 85);

                int rowsInserted = pstmt.executeUpdate(); // returns number of rows affected
                System.out.println("✅ Inserted " + rowsInserted + " row(s).");
            }

            // ──────────────────────────────────────────────
            //  STEP 4 — Query Data (SELECT)
            // ──────────────────────────────────────────────
            String selectSQL = "SELECT id, name, email, marks FROM students";

            try (Statement stmt = connection.createStatement();
                 ResultSet rs = stmt.executeQuery(selectSQL)) {

                System.out.println("\n📋 All Students:");
                System.out.println("─".repeat(60));

                // ResultSet works like a cursor
                // rs.next() moves to the next row; returns false when no more rows
                while (rs.next()) {
                    int    id    = rs.getInt("id");
                    String name  = rs.getString("name");
                    String email = rs.getString("email");
                    int    marks = rs.getInt("marks");

                    System.out.printf("  ID: %d | Name: %-15s | Email: %-25s | Marks: %d%n",
                                      id, name, email, marks);
                }
            }

            // ──────────────────────────────────────────────
            //  STEP 5 — Update Data
            // ──────────────────────────────────────────────
            String updateSQL = "UPDATE students SET marks = ? WHERE name = ?";

            try (PreparedStatement pstmt = connection.prepareStatement(updateSQL)) {
                pstmt.setInt(1, 95);
                pstmt.setString(2, "Rahul Sharma");

                int rowsUpdated = pstmt.executeUpdate();
                System.out.println("\n✅ Updated " + rowsUpdated + " row(s).");
            }

            // ──────────────────────────────────────────────
            //  STEP 6 — Delete Data
            // ──────────────────────────────────────────────
            String deleteSQL = "DELETE FROM students WHERE marks < ?";

            try (PreparedStatement pstmt = connection.prepareStatement(deleteSQL)) {
                pstmt.setInt(1, 40);

                int rowsDeleted = pstmt.executeUpdate();
                System.out.println("✅ Deleted " + rowsDeleted + " row(s) with marks < 40.");
            }

        } catch (SQLException e) {
            System.err.println("❌ Database error: " + e.getMessage());
            e.printStackTrace();
        }
        // Connection is auto-closed here by try-with-resources
    }
}
```

### 5.3 How the program flows

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                         JdbcDemo.main()                         │
  └─────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
                DriverManager.getConnection(url, user, pwd)
                                │
                   ┌────────────┴────────────┐
                   │  MySQL Driver found     │
                   │  TCP socket opened      │
                   │  Authentication done    │
                   └────────────┬────────────┘
                                │
                                ▼
                    Connection object returned
                                │
           ┌────────────────────┼─────────────────────┐
           ▼                    ▼                      ▼
     createStatement()   prepareStatement(sql)   prepareStatement(sql)
           │                    │                      │
           ▼                    ▼                      ▼
     execute(DDL)        executeUpdate()         executeQuery()
     (CREATE TABLE)      (INSERT/UPDATE/         (SELECT → ResultSet)
                          DELETE → int)                │
                                                       ▼
                                                 rs.next() loop
                                                 (read row by row)
                                                       │
                                                       ▼
                                              connection.close()
                                           (TCP socket closed)
```

---

## 6. Statement vs PreparedStatement

| Feature | Statement | PreparedStatement |
|---|---|---|
| SQL | Written as plain string | Uses `?` placeholders |
| SQL Injection | ❌ Vulnerable | ✅ Safe (parameters are escaped) |
| Performance | Compiled every time | Compiled once, executed many times |
| Use case | DDL (CREATE, DROP) | DML (INSERT, UPDATE, DELETE, SELECT) |

### SQL Injection Example — Why PreparedStatement matters

```java
// ❌ DANGEROUS — Statement with string concatenation
String name = "'; DROP TABLE students; --";
String sql = "SELECT * FROM students WHERE name = '" + name + "'";
// This becomes: SELECT * FROM students WHERE name = ''; DROP TABLE students; --'
// YOUR TABLE IS GONE!

// ✅ SAFE — PreparedStatement
String sql = "SELECT * FROM students WHERE name = ?";
PreparedStatement pstmt = connection.prepareStatement(sql);
pstmt.setString(1, name);  // The driver escapes the value, no injection possible
```

---

## 7. Problems with Raw JDBC

While JDBC works, it has **major pain points** in real applications:

| Problem | Description |
|---|---|
| **Too much boilerplate** | Open connection, create statement, set params, execute, iterate ResultSet, close — for EVERY query |
| **Manual mapping** | You must manually convert `ResultSet` rows → Java objects |
| **No caching** | Same query runs again and again, no first-level cache |
| **Connection management** | You must manage opening/closing yourself (risk of leaks) |
| **No relationships** | Joining tables and mapping `@OneToMany`, `@ManyToOne` is all manual |
| **Database-specific SQL** | If you switch from MySQL to PostgreSQL, your SQL may break |

> This is where **JPA** and **Spring Data JPA** come in!

---

## 8. What is JPA?

**JPA** = **Java Persistence API**

JPA is a **specification** (a set of interfaces and annotations) that defines how Java objects should be mapped to database tables. JPA itself is **not an implementation** — it is a **contract**.

### JPA Implementations (the ones that do the actual work)

| Implementation | Notes |
|---|---|
| **Hibernate** | Most popular, default in Spring Boot |
| EclipseLink | Reference implementation |
| OpenJPA | Apache project |

> **Spring Boot uses Hibernate as the default JPA implementation.**

### Core JPA Concepts

| Concept | What it does |
|---|---|
| `@Entity` | Marks a Java class as a database table |
| `@Table` | Specifies the table name (optional if class name = table name) |
| `@Id` | Marks the primary key field |
| `@GeneratedValue` | Auto-generate primary key (like AUTO_INCREMENT) |
| `@Column` | Map a field to a specific column (optional if names match) |
| `EntityManager` | JPA's interface for CRUD operations (like JDBC's Connection + Statement combined) |
| `JPQL` | Java Persistence Query Language — SQL-like but works on **entities**, not tables |

### What JPA Provides vs What Hibernate Provides

JPA is the **specification** (interfaces + annotations). Hibernate is the **implementation** (actual working code). Here's a clear breakdown with code examples:

```
┌──────────────────────────────────────────────────────────────────┐
│                     JPA (Specification)                          │
│                                                                  │
│   "I DEFINE the rules — what methods exist, what annotations     │
│    mean, what behavior should happen. I am just interfaces."     │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                  HIBERNATE (Implementation)                      │
│                                                                  │
│   "I IMPLEMENT those rules — I write the actual code that        │
│    makes @Entity, EntityManager, JPQL etc. actually work."       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### JPA Interface Methods → Hibernate's Implementation (Under the Hood)

JPA defines **interfaces with method signatures** — it says "these methods MUST exist". Hibernate writes the **actual code** inside those methods. Here's exactly what that looks like:

---

**1. `EntityManagerFactory` (JPA) → implemented by `SessionFactoryImpl` (Hibernate)**

```java
// ╔═══════════════════════════════════════════════════════════════╗
// ║  JPA defines this INTERFACE (no code, just method signatures) ║
// ╚═══════════════════════════════════════════════════════════════╝

package jakarta.persistence;

public interface EntityManagerFactory {
    EntityManager createEntityManager();           // "must exist"
    void close();                                  // "must exist"
    boolean isOpen();                              // "must exist"
    // ... more methods
}

// ╔═══════════════════════════════════════════════════════════════╗
// ║  HIBERNATE implements this interface with REAL CODE           ║
// ╚═══════════════════════════════════════════════════════════════╝

package org.hibernate.internal;

public class SessionFactoryImpl implements EntityManagerFactory {

    private final ConnectionProvider connectionProvider; // manages JDBC connections

    @Override
    public EntityManager createEntityManager() {
        // Hibernate's actual code:
        // 1. Gets a JDBC Connection from the connection pool (HikariCP)
        // 2. Creates a Session (Hibernate's EntityManager)
        // 3. Sets up first-level cache (persistence context)
        return new SessionImpl(this, connectionProvider.getConnection());
    }

    @Override
    public void close() {
        // Closes connection pool, releases all resources
        connectionProvider.close();
    }
}
```

---

**2. `EntityManager` (JPA) → implemented by `SessionImpl` (Hibernate)**

This is the **most important** interface — all your CRUD operations go through here:

```java
// ╔═══════════════════════════════════════════════════════════════╗
// ║  JPA defines this INTERFACE                                   ║
// ╚═══════════════════════════════════════════════════════════════╝

package jakarta.persistence;

public interface EntityManager {
    void persist(Object entity);                    // INSERT
    <T> T find(Class<T> entityClass, Object id);    // SELECT by ID
    <T> T merge(T entity);                          // UPDATE
    void remove(Object entity);                     // DELETE
    EntityTransaction getTransaction();             // transaction control
    Query createQuery(String jpql);                 // JPQL query
    void close();                                   // close session
    // ... more methods
}
```

Now here's how **Hibernate implements each method** using JDBC internally:

```java
// ╔═══════════════════════════════════════════════════════════════╗
// ║  HIBERNATE's SessionImpl — the REAL CODE behind EntityManager ║
// ╚═══════════════════════════════════════════════════════════════╝

package org.hibernate.internal;

public class SessionImpl implements EntityManager {

    private Connection jdbcConnection;           // plain JDBC Connection!
    private Map<Object, Object> persistenceContext; // first-level cache

    // ─────────────────────────────────────────────────────────
    //  persist() — JPA says "save entity", Hibernate does this:
    // ─────────────────────────────────────────────────────────
    @Override
    public void persist(Object entity) {
        // Step 1: Read @Entity, @Table, @Column annotations
        //         to know table name and column names
        String table = entity.getClass().getAnnotation(Table.class).name();
        // → "students"

        // Step 2: Read field values from the entity object
        String name = getFieldValue(entity, "name");   // "Rahul"
        String email = getFieldValue(entity, "email");  // "rahul@example.com"

        // Step 3: Generate INSERT SQL
        String sql = "INSERT INTO students (name, email, marks) VALUES (?, ?, ?)";

        // Step 4: Use JDBC PreparedStatement (same as raw JDBC!)
        PreparedStatement pstmt = jdbcConnection.prepareStatement(sql);
        pstmt.setString(1, name);
        pstmt.setString(2, email);
        pstmt.setInt(3, marks);
        pstmt.executeUpdate();     // ← THIS IS JDBC!

        // Step 5: Put entity in first-level cache
        persistenceContext.put(entity.getId(), entity);
    }

    // ─────────────────────────────────────────────────────────
    //  find() — JPA says "find by ID", Hibernate does this:
    // ─────────────────────────────────────────────────────────
    @Override
    public <T> T find(Class<T> entityClass, Object id) {
        // Step 1: Check first-level cache first!
        T cached = (T) persistenceContext.get(id);
        if (cached != null) {
            return cached;   // cache HIT — no DB call! 🚀
        }

        // Step 2: Cache miss — generate SELECT SQL
        String sql = "SELECT id, name, email, marks FROM students WHERE id = ?";

        // Step 3: Use JDBC PreparedStatement
        PreparedStatement pstmt = jdbcConnection.prepareStatement(sql);
        pstmt.setObject(1, id);
        ResultSet rs = pstmt.executeQuery();   // ← THIS IS JDBC!

        // Step 4: Map ResultSet → Java object (Hibernate does this for you!)
        if (rs.next()) {
            T entity = createEntityFromResultSet(rs, entityClass);
            // → new Student(1, "Rahul", "rahul@example.com", 85)

            // Step 5: Put in cache for next time
            persistenceContext.put(id, entity);
            return entity;
        }
        return null;
    }

    // ─────────────────────────────────────────────────────────
    //  merge() — JPA says "update entity", Hibernate does this:
    // ─────────────────────────────────────────────────────────
    @Override
    public <T> T merge(T entity) {
        // Step 1: Compare entity fields with cached version (dirty checking)
        T original = (T) persistenceContext.get(entity.getId());

        // Step 2: Find what changed
        // original.name = "Rahul"  →  entity.name = "Rahul Updated"
        // original.marks = 85      →  entity.marks = 85  (no change)

        // Step 3: Generate UPDATE SQL only for changed fields
        String sql = "UPDATE students SET name = ? WHERE id = ?";

        // Step 4: Use JDBC PreparedStatement
        PreparedStatement pstmt = jdbcConnection.prepareStatement(sql);
        pstmt.setString(1, "Rahul Updated");
        pstmt.setObject(2, entity.getId());
        pstmt.executeUpdate();   // ← THIS IS JDBC!

        // Step 5: Update cache
        persistenceContext.put(entity.getId(), entity);
        return entity;
    }

    // ─────────────────────────────────────────────────────────
    //  remove() — JPA says "delete entity", Hibernate does this:
    // ─────────────────────────────────────────────────────────
    @Override
    public void remove(Object entity) {
        // Step 1: Generate DELETE SQL
        String sql = "DELETE FROM students WHERE id = ?";

        // Step 2: Use JDBC PreparedStatement
        PreparedStatement pstmt = jdbcConnection.prepareStatement(sql);
        pstmt.setObject(1, entity.getId());
        pstmt.executeUpdate();   // ← THIS IS JDBC!

        // Step 3: Remove from cache
        persistenceContext.remove(entity.getId());
    }
}
```

> **Key takeaway**: Every `EntityManager` method that Hibernate implements eventually creates a `PreparedStatement` and calls `executeUpdate()` or `executeQuery()` — **it's JDBC all the way down!**

---

**3. `EntityTransaction` (JPA) → implemented by `TransactionImpl` (Hibernate)**

```java
// ╔═══════════════════════════════════════════════════════════════╗
// ║  JPA defines this INTERFACE                                   ║
// ╚═══════════════════════════════════════════════════════════════╝

package jakarta.persistence;

public interface EntityTransaction {
    void begin();       // start transaction
    void commit();      // save changes to DB
    void rollback();    // undo changes
    boolean isActive(); // is transaction running?
}

// ╔═══════════════════════════════════════════════════════════════╗
// ║  HIBERNATE implements it — maps directly to JDBC transactions ║
// ╚═══════════════════════════════════════════════════════════════╝

package org.hibernate.internal;

public class TransactionImpl implements EntityTransaction {

    private Connection jdbcConnection;

    @Override
    public void begin() {
        jdbcConnection.setAutoCommit(false);   // ← JDBC method!
    }

    @Override
    public void commit() {
        jdbcConnection.commit();               // ← JDBC method!
        jdbcConnection.setAutoCommit(true);
    }

    @Override
    public void rollback() {
        jdbcConnection.rollback();             // ← JDBC method!
        jdbcConnection.setAutoCommit(true);
    }
}
```

---

**4. `Query` (JPA) → implemented by `QueryImpl` (Hibernate)**

```java
// ╔═══════════════════════════════════════════════════════════════╗
// ║  JPA defines this INTERFACE                                   ║
// ╚═══════════════════════════════════════════════════════════════╝

package jakarta.persistence;

public interface TypedQuery<T> {
    TypedQuery<T> setParameter(String name, Object value);
    List<T> getResultList();            // execute SELECT, return list
    T getSingleResult();                // execute SELECT, return one
    int executeUpdate();                // execute UPDATE/DELETE
}

// ╔═══════════════════════════════════════════════════════════════╗
// ║  HIBERNATE implements it                                      ║
// ╚═══════════════════════════════════════════════════════════════╝

package org.hibernate.query.internal;

public class QueryImpl<T> implements TypedQuery<T> {

    private String jpql;                        // "SELECT s FROM Student s WHERE s.marks > :min"
    private Map<String, Object> parameters;
    private Connection jdbcConnection;

    @Override
    public TypedQuery<T> setParameter(String name, Object value) {
        parameters.put(name, value);            // store param for later
        return this;
    }

    @Override
    public List<T> getResultList() {
        // Step 1: Parse JPQL and convert to SQL
        // JPQL: "SELECT s FROM Student s WHERE s.marks > :min"
        //   ↓ Hibernate translates ↓
        // SQL:  "SELECT id, name, email, marks FROM students WHERE marks > ?"
        String sql = jpqlToSqlTranslator.translate(jpql);

        // Step 2: Use JDBC PreparedStatement
        PreparedStatement pstmt = jdbcConnection.prepareStatement(sql);
        pstmt.setObject(1, parameters.get("min"));
        ResultSet rs = pstmt.executeQuery();     // ← JDBC!

        // Step 3: Map each ResultSet row → entity object
        List<T> results = new ArrayList<>();
        while (rs.next()) {                      // ← JDBC ResultSet loop!
            T entity = mapRowToEntity(rs);
            results.add(entity);
        }
        return results;
    }
}
```

---

#### The Complete Picture — JPA Interface → Hibernate Class → JDBC Method

```
┌─────────────────────────┬──────────────────────────┬──────────────────────────────┐
│     JPA Interface       │   Hibernate Implements   │    JDBC Used Internally      │
├─────────────────────────┼──────────────────────────┼──────────────────────────────┤
│ EntityManagerFactory    │ SessionFactoryImpl       │ ConnectionProvider (pool)    │
│   .createEntityManager()│   .openSession()         │   .getConnection()           │
├─────────────────────────┼──────────────────────────┼──────────────────────────────┤
│ EntityManager           │ SessionImpl              │ Connection                   │
│   .persist(entity)      │   .save(entity)          │   PreparedStatement          │
│   .find(Class, id)      │   .get(Class, id)        │   .executeQuery() → ResultSet│
│   .merge(entity)        │   .update(entity)        │   .executeUpdate()           │
│   .remove(entity)       │   .delete(entity)        │   .executeUpdate()           │
├─────────────────────────┼──────────────────────────┼──────────────────────────────┤
│ EntityTransaction       │ TransactionImpl          │ Connection                   │
│   .begin()              │   .begin()               │   .setAutoCommit(false)      │
│   .commit()             │   .commit()              │   .commit()                  │
│   .rollback()           │   .rollback()            │   .rollback()                │
├─────────────────────────┼──────────────────────────┼──────────────────────────────┤
│ TypedQuery<T>           │ QueryImpl<T>             │ PreparedStatement            │
│   .setParameter(n, v)   │   stores params          │   .setObject(index, value)   │
│   .getResultList()      │   JPQL → SQL translator  │   .executeQuery() → ResultSet│
│   .executeUpdate()      │   generates DML SQL      │   .executeUpdate()           │
└─────────────────────────┴──────────────────────────┴──────────────────────────────┘
```

> **In simple words**: JPA says **"what"** (persist, find, merge, remove). Hibernate decides **"how"** (generate SQL, use PreparedStatement, map ResultSet). JDBC does the **"actual work"** (send bytes over TCP to the database).

---

#### What JPA Gives You (from `jakarta.persistence` package)

These are **standard** — they work with ANY JPA implementation (Hibernate, EclipseLink, etc.):

```java
// ═══════════════════════════════════════════════════════════
//  1. ANNOTATIONS — JPA defines these (jakarta.persistence.*)
// ═══════════════════════════════════════════════════════════

import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Column;
import jakarta.persistence.OneToMany;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.JoinColumn;

@Entity                                          // JPA annotation
@Table(name = "students")                        // JPA annotation
public class Student {

    @Id                                          // JPA annotation
    @GeneratedValue(strategy = GenerationType.IDENTITY) // JPA annotation
    private int id;

    @Column(nullable = false)                    // JPA annotation
    private String name;

    @ManyToOne                                   // JPA annotation
    @JoinColumn(name = "dept_id")                // JPA annotation
    private Department department;

    // getters, setters...
}
```

```java
// ═══════════════════════════════════════════════════════════
//  2. EntityManager — JPA's core interface for DB operations
// ═══════════════════════════════════════════════════════════

import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;

// Create EntityManagerFactory from persistence.xml (JPA standard config)
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");

// Create EntityManager (like getting a Connection in JDBC)
EntityManager em = emf.createEntityManager();

// ──── INSERT ────
em.getTransaction().begin();          // JPA method
em.persist(new Student("Rahul"));     // JPA method — saves to DB
em.getTransaction().commit();         // JPA method

// ──── FIND BY ID ────
Student s = em.find(Student.class, 1);  // JPA method — SELECT WHERE id=1

// ──── UPDATE ────
em.getTransaction().begin();
s.setName("Rahul Updated");           // Just change the object!
em.getTransaction().commit();          // JPA auto-detects change → UPDATE SQL

// ──── DELETE ────
em.getTransaction().begin();
em.remove(s);                          // JPA method — DELETE FROM students
em.getTransaction().commit();

em.close();
emf.close();
```

```java
// ═══════════════════════════════════════════════════════════
//  3. JPQL — JPA's Query Language (works on entities, not tables)
// ═══════════════════════════════════════════════════════════

import jakarta.persistence.TypedQuery;

// JPQL uses CLASS NAMES (Student), not TABLE NAMES (students)
// JPQL uses FIELD NAMES (name), not COLUMN NAMES
TypedQuery<Student> query = em.createQuery(
    "SELECT s FROM Student s WHERE s.marks > :minMarks", Student.class
);
query.setParameter("minMarks", 80);
List<Student> toppers = query.getResultList();  // JPA handles mapping to objects

// Named Query (define once, reuse)
// On the entity class:
// @NamedQuery(name = "Student.findToppers",
//             query = "SELECT s FROM Student s WHERE s.marks > :min")
TypedQuery<Student> q = em.createNamedQuery("Student.findToppers", Student.class);
```

```java
// ═══════════════════════════════════════════════════════════
//  4. persistence.xml — JPA's standard configuration file
// ═══════════════════════════════════════════════════════════
```

```xml
<!-- META-INF/persistence.xml — JPA standard config -->
<persistence xmlns="https://jakarta.ee/xml/ns/persistence" version="3.0">
    <persistence-unit name="myPU">
        <class>com.example.Student</class>
        <properties>
            <property name="jakarta.persistence.jdbc.url"
                      value="jdbc:mysql://localhost:3306/mydb"/>
            <property name="jakarta.persistence.jdbc.user"     value="root"/>
            <property name="jakarta.persistence.jdbc.password" value="root"/>
        </properties>
    </persistence-unit>
</persistence>
```

> **Notice**: Everything above uses `jakarta.persistence.*` — this is **pure JPA**. If you swap Hibernate for EclipseLink, this code still works **without any changes**.

---

#### What Hibernate Gives You (on top of JPA)

These are **Hibernate-specific** extras — they only work with Hibernate, not other JPA providers:

```java
// ═══════════════════════════════════════════════════════════
//  1. Session & SessionFactory — Hibernate's own API
//     (existed BEFORE JPA — Hibernate came first!)
// ═══════════════════════════════════════════════════════════

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

// Hibernate's own way to configure (not JPA standard)
SessionFactory sessionFactory = new Configuration()
    .configure("hibernate.cfg.xml")  // Hibernate-specific config file
    .addAnnotatedClass(Student.class)
    .buildSessionFactory();

// Session ≈ EntityManager (Hibernate's version)
Session session = sessionFactory.openSession();

session.beginTransaction();
session.save(new Student("Priya"));       // Hibernate method (not JPA)
session.getTransaction().commit();

Student s = session.get(Student.class, 1); // Hibernate method (not JPA)

session.close();
sessionFactory.close();
```

```java
// ═══════════════════════════════════════════════════════════
//  2. HQL — Hibernate Query Language (Hibernate's version of JPQL)
//     HQL came first, then JPA modeled JPQL after it
// ═══════════════════════════════════════════════════════════

import org.hibernate.query.Query;

// HQL looks very similar to JPQL but has Hibernate-specific features
Query<Student> query = session.createQuery(
    "FROM Student s WHERE s.marks > :min", Student.class
);
query.setParameter("min", 80);
List<Student> results = query.getResultList();
```

```java
// ═══════════════════════════════════════════════════════════
//  3. Hibernate-Specific Annotations (extras beyond JPA)
// ═══════════════════════════════════════════════════════════

import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;
import org.hibernate.annotations.Formula;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@SQLDelete(sql = "UPDATE students SET deleted = true WHERE id = ?")  // Hibernate only — soft delete
@Where(clause = "deleted = false")                                    // Hibernate only — auto-filter
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)                   // Hibernate only — 2nd level cache
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;

    private String name;
    private int marks;
    private boolean deleted = false;

    @CreationTimestamp       // Hibernate only — auto-set when created
    private LocalDateTime createdAt;

    @UpdateTimestamp         // Hibernate only — auto-set when updated
    private LocalDateTime updatedAt;

    @Formula("(SELECT AVG(s.marks) FROM students s)")  // Hibernate only — computed column
    private double averageMarks;
}
```

```java
// ═══════════════════════════════════════════════════════════
//  4. Second-Level Cache — Hibernate extra (JPA has NO caching spec)
// ═══════════════════════════════════════════════════════════

// First-level cache  → per EntityManager/Session (automatic, both JPA & Hibernate)
// Second-level cache → shared across sessions (Hibernate only, needs config)

// In application.properties:
// spring.jpa.properties.hibernate.cache.use_second_level_cache=true
// spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory

// On the entity:
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Hibernate only
public class Student { ... }

// Now when two different sessions load Student id=1,
// the second one gets it from cache — no DB hit!
```

---

#### Summary Table — JPA vs Hibernate

| Feature | JPA (Specification) | Hibernate (Implementation) |
|---|---|---|
| **Package** | `jakarta.persistence.*` | `org.hibernate.*` |
| **Config file** | `persistence.xml` | `hibernate.cfg.xml` |
| **Core interface** | `EntityManager` | `Session` (extends EntityManager) |
| **Factory** | `EntityManagerFactory` | `SessionFactory` |
| **Query language** | JPQL | HQL (superset of JPQL) |
| **Save** | `em.persist(entity)` | `session.save(entity)` |
| **Find** | `em.find(Class, id)` | `session.get(Class, id)` |
| **Delete** | `em.remove(entity)` | `session.delete(entity)` |
| **Annotations** | `@Entity, @Id, @Column, @OneToMany` | `@CreationTimestamp, @Formula, @SQLDelete, @Where, @Cache` |
| **Caching** | First-level only (per EntityManager) | First-level + **Second-level** (shared) |
| **Soft delete** | ❌ Not supported | ✅ `@SQLDelete` + `@Where` |
| **Auto timestamps** | ❌ Not built-in | ✅ `@CreationTimestamp`, `@UpdateTimestamp` |
| **Computed columns** | ❌ Not supported | ✅ `@Formula` |
| **Portable?** | ✅ Works with any JPA provider | ❌ Locked to Hibernate |

> **Best practice in Spring Boot**: Use **JPA annotations** (`jakarta.persistence.*`) for everything you can. Only use Hibernate-specific features (`org.hibernate.*`) when JPA doesn't provide what you need (like `@CreationTimestamp`, `@Formula`, second-level cache).

---

## 9. How Spring Boot JPA Works on Top of JDBC

### The Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR CODE                            │
│              studentRepository.findAll()                │
└─────────────────────┬───────────────────────────────────┘
                      │  calls
                      ▼
┌─────────────────────────────────────────────────────────┐
│              SPRING DATA JPA                            │
│    (auto-generates implementation of your Repository    │
│     interface at runtime — you write ZERO SQL!)         │
└─────────────────────┬───────────────────────────────────┘
                      │  delegates to
                      ▼
┌─────────────────────────────────────────────────────────┐
│                 JPA (Hibernate)                         │
│    - EntityManager manages entities                     │
│    - Translates method calls → JPQL → SQL               │
│    - First-level cache (don't hit DB twice for same id) │
│    - Dirty checking (auto-detect changes, auto-UPDATE)  │
└─────────────────────┬───────────────────────────────────┘
                      │  uses internally
                      ▼
┌─────────────────────────────────────────────────────────┐
│                      JDBC                               │
│    - Connection, PreparedStatement, ResultSet           │
│    - Hibernate generates SQL and executes it via JDBC   │
│    - Connection pooling via HikariCP (default)          │
└─────────────────────┬───────────────────────────────────┘
                      │  talks to
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    DATABASE                             │
│              (MySQL / PostgreSQL / H2)                  │
└─────────────────────────────────────────────────────────┘
```

> **Key insight**: Spring Data JPA → calls Hibernate (JPA) → which internally uses **JDBC** to talk to the database. **JDBC is always at the bottom!**

---

## 10. Spring Boot JPA — Complete Example

### 10.1 Dependencies (pom.xml)

```xml
<dependencies>
    <!-- Spring Data JPA — includes Hibernate + JDBC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- MySQL Driver (the JDBC driver) -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 10.2 application.properties

```properties
# JDBC connection details — same URL, user, password as raw JDBC!
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=root

# Tell Hibernate to auto-create/update tables based on @Entity classes
spring.jpa.hibernate.ddl-auto=update

# Show the SQL that Hibernate generates (great for learning)
spring.jpa.show-sql=true

# HikariCP connection pool settings (Spring Boot uses HikariCP by default)
spring.datasource.hikari.maximum-pool-size=10
```

### 10.3 Entity Class (replaces CREATE TABLE)

```java
import jakarta.persistence.*;

@Entity                         // This class = a database table
@Table(name = "students")      // Table name in the database
public class Student {

    @Id                                              // Primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY) // AUTO_INCREMENT
    private int id;

    @Column(nullable = false, length = 100)          // NOT NULL, VARCHAR(100)
    private String name;

    @Column(nullable = false, length = 100)
    private String email;

    private int marks;                               // column name = "marks" by default

    // No-arg constructor required by JPA
    public Student() {}

    public Student(String name, String email, int marks) {
        this.name = name;
        this.email = email;
        this.marks = marks;
    }

    // Getters and Setters
    public int getId()        { return id; }
    public String getName()   { return name; }
    public String getEmail()  { return email; }
    public int getMarks()     { return marks; }

    public void setId(int id)          { this.id = id; }
    public void setName(String name)   { this.name = name; }
    public void setEmail(String email) { this.email = email; }
    public void setMarks(int marks)    { this.marks = marks; }

    @Override
    public String toString() {
        return "Student{id=" + id + ", name='" + name + "', email='" + email + "', marks=" + marks + "}";
    }
}
```

### 10.4 Repository Interface (replaces ALL your JDBC code!)

```java
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

// JpaRepository<EntityType, PrimaryKeyType>
// Spring auto-generates the implementation at runtime!
public interface StudentRepository extends JpaRepository<Student, Integer> {

    // Spring Data JPA reads the method name and auto-generates the query!
    // findByName → SELECT * FROM students WHERE name = ?
    List<Student> findByName(String name);

    // findByMarksGreaterThan → SELECT * FROM students WHERE marks > ?
    List<Student> findByMarksGreaterThan(int marks);

    // findByEmailContaining → SELECT * FROM students WHERE email LIKE '%value%'
    List<Student> findByEmailContaining(String keyword);
}
```

### 10.5 Using the Repository (Service / Controller)

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
public class StudentRunner implements CommandLineRunner {

    @Autowired
    private StudentRepository studentRepository;

    @Override
    public void run(String... args) {

        // ──────── INSERT (replaces PreparedStatement + executeUpdate) ────────
        Student s1 = new Student("Rahul Sharma", "rahul@example.com", 85);
        Student s2 = new Student("Priya Singh",  "priya@example.com", 92);
        studentRepository.save(s1);       // Hibernate generates: INSERT INTO students ...
        studentRepository.save(s2);

        // ──────── SELECT ALL (replaces Statement + ResultSet loop) ────────
        List<Student> all = studentRepository.findAll();
        // Hibernate generates: SELECT * FROM students
        all.forEach(System.out::println);

        // ──────── FIND BY ID ────────
        Student found = studentRepository.findById(1).orElse(null);
        // Hibernate generates: SELECT * FROM students WHERE id = ?

        // ──────── UPDATE (just save again — Hibernate detects change) ────────
        if (found != null) {
            found.setMarks(95);
            studentRepository.save(found);
            // Hibernate generates: UPDATE students SET marks=95 WHERE id=1
        }

        // ──────── DELETE ────────
        studentRepository.deleteById(2);
        // Hibernate generates: DELETE FROM students WHERE id = 2

        // ──────── CUSTOM QUERY (auto-generated from method name!) ────────
        List<Student> toppers = studentRepository.findByMarksGreaterThan(90);
        toppers.forEach(System.out::println);
    }
}
```

---

## 11. Side-by-Side Comparison — JDBC vs Spring Data JPA

| Operation | Raw JDBC | Spring Data JPA |
|---|---|---|
| **Setup connection** | `DriverManager.getConnection(url, user, pwd)` | Just set `spring.datasource.*` in properties |
| **Connection pooling** | You manage manually (or add HikariCP yourself) | Auto-configured by Spring Boot (HikariCP) |
| **Create table** | Write `CREATE TABLE` SQL manually | `@Entity` + `ddl-auto=update` — auto-created |
| **Insert** | `PreparedStatement` + `setString()` + `executeUpdate()` | `repository.save(entity)` — one line |
| **Select** | `Statement` + `executeQuery()` + `ResultSet` loop | `repository.findAll()` — one line |
| **Find by ID** | Write SQL + iterate `ResultSet` + map to object | `repository.findById(id)` — one line |
| **Update** | Write `UPDATE` SQL + `PreparedStatement` | Change the entity + `repository.save()` |
| **Delete** | Write `DELETE` SQL + `PreparedStatement` | `repository.deleteById(id)` — one line |
| **Custom query** | Write full SQL yourself | Just name the method: `findByNameAndMarksGreaterThan()` |
| **Close resources** | You must close `ResultSet`, `Statement`, `Connection` | Spring handles everything |
| **SQL injection** | Risk if using `Statement` | Not possible (parameterized internally) |

---

## 12. How Hibernate (JPA) Uses JDBC Internally

When you call `studentRepository.save(student)`, here's what happens **under the hood**:

```
1. Spring Data JPA calls → Hibernate's EntityManager.persist(student)

2. Hibernate checks its first-level cache
   → Is this entity already tracked? If yes, just mark as dirty.
   → If no, generate INSERT SQL.

3. Hibernate generates SQL:
   INSERT INTO students (name, email, marks) VALUES (?, ?, ?)

4. Hibernate gets a Connection from HikariCP connection pool
   (NOT DriverManager — pool reuses existing connections!)

5. Hibernate creates a PreparedStatement:
   PreparedStatement pstmt = connection.prepareStatement(sql);
   pstmt.setString(1, "Rahul Sharma");
   pstmt.setString(2, "rahul@example.com");
   pstmt.setInt(3, 85);

6. Hibernate calls pstmt.executeUpdate()

7. Connection is returned to the pool (NOT closed!)

8. Transaction is committed (if @Transactional)
```

> **So Hibernate is literally writing the same JDBC code you would write — but automatically!**

---

## 13. Connection Pooling — HikariCP

In raw JDBC you open and close connections every time (expensive!). Spring Boot uses **HikariCP** by default — a connection pool.

```
                    ┌──────────────────────┐
                    │    HikariCP Pool     │
                    │                      │
   request 1  ──►  │  conn1 ✅ (in use)   │
   request 2  ──►  │  conn2 ✅ (in use)   │
   request 3  ──►  │  conn3 ✅ (in use)   │
                    │  conn4 💤 (idle)     │
                    │  conn5 💤 (idle)     │
                    └──────────────────────┘
                              │
                              ▼
                         Database

- Connections are created once and REUSED
- When your code needs a connection, it borrows one from the pool
- When done, the connection goes BACK to the pool (not closed)
- Default pool size: 10 connections
```

---

## 14. Summary — The Full Picture

```
┌────────────────────────────────────────────────────────────┐
│                    YOU WRITE THIS                          │
│                                                            │
│   @Entity class + Repository interface + properties file   │
│   (That's it! Maybe 30 lines of code)                     │
└──────────────────────────┬─────────────────────────────────┘
                           │
          Spring Data JPA auto-generates implementation
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│              HIBERNATE (JPA Implementation)                │
│                                                            │
│   - Reads @Entity → knows table structure                  │
│   - Reads method names → generates JPQL → generates SQL    │
│   - Manages first-level cache                              │
│   - Tracks dirty entities → auto-generates UPDATE SQL      │
│   - Manages transactions                                   │
└──────────────────────────┬─────────────────────────────────┘
                           │
               Uses JDBC internally
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│                     JDBC + HikariCP                        │
│                                                            │
│   - Connection pool manages DB connections                 │
│   - PreparedStatement executes parameterized SQL           │
│   - ResultSet returns query results                        │
│   - All the low-level DB communication                     │
└──────────────────────────┬─────────────────────────────────┘
                           │
                     TCP/IP Socket
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│                      DATABASE                              │
│                  (MySQL / PostgreSQL / H2)                  │
└────────────────────────────────────────────────────────────┘
```

### In one line:

> **Spring Data JPA** makes your life easy by auto-generating code → **Hibernate** translates your entities into SQL → **JDBC** sends that SQL to the database over a network connection. **JDBC is always the foundation.**
