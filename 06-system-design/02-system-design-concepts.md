# Chapter 2: System Design Concepts

> **Estimated study time:** 3 days | **Priority:** 🔴 High

---

## Key Concepts

- CAP theorem
- Consistency models
- Load balancing
- Horizontal vs vertical scaling
- Sharding and replication
- Rate limiting
- Message queues
- CDN
- API gateways

---

## Questions & Answers

---

### Q1 — 🟢 What is the CAP theorem and what are its practical implications?

<details><summary>Click to reveal answer</summary>

**CAP theorem** (Brewer, 2000): a distributed system can only guarantee two of the following three properties simultaneously:

| Property | Meaning |
|----------|---------|
| **C**onsistency | Every read receives the most recent write or an error |
| **A**vailability | Every request receives a non-error response (not necessarily the most recent write) |
| **P**artition tolerance | The system continues to operate despite arbitrary network partitions |

**In practice:** Network partitions are inevitable in distributed systems. So the real choice is: **CP or AP?**

| Choice | Behavior during partition | Examples |
|--------|--------------------------|---------|
| **CP** | Return error rather than stale data | PostgreSQL, HBase, etcd, ZooKeeper |
| **AP** | Return stale data rather than error | Cassandra, CouchDB, DynamoDB |

**Important nuance:** CAP is a spectrum, not a binary. Modern systems tune the trade-off:
- Cassandra: tunable consistency (`ONE`, `QUORUM`, `ALL`)
- DynamoDB: eventually consistent reads (AP) or strongly consistent reads (CP), per request

**PACELC** is a more refined model:
> Even when there is no partition (E), the system must choose between Latency (L) and Consistency (C).

</details>

---

### Q2 — 🟢 What are the main consistency models in distributed systems?

<details><summary>Click to reveal answer</summary>

From strongest to weakest:

**1. Linearizability (Strong Consistency)**
- Every operation appears to take effect instantly at some point between its start and end
- All clients see operations in the same order
- Cost: high latency, reduced availability
- Used by: Google Spanner, etcd

**2. Sequential Consistency**
- All operations appear in some sequential order consistent with each process's order
- Different from linearizability: doesn't require real-time constraint
- Used by: some distributed databases

**3. Causal Consistency**
- Causally related operations are seen in order; concurrent operations may be seen in different orders by different nodes
- "If A caused B, all nodes see A before B"
- Used by: MongoDB causal sessions, some distributed caches

**4. Read-Your-Writes Consistency**
- A client always sees its own writes
- Common requirement for user-facing applications
- Implementation: sticky sessions or session tokens

**5. Eventual Consistency**
- Given no new updates, all replicas will converge to the same value eventually
- Very high availability and low latency
- Used by: DynamoDB (default), Cassandra, DNS

```java
// DynamoDB: choose consistency per request
GetItemRequest req = GetItemRequest.builder()
    .tableName("users")
    .key(key)
    .consistentRead(true)   // strongly consistent (CP-mode, costs 2× RCU)
    .build();

// vs default: eventually consistent (AP-mode, costs 1× RCU)
GetItemRequest req = GetItemRequest.builder()
    .tableName("users")
    .key(key)
    // consistentRead defaults to false (eventual)
    .build();
```

</details>

---

### Q3 — 🟡 Explain load balancing strategies. When would you use each?

<details><summary>Click to reveal answer</summary>

**L4 (Transport layer) — routes by IP/port:**
- Works at TCP level; doesn't inspect HTTP content
- Lower latency, higher throughput
- Can't route based on URL path, headers, or cookies
- Examples: AWS NLB, HAProxy (TCP mode)

**L7 (Application layer) — routes by content:**
- Inspects HTTP headers, URLs, cookies
- Enables path-based routing, sticky sessions, SSL termination
- Higher latency than L4 (must parse HTTP)
- Examples: AWS ALB, NGINX, Envoy

**Load balancing algorithms:**

| Algorithm | How | Best for |
|-----------|-----|---------|
| **Round-robin** | Requests distributed sequentially to each server | Homogeneous servers, uniform request cost |
| **Weighted round-robin** | Servers with higher capacity get more requests | Heterogeneous server capacity |
| **Least connections** | Route to server with fewest active connections | Long-lived connections, variable request duration |
| **Least response time** | Route to server with lowest latency + fewest connections | Latency-sensitive APIs |
| **IP hash** | Hash of client IP → consistent server | Session affinity (sticky sessions) |
| **Random** | Random server selection | Simple; statistically even distribution at scale |

**Sticky sessions (session affinity):**
- Route same client to same server every time
- Required when sessions are stored in-memory (bad practice!)
- Problems: if server dies, all its sessions are lost
- Better: store sessions in Redis — no sticky sessions needed

```yaml
# AWS ALB target group: enable stickiness
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=stickiness.enabled,Value=true \
               Key=stickiness.duration_seconds,Value=86400
```

</details>

---

### Q4 — 🟡 What is database sharding and what are the main sharding strategies?

<details><summary>Click to reveal answer</summary>

**Sharding** (horizontal partitioning) splits data across multiple databases (shards). Each shard holds a subset of the data.

**Why shard:**
- Single DB can't store all the data (storage limit)
- Single DB can't handle all the read/write load (compute limit)

**Sharding strategies:**

**1. Range-based sharding:**
```
Shard 1: user_id 1 – 1,000,000
Shard 2: user_id 1,000,001 – 2,000,000
Shard 3: user_id 2,000,001 – 3,000,000
```
✅ Simple; range queries are efficient
❌ Hotspot risk (new users always go to last shard)

**2. Hash-based sharding:**
```
shard_id = hash(user_id) % num_shards
```
✅ Even distribution; no hotspots
❌ Range queries must hit all shards; adding shards requires resharding

**3. Directory-based sharding:**
```
Lookup service: user_id 12345 → shard_4
```
✅ Flexible; can rebalance shards by updating the directory
❌ Lookup service is a single point of failure

**Consistent hashing** (used in Cassandra, DynamoDB):
- Place shards on a ring
- Hash data key to a point on the ring
- Route to the next shard clockwise
- Adding/removing a shard only moves O(1/n) of keys

**Sharding challenges:**
- Cross-shard JOIN queries (must do application-level joins or denormalize)
- Cross-shard transactions (use saga pattern)
- Rebalancing when adding shards (resharding is complex)
- Schema changes must be applied to all shards

</details>

---

### Q5 — 🟡 What are the main message queue patterns and when do you use Kafka vs RabbitMQ?

<details><summary>Click to reveal answer</summary>

**Core patterns:**

**Point-to-point (queue):** one producer → one consumer
- Message is consumed by exactly one consumer
- Used for: work distribution, task queues

**Publish-subscribe (topic):** one producer → many consumers
- Every subscriber gets a copy of the message
- Used for: event broadcasting, fan-out

**Kafka architecture:**

```
Producer → Topic (with partitions) → Consumer Groups

Partition 0: [msg1, msg4, msg7, ...]
Partition 1: [msg2, msg5, msg8, ...]
Partition 2: [msg3, msg6, msg9, ...]

Consumer Group A: 3 consumers (one per partition) — fast parallel processing
Consumer Group B: 1 consumer — reads all partitions sequentially
```

**Key Kafka properties:**
- Messages are **retained** on disk (default 7 days) — consumers can replay
- Ordering guaranteed within a partition (not across partitions)
- Consumers pull messages (vs RabbitMQ push)
- High throughput (millions of msg/sec)

**Kafka vs RabbitMQ:**

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Throughput | Very high (100k–1M/sec) | High (20k–100k/sec) |
| Message retention | Yes (disk, configurable) | No (deleted after ACK) |
| Consumer model | Pull | Push |
| Ordering | Per partition | Per queue |
| Routing | Topic + partition key | Exchanges (direct, topic, fanout, headers) |
| Use case | Event streaming, log aggregation | Task queues, RPC, complex routing |

**When to choose:**
- **Kafka**: event streaming, audit log, microservice event bus, real-time analytics
- **RabbitMQ**: task queues, request-reply RPC, complex routing, low-latency delivery

</details>

---

### Q6 — 🟡 How does a Content Delivery Network (CDN) work and when should you use one?

<details><summary>Click to reveal answer</summary>

A **CDN** is a geographically distributed network of servers (edge nodes / Points of Presence) that cache content close to end users.

**How it works:**
```
User (Sydney) → CDN edge (Sydney) → Cache HIT → return content (10ms)
                                  → Cache MISS → Origin server (US) → cache → return (200ms)
```

**Pull CDN (most common):**
- CDN caches content on the first request
- TTL controlled by `Cache-Control` headers from origin
- Example: CloudFront, Cloudflare

**Push CDN:**
- You proactively push content to CDN edge nodes before requests arrive
- Better for large files that you know will be popular
- Example: Akamai NetStorage

**What to cache on CDN:**
- ✅ Static assets (JS, CSS, images, fonts)
- ✅ Public API responses with short TTL (product catalog, prices)
- ✅ Large file downloads (software packages, videos)
- ❌ User-specific or personalized content
- ❌ Authenticated API responses (security risk)

**Cache-Control headers:**
```http
# Cache publicly on CDN for 24 hours
Cache-Control: public, max-age=86400, stale-while-revalidate=3600

# Don't cache sensitive API responses
Cache-Control: private, no-store

# Cache static assets with content-hash filename for long TTL
Cache-Control: public, max-age=31536000, immutable
# File: /static/main.a1b2c3d4.js (content hash in filename)
```

**CDN benefits:**
- Reduced latency (geographic proximity)
- Reduced origin load
- DDoS mitigation (CDNs absorb large traffic spikes)
- HTTPS termination at the edge

</details>

---

### Q7 — 🔴 Explain the token bucket and sliding window rate limiting algorithms.

<details><summary>Click to reveal answer</summary>

**Token Bucket:**

A bucket holds up to `capacity` tokens. Tokens are added at a `refill_rate` per second. Each request consumes one token. If the bucket is empty, the request is rejected.

```
capacity = 100 tokens
refill_rate = 10 tokens/second

t=0: 100 tokens available
t=0: burst of 100 requests → all pass, 0 tokens left
t=1s: 10 tokens added → 10 tokens
t=1s: 10 requests pass, 0 tokens left
```

✅ Allows short bursts up to bucket capacity
✅ Smooth long-term rate enforcement
❌ Memory: need one bucket per user/IP

**Sliding Window Counter:**

Track request counts in fine-grained time buckets. Sum counts in the rolling window.

```
Window: 60 seconds, bucket size: 1 second
Max: 100 requests/minute

Current time: 12:00:45
Window: 12:00:00 – 12:00:59 (60 buckets)
Count requests in all buckets → if < 100, allow
```

✅ Accurate rate enforcement
✅ No burst problem (vs fixed window which resets at boundary)
❌ Higher memory (one counter per time bucket per user)

**Distributed implementation with Redis:**
```java
// Sliding window using sorted set (see Chapter 4: Redis Caching Strategies)
// Token bucket using Redis INCR + EXPIRE (simple version)
String key = "ratelimit:" + userId + ":" + (System.currentTimeMillis() / 1000);
Long count = redisTemplate.opsForValue().increment(key);
redisTemplate.expire(key, Duration.ofSeconds(60));
if (count > 100) { throw new RateLimitExceededException(); }
```

**Response headers:**
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1712345678
Retry-After: 30
```

</details>

---

### Q8 — 🔴 What is an API Gateway and what responsibilities should it have?

<details><summary>Click to reveal answer</summary>

An **API Gateway** is the single entry point for all client requests to a microservices backend. It handles cross-cutting concerns so individual services don't have to.

**API Gateway responsibilities:**

| Responsibility | Details |
|---------------|---------|
| **Request routing** | Route `/api/users` to User Service, `/api/orders` to Order Service |
| **Authentication** | Validate JWT / OAuth2 token before forwarding |
| **Rate limiting** | Per-client, per-endpoint limits |
| **SSL termination** | HTTPS → HTTP to internal services |
| **Request/response transformation** | Field mapping, protocol translation (REST → gRPC) |
| **Aggregation** | Fan-out to multiple services, combine responses |
| **Load balancing** | Distribute traffic to service instances |
| **Circuit breaking** | Fail fast when downstream is down |
| **Observability** | Centralized access logging, request tracing |
| **CORS handling** | Centralized cross-origin policy |
| **API versioning** | Route `/v1` vs `/v2` to different versions |

**Common choices:**
- **Kong**: open-source, Lua plugins, PostgreSQL config store
- **AWS API Gateway**: managed, Serverless-friendly, native Lambda integration
- **NGINX**: high-performance, manual configuration
- **Spring Cloud Gateway**: Spring-native, reactive, programmatic routing
- **Istio / Envoy**: service mesh, more powerful but complex

**Spring Cloud Gateway example:**
```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway", "spring-cloud-gateway")
                    .circuitBreaker(c -> c
                        .setName("userServiceCB")
                        .setFallbackUri("forward:/fallback/users"))
                    .requestRateLimiter(l -> l
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://user-service"))
            .route("order-service", r -> r
                .path("/api/orders/**")
                .uri("lb://order-service"))
            .build();
    }
}
```

**What should NOT be in the gateway:**
- Business logic
- DB access
- Complex orchestration (use an orchestrator service for that)

</details>
