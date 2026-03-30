# Chapter 1: Docker Fundamentals for Java

This chapter covers everything a Java backend engineer needs to know about Docker fundamentals — from Dockerfile anatomy to Docker Compose for local development stacks.

---

## 1. Dockerfile Anatomy

```dockerfile
# syntax=docker/dockerfile:1

# --- Base image ---
FROM eclipse-temurin:21-jre-jammy

# --- Metadata ---
LABEL maintainer="team@example.com"

# --- Working directory ---
WORKDIR /app

# --- Environment variables ---
ENV SPRING_PROFILES_ACTIVE=production \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

# --- Copy artifact ---
COPY target/my-service-1.0.0.jar app.jar

# --- Expose port (documentation only, does not publish) ---
EXPOSE 8080

# --- Health check ---
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# --- Entrypoint ---
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

## 2. Key Dockerfile Instructions

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image — always the first instruction |
| `RUN` | Executes a shell command during build; creates a new layer |
| `COPY` | Copies files from build context into image |
| `ADD` | Like COPY but also extracts archives and supports URLs (prefer COPY) |
| `WORKDIR` | Sets the working directory for subsequent instructions |
| `EXPOSE` | Documents which port the container listens on |
| `ENV` | Sets environment variables available at runtime |
| `CMD` | Default command — **overridable** by `docker run <image> <cmd>` |
| `ENTRYPOINT` | Fixed entrypoint — not overridden by `docker run` args |
| `ARG` | Build-time variable (not available at runtime) |
| `VOLUME` | Declares a mount point |

### CMD vs ENTRYPOINT

```dockerfile
# CMD only — fully overridable
CMD ["java", "-jar", "app.jar"]

# ENTRYPOINT only — docker run args are appended
ENTRYPOINT ["java", "-jar", "app.jar"]

# Both — ENTRYPOINT is the executable, CMD is default args (overridable)
ENTRYPOINT ["java"]
CMD ["-jar", "app.jar"]
# docker run myimage -version   → runs: java -version
```

---

## 3. Images vs Containers vs Layers

```
Image  = read-only template (stack of layers)
Container = running instance of an image + thin writable layer (copy-on-write)
Layer  = result of each RUN/COPY/ADD instruction; cached and reused
```

**Copy-on-write (CoW):** When a container modifies a file, Docker copies that file into the container's writable layer first. The underlying image layers are never modified.

---

## 4. Essential Docker Commands

```bash
# Build
docker build -t myapp:1.0.0 .
docker build --no-cache -t myapp:1.0.0 .
docker build --build-arg JAR_FILE=target/app.jar -t myapp .

# Run
docker run -d -p 8080:8080 --name myapp myapp:1.0.0
docker run -e SPRING_PROFILES_ACTIVE=dev -p 8080:8080 myapp:1.0.0
docker run --rm -it myapp:1.0.0 /bin/bash  # interactive, remove on exit

# Inspect running containers
docker ps
docker ps -a  # include stopped containers
docker logs myapp
docker logs -f myapp  # follow
docker exec -it myapp /bin/bash
docker inspect myapp

# Lifecycle
docker stop myapp   # SIGTERM → SIGKILL after grace period
docker kill myapp   # immediate SIGKILL
docker rm myapp
docker rmi myapp:1.0.0

# Cleanup
docker system prune          # remove stopped containers, dangling images, unused networks
docker system prune -a       # also remove unused images
docker volume prune          # remove unused volumes
```

---

## 5. Volumes: Bind Mounts vs Named Volumes

```bash
# Named volume — managed by Docker, portable
docker run -v myapp-data:/var/lib/postgresql/data postgres:16

# Bind mount — maps host path into container (good for local dev)
docker run -v $(pwd)/config:/app/config myapp:1.0.0

# tmpfs mount — in-memory, not persisted
docker run --tmpfs /tmp myapp:1.0.0
```

| | Named Volume | Bind Mount |
|---|---|---|
| Managed by Docker | ✅ | ❌ |
| Portable | ✅ | ❌ (host path) |
| Good for prod data | ✅ | ❌ |
| Good for local dev config | ❌ | ✅ |

---

## 6. Networking

```bash
# Bridge (default) — containers on same network can communicate by name
docker network create mynet
docker run --network mynet --name app myapp:1.0.0
docker run --network mynet --name db postgres:16

# Host — container shares host network stack (Linux only)
docker run --network host myapp:1.0.0

# None — no network access
docker run --network none myapp:1.0.0

# Port mapping
docker run -p 8080:8080 myapp:1.0.0      # host:container
docker run -p 127.0.0.1:8080:8080 myapp  # bind to localhost only
```

---

## 7. Docker Compose: Spring Boot + PostgreSQL + Redis

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:dev
    container_name: myapp
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_DATASOURCE_USERNAME: myuser
      SPRING_DATASOURCE_PASSWORD: mypassword
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  db:
    image: postgres:16-alpine
    container_name: mydb
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    container_name: myredis
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  postgres-data:
  redis-data:

networks:
  backend:
    driver: bridge
```

```bash
docker compose up -d          # start all services in background
docker compose logs -f app    # tail app logs
docker compose down           # stop and remove containers
docker compose down -v        # also remove volumes
```

---

## 8. Java Base Image Choices

| Image | Compressed Size | Notes |
|-------|----------------|-------|
| `eclipse-temurin:21-jdk-jammy` | ~220 MB | Full JDK — use for build stage only |
| `eclipse-temurin:21-jre-jammy` | ~150 MB | JRE only — good for runtime |
| `eclipse-temurin:21-jre-alpine` | ~95 MB | Smaller, musl libc (check compatibility) |
| `amazoncorretto:21` | ~200 MB | AWS-optimized, Amazon-maintained |
| `gcr.io/distroless/java21` | ~75 MB | No shell, minimal attack surface |
| `openjdk:21` | ~400 MB | Deprecated on Docker Hub — avoid |

**Recommendation:** Use `eclipse-temurin:21-jre-jammy` for most production workloads. Use distroless for security-sensitive environments.

---

## 9. .dockerignore

```
# .dockerignore
target/
.git/
.github/
.idea/
*.iml
*.md
*.log
logs/
node_modules/
.env
.env.*
docker-compose*.yml
Dockerfile*
```

**Why it matters:** Without `.dockerignore`, the entire `target/` directory (can be hundreds of MB) is sent to the Docker daemon as build context, slowing every build.

---

## Q&A

### Q1 🟢 What is the difference between CMD and ENTRYPOINT in a Dockerfile?

<details><summary>Click to reveal answer</summary>

**ENTRYPOINT** defines the executable that always runs. Arguments passed via `docker run myimage <args>` are **appended** to ENTRYPOINT.

**CMD** provides default arguments. It is **completely replaced** when you pass arguments to `docker run`.

```dockerfile
# With both: ENTRYPOINT is the binary, CMD is the default args
ENTRYPOINT ["java", "-jar"]
CMD ["app.jar"]

# docker run myimage             → java -jar app.jar
# docker run myimage other.jar   → java -jar other.jar
```

Use `ENTRYPOINT` + `CMD` together for Java applications: ENTRYPOINT sets `java -jar` (or the JVM flags), CMD sets the jar name so it can be overridden without changing the Dockerfile.

Always prefer the **exec form** (`["java", "-jar", "app.jar"]`) over shell form (`java -jar app.jar`). Shell form wraps the command in `/bin/sh -c`, which makes PID 1 a shell process — meaning `SIGTERM` won't reach your JVM, breaking graceful shutdown.

</details>

---

### Q2 🟢 What is a Docker layer, and why does layer order matter?

<details><summary>Click to reveal answer</summary>

Each `RUN`, `COPY`, and `ADD` instruction creates a new **read-only layer** in the image. Layers are cached by Docker. If an instruction and all its inputs are unchanged, Docker reuses the cached layer instead of re-running it.

**Layer order matters for cache efficiency:**

```dockerfile
# BAD — copying source before downloading dependencies
COPY . .
RUN mvn package  # cache invalidated whenever any source file changes

# GOOD — download deps first (rarely changes), then copy source
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests
```

In the good example, the `mvn dependency:go-offline` layer is cached as long as `pom.xml` doesn't change — typically saving 2–5 minutes per build.

</details>

---

### Q3 🟢 What does `docker system prune` do?

<details><summary>Click to reveal answer</summary>

`docker system prune` removes:
- All **stopped containers**
- All **dangling images** (images with no tag, typically old build stages)
- All **unused networks**
- All **build cache**

`docker system prune -a` also removes **all unused images** (not just dangling ones — any image not referenced by a running container).

`docker system prune --volumes` additionally removes **unused volumes** (use with caution — this is irreversible data loss).

```bash
docker system prune -a --force  # non-interactive cleanup
```

Run this regularly in CI environments to prevent disk exhaustion.

</details>

---

### Q4 🟡 Explain the difference between a bind mount and a named volume. When would you use each?

<details><summary>Click to reveal answer</summary>

**Named volume:** Managed entirely by Docker. Stored in Docker's data directory (e.g., `/var/lib/docker/volumes/`). Portable across machines (as long as you don't need the actual data).

```bash
docker run -v myapp-data:/var/lib/postgresql/data postgres:16
```

**Bind mount:** Maps a specific host filesystem path into the container. Tightly coupled to the host machine.

```bash
docker run -v /home/user/myapp/config:/app/config myapp:1.0.0
```

**Use named volumes** for:
- Database data in production/staging
- Any data that needs to persist across container restarts

**Use bind mounts** for:
- Local development: hot-reload source code without rebuilding
- Injecting config files (e.g., `application-local.yml`)
- Reading secrets from the host filesystem during development

</details>

---

### Q5 🟡 How do containers on the same Docker bridge network communicate?

<details><summary>Click to reveal answer</summary>

Containers on the same **user-defined bridge network** can communicate using **container names as DNS hostnames**. Docker provides an embedded DNS server for this.

```yaml
services:
  app:
    networks: [backend]
  db:
    container_name: mydb
    networks: [backend]

networks:
  backend:
```

The `app` container can connect to PostgreSQL at `jdbc:postgresql://mydb:5432/mydb` — no IP addresses needed.

**Note:** The default bridge network (`docker0`) does NOT provide automatic DNS resolution by container name. Always create user-defined networks.

</details>

---

### Q6 🟡 Why should you prefer `eclipse-temurin` over `openjdk` as a base image?

<details><summary>Click to reveal answer</summary>

The `openjdk` images on Docker Hub are **officially deprecated** and receive no security updates. The recommended alternatives are:

- **`eclipse-temurin`** — maintained by the Adoptium project (Eclipse Foundation), widely used in enterprise
- **`amazoncorretto`** — maintained by Amazon, optimized for AWS workloads
- **`microsoft/openjdk`** — Microsoft's distribution, optimized for Azure

`eclipse-temurin:21-jre-jammy` is the safest general-purpose choice:
- Based on Ubuntu Jammy (22.04 LTS) — well-understood, good glibc compatibility
- Regular security patches
- Available for `linux/amd64` and `linux/arm64`

</details>

---

### Q7 🟢 What is the purpose of EXPOSE in a Dockerfile?

<details><summary>Click to reveal answer</summary>

`EXPOSE` is **documentation only** — it does not actually publish the port or make it accessible from the host. It tells Docker (and humans reading the Dockerfile) which port the application listens on.

To actually publish the port, you must use `-p` at runtime:

```bash
docker run -p 8080:8080 myapp:1.0.0   # host:container
```

`EXPOSE` is still worth including because:
1. `docker run -P` (uppercase P) auto-publishes all EXPOSED ports to random host ports
2. It serves as living documentation for the image
3. Some orchestration tools (like Docker Compose and Kubernetes) use it as a hint

</details>

---

### Q8 🟡 What is the build context and why does .dockerignore matter?

<details><summary>Click to reveal answer</summary>

The **build context** is the directory (and its contents) sent from the Docker CLI to the Docker daemon when you run `docker build`. By default it's the current directory (`.`).

The entire context is transmitted before building starts. A large context (e.g., including `target/`, `.git/`, `node_modules/`) causes:
- Slow build starts (hundreds of MB sent over a socket)
- Risk of accidentally `COPY`ing secrets or build artifacts into the image

`.dockerignore` filters what's included in the context:

```
target/
.git/
.github/
*.log
.env
```

This is similar to `.gitignore`. Always create a `.dockerignore` before your first `docker build`.

</details>

---

### Q9 🔴 Walk me through what happens when you run `docker run -d -p 8080:8080 myapp:1.0.0`.

<details><summary>Click to reveal answer</summary>

1. **Image pull check:** Docker checks if `myapp:1.0.0` exists locally. If not, it pulls from the configured registry (Docker Hub by default).
2. **Container creation:** Docker creates a new container from the image — a thin writable layer on top of the image layers.
3. **Network setup:** Docker creates a virtual ethernet interface pair, connects one end to the container's network namespace, the other to the `docker0` bridge (or a user-defined bridge). Assigns a private IP (e.g., `172.17.0.x`).
4. **Port mapping:** Docker sets up `iptables` (or nftables) rules to forward traffic from host port `8080` to container port `8080`.
5. **Filesystem mounting:** Any volumes or bind mounts are mounted into the container namespace.
6. **Process start:** Docker runs the container's `ENTRYPOINT`/`CMD` as PID 1 inside the container's PID namespace.
7. **Detached mode (`-d`):** The container runs in the background; `docker run` returns the container ID immediately.

</details>

---

### Q10 🟡 What's wrong with using `:latest` as your Docker image tag?

<details><summary>Click to reveal answer</summary>

`:latest` is a **mutable tag** — it can point to a different image digest every time. This causes:

1. **Non-reproducible builds:** `docker pull myapp:latest` today and tomorrow may give different images
2. **Silent breakage:** A bad push to `:latest` immediately affects everyone pulling it
3. **No rollback story:** You can't roll back to "the previous latest"
4. **Kubernetes won't re-pull:** With `imagePullPolicy: IfNotPresent`, Kubernetes won't re-pull `:latest` if the image already exists locally — you might run stale code

**Instead:**
```dockerfile
FROM eclipse-temurin:21.0.5_11-jre-jammy   # pinned version
# OR pin by digest (most secure):
FROM eclipse-temurin@sha256:abc123...
```

Tag your own images with semantic versions or Git SHAs:
```bash
docker build -t myapp:1.4.2 -t myapp:$(git rev-parse --short HEAD) .
```

</details>

---

### Q11 🟡 How does `depends_on` work in Docker Compose, and what are its limitations?

<details><summary>Click to reveal answer</summary>

`depends_on` controls **start order** of services. With `condition: service_healthy`, Compose waits for the dependency's healthcheck to pass before starting the dependent service.

```yaml
depends_on:
  db:
    condition: service_healthy
```

**Limitation:** Even with `service_healthy`, your Spring Boot app might still fail to connect on startup if:
- The healthcheck passes but the DB isn't fully ready for connections yet
- There's a race condition in connection pool initialization

**Solution:** Use Spring Boot's built-in retry with a connection validation:
```properties
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.initialization-fail-timeout=60000
```

Or implement a startup retry loop using `spring.retry` or a `@Retryable` data source bean.

</details>

---

### Q12 🟢 How do you pass environment variables to a running container?

<details><summary>Click to reveal answer</summary>

```bash
# Single variable
docker run -e SPRING_PROFILES_ACTIVE=prod myapp:1.0.0

# Multiple variables
docker run \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_URL=jdbc:postgresql://db:5432/mydb \
  myapp:1.0.0

# From a file (.env file)
docker run --env-file .env myapp:1.0.0
```

In Docker Compose:
```yaml
environment:
  SPRING_PROFILES_ACTIVE: prod
  DB_URL: jdbc:postgresql://db:5432/mydb
# OR
env_file:
  - .env
```

**Security note:** Never put secrets directly in `docker-compose.yml` if it's committed to source control. Use `.env` files (gitignored), Docker secrets, or a secrets manager.

</details>

---

### Q13 🟡 Explain copy-on-write (CoW) in the context of Docker containers.

<details><summary>Click to reveal answer</summary>

Docker images are composed of multiple **read-only layers**. When a container starts, Docker adds a thin **writable layer** on top (the "container layer").

When a process in the container **reads** a file, Docker looks through the layers from top to bottom until it finds the file — no copying needed.

When a process **writes** to or **deletes** a file that exists only in a read-only image layer, Docker first **copies** that file up to the container's writable layer (copy-on-write), then modifies it there.

**Implications:**
- Image layers are shared between containers — starting 100 containers from the same image doesn't multiply disk usage by 100
- The writable layer is lost when the container is removed (use volumes for persistence)
- CoW can have a performance cost for write-heavy workloads — use volumes for databases

</details>

---

### Q14 🟢 What does `docker exec -it myapp /bin/bash` do?

<details><summary>Click to reveal answer</summary>

It runs `/bin/bash` inside the already-running `myapp` container in **interactive** (`-i`) **TTY** (`-t`) mode — giving you a shell session inside the container.

```bash
docker exec -it myapp /bin/bash   # bash shell (if available)
docker exec -it myapp /bin/sh     # sh (works on Alpine-based images)
docker exec -it myapp env         # print environment variables
docker exec myapp cat /etc/hosts  # non-interactive command
```

**Note:** Distroless images have no shell — you can't `exec` into them. For debugging, use ephemeral debug containers:
```bash
kubectl debug -it <pod> --image=busybox --target=app
```

</details>

---

### Q15 🟡 What is a multi-container application and how does Docker Compose help?

<details><summary>Click to reveal answer</summary>

A multi-container application runs multiple services (e.g., Spring Boot app + PostgreSQL + Redis + Nginx) where each service runs in its own container.

Docker Compose lets you define and manage all services in a single `docker-compose.yml`:

- **Single command startup:** `docker compose up -d` starts all services
- **Network isolation:** Compose creates a default network — services communicate by name
- **Lifecycle management:** `docker compose down` stops and removes everything cleanly
- **Environment parity:** Developers run the same stack locally as in CI

Without Compose, you'd need to run multiple `docker run` commands, manage networks manually, and coordinate startup order yourself.

</details>

---

### Q16 🟢 How do you view logs for a running container?

<details><summary>Click to reveal answer</summary>

```bash
docker logs myapp              # print all logs
docker logs -f myapp           # follow (tail -f equivalent)
docker logs --tail 100 myapp   # last 100 lines
docker logs --since 5m myapp   # logs from last 5 minutes
docker logs --since 2024-01-15T10:00:00 myapp

# In Docker Compose
docker compose logs -f app
docker compose logs --tail 50 app
```

Docker captures stdout and stderr from PID 1. For Java/Spring Boot, make sure your logging framework (Logback/Log4j2) writes to stdout — **not** to files — inside containers.

</details>

---

### Q17 🔴 What happens to PID 1 in a Docker container and why does it matter for Java apps?

<details><summary>Click to reveal answer</summary>

PID 1 in a Linux process is the **init process** — it receives signals and is responsible for reaping zombie processes. In a container, your application becomes PID 1.

**The problem:** Many applications (including the JVM when started via shell form) don't handle signals properly as PID 1:

```dockerfile
# WRONG — shell form; /bin/sh becomes PID 1, Java is a child process
CMD java -jar app.jar

# CORRECT — exec form; Java becomes PID 1 directly
CMD ["java", "-jar", "app.jar"]
```

With shell form, `docker stop` sends `SIGTERM` to `/bin/sh`, not to the JVM — meaning Spring Boot's graceful shutdown never triggers, and Docker kills the JVM with `SIGKILL` after the grace period.

If you need a proper init process (for zombie reaping in complex setups), use `--init`:
```bash
docker run --init myapp:1.0.0
```
This injects `tini` as PID 1, which forwards signals correctly.

</details>

---

### Q18 🟡 How would you pass build-time arguments to a Dockerfile?

<details><summary>Click to reveal answer</summary>

Use `ARG` for build-time variables and `--build-arg` at build time:

```dockerfile
ARG JAR_FILE=target/app.jar
ARG APP_VERSION=unknown

COPY ${JAR_FILE} app.jar

LABEL version="${APP_VERSION}"
```

```bash
docker build \
  --build-arg JAR_FILE=target/myservice-2.1.0.jar \
  --build-arg APP_VERSION=2.1.0 \
  -t myapp:2.1.0 .
```

**Key difference from ENV:** `ARG` values are only available during the build; they do not persist in the image or running container. Use `ENV` for runtime environment variables.

**Security warning:** `ARG` values appear in `docker history` — never pass secrets via `ARG`.

</details>

---

### Q19 🟡 How do you run a Spring Boot app container with a specific Spring profile?

<details><summary>Click to reveal answer</summary>

```bash
# Via -e flag
docker run -e SPRING_PROFILES_ACTIVE=staging -p 8080:8080 myapp:1.0.0

# Via docker-compose.yml
services:
  app:
    environment:
      SPRING_PROFILES_ACTIVE: staging
```

Spring Boot maps environment variables to properties using relaxed binding:
- `SPRING_PROFILES_ACTIVE` → `spring.profiles.active`
- `SPRING_DATASOURCE_URL` → `spring.datasource.url`

You can also set it in the Dockerfile as a default (overridable at runtime):
```dockerfile
ENV SPRING_PROFILES_ACTIVE=production
```

</details>

---

### Q20 🟢 What is `docker inspect` used for?

<details><summary>Click to reveal answer</summary>

`docker inspect` returns detailed JSON metadata about a container, image, network, or volume.

```bash
docker inspect myapp              # full JSON dump
docker inspect myapp | jq '.[0].NetworkSettings.IPAddress'  # container IP
docker inspect myapp | jq '.[0].State'                      # running state
docker inspect myapp | jq '.[0].Mounts'                     # volume mounts
docker inspect myapp | jq '.[0].Config.Env'                 # env vars
docker inspect myimage:1.0.0     # image layers and config
```

Useful for debugging: port mappings, environment variables, network config, restart policy, and whether a container exited and why.

</details>

---

### Q21 🟡 What does `docker stop` do differently from `docker kill`?

<details><summary>Click to reveal answer</summary>

**`docker stop`** sends `SIGTERM` to PID 1 and waits (default 10 seconds) for the container to exit gracefully. If it doesn't exit within the grace period, Docker sends `SIGKILL`.

```bash
docker stop myapp          # SIGTERM, 10s grace period
docker stop -t 30 myapp    # SIGTERM, 30s grace period
```

**`docker kill`** sends `SIGKILL` immediately (or a specified signal):

```bash
docker kill myapp              # immediate SIGKILL
docker kill -s SIGTERM myapp   # send SIGTERM manually
```

For Spring Boot apps with graceful shutdown enabled (`server.shutdown=graceful`), always use `docker stop` with adequate timeout — `docker kill` bypasses graceful shutdown entirely.

</details>

---

### Q22 🟡 How do you build and run a Docker image for a Maven project without multi-stage builds?

<details><summary>Click to reveal answer</summary>

```bash
# Step 1: Build the JAR on the host
mvn clean package -DskipTests

# Step 2: Dockerfile (single stage)
```

```dockerfile
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY target/myapp-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# Step 3: Build image
docker build -t myapp:1.0.0 .

# Step 4: Run
docker run -d -p 8080:8080 myapp:1.0.0
```

**Downsides of single-stage:** Requires Maven and JDK on the host machine (breaks "works on my machine"). Multi-stage builds (Chapter 2) solve this by building inside Docker.

</details>

---

### Q23 🔴 How would you debug a container that starts and immediately exits?

<details><summary>Click to reveal answer</summary>

```bash
# 1. Check the exit code and logs
docker ps -a                          # see container status and exit code
docker logs <container_id>            # see what was printed before exit

# 2. Override the entrypoint to get a shell
docker run --rm -it --entrypoint /bin/sh myapp:1.0.0

# 3. Run with no command (inspect the image config)
docker inspect myapp:1.0.0 | jq '.[0].Config'

# 4. Check environment variables
docker run --rm myapp:1.0.0 env

# 5. Start with verbose JVM output
docker run --rm -e JAVA_OPTS="-verbose:class" myapp:1.0.0
```

**Common causes:**
- Missing JAR file (wrong path in `COPY`)
- Missing environment variable (e.g., `DB_URL` not set)
- Port already in use (check exit code 1)
- OOM on startup (exit code 137 = SIGKILL, often OOM)
- Missing entrypoint binary (e.g., `/bin/bash` not present in distroless)

</details>

---

### Q24 🟡 How do you reduce Docker image size for a Java application?

<details><summary>Click to reveal answer</summary>

1. **Use JRE instead of JDK** at runtime — saves ~70 MB
2. **Use multi-stage builds** — don't include Maven/Gradle in the final image
3. **Use Alpine or distroless** base image — Alpine saves ~50 MB vs Jammy
4. **Use `.dockerignore`** — don't copy unnecessary files
5. **Minimize layers** — combine related `RUN` commands:

```dockerfile
# BAD
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

6. **Use `--no-install-recommends`** for apt packages
7. **Use `jlink`** to create a custom minimal JRE with only needed modules

</details>

---

### Q25 🟡 What is a Docker registry and how do you push/pull images?

<details><summary>Click to reveal answer</summary>

A Docker registry stores and distributes Docker images. Docker Hub is the default public registry. Others include:
- **GitHub Container Registry (ghcr.io)**
- **Amazon ECR**
- **Google Artifact Registry**
- **Self-hosted:** Harbor, Nexus

```bash
# Login
docker login ghcr.io -u USERNAME --password-stdin <<< "$GITHUB_TOKEN"

# Tag image for a registry
docker tag myapp:1.0.0 ghcr.io/myorg/myapp:1.0.0

# Push
docker push ghcr.io/myorg/myapp:1.0.0

# Pull
docker pull ghcr.io/myorg/myapp:1.0.0

# Pull specific digest (immutable)
docker pull ghcr.io/myorg/myapp@sha256:abc123...
```

In CI/CD, always push images tagged with the Git commit SHA for traceability.

</details>
