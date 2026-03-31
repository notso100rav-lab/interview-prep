# JDBC (Java Database Connectivity)

> 💡 **Interviewer Tip:** JDBC questions test your understanding of Java's database layer — the foundation beneath every ORM. Interviewers expect knowledge of `PreparedStatement` (and why it prevents SQL injection), connection pooling, transaction management, and batch processing for performance.

---

## Table of Contents
1. [JDBC Architecture](#jdbc-architecture)
2. [Connection Management](#connection-management)
3. [Statements](#statements)
4. [ResultSet](#resultset)
5. [Transactions](#transactions)
6. [Connection Pooling](#connection-pooling)
7. [Batch Processing](#batch-processing)
8. [SQL Injection Prevention](#sql-injection-prevention)
9. [Theory Questions](#theory-questions)
10. [Scenario-Based Questions](#scenario-based-questions)

---

## Quick Reference

| Component | Interface | Purpose |
|---|---|---|
| Connection | `java.sql.Connection` | Database connection |
| Statement | `java.sql.Statement` | Simple SQL execution |
| PreparedStatement | `java.sql.PreparedStatement` | Parameterized queries |
| CallableStatement | `java.sql.CallableStatement` | Stored procedures |
| ResultSet | `java.sql.ResultSet` | Query results |
| DataSource | `javax.sql.DataSource` | Connection factory (pooling) |

---

## Theory Questions

### JDBC Architecture

**Q1.** 🟢 What is JDBC and what is its architecture?

<details><summary>Click to reveal answer</summary>

JDBC (Java Database Connectivity) is a standard Java API that enables Java applications to interact with relational databases using SQL. It provides a vendor-neutral abstraction layer.

**Architecture layers:**

```
Java Application
      ↓
JDBC API (java.sql.*, javax.sql.*)   ← Standard interfaces
      ↓
JDBC Driver Manager                  ← Selects appropriate driver
      ↓
JDBC Driver (vendor-specific)        ← Implements JDBC interfaces
      ↓
Database (MySQL, PostgreSQL, Oracle, etc.)
```

**Driver types:**

| Type | Name | Description |
|---|---|---|
| Type 1 | JDBC-ODBC Bridge | Translates to ODBC (obsolete) |
| Type 2 | Native API | Uses native client libraries |
| Type 3 | Network Protocol | Middleware server |
| Type 4 | Thin Driver | Pure Java, direct DB protocol |

Most modern drivers are **Type 4** (e.g., MySQL Connector/J, PostgreSQL JDBC Driver).

```java
// Basic JDBC example
try (Connection conn = DriverManager.getConnection(
        "jdbc:postgresql://localhost:5432/mydb", "user", "password");
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {

    while (rs.next()) {
        System.out.println(rs.getLong("id") + ": " + rs.getString("name"));
    }
}
```

</details>

---

**Q2.** 🟢 How do you establish a JDBC connection?

<details><summary>Click to reveal answer</summary>

```java
// Method 1: DriverManager (simple, no pooling)
String url = "jdbc:mysql://localhost:3306/mydb?useSSL=true&serverTimezone=UTC";
try (Connection conn = DriverManager.getConnection(url, "username", "password")) {
    // use connection
}

// Method 2: DataSource (preferred, supports pooling)
// HikariCP example
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("user");
config.setPassword("password");
config.setMaximumPoolSize(20);

DataSource dataSource = new HikariDataSource(config);
try (Connection conn = dataSource.getConnection()) {
    // use connection
}

// Common JDBC URLs
// MySQL:      jdbc:mysql://host:3306/database
// PostgreSQL: jdbc:postgresql://host:5432/database
// H2:         jdbc:h2:mem:testdb  (in-memory)
// Oracle:     jdbc:oracle:thin:@host:1521:SID
// SQL Server: jdbc:sqlserver://host:1433;databaseName=db
```

</details>

---

**Q3.** 🟡 What is `DriverManager` vs `DataSource`?

<details><summary>Click to reveal answer</summary>

| Feature | `DriverManager` | `DataSource` |
|---|---|---|
| Connection pooling | No | Yes (via pool implementations) |
| JNDI support | No | Yes |
| Configuration | Hardcoded URL/user/pass | Configurable externally |
| Thread safety | Thread-safe | Thread-safe |
| Recommended for | Testing, simple scripts | Production |

```java
// DriverManager - creates new connection every call (expensive!)
Connection conn1 = DriverManager.getConnection(url, user, password); // new connection
Connection conn2 = DriverManager.getConnection(url, user, password); // another new connection

// DataSource - returns pooled connection (fast!)
Connection conn3 = dataSource.getConnection(); // from pool
conn3.close(); // returns to pool, not actually closed
Connection conn4 = dataSource.getConnection(); // same physical connection reused
```

✅ **Best Practice:** Always use `DataSource` (with HikariCP) in production applications. `DriverManager` creates a new TCP connection to the database on every call — very expensive (20–100ms per connection).

</details>

---

### Statements

**Q4.** 🟢 What is the difference between `Statement`, `PreparedStatement`, and `CallableStatement`?

<details><summary>Click to reveal answer</summary>

| Type | Use Case | Parameters | Pre-compiled |
|---|---|---|---|
| `Statement` | Simple, one-time SQL | No | No |
| `PreparedStatement` | Repeated/parameterized SQL | Yes (?) | Yes |
| `CallableStatement` | Stored procedures | IN/OUT/INOUT | Yes |

```java
// Statement - simple but vulnerable to SQL injection!
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM users WHERE name = '" + name + "'");

// PreparedStatement - safe and efficient
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, name); // safely bound parameter
ResultSet rs2 = ps.executeQuery();

// CallableStatement - stored procedure
CallableStatement cs = conn.prepareCall("{call get_user(?, ?)}");
cs.setLong(1, userId);          // IN parameter
cs.registerOutParameter(2, Types.VARCHAR); // OUT parameter
cs.execute();
String result = cs.getString(2);
```

</details>

---

**Q5.** 🟡 How does `PreparedStatement` prevent SQL injection?

<details><summary>Click to reveal answer</summary>

`PreparedStatement` uses **parameterized queries**: the SQL structure is sent to the database first (pre-compiled), and parameters are sent separately. The database treats parameters as **data, never as SQL code**.

```java
// SQL INJECTION VULNERABILITY - NEVER DO THIS
String username = "' OR '1'='1"; // malicious input
Statement stmt = conn.createStatement();
// Executes: SELECT * FROM users WHERE username = '' OR '1'='1'
// Returns ALL users!
ResultSet rs = stmt.executeQuery(
    "SELECT * FROM users WHERE username = '" + username + "'");

// SAFE - PreparedStatement
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ?");
ps.setString(1, username); // treated as literal data, not SQL
// Database receives: SELECT * FROM users WHERE username = ?' OR '1'='1''
// (the quotes are escaped - no injection possible)
ResultSet rs2 = ps.executeQuery();

// More examples
ps = conn.prepareStatement("INSERT INTO users (name, email, age) VALUES (?, ?, ?)");
ps.setString(1, "Alice");
ps.setString(2, "alice@example.com");
ps.setInt(3, 30);
ps.executeUpdate();
```

**Why it works:** The database parses the SQL structure (with `?` placeholders) once. When parameters are bound, they are transmitted in a separate protocol message and never interpreted as SQL syntax.

</details>

---

**Q6.** 🟡 What are all the `PreparedStatement` setter methods?

<details><summary>Click to reveal answer</summary>

```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO demo VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");

ps.setBoolean(1, true);
ps.setByte(2, (byte) 42);
ps.setShort(3, (short) 100);
ps.setInt(4, 12345);
ps.setLong(5, 1234567890L);
ps.setFloat(6, 3.14f);
ps.setDouble(7, 2.71828);
ps.setBigDecimal(8, new BigDecimal("99.99"));
ps.setString(9, "text value");
ps.setDate(10, java.sql.Date.valueOf(LocalDate.now()));
ps.setTimestamp(11, java.sql.Timestamp.valueOf(LocalDateTime.now()));

// Binary data
ps.setBytes(1, byteArray);
ps.setBinaryStream(1, inputStream, length);

// NULL values
ps.setNull(1, Types.VARCHAR);

// Object (JDBC driver determines type)
ps.setObject(1, someValue);
ps.setObject(1, someValue, Types.VARCHAR); // with explicit type
```

</details>

---

**Q7.** 🟡 What is `CallableStatement` and how do you use OUT parameters?

<details><summary>Click to reveal answer</summary>

`CallableStatement` is used to call stored procedures and functions.

```sql
-- Stored procedure in PostgreSQL
CREATE OR REPLACE FUNCTION get_user_count(dept_id INTEGER, OUT user_count INTEGER)
AS $$ BEGIN
    SELECT COUNT(*) INTO user_count FROM users WHERE department_id = dept_id;
END; $$ LANGUAGE plpgsql;
```

```java
// Calling with OUT parameter
try (CallableStatement cs = conn.prepareCall("{call get_user_count(?, ?)}")) {
    cs.setInt(1, 5);                        // IN: department_id
    cs.registerOutParameter(2, Types.INTEGER); // OUT: user_count

    cs.execute();
    int count = cs.getInt(2); // retrieve OUT parameter
    System.out.println("Users in dept: " + count);
}

// Function that returns a value
try (CallableStatement cs = conn.prepareCall("{? = call my_function(?)}")) {
    cs.registerOutParameter(1, Types.VARCHAR); // return value
    cs.setInt(2, 42);                          // function parameter
    cs.execute();
    String result = cs.getString(1);
}

// Named parameters (if driver supports it)
try (CallableStatement cs = conn.prepareCall("{call update_salary(:employee_id, :amount)}")) {
    cs.setLong("employee_id", 42L);
    cs.setBigDecimal("amount", new BigDecimal("5000.00"));
    cs.execute();
}
```

</details>

---

### ResultSet

**Q8.** 🟡 What are the different types of `ResultSet`?

<details><summary>Click to reveal answer</summary>

When creating a `Statement` or `PreparedStatement`, you can specify `ResultSet` type and concurrency:

```java
// Type + Concurrency
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,  // scrollable, doesn't see external changes
    ResultSet.CONCUR_UPDATABLE          // can update rows
);
```

**ResultSet Types:**

| Type | Description |
|---|---|
| `TYPE_FORWARD_ONLY` | Default; cursor moves forward only |
| `TYPE_SCROLL_INSENSITIVE` | Scrollable; doesn't reflect external changes |
| `TYPE_SCROLL_SENSITIVE` | Scrollable; reflects external changes |

**ResultSet Concurrency:**

| Concurrency | Description |
|---|---|
| `CONCUR_READ_ONLY` | Default; cannot update |
| `CONCUR_UPDATABLE` | Can update rows directly |

```java
// Scrollable ResultSet
try (Statement stmt = conn.createStatement(
        ResultSet.TYPE_SCROLL_INSENSITIVE, ResultSet.CONCUR_READ_ONLY);
     ResultSet rs = stmt.executeQuery("SELECT * FROM users ORDER BY id")) {

    rs.last();  // move to last row
    int rowCount = rs.getRow(); // get total row count

    rs.first(); // back to first
    rs.absolute(5); // move to 5th row
    rs.relative(-2); // move back 2 rows
    rs.previous(); // move backward
}

// Updatable ResultSet
try (ResultSet rs = stmt.executeQuery("SELECT id, salary FROM employees")) {
    while (rs.next()) {
        if (rs.getDouble("salary") < 50000) {
            rs.updateDouble("salary", rs.getDouble("salary") * 1.1); // 10% raise
            rs.updateRow(); // commit the row update
        }
    }
}
```

</details>

---

**Q9.** 🟡 How do you read data from a `ResultSet`?

<details><summary>Click to reveal answer</summary>

```java
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT id, name, email, age, salary, hire_date, active FROM users WHERE dept = ?")) {

    ps.setString(1, "Engineering");
    try (ResultSet rs = ps.executeQuery()) {

        // ResultSetMetaData - inspect column info
        ResultSetMetaData meta = rs.getMetaData();
        int colCount = meta.getColumnCount();
        for (int i = 1; i <= colCount; i++) {
            System.out.println(meta.getColumnName(i) + " (" + meta.getColumnTypeName(i) + ")");
        }

        while (rs.next()) {
            // By column name (preferred - clearer, not affected by query column order)
            long id = rs.getLong("id");
            String name = rs.getString("name");
            String email = rs.getString("email");
            int age = rs.getInt("age");
            double salary = rs.getDouble("salary");
            LocalDate hireDate = rs.getDate("hire_date").toLocalDate();
            boolean active = rs.getBoolean("active");

            // Null checking
            Integer nullableAge = rs.getInt("age");
            if (rs.wasNull()) nullableAge = null; // getInt returns 0 for NULL

            // By column index (1-based, faster but fragile)
            long id2 = rs.getLong(1);
        }
    }
}
```

</details>

---

**Q10.** 🟢 What does `executeQuery()`, `executeUpdate()`, and `execute()` return?

<details><summary>Click to reveal answer</summary>

| Method | Returns | Use For |
|---|---|---|
| `executeQuery()` | `ResultSet` | SELECT queries |
| `executeUpdate()` | `int` (rows affected) | INSERT, UPDATE, DELETE, DDL |
| `execute()` | `boolean` | Any SQL (true if ResultSet returned) |

```java
// executeQuery - SELECT
try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { ... }
}

// executeUpdate - DML/DDL
int rowsAffected = ps.executeUpdate();
System.out.println("Updated " + rowsAffected + " rows");

// executeUpdate with generated keys
PreparedStatement ps2 = conn.prepareStatement(
    "INSERT INTO users (name) VALUES (?)",
    Statement.RETURN_GENERATED_KEYS);
ps2.setString(1, "Alice");
ps2.executeUpdate();

try (ResultSet generatedKeys = ps2.getGeneratedKeys()) {
    if (generatedKeys.next()) {
        long newId = generatedKeys.getLong(1);
        System.out.println("New user ID: " + newId);
    }
}

// execute - dynamic SQL
boolean hasResultSet = stmt.execute(anySql);
if (hasResultSet) {
    ResultSet rs = stmt.getResultSet();
} else {
    int updateCount = stmt.getUpdateCount();
}
```

</details>

---

### Transactions

**Q11.** 🟡 How do you manage transactions in JDBC?

<details><summary>Click to reveal answer</summary>

By default, JDBC runs in **auto-commit mode** (each statement is its own transaction). For multi-statement transactions, disable auto-commit.

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false); // start manual transaction management

try {
    // Multiple operations in one transaction
    PreparedStatement debit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?");
    debit.setBigDecimal(1, amount);
    debit.setLong(2, fromAccountId);
    debit.executeUpdate();

    PreparedStatement credit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?");
    credit.setBigDecimal(1, amount);
    credit.setLong(2, toAccountId);
    credit.executeUpdate();

    conn.commit(); // all or nothing

} catch (SQLException e) {
    conn.rollback(); // undo all changes if anything fails
    throw e;
} finally {
    conn.setAutoCommit(true); // restore for connection pool reuse
    conn.close();             // return to pool
}
```

🚨 **Common Mistake:** Forgetting to reset `autoCommit` before returning connection to pool — the next user gets a connection with `autoCommit=false`, causing unexpected behavior.

</details>

---

**Q12.** 🟡 What are transaction isolation levels in JDBC?

<details><summary>Click to reveal answer</summary>

Transaction isolation levels control how concurrent transactions interact. Higher isolation = more consistency, less concurrency.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE_READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

**Anomaly definitions:**
- **Dirty Read:** Read uncommitted data from another transaction
- **Non-Repeatable Read:** Same query returns different rows within a transaction
- **Phantom Read:** Same range query returns different number of rows

```java
// Setting isolation level
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED); // common default

// Check supported levels
DatabaseMetaData meta = conn.getMetaData();
boolean supported = meta.supportsTransactionIsolationLevel(
    Connection.TRANSACTION_SERIALIZABLE);

// Check current level
int level = conn.getTransactionIsolation();
```

✅ **Best Practice:** Most applications use `READ_COMMITTED` (PostgreSQL default) or `REPEATABLE_READ` (MySQL InnoDB default). `SERIALIZABLE` for financial transactions where consistency is paramount.

</details>

---

**Q13.** 🔴 What is a `Savepoint` in JDBC?

<details><summary>Click to reveal answer</summary>

A `Savepoint` marks a point within a transaction to which you can roll back partially without rolling back the entire transaction.

```java
conn.setAutoCommit(false);
Savepoint sp1 = null;

try {
    // Step 1: Create order
    insertOrder(conn, order);
    sp1 = conn.setSavepoint("after_order_insert");

    // Step 2: Try to reserve inventory
    for (OrderItem item : order.getItems()) {
        boolean reserved = reserveInventory(conn, item);
        if (!reserved) {
            // Can't rollback just one item easily — rollback to savepoint
            conn.rollback(sp1);
            // Handle: mark order as pending, notify user
            markOrderPending(conn, order);
            conn.commit();
            return;
        }
    }

    // Step 3: Process payment
    processPayment(conn, order);

    conn.commit(); // everything succeeded

} catch (SQLException e) {
    conn.rollback(); // roll back everything
    throw e;
} finally {
    if (sp1 != null) {
        try { conn.releaseSavepoint(sp1); } catch (SQLException ignored) { }
    }
}
```

</details>

---

### Connection Pooling

**Q14.** 🟡 What is connection pooling and why is it necessary?

<details><summary>Click to reveal answer</summary>

Creating a JDBC connection is expensive (TCP handshake, authentication, session setup: 20–100ms). Connection pooling maintains a **pool of reusable connections** that are handed out and returned rather than created/destroyed.

```
Without Pool:
Request → Create Connection (50ms) → Execute Query (5ms) → Close Connection → 55ms total

With Pool:
Request → Borrow Connection (1ms) → Execute Query (5ms) → Return to Pool → 6ms total
```

**Pool lifecycle:**
1. Pool initializes with `minimumIdle` connections
2. `getConnection()` borrows a connection (blocks if pool exhausted)
3. `connection.close()` returns connection to pool (not actually closed)
4. Pool monitors idle connections, creates/destroys as needed

```java
// HikariCP - fastest connection pool for JDBC
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("user");
config.setPassword("password");

// Pool settings
config.setMaximumPoolSize(20);        // max connections
config.setMinimumIdle(5);            // min idle connections
config.setIdleTimeout(600_000);       // 10min idle before removal
config.setConnectionTimeout(30_000);  // 30s wait for connection
config.setMaxLifetime(1_800_000);     // 30min max connection lifetime

// Connection validation
config.setConnectionTestQuery("SELECT 1"); // or use isValid()
config.setKeepaliveTime(60_000);          // keep-alive ping

DataSource ds = new HikariDataSource(config);
```

</details>

---

**Q15.** 🟡 How does HikariCP differ from other connection pools?

<details><summary>Click to reveal answer</summary>

| Feature | HikariCP | Apache DBCP2 | c3p0 |
|---|---|---|---|
| Performance | **Fastest** | Moderate | Moderate |
| Memory footprint | Small | Medium | Medium |
| Bytes allocated | Very low | Higher | Higher |
| Configuration | Simple | Complex | Complex |
| Active development | Yes | Yes | Slow |
| Spring Boot default | **Yes** | No | No |

**HikariCP's performance secrets:**
1. **ConcurrentBag** — lock-free connection borrowing
2. **Bytecode optimization** — uses proxy stubs generated at startup
3. **Minimal object creation** — reduces GC pressure
4. **FastList** — custom `ArrayList` without range-check overhead

```java
// Spring Boot auto-configuration (application.properties)
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=user
spring.datasource.password=password
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.max-lifetime=1800000
```

</details>

---

**Q16.** 🔴 What is connection pool starvation and how do you prevent it?

<details><summary>Click to reveal answer</summary>

Pool starvation occurs when all connections are in use and requests block waiting — typically causing timeouts and cascading failures.

**Common causes:**
1. `maximumPoolSize` too small for load
2. Slow queries holding connections too long
3. Connections not being closed (resource leaks)
4. Nested transactions (Spring anti-pattern)

```java
// WRONG: Connection leak — close() never called if exception
Connection conn = ds.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT ..."); // if this throws, conn leaks!
conn.close();

// CORRECT: try-with-resources
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT ...");
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { ... }
} // all closed automatically

// WRONG: Spring anti-pattern - nested @Transactional REQUIRED
@Transactional
public void outer() {
    inner(); // calls another @Transactional REQUIRED method
    // Both use the SAME connection - not pool starvation but can cause deadlocks
}

// Detecting pool starvation
config.setConnectionTimeout(5000); // throw exception after 5s wait
// Monitor with HikariCP metrics:
HikariDataSource hds = (HikariDataSource) ds;
HikariPoolMXBean pool = hds.getHikariPoolMXBean();
int waiting = pool.getThreadsAwaitingConnection();
```

✅ **Best Practice:** `maximumPoolSize = (core_count * 2) + effective_spindle_count` — HikariCP's recommendation.

</details>

---

### Batch Processing

**Q17.** 🟡 How does batch processing work in JDBC?

<details><summary>Click to reveal answer</summary>

Batch processing groups multiple SQL operations and sends them to the database in a single network round trip, drastically improving performance for bulk operations.

```java
// Without batching: N round trips for N inserts
for (User user : users) {
    ps.setString(1, user.getName());
    ps.setString(2, user.getEmail());
    ps.executeUpdate(); // 1 network round trip each!
}

// With batching: ~1 round trip for N inserts
String sql = "INSERT INTO users (name, email, age) VALUES (?, ?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    conn.setAutoCommit(false);

    int batchSize = 1000;
    int count = 0;

    for (User user : users) {
        ps.setString(1, user.getName());
        ps.setString(2, user.getEmail());
        ps.setInt(3, user.getAge());
        ps.addBatch(); // add to batch

        if (++count % batchSize == 0) {
            int[] results = ps.executeBatch(); // send batch
            conn.commit();                      // commit batch
        }
    }

    // Process remaining
    int[] results = ps.executeBatch();
    conn.commit();

    // Check results
    for (int r : results) {
        if (r == Statement.EXECUTE_FAILED) {
            System.err.println("A batch item failed");
        }
    }
}
```

**Performance comparison (inserting 100,000 rows):**
| Method | Time |
|---|---|
| No batch, auto-commit | 45 seconds |
| No batch, manual commit | 8 seconds |
| Batch of 1000 + manual commit | 0.8 seconds |

</details>

---

**Q18.** 🟡 What are the return values of `executeBatch()`?

<details><summary>Click to reveal answer</summary>

`executeBatch()` returns `int[]` where each element corresponds to one batched statement:

| Value | Meaning |
|---|---|
| `≥ 0` | Success; number of rows affected |
| `Statement.SUCCESS_NO_INFO (-2)` | Success but row count unknown |
| `Statement.EXECUTE_FAILED (-3)` | Statement failed |

```java
int[] results = ps.executeBatch();

int totalAffected = 0;
int failures = 0;
for (int i = 0; i < results.length; i++) {
    if (results[i] == Statement.EXECUTE_FAILED) {
        failures++;
        System.err.println("Batch item " + i + " failed");
    } else if (results[i] >= 0) {
        totalAffected += results[i];
    } else { // SUCCESS_NO_INFO
        totalAffected++; // assume 1
    }
}

// If batch fails: BatchUpdateException
try {
    ps.executeBatch();
} catch (BatchUpdateException bue) {
    int[] updateCounts = bue.getUpdateCounts();
    // Counts for statements that ran before the failure
    // Behavior depends on driver: some continue, some stop on first failure
}
```

</details>

---

### SQL Injection Prevention

**Q19.** 🔴 Demonstrate SQL injection and how to prevent it.

<details><summary>Click to reveal answer</summary>

```java
// VULNERABLE CODE - String concatenation
public User findUser(String username, String password) throws SQLException {
    String sql = "SELECT * FROM users WHERE username='" + username +
                 "' AND password='" + password + "'";
    // Attack: username = "admin'--"
    // Resulting SQL: SELECT * FROM users WHERE username='admin'--' AND password='anything'
    // The '--' comments out the password check!
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery(sql);
    return rs.next() ? mapUser(rs) : null;
}

// More attacks:
// username = "' OR '1'='1" → returns all users
// username = "'; DROP TABLE users;--" → drops table (if driver allows multiple statements)
// username = "' UNION SELECT username, password FROM admin_users--" → data exfiltration

// SECURE CODE - PreparedStatement
public User findUserSecure(String username, String password) throws SQLException {
    String sql = "SELECT * FROM users WHERE username = ? AND password_hash = ?";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, username);
        ps.setString(2, hashPassword(password)); // hash before comparing
        try (ResultSet rs = ps.executeQuery()) {
            return rs.next() ? mapUser(rs) : null;
        }
    }
}

// ALSO VULNERABLE: dynamic column/table names (PreparedStatement can't help here)
// Use allowlist approach for dynamic identifiers
public ResultSet sortBy(String columnName, String direction) throws SQLException {
    // WRONG:
    // return stmt.executeQuery("SELECT * FROM users ORDER BY " + columnName);

    // CORRECT: allowlist validation
    Set<String> ALLOWED_COLUMNS = Set.of("id", "name", "email", "created_at");
    Set<String> ALLOWED_DIRECTIONS = Set.of("ASC", "DESC");

    if (!ALLOWED_COLUMNS.contains(columnName) || !ALLOWED_DIRECTIONS.contains(direction)) {
        throw new IllegalArgumentException("Invalid sort parameters");
    }

    return conn.createStatement().executeQuery(
        "SELECT * FROM users ORDER BY " + columnName + " " + direction);
}
```

</details>

---

**Q20.** 🟡 What is `DatabaseMetaData` and what can you learn from it?

<details><summary>Click to reveal answer</summary>

`DatabaseMetaData` provides comprehensive information about the database's capabilities and structure.

```java
DatabaseMetaData meta = conn.getMetaData();

// Database info
System.out.println("DB: " + meta.getDatabaseProductName()); // PostgreSQL
System.out.println("Version: " + meta.getDatabaseProductVersion());
System.out.println("Driver: " + meta.getDriverName());

// Feature support
System.out.println("Transactions: " + meta.supportsTransactions());
System.out.println("Batch updates: " + meta.supportsBatchUpdates());
System.out.println("Stored procedures: " + meta.supportsStoredProcedures());

// Schema exploration
try (ResultSet tables = meta.getTables("mydb", "public", "%", new String[]{"TABLE"})) {
    while (tables.next()) {
        System.out.println("Table: " + tables.getString("TABLE_NAME"));
    }
}

try (ResultSet columns = meta.getColumns("mydb", "public", "users", "%")) {
    while (columns.next()) {
        System.out.println(columns.getString("COLUMN_NAME") + " " +
                           columns.getString("TYPE_NAME") + " " +
                           (columns.getInt("NULLABLE") == 0 ? "NOT NULL" : "NULLABLE"));
    }
}

try (ResultSet pks = meta.getPrimaryKeys("mydb", "public", "users")) {
    while (pks.next()) {
        System.out.println("PK: " + pks.getString("COLUMN_NAME"));
    }
}
```

</details>

---

**Q21.** 🟡 How do you work with stored procedures that return result sets?

<details><summary>Click to reveal answer</summary>

```sql
-- PostgreSQL stored procedure returning a refcursor
CREATE OR REPLACE FUNCTION get_users_by_dept(dept_id INT)
RETURNS REFCURSOR AS $$ DECLARE ref REFCURSOR;
BEGIN
    OPEN ref FOR SELECT * FROM users WHERE department_id = dept_id;
    RETURN ref;
END; $$ LANGUAGE plpgsql;
```

```java
conn.setAutoCommit(false); // needed for cursors

try (CallableStatement cs = conn.prepareCall("{? = call get_users_by_dept(?)}")) {
    cs.registerOutParameter(1, Types.OTHER); // REFCURSOR
    cs.setInt(2, 42);
    cs.execute();

    try (ResultSet rs = (ResultSet) cs.getObject(1)) {
        while (rs.next()) {
            System.out.println(rs.getString("name"));
        }
    }
}
conn.commit();

// MySQL stored procedure with result set
try (CallableStatement cs2 = conn.prepareCall("{call list_users(?)}")) {
    cs2.setString(1, "Engineering");
    boolean hasResults = cs2.execute();

    while (hasResults || cs2.getUpdateCount() != -1) {
        if (hasResults) {
            try (ResultSet rs = cs2.getResultSet()) {
                while (rs.next()) { ... }
            }
        }
        hasResults = cs2.getMoreResults();
    }
}
```

</details>

---

**Q22.** 🟡 How do you handle `SQLException` properly?

<details><summary>Click to reveal answer</summary>

```java
try {
    // JDBC operation
} catch (SQLException e) {
    // SQL state (standard code)
    String sqlState = e.getSQLState();
    // e.g., "23505" = unique constraint violation in PostgreSQL
    //        "23000" = integrity constraint violation (generic)

    // Vendor error code (DB-specific)
    int errorCode = e.getErrorCode();
    // MySQL: 1062 = duplicate entry
    // PostgreSQL: 0 (use sqlState instead)

    // Error message
    String message = e.getMessage();

    // Chained exceptions (batch updates, multi-error)
    SQLException next = e.getNextException();
    while (next != null) {
        System.err.println("  Chained: " + next.getMessage());
        next = next.getNextException();
    }

    // Handle specific cases
    if ("23505".equals(sqlState)) {
        throw new DuplicateKeyException("Record already exists", e);
    } else if ("57014".equals(sqlState)) {
        throw new QueryTimeoutException("Query timed out", e);
    } else {
        throw new DatabaseException("Database error: " + message, e);
    }
}

// Java 7+ - multiple specific exceptions
try { ... }
catch (SQLTimeoutException e) { handleTimeout(e); }
catch (SQLIntegrityConstraintViolationException e) { handleDuplicate(e); }
catch (SQLTransientException e) { retry(); }
catch (SQLNonTransientException e) { fail(e); }
```

</details>

---

**Q23.** 🟡 What is `ResultSetMetaData` and when is it useful?

<details><summary>Click to reveal answer</summary>

`ResultSetMetaData` provides information about the columns in a `ResultSet` — useful for generic JDBC utilities and tools.

```java
try (ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
    ResultSetMetaData meta = rs.getMetaData();
    int cols = meta.getColumnCount();

    // Print column headers
    for (int i = 1; i <= cols; i++) {
        System.out.printf("%-20s %-15s %-5s %s%n",
            meta.getColumnName(i),
            meta.getColumnTypeName(i),
            meta.getColumnDisplaySize(i),
            meta.isNullable(i) == ResultSetMetaData.columnNullable ? "NULL" : "NOT NULL"
        );
    }

    // Generic row-to-map converter
    while (rs.next()) {
        Map<String, Object> row = new LinkedHashMap<>();
        for (int i = 1; i <= cols; i++) {
            row.put(meta.getColumnLabel(i), rs.getObject(i));
        }
        System.out.println(row);
    }
}
```

</details>

---

**Q24.** 🟡 What is lazy vs eager loading in the context of JDBC and JPA?

<details><summary>Click to reveal answer</summary>

In JDBC context, lazy loading refers to fetching related data **only when accessed**, rather than upfront.

```java
// EAGER loading - fetch everything in one query (JOIN)
String sql = """
    SELECT u.id, u.name, o.id as order_id, o.total
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.id = ?
    """;

// LAZY loading - separate queries (N+1 problem risk!)
// Fetch user
User user = findUserById(id);
// Fetch orders only when needed
List<Order> orders = findOrdersByUserId(user.getId()); // 2nd query

// In JPA:
// @OneToMany(fetch = FetchType.LAZY)  // default for collections
// @ManyToOne(fetch = FetchType.EAGER) // default for single references

// N+1 problem in JPA:
List<User> users = userRepo.findAll(); // 1 query for users
for (User u : users) {
    u.getOrders().size(); // N queries for orders!
}
// Fix: use JOIN FETCH or @EntityGraph
```

</details>

---

**Q25.** 🔴 How do you implement optimistic locking in JDBC?

<details><summary>Click to reveal answer</summary>

Optimistic locking prevents lost updates by checking if a row was modified since you last read it.

```java
// Table: products (id, name, price, version)

// Read with version
PreparedStatement selectPs = conn.prepareStatement(
    "SELECT id, name, price, version FROM products WHERE id = ?");
selectPs.setLong(1, productId);
ResultSet rs = selectPs.executeQuery();
rs.next();
int currentVersion = rs.getInt("version");
double currentPrice = rs.getDouble("price");

// Modify and update with version check
PreparedStatement updatePs = conn.prepareStatement(
    "UPDATE products SET price = ?, version = version + 1 " +
    "WHERE id = ? AND version = ?");
updatePs.setDouble(1, newPrice);
updatePs.setLong(2, productId);
updatePs.setInt(3, currentVersion); // will fail if someone else updated

int rowsUpdated = updatePs.executeUpdate();

if (rowsUpdated == 0) {
    // Another transaction updated the row - handle conflict
    throw new OptimisticLockException(
        "Product " + productId + " was modified by another transaction");
}
// Success: 1 row updated

// JPA equivalent:
// @Version
// private int version; // JPA handles this automatically
```

</details>

---

**Q26.** 🟡 How do you use `JNDI` to look up a DataSource?

<details><summary>Click to reveal answer</summary>

In Java EE / Jakarta EE containers (Tomcat, JBoss, WebLogic), `DataSource` is configured in the container and accessed via JNDI.

```java
// JNDI lookup
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/MyDataSource");

// Or in a Java EE environment:
@Resource(name = "jdbc/MyDataSource")
private DataSource dataSource; // container injects it

// Tomcat context.xml configuration:
/*
<Resource name="jdbc/MyDataSource"
          auth="Container"
          type="javax.sql.DataSource"
          driverClassName="org.postgresql.Driver"
          url="jdbc:postgresql://localhost/mydb"
          username="user"
          password="password"
          maxTotal="20"
          maxIdle="10"
          maxWaitMillis="10000"/>
*/

// Spring with JNDI:
// @Bean
// public DataSource dataSource() throws NamingException {
//     JndiDataSourceLookup lookup = new JndiDataSourceLookup();
//     return lookup.getDataSource("java:comp/env/jdbc/MyDataSource");
// }
```

</details>

---

**Q27.** 🟡 What is the difference between `java.sql.Date`, `java.sql.Timestamp`, and Java 8 time types?

<details><summary>Click to reveal answer</summary>

```java
// Legacy Java SQL types
java.sql.Date sqlDate = java.sql.Date.valueOf(LocalDate.now()); // date only, no time
java.sql.Time sqlTime = java.sql.Time.valueOf(LocalTime.now()); // time only, no date
java.sql.Timestamp sqlTs = java.sql.Timestamp.valueOf(LocalDateTime.now()); // date+time

// Setting parameters
ps.setDate(1, sqlDate);
ps.setTime(2, sqlTime);
ps.setTimestamp(3, sqlTs);

// Java 8 approach (JDBC 4.2+)
ps.setObject(1, LocalDate.now());      // date only
ps.setObject(2, LocalTime.now());      // time only
ps.setObject(3, LocalDateTime.now());  // date+time (no timezone)
ps.setObject(4, OffsetDateTime.now()); // date+time+offset
ps.setObject(5, Instant.now());        // UTC timestamp

// Reading Java 8 types
LocalDate date = rs.getObject("birth_date", LocalDate.class);
LocalDateTime dateTime = rs.getObject("created_at", LocalDateTime.class);
OffsetDateTime offsetDt = rs.getObject("updated_at", OffsetDateTime.class);
```

✅ **Best Practice:** Use `setObject()` with Java 8 time types when your JDBC driver supports it (most modern drivers do). Store timestamps as `TIMESTAMP WITH TIME ZONE` and use `OffsetDateTime`/`Instant`.

</details>

---

**Q28.** 🟡 How do you handle LOBs (Large Objects) in JDBC?

<details><summary>Click to reveal answer</summary>

```java
// BLOB - Binary Large Object
// Writing BLOB
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO documents (id, content) VALUES (?, ?)");
ps.setInt(1, docId);

// Small BLOB
ps.setBytes(2, smallFileBytes);

// Large BLOB (stream)
try (InputStream is = Files.newInputStream(largefile)) {
    ps.setBinaryStream(2, is, Files.size(largefile));
}
ps.executeUpdate();

// Reading BLOB
PreparedStatement selectPs = conn.prepareStatement(
    "SELECT content FROM documents WHERE id = ?");
selectPs.setInt(1, docId);
ResultSet rs = selectPs.executeQuery();
if (rs.next()) {
    Blob blob = rs.getBlob("content");
    byte[] bytes = blob.getBytes(1, (int) blob.length());
    // Or stream:
    InputStream is = blob.getBinaryStream();
}

// CLOB - Character Large Object (text)
PreparedStatement ps2 = conn.prepareStatement(
    "INSERT INTO articles (id, body) VALUES (?, ?)");
ps2.setInt(1, articleId);
ps2.setCharacterStream(2, new FileReader("article.txt"), fileLength);
ps2.executeUpdate();

ResultSet rs2 = ps2.getGeneratedKeys(); // if applicable
Clob clob = rs2.getClob("body");
String text = clob.getSubString(1, (int) clob.length());
```

</details>

---

**Q29.** 🔴 Explain JDBC 4.0 features and automatic driver loading.

<details><summary>Click to reveal answer</summary>

**JDBC 4.0 (Java 6+) improvements:**

1. **Automatic driver loading:** No more `Class.forName("com.mysql.jdbc.Driver")` — drivers register themselves via `ServiceLoader` (`META-INF/services/java.sql.Driver`)

```java
// Before JDBC 4.0:
Class.forName("com.mysql.jdbc.Driver"); // required explicit loading
Connection conn = DriverManager.getConnection(url, user, pass);

// JDBC 4.0+ - just this:
Connection conn = DriverManager.getConnection(url, user, pass);
// Driver auto-detected from classpath
```

2. **`Connection.isValid(timeout)`** — lighter connection validation than test query
3. **`SQLException` hierarchy** — typed subclasses for different error categories
4. **`SQLXML` support** — native XML type
5. **`RowId` support** — database row identifiers
6. **Enhanced `BLOB`/`CLOB`** — creation methods on `Connection`

```java
// JDBC 4.0 connection validation
boolean valid = conn.isValid(5); // returns false if connection broken within 5s

// Creating LOBs
Blob blob = conn.createBlob();
blob.setBytes(1, data);
Clob clob = conn.createClob();
clob.setString(1, text);
```

</details>

---

**Q30.** 🟡 How do you use `RowMapper` pattern with JDBC?

<details><summary>Click to reveal answer</summary>

The `RowMapper` pattern (popularized by Spring's `JdbcTemplate`) separates result set mapping from query execution.

```java
// Without RowMapper (repetitive)
List<User> users = new ArrayList<>();
try (ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
    while (rs.next()) {
        User u = new User();
        u.setId(rs.getLong("id"));
        u.setName(rs.getString("name"));
        users.add(u);
    }
}

// RowMapper interface
@FunctionalInterface
interface RowMapper<T> {
    T mapRow(ResultSet rs, int rowNum) throws SQLException;
}

// Reusable mapper
RowMapper<User> userMapper = (rs, rowNum) -> {
    User u = new User();
    u.setId(rs.getLong("id"));
    u.setName(rs.getString("name"));
    u.setEmail(rs.getString("email"));
    return u;
};

// Generic query executor
public <T> List<T> query(String sql, RowMapper<T> mapper, Object... params) throws SQLException {
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        for (int i = 0; i < params.length; i++) {
            ps.setObject(i + 1, params[i]);
        }
        List<T> results = new ArrayList<>();
        try (ResultSet rs = ps.executeQuery()) {
            int rowNum = 0;
            while (rs.next()) results.add(mapper.mapRow(rs, rowNum++));
        }
        return results;
    }
}

// Usage
List<User> users2 = query("SELECT * FROM users WHERE age > ?", userMapper, 18);
```

</details>

---

**Q31–Q42.** Additional theory questions:

**Q31.** 🟢 What is `Statement.RETURN_GENERATED_KEYS`? → Tells JDBC to return auto-generated keys (like auto-increment IDs) after INSERT.

**Q32.** 🟡 What is connection leak detection in HikariCP? → `leakDetectionThreshold` — logs stack trace if connection is held longer than specified time.

**Q33.** 🔴 What is a distributed transaction (XA)? → Transaction spanning multiple databases/resources; uses `XADataSource` and two-phase commit protocol.

**Q34.** 🟡 How does `Statement.cancel()` work? → Sends a cancellation signal to the DB; may not work for all drivers.

**Q35.** 🟡 What is `fetchSize` and how does it affect memory? → Controls how many rows JDBC fetches per network round trip (`rs.setFetchSize()`); prevents OOM for large result sets.

**Q36.** 🟡 What is JDBC URL parameter for SSL? → MySQL: `useSSL=true`; PostgreSQL: `ssl=true&sslmode=require`.

**Q37.** 🔴 What is `IsolationLevel.REPEATABLE_READ` and phantom reads? → Prevents non-repeatable reads but range queries may return different rows if another TX inserts data.

**Q38.** 🟡 How does `setQueryTimeout()` work? → Sets max time a statement may run before `SQLTimeoutException`.

**Q39.** 🟢 What is the purpose of `conn.rollback(Savepoint sp)`? → Rolls back to the named savepoint, keeping work done before that point.

**Q40.** 🔴 How do you implement retry logic for transient SQL errors? → Catch `SQLTransientException`, check `getSQLState()`, retry with exponential backoff.

**Q41.** 🟡 What is JNDI resource reference in web.xml? → `<resource-ref>` entry maps `java:comp/env/jdbc/MyDS` to the actual container resource.

**Q42.** 🟡 What is `PreparedStatement` cache in connection pools? → HikariCP's `cachePrepStmts=true` caches `PreparedStatement` objects per connection to avoid re-parsing.

---

## Scenario-Based Questions

**S1.** 🔴 Design a JDBC utility class that handles transactions with proper exception handling and connection management.

<details><summary>Click to reveal answer</summary>

```java
@FunctionalInterface
interface TransactionCallback<T> {
    T execute(Connection conn) throws SQLException;
}

public class JdbcTemplate {
    private final DataSource dataSource;

    public JdbcTemplate(DataSource dataSource) { this.dataSource = dataSource; }

    public <T> T executeInTransaction(TransactionCallback<T> callback) throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            boolean originalAutoCommit = conn.getAutoCommit();
            conn.setAutoCommit(false);
            try {
                T result = callback.execute(conn);
                conn.commit();
                return result;
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            } catch (Exception e) {
                conn.rollback();
                throw new SQLException("Transaction failed", e);
            } finally {
                conn.setAutoCommit(originalAutoCommit); // restore for pool
            }
        }
    }

    public <T> T queryForObject(String sql, RowMapper<T> mapper, Object... params) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.length; i++) ps.setObject(i + 1, params[i]);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    T result = mapper.mapRow(rs, 0);
                    if (rs.next()) throw new SQLException("Query returned more than one row");
                    return result;
                }
                return null;
            }
        }
    }

    public int update(String sql, Object... params) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.length; i++) ps.setObject(i + 1, params[i]);
            return ps.executeUpdate();
        }
    }
}

// Usage
JdbcTemplate jdbc = new JdbcTemplate(dataSource);

// Transaction
jdbc.executeInTransaction(conn -> {
    try (PreparedStatement debit = conn.prepareStatement(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?");
         PreparedStatement credit = conn.prepareStatement(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?")) {
        debit.setBigDecimal(1, amount); debit.setLong(2, fromId); debit.executeUpdate();
        credit.setBigDecimal(1, amount); credit.setLong(2, toId); credit.executeUpdate();
    }
    return null;
});
```

</details>

---

**S2.** 🟡 You need to import 1 million rows from a CSV into the database. Design an efficient solution.

<details><summary>Click to reveal answer</summary>

```java
public void bulkImportCsv(Path csvFile, DataSource ds) throws Exception {
    String insertSql = "INSERT INTO products (sku, name, price, category) VALUES (?, ?, ?, ?)";

    try (Connection conn = ds.getConnection();
         BufferedReader reader = Files.newBufferedReader(csvFile, StandardCharsets.UTF_8);
         PreparedStatement ps = conn.prepareStatement(insertSql)) {

        conn.setAutoCommit(false);

        final int BATCH_SIZE = 5000;
        int rowCount = 0;
        String line;

        reader.readLine(); // skip header

        while ((line = reader.readLine()) != null) {
            String[] parts = line.split(",", 4); // limit split for last field

            ps.setString(1, parts[0].trim());
            ps.setString(2, parts[1].trim());
            ps.setBigDecimal(3, new BigDecimal(parts[2].trim()));
            ps.setString(4, parts[3].trim());
            ps.addBatch();

            if (++rowCount % BATCH_SIZE == 0) {
                ps.executeBatch();
                conn.commit();
                System.out.printf("Imported %,d rows%n", rowCount);
            }
        }

        // Process remaining
        ps.executeBatch();
        conn.commit();
        System.out.printf("Import complete: %,d total rows%n", rowCount);
    }
}

// Even faster: use DB-native bulk import
// PostgreSQL COPY:
// COPY products FROM '/path/data.csv' CSV HEADER DELIMITER ',' ENCODING 'UTF-8';
// CopyManager manager = ((PGConnection) conn).getCopyAPI();
// manager.copyIn("COPY products FROM STDIN CSV HEADER", csvInputStream);
```

</details>

---

**S3.** 🟡 How would you implement a repository pattern with JDBC?

<details><summary>Click to reveal answer</summary>

```java
public class UserJdbcRepository {
    private final DataSource dataSource;

    private static final RowMapper<User> USER_MAPPER = (rs, rowNum) -> User.builder()
        .id(rs.getLong("id"))
        .username(rs.getString("username"))
        .email(rs.getString("email"))
        .createdAt(rs.getObject("created_at", LocalDateTime.class))
        .build();

    public Optional<User> findById(Long id) throws SQLException {
        String sql = "SELECT * FROM users WHERE id = ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(USER_MAPPER.mapRow(rs, 0)) : Optional.empty();
            }
        }
    }

    public List<User> findAll(int page, int pageSize) throws SQLException {
        String sql = "SELECT * FROM users ORDER BY id LIMIT ? OFFSET ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, pageSize);
            ps.setInt(2, page * pageSize);
            try (ResultSet rs = ps.executeQuery()) {
                List<User> users = new ArrayList<>();
                while (rs.next()) users.add(USER_MAPPER.mapRow(rs, 0));
                return users;
            }
        }
    }

    public User save(User user) throws SQLException {
        if (user.getId() == null) return insert(user);
        else return update(user);
    }

    private User insert(User user) throws SQLException {
        String sql = "INSERT INTO users (username, email) VALUES (?, ?) RETURNING id, created_at";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, user.getUsername());
            ps.setString(2, user.getEmail());
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return user.toBuilder()
                    .id(rs.getLong("id"))
                    .createdAt(rs.getObject("created_at", LocalDateTime.class))
                    .build();
            }
        }
    }
}
```

</details>

---

**S4.** 🔴 How would you detect and fix the N+1 query problem in JDBC?

<details><summary>Click to reveal answer</summary>

```java
// N+1 PROBLEM
// 1 query for orders + N queries for each order's items
List<Order> orders = findAllOrders(conn); // 1 query
for (Order order : orders) {
    List<OrderItem> items = findItemsByOrderId(conn, order.getId()); // N queries!
    order.setItems(items);
}

// SOLUTION 1: JOIN query
String sql = """
    SELECT o.id, o.customer_id, o.total,
           oi.id as item_id, oi.product_name, oi.quantity, oi.price
    FROM orders o
    LEFT JOIN order_items oi ON oi.order_id = o.id
    ORDER BY o.id
    """;
Map<Long, Order> orderMap = new LinkedHashMap<>();
try (ResultSet rs = stmt.executeQuery(sql)) {
    while (rs.next()) {
        Long orderId = rs.getLong("id");
        Order order = orderMap.computeIfAbsent(orderId, id ->
            new Order(id, rs.getLong("customer_id")));
        if (rs.getLong("item_id") != 0) {
            order.getItems().add(new OrderItem(
                rs.getLong("item_id"),
                rs.getString("product_name"),
                rs.getInt("quantity"),
                rs.getBigDecimal("price")));
        }
    }
}
List<Order> orders2 = new ArrayList<>(orderMap.values());

// SOLUTION 2: IN clause (for specific IDs)
List<Long> orderIds = fetchOrderIds(conn);
String inClause = orderIds.stream().map(id -> "?").collect(joining(","));
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM order_items WHERE order_id IN (" + inClause + ")");
for (int i = 0; i < orderIds.size(); i++) ps.setLong(i + 1, orderIds.get(i));
// Group results by order_id and merge
```

</details>

---

**S5.** 🟡 You see slow queries in your application. How do you diagnose and fix with JDBC?

<details><summary>Click to reveal answer</summary>

```java
// 1. Log slow queries
long start = System.currentTimeMillis();
ResultSet rs = ps.executeQuery();
long elapsed = System.currentTimeMillis() - start;
if (elapsed > 100) {
    log.warn("Slow query ({}ms): {}", elapsed, sql);
}

// 2. Use query timeout
ps.setQueryTimeout(10); // seconds before SQLTimeoutException

// 3. Analyze with EXPLAIN
Statement stmt = conn.createStatement();
ResultSet explain = stmt.executeQuery("EXPLAIN ANALYZE " + sql);
while (explain.next()) System.out.println(explain.getString(1));

// 4. Use PreparedStatement caching (HikariCP)
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");

// 5. Optimize fetchSize for large result sets
ps.setFetchSize(1000); // fetch 1000 rows at a time instead of all at once

// 6. Select only needed columns
// BAD: SELECT *
// GOOD: SELECT id, name, email FROM users WHERE ...

// 7. Add indexes (can verify missing indexes via slow query log)
stmt.execute("CREATE INDEX CONCURRENTLY idx_users_email ON users(email)");
```

</details>

---

**S6–S30.** Additional scenario topics (brief):

**S6.** 🟡 Implement `findByExample` using dynamic WHERE clause with JDBC.

**S7.** 🔴 Handle a database failover (primary → replica) with retry logic.

**S8.** 🟡 Implement soft delete pattern in JDBC (`deleted_at` column).

**S9.** 🟡 Multi-tenant JDBC: different schemas per tenant, same pool.

**S10.** 🔴 Implement a changelog/audit table that captures all changes.

**S11.** 🟡 How to unit test JDBC code? → Use H2 in-memory DB or Testcontainers.

**S12.** 🟡 How to handle database connection in a multithreaded application? → Each thread gets its own connection from the pool; never share connections.

**S13.** 🔴 How to implement cursor-based pagination in JDBC? → `WHERE id > lastSeenId ORDER BY id LIMIT ?` instead of OFFSET.

**S14.** 🟡 How to handle timezone issues with JDBC timestamps? → Store as UTC, use `TIMESTAMP WITH TIME ZONE`, set JVM timezone.

**S15.** 🟡 How to implement read/write splitting with JDBC? → Use separate DataSource instances for read and write; route queries based on type.

---

## 🔗 Related Topics
- [Spring Framework & Hibernate](18-spring-framework-and-hibernate-intro.md)
- [Java Security](16-java-security.md) — SQL injection
- [Java Testing](12-java-testing.md) — Testing JDBC code

---

*✅ Best Practice: Always use `PreparedStatement`, always use try-with-resources, always use connection pooling (HikariCP), never store passwords in plain text.*

*🚨 Common Mistake: Using `Statement` with string concatenation (SQL injection), not closing resources (connection leaks), or forgetting to reset `autoCommit` before returning connections to pool.*
