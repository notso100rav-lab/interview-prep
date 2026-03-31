# Chapter 5: Real Backend Interview Designs

> **Estimated study time:** 3 days | **Priority:** 🔴 High

---

## Overview

This chapter provides step-by-step designs for real-world backend systems commonly asked in senior engineer interviews. Each design follows the structured approach:

1. Requirements (Functional + Non-Functional)
2. Capacity Estimation
3. High-Level Design
4. Deep Dive on Key Components
5. Trade-offs & Alternatives

---

## Design 1: URL Shortener

### Requirements

**Functional:**
- Create a short URL from a long URL
- Redirect short URL → original long URL
- Custom alias support
- Expiry (optional)

**Non-Functional:**
- 100M URLs created/day (write: ~1,200/sec)
- 10B redirects/day (read: ~115,000/sec → 100:1 read/write ratio)
- Short URL length: 7 characters
- < 100ms redirect latency (p99)
- 99.99% availability for reads

### Capacity Estimation

```
Write QPS: 100M / 86,400 ≈ 1,200/sec
Read QPS:  10B / 86,400  ≈ 115,000/sec

Storage per URL: ~500 bytes
New URLs/year: 1,200 × 86,400 × 365 ≈ 38 billion/year
Storage/year: 38B × 500B ≈ 19 TB/year (keep 5 years → 95 TB)

Cache for hot URLs (80/20): 20% of 115K reads ≈ 23K hot URLs
Cache memory: 23,000 × 500B ≈ 11 MB (trivially fits in RAM)
```

### High-Level Design

```
Client → DNS → CDN (cache hot redirects)
                 ↓ (miss)
             Load Balancer
                 ↓
          Redirect Service (stateless, N instances)
                 ↓
          Redis Cache (hot URL → long URL)
                 ↓ (cache miss)
          PostgreSQL (sharded by short URL prefix)
```

### Short URL Generation

**Option A: Hash-based (MD5/SHA256 + Base62 encode)**
```java
public String shorten(String longUrl) {
    String hash = DigestUtils.md5Hex(longUrl + UUID.randomUUID());
    return Base62.encode(hash.getBytes()).substring(0, 7);
}
```
❌ Collision risk; must check DB before storing

**Option B: ID-based (Snowflake ID → Base62)**
```java
// Snowflake: 64-bit ID = timestamp(41) + datacenter(5) + worker(5) + sequence(12)
public String shorten(String longUrl) {
    long id = snowflakeIdGenerator.nextId();
    return Base62.encode(id);  // 7 chars can encode up to 62^7 = 3.5 trillion IDs
}
```
✅ No collision, globally unique, monotonically increasing (good for DB sharding)

**Base62 alphabet:** `0-9A-Za-z` (62 characters)

### Redirect Flow

```java
@GetMapping("/{shortCode}")
public ResponseEntity<Void> redirect(@PathVariable String shortCode) {
    // 1. Check cache
    String longUrl = redisTemplate.opsForValue().get("url:" + shortCode);

    // 2. Cache miss: check DB
    if (longUrl == null) {
        ShortUrl record = shortUrlRepository.findByCode(shortCode)
            .orElseThrow(() -> new UrlNotFoundException(shortCode));

        if (record.isExpired()) throw new UrlExpiredException(shortCode);

        longUrl = record.getLongUrl();
        redisTemplate.opsForValue().set("url:" + shortCode, longUrl,
            Duration.ofHours(24));
    }

    // 3. Async analytics (don't block redirect)
    analyticsPublisher.recordClick(shortCode, request);

    return ResponseEntity.status(HttpStatus.MOVED_PERMANENTLY)
        .header(HttpHeaders.LOCATION, longUrl)
        .build();
}
```

### Trade-offs

| Decision | Chose | Alternative |
|----------|-------|------------|
| ID generation | Snowflake (distributed) | DB auto-increment (single point) |
| Redirect code | 301 (permanent, cached by browser) | 302 (temporary, every click hits server → better analytics) |
| Storage | Relational (PostgreSQL) | DynamoDB (simpler operations, serverless) |
| Cache | Redis (hot URLs) | CDN (push hot URLs to edge) |

---

## Design 2: Notification Service

### Requirements

**Functional:**
- Send notifications via email, push (iOS/Android), SMS
- Templates with variable substitution
- Bulk notifications (marketing campaigns to 10M users)
- Delivery tracking (sent, delivered, failed)
- User preference management (opt-out per channel)

**Non-Functional:**
- 10M notifications/day (single sends + campaigns)
- Campaign sends: deliver within 1 hour to all subscribers
- Single sends: < 5 seconds end-to-end
- At-least-once delivery guarantee
- 99.9% availability

### High-Level Design

```
API Layer → Notification Request → Kafka (notification-requests)
                                       ↓
                             Notification Processor
                             (validates, templates, checks preferences)
                                       ↓
                      ┌────────────────┼────────────────┐
                  Kafka               Kafka           Kafka
               (email-queue)     (push-queue)     (sms-queue)
                      ↓               ↓               ↓
               Email Worker      Push Worker      SMS Worker
               (SES/SendGrid)   (FCM/APNs)       (Twilio)
                      ↓               ↓               ↓
                 DLQ (failed)    DLQ (failed)    DLQ (failed)
                      ↓
            Delivery Status DB (PostgreSQL)
```

### Key Design Decisions

**Rate limiting per channel:**
```java
@Service
public class NotificationSender {
    // Email providers have per-second limits
    @RateLimiter(name = "emailSender", fallbackMethod = "queueForLater")
    public void sendEmail(EmailNotification notification) {
        sesClient.sendEmail(buildRequest(notification));
    }
}
```

**Campaign fan-out:**
```java
@KafkaListener(topics = "campaign-requests")
public void handleCampaign(CampaignRequest campaign) {
    // Don't load all 10M users at once — stream in batches
    userRepository.streamBySegment(campaign.getSegmentId())
        .buffer(1000)  // Reactor batch
        .subscribe(batch -> {
            List<NotificationRequest> requests = batch.stream()
                .filter(user -> user.hasOptedIn(campaign.getChannel()))
                .map(user -> buildNotification(user, campaign))
                .collect(toList());
            kafkaTemplate.send("notification-requests", requests);
        });
}
```

**Idempotency for at-least-once delivery:**
```java
@KafkaListener(topics = "email-queue")
@Transactional
public void sendEmail(EmailNotification notification) {
    // Idempotency: don't send twice if message is redelivered
    if (deliveryRepo.existsByNotificationId(notification.getId())) {
        return;
    }
    sesClient.sendEmail(buildSESRequest(notification));
    deliveryRepo.save(new DeliveryRecord(
        notification.getId(), DeliveryStatus.SENT, Instant.now()));
}
```

---

## Design 3: Distributed Rate Limiter

### Requirements

**Functional:**
- Limit requests per user, per API key, or per IP
- Configurable limits (requests per second/minute/hour)
- Per-endpoint limits (e.g., /login stricter than /products)
- Return 429 with `Retry-After` header

**Non-Functional:**
- < 5ms overhead per request
- 1M QPS total
- 99.99% availability (limiter failure should fail open, not block all traffic)

### Algorithm: Sliding Window with Redis

```java
@Component
@RequiredArgsConstructor
public class SlidingWindowRateLimiter {

    private final StringRedisTemplate redisTemplate;

    private static final String SCRIPT =
        "local key = KEYS[1] " +
        "local now = tonumber(ARGV[1]) " +
        "local window_ms = tonumber(ARGV[2]) " +
        "local max_requests = tonumber(ARGV[3]) " +
        "local clear_before = now - window_ms " +
        "redis.call('ZREMRANGEBYSCORE', key, '-inf', clear_before) " +
        "local current_count = redis.call('ZCARD', key) " +
        "if current_count < max_requests then " +
        "  redis.call('ZADD', key, now, now .. '-' .. math.random()) " +
        "  redis.call('PEXPIRE', key, window_ms) " +
        "  return current_count + 1 " +
        "else " +
        "  return -1 " +
        "end";

    private final RedisScript<Long> script = RedisScript.of(SCRIPT, Long.class);

    public RateLimitResult checkLimit(String key, RateLimitConfig config) {
        long now = System.currentTimeMillis();
        Long result = redisTemplate.execute(
            script,
            List.of(key),
            String.valueOf(now),
            String.valueOf(config.getWindowMs()),
            String.valueOf(config.getMaxRequests())
        );

        if (result == null || result == -1) {
            return RateLimitResult.rejected(config.getWindowMs() / 1000);
        }
        return RateLimitResult.allowed(config.getMaxRequests() - result);
    }
}

// Spring filter
@Component
@RequiredArgsConstructor
public class RateLimitFilter extends OncePerRequestFilter {

    private final SlidingWindowRateLimiter rateLimiter;
    private final RateLimitConfigRepository configRepo;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        String apiKey = req.getHeader("X-API-Key");
        String endpoint = req.getRequestURI();
        String limitKey = "ratelimit:" + apiKey + ":" + endpoint;

        RateLimitConfig config = configRepo.getConfig(apiKey, endpoint);

        try {
            RateLimitResult result = rateLimiter.checkLimit(limitKey, config);
            res.setHeader("X-RateLimit-Limit", String.valueOf(config.getMaxRequests()));
            res.setHeader("X-RateLimit-Remaining", String.valueOf(result.remaining()));

            if (!result.allowed()) {
                res.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
                res.setHeader("Retry-After", String.valueOf(result.retryAfterSeconds()));
                return;
            }
        } catch (RedisException e) {
            // Redis down: fail open (allow traffic) to avoid blocking all users
            log.error("Rate limiter Redis failure, failing open", e);
        }

        chain.doFilter(req, res);
    }
}
```

### Trade-offs

| Decision | Chose | Why |
|----------|-------|-----|
| Algorithm | Sliding window | Accurate; no boundary burst problem |
| Storage | Redis | Sub-millisecond; atomic Lua script prevents races |
| Failure behavior | Fail open | Rate limiting is a best-effort protection; better to serve traffic than block all |
| Key granularity | API key + endpoint | Balances flexibility with Redis key count |

---

## Design 4: Chat System

### Requirements

**Functional:**
- 1:1 messaging and group messaging (up to 500 members)
- Message delivery receipts: sent, delivered, read
- Presence (online/offline/last seen)
- Push notifications for offline users
- Message history

**Non-Functional:**
- 50M DAU, each sends 40 messages/day → 2B messages/day ≈ 23K/sec
- Messages delivered in < 500ms (p99) for online users
- Store messages for 7 years
- 99.95% availability

### Architecture

```
Client → WebSocket Gateway (sticky connection per user)
              ↓ (message)
         Message Service → Kafka (messages topic)
                                ↓
                         Delivery Service (fan-out)
                         ├── WebSocket Gateway (recipient online)
                         └── Push Notification Service (offline)

Storage:
  Messages → Cassandra (append-heavy, time-series, no joins)
  User presence → Redis (expiring keys, pub/sub)
  Group membership → PostgreSQL
```

### WebSocket Connection Management

```java
@Component
@RequiredArgsConstructor
public class WebSocketConnectionManager {

    private final Map<Long, WebSocketSession> activeSessions =
        new ConcurrentHashMap<>();
    private final RedisTemplate<String, String> redisTemplate;

    public void onConnect(Long userId, WebSocketSession session) {
        activeSessions.put(userId, session);
        // Mark presence in Redis: expires if no heartbeat
        redisTemplate.opsForValue().set(
            "presence:" + userId, "online", Duration.ofSeconds(30));
        // Publish presence event
        redisTemplate.convertAndSend("presence-events",
            new PresenceEvent(userId, "ONLINE").toJson());
    }

    public void onDisconnect(Long userId) {
        activeSessions.remove(userId);
        redisTemplate.delete("presence:" + userId);
        redisTemplate.convertAndSend("presence-events",
            new PresenceEvent(userId, "OFFLINE").toJson());
    }

    public boolean isOnThisServer(Long userId) {
        return activeSessions.containsKey(userId);
    }

    public void sendMessage(Long userId, ChatMessage message) throws IOException {
        WebSocketSession session = activeSessions.get(userId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage(message.toJson()));
        }
    }
}
```

### Message Fan-out

```java
@KafkaListener(topics = "messages", groupId = "delivery-service")
public void deliverMessage(ChatMessage message) {
    List<Long> recipients = getRecipients(message);  // group members or [recipient]

    for (Long recipientId : recipients) {
        // Persist to Cassandra (regardless of online status)
        messageRepository.save(message, recipientId);

        // Route to correct WebSocket server
        String serverNodeId = redisTemplate.opsForValue()
            .get("user-server:" + recipientId);

        if (serverNodeId != null) {
            // User is online: publish to that server's Redis channel
            redisTemplate.convertAndSend(
                "ws-delivery:" + serverNodeId, message.toJson());
        } else {
            // User is offline: push notification
            pushNotificationService.send(recipientId, message);
        }
    }
}
```

### Cassandra Schema for Messages

```cql
-- Partition by conversation_id to co-locate messages
-- Cluster by message_time DESC for efficient recent-message queries
CREATE TABLE messages (
    conversation_id  UUID,
    message_id       TIMEUUID,           -- includes timestamp, globally unique
    sender_id        BIGINT,
    content          TEXT,
    message_type     TEXT,               -- TEXT, IMAGE, FILE
    status           TEXT,               -- SENT, DELIVERED, READ
    created_at       TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND gc_grace_seconds = 86400
  AND default_time_to_live = 0;          -- no expiry, archive via TTL tier

-- Get last 20 messages
SELECT * FROM messages
WHERE conversation_id = ?
LIMIT 20;

-- Paginate older messages
SELECT * FROM messages
WHERE conversation_id = ?
AND message_id < ?    -- cursor: last seen message_id
LIMIT 20;
```

---

## Design 5: Distributed Job Scheduler

### Requirements

**Functional:**
- Schedule one-time and recurring (cron) jobs
- Execute jobs with retry on failure
- Job result tracking (success, failure, last run time)
- Cancel or pause jobs
- Job priority

**Non-Functional:**
- 10M active jobs
- < 1 second scheduling delay
- At-least-once execution guarantee
- 99.9% availability

### Architecture

```
Admin API → Job Store (PostgreSQL) → Job Fetcher (leader)
                                          ↓
                                    Kafka (job-execution queue)
                                          ↓
                                    Worker Pool (N workers)
                                          ↓
                                    Job Result → PostgreSQL
```

### Leader Election and Job Fetching

```java
@Component
@RequiredArgsConstructor
public class JobScheduler {

    private final LeaderElection leaderElection;
    private final JobRepository jobRepository;
    private final KafkaTemplate<String, JobExecution> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)  // run every second
    public void fetchDueJobs() {
        // Only the leader fetches jobs (prevents double-scheduling)
        if (!leaderElection.isLeader()) return;

        List<Job> dueJobs = jobRepository.findDueJobs(
            Instant.now(), PageRequest.of(0, 1000));

        for (Job job : dueJobs) {
            // Optimistic lock: claim the job (prevents concurrent fetch)
            boolean claimed = jobRepository.claim(job.getId(),
                job.getVersion(), Instant.now().plusSeconds(30));

            if (claimed) {
                kafkaTemplate.send("job-execution",
                    new JobExecution(job.getId(), job.getPayload(), job.getPriority()));

                // Schedule next occurrence for cron jobs
                if (job.isCron()) {
                    Instant nextRun = cronExpression(job.getCron()).next(Instant.now());
                    jobRepository.scheduleNext(job.getId(), nextRun);
                }
            }
        }
    }
}
```

### Database Schema

```sql
CREATE TABLE jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    handler_class   VARCHAR(500) NOT NULL,  -- which worker handles this job type
    payload         JSONB,
    cron_expression VARCHAR(100),           -- null for one-time jobs
    next_run_at     TIMESTAMPTZ NOT NULL,
    priority        INT DEFAULT 5,          -- 1 (highest) to 10 (lowest)
    status          VARCHAR(50) DEFAULT 'PENDING',
    retry_count     INT DEFAULT 0,
    max_retries     INT DEFAULT 3,
    claimed_until   TIMESTAMPTZ,            -- optimistic lock: claimed by a worker until
    version         BIGINT DEFAULT 0,       -- optimistic locking version
    last_run_at     TIMESTAMPTZ,
    last_error      TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Index for efficient due-job fetching
CREATE INDEX idx_jobs_due ON jobs(next_run_at, priority)
WHERE status = 'PENDING' AND (claimed_until IS NULL OR claimed_until < NOW());
```

### Worker with Retry

```java
@KafkaListener(topics = "job-execution",
               groupId = "job-workers",
               concurrency = "10")  // 10 parallel consumers
public void executeJob(JobExecution execution) {
    Job job = jobRepository.findById(execution.getJobId()).orElseThrow();

    try {
        JobHandler handler = handlerRegistry.get(job.getHandlerClass());
        handler.execute(job.getPayload());

        jobRepository.markSuccess(job.getId(), Instant.now());

    } catch (Exception e) {
        log.error("Job execution failed: {}", job.getId(), e);
        int nextRetry = job.getRetryCount() + 1;

        if (nextRetry <= job.getMaxRetries()) {
            // Exponential backoff retry
            Instant nextAttempt = Instant.now()
                .plusSeconds((long) Math.pow(2, nextRetry) * 10);
            jobRepository.scheduleRetry(job.getId(), nextRetry, nextAttempt,
                e.getMessage());
        } else {
            jobRepository.markFailed(job.getId(), e.getMessage());
            alertService.notifyJobFailure(job);
        }
    }
}
```

### Trade-offs

| Decision | Chose | Alternative |
|----------|-------|------------|
| Leader election | Redis SETNX (simple) | ZooKeeper, etcd (stronger guarantees) |
| Job queue | Kafka (durable, replayable) | Redis Queue, RabbitMQ (simpler) |
| Claiming | Optimistic lock (version column) | `SELECT FOR UPDATE SKIP LOCKED` |
| Scheduling | Single leader polls every 1s | Timer wheel per worker (more complex) |

---

## Interview Tips for System Design

### The RESHADED Framework

| Step | Time | What to cover |
|------|------|--------------|
| **R**equirements | 5 min | Functional + non-functional; clarifying questions |
| **E**stimations | 3 min | QPS, storage, bandwidth; round numbers are fine |
| **S**ervice API | 3 min | Key API endpoints or event schemas |
| **H**igh-level design | 10 min | Component diagram, data flows |
| **A**rchitecture deep dive | 15 min | Most interesting or challenging component |
| **D**ata design | 5 min | Schema, storage choice rationale |
| **E**volution | 5 min | How the design scales; failure handling |
| **D**iscussion | remaining | Trade-offs, alternatives, what you would do differently |

### Common Mistakes to Avoid

1. **Jumping to solution without clarifying requirements** — always ask: "Is this read-heavy or write-heavy?", "What's the expected QPS?", "What's the consistency requirement?"

2. **Designing for unnecessary scale** — if they say 1,000 users, a monolith with PostgreSQL is the right answer; don't over-engineer.

3. **Not discussing trade-offs** — interviewers want to see that you know you're making choices, not that there's one "correct" answer.

4. **Ignoring failure modes** — proactively bring up: "What happens if Redis goes down?", "How do we handle a partition?"

5. **Not knowing when to stop** — time-box each section; don't get lost in one detail.
