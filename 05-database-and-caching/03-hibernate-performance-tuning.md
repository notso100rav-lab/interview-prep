# Chapter 3: Hibernate Performance Tuning

> **Estimated study time:** 2 days | **Priority:** 🔴 High

---

## Key Concepts

- Second-level cache (L2 cache) with Ehcache / Caffeine / Redis
- Query cache
- Hibernate statistics
- Batch inserts and updates
- Stateless sessions
- DTO projections vs entity fetching
- Connection pool tuning (HikariCP)
- `@BatchSize` and `@Fetch` strategies
- `JPQL` vs native queries for bulk operations

---

## Questions & Answers

---

### Q1 — 🟢 What is Hibernate's first-level cache (persistence context)?

<details><summary>Click to reveal answer</summary>

The **first-level cache** (also called the **persistence context**) is scoped to a single `EntityManager` / `Session`. It is always enabled and cannot be disabled.

- Within the same transaction, loading the same entity twice returns the **same Java object** (no second DB hit).
- Hibernate tracks changes to managed entities and flushes them to the database at the end of the transaction (dirty checking).

```java
// Both calls hit DB only once — second returns from L1 cache
User u1 = em.find(User.class, 1L);
User u2 = em.find(User.class, 1L);
assert u1 == u2; // same object reference

// Modification is detected automatically (dirty checking)
u1.setName("Alice Updated");
// No explicit save needed — flushed on transaction commit
```

**Limitation:** scoped to one session/transaction. Cross-request caching requires the second-level cache.

</details>

---

### Q2 — 🟡 How does Hibernate's second-level (L2) cache work? How do you configure it?

<details><summary>Click to reveal answer</summary>

The **second-level cache** is scoped to the `SessionFactory` (application-level) and shared across sessions. It caches entity data by ID.

**Dependencies (Ehcache 3):**
```xml
<dependency>
  <groupId>org.hibernate.orm</groupId>
  <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
  <groupId>org.ehcache</groupId>
  <artifactId>ehcache</artifactId>
</dependency>
```

**Configuration:**
```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
            uri: classpath:ehcache.xml
```

```xml
<!-- ehcache.xml -->
<config xmlns='http://www.ehcache.org/v3'>
  <cache alias="com.example.User">
    <expiry>
      <ttl unit="minutes">10</ttl>
    </expiry>
    <resources>
      <heap unit="entries">1000</heap>
      <offheap unit="MB">10</offheap>
    </resources>
  </cache>
</config>
```

**Mark entities as cacheable:**
```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // or READ_ONLY, NONSTRICT_READ_WRITE
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;

    @OneToMany
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // cache the collection too
    private List<Category> categories;
}
```

| Concurrency Strategy | Use when |
|---------------------|---------|
| `READ_ONLY` | Data never changes (countries, currencies) — fastest |
| `NONSTRICT_READ_WRITE` | Rarely updated; stale data briefly acceptable |
| `READ_WRITE` | Updated data; uses soft locks for consistency |
| `TRANSACTIONAL` | JTA-managed transactions; strict consistency |

</details>

---

### Q3 — 🟡 How do you enable and use the Hibernate query cache?

<details><summary>Click to reveal answer</summary>

The **query cache** stores the IDs returned by a query (not the entities themselves). The entity data is still loaded from L2 cache or the DB.

```java
// Repository method
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
@Query("SELECT p FROM Product p WHERE p.category = :cat AND p.active = true")
List<Product> findActiveByCategory(@Param("cat") String category);

// Or via EntityManager directly
List<Product> results = em.createQuery("SELECT p FROM Product p WHERE p.active = true", Product.class)
    .setHint("org.hibernate.cacheable", true)
    .setHint("org.hibernate.cacheRegion", "product.active")
    .getResultList();
```

**When the query cache helps:**
- The same query with the same parameters is executed frequently
- The underlying table changes infrequently
- Result sets are small to medium sized

**Important:** whenever ANY entity in the query's result type is modified, the entire query cache region for that entity type is invalidated. On write-heavy tables, the query cache can hurt performance due to constant invalidation.

</details>

---

### Q4 — 🟡 How do you enable Hibernate statistics and what should you monitor?

<details><summary>Click to reveal answer</summary>

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 50   # log slow queries
logging:
  level:
    org.hibernate.stat: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE   # bind parameters
```

```java
// Programmatic access via Spring Boot Actuator or direct API
@Autowired
private EntityManagerFactory emf;

public void printStats() {
    Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
    log.info("Queries executed: {}", stats.getQueryExecutionCount());
    log.info("L2 cache hit ratio: {}/{}",
        stats.getSecondLevelCacheHitCount(),
        stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount());
    log.info("Entities loaded: {}", stats.getEntityLoadCount());
    log.info("Collections fetched: {}", stats.getCollectionFetchCount());
}
```

**Key metrics to watch:**
| Metric | Problem if high |
|--------|-----------------|
| `QueryExecutionCount` | Too many queries (N+1?) |
| `CollectionFetchCount` | Lazy collection fetches happening unexpectedly |
| `SecondLevelCacheMissCount` | L2 cache not effective |
| `EntityUpdateCount` | Unexpected dirty checking updates |
| `FlushCount` | Too many flushes within a transaction |

</details>

---

### Q5 — 🟡 How do you perform batch inserts and updates in Hibernate?

<details><summary>Click to reveal answer</summary>

By default, Hibernate sends one `INSERT` or `UPDATE` per entity. For bulk operations, this is very slow.

**Configure batch size:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50              # batch up to 50 statements
          batch_versioned_data: true  # enable batching for versioned entities
        order_inserts: true           # group inserts of the same entity type
        order_updates: true           # group updates of the same entity type
```

**Use case — bulk insert:**
```java
@Transactional
public void bulkInsert(List<Product> products) {
    for (int i = 0; i < products.size(); i++) {
        em.persist(products.get(i));

        // Flush and clear every batch_size to avoid L1 cache memory pressure
        if (i > 0 && i % 50 == 0) {
            em.flush();
            em.clear();
        }
    }
}
```

**Important:** Hibernate disables batching for entities with `GenerationType.IDENTITY` (AUTO_INCREMENT) because it needs the generated ID immediately after each insert. Use `GenerationType.SEQUENCE` instead:

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "product_seq")
    @SequenceGenerator(name = "product_seq",
                       sequenceName = "product_id_seq",
                       allocationSize = 50)  // pre-allocate 50 IDs at a time
    private Long id;
}
```

**Bulk update/delete without loading entities (Spring Data JPA):**
```java
@Modifying
@Transactional
@Query("UPDATE Product p SET p.price = p.price * :factor WHERE p.category = :cat")
int adjustPrices(@Param("factor") BigDecimal factor, @Param("cat") String category);
```

</details>

---

### Q6 — 🟡 What is a stateless session and when should you use it?

<details><summary>Click to reveal answer</summary>

A `StatelessSession` bypasses the L1 cache, dirty checking, and event listeners. It operates directly against the database with minimal overhead — useful for bulk ETL-style operations.

```java
@Autowired
private SessionFactory sessionFactory;

public void migrateLargeDataset(List<LegacyRecord> records) {
    StatelessSession session = sessionFactory.openStatelessSession();
    Transaction tx = session.beginTransaction();
    try {
        for (LegacyRecord rec : records) {
            NewEntity entity = transform(rec);
            session.insert(entity);           // direct INSERT, no L1 cache
        }
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    } finally {
        session.close();
    }
}
```

**StatelessSession limitations:**
- No first-level cache — you CAN get duplicate instances of the same row
- No dirty checking — you must call `session.update(entity)` explicitly
- No cascading or event listeners (no `@PrePersist`, `@PostLoad`, etc.)
- Collections are NOT loaded lazily (they aren't tracked)

**When to use:**
- Batch import of millions of records
- Read-only reporting queries over large result sets
- Purging old data in bulk

</details>

---

### Q7 — 🔴 How does HikariCP connection pool tuning affect Hibernate performance?

<details><summary>Click to reveal answer</summary>

HikariCP is the default connection pool in Spring Boot. Getting the pool size wrong is a common performance bottleneck.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # don't set this too high!
      minimum-idle: 5
      connection-timeout: 3000     # fail fast if no connection in 3s
      idle-timeout: 600000         # remove idle connections after 10 min
      max-lifetime: 1800000        # recycle connections after 30 min (< DB timeout)
      keepalive-time: 30000        # prevent firewall from closing idle connections
      connection-test-query: SELECT 1   # for DBs without JDBC4 isValid()
```

**Pool sizing formula (HikariCP docs):**

> `pool_size = Tn × (Cm - 1) + 1`
> where `Tn` = number of threads, `Cm` = number of concurrent connections per thread

**In practice**, for most OLTP workloads: `pool_size = (cores × 2) + effective_spindle_count`

A PostgreSQL server with 4 cores: `pool_size ≈ 9-10` per application instance.

**Common mistake:** setting `maximum-pool-size=100` when the DB can only handle 200 total connections, and you have 3 app instances → 300 connections requested, DB overwhelmed.

```yaml
# Also configure Hibernate to match HikariCP
spring:
  jpa:
    properties:
      hibernate:
        connection:
          provider_disables_autocommit: true  # Spring manages transactions, let HikariCP benefit
```

**Monitoring:**
```java
// Expose HikariCP metrics to Prometheus
@Bean
public HikariDataSource dataSource() {
    HikariDataSource ds = new HikariDataSource(hikariConfig());
    ds.setMetricRegistry(meterRegistry);  // auto-exposes pool metrics
    return ds;
}
// Metrics: hikaricp.connections.active, .idle, .pending, .timeout
```

</details>

---

### Q8 — 🔴 Compare `@BatchSize`, `JOIN FETCH`, `@EntityGraph`, and DTO projections for solving the N+1 problem from a performance perspective.

<details><summary>Click to reveal answer</summary>

**Scenario:** Load 100 orders with their items.

```java
// N+1 problem: 1 query for orders + 100 queries for items
List<Order> orders = orderRepo.findAll();
orders.forEach(o -> o.getItems().size()); // 100 lazy loads
```

**Option 1: `@BatchSize`**
```java
@Entity
public class Order {
    @OneToMany
    @BatchSize(size = 25)
    private List<OrderItem> items;
}
// Result: 1 query for orders + ceil(100/25) = 4 queries for items
// SELECT * FROM order_items WHERE order_id IN (1,2,...,25)
```
✅ Easy to configure | ❌ Still multiple queries | ❌ No control at call site

**Option 2: `JOIN FETCH` (JPQL)**
```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.userId = :uid")
List<Order> findWithItems(@Param("uid") Long uid);
// Result: 1 query with JOIN
```
✅ Single query | ❌ Cartesian product (row duplication) | ❌ Can't paginate with `LIMIT`

**Option 3: `@EntityGraph`**
```java
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByUserId(Long userId);
// Result: 1 query with LEFT JOIN (same as JOIN FETCH but declarative)
```
✅ Declarative, reusable | ❌ Same cartesian product issue | ✅ Cleaner than JPQL

**Option 4: DTO Projection (best for read performance)**
```java
// Interface projection — Spring Data generates the query
interface OrderSummary {
    Long getId();
    LocalDateTime getCreatedAt();
    @Value("#{target.items.size()}")  // SPEL
    int getItemCount();
}
List<OrderSummary> findByUserId(Long userId);

// Class-based DTO projection with JPQL constructor expression
@Query("SELECT new com.example.OrderDto(o.id, o.createdAt, COUNT(i)) " +
       "FROM Order o LEFT JOIN o.items i WHERE o.userId = :uid GROUP BY o.id, o.createdAt")
List<OrderDto> findSummaryByUserId(@Param("uid") Long userId);
```
✅ Minimal data transferred | ✅ Single query | ✅ Paginatable | ❌ No entity tracking

**Performance ranking:** DTO > JOIN FETCH / EntityGraph > @BatchSize > N+1

**Rule of thumb:** Use DTO projections for read-heavy APIs. Use JOIN FETCH/EntityGraph when you need entities with collections and the result set is bounded. Use @BatchSize as a safety net for lazy associations.

</details>
