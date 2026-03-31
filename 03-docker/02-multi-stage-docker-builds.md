# Chapter 2: Multi-Stage Docker Builds

Multi-stage builds are the single most impactful Dockerfile improvement for Java applications. They allow you to use a full JDK + build tool in one stage and produce a lean, production-safe image in the next — without any build tools leaking into the final artifact.

---

## 1. Why Multi-Stage Builds?

| Concern | Without Multi-Stage | With Multi-Stage |
|---------|-------------------|-----------------|
| Final image contains Maven/Gradle | ✅ (bad) | ❌ (good) |
| Final image contains JDK | ✅ (bad) | ❌ (good) |
| Attack surface | Large | Minimal |
| Image size | ~500–700 MB | ~150–200 MB |
| Build reproducibility | Requires host toolchain | Self-contained |

**Core idea:** Docker stages are like pipeline steps. Earlier stages produce artifacts. Later stages copy only what they need.

---

## 2. Maven Multi-Stage Build (Standard)

```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Stage 1: Build
# ============================================================
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /build

# Copy POM first — allows Docker to cache the dependency download layer.
# This layer is only invalidated when pom.xml changes, not when source changes.
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Now copy source — this layer changes on every code edit
COPY src ./src

# Build the fat JAR, skip tests (tests run in CI before docker build)
RUN mvn package -DskipTests -B

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM eclipse-temurin:21-jre-jammy AS runtime

WORKDIR /app

# Non-root user for security
RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup appuser

# Copy only the JAR from the builder stage
COPY --from=builder /build/target/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

---

## 3. Maven Multi-Stage Build (Multi-Module Project)

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build

# Copy root POM and all module POMs first
COPY pom.xml .
COPY api/pom.xml api/
COPY service/pom.xml service/
COPY domain/pom.xml domain/

RUN mvn dependency:go-offline -B

# Now copy all source
COPY api/src api/src
COPY service/src service/src
COPY domain/src domain/src

RUN mvn package -DskipTests -B -pl service -am

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
COPY --from=builder /build/service/target/*.jar app.jar
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

## 4. Gradle Multi-Stage Build

```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Stage 1: Build with Gradle
# ============================================================
FROM eclipse-temurin:21-jdk-jammy AS builder

WORKDIR /build

# Install Gradle wrapper dependencies first (layer cache)
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./

# Download dependencies — cached until build files change
RUN ./gradlew dependencies --no-daemon

# Copy source and build
COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM eclipse-temurin:21-jre-jammy AS runtime

WORKDIR /app

RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup appuser

COPY --from=builder /build/build/libs/*.jar app.jar

USER appuser
EXPOSE 8080

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

### Gradle with Wrapper (Recommended for Reproducible Builds)

```dockerfile
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /build

COPY gradlew .
COPY gradle/wrapper gradle/wrapper
RUN ./gradlew --version  # warm up wrapper download

COPY build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon

COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
COPY --from=builder /build/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

## 5. Layer Caching Deep Dive

### The Golden Rule

> Copy files that **change rarely** before files that **change often**.

```
Change frequency (rarest → most frequent):
  Base image → JRE patches (weeks)
  pom.xml / build.gradle → dependency changes (days)
  src/ → source code changes (minutes)
```

### Maven Layer Cache Timeline

```
docker build (first time, cold cache):
  Layer 1: FROM maven:3.9-eclipse-temurin-21        [PULL]   ~30s
  Layer 2: COPY pom.xml .                           [RUN]    <1s
  Layer 3: RUN mvn dependency:go-offline            [RUN]    ~3min  ← expensive
  Layer 4: COPY src ./src                           [RUN]    <1s
  Layer 5: RUN mvn package -DskipTests              [RUN]    ~45s

docker build (source-only change):
  Layer 1: FROM maven:3.9-eclipse-temurin-21        [CACHED] instant
  Layer 2: COPY pom.xml .                           [CACHED] instant
  Layer 3: RUN mvn dependency:go-offline            [CACHED] instant  ← saved 3 min!
  Layer 4: COPY src ./src                           [RUN]    <1s
  Layer 5: RUN mvn package -DskipTests              [RUN]    ~30s
```

---

## 6. .dockerignore for Multi-Stage Builds

```
# .dockerignore
# Build artifacts — we don't want to copy these into builder stage
target/
build/
.gradle/
out/

# Version control
.git/
.github/
.gitignore

# IDE files
.idea/
*.iml
.vscode/
*.swp

# Docs and tests (if not needed in image)
*.md
docs/

# Docker files themselves
Dockerfile*
docker-compose*.yml
.dockerignore

# Secrets
.env
.env.*
*.pem
*.key
secrets/

# Logs
*.log
logs/
```

**Impact:** Without `.dockerignore`, a project with a populated `target/` (50–200 MB) sends all of that to the Docker daemon as build context on every build — wasting time before a single layer executes.

---

## 7. Image Size Comparison

| Base Image | Approx Size | Notes |
|-----------|------------|-------|
| `maven:3.9-eclipse-temurin-21` | ~500 MB | Build stage — never ship to prod |
| `eclipse-temurin:21-jdk-jammy` | ~420 MB | JDK — avoid in production |
| `eclipse-temurin:21-jre-jammy` | ~220 MB | Good default for production |
| `eclipse-temurin:21-jre-alpine` | ~130 MB | Smaller, musl libc — test compatibility |
| `gcr.io/distroless/java21-debian12` | ~170 MB | No shell, minimal attack surface |
| `gcr.io/distroless/java21:nonroot` | ~170 MB | Distroless + non-root user |

*Sizes are approximate and change with updates.*

---

## 8. Distroless Images

Google's [distroless images](https://github.com/GoogleContainerTools/distroless) contain only your application and its runtime dependencies — no shell, no package manager, no `apt`.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# Distroless runtime
FROM gcr.io/distroless/java21-debian12 AS runtime
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
# No ENTRYPOINT needed — distroless java images default to java -jar
CMD ["app.jar"]
```

### Distroless Pros and Cons

| Pros | Cons |
|------|------|
| Minimal attack surface (no shell exploits) | Cannot `docker exec -it ... /bin/bash` |
| Smaller image | Harder to debug in production |
| Passes many security scanner policies | No curl for health checks — use exec health probes in K8s |
| No unnecessary tools | Less familiar to many teams |

**Kubernetes health probe for distroless (no curl):**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

---

## 9. Spring Boot Layered JAR (Advanced)

Spring Boot 2.3+ supports layered JARs — further splitting the fat JAR into layers by change frequency for optimal Docker caching.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# Extract layers from the fat JAR
FROM eclipse-temurin:21-jre-jammy AS extractor
WORKDIR /extract
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Final image — layers ordered by change frequency
FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
# Dependencies change rarely
COPY --from=extractor /extract/dependencies/ ./
COPY --from=extractor /extract/spring-boot-loader/ ./
# Snapshot dependencies change occasionally
COPY --from=extractor /extract/snapshot-dependencies/ ./
# Application code changes frequently
COPY --from=extractor /extract/application/ ./

EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

**Benefit:** When only your application code changes, Docker reuses the cached `dependencies` and `spring-boot-loader` layers — rebuilds in seconds instead of minutes.

---

## Q&A

### Q1 🟢 What problem do multi-stage builds solve for Java applications?

<details><summary>Click to reveal answer</summary>

Multi-stage builds solve two key problems:

1. **Build tool contamination:** Maven/Gradle + JDK are ~500 MB of tools needed only during compilation. Without multi-stage, they'd end up in the production image, increasing attack surface and image size.

2. **Host dependency:** Without multi-stage, developers need Maven and JDK installed locally to build the JAR before `docker build`. Multi-stage makes the entire build self-contained — any machine with Docker can produce the final image.

```
With multi-stage:
  Build stage image: ~500 MB (Maven + JDK)
  Runtime image:     ~220 MB (JRE + your JAR only)

Without multi-stage:
  Single image:      ~500+ MB (Maven + JDK + JAR)
```

</details>

---

### Q2 🟡 Why do we `COPY pom.xml` before `COPY src` in the builder stage?

<details><summary>Click to reveal answer</summary>

To exploit Docker's layer caching. The `COPY pom.xml` + `RUN mvn dependency:go-offline` layer is cached until `pom.xml` changes. If you copy source code first, any code change invalidates the dependency download layer:

```dockerfile
# BAD — source change invalidates dependency download cache
COPY . .
RUN mvn dependency:go-offline   # re-runs every time any file changes

# GOOD — dependency download only re-runs when pom.xml changes
COPY pom.xml .
RUN mvn dependency:go-offline   # cached for most builds
COPY src ./src
RUN mvn package -DskipTests
```

In a team with multiple developers, this can save 2–5 minutes per build per developer — and is even more valuable in CI where cold starts are common.

</details>

---

### Q3 🟡 How does `COPY --from=builder` work?

<details><summary>Click to reveal answer</summary>

`COPY --from=<stage>` copies files from a previous build stage into the current stage.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
# ... builds app.jar ...

FROM eclipse-temurin:21-jre-jammy AS runtime
# Copy just the JAR from the builder stage — everything else is discarded
COPY --from=builder /build/target/myapp-1.0.0.jar app.jar
```

You can also reference:
- **A named stage:** `COPY --from=builder`
- **A numbered stage:** `COPY --from=0`
- **An external image:** `COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/nginx.conf`

The builder stage is fully discarded from the final image — its filesystem, installed tools, and intermediate files are not included.

</details>

---

### Q4 🟢 What is the `AS builder` syntax in a FROM instruction?

<details><summary>Click to reveal answer</summary>

`AS <name>` assigns a human-readable name to a build stage, allowing other stages to reference it:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder   # named "builder"
FROM eclipse-temurin:21-jre-jammy AS runtime    # named "runtime"
```

Without a name, you reference stages by index (`0`, `1`, etc.) — fragile and unreadable.

You can also build only a specific stage:
```bash
docker build --target builder -t myapp:builder .   # stop after builder stage
```

This is useful for:
- Debugging a specific stage
- Extracting test results from the builder stage in CI
- Building a separate "test" stage

</details>

---

### Q5 🟡 How would you run tests inside a multi-stage build and fail the build if tests fail?

<details><summary>Click to reveal answer</summary>

Add a separate test stage, or run tests in the builder stage before packaging:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src

# Run tests — build fails here if tests fail
RUN mvn test -B

# Package after tests pass
RUN mvn package -DskipTests -B

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

In CI, you might prefer running tests separately (before `docker build`) to get better test reports and faster feedback. Then use `-DskipTests` in the Dockerfile to avoid running them twice.

```yaml
# GitHub Actions example
- run: mvn test
- run: docker build -t myapp:${{ github.sha }} .
```

</details>

---

### Q6 🟡 Compare using `eclipse-temurin:21-jre-jammy` vs `gcr.io/distroless/java21` as a runtime base image.

<details><summary>Click to reveal answer</summary>

| Aspect | `eclipse-temurin:21-jre-jammy` | `gcr.io/distroless/java21` |
|--------|-------------------------------|---------------------------|
| Size | ~220 MB | ~170 MB |
| Shell access | ✅ `/bin/bash` available | ❌ No shell |
| curl/wget | ✅ Available (install) | ❌ Not available |
| HEALTHCHECK via curl | ✅ | ❌ (use K8s HTTP probes) |
| Attack surface | Medium | Very low |
| Security scanner pass rate | Good | Excellent |
| Debugging ease | Easy | Hard |
| Suitable for | Most production workloads | Security-sensitive environments |

**Recommendation:** Start with `eclipse-temurin:21-jre-jammy`. Move to distroless if security requirements demand it, and compensate for the lack of shell with proper Kubernetes probes and ephemeral debug containers.

</details>

---

### Q7 🔴 Explain the Spring Boot layered JAR approach and how it improves Docker build caching.

<details><summary>Click to reveal answer</summary>

Spring Boot 2.3+ fat JARs can be extracted into layers ordered by change frequency:

1. **dependencies** — third-party libraries (rarely change)
2. **spring-boot-loader** — Spring Boot's JAR launcher (changes only on Spring version upgrade)
3. **snapshot-dependencies** — SNAPSHOT versions (change occasionally)
4. **application** — your code (changes on every commit)

```bash
# Inspect layers
java -Djarmode=layertools -jar app.jar list
# dependencies
# spring-boot-loader
# snapshot-dependencies
# application
```

In a normal fat JAR Docker build, any code change forces Docker to re-copy the entire JAR (~80 MB). With layered JARs, only the `application` layer (~1–5 MB) is invalidated — Docker reuses the `dependencies` cache.

Enable in `pom.xml`:
```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <layers>
      <enabled>true</enabled>
    </layers>
  </configuration>
</plugin>
```

This saves 20–60 seconds per build in CI when only application code changes.

</details>

---

### Q8 🟡 How do you build only a specific stage in a multi-stage Dockerfile?

<details><summary>Click to reveal answer</summary>

Use `--target`:

```bash
# Build only the "builder" stage (e.g., to run tests or debug build issues)
docker build --target builder -t myapp:builder .

# Build up to the "test" stage
docker build --target test -t myapp:test .

# Normal build (all stages to the last one)
docker build -t myapp:1.0.0 .
```

Common uses:
- **CI test stage:** Build a `test` stage, extract test reports with `docker cp`, then build the final `runtime` stage
- **Debugging:** Build to the `builder` stage and run it interactively to inspect the build environment
- **Conditional builds:** Build different final stages for different environments (e.g., `dev` vs `prod`)

</details>

---

### Q9 🟢 What's wrong with this Dockerfile? Fix it.

```dockerfile
FROM maven:3.9-eclipse-temurin-21
COPY . .
RUN mvn package
FROM eclipse-temurin:21-jre-jammy
COPY target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

<details><summary>Click to reveal answer</summary>

**Problems:**
1. No `WORKDIR` — files copied to `/` root, messy and risky
2. `COPY . .` before downloading dependencies — cache is invalidated on any file change
3. No `-DskipTests` — tests run in Docker build (slow, and test failures block image creation)
4. No `AS` name on stages — harder to read and reference
5. `COPY target/app.jar` — fragile if JAR name has version number; also copies from host `target/` not builder stage
6. No non-root user in runtime stage
7. No JVM container flags

**Fixed:**
```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
RUN groupadd --system appgroup && useradd --system --gid appgroup appuser
COPY --from=builder /build/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

</details>

---

### Q10 🟡 How would you pass the application version into the Docker image at build time?

<details><summary>Click to reveal answer</summary>

Use `ARG` for build-time values and optionally promote to `ENV` or `LABEL`:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
ARG APP_VERSION=0.0.1-SNAPSHOT
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B -Drevision=${APP_VERSION}

FROM eclipse-temurin:21-jre-jammy AS runtime
ARG APP_VERSION=unknown
LABEL org.opencontainers.image.version="${APP_VERSION}"
ENV APP_VERSION=${APP_VERSION}
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build \
  --build-arg APP_VERSION=1.4.2 \
  -t myapp:1.4.2 .
```

**Note:** `ARG` is scoped per stage — you need to re-declare it in each stage where it's needed.

</details>

---

### Q11 🟡 How do you cache Maven dependencies in GitHub Actions with multi-stage Docker builds?

<details><summary>Click to reveal answer</summary>

Docker BuildKit can export/import cache to/from a registry or local path. Use `--cache-from` and `--cache-to`:

```yaml
# .github/workflows/build.yml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/myorg/myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`type=gha` stores the build cache in GitHub Actions cache storage. On subsequent runs, Docker reuses cached layers — the `mvn dependency:go-offline` layer is served from cache as long as `pom.xml` hasn't changed.

`mode=max` caches all layers (including intermediate stages), not just the final image.

</details>

---

### Q12 🔴 What is BuildKit and how does it improve multi-stage build performance?

<details><summary>Click to reveal answer</summary>

BuildKit is Docker's next-generation build engine (enabled by default since Docker 23.0). Key improvements over the legacy builder:

1. **Parallel stage execution:** Stages that don't depend on each other build in parallel
2. **Lazy stage evaluation:** Unused stages are not built (e.g., `--target runtime` skips test stages)
3. **Better cache management:** Content-addressable cache, registry cache export/import
4. **Secrets mounting:** `RUN --mount=type=secret,id=npmrc` — secrets never written to image layers
5. **SSH agent forwarding:** `RUN --mount=type=ssh` for private Git repos
6. **Bind mounts for caching:**

```dockerfile
# Cache Maven local repo across builds (never written to image layer)
RUN --mount=type=cache,target=/root/.m2 mvn dependency:go-offline -B
RUN --mount=type=cache,target=/root/.m2 mvn package -DskipTests -B
```

Enable with `DOCKER_BUILDKIT=1` or in `daemon.json`:
```json
{ "features": { "buildkit": true } }
```

The `# syntax=docker/dockerfile:1` comment at the top enables the latest BuildKit frontend.

</details>

---

### Q13 🟡 What is `RUN mvn dependency:go-offline` and why is it used?

<details><summary>Click to reveal answer</summary>

`mvn dependency:go-offline` downloads all project dependencies (and their transitive dependencies) into the local Maven repository (`~/.m2`), making them available without network access during subsequent Maven commands.

In a Docker build:

```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline -B   # downloads ALL deps → cached layer

COPY src ./src
RUN mvn package -DskipTests -B     # uses cached deps, no network needed
```

**Why `-B`?** `-B` (batch mode) disables Maven's interactive output and progress indicators — important for CI/Docker where there's no TTY.

**Alternative:** Some teams use `mvn install -N` (install parent POMs only) or just rely on Docker BuildKit's `--mount=type=cache` to cache `~/.m2` between builds:

```dockerfile
RUN --mount=type=cache,target=/root/.m2 mvn package -DskipTests -B
```

</details>

---

### Q14 🟢 How do you verify the final image doesn't contain Maven or the JDK?

<details><summary>Click to reveal answer</summary>

```bash
# Check image layers
docker history myapp:1.0.0

# Run the image and check for mvn/javac
docker run --rm myapp:1.0.0 which mvn     # should return: not found
docker run --rm myapp:1.0.0 which javac   # should return: not found
docker run --rm myapp:1.0.0 java -version # should work (JRE present)

# Inspect the image filesystem
docker run --rm -it myapp:1.0.0 find / -name "mvn" 2>/dev/null
docker run --rm -it myapp:1.0.0 ls /usr/local/  # no Maven here

# Check image size
docker images myapp:1.0.0
```

A properly built multi-stage image with `eclipse-temurin:21-jre-jammy` + your JAR should be around 220–250 MB — not 500+ MB.

</details>

---

### Q15 🟡 Explain how to use Docker multi-stage builds to produce both a development and production image from the same Dockerfile.

<details><summary>Click to reveal answer</summary>

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# Development image — includes JDK for debugging, relaxed settings
FROM eclipse-temurin:21-jdk-jammy AS development
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
ENV SPRING_PROFILES_ACTIVE=dev
ENTRYPOINT ["java", "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005", "-jar", "app.jar"]
EXPOSE 8080 5005

# Production image — JRE only, non-root, optimized JVM flags
FROM eclipse-temurin:21-jre-jammy AS production
RUN groupadd --system appgroup && useradd --system --gid appgroup appuser
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
USER appuser
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
EXPOSE 8080
```

```bash
# Build dev image (with remote debugging)
docker build --target development -t myapp:dev .

# Build prod image
docker build --target production -t myapp:1.0.0 .
```

</details>

---

### Q16 🟡 What does the `-DskipTests` Maven flag do and when should you use it in Docker builds?

<details><summary>Click to reveal answer</summary>

`-DskipTests` skips test execution during `mvn package`. The tests are compiled but not run.

(`-Dmaven.test.skip=true` skips both compilation and execution of tests.)

**In Docker builds:** Skip tests in the Dockerfile if:
- Tests already run in a prior CI step (before `docker build`)
- Tests require external services (databases, Kafka) not available during `docker build`
- Tests are slow and slowing down image creation in CI

```bash
# CI pipeline order (best practice)
mvn test                                     # 1. Run tests with full environment
docker build -t myapp:$SHA .                 # 2. Build image (DskipTests in Dockerfile)
docker push myapp:$SHA                       # 3. Push image
```

Don't skip tests if `docker build` is the ONLY place your code is tested — that defeats the purpose of having tests.

</details>

---

### Q17 🟢 What is the `syntax=docker/dockerfile:1` comment at the top of a Dockerfile?

<details><summary>Click to reveal answer</summary>

```dockerfile
# syntax=docker/dockerfile:1
```

This is a **BuildKit directive** (parser directive) that specifies which Dockerfile frontend to use. `docker/dockerfile:1` means "use the latest stable 1.x version of the official Dockerfile syntax."

It enables advanced BuildKit features like:
- `RUN --mount=type=cache` — persistent build caches
- `RUN --mount=type=secret` — secrets without baking into layers
- `RUN --mount=type=ssh` — SSH agent forwarding
- `COPY --link` — improved layer linking

Without this comment, Docker uses whatever frontend version is built into the daemon — which may be older and lack these features.

It must be the **very first line** of the Dockerfile (no blank lines before it).

</details>

---

### Q18 🔴 How would you use `RUN --mount=type=cache` to speed up Maven builds in Docker?

<details><summary>Click to reveal answer</summary>

BuildKit's cache mounts persist across builds without being committed to image layers:

```dockerfile
# syntax=docker/dockerfile:1

FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build

COPY pom.xml .
# Maven local repo is cached across builds — never included in image
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    mvn dependency:go-offline -B

COPY src ./src
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    mvn package -DskipTests -B

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

**Benefits:**
- Maven downloads are cached on the Docker builder machine across multiple builds
- The `~/.m2` cache is **never** written into an image layer (no bloat, no secrets exposure)
- `sharing=locked` prevents parallel builds from corrupting the cache

**Gradle equivalent:**
```dockerfile
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew dependencies --no-daemon
```

</details>

---

### Q19 🟡 How do you handle private Maven repositories (Nexus/Artifactory) in a multi-stage Docker build securely?

<details><summary>Click to reveal answer</summary>

**Wrong approach (secrets baked into image layer):**
```dockerfile
# BAD — credentials visible in docker history
RUN mvn -s settings.xml package  # settings.xml committed to repo with creds
```

**Correct approach using BuildKit secrets:**
```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .

# settings.xml is mounted as a secret — never written to any layer
RUN --mount=type=secret,id=maven_settings,target=/root/.m2/settings.xml \
    mvn dependency:go-offline -B

COPY src ./src
RUN --mount=type=secret,id=maven_settings,target=/root/.m2/settings.xml \
    mvn package -DskipTests -B
```

```bash
docker build \
  --secret id=maven_settings,src=$HOME/.m2/settings.xml \
  -t myapp:1.0.0 .
```

The `settings.xml` is never written to a layer and won't appear in `docker history`. This is the recommended way to handle credentials in Docker builds.

</details>

---

### Q20 🟡 What's the difference between `COPY` and `ADD` in a Dockerfile? Which should you prefer?

<details><summary>Click to reveal answer</summary>

**`COPY`:** Copies files/directories from the build context into the image. Simple, predictable.

```dockerfile
COPY target/app.jar app.jar
COPY src/ /app/src/
```

**`ADD`:** Everything `COPY` does, plus:
- Automatically **extracts** `.tar`, `.tar.gz`, `.tgz`, `.xz`, `.bz2` archives
- Supports **URLs** as the source (downloads from internet during build)

```dockerfile
ADD app.tar.gz /app/          # extracts the archive
ADD https://example.com/file /app/  # downloads and copies
```

**Always prefer `COPY` unless you specifically need `ADD`'s extra features.** Reasons:
- `COPY` intent is clear and explicit
- `ADD` with URLs adds network dependency to the build (non-reproducible, can fail)
- `ADD` with tarballs is surprising — a `.tar.gz` isn't copied as a file, it's extracted

**Exception:** `ADD` is appropriate when you genuinely need to extract a tarball into the image.

</details>
