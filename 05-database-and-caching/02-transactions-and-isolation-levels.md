# Chapter 2: Transactions & Isolation Levels

> **Estimated study time:** 2 days | **Priority:** 🔴 High

---

## Key Concepts

- ACID properties
- Transaction isolation levels: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE
- Read anomalies: dirty read, non-repeatable read, phantom read
- Optimistic vs pessimistic locking
- MVCC (Multi-Version Concurrency Control)
- Deadlocks and how to avoid them
- Two-phase locking (2PL)

---

## Questions & Answers

---

### Q1 — 🟢 What are the ACID properties?

<details><summary>Click to reveal answer</summary>

**ACID** is a set of guarantees that a database transaction must provide:

| Property | Meaning |
|----------|---------|
| **Atomicity** | All operations in a transaction succeed, or none do. No partial updates. |
| **Consistency** | A transaction brings the database from one valid state to another; all constraints are satisfied. |
| **Isolation** | Concurrent transactions execute as if they were sequential. One transaction's intermediate state is not visible to others (to some degree, depending on isolation level). |
| **Durability** | Once a transaction is committed, it persists even if the system crashes. Data is written to non-volatile storage. |

**Example:**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- debit
UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- credit
COMMIT;
-- Atomicity: if the credit fails, the debit is rolled back
-- Consistency: total balance in the system remains the same
-- Isolation: another transaction reading account 1 won't see the intermediate -100
-- Durability: after COMMIT, data survives a crash
```

</details>

---

### Q2 — 🟢 What are the four standard isolation levels in SQL?

<details><summary>Click to reveal answer</summary>

From weakest to strongest:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Default in |
|-------|-----------|---------------------|-------------|-----------|
| READ UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible | (rare) |
| READ COMMITTED | ❌ Prevented | ✅ Possible | ✅ Possible | PostgreSQL, Oracle |
| REPEATABLE READ | ❌ Prevented | ❌ Prevented | ✅ Possible | MySQL InnoDB |
| SERIALIZABLE | ❌ Prevented | ❌ Prevented | ❌ Prevented | (explicit) |

```sql
-- Set isolation level for the current transaction
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
-- ... operations ...
COMMIT;
```

Higher isolation levels prevent more anomalies but reduce concurrency (more locking or conflict detection).

</details>

---

### Q3 — 🟡 Explain the three read anomalies: dirty read, non-repeatable read, and phantom read.

<details><summary>Click to reveal answer</summary>

**Dirty Read** — reading uncommitted data from another transaction:
```
T1: UPDATE balance = 500 (not yet committed)
T2: SELECT balance → sees 500  ← dirty read
T1: ROLLBACK
T2: operated on data that never existed
```
Prevented by: READ COMMITTED and above.

**Non-Repeatable Read** — reading the same row twice in the same transaction returns different values:
```
T1: SELECT balance → 1000
T2: UPDATE balance = 500; COMMIT
T1: SELECT balance → 500  ← different!
```
Prevented by: REPEATABLE READ and above.

**Phantom Read** — a range query returns different rows across two reads in the same transaction:
```
T1: SELECT * FROM orders WHERE amount > 100 → returns 5 rows
T2: INSERT INTO orders (amount=200); COMMIT
T1: SELECT * FROM orders WHERE amount > 100 → returns 6 rows  ← phantom!
```
Prevented by: SERIALIZABLE.

Note: PostgreSQL's REPEATABLE READ also prevents phantom reads due to MVCC snapshot semantics.

</details>

---

### Q4 — 🟡 How does MVCC (Multi-Version Concurrency Control) work in PostgreSQL?

<details><summary>Click to reveal answer</summary>

PostgreSQL uses MVCC to allow readers and writers to not block each other. Instead of locking rows, each row has hidden system columns:

- `xmin` — transaction ID that created this row version
- `xmax` — transaction ID that deleted/updated this row version (0 if current)

When a transaction reads, it sees row versions whose `xmin` was committed before the transaction started and whose `xmax` is 0 or committed after the transaction started.

```
T1 starts (txid=100): sees rows committed before txid=100
T2 (txid=101): UPDATE user SET name='Bob'
  → marks old row: xmax=101
  → inserts new row: xmin=101, name='Bob'
T2 COMMIT
T1 (still running): SELECT name → still sees 'Alice' (xmin<100, xmax=101 committed AFTER T1 start)
```

**Benefits:**
- Readers never block writers, writers never block readers
- Readers see a consistent snapshot of the database

**Downside:**
- Dead row versions accumulate → `VACUUM` must reclaim them
- Table bloat if VACUUM can't keep up

</details>

---

### Q5 — 🟡 What is the difference between optimistic and pessimistic locking?

<details><summary>Click to reveal answer</summary>

**Pessimistic Locking** — assumes conflicts are likely; locks the row when reading to prevent concurrent modification.

```sql
-- PostgreSQL: lock the row until transaction ends
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;
-- Now update safely
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 1;
COMMIT;
```

In JPA:
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT i FROM Inventory i WHERE i.productId = :id")
Optional<Inventory> findByIdForUpdate(@Param("id") Long id);
```

**Optimistic Locking** — assumes conflicts are rare; doesn't lock, but validates at commit time using a version number.

```java
@Entity
public class Inventory {
    @Id
    private Long productId;
    private int quantity;

    @Version  // JPA manages this column automatically
    private Long version;
}
```

```sql
-- JPA translates to:
UPDATE inventory
SET quantity = 9, version = 6
WHERE product_id = 1 AND version = 5;
-- If 0 rows updated → someone else changed it → OptimisticLockException
```

| Aspect | Pessimistic | Optimistic |
|--------|------------|-----------|
| Conflict assumption | Frequent | Rare |
| Performance | Lower (blocking) | Higher (no blocking) |
| Failure mode | Deadlock risk | `OptimisticLockException` on commit |
| Best for | High-contention (inventory, seats) | Low-contention (user profiles) |

</details>

---

### Q6 — 🟡 How does Spring Boot configure transaction isolation levels?

<details><summary>Click to reveal answer</summary>

```java
@Service
public class OrderService {

    // Default isolation (READ_COMMITTED on PostgreSQL)
    @Transactional
    public Order createOrder(CreateOrderRequest req) { ... }

    // Use REPEATABLE_READ to prevent non-repeatable reads
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void processPayment(Long orderId) { ... }

    // Use SERIALIZABLE for critical financial operations
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) { ... }

    // Read-only hint — allows DB to optimize (no undo log, no dirty check)
    @Transactional(readOnly = true)
    public List<Order> getOrders(Long userId) { ... }
}
```

**Spring isolation constants** map to JDBC `Connection` constants:

| Spring Isolation | SQL Isolation |
|-----------------|--------------|
| `DEFAULT` | Database default |
| `READ_UNCOMMITTED` | READ UNCOMMITTED |
| `READ_COMMITTED` | READ COMMITTED |
| `REPEATABLE_READ` | REPEATABLE READ |
| `SERIALIZABLE` | SERIALIZABLE |

**Important:** isolation applies only to the database transaction. For distributed transactions across microservices, you need the Saga pattern or 2PC.

</details>

---

### Q7 — 🔴 What is a deadlock, how does it occur in databases, and how do you prevent it?

<details><summary>Click to reveal answer</summary>

A **deadlock** occurs when two (or more) transactions each hold a lock that the other needs:

```
T1: LOCK row A → waits for row B
T2: LOCK row B → waits for row A
→ Deadlock: neither can proceed
```

**PostgreSQL** detects deadlocks automatically and rolls back one transaction with:
```
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 678
```

**Prevention strategies:**

1. **Consistent lock ordering** — always acquire locks in the same order:
```java
// WRONG: T1 locks account 1 then 2; T2 locks account 2 then 1
void transfer(Long fromId, Long toId) {
    lock(fromId); lock(toId);   // inconsistent!
}

// RIGHT: always lock lower ID first
void transfer(Long fromId, Long toId) {
    Long first  = Math.min(fromId, toId);
    Long second = Math.max(fromId, toId);
    lock(first); lock(second);   // consistent order
}
```

2. **Short transactions** — minimize time locks are held; don't do network calls inside a transaction.

3. **`SELECT FOR UPDATE SKIP LOCKED`** — skip rows locked by others instead of waiting:
```sql
SELECT * FROM job_queue WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

4. **Retry on deadlock** — catch `CannotAcquireLockException` and retry:
```java
@Retryable(
    retryFor = CannotAcquireLockException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 50, multiplier = 2)
)
@Transactional
public void process() { ... }
```

5. **Use optimistic locking** where contention is low — no lock held, so no deadlock possible.

</details>

---

### Q8 — 🔴 Explain the SERIALIZABLE isolation level and serialization anomalies.

<details><summary>Click to reveal answer</summary>

SERIALIZABLE guarantees that the result of concurrent transactions is equivalent to some serial (one-at-a-time) execution order. PostgreSQL uses **SSI (Serializable Snapshot Isolation)** which detects serialization anomalies without heavy locking.

**Write Skew anomaly** (not prevented by REPEATABLE READ):
```
Business rule: at least one doctor must be on-call

T1: SELECT COUNT(*) FROM on_call WHERE shift='night' → 2
T1: DELETE FROM on_call WHERE doctor='Alice' AND shift='night'

T2: SELECT COUNT(*) FROM on_call WHERE shift='night' → 2
T2: DELETE FROM on_call WHERE doctor='Bob' AND shift='night'

Both COMMIT → 0 doctors on-call! Rule violated.
```
Only SERIALIZABLE (SSI) detects and prevents this by aborting one transaction.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- Reads and writes that form a dependency cycle
-- PostgreSQL tracks read/write sets and aborts if serialization would be violated
COMMIT;
-- May get: ERROR: could not serialize access due to read/write dependencies
```

**Performance note:** SSI adds overhead (tracking anti-dependencies) but is significantly better than 2PL (Two-Phase Locking) which causes more blocking. For high-throughput systems, use REPEATABLE READ and handle write skew with explicit locks or application-level constraints.

</details>
