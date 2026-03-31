# Lazy vs Eager Loading

> 💡 **Interviewer Tip:** Lazy loading is one of the most common sources of bugs and performance problems in JPA applications. Interviewers test whether you understand `LazyInitializationException`, the Open Session in View anti-pattern, and practical solutions like `@EntityGraph` and `JOIN FETCH`.

---

## Table of Contents
1. [Default Fetch Types](#default-fetch-types)
2. [LazyInitializationException](#lazyinitializationexception)
3. [Solutions and Patterns](#solutions-and-patterns)
4. [Interview Questions](#interview-questions)

---

## Default Fetch Types

| Relationship | Default Fetch |
|-------------|--------------|
| `@OneToMany` | `LAZY` |
| `@ManyToMany` | `LAZY` |
| `@ManyToOne` | `EAGER` ⚠️ |
| `@OneToOne` | `EAGER` ⚠️ |
| `@ElementCollection` | `LAZY` |
| `@Basic` | `EAGER` |

> ⚠️ The EAGER defaults for `@ManyToOne` and `@OneToOne` are a JPA spec choice that many consider a mistake. Always override to `LAZY` unless you have a strong reason.

---

## Interview Questions

### 🟢 Easy

---

**Q1. What is the difference between `FetchType.LAZY` and `FetchType.EAGER`?**

<details><summary>Click to reveal answer</summary>

```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;
    private String status;

    // LAZY — items are NOT loaded when Order is fetched
    // SQL for items executes only when getItems() is called
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER — address IS loaded immediately with Order
    // SQL JOIN is executed in the same query as Order
    @ManyToOne(fetch = FetchType.EAGER)
    private Address shippingAddress;
}
```

**How it works internally:**
```java
@Service
public class OrderService {

    @Transactional
    public void demonstrateFetchTypes() {
        Order order = orderRepo.findById(1L).orElseThrow();
        // SQL: SELECT o.*, a.* FROM orders o LEFT JOIN addresses a ON o.address_id = a.id
        //                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //                             EAGER — address loaded immediately

        // At this point: order.getShippingAddress() → in-memory, no SQL
        // At this point: order.getItems() → returns Hibernate proxy (not yet loaded)

        List<OrderItem> items = order.getItems(); // ← LAZY triggers SQL here
        // SQL: SELECT * FROM order_items WHERE order_id = 1
    }
}
```

**LAZY loading — Hibernate uses a proxy:**
```java
@Transactional
public void lazyProxy() {
    Order order = orderRepo.findById(1L).orElseThrow();
    List<OrderItem> items = order.getItems();

    // items is a PersistentBag (Hibernate proxy) — not a real ArrayList yet!
    System.out.println(items.getClass()); // org.hibernate.collection.internal.PersistentBag
    System.out.println(items.isEmpty());  // ← THIS triggers the SQL SELECT
}
```

✅ **Best Practice:** Always override `@ManyToOne` and `@OneToOne` to `FetchType.LAZY`:
```java
@ManyToOne(fetch = FetchType.LAZY)  // override the EAGER default
@JoinColumn(name = "category_id")
private Category category;
```

</details>

---

**Q2. What is `LazyInitializationException` and why does it occur?**

<details><summary>Click to reveal answer</summary>

`LazyInitializationException` is thrown when you try to access a lazily-loaded association **outside** of an active Hibernate session (persistence context).

```java
// ❌ Classic LazyInitializationException scenario
@Service
public class OrderService {

    // NOT @Transactional — no Hibernate session
    public List<OrderItem> getOrderItems(Long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        // findById() opens and closes a session internally

        // Session is now CLOSED. order is DETACHED.
        return order.getItems(); // ❌ LazyInitializationException!
        // Cannot load 'items' because the session that managed 'order' is closed
    }
}

// Stack trace:
// org.hibernate.LazyInitializationException:
//   failed to lazily initialize a collection of role: com.example.Order.items
//   - no session or session was closed
```

**Why it happens:**
1. Spring Data `findById()` opens a transaction, loads the entity, closes the transaction
2. The entity is now **detached** (outside any session)
3. Accessing a lazy collection on a detached entity attempts to go back to the database
4. But there's no active session → exception!

```java
// This also triggers it in web layer (if no OSIV or transaction):
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id); // transaction ends here

        // Mapping in controller — no transaction
        return OrderDto.builder()
            .items(order.getItems().stream()  // ❌ LazyInitializationException
                .map(this::toItemDto)
                .toList())
            .build();
    }
}
```

</details>

---

**Q3. What is the Open Session in View (OSIV) pattern?**

<details><summary>Click to reveal answer</summary>

OSIV extends the Hibernate Session (persistence context) for the duration of the entire HTTP request — including the view rendering phase. Spring Boot enables OSIV by default.

```yaml
# Spring Boot: OSIV is enabled by default
spring:
  jpa:
    open-in-view: true  # default — logs a WARNING on startup

# Disable for production:
spring:
  jpa:
    open-in-view: false
```

```
Request lifecycle WITH OSIV (open-in-view: true):
────────────────────────────────────────────────────
[HTTP Request] → [Filter] → [Controller] → [Service @Transactional]
    ↑                           │                    │
    │                           │                ── tx commits ──
    └──── Session remains open ──────────────────────────────────
    (lazy loading still works in controller/view, but DB connection held!)
```

**Why OSIV is an anti-pattern:**
```java
// With OSIV enabled — lazy loading "works" everywhere (but hides problems)
@RestController
public class ProductController {

    @Autowired ProductService productService;

    @GetMapping("/products/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);

        // With OSIV: this works — session is still open
        // Without OSIV: LazyInitializationException
        return ProductResponse.builder()
            .name(product.getName())
            .categoryName(product.getCategory().getName()) // lazy load "works"
            .reviews(product.getReviews().size())          // lazy load "works"
            .build();
        // BUT: each lazy access fires a separate SQL query — N+1 problem!
    }
}
```

**OSIV problems:**
1. Database connection held for the entire HTTP request (including view rendering time)
2. Lazy loading in controller/view masks N+1 problems
3. Under high load: connection pool exhaustion
4. Logic that should be in service layer leaks into presentation layer

🚨 **Common Mistake:** Enabling OSIV in production and thinking it "fixes" `LazyInitializationException`. It hides the problem and creates worse issues (connection pool exhaustion, N+1 queries).

✅ **Best Practice:** Disable OSIV (`open-in-view: false`) and use explicit fetch strategies:
- `JOIN FETCH` in JPQL queries
- `@EntityGraph`
- DTO projections

</details>

---

### 🟡 Medium

---

**Q4. What are the solutions to `LazyInitializationException`?**

<details><summary>Click to reveal answer</summary>

**Solution 1: `JOIN FETCH` in JPQL**
```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);

    // Multiple eager loads (use DISTINCT for collections)
    @Query("SELECT DISTINCT o FROM Order o " +
           "JOIN FETCH o.items i " +
           "JOIN FETCH i.product " +
           "WHERE o.userId = :userId")
    List<Order> findUserOrdersWithDetails(@Param("userId") Long userId);
}
```

**Solution 2: `@EntityGraph`**
```java
// Named EntityGraph
@Entity
@NamedEntityGraph(
    name = "Order.withItems",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode(value = "items", subgraph = "items.product")
    },
    subgraphs = {
        @NamedSubgraph(name = "items.product",
                       attributeNodes = @NamedAttributeNode("product"))
    }
)
public class Order { /* ... */ }

// Usage in repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph("Order.withItems")
    Optional<Order> findById(Long id); // overrides the named entity graph

    @EntityGraph(attributePaths = {"items", "items.product", "shippingAddress"})
    Optional<Order> findWithAllDetailsById(Long id); // ad-hoc entity graph
}
```

**Solution 3: @Transactional boundary fix**
```java
@Service
public class OrderService {

    // ✅ Keep lazy loading within the transaction
    @Transactional(readOnly = true)
    public OrderDto findByIdWithDetails(Long id) {
        Order order = orderRepo.findById(id).orElseThrow();
        // Trigger lazy loads WITHIN the transaction — no exception
        order.getItems().size(); // initialize collection
        return orderMapper.toDto(order); // map while session is open
    }

    // Even better: use JOIN FETCH and return DTO directly
    @Transactional(readOnly = true)
    public OrderDto findByIdOptimized(Long id) {
        return orderRepo.findByIdWithItems(id)
            .map(orderMapper::toDto)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
    }
}
```

**Solution 4: DTO projection (best for read-heavy operations)**
```java
// Interface projection
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
    // Spring Data resolves this from the order's customer association
    @Value("#{target.customer.name}")
    String getCustomerName();
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<OrderSummary> findAllProjectedBy(); // no LazyInitializationException possible
}

// Class-based DTO projection (more control)
@Query("SELECT new com.example.dto.OrderSummaryDto(o.id, o.status, c.name) " +
       "FROM Order o JOIN o.customer c")
List<OrderSummaryDto> findAllSummaries();
```

> 💡 **Interviewer Tip:** List ALL four solutions, then say "I prefer DTO projections for read operations because they avoid loading unnecessary data and prevent LazyInitializationException by design."

</details>

---

**Q5. How does `@EntityGraph` work and when do you use it?**

<details><summary>Click to reveal answer</summary>

`@EntityGraph` overrides the default fetch strategy for a specific query. It can load associations eagerly for that query without permanently changing the entity's mapping.

```java
@Entity
@NamedEntityGraph(
    name = "Product.withCategoryAndReviews",
    attributeNodes = {
        @NamedAttributeNode("category"),
        @NamedAttributeNode(value = "reviews", subgraph = "reviews-with-author")
    },
    subgraphs = @NamedSubgraph(
        name = "reviews-with-author",
        attributeNodes = @NamedAttributeNode("author")
    )
)
public class Product {
    @Id @GeneratedValue private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;

    @OneToMany(mappedBy = "product", fetch = FetchType.LAZY)
    private List<Review> reviews;
}

// Repository methods with different entity graphs
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Named entity graph — loads category and reviews with author
    @EntityGraph("Product.withCategoryAndReviews")
    Optional<Product> findById(Long id);

    // Ad-hoc entity graph — only load category
    @EntityGraph(attributePaths = {"category"})
    Optional<Product> findByIdWithCategory(Long id);

    // For collections — fine-grained control
    @EntityGraph(attributePaths = {"category"})
    List<Product> findByCategoryId(Long categoryId);

    // Override: FETCH (default) vs LOAD
    // FETCH: only load specified attributes eagerly, rest stay lazy
    // LOAD: load specified attributes eagerly, rest use their mapping defaults
    @EntityGraph(value = "Product.withCategoryAndReviews",
                 type = EntityGraph.EntityGraphType.FETCH)
    List<Product> findAllWithDetails();
}
```

**EntityGraph vs JOIN FETCH:**

```java
// @EntityGraph — Spring Data generates the query (may not be optimal)
@EntityGraph(attributePaths = {"items"})
List<Order> findByStatus(String status);
// Generated: SELECT o, i FROM orders o LEFT JOIN FETCH o.items i WHERE o.status = ?

// JOIN FETCH — you control the query (more explicit, often better for complex cases)
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") String status);
```

✅ **Best Practice:** Use `@EntityGraph` for simple cases and repository method-level control. Use `JOIN FETCH` in `@Query` for complex queries or when you need DISTINCT or filtering on associations.

</details>

---

**Q6. What is `@BatchSize` and how does it help with lazy loading?**

<details><summary>Click to reveal answer</summary>

`@BatchSize` controls how many IDs Hibernate uses in a batch when lazily loading collections. Instead of N individual queries (N+1), it loads in batches using `WHERE id IN (?, ?, ?, ...)`.

```java
@Entity
public class Department {
    @Id @GeneratedValue private Long id;
    private String name;

    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    @BatchSize(size = 25) // load 25 departments' employees at once
    private List<Employee> employees;
}

@Entity
public class Employee {
    @Id @GeneratedValue private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 25) // on @ManyToOne: batch-loads parent entities
    private Department department;
}
```

**Without @BatchSize:**
```
Fetching 100 departments:
  Query 1: SELECT * FROM departments → 100 rows
  Query 2: SELECT * FROM employees WHERE department_id = 1
  Query 3: SELECT * FROM employees WHERE department_id = 2
  ...
  Query 101: SELECT * FROM employees WHERE department_id = 100
  Total: 101 queries (classic N+1)
```

**With `@BatchSize(size = 25)`:**
```
Fetching 100 departments:
  Query 1: SELECT * FROM departments → 100 rows
  (access employees for all 100 departments)
  Query 2: SELECT * FROM employees WHERE department_id IN (1,2,3,...,25)
  Query 3: SELECT * FROM employees WHERE department_id IN (26,27,...,50)
  Query 4: SELECT * FROM employees WHERE department_id IN (51,...,75)
  Query 5: SELECT * FROM employees WHERE department_id IN (76,...,100)
  Total: 5 queries instead of 101!
```

**Global batch size configuration:**
```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25  # applies globally to all lazy associations
```

```java
// Equivalent Hibernate property:
// hibernate.default_batch_fetch_size = 25

@Configuration
public class JpaConfig {
    @Bean
    public HibernatePropertiesCustomizer hibernateCustomizer() {
        return props -> props.put("hibernate.default_batch_fetch_size", 25);
    }
}
```

**Subselect fetching (alternative to BatchSize):**
```java
@OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
@Fetch(FetchMode.SUBSELECT) // uses: WHERE department_id IN (SELECT id FROM departments ...)
private List<Employee> employees;
```

</details>

---

**Q7. What is the N+1 relationship with fetch type and how do entity graphs solve it?**

<details><summary>Click to reveal answer</summary>

N+1 is directly caused by `FetchType.LAZY` combined with iterating over a collection and accessing associations without pre-fetching.

```java
// Setup
@Entity
public class Author {
    @Id @GeneratedValue private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

// N+1 PROBLEM:
@Service
public class AuthorService {

    @Transactional(readOnly = true)
    public List<AuthorDto> getAllAuthorsWithBooks() {
        List<Author> authors = authorRepo.findAll(); // Query 1: SELECT * FROM authors

        return authors.stream()
            .map(author -> {
                List<Book> books = author.getBooks(); // Queries 2..N: SELECT * FROM books WHERE author_id=?
                // Each author triggers a separate query!
                return new AuthorDto(author.getName(), books.size());
            })
            .toList();
        // Total: 1 + N queries where N = number of authors
    }
}

// SOLUTION 1: JOIN FETCH
@Query("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();
// Total: 1 query

// SOLUTION 2: @EntityGraph
@EntityGraph(attributePaths = {"books"})
List<Author> findAll();
// Total: 1 query (with LEFT JOIN FETCH)

// SOLUTION 3: @BatchSize(size = 20)
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
@BatchSize(size = 20)
private List<Book> books;
// Total: 1 + ceil(N/20) queries

// SOLUTION 4: DTO projection — avoid entity loading entirely
@Query("SELECT new com.example.dto.AuthorBookCountDto(a.name, COUNT(b)) " +
       "FROM Author a LEFT JOIN a.books b GROUP BY a.id, a.name")
List<AuthorBookCountDto> findAuthorsWithBookCounts();
// Total: 1 query, no N+1 possible
```

**How to detect N+1:**
```yaml
# application.yml — enable SQL logging
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

</details>

---

**Q8. When should you use EAGER fetching despite it being an anti-pattern?**

<details><summary>Click to reveal answer</summary>

Eager fetching is appropriate when:
1. The association is **always** needed together with the parent entity
2. The association is a **single object** (not a collection) that is small
3. Performance profiling shows no benefit from lazy loading

```java
// Cases where EAGER is acceptable:
@Entity
public class UserAccount {

    // ✅ EAGER acceptable: user profile is always needed with user account
    // and it's a single object, not a collection
    @OneToOne(fetch = FetchType.EAGER)
    private UserProfile profile;

    // ❌ EAGER problematic: roles could be large, not always needed
    @ManyToMany(fetch = FetchType.EAGER)
    private Set<Role> roles;

    // ❌ EAGER almost always wrong for collections
    @OneToMany(fetch = FetchType.EAGER)
    private List<OrderHistory> orders;
}

// Even for "always needed" associations, prefer explicit fetch over EAGER mapping:
@Repository
public interface UserRepository extends JpaRepository<UserAccount, Long> {

    // Better than EAGER mapping: explicit fetch only where needed
    @EntityGraph(attributePaths = {"profile"})
    Optional<UserAccount> findByUsername(String username);

    // Use simple find for contexts that don't need the profile
    Optional<UserAccount> findById(Long id);
}
```

**Problems with EAGER on collections:**
```java
// EAGER collection → Hibernate uses multiple queries (not a join):
// Even with FetchType.EAGER on @OneToMany, Hibernate still does N+1 in some cases!

@Entity
public class Order {
    @OneToMany(fetch = FetchType.EAGER) // bad!
    private List<OrderItem> items;
}

// Loading 100 orders with EAGER items can still result in:
// SELECT * FROM orders             → 100 rows
// SELECT * FROM order_items WHERE order_id = 1
// SELECT * FROM order_items WHERE order_id = 2
// ... (still N+1 for some Hibernate versions/configurations)
```

> 💡 **Interviewer Tip:** The correct answer is "EAGER is almost never the right choice. Prefer LAZY with explicit fetch using @EntityGraph or JOIN FETCH where needed." This shows mature understanding of JPA.

</details>

---

### 🔴 Hard

---

**Q9. What is the `MultipleBagFetchException` and how do you fix it?**

<details><summary>Click to reveal answer</summary>

Hibernate throws `MultipleBagFetchException` when you try to `JOIN FETCH` two or more `List` (Bag) collections simultaneously.

```java
// ❌ This throws MultipleBagFetchException
@Query("SELECT DISTINCT o FROM Order o " +
       "JOIN FETCH o.items " +      // items is a List
       "JOIN FETCH o.payments")     // payments is also a List
// Hibernate cannot fetch two Bags simultaneously — cartesian product problem
List<Order> findAllWithItemsAndPayments();
```

**Why it happens:** Fetching two collections as bags (Lists) creates a Cartesian product — if Order has 5 items and 3 payments, Hibernate gets 15 rows and can't reliably map them back to the correct parent.

**Solutions:**

**Solution 1: Use `Set` instead of `List` for at least one collection**
```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<OrderItem> items = new HashSet<>(); // Set instead of List

    @OneToMany(mappedBy = "order")
    private Set<Payment> payments = new HashSet<>(); // Set allows multi-bag fetch

    // JPA spec allows fetching multiple Sets simultaneously
}

@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items JOIN FETCH o.payments")
List<Order> findAllWithDetails(); // Works with Sets!
```

**Solution 2: Multiple queries (two-step loading)**
```java
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public List<Order> findAllWithDetails() {
        // Query 1: load orders with items
        List<Order> orders = orderRepo.findAllWithItems();

        // Extract IDs
        List<Long> orderIds = orders.stream().map(Order::getId).toList();

        // Query 2: load payments for those orders (Hibernate merges into 1st-level cache)
        orderRepo.findAllWithPaymentsByIds(orderIds);

        return orders; // now both items and payments are loaded in persistence context
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items")
    List<Order> findAllWithItems();

    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.payments WHERE o.id IN :ids")
    List<Order> findAllWithPaymentsByIds(@Param("ids") List<Long> ids);
}
```

**Solution 3: `@BatchSize` (simplest)**
```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 25)
    private List<OrderItem> items;

    @OneToMany(mappedBy = "order")
    @BatchSize(size = 25)
    private List<Payment> payments;
}
// No MultipleBagFetchException because we're not using JOIN FETCH
// Loads in batches: N+ceil(N/25) queries instead of 2N+1
```

</details>

---

**Q10. How do you implement DTO projections to avoid lazy loading issues entirely?**

<details><summary>Click to reveal answer</summary>

DTO projections bypass the entity model entirely, fetching only the data you need.

```java
// 1. Interface-based projection
public interface OrderSummaryView {
    Long getId();
    String getStatus();
    BigDecimal getTotalAmount();
    Instant getCreatedAt();
    // Nested projection
    CustomerView getCustomer();

    interface CustomerView {
        String getName();
        String getEmail();
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Spring Data auto-generates the projection query
    List<OrderSummaryView> findByStatus(String status);

    // Class-based projection (record works great here!)
    @Query("SELECT new com.example.dto.OrderSummaryDto(" +
           "o.id, o.status, o.totalAmount, c.name) " +
           "FROM Order o JOIN o.customer c WHERE o.status = :status")
    List<OrderSummaryDto> findSummariesByStatus(@Param("status") String status);
}

// 2. Record-based DTO (Java 16+)
public record OrderSummaryDto(
    Long id,
    String status,
    BigDecimal totalAmount,
    String customerName
) {}

// 3. Complex DTO with @Query and constructor
@Query("""
    SELECT new com.example.dto.OrderDetailDto(
        o.id, o.status, o.createdAt,
        c.id, c.name, c.email,
        COUNT(i)
    )
    FROM Order o
    JOIN o.customer c
    LEFT JOIN o.items i
    WHERE o.id = :id
    GROUP BY o.id, o.status, o.createdAt, c.id, c.name, c.email
    """)
Optional<OrderDetailDto> findDetailById(@Param("id") Long id);

// 4. Native query with DTO
@Query(value = """
    SELECT
        o.id,
        o.status,
        o.total_amount,
        c.name AS customer_name,
        COUNT(i.id) AS item_count
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    LEFT JOIN order_items i ON i.order_id = o.id
    WHERE o.id = :id
    GROUP BY o.id, c.id
    """, nativeQuery = true)
Optional<OrderSummaryView> findNativeSummaryById(@Param("id") Long id);
```

```java
// Using Blaze-Persistence (advanced projections)
@Transactional(readOnly = true)
public OrderDetailView getOrderDetail(Long orderId) {
    return entityViewManager.find(em, OrderDetailView.class, orderId);
}

// @EntityView (Blaze-Persistence)
@EntityView(Order.class)
public interface OrderDetailView {
    Long getId();
    String getStatus();

    @Mapping("customer.name")
    String getCustomerName();

    @CollectionMapping(ignoreIndex = true)
    List<ItemView> getItems();

    @EntityView(OrderItem.class)
    interface ItemView {
        String getProductName();
        Integer getQuantity();
        BigDecimal getUnitPrice();
    }
}
```

✅ **Best Practice:** For read-only APIs (GET endpoints), always use DTO projections instead of returning managed entities. This:
- Prevents LazyInitializationException by design
- Reduces data transfer (SELECT only needed columns)
- Prevents accidental data exposure (no hidden fields)
- Improves performance significantly

</details>

---

**Q11. How does Hibernate handle lazy loading with `@OneToOne` and why is it tricky?**

<details><summary>Click to reveal answer</summary>

`@OneToOne` with lazy loading is deceptively tricky because Hibernate needs to know whether the associated entity exists before creating a proxy.

```java
@Entity
public class User {
    @Id @GeneratedValue private Long id;
    private String name;

    // Owning side — works fine with LAZY
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")
    private UserProfile profile; // ✅ Hibernate can proxy this
}

@Entity
public class UserProfile {
    @Id @GeneratedValue private Long id;

    // Non-owning (mappedBy) side — LAZY is tricky here!
    @OneToOne(mappedBy = "profile", fetch = FetchType.LAZY)
    private User user; // ⚠️ Hibernate may not lazily load this correctly
}
```

**Why the non-owning side is problematic:**
- To create a proxy, Hibernate needs to know if the association exists
- For the non-owning side, it must query the owning table to check if the FK exists
- This query might as well load the full entity
- Result: LAZY is effectively EAGER for `mappedBy` `@OneToOne`

```java
// Solutions for non-owning @OneToOne LAZY:

// Option 1: Use @LazyToOne (Hibernate-specific, requires bytecode enhancement)
@OneToOne(mappedBy = "profile", fetch = FetchType.LAZY)
@LazyToOne(LazyToOneOption.NO_PROXY) // Hibernate 5
private User user;

// Option 2: Reverse the owning side (put FK on the entity you fetch more often)
@Entity
public class UserProfile {
    @Id @GeneratedValue private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")  // put FK here — owning side, LAZY works correctly
    private User user;
}

// Option 3: Use @MapsId (shared primary key — no separate FK needed)
@Entity
public class UserProfile {
    @Id
    private Long id; // same as User.id

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId
    @JoinColumn(name = "id")
    private User user;
}

// Option 4: Use explicit query instead of relying on association
@Repository
public interface UserProfileRepository extends JpaRepository<UserProfile, Long> {
    Optional<UserProfile> findByUserId(Long userId); // explicit query, no proxy needed
}
```

> 💡 **Interviewer Tip:** `@OneToOne` LAZY on the non-owning (mappedBy) side is a well-known Hibernate limitation. Knowing this demonstrates deep JPA knowledge. The recommended fix is to reverse the owning side or use `@MapsId` (shared primary key).

</details>

---

**Q12. How do you profile and measure the impact of different fetch strategies?**

<details><summary>Click to reveal answer</summary>

```java
// 1. Enable SQL logging
```
```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    org.hibernate.stat: DEBUG  # Hibernate statistics

spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true  # track query counts, cache hits, etc.
        format_sql: true
        use_sql_comments: true
```

```java
// 2. Use Hibernate Statistics programmatically
@Service
public class PerformanceMonitor {

    @PersistenceContext
    private EntityManager em;

    @Transactional(readOnly = true)
    public void compareStrategies() {
        SessionFactory sf = em.unwrap(Session.class).getSessionFactory();
        Statistics stats = sf.getStatistics();
        stats.setStatisticsEnabled(true);
        stats.clear();

        // Strategy 1: Naive loading
        List<Author> authors = authorRepo.findAll();
        authors.forEach(a -> a.getBooks().size()); // trigger N+1

        System.out.println("Naive queries: " + stats.getQueryExecutionCount());
        // Queries: 1 + N

        stats.clear();

        // Strategy 2: JOIN FETCH
        List<Author> authorsWithBooks = authorRepo.findAllWithBooks();

        System.out.println("JOIN FETCH queries: " + stats.getQueryExecutionCount());
        // Queries: 1

        stats.clear();

        // Strategy 3: @BatchSize
        List<Author> batchedAuthors = authorRepo.findAll();
        batchedAuthors.forEach(a -> a.getBooks().size());

        System.out.println("BatchSize queries: " + stats.getQueryExecutionCount());
        // Queries: 1 + ceil(N/batchSize)
    }
}

// 3. Using datasource-proxy / p6spy for query counting in tests
@SpringBootTest
class FetchStrategyTest {

    @Autowired private AuthorService authorService;

    @Test
    void shouldNotHaveNPlusOneWithEntityGraph() {
        // Using datasource-proxy to count queries
        QueryCountHolder.clear();

        authorService.findAllWithBooks();

        long queryCount = QueryCountHolder.getGrandTotal().getSelect();
        assertThat(queryCount).isEqualTo(1); // exactly 1 query, no N+1
    }
}
```

```java
// 4. Quick test assertion with StatisticsAssert (Hypersistence Utils)
@Test
void shouldLoadOrdersWithOneQuery() {
    SessionFactory sf = entityManagerFactory.unwrap(SessionFactory.class);
    sf.getStatistics().setStatisticsEnabled(true);
    sf.getStatistics().clear();

    List<Order> orders = orderService.findAllWithItems();

    long queryCount = sf.getStatistics().getPrepareStatementCount();
    assertThat(queryCount)
        .as("Expected 1 query but got N+1: " + queryCount)
        .isEqualTo(1);
}
```

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| `@ManyToOne` default | `EAGER` — always override to `LAZY` |
| `@OneToMany` default | `LAZY` — correct default |
| `LazyInitializationException` | Accessing lazy collection outside Hibernate session |
| OSIV anti-pattern | Keeps connection open for entire request; hides N+1 issues |
| `JOIN FETCH` | Best for known, always-needed associations |
| `@EntityGraph` | Flexible, query-level override of fetch strategy |
| `@BatchSize` | Reduces N+1 without JOIN FETCH; uses `IN` clause |
| `MultipleBagFetchException` | Can't JOIN FETCH two List collections; use Set or two queries |
| `@OneToOne` LAZY pitfall | Non-owning side often loads eagerly despite `LAZY` setting |
| DTO projection | Best for read-only data; eliminates LazyInitializationException by design |
