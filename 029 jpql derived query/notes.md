# Derived Queries and JPQL

## Why Derived Queries

Until now, the repository interface just extended `JpaRepository`, and the service layer invoked methods already available in the JPA framework, like `save()` and `findById()`:

```java
@Repository
public interface UserDetailsRepository extends JpaRepository<UserDetails, Long> {
}
```

```java
@Service
public class UserDetailsService {

    @Autowired
    UserDetailsRepository userDetailsRepository;

    public UserDetails saveUser(UserDetails user) {
        return userDetailsRepository.save(user);
    }

    public UserDetails findByID(Long primaryKey) {
        return userDetailsRepository.findById(primaryKey).get();
    }
}
```

But there's often a requirement to write more specific queries than what these built-in methods offer. **Derived Query** is JPA's answer to this for simple cases: it automatically generates a query just from the method's name, as long as the method follows a specific naming convention. Derived queries are used for GET/REMOVE operations — not for INSERT/UPDATE, since those are already covered by `save()`.

Internally, this is handled by JPA's `PartTree.java`, which recognizes prefixes like `find`, `read`, `get`, `query`, `search`, `stream` (for queries), `count`, `exists`, and `delete`/`remove` (for deletes), matched with a regex roughly like:

```
^(find|read|get|query|search|stream|count|exists|delete|remove)((\p{Lu}.*?))??By
```

In plain terms: the method name must start with one of those prefixes, optionally followed by any characters, and must contain `By` — after `By` you name an entity field. For example:

```java
@Repository
public interface UserDetailsRepository extends JpaRepository<UserDetails, Long> {
    List<UserDetails> findUserDetailsByName(String userName);
}
```

```java
@Table(name = "user_details")
@Entity
public class UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long userId;

    @Column(name = "user_name")
    private String name;
    private String phone;

    // getters and setters
}
```

This translates internally to:

```sql
select ud1_0.user_id, ud1_0.user_name, ud1_0.phone
from user_details ud1_0
where ud1_0.user_name = ?
```

## Combining Conditions: And / Or

```java
List<UserDetails> findUserDetailsByNameAndPhone(String userName, String phone);
```

```sql
where ud1_0.user_name = ? and ud1_0.phone = ?
```

```java
List<UserDetails> findUserDetailsByNameAndPhoneOrUserId(String userName, String phone, Long id);
```

```sql
where ud1_0.user_name = ? and ud1_0.phone = ? or ud1_0.user_id = ?
```

JPA automatically derives the query purely from the method name — hence the name Derived Query.

## Other Keywords (from Part.java)

JPA ships an enum of supported keywords for building derived query method names, including: `Between`, `IsNotNull`/`NotNull`, `IsNull`/`Null`, `LessThan`, `LessThanEqual`, `GreaterThan`, `GreaterThanEqual`, `Before`, `After`, `NotLike`/`Like`, `StartingWith`/`StartsWith`, `EndingWith`/`EndsWith`, `IsNotEmpty`/`NotEmpty`, `IsEmpty`/`Empty`, `NotContaining`/`Containing`/`Contains`, `NotIn`, `In`, `Near`, `Within`, `MatchesRegex`/`Matches`, `Exists`, `IsTrue`/`True`, `IsFalse`/`False`, `IsNot`/`Not`, and the default `Is`/`Equals`.

Example with `IsIn`:

```java
List<UserDetails> findUserDetailsByNameIsIn(List<String> userName);
```

```sql
where ud1_0.user_name in (?)
```

Example with `Like`:

```java
List<UserDetails> findUserDetailsByNameLike(String userName);
```

```sql
where ud1_0.user_name like ? escape '\'
```

Derived Query is not meant for complex queries like joins.

## Delete via Derived Query

```java
@Transactional
void deleteByName(String userName);
```

Deleting needs the `@Transactional` annotation. Internally JPA first selects the matching rows, then issues a delete for each one by ID:

```sql
select ud1_0.user_id, ud1_0.user_name, ud1_0.phone from user_details ud1_0 where ud1_0.user_name = ?
delete from user_details where user_id = ?
delete from user_details where user_id = ?
```

## Pagination and Sorting in Derived Query

JPA provides two interfaces to support pagination and sorting: `Pageable` and `Sort` (both from `org.springframework.data.domain`). `Pageable` carries `pageNumber` and `pageSize` (number of records per page).

```java
@Repository
public interface UserDetailsRepository extends JpaRepository<UserDetails, Long> {
    List<UserDetails> findUserDetailsByNameStartingWith(String userName, Pageable page);
}
```

```java
@Service
public class UserDetailsService {

    @Autowired
    UserDetailsRepository userDetailsRepository;

    public List<UserDetails> findByNameDerived(String name) {
        Pageable pageable = PageRequest.of(0, 5); // page 0, 5 records per page
        return userDetailsRepository.findUserDetailsByNameStartingWith(name, pageable);
    }
}
```

`Pageable` can technically be placed as a parameter anywhere, but by convention it's placed last.

If more information about the pages themselves is needed (total pages, whether it's the first/last page), return type `Page<T>` instead of `List<T>`:

```java
@Repository
public interface UserDetailsRepository extends JpaRepository<UserDetails, Long> {
    Page<UserDetails> findUserDetailsByNameStartingWith(String userName, Pageable page);
}
```

```java
public List<UserDetails> findByNameDerived(String name) {
    Pageable pageable = PageRequest.of(0, 5);
    Page<UserDetails> userDetailsPage = userDetailsRepository.findUserDetailsByNameStartingWith(name, pageable);
    List<UserDetails> userDetailsList = userDetailsPage.getContent();
    System.out.println("total pages: " + userDetailsPage.getTotalPages());
    System.out.println("is first page: " + userDetailsPage.isFirst());
    System.out.println("is last page: " + userDetailsPage.isLast());
    return userDetailsList;
}
```

Internally this generates the same base query plus an `offset ? rows fetch first ? rows only` clause.

Sorting can be attached directly to the `Pageable`:

```java
Pageable pageable = PageRequest.of(0, 5, Sort.by("name").descending());
```

This effectively adds an `order by <field-name>` clause. `Sort.by()` accepts a list of fields — when multiple fields are provided, sorting is applied in that order: it sorts by the first field, and if there are duplicate values there, the second field is used as a tiebreaker, and so on (same idea as SQL's multi-column `ORDER BY`).

If different sort directions are needed per field:

```java
Sort sort = Sort.by(
        Sort.Order.asc("name"),
        Sort.Order.desc("phone")
);
return userDetailsRepository.findUserDetailsByNameStartingWith(name, sort);
```

Sorting alone (without pagination) can also be passed directly as a `Sort` parameter instead of wrapping it in a `Pageable`.

## JPQL (Java Persistence Query Language)

For queries a bit too complex for Derived Query (but not complex enough to need native SQL), JPA provides **JPQL**:

- Similar to SQL, but it works on the Entity object instead of directly on the database.
- Database independent.
- Works with entity names and entity field names, not table names or column names.

Syntax uses `@Query` with an entity alias instead of `*`:

```java
@Query("SELECT u FROM UserDetails u WHERE u.name = :userFirstName")
List<UserDetails> findByUserName(@Param("userFirstName") String userName);
```

Here `UserDetails` is the entity (not table) name, `u.name` is the entity field (not column) name, `u` is the alias returning all fields, and `@Param` binds the method parameter to the named parameter in the query.

There's no strict rule on return type — it can be a `List` or a single object. But if the query actually returns more than one row while the return type is a single object, JPQL throws an exception.

## JPQL Joins

### One-to-One

```java
@Table(name = "user_details")
@Entity
public class UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long userId;

    @Column(name = "user_name")
    private String name;
    private String phone;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "user_address")
    private UserAddress userAddress;

    // getters and setters
}
```

```java
@Entity
@Table(name = "user_address")
public class UserAddress {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String street;
    private String city;
    private String state;
    private String country;
    private String pinCode;

    // getters and setters
}
```

```java
@Repository
public interface UserDetailsRepository extends JpaRepository<UserDetails, Long> {
    @Query("SELECT ud FROM UserDetails ud JOIN ud.userAddress ad WHERE ud.name = :userFirstName")
    List<UserDetails> findUserDetailsWithAddress(@Param("userFirstName") String userName);
}
```

We don't need to specify an `ON` clause — JPA figures out the join condition automatically from the entity mapping. This generates:

```sql
select ud1_0.user_id, ud1_0.user_name, ud1_0.phone, ud1_0.user_address
from user_details ud1_0
join user_address ua1_0
    on ua1_0.id = ud1_0.user_address
where ud1_0.user_name = ?
```

### Selecting Specific Fields Across Joined Entities

If we want some fields from one table and some from another, return type becomes `Object[]` (a row per array, one entry per selected column):

```java
@Query("SELECT ud.name, ad.country FROM UserDetails ud JOIN ud.userAddress ad WHERE ud.name = :userFirstName")
List<Object[]> findUserDetailsWithAddress(@Param("userFirstName") String userName);
```

This must be manually parsed, e.g. into a custom DTO:

```java
public class UserDTO {
    String userName;
    String country;

    public UserDTO(String userName, String country) {
        this.userName = userName;
        this.country = country;
    }

    // getters and setters
}
```

```java
public List<UserDTO> findByNameDerived(String name) {
    List<Object[]> dbOutput = userDetailsRepository.findUserDetailsWithAddress(name);
    List<UserDTO> output = new ArrayList<>();
    for (Object[] val : dbOutput) {
        String userName = (String) val[0];
        String country = (String) val[1];
        UserDTO dto = new UserDTO(userName, country);
        output.add(dto);
    }
    return output;
}
```

If we don't want to deal with `Object[]` manually, JPQL supports constructing the DTO directly in the query itself using its fully qualified class name:

```java
@Query("SELECT new com.conceptandcoding.learningspringboot.jpa.DTO.UserDTO(ud.name, ad.country) " +
       "FROM UserDetails ud JOIN ud.userAddress ad WHERE ud.name = :userFirstName")
List<UserDTO> findUserDetailsWithAddress(@Param("userFirstName") String userName);
```

## One-to-Many Joins and the N+1 Problem

```java
@Table(name = "user_details")
@Entity
public class UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long userId;

    @Column(name = "user_name")
    private String name;
    private String phone;

    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = "user_id") // FK in user_address table
    private List<UserAddress> userAddressList = new ArrayList<>();

    // getters and setters
}
```

```java
@Query("SELECT ud FROM UserDetails ud JOIN ud.userAddressList ad WHERE ud.name = :userFirstName")
List<UserDetails> findUserDetailsWithAddress(@Param("userFirstName") String userName);
```

### The Problem

Say one user can have many addresses, and our query fetches more than one user at once (e.g. filtering by a name shared by multiple users). If we have "N" matching users, JPA hits:

- 1 query to fetch all the users.
- For each user, a separate query to fetch its addresses — so N queries for N users.

Total query hits: **N+1**. We want to reduce this to ideally just 1 query.

It's tempting to think switching the relationship to `FetchType.EAGER` would fix this, but it doesn't — eager fetching works fine when fetching a single parent (e.g. `findByID(id)`, where JPA internally drafts one JOIN query for that one parent and its children). But when multiple parents with multiple children are involved, eager loading still first fetches all the parents in one query, then separately fetches each parent's children — so eager loading has the exact same N+1 problem.

### Solution 1: JOIN FETCH

```java
@Query("SELECT ud FROM UserDetails ud JOIN FETCH ud.userAddressList ad WHERE ud.name = :userFirstName")
List<UserDetails> findUserDetailsWithAddress(@Param("userFirstName") String userName);
```

Instead of a plain `JOIN`, `JOIN FETCH` tells JPQL to enforce fetching everything — parent and child — in exactly one query, using a real SQL join, rather than issuing separate queries per parent.

### Solution 2: @BatchSize

```java
@OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
@BatchSize(size = 10)
@JoinColumn(name = "user_id") // FK in user_address table
private List<UserAddress> userAddressList;
```

This doesn't get everything down to a single query, but it reduces the number of queries by batching: even with eager loading, JPA still first fetches all the parent rows in one query, but then fetches children for a whole batch of parents (up to `size`) in a single follow-up query using an `IN (...)` clause, instead of one query per parent.

### Solution 3: @EntityGraph

```java
@EntityGraph(attributePaths = "userAddressList")
List<UserDetails> findUsersBy();
```

Used over a method (helpful for derived query methods, where you can't easily inject a `JOIN FETCH`), `attributePaths` names the child collection to fetch, telling JPA to fetch all entries of that child collection along with the parent details, in one go.

All three solutions above force JPA to fetch all children in a single query alongside the N parent rows, avoiding the N+1 problem.

### Joining Multiple (Chained) Tables

Joining across more than two related tables works almost exactly like SQL. Say Table A has a one-to-many relationship with Table B, and Table B has a one-to-many relationship with Table C:

```java
@Query("SELECT a FROM A a JOIN a.bList b JOIN b.cList c WHERE c.someProperty = :someValue")
List<A> findAWithBAndC(@Param("someValue") String someValue);
```

No `ON` clause is needed for any of the joins — JPA infers each join condition automatically from the entity mappings.

## Modifying Data with JPQL: @Modifying

By default, `@Query` expects a SELECT statement. Attempting to use DELETE, INSERT, or UPDATE directly with `@Query` throws:

```
query.IllegalSelectQueryException: Expecting a SELECT Query
```

`@Modifying` tells JPA to expect a DELETE/INSERT/UPDATE query instead of a SELECT. Since this updates the DB, `@Transactional` is also required:

```java
@Modifying
@Transactional
@Query("DELETE FROM UserDetails ud WHERE ud.name = :userFirstName")
void deleteByUserName(@Param("userFirstName") String userName);
```

## Flush and Clear (Keeping the Persistence Context in Sync)

- **Flush** pushes persistence context changes to the DB, but the persistence context still holds onto those values (its in-memory cache is not cleared).
- **Clear** purges the persistence context entirely, forcing a fresh DB call on the next read.

Without either, a sequence like: find (loads into persistence context) → delete via JPQL → find again can incorrectly return the "old" object, because the second find is served from the persistence context rather than the DB, even though the row was deleted:

```java
@Transactional
public void deleteByUserName(String name) {
    userDetailsRepository.findById(1L).get();
    userDetailsRepository.deleteByUserName(name);
    Optional<UserDetails> output = userDetailsRepository.findById(1L);
    System.out.println("output present: " + output.isPresent()); // prints true, incorrectly
}
```

This happens because the persistence context isn't automatically synced with what actually happened in the DB via the JPQL delete. The fix is `@Modifying(flushAutomatically = true, clearAutomatically = true)`:

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("DELETE FROM UserDetails ud WHERE ud.name = :userFirstName")
void deleteByUserName(@Param("userFirstName") String userName);
```

With this, flush pushes the persistence context state (and the pending delete) to the DB, and clear purges the persistence context — so the second `findById` call is forced to go back to the DB, correctly finding nothing (`output present: false`).

## Pagination and Sorting in JPQL

Works exactly the same way as in Derived Query — `Pageable` can be passed as a method parameter:

```java
@Query("SELECT ud FROM UserDetails ud WHERE ud.name = :userFirstName")
List<UserDetails> findUserDetails(@Param("userFirstName") String userName, Pageable pageable);
```

```java
public List<UserDetails> findByUserName(String name) {
    Pageable page = PageRequest.of(1, 5);
    return userDetailsRepository.findUserDetails(name, page);
}
```

## @NamedQuery

Lets you name a query so it can be reused without repeating the JPQL string. Generally declared on the entity itself:

```java
@Table(name = "user_details")
@Entity
@NamedQuery(name = "findByUserName",
        query = "SELECT u FROM UserDetails u WHERE u.name = :userFirstName")
public class UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long userId;

    @Column(name = "user_name")
    private String name;
    private String phone;

    @OneToOne(cascade = CascadeType.ALL)
    private UserAddress userAddress;

    // getters and setters
}
```

Then in the repository, reference it by name instead of writing the query again:

```java
@Query(name = "findByUserName")
List<UserDetails> findUserDetails(@Param("userFirstName") String userName, Pageable pageable);
```
