
---

# How do DB and springBoot they communicate?

Using a **network connection** (usually TCP/IP).

Just like when you open a website:

```text
Browser
    |
    | Internet
    |
Google Server
```

Similarly:

```text
Spring Boot
    |
    | TCP Connection
    |
MySQL
```

That TCP connection is what we commonly call a **database connection**.

---

# What happens when a connection is created?

Suppose your application starts.

You execute:

```java
Connection conn =
DriverManager.getConnection(
    url,
    username,
    password
);
```

Java contacts MySQL.

```
Hello MySQL

↓

Username: root
Password: ****

↓

Authentication successful
```

MySQL replies:

```
Connection Established
```

Now both sides keep a socket open.

```
Spring Boot
      |
      |  Connection
      |
MySQL Server
```

This connection remains alive until you close it.

---

# What travels through this connection?

Everything.

For example:

```java
SELECT * FROM employee;
```

Spring sends:

```
SELECT * FROM employee;
```

through the connection.

MySQL executes it.

Returns:

```
id  name

1   Mohit
2   Rahul
```

Same connection.

---




```
Open Connection

↓

Send SQL

↓

Receive Results

↓

Close Connection
```

---

# Why not open a connection every time?

Suppose one request comes.

```
Open Connection

↓

Authenticate

↓

Create TCP Socket

↓

Execute SQL

↓

Close Connection
```

Opening a database connection is **expensive**.

It involves:

- TCP handshake
- Authentication
- Memory allocation
- Session creation inside the database

This can take milliseconds, which is slow compared to executing many simple queries.

---

# That's why Connection Pools exist

Instead of:

```
Request 1

↓

Open Connection

↓

Close Connection
```

Spring Boot creates several connections when the application starts.

```
MySQL

↑
│
├── Connection 1
├── Connection 2
├── Connection 3
├── Connection 4
├── Connection 5
```

This collection is called a **Connection Pool**.

---

# Request Flow

Suppose a request arrives.

```
HTTP Request

↓

Need Database

↓

Take Connection #2

↓

Execute SQL

↓

Return Connection #2
```

Notice:

It is **not closed**.

It goes back into the pool.

```
Pool

Connection 1

Connection 2  ← returned

Connection 3

Connection 4
```

Another request may reuse it.

---

# Where does Hibernate fit?

Hibernate never directly talks to MySQL.

```
Your Code

↓

Hibernate

↓

JDBC

↓

Connection

↓

MySQL
```

---

Example:

```java
employeeRepository.findById(1);
```

Hibernate eventually does something like:

```java
Connection conn = dataSource.getConnection();

PreparedStatement ps =
conn.prepareStatement(
"SELECT * FROM employee WHERE id=?"
);

ResultSet rs = ps.executeQuery();
```

---

# What is DataSource?

Instead of creating connections:

```java
DriverManager.getConnection(...)
```

Spring injects:

```java
DataSource
```

Think of it as:

```
Connection Factory
```

Actually:

```
Connection Pool

↓

Give me one connection

↓

Return one connection
```

---

In Spring Boot usually:

```java
@Autowired
DataSource dataSource;
```

When Hibernate needs a connection:

```
Connection c =
dataSource.getConnection();
```

It receives one from **HikariCP** (Spring Boot's default connection pool).

---

## HirakiCP

How many connectiosn HirakiCp has??

> **It depends on your configuration.** HikariCP doesn't have a fixed number of connections.

By default in Spring Boot:

```properties
spring.datasource.hikari.maximum-pool-size=10
```

So if you don't configure anything, **HikariCP creates a pool with a maximum of 10 database connections**.

---

# Example

Suppose your application starts.

HikariCP creates (or gradually creates as needed) connections up to the configured maximum.

```text
HikariCP Pool

Connection 1
Connection 2
Connection 3
...
Connection 10
```

Now 5 users send requests simultaneously.

```text
Request 1 → Connection 1

Request 2 → Connection 2

Request 3 → Connection 3

Request 4 → Connection 4

Request 5 → Connection 5
```

The remaining connections are still available.

---

# What if 20 users come at the same time?

Suppose:

```text
Maximum Pool Size = 10
```

```text
20 Requests

↓

10 connections available

↓

10 requests get connections immediately

↓

10 requests wait
```

When one request finishes:

```text
Connection 3 returned

↓

Waiting request gets Connection 3
```

Connections are **reused**, not recreated.

---

# What if all connections are busy?

Hikari waits for a connection to become free.

The default timeout is:

```properties
spring.datasource.hikari.connection-timeout=30000
```

which means **30 seconds**.

If no connection becomes available within 30 seconds:

```text
SQLTransientConnectionException:
Connection is not available,
request timed out after 30000ms
```

---

# Can we change it?

Yes.

```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

Meaning:

- `maximum-pool-size=20` → At most 20 connections.
- `minimum-idle=5` → Try to keep at least 5 idle connections ready.
- `connection-timeout=30000` → Wait up to 30 seconds for a free connection.
- `idle-timeout=600000` → Close idle connections after 10 minutes (if above the minimum idle).
- `max-lifetime=1800000` → Replace a connection after 30 minutes to avoid stale connections.

---

# Does Hikari create all 10 connections immediately?

**Not necessarily.**

Suppose:

```properties
maximum-pool-size=10
minimum-idle=2
```

At startup:

```text
Connection 1

Connection 2
```

Only two connections may exist.

If traffic increases:

```text
Request 3

↓

Create Connection 3
```

Later:

```text
Connection 4

Connection 5

...
```

Until the maximum of 10 is reached.

---

# How do you choose the pool size?

A common misconception is:

> **"More connections = better performance."**

Not always.

Suppose your database server can efficiently handle about **50 active connections**.

If you configure:

```properties
maximum-pool-size=500
```

then 500 queries may compete for the database's CPU, memory, and locks, often making performance worse rather than better.

The ideal pool size depends on:

- Database capacity
- Number of CPU cores
- Query execution time
- Expected concurrency

---

# Interview Answer

If an interviewer asks:

> **How many connections does HikariCP have?**

A good answer is:

> HikariCP doesn't have a fixed number of connections. In Spring Boot, the default `maximumPoolSize` is **10**, meaning up to 10 connections can exist in the pool. The pool grows as needed (subject to configuration) and reuses connections instead of creating a new one for each request. If all connections are busy, incoming requests wait for a free connection until the configured `connectionTimeout` is reached.

# What happens in one HTTP request?

Suppose:

```
GET /employees/1
```

Flow:

```
Request

↓

Controller

↓

Service

↓

Repository

↓

DataSource

↓

Borrow Connection

↓

SQL

↓

Return Connection

↓

Response
```

The connection is **reused**, not destroyed.

---

# Difference between Connection and EntityManager

Many beginners confuse these.

## Database Connection

```
Spring

↓

TCP Socket

↓

MySQL
```

Purpose:

> Communicate with the database.

---

---

# Interview Answer

If an interviewer asks **"What is a database connection?"**, a good answer is:

> A database connection is a live communication channel, usually a TCP socket, between an application and the database server. It allows the application to send SQL statements and receive results. Creating a connection is relatively expensive because it involves network setup and authentication, so Spring Boot typically uses a connection pool (such as HikariCP) to reuse existing connections instead of creating a new one for every request.

Once you understand database connections, the next natural topics are **connection pooling (HikariCP)** and **how `@Transactional` borrows and returns connections**, which tie together Spring, Hibernate, and the database.



![alt text](<022 jpa-1_250130_223510_250716_002509_1.jpg>) ![alt text](<022 jpa-1_250130_223510_250716_002509_2.jpg>) ![alt text](<022 jpa-1_250130_223510_250716_002509_3.jpg>) ![alt text](<022 jpa-1_250130_223510_250716_002509_4.jpg>) ![alt text](<022 jpa-1_250130_223510_250716_002509_5.jpg>) ![alt text](<022 jpa-1_250130_223510_250716_002509_6.jpg>) ![alt text](<022 jpa-1_250130_223510_250716_002509_7.jpg>)

## EntityManager

This is one of the most important concepts in JPA/Hibernate. Once you understand `EntityManager`, you'll understand:

- First-Level Cache
- Dirty Checking
- Persistence Context
- Transactions
- Lazy Loading

Let's build it from scratch.

---

# First, what is JPA?

JPA is just a **specification**.

It says:

> "There should be an object that manages entity objects."

That object is called:

```text
EntityManager
```

Hibernate provides the actual implementation.

---

# Imagine You Have This Entity

```java
@Entity
class Employee {

    @Id
    Long id;

    String name;
}
```

You do:

```java
Employee e = repository.findById(1).get();
```

Who actually loads this employee?

Not the repository.

Internally:

```text
Repository

↓

EntityManager

↓

Hibernate

↓

Database
```

The `EntityManager` is the central object that manages your entities.

---

# Think of EntityManager as a Manager

Imagine a company.

```text
Database
```

contains millions of employees.

You don't bring everyone into your office.

You only bring the ones you're currently working with.

The office is called:

```text
Persistence Context
```

The manager of that office is:

```text
EntityManager
```

Diagram:

```text
                 EntityManager
                       │
                       │ manages
                       ▼
              Persistence Context
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   Employee#1     Employee#2     Department#10
```

---

# What Happens on `findById()`?

Suppose:

```java
Employee emp = repository.findById(1).get();
```

Flow:

```text
EntityManager

↓

First-Level Cache

↓

Employee already present?
```

### If YES

Return it immediately.

No SQL.

---

### If NO

```text
EntityManager

↓

Database

↓

SELECT * FROM employee WHERE id=1

↓

Create Employee Object

↓

Store inside Persistence Context

↓

Return Employee
```

---

# Example

```java
Employee e1 = repository.findById(1).get();

Employee e2 = repository.findById(1).get();
```

SQL:

```sql
SELECT * FROM employee WHERE id=1;
```

Only one query.

Why?

Because:

```text
EntityManager

↓

Persistence Context

↓

Employee(id=1)
```

The second call returns the cached object.

---

# What Else Does EntityManager Do?

It performs all CRUD operations.

---

## Persist

```java
Employee e = new Employee();

entityManager.persist(e);
```

Meaning:

```text
Save this object.

Manage it.

Insert into database on flush/commit.
```

---

## Find

```java
Employee e =
entityManager.find(Employee.class,1L);
```

Meaning:

```text
Load Employee with id = 1
```

---

## Remove

```java
entityManager.remove(employee);
```

Meaning:

```text
DELETE FROM employee...
```

---

## Merge

```java
entityManager.merge(employee);
```

Used to attach a detached entity back to the persistence context.

---

# EntityManager Tracks Changes

Suppose:

```java
@Transactional
public void update(){

    Employee e =
        repository.findById(1).get();

    e.setName("Rahul");
}
```

You never call:

```java
repository.save(e);
```

Still database updates.

Why?

Because EntityManager is watching.

```text
EntityManager

↓

Employee Loaded

↓

Name Changed

↓

Transaction Ends

↓

UPDATE employee SET name='Rahul'
```

This is called:

```text
Dirty Checking
```

---

# EntityManager Owns the First-Level Cache

Remember:

```text
EntityManager

↓

Persistence Context

↓

First-Level Cache
```

Every managed entity lives here.

When EntityManager dies:

```text
EntityManager Closed

↓

Persistence Context Destroyed

↓

Cache Gone
```

---

##  Why not call DB directly?

the real comparison is:

```text
Using JDBC directly
vs
Using JPA/Hibernate through EntityManager
```

The `EntityManager` is useful because it does more than merely send SQL. It keeps Java objects synchronized with database rows.

---

# Basic mental model

Suppose the database contains:

### `employee`

| id | name | department_id |
|---:|---|---:|
| 1 | Rahul | 10 |

### `department`

| id | name |
|---:|---|
| 10 | Engineering |

And your entities are:

```java
@Entity
public class Employee {

    @Id
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Department department;
}
```

```java
@Entity
public class Department {

    @Id
    private Long id;

    private String name;
}
```

The database stores **rows**. Your application works with **Java objects**.

The `EntityManager` connects these two worlds:

```text
Database row
    ↓
Hibernate
    ↓
Employee Java object
    ↓
Persistence Context
    ↓
Managed by EntityManager
```

---

# 1. First-level cache

Every `EntityManager` has its own persistence context, also called the **first-level cache**.

Conceptually, it contains a map like this:

```text
(Entity type, primary key) → Java object

(Employee, 1)   → Employee object
(Department, 10) → Department object
```

## First lookup

```java
Employee e1 = entityManager.find(Employee.class, 1L);
```

The `EntityManager` first checks:

```text
Is Employee with ID 1 already in my persistence context?
```

Initially, no.

So Hibernate executes:

```sql
select
    id,
    name,
    department_id
from employee
where id = 1;
```

The database returns:

```text
id = 1
name = Rahul
department_id = 10
```

Hibernate creates an object:

```java
Employee employee = new Employee();
employee.setId(1L);
employee.setName("Rahul");
```

It puts that object into the persistence context:

```text
Persistence Context

(Employee, 1) → Employee{id=1, name="Rahul"}
```

Then it returns the object to you.

---

## Second lookup in the same persistence context

```java
Employee e2 = entityManager.find(Employee.class, 1L);
```

This time:

```text
EntityManager checks persistence context
               ↓
Employee 1 is already present
               ↓
Return existing Java object
```

No second `SELECT` is needed.

Also:

```java
System.out.println(e1 == e2);
```

prints:

```text
true
```

They are not merely two objects with the same values. They are the **same Java object reference**.

---

## Why is this useful?

Without identity management, your application might have:

```text
Employee object A: id=1, name=Rahul
Employee object B: id=1, name=Rahul
```

If you change object A but use object B somewhere else, your program can become inconsistent.

The persistence context guarantees:

> Inside one persistence context, one database row corresponds to one managed Java entity instance.

---

## Important limitation

The first-level cache normally lasts only as long as that `EntityManager`.

```text
Transaction/request 1
    EntityManager A
    Employee 1 cached

Transaction/request ends
    EntityManager A closes
    Cache disappears

Transaction/request 2
    EntityManager B
    Cache initially empty
    Employee 1 requires another SQL query
```

So it is **not a global application cache**.

---

# 2. Dirty checking

Dirty checking means Hibernate automatically detects changes made to a **managed entity**.

Consider:

```java
@Transactional
public void updateEmployee() {
    Employee employee =
            entityManager.find(Employee.class, 1L);

    employee.setName("Mohit");
}
```

There is no explicit:

```java
entityManager.update(employee);
```

and normally no need for:

```java
repository.save(employee);
```

Why?

Because after `find()`, the employee is a **managed entity**.

---

## Step-by-step flow

### Step 1: Transaction starts

Spring conceptually does:

```text
Open/bind EntityManager
        ↓
Begin database transaction
```

### Step 2: Employee is loaded

```java
Employee employee =
        entityManager.find(Employee.class, 1L);
```

SQL:

```sql
select id, name, department_id
from employee
where id = 1;
```

Hibernate gets:

```text
id = 1
name = Rahul
```

It keeps both:

```text
Original snapshot:
name = Rahul

Managed object:
name = Rahul
```

The snapshot is used to determine whether something changed.

### Step 3: You change the Java object

```java
employee.setName("Mohit");
```

Now memory contains:

```text
Original snapshot:
name = Rahul

Current managed object:
name = Mohit
```

At this moment, Hibernate usually has not yet executed an `UPDATE`.

The database may still contain:

```text
name = Rahul
```

### Step 4: Flush happens

Before transaction commit, Hibernate flushes the persistence context.

It compares:

```text
Original value: Rahul
Current value:  Mohit
```

The entity is dirty, so Hibernate generates:

```sql
update employee
set name = 'Mohit'
where id = 1;
```

### Step 5: Commit

The database transaction commits, making the change permanent.

Complete flow:

```text
Transaction starts
        ↓
Employee loaded
        ↓
Employee becomes managed
        ↓
Original state remembered
        ↓
employee.setName("Mohit")
        ↓
Flush
        ↓
Hibernate detects changed field
        ↓
UPDATE SQL generated
        ↓
Transaction commits
```

---

## Why does `save()` sometimes appear in code?

Because the rule applies specifically to **managed entities**.

### Managed entity: no `save()` required

```java
@Transactional
public void update() {
    Employee employee = repository.findById(1L).orElseThrow();
    employee.setName("Mohit");
}
```

`employee` was loaded in the current persistence context, so dirty checking works.

### New entity: must be persisted

```java
Employee employee = new Employee();
employee.setName("Mohit");

entityManager.persist(employee);
```

The object did not come from the database and is not yet managed until you persist it.

With Spring Data:

```java
repository.save(employee);
```

### Detached entity: may need `merge()` or `save()`

Suppose the `EntityManager` has closed:

```text
Employee was managed
        ↓
EntityManager closed
        ↓
Employee becomes detached
```

Changing it later:

```java
employee.setName("Mohit");
```

does not automatically update the database because no active persistence context is tracking it.

You may need:

```java
Employee managedCopy = entityManager.merge(employee);
```

or:

```java
repository.save(employee);
```

So the accurate rule is:

> Changes to a managed entity are automatically detected. New or detached entities may require `persist`, `merge`, or `save`.

---

# 3. Transaction management

The earlier statement:

```text
EntityManager knows:
Transaction started → objects changed → commit → SQL
```

is a simplified explanation.

More accurately:

- **Spring’s transaction manager** starts, commits, or rolls back the transaction.
- The `EntityManager` participates in that transaction.
- Hibernate tracks entities and generates SQL during flushing.
- A JDBC connection performs the actual SQL against the database.

The complete relationship is:

```text
@Transactional
      ↓
Spring Transaction Manager
      ↓
EntityManager / Persistence Context
      ↓
Hibernate
      ↓
JDBC Connection
      ↓
Database
```

---

## Example transaction

```java
@Transactional
public void transferEmployee() {
    Employee employee =
            repository.findById(1L).orElseThrow();

    employee.setName("Mohit");

    Department department =
            departmentRepository.findById(20L).orElseThrow();

    employee.setDepartment(department);
}
```

Conceptual process:

```text
1. Spring begins transaction

2. EntityManager is associated with transaction

3. Employee is loaded and managed

4. Department is loaded and managed

5. Java objects are changed

6. Hibernate flushes changes

7. SQL is executed using a database connection

8. Spring commits transaction
```

Possible SQL:

```sql
select * from employee where id = 1;

select * from department where id = 20;

update employee
set name = 'Mohit',
    department_id = 20
where id = 1;
```

---

## What if something fails?

```java
@Transactional
public void update() {
    Employee employee =
            repository.findById(1L).orElseThrow();

    employee.setName("Mohit");

    throw new RuntimeException("Something failed");
}
```

Typical flow:

```text
Transaction started
        ↓
Employee changed in memory
        ↓
Exception thrown
        ↓
Spring rolls back transaction
        ↓
Database change is not committed
```

Even if an `UPDATE` was sent during a flush, rollback reverses the uncommitted database work.

---

# 4. Lazy loading

Suppose this association is lazy:

```java
@ManyToOne(fetch = FetchType.LAZY)
private Department department;
```

The employee table already contains:

```text
department_id = 10
```

But when loading `Employee`, Hibernate may avoid loading the complete department row.

Initial query:

```sql
select id, name, department_id
from employee
where id = 1;
```

Hibernate now knows:

```text
Employee:
id = 1
name = Rahul
department_id = 10
```

But it does not yet know:

```text
Department name
Department location
Department manager
...
```

---

## What is stored in `employee.department`?

Hibernate usually places a proxy:

```text
Employee
   |
   └── department → DepartmentProxy(id=10)
```

The proxy has the correct Java type:

```java
Department department = employee.getDepartment();
```

but its full data may not have been fetched yet.

---

## Does `getDepartment()` immediately execute SQL?

Not necessarily.

```java
Department department = employee.getDepartment();
```

may simply return the proxy.

Even this may often avoid a query:

```java
department.getId();
```

because the proxy already knows the identifier.

But when you access other information:

```java
String name = department.getName();
```

the proxy needs the real department data.

Hibernate executes:

```sql
select id, name
from department
where id = 10;
```

Then the proxy becomes initialized and returns:

```text
Engineering
```

Flow:

```text
employee.getDepartment()
        ↓
Return proxy containing department ID 10
        ↓
department.getName()
        ↓
Proxy asks Hibernate to initialize it
        ↓
EntityManager checks persistence context
        ↓
If Department 10 is absent, execute SELECT
        ↓
Create/manage Department object
        ↓
Return department name
```

---

## How does the first-level cache help lazy loading?

Suppose Department 10 was already loaded:

```java
Department department =
        entityManager.find(Department.class, 10L);

Employee employee =
        entityManager.find(Employee.class, 1L);

employee.getDepartment().getName();
```

The persistence context already contains:

```text
(Department, 10) → Department object
```

Therefore, Hibernate can associate the employee with the already-managed department instead of creating another independent department object.

---

## What if the EntityManager is already closed?

```java
Employee employee = getEmployeeFromDatabase();

// EntityManager/transaction has ended

employee.getDepartment().getName();
```

If the department was not initialized earlier, Hibernate cannot load it because the proxy no longer has an active persistence context.

You may receive:

```text
LazyInitializationException
```

Conceptually:

```text
Proxy says:
"I need Department 10"

But EntityManager is closed
        ↓
No active Hibernate session
        ↓
Cannot execute lazy-loading query
```

Typical solutions include:

- Access the relationship inside a transaction.
- Use `JOIN FETCH` when the relationship is needed.
- Use an entity graph.
- Map the required data directly into a DTO.

For example:

```java
@Query("""
    select e
    from Employee e
    join fetch e.department
    where e.id = :id
""")
Optional<Employee> findWithDepartment(Long id);
```

This loads employee and department together.

---

# Direct JDBC versus EntityManager

Using JDBC directly, you are responsible for everything:

```java
Connection connection = dataSource.getConnection();

try {
    connection.setAutoCommit(false);

    PreparedStatement select = connection.prepareStatement(
        "select id, name, department_id from employee where id = ?"
    );

    select.setLong(1, 1L);

    ResultSet resultSet = select.executeQuery();

    Employee employee = null;

    if (resultSet.next()) {
        employee = new Employee();
        employee.setId(resultSet.getLong("id"));
        employee.setName(resultSet.getString("name"));
    }

    employee.setName("Mohit");

    PreparedStatement update = connection.prepareStatement(
        "update employee set name = ? where id = ?"
    );

    update.setString(1, employee.getName());
    update.setLong(2, employee.getId());
    update.executeUpdate();

    connection.commit();

} catch (Exception exception) {
    connection.rollback();
    throw exception;
} finally {
    connection.close();
}
```

You manually handle:

- SQL generation
- Result-set-to-object conversion
- Transaction begin/commit/rollback
- Update detection
- Entity identity
- Relationship loading
- Resource cleanup

With JPA:

```java
@Transactional
public void updateEmployee() {
    Employee employee =
            entityManager.find(Employee.class, 1L);

    employee.setName("Mohit");
}
```

The apparent simplicity comes from the work done by Spring, Hibernate, the `EntityManager`, and the persistence context.

---

# Complete example combining all four features

```java
@Transactional
public void processEmployee() {
    Employee e1 =
            entityManager.find(Employee.class, 1L);

    Employee e2 =
            entityManager.find(Employee.class, 1L);

    System.out.println(e1 == e2); // true

    e1.setName("Mohit");

    String departmentName =
            e1.getDepartment().getName();
}
```

What happens:

```text
1. Spring starts transaction

2. EntityManager has an empty persistence context

3. First find(Employee, 1)
   → SQL executes
   → Employee object created
   → Employee stored in persistence context

4. Second find(Employee, 1)
   → Found in first-level cache
   → No new Employee SQL
   → Same object returned

5. e1.setName("Mohit")
   → Only Java object changes for now
   → EntityManager/Hibernate continues tracking it

6. e1.getDepartment().getName()
   → Lazy proxy requires department data
   → Department query executes if not already loaded
   → Department becomes managed

7. Method finishes
   → Hibernate flushes
   → Detects Employee name changed
   → Generates UPDATE

8. Spring commits transaction

9. EntityManager eventually closes
   → Persistence context and first-level cache disappear
```

Possible SQL:

```sql
select id, name, department_id
from employee
where id = 1;

select id, name
from department
where id = 10;

update employee
set name = 'Mohit'
where id = 1;
```

Notice that the second `find(Employee.class, 1L)` did not generate another query.

---

# Final mental model

```text
Database
   │
   │ rows
   ▼
Hibernate
   │
   │ creates Java entities
   ▼
Persistence Context
   │
   ├── stores one managed object per entity ID
   ├── remembers original state
   ├── tracks modifications
   └── supports lazy-loaded relationships
   ▲
   │ managed by
EntityManager
```

So the four features mean:

- **First-level cache:** Reuses already-managed entities within the same persistence context.
- **Dirty checking:** Detects modifications to managed entities and generates updates automatically.
- **Transaction participation:** Synchronizes entity changes with a database transaction during flush and commit.
- **Lazy loading:** Delays loading related entities until their data is actually required.


## Complete Flow

Suppose:

```java
@GetMapping("/employee")
```

Flow:

```text
HTTP Request

↓

Controller

↓

Service

↓

Repository

↓

EntityManager

↓

Persistence Context

↓

Hibernate

↓

Database
```

The EntityManager sits between your application and Hibernate.

---


---

# Where Does EntityManager Get a Database Connection?

It **doesn't own one permanently**.

Instead:

```text
EntityManager

↓

DataSource (HikariCP)

↓

Borrow Connection

↓

Execute SQL

↓

Return Connection
```

So:

- **EntityManager manages entities.**
- **HikariCP manages database connections.**

They have different responsibilities.

---

# Interview Definition

If an interviewer asks:

> **What is an EntityManager?**

A strong answer is:

> **EntityManager is the central JPA interface responsible for managing the lifecycle of entity objects. It manages the persistence context (first-level cache), performs CRUD operations, tracks changes to managed entities (dirty checking), coordinates with Hibernate to execute SQL, and works within transactions. It obtains database connections from the configured DataSource when it needs to interact with the database.**

---

# One-Line Memory Trick

Think of it this way:

```text
DataSource (HikariCP)
        ↓
Provides Database Connections

EntityManager
        ↓
Manages Java Entity Objects

Hibernate
        ↓
Converts Entity Operations into SQL

Database
```

So, **EntityManager is not a database connection**. It is an **object manager** that keeps track of your entity instances, their state, and synchronizes them with the database when necessary.


# Persistence Context




> **The objects in the Persistence Context are Java objects that Hibernate created using data fetched from the database.**

Let's go step by step.

---

# Initially

Suppose your database has:

### Employee Table

| id | name |
|----|------|
|1|Mohit|
|2|Rahul|

Nothing is in memory yet.

```text
Database

Employee Table
-------------------
1  Mohit
2  Rahul
```

Your application memory:

```text
EntityManager

↓

Persistence Context

(empty)
```

No `Employee` objects exist yet.

---

# Now you execute

```java
Employee emp = entityManager.find(Employee.class, 1L);
```

---

## Step 1: EntityManager checks its cache

```text
Persistence Context

Employee id=1 ?

↓

No
```

---

## Step 2: Hibernate asks the database

```sql
SELECT *
FROM employee
WHERE id = 1;
```

Database returns:

```text
id = 1
name = Mohit
```

---

## Step 3: Hibernate creates a Java object

It literally executes something conceptually like:

```java
Employee emp = new Employee();

emp.setId(1L);
emp.setName("Mohit");
```

Notice:

This object **did not exist before**.

Hibernate created it using the database row.

---

## Step 4: EntityManager stores it

```text
Database

Employee Table
-------------------
1 Mohit
2 Rahul

          SQL
           ▲
           │
           │
EntityManager
      │
      ▼
Persistence Context

Employee(id=1,name=Mohit)
```

Now the object is **managed**.

---

# Why store it?

Suppose later you call:

```java
Employee e2 =
entityManager.find(Employee.class,1L);
```

Should Hibernate execute SQL again?

No.

Instead:

```text
EntityManager

↓

Persistence Context

↓

Employee(id=1)
```

Returns the same object.

No SQL.

---

# Think of Persistence Context as a Workspace

Imagine you're editing a Word document.

The original file is on disk.

```text
Hard Disk

resume.docx
```

When you open it:

```text
Hard Disk

↓

Microsoft Word

↓

RAM

↓

Editable Document
```

You edit the RAM copy.

Only when you click **Save** does the disk change.

Exactly the same happens with Hibernate.

```text
Database

↓

SQL

↓

Java Object

↓

Persistence Context

↓

Modify Object

↓

Commit

↓

UPDATE Database
```

The database is like the hard disk.

The persistence context is like the document open in memory.

---

# Let's update an employee

Database:

```text
id=1
name=Mohit
```

Load:

```java
Employee emp =
entityManager.find(Employee.class,1L);
```

Memory:

```text
Persistence Context

Employee

id=1

name=Mohit
```

Now:

```java
emp.setName("Rahul");
```

Memory becomes:

```text
Persistence Context

Employee

id=1

name=Rahul
```

Notice:

**Database is still unchanged.**

Database:

```text
id=1
name=Mohit
```

Memory:

```text
id=1
name=Rahul
```

---

At transaction commit:

Hibernate compares:

```text
Original

Mohit

↓

Current

Rahul
```

It generates:

```sql
UPDATE employee
SET name='Rahul'
WHERE id=1;
```

Now the database changes.

---

# So what exactly is inside the Persistence Context?

It contains **Java objects**.

```text
Persistence Context

Employee Object
Department Object
Address Object
Order Object
```

Not database rows.

Not SQL.

Just ordinary Java objects that Hibernate created from database rows.

---

# Complete Picture

```text
          Database

+----------------------+
| Employee Table       |
|                      |
| 1  Mohit             |
| 2  Rahul             |
+----------------------+
           │
           │ SQL
           ▼
      Hibernate
           │
Creates Java Objects
           ▼
+------------------------------+
| Persistence Context          |
|                              |
| Employee(id=1,"Mohit")       |
| Employee(id=2,"Rahul")       |
| Department(id=10,"IT")       |
+------------------------------+
           ▲
           │
    EntityManager manages
```

---

# Interview Answer

If an interviewer asks:

> **What is stored inside the Persistence Context?**

A strong answer is:

> The Persistence Context stores **managed entity objects**. These are regular Java objects that Hibernate creates by reading rows from the database. The `EntityManager` manages these objects, tracks any changes made to them, and synchronizes those changes with the database when the transaction is flushed or committed.

So yes, **the database provides the data**, and **Hibernate converts that data into Java objects and stores those objects in the Persistence Context**. That's why it's often called an **in-memory representation of a portion of the database**.