# Transactions & Propagation

> 💡 **Interviewer Tip:** Transaction management is one of the most heavily tested Spring topics at senior level. The self-invocation pitfall, propagation types, and isolation levels are asked in almost every interview. Be ready to draw diagrams and walk through scenarios.

---

## Table of Contents
1. [Propagation Types](#propagation-types)
2. [Isolation Levels](#isolation-levels)
3. [Common Pitfalls](#common-pitfalls)
4. [Interview Questions](#interview-questions)

---

## Propagation Types

| Propagation | Behaviour |
|------------|-----------|
| `REQUIRED` | Join existing tx; create new if none (default) |
| `REQUIRES_NEW` | Always create a new tx; suspend existing |
| `NESTED` | Create a savepoint within existing tx |
| `SUPPORTS` | Join if exists; run non-transactionally if not |
| `NOT_SUPPORTED` | Always run non-transactionally; suspend existing |
| `MANDATORY` | Must join existing tx; throw if none |
| `NEVER` | Must NOT be in a tx; throw if one exists |

## Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE_READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

---

## Interview Questions

### 🟢 Easy

---

**Q1. What does `@Transactional` do in Spring?**

<details><summary>Click to reveal answer</summary>

`@Transactional` is a declarative way to demarcate transaction boundaries. Spring creates an AOP proxy around the annotated method/class that:
1. Starts a transaction before the method executes
2. Commits on successful completion
3. Rolls back on unchecked exceptions (by default)

```java
@Service
@Transactional  // class-level: all public methods are transactional
public class OrderService {

    private final OrderRepository orderRepo;
    private final InventoryRepository inventoryRepo;

    @Transactional  // method-level: can override class-level settings
    public Order placeOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));            // 1. Insert order
        inventoryRepo.decrementStock(req.getProductId(), req.getQty()); // 2. Update stock

        if (order.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalStateException("Order total must be positive");
            // RuntimeException → automatic rollback of BOTH operations
        }

        return order;
    }

    @Transactional(readOnly = true) // optimization for read-only operations
    public Order findById(Long id) {
        return orderRepo.findById(id).orElseThrow();
    }
}
```

Key attributes of `@Transactional`:
- `propagation` — how to handle existing transactions
- `isolation` — database isolation level
- `readOnly` — hints to optimize read operations
- `timeout` — seconds before transaction times out
- `rollbackFor` — which exceptions trigger rollback
- `noRollbackFor` — which exceptions do NOT trigger rollback

</details>

---

**Q2. What is the default rollback behavior of `@Transactional`?**

<details><summary>Click to reveal answer</summary>

By default, Spring rolls back **only on unchecked exceptions** (`RuntimeException` and `Error`). Checked exceptions do **not** trigger a rollback.

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment(Payment payment) throws PaymentException {
        paymentRepo.save(payment);
        chargeCard(payment); // throws checked PaymentException
        // ❌ NOT rolled back by default — PaymentException is checked!
    }

    @Transactional(rollbackFor = PaymentException.class)
    public void processPaymentSafely(Payment payment) throws PaymentException {
        paymentRepo.save(payment);
        chargeCard(payment);
        // ✅ Rolled back — explicitly specified
    }

    @Transactional
    public void processWithRuntimeException(Payment payment) {
        paymentRepo.save(payment);
        throw new PaymentProcessingException("Card declined"); // RuntimeException subclass
        // ✅ Rolled back automatically
    }

    // Prevent rollback on specific RuntimeException
    @Transactional(noRollbackFor = OptimisticLockingFailureException.class)
    public void updateWithRetry(Payment payment) {
        paymentRepo.save(payment);
        // OptimisticLockingFailureException → NOT rolled back (caller handles retry)
    }
}
```

```java
// Custom exception hierarchy example
public class BusinessException extends RuntimeException { }   // unchecked → rolls back
public class ValidationException extends RuntimeException { } // unchecked → rolls back
public class RecoverableException extends Exception { }       // checked → NO rollback by default
```

🚨 **Common Mistake:** Using checked exceptions for business errors and expecting them to trigger rollback. You must either:
1. Use `rollbackFor = MyCheckedException.class`
2. Or convert checked to unchecked before throwing

</details>

---

**Q3. What does `@Transactional(readOnly = true)` mean?**

<details><summary>Click to reveal answer</summary>

`readOnly = true` is a **hint** to the underlying infrastructure that the transaction will not modify data. Benefits:

1. **Hibernate optimization**: disables dirty checking (no need to detect changed entities)
2. **Connection pool**: some pools provide read replicas for read-only transactions
3. **Database optimization**: some databases (e.g., MySQL) can optimize read-only transactions
4. **Intent documentation**: communicates that this method should not modify state

```java
@Service
public class ReportService {

    // readOnly = true: Hibernate won't track entity changes, flushes are skipped
    @Transactional(readOnly = true)
    public ReportDto generateReport(Long userId) {
        User user = userRepo.findById(userId).orElseThrow();
        List<Order> orders = orderRepo.findByUser(user);
        // Hibernate persistence context won't dirty-check user and orders at flush time
        return reportMapper.toDto(user, orders);
    }

    // readOnly = false (default): dirty checking is active
    @Transactional
    public User updateUser(Long id, UpdateUserRequest req) {
        User user = userRepo.findById(id).orElseThrow();
        user.setName(req.getName()); // change tracked → auto-flushed on commit
        return user; // no explicit save needed due to dirty checking
    }
}
```

```java
// Class-level readOnly with method-level override
@Service
@Transactional(readOnly = true)  // most methods are reads
public class ProductService {

    public List<Product> findAll() { /* ... */ return null; }
    public Product findById(Long id) { /* ... */ return null; }

    @Transactional  // overrides class-level readOnly for write operations
    public Product create(CreateProductRequest req) { /* ... */ return null; }

    @Transactional  // readOnly = false for writes
    public void delete(Long id) { /* ... */ }
}
```

✅ **Best Practice:** Set `readOnly = true` at class level for service classes that are primarily read operations, then override with `@Transactional` (readOnly = false) for write methods.

</details>

---

**Q4. What is `REQUIRED` propagation and why is it the default?**

<details><summary>Click to reveal answer</summary>

`REQUIRED` is the default propagation. It means:
- If there is an existing transaction, **join it**
- If there is no transaction, **create a new one**

```java
@Service
public class OrderService {

    @Autowired private PaymentService paymentService;
    @Autowired private InventoryService inventoryService;

    @Transactional  // propagation = REQUIRED (default)
    public Order placeOrder(CreateOrderRequest req) {
        Order order = createOrder(req);          // in transaction T1
        paymentService.charge(req);              // joins T1 (REQUIRED)
        inventoryService.decrementStock(req);    // joins T1 (REQUIRED)
        return order;
        // T1 commits → all three operations commit together
        // If any fails → all three roll back together
    }
}

@Service
public class PaymentService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void charge(CreateOrderRequest req) {
        // Joins the existing transaction from OrderService
        // Does NOT create a new transaction
        paymentRepo.save(new Payment(req));
    }
}

@Service
public class InventoryService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void decrementStock(CreateOrderRequest req) {
        // Also joins the same transaction
        Product p = productRepo.findById(req.getProductId()).orElseThrow();
        p.decrementStock(req.getQty());
    }
}
```

Why it's the default: in most cases, you want all operations within a use case to share a single atomic transaction. `REQUIRED` achieves this without requiring coordination between service methods.

</details>

---

### 🟡 Medium

---

**Q5. What is the difference between `REQUIRES_NEW` and `NESTED` propagation?**

<details><summary>Click to reveal answer</summary>

```java
// REQUIRES_NEW: completely independent transaction
@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAuditEvent(AuditEvent event) {
        // Suspends the caller's transaction
        // Creates a brand-new, independent transaction
        // This transaction commits/rolls back independently
        auditRepo.save(event);
        // Even if the outer transaction fails, this audit log is saved
    }
}

// NESTED: savepoint within the same transaction
@Service
public class OrderService {

    @Transactional
    public void processOrder(Order order) {
        orderRepo.save(order);
        try {
            notificationService.sendEmail(order); // NESTED propagation
        } catch (Exception e) {
            // Rollback to savepoint — only sendEmail is rolled back
            // order is still saved
            log.warn("Notification failed, but order is saved");
        }
        // Outer transaction still commits with the order
    }
}

@Service
public class NotificationService {
    @Transactional(propagation = Propagation.NESTED)
    public void sendEmail(Order order) {
        // Creates a savepoint in the outer transaction
        notificationRepo.save(new Notification(order));
        // If this fails, only rolls back to the savepoint
        // The outer transaction can still commit
    }
}
```

**Key differences:**

| Aspect | `REQUIRES_NEW` | `NESTED` |
|--------|---------------|---------|
| New transaction? | Yes — fully independent | No — savepoint within existing |
| Outer tx rollback affects it? | No | Yes — inner rolls back with outer |
| If inner fails | Only inner rolls back | Rolls back to savepoint |
| Database support | Universal | Requires savepoint support |
| Use case | Audit logging, independent operations | Optional sub-operations |

🚨 **Common Mistake:** Using `REQUIRES_NEW` when you need the inner operation to also rollback if the outer fails. Use `REQUIRED` (default) for that — they share the same transaction.

</details>

---

**Q6. What is the self-invocation problem with `@Transactional`?**

<details><summary>Click to reveal answer</summary>

Spring AOP creates a proxy around your bean. When a method calls another method **on the same class**, it bypasses the proxy — no AOP interception occurs.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        // ... save order
        validateOrder(order);  // ❌ DIRECT CALL — bypasses proxy!
                               // The @Transactional on validateOrder is IGNORED
                               // It runs in placeOrder's transaction (or no new tx)
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) {
        // This @Transactional is NEVER processed when called from placeOrder
        // Because: OrderService.placeOrder() → this.validateOrder() (not proxy.validateOrder())
    }
}
```

**Solutions:**

**1. Inject self (use the proxy)**
```java
@Service
public class OrderService {

    @Autowired
    private OrderService self; // inject proxy of self

    @Transactional
    public void placeOrder(Order order) {
        self.validateOrder(order); // ✅ Goes through the proxy
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) { /* ... */ }
}
```

**2. Extract to a separate class (best approach)**
```java
@Service
public class OrderService {

    @Autowired private OrderValidator orderValidator;

    @Transactional
    public void placeOrder(Order order) {
        orderValidator.validate(order); // ✅ Different bean → proxy intercepts
    }
}

@Service
public class OrderValidator {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validate(Order order) { /* ... */ }
}
```

**3. `AopContext.currentProxy()` (not recommended)**
```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        ((OrderService) AopContext.currentProxy()).validateOrder(order); // ✅ but ugly
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) { /* ... */ }
}
// Requires: @EnableAspectJAutoProxy(exposeProxy = true)
```

> 💡 **Interviewer Tip:** The self-invocation problem is the most common `@Transactional` gotcha. Knowing all three solutions and recommending "extract to separate class" demonstrates senior-level thinking.

</details>

---

**Q7. Explain how isolation levels prevent concurrency anomalies.**

<details><summary>Click to reveal answer</summary>

**Concurrency anomalies:**

```java
// 1. Dirty Read — reading uncommitted data from another transaction
// Transaction T1 writes row, T2 reads it before T1 commits, T1 rollbacks
// T2 has now read data that never existed
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public BigDecimal getBalance(Long accountId) {
    // Can read uncommitted changes from other transactions — dangerous!
    return accountRepo.getBalance(accountId);
}

// 2. Non-Repeatable Read — same row returns different values within a transaction
// T1 reads row, T2 updates+commits, T1 reads same row → different value
@Transactional(isolation = Isolation.READ_COMMITTED)  // default in most DBs
public void processOrder(Long orderId) {
    Product p = productRepo.findById(1L); // reads price = 100
    // Another transaction updates price to 200 and commits
    BigDecimal price = productRepo.getPrice(1L); // reads price = 200 (non-repeatable!)
    // p.getPrice() != price — inconsistency within the same transaction!
}

// 3. Phantom Read — re-executing a query returns different rows
// T1 counts rows, T2 inserts+commits, T1 re-counts → different count
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void auditOrders() {
    long count1 = orderRepo.countPendingOrders(); // 10
    // Another tx inserts an order
    long count2 = orderRepo.countPendingOrders(); // 11 — phantom row!
}
```

```java
// Choosing isolation level in Spring
@Transactional(isolation = Isolation.READ_COMMITTED)  // balance: performance vs safety
public void readUserData(Long userId) { /* ... */ }

@Transactional(isolation = Isolation.SERIALIZABLE)  // safest, slowest
public void criticalBankingOperation(Long accountId) { /* ... */ }

@Transactional(isolation = Isolation.REPEATABLE_READ)
public void generateFinancialReport() { /* ... */ }
```

**Performance vs. safety tradeoff:**
```
READ_UNCOMMITTED → READ_COMMITTED → REPEATABLE_READ → SERIALIZABLE
      ↑                                                      ↑
  Max throughput                                      Max correctness
  Min locking                                         Max locking
```

✅ **Best Practice:** Use the database's default isolation level (`READ_COMMITTED` for PostgreSQL/Oracle, `REPEATABLE_READ` for MySQL/InnoDB) unless you have a specific reason to override.

</details>

---

**Q8. What is `MANDATORY` propagation and when would you use it?**

<details><summary>Click to reveal answer</summary>

`MANDATORY` requires an existing transaction. If none exists, it throws `IllegalTransactionStateException`.

```java
@Service
public class AuditRepository {

    // MANDATORY: this method MUST be called within an existing transaction
    // Guarantees audit writes are always atomic with the business operation
    @Transactional(propagation = Propagation.MANDATORY)
    public void recordChange(EntityChangeEvent event) {
        changeLogRepo.save(event);
    }
}

// Correct usage — called from within a transaction
@Service
public class UserService {
    @Transactional
    public void updateUser(Long id, UpdateRequest req) {
        User user = userRepo.findById(id).orElseThrow();
        user.setName(req.getName());
        auditRepo.recordChange(new EntityChangeEvent(user)); // ✅ OK: in a transaction
    }
}

// Incorrect usage — no outer transaction
public class BackgroundJob {
    public void run() {
        auditRepo.recordChange(new EntityChangeEvent()); // ❌ Throws IllegalTransactionStateException
    }
}
```

**Use cases for `MANDATORY`:**
- Internal repository/DAO methods that must be part of a larger transaction
- Enforce that certain operations are never called standalone (architectural guard)
- Audit/change-tracking that must be atomic with business writes

</details>

---

**Q9. How do you implement distributed transactions, and what are the alternatives?**

<details><summary>Click to reveal answer</summary>

**Two-Phase Commit (2PC) — traditional but problematic:**
```java
// XA Transaction (distributed ACID)
@Configuration
public class XaDataSourceConfig {

    @Bean
    public XADataSource xaDataSource() {
        PGXADataSource ds = new PGXADataSource();
        ds.setUrl("jdbc:postgresql://localhost:5432/mydb");
        return ds;
    }
}
// Problems: blocking protocol, performance bottleneck, coordinator SPOF
```

**Saga Pattern — preferred for microservices:**
```java
// Choreography-based Saga (event-driven)
@Service
public class OrderSaga {

    // Step 1: Create order
    @Transactional
    public void handlePlaceOrderCommand(PlaceOrderCommand cmd) {
        Order order = orderRepo.save(new Order(cmd));
        eventPublisher.publish(new OrderCreatedEvent(order.getId(), cmd.getUserId(),
            cmd.getAmount()));
    }
}

@Service
public class PaymentSaga {

    // Step 2: Charge payment (on OrderCreatedEvent)
    @Transactional
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            paymentRepo.save(new Payment(event.getOrderId(), event.getAmount()));
            eventPublisher.publish(new PaymentSucceededEvent(event.getOrderId()));
        } catch (InsufficientFundsException e) {
            // Compensating transaction
            eventPublisher.publish(new PaymentFailedEvent(event.getOrderId()));
        }
    }
}

@Service
public class OrderCompensationSaga {

    // Compensating transaction: cancel order if payment fails
    @Transactional
    @EventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepo.findById(event.getOrderId()).orElseThrow();
        order.cancel();  // compensating action
        orderRepo.save(order);
    }
}
```

**Outbox Pattern — ensure event delivery:**
```java
@Service
public class OrderService {

    @Transactional
    public Order placeOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));

        // Save event to outbox table IN THE SAME TRANSACTION
        outboxRepo.save(new OutboxEvent(
            "ORDER_CREATED",
            objectMapper.writeValueAsString(new OrderCreatedEvent(order.getId()))
        ));
        // A separate background process reads the outbox and publishes to message broker
        // This guarantees at-least-once delivery

        return order;
    }
}
```

> 💡 **Interviewer Tip:** In microservices interviews, the preferred answer is "Saga pattern with the Outbox Pattern for reliable messaging" — not XA/2PC which has too many operational drawbacks.

</details>

---

**Q10. What happens if a `@Transactional` method is called by a non-Spring-managed class?**

<details><summary>Click to reveal answer</summary>

If a Spring bean's `@Transactional` method is called by a class that is not a Spring-managed bean (i.e., instantiated with `new`), the transaction proxy is **bypassed** entirely.

```java
// Non-Spring class
public class ExternalCaller {

    public void doSomething() {
        // Instantiated with new — Spring doesn't manage this
        OrderService service = new OrderService(); // ❌ NO proxy!
        service.placeOrder(new CreateOrderRequest()); // @Transactional IGNORED
    }
}

// Correct: use Spring-managed beans
@Component
public class SpringManagedCaller {

    @Autowired
    private OrderService orderService; // ✅ Spring proxy injected

    public void doSomething() {
        orderService.placeOrder(new CreateOrderRequest()); // @Transactional WORKS
    }
}
```

```java
// Another gotcha: @Transactional on private methods is IGNORED
@Service
public class OrderService {

    @Transactional  // ❌ IGNORED — CGLIB cannot override private methods
    private void internalProcess() {
        orderRepo.save(new Order());
    }

    @Transactional  // ✅ Works — public methods can be proxied
    public void publicProcess() {
        internalProcess(); // self-invocation problem applies here too
    }
}
```

🚨 **Common Mistake:** `@Transactional` does NOT work on:
1. `private` methods
2. `final` methods (CGLIB can't subclass)
3. Methods called via `this` (self-invocation)
4. Instances created with `new` (non-Spring beans)
5. `static` methods

</details>

---

### 🔴 Hard

---

**Q11. How does `@Transactional` interact with Spring's `PlatformTransactionManager`?**

<details><summary>Click to reveal answer</summary>

```java
// Spring uses a PlatformTransactionManager abstraction
// Different implementations for different persistence technologies:
// - DataSourceTransactionManager (JDBC)
// - JpaTransactionManager (JPA/Hibernate)
// - JtaTransactionManager (XA/distributed)
// - ReactiveTransactionManager (WebFlux/R2DBC)

@Configuration
public class TransactionConfig {

    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        JpaTransactionManager txManager = new JpaTransactionManager();
        txManager.setEntityManagerFactory(emf);
        return txManager;
    }
}

// Using multiple transaction managers
@Service
public class MultiDataSourceService {

    // Primary transaction manager (default)
    @Transactional("primaryTxManager")
    public void writeToDatabase1() { /* ... */ }

    // Secondary transaction manager
    @Transactional("secondaryTxManager")
    public void writeToDatabase2() { /* ... */ }
}

// Programmatic transaction management (for fine-grained control)
@Service
public class ManualTransactionService {

    @Autowired private PlatformTransactionManager txManager;

    public void complexOperation() {
        TransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = txManager.getTransaction(def);

        try {
            // ... do work
            txManager.commit(status);
        } catch (Exception e) {
            txManager.rollback(status);
            throw e;
        }
    }

    // Or using TransactionTemplate (simpler)
    @Autowired private TransactionTemplate transactionTemplate;

    public User createUserProgrammatically(CreateUserRequest req) {
        return transactionTemplate.execute(status -> {
            try {
                User user = userRepo.save(new User(req));
                auditRepo.log(user);
                return user;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

</details>

---

**Q12. How do you test `@Transactional` behavior in Spring Boot tests?**

<details><summary>Click to reveal answer</summary>

```java
// @Transactional on test method — rolls back after each test
@SpringBootTest
@Transactional  // each test runs in a transaction that is rolled back after
class OrderServiceTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepo;

    @Test
    void placeOrder_persistsToDatabase() {
        // Data setup is within the test transaction
        CreateOrderRequest req = new CreateOrderRequest(userId, productId, qty);
        Order order = orderService.placeOrder(req);

        assertThat(order.getId()).isNotNull();
        assertThat(orderRepo.findById(order.getId())).isPresent();
        // Transaction is rolled back after this test — no cleanup needed
    }

    @Test
    @Rollback(false)  // override — do NOT rollback this test's data
    void placeOrder_committedData() {
        // Data will persist in the database after this test
    }
}

// Testing that rollback ACTUALLY happens
@SpringBootTest
class TransactionRollbackTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepo;

    @Test
    void failedOrder_doesNotPersist() {
        CreateOrderRequest req = new CreateOrderRequest(null, null, -1); // invalid

        assertThrows(Exception.class, () -> orderService.placeOrder(req));

        // Verify rollback occurred
        assertThat(orderRepo.count()).isZero();
    }
}

// Testing REQUIRES_NEW — separate transaction, always commits
@SpringBootTest
class RequiresNewTest {

    @Autowired AuditService auditService; // REQUIRES_NEW

    @Test
    @Transactional  // outer test transaction
    void auditLog_survivesOuterRollback() {
        auditService.log("test event"); // commits independently (REQUIRES_NEW)
        // Force outer tx to rollback
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

        // After this test (outer tx rolled back), audit log should still exist
        // But since we can't verify inside a rolled-back tx, use a separate query
    }
}

// Using @TestTransaction for fine-grained control
@SpringBootTest
class TestTransactionTest {

    @Test
    void testWithManualControl() {
        TestTransaction.start();

        orderService.placeOrder(new CreateOrderRequest());

        TestTransaction.flagForRollback();
        TestTransaction.end(); // rolls back
    }
}
```

</details>

---

**Q13. What is an `@TransactionalEventListener` and how does it differ from `@EventListener`?**

<details><summary>Click to reveal answer</summary>

```java
// @EventListener — receives events immediately when published
// Problem: if the publisher's transaction rolls back, the listener already ran!

@EventListener
public void handleOrderCreated(OrderCreatedEvent event) {
    // Called immediately when event is published
    // If the publisher's tx rolls back, this already fired — inconsistency!
    emailService.sendOrderConfirmation(event.getOrderId()); // email sent for rolled-back order!
}

// @TransactionalEventListener — receives events only AFTER transaction commits
@Component
public class OrderEventHandler {

    // Default: AFTER_COMMIT — fires only if the publishing transaction commits
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // ✅ Only fires if the order was actually saved
        emailService.sendOrderConfirmation(event.getOrderId());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleOrderFailed(OrderCreatedEvent event) {
        // Fires when the publishing transaction rolls back
        alertService.notifyOrderFailed(event.getOrderId());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void handleOrderCompletion(OrderCreatedEvent event) {
        // Fires after transaction completes (commit OR rollback)
        metricsService.recordAttempt(event.getOrderId());
    }

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handleBeforeCommit(OrderCreatedEvent event) {
        // Fires before the publishing transaction commits
        // Can still rollback the transaction by throwing an exception
        auditService.validateBeforeCommit(event);
    }
}
```

```java
// Publishing events
@Service
public class OrderService {

    @Autowired private ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order placeOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));
        // Event is held until transaction commits (with @TransactionalEventListener)
        return order;
    }
}
```

🚨 **Common Mistake:** The `@TransactionalEventListener` method **cannot** run in the same transaction as the publisher (that transaction has already committed). If you need a new transaction in the handler, annotate it with `@Transactional(propagation = REQUIRES_NEW)`.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void processAfterCommit(OrderCreatedEvent event) {
    // Runs in its own NEW transaction (not the committed publisher's tx)
    rewardService.calculateRewards(event.getOrderId());
}
```

</details>

---

**Q14. How does Hibernate's session flush interact with `@Transactional`?**

<details><summary>Click to reveal answer</summary>

Hibernate's persistence context (first-level cache) accumulates changes and flushes them to the database at certain points.

```java
@Service
public class UserService {

    @Autowired private EntityManager em;
    @Autowired private UserRepository userRepo;

    @Transactional
    public void demonstrateFlush() {
        // AUTO flush mode (default):
        // Flush occurs before queries to ensure consistency
        // Flush also occurs before commit

        User user = new User("john@example.com");
        em.persist(user);
        // SQL INSERT not yet sent to DB

        // Hibernate flushes automatically BEFORE this query
        // to ensure the query sees the just-persisted user
        List<User> users = em.createQuery("FROM User", User.class).getResultList();
        // At this point, INSERT was sent (AUTO flush)

        // Transaction commits → final flush + commit
    }

    @Transactional
    public void optimizedBulkInsert(List<CreateUserRequest> requests) {
        em.setFlushMode(FlushModeType.COMMIT); // only flush on commit

        for (int i = 0; i < requests.size(); i++) {
            User user = new User(requests.get(i));
            em.persist(user);

            if (i % 50 == 0) {
                em.flush();   // explicit flush every 50
                em.clear();   // clear 1st level cache to prevent OOM
            }
        }
        // Final flush on commit
    }

    @Transactional
    public void showDirtyChecking() {
        User user = userRepo.findById(1L).orElseThrow();
        user.setName("New Name"); // entity is MANAGED — change is tracked

        // NO explicit save needed!
        // Hibernate detects the change (dirty checking) at flush time
        // → generates UPDATE SQL automatically
    }
}
```

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| Default rollback | Unchecked (`RuntimeException`) — not checked exceptions |
| `readOnly = true` | Disables dirty checking; use for all read operations |
| `REQUIRED` (default) | Join existing tx; create new if none |
| `REQUIRES_NEW` | Always new, independent tx; suspend existing |
| `NESTED` | Savepoint within existing tx |
| `MANDATORY` | Must have existing tx; throw if none |
| Self-invocation | Bypasses proxy → `@Transactional` ignored; extract to separate class |
| Private methods | `@Transactional` has no effect — CGLIB can't proxy private methods |
| Distributed tx | Saga + Outbox pattern preferred over XA/2PC |
| `@TransactionalEventListener` | Default `AFTER_COMMIT` — event fires only if tx commits |
