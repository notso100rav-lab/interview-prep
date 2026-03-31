# Chapter 4: Redis Caching Strategies

> **Estimated study time:** 2 days | **Priority:** 🔴 High

---

## Key Concepts

- Cache-aside (lazy loading)
- Write-through
- Write-behind (write-back)
- Read-through
- TTL and eviction policies
- Redis data structures for caching
- Spring Boot + Redis integration
- Cache serialization

---

## Questions & Answers

---

### Q1 — 🟢 What is the cache-aside (lazy loading) pattern and how do you implement it in Spring Boot?

<details><summary>Click to reveal answer</summary>

**Cache-aside** is the most common pattern. The application manages the cache explicitly:

1. Read from cache → if hit, return cached value
2. If miss → read from DB → write to cache → return

```
Application → Cache (HIT) → return value
Application → Cache (MISS) → DB → Cache.set(value) → return value
```

**Spring Boot with `@Cacheable`:**
```java
@Service
public class ProductService {

    @Cacheable(
        value = "products",
        key = "#id",
        condition = "#id > 0",       // only cache if condition true
        unless = "#result == null"   // don't cache null results
    )
    public Product getProduct(Long id) {
        return productRepository.findById(id)
            .orElse(null);
    }

    @CacheEvict(value = "products", key = "#product.id")
    public void updateProduct(Product product) {
        productRepository.save(product);
    }

    @CachePut(value = "products", key = "#result.id")
    public Product createProduct(CreateProductRequest req) {
        return productRepository.save(mapper.toEntity(req));
    }

    // Evict all entries from the cache
    @CacheEvict(value = "products", allEntries = true)
    @Scheduled(fixedDelay = 3_600_000)  // hourly
    public void clearProductCache() {}
}
```

**Redis configuration:**
```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
  cache:
    type: redis
    redis:
      time-to-live: 600000   # 10 minutes default TTL
      cache-null-values: false
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        cacheConfigs.put("products", defaultConfig.entryTtl(Duration.ofMinutes(30)));
        cacheConfigs.put("user-sessions", defaultConfig.entryTtl(Duration.ofHours(1)));
        cacheConfigs.put("rate-limit", defaultConfig.entryTtl(Duration.ofSeconds(60)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

</details>

---

### Q2 — 🟡 What is the write-through pattern, and how does it differ from cache-aside?

<details><summary>Click to reveal answer</summary>

In **write-through**, the cache is updated synchronously on every write. Every write goes to cache AND database before returning to the caller.

```
Write request
  → Cache.set(key, value)
  → DB.write(value)
  → return OK
```

**Advantages:**
- Cache is always consistent with the database
- Reads after writes always hit the cache (no cold-start miss)

**Disadvantages:**
- Write latency includes both cache and DB write
- Unused data is cached (not all written data is read later → cache pollution)

**Manual implementation in Spring:**
```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository repository;
    private final RedisTemplate<String, Product> redisTemplate;

    public Product updateProduct(Long id, UpdateProductRequest req) {
        Product updated = repository.save(mapper.toEntity(id, req));

        // Write-through: update cache synchronously with DB
        redisTemplate.opsForValue().set(
            "product:" + id, updated, Duration.ofMinutes(30));

        return updated;
    }
}
```

**Cache-aside vs Write-through:**

| Aspect | Cache-aside | Write-through |
|--------|-------------|---------------|
| Who manages cache | Application | Application |
| Consistency on write | Eventual (evict on write) | Strong |
| Read miss on startup | Yes | No (if data pre-loaded) |
| Cache pollution | Low | Possible |
| Write latency | Lower | Higher |
| Best for | Read-heavy workloads | Read-after-write workloads |

</details>

---

### Q3 — 🟡 Explain the write-behind (write-back) pattern.

<details><summary>Click to reveal answer</summary>

In **write-behind**, writes go to the cache immediately and are persisted to the database asynchronously (in batches or after a delay).

```
Write request → Cache.set(key, value) → return OK (fast)
                      ↓ (async, batched)
               DB.write(value) [later]
```

**Advantages:**
- Very low write latency (response before DB write)
- Reduces DB write load through batching

**Disadvantages:**
- Risk of data loss if cache fails before flushing to DB
- More complex implementation (requires reliable async write pipeline)
- Hard to implement consistently across clustered Redis

**Implementation with Spring + async:**
```java
@Service
@RequiredArgsConstructor
public class MetricsService {

    private final RedisTemplate<String, Long> redisTemplate;
    private final MetricsRepository repository;

    // Fast write to Redis
    public void incrementPageView(String page) {
        redisTemplate.opsForValue().increment("views:" + page);
    }

    // Async flush to DB every minute
    @Scheduled(fixedDelay = 60_000)
    @Transactional
    public void flushViewCountsToDB() {
        Set<String> keys = redisTemplate.keys("views:*");
        if (keys == null || keys.isEmpty()) return;

        for (String key : keys) {
            String page = key.substring("views:".length());
            Long count = redisTemplate.opsForValue().getAndDelete(key);
            if (count != null && count > 0) {
                repository.incrementPageViews(page, count);
            }
        }
    }
}
```

**When to use:** analytics counters, session state, shopping cart, rate limiting — anywhere you can tolerate a small window of data loss or eventual persistence.

</details>

---

### Q4 — 🟡 What Redis eviction policies exist and how do you choose the right one?

<details><summary>Click to reveal answer</summary>

When Redis runs out of memory, it uses the configured eviction policy to decide which keys to remove:

| Policy | Behavior |
|--------|----------|
| `noeviction` | Return error on writes when memory full (default) |
| `allkeys-lru` | Evict least recently used key from all keys |
| `volatile-lru` | Evict LRU key among keys with a TTL set |
| `allkeys-lfu` | Evict least frequently used key from all keys |
| `volatile-lfu` | Evict LFU key among keys with a TTL set |
| `allkeys-random` | Evict random key from all keys |
| `volatile-random` | Evict random key among keys with a TTL set |
| `volatile-ttl` | Evict key with shortest remaining TTL |

```bash
# redis.conf / environment variable
maxmemory 512mb
maxmemory-policy allkeys-lru
```

**Choosing the right policy:**

- **Cache with TTLs** → `volatile-lru` or `volatile-lfu` (only evict cached data, not persistent keys)
- **Pure LRU cache** → `allkeys-lru` (simpler, don't need to set TTL on every key)
- **Frequency-based access** → `allkeys-lfu` (better for Zipf-distributed access patterns)
- **Session store** → `volatile-ttl` (evict sessions closest to expiry first)
- **Primary data store** → `noeviction` (never lose data silently)

```yaml
# Spring Boot
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 10
# redis.conf on server:
# maxmemory 1gb
# maxmemory-policy allkeys-lru
```

</details>

---

### Q5 — 🟡 What Redis data structures are useful for caching, and when would you choose each?

<details><summary>Click to reveal answer</summary>

| Structure | Spring Template | Best for |
|-----------|----------------|---------|
| **String** | `opsForValue()` | Simple key-value: serialized objects, counters |
| **Hash** | `opsForHash()` | Object fields; update individual fields without re-serializing |
| **List** | `opsForList()` | Ordered collections, queues, activity feeds |
| **Set** | `opsForSet()` | Unique members, tagging, membership checks |
| **Sorted Set** | `opsForZSet()` | Ranked leaderboards, scheduled jobs, rate limiting |

```java
@Service
@RequiredArgsConstructor
public class SessionCacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    // Hash: store user session fields individually
    public void storeUserSession(String sessionId, UserSession session) {
        String key = "session:" + sessionId;
        HashOperations<String, String, Object> hashOps = redisTemplate.opsForHash();
        hashOps.put(key, "userId", session.getUserId());
        hashOps.put(key, "roles", session.getRoles());
        hashOps.put(key, "loginTime", session.getLoginTime().toString());
        redisTemplate.expire(key, Duration.ofHours(1));
    }

    // Sorted Set: leaderboard
    public void updateScore(String userId, double score) {
        redisTemplate.opsForZSet().add("leaderboard", userId, score);
    }

    public Set<ZSetOperations.TypedTuple<Object>> getTopPlayers(int count) {
        return redisTemplate.opsForZSet()
            .reverseRangeWithScores("leaderboard", 0, count - 1);
    }

    // Set: track unique daily active users
    public void recordUserActivity(String date, Long userId) {
        redisTemplate.opsForSet().add("dau:" + date, userId.toString());
        redisTemplate.expire("dau:" + date, Duration.ofDays(2));
    }

    public Long getDailyActiveUsers(String date) {
        return redisTemplate.opsForSet().size("dau:" + date);
    }
}
```

</details>

---

### Q6 — 🔴 How do you implement a distributed rate limiter using Redis?

<details><summary>Click to reveal answer</summary>

A token-bucket or sliding-window rate limiter in Redis uses atomic operations to prevent race conditions across multiple app instances.

**Sliding window counter (Lua script for atomicity):**
```java
@Component
@RequiredArgsConstructor
public class RateLimiter {

    private final StringRedisTemplate redisTemplate;

    // Sliding window: allow maxRequests per windowSeconds
    private static final String SCRIPT =
        "local key = KEYS[1] " +
        "local now = tonumber(ARGV[1]) " +
        "local window = tonumber(ARGV[2]) " +
        "local max = tonumber(ARGV[3]) " +
        "local clearBefore = now - window * 1000 " +
        "redis.call('zremrangebyscore', key, '-inf', clearBefore) " +
        "local count = redis.call('zcard', key) " +
        "if count < max then " +
        "  redis.call('zadd', key, now, now) " +
        "  redis.call('pexpire', key, window * 1000) " +
        "  return 1 " +
        "else " +
        "  return 0 " +
        "end";

    private final RedisScript<Long> rateLimitScript =
        RedisScript.of(SCRIPT, Long.class);

    public boolean isAllowed(String userId, int maxRequests, int windowSeconds) {
        String key = "rate:" + userId;
        long now = System.currentTimeMillis();

        Long result = redisTemplate.execute(
            rateLimitScript,
            Collections.singletonList(key),
            String.valueOf(now),
            String.valueOf(windowSeconds),
            String.valueOf(maxRequests)
        );

        return Long.valueOf(1L).equals(result);
    }
}
```

**Spring MVC integration:**
```java
@RestControllerAdvice
@RequiredArgsConstructor
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RateLimiter rateLimiter;

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        String userId = extractUserId(req);
        if (!rateLimiter.isAllowed(userId, 100, 60)) {  // 100 req/min
            res.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            res.setHeader("Retry-After", "60");
            return false;
        }
        return true;
    }
}
```

**Why Lua script?** Redis executes Lua scripts atomically — the check-then-set is a single operation, preventing race conditions between concurrent requests.

</details>

---

### Q7 — 🔴 What is cache warming and how do you avoid cold-start problems in production?

<details><summary>Click to reveal answer</summary>

**Cold-start problem:** after a deployment or Redis restart, all cache entries are gone. The first wave of traffic hits the database directly, potentially causing:
- Spike in DB load → slow response times or outage
- Cache stampede (thundering herd) — many requests miss simultaneously and all reload the same data

**Strategies:**

**1. Pre-warm on startup:**
```java
@Component
@RequiredArgsConstructor
public class CacheWarmer implements ApplicationListener<ApplicationReadyEvent> {

    private final ProductService productService;
    private final ProductRepository repository;

    @Override
    @Async
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("Starting cache warm-up...");
        // Load top 1000 most-viewed products into cache
        repository.findTopByViewCountDesc(PageRequest.of(0, 1000))
            .forEach(p -> productService.getProduct(p.getId()));
        log.info("Cache warm-up complete");
    }
}
```

**2. Probabilistic early expiration (PER) — prevent stampede:**
```java
public Product getProductWithPER(Long id) {
    String key = "product:" + id;
    CachedValue<Product> cached = getWithMetadata(key);

    if (cached != null) {
        double ttlRemaining = cached.getTtlSeconds();
        double beta = 1.0;  // tunable aggressiveness
        double delta = cached.getRecomputeTimeMs() / 1000.0;

        // Randomly decide to refresh early to prevent stampede
        if (-delta * beta * Math.log(Math.random()) >= ttlRemaining) {
            // Pre-emptively refresh in background
            refreshAsync(id);
        }
        return cached.getValue();
    }

    return loadAndCache(id);
}
```

**3. Two-tier TTL — background refresh:**
```java
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return repository.findById(id).orElseThrow();
}

@Scheduled(fixedDelay = 25 * 60 * 1000)  // refresh every 25 min (TTL is 30 min)
public void refreshFrequentlyAccessedProducts() {
    popularProductIds.forEach(id -> {
        Product fresh = repository.findById(id).orElse(null);
        if (fresh != null) {
            cacheManager.getCache("products").put(id, fresh);
        }
    });
}
```

</details>
