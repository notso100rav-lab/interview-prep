# Hibernate & JPA Internals

> 💡 **Interviewer Tip:** JPA internals separate developers who "use Hibernate" from those who "understand Hibernate." Key areas: entity state transitions, dirty checking, the persistence context lifecycle, and the difference between JPQL and native queries.

---

## Table of Contents
1. [Entity Lifecycle States](#entity-lifecycle-states)
2. [Persistence Context](#persistence-context)
3. [Relationships and Mappings](#relationships-and-mappings)
4. [Interview Questions](#interview-questions)

---

## Entity Lifecycle States

```
new/transient  ──persist()──→  managed/persistent
                                     │
                    remove()         │  detach() / clear() / close()
                       ↓             ↓
                    removed      detached
                       │             │
                    commit()      merge()
                       ↓             ↓
                   (deleted)      managed
```

## Persistence Context

The persistence context is Hibernate's **first-level cache** — a map of all managed entities within a Session/EntityManager. It:
- Caches entity reads (avoids redundant DB hits within same transaction)
- Tracks changes (dirty checking) for automatic UPDATE on flush
- Ensures identity guarantee (same DB row → same Java object)

---

## Interview Questions

### 🟢 Easy

---

**Q1. What are the four states of a JPA entity?**

<details><summary>Click to reveal answer</summary>

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;

    // constructors, getters, setters
}

@Service
public class EntityStateDemo {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void demonstrateStates() {

        // 1. TRANSIENT — not associated with any persistence context
        User user = new User();
        user.setName("John");
        // user has no id, not tracked by Hibernate, not in DB

        // 2. MANAGED (persistent) — associated with persistence context
        em.persist(user);
        // user now has an id (auto-generated)
        // Hibernate tracks all changes to user
        // SELECT not yet executed; entity buffered for INSERT

        // 3. DETACHED — was managed, now disconnected
        em.detach(user);
        // user still has its id and data
        // but changes are NOT tracked anymore
        // modifications will not be synced to DB

        user.setName("Jane"); // change not tracked!

        // 4. REMOVED
        User managedUser = em.find(User.class, user.getId());
        em.remove(managedUser);
        // Marked for deletion; DELETE executed on flush/commit
        // managedUser transitions to removed state
    }

    // State transitions:
    // transient  → managed:  em.persist(entity)
    // managed    → detached: em.detach(entity) or em.clear() or end of tx
    // managed    → removed:  em.remove(entity)
    // detached   → managed:  em.merge(entity)  ← returns new managed instance
    // removed    → managed:  em.persist(removedEntity) (if still in persistence context)
}
```

🚨 **Common Mistake:** Calling `em.merge()` and expecting the original object to become managed:
```java
User detached = ...; // detached entity
User managed = em.merge(detached); // ✅ managed is the new managed copy
// detached is still DETACHED — only managed is tracked!
em.remove(detached); // ❌ This throws IllegalArgumentException
em.remove(managed);  // ✅ This works
```

</details>

---

**Q2. What is the difference between `@Id @GeneratedValue` strategies?**

<details><summary>Click to reveal answer</summary>

```java
// AUTO — lets JPA choose the strategy (depends on DB, usually SEQUENCE)
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;

// IDENTITY — uses DB auto-increment column (MySQL, PostgreSQL SERIAL)
// Downside: requires INSERT to get the ID → can't batch inserts
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// SEQUENCE — uses a DB sequence (PostgreSQL, Oracle)
// Allows pre-allocation of IDs → enables batch inserts
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_id_seq",
                   allocationSize = 50)  // pre-allocates 50 IDs at once
private Long id;

// TABLE — uses a separate table to simulate sequences (rarely used)
// Very slow due to table-level locking
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "user_table_gen")
@TableGenerator(name = "user_table_gen", table = "id_generator",
                pkColumnName = "gen_name", valueColumnName = "gen_value",
                pkColumnValue = "user_id", allocationSize = 1)
private Long id;

// UUID (Hibernate 6 / JPA 3.1)
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;

// Custom Generator
@Id
@GenericGenerator(name = "nanoid-gen", strategy = "com.example.NanoIdGenerator")
@GeneratedValue(generator = "nanoid-gen")
private String id;
```

✅ **Best Practice:** Use `SEQUENCE` with `allocationSize > 1` for high-throughput applications — it minimizes round trips to the DB for ID generation and enables JDBC batch inserts.

</details>

---

**Q3. What are the basic JPA mapping annotations?**

<details><summary>Click to reveal answer</summary>

```java
@Entity                         // marks class as JPA entity
@Table(name = "products",       // optional: customize table name
       schema = "inventory",
       indexes = {
           @Index(name = "idx_product_sku", columnList = "sku"),
           @Index(name = "idx_product_category", columnList = "category_id, active")
       },
       uniqueConstraints = {
           @UniqueConstraint(name = "uk_product_sku", columnNames = "sku")
       })
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "product_seq")
    @SequenceGenerator(name = "product_seq",
                       sequenceName = "product_id_seq",
                       allocationSize = 50)
    private Long id;

    @Column(name = "product_name",  // column name
            nullable = false,       // NOT NULL constraint
            length = 200,           // VARCHAR(200)
            unique = false)
    private String name;

    @Column(name = "sku",
            unique = true,
            length = 50,
            updatable = false)      // immutable after creation
    private String sku;

    @Column(precision = 10, scale = 2)  // DECIMAL(10,2)
    private BigDecimal price;

    @Column(columnDefinition = "TEXT")  // custom column DDL
    private String description;

    @Enumerated(EnumType.STRING)   // store enum as string (not ordinal!)
    @Column(nullable = false)
    private ProductStatus status;

    @Temporal(TemporalType.TIMESTAMP)   // for java.util.Date (use LocalDateTime instead)
    private Date legacyDate;

    // Modern: LocalDate, LocalDateTime — no @Temporal needed
    private LocalDateTime createdAt;

    @Transient  // not persisted to database
    private String computedField;

    @Lob        // Large Object (BLOB/CLOB)
    private byte[] thumbnail;

    @Embedded   // inline another class's fields into this table
    private Address address;

    @Version    // optimistic locking
    private Long version;
}

@Embeddable
public class Address {
    private String street;
    private String city;
    @Column(name = "zip_code", length = 10)
    private String zipCode;
}
```

🚨 **Common Mistake:** Using `@Enumerated(EnumType.ORDINAL)` (the default). This stores the enum's position (0, 1, 2...) which breaks if you insert a new enum value in the middle of the enum definition.

</details>

---

**Q4. What is the difference between `CascadeType.PERSIST`, `CascadeType.MERGE`, and `CascadeType.ALL`?**

<details><summary>Click to reveal answer</summary>

```java
@Entity
public class Order {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // CascadeType.PERSIST: persisting Order also persists its items
    // CascadeType.MERGE: merging Order also merges its items
    // CascadeType.REMOVE: removing Order also removes its items
    // CascadeType.REFRESH: refreshing Order also refreshes its items
    // CascadeType.DETACH: detaching Order also detaches its items
    // CascadeType.ALL: all of the above

    @OneToMany(mappedBy = "order",
               cascade = CascadeType.ALL,  // persist, merge, remove, refresh, detach
               orphanRemoval = true)        // remove items not in collection
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this); // maintain bidirectional consistency
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null); // orphanRemoval will delete it from DB
    }
}

@Entity
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY) // default for @ManyToOne is EAGER — override it!
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    private Integer quantity;
    private BigDecimal unitPrice;
}

// Usage:
@Transactional
public Order createOrder(CreateOrderRequest req) {
    Order order = new Order();
    req.getItems().forEach(itemReq -> {
        OrderItem item = new OrderItem();
        item.setQuantity(itemReq.getQuantity());
        item.setUnitPrice(itemReq.getUnitPrice());
        order.addItem(item); // cascade persist handles item saving
    });
    return orderRepo.save(order); // saves order AND all items (CASCADE.PERSIST)
}
```

**`CascadeType.ALL` vs `orphanRemoval`:**
```java
// CascadeType.REMOVE — deletes children when parent is deleted
// orphanRemoval = true — deletes children when removed from parent's collection

order.getItems().clear(); // with orphanRemoval=true → DELETE for all items
em.remove(order);         // with CascadeType.REMOVE → also DELETE for all items
```

</details>

---

### 🟡 Medium

---

**Q5. How does Hibernate dirty checking work?**

<details><summary>Click to reveal answer</summary>

Dirty checking is Hibernate's mechanism to detect changes in managed entities and automatically generate UPDATE SQL without explicit `save()` calls.

```java
@Entity
public class Product {
    @Id @GeneratedValue private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;
}

@Service
public class ProductService {

    @PersistenceContext
    private EntityManager em;

    @Autowired private ProductRepository repo;

    @Transactional
    public void updateProduct(Long id, String newName, BigDecimal newPrice) {
        Product product = repo.findById(id).orElseThrow();
        // product is now MANAGED by the persistence context

        product.setName(newName);
        product.setPrice(newPrice);
        // No explicit save needed!
        // Hibernate records a "snapshot" at load time
        // At flush, it compares current state vs snapshot
        // Differences → generates UPDATE SQL

        // Transaction commits → Hibernate flushes → UPDATE product SET name=?, price=? WHERE id=?
    }

    @Transactional
    public void demonstrateSnapshot() {
        Product p = repo.findById(1L).orElseThrow();
        // Hibernate stores snapshot: {id=1, name="Old", price=100, stock=50}

        p.setName("New"); // change detected vs snapshot

        // Before query (AUTO flush mode), Hibernate compares:
        // current: {id=1, name="New", price=100, stock=50}
        // snapshot: {id=1, name="Old", price=100, stock=50}
        // → generates: UPDATE product SET name='New' WHERE id=1

        List<Product> products = repo.findAll(); // flush happens here
    }
}
```

**How Hibernate implements dirty checking:**
1. At entity load, Hibernate stores a **snapshot** (deep copy) of all field values
2. At flush time, it compares current field values vs snapshot
3. For changed fields, it generates an UPDATE statement

**Performance note:**
- With many entities, dirty checking can be expensive
- `@Transactional(readOnly = true)` disables dirty checking for read-only operations
- `@DynamicUpdate` generates UPDATE only for changed columns:

```java
@Entity
@DynamicUpdate // generates: UPDATE SET name=? WHERE id=? (only changed columns)
public class Product { /* ... */ }
// Without @DynamicUpdate: UPDATE SET name=?, price=?, stock=? WHERE id=? (all columns)
```

</details>

---

**Q6. What is the difference between JPQL, Criteria API, and native queries?**

<details><summary>Click to reveal answer</summary>

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // JPQL — object-oriented query language (entity names, not table names)
    @Query("SELECT p FROM Product p WHERE p.category.name = :categoryName AND p.price < :maxPrice")
    List<Product> findByCategoryAndMaxPrice(
        @Param("categoryName") String categoryName,
        @Param("maxPrice") BigDecimal maxPrice
    );

    // JPQL with JOIN FETCH (prevents N+1)
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
    List<Order> findOrdersWithItemsByUser(@Param("userId") Long userId);

    // Native SQL — use when JPQL can't express the query
    @Query(value = """
        SELECT p.*, c.name as category_name
        FROM products p
        JOIN categories c ON p.category_id = c.id
        WHERE p.created_at > NOW() - INTERVAL '7 days'
        ORDER BY p.created_at DESC
        """,
        nativeQuery = true)
    List<Object[]> findRecentProducts();

    // Native with DTO projection
    @Query(value = "SELECT p.id, p.name, p.price FROM products p WHERE p.active = true",
           nativeQuery = true)
    List<ProductSummary> findActiveSummaries(); // interface-based projection
}

// Interface-based projection (works with both JPQL and native)
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();

    @Value("#{target.name + ' (' + target.id + ')'}")
    String getDisplayName(); // SpEL expression
}
```

**Criteria API — type-safe, programmatic queries:**
```java
@Repository
public class ProductCriteriaRepository {

    @PersistenceContext
    private EntityManager em;

    public List<Product> findWithCriteria(ProductFilter filter) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> query = cb.createQuery(Product.class);
        Root<Product> product = query.from(Product.class);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getName() != null) {
            predicates.add(cb.like(
                cb.lower(product.get("name")),
                "%" + filter.getName().toLowerCase() + "%"
            ));
        }

        if (filter.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                product.get("price"), filter.getMinPrice()
            ));
        }

        if (filter.getCategoryId() != null) {
            Join<Product, Category> category = product.join("category");
            predicates.add(cb.equal(category.get("id"), filter.getCategoryId()));
        }

        query.select(product)
             .where(predicates.toArray(new Predicate[0]))
             .orderBy(cb.desc(product.get("createdAt")));

        return em.createQuery(query)
            .setMaxResults(filter.getLimit())
            .getResultList();
    }
}
```

**When to use which:**

| Query Type | When to Use |
|-----------|-------------|
| JPQL | Most queries; benefits from entity mapping and portability |
| Criteria API | Dynamic, programmatic queries with optional filters |
| Native SQL | DB-specific functions, complex CTEs, performance-critical queries |
| Projections | Read-only subsets of data (DTOs) |

</details>

---

**Q7. How do `@OneToMany` and `@ManyToOne` work, and what is the owning side?**

<details><summary>Click to reveal answer</summary>

In a bidirectional relationship, one side **owns** the foreign key — this is the side that controls the join column in the database.

```java
@Entity
public class Department {
    @Id @GeneratedValue private Long id;
    private String name;

    // Non-owning side (mappedBy points to the field in the owning side)
    // "mappedBy" means: "the foreign key is on the other side"
    @OneToMany(mappedBy = "department",
               cascade = {CascadeType.PERSIST, CascadeType.MERGE},
               fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();

    // Convenience methods to maintain bidirectional consistency
    public void addEmployee(Employee emp) {
        employees.add(emp);
        emp.setDepartment(this);
    }

    public void removeEmployee(Employee emp) {
        employees.remove(emp);
        emp.setDepartment(null);
    }
}

@Entity
public class Employee {
    @Id @GeneratedValue private Long id;
    private String name;

    // OWNING side — has the @JoinColumn, controls the foreign key
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id", nullable = false)
    private Department department;
}
```

**Why the owning side matters:**
```java
@Transactional
public void demonstrateOwnership() {
    Department dept = em.find(Department.class, 1L);
    Employee emp = em.find(Employee.class, 2L);

    // ❌ This does NOT update the database (non-owning side)
    dept.getEmployees().add(emp);
    // Hibernate ignores changes to the mappedBy (non-owning) side

    // ✅ This DOES update the database (owning side)
    emp.setDepartment(dept);
    // Hibernate generates: UPDATE employee SET department_id=1 WHERE id=2

    // ✅ Best: use the convenience method that updates BOTH sides
    dept.addEmployee(emp); // updates both sides consistently
}
```

> 💡 **Interviewer Tip:** A very common interview question is "what is `mappedBy` for?" — Answer: it marks the non-owning side of the relationship and tells Hibernate which field on the other side holds the foreign key. Only the owning side's changes are persisted.

</details>

---

**Q8. How does `@ManyToMany` work and what is `@JoinTable`?**

<details><summary>Click to reveal answer</summary>

```java
@Entity
public class Student {
    @Id @GeneratedValue private Long id;
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",          // junction table name
        joinColumns = @JoinColumn(
            name = "student_id",          // FK to this entity
            referencedColumnName = "id"
        ),
        inverseJoinColumns = @JoinColumn(
            name = "course_id",           // FK to the other entity
            referencedColumnName = "id"
        )
    )
    private Set<Course> courses = new HashSet<>();

    public void enroll(Course course) {
        courses.add(course);
        course.getStudents().add(this);
    }

    public void unenroll(Course course) {
        courses.remove(course);
        course.getStudents().remove(this);
    }
}

@Entity
public class Course {
    @Id @GeneratedValue private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses") // Student owns the relationship
    private Set<Student> students = new HashSet<>();
}
```

**Modeling the junction table as an entity (when extra columns needed):**
```java
// Instead of @ManyToMany, use two @ManyToOne with an explicit join entity
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    private Course course;

    // Extra columns on the junction table
    private LocalDate enrollmentDate;
    private String grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;

    // equals, hashCode based on both fields
}
```

✅ **Best Practice:** Prefer explicit junction entity over `@ManyToMany` whenever you need extra columns on the join table (enrollment date, status, etc.) or need to query the join directly.

</details>

---

**Q9. What is `orphanRemoval` and how does it differ from `CascadeType.REMOVE`?**

<details><summary>Click to reveal answer</summary>

```java
@Entity
public class Article {

    @OneToMany(mappedBy = "article",
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
}

// CascadeType.REMOVE — triggered when the PARENT is removed
// orphanRemoval — triggered when the CHILD is removed from the COLLECTION

@Transactional
public void differences() {

    // CascadeType.REMOVE: removing the parent cascades to children
    Article article = articleRepo.findById(1L).orElseThrow();
    articleRepo.delete(article); // ← triggers CascadeType.REMOVE → deletes all comments

    // orphanRemoval: removing child from parent's collection
    Article article2 = articleRepo.findById(2L).orElseThrow();
    Comment comment = article2.getComments().get(0);
    article2.getComments().remove(comment); // ← orphanRemoval → deletes the comment

    // Without orphanRemoval:
    // article2.getComments().remove(comment) would just update the in-memory list
    // comment.setArticle(null) would set the FK to null (or fail if NOT NULL)
    // but the comment row would NOT be deleted from the database
}

// ❌ Mistake: doing this without orphanRemoval
@OneToMany(mappedBy = "article", cascade = CascadeType.ALL) // no orphanRemoval
// article.getComments().remove(c) → comment remains in DB with article_id=null
// This creates orphaned rows!

// ✅ With orphanRemoval:
@OneToMany(mappedBy = "article", cascade = CascadeType.ALL, orphanRemoval = true)
// article.getComments().remove(c) → DELETE FROM comments WHERE id=?
```

**Use `orphanRemoval = true` when:**
- The child entity has no independent existence outside the parent
- A `Comment` without an `Article`, or an `OrderItem` without an `Order`

**Avoid `orphanRemoval = true` when:**
- The child can belong to multiple parents
- The child has independent meaning outside the parent

</details>

---

**Q10. What are flush modes in Hibernate?**

<details><summary>Click to reveal answer</summary>

```java
// FlushMode controls WHEN Hibernate synchronizes the persistence context with the DB

@Service
public class FlushModeDemo {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void autoFlushMode() {
        // FlushModeType.AUTO (default)
        // Flushes:
        // 1. Before executing JPQL/HQL queries (to ensure query sees latest data)
        // 2. Before transaction commits

        Product product = new Product("Widget");
        em.persist(product); // buffered — no INSERT yet

        // Hibernate flushes here because the query might need to see "Widget"
        List<Product> all = em.createQuery("FROM Product", Product.class).getResultList();
        // Before the above query: INSERT INTO product VALUES ('Widget') is executed

        System.out.println(all.size()); // includes "Widget"
    }

    @Transactional
    public void commitFlushMode() {
        em.setFlushMode(FlushModeType.COMMIT); // override to COMMIT-only flushing

        Product product = new Product("Widget");
        em.persist(product);

        // Hibernate does NOT flush before this query
        List<Product> all = em.createQuery("FROM Product", Product.class).getResultList();
        System.out.println(all.contains(product)); // false! — not yet flushed

        // INSERT happens at transaction commit only
    }

    @Transactional
    public void manualFlush() {
        // Hibernate-specific FlushMode.MANUAL (not JPA standard)
        Session session = em.unwrap(Session.class);
        session.setHibernateFlushMode(FlushMode.MANUAL);

        Product product = new Product("Widget");
        em.persist(product);

        // Must explicitly flush — nothing automatic
        em.flush(); // ← manual trigger

        List<Product> all = em.createQuery("FROM Product", Product.class).getResultList();
        System.out.println(all.contains(product)); // true
    }
}
```

**Use cases:**
- `AUTO`: default, best for correctness
- `COMMIT`: batch processing — avoid intermediate flushes for performance
- `MANUAL`: bulk imports where you control the flush cycle explicitly

</details>

---

### 🔴 Hard

---

**Q11. Explain the `EntityManager` vs `Session` API.**

<details><summary>Click to reveal answer</summary>

```java
// JPA standard — EntityManager (portable across JPA providers)
@PersistenceContext
private EntityManager em;

// Hibernate-specific — Session (extends EntityManager, adds extra features)
@PersistenceContext
private EntityManager em;
// Unwrap to get Session when you need Hibernate-specific features:
Session session = em.unwrap(Session.class);

// Comparison:
@Service
public class JpaVsHibernateDemo {

    @PersistenceContext EntityManager em;

    @Transactional
    public void jpaApis() {
        // Standard JPA EntityManager APIs:
        em.persist(entity);           // insert
        em.find(User.class, 1L);      // find by PK (uses 1st-level cache)
        em.merge(detachedEntity);     // reattach + update
        em.remove(managedEntity);     // delete
        em.flush();                   // sync to DB
        em.clear();                   // clear persistence context (detaches all)
        em.detach(entity);            // detach single entity
        em.refresh(entity);           // reload from DB (discard in-memory changes)
        em.contains(entity);          // check if entity is managed

        em.createQuery("FROM User", User.class).getResultList();
        em.createNamedQuery("User.findAll", User.class).getResultList();
        em.createNativeQuery("SELECT * FROM users", User.class).getResultList();
    }

    @Transactional
    public void hibernateSpecificApis() {
        Session session = em.unwrap(Session.class);

        // Hibernate-specific features:
        session.saveOrUpdate(entity);  // deprecated in Hibernate 6, use merge
        session.load(User.class, 1L);  // returns proxy without hitting DB
        session.get(User.class, 1L);   // like find(), returns null if not found

        session.evict(entity);         // same as detach
        session.setHibernateFlushMode(FlushMode.MANUAL);

        // Stateless session (no 1st-level cache, ideal for bulk operations)
        StatelessSession stateless = session.getSessionFactory()
            .openStatelessSession();
        // bulk insert without loading into memory
        stateless.insert(new Product(...));
    }
}
```

> 💡 **Interviewer Tip:** Prefer `EntityManager` (JPA standard) in application code. Use `Session` (unwrapped) only when you need Hibernate-specific features like `StatelessSession` for bulk operations or `FlushMode.MANUAL`.

</details>

---

**Q12. How do you implement optimistic locking with `@Version`?**

<details><summary>Click to reveal answer</summary>

```java
@Entity
public class BankAccount {
    @Id @GeneratedValue private Long id;
    private String owner;
    private BigDecimal balance;

    @Version  // Hibernate manages this field automatically
    private Long version; // or Integer or Timestamp
}

// How it works:
// SELECT * FROM bank_account WHERE id=1  →  {id=1, balance=1000, version=5}
// Application modifies: balance = 900
// UPDATE bank_account SET balance=900, version=6 WHERE id=1 AND version=5
//                                                       ^^^^^^^^^^^^^^
//                                                  Checks version hasn't changed!
// If another thread already updated (version is now 6):
//   UPDATE affects 0 rows → Hibernate throws OptimisticLockException

@Service
public class BankService {

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        BankAccount from = accountRepo.findById(fromId).orElseThrow();
        BankAccount to   = accountRepo.findById(toId).orElseThrow();

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        // If another thread modified either account concurrently:
        // Hibernate throws OptimisticLockException → @Transactional rolls back
    }

    // Retry on optimistic lock failure
    @Transactional
    @Retryable(value = OptimisticLockingFailureException.class,
               maxAttempts = 3,
               backoff = @Backoff(delay = 100, multiplier = 2))
    public void safeTransfer(Long fromId, Long toId, BigDecimal amount) {
        // Each retry gets fresh data with latest version
        transfer(fromId, toId, amount);
    }
}
```

**Pessimistic locking as alternative:**
```java
// Pessimistic WRITE lock — DB-level lock, no one else can read or write
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<BankAccount> findById(Long id);

// Or with timeout
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "3000"))
Optional<BankAccount> findByIdWithLock(Long id);
```

</details>

---

**Q13. How do you perform bulk updates and deletes efficiently with JPA?**

<details><summary>Click to reveal answer</summary>

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Bulk UPDATE via JPQL — bypasses persistence context (no dirty checking overhead)
    @Modifying
    @Query("UPDATE Product p SET p.price = p.price * :factor WHERE p.category = :category")
    int applyPriceAdjustment(@Param("category") String category,
                              @Param("factor") BigDecimal factor);

    // Bulk DELETE via JPQL
    @Modifying
    @Query("DELETE FROM Product p WHERE p.status = 'DISCONTINUED' AND p.lastSoldDate < :cutoff")
    int deleteDiscontinuedProducts(@Param("cutoff") LocalDate cutoff);

    // Bulk DELETE by IDs
    @Modifying
    @Query("DELETE FROM Product p WHERE p.id IN :ids")
    int deleteByIds(@Param("ids") List<Long> ids);

    // Native SQL for complex bulk ops
    @Modifying
    @Query(value = """
        UPDATE products
        SET price = price * :factor,
            updated_at = NOW()
        WHERE category_id = :categoryId
        """, nativeQuery = true)
    int bulkUpdateByCategory(@Param("categoryId") Long categoryId,
                             @Param("factor") BigDecimal factor);
}

@Service
public class BulkOperationService {

    @Autowired private ProductRepository repo;

    @Transactional
    public void bulkUpdate() {
        int updated = repo.applyPriceAdjustment("Electronics", BigDecimal.valueOf(0.9));
        System.out.println("Updated " + updated + " products");

        // IMPORTANT: after @Modifying, persistence context may be stale
        // Clear it to avoid reading cached (old) data
        // Spring auto-clears if you use @Modifying(clearAutomatically = true)
    }
}

// @Modifying annotation options
@Modifying(
    clearAutomatically = true,  // clears persistence context after execution
    flushAutomatically = true   // flushes pending changes before execution
)
@Query("UPDATE Product p SET p.active = false WHERE p.expiryDate < :now")
int deactivateExpiredProducts(@Param("now") LocalDate now);
```

🚨 **Common Mistake:** Forgetting `@Modifying` on UPDATE/DELETE queries. Without it, Hibernate throws `QueryExecutionRequestException: Not supported for DML operations`.

Also watch out: bulk updates bypass the persistence context — if you have loaded entities in the same session that were affected by the bulk update, they won't reflect the changes. Use `clearAutomatically = true` or reload entities after bulk operations.

</details>

---

**Q14. How does inheritance mapping work in JPA?**

<details><summary>Click to reveal answer</summary>

```java
// Strategy 1: SINGLE_TABLE — all subclasses in one table (default)
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "vehicle_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Vehicle {
    @Id @GeneratedValue private Long id;
    private String make;
    private String model;
}

@Entity
@DiscriminatorValue("CAR")
public class Car extends Vehicle {
    private int numberOfDoors;
}

@Entity
@DiscriminatorValue("TRUCK")
public class Truck extends Vehicle {
    private double payloadCapacity; // null for Car rows (nullable in DB)
}
// Pros: best performance (single JOIN), polymorphic queries easy
// Cons: nullable columns for subclass-specific fields

// Strategy 2: TABLE_PER_CLASS — each class gets its own table
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment {
    @Id @GeneratedValue private Long id;
    private BigDecimal amount;
    private LocalDateTime createdAt;
}

@Entity
public class CreditCardPayment extends Payment {
    private String cardNumber;
    private String cvv;
}

@Entity
public class BankTransferPayment extends Payment {
    private String bankAccount;
    private String routingNumber;
}
// Pros: clean schema, no nulls
// Cons: polymorphic queries use UNION ALL (expensive)

// Strategy 3: JOINED — parent and child tables joined by FK
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Employee {
    @Id @GeneratedValue private Long id;
    private String name;
    private String email;
}

@Entity
@PrimaryKeyJoinColumn(name = "employee_id")
public class Manager extends Employee {
    private String department;
    private int teamSize;
}

@Entity
@PrimaryKeyJoinColumn(name = "employee_id")
public class Engineer extends Employee {
    private String techStack;
    private int seniorityLevel;
}
// Pros: clean schema, no nullable columns
// Cons: every query requires JOINs; polymorphic queries expensive
```

**Which to choose:**

| Strategy | Best For |
|---------|---------|
| `SINGLE_TABLE` | Small hierarchies, performance critical |
| `JOINED` | Clean schema required, complex hierarchies |
| `TABLE_PER_CLASS` | Rarely used; avoid for polymorphic queries |

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| Entity states | transient → managed → detached/removed |
| Dirty checking | Hibernate tracks changes automatically; no explicit `save()` needed for managed entities |
| `mappedBy` | Marks non-owning side; only owning side's changes are persisted |
| `orphanRemoval` | Deletes child when removed from parent collection |
| `@Version` | Optimistic locking; prevents lost updates |
| `@Modifying` | Required for bulk UPDATE/DELETE JPQL queries |
| JPQL | Object-oriented; uses entity/field names (not table/column names) |
| Native query | DB-specific SQL; use when JPQL can't express the query |
| `FlushMode.AUTO` | Flushes before queries and on commit |
| `StatelessSession` | No 1st-level cache; ideal for bulk imports |
