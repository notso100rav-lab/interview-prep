# Chapter 3: Multithreading & Concurrency

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- Thread lifecycle and states
- Creating threads: `Thread`, `Runnable`, `Callable`
- `synchronized` keyword (method and block)
- `volatile` keyword
- `ReentrantLock` and `ReadWriteLock`
- `ExecutorService` and thread pools
- `CompletableFuture` and async programming
- `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`
- `ForkJoinPool` and work-stealing
- `ThreadLocal`
- Deadlock, livelock, starvation detection and prevention
- Atomic classes (`AtomicInteger`, `LongAdder`, etc.)
- `BlockingQueue` and producer-consumer pattern

---

## 🔑 Key Concepts at a Glance

```
Thread Lifecycle:
  NEW → RUNNABLE → (BLOCKED/WAITING/TIMED_WAITING) → TERMINATED

Thread Pool (ExecutorService):
  Task Queue → Worker Threads → Results
  
  Fixed Pool: fixed N threads
  Cached Pool: grow/shrink as needed
  Scheduled Pool: run at delay/period
  ForkJoin Pool: work-stealing for divide-and-conquer

Concurrency Primitives:
  synchronized → mutual exclusion (monitor lock)
  volatile     → visibility guarantee (no caching)
  ReentrantLock → flexible locking (tryLock, conditions)
  AtomicInteger → lock-free CAS operations
```

---

## 🧠 Theory Questions

### Thread Basics

---

🟢 **Q1. What are the different ways to create a thread in Java?**

<details><summary>Click to reveal answer</summary>

**Method 1: Extend Thread class**
```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}
MyThread t = new MyThread();
t.start();  // NOT t.run() — that would execute synchronously!
```

**Method 2: Implement Runnable (preferred)**
```java
Runnable task = () -> System.out.println("Runnable running");
Thread t = new Thread(task, "my-thread");
t.start();
```

**Method 3: Implement Callable (can return value)**
```java
Callable<Integer> callable = () -> {
    Thread.sleep(100);
    return 42;
};
FutureTask<Integer> future = new FutureTask<>(callable);
new Thread(future).start();
Integer result = future.get(); // blocks until done
```

**Method 4: ExecutorService (preferred for production)**
```java
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> future = executor.submit(() -> computeValue());
executor.submit(() -> doFireAndForget());
executor.shutdown();
```

**Why prefer `Runnable` over extending `Thread`?**
- Single inheritance: Extending Thread prevents extending another class
- Separation of concerns: Task logic separate from thread management
- Reusability: Same Runnable can run on different executors

> 💡 **Interviewer Tip:** In production, you should almost never create raw threads. Use `ExecutorService` for lifecycle management, thread pooling, and proper shutdown.

</details>

---

🟢 **Q2. What is the difference between `start()` and `run()` in Thread?**

<details><summary>Click to reveal answer</summary>

| | `start()` | `run()` |
|-|----------|---------|
| **Creates new thread?** | ✅ Yes | ❌ No |
| **Executes in** | New thread | Calling thread (synchronous) |
| **Can call twice?** | ❌ IllegalThreadStateException | ✅ Yes |

```java
Thread t = new Thread(() -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
});

t.start(); // Prints: "Running in: Thread-0"
t.run();   // Prints: "Running in: main" — no new thread created!
```

🚨 **Common Mistake:** Calling `t.run()` instead of `t.start()`. No compile error, but no parallelism — runs synchronously on the current thread.

</details>

---

🟡 **Q3. Describe the complete thread lifecycle (states).**

<details><summary>Click to reveal answer</summary>

```
                    ┌──────────────────────────────────────────┐
                    │              Thread States                 │
                    └──────────────────────────────────────────┘

   new Thread()         t.start()        scheduled by OS
  ┌───────────┐      ┌──────────┐      ┌──────────────┐
  │    NEW    │ ───► │ RUNNABLE │ ◄──► │   RUNNING    │
  └───────────┘      └──────────┘      └──────────────┘
                           ▲                  │
                           │    ┌─────────────┴──────────────┐
                           │    ▼                             ▼
                      ┌─────────┐   lock unavail    ┌────────────────┐
                      │  BLOCKED│ ◄───────────────  │  wait()/join() │
                      └─────────┘                   │    WAITING     │
                           │    synchronized block  └────────────────┘
                           │    becomes available          │
                           └────────────────────────────────┘
                                                            │  run() completes
                                                            ▼
                                                    ┌──────────────┐
                                                    │  TERMINATED  │
                                                    └──────────────┘
```

**States explained:**
| State | When | How to exit |
|-------|------|-------------|
| `NEW` | After `new Thread()`, before `start()` | Call `start()` |
| `RUNNABLE` | After `start()`, ready to run or running | OS scheduler picks it |
| `BLOCKED` | Waiting for a `synchronized` lock | Lock becomes available |
| `WAITING` | `wait()`, `join()`, `LockSupport.park()` | `notify()`, `notifyAll()`, `unpark()`, thread terminates |
| `TIMED_WAITING` | `sleep(n)`, `wait(n)`, `join(n)` | Timeout expires or notified |
| `TERMINATED` | `run()` returns or throws exception | (final state) |

```java
Thread t = new Thread(() -> {
    try { Thread.sleep(5000); } catch (InterruptedException e) { }
});

System.out.println(t.getState()); // NEW
t.start();
System.out.println(t.getState()); // RUNNABLE or TIMED_WAITING
t.join();
System.out.println(t.getState()); // TERMINATED
```

</details>

---

### synchronized and volatile

---

🟡 **Q4. How does the `synchronized` keyword work?**

<details><summary>Click to reveal answer</summary>

`synchronized` provides **mutual exclusion** (only one thread at a time) and **memory visibility** (changes are flushed to main memory).

**1. Synchronized method — locks on `this` (instance method) or `Class` (static method):**
```java
public class Counter {
    private int count = 0;

    // Instance method: locks on 'this'
    public synchronized void increment() {
        count++;
    }

    // Static method: locks on Counter.class
    public static synchronized void staticMethod() { ... }
}
```

**2. Synchronized block — lock on specified object:**
```java
public class Cache {
    private final Map<String, Object> cache = new HashMap<>();
    private final Object lock = new Object();  // dedicated lock object

    public Object get(String key) {
        synchronized (lock) {
            return cache.get(key);
        }
    }

    public void put(String key, Object value) {
        synchronized (lock) {
            cache.put(key, value);
        }
    }
}
```

**How it works internally:**
- Each object has an associated **monitor** (intrinsic lock / mutex)
- Thread that enters `synchronized` acquires the monitor
- Other threads trying to enter block until monitor is released
- On exit (normal or exception), monitor is released

**Reentrancy:**
```java
public synchronized void outer() {
    inner();  // Same thread can re-acquire same lock without deadlock
}
public synchronized void inner() {
    System.out.println("inner");  // works because lock is reentrant
}
```

> 💡 **Performance Note:** `synchronized` is more expensive than no synchronization but much cheaper than it used to be (JVM has biased locking, lock elision via escape analysis).

</details>

---

🟡 **Q5. What is the `volatile` keyword and what does it guarantee?**

<details><summary>Click to reveal answer</summary>

`volatile` guarantees **visibility** of writes to all threads, but does NOT guarantee **atomicity**.

**The visibility problem:**
```java
// Without volatile:
class LoopThread extends Thread {
    private boolean stop = false;  // May be cached in CPU register/cache!

    public void run() {
        while (!stop) {  // Thread might NEVER see stop=true
            doWork();
        }
    }

    public void stopLoop() { stop = true; }
}

// With volatile:
private volatile boolean stop = false;
// Now: write to 'stop' is flushed to main memory immediately
//      read of 'stop' always reads from main memory
```

**What volatile provides:**
1. **Visibility:** Writes visible to all threads immediately
2. **Ordering:** Prevents instruction reordering around volatile reads/writes
3. **64-bit atomicity:** Atomic reads/writes for `long` and `double` (otherwise may be non-atomic on 32-bit JVMs)

**What volatile does NOT provide:**
```java
volatile int count = 0;

// Thread 1 and Thread 2 both:
count++;  // NOT ATOMIC! Still: read → increment → write (three ops)
         // Two threads can both read 5, both increment to 6, lose an update
```

**Comparison:**

| | `volatile` | `synchronized` |
|-|-----------|----------------|
| Visibility | ✅ | ✅ |
| Atomicity | ❌ | ✅ |
| Mutual exclusion | ❌ | ✅ |
| Performance | Better | Worse |

**Correct use cases for volatile:**
- Simple flags (start/stop signals)
- Double-checked locking (with `synchronized`)
- Publishing immutable objects
- Single writer, multiple readers

</details>

---

🔴 **Q6. What is the happens-before relationship in the context of `volatile`?**

<details><summary>Click to reveal answer</summary>

The **happens-before** guarantee for `volatile` ensures that all writes before a volatile write are visible to all reads after the volatile read.

```java
int x = 0;
int y = 0;
volatile boolean initialized = false;

// Thread 1:
x = 10;          // happens-before the volatile write
y = 20;          // happens-before the volatile write
initialized = true;  // volatile write

// Thread 2:
if (initialized) {  // volatile read
    // GUARANTEED to see x=10, y=20
    System.out.println(x);  // always 10 (not 0)
    System.out.println(y);  // always 20 (not 0)
}
```

**Why this matters:** Without `volatile`, Thread 2 might see `initialized=true` but stale values of `x=0` or `y=0` due to CPU caching or instruction reordering.

**Practical use — safe publication:**
```java
public class Publisher {
    private int data;
    private volatile boolean dataReady = false;

    // Thread 1: producer
    public void publish(int value) {
        data = value;        // writes data
        dataReady = true;    // volatile write — acts as a memory barrier
    }

    // Thread 2: consumer
    public void consume() {
        while (!dataReady) { }  // volatile read — spins until data is ready
        System.out.println(data);  // guaranteed to see the latest data
    }
}
```

</details>

---

### Locks and ReentrantLock

---

🟡 **Q7. What is `ReentrantLock` and how does it differ from `synchronized`?**

<details><summary>Click to reveal answer</summary>

`ReentrantLock` (from `java.util.concurrent.locks`) provides more flexible locking than `synchronized`.

```java
import java.util.concurrent.locks.*;

public class Cache {
    private final ReentrantLock lock = new ReentrantLock();
    private final Map<String, Object> data = new HashMap<>();

    public void put(String key, Object value) {
        lock.lock();
        try {
            data.put(key, value);
        } finally {
            lock.unlock();  // MUST unlock in finally!
        }
    }

    // tryLock — non-blocking attempt
    public boolean tryPut(String key, Object value) {
        if (lock.tryLock()) {
            try {
                data.put(key, value);
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // lock not acquired, don't block
    }

    // tryLock with timeout
    public boolean tryPutWithTimeout(String key, Object value) throws InterruptedException {
        if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
            try {
                data.put(key, value);
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

**Feature Comparison:**

| Feature | `synchronized` | `ReentrantLock` |
|---------|---------------|----------------|
| Syntax | Simple, intrinsic | Explicit lock/unlock |
| `tryLock()` | ❌ | ✅ |
| Timeout | ❌ | ✅ |
| Interruptible wait | ❌ | ✅ `lockInterruptibly()` |
| Fairness policy | ❌ | ✅ `new ReentrantLock(true)` |
| Multiple conditions | ❌ | ✅ `lock.newCondition()` |
| Can cause lock leak? | ❌ (auto-released) | ✅ (must `unlock()` in finally) |
| JVM optimization | ✅ (biased locking, etc.) | Limited |

**When to use ReentrantLock:**
- Need `tryLock()` or timeout
- Need interruptible waiting
- Need fair queuing
- Need multiple wait conditions

</details>

---

🔴 **Q8. Explain `ReadWriteLock` and when to use it.**

<details><summary>Click to reveal answer</summary>

`ReentrantReadWriteLock` allows **multiple concurrent readers** but **exclusive writer access**.

```java
public class Cache<K, V> {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private final Map<K, V> map = new HashMap<>();

    public V get(K key) {
        readLock.lock();  // Multiple threads can hold read lock simultaneously
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(K key, V value) {
        writeLock.lock();  // Exclusive — all readers and writers blocked
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}

// When read lock is held: other reads OK, writes blocked
// When write lock is held: all reads and writes blocked
```

**`StampedLock` (Java 8+) — even better for read-heavy workloads:**
```java
public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // Optimistic read — NO lock acquired (fastest path)
    double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();
        double cx = x, cy = y;
        if (!lock.validate(stamp)) {  // Check if write occurred
            stamp = lock.readLock();  // Fall back to read lock
            try { cx = x; cy = y; }
            finally { lock.unlockRead(stamp); }
        }
        return Math.sqrt(cx * cx + cy * cy);
    }

    void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try { x += deltaX; y += deltaY; }
        finally { lock.unlockWrite(stamp); }
    }
}
```

</details>

---

### ExecutorService and Thread Pools

---

🟡 **Q9. What is `ExecutorService` and what thread pool types are available?**

<details><summary>Click to reveal answer</summary>

`ExecutorService` manages a pool of worker threads, abstracting thread creation and lifecycle.

```java
// 1. Fixed thread pool — N threads, tasks queue if all busy
ExecutorService fixed = Executors.newFixedThreadPool(4);

// 2. Cached thread pool — grows/shrinks as needed (idle threads expire after 60s)
ExecutorService cached = Executors.newCachedThreadPool();

// 3. Single thread executor — single thread, tasks execute serially
ExecutorService single = Executors.newSingleThreadExecutor();

// 4. Scheduled executor — run at delay/interval
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.schedule(task, 5, TimeUnit.SECONDS);          // once after 5s
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS); // every 1s
scheduled.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS); // 1s AFTER last completion

// 5. Work-stealing pool (ForkJoinPool)
ExecutorService workStealing = Executors.newWorkStealingPool();
```

**Submitting work:**
```java
// Runnable (no return value)
executor.execute(runnable);           // fire-and-forget
Future<?> f = executor.submit(runnable); // can check completion

// Callable (returns value)
Future<Integer> future = executor.submit(callable);
Integer result = future.get();        // blocks until done
Integer result2 = future.get(5, TimeUnit.SECONDS);  // with timeout

// Batch submission
List<Callable<Integer>> tasks = List.of(...);
List<Future<Integer>> futures = executor.invokeAll(tasks);  // all must complete
Integer first = executor.invokeAny(tasks);  // returns first completed result
```

**Proper shutdown:**
```java
executor.shutdown();  // graceful: accepts no new tasks, waits for running ones
if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
    executor.shutdownNow();  // forceful: interrupts running tasks
}
```

> 💡 **Interviewer Tip:** In production (Spring Boot etc.), avoid `Executors.newCachedThreadPool()` for unbounded task submission — it can create unlimited threads and cause OOM. Prefer `ThreadPoolExecutor` with explicit bounds.

</details>

---

🔴 **Q10. How do you configure a custom `ThreadPoolExecutor`?**

<details><summary>Click to reveal answer</summary>

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize: always-alive threads
    8,                              // maximumPoolSize: max threads when queue full
    60L, TimeUnit.SECONDS,          // keepAliveTime: idle time before extra thread dies
    new ArrayBlockingQueue<>(100),  // workQueue: bounded queue (100 tasks)
    new ThreadFactory() {           // threadFactory: custom thread naming
        int count = 0;
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "worker-" + count++);
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);

// Rejection policies when queue is full AND maxPoolSize threads busy:
// AbortPolicy (default)   → throws RejectedExecutionException
// CallerRunsPolicy        → caller thread runs the task (provides backpressure)
// DiscardPolicy           → silently drops the task
// DiscardOldestPolicy     → drops oldest queued task, retries submission
```

**How tasks are scheduled:**
```
Task submitted:
  1. If active threads < corePoolSize  → create new thread
  2. Else if queue not full           → add to queue
  3. Else if active threads < maxPool  → create new thread
  4. Else                             → apply rejection policy
```

**Queue types matter:**
```java
new LinkedBlockingQueue<>()        // unbounded (dangerous — can OOM)
new ArrayBlockingQueue<>(100)      // bounded (recommended)
new SynchronousQueue<>()           // no buffering, direct handoff (like cached pool)
new PriorityBlockingQueue<>()      // priority-ordered tasks
```

</details>

---

### CompletableFuture

---

🟡 **Q11. What is `CompletableFuture` and how does it improve on `Future`?**

<details><summary>Click to reveal answer</summary>

`CompletableFuture` (Java 8+) provides non-blocking, composable async programming.

**Problems with `Future`:**
```java
Future<String> future = executor.submit(() -> fetchData());
String result = future.get();  // BLOCKS the calling thread!
// Can't chain operations, can't handle errors elegantly
```

**CompletableFuture advantages:**
```java
// 1. Non-blocking async execution
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> fetchData());

// 2. Chain transformations
CompletableFuture<Integer> length = cf
    .thenApply(s -> s.toUpperCase())    // transform result
    .thenApply(String::length);          // another transformation

// 3. Chain with another async operation
CompletableFuture<String> result = cf
    .thenCompose(data -> CompletableFuture.supplyAsync(() -> process(data)));

// 4. Combine two independent futures
CompletableFuture<String> user = CompletableFuture.supplyAsync(() -> fetchUser(id));
CompletableFuture<String> order = CompletableFuture.supplyAsync(() -> fetchOrder(id));

CompletableFuture<String> combined = user.thenCombine(order,
    (u, o) -> "User: " + u + ", Order: " + o);

// 5. Handle errors
CompletableFuture<String> safe = cf
    .exceptionally(ex -> "default-value")
    .handle((result2, ex) -> ex != null ? "error" : result2);

// 6. Wait for all to complete
CompletableFuture.allOf(cf1, cf2, cf3).join();

// 7. First to complete wins
CompletableFuture.anyOf(cf1, cf2, cf3).thenAccept(System.out::println);
```

**Practical example — parallel service calls:**
```java
public UserProfile buildProfile(long userId) {
    CompletableFuture<User> userFuture =
        CompletableFuture.supplyAsync(() -> userService.getUser(userId));

    CompletableFuture<List<Order>> ordersFuture =
        CompletableFuture.supplyAsync(() -> orderService.getOrders(userId));

    CompletableFuture<List<Review>> reviewsFuture =
        CompletableFuture.supplyAsync(() -> reviewService.getReviews(userId));

    return CompletableFuture.allOf(userFuture, ordersFuture, reviewsFuture)
        .thenApply(v -> new UserProfile(
            userFuture.join(),
            ordersFuture.join(),
            reviewsFuture.join()
        ))
        .join();  // blocks final result
}
// userService, orderService, reviewService calls happen in parallel!
```

**Specifying executor (avoid using common ForkJoinPool for blocking ops):**
```java
ExecutorService ioExecutor = Executors.newFixedThreadPool(20);
CompletableFuture.supplyAsync(() -> callDatabase(), ioExecutor);
```

</details>

---

### Concurrency Utilities

---

🟡 **Q12. What is `CountDownLatch` and when do you use it?**

<details><summary>Click to reveal answer</summary>

`CountDownLatch` allows one or more threads to wait until a set of operations completes. The count is initialized and decremented; when it reaches zero, all waiting threads proceed.

**Use case:** Wait for N service initializations before starting the application.

```java
public class ServiceStartup {
    public static void main(String[] args) throws InterruptedException {
        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);

        // Start services concurrently
        List.of("DatabaseService", "CacheService", "MessageQueue").forEach(service ->
            new Thread(() -> {
                try {
                    System.out.println(service + " starting...");
                    Thread.sleep((long)(Math.random() * 2000));
                    System.out.println(service + " ready!");
                    latch.countDown();  // decrement count
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }).start()
        );

        System.out.println("Waiting for all services...");
        latch.await();  // blocks until count == 0
        // OR: latch.await(10, TimeUnit.SECONDS); // with timeout

        System.out.println("All services ready! Starting application...");
    }
}
```

**Key properties:**
- Count can only go down (cannot be reset)
- For one-time synchronization only
- Multiple threads can call `await()` simultaneously
- `countDown()` never blocks

**vs `CyclicBarrier`:**
| | `CountDownLatch` | `CyclicBarrier` |
|-|----------------|----------------|
| Reusable? | ❌ One-time | ✅ Can be reset |
| Who counts down | Any thread | Participating threads |
| Who waits | External threads | The N threads themselves |
| Use case | Wait for N events | N threads meet at a barrier |

</details>

---

🟡 **Q13. What is a `Semaphore` and how is it used?**

<details><summary>Click to reveal answer</summary>

`Semaphore` controls access to a resource by maintaining a count of permits.

```java
// Limit concurrent database connections to 10
public class ConnectionPool {
    private final Semaphore semaphore = new Semaphore(10, true); // 10 permits, fair

    public Connection acquire() throws InterruptedException {
        semaphore.acquire();  // blocks if no permits available
        return createConnection();
    }

    public void release(Connection conn) {
        closeConnection(conn);
        semaphore.release();  // returns a permit
    }

    // Non-blocking attempt
    public Optional<Connection> tryAcquire() {
        if (semaphore.tryAcquire()) {
            return Optional.of(createConnection());
        }
        return Optional.empty();
    }
}

// Rate limiter example (N requests per second)
public class RateLimiter {
    private final Semaphore semaphore;
    private final ScheduledExecutorService scheduler;

    public RateLimiter(int requestsPerSecond) {
        this.semaphore = new Semaphore(requestsPerSecond);
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        // Release permits every second
        scheduler.scheduleAtFixedRate(
            () -> semaphore.release(requestsPerSecond - semaphore.availablePermits()),
            1, 1, TimeUnit.SECONDS
        );
    }

    public void acquirePermit() throws InterruptedException {
        semaphore.acquire();
    }
}
```

**Special case — mutex (1 permit = binary semaphore):**
```java
Semaphore mutex = new Semaphore(1);
mutex.acquire();  // enter critical section
try {
    // critical section
} finally {
    mutex.release();
}
```

</details>

---

🔴 **Q14. What is `ForkJoinPool` and how does work-stealing work?**

<details><summary>Click to reveal answer</summary>

`ForkJoinPool` is designed for divide-and-conquer problems. It uses **work-stealing** where idle threads steal tasks from busy threads' queues.

```java
// RecursiveTask — returns a value
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] array;
    private final int start, end;

    public SumTask(long[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Base case: compute directly
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }

        // Divide and conquer
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);

        left.fork();              // submit left to ForkJoinPool asynchronously
        long rightResult = right.compute();  // compute right directly
        long leftResult = left.join();       // wait for left result

        return leftResult + rightResult;
    }
}

// Usage
ForkJoinPool pool = new ForkJoinPool(Runtime.getRuntime().availableProcessors());
long[] array = new long[1_000_000];
Arrays.fill(array, 1L);
long total = pool.invoke(new SumTask(array, 0, array.length));
// total = 1,000,000

// Common ForkJoinPool (used by parallel streams):
ForkJoinPool.commonPool().submit(() -> { ... });
```

**Work-stealing mechanism:**
```
Thread 1 Queue: [task8, task7, task6]  ← Thread 1 pops from head
Thread 2 Queue: []                     ← Thread 2 is idle
Thread 3 Queue: [task5, task4, task3]  ← Thread 3 pops from head

Thread 2 steals from tail of Thread 3:
Thread 2 Queue: [task3]                ← Thread 2 now working

Why steal from tail? Larger (coarser) tasks are at the tail → less overhead
```

> 💡 **Interviewer Tip:** Java 8+ parallel streams use the common ForkJoinPool. Be careful — blocking I/O in parallel streams starves the common pool, affecting all users. Use a custom ForkJoinPool for I/O-bound parallel stream work.

</details>

---

### ThreadLocal

---

🟡 **Q15. What is `ThreadLocal` and when should you use it?**

<details><summary>Click to reveal answer</summary>

`ThreadLocal` provides thread-local storage — each thread has its own independent copy of the variable.

```java
// Classic use case: per-thread database connection / session
public class DatabaseContext {
    private static final ThreadLocal<Connection> connectionHolder =
        ThreadLocal.withInitial(() -> createNewConnection());

    public static Connection getConnection() {
        return connectionHolder.get();
    }

    public static void removeConnection() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            conn.close();
            connectionHolder.remove();  // CRITICAL: prevent memory leaks in thread pools!
        }
    }
}

// Per-request context in web apps
public class RequestContext {
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();
    private static final ThreadLocal<String> requestId = new ThreadLocal<>();

    public static void set(User user, String reqId) {
        currentUser.set(user);
        requestId.set(reqId);
    }

    public static User getCurrentUser() { return currentUser.get(); }
    public static String getRequestId() { return requestId.get(); }

    public static void clear() {
        currentUser.remove();
        requestId.remove();
    }
}

// In a servlet filter:
public class RequestFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        RequestContext.set(extractUser(req), generateRequestId());
        try {
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear();  // MUST clear to prevent memory leaks in thread pools
        }
    }
}
```

**⚠️ ThreadLocal memory leak in thread pools:**
```java
// Thread pool threads are reused — ThreadLocal values persist between requests!
// If you set a ThreadLocal in a task but don't remove() it:
// - Next task on same thread sees old value (correctness bug)
// - Held values never GC'd (memory leak)

// ALWAYS remove() ThreadLocal values when done (use finally block)
```

</details>

---

### Deadlock, Livelock, Starvation

---

🔴 **Q16. What is a deadlock? How do you detect and prevent it?**

<details><summary>Click to reveal answer</summary>

**Deadlock:** Two or more threads are permanently blocked, each waiting for a lock held by another.

**Four conditions for deadlock (Coffman conditions):**
1. **Mutual exclusion:** Resources cannot be shared
2. **Hold and wait:** Thread holds one resource while waiting for another
3. **No preemption:** Resources cannot be forcibly taken
4. **Circular wait:** Thread A waits for B, B waits for A (cycle)

**Classic deadlock example:**
```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        System.out.println("T1: locked lock1");
        Thread.sleep(100);  // pause so T2 can grab lock2
        synchronized (lock2) {  // waits for T2 to release lock2
            System.out.println("T1: locked lock2");
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock2) {
        System.out.println("T2: locked lock2");
        Thread.sleep(100);
        synchronized (lock1) {  // waits for T1 to release lock1
            System.out.println("T1: locked lock1");
        }
    }
});
// DEADLOCK: T1 holds lock1 waiting for lock2, T2 holds lock2 waiting for lock1
```

**Prevention strategies:**

**1. Lock ordering — always acquire locks in same order:**
```java
// Both threads lock lock1 first, then lock2
Thread t1 = new Thread(() -> {
    synchronized (lock1) { synchronized (lock2) { /* work */ } }
});
Thread t2 = new Thread(() -> {
    synchronized (lock1) { synchronized (lock2) { /* work */ } }  // same order!
});
```

**2. `tryLock` with timeout:**
```java
Lock lock1 = new ReentrantLock();
Lock lock2 = new ReentrantLock();

boolean locked = false;
while (!locked) {
    boolean l1 = lock1.tryLock(100, TimeUnit.MILLISECONDS);
    boolean l2 = lock2.tryLock(100, TimeUnit.MILLISECONDS);
    if (l1 && l2) {
        locked = true;
        try { /* work */ }
        finally { lock1.unlock(); lock2.unlock(); }
    } else {
        if (l1) lock1.unlock();  // release what we got, retry
        if (l2) lock2.unlock();
    }
}
```

**3. Use higher-level concurrency utilities:**
```java
// Prefer ConcurrentHashMap, BlockingQueue, etc. over raw synchronized
```

**Detection:**
```bash
# JVM thread dump analysis
kill -3 <pid>                    # send SIGQUIT to get thread dump
jstack <pid>                     # Java stack trace with deadlock detection
jconsole / VisualVM              # GUI tools
ThreadMXBean.findDeadlockedThreads()  # programmatic
```

</details>

---

🔴 **Q17. What is livelock and starvation? How do they differ from deadlock?**

<details><summary>Click to reveal answer</summary>

**Deadlock:** Threads are blocked, making no progress.

**Livelock:** Threads are active but making no useful progress — they keep responding to each other.

```java
// Livelock example: Two polite people in a hallway trying to let each other pass
class Person {
    private String name;
    private boolean isMovingLeft;

    void passBy(Person other) {
        while (isBlocking(other)) {
            System.out.println(name + " steps aside to let " + other.name + " pass");
            // Move to opposite side
            isMovingLeft = !isMovingLeft;
            Thread.sleep(50);
            // But other person does the same thing!
        }
    }
}
// Both keep moving but never actually pass each other
```

**Starvation:** Thread is perpetually denied access to resources because other threads are always preferred.

```java
// Starvation with unfair locking:
ReentrantLock lock = new ReentrantLock(false);  // UNFAIR lock
// Under high contention, some threads may rarely get the lock

// Fix: Use fair lock
ReentrantLock fairLock = new ReentrantLock(true);  // FIFO ordering

// Starvation with priority:
Thread lowPriority = new Thread(...);
lowPriority.setPriority(Thread.MIN_PRIORITY);  // may be starved
```

| | Deadlock | Livelock | Starvation |
|-|---------|---------|-----------|
| Thread state | BLOCKED/WAITING | RUNNABLE | BLOCKED/WAITING |
| Progress | None | None (busy) | None (not selected) |
| CPU usage | Low | High | Low |
| Resolution | Break circular wait | Add randomness/backoff | Fair scheduling |

</details>

---

### Atomic Classes

---

🟡 **Q18. What are atomic classes and how do they work without locks?**

<details><summary>Click to reveal answer</summary>

Atomic classes in `java.util.concurrent.atomic` provide **lock-free thread-safe operations** using **Compare-And-Swap (CAS)** CPU instructions.

**CAS operation:** "If current value == expected, set it to new value atomically"

```java
// AtomicInteger operations
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();        // atomic ++
counter.getAndIncrement();        // atomic x++ (return old)
counter.addAndGet(5);             // atomic += 5
counter.compareAndSet(10, 20);    // CAS: if value==10, set to 20

// AtomicReference — CAS on object references
AtomicReference<String> ref = new AtomicReference<>("initial");
ref.compareAndSet("initial", "updated");  // atomic reference swap

// AtomicLong, AtomicBoolean similar

// LongAdder — better than AtomicLong under high contention
LongAdder adder = new LongAdder();
adder.increment();        // updates internal cell, reduces contention
adder.add(5);
long total = adder.sum(); // combines all cells at read time
// Useful for counters with many writers, few readers
```

**CAS internals (simplified):**
```
int compareAndSwap(int* addr, int expected, int newValue) {
    if (*addr == expected) {
        *addr = newValue;
        return expected;  // success
    }
    return *addr;  // failure — try again
}

// Java AtomicInteger.incrementAndGet():
int current;
do {
    current = get();
} while (!compareAndSet(current, current + 1));  // retry if another thread changed it
return current + 1;
```

**When to use which:**
| Scenario | Tool |
|---------|------|
| Single counter, low contention | `AtomicInteger`/`AtomicLong` |
| Counter, high contention | `LongAdder` |
| Complex state update | `synchronized` or `ReentrantLock` |
| Read-heavy cache | `ConcurrentHashMap` with `computeIfAbsent` |

</details>

---

## 🎭 Scenario-Based Questions

---

🔴 **S1. Implement a thread-safe bounded blocking queue.**

<details><summary>Click to reveal answer</summary>

```java
public class BoundedBlockingQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // wait until space available
            }
            queue.add(item);
            notEmpty.signalAll();  // notify waiting consumers
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // wait until item available
            }
            T item = queue.poll();
            notFull.signalAll();  // notify waiting producers
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try { return queue.size(); }
        finally { lock.unlock(); }
    }
}
```

> Note: `java.util.concurrent.ArrayBlockingQueue` provides this built-in.

</details>

---

🔴 **S2. What's wrong with this double-checked locking?**

```java
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

<details><summary>Click to reveal answer</summary>

The problem: `instance` is not `volatile`, allowing CPU/compiler instruction reordering.

`instance = new Singleton()` involves three steps:
1. Allocate memory
2. Initialize the `Singleton` object
3. Assign address to `instance`

Without `volatile`, steps 2 and 3 can be reordered. Thread A might set `instance` to a non-null but **not fully initialized** object (step 3 before step 2). Thread B reads `instance != null` (first check passes) and returns a partially constructed object!

**Fix:**
```java
private static volatile Singleton instance;  // volatile prevents reordering!
```

`volatile` establishes a happens-before relationship — all writes before the volatile write are visible after the volatile read.

</details>

---

🟡 **S3. What happens when you call `wait()` outside a synchronized block?**

```java
Object lock = new Object();
lock.wait();  // What happens?
```

<details><summary>Click to reveal answer</summary>

Throws `IllegalMonitorStateException` at runtime.

`wait()`, `notify()`, and `notifyAll()` can only be called from **within a synchronized block or method** — the thread must own the object's monitor.

```java
// CORRECT usage:
synchronized (lock) {
    while (condition not met) {
        lock.wait();   // releases lock and waits
    }
    // condition is now met, do work
}

// Awakening thread:
synchronized (lock) {
    // change condition
    lock.notifyAll();  // wake all waiting threads
}
```

**Why use `while` not `if` for condition check?**
- **Spurious wakeups:** `wait()` can return without `notify()` being called
- **Multiple consumers:** After `notifyAll()`, multiple threads wake up — only one should proceed

</details>

---

🟡 **S4. What's the output? (Thread interleaving)**

```java
public class Counter {
    static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) count++;
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join(); t2.join();

        System.out.println(count);
    }
}
```

<details><summary>Click to reveal answer</summary>

The output is **non-deterministic** — could be any value from 1000 to 2000.

Because `count++` is not atomic (read-increment-write), threads can interleave and lose updates.

**Possible scenario:**
```
T1 reads count=500
T2 reads count=500
T1 writes count=501
T2 writes count=501  ← update lost!
```

**Fix options:**
```java
// 1. AtomicInteger
static AtomicInteger count = new AtomicInteger(0);
// task: count.incrementAndGet();

// 2. synchronized
static synchronized void increment() { count++; }

// 3. LongAdder (best for high-contention counters)
static LongAdder count = new LongAdder();
// task: count.increment();
// result: count.sum();
```

</details>

---

🔴 **S5. Implement a thread-safe Singleton using enum.**

<details><summary>Click to reveal answer</summary>

```java
public enum DatabaseConnectionSingleton {
    INSTANCE;

    private final Connection connection;

    DatabaseConnectionSingleton() {
        // Constructor called once by JVM during enum class loading
        try {
            this.connection = DriverManager.getConnection(
                System.getenv("DB_URL"),
                System.getenv("DB_USER"),
                System.getenv("DB_PASSWORD")
            );
        } catch (SQLException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public Connection getConnection() { return connection; }

    public static DatabaseConnectionSingleton getInstance() {
        return INSTANCE;  // thread-safe, serialization-safe, reflection-safe
    }
}

// Usage:
Connection conn = DatabaseConnectionSingleton.INSTANCE.getConnection();
```

**Why enum singleton is best:**
1. **Thread-safe:** JVM guarantees enum values are initialized once
2. **Serialization-safe:** Enum deserialization doesn't create new instances
3. **Reflection-safe:** Cannot create enum instances via reflection
4. **Concise:** Least boilerplate

</details>

---

🔴 **S6. How do you implement a producer-consumer pattern?**

<details><summary>Click to reveal answer</summary>

```java
public class ProducerConsumerDemo {
    private static final BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
    private static final AtomicBoolean done = new AtomicBoolean(false);

    static class Producer implements Runnable {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 20; i++) {
                    System.out.println("Producing: " + i);
                    queue.put(i);  // blocks if queue is full
                    Thread.sleep(50);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.set(true);
            }
        }
    }

    static class Consumer implements Runnable {
        @Override
        public void run() {
            try {
                while (!done.get() || !queue.isEmpty()) {
                    Integer item = queue.poll(100, TimeUnit.MILLISECONDS);
                    if (item != null) {
                        System.out.println("Consuming: " + item);
                        Thread.sleep(100);  // consumer is slower
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(4);
        executor.submit(new Producer());
        // Multiple consumers
        executor.submit(new Consumer());
        executor.submit(new Consumer());
        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);
    }
}
```

</details>

---

🟡 **S7. What is the difference between `sleep()` and `wait()`?**

<details><summary>Click to reveal answer</summary>

| | `Thread.sleep(ms)` | `Object.wait()` |
|-|-------------------|--------------| 
| **Class** | `Thread` (static) | `Object` |
| **Releases lock?** | ❌ No | ✅ Yes |
| **Wake up** | After timeout | `notify()`/`notifyAll()` or timeout |
| **Synchronized required?** | ❌ No | ✅ Yes |
| **Use case** | Pause execution | Inter-thread communication |
| **Interruptible?** | ✅ Yes | ✅ Yes |

```java
// sleep: thread holds any locks it has
synchronized (lock) {
    Thread.sleep(1000);  // still holds 'lock', other threads blocked!
}

// wait: releases the lock while waiting
synchronized (lock) {
    lock.wait(1000);  // releases 'lock', other threads can enter!
    // re-acquires lock when notified or timeout
}
```

🚨 **Common Mistake:** Using `Thread.sleep()` inside synchronized blocks when you intended to allow other threads to proceed — use `wait()` instead.

</details>

---

🔴 **S8. What is `CyclicBarrier` and how does it differ from `CountDownLatch`?**

<details><summary>Click to reveal answer</summary>

`CyclicBarrier` makes N threads wait for each other at a common barrier point, then all proceed together (can be reused).

```java
public class PhaseBasedComputation {
    private static final int NUM_THREADS = 4;

    public static void main(String[] args) throws InterruptedException {
        // Barrier action runs when all threads arrive
        CyclicBarrier barrier = new CyclicBarrier(NUM_THREADS, () -> {
            System.out.println("All threads reached barrier — proceeding to next phase!");
        });

        ExecutorService executor = Executors.newFixedThreadPool(NUM_THREADS);
        for (int i = 0; i < NUM_THREADS; i++) {
            final int threadId = i;
            executor.submit(() -> {
                try {
                    // Phase 1
                    System.out.println("Thread " + threadId + ": Phase 1 processing");
                    Thread.sleep((long)(Math.random() * 1000));
                    barrier.await();  // wait for all threads

                    // Phase 2 — starts only after ALL threads finish Phase 1
                    System.out.println("Thread " + threadId + ": Phase 2 processing");
                    Thread.sleep((long)(Math.random() * 1000));
                    barrier.await();  // reuse barrier!

                    System.out.println("Thread " + threadId + ": Done");
                } catch (Exception e) { Thread.currentThread().interrupt(); }
            });
        }
        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);
    }
}
```

**Key difference:**
- `CountDownLatch`: One or more external observers wait for N events from any threads
- `CyclicBarrier`: N threads synchronize with each other (meet at barrier), reusable

</details>

---

🔴 **S9. Explain `CompletableFuture.thenCompose` vs `thenApply`.**

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "hello");

// What's the difference?
cf.thenApply(s -> s.toUpperCase());
cf.thenCompose(s -> CompletableFuture.supplyAsync(() -> s.toUpperCase()));
```

<details><summary>Click to reveal answer</summary>

- **`thenApply(Function<T, R>)`:** Transforms the result synchronously. Returns `CompletableFuture<R>`. Like `Stream.map()`.

- **`thenCompose(Function<T, CompletableFuture<R>>)`:** Chains another async operation. Returns `CompletableFuture<R>` (flattened). Like `Stream.flatMap()`.

```java
// thenApply: for synchronous transformations
CompletableFuture<String> upper = cf.thenApply(String::toUpperCase);
// Type: CompletableFuture<String>

// thenApply with async function: WRONG — creates nested future
CompletableFuture<CompletableFuture<String>> nested =
    cf.thenApply(s -> CompletableFuture.supplyAsync(() -> s.toUpperCase()));
// Type: CompletableFuture<CompletableFuture<String>> — awkward!

// thenCompose: flattens the nested future
CompletableFuture<String> composed =
    cf.thenCompose(s -> CompletableFuture.supplyAsync(() -> s.toUpperCase()));
// Type: CompletableFuture<String> — correct!

// Practical example:
CompletableFuture<User> userFuture = getUserAsync(userId);
CompletableFuture<List<Order>> ordersFuture =
    userFuture.thenCompose(user -> getOrdersAsync(user.getId()));
// Sequential async: get user THEN get their orders
```

</details>

---

🔴 **S10. What is the `happens-before` guarantee in Java and why does it matter?**

<details><summary>Click to reveal answer</summary>

`happens-before` is the formal guarantee in the Java Memory Model that ensures memory visibility between threads.

**Without happens-before:** Thread A's writes may not be visible to Thread B due to CPU caching, write buffers, or instruction reordering.

```java
int x = 0;
boolean ready = false;

// Thread A:
x = 42;         // write x
ready = true;   // write ready (may be reordered before x=42!)

// Thread B:
while (!ready) {}  // spin
System.out.println(x);  // might print 0! (x=42 not visible yet)
```

**Establishes happens-before:**
1. `synchronized` block exit → subsequent acquisition of same lock
2. `volatile` write → subsequent `volatile` read of same field
3. `Thread.start()` → first action in started thread
4. Thread termination → `Thread.join()` returning
5. `CountDownLatch.countDown()` → `await()` returning

```java
// Fix using volatile:
volatile boolean ready = false;

// Thread A:
x = 42;       // happens-before volatile write
ready = true; // volatile write

// Thread B:
while (!ready) {}  // volatile read
System.out.println(x);  // guaranteed to see 42
```

> 💡 **Interviewer Tip:** Mentioning happens-before shows deep JMM knowledge. Many senior candidates can't explain this precisely. It shows you understand WHY synchronized/volatile work, not just HOW to use them.

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| Calling `t.run()` instead of `t.start()` | Always use `start()` to create a new thread |
| `volatile` for compound operations | Use `AtomicInteger` or `synchronized` |
| Forgetting `unlock()` in `finally` | Always unlock in `finally` block |
| `if` instead of `while` for `wait()` | Always use `while` to handle spurious wakeups |
| ThreadLocal without `remove()` in thread pools | Always `remove()` in `finally` to prevent leaks |
| Double-checked locking without `volatile` | Declare the instance field as `volatile` |
| `synchronized` on different objects | Ensure same lock object protects same data |
| Blocking I/O in common ForkJoinPool | Use a separate executor for I/O-bound tasks |
| Catching and ignoring `InterruptedException` | Call `Thread.currentThread().interrupt()` to restore flag |
| Unbounded thread pools (`newCachedThreadPool`) | Use bounded `ThreadPoolExecutor` |

---

## ✅ Best Practices

- ✅ Prefer `ExecutorService` over raw `Thread` creation
- ✅ Use `AtomicInteger`/`LongAdder` for counters instead of `synchronized int`
- ✅ Use `ConcurrentHashMap` instead of `Collections.synchronizedMap()`
- ✅ Use `BlockingQueue` for producer-consumer patterns
- ✅ Always restore interrupted status: `Thread.currentThread().interrupt()`
- ✅ Use `CompletableFuture` for composable async pipelines
- ✅ Always shut down `ExecutorService` gracefully
- ✅ Minimize synchronized block scope (only protect critical section)
- ✅ Use `ThreadLocal.remove()` in `finally` for thread pool contexts
- ✅ Profile before optimizing concurrency — premature optimization is costly

---

## 🔗 Related Sections

- [Chapter 1: Java Basics](./01-java-basics.md) — JVM memory model
- [Chapter 2: OOP](./02-oops.md) — thread-safe singleton design patterns
- [Chapter 6: Collections](./06-collections-framework.md) — `ConcurrentHashMap`, `CopyOnWriteArrayList`

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 2: OOP](./02-oops.md) | [Section README](./README.md) | [Chapter 4: Networking →](./04-java-networking.md) |
