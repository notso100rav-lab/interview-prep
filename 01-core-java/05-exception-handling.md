# Chapter 5: Exception Handling

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- Exception hierarchy
- Checked vs unchecked exceptions
- `try-with-resources`
- Custom exceptions
- Exception chaining (cause)
- `finally` block traps and edge cases
- Multi-catch and rethrowing
- Exception handling in multithreaded code
- Anti-patterns and best practices
- Exception handling strategies for production

---

## 🔑 Key Concepts at a Glance

```
Exception Hierarchy:
  Throwable
  ├── Error                 (unchecked — JVM issues, don't catch)
  │   ├── OutOfMemoryError
  │   ├── StackOverflowError
  │   └── AssertionError
  └── Exception
      ├── RuntimeException  (unchecked — programming errors)
      │   ├── NullPointerException
      │   ├── IllegalArgumentException
      │   ├── IllegalStateException
      │   ├── ClassCastException
      │   ├── ArrayIndexOutOfBoundsException
      │   └── ArithmeticException
      └── (Others)          (checked — recoverable conditions)
          ├── IOException
          ├── SQLException
          ├── ClassNotFoundException
          └── InterruptedException
```

---

## 🧠 Theory Questions

### Exception Hierarchy

---

🟢 **Q1. What is the difference between checked and unchecked exceptions?**

<details><summary>Click to reveal answer</summary>

| | Checked | Unchecked |
|-|---------|----------|
| **Extends** | `Exception` (not `RuntimeException`) | `RuntimeException` or `Error` |
| **Compile check** | ✅ Must handle or declare | ❌ Optional |
| **Recovery** | Expected — caller should handle | Programming error or unrecoverable |
| **Examples** | `IOException`, `SQLException`, `InterruptedException` | `NullPointerException`, `IllegalArgumentException`, `StackOverflowError` |
| **Method signature** | `void method() throws IOException` | No declaration needed |

```java
// Checked — must be declared or caught
public String readFile(String path) throws IOException {  // must declare!
    return Files.readString(Path.of(path));
}

// Calling checked exception method:
try {
    String content = readFile("data.txt");
} catch (IOException e) {
    // handle
}

// Unchecked — no declaration needed
public int divide(int a, int b) {
    if (b == 0) throw new ArithmeticException("Division by zero");  // no throws clause
    return a / b;
}

// Can be called without try-catch:
int result = divide(10, 2);
```

**Debate: Checked vs Unchecked?**
- **Pro-checked:** Forces callers to handle expected failure cases
- **Pro-unchecked:** Reduces boilerplate, easier to propagate, no tight coupling

**Modern Java trend (Spring, etc.):** Prefer unchecked exceptions (wrap checked in unchecked), let a global exception handler deal with them.

> 💡 **Interviewer Tip:** The checked exception debate is a common topic. Know both sides. In practice, Spring wraps almost everything in `RuntimeException` subclasses for this reason.

</details>

---

🟡 **Q2. What is the exception hierarchy? What's the difference between `Error` and `Exception`?**

<details><summary>Click to reveal answer</summary>

```
Throwable
├── Error                     → Serious JVM-level problems (should NOT be caught)
│   ├── OutOfMemoryError      → Heap exhausted
│   ├── StackOverflowError    → Recursion too deep
│   ├── VirtualMachineError   → JVM internal error
│   └── AssertionError        → assert statement failed
└── Exception                 → Application-level problems (should handle)
    ├── RuntimeException      → Unchecked (programming errors)
    └── ... others ...        → Checked (expected failures)
```

**`Error` — don't catch (usually):**
```java
// Generally wrong:
try {
    recursiveMethod();
} catch (StackOverflowError e) {
    // Nothing meaningful you can do — JVM state is corrupted
}

// Exception: you might catch Error in framework code for cleanup
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    if (throwable instanceof Error) {
        log.error("Fatal error in thread " + thread.getName(), throwable);
        // attempt graceful shutdown
    }
});
```

**Exception — catch and handle:**
```java
try {
    connection = dataSource.getConnection();
} catch (SQLException e) {
    log.error("Database error", e);
    throw new DataAccessException("Failed to get connection", e);
}
```

</details>

---

### try-with-resources

---

🟡 **Q3. What is `try-with-resources` and how does it work?**

<details><summary>Click to reveal answer</summary>

`try-with-resources` (Java 7+) automatically closes resources that implement `AutoCloseable` at the end of the try block.

```java
// OLD way — prone to resource leaks
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    return reader.readLine();
} catch (IOException e) {
    throw new RuntimeException(e);
} finally {
    if (reader != null) {
        try { reader.close(); } catch (IOException ignored) {}  // ugly!
    }
}

// NEW way — automatic, clean, exception-safe
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    return reader.readLine();
}  // reader.close() called automatically — even if exception thrown

// Multiple resources — closed in REVERSE order of declaration
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT * FROM users");
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        // process
    }
}
// rs.close() → ps.close() → conn.close() (reverse order)
```

**Custom AutoCloseable:**
```java
public class DatabaseTransaction implements AutoCloseable {
    private final Connection connection;
    private boolean committed = false;

    public DatabaseTransaction(DataSource ds) throws SQLException {
        this.connection = ds.getConnection();
        this.connection.setAutoCommit(false);
    }

    public void commit() throws SQLException {
        connection.commit();
        committed = true;
    }

    @Override
    public void close() throws SQLException {
        if (!committed) {
            connection.rollback();  // auto-rollback on exception
        }
        connection.close();
    }
}

// Usage:
try (DatabaseTransaction tx = new DatabaseTransaction(dataSource)) {
    updateUser(tx.connection, user);
    createOrder(tx.connection, order);
    tx.commit();
}  // auto-rollback if exception was thrown before commit
```

**Suppressed exceptions:**
```java
try (MyResource r = new MyResource()) {
    throw new RuntimeException("primary exception");
    // If r.close() also throws, the close exception is SUPPRESSED
    // (attached to primary exception, not lost)
}

// Access suppressed exceptions:
catch (Exception e) {
    for (Throwable suppressed : e.getSuppressed()) {
        log.warn("Suppressed during close: ", suppressed);
    }
}
```

</details>

---

### Custom Exceptions

---

🟡 **Q4. How do you design custom exceptions?**

<details><summary>Click to reveal answer</summary>

```java
// 1. Domain-specific unchecked exception (preferred for application exceptions)
public class UserNotFoundException extends RuntimeException {
    private final Long userId;

    public UserNotFoundException(Long userId) {
        super("User not found with id: " + userId);
        this.userId = userId;
    }

    public UserNotFoundException(Long userId, Throwable cause) {
        super("User not found with id: " + userId, cause);
        this.userId = userId;
    }

    public Long getUserId() { return userId; }
}

// 2. Exception hierarchy for a module
public class PaymentException extends RuntimeException {
    public PaymentException(String message) { super(message); }
    public PaymentException(String message, Throwable cause) { super(message, cause); }
}

public class InsufficientFundsException extends PaymentException {
    private final double available;
    private final double required;

    public InsufficientFundsException(double available, double required) {
        super(String.format("Insufficient funds: available=%.2f, required=%.2f", available, required));
        this.available = available;
        this.required = required;
    }

    public double getAvailable() { return available; }
    public double getRequired() { return required; }
}

public class PaymentGatewayException extends PaymentException {
    private final String gatewayErrorCode;

    public PaymentGatewayException(String message, String gatewayErrorCode, Throwable cause) {
        super(message, cause);
        this.gatewayErrorCode = gatewayErrorCode;
    }

    public String getGatewayErrorCode() { return gatewayErrorCode; }
}

// Usage:
try {
    paymentService.charge(customer, amount);
} catch (InsufficientFundsException e) {
    return ResponseEntity.badRequest()
        .body(Map.of("error", "insufficient_funds",
                     "available", e.getAvailable()));
} catch (PaymentGatewayException e) {
    log.error("Gateway error: {}", e.getGatewayErrorCode(), e);
    return ResponseEntity.status(502).body(Map.of("error", "gateway_error"));
} catch (PaymentException e) {
    log.error("Payment failed", e);
    return ResponseEntity.internalServerError().build();
}
```

**Design guidelines:**
- Name ends with `Exception`
- Extend appropriate base class
- Always provide: `String message` and `Throwable cause` constructors
- Include relevant context in the exception (not just message)
- Avoid exception hierarchies deeper than 3 levels

</details>

---

### Exception Chaining

---

🟡 **Q5. What is exception chaining and why is it important?**

<details><summary>Click to reveal answer</summary>

Exception chaining wraps a lower-level exception in a higher-level one, preserving the full stack trace while adding context.

```java
// BAD: Losing the original cause
public User findUser(long id) throws UserNotFoundException {
    try {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    } catch (DataAccessException e) {
        throw new UserNotFoundException(id);  // LOST the database error!
    }
}

// GOOD: Chain the cause
public User findUser(long id) throws UserNotFoundException {
    try {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    } catch (DataAccessException e) {
        throw new UserNotFoundException(id, e);  // e is preserved as cause
    }
}

// Logging shows full chain:
// UserNotFoundException: User not found with id: 123
//   Caused by: DataAccessException: Unable to execute query
//   Caused by: SQLException: Connection timeout

// Access chain programmatically:
catch (UserNotFoundException e) {
    Throwable cause = e.getCause();
    while (cause != null) {
        log.debug("Caused by: " + cause.getMessage());
        cause = cause.getCause();
    }
}
```

**Wrapping checked as unchecked:**
```java
// Common pattern in modern Java
try {
    return Files.readString(Path.of(path));
} catch (IOException e) {
    throw new UncheckedIOException(e);  // standard wrapper from java.io
    // OR: throw new RuntimeException("Failed to read " + path, e);
}
```

</details>

---

### Finally Block Traps

---

🟡 **Q6. What are the tricky behaviors of the `finally` block?**

<details><summary>Click to reveal answer</summary>

**Trap 1: `return` in `finally` overrides `try`'s `return`:**
```java
public int getValue() {
    try {
        return 1;
    } finally {
        return 2;  // BAD: overrides return 1
    }
}
// Returns 2 — but the original exception (if any) is LOST!
```

**Trap 2: Exception in `finally` suppresses `try`'s exception:**
```java
public void risky() throws IOException {
    try {
        throw new IOException("original");
    } finally {
        throw new RuntimeException("from finally");  // BAD: original IOException LOST!
    }
}
// Caller only sees RuntimeException, original IOException silently discarded
```

**Trap 3: `finally` doesn't run with `System.exit()`:**
```java
try {
    System.exit(0);
} finally {
    System.out.println("This never runs");  // not printed
}
```

**Trap 4: Modifying the return value in `finally` (primitive):**
```java
public int count() {
    int x = 10;
    try {
        return x;   // saves x=10
    } finally {
        x = 20;     // modifies local x, but saved return value is still 10
    }
}
// Returns 10 (for primitives)
```

**Trap 5: `finally` always runs, even with break/continue:**
```java
for (int i = 0; i < 3; i++) {
    try {
        if (i == 1) break;
    } finally {
        System.out.println("finally " + i);  // runs: "finally 0", "finally 1"
    }
}
// break doesn't prevent finally from running
```

> 💡 **Best Practice:** Never use `return` or `throw` in `finally` blocks. Keep `finally` for cleanup only.

</details>

---

### Multi-Catch and Re-throwing

---

🟡 **Q7. What is multi-catch syntax and how does re-throwing work?**

<details><summary>Click to reveal answer</summary>

**Multi-catch (Java 7+):**
```java
// OLD way — duplicate code
try {
    risky();
} catch (IOException e) {
    log.error("IO error", e);
    throw new ServiceException(e);
} catch (SQLException e) {
    log.error("SQL error", e);
    throw new ServiceException(e);
}

// NEW way — multi-catch
try {
    risky();
} catch (IOException | SQLException e) {
    // e is effectively final — cannot be reassigned
    log.error("Error accessing data", e);
    throw new ServiceException(e);
}
```

**Re-throwing (preserve type information):**
```java
// Re-throw as-is
try {
    riskyOperation();
} catch (RuntimeException e) {
    log.error("Error", e);
    throw e;  // re-throw — same exception, stack trace preserved
}

// Re-throw with type (Java 7+ — compiler is smarter)
public void process() throws IOException {
    try {
        Files.readString(Path.of("file.txt"));
    } catch (Exception e) {
        log.error("Failed", e);
        throw e;  // Java knows this can only be IOException in this context
                  // So compiles even though catch is Exception
    }
}

// Wrapping (exception chaining)
try {
    repository.save(entity);
} catch (DataAccessException e) {
    throw new ServiceException("Failed to save entity", e);  // wrap with context
}
```

**Catching `Throwable` — rarely correct:**
```java
// Usually wrong (catches Errors!)
catch (Throwable t) { ... }

// Exception: top-level request handler to prevent server crashes
// Exception: cleanup code that must absolutely run
try {
    processRequest(request);
} catch (Exception e) {
    log.error("Request processing failed", e);
    // catch Exception (not Throwable) — let Errors propagate to JVM
}
```

</details>

---

### Exception Handling in Multithreaded Code

---

🔴 **Q8. How are exceptions handled in multithreaded code?**

<details><summary>Click to reveal answer</summary>

Exceptions thrown in a thread do **not** propagate to the calling thread — they need special handling.

**Problem:**
```java
Thread t = new Thread(() -> {
    throw new RuntimeException("error in thread");
});
t.start();
// Main thread has NO idea this exception occurred!
```

**Solution 1: `UncaughtExceptionHandler`:**
```java
Thread t = new Thread(() -> {
    throw new RuntimeException("thread error");
});

t.setUncaughtExceptionHandler((thread, throwable) -> {
    System.err.println("Exception in " + thread.getName() + ": " + throwable.getMessage());
    // log, alert, restart, etc.
});

// Global handler for all threads without a specific handler:
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    log.error("Unhandled exception in thread " + thread.getName(), throwable);
});
```

**Solution 2: `Future.get()` wraps exceptions:**
```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("task error");
});

try {
    future.get();  // blocks and rethrows wrapped in ExecutionException
} catch (ExecutionException e) {
    Throwable cause = e.getCause();  // the original RuntimeException
    log.error("Task failed: " + cause.getMessage());
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    log.warn("Interrupted while waiting for task");
}
```

**Solution 3: `CompletableFuture.exceptionally()`:**
```java
CompletableFuture.supplyAsync(() -> {
    if (new Random().nextBoolean()) throw new RuntimeException("async error");
    return "result";
})
.exceptionally(ex -> {
    log.error("Async task failed", ex);
    return "fallback-value";  // handle error, provide default
})
.thenAccept(result -> System.out.println("Result: " + result));
```

**Solution 4: `ScheduledExecutorService` — silent exception swallowing:**
```java
// BUG: ScheduledExecutorService silently stops scheduled tasks on uncaught exception!
ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
scheduler.scheduleAtFixedRate(() -> {
    try {
        doWork();
    } catch (Exception e) {  // MUST catch all exceptions!
        log.error("Scheduled task failed", e);
        // Without catch: task stops executing silently after first exception
    }
}, 0, 1, TimeUnit.MINUTES);
```

🚨 **Critical Bug Pattern:** Forgetting to catch exceptions in `scheduleAtFixedRate` tasks. The task silently stops after the first uncaught exception, often unnoticed until something fails to happen.

</details>

---

### InterruptedException

---

🔴 **Q9. Why is `InterruptedException` special? How should you handle it?**

<details><summary>Click to reveal answer</summary>

`InterruptedException` is thrown when a blocking method (e.g., `Thread.sleep()`, `Object.wait()`, `BlockingQueue.take()`) is interrupted via `Thread.interrupt()`.

**The interrupt mechanism:**
- `Thread.interrupt()` sets the **interrupted flag** on the target thread
- Blocking methods check this flag and throw `InterruptedException` (clearing the flag)
- Non-blocking code should check `Thread.currentThread().isInterrupted()` periodically

**The #1 anti-pattern — swallowing the interrupt:**
```java
// BAD: hides the interrupt, upper code never knows thread was interrupted
public void doWork() {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        // do nothing — WRONG! The interrupted status is now cleared.
    }
}
```

**Option 1: Propagate by declaring `throws InterruptedException`:**
```java
// GOOD: Let callers handle it
public void doWork() throws InterruptedException {
    Thread.sleep(1000);  // just don't catch it
}
```

**Option 2: Restore the interrupt flag:**
```java
// GOOD: Restore flag so calling code can detect it
public void doWork() {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();  // restore the flag!
        // optionally log
        // optionally return early or throw unchecked
    }
}
```

**Option 3: Wrap in unchecked (with flag restored):**
```java
public void doWork() {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new RuntimeException("Interrupted", e);
    }
}
```

**Cooperative cancellation:**
```java
public void longRunningTask() {
    while (!Thread.currentThread().isInterrupted()) {
        // check flag periodically
        processBatch();
    }
    // cleanup if interrupted
}
// Caller: workerThread.interrupt() — signals the worker to stop
```

</details>

---

### Anti-Patterns

---

🟡 **Q10. What are the most common exception handling anti-patterns?**

<details><summary>Click to reveal answer</summary>

**1. Empty catch block:**
```java
// TERRIBLE: Silently swallows exceptions
try {
    connection = getConnection();
} catch (Exception e) {
    // nothing here
}
// If getConnection() throws, connection is null and NPE follows elsewhere
```

**2. Catching `Exception` too broadly:**
```java
try {
    processOrder(order);
} catch (Exception e) {  // BAD: catches EVERYTHING including programming errors
    log.error("Error", e);
    return null;
}
// Catches NullPointerException, ClassCastException, etc. — masks bugs!
```

**3. Using exceptions for flow control:**
```java
// BAD: Expensive and misleading
try {
    int value = Integer.parseInt(input);
} catch (NumberFormatException e) {
    value = 0;  // default
}

// GOOD: Check first
int value = input.matches("\\d+") ? Integer.parseInt(input) : 0;
// Or: value = tryParseInt(input).orElse(0);
```

**4. Losing the original cause:**
```java
// BAD: original cause lost
catch (IOException e) {
    throw new ServiceException("IO failed");  // no cause!
}

// GOOD:
catch (IOException e) {
    throw new ServiceException("IO failed", e);  // preserve cause
}
```

**5. Logging and rethrowing (double logging):**
```java
// BAD: logged here AND at the top level — duplicate log entries
catch (SQLException e) {
    log.error("DB error", e);  // logs here
    throw e;                   // AND logs again at caller!
}

// GOOD: Either log OR rethrow (usually rethrow, let global handler log)
catch (SQLException e) {
    throw new DataAccessException("Failed to query users", e);
}
```

**6. Catching `NullPointerException`:**
```java
// BAD: NPE indicates a programming error — fix the root cause!
try {
    result = obj.process();
} catch (NullPointerException e) {
    result = defaultValue;
}

// GOOD: Check for null
if (obj != null) {
    result = obj.process();
} else {
    result = defaultValue;
}
// OR use Optional<>
```

**7. `printStackTrace()` without a logger:**
```java
// BAD: Goes to stderr, not captured by logging framework
catch (Exception e) {
    e.printStackTrace();
}

// GOOD:
catch (Exception e) {
    log.error("Operation failed", e);
}
```

</details>

---

## 🎭 Scenario-Based Questions

---

🟡 **S1. What's the output?**

```java
public class Test {
    public static int getNumber() {
        try {
            return 1;
        } catch (Exception e) {
            return 2;
        } finally {
            return 3;
        }
    }

    public static void main(String[] args) {
        System.out.println(getNumber());
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output: `3`**

The `finally` block's `return 3` overrides `try`'s `return 1`. This is why putting `return` statements in `finally` is a serious anti-pattern — it can silently suppress exceptions too.

```java
// Even more dangerous:
public static int getNumber() {
    try {
        throw new RuntimeException("error");
    } finally {
        return 3;  // exception is SILENTLY SWALLOWED!
    }
}
// No exception is thrown — returns 3
```

🚨 **Never** use `return`, `break`, or `throw` in `finally` blocks.

</details>

---

🟡 **S2. What's wrong with this code?**

```java
public List<User> getUsers() {
    try {
        return userRepository.findAll();
    } catch (Exception e) {
        return null;  // caller gets null, null checks everywhere...
    }
}
```

<details><summary>Click to reveal answer</summary>

**Multiple issues:**

1. **Returns `null`** — forces every caller to null-check
2. **Swallows the exception** — no logging, no error propagation
3. **Catches `Exception` too broadly** — hides programming errors

**Better approach:**
```java
// Option 1: Let exception propagate (often best)
public List<User> getUsers() {
    return userRepository.findAll();
    // Let checked exceptions propagate if any
}

// Option 2: Wrap and rethrow with context
public List<User> getUsers() {
    try {
        return userRepository.findAll();
    } catch (DataAccessException e) {
        throw new ServiceException("Failed to retrieve users", e);
    }
}

// Option 3: Return empty collection (not null) with logging
public List<User> getUsers() {
    try {
        return userRepository.findAll();
    } catch (DataAccessException e) {
        log.error("Failed to retrieve users", e);
        return Collections.emptyList();  // never null
    }
}

// Option 4: Return Optional/Result type
public Optional<List<User>> getUsers() {
    try {
        return Optional.of(userRepository.findAll());
    } catch (DataAccessException e) {
        log.error("Failed to retrieve users", e);
        return Optional.empty();
    }
}
```

</details>

---

🔴 **S3. Spot the resource leak:**

```java
public String processFile(String path) throws IOException {
    FileInputStream fis = new FileInputStream(path);
    BufferedReader reader = new BufferedReader(new InputStreamReader(fis));

    try {
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }
        return sb.toString();
    } catch (IOException e) {
        throw new RuntimeException("Failed to read file: " + path, e);
    }
    // WHERE IS THE CLOSE?!
}
```

<details><summary>Click to reveal answer</summary>

The `fis` and `reader` are **never closed** — resource leak on both happy path and exception path.

**Fix using try-with-resources:**
```java
public String processFile(String path) throws IOException {
    try (FileInputStream fis = new FileInputStream(path);
         BufferedReader reader = new BufferedReader(new InputStreamReader(fis, StandardCharsets.UTF_8))) {

        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }
        return sb.toString();
    }
    // reader and fis auto-closed in reverse order
}

// Even simpler with NIO:
public String processFile(String path) throws IOException {
    return Files.readString(Path.of(path), StandardCharsets.UTF_8);
}
```

</details>

---

🟡 **S4. What happens when an exception is thrown in the constructor?**

```java
public class ResourceHolder {
    private final Connection conn;
    private final Statement stmt;

    public ResourceHolder() throws SQLException {
        conn = DriverManager.getConnection(DB_URL);
        stmt = conn.createStatement();  // What if this throws?
    }

    public void close() throws SQLException {
        stmt.close();
        conn.close();
    }
}
```

<details><summary>Click to reveal answer</summary>

If `conn.createStatement()` throws, the constructor fails — `ResourceHolder` object is not created. The problem: `conn` was opened but never closed! Resource leak.

**Fix 1: Close in constructor's catch block:**
```java
public ResourceHolder() throws SQLException {
    conn = DriverManager.getConnection(DB_URL);
    try {
        stmt = conn.createStatement();
    } catch (SQLException e) {
        conn.close();  // close what we opened
        throw e;
    }
}
```

**Fix 2: Factory method with try-with-resources:**
```java
public static ResourceHolder create() throws SQLException {
    Connection conn = DriverManager.getConnection(DB_URL);
    try {
        Statement stmt = conn.createStatement();
        return new ResourceHolder(conn, stmt);
    } catch (SQLException e) {
        conn.close();
        throw e;
    }
}
```

**Fix 3: Implement `Closeable` and use try-with-resources:**
```java
public class ResourceHolder implements AutoCloseable {
    @Override
    public void close() {
        // close resources
    }
}
// Caller uses try-with-resources:
try (ResourceHolder rh = new ResourceHolder()) { ... }
```

</details>

---

🔴 **S5. What happens when this code runs?**

```java
public class Test {
    public static void main(String[] args) {
        try {
            method1();
        } catch (RuntimeException e) {
            System.out.println("Caught: " + e.getMessage());
            System.out.println("Cause: " + e.getCause().getMessage());
        }
    }

    static void method1() {
        try {
            method2();
        } catch (Exception e) {
            throw new RuntimeException("method1 failed", e);
        }
    }

    static void method2() {
        throw new IllegalArgumentException("method2 failed");
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output:**
```
Caught: method1 failed
Cause: method2 failed
```

**Execution trace:**
1. `method2()` throws `IllegalArgumentException("method2 failed")`
2. `method1()` catches it as `Exception e`, wraps it: `new RuntimeException("method1 failed", e)` — exception chain created
3. `main()` catches the `RuntimeException`, prints message and cause

**Stack trace would show:**
```
RuntimeException: method1 failed
    at Test.method1(Test.java:14)
    at Test.main(Test.java:4)
Caused by: IllegalArgumentException: method2 failed
    at Test.method2(Test.java:19)
    at Test.method1(Test.java:11)
```

This is the ideal exception chaining pattern — maintains full context of what went wrong.

</details>

---

🟡 **S6. Design a global exception handler for a REST API (Spring style).**

<details><summary>Click to reveal answer</summary>

```java
// Spring Boot global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handle specific domain exceptions
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleUserNotFound(UserNotFoundException ex, HttpServletRequest request) {
        return ErrorResponse.builder()
            .status(404)
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(ValidationException ex, HttpServletRequest request) {
        return ErrorResponse.builder()
            .status(400)
            .error("Bad Request")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }

    // Handle Spring validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());

        return Map.of(
            "status", 400,
            "errors", errors,
            "timestamp", Instant.now()
        );
    }

    // Catch-all — prevents stack traces leaking to clients
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleUnexpected(Exception ex, HttpServletRequest request) {
        // Log the full exception with stack trace
        log.error("Unexpected error for request {}", request.getRequestURI(), ex);

        // Return generic error to client (don't leak internals!)
        return ErrorResponse.builder()
            .status(500)
            .error("Internal Server Error")
            .message("An unexpected error occurred")  // generic message
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }
}

@Data @Builder
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private String path;
    private Instant timestamp;
}
```

</details>

---

🟡 **S7. What's wrong? (Exception in streams)**

```java
List<String> paths = List.of("file1.txt", "file2.txt", "file3.txt");

List<String> contents = paths.stream()
    .map(path -> Files.readString(Path.of(path)))  // COMPILE ERROR
    .collect(Collectors.toList());
```

<details><summary>Click to reveal answer</summary>

`Files.readString()` throws checked `IOException`, but lambdas in streams cannot throw checked exceptions.

**Fix 1: Wrap in try-catch inside lambda:**
```java
List<String> contents = paths.stream()
    .map(path -> {
        try {
            return Files.readString(Path.of(path));
        } catch (IOException e) {
            throw new UncheckedIOException(e);  // wrap as unchecked
        }
    })
    .collect(Collectors.toList());
```

**Fix 2: Helper method:**
```java
private static String readFile(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}

List<String> contents = paths.stream()
    .map(Test::readFile)  // method reference, clean
    .collect(Collectors.toList());
```

**Fix 3: Utility method (general "sneaky throws" pattern — careful!):**
```java
@FunctionalInterface
public interface CheckedFunction<T, R> {
    R apply(T t) throws Exception;
}

public static <T, R> Function<T, R> wrap(CheckedFunction<T, R> fn) {
    return t -> {
        try { return fn.apply(t); }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}

// Usage:
List<String> contents = paths.stream()
    .map(wrap(path -> Files.readString(Path.of(path))))
    .collect(Collectors.toList());
```

</details>

---

🔴 **S8. Implement retry logic with exception handling.**

<details><summary>Click to reveal answer</summary>

```java
public class RetryHandler {
    private static final Logger log = LoggerFactory.getLogger(RetryHandler.class);

    public static <T> T withRetry(
            Callable<T> operation,
            int maxAttempts,
            long initialDelayMs,
            Class<? extends Exception>... retryableExceptions) {

        int attempt = 0;
        long delay = initialDelayMs;

        while (attempt < maxAttempts) {
            try {
                return operation.call();
            } catch (Exception e) {
                attempt++;

                // Check if this exception is retryable
                boolean isRetryable = Arrays.stream(retryableExceptions)
                    .anyMatch(retryable -> retryable.isInstance(e));

                if (!isRetryable || attempt >= maxAttempts) {
                    // Not retryable or max attempts reached
                    if (e instanceof RuntimeException re) throw re;
                    throw new RuntimeException("Operation failed after " + attempt + " attempts", e);
                }

                log.warn("Attempt {}/{} failed: {}. Retrying in {}ms...",
                    attempt, maxAttempts, e.getMessage(), delay);

                try {
                    Thread.sleep(delay + (long)(Math.random() * delay));  // jitter
                    delay = Math.min(delay * 2, 30_000);  // exponential backoff, max 30s
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Retry interrupted", ie);
                }
            }
        }
        throw new RuntimeException("Unreachable");
    }
}

// Usage:
User user = RetryHandler.withRetry(
    () -> userService.findUser(id),
    3,         // max attempts
    1000L,     // initial delay 1s
    IOException.class, SqlTransientConnectionException.class
);

// Spring's @Retryable (production approach):
@Retryable(
    value = {IOException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public User findUser(long id) throws IOException { ... }
```

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| Empty catch blocks | Always log or rethrow |
| `return` in `finally` | Only use `finally` for cleanup |
| Catching `Exception` too broadly | Catch specific exception types |
| Losing exception cause | Always pass `cause` to constructors |
| Not restoring interrupt flag | `Thread.currentThread().interrupt()` |
| `e.printStackTrace()` | Use a logger |
| Null from catch block | Throw or return empty collection |
| Exceptions for flow control | Use Optional, boolean checks, validation |
| Silent exception in scheduled tasks | Always catch Exception in scheduled tasks |
| Resource leaks | Use try-with-resources |

---

## ✅ Best Practices

- ✅ Use try-with-resources for all `AutoCloseable` resources
- ✅ Catch the most specific exception type
- ✅ Always preserve the original cause when wrapping
- ✅ Log at the boundary where exception is handled (not where thrown)
- ✅ Use unchecked exceptions for programming errors
- ✅ Restore interrupt flag when catching `InterruptedException`
- ✅ Design a global exception handler for REST APIs
- ✅ Include context (userId, requestId) in exception messages
- ✅ Return `Optional<>` or empty collections instead of `null`
- ✅ Test exception paths — write unit tests for error scenarios

---

## 🔗 Related Sections

- [Chapter 3: Multithreading](./03-multithreading-and-concurrency.md) — `InterruptedException`, thread exception handling
- [Chapter 4: Networking](./04-java-networking.md) — network exception handling
- [Chapter 6: Collections](./06-collections-framework.md) — `ConcurrentModificationException`

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 4: Networking](./04-java-networking.md) | [Section README](./README.md) | [Chapter 6: Collections →](./06-collections-framework.md) |
