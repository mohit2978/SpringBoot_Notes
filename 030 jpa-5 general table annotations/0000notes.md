**spring.jpa.hibernate.ddl-auto** configuration tells hibernate regarding how to create and manage the DB Schema.

| S.No. | Values | Create Schema | Update Schema | Delete Schema | Details |
|---|---|---|---|---|---|
| 1. | none | no | no | no | Do nothing. Good for **Production** |
| 2. | update | yes | yes | no | Does update but without deleting any existing data or schema. Good for **Development** environment |
| 3. | validate | no | no | no | During application startup, does matching between entities and DB Schema. If mismatch found, throws exception. |
| 4. | create | yes | yes | yes | Drops and re-create the schema during application startup. |
| 5. | create-drop | Yes | yes | yes | Creates the schema during startup and drops the schema when the application shutdown. *(generally by-default for in memory databases like H2)* |

This needs to be done first in Db which tell how to manage schema!! We use 1 for production as DBA takes queries and creates tables in production!!

Validate does nothing if there is a difference in Entity and Db schema it throws exception!!

Create→ drops and recreates!! So in development we use only update!!

**Mapping Classes to Tables**

**@Tables Annotation**

- Its an Optional field, if not defined, hibernate will generate table name based on entity name.
- Generally it follows CamelCase to UPPER_SNAKE_CASE means: UserDetails -> USER_DETAILS

```java
@Target(TYPE)
@Retention(RUNTIME)
public @interface Table{
String name() default "";
String schema() default "";
UniqueConstraint[] uniqueConstraints() default{};
Index[] indexes() default{};
}
```

 ![svg](<svgs/img1-table-annotation.svg>) 





We were using Entity till now!! Hibernate internally creates tables based on Entity class!!For camel case in class it(H2) put underScore in DB!!

But with table the name you define will come to that table name!!

```java
@Table(name= "USER_DETAILS")
@Entity
public class UserDetails {
//fields and their getters and setters
}
```
Result: `SELECT * FROM USER_DETAILS;`

```java
@Table(name= "USER_DETAILS", schema= "ONBOARDING")
@Entity
public class UserDetails {
//fields and their getters and setters
}
```
Result: `SELECT * FROM ONBOARDING.USER_DETAILS;`

Schema is logical grouping of schema!! Can set up some group can view table in this schema !! So there can be multiple tables in a schema!!Hibernate never create schema itself we need to create schema!!

Hibernate itself do not create schema itself we need to create it manually using   `INIT=CREATE SCHEMA IF NOT EXISTS ONBOARDING` in url!!

```properties
#database connection
spring.datasource.url=jdbc:h2:mem:userDB;INIT=CREATE SCHEMA IF NOT EXISTS ONBOARDING
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=create
```

If you want schema to be defined by springboot use above statement in application.properties!!we can add multiple schema creation statements in this separated by semicolon!!

Then we have unique constraint!! We can put this columns like below!!
We first tell phone should be unique then we tell that name and email together should be unique!!


 
```java
@Table(name= "USER_DETAILS",
        schema= "ONBOARDING",
         uniqueConstraints={
               @UniqueConstraint(columnNames="phone"), //single column unique constraint
               @UniqueConstraint(columnNames={"name","email"}) //composite unique constraint
        })
@Entity
public class UserDetails {

	@Id
	private  Long id;
	private  String name;
	private  String email;
	private  String phone;

	//Constructors
	public UserDetails(){
	}

	//Getters and setters
}
```

Result (INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE): ID, PHONE, EMAIL, NAME columns listed with their CONSTRAINT_NAME — can see above constraintName for email and name is same!!
For indexes we use this as below!!

```java
@Table(name="USER_DETAILS",
        schema= "ONBOARDING",
         uniqueConstraints={
               @UniqueConstraint(columnNames="phone"), //single column unique constraint
               @UniqueConstraint(columnNames={"name","email"}) //composite unique constraint
        },
         indexes={
               @Index(name="index_phone", columnList="phone"), //index on single column
               @Index(name="index_name_email", columnList="name, email") //index on composite column
        })
@Entity
public class UserDetails {

	@Id
	private  Long id;
	private  String name;
	private  String email;
	private  String phone;

	//Constructors
	public UserDetails(){
	}

	//Getters and setters
}
```

Result (INFORMATION_SCHEMA.INDEX_COLUMNS): PRIMARY_KEY_3, INDEX_PHONE, INDEX_NAME_EMAIL and the constraint-backed unique indexes are all listed against USERDB.ONBOARDING.USER_DETAILS.

 ![alt text](<026 jpa-5 map dto imp_250716_002919_3.jpg>) 
 
 
 **@Column Annotation**

- Its an Optional field, if not defined, JPA will add it with default values.

```java
@Table(name = "user_details")
@Entity
public class UserDetails {

    @Id
    private Long id;
    @Column(name = "full_name", unique = true, nullable = false, length = 255)
    private String name;
    private String email;
    private String phone;

    // Constructors
    public UserDetails() {
    }

    // Getters and setters
}
```

Result columns: `SELECT * FROM USER_DETAILS;` -> ID, EMAIL, FULL_NAME, PHONE

Tell your own column name of table!!

unique=true tells column is unique ,we telling at column level!!

nullable=false tells cannot be null!!

We even put the length of the field!!


 **@Id Annotation and @GeneratedValue Annotation**

**Primary Key:** must be unique, not null and used to uniquely identify each record.
- **@Id** annotation is used to mark the field as primary key.
-  Each entity can have only 1 primary key.
- Only 1 field can be annotated with @Id.

**Composite Primary key:** combination of two or more columns to form a primary key.

Using **@Embeddable** and  **@EmbeddedId** annotation. ↔ Using **@IdClass** and **@Id** annotation

**Rules to follow for both the approach:**

- Must be a public class.
- Must Implement the Serializable interface.
- Must have no-arg constructor
- Must override the equals() and hashCode() methods

If you do not write no-args constructor ,default one is still there!!

![svg](<svgs/img5-id-generatedvalue-composite.svg>) 




Using **@IdClass** and **@Id** annotation

```java
@Table(name = "user_details")
@Entity
public class UserDetails {

    private String name;
    private String address;
    private String phone;

    // Constructors
    public UserDetails() {
    }

    // Getters and setters

}
```
I want these 2 columns (name, address) to be defined as Composite Key

```java
@Table(name = "user_details")
@IdClass(UserDetailsCK.class)
@Entity
public class UserDetails {

    @Id
    private String name;
    @Id
    private String address;
    private String phone;

    // Constructors
    public UserDetails() {
    }

    // Getters and setters
}
```
```java
public class UserDetailsCK implements Serializable {

    private String name;
    private String address;

    public UserDetailsCK() {
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof UserDetailsCK)) {
            return false;
        }
        UserDetailsCK userCK = (UserDetailsCK) obj;
        return this.name.equals(userCK.name) && this.address.equals(userCK.address);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, address);
    }
}
```

Equals if both object same return true else return false
If obj is of UserDetailsCK then compare all fields!!

Here above we have put Id is not same as PK!! These are for composite key!!

![alt text](<026 jpa-5 map dto imp_250716_002919_6.jpg>)


 

![alt text](<026 jpa-5 map dto imp_250716_002919_7.jpg>) 


Using **@Embeddable** and **@EmbeddedId** annotation.

```java
@Table(name = "user_details")
@Entity
public class UserDetails {

    @EmbeddedId
    UserDetailsCK userDetailsCK;
    private String phone;

    // Constructors
    public UserDetails() {
    }

    // Getters and setters
}
```
```java
@Embeddable
public class UserDetailsCK implements Serializable {

    private String name;
    private String address;

    public UserDetailsCK() {
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof UserDetailsCK)) {
            return false;
        }
        UserDetailsCK userCK = (UserDetailsCK) obj;
        return this.name.equals(userCK.name) && this.address.equals(userCK.address);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, address);
    }
}
```

Result: `SELECT * FROM USER_DETAILS;` -> ADDRESS, NAME, PHONE (no rows, 4 ms)
Result (INFORMATION_SCHEMA.INDEX_COLUMNS): PRIMARY_KEY_3 over USER_DETAILS.ADDRESS (1, ASC), USER_DETAILS.NAME (2, ASC) — (2 rows, 2 ms)

To be clear with code!!

![alt text](<026 jpa-5 map dto imp_250716_002919_8.jpg>) 


```java
@Embeddable
public class UserDetailsCK implements Serializable {

    private String name;
    private String address;

    public UserDetailsCK() {
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof UserDetailsCK)) {
            return false;
        }
        UserDetailsCK userCK = (UserDetailsCK) obj;
        return this.name.equals(userCK.name) && this.address.equals(userCK.address);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, address);
    }
}
```

![svg](<svgs/img9-userdetailsck-embeddable.svg>)




```java
@Table(name = "user_details")
@Entity
public class UserDetails {

    @EmbeddedId
    UserDetailsCK userDetailsCK;
    private String phone;

    // Constructors
    public UserDetails() {
    }

    // Getters and setters
}
```


Result (INFORMATION_SCHEMA.INDEX_COLUMNS): PRIMARY_KEY_3 over USER_DETAILS.ADDRESS (1, ASC), USER_DETAILS.NAME (2, ASC) — (2 rows, 2 ms)

Even address and name will be indexed together!!

`SELECT * FROM USER_DETAILS;` -> ADDRESS, NAME, PHONE (no rows, 4 ms)

By default Pk is not Autofill , you need to tell how to fill PK!!



![alt text](<026 jpa-5 map dto imp_250716_002919_11.jpg>) 


**@GeneratedValue Annotation**

- Now we know, how to define Primary key.
-  But we can also define its generation strategy too. By default, primary key columns are not autofill.
-  It works with @Id annotation (only for single primary key not for composite one)

1.  GenerationType.IDENTITY

    - Each insert, generates a new identifier (auto-increment field)

```java
@Table(name = "user_details")
@Entity
public class UserDetails {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	private String name;
	private String phone;

	// Constructors
	public UserDetails() {
	}

	public UserDetails(String name, String phone) {
		this.name = name;
		this.phone = phone;
	}

    //getters and setters
}
```



![svg](<svgs/img14-postman-post-sj.svg>)  

![svg](<svgs/img15-postman-post-zj.svg>) 





![svg](<svgs/img16-select-result-incrementing-id.svg>) 


2.  GenerationType.SEQUENCE

- Used to generate Unique numbers.
- Speed up the efficiency when we cache sequence values.
- More control than IDENTITY.

> CREATE SEQUENCE user_seq INCREMENT BY 25 START WITH 100 MAXVALUE 9999;

```java
@Table(name = "user_details")
@Entity
public class UserDetails {

	@Id
	@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "unique_user_seq")
	@SequenceGenerator(name = "unique_user_seq", sequenceName = "db_seq_name", initialValue = 100, allocationSize = 5)
	private Long id;
	private String name;
	private String phone;

	// Constructors
	public UserDetails() {
	}

	public UserDetails(String name, String phone) {
		this.name = name;
		this.phone = phone;
	}

	//getters and setters
}
```

We can cache sequence values means hibernate asks the sequence generator to generate all values at once then store it!!

For Sequence we first need sequence generator by annotation!!If db has the sequenceGenerator  with name it will not create new!!






![svg](<svgs/img18-sequence-postman-and-params.svg>)




 Hibernate insert statements for 1st call, 2nd call, 3rd call, 4th call, 5th call into user_details (name, phone, id) values (?, ?, ?) each time. After 5th call, hibernate will fetch another 5 values via `Hibernate: select next value for db_seq_name`, then continues inserting.

See after 5th call we get from Sequence Generator!!

![svg](<svgs/img19-sequence-call-diagram.svg>)


 Advantage of SEQUENCE over IDENTITY:

    1.  Custom logic (start point, increment etc.)
    2.  Sequence generation logic is independent of table, so multiple tables an use it.
    3.  Range of IDs can be cached, so we can avoid hitting database each time a new id is required.
        (during IDENTITY, while INSERTION internally DB is auto generating the next ID , which require additional DB call)
    4.  Better portability, means IDENTITY is very DB specific while SEQUENCE can provide more consistent behavior across multiple DBs.



3.  GenerationType.TABLE

- @TableGenerator annotation is used but its very less efficient.
- Because:
	- Separate Table is created, just for managing unique IDs.
	- Each time id is required, SELECT-UPDATE query is executed.
	- Complex concurrency handling, when multiple operations happening in parallel, it requires LOCK/UNLOCK functionality. Which can lead to performance bottleneck.
	  In SEQUENCE type, its handle internally by DB using atomic counter, so its much more efficient.

Very less efficient we have table_Generator here!

Need to put lock if multiple tables using it!! So additional overhead here!!

---

### **Difference between `@Entity` vs `@Table` Annotation**

Both `@Entity` and `@Table` are JPA annotations (from `jakarta.persistence` / `javax.persistence`) placed at the class level, but they serve completely different purposes:

| Feature | `@Entity` | `@Table` |
| :--- | :--- | :--- |
| **Purpose** | Marks a Java class as a persistent entity. Tells JPA/Hibernate to manage instances of this class in the persistence context. | Customizes the physical database table mapping (name, schema, constraints, indexes). |
| **Requirement** | **Mandatory** — Every entity class must have `@Entity`. Without it, Hibernate ignores the class. | **Optional** — If omitted, JPA uses default naming strategies based on the entity class name. |
| **Abstraction Level** | **Logical / Object layer** (Java & JPA / JPQL level). | **Physical / Relational layer** (Database / SQL level). |
| **JPQL / HQL Queries** | Entity name used in JPQL queries is defined by `@Entity(name = "...")`. | Not used in JPQL; only used in native SQL queries and DB schema generation. |
| **Supported Attributes** | `name` (Entity name for JPQL queries) | `name`, `schema`, `catalog`, `uniqueConstraints`, `indexes` |

---

#### 1. When to use what?

- **Only `@Entity`**: When the default database table name (derived from the class name) and default schema are acceptable.
  ```java
  @Entity
  public class User {
      @Id
      private Long id;
      private String name;
  }
  ```
  - **JPQL Query:** `SELECT u FROM User u`
  - **Generated DB Table:** `user` (or `USER` depending on naming strategy)

- **Both `@Entity` and `@Table`**: When you need to specify a custom table name (e.g., reserved SQL keyword, legacy DB table), schema, indexes, or composite unique constraints.
  ```java
  @Entity
  @Table(name = "app_users", schema = "auth_schema")
  public class User {
      @Id
      private Long id;
      private String name;
  }
  ```
  - **JPQL Query:** `SELECT u FROM User u` (uses class/entity name)
  - **Generated DB Table:** `auth_schema.app_users`

---

#### 2. Key Confusion: `@Entity(name = "...")` vs `@Table(name = "...")`

This is a common interview question and area of confusion:

```java
@Entity(name = "AppUser")          // Entity name for JPQL/HQL queries
@Table(name = "tbl_user_records")  // Physical DB table name
public class User {
    @Id
    private Long id;
    private String name;
}
```

- **In JPQL / HQL:** You **must** use the entity name `AppUser`:
  ```java
  // Correct JPQL:
  entityManager.createQuery("SELECT u FROM AppUser u", User.class);

  // WRONG (will throw IllegalArgumentException / org.hibernate.hql.internal.ast.QuerySyntaxException):
  // entityManager.createQuery("SELECT u FROM tbl_user_records u", User.class);
  ```

- **In Native SQL / Database:** The actual table created or queried in the database is `tbl_user_records`:
  ```sql
  SELECT * FROM tbl_user_records;
  ```

---

#### Quick Summary:
1. **Can we have `@Table` without `@Entity`?**
   - **No.** `@Table` has no effect on its own. JPA only processes it if `@Entity` is present on the class.
2. **Can we have `@Entity` without `@Table`?**
   - **Yes.** JPA will automatically map it to a table using the class name.

