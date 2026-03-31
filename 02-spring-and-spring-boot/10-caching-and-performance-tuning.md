# Caching & Performance Tuning in Spring Boot

> **High-throughput systems live or die by their caching strategy.**  
> Know the full stack: Spring Cache abstraction, Caffeine, Redis, Hibernate 2LC, HikariCP, and async processing.

---

## Table of Contents
1. [Spring Cache Abstraction](#spring-cache-abstraction)
2. [Cache Key Generation](#cache-key-generation)
3. [Cache Providers](#cache-providers)
4. [Caffeine Configuration](#caffeine-configuration)
5. [Redis as Cache Provider](#redis-as-cache-provider)
6. [Hibernate Second-Level Cache](#hibernate-second-level-cache)
7. [Hibernate Query Cache](#hibernate-query-cache)
8. [HikariCP Tuning](#hikaricp-tuning)
9. [Async Processing with @Async](#async-processing-with-async)
10. [Virtual Threads (Java 21)](#virtual-threads-java-21)
11. [Q&A](#qa)

---

## Spring Cache Abstraction

Enable caching with a single annotation on your configuration class:

```java
@SpringBootApplication
@EnableCaching
public class Application { }
```

### Core Annotations

```java
@Service
public class ProductService {

    // Cache the return value; key = method argument
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    // Cache the return value; key = composite
    @Cacheable(value = "products", key = "#category + '-' + #page")
    public List<Product> findByCategory(String category, int page) {
        return productRepository.findByCategory(category, PageRequest.of(page, 20));
    }

    // Update cache entry after the method runs (used for PUT/PATCH)
    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }

    // Remove a specific entry
    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    // Remove all entries in the cache
    @CacheEvict(value = "products", allEntries = true)
    public void clearAll() { }

    // Combine multiple cache operations on one method
    @Caching(
        evict = { @CacheEvict(value = "products", key = "#product.id") },
        put   = { @CachePut(value = "products", key = "#product.id") }
    )
    public Product replace(Product product) {
        return productRepository.save(product);
    }
}
```

### Conditional Caching

```java
// Only cache if the result is not null
@Cacheable(value = "products", key = "#id", unless = "#result == null")
public Product findById(Long id) { ... }

// Only cache for premium users
@Cacheable(value = "reports", key = "#id", condition = "#user.premium")
public Report getReport(Long id, User user) { ... }
```

---

## Cache Key Generation

### Default Key Strategy

Spring uses `SimpleKeyGenerator` by default:
- Zero args → `SimpleKey.EMPTY`
- One arg → the arg itself
- Multiple args → `new SimpleKey(arg1, arg2, ...)`

```java
@Cacheable("products")                   // key = SimpleKey.EMPTY (no args)
@Cacheable("products", key = "#id")      // key = id value
@Cacheable("products", key = "#p0")      // key = first param (positional)
@Cacheable("products", key = "#root.args[0]") // same as above
@Cacheable("products", key = "#root.methodName + #id") // "findById-42"
```

### SpEL Context Variables

| SpEL Expression | Description |
|---|---|
| `#paramName` | Method parameter by name |
| `#p0`, `#p1` | Positional parameters |
| `#root.methodName` | Name of the cached method |
| `#root.method` | The Method object |
| `#root.target` | The target bean instance |
| `#root.caches[0].name` | Name of the first targeted cache |
| `#result` | Return value (only in `unless` / `@CachePut`) |

### Custom KeyGenerator

```java
@Component("customKeyGenerator")
public class CustomKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        return target.getClass().getSimpleName()
            + "::" + method.getName()
            + "::" + Arrays.stream(params)
                          .map(Object::toString)
                          .collect(Collectors.joining("-"));
    }
}
```

```java
@Cacheable(value = "products", keyGenerator = "customKeyGenerator")
public Product findById(Long id) { ... }
```

---

## Cache Providers

| Provider | Type | Best For |
|---|---|---|
| `ConcurrentHashMap` | In-memory, no eviction | Dev/test only |
| `Caffeine` | In-memory, eviction, stats | Single-instance prod apps |
| `EhCache` | In-memory + disk, JCache | Legacy enterprise |
| `Redis` | Distributed, persistent | Multi-instance / microservices |
| `Hazelcast` | Distributed, in-memory | Clustered in-memory grid |

---

## Caffeine Configuration

Caffeine is the recommended local cache — it's a high-performance, near-optimal replacement for Guava Cache.

**Dependency:**
```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### Via `application.yml`

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=500,expireAfterWrite=10m
    cache-names:
      - products
      - categories
      - users
```

### Via `@Bean` (per-cache configuration)

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();

        // Default spec for all caches
        manager.setCaffeine(defaultCaffeine());

        // Register named caches with custom specs
        manager.registerCustomCache("products", buildCache(500, 10, TimeUnit.MINUTES));
        manager.registerCustomCache("users",    buildCache(1000, 30, TimeUnit.MINUTES));
        manager.registerCustomCache("reports",  buildCache(50, 1, TimeUnit.HOURS));

        return manager;
    }

    private Caffeine<Object, Object> defaultCaffeine() {
        return Caffeine.newBuilder()
            .maximumSize(200)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats(); // Enables cache hit/miss metrics
    }

    private Cache<Object, Object> buildCache(int maxSize, long duration, TimeUnit unit) {
        return Caffeine.newBuilder()
            .maximumSize(maxSize)
            .expireAfterWrite(duration, unit)
            .expireAfterAccess(duration * 2, unit)
            .recordStats()
            .build();
    }
}
```

### Caffeine Spec Parameters

| Parameter | Description |
|---|---|
| `maximumSize=N` | Max number of entries |
| `maximumWeight=N` | Max total weight (requires Weigher) |
| `expireAfterWrite=Nd/h/m/s` | TTL after write |
| `expireAfterAccess=Nd/h/m/s` | TTL after last access |
| `refreshAfterWrite=Nd/h/m/s` | Async refresh after write (stale-while-revalidate) |
| `recordStats` | Enable statistics |
| `weakKeys` / `weakValues` / `softValues` | GC-aware references |

---

## Redis as Cache Provider

**Dependencies:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### `application.yml`

```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: localhost
      port: 6379
      password: secret
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 2
          max-wait: 1000ms
```

### `RedisCacheManager` Configuration

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        // Default config: JSON serialization, 10 min TTL
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();

        // Per-cache TTL overrides
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products",   defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "users",      defaultConfig.entryTtl(Duration.ofHours(1)),
            "reports",    defaultConfig.entryTtl(Duration.ofHours(6)),
            "sessions",   defaultConfig.entryTtl(Duration.ofMinutes(15))
        );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .transactionAware() // Participate in Spring @Transactional
            .build();
    }
}
```

### Ensuring Serializability

Cached objects must be serializable. With `GenericJackson2JsonRedisSerializer`, use Jackson-compatible classes:

```java
// Add @JsonTypeInfo for polymorphic deserialization
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
public class Product implements Serializable {
    private Long id;
    private String name;
    private BigDecimal price;
    // getters/setters or use @Data (Lombok)
}
```

---

## Hibernate Second-Level Cache

The **second-level cache (2LC)** caches entity state across sessions (shared cache per `SessionFactory`).

**Dependency (EhCache or Caffeine + JCache):**
```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

### `application.properties`

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
spring.jpa.properties.javax.persistence.sharedCache.mode=ENABLE_SELECTIVE
spring.jpa.properties.hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
```

### Entity-Level Cache Annotation

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @Cache(usage = CacheConcurrencyStrategy.READ_ONLY) // cache the association too
    private Category category;
}
```

### Cache Concurrency Strategies

| Strategy | Use Case |
|---|---|
| `READ_ONLY` | Immutable data (e.g., reference tables). Fastest. |
| `NONSTRICT_READ_WRITE` | Rarely updated, eventual consistency acceptable. |
| `READ_WRITE` | Frequently read, occasionally updated. Uses soft locks. |
| `TRANSACTIONAL` | Fully transactional cache (JTA required). |

---

## Hibernate Query Cache

Caches the **result set** (list of entity IDs) of a JPQL query:

```properties
spring.jpa.properties.hibernate.cache.use_query_cache=true
```

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(String category);
```

### ⚠️ Query Cache Risks

- The query cache stores **IDs**, not entity data. Entities themselves must also be in the 2LC.
- Any DML on a table **invalidates all query cache regions** for that table — renders it useless for frequently-updated tables.
- Best for: truly static reference data (country codes, configuration tables).

---

## HikariCP Tuning

Spring Boot auto-configures HikariCP as the default connection pool.

### Key Properties

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10          # See formula below
      minimum-idle: 5                # Keep at least 5 idle connections
      connection-timeout: 30000      # 30s — max wait for a connection
      idle-timeout: 600000           # 10min — remove idle connections after this
      max-lifetime: 1800000          # 30min — retire connections to prevent stale state
      keepalive-time: 300000         # 5min — heartbeat to keep connections alive
      pool-name: MyAppHikariPool
      connection-test-query: SELECT 1  # Only needed for JDBC3 drivers
      leak-detection-threshold: 60000  # 1min — log warning if connection held this long
```

### Pool Size Formula

From HikariCP documentation:

```
pool size = Tn × (Cm - 1) + 1

Where:
  Tn = number of threads that will execute simultaneously
  Cm = number of simultaneous DB connections each thread needs
```

For a web app with 10 concurrent request threads, each needing 1 DB connection:
```
pool size = 10 × (1 - 1) + 1 = 1   (theoretical minimum)
practical: 10 connections (= number of concurrent threads)
```

**PostgreSQL-specific recommendation:** For most web applications, `maximumPoolSize` between **10 and 20** is appropriate. Larger pools are counter-productive due to DB lock contention.

### Common Tuning Mistakes

```yaml
# ❌ BAD: oversized pool
maximum-pool-size: 100  # Causes DB-side thread contention

# ❌ BAD: minimum-idle > maximum-pool-size
minimum-idle: 20
maximum-pool-size: 10

# ✅ GOOD: for fixed-size pool (recommended)
maximum-pool-size: 10
minimum-idle: 10  # Equal to max = fixed pool, no resizing overhead
```

---

## Async Processing with @Async

### Enable Async Support

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }
}
```

### Service Methods

```java
@Service
public class EmailService {

    // Fire-and-forget
    @Async("taskExecutor")
    public void sendWelcomeEmail(String to) {
        // Runs in a separate thread — caller returns immediately
        emailClient.send(to, "Welcome!");
    }

    // Return a CompletableFuture for the result
    @Async("taskExecutor")
    public CompletableFuture<String> fetchReport(Long reportId) {
        String result = reportGenerator.generate(reportId); // slow operation
        return CompletableFuture.completedFuture(result);
    }
}
```

### Combining Multiple Async Calls

```java
@Service
public class DashboardService {

    private final EmailService emailService;
    private final ReportService reportService;
    private final StatsService statsService;

    public DashboardData loadDashboard(Long userId) throws ExecutionException, InterruptedException {
        // All three start in parallel
        CompletableFuture<String> report = reportService.fetchReport(userId);
        CompletableFuture<Stats> stats  = statsService.fetchStats(userId);

        // Wait for both
        CompletableFuture.allOf(report, stats).join();

        return new DashboardData(report.get(), stats.get());
    }
}
```

### Exception Handling for @Async

```java
@Component
public class AsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(AsyncExceptionHandler.class);

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error("Async method {} threw exception: {}", method.getName(), ex.getMessage(), ex);
    }
}

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new AsyncExceptionHandler();
    }
}
```

---

## Virtual Threads (Java 21)

Java 21 Project Loom introduces **virtual threads** — lightweight threads managed by the JVM, not the OS.

### Enable in Spring Boot 3.2+

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

This switches Tomcat's request-handling threads to virtual threads automatically.

### Custom Virtual Thread Executor

```java
@Bean
public Executor virtualThreadExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

### Why Virtual Threads Matter for I/O

```java
// Traditional: platform threads block on I/O → wastes OS thread
// With virtual threads: JVM parks the virtual thread, releases carrier thread
// Net effect: can handle millions of concurrent I/O-bound tasks with minimal memory

@Async
public CompletableFuture<String> callExternalApi() {
    // With virtual threads, this blocking call doesn't waste a platform thread
    String result = restClient.get("/api/data"); // blocks virtual thread, not platform thread
    return CompletableFuture.completedFuture(result);
}
```

**When to use virtual threads:**
- High-concurrency, I/O-bound workloads (REST calls, DB queries)
- When you want to write simple blocking code without reactive programming

**When NOT to use virtual threads:**
- CPU-bound tasks (no benefit — still need platform threads for computation)
- Holding synchronized locks on pinned carrier threads (avoid `synchronized` blocks with virtual threads; use `ReentrantLock` instead)

---

## Q&A

---

🟢 **Q1: What does `@EnableCaching` do?**

<details><summary>Click to reveal answer</summary>

`@EnableCaching` activates Spring's annotation-driven cache management. It registers a `CacheInterceptor` (AOP proxy) that intercepts calls to methods annotated with `@Cacheable`, `@CachePut`, `@CacheEvict`, etc.

Without `@EnableCaching`, those annotations are silently ignored.

```java
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Spring Boot auto-configures a `CacheManager` bean. If no cache provider is found on the classpath, it defaults to `ConcurrentMapCacheManager` (in-memory, no eviction — development only).

</details>

---

🟢 **Q2: What is the difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`?**

<details><summary>Click to reveal answer</summary>

| Annotation | Executes Method? | Updates Cache? | Use Case |
|---|---|---|---|
| `@Cacheable` | Only on cache miss | Yes (on miss) | Read operations |
| `@CachePut` | Always | Yes (always) | Write/update operations |
| `@CacheEvict` | Yes | Removes entry | Delete operations |

```java
@Cacheable("products")          // Skip method if cache hit
public Product get(Long id) { ... }

@CachePut("products", key="#product.id")  // Always call method, always update cache
public Product update(Product p) { ... }

@CacheEvict("products", key="#id")       // Always call method, remove from cache
public void delete(Long id) { ... }
```

**Common mistake:** Using `@Cacheable` on a PUT endpoint — it skips the method if there's a cache hit, meaning updates are ignored.

</details>

---

🟢 **Q3: What is the default cache provider in Spring Boot and what are its limitations?**

<details><summary>Click to reveal answer</summary>

The default provider is `ConcurrentMapCacheManager` backed by `ConcurrentHashMap`.

**Limitations:**
- **No eviction**: Entries are never removed — memory grows unboundedly.
- **No TTL**: Entries never expire.
- **No size limits**: Can cause `OutOfMemoryError` under load.
- **Not distributed**: Cache is not shared across multiple JVM instances.
- **Development only**: Never use in production.

**Solution:** Add Caffeine (local) or Redis (distributed) to the classpath and configure accordingly.

</details>

---

🟡 **Q4: How does Caffeine's `expireAfterWrite` differ from `expireAfterAccess`?**

<details><summary>Click to reveal answer</summary>

- **`expireAfterWrite(duration)`**: Entry expires `duration` after it was **written** (put into cache), regardless of access. Equivalent to TTL (time-to-live). Best for data that becomes stale after a fixed time.

- **`expireAfterAccess(duration)`**: Entry expires `duration` after its **last access** (read or write). Equivalent to TTI (time-to-idle). Best for evicting unused entries.

```java
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)   // stale after 10 min of being written
    .expireAfterAccess(5, TimeUnit.MINUTES)   // evict if not touched for 5 min
    .build();
```

You can combine both: the entry expires on whichever comes first.

</details>

---

🟡 **Q5: How do you configure different TTLs for different caches with Redis?**

<details><summary>Click to reveal answer</summary>

Use `withInitialCacheConfigurations` in `RedisCacheManager`:

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration base = RedisCacheConfiguration.defaultCacheConfig()
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer()));

    return RedisCacheManager.builder(factory)
        .cacheDefaults(base.entryTtl(Duration.ofMinutes(10)))
        .withInitialCacheConfigurations(Map.of(
            "products", base.entryTtl(Duration.ofMinutes(30)),
            "sessions", base.entryTtl(Duration.ofMinutes(15)),
            "reports",  base.entryTtl(Duration.ofHours(6))
        ))
        .build();
}
```

Each named cache gets its own TTL while sharing the same serialization configuration.

</details>

---

🟡 **Q6: What are the risks of using Hibernate's query cache?**

<details><summary>Click to reveal answer</summary>

The query cache stores the **list of entity IDs** returned by a query, not the full entities.

**Risks:**
1. **Aggressive invalidation**: Any INSERT, UPDATE, or DELETE on a queried table invalidates **all** query cache regions for that table — even unrelated queries.
2. **Double lookup**: Query cache hit → still requires entity lookups (second-level cache or DB).
3. **Stale risk**: With `NONSTRICT_READ_WRITE`, concurrent writes can serve stale data briefly.
4. **Memory overhead**: Poorly configured regions grow unboundedly.

**Safe use case:**
- Truly static reference data (country codes, categories that change once a day at most).
- Combine with `READ_ONLY` second-level cache strategy and scheduled full invalidation.

</details>

---

🟡 **Q7: What is the HikariCP pool size formula and why should you avoid oversizing?**

<details><summary>Click to reveal answer</summary>

HikariCP's recommended formula (from their documentation):

```
pool size = Tn × (Cm - 1) + 1
```

Where `Tn` = max concurrent threads, `Cm` = max simultaneous connections per thread.

**Why oversizing is harmful:**
- Database engines (PostgreSQL, MySQL) have limited concurrency. Additional connections cause OS thread context-switching and lock contention on the DB side.
- Each idle connection consumes memory on both the app server and DB server.
- HikariCP's own benchmarks show performance **degrades** beyond 10–20 connections for most web workloads.

**PostgreSQL recommendation:** ~`(2 × CPU cores) + effective_spindle_count` connections for the entire DB server, shared across all apps.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # Better to start low and measure
      minimum-idle: 10        # Fixed pool = no resizing overhead
```

</details>

---

🟡 **Q8: What is `@CacheEvict(allEntries = true)` and when would you use it?**

<details><summary>Click to reveal answer</summary>

`allEntries = true` removes **every entry** in the specified cache, not just the one matching the key.

```java
@CacheEvict(value = "products", allEntries = true)
public void importProducts(List<Product> products) {
    productRepository.saveAll(products);
}
```

**Use when:**
- A bulk operation affects many (or unknown) cached entries.
- A configuration change renders all cached data stale.
- You can't enumerate individual cache keys to evict.

**Drawback:** Causes a thundering herd — all subsequent requests are cache misses until the cache warms up. Mitigate with `beforeInvocation = false` (default — evicts after method succeeds) and consider cache warming logic.

</details>

---

🔴 **Q9: Explain the `@Async` proxy limitation and how self-invocation breaks it.**

<details><summary>Click to reveal answer</summary>

`@Async` works via Spring AOP proxy. When a method calls another `@Async` method **on the same bean** (self-invocation), it bypasses the proxy and runs synchronously.

```java
@Service
public class ReportService {

    public void generateAll() {
        generateSingle(1L); // ❌ Calls proxy? NO — direct this.generateSingle()
        generateSingle(2L); // ❌ Also synchronous — no new thread
    }

    @Async
    public CompletableFuture<Void> generateSingle(Long id) { ... }
}
```

**Fixes:**
1. Move `generateSingle` to a different bean (preferred).
2. Inject `ApplicationContext` and look up the bean (awkward).
3. Use `@EnableAspectJAutoProxy(exposeProxy = true)` + `((ReportService) AopContext.currentProxy()).generateSingle(id)` (fragile).

```java
// Fix 1: separate bean
@Service
public class SingleReportService {
    @Async
    public CompletableFuture<Void> generate(Long id) { ... }
}

@Service
public class ReportService {
    private final SingleReportService singleReportService;

    public void generateAll() {
        singleReportService.generate(1L); // ✅ Goes through proxy
        singleReportService.generate(2L); // ✅ Also async
    }
}
```

</details>

---

🟡 **Q10: What is `@Caching` and when do you need it?**

<details><summary>Click to reveal answer</summary>

`@Caching` is a container annotation for grouping multiple cache annotations of the same type on a single method — something Java doesn't natively support with repeated annotations of the same type.

```java
@Caching(
    cacheable = {
        @Cacheable(value = "products", key = "#id"),
        @Cacheable(value = "catalog",  key = "'all'")
    }
)
public Product findById(Long id) { ... }

@Caching(
    evict = {
        @CacheEvict(value = "products",   key = "#id"),
        @CacheEvict(value = "catalog",    allEntries = true),
        @CacheEvict(value = "searchIndex", allEntries = true)
    }
)
public void delete(Long id) { ... }
```

**Use when:** A single operation should affect multiple caches (e.g., deleting a product should evict it from the product cache, the category cache, and the search index cache).

</details>

---

🔴 **Q11: How do you implement cache-aside pattern with Spring Cache when you need manual control?**

<details><summary>Click to reveal answer</summary>

Use `CacheManager` directly for programmatic access:

```java
@Service
public class ProductService {

    private final CacheManager cacheManager;
    private final ProductRepository repository;

    public Product findWithFallback(Long id) {
        Cache cache = cacheManager.getCache("products");

        // Try cache first
        Cache.ValueWrapper cached = cache.get(id);
        if (cached != null) {
            return (Product) cached.get();
        }

        // Cache miss — load from DB
        Product product = repository.findById(id).orElseThrow();

        // Populate cache conditionally
        if (product.isActive()) {
            cache.put(id, product);
        }

        return product;
    }

    public void invalidate(Long id) {
        Cache cache = cacheManager.getCache("products");
        cache.evict(id);
    }
}
```

This is useful when you need logic that `@Cacheable` annotations can't express (e.g., conditional caching based on the loaded entity's state).

</details>

---

🟡 **Q12: What is the Hibernate second-level cache and how does it differ from the first-level cache?**

<details><summary>Click to reveal answer</summary>

| | First-Level Cache | Second-Level Cache |
|---|---|---|
| Scope | Per `Session` (EntityManager) | Per `SessionFactory` (shared) |
| Always on? | Yes — cannot disable | No — must enable explicitly |
| Lifetime | Duration of transaction | Configurable (until eviction/expiry) |
| Shared across requests? | No | Yes |
| Stores | Full entity state | Entity state by ID |

```java
// First-level cache: automatic
EntityManager em = emf.createEntityManager();
Product p1 = em.find(Product.class, 1L); // DB query
Product p2 = em.find(Product.class, 1L); // Cache hit — same object as p1
assert p1 == p2; // true

// Second-level cache: enabled with @Cache
// After em.close(), a new session still hits cache, not DB
```

The second-level cache is populated on entity load and invalidated on update/delete. It's especially valuable for read-heavy, infrequently-changing data like reference tables.

</details>

---

🟢 **Q13: What is the purpose of `connection-timeout` vs `idle-timeout` vs `max-lifetime` in HikariCP?**

<details><summary>Click to reveal answer</summary>

- **`connection-timeout`** (default: 30s): Maximum time a thread will wait to acquire a connection from the pool. If exceeded, throws `SQLTimeoutException`. Reduce to fail fast and surface pool exhaustion.

- **`idle-timeout`** (default: 10min): Connections idle longer than this are removed from the pool (down to `minimumIdle`). Prevents holding unnecessary connections to the DB.

- **`max-lifetime`** (default: 30min): Maximum lifetime of any connection in the pool. Connections are retired when they reach this age to avoid issues with database-side connection timeouts and stale state. **Must be less than the DB's `wait_timeout`.**

```yaml
spring.datasource.hikari:
  connection-timeout: 5000    # Fail fast: 5s
  idle-timeout: 300000        # 5 min idle → removed
  max-lifetime: 1800000       # 30 min max age
```

</details>

---

🟡 **Q14: How do you measure cache effectiveness in production?**

<details><summary>Click to reveal answer</summary>

**1. Caffeine Statistics (via Micrometer):**

```java
Caffeine.newBuilder()
    .recordStats()
    .build();
```

Auto-exported to Micrometer/Actuator when `spring-boot-starter-actuator` is present:

```
cache.gets{name="products", result="hit"}    → hit count
cache.gets{name="products", result="miss"}   → miss count
cache.evictions{name="products"}             → eviction count
cache.size{name="products"}                  → current size
```

**2. Hit ratio formula:**
```
hit ratio = hits / (hits + misses)
```

Target: >80% for hot caches. Below 50% means caching is adding overhead without benefit.

**3. Spring Boot Actuator endpoint:**
```
GET /actuator/caches
GET /actuator/metrics/cache.gets
```

**4. Redis INFO command:**
```
keyspace_hits / (keyspace_hits + keyspace_misses)
```

</details>

---

🔴 **Q15: What is cache stampede (thundering herd) and how do you prevent it?**

<details><summary>Click to reveal answer</summary>

Cache stampede occurs when a popular cache entry expires and multiple concurrent requests all experience a cache miss simultaneously, all hitting the database at once.

**Prevention strategies:**

**1. Lock-based (one recomputes, others wait):**
```java
@Service
public class ProductService {
    private final ConcurrentHashMap<Long, CompletableFuture<Product>> pending = new ConcurrentHashMap<>();

    public Product findById(Long id) {
        Cache.ValueWrapper cached = cache.get(id);
        if (cached != null) return (Product) cached.get();

        CompletableFuture<Product> future = pending.computeIfAbsent(id, k ->
            CompletableFuture.supplyAsync(() -> {
                Product p = repository.findById(k).orElseThrow();
                cache.put(k, p);
                pending.remove(k);
                return p;
            })
        );

        return future.join();
    }
}
```

**2. Caffeine `refreshAfterWrite` (stale-while-revalidate):**
```java
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .refreshAfterWrite(8, TimeUnit.MINUTES) // Async refresh before expiry
    .build(key -> loadFromDb(key));         // CacheLoader required
```

**3. Probabilistic early expiration:** Refresh the cache slightly before it expires with probability proportional to how close it is to expiry.

**4. Jitter on TTL:** Add random jitter so not all entries expire simultaneously.

</details>

---

🟡 **Q16: When should you use `@Async` vs reactive programming (WebFlux)?**

<details><summary>Click to reveal answer</summary>

| Criterion | `@Async` + CompletableFuture | Reactive (WebFlux) |
|---|---|---|
| Learning curve | Low | High |
| Backpressure | No | Yes |
| Stack trace readability | Good | Poor (reactive chain) |
| Thread model | Thread-per-task | Event loop |
| Best for | Simple parallelism | High-throughput streaming |
| Java 21 alternative | Virtual threads | Still relevant for backpressure |

**Use `@Async`** when: you need to parallelize a few slow operations in a traditional Spring MVC app, or when migrating legacy code.

**Use WebFlux** when: you need full non-blocking I/O end-to-end, stream processing, or backpressure-aware pipelines.

**Java 21 note:** Virtual threads largely eliminate the thread blocking concern for `@Async` workloads, making WebFlux less necessary for I/O-bound concurrency.

</details>

---

🟡 **Q17: What is `@CacheConfig` and how does it reduce repetition?**

<details><summary>Click to reveal answer</summary>

`@CacheConfig` is a class-level annotation that sets default cache settings for all caching annotations in the class, avoiding repetition.

```java
// Without @CacheConfig — repetitive
@Service
public class ProductService {
    @Cacheable(value = "products", cacheManager = "redisCacheManager", keyGenerator = "customKeyGen")
    public Product get(Long id) { ... }

    @CacheEvict(value = "products", cacheManager = "redisCacheManager", keyGenerator = "customKeyGen")
    public void delete(Long id) { ... }
}

// With @CacheConfig — DRY
@Service
@CacheConfig(cacheNames = "products", cacheManager = "redisCacheManager", keyGenerator = "customKeyGen")
public class ProductService {
    @Cacheable
    public Product get(Long id) { ... }

    @CacheEvict
    public void delete(Long id) { ... }
}
```

Method-level annotations can still override `@CacheConfig` values when needed.

</details>

---

🔴 **Q18: How would you implement a write-through cache strategy with Spring?**

<details><summary>Click to reveal answer</summary>

Write-through ensures the cache and database are always updated together:

```java
@Service
public class ProductService {

    @CachePut(value = "products", key = "#result.id")
    @Transactional
    public Product create(CreateProductRequest request) {
        Product product = new Product(request.getName(), request.getPrice());
        return productRepository.save(product); // Saved to DB; return value cached
    }

    @CachePut(value = "products", key = "#product.id")
    @Transactional
    public Product update(Product product) {
        Product saved = productRepository.save(product);
        return saved; // Cache updated with latest DB state
    }

    @CacheEvict(value = "products", key = "#id", beforeInvocation = false)
    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id); // DB deleted; then cache evicted
    }
}
```

**Key principle:** `@CachePut` always executes the method AND updates the cache. `beforeInvocation = false` (default for `@CacheEvict`) evicts only after successful method execution — preventing cache inconsistency on rollback.

</details>

---

🟡 **Q19: What is the `leak-detection-threshold` in HikariCP?**

<details><summary>Click to reveal answer</summary>

`leakDetectionThreshold` causes HikariCP to log a warning if a connection is held longer than the specified duration without being returned to the pool.

```yaml
spring.datasource.hikari.leak-detection-threshold: 60000  # 60 seconds
```

When a leak is detected, HikariCP logs a stack trace showing where the connection was acquired, making it easy to find the culprit.

**Common causes of connection leaks:**
- Long-running transactions holding a connection
- Missing `@Transactional` causing a connection to be checked out but not properly bounded
- Exception paths that bypass connection release
- `EntityManager.close()` not called in manual EM management

**Note:** Set to `0` (default) to disable. Enable only in staging/production to detect issues; the check itself has minor overhead.

</details>

---

🟢 **Q20: Can you use Spring Cache with `@Transactional`? What are the interactions?**

<details><summary>Click to reveal answer</summary>

Yes, but there are important interactions:

```java
@Cacheable("products")
@Transactional(readOnly = true)
public Product findById(Long id) {
    return repository.findById(id).orElseThrow();
}
```

**Key behaviors:**

1. **`@Cacheable` runs before the transaction**: On a cache hit, the transaction is never started (good — saves DB resources).

2. **`@CacheEvict` default (`beforeInvocation=false`)**: Cache is evicted **after** the method returns, meaning after the transaction commits. This is correct — prevents evicting cache before the DB is updated.

3. **Transaction rollback**: If the method throws, `@CachePut` still updates the cache (the annotation runs before rollback). Use `unless="#result == null"` or programmatic caching within a `@TransactionalEventListener` for truly safe write-through.

4. **Redis transaction-aware cache**: Use `.transactionAware()` in `RedisCacheManager` to participate in Spring transactions:

```java
RedisCacheManager.builder(factory)
    .transactionAware() // Cache writes queued until TX commits
    .build();
```

</details>

---

🔴 **Q21: What are virtual threads (Project Loom) and how do they change the async programming model?**

<details><summary>Click to reveal answer</summary>

Virtual threads are **lightweight threads** managed by the JVM, not the OS. They are cheap to create (millions per JVM) and block cooperatively — when a virtual thread blocks on I/O, the JVM parks it and reuses the underlying carrier thread.

**Before Java 21:** Blocking I/O requires a platform thread. To handle 10,000 concurrent requests, you need ~10,000 OS threads (RAM-intensive, context-switch-heavy).

**With Java 21 virtual threads:** A small number of carrier threads (= CPU cores) serve millions of virtual threads. Blocking is fine again.

```java
// Spring Boot 3.2+ — just set this:
spring.threads.virtual.enabled=true

// Tomcat now uses virtual threads for request handling
// Each request gets its own virtual thread — can block freely
```

**Impact on async code:**
- `@Async` becomes less critical for I/O-bound tasks — blocking in a virtual thread is fine.
- `CompletableFuture` chains still useful for result composition.
- Reactive WebFlux: still relevant for backpressure and streaming, but less necessary for pure concurrency.

**Avoid:** `synchronized` blocks on virtual threads (they "pin" the carrier thread). Use `ReentrantLock` instead.

```java
// ❌ Pins carrier thread with virtual threads
synchronized (this) { dbOperation(); }

// ✅ Safe with virtual threads
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { dbOperation(); } finally { lock.unlock(); }
```

</details>

---

🟡 **Q22: How does Caffeine's `refreshAfterWrite` work, and how does it differ from `expireAfterWrite`?**

<details><summary>Click to reveal answer</summary>

- **`expireAfterWrite`**: Entry becomes **invalid** after the duration. Next access triggers a synchronous cache miss → reload.
- **`refreshAfterWrite`**: Entry becomes **eligible for refresh** after the duration. Next access returns the **stale value immediately** and triggers an **asynchronous reload** in the background.

```java
LoadingCache<Long, Product> cache = Caffeine.newBuilder()
    .expireAfterWrite(1, TimeUnit.HOURS)   // Hard expiry
    .refreshAfterWrite(10, TimeUnit.MINUTES) // Soft refresh
    .build(id -> productRepository.findById(id).orElseThrow());
```

**Effect:**
- At t=0: entry written
- At t=10min: next access returns stale value, refresh triggered async
- At t=10min+ε: async refresh completes, cache updated
- At t=1hr: entry hard-expires, next access blocks to load synchronously

`refreshAfterWrite` eliminates cache stampede while keeping data reasonably fresh. Requires a `CacheLoader` to work.

</details>

---

*Last updated: 2025 | Spring Boot 3.x | Java 21 | Caffeine 3.x | Redis 7.x*
