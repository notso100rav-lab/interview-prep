# Chapter 4: Common Docker Production Mistakes

Production incidents are the best teacher — but other people's incidents are even better. This chapter covers the 10 most common Docker mistakes Java backend teams make in production, with concrete WRONG ❌ vs CORRECT ✅ examples for each.

---

## Mistake 1: Running as Root

### ❌ WRONG — Container runs as root

```dockerfile
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
# No USER instruction → runs as root (uid 0) by default
```

```bash
docker run myapp:1.0.0 whoami
# root  ← attacker has root access if they escape the container
```

### ✅ CORRECT — Run as a non-root system user

```dockerfile
FROM eclipse-temurin:21-jre-jammy

WORKDIR /app

# Create a system user with no login shell and no home directory
RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup --no-create-home appuser

COPY target/app.jar app.jar

# Switch to non-root user before the ENTRYPOINT
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

```bash
docker run myapp:1.0.0 whoami
# appuser  ← minimal privilege principle
```

**Why it matters:** If an attacker exploits a vulnerability in your app and escapes the container, running as root on the host gives them full system access. A non-root user limits blast radius significantly.

**Additional security layers:**
```bash
docker run --read-only --tmpfs /tmp myapp:1.0.0    # read-only filesystem
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp:1.0.0  # minimal capabilities
```

---

## Mistake 2: Using the `:latest` Tag

### ❌ WRONG — Mutable, non-reproducible tag

```dockerfile
FROM eclipse-temurin:latest    # what version is "latest" today? Tomorrow?

# In deployment
image: myapp:latest            # Kubernetes may not pull updated image
```

```bash
docker pull myapp:latest  # Different image on day 1 vs day 30 — silent breakage
```

### ✅ CORRECT — Pin specific, immutable versions

```dockerfile
# Pin to a specific version
FROM eclipse-temurin:21.0.5_11-jre-jammy

# Or pin to a digest (most immutable — the SHA never changes)
FROM eclipse-temurin@sha256:3f5f09c47a856c78e6e9f0e2c9e1e2a1f23bbf456890abcdef1234567890abcd
```

```bash
# Tag your own images with semantic version AND git SHA
docker build \
  -t myapp:1.4.2 \
  -t myapp:$(git rev-parse --short HEAD) \
  -t myapp:latest \    # can also tag latest, but deploy using sha/version
  .

# Deploy using the immutable tag
kubectl set image deployment/myapp app=myapp:1.4.2
# OR
kubectl set image deployment/myapp app=myapp:a3b7c9d
```

**Why it matters:**
- `:latest` is mutable — the same tag can point to completely different images
- Kubernetes `imagePullPolicy: IfNotPresent` (default) won't re-pull if `:latest` already exists locally
- No rollback story — you can't "roll back to the previous latest"

---

## Mistake 3: JDK in Production Image

### ❌ WRONG — Full JDK in the runtime image

```dockerfile
FROM eclipse-temurin:21-jdk-jammy   # JDK = JRE + compiler + tools + ~200MB extra
COPY target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```
Image size: ~420 MB — includes javac, jlink, jshell, jmap, jstack, etc.
Attack surface: all those extra tools are available to an attacker
```

### ✅ CORRECT — JRE only (or distroless) in production

```dockerfile
# Option A: JRE-only image
FROM eclipse-temurin:21-jre-jammy   # JRE only, ~220 MB — no compiler
COPY target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]

# Option B: Distroless (no shell, no tools, smallest attack surface)
FROM gcr.io/distroless/java21-debian12
COPY target/app.jar app.jar
CMD ["app.jar"]

# Option C: Multi-stage build (build with JDK, run with JRE)
FROM eclipse-temurin:21-jdk-jammy AS builder
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-jammy AS runtime
COPY --from=builder /build/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| Image | Size | Has `javac`? | Has shell? |
|-------|------|-------------|-----------|
| JDK | ~420 MB | ✅ | ✅ |
| JRE | ~220 MB | ❌ | ✅ |
| Distroless | ~170 MB | ❌ | ❌ |

---

## Mistake 4: Missing HEALTHCHECK

### ❌ WRONG — No health check

```dockerfile
FROM eclipse-temurin:21-jre-jammy
COPY target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
# Docker and load balancers have no way to know if the app is actually healthy
```

```bash
docker ps
# STATUS: Up 2 minutes  ← Up doesn't mean healthy; app might be in deadlock
```

### ✅ CORRECT — Add HEALTHCHECK with appropriate timing

```dockerfile
FROM eclipse-temurin:21-jre-jammy

RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

COPY target/app.jar app.jar

# --start-period: grace period after container starts (allow for slow JVM startup)
# --interval: check frequency
# --timeout: time before check is considered failed
# --retries: failed checks before marking unhealthy
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

```bash
docker ps
# STATUS: Up 2 minutes (healthy)  ← now Docker knows the app is ready
```

**Spring Boot Actuator setup:**
```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
# application.properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when-authorized
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true
```

**Kubernetes probes (preferred over HEALTHCHECK in K8s):**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

---

## Mistake 5: No Graceful Shutdown

### ❌ WRONG — Abrupt shutdown drops in-flight requests

```properties
# No graceful shutdown configuration
# Spring Boot default: immediate shutdown on SIGTERM
```

```dockerfile
# Shell form — SIGTERM goes to /bin/sh, not the JVM
CMD java -jar app.jar
# docker stop → /bin/sh gets SIGTERM, JVM gets SIGKILL after timeout
```

### ✅ CORRECT — Configure graceful shutdown and use exec form

```properties
# application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

```dockerfile
# Exec form — Java becomes PID 1 and receives SIGTERM directly
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

```bash
# Give Docker enough time for graceful shutdown (must be > timeout-per-shutdown-phase)
docker stop --time 35 myapp
```

**Kubernetes:**
```yaml
spec:
  terminationGracePeriodSeconds: 60   # must be > Spring Boot shutdown timeout
  containers:
  - lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]   # allow load balancer to deregister before SIGTERM
```

**How graceful shutdown works:**
1. `SIGTERM` received → Spring Boot stops accepting new requests
2. Existing requests complete (up to `timeout-per-shutdown-phase`)
3. Spring beans destroyed (DataSource closed, Kafka consumers stopped, etc.)
4. JVM exits cleanly

---

## Mistake 6: No Resource Limits

### ❌ WRONG — Container can consume unlimited host resources

```bash
docker run -d myapp:1.0.0
# No memory or CPU limits → one bad request can OOM the entire host
```

```yaml
# docker-compose.yml — no limits
services:
  app:
    image: myapp:1.0.0
    ports:
      - "8080:8080"
    # No resource limits → noisy neighbor problem
```

### ✅ CORRECT — Always set memory and CPU limits

```bash
docker run -d \
  --memory=512m \
  --memory-swap=512m \      # disable swap (swap = memory + swap; set equal = no swap)
  --cpus=1.0 \
  --memory-reservation=256m \  # soft limit for scheduler hints
  myapp:1.0.0
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:1.0.0
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.5"
```

```yaml
# Kubernetes pod spec
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"    # set equal to requests for Guaranteed QoS
    cpu: "1000m"
```

**JVM flags to match limits:**
```dockerfile
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \   # 75% of 512m = 384m heap
  "-jar", "app.jar"]
```

---

## Mistake 7: Secrets in ENV or Dockerfile

### ❌ WRONG — Secrets baked into the image

```dockerfile
# BAD — secret visible in image layers forever
ENV DB_PASSWORD=super_secret_prod_password
ENV API_KEY=sk-1234abcd...

ARG DB_PASSWORD
ENV DB_PASSWORD=${DB_PASSWORD}
# docker build --build-arg DB_PASSWORD=... → visible in docker history
```

```bash
docker history myapp:1.0.0
# sha256:abc... ENV DB_PASSWORD=super_secret_prod_password  ← exposed!
```

### ✅ CORRECT — Inject secrets at runtime, never at build time

```bash
# Option 1: Environment variable at runtime (acceptable for low-sensitivity secrets)
docker run -e DB_PASSWORD="${DB_PASSWORD}" myapp:1.0.0

# Option 2: Docker secrets (Swarm)
echo "super_secret" | docker secret create db_password -
docker service create --secret db_password myapp:1.0.0
# Secret available at /run/secrets/db_password inside container

# Option 3: --env-file (never commit the file)
docker run --env-file .env.prod myapp:1.0.0
# .env.prod is in .gitignore
```

**Kubernetes secrets:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  db-password: super_secret_prod_password
---
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: myapp-secrets
          key: db-password
    # OR mount as file
    volumeMounts:
    - name: secrets
      mountPath: /run/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: myapp-secrets
```

**BuildKit secret mounting (build-time secrets):**
```dockerfile
# Never written to a layer
RUN --mount=type=secret,id=maven_settings,target=/root/.m2/settings.xml \
    mvn package -B
```

```bash
docker build --secret id=maven_settings,src=$HOME/.m2/settings.xml -t myapp .
```

---

## Mistake 8: Logging to Files Instead of stdout

### ❌ WRONG — Logging to files inside container

```xml
<!-- logback-spring.xml — logging to a file inside the container -->
<configuration>
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/myapp/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>/var/log/myapp/app.%d{yyyy-MM-dd}.log</fileNamePattern>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="FILE" />
  </root>
</configuration>
```

**Problems:**
- Logs lost when container is removed (`docker rm`)
- `docker logs` returns nothing — no visibility
- Logs fill up container's writable layer → performance and disk issues
- Log aggregation tools (Fluentd, Loki) can't collect container logs automatically

### ✅ CORRECT — Log to stdout/stderr; let the platform aggregate

```xml
<!-- logback-spring.xml — structured JSON logging to stdout -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <!-- Outputs structured JSON — works with ELK, Loki, CloudWatch, Datadog -->
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

```xml
<!-- pom.xml — logstash-logback-encoder for JSON logs -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>7.4</version>
</dependency>
```

```properties
# application.properties — simpler alternative
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger{36} - %msg%n
# Or use Spring Boot's structured logging (Boot 3.4+)
logging.structured.format.console=ecs
```

**Why stdout/stderr:**
- `docker logs myapp` works
- Kubernetes log aggregation (Fluentd/Promtail sidecar) reads from `/var/log/containers/*.log` which captures stdout
- CloudWatch, Datadog, Loki all integrate with container stdout automatically
- No disk space management needed inside container

---

## Mistake 9: Missing or Wrong .dockerignore

### ❌ WRONG — No .dockerignore

```bash
docker build -t myapp:1.0.0 .
# Sending build context to Docker daemon  847.3 MB  ← entire project including target/
# Slow build start every time
# target/ JARs accidentally available to COPY
# .git/ directory sent to daemon (may contain credentials in git history)
# .env files (secrets!) potentially included
```

### ✅ CORRECT — Comprehensive .dockerignore

```
# .dockerignore

# Build output — never send to Docker daemon
target/
build/
out/
.gradle/
.m2/

# Version control
.git/
.gitignore
.gitattributes

# CI/CD
.github/
.circleci/
.travis.yml
Jenkinsfile

# IDE and OS files
.idea/
*.iml
.vscode/
.DS_Store
Thumbs.db
*.swp
*.swo

# Documentation
*.md
docs/
LICENSE

# Docker files (avoid recursive context issues)
Dockerfile*
docker-compose*.yml
.dockerignore

# Secrets — CRITICAL
.env
.env.*
*.pem
*.key
*.p12
*.jks
secrets/
credentials/

# Test output
surefire-reports/
coverage/
*.xml.bak

# Logs
*.log
logs/
```

**Measure the difference:**
```bash
# Before .dockerignore
docker build -t myapp:before .
# Sending build context to Docker daemon  847.3MB

# After .dockerignore
docker build -t myapp:after .
# Sending build context to Docker daemon  2.048kB
```

**Security impact:** Without `.dockerignore`, files like `.env`, `*.pem`, and `*.key` can be accidentally `COPY`-ed into the image — and are then present in every layer, visible via `docker history`.

---

## Mistake 10: Large Unordered COPY Layers

### ❌ WRONG — Copy everything first, then build

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build

# Copies everything including src — invalidated on every source change
COPY . .
RUN mvn package -DskipTests -B
# Every source change forces full rebuild including dependency download (3+ min)

FROM eclipse-temurin:21-jre-jammy AS runtime
COPY --from=builder /build/target/*.jar app.jar
# Copies entire JAR as one big layer — no granularity for caching
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### ✅ CORRECT — Order by change frequency, coarsest to finest

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build

# Layer 1: pom.xml (changes rarely — when adding dependencies)
COPY pom.xml .
# Layer 2: Download all dependencies — cached until pom.xml changes
RUN mvn dependency:go-offline -B

# Layer 3: Source code (changes frequently — on every commit)
COPY src ./src
# Layer 4: Compile & package — only re-runs on source change
RUN mvn package -DskipTests -B

FROM eclipse-temurin:21-jre-jammy AS runtime

RUN groupadd --system appgroup && useradd --system --gid appgroup appuser
WORKDIR /app

# Layer A: System deps (rarely changes)
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Layer B: Application JAR
COPY --from=builder /build/target/*.jar app.jar

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

**Spring Boot Layered JAR (ultimate caching):**

```dockerfile
FROM eclipse-temurin:21-jre-jammy AS extractor
WORKDIR /extract
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
# Ordered from least to most frequently changed
COPY --from=extractor /extract/dependencies/ ./
COPY --from=extractor /extract/spring-boot-loader/ ./
COPY --from=extractor /extract/snapshot-dependencies/ ./
COPY --from=extractor /extract/application/ ./
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

**Layer caching impact:**

| Scenario | Unordered COPY | Ordered COPY | Layered JAR |
|----------|---------------|-------------|-------------|
| Source change only | Full rebuild (~4 min) | Deps cached (~30s) | App layer only (~5s) |
| Dependency added | Full rebuild (~4 min) | Full rebuild (~4 min) | Full rebuild (~4 min) |
| Config change | Full rebuild (~4 min) | Deps cached (~30s) | App layer only (~5s) |

---

## Bonus: The All-Mistakes Dockerfile

A "hall of shame" showing all 10 mistakes at once:

```dockerfile
# EVERY MISTAKE IN ONE FILE — do not use

FROM openjdk:latest                          # Mistake 2: :latest, Mistake 3: JDK
WORKDIR /app
ENV DB_PASSWORD=super_secret_123             # Mistake 7: secret in ENV
COPY . .                                     # Mistake 10: unordered layers, Mistake 9: no .dockerignore
RUN mvn package                              # no -DskipTests, no -B
EXPOSE 8080
CMD java -jar target/app.jar                 # Mistake 1: runs as root, Mistake 5: shell form
# No HEALTHCHECK                             # Mistake 4
# No resource limits (set at runtime)        # Mistake 6
# Logging to file (configured in app)        # Mistake 8
```

---

## Q&A

### Q1 🟢 Why is running a container as root dangerous?

<details><summary>Click to reveal answer</summary>

Running as root means UID 0 inside the container maps to root on the host (in non-user-namespace setups). If an attacker exploits your application (e.g., remote code execution via a deserialization vulnerability) and escapes the container:

- They have full root access to the host OS
- They can read/write all files on the host
- They can run privileged commands
- In shared Kubernetes environments, they can access other containers' secrets

Even within the container, root can:
- Install malware
- Modify configuration files
- Bypass file permission restrictions

**Fix:**
```dockerfile
RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup --no-create-home appuser
USER appuser
```

This is a security best practice required by many compliance frameworks (SOC 2, PCI-DSS, CIS Docker Benchmark).

</details>

---

### Q2 🟡 What problems does `:latest` tag cause in Kubernetes specifically?

<details><summary>Click to reveal answer</summary>

1. **`imagePullPolicy` confusion:** By default, Kubernetes uses `imagePullPolicy: IfNotPresent` for tags other than `:latest`. For `:latest`, it defaults to `Always`. This inconsistency causes unexpected behavior.

2. **Stale image usage:** If `imagePullPolicy: IfNotPresent` and `:latest` already exists on the node, Kubernetes won't pull a newer image — you might deploy "v2" code but run "v1" because `:latest` is cached.

3. **No rollback:** `kubectl rollout undo deployment/myapp` rolls back to the previous revision — but if both revisions use `:latest`, Kubernetes can't distinguish them.

4. **Non-deterministic deployments:** The same `kubectl apply` command on different days deploys different code.

```yaml
# WRONG
image: myapp:latest

# CORRECT — immutable reference
image: myapp:1.4.2
# OR
image: myapp:a3b7c9d  # git sha
# OR most secure
image: myapp@sha256:3f5f09c47a856c78e...  # digest
```

</details>

---

### Q3 🟡 How does Spring Boot's graceful shutdown work, and what configuration is needed?

<details><summary>Click to reveal answer</summary>

When Spring Boot receives `SIGTERM` (with `server.shutdown=graceful`):

1. **Phase 1 — Stop accepting new requests:** The web server (Tomcat/Netty) stops accepting new HTTP connections
2. **Phase 2 — Drain in-flight requests:** Waits for active requests to complete (up to `timeout-per-shutdown-phase`)
3. **Phase 3 — Spring context shutdown:** `@PreDestroy` methods, `DisposableBean.destroy()`, database connection pool shutdown, Kafka consumer shutdown
4. **JVM exit:** Clean shutdown

```properties
# application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

```dockerfile
# SIGTERM must reach Java directly — use exec form
ENTRYPOINT ["java", "-jar", "app.jar"]

# NOT shell form (SIGTERM goes to sh, not java)
# CMD java -jar app.jar  ← WRONG
```

```bash
# Give enough time for graceful shutdown
docker stop --time 35 myapp   # 35s > 30s timeout
```

In Kubernetes, set `terminationGracePeriodSeconds: 60` and add a `preStop` sleep of 5–10s to let the load balancer deregister the pod before SIGTERM is sent.

</details>

---

### Q4 🟢 What is the HEALTHCHECK instruction and how does Docker use it?

<details><summary>Click to reveal answer</summary>

`HEALTHCHECK` defines a command Docker runs periodically to check if the container is healthy:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

Parameters:
- `--interval`: How often to check (default: 30s)
- `--timeout`: Time before check considered failed (default: 30s)
- `--start-period`: Grace period after container start (JVM startup time)
- `--retries`: Consecutive failures before marking `unhealthy`

Container states:
- `starting` → during start-period
- `healthy` → checks passing
- `unhealthy` → `retries` consecutive failures

Docker Compose restarts unhealthy containers if `restart: unless-stopped` is set. In Swarm, unhealthy containers are replaced. **In Kubernetes, use liveness/readiness probes instead** — Kubernetes ignores Docker HEALTHCHECK.

```bash
docker inspect myapp | jq '.[0].State.Health'
```

</details>

---

### Q5 🔴 Describe a scenario where not setting Docker resource limits caused a production incident.

<details><summary>Click to reveal answer</summary>

**Scenario:** A Spring Boot service with a memory leak (Hibernate session not closed, accumulating objects in heap) runs without container memory limits.

**Timeline:**
1. 2:00 AM — Unusual traffic spike causes memory leak to grow faster than usual
2. 2:15 AM — Container memory exceeds available host RAM
3. 2:16 AM — Linux OOM Killer kills the Java process (PID 1 in container)
4. 2:16 AM — Container crashes; Docker restarts it immediately
5. 2:17 AM — Container starts, loads into memory, leak begins again
6. 2:17 AM — OOM Killer kills it again (restart loop)
7. 2:17 AM — **Host OOM Killer starts killing other containers** on the same node
8. 2:18 AM — PostgreSQL container killed → **entire node goes down**

**With resource limits:**
```bash
docker run --memory=512m myapp:1.0.0
```

The leaky container is OOMKilled at 512 MB. Only `myapp` goes down — not the entire node. The incident is isolated.

**Fix:**
1. Set `--memory=512m` (or Kubernetes `limits.memory`)
2. Set JVM `-XX:MaxRAMPercentage=75.0` so heap is capped
3. Set `-XX:MaxMetaspaceSize=256m` to cap Metaspace
4. Fix the leak (close Hibernate sessions)

</details>

---

### Q6 🟡 What are the risks of putting secrets in Docker environment variables?

<details><summary>Click to reveal answer</summary>

**Risk 1: `docker inspect` exposure**
```bash
docker inspect myapp | jq '.[0].Config.Env'
# ["DB_PASSWORD=super_secret", "API_KEY=sk-abc123"]
# Anyone with docker access on the host can read this
```

**Risk 2: Image layer exposure**
```dockerfile
ENV DB_PASSWORD=super_secret  # baked into image layer permanently
```
```bash
docker history myapp:1.0.0  # visible in all image history
```

**Risk 3: Kubernetes pod spec exposure**
```yaml
env:
- name: DB_PASSWORD
  value: "super_secret"  # plaintext in pod spec, visible in etcd
```

**Risk 4: Logging**
Spring Boot logs environment at startup (with `DEBUG` level or actuator `/env`) — secrets can appear in logs.

**Mitigations:**
1. Use Kubernetes `Secret` objects (base64, not encrypted by default — enable etcd encryption at rest)
2. Use external secret managers: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager
3. Use sealed secrets or external-secrets-operator in Kubernetes
4. Mask secrets in log output with Logback's `%maskedMessage` or Spring Boot's `spring.security.oauth2.*` masking

</details>

---

### Q7 🟢 Why should Java containers log to stdout instead of log files?

<details><summary>Click to reveal answer</summary>

**Container-native logging requires stdout/stderr** because:

1. **`docker logs` only captures stdout/stderr** — logging to files means `docker logs myapp` returns nothing

2. **Kubernetes log aggregation** (Fluentd, Fluent Bit, Promtail) works by reading `/var/log/containers/*.log` — which is the container's stdout. File-based logging requires a sidecar container to ship logs.

3. **No disk management** — log files inside a container's writable layer fill up the Docker host's disk. Log rotation inside containers is complex to configure correctly.

4. **Ephemeral storage** — container is removed → logs are lost. stdout logs are captured by Docker's log driver and persisted externally.

5. **Centralized log shipping** — CloudWatch Logs agent, Datadog agent, Loki all integrate with Docker's log driver to capture stdout automatically.

```xml
<!-- Correct: log to stdout with structured JSON -->
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

</details>

---

### Q8 🟡 What can accidentally end up in a Docker image if you don't use .dockerignore?

<details><summary>Click to reveal answer</summary>

Without `.dockerignore`, `COPY . .` or `ADD . .` includes everything in the build context:

**Secrets and credentials:**
- `.env` files with database passwords, API keys
- `*.pem`, `*.key`, `*.p12` — SSL/TLS certificates and private keys
- `~/.m2/settings.xml` (if in project dir) — Nexus/Artifactory credentials
- `.git/config` — may contain authenticated remote URLs

**Large unnecessary files:**
- `target/` — compiled JARs, class files (hundreds of MB)
- `.git/` — entire git history (can be large)
- `node_modules/` — frontend dependency (can be hundreds of MB)
- IDE files `.idea/`, `.vscode/`

**Build configuration (not needed at runtime):**
- `Dockerfile` itself (slightly ironic)
- `docker-compose.yml`
- CI configuration files

**Impact:**
```bash
# Without .dockerignore
Sending build context to Docker daemon  847.3 MB  ← 3 minutes just to send context

# With .dockerignore
Sending build context to Docker daemon  2.048 kB  ← instant
```

</details>

---

### Q9 🟡 Show how to use Docker secrets in a Swarm setup to inject a database password.

<details><summary>Click to reveal answer</summary>

```bash
# Create the secret
echo "super_secret_prod_password" | docker secret create db_password -

# Or from a file
docker secret create db_password ./db_password.txt

# Deploy service with the secret
docker service create \
  --name myapp \
  --secret db_password \
  --replicas 3 \
  -p 8080:8080 \
  myapp:1.0.0
```

Inside the container, the secret is mounted at `/run/secrets/db_password`:

```java
// Read secret from file (Spring Boot custom configuration)
@Bean
public DataSource dataSource() throws IOException {
    String password = Files.readString(Path.of("/run/secrets/db_password")).strip();
    // use password...
}
```

Or configure Spring Boot to read from file:
```properties
# application.properties
spring.datasource.password=${DB_PASSWORD_FILE:/run/secrets/db_password}
```

**Why file-based is better than env vars:** Secrets mounted as files are:
- Not visible in `docker inspect` environment section
- Not passed to child processes
- Easier to rotate (update the secret, redeploy)

</details>

---

### Q10 🟡 What is the difference between shell form and exec form for CMD/ENTRYPOINT, and which should you use for Java?

<details><summary>Click to reveal answer</summary>

**Shell form:** Runs the command inside a shell (`/bin/sh -c`)
```dockerfile
CMD java -jar app.jar
ENTRYPOINT java -jar app.jar
```

Process tree:
```
PID 1: /bin/sh -c "java -jar app.jar"
PID 2: java -jar app.jar  ← child of sh
```

**Exec form:** Runs the command directly, no shell wrapper
```dockerfile
CMD ["java", "-jar", "app.jar"]
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Process tree:
```
PID 1: java -jar app.jar  ← Java IS the init process
```

**For Java, always use exec form because:**

1. **Signal handling:** `SIGTERM` from `docker stop` goes to PID 1. With shell form, `/bin/sh` receives `SIGTERM` but doesn't forward it to Java — Spring Boot graceful shutdown never triggers, and Docker kills Java with `SIGKILL` after the timeout.

2. **Environment variable expansion:** Shell form supports `$VAR` expansion, exec form does not. But this is easily solved with an entrypoint script using `exec`.

3. **`PID 1` zombie reaping:** Shell form means `sh` is PID 1 and handles zombie reaping; but many shells don't do this properly. With exec form, use `--init` or `tini` if you need zombie reaping.

</details>

---

### Q11 🔴 A security scanner flagged a critical CVE in your production Docker image. Walk through your remediation process.

<details><summary>Click to reveal answer</summary>

**Step 1: Identify scope**
```bash
# Scan with Trivy
trivy image myapp:1.4.2 --severity CRITICAL,HIGH

# Or Grype
grype myapp:1.4.2

# Or Snyk
snyk container test myapp:1.4.2
```

**Step 2: Identify source**
```
CVE in base OS packages? → Update base image
CVE in Java library (JAR)? → Update dependency in pom.xml
CVE in JDK itself? → Update eclipse-temurin version
```

**Step 3: Fix**

*Base image CVE:*
```dockerfile
# Pin to latest patched version
FROM eclipse-temurin:21.0.5_11-jre-jammy  # check Temurin release notes for CVE fix
```

*Library CVE:*
```xml
<!-- pom.xml — force dependency version -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.vulnerable.lib</groupId>
      <artifactId>vulnerable-artifact</artifactId>
      <version>2.1.0</version>  <!-- patched version -->
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Step 4: Rebuild and re-scan**
```bash
docker build --no-cache -t myapp:1.4.3 .   # --no-cache ensures fresh layer pull
trivy image myapp:1.4.3  # verify CVE is gone
```

**Step 5: Deploy**
```bash
docker push myapp:1.4.3
kubectl set image deployment/myapp app=myapp:1.4.3
```

**Prevention:** Add Trivy/Grype to CI pipeline, block merge on CRITICAL CVEs, enable Docker Scout or GitHub Dependabot for continuous scanning.

</details>

---

### Q12 🟡 How do you make a container filesystem read-only for extra security?

<details><summary>Click to reveal answer</summary>

```bash
# Read-only filesystem — any write attempt fails
docker run --read-only myapp:1.0.0

# But Java needs to write to /tmp for temp files
docker run --read-only --tmpfs /tmp:size=64m myapp:1.0.0

# Also allow writes to specific directories
docker run --read-only \
  --tmpfs /tmp:size=64m \
  --tmpfs /app/logs:size=10m \
  myapp:1.0.0
```

**In Kubernetes:**
```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
- name: tmp
  mountPath: /tmp
volumes:
- name: tmp
  emptyDir:
    medium: Memory
    sizeLimit: 64Mi
```

**Why it matters:** If an attacker gains code execution, they can't:
- Drop malware onto the filesystem
- Modify application binaries
- Create cron jobs or init scripts

**Java-specific:** Spring Boot creates temp files in `/tmp` by default. Some JVM operations write to `/tmp`. Always add `--tmpfs /tmp` when using `--read-only`.

</details>

---

### Q13 🟢 How do you configure Spring Boot Actuator health endpoint for container health checks?

<details><summary>Click to reveal answer</summary>

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
# application.properties

# Expose health endpoint
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always

# Enable liveness and readiness probes (Kubernetes-style)
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true

# Separate management port (optional — keeps health off main port)
management.server.port=8081
```

```dockerfile
# Dockerfile health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

**Endpoints:**
- `/actuator/health` — overall health (UP/DOWN)
- `/actuator/health/liveness` — is the app alive? (restart if DOWN)
- `/actuator/health/readiness` — is the app ready for traffic? (remove from load balancer if DOWN)

**Custom health indicator:**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // check DB connectivity
        return Health.up().withDetail("database", "connected").build();
    }
}
```

</details>

---

### Q14 🟡 What's wrong with building and deploying a Docker image with `docker build` and `docker run` directly in production?

<details><summary>Click to reveal answer</summary>

**Problems:**

1. **No image registry:** Images aren't versioned or stored — you can't roll back
2. **Build on production hosts:** Build tools, source code, and build credentials are on production machines
3. **No testing gate:** Images aren't tested before deployment
4. **Manual process:** Human error, inconsistent — not reproducible
5. **No audit trail:** Who deployed what, when?
6. **Security risk:** Build context (with source code) transmitted over the network to the production host

**Correct CI/CD pipeline:**
```
Git push → CI tests (mvn test) → docker build → docker push (registry) → CD deploys tagged image
```

```yaml
# GitHub Actions example
jobs:
  build-and-push:
    steps:
    - uses: actions/checkout@v4
    - name: Run tests
      run: mvn test
    - name: Build image
      run: docker build -t ghcr.io/myorg/myapp:${{ github.sha }} .
    - name: Push image
      run: docker push ghcr.io/myorg/myapp:${{ github.sha }}
  deploy:
    needs: build-and-push
    steps:
    - name: Deploy to Kubernetes
      run: kubectl set image deployment/myapp app=ghcr.io/myorg/myapp:${{ github.sha }}
```

</details>

---

### Q15 🟡 How do you implement a zero-downtime deployment with Docker and Kubernetes?

<details><summary>Click to reveal answer</summary>

**Kubernetes Rolling Update (default):**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never go below desired replica count
      maxSurge: 1          # allow 1 extra pod during rollout
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: myapp:1.4.2
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "10"]  # allow LB deregistration before SIGTERM
```

**Spring Boot requirements:**
```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

**Flow:**
1. New pod starts → readiness probe starts checking
2. New pod passes readiness → added to Service (receives traffic)
3. Old pod receives SIGTERM → `preStop` sleep (LB deregisters)
4. Old pod drains in-flight requests (graceful shutdown)
5. Old pod exits cleanly

**Docker Compose** doesn't natively support rolling updates — use Swarm or Kubernetes.

</details>

---

### Q16 🟢 What is Docker Scout / Trivy and why should you use them?

<details><summary>Click to reveal answer</summary>

**Container image vulnerability scanners** analyze the software packages in your Docker images against known CVE databases (NVD, GitHub Advisory, OS vendor advisories).

**Trivy** (open source, by Aqua Security):
```bash
# Install
brew install trivy   # or apt/yum

# Scan an image
trivy image myapp:1.0.0

# Scan in CI (fail on HIGH or CRITICAL)
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:1.0.0

# Scan a Dockerfile for misconfigurations
trivy config Dockerfile
```

**Docker Scout** (built into Docker Desktop and CLI):
```bash
docker scout cves myapp:1.0.0
docker scout recommendations myapp:1.0.0  # suggests base image updates
```

**Grype** (by Anchore):
```bash
grype myapp:1.0.0
```

**Integrate in CI:**
```yaml
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    severity: CRITICAL,HIGH
    exit-code: 1
```

Regular scanning catches CVEs in base images and transitive dependencies that your `pom.xml` dependency checks miss.

</details>

---

### Q17 🔴 Design a secure, production-ready Dockerfile for a Spring Boot microservice. Explain each decision.

<details><summary>Click to reveal answer</summary>

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: Build ----
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build

# Cache dependencies separately from source (faster rebuilds)
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn dependency:go-offline -B

COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn package -DskipTests -B

# ---- Stage 2: Extract layered JAR ----
FROM eclipse-temurin:21-jre-jammy AS extractor
WORKDIR /extract
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ---- Stage 3: Production runtime ----
FROM eclipse-temurin:21-jre-jammy AS runtime

# 1. Non-root user (Mistake 1 fix)
RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup --no-create-home appuser

WORKDIR /app

# 2. curl for health check (Mistake 4 fix)
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# 3. Layered JAR for optimal caching (Mistake 10 fix)
COPY --from=extractor /extract/dependencies/ ./
COPY --from=extractor /extract/spring-boot-loader/ ./
COPY --from=extractor /extract/snapshot-dependencies/ ./
COPY --from=extractor /extract/application/ ./

# 4. Switch to non-root (Mistake 1 fix)
USER appuser

EXPOSE 8080

# 5. Health check (Mistake 4 fix)
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# 6. Exec form + JVM tuning + container support (Mistakes 3, 5 fix)
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-XX:MaxMetaspaceSize=256m", \
  "-XX:+UseG1GC", \
  "-XX:MaxGCPauseMillis=200", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

**Each decision:**
- `BuildKit cache mount` → fast rebuilds without caching Maven in image layers
- `Non-root user` → principle of least privilege
- `JRE base image` → no compiler tools, smaller attack surface (Mistake 3)
- `--no-install-recommends` → minimal OS packages installed
- `Layered JAR` → Docker cache reuse when only application code changes
- `HEALTHCHECK` → Docker and orchestrators know app health (Mistake 4)
- `exec form ENTRYPOINT` → SIGTERM reaches JVM for graceful shutdown (Mistake 5)
- `UseContainerSupport + MaxRAMPercentage` → correct heap sizing in containers

</details>

---

### Q18 🟡 How do you handle application configuration that differs between environments (dev/staging/prod) in Docker?

<details><summary>Click to reveal answer</summary>

**Option 1: Spring Profiles via environment variable**
```dockerfile
ENV SPRING_PROFILES_ACTIVE=production
```
```bash
docker run -e SPRING_PROFILES_ACTIVE=staging myapp:1.0.0
```

**Option 2: Environment variable overrides (12-Factor App)**
```bash
docker run \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://prod-db:5432/mydb \
  -e SPRING_DATASOURCE_USERNAME=produser \
  -e SPRING_DATASOURCE_PASSWORD="$(cat /run/secrets/db_pass)" \
  myapp:1.0.0
```

**Option 3: Config file injection (mount at runtime)**
```bash
docker run \
  -v /etc/myapp/application-prod.yml:/app/config/application-prod.yml:ro \
  -e SPRING_CONFIG_ADDITIONAL_LOCATION=file:/app/config/ \
  myapp:1.0.0
```

**Option 4: Kubernetes ConfigMap**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  SPRING_PROFILES_ACTIVE: production
  LOGGING_LEVEL_ROOT: WARN
---
spec:
  containers:
  - envFrom:
    - configMapRef:
        name: myapp-config
```

**Anti-pattern:** Don't build separate images per environment. One image, multiple configurations — bake the application, inject the config at runtime.

</details>

---

### Q19 🟡 What is a distroless image and when should a Java team choose it over a JRE image?

<details><summary>Click to reveal answer</summary>

**Distroless images** (by Google) contain only the application runtime and its direct dependencies — no shell, no package manager, no OS utilities.

```dockerfile
FROM gcr.io/distroless/java21-debian12

WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar

# No USER needed — distroless :nonroot variant handles this
# FROM gcr.io/distroless/java21-debian12:nonroot

CMD ["app.jar"]
# CMD with exec form — distroless java images auto-prefix with "java -jar"
```

**Choose distroless when:**
- Strict security requirements (PCI-DSS, FedRAMP, financial services)
- Security scanners reject non-distroless images
- You want minimal CVE surface (no bash CVEs, no curl CVEs, etc.)
- Team has Kubernetes HTTP probes configured (no curl for HEALTHCHECK needed)

**Stick with JRE when:**
- Team needs `docker exec -it ... /bin/bash` for debugging
- HEALTHCHECK via `curl` is required
- Simpler operational model is preferred
- Security requirements don't mandate distroless

**Debugging distroless in Kubernetes:**
```bash
kubectl debug -it <pod> --image=eclipse-temurin:21-jre-jammy --target=myapp -- /bin/bash
```

This attaches an ephemeral debug container with a shell to the same pod, sharing the process namespace.

</details>

---

### Q20 🔴 How would you set up automatic image vulnerability scanning in a GitHub Actions CI pipeline for a Spring Boot application?

<details><summary>Click to reveal answer</summary>

```yaml
# .github/workflows/build-and-scan.yml
name: Build, Scan, and Push

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-test-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write   # for SARIF upload

    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'

    - name: Run tests
      run: mvn test -B

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        load: true   # load into local daemon for scanning
        tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL,HIGH
        exit-code: 1   # fail pipeline on CRITICAL/HIGH CVEs

    - name: Upload Trivy results to GitHub Security tab
      if: always()  # upload even if scan failed
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: trivy-results.sarif

    - name: Push image (only if scan passes and on main branch)
      if: github.ref == 'refs/heads/main'
      run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Key design decisions:**
- Tests run before image build (fast feedback on failures)
- Image is built but not pushed until scan passes
- SARIF format integrates with GitHub's Security tab
- Pipeline fails on CRITICAL/HIGH CVEs — images with known critical vulnerabilities never reach registry
- Only pushes on `main` branch — PRs build and scan but don't push

</details>
