# The N+1 Problem in Spring Data JPA

> **The single most common performance pitfall in JPA-based applications.**  
> Understanding it deeply — and knowing every fix — is essential for senior Java backend interviews.

---

## Table of Contents
1. [What Is the N+1 Problem?](#what-is-the-n1-problem)
2. [How to Reproduce It](#how-to-reproduce-it)
3. [How to Detect It](#how-to-detect-it)
4. [Solutions](#solutions)
5. [JOIN FETCH + Pagination Warning](#join-fetch--pagination-warning)
6. [Countermeasures per Relationship Type](#countermeasures-per-relationship-type)
7. [Q&A](#qa)

---

## What Is the N+1 Problem?

The N+1 problem occurs when an ORM issues **1 query** to load a list of parent entities, then **N additional queries** — one per parent — to load a lazily-fetched association.

```
SELECT * FROM authors;                  -- 1 query → 100 rows
SELECT * FROM books WHERE author_id=1;  -- query 1 of N
SELECT * FROM books WHERE author_id=2;  -- query 2 of N
...
SELECT * FROM books WHERE author_id=100;-- query 100 of N
-- Total: 101 queries instead of 1 or 2
```

---

## How to Reproduce It

### Domain Model

```java
@Entity
public class Author {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY) // LAZY is the default
    private List<Book> books = new ArrayList<>();
}

@Entity
public class Book {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
}
```

### Repository

```java
public interface AuthorRepository extends JpaRepository<Author, Long> { }
```

### Service That Triggers N+1

```java
@Service
@Transactional(readOnly = true)
public class AuthorService {

    private final AuthorRepository authorRepository;

    public List<String> getAllBookTitles() {
        List<Author> authors = authorRepository.findAll(); // Query #1

        return authors.stream()
            .flatMap(a -> a.getBooks().stream()) // Each call → Query #2..N+1
            .map(Book::getTitle)
            .toList();
    }
}
```

---

## How to Detect It

### 1. Enable SQL Logging in `application.properties`

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### 2. P6Spy — Log Every Query with Timing

**Dependency:**
```xml
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

**`spy.properties` in `src/main/resources`:**
```properties
driverlist=org.h2.Driver,com.mysql.cj.jdbc.Driver
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
```

**`application.properties` datasource change:**
```properties
spring.datasource.url=jdbc:p6spy:mysql://localhost:3306/mydb
spring.datasource.driver-class-name=com.p6spy.engine.spy.P6SpyDriver
```

### 3. Query Count Assertions in Tests (datasource-proxy)

```xml
<dependency>
    <groupId>net.ttddyy</groupId>
    <artifactId>datasource-proxy</artifactId>
    <version>1.9</version>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
class AuthorServiceTest {

    @Autowired
    private AuthorService authorService;

    @Autowired
    private DataSource dataSource;

    private ProxyTestDataSource proxyDataSource;

    @BeforeEach
    void setUp() {
        proxyDataSource = new ProxyTestDataSource(dataSource);
    }

    @Test
    void shouldNotTriggerNPlusOneQueries() {
        authorService.getAllBookTitles();

        // Assert only 1 or 2 queries were executed (not N+1)
        assertThat(proxyDataSource.getQueryExecutionFactList()).hasSizeLessThanOrEqualTo(2);
    }
}
```

### 4. Hibernate Statistics

```properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```

```java
@Autowired
private EntityManagerFactory entityManagerFactory;

Statistics stats = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
long queryCount = stats.getQueryExecutionCount();
```

---

## Solutions

### Solution 1: JOIN FETCH in JPQL

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}
```

**Result:** Single SQL query with JOIN — books loaded in one shot.

```sql
SELECT a.*, b.*
FROM author a
INNER JOIN book b ON b.author_id = a.id
```

**Limitation:** Fetches only authors that **have** at least one book (INNER JOIN). Use `LEFT JOIN FETCH` to include authors with no books:

```java
@Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
List<Author> findAllWithBooks();
```

> **Note:** `DISTINCT` prevents duplicate `Author` objects in the result list (one per joined `Book` row). As of Hibernate 6, `DISTINCT` in JPQL no longer passes through to SQL, so no performance penalty.

---

### Solution 2: @EntityGraph

Declarative — no JPQL needed:

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    List<Author> findAll();

    // Works with derived query methods too
    @EntityGraph(attributePaths = {"books"})
    Optional<Author> findById(Long id);
}
```

**Named EntityGraph on the entity class:**

```java
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author { ... }
```

```java
@EntityGraph("Author.withBooks")
List<Author> findAll();
```

---

### Solution 3: @BatchSize on Collection

Hibernate fetches associations in batches instead of one-by-one:

```java
@Entity
public class Author {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    @BatchSize(size = 25) // Fetch books for up to 25 authors at a time
    private List<Book> books;
}
```

**SQL generated (for 100 authors, batch size 25):**
```sql
SELECT * FROM book WHERE author_id IN (1,2,3,...,25)  -- batch 1
SELECT * FROM book WHERE author_id IN (26,27,...,50)  -- batch 2
SELECT * FROM book WHERE author_id IN (51,52,...,75)  -- batch 3
SELECT * FROM book WHERE author_id IN (76,77,...,100) -- batch 4
-- Total: 4 queries instead of 100
```

**Global default batch size:**
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

---

### Solution 4: DTO Projection

Fetch only the data you need — avoids loading full entities entirely.

**Interface Projection:**
```java
public interface BookSummary {
    String getTitle();
    String getAuthorName();
}
```

```java
public interface BookRepository extends JpaRepository<Book, Long> {

    @Query("SELECT b.title AS title, b.author.name AS authorName FROM Book b")
    List<BookSummary> findAllBookSummaries();
}
```

**Class (DTO) Projection:**
```java
public record AuthorBookDTO(String authorName, String bookTitle) {}
```

```java
@Query("""
    SELECT new com.example.dto.AuthorBookDTO(a.name, b.title)
    FROM Author a
    JOIN a.books b
    """)
List<AuthorBookDTO> findAllAuthorBooks();
```

**Spring Data Projection with `@Value` SpEL:**
```java
public interface AuthorView {
    String getName();

    @Value("#{target.books.size()}")
    int getBookCount();
}
```

---

### Solution 5: @Fetch(FetchMode.SUBSELECT)

Hibernate uses a subquery to fetch all related collections in a second query:

```java
@Entity
public class Author {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    @Fetch(FetchMode.SUBSELECT)
    private List<Book> books;
}
```

**SQL generated:**
```sql
SELECT * FROM author;                                 -- Query 1
SELECT * FROM book WHERE author_id IN               -- Query 2
    (SELECT id FROM author);
```

**Trade-off:** Always fetches all collections regardless of which authors you actually access — may load more than needed.

---

## JOIN FETCH + Pagination Warning

```java
// ⚠️ DANGEROUS — triggers HHH90003004 warning
@Query("SELECT a FROM Author a JOIN FETCH a.books")
Page<Author> findAllWithBooks(Pageable pageable);
```

**Hibernate Warning:**
```
HHH90003004: firstResult/maxResults specified with collection fetch;
applying in memory!
```

Hibernate cannot apply `LIMIT`/`OFFSET` at the database level when joining a collection (row count multiplied), so it loads **all rows into memory** and paginates in Java — catastrophic for large tables.

### Fix: Two-Query Approach

```java
// Step 1: paginate IDs only
@Query(
    value = "SELECT a.id FROM Author a",
    countQuery = "SELECT COUNT(a) FROM Author a"
)
Page<Long> findAuthorIds(Pageable pageable);

// Step 2: fetch full entities with JOIN FETCH using those IDs
@Query("SELECT a FROM Author a JOIN FETCH a.books WHERE a.id IN :ids")
List<Author> findByIdWithBooks(@Param("ids") List<Long> ids);
```

### Fix with @EntityGraph + Separate Count Query

```java
@Query(
    value = "SELECT a FROM Author a",
    countQuery = "SELECT COUNT(a) FROM Author a"
)
@EntityGraph(attributePaths = {"books"})
Page<Author> findAll(Pageable pageable);
```

> Hibernate 6 handles this better via `HibernateJpaDialect`, but the two-query approach is the safest for all versions.

---

## Countermeasures per Relationship Type

| Relationship | Default Fetch | Recommended Fix |
|---|---|---|
| `@ManyToOne` | EAGER | Change to LAZY + JOIN FETCH when needed |
| `@OneToOne` (owner) | EAGER | Change to LAZY + JOIN FETCH or @EntityGraph |
| `@OneToOne` (inverse) | LAZY (can be EAGER) | Keep LAZY; use JOIN FETCH |
| `@OneToMany` | LAZY | JOIN FETCH or @BatchSize |
| `@ManyToMany` | LAZY | JOIN FETCH or @BatchSize (prefer @OneToMany with junction entity) |

> **Golden Rule:** All associations should be `FetchType.LAZY`. Fetch eagerly only when needed, and explicitly via JPQL/EntityGraph.

---

## Q&A

---

🟢 **Q1: What is the N+1 problem in JPA?**

<details><summary>Click to reveal answer</summary>

The N+1 problem occurs when an application executes **1 query** to retrieve a list of parent entities and then **N additional queries** (one per parent) to fetch a lazily-loaded association. For example, loading 100 authors and then triggering 100 separate queries for their books results in 101 total queries instead of 1 efficient JOIN query.

</details>

---

🟢 **Q2: What is `FetchType.LAZY` vs `FetchType.EAGER`?**

<details><summary>Click to reveal answer</summary>

- **LAZY**: The association is loaded on-demand, only when accessed. This is the JPA default for `@OneToMany` and `@ManyToMany`.
- **EAGER**: The association is always loaded immediately with the parent entity. This is the default for `@ManyToOne` and `@OneToOne`.

**Best practice**: Always use `LAZY` for all associations and fetch eagerly only when needed via JOIN FETCH or `@EntityGraph`. EAGER loading hides N+1 in some cases but can cause over-fetching in others.

```java
// Preferred default — lazy everywhere
@ManyToOne(fetch = FetchType.LAZY)
private Author author;

@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
private List<Book> books;
```

</details>

---

🟢 **Q3: How do you enable SQL logging in a Spring Boot application?**

<details><summary>Click to reveal answer</summary>

```properties
# application.properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

`show-sql` prints queries to stdout; `org.hibernate.SQL=DEBUG` routes them through SLF4J. The `BasicBinder` logger shows bound parameter values. For production diagnostics, prefer P6Spy or datasource-proxy.

</details>

---

🟡 **Q4: How does `JOIN FETCH` solve the N+1 problem?**

<details><summary>Click to reveal answer</summary>

`JOIN FETCH` instructs Hibernate to load the association in the **same query** as the parent entity using a SQL JOIN, eliminating extra round-trips.

```java
@Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
List<Author> findAllWithBooks();
```

Generated SQL:
```sql
SELECT DISTINCT a.*, b.*
FROM author a
LEFT JOIN book b ON b.author_id = a.id
```

`DISTINCT` in JPQL prevents duplicate `Author` references in the result list (one per joined book row). In Hibernate 6, this no longer adds SQL-level DISTINCT, so there is no performance concern.

</details>

---

🟡 **Q5: What is `@EntityGraph` and how does it differ from `JOIN FETCH`?**

<details><summary>Click to reveal answer</summary>

`@EntityGraph` is a declarative alternative to `JOIN FETCH`. You specify which attributes to eagerly fetch without writing JPQL.

```java
@EntityGraph(attributePaths = {"books", "books.publisher"})
List<Author> findAll();
```

**Differences:**
| | JOIN FETCH | @EntityGraph |
|---|---|---|
| Style | JPQL | Declarative annotation |
| Works with derived queries | No | Yes |
| Nested paths | Verbose | Simple (`"books.publisher"`) |
| Multiple collections | Risks Cartesian product | Same risk |

Both generate an SQL JOIN under the hood. `@EntityGraph` is preferred for its clarity with Spring Data derived query methods.

</details>

---

🟡 **Q6: What does `@BatchSize` do and when should you use it?**

<details><summary>Click to reveal answer</summary>

`@BatchSize` tells Hibernate to load lazy associations in batches using SQL `IN` clauses instead of individual queries.

```java
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
@BatchSize(size = 25)
private List<Book> books;
```

For 100 authors with `batchSize=25`:
- Without: 101 queries
- With: 5 queries (1 + 4 batch queries)

**Best for:** Situations where JOIN FETCH is impractical (e.g., multiple collection fetches, or when pagination is needed). Set globally with:

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

</details>

---

🟡 **Q7: Why is using multiple JOIN FETCHes on different collections problematic?**

<details><summary>Click to reveal answer</summary>

Fetching two bag (List) collections simultaneously causes a **Cartesian product** in SQL:

```java
// Author has books (List) AND awards (List)
@Query("SELECT a FROM Author a JOIN FETCH a.books JOIN FETCH a.awards")
List<Author> findAll(); // ⚠️ MultipleBagFetchException or Cartesian product
```

Hibernate throws `MultipleBagFetchException` if both are `List`. Workarounds:
1. Change one collection to `Set` (but loses ordering).
2. Use two separate queries — one JOIN FETCH per collection.
3. Use `@BatchSize` for one or both collections.

```java
// Safe: fetch books via JOIN FETCH, awards via @BatchSize
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// awards annotated with @BatchSize(size=25) — loaded lazily in batches
```

</details>

---

🟡 **Q8: What is a DTO projection and how does it prevent N+1?**

<details><summary>Click to reveal answer</summary>

A DTO projection retrieves only the columns needed via a JPQL constructor expression or interface projection — no entity graph is loaded, so no lazy associations are triggered.

```java
public record BookDTO(Long id, String title, String authorName) {}

@Query("""
    SELECT new com.example.dto.BookDTO(b.id, b.title, a.name)
    FROM Book b
    JOIN b.author a
    """)
List<BookDTO> findAllBookDTOs();
```

Since `BookDTO` is not a managed entity, Hibernate does not track it and will never issue extra queries for associations. This is the most efficient approach when you don't need to modify the data.

</details>

---

🟡 **Q9: What is `FetchMode.SUBSELECT` and when is it appropriate?**

<details><summary>Click to reveal answer</summary>

`@Fetch(FetchMode.SUBSELECT)` causes Hibernate to load all collections for the loaded parent entities in a **single second query** using a subselect.

```java
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
@Fetch(FetchMode.SUBSELECT)
private List<Book> books;
```

```sql
-- Query 1
SELECT * FROM author WHERE ...;

-- Query 2 (triggered on first access to any author's books)
SELECT * FROM book
WHERE author_id IN (SELECT id FROM author WHERE ...);
```

**When to use:** Useful when you've already loaded a list of authors and know you'll need all their books. **Avoid** when you only need books for a subset of authors (it fetches books for all of them regardless).

</details>

---

🔴 **Q10: Why does `JOIN FETCH` with pagination cause the HHH90003004 warning?**

<details><summary>Click to reveal answer</summary>

When a JPQL query with `JOIN FETCH` on a collection is combined with `Pageable`, Hibernate cannot apply `LIMIT`/`OFFSET` at the SQL level. Each author maps to multiple rows (one per book), so the row count does not equal the entity count.

Hibernate's workaround: load **all results into memory**, then paginate in Java.

```
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!
```

**Fix: Two-query pattern**

```java
// 1. Paginate by entity ID
@Query(value = "SELECT a.id FROM Author a", countQuery = "SELECT COUNT(a) FROM Author a")
Page<Long> findAuthorIds(Pageable pageable);

// 2. Fetch full data for those IDs
@Query("SELECT a FROM Author a JOIN FETCH a.books WHERE a.id IN :ids")
List<Author> findByIdsWithBooks(@Param("ids") List<Long> ids);
```

```java
// Service layer
public Page<Author> getAuthorsWithBooks(Pageable pageable) {
    Page<Long> idPage = authorRepository.findAuthorIds(pageable);
    List<Author> authors = authorRepository.findByIdsWithBooks(idPage.getContent());
    return new PageImpl<>(authors, pageable, idPage.getTotalElements());
}
```

</details>

---

🟡 **Q11: What is the difference between `@NamedEntityGraph` and inline `@EntityGraph`?**

<details><summary>Click to reveal answer</summary>

**`@NamedEntityGraph`** is defined on the entity class and referenced by name:

```java
@Entity
@NamedEntityGraph(
    name = "Author.withBooksAndPublisher",
    attributeNodes = {
        @NamedAttributeNode(value = "books", subgraph = "books-subgraph")
    },
    subgraphs = @NamedSubgraph(
        name = "books-subgraph",
        attributeNodes = @NamedAttributeNode("publisher")
    )
)
public class Author { ... }
```

```java
@EntityGraph("Author.withBooksAndPublisher")
List<Author> findAll();
```

**Inline `@EntityGraph`** is defined directly on the repository method:

```java
@EntityGraph(attributePaths = {"books", "books.publisher"})
List<Author> findAll();
```

**Recommendation:** Use inline `@EntityGraph` for simplicity unless you need to share the graph definition across multiple repositories.

</details>

---

🟢 **Q12: Is the N+1 problem only related to `@OneToMany`?**

<details><summary>Click to reveal answer</summary>

No. N+1 can occur with any lazily-fetched association:

- **`@ManyToOne` (EAGER by default!)**: If changed to LAZY, accessing `book.getAuthor()` in a loop triggers N queries.
- **`@OneToOne` (inverse side)**: The inverse side cannot be lazily loaded by proxy in some JPA implementations without bytecode enhancement.
- **`@ManyToMany`**: Accessing the join table for each entity in a loop.

The "1" can also itself be a result of a previous N+1, creating cascading performance problems. Always check all associations, not just `@OneToMany`.

</details>

---

🟢 **Q13: What is the "Open Session in View" (OSIV) pattern and how does it relate to N+1?**

<details><summary>Click to reveal answer</summary>

OSIV keeps the Hibernate session open through the entire HTTP request lifecycle (including the view rendering layer). Spring Boot enables it by default:

```properties
spring.jpa.open-in-view=true  # default
```

**Problem:** This allows lazy loading to succeed in view/controller layers, masking N+1 problems during development. The queries happen silently in the view.

**Recommended:** Disable OSIV in production APIs and load all required data in the service layer:

```properties
spring.jpa.open-in-view=false
```

This forces `LazyInitializationException` to surface during testing, making N+1 issues visible and requiring explicit fixes.

</details>

---

🔴 **Q14: How do you write an integration test to assert no N+1 queries occur?**

<details><summary>Click to reveal answer</summary>

Using `datasource-proxy` library:

```java
@SpringBootTest
@Transactional
class AuthorServiceIntegrationTest {

    @Autowired
    private AuthorService authorService;

    @Autowired
    private DataSource dataSource;

    @Test
    void findAllWithBooks_shouldExecuteExactlyTwoQueries() {
        // Arrange: datasource-proxy wraps the DataSource
        ProxyTestDataSource proxyDataSource = new ProxyTestDataSource(dataSource);

        // Act
        authorService.findAllAuthorsWithBooks();

        // Assert
        JdbcProxyAssertions.assertThat(proxyDataSource)
            .hasSelectCount(1);  // Only one SELECT, not N+1
    }
}
```

Using Hibernate statistics:

```java
@Test
void shouldNotIssueMoreThanTwoQueries() {
    Statistics stats = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
    stats.clear();

    authorService.findAllAuthorsWithBooks();

    assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(2);
}
```

</details>

---

🟡 **Q15: How does Hibernate `DISTINCT` in JPQL differ from SQL `DISTINCT`?**

<details><summary>Click to reveal answer</summary>

In JPQL, `DISTINCT` removes **duplicate entity references** from the result list (not duplicate SQL rows). When you JOIN FETCH a collection, each parent entity appears once per child row in SQL, causing duplicates in the Java list.

```java
// Without DISTINCT: authors list may have [Author1, Author1, Author2, Author2, Author2]
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// With DISTINCT: [Author1, Author2]
@Query("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();
```

In **Hibernate 6+**, JPQL `DISTINCT` no longer propagates to SQL (it passes a `QueryHint` to deduplicate in memory instead), so there is no SQL-level performance concern.

In **Hibernate 5**, you can achieve the same with a query hint:

```java
@Query("SELECT a FROM Author a JOIN FETCH a.books")
@QueryHints(@QueryHint(name = "hibernate.query.passDistinctThrough", value = "false"))
List<Author> findAllWithBooks();
```

</details>

---

🟡 **Q16: What are the trade-offs between JOIN FETCH and @BatchSize?**

<details><summary>Click to reveal answer</summary>

| Criterion | JOIN FETCH | @BatchSize |
|---|---|---|
| Number of queries | 1 | ceil(N / batchSize) + 1 |
| Pagination safe | ❌ No (HHH90003004) | ✅ Yes |
| Multiple collections | ❌ Cartesian product risk | ✅ Safe |
| Selectivity | Loads all, always | Loads on access |
| Setup effort | JPQL change | Annotation on field |
| Best for | Single collection, no pagination | Pagination, multiple collections |

**Recommendation:** Use JOIN FETCH for simple "load all with association" queries. Use `@BatchSize` (or the global `default_batch_fetch_size`) as a safe default for paginated queries.

</details>

---

🟢 **Q17: What happens if you access a lazy association outside of a transaction?**

<details><summary>Click to reveal answer</summary>

You get a `LazyInitializationException`:

```
org.hibernate.LazyInitializationException: 
  failed to lazily initialize a collection of role: 
  com.example.Author.books, could not initialize proxy - no Session
```

**Solutions:**
1. Ensure the access happens within a `@Transactional` method.
2. Use JOIN FETCH or `@EntityGraph` to load eagerly within the transaction.
3. Disable OSIV carefully if it was masking this.
4. Use DTOs or projections to avoid entity associations entirely.

</details>

---

🔴 **Q18: How would you handle N+1 in a Spring Batch job processing millions of records?**

<details><summary>Click to reveal answer</summary>

In Spring Batch, use `JpaPagingItemReader` with a JOIN FETCH query and a reasonable chunk size:

```java
@Bean
public JpaPagingItemReader<Author> authorItemReader() {
    return new JpaPagingItemReaderBuilder<Author>()
        .name("authorReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
        .pageSize(100)
        .build();
}
```

**Key considerations:**
- `JpaPagingItemReader` opens a new `EntityManager` per page — JOIN FETCH is safe here because pagination happens at the ID level per page fetch.
- Set `spring.jpa.properties.hibernate.default_batch_fetch_size=100` as a belt-and-suspenders measure.
- Consider using a native SQL query with a `ResultSetExtractor` for maximum throughput in heavy batch scenarios.
- Use `@QueryHints` with `HINT_FETCH_SIZE` to control JDBC fetch size:

```java
.queryString("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
.queryHints(Map.of("org.hibernate.fetchSize", "100"))
```

</details>

---

🟡 **Q19: Can `@ManyToOne` (EAGER by default) cause N+1? How?**

<details><summary>Click to reveal answer</summary>

Yes. While `@ManyToOne` is EAGER by default (which loads the association in the same query via a JOIN), problems arise when:

1. You change it to `LAZY` for performance and forget to JOIN FETCH it.
2. You use `findAll()` with a Specification or criteria query that doesn't include the JOIN — Hibernate may issue individual SELECTs per book to load the associated author.

```java
// Book.author changed to LAZY
@ManyToOne(fetch = FetchType.LAZY)
private Author author;

// Service
List<Book> books = bookRepository.findAll(); // Query 1
books.forEach(b -> System.out.println(b.getAuthor().getName())); // N queries for authors
```

**Fix:**
```java
@Query("SELECT b FROM Book b JOIN FETCH b.author")
List<Book> findAllWithAuthor();
```

</details>

---

🔴 **Q20: What is the `@OneToOne` lazy loading problem and why is it hard to fix?**

<details><summary>Click to reveal answer</summary>

The **inverse side** of a `@OneToOne` relationship cannot be truly lazy-loaded by Hibernate's default proxy mechanism. Hibernate needs to know whether the association is `null` or not, which requires a query — making it effectively EAGER.

```java
@Entity
public class User {
    @Id Long id;

    // This is the OWNER side — can be LAZY (uses FK column)
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id Long id;

    // INVERSE side — cannot be truly LAZY without bytecode enhancement
    @OneToOne(mappedBy = "profile", fetch = FetchType.LAZY)
    private User user; // Still loaded eagerly by proxy initialization check
}
```

**Solutions:**
1. **Bytecode enhancement** (Hibernate build plugin) — enables true field-level lazy loading.
2. **Avoid bidirectional `@OneToOne`** — use `@ManyToOne` on one side if possible, or restructure.
3. **Use DTO projections** to avoid the relationship entirely in read queries.
4. **`@LazyToOne(LazyToOneOption.NO_PROXY)`** with bytecode instrumentation.

```xml
<!-- Bytecode enhancement in pom.xml -->
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <configuration>
        <enableLazyInitialization>true</enableLazyInitialization>
    </configuration>
</plugin>
```

</details>

---

🟡 **Q21: How does Hibernate's second-level cache interact with the N+1 problem?**

<details><summary>Click to reveal answer</summary>

The second-level cache (2LC) can **mitigate** N+1 by caching entity lookups, but it doesn't eliminate the extra queries from Hibernate's perspective — it just serves them from cache instead of the database.

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Author { ... }
```

If `Author` is cached:
- First request: 1 + N queries (N cache misses → DB)
- Subsequent requests: 1 query for IDs + N cache hits (no DB roundtrip)

**Important:** This is **not** a real fix for N+1 — it's a band-aid. Under cache pressure or after eviction, the full N+1 pattern returns. Always fix the root cause with JOIN FETCH or @BatchSize.

</details>

---

🟢 **Q22: What is the simplest global configuration to reduce N+1 impact across all entities?**

<details><summary>Click to reveal answer</summary>

Set the global default batch fetch size in `application.properties`:

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

This applies `@BatchSize(size=25)` behavior to **all** lazy collections and proxies without changing any entity code. For 100 parent entities:
- Without: 101 queries
- With size=25: 5 queries (1 + 4 batch queries)

This is the recommended "baseline safety net" for Spring Boot applications, combined with explicit JOIN FETCH for known hot paths.

</details>

---

*Last updated: 2025 | Spring Boot 3.x | Hibernate 6.x | Java 21*
