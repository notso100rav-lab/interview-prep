# Chapter 5: Cache Invalidation Scenarios

> **Estimated study time:** 1 day | **Priority:** 🟡 Medium-High

---

## Key Concepts

- Stale data and consistency trade-offs
- Cache stampede / thundering herd
- Probabilistic early expiration
- Cache invalidation patterns: TTL, event-driven, write-through
- Distributed cache coherence
- Dog-piling prevention

---

## Questions & Answers

---

### Q1 — 🟢 What is cache invalidation and why is it considered hard?

<details><summary>Click to reveal answer</summary>

**Cache invalidation** is the process of removing or updating stale entries from the cache when the underlying data changes. Phil Karlton famously said:

> "There are only two hard things in Computer Science: cache invalidation and naming things."

**Why it's hard:**
1. **Distributed systems complexity**: multiple cache nodes, multiple app instances — ensuring all see the same state is a consensus problem.
2. **Timing**: the window between a DB write and cache update is a consistency gap.
3. **Dependency**: an object in cache may depend on multiple DB tables; any change to any table could stale the cache.
4. **Trade-off**: stronger consistency = lower availability/performance; weaker consistency = stale data risk.

**Main invalidation strategies:**

| Strategy | How | Consistency | Complexity |
|----------|-----|-------------|------------|
| TTL expiry | Cache entry expires after fixed duration | Eventual | Low |
| Event-driven | DB change → event → invalidate cache | Near-real-time | Medium |
| Write-through | Write updates cache and DB atomically | Strong | Medium |
| Manual eviction | Code explicitly deletes cache on write | Strong | Low (but error-prone) |

</details>

---

### Q2 — 🟡 What is a cache stampede (thundering herd) and how do you prevent it?

<details><summary>Click to reveal answer</summary>

A **cache stampede** (also called thundering herd) occurs when:
1. A popular cache entry expires
2. Many concurrent requests all miss the cache at the same time
3. All of them simultaneously hit the database to reload the same data
4. The DB becomes overwhelmed with identical queries

```
t=0: cache entry for /products/popular expires
t=0: 500 concurrent requests → all miss cache → all query DB
→ DB load spike → slow responses → potential outage
```

**Prevention strategies:**

**1. Mutex / distributed lock — only one thread fetches:**
```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final RedisTemplate<String, String> redisTemplate;
    private final ProductRepository repository;
    private final ObjectMapper objectMapper;

    public Product getProductWithLock(Long id) throws Exception {
        String cacheKey = "product:" + id;
        String lockKey = "lock:" + cacheKey;

        // Try to get from cache
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return objectMapper.readValue(cached, Product.class);
        }

        // Try to acquire lock (atomic SET NX PX)
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

        if (Boolean.TRUE.equals(acquired)) {
            try {
                // Re-check cache (another thread may have populated it)
                cached = redisTemplate.opsForValue().get(cacheKey);
                if (cached != null) {
                    return objectMapper.readValue(cached, Product.class);
                }

                // Load from DB and cache it
                Product product = repository.findById(id).orElseThrow();
                redisTemplate.opsForValue().set(
                    cacheKey,
                    objectMapper.writeValueAsString(product),
                    Duration.ofMinutes(10));
                return product;
            } finally {
                redisTemplate.delete(lockKey);
            }
        } else {
            // Lock held by another thread — wait briefly and retry
            Thread.sleep(50);
            return getProductWithLock(id);
        }
    }
}
```

**2. Probabilistic Early Expiration (PER):**
```java
public Product getWithEarlyExpiration(Long id) {
    // Instead of all threads expiring at the same time,
    // each thread independently decides to refresh early
    // based on recompute time and remaining TTL

    CachedEntry entry = fetchFromCache(id);
    if (entry == null) return loadFromDB(id);

    double ttl = entry.getRemainingTtlSeconds();
    double recomputeTime = entry.getLastComputeTimeSeconds();
    double beta = 1.0;  // tunable: higher = more aggressive early refresh

    // XFetch algorithm: refresh with probability proportional to recompute cost
    double threshold = -recomputeTime * beta * Math.log(Math.random());
    if (threshold >= ttl) {
        // This request proactively refreshes before expiry
        return loadAndCache(id);
    }
    return entry.getValue();
}
```

**3. Background refresh with stale-while-revalidate:**
```java
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return repository.findById(id).orElseThrow();
}

// Proactively refresh before TTL expires — no stampede because only scheduler runs this
@Scheduled(fixedDelay = 8 * 60 * 1000)  // every 8 min, TTL is 10 min
@Async
public void refreshProductCache(Long id) {
    Product fresh = repository.findById(id).orElse(null);
    if (fresh != null) {
        cacheManager.getCache("products").put(id, fresh);
    }
}
```

</details>

---

### Q3 — 🟡 How do you implement event-driven cache invalidation using Spring events or messaging?

<details><summary>Click to reveal answer</summary>

Event-driven invalidation decouples the write path from cache management. When the DB changes, an event is published, and all interested cache consumers invalidate or refresh their entries.

**Within a single application (Spring Events):**
```java
// Event
public record ProductUpdatedEvent(Long productId) {}

// Publisher
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ApplicationEventPublisher eventPublisher;
    private final ProductRepository repository;

    @Transactional
    public Product updateProduct(Long id, UpdateProductRequest req) {
        Product updated = repository.save(mapper.toEntity(id, req));
        // Event is published AFTER transaction commits (TransactionalEventListener)
        eventPublisher.publishEvent(new ProductUpdatedEvent(id));
        return updated;
    }
}

// Listener
@Component
@RequiredArgsConstructor
public class ProductCacheInvalidator {
    private final CacheManager cacheManager;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductUpdated(ProductUpdatedEvent event) {
        Cache cache = cacheManager.getCache("products");
        if (cache != null) {
            cache.evict(event.productId());
        }
    }
}
```

**Across microservices (Kafka):**
```java
// Producer
@Service
@RequiredArgsConstructor
public class ProductService {
    private final KafkaTemplate<String, ProductUpdatedMessage> kafka;

    @Transactional
    public Product updateProduct(Long id, UpdateProductRequest req) {
        Product updated = repository.save(mapper.toEntity(id, req));
        kafka.send("product-updates",
            String.valueOf(id),
            new ProductUpdatedMessage(id, updated.getVersion()));
        return updated;
    }
}

// Consumer (in any service that caches products)
@Component
@RequiredArgsConstructor
public class ProductCacheConsumer {
    private final CacheManager cacheManager;

    @KafkaListener(topics = "product-updates", groupId = "product-cache-invalidator")
    public void onProductUpdated(ProductUpdatedMessage msg) {
        Cache cache = cacheManager.getCache("products");
        if (cache != null) {
            cache.evict(msg.getProductId());
        }
        log.info("Invalidated product cache for id={}", msg.getProductId());
    }
}
```

**Key concern — ordering:** use the entity's `version` or event timestamp. If messages arrive out of order, a stale invalidation might restore old data. Always check whether the cached value is newer than the event before skipping invalidation.

</details>

---

### Q4 — 🟡 What is the "stale read" problem and what consistency guarantees does Redis cache-aside provide?

<details><summary>Click to reveal answer</summary>

The **stale read** problem occurs in cache-aside when:
```
T1: UPDATE product SET price=200 WHERE id=1
T1: Cache.delete("product:1")      ← evict
T2: Cache.get("product:1")         ← miss
T3: DB.select WHERE id=1 → price=200
T3: (delayed by GC pause)
T4: T1 COMMITTED to DB
T2: Cache.set("product:1", price=100)  ← T2 read BEFORE T1 committed!
→ Cache now has stale price=100 while DB has price=200
```

This race condition (read → update committed → cached old read) is rare but possible.

**Mitigation strategies:**

1. **Short TTL** — stale data expires quickly anyway; acceptable for most use cases.

2. **Versioned cache keys:**
```java
public Product getProduct(Long id) {
    Long version = productVersionRepository.getVersion(id);  // fast key-value store
    String key = "product:" + id + ":v" + version;
    Product cached = cache.get(key);
    if (cached != null) return cached;

    Product product = repository.findById(id).orElseThrow();
    cache.set(key, product, Duration.ofMinutes(10));
    return product;
}

// On update:
public void updateProduct(Long id, UpdateProductRequest req) {
    repository.save(mapper.toEntity(id, req));
    productVersionRepository.increment(id);  // old versioned key naturally becomes orphaned
}
```

3. **`@TransactionalEventListener(phase = AFTER_COMMIT)`** — ensures cache eviction happens only after the DB write is durably committed, not during the transaction.

4. **Accept eventual consistency** — for most use cases, a small window of stale data (bounded by TTL) is an acceptable trade-off for the performance benefits of caching.

</details>

---

### Q5 — 🔴 Explain distributed cache coherence. How do you keep multiple Redis nodes or application-level caches consistent?

<details><summary>Click to reveal answer</summary>

In distributed systems, multiple nodes may each have local or shared caches. Keeping them coherent is the distributed cache coherence problem.

**Scenario 1: Multiple app instances, shared Redis**
- Coherence is automatic for Redis-backed caches: all instances read from and write to the same Redis.
- Race condition risk still exists (see stale read above) but is minimized.

**Scenario 2: Multiple app instances with local (in-memory) caches (Caffeine)**
```
App1 caches product:1 locally → price=100
App2 updates product:1 → price=200, evicts its own local cache
App1 still serves price=100 from its local Caffeine cache
```

**Solution: Redis Pub/Sub for invalidation broadcast:**
```java
@Configuration
public class CacheInvalidationConfig {

    @Bean
    public MessageListenerAdapter cacheInvalidationListener(CacheInvalidationService service) {
        return new MessageListenerAdapter(service, "onInvalidationMessage");
    }

    @Bean
    public RedisMessageListenerContainer listenerContainer(
            RedisConnectionFactory factory,
            MessageListenerAdapter listener) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(listener, new PatternTopic("cache-invalidation:*"));
        return container;
    }
}

@Service
@RequiredArgsConstructor
public class CacheInvalidationService {
    private final CacheManager localCacheManager;   // Caffeine
    private final StringRedisTemplate redisTemplate;

    // Called when update happens on ANY app instance
    public void invalidate(String cacheName, Object key) {
        // Evict local cache
        Cache cache = localCacheManager.getCache(cacheName);
        if (cache != null) cache.evict(key);

        // Broadcast to all other app instances
        redisTemplate.convertAndSend(
            "cache-invalidation:" + cacheName,
            key.toString());
    }

    // Called on all app instances receiving the broadcast
    public void onInvalidationMessage(String keyStr, String channel) {
        String cacheName = channel.substring("cache-invalidation:".length());
        Cache cache = localCacheManager.getCache(cacheName);
        if (cache != null) cache.evict(keyStr);
    }
}
```

**Two-tier caching (L1 local + L2 Redis):**
```java
public Product getProduct(Long id) {
    // L1: local Caffeine (< 1ms)
    Product local = caffeineCache.getIfPresent(id);
    if (local != null) return local;

    // L2: Redis (1-5ms)
    Product redis = redisCache.get(id);
    if (redis != null) {
        caffeineCache.put(id, redis);   // populate L1
        return redis;
    }

    // DB fallback (~10ms+)
    Product fromDB = repository.findById(id).orElseThrow();
    redisCache.put(id, fromDB);
    caffeineCache.put(id, fromDB);
    return fromDB;
}
```

**On invalidation:** evict from L1 (local) AND L2 (Redis) AND broadcast to all app instances to evict their L1 caches.

</details>

---

### Q6 — 🔴 What are the failure modes of a caching layer and how do you design for resilience?

<details><summary>Click to reveal answer</summary>

**Failure mode 1: Cache miss (cold cache)**
- All traffic falls through to DB simultaneously → DB overload
- Prevention: pre-warming, staggered TTLs (jitter), gradual cache roll-out

**Failure mode 2: Redis unavailable**
- If cache is mandatory: all requests fail
- If cache is optional (circuit breaker): fall through to DB — may overload DB
```java
@Service
public class ResilientCacheService {

    @CircuitBreaker(name = "redis", fallbackMethod = "getProductFromDB")
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return repository.findById(id).orElseThrow();
    }

    // Fallback: bypass cache, go direct to DB
    public Product getProductFromDB(Long id, Throwable ex) {
        log.warn("Redis unavailable, falling back to DB for product {}", id);
        return repository.findById(id).orElseThrow();
    }
}
```

**Failure mode 3: Cache poisoning**
- Corrupted/wrong data written to cache → all reads get wrong data
- Prevention: validate data before caching; use versioned keys; monitoring on cache hit rates and value correctness

**Failure mode 4: Memory pressure / eviction causing DB flood**
- Redis evicts many popular keys → sudden spike in DB queries
- Prevention: right-size Redis memory, use `allkeys-lru` policy, set `maxmemory` alerts

**Resilience checklist:**
- [ ] Set `spring.cache.redis.time-to-live` — always use TTLs; never cache indefinitely
- [ ] Add circuit breaker around Redis calls
- [ ] Use `@Retryable` with backoff for transient Redis failures
- [ ] Monitor cache hit rate (alert if < 80%)
- [ ] Implement health check that includes Redis availability
- [ ] Add TTL jitter (`TTL + random(0..TTL*0.1)`) to spread expiry load

```java
// TTL jitter to prevent synchronized expiry
Duration baseTtl = Duration.ofMinutes(10);
Duration jitter = Duration.ofSeconds(new Random().nextInt(60));  // ±60s jitter
redisTemplate.opsForValue().set(key, value, baseTtl.plus(jitter));
```

</details>
