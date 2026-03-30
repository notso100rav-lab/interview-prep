# Chapter 3: JVM Memory Tuning in Containers

Understanding how the JVM interacts with container memory limits is critical for Java engineers deploying to Docker and Kubernetes. Misconfigurations here cause the most common production incident for Java teams: `OOMKilled` pods.

---

## 1. The Core Problem: JVM + Containers Pre-Java 10

Before Java 10 (and the backport to Java 8u191), the JVM was **not container-aware**. It read memory information from `/proc/meminfo`, which reports the **host machine's total memory** — completely ignoring Docker/Kubernetes memory limits set via cgroups.

```
Host machine: 64 GB RAM
Container limit: 512 MB
JVM sees (pre-8u191): 64 GB → sets default heap to ~16 GB
JVM actually gets: 512 MB
Result: OOMKilled immediately on startup or shortly after
```

This caused countless production outages when teams moved Java services to containers without updating JVM flags.

---

## 2. The Fix: `-XX:+UseContainerSupport`

```
Enabled by default since: Java 10
Backported to: Java 8u191+
```

`UseContainerSupport` makes the JVM read memory limits from cgroup data (what the container actually has) rather than from `/proc/meminfo`.

```bash
# Verify it's active (default: true in modern JDKs)
java -XX:+PrintFlagsFinal -version | grep UseContainerSupport
# Output: bool UseContainerSupport = true  {product} {default}
```

```dockerfile
# If using Java 8 (pre-8u191), explicitly enable:
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-jar", "app.jar"]

# Java 11+ — UseContainerSupport is on by default, but be explicit for clarity
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

---

## 3. Heap Sizing: `-Xmx` vs `-XX:MaxRAMPercentage`

### The Old Way: `-Xmx` (Absolute)

```bash
java -Xmx512m -jar app.jar   # hard-coded heap limit: 512 MB
```

**Problem:** If you change the container memory limit (e.g., from 512 MB to 1 GB in Kubernetes), you must also update `-Xmx` — they become out of sync. Fragile in a cloud-native environment.

### The Container-Aware Way: `-XX:MaxRAMPercentage`

```bash
java -XX:MaxRAMPercentage=75.0 -jar app.jar
```

The JVM sets the max heap to **75% of the container's memory limit** (as reported by cgroups). This adapts automatically when you change the container's memory allocation.

```
Container limit 512 MB → MaxHeap = 384 MB
Container limit 1 GB   → MaxHeap = 768 MB
Container limit 2 GB   → MaxHeap = 1.5 GB
```

### Key Flags

| Flag | Description | Recommended Value |
|------|-------------|------------------|
| `-XX:MaxRAMPercentage` | Maximum heap as % of container RAM | `75.0` |
| `-XX:InitialRAMPercentage` | Initial heap as % of container RAM | `50.0` |
| `-XX:MinRAMPercentage` | Min heap for very small containers (<250 MB) | `50.0` |
| `-Xms` | Initial heap (absolute) — alternative to `InitialRAMPercentage` | — |
| `-Xmx` | Max heap (absolute) — alternative to `MaxRAMPercentage` | — |

---

## 4. Why 75%? Memory Beyond the Heap

The JVM uses memory beyond just the heap:

```
Total JVM memory = Heap + Metaspace + Code Cache + Thread Stacks + Native Memory

Container limit: 1 GB
  Heap (75%):          768 MB   ← MaxRAMPercentage=75.0
  Metaspace:           ~100 MB  ← class metadata
  JIT code cache:      ~50 MB   ← compiled native code
  Thread stacks:       ~30 MB   ← 1 MB per thread × ~30 threads
  Native/off-heap:     ~50 MB   ← DirectByteBuffers, NIO, etc.
  Total approx:        ~998 MB  ← just under limit
```

Setting `MaxRAMPercentage=75.0` leaves ~25% for non-heap memory. This is a safe starting point; adjust based on your app's actual memory profile.

**For memory-intensive non-heap usage** (e.g., heavy Netty/NIO, large Metaspace from many classes):
- Use `MaxRAMPercentage=60.0` to leave more headroom
- Set explicit `-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m` to cap metaspace growth

---

## 5. JVM Memory Areas in Detail

```
┌──────────────────────────────────────────────────────────┐
│                     JVM Process                          │
│  ┌─────────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │   Java Heap     │  │ Metaspace  │  │  Code Cache  │  │
│  │ (Objects, GC'd) │  │  (Classes) │  │ (JIT output) │  │
│  │  controlled by  │  │  grows     │  │  ~50MB       │  │
│  │  -Xmx/-Xms or   │  │  until     │  │              │  │
│  │  MaxRAMPercent  │  │  OOM       │  │              │  │
│  └─────────────────┘  └────────────┘  └──────────────┘  │
│  ┌─────────────────┐  ┌──────────────────────────────┐   │
│  │  Thread Stacks  │  │    Native Memory (off-heap)  │   │
│  │  ~1MB per thread│  │  DirectByteBuffers, NIO,    │   │
│  │                 │  │  JNI, GC bookkeeping         │   │
│  └─────────────────┘  └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Metaspace

```bash
# Cap Metaspace to prevent unbounded growth
-XX:MetaspaceSize=128m       # initial
-XX:MaxMetaspaceSize=256m    # cap

# View current Metaspace usage
jcmd <pid> VM.metaspace
```

Spring Boot apps with many beans, Hibernate entity classes, and reflection-heavy libraries can grow Metaspace significantly.

### Thread Stacks

Each Java thread uses ~1 MB of native memory by default. An application with 200 threads uses ~200 MB of native memory outside the heap.

```bash
-Xss512k   # reduce thread stack size (default 1MB) — use carefully
```

---

## 6. GC Selection for Containers

### G1GC (Default since Java 9)

```bash
java -XX:+UseG1GC -jar app.jar   # explicit (already default)
```

- Good all-around choice for heap sizes 2 GB and above
- Region-based, concurrent marking
- Default GC pause target: 200ms (`-XX:MaxGCPauseMillis=200`)
- Well-tuned for container environments since Java 10

### ZGC (Production-ready since Java 15)

```bash
java -XX:+UseZGC -jar app.jar
# Java 21+: enabled with Generational ZGC
java -XX:+UseZGC -XX:+ZGenerational -jar app.jar
```

- Sub-millisecond GC pauses (pause time doesn't scale with heap size)
- Scales to multi-terabyte heaps
- Trades slightly higher CPU overhead for much lower latency
- Ideal for: low-latency APIs, real-time services, heap sizes > 4 GB

### Shenandoah (RedHat-developed)

```bash
java -XX:+UseShenandoahGC -jar app.jar
```

- Similar goals to ZGC: concurrent compaction, low pauses
- Available in OpenJDK 15+, backported to some JDK 8/11 builds
- Good alternative to ZGC on non-OpenJDK distributions

### SerialGC (Small Containers)

```bash
java -XX:+UseSerialGC -jar app.jar
```

- Single-threaded GC — minimal overhead
- Best for containers with very small heaps (<256 MB) or single-CPU
- Java 14+ selects SerialGC automatically for very small heaps

### GC Selection Guide

| Heap Size | Latency Priority | Recommended GC |
|-----------|-----------------|----------------|
| < 256 MB | Any | SerialGC |
| 256 MB – 4 GB | Throughput | G1GC (default) |
| 256 MB – 4 GB | Low latency | ZGC or Shenandoah |
| > 4 GB | Any | ZGC or G1GC |

---

## 7. Inspecting Effective JVM Flags

```bash
# Print all final JVM flags (after defaults, command-line, ergonomics)
java -XX:+PrintFlagsFinal -version | grep -E "MaxHeap|InitialHeap|MaxRAM|GC"

# In a running container
docker exec myapp java -XX:+PrintFlagsFinal -version | grep MaxHeapSize

# Using jcmd (if jcmd is in the image)
docker exec myapp jcmd <pid> VM.flags

# Print only flags that differ from default
java -XX:+PrintCommandLineFlags -version
```

**Example output:**
```
uintx MaxHeapSize = 805306368   {product} {ergonomic}
# 805,306,368 bytes = 768 MB = 75% of 1 GB container
```

---

## 8. Setting JVM Flags in Docker

### Method 1: ENTRYPOINT in Dockerfile (Recommended)

```dockerfile
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-XX:+UseG1GC", \
  "-XX:MaxGCPauseMillis=200", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### Method 2: ENV JAVA_OPTS (Flexible, overridable)

```dockerfile
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

```bash
# Override at runtime
docker run -e JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=60.0" myapp:1.0.0
```

**Caveat:** Using `sh -c` wraps the JVM in a shell (PID 1 is `sh`), breaking graceful shutdown. Prefer a script or the exec form.

### Method 3: JVM Tool Options (Environment Variable)

```bash
# JVM reads this automatically
docker run -e JDK_JAVA_OPTIONS="-XX:MaxRAMPercentage=75.0" myapp:1.0.0
# JDK_JAVA_OPTIONS is the modern replacement for JAVA_TOOL_OPTIONS
```

### Method 4: entrypoint.sh script (Best of Both)

```bash
#!/bin/sh
exec java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=${MAX_RAM_PERCENTAGE:-75.0} \
  ${JAVA_OPTS} \
  -jar /app/app.jar
```

```dockerfile
COPY entrypoint.sh /app/
RUN chmod +x /app/entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]
```

`exec` replaces the shell with Java (so Java becomes PID 1 and receives signals).

---

## 9. OOMKilled: Cause and Fix

### What Happens

```
1. Container memory limit: 512 MB (set in K8s pod spec)
2. JVM heap: 400 MB + Metaspace: 150 MB + other: 50 MB = 600 MB total
3. Container exceeds limit → Linux OOM Killer sends SIGKILL
4. Pod exits with exit code 137
5. Kubernetes shows: OOMKilled: true
```

### Diagnosing

```bash
# Kubernetes
kubectl describe pod <pod-name>
# Look for:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

kubectl top pod <pod-name>   # current memory usage
```

### Fix Checklist

```dockerfile
# 1. Enable container support
-XX:+UseContainerSupport

# 2. Use percentage-based heap
-XX:MaxRAMPercentage=75.0

# 3. Cap Metaspace
-XX:MaxMetaspaceSize=256m

# 4. Cap code cache (if needed)
-XX:ReservedCodeCacheSize=128m

# 5. Reduce thread stack size (if many threads)
-Xss256k
```

```yaml
# 6. Set appropriate K8s resource limits
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "1"
```

**Rule of thumb:** Set requests == limits for memory (Guaranteed QoS class in Kubernetes).

---

## 10. Kubernetes Resources and JVM Interaction

```yaml
# pod.yaml
containers:
- name: myapp
  image: myapp:1.0.0
  resources:
    requests:
      memory: "512Mi"   # scheduler uses this for placement
      cpu: "250m"
    limits:
      memory: "512Mi"   # OOMKill if exceeded
      cpu: "1000m"      # throttled if exceeded (not killed)
  env:
  - name: JAVA_OPTS
    value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
```

**CPU limits and GC:** With `UseContainerSupport`, the JVM also reads the CPU limit and adjusts the number of GC threads accordingly. This prevents GC thread explosion in CPU-constrained environments.

```bash
# With 1 CPU limit, JVM adjusts GC threads (avoids spawning 32 GC threads on 32-core host)
-XX:+UseContainerSupport   # automatically adjusts GC and JIT threads to CPU quota
```

---

## 11. Complete Recommended Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn dependency:go-offline -B
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn package -DskipTests -B

FROM eclipse-temurin:21-jre-jammy AS runtime

# Non-root user
RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup appuser

WORKDIR /app

# Install curl for health checks only (remove if using distroless or K8s HTTP probes)
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/target/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

USER appuser

EXPOSE 8080

# JVM memory tuning flags
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-XX:MaxMetaspaceSize=256m", \
  "-XX:+UseG1GC", \
  "-XX:MaxGCPauseMillis=200", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

---

## Q&A

### Q1 🟢 What is `-XX:+UseContainerSupport` and when was it introduced?

<details><summary>Click to reveal answer</summary>

`-XX:+UseContainerSupport` makes the JVM container-aware — it reads memory and CPU limits from **cgroup data** (what the container actually has) instead of `/proc/meminfo` (the host machine's total resources).

**Timeline:**
- Java 10: Introduced, enabled by default
- Java 8u191: Backported, enabled by default
- Before 8u191: Not available — must use absolute `-Xmx`

**Verify it's active:**
```bash
java -XX:+PrintFlagsFinal -version | grep UseContainerSupport
# bool UseContainerSupport = true  {product} {default}
```

Without this flag on older JVMs, a container with 512 MB memory limit might have a JVM that thinks it has 64 GB and tries to allocate a 16 GB heap — causing immediate OOMKill.

</details>

---

### Q2 🟡 What is the difference between `-Xmx` and `-XX:MaxRAMPercentage`?

<details><summary>Click to reveal answer</summary>

**`-Xmx`** sets an absolute maximum heap size:
```bash
-Xmx512m    # max heap = 512 MB regardless of container size
-Xmx2g      # max heap = 2 GB
```

**`-XX:MaxRAMPercentage`** sets heap as a percentage of the container's available memory:
```bash
-XX:MaxRAMPercentage=75.0   # max heap = 75% of container limit
```

| Aspect | `-Xmx` | `MaxRAMPercentage` |
|--------|--------|-------------------|
| Container-aware | ❌ (fixed) | ✅ (adapts) |
| Cloud-native | ❌ | ✅ |
| Requires reconfiguration on resize | ✅ | ❌ |
| Predictable value | ✅ | Depends on container size |

**Recommendation:** Use `MaxRAMPercentage` in containerized environments. Use `-Xmx` only when you need a precise, fixed heap size and the container memory will never change.

</details>

---

### Q3 🟢 What does exit code 137 mean for a Docker container?

<details><summary>Click to reveal answer</summary>

Exit code 137 = **128 + 9** = container received **SIGKILL** (signal 9).

SIGKILL from the Linux OOM Killer means the container **exceeded its memory limit** and was forcefully killed.

```bash
docker ps -a
# STATUS: Exited (137) 2 minutes ago

kubectl describe pod myapp-pod
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

**Diagnosis steps:**
1. Check `kubectl describe pod` for `OOMKilled: true`
2. Review JVM flags — is `UseContainerSupport` enabled? Is `MaxRAMPercentage` set?
3. Check actual memory usage: `kubectl top pod`
4. Increase container memory limit OR reduce `MaxRAMPercentage` to leave more headroom for non-heap memory

</details>

---

### Q4 🟡 Why is 75% a commonly recommended value for `MaxRAMPercentage`?

<details><summary>Click to reveal answer</summary>

The JVM uses memory beyond the heap:

- **Metaspace** (~100–200 MB for a Spring Boot app with many classes)
- **JIT code cache** (~50–100 MB)
- **Thread stacks** (~1 MB × number of threads)
- **Native/off-heap** (DirectByteBuffers, NIO, GC bookkeeping)

Setting `MaxRAMPercentage=75.0` reserves **25% of container memory** for non-heap JVM overhead. For a 1 GB container:

```
Heap:    768 MB (75%)
Others:  ~200-250 MB (25%)
Total:   ~968-1018 MB → safely within 1 GB
```

**Adjust downward** to 60–65% if:
- Your app uses many threads (many thread stacks)
- Heavy use of NIO/Netty (large off-heap buffers)
- Large Metaspace (many classes, heavy reflection frameworks)

</details>

---

### Q5 🟡 What JVM memory areas exist beyond the heap?

<details><summary>Click to reveal answer</summary>

1. **Metaspace** — class metadata (class definitions, method bytecode, constant pools). Replaced PermGen in Java 8. Grows dynamically; cap with `-XX:MaxMetaspaceSize`.

2. **Code Cache** — stores JIT-compiled native code. Cap with `-XX:ReservedCodeCacheSize`. Usually 50–250 MB.

3. **Thread Stacks** — each thread gets a native memory stack (~1 MB by default). 100 threads = ~100 MB of native memory.

4. **Native Memory** — DirectByteBuffers, JNI, GC overhead, class loading structures, file descriptors. Can be significant in I/O-heavy apps (Netty, NIO).

5. **Off-heap (Direct)** — explicitly allocated with `ByteBuffer.allocateDirect()` or via libraries like Chronicle Map. Not garbage collected by GC.

```bash
# Native Memory Tracking (NMT)
java -XX:NativeMemoryTracking=summary -jar app.jar
jcmd <pid> VM.native_memory summary
```

</details>

---

### Q6 🔴 A Spring Boot pod is OOMKilled after running for 2 hours. The heap is only at 60% when monitoring shows the kill. What could be causing this?

<details><summary>Click to reveal answer</summary>

If the heap is within limits but the pod is OOMKilled, the issue is **non-heap memory growth**:

**Likely culprits:**

1. **Metaspace leak** — dynamic class generation (reflection, Groovy scripts, CGLIB proxies, Hibernate) creating new classes without unloading old ones.
   ```bash
   # Check Metaspace usage over time
   jcmd <pid> VM.metaspace
   # Fix: -XX:MaxMetaspaceSize=256m  (will cause OOM inside JVM instead of OOMKill)
   ```

2. **Thread leak** — unbounded thread pool or virtual threads accumulating.
   ```bash
   jcmd <pid> Thread.print | grep -c java.lang.Thread
   ```

3. **Native memory leak** — DirectByteBuffers not being released (common with Netty, gRPC).
   ```bash
   java -XX:NativeMemoryTracking=summary -jar app.jar
   jcmd <pid> VM.native_memory summary scale=MB
   ```

4. **GC metadata growth** — G1GC remembered sets growing with fragmented heap.

**Fix strategy:**
- Enable NMT: `-XX:NativeMemoryTracking=summary`
- Add `-XX:MaxMetaspaceSize=256m` to get a proper OOM error instead of silent OOMKill
- Increase container memory limit and monitor actual footprint
- Analyze heap dump + thread dump at the time of incident

</details>

---

### Q7 🟡 When would you choose ZGC over G1GC?

<details><summary>Click to reveal answer</summary>

**Choose ZGC when:**
- Your application requires **sub-millisecond GC pauses** (payment processing, real-time APIs, gaming)
- Heap size is **large (4 GB+)** — G1GC pause times scale with heap size; ZGC does not
- You're on **Java 15+** (production-ready) or Java 21+ for Generational ZGC

**Stick with G1GC when:**
- **Throughput** is the priority over latency
- Heap is small-to-medium (< 4 GB)
- You want the proven, well-understood default

```bash
# ZGC with Generational mode (Java 21+ - recommended)
java -XX:+UseZGC -XX:+ZGenerational -XX:MaxRAMPercentage=75.0 -jar app.jar

# G1GC with tuning
java -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:MaxRAMPercentage=75.0 -jar app.jar
```

**Trade-off:** ZGC uses slightly more CPU and memory overhead to achieve concurrent compaction. In CPU-constrained containers, this overhead may be noticeable.

</details>

---

### Q8 🟢 How do you verify the JVM is seeing the correct container memory limit?

<details><summary>Click to reveal answer</summary>

```bash
# Option 1: Print MaxHeapSize and compare with expected 75% of container limit
docker run --memory=512m myapp:1.0.0 \
  java -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 \
       -XX:+PrintFlagsFinal -version | grep MaxHeapSize
# Expected: MaxHeapSize = 402653184 (384 MB = 75% of 512 MB)

# Option 2: Add a startup log in application.properties
# logging.level.org.springframework.boot=INFO
# The JVM logs MaxHeap at startup with -verbose:gc or actuator/env

# Option 3: Check via Spring Boot Actuator
curl http://localhost:8080/actuator/metrics/jvm.memory.max

# Option 4: Exec into running container
docker exec myapp java -XX:+PrintFlagsFinal -version | grep -E "MaxHeap|Initial"
```

If `MaxHeapSize` reflects the **host** memory (e.g., 25% of 64 GB = 16 GB) instead of 75% of 512 MB, then `UseContainerSupport` is not active or the JVM is too old.

</details>

---

### Q9 🟡 What is `-Djava.security.egd=file:/dev/./urandom` and why is it commonly added?

<details><summary>Click to reveal answer</summary>

This flag changes the entropy source for Java's `SecureRandom` from `/dev/random` to `/dev/urandom`.

**Problem:** In containers, `/dev/random` blocks if the entropy pool is exhausted — which happens frequently in containerized environments with limited hardware entropy sources. This causes **slow application startup** (sometimes 30+ seconds for cryptographic initialization).

```bash
-Djava.security.egd=file:/dev/./urandom
# Note: /dev/./urandom (with extra /) — a quirk to bypass JDK's hardcoded /dev/urandom check
```

**In Java 8+:** Modern JDKs on Linux default to `NativePRNG` which uses `/dev/urandom` already. This flag is more relevant for older JDKs or environments with strict security configurations.

**Alternative (Java 9+):**
```bash
-Djava.security.egd=file:/dev/urandom   # works in newer JDKs
```

Add this to your Dockerfile's ENTRYPOINT for safety, especially in environments running on cloud VMs or containers with shared entropy pools.

</details>

---

### Q10 🔴 Explain the interaction between Kubernetes CPU limits and JVM GC thread count.

<details><summary>Click to reveal answer</summary>

The JVM uses the available CPU count to determine the number of GC threads:

```
Default GC threads = max(5, available_cpus × 5/8)

On a 32-core host with no CPU limit:  JVM sees 32 CPUs → 20 GC threads
In a container with 1 CPU limit:      JVM should see 1 CPU → 1 GC thread
```

**Without `UseContainerSupport`:** The JVM reads host CPU count (32) and spawns 20 GC threads in a container with a 1-CPU limit. This causes severe CPU throttling — GC becomes a bottleneck as threads fight for a single CPU.

**With `UseContainerSupport` (default Java 10+):** The JVM reads the CPU limit from the cgroup CPU quota and adjusts thread counts accordingly.

```bash
# Explicitly set GC threads if needed
-XX:ParallelGCThreads=2     # for parallel phases
-XX:ConcGCThreads=1         # for concurrent G1GC/ZGC phases

# Verify
java -XX:+PrintFlagsFinal -version | grep GCThreads
```

**Kubernetes note:** CPU requests != limits. The JVM may see the limit value. Set requests close to limits to ensure consistent JVM behavior.

</details>

---

### Q11 🟡 What is Metaspace and how does it differ from the old PermGen?

<details><summary>Click to reveal answer</summary>

**PermGen (Java 7 and earlier):** A fixed-size region of the heap where class metadata was stored. Caused the infamous `java.lang.OutOfMemoryError: PermGen space` when too many classes were loaded. Set with `-XX:MaxPermSize=256m`.

**Metaspace (Java 8+):** Stores class metadata in **native memory** (outside the JVM heap). By default, it grows without limit — preventing PermGen-style errors but potentially causing native OOM.

```bash
# PermGen (legacy — don't use in Java 8+)
-XX:PermSize=128m -XX:MaxPermSize=256m

# Metaspace (Java 8+)
-XX:MetaspaceSize=128m      # initial size (avoids resize overhead at startup)
-XX:MaxMetaspaceSize=256m   # cap to prevent unbounded growth
```

**In containers:** Without `-XX:MaxMetaspaceSize`, a Metaspace leak can grow until the container OOMKills the process — even though the heap is fine. Always set `-XX:MaxMetaspaceSize` in containerized environments.

</details>

---

### Q12 🟢 How do you set JVM flags through environment variables in Docker?

<details><summary>Click to reveal answer</summary>

**Option 1: `JDK_JAVA_OPTIONS`** (preferred, modern standard)
```bash
docker run -e JDK_JAVA_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC" myapp:1.0.0
```

The JVM reads this automatically — no changes to Dockerfile needed.

**Option 2: `JAVA_TOOL_OPTIONS`** (older standard, still works)
```bash
docker run -e JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0" myapp:1.0.0
```

Note: This prints a message: `Picked up JAVA_TOOL_OPTIONS: ...` — can be noisy in logs.

**Option 3: Custom `JAVA_OPTS` with shell entrypoint**
```dockerfile
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]
```

```bash
docker run -e JAVA_OPTS="-XX:MaxRAMPercentage=60.0" myapp:1.0.0
```

`exec` is important — it replaces the shell with Java so Java becomes PID 1.

</details>

---

### Q13 🟡 What GC does the JVM use by default in Java 21, and is it appropriate for containers?

<details><summary>Click to reveal answer</summary>

**G1GC** is the default GC in Java 9 through Java 21 (for heap sizes ≥ a few hundred MB). For very small heaps (<~256 MB), Java 14+ ergonomically selects **SerialGC**.

G1GC is appropriate for most containerized workloads:
- Region-based, concurrent marking — good pause time control
- `UseContainerSupport` ensures G1GC thread counts are container-aware
- Default pause target: 200ms (`-XX:MaxGCPauseMillis=200`) — adjust for lower latency

For **Java 21+**, Generational ZGC is available and production-ready for latency-sensitive workloads:
```bash
java -XX:+UseZGC -XX:+ZGenerational -XX:MaxRAMPercentage=75.0 -jar app.jar
```

Virtual threads (Project Loom, Java 21) interact well with G1GC but benefit from ZGC's low-pause behavior in high-throughput scenarios.

</details>

---

### Q14 🔴 How would you diagnose a Java application that is running slowly in a container but fine on bare metal?

<details><summary>Click to reveal answer</summary>

**Suspect list (container-specific issues):**

1. **CPU throttling** — check if container CPU limit is too low
   ```bash
   cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled
   # throttled_time should be near 0
   ```

2. **JVM not container-aware** — JVM spawning too many GC/JIT threads for a CPU-limited container
   ```bash
   java -XX:+PrintFlagsFinal -version | grep "Active"
   # ActiveProcessorCount should match CPU limit, not host CPU count
   ```

3. **GC pressure** — heap too small, GC running too frequently
   ```bash
   java -Xlog:gc*:stdout:time -jar app.jar
   # Look for frequent GC events or long pause times
   ```

4. **Entropy starvation** — crypto operations blocking on `/dev/random`
   ```bash
   # Add -Djava.security.egd=file:/dev/./urandom
   ```

5. **Disk I/O throttling** — container storage backend (overlay2) is slower for temp files
   ```bash
   # Move temp files to a memory-backed volume (tmpfs)
   docker run --tmpfs /tmp myapp:1.0.0
   ```

6. **Network latency** — service mesh, NAT traversal, DNS resolution slower in container overlay network

**Profiling approach:**
```bash
# Attach async-profiler in container
docker exec -it myapp /bin/bash
# Run async-profiler, JFR, or export JMX metrics
```

</details>

---

### Q15 🟡 What is `-XX:+PrintFlagsFinal` and when would you use it?

<details><summary>Click to reveal answer</summary>

`-XX:+PrintFlagsFinal` prints all JVM flags and their **effective values** (after applying defaults, command-line args, and ergonomic adjustments) at JVM startup.

```bash
# One-shot inspection
java -XX:+PrintFlagsFinal -version | grep -E "MaxHeap|InitialHeap|GC|RAM"

# In a running container — check what the app actually got
docker run --rm --memory=512m myapp:1.0.0 \
  java -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 \
       -XX:+PrintFlagsFinal -version 2>&1 | grep MaxHeapSize
```

Key flags to verify:
```
MaxHeapSize               → should be 75% of container limit
UseContainerSupport       → should be true
UseG1GC / UseZGC          → should reflect your chosen GC
ActiveProcessorCount      → should reflect CPU limit, not host CPU count
MaxMetaspaceSize          → should be set (not MaxJavaHeap)
```

Use this to **verify** that your JVM flags are having the intended effect, especially after OS/JDK upgrades that may change ergonomic defaults.

</details>

---

### Q16 🟢 What is the difference between Kubernetes memory `requests` and `limits`, and how do they affect the JVM?

<details><summary>Click to reveal answer</summary>

**Requests:** The minimum memory guaranteed for scheduling. The scheduler places the pod on a node with at least this much free memory. The JVM is NOT directly affected by requests.

**Limits:** The maximum memory the container can use. The Linux cgroup enforces this hard cap. The JVM **reads this value** via `UseContainerSupport` to set heap size.

```yaml
resources:
  requests:
    memory: "512Mi"   # scheduler guarantee
  limits:
    memory: "512Mi"   # OOMKill if exceeded; JVM reads this
```

With `MaxRAMPercentage=75.0` and a `512Mi` limit → JVM max heap = 384 MB.

**Burstable vs Guaranteed QoS:**
- `requests < limits` → Burstable (can use up to limits, but preempted when node is under pressure)
- `requests == limits` → Guaranteed (not preempted, more predictable JVM behavior)

**Recommendation for Java:** Set `requests == limits` for memory. This gives Guaranteed QoS and ensures the JVM always has a consistent, predictable memory ceiling.

</details>

---

### Q17 🟡 How do virtual threads (Java 21) affect JVM memory tuning in containers?

<details><summary>Click to reveal answer</summary>

Virtual threads (Project Loom, Java 21) are lightweight threads scheduled by the JVM, not the OS. Key memory implications:

**Reduced native memory:** Traditional OS threads use ~1 MB of native memory each. Virtual threads are very cheap — millions can exist simultaneously with minimal native memory overhead.

```java
// Before (thread per request = 200 OS threads = 200 MB native memory)
ExecutorService pool = Executors.newFixedThreadPool(200);

// After (virtual threads = millions possible, ~KB each)
ExecutorService pool = Executors.newVirtualThreadPerTaskExecutor();
```

**Implications for container tuning:**
- Less native memory needed for thread stacks → can reduce memory limit or reclaim for heap
- Watch for carrier thread pinning (synchronized blocks): pinned virtual threads consume a carrier OS thread
- G1GC and ZGC both work well with virtual threads; ZGC's low pause times complement Loom's high-concurrency model

**New monitoring:** Virtual thread count appears in JVM metrics — use Spring Boot Actuator or JFR to monitor.

</details>

---

### Q18 🟡 Show a complete JVM flags example for a latency-sensitive Spring Boot microservice running in a 1 GB container.

<details><summary>Click to reveal answer</summary>

```dockerfile
ENTRYPOINT ["java", \
  # Container awareness
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=70.0", \
  "-XX:InitialRAMPercentage=50.0", \
  # GC: ZGC for low latency (Java 21+)
  "-XX:+UseZGC", \
  "-XX:+ZGenerational", \
  # Metaspace cap
  "-XX:MaxMetaspaceSize=200m", \
  # Code cache
  "-XX:ReservedCodeCacheSize=128m", \
  # Startup: tiered compilation
  "-XX:TieredStopAtLevel=4", \
  # Entropy
  "-Djava.security.egd=file:/dev/./urandom", \
  # Logging
  "-Xlog:gc*:stdout:time,level,tags", \
  # JFR for profiling (low overhead)
  "-XX:StartFlightRecording=duration=0,filename=/tmp/app.jfr,settings=profile", \
  "-jar", "app.jar"]
```

With a 1 GB limit:
- Heap: ~700 MB (70%)
- Metaspace: ≤200 MB
- Code cache: ≤128 MB
- Remaining for threads, NIO, ZGC overhead: ~150+ MB

**Trade-off:** ZGC has higher memory overhead than G1GC — especially with `ZGenerational`. For 1 GB containers, test under load to confirm no OOMKill.

</details>

---

### Q19 🔴 What is Native Memory Tracking (NMT) and how do you use it to investigate memory leaks?

<details><summary>Click to reveal answer</summary>

NMT records all JVM native memory allocations categorized by usage (heap, metaspace, code cache, thread stacks, etc.).

```bash
# Enable (low overhead: summary, higher overhead: detail)
java -XX:NativeMemoryTracking=summary -jar app.jar

# Take a baseline snapshot
jcmd <pid> VM.native_memory baseline

# After some time, see the diff
jcmd <pid> VM.native_memory summary.diff scale=MB
```

**Sample output:**
```
Native Memory Tracking:

Total: reserved=2GB, committed=512MB
-              Java Heap (reserved=384MB, committed=384MB)
-          Class (reserved=300MB, committed=80MB)     ← Metaspace
-               Code (reserved=128MB, committed=48MB)  ← JIT code
-             Thread (reserved=50MB, committed=50MB)
-               GC (reserved=30MB, committed=30MB)
-     Internal (reserved=5MB, committed=5MB)
-      Compiler (reserved=1MB, committed=1MB)
```

If `Class` (Metaspace) or `Thread` keeps growing in the diff output → memory leak.

**Enable in Docker:**
```dockerfile
ENTRYPOINT ["java", \
  "-XX:NativeMemoryTracking=summary", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

Note: NMT has ~5–10% overhead. Use in staging, not high-traffic production unless investigating an incident.

</details>

---

### Q20 🟡 How do you configure InitialRAMPercentage and why does it matter?

<details><summary>Click to reveal answer</summary>

`-XX:InitialRAMPercentage` sets the **initial heap size** as a percentage of container memory. It mirrors what `-Xms` does but in a container-aware way.

```bash
java -XX:InitialRAMPercentage=50.0 -XX:MaxRAMPercentage=75.0 -jar app.jar

# For a 1 GB container:
# Initial heap = 512 MB (50%)
# Max heap = 768 MB (75%)
```

**Why it matters:**

1. **Startup time:** A larger initial heap means fewer heap expansions during startup (GC doesn't need to grow the heap repeatedly). This is important for containers that must pass readiness probes quickly.

2. **Memory reservation:** On Kubernetes, if the container's initial footprint is much smaller than the limit, the cluster may schedule the pod on an undersized node (if using Burstable QoS). Starting with a reasonable initial heap avoids heap expansion surprises.

3. **GC efficiency:** Starting too small causes many early GC cycles as the heap tries to grow to accommodate the workload.

**Rule of thumb:** Set `InitialRAMPercentage = MaxRAMPercentage / 1.5` as a starting point, then tune based on your startup memory profile.

</details>
