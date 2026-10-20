 ![svg](<svgs/img1-orm-framework-diagram.svg>)



ORM Framework → Bridge B/w Java Object & Table

JPA is a interface. JPA internally needs to be implemented. Hibernate is one of implementation of JPA ,other implementations are OpenJPA,EclipseLink 

Then Hibernate interact with JDBC (also an interface) So DB Driver is implementation of JDBC which tells how to connect to DB. Every DB Vendor like PostgreSQL, MySQL has its own driver

**Before going to JPA, lets recall JDBC**

JDBC (Java Database Connectivity) provides an Interface to:

- Make connection with DB
- Query DB
- and process the result

With JPA/hibernate we need not write SQL Queries !!

Actual implementation is provided by Specific DB Drivers.

For example:

MySQL
- Driver : Connector/J
- Class : *com.mysql.cj.jdbc.Driver*

PostgreSQL
- Driver : PostgreSQL JDBC Driver
- Class : *org.postgresql.Driver*

H2 (in-memory)
- Driver : H2 Database Engine
- Class : *org.h2.Driver*

These drivers has implementation how to do use Above operation


 

Using JDBC without Spring Boot
First next page Steps check & then this

```java
public class UserDAO {

    public void createUserTable() {
        try {
            Connection connection = new DatabaseConnection().getConnection();
            Statement statementQuery = connection.createStatement();
            String sql = "CREATE TABLE users(user_id INT AUTO_INCREMENT PRIMARY KEY, user_name VARCHAR(100), age INT)";
            statementQuery.executeUpdate(sql);
        }
        catch (SQLException e) { /** handle exception **/ }
        finally { /** close statementQuery and db connection **/ }
    }

    public void createUser(String userName, int userAge) {
        try {
            Connection connection = new DatabaseConnection().getConnection();
            String sqlQuery = "INSERT INTO users(user_name, age) VALUES (?, ?)";
            PreparedStatement preparedQuery = connection.prepareStatement(sqlQuery);
            preparedQuery.setString( parameterIndex: 1, userName);
            preparedQuery.setInt( parameterIndex: 2, userAge);
            preparedQuery.executeUpdate();
        }
        catch (SQLException e) { /** handle exception **/ }
        finally { /** close preparedQuery and db connection **/ }
    }

    public void readUsers() {
        try {
            Connection connection = new DatabaseConnection().getConnection();
            String sqlQuery = "SELECT * FROM users";
            PreparedStatement preparedQuery = connection.prepareStatement(sqlQuery);
            ResultSet output = preparedQuery.executeQuery();
            while (output.next()) {
                String userDetails = output.getInt( columnLabel: "user_id") +
                        ":" + output.getString( columnLabel: "user_name") +
                        ":" + output.getInt( columnLabel: "age");
                System.out.println(userDetails);
            }
        }
        catch (SQLException e) { /** handle exception **/ }
        finally { /** close preparedQuery and db connection **/ }
    }
}
```

1. Get connection of DB for Every operation
2. Run Query
3. Close Connection



To create a Connection

```java
public class DatabaseConnection {

    public Connection getConnection() {
        try {
            // H2 Driver loading
            Class.forName( className: "org.h2.Driver");

            // Establish connection with DB
            return DriverManager.getConnection( url: "jdbc:h2:mem:userDB", user: "sa", password: "");

        }
        catch (ClassNotFoundException | SQLException e) {
            //handle  exception
        }

        return null;
    }
}
```

1. load driver (Using In Memory DB)
2. Establish connection with DB
(DB Name → "jdbc:h2:mem:userDB")

If we see above example:
- Connection
- Statement
- PreparedStatement
- ResultSet etc.
All are interfaces which JDBC provide and each specific driver provide the implementation for it.

But there are so much of BOILERCODE present like:

- Driver class loading
- DB Connection Making
- Exception Handling
- Closing of the DB connection and other objects like Statement etc.
- Manual handling of DB Connection Pool
  Etc..

for Each Query need to create a new connection

If you dont do this then there can be Memory Leak

Ideal way should be have a pool of pre defined size & take connection from that & Execute & return to the pool !!


**Using JDBC with Springboot**

Spring Boot help to reduce All boilerplate code

pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

Spring Boot has jdbcTemplate which remove All boilerplate code

```java
@Repository
public class UserRepository {

    @Autowired
    JdbcTemplate jdbcTemplate;

    public void createTable() {
        jdbcTemplate.execute( sql: "CREATE TABLE users (user_id INT AUTO_INCREMENT PRIMARY KEY, " +
                "user_name VARCHAR(100), age INT)");
    }

    public void insertUser(String name, int age) {
        String insertQuery = "INSERT INTO users (user_name, age) VALUES (?, ?)";
        jdbcTemplate.update(insertQuery, name, age);
    }

    public List<User> getUsers() {
        String selectQuery = "SELECT * FROM users";
        return jdbcTemplate.query(selectQuery, (rs, rowNum) -> {
            User user = new User();
            user.setUserId(rs.getInt( columnLabel: "user_id"));
            user.setUserName(rs.getString( columnLabel: "user_name"));
            user.setAge(rs.getInt( columnLabel: "age"));
            return user;
        });
    }
}
```

↳ Plain SQL Query


Business Logic class

```java
@Component
public class UserService {

    @Autowired
    UserRepository userRepository;

    public void createTable() {
        userRepository.createTable();
    }

    public void insertUser(String userName, int age) {
        userRepository.insertUser(userName, age);
    }

    public List<User> getUsers() {
        List<User> users = userRepository.getUsers();
        for(User user : users) {
            System.out.println(user.userId + ":" + user.getUserName() + ":" + user.getAge());
        }
        return users;
    }
}
```

application.properties

```properties
spring.datasource.url=jdbc:h2:mem:userDB
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.h2.console.enabled=true
```

Simple POJO class

```java
public class User {

    int userId;
    String userName;
    int age;

    //getters and setters
}
```



**Driver class loading** -
JdbcTemplate load it at the time of application startup in DriverManager class.

**DB Connection Making** -
jdbcTemplate takes care of it, whenever we execute any query.

**Exception Handling** -
in Plain JDBC, we get very abstracted *'SQLException'* but in jdbcTemplate, we get granular error like DuplicateKeyException, QueryTimeoutException etc.. (defined in org.springframework.dao package).

**Closing of the DB connection and other resources** -
when we invoke update or query method, after success or failure of the operation, jdbcTemplate takes care of either closing or return the connection to Pool itself.

- **Manual handling of DB Connection Pool** -
Springboot provides default jdbc connection pool i.e. *'HikariCP'* with Min and Max pool size of 10.
And we can change the configuration in *'application.properties'*

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
```

We can also configure different jdbc connection pool if we want like below:

↳ Spring Boot provide a connection pool called 'Hikari CP'
By default both are 10

```java
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setJdbcUrl("jdbc:h2:mem:userDB");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }
}
```

If you want some other connection pool you can configure here, for this we put hikari only

**JdbcTemplate frequently used methods**

| Method Name | Use For | Sample |
|---|---|---|
| update(String sql, Object... args) | Insert Update Delete | `String insertQuery = "INSERT INTO users (user_name, age) VALUES (?, ?)";`<br>`int rowsAffected = jdbcTemplate.update(insertQuery , "X", 27);`<br><br>`String updateQuery= "UPDATE users SET age = ? WHERE user_id = ?";`<br>`int rowsAffected = jdbcTemplate.update(updateQuery, 29, 1);` |
| update(String sql, PreparedStatementSetter pss) | Insert Update Delete | `String insertQuery= "INSERT INTO users (user_name, age) VALUES (?, ?)";`<br>`jdbcTemplate.update(insertQuery, (PreparedStatement ps) -> {`<br>`    ps.setString(1, "X");`<br>`    ps.setInt(2, 25); });`<br><br>`String updateQuery= "UPDATE users SET age = ? WHERE user_id = ?";`<br>`jdbcTemplate.update(updateQuery, (PreparedStatement ps) -> {`<br>`    ps.setString(1, 29);`<br>`    ps.setInt(2, 1);`<br>`});` |
| query(String sql, RowMapper&lt;T&gt; rowMapper) | Get multiple Rows | `List<User> users = jdbcTemplate.query("SELECT * FROM users",  (rs, rowNum) -> {`<br>`    User user = new User();`<br>`    user.setUserId(rs.getInt("user_id"));`<br>`    user.setUserName(rs.getString("user_name"));`<br>`    user.setAge(rs.getInt("age"));`<br>`    return user;  });` |
| queryForList(String sql, Class&lt;T&gt; elementType) | Get Single Column of Multiple Rows | `List<String> userNames =`<br>`    jdbcTemplate.queryForList("SELECT user_name FROM users", String.class);` |
| queryForObject(String sql, Object[] args, Class&lt;T&gt; requiredType) | Get single Row | `User user =`<br>`    jdbcTemplate.queryForObject("SELECT * FROM users WHERE user_id = ?", new Object[]{1}, User.class);` |
| queryForObject(String sql, Class&lt;T&gt; requiredType) | Get Single Value | `int userCount =`<br>`    jdbcTemplate.queryForObject("SELECT COUNT(*) FROM users", Integer.class);` |

Variable Arguments (Object... args)

↳ if Queries are Complex use this (PreparedStatementSetter)

 ![svg](<svgs/img7-jdbctemplate-methods-table.svg>) 
