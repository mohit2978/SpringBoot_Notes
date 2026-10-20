**ORM (Object-Relational Mapping)**

![svg](<svgs/img1-orm-framework-diagram.svg>) 

- Act as a bridge between Java Object and Database tables.
- Unlike JDBC, where we have to work with SQL, with this, we can interact with database using Java Objects.



ORM→object relational mapping!! With JPA you need not know SQL!!

JPA is interface , hibernate, openJpa ,eclipseLink is implementation!! Hibernate is default implementation!! You can use others too by overriding this!!

In previous video seen jdbc !! previously Application Logic directly connected by JDBC no ORM !! as need to write queries!!

**Spring JDBC** can be used for complex, custom SQL queries or where performance optimization is required.

**JPA repositories** (via `@Repository` and extending `JpaRepository`) can be used for more conventional CRUD operations with entities.

If you use both, they will be defined separately: Spring JDBC would use `JdbcTemplate` or `NamedParameterJdbcTemplate`, while JPA repositories would be defined via `@Repository` and extending `JpaRepository` or `CrudRepository`.

 The `@Entity` annotation is part of the **JPA (Java Persistence API)**, not Spring JDBC. It is used to mark a class as an entity that will be managed by the JPA provider (typically Hibernate, EclipseLink, or another JPA implementation).

When you annotate a class with `@Entity`, it signifies that the class represents a table in a relational database. The fields within the class are mapped to columns in that table. JPA handles the mapping between the class and the database table, including managing the persistence of the object's state (saving, updating, deleting, etc.) in the database.

 **Lets first see, 1 happy flow first, before deep diving into JPA**

pom.xml:
```xml
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

application.properties:
```properties
#database connection properties
spring.datasource.url=jdbc:h2:mem:userDB
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
```

Above see dependency , this jpa brings hibernate too with it!! Then provide h2-database properties!!

```java
@RestController
@RequestMapping(value = "/api/")
public class UserController {

	@Autowired
	UserDetailsService userDetailsService;

	@GetMapping(path = "/test-jpa")
	public List<UserDetails> getUser() {
		UserDetails userDetails = new UserDetails( name: "xyx",
				email: "xyz@conceptandcoding.com");
		userDetailsService.saveUser(userDetails);
		return userDetailsService.getAllUsers();
	}
}
```
```java
@Service
public class UserDetailsService {

	@Autowired
	private UserDetailsRepository userDetailsRepository;

	public void saveUser(UserDetails user) {
		userDetailsRepository.save(user);
	}

	public List<UserDetails> getAllUsers() {
		return userDetailsRepository.findAll();
	}
}
```
```java
@Repository
public interface UserDetailsRepository extends
		JpaRepository<UserDetails, Long> {
}

@Entity
public class UserDetails {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	private String name;
	private String email;

	// Constructors
	public UserDetails() {
	}

	public UserDetails(String name, String email) {
		this.name = name;
		this.email = email;
	}

	// Getters and setters
}
```

```java
@RestController
@RequestMapping(value = "/api/")
public class UserController {

	@Autowired
	UserDetailsService userDetailsService;

	@GetMapping(path = "/test-jpa")
	public List<UserDetails> getUser() {
		UserDetails userDetails = new UserDetails( name: "xyx",
				email: "xyz@conceptandcoding.com");
		userDetailsService.saveUser(userDetails);
		return userDetailsService.getAllUsers();
	}
}
```




JpaRespository provide APIs!! For each entity we create Separate jpa Repository as sometimes we need to write HQL (hibernate query language) or JpQL(java persistence query language)



Entity is java class equivalent to Java class in DB!!

Id annotation tells its a primary key!!

Every new instance is a new row!!



**To enable the console, we can add, below two properties in  "application.properties"**

```properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

These properties are too see h2-console!! We above hitted save method and then fetch all Users in DB !!In h2 everytime you start old data is deleted!!


 [http://localhost:8080/h2-console](http://localhost:8080/h2-console)


```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

These two are just  to show SQl queries !! we don't do this prd!! In 2nd line we telling show in hibernate format!!

See JPA interface

```java
@NoRepositoryBean
public interface JpaRepository<T, ID> extends ListCrudRepository<T, ID>,
ListPagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
    void flush();

    <S extends T> S saveAndFlush(S entity);

    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);

    /** @deprecated */
    @Deprecated
    default void deleteInBatch(Iterable<T> entities) {
        this.deleteAllInBatch(entities);
    }

    void deleteAllInBatch(Iterable<T> entities);

    void deleteAllByIdInBatch(Iterable<ID> ids);

    void deleteAllInBatch();

    /** @deprecated */
    @Deprecated
    T getOne(ID id);

    /** @deprecated */
    @Deprecated
    T getById(ID id);

    T getReferenceById(ID id);

    <S extends T> List<S> findAll(Example<S> example);

    <S extends T> List<S> findAll(Example<S> example, Sort sort);
}
```

See all the methods it has provided!!

Let us see the architecture!!



 
 
**Looks very simple, but what's happening inside?**

**JPA Architecture/Components involved:**
![svg](<svgs/img8-jpa-architecture-components.svg>)



First is the Persistence Unit!!
Here we provide configuration!!

In spring boot persistence unit is Application.properties!!

Using that Persistence Unit an object of Entity managerFactory is created which creates Entity manager !!

One persistence unit has One Entitymanager Factory!! If we have Multiple Db then do not use application.properties! Use Configuration Annotation class and provide properties of both DB!!

EntityManager in hibernate is Session, In JPA we call it Entity Manager!!

If you see Jpa it internally has EntityManager

 ![alt text](023jpa-2_250716_002741_8.jpg) 

From Entity Manager you can put data in Persistence context which is place holder for Entity!!For every Entity Manager we have a Persistence context!!

This Persistence context help in first level caching!! We see it next video!! Persistence context holds object for EntityManager!! Persistence context only saves in Entity!!

Now we know underlying framework in JDBC ,and it does not understand Jpql or HQL so Diaect helps to convert these to SQL!! So that JDBC can understand it!!

In application.properties we do not put dialect as in that we have a lot of AutoConfiguration !! But when you define in configuration class you need to put dialect!!

pom.xml:
```xml
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

1.  **Persistence Unit**

- Logical grouping of Entity classes which share same configurations.
- Configuration details like:
	- Database connection properties
	- JPA Provider (hibernate etc. ) etc.



  Persistence.xml:
```xml
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">

    <persistence-unit name="persistentUnitNameHere" transaction-type="RESOURCE_LOCAL">

        <!-- Logical grouping of Entity Classes which all shares same configurations -->
        <class>com.conceptandcoding.entity.SampleEntity1</class>
        <class>com.conceptandcoding.entity.SampleEntity2</class>

        <!-- which JPA provider we want to use, hibernate or OpenJPA or EclipseLink etc... -->
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>


        <!-- DB Connection Properties -->
        <properties>

            <!-- specific provider dialect -->
            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL5Dialect" />

            <property name="javax.persistence.jdbc.driver" value="org.h2.Driver" />
            <property name="javax.persistence.jdbc.url" value="jdbc:h2:mem:userDB" />
            <property name="javax.persistence.jdbc.user" value="sa" />
            <property name="javax.persistence.jdbc.password" value="" />
            <!-- more properties here -->
        </properties>

    </persistence-unit>

</persistence>
```

application.properties:
```properties
#database connection properties
spring.datasource.url=jdbc:h2:mem:userDB
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# Define packages to scan
# (its optional, spring boot looks for all @Entity, but still if we want specific package lookup only)
spring.jpa.packages-to-scan=com.conceptandcoding.entity

#define provider
#(its optional, spring boot automatically picks based on provider)
spring.jpa.properties.javax.persistence.provider=org.hibernate.jpa.HibernatePersistenceProvider
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect

#transaction type, Default is RESOURCE_LOCAL only
spring.jpa.properties.javax.persistence.transactionType=RESOURCE_LOCAL
```
Can read above what is optional here!!

If not using spring boot we put persistence.xml!!

In that we put transaction_type=RESOURCE_LOCAL as we put transaction across one DB!!if need multiple Db we use JTA!!










2. **EntityManagerFactory**

- Using Persistence Unit configuration, EntityManagerFactory object get created during application startup.
- If any property is not provided, default one is picked and set.
- 1 EntityManagerFactory  for 1 Persistence Unit.
- This class act as a Factory to create an object of EntityManager.

LocalContainerEntityManagerFactoryBean.java:
```java
@Override
protected EntityManagerFactory createNativeEntityManagerFactory() throws PersistenceException {
    Assert.state( expression: this.persistenceUnitInfo != null, message: "PersistenceUnitInfo not initialized");

    PersistenceProvider provider = getPersistenceProvider();
    if (provider == null) {
        String providerClassName = this.persistenceUnitInfo.getPersistenceProviderClassName();
        if (providerClassName == null) {
            throw new IllegalArgumentException(
                "No PersistenceProvider specified in EntityManagerFactory configuration, " +
                    "and chosen PersistenceUnitInfo does not specify a provider class name either");
        }
        Class<?> providerClass = ClassUtils.resolveClassName(providerClassName, getBeanClassLoader());
        provider = (PersistenceProvider) BeanUtils.instantiateClass(providerClass);
    }

    if (logger.isDebugEnabled()) {
        logger.debug("Building JPA container EntityManagerFactory for persistence unit '" +
            this.persistenceUnitInfo.getPersistenceUnitName() + "'");
    }
    EntityManagerFactory emf =
        provider.createContainerEntityManagerFactory(this.persistenceUnitInfo, getJpaPropertyMap());
    postProcessEntityManagerFactory(emf, this.persistenceUnitInfo);

    return emf;
}
```


See in 2ns last line it is creating EntityManagerfactory object!! This is from autoconfiguration !!
now we want to configure manually!! So we use as below!!

 **What if we want to manually create and object of EntityManagerFactory?**

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

    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setGenerateDdl(true);
        adapter.setDatabasePlatform("org.hibernate.dialect.H2Dialect");
        return adapter;
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter) {
        LocalContainerEntityManagerFactoryBean emf1 = new LocalContainerEntityManagerFactoryBean();
        emf1.setDataSource(dataSource);
        emf1.setJpaVendorAdapter(jpaVendorAdapter);
        emf1.setPackagesToScan("com.conceptandcoding.learningspringboot.jpa");
        emf1.setPersistenceUnitName("uniqueFactoryName"); //  unique name for our EntityManagerFactory
        return emf1;
    }
}
```

Here you need to set everything manually!! You are creating your object!! To create another Persistence Unit you can create another Configuration class!!

For each persistence unit we have one DB!! For each DB we have one Transaction Manager but we want Transaction to be managed in multiple DB!!

3.  **Transaction Manager association with EntityManagerFactory**

During  persistence unit, we have specified the value for "transaction-type" value either:
- RESOURCE_LOCAL (default)
- JTA (Java Transaction API)

Transaction Manager could be of 2 type:

- Manager, which managing transaction For 1 DB.

- Manager, which managing transaction which can span across multiple DB. That's also possible using JTA.

During application startup, after EntityManagerFactory object created, based on  RESOURCE_LOCAL or JTA, transaction manager object get created.

Default is RESOURCE_LOCAL which creates JPATransactionmanager which is for 1 Db only!!
JTA do 2 phase commit!! Can see video of transactions, there we have seen types of transaction manager!!And JTA is used for multiple DB!!



### Use case-1: Transaction Manager for managing txn for 1 DB

application.properties:
```properties
#database connection properties
spring.datasource.url=jdbc:h2:mem:userDB
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# Define packages to scan
# (its optional, spring boot looks for all @Entity, but still if we want specific package lookup only)
spring.jpa.packages-to-scan=com.conceptandcoding.entity

#define provider
#(its optional, spring boot automatically picks based on provider)
spring.jpa.properties.javax.persistence.provider=org.hibernate.jpa.HibernatePersistenceProvider
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect

#transaction type, Default is RESOURCE_LOCAL only
spring.jpa.properties.javax.persistence.transactionType=RESOURCE_LOCAL
```

JpaTransactionManager.java (implementation of PlatformTransactionManager):
```java
@Override
public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    if (getEntityManagerFactory() == null) {
        if (!(beanFactory instanceof ListableBeanFactory lbf)) {
            throw new IllegalStateException("Cannot retrieve EntityManagerFactory by persistence unit name " +
                "in a non-Listable BeanFactory: " + beanFactory);
        }
        setEntityManagerFactory(EntityManagerFactoryUtils.findEntityManagerFactory(lbf, getPersistenceUnitName()));
    }
}
```
Debugger view: beanType = {Class@10098} "class org.springframework.orm.jpa.JpaTransactionManager"

 
 
 **We can create it manually too:**

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

    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setGenerateDdl(true);
        adapter.setDatabasePlatform("org.hibernate.dialect.H2Dialect");
        return adapter;
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter) {
        LocalContainerEntityManagerFactoryBean emf1 = new LocalContainerEntityManagerFactoryBean();
        emf1.setDataSource(dataSource);
        emf1.setJpaVendorAdapter(jpaVendorAdapter);
        emf1.setPackagesToScan("com.conceptandcoding.learningspringboot.jpa");
        emf1.setPersistenceUnitName("uniqueFactoryName"); //  unique name for our EntityManagerFactory
        return emf1;
    }

    @Bean
    public JpaTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

See above how we have manually connected!!

TransactionManager is created also and it has 1:1 relationship with EntityManagerfactory!!

### UseCase2: Transaction Manager for creating txn which can span across multiple DB

*I will create a separate video, in which I will explain, how we can handle txn which can span across multiple database.*
*As it required some time for explaining JTA related nuiance and keyworks like Atomikos, XADataSource and how it uses 2PC to orchestrate txn etc.. which is out of the context for todays topic.*

But at this point, we can understand that, TRANSACTION MANAGER is created based on what "transaction-type" we provided in the persistence unit (application.properties) file.

-------------- Above all steps happened during application startup, below steps happen when API gets invoked -----------------

4.  **EntityManager and Persistence Context**

**EntityManager :**
 - Its an interface in JPA that provides methods to perform CRUD operations on entities.
	- **persist()** (for saving)
	- **merge()** (for updating)
	- **find()** (for fetching)
	- **remove()** (for deleting)
	- **createQuery()** (for executing JPQL queries)

 - EntityManager interface methods are implemented by JPA Vendors like Hibernate etc.
 - EntityManagerFactory helps to create an Object of EntityManager.

**PersistenceContext:**

 - Consider its a first level cache.
 - For each EntityManager, PersistenceContext object is created, which hold list of Entities its working on.
 - Also manage the life cycle of entity.

Entity Manager Insert/Update/Delete operations are Transaction bounded. Means, it first check if 'Transaction' is open, if not, it will throw exception.
But not all (READ operations are not Transaction bounded)

Internally Jpa uses entity manager only but JPA provides some other things too like:
1.  Methods like findAll() , deleteALLL() etc
2.  Start and closing of entity manager object managed by JPA

3.  Pagination and sorting capability
4.  Transactional annotation support is also in JPA only!! If not provide Transactional on insert ,modify and delete then Entity manager will not do it!! JPA repository automatically adds it!!
Can check the save method it has Transactional on it!!

```java
@Service
public class UserDetailsService {

	@Autowired
	private UserDetailsRepository userDetailsRepository;

	public void saveUser(UserDetails user) {
		userDetailsRepository.save(user);
	}

	public List<UserDetails> getAllUsers() {
		return userDetailsRepository.findAll();
	}
}
```
```java
@Repository
public interface UserDetailsRepository extends
		JpaRepository<UserDetails, Long> {
}
```

>Note:Internally JpaRepository all Insert, Update, Delete methods are annotated with @Transactional,
so even if we do not write, spring framework takes care of it.

```java
@Transactional
public <S extends T> S save(S entity) {
    Assert.notNull(entity, message: "Entity must not be null");
    if (this.entityInformation.isNew(entity)) {
        this.entityManager.persist(entity);
        return entity;
    } else {
        return this.entityManager.merge(entity);
    }
}
```

**What if, I try to directly call EntityManager persist method, instead of spring framework?**

```java
@Service
public class UserDetailsService {

    @PersistenceContext
    EntityManager entityManager;

    public void saveUser(UserDetails user) {
        entityManager.persist(user);
    }
}
```

Error: `jakarta.persistence.TransactionRequiredException`: No EntityManager with actual transaction available for current thread
```
	at org.springframework.orm.jpa.SharedEntityManagerCreator$SharedEntityManagerInvocationHandler.invoke(SharedEntityManagerCreator.ja...
	at jdk.proxy2/jdk.proxy2.$Proxy108.persist(Unknown Source) ~[na:na]
	at com.conceptandcoding.learningspringboot.jpa.service.UserDetailsService.saveUser(UserDetailsService.java:23) ~[classes/:na]
	at com.conceptandcoding.learningspringboot.UserController.getUser(UserController.java:22) ~[classes/:na] <4 internal lines>
```

 Now here above we are creating Entity Manger ourself!! We put PersistanceContext on entitymanager,it has internally Autowired also with some business logic!!

**If I add @Transactional, it works fine now.**

```java
@Service
public class UserDetailsService {

    @PersistenceContext
    EntityManager entityManager;

    @Transactional
    public void saveUser(UserDetails user) {
        entityManager.persist(user);
    }
}
```
```json
[
    {
        "id": 1,
        "name": "xyx",
        "email": "xyz@conceptandcoding.com"
    }
]
```

In above image it worked with Transactional!! But JPA provides all of this!!
We no need to do it!!

Easy to understand i guess now!!

With entity Manager only we can interact with DB!!

 ![alt text](023jpa-2_250716_002741_22.jpg) **Life cycle of Entity in persitenceContext:**

