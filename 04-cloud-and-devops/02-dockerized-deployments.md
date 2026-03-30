# Dockerized Deployments

> Java Backend Engineer Interview Prep — Chapter 4.2

---

## Table of Contents
1. [Full docker-compose.yml Stack](#full-docker-composeyml-stack)
2. [Environment Variables & .env Files](#environment-variables--env-files)
3. [Override Files (Dev vs Prod)](#override-files-dev-vs-prod)
4. [Secrets Management](#secrets-management)
5. [Container Networking](#container-networking)
6. [Scaling with Docker Compose](#scaling-with-docker-compose)
7. [Q&A](#qa)

---

## Full docker-compose.yml Stack

Complete production-grade `docker-compose.yml` for Spring Boot + PostgreSQL + Redis + Nginx:

```yaml
# docker-compose.yml
version: "3.9"

networks:
  backend:
    driver: bridge
    name: myapp-network
  frontend:
    driver: bridge

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  nginx_logs:
    driver: local

services:

  # ─────────────────────────────────────────────
  # PostgreSQL
  # ─────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: myapp-postgres
    restart: unless-stopped
    networks:
      - backend
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-myapp}
      POSTGRES_USER: ${POSTGRES_USER:-myapp}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}
      PGDATA: /var/lib/postgresql/data/pgdata
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-myapp} -d ${POSTGRES_DB:-myapp}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    ports:
      - "${POSTGRES_PORT:-5432}:5432"

  # ─────────────────────────────────────────────
  # Redis
  # ─────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    networks:
      - backend
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf --requirepass ${REDIS_PASSWORD:-changeme}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-changeme}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s
    ports:
      - "${REDIS_PORT:-6379}:6379"

  # ─────────────────────────────────────────────
  # Spring Boot Application
  # ─────────────────────────────────────────────
  app:
    image: ${DOCKER_IMAGE:-myapp}:${APP_VERSION:-latest}
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
      args:
        - APP_VERSION=${APP_VERSION:-dev}
    container_name: myapp-app
    restart: unless-stopped
    networks:
      - backend
      - frontend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-prod}
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-myapp}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER:-myapp}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD:-changeme}
      SERVER_PORT: 8080
      JAVA_OPTS: >-
        -Xms${JVM_XMS:-256m}
        -Xmx${JVM_XMX:-512m}
        -XX:+UseG1GC
        -XX:+ExitOnOutOfMemoryError
        -Djava.security.egd=file:/dev/./urandom
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 768M
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.25"
    ports:
      - "${APP_PORT:-8080}:8080"
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"

  # ─────────────────────────────────────────────
  # Nginx Reverse Proxy
  # ─────────────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    restart: unless-stopped
    networks:
      - frontend
    depends_on:
      app:
        condition: service_healthy
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
    ports:
      - "80:80"
      - "443:443"
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 5s
      retries: 3
```

### Nginx Configuration

```nginx
# nginx/conf.d/app.conf
upstream spring_backend {
    least_conn;
    server app:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com;

    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location /actuator/health {
        proxy_pass http://spring_backend;
        access_log off;
    }

    location / {
        proxy_pass http://spring_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
    }
}
```

---

## Environment Variables & .env Files

### .env File

```bash
# .env — committed to git (no secrets, only defaults/structure)
POSTGRES_DB=myapp
POSTGRES_USER=myapp
POSTGRES_PORT=5432
REDIS_PORT=6379
APP_PORT=8080
SPRING_PROFILES_ACTIVE=prod
JVM_XMS=256m
JVM_XMX=512m
DOCKER_IMAGE=myapp
APP_VERSION=latest
```

```bash
# .env.local — NOT committed to git (actual secrets)
POSTGRES_PASSWORD=supersecretpassword123
REDIS_PASSWORD=redissecretpassword456
```

```bash
# .gitignore
.env.local
.env.prod
*.env.secret
```

### Variable Substitution Syntax

```yaml
# Syntax options in docker-compose.yml:

# Required — fails if not set
DB_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD must be set}

# Optional with default
DB_NAME: ${DB_NAME:-myapp}

# Optional, empty string if not set
FEATURE_FLAG: ${FEATURE_FLAG-}

# Use from shell environment (no default)
AWS_REGION: ${AWS_REGION}
```

### Loading Multiple .env Files

```bash
# Docker Compose 2.x supports multiple env files
docker compose --env-file .env --env-file .env.local up -d

# Or via compose file
```

```yaml
# docker-compose.yml (Compose spec)
services:
  app:
    env_file:
      - .env
      - .env.local        # overrides .env
      - path: .env.prod   # only loaded if exists
        required: false
```

---

## Override Files (Dev vs Prod)

### Base File (docker-compose.yml)

Contains shared service definitions. Already shown above.

### Development Override (docker-compose.override.yml)

Docker Compose automatically merges `docker-compose.override.yml` when you run `docker compose up`:

```yaml
# docker-compose.override.yml (auto-loaded in dev)
version: "3.9"

services:
  app:
    build:
      target: development    # use dev build stage
    volumes:
      - ./src:/workspace/src  # hot reload with spring-boot-devtools
      - ~/.m2:/root/.m2       # cache maven repo
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DEVTOOLS_RESTART_ENABLED: "true"
      JAVA_OPTS: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
    ports:
      - "5005:5005"    # remote debug port
      - "8080:8080"

  postgres:
    ports:
      - "5432:5432"    # expose DB port in dev

  redis:
    ports:
      - "6379:6379"

  # Add pgAdmin in dev only
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@local.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    networks:
      - backend
```

### Production Override (docker-compose.prod.yml)

```yaml
# docker-compose.prod.yml — explicitly loaded in CI/prod
version: "3.9"

services:
  app:
    image: ${DOCKER_IMAGE}:${APP_VERSION}  # pre-built image, no build in prod
    restart: always
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        max_attempts: 3

  postgres:
    ports: []   # remove exposed port in prod — DB internal only

  redis:
    ports: []   # same

  nginx:
    deploy:
      resources:
        limits:
          memory: 128M
```

```bash
# Dev (auto-loads override file)
docker compose up -d

# Production (explicit file selection)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# CI environment
docker compose -f docker-compose.yml -f docker-compose.prod.yml \
  --env-file .env --env-file .env.prod \
  up -d --no-build
```

---

## Secrets Management

### Option 1: Docker Secrets (Swarm Mode)

```yaml
# docker-compose.yml with Docker secrets
version: "3.9"

secrets:
  db_password:
    external: true     # created via `docker secret create`
  redis_password:
    file: ./secrets/redis_password.txt

services:
  app:
    secrets:
      - db_password
      - redis_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
```

```java
// Reading Docker secret in Spring Boot
@Configuration
public class SecretConfig {
    @Bean
    public String dbPassword() throws IOException {
        String secretFile = System.getenv("DB_PASSWORD_FILE");
        if (secretFile != null) {
            return Files.readString(Paths.get(secretFile)).trim();
        }
        return System.getenv("DB_PASSWORD");
    }
}
```

```bash
# Create Docker secret
echo "supersecretpassword" | docker secret create db_password -
```

### Option 2: External Secret Managers

```yaml
# Using HashiCorp Vault via environment injection
services:
  vault-agent:
    image: hashicorp/vault:latest
    command: agent -config=/vault/config/agent.hcl
    volumes:
      - ./vault/agent.hcl:/vault/config/agent.hcl
      - shared_secrets:/run/secrets

  app:
    volumes:
      - shared_secrets:/run/secrets:ro
    environment:
      DB_PASSWORD_FILE: /run/secrets/db-password
```

### Option 3: .env.secret (Never Commit)

```bash
# .env.secret
POSTGRES_PASSWORD=supersecret
REDIS_PASSWORD=anothersecret
JWT_SECRET=jwt-signing-key

# Load in compose
docker compose --env-file .env --env-file .env.secret up -d
```

---

## Container Networking

### Service DNS Resolution

In a Docker Compose network, services communicate using their **service name** as the hostname:

```yaml
services:
  app:
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/myapp
      #                                         ^^^^^^^^ service name = DNS hostname
      SPRING_DATA_REDIS_HOST: redis
      #                        ^^^^^ service name
```

```java
// Spring Boot properties using service names
@Value("${SPRING_DATASOURCE_URL}")
private String datasourceUrl;
// Value: jdbc:postgresql://postgres:5432/myapp
// Docker resolves "postgres" to container IP
```

### Network Isolation

```yaml
networks:
  backend:    # internal — postgres, redis, app communicate here
    internal: true    # no external access
  frontend:   # external — nginx, app communicate here
    driver: bridge

services:
  postgres:
    networks:
      - backend    # only backend, not reachable from nginx

  app:
    networks:
      - backend    # can talk to postgres and redis
      - frontend   # can receive traffic from nginx

  nginx:
    networks:
      - frontend   # can only reach app, not postgres/redis
```

### Custom Bridge Network with Subnet

```yaml
networks:
  myapp-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
    driver_opts:
      com.docker.network.bridge.name: myapp-bridge
```

---

## Scaling with Docker Compose

```bash
# Scale app service to 3 instances
docker compose up -d --scale app=3

# Note: must remove container_name from service definition to allow multiple instances
# And use a load balancer (nginx with upstream)
```

```yaml
# docker-compose.yml for scaling (no container_name, no fixed host port)
services:
  app:
    # container_name: myapp-app   <-- REMOVE for scaling
    ports: []                     # <-- REMOVE fixed port mapping
    # expose the port instead
    expose:
      - "8080"
```

```nginx
# nginx upstream auto-discovers scaled containers via Docker DNS
upstream spring_backend {
    server app:8080;    # Docker DNS round-robins to all "app" containers
}
```

```bash
# Full scaling workflow
docker compose up -d --scale app=1          # start with 1
docker compose up -d --scale app=3 --no-recreate  # scale up without restarting existing
docker compose ps                            # view running instances
docker compose up -d --scale app=1 --no-recreate  # scale down
```

---

## Q&A

### Q1 🟢 What does `depends_on: condition: service_healthy` mean in Docker Compose?

<details><summary>Click to reveal answer</summary>

It tells Docker Compose to **wait until the dependency's healthcheck passes** before starting the dependent service. Without `condition: service_healthy`, the default `condition: service_started` only waits for the container to start — not for the application inside to be ready.

```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy   # waits for pg_isready to pass
      redis:
        condition: service_healthy   # waits for redis PING to succeed
```

The healthcheck must be defined on the dependency:
```yaml
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

Without this, Spring Boot would crash on startup trying to connect to a not-yet-ready database.

</details>

---

### Q2 🟢 What is the difference between `expose` and `ports` in docker-compose.yml?

<details><summary>Click to reveal answer</summary>

- **`ports`**: Maps container port to **host machine** port. Accessible from outside Docker.
  ```yaml
  ports:
    - "8080:8080"   # host:container
  ```

- **`expose`**: Makes the port available to **other containers in the same network** only. The host machine cannot access it.
  ```yaml
  expose:
    - "8080"
  ```

**Rule of thumb**:
- Use `ports` for services that need external access (Nginx, dev DB access)
- Use `expose` for internal-only services (app behind a reverse proxy)
- In production, only expose ports 80/443 on Nginx — everything else uses `expose` or no declaration

</details>

---

### Q3 🟡 Explain the difference between named volumes and bind mounts.

<details><summary>Click to reveal answer</summary>

| Feature | Named Volume | Bind Mount |
|---------|-------------|------------|
| **Managed by** | Docker | Host OS |
| **Location** | Docker's storage area | Specific host path |
| **Portability** | High | Low (host-dependent) |
| **Use case** | Persistent data (DB, Redis) | Dev hot-reload, config files |
| **Backup** | `docker volume` commands | Regular filesystem |
| **Performance** | Better on Linux | Same on Linux, slower on Mac/Win |

```yaml
volumes:
  # Named volume (Docker manages location)
  postgres_data:
    driver: local

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data  # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # bind mount (read-only)
      - /host/path:/container/path  # absolute bind mount
```

Named volumes persist between `docker compose down` and `up`. To delete: `docker compose down -v`.

</details>

---

### Q4 🟡 How do you handle database migrations with Docker Compose?

<details><summary>Click to reveal answer</summary>

**Option 1: Flyway/Liquibase in Spring Boot** (most common)

Spring Boot runs migrations automatically on startup. Since `app` depends on `postgres` with `service_healthy`, the DB is ready before migrations run:

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

**Option 2: Init scripts via volume mount**

```yaml
postgres:
  volumes:
    - ./db/init:/docker-entrypoint-initdb.d:ro
    # Files executed alphabetically on first start only
    # 01-schema.sql, 02-seed.sql, etc.
```

**Option 3: Migration service (runs once and exits)**

```yaml
services:
  flyway:
    image: flyway/flyway:latest
    command: migrate
    volumes:
      - ./db/migration:/flyway/sql
    environment:
      FLYWAY_URL: jdbc:postgresql://postgres:5432/myapp
      FLYWAY_USER: myapp
      FLYWAY_PASSWORD: ${POSTGRES_PASSWORD}
    depends_on:
      postgres:
        condition: service_healthy
```

</details>

---

### Q5 🟡 What is the purpose of `restart: unless-stopped` and how does it differ from `always`?

<details><summary>Click to reveal answer</summary>

| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `always` | Always restart, even if manually stopped |
| `unless-stopped` | Restart unless explicitly stopped with `docker stop` |
| `on-failure[:N]` | Restart only on non-zero exit code, max N times |

```yaml
services:
  app:
    restart: unless-stopped   # recommended for production services
```

**Key difference**: If you run `docker stop myapp`, then reboot the host:
- `always` → container starts again after reboot
- `unless-stopped` → container does NOT start after reboot (respects manual stop)

For production workloads: `unless-stopped` or `always`. For one-time jobs: `on-failure:3`.

</details>

---

### Q6 🔴 How do you perform a zero-downtime rolling update with Docker Compose?

<details><summary>Click to reveal answer</summary>

Docker Compose alone doesn't provide true rolling updates like Kubernetes. But you can approximate it:

```bash
# 1. Scale to 2 instances (if starting from 1)
docker compose up -d --scale app=2 --no-recreate

# 2. Pull new image
docker compose pull app

# 3. Update one container at a time
docker compose up -d --scale app=2 --no-recreate

# Using docker compose (v2) rolling approach:
docker compose up -d app --no-recreate
docker compose stop app    # stops old container

# Better: use --wait flag
docker compose up -d --wait app   # waits for healthcheck before returning
```

**Proper approach with Docker Swarm mode:**
```yaml
# docker-compose.yml (Swarm)
services:
  app:
    deploy:
      replicas: 3
      update_config:
        parallelism: 1      # update 1 at a time
        delay: 15s          # wait 15s between each
        failure_action: rollback
        monitor: 30s        # monitor for 30s after update
      rollback_config:
        parallelism: 1
        delay: 10s
```

```bash
docker stack deploy -c docker-compose.yml myapp
docker service update --image myapp:v2 myapp_app
```

For true production zero-downtime, prefer Kubernetes or ECS.

</details>

---

### Q7 🟢 How do you view logs from a Docker Compose service?

<details><summary>Click to reveal answer</summary>

```bash
# All services
docker compose logs

# Specific service
docker compose logs app

# Follow/tail logs
docker compose logs -f app

# Last 100 lines
docker compose logs --tail=100 app

# With timestamps
docker compose logs -t app

# Multiple services
docker compose logs -f app nginx

# Since a time
docker compose logs --since 30m app
```

For production, configure log rotation to prevent disk exhaustion:

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "50m"      # max file size
        max-file: "3"        # keep 3 rotated files
```

Or use centralized logging:
```yaml
    logging:
      driver: "awslogs"
      options:
        awslogs-group: "/myapp/prod"
        awslogs-region: "us-east-1"
        awslogs-stream-prefix: "app"
```

</details>

---

### Q8 🟡 What is the difference between `docker compose down` and `docker compose stop`?

<details><summary>Click to reveal answer</summary>

| Command | Containers | Networks | Volumes | Images |
|---------|-----------|----------|---------|--------|
| `stop` | Stopped | Preserved | Preserved | Preserved |
| `down` | Removed | Removed | Preserved | Preserved |
| `down -v` | Removed | Removed | **Removed** | Preserved |
| `down --rmi all` | Removed | Removed | Preserved | **Removed** |

```bash
docker compose stop          # stop containers, keep everything else
docker compose start         # restart stopped containers

docker compose down          # stop + remove containers and networks
docker compose up -d         # recreate from scratch

docker compose down -v       # ⚠️ also deletes named volumes (DATA LOSS)
docker compose down --rmi all -v  # nuclear option: delete everything
```

</details>

---

### Q9 🔴 How would you implement health-check-based traffic shifting with Nginx in Docker Compose?

<details><summary>Click to reveal answer</summary>

Configure Nginx to only route to healthy upstream servers:

```nginx
upstream spring_backend {
    server app_1:8080 max_fails=3 fail_timeout=30s;
    server app_2:8080 max_fails=3 fail_timeout=30s;
    server app_3:8080 max_fails=3 fail_timeout=30s backup;  # only if others fail
    keepalive 16;
}

server {
    location / {
        proxy_pass http://spring_backend;
        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_next_upstream_tries 2;
    }
}
```

For active health checks (Nginx Plus or OpenResty):
```nginx
upstream spring_backend {
    zone backend 64k;
    server app:8080;
    keepalive 32;
}

# Active health check (Nginx Plus only)
server {
    location /api/ {
        health_check interval=5s fails=3 passes=2 uri=/actuator/health;
        proxy_pass http://spring_backend;
    }
}
```

Open-source alternative: Use **Traefik** as reverse proxy with built-in Docker label-based health checks:

```yaml
services:
  app:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`myapp.com`)"
      - "traefik.http.services.app.loadbalancer.healthcheck.path=/actuator/health"
      - "traefik.http.services.app.loadbalancer.healthcheck.interval=10s"
```

</details>

---

### Q10 🟡 How do you pass secrets to containers without storing them in docker-compose.yml or .env files?

<details><summary>Click to reveal answer</summary>

**Option 1: Docker Secrets (Swarm)**
```bash
echo "mysecretpassword" | docker secret create db_password -
```
```yaml
secrets:
  db_password:
    external: true
services:
  app:
    secrets:
      - db_password
    # Secret available at /run/secrets/db_password
```

**Option 2: External secret manager at runtime**
```yaml
services:
  app:
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        export DB_PASSWORD=$(aws secretsmanager get-secret-value \
          --secret-id myapp/db-password --query SecretString --output text)
        exec java -jar app.jar
```

**Option 3: HashiCorp Vault**
```yaml
services:
  vault-agent:
    image: vault:latest
    command: agent -config=/vault/config.hcl
    volumes:
      - shared-secrets:/run/secrets
  app:
    volumes:
      - shared-secrets:/run/secrets:ro
    environment:
      DB_PASSWORD_FILE: /run/secrets/db-password
```

**Option 4: Environment injection via CI/CD**
```bash
# CI pipeline sets secrets as environment variables
docker compose --env-file <(aws ssm get-parameters-by-path --path /myapp --query "Parameters[*].[Name,Value]" --output text | awk '{print $1"="$2}') up -d
```

</details>

---

### Q11 🟢 What does the `${VAR:-default}` syntax mean in Docker Compose?

<details><summary>Click to reveal answer</summary>

Docker Compose supports shell-style variable expansion in compose files:

| Syntax | Meaning |
|--------|---------|
| `${VAR}` | Use value of VAR; error if not set |
| `${VAR:-default}` | Use VAR if set and non-empty, else `default` |
| `${VAR-default}` | Use VAR if set (even if empty), else `default` |
| `${VAR:?error msg}` | Use VAR if set, else abort with error message |
| `${VAR:+alt}` | Use `alt` if VAR is set and non-empty, else empty |

```yaml
services:
  app:
    environment:
      # Required — compose fails to start if not set
      DB_PASSWORD: ${DB_PASSWORD:?Database password is required}

      # Optional with default
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-prod}

      # Optional, empty if not set (no default)
      FEATURE_FLAGS: ${FEATURE_FLAGS-}

      # Conditional value
      LOG_LEVEL: ${DEBUG:+DEBUG}  # "DEBUG" if $DEBUG is set, else empty
```

</details>

---

### Q12 🟡 How does container DNS resolution work in Docker Compose?

<details><summary>Click to reveal answer</summary>

Docker Compose creates a **custom bridge network** for all services. Each service gets:
1. A DNS entry using its **service name** as hostname
2. Automatic DNS resolution within the network

```yaml
services:
  app:
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/myapp
      #                                         ^^^^^^^^
      # "postgres" resolves to the postgres container's IP via Docker's embedded DNS server (127.0.0.11)
```

**How it works:**
- Docker runs a DNS server at `127.0.0.11` inside each container
- Service names are registered with this DNS server
- Containers query `127.0.0.11` for name resolution
- If multiple containers share a service name (scaled), DNS round-robins between them

```bash
# Test DNS from within a container
docker compose exec app nslookup postgres
docker compose exec app curl http://postgres:5432
```

**Service aliases** for custom hostnames:
```yaml
services:
  app:
    networks:
      backend:
        aliases:
          - api
          - myapp-service
```

</details>

---

### Q13 🔴 What are the security best practices for Docker Compose in production?

<details><summary>Click to reveal answer</summary>

```yaml
services:
  app:
    # 1. Run as non-root user
    user: "1001:1001"

    # 2. Read-only filesystem (write only to explicit volumes)
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs

    # 3. Drop Linux capabilities
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # only if binding to port < 1024

    # 4. No new privileges
    security_opt:
      - no-new-privileges:true

    # 5. Resource limits (prevent DoS)
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

    # 6. Health check (required for depends_on)
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
      interval: 30s

    # 7. No secrets in environment — use secrets or external vault
    # BAD:  environment: DB_PASSWORD: secret123
    # GOOD: environment: DB_PASSWORD_FILE: /run/secrets/db_password

    # 8. Network isolation
    networks:
      - backend  # not on frontend network

  nginx:
    # Only service on frontend network
    networks:
      - frontend
      - backend

networks:
  backend:
    internal: true   # no external access to backend network
  frontend:
    driver: bridge
```

</details>

---

### Q14 🟢 How do you rebuild and restart a single service in Docker Compose?

<details><summary>Click to reveal answer</summary>

```bash
# Rebuild image and restart service (without stopping others)
docker compose up -d --build app

# Force recreate even if config unchanged
docker compose up -d --force-recreate app

# Pull latest image and recreate
docker compose pull app && docker compose up -d --no-build app

# Stop and remove specific service only
docker compose stop app && docker compose rm -f app

# Restart (reuses existing container)
docker compose restart app

# Check which services need updating (dry run)
docker compose up --dry-run app
```

</details>

---

### Q15 🟡 What is `docker compose watch` and how does it help development?

<function_calls>