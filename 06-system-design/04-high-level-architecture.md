# Chapter 4: High-Level Architecture

> **Estimated study time:** 3 days | **Priority:** 🔴 High

---

## Key Concepts

- Microservices vs monolith
- Service mesh
- Event-driven architecture
- CQRS (Command Query Responsibility Segregation)
- Event sourcing
- Saga pattern
- API design at scale

---

## Questions & Answers

---

### Q1 — 🟢 What is the difference between a monolith and microservices? When should you choose each?

<details><summary>Click to reveal answer</summary>

**Monolith:** a single deployable unit where all functionality (UI, business logic, data access) is packaged together.

**Microservices:** the application is split into small, independently deployable services, each owning its own data and communicating via APIs.

| Aspect | Monolith | Microservices |
|--------|---------|---------------|
| Deployment | Single artifact | Many independent services |
| Scalability | Scale the whole app | Scale individual services |
| Technology | One language/framework | Polyglot allowed |
| Data management | Shared database | Each service owns its DB |
| Development speed | Fast initially | Slower due to coordination |
| Operational complexity | Low | High (service discovery, tracing, etc.) |
| Fault isolation | One bug can crash everything | Failures are contained |
| Team structure | Single team | Multiple autonomous teams |

**When to choose monolith:**
- Early-stage startup (speed > complexity)
- Small team (< 10 engineers)
- Domain is not yet well-understood
- Simple, low-traffic application
- "Modular monolith" as a starting point

**When to choose microservices:**
- Large team needing independent deployment cadences
- Different components have very different scaling needs
- Different parts of the system benefit from different technology stacks
- High availability requirements for specific features

**Common anti-pattern:** starting with microservices before the domain is understood leads to a "distributed monolith" — all the operational complexity with none of the independence.

> **Martin Fowler:** "Don't start with microservices. Start with a monolith and split only when you have proven scaling needs and team structures that justify it."

</details>

---

### Q2 — 🟡 What is a service mesh and what problems does it solve?

<details><summary>Click to reveal answer</summary>

A **service mesh** is an infrastructure layer that handles service-to-service communication in a microservices architecture. It adds a sidecar proxy (Envoy/Linkerd-proxy) alongside each service instance.

**Problems it solves:**

| Problem | Without Service Mesh | With Service Mesh |
|---------|---------------------|------------------|
| Mutual TLS (mTLS) | Implemented in each service | Automatic, transparent |
| Load balancing | Client-side libraries | Proxy-level, consistent |
| Circuit breaking | Per-service Resilience4j config | Centrally configured |
| Retries | Per-service | Proxy-level, consistent |
| Observability | Each service must instrument | Automatic metrics, traces |
| Traffic management | Code changes needed | Config-driven (canary, A/B) |

**Architecture:**

```
Service A → [Envoy sidecar] → [Envoy sidecar] → Service B
                ↑                     ↑
          Control plane (Istiod) manages both proxies
```

**Istio traffic management example:**
```yaml
# Canary deployment: 90% → v1, 10% → v2
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10
```

**Trade-offs:**
- ✅ Eliminates cross-cutting concerns from application code
- ✅ Consistent observability and security across all services
- ❌ Added latency (two extra network hops per call)
- ❌ Significant operational complexity
- ❌ Learning curve for Kubernetes + Istio configuration

**When to use:** large organizations with many services (20+) where maintaining network policies, observability, and security consistently across services is infeasible without automation.

</details>

---

### Q3 — 🟡 Explain event-driven architecture. What are its advantages and challenges?

<details><summary>Click to reveal answer</summary>

In **event-driven architecture (EDA)**, services communicate by publishing and consuming events rather than making direct API calls. An event represents something that happened in the domain.

```
Order Service publishes:
  → OrderPlaced { orderId, userId, items, total }

Consumers:
  → Inventory Service: reserve items
  → Payment Service: charge the user
  → Notification Service: send confirmation email
  → Analytics Service: record the sale
```

**Patterns:**

**Event notification:** minimal event payload; consumers call back for details
```json
{ "type": "OrderPlaced", "orderId": "12345" }
```

**Event-carried state transfer:** full state in the event; consumers don't need to call back
```json
{ "type": "OrderPlaced", "orderId": "12345", "items": [...], "total": 99.99 }
```

**Event sourcing:** events are the source of truth (covered in Q5)

**Advantages:**

| Advantage | Detail |
|-----------|--------|
| **Loose coupling** | Publishers don't know or depend on consumers |
| **Scalability** | Consumers can scale independently |
| **Resilience** | Consumer failure doesn't affect publisher |
| **Extensibility** | Add new consumers without changing publisher |
| **Auditability** | Event stream is a full history |

**Challenges:**

| Challenge | Detail |
|-----------|--------|
| **Eventual consistency** | Data is inconsistent for a brief window |
| **Debugging** | Harder to trace a business flow across events |
| **Idempotency** | Consumers must handle duplicate events |
| **Event ordering** | Events may arrive out of order |
| **Schema evolution** | Changing event structure without breaking consumers |
| **Dead letters** | Failed events must be captured and retried |

**Idempotent consumer pattern:**
```java
@KafkaListener(topics = "order-events")
@Transactional
public void handleOrderPlaced(OrderPlacedEvent event) {
    // Idempotency check: have we already processed this event?
    if (processedEventRepo.existsByEventId(event.getEventId())) {
        log.warn("Duplicate event, skipping: {}", event.getEventId());
        return;
    }

    inventoryService.reserveItems(event.getOrderId(), event.getItems());
    processedEventRepo.save(new ProcessedEvent(event.getEventId()));
}
```

</details>

---

### Q4 — 🟡 What is CQRS and when is it beneficial?

<details><summary>Click to reveal answer</summary>

**CQRS (Command Query Responsibility Segregation):** separate the write model (commands) from the read model (queries). They can use different data stores, schemas, and scaling strategies.

```
Command side:              Query side:
POST /orders    →    Write DB (normalized, ACID)
                          ↓ (event / CDC)
GET /orders     ←    Read DB (denormalized, optimized for reads)
```

**Simple CQRS (same DB, different models):**
```java
// Command: validates and writes via domain model
@Service
public class OrderCommandService {
    @Transactional
    public String placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.getUserId(), cmd.getItems());
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order));
        return order.getId();
    }
}

// Query: reads via lightweight DTO projection (no domain model overhead)
@Service
public class OrderQueryService {
    public List<OrderSummaryDto> getOrdersByUser(Long userId) {
        return orderSummaryRepository.findByUserId(userId);
        // Uses a DB view or pre-computed read table
    }
}
```

**Full CQRS (separate read store):**
```
Commands → PostgreSQL (normalized write model)
         → Kafka (OrderPlaced events)
         → Elasticsearch (read model, full-text search)
         → Redis (read model, dashboard aggregates)

Queries  → PostgreSQL (transactional queries)
         OR Elasticsearch (search queries)
         OR Redis (dashboard/leaderboard queries)
```

**Benefits:**
- Read and write models can be optimized independently
- Read side can scale without affecting write consistency
- Read model can be tailored for each query pattern

**When to use:**
- High read/write ratio with different shapes (common in e-commerce)
- Complex domain that benefits from separate read and write models
- Need for multiple denormalized read views of the same data

**When NOT to use:**
- Simple CRUD applications — CQRS adds significant complexity
- Small teams — the overhead of maintaining two models is not worth it

</details>

---

### Q5 — 🟡 What is Event Sourcing and how does it differ from traditional state storage?

<details><summary>Click to reveal answer</summary>

**Traditional storage:** stores current state only.
```sql
-- Only the latest state is persisted
| order_id | status   | total  |
|----------|----------|--------|
| 1        | SHIPPED  | 99.99  |
```

**Event sourcing:** stores all events that led to the current state. State is derived by replaying events.

```
Events for order 1:
  t=0: OrderPlaced   { items: [...], total: 99.99 }
  t=1: PaymentTaken  { amount: 99.99 }
  t=2: OrderShipped  { trackingId: "ABC123" }

Current state: replay all events → Order { status: SHIPPED }
```

**Implementation:**
```java
@Entity
public class OrderEvent {
    @Id @GeneratedValue
    private Long id;
    private String aggregateId;   // orderId
    private long version;          // sequence number
    private String eventType;      // "OrderPlaced", "PaymentTaken"
    private String payload;        // JSON
    private Instant occurredAt;
}

// Aggregate: rebuilds state from events
public class OrderAggregate {
    private String orderId;
    private OrderStatus status;
    private List<OrderItem> items;
    private long version;

    // Apply events to reconstruct state
    public static OrderAggregate reconstitute(List<OrderEvent> events) {
        OrderAggregate order = new OrderAggregate();
        events.forEach(order::apply);
        return order;
    }

    private void apply(OrderEvent event) {
        switch (event.getEventType()) {
            case "OrderPlaced" -> applyOrderPlaced(deserialize(event.getPayload()));
            case "PaymentTaken" -> this.status = OrderStatus.PAID;
            case "OrderShipped" -> this.status = OrderStatus.SHIPPED;
        }
        this.version = event.getVersion();
    }
}
```

**Benefits:**
- Complete audit log — know exactly what happened and when
- Time travel — rebuild state at any point in time
- Event replay — rebuild read models, fix bugs by replaying
- Debugging — full history of what went wrong

**Challenges:**
- Querying current state is expensive without snapshots
- Schema evolution — what if `OrderPlaced` structure changes?
- Eventual consistency between event store and read models
- Learning curve — requires a different mental model

**Snapshots to optimize reads:**
```java
// Instead of replaying 10,000 events, load snapshot + recent events
public OrderAggregate load(String orderId) {
    Optional<Snapshot> snapshot = snapshotRepo.findLatest(orderId);
    List<OrderEvent> events = snapshot
        .map(s -> eventRepo.findAfterVersion(orderId, s.getVersion()))
        .orElseGet(() -> eventRepo.findAll(orderId));

    OrderAggregate order = snapshot
        .map(s -> deserialize(s.getPayload(), OrderAggregate.class))
        .orElseGet(OrderAggregate::new);

    events.forEach(order::apply);
    return order;
}
```

</details>

---

### Q6 — 🔴 Explain the Saga pattern for distributed transactions.

<details><summary>Click to reveal answer</summary>

A **saga** is a sequence of local transactions, each of which publishes events or sends messages that trigger the next local transaction. If one step fails, compensating transactions are executed to undo the previous steps.

**Why not 2PC?** Two-phase commit requires all participants to be available and blocks for a long time — it's not practical for microservices.

**Choreography-based saga** (event-driven):
```
Order Service    → publishes OrderPlaced
Inventory Service → reserves items → publishes ItemsReserved
Payment Service  → charges user → publishes PaymentSucceeded
Shipping Service → creates shipment → publishes OrderFulfilled

Failure path:
Payment Service  → payment failed → publishes PaymentFailed
Inventory Service (listens to PaymentFailed) → releases items
Order Service (listens to PaymentFailed) → cancels order
```

✅ Loose coupling, no central coordinator
❌ Hard to follow the business flow across events
❌ Risk of cyclic dependencies between services

**Orchestration-based saga** (command-driven):
```java
// Saga orchestrator: one place manages the entire flow
@Component
public class PlaceOrderSaga {

    @SagaEventHandler(associationProperty = "orderId")
    void on(OrderCreatedEvent event) {
        // Step 1: reserve inventory
        commandGateway.send(new ReserveInventoryCommand(
            event.getOrderId(), event.getItems()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    void on(InventoryReservedEvent event) {
        // Step 2: process payment
        commandGateway.send(new ProcessPaymentCommand(
            event.getOrderId(), event.getUserId(), event.getTotal()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    void on(PaymentProcessedEvent event) {
        // Step 3: create shipment
        commandGateway.send(new CreateShipmentCommand(event.getOrderId()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    void on(PaymentFailedEvent event) {
        // Compensate: release inventory
        commandGateway.send(new ReleaseInventoryCommand(event.getOrderId()));
        commandGateway.send(new CancelOrderCommand(event.getOrderId()));
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "orderId")
    void on(ShipmentCreatedEvent event) {
        log.info("Order saga completed for orderId: {}", event.getOrderId());
    }
}
```

✅ Clear, centralized business logic
✅ Easy to understand the entire flow
❌ Orchestrator can become a bottleneck
❌ Orchestrator is an additional service to deploy and maintain

**Compensating transactions must be idempotent:** if the compensating message is delivered twice, the second execution should have no additional effect.

| Saga step | Compensating transaction |
|-----------|------------------------|
| Create order | Cancel order |
| Reserve inventory | Release inventory |
| Charge payment | Issue refund |
| Send email | (Not easily compensatable — log for manual review) |

</details>

---

### Q7 — 🔴 How do you design for high availability and fault tolerance in a distributed system?

<details><summary>Click to reveal answer</summary>

**High availability (HA)** means the system remains operational despite failures. **Fault tolerance** means the system continues functioning correctly when components fail.

**Availability calculation:**
```
Availability = Uptime / (Uptime + Downtime)
99.9%  = 8.7 hours downtime/year
99.99% = 52.6 minutes downtime/year
99.999% = 5.26 minutes downtime/year
```

**Key patterns:**

**1. Eliminate single points of failure:**
```
Single DB → Primary + Read Replicas + Multi-AZ failover
Single cache → Redis Cluster or Redis Sentinel
Single app server → 3+ instances behind a load balancer
Single region → Multi-region active-active or active-passive
```

**2. Circuit breaker:**
```java
@Bean
public CircuitBreakerConfig circuitBreakerConfig() {
    return CircuitBreakerConfig.custom()
        .failureRateThreshold(50)          // open if 50% of calls fail
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .permittedNumberOfCallsInHalfOpenState(3)
        .slidingWindowSize(10)
        .build();
}

@Service
public class InventoryService {
    private final CircuitBreaker circuitBreaker;

    public InventoryDto getInventory(Long productId) {
        return circuitBreaker.executeSupplier(() ->
            inventoryClient.getInventory(productId));
        // Falls back to cached data or default response when open
    }
}
```

**3. Retry with exponential backoff and jitter:**
```java
@Retryable(
    retryFor = { TemporaryServiceException.class },
    maxAttempts = 3,
    backoff = @Backoff(
        delay = 200,
        multiplier = 2,
        random = true,     // jitter: prevents synchronized retry storms
        maxDelay = 2000
    )
)
public PaymentResult chargePayment(PaymentRequest req) { ... }
```

**4. Bulkhead isolation:**
```java
// Separate thread pools for different downstream services
// Prevents one slow service from consuming all threads
@Bulkhead(name = "inventoryService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<InventoryDto> getInventoryAsync(Long productId) {
    return CompletableFuture.supplyAsync(() ->
        inventoryClient.getInventory(productId));
}
```

**5. Graceful degradation:**
```java
public ProductDto getProduct(Long id) {
    try {
        return productService.getProduct(id);
    } catch (ServiceUnavailableException e) {
        // Fall back to cached/stale data rather than returning an error
        return cacheService.getStaleProduct(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
}
```

**6. Health checks and readiness probes:**
- Kubernetes removes unhealthy pods from the load balancer (readiness)
- Kubernetes restarts crashed pods (liveness)
- Ensures traffic only reaches healthy instances

**Availability targets by tier:**
| Component | Target | Strategy |
|-----------|--------|---------|
| Load balancer | 99.99% | Multi-AZ, managed (ALB) |
| Application servers | 99.9% | 3+ instances, rolling deploys |
| Database (primary) | 99.9% | Multi-AZ RDS, automated failover |
| Cache | 99.5% | Redis Sentinel + circuit breaker |
| External APIs | 99.0% | Circuit breaker + cache fallback |

</details>
