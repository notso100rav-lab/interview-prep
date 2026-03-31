# Mini Guide: Deploying a Microservice to Production

> Java Backend Engineer Interview Prep — Chapter 4.8

---

## Overview

This end-to-end walkthrough covers the full journey of a Spring Boot microservice from local code to a production-like environment. Each phase maps to a real-world engineering concern.

```
Code → Docker → Container Registry → Kubernetes → Monitoring → CI/CD
```

---

## Table of Contents
1. [The Application](#the-application)
2. [Dockerizing the Service](#dockerizing-the-service)
3. [Pushing to a Container Registry](#pushing-to-a-container-registry)
4. [Kubernetes Manifests](#kubernetes-manifests)
5. [Configuration & Secrets](#configuration--secrets)
6. [Health Checks & Readiness](#health-checks--readiness)
7. [Observability: Logs, Metrics & Traces](#observability-logs-metrics--traces)
8. [CI/CD Pipeline](#cicd-pipeline)
9. [Zero-Downtime Deployments](#zero-downtime-deployments)
10. [Q&A](#qa)

---

## The Application

A minimal Spring Boot order service that other services depend on.

```
order-service/
├── src/main/java/com/example/orders/
│   ├── OrderServiceApplication.java
│   ├── controller/OrderController.java
│   ├── service/OrderService.java
│   └── repository/OrderRepository.java
├── src/main/resources/
│   ├── application.yml
│   └── application-prod.yml
├── pom.xml
└── Dockerfile
```

```yaml
# application.yml
spring:
  application:
    name: order-service
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/orders}
    username: ${DB_USER:orders}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      connection-timeout: 3000
  jpa:
    open-in-view: false

server:
  port: 8080
  shutdown: graceful   # waits for in-flight requests on SIGTERM

spring.lifecycle.timeout-per-shutdown-phase: 30s

management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  endpoint.health.probes.enabled: true
```

---

## Dockerizing the Service

### Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /build

COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
# Download dependencies first (cache layer)
RUN ./mvnw dependency:go-offline -q

COPY src ./src
RUN ./mvnw package -DskipTests -q

# Stage 2: Extract layers for better caching
FROM builder AS layertools
RUN java -Djarmode=layertools -jar target/order-service.jar extract --destination /layers

# Stage 3: Runtime image
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy extracted layers (most-stable layers first)
COPY --from=layertools --chown=appuser:appgroup /layers/dependencies ./
COPY --from=layertools --chown=appuser:appgroup /layers/spring-boot-loader ./
COPY --from=layertools --chown=appuser:appgroup /layers/snapshot-dependencies ./
COPY --from=layertools --chown=appuser:appgroup /layers/application ./

USER appuser
EXPOSE 8080

# Container-aware JVM flags
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

```bash
# Build and verify
docker build -t order-service:local .
docker run --rm -e DB_PASSWORD=test order-service:local --spring.profiles.active=local
```

---

## Pushing to a Container Registry

### AWS ECR

```bash
# Authenticate
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    123456789.dkr.ecr.us-east-1.amazonaws.com

# Tag with semantic version and Git SHA
VERSION=1.2.3
SHA=$(git rev-parse --short HEAD)
IMAGE=123456789.dkr.ecr.us-east-1.amazonaws.com/order-service

docker tag order-service:local $IMAGE:$VERSION
docker tag order-service:local $IMAGE:$SHA
docker tag order-service:local $IMAGE:latest

docker push $IMAGE:$VERSION
docker push $IMAGE:$SHA
docker push $IMAGE:latest
```

**ECR lifecycle policy** — keep only the last 30 tagged images to control storage costs:

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep last 30 tagged images",
    "selection": {
      "tagStatus": "tagged",
      "tagPrefixList": ["v"],
      "countType": "imageCountMoreThan",
      "countNumber": 30
    },
    "action": { "type": "expire" }
  }]
}
```

---

## Kubernetes Manifests

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: orders
  labels:
    app.kubernetes.io/managed-by: kubectl
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: orders
  labels:
    app: order-service
    version: "1.2.3"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # one extra pod during update
      maxUnavailable: 0  # never reduce below desired replicas
  template:
    metadata:
      labels:
        app: order-service
        version: "1.2.3"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: order-service
      terminationGracePeriodSeconds: 60

      containers:
        - name: order-service
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:1.2.3
          ports:
            - containerPort: 8080
              name: http

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: DB_URL
              valueFrom:
                configMapKeyRef:
                  name: order-service-config
                  key: db-url
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: db-user
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: db-password

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "768Mi"

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
            initialDelaySeconds: 15
            periodSeconds: 5
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]  # wait for LB drain

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [order-service]
                topologyKey: kubernetes.io/hostname
```

### Service & HPA

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: orders
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
      name: http
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
  namespace: orders
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Configuration & Secrets

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: orders
data:
  db-url: "jdbc:postgresql://postgres.orders.svc.cluster.local:5432/orders"
  log-level: "INFO"
```

```yaml
# secret.yaml — base64 encoded values
# In practice, use Sealed Secrets, External Secrets Operator, or AWS Secrets Manager CSI Driver
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: orders
type: Opaque
data:
  db-user: b3JkZXJz          # echo -n "orders" | base64
  db-password: czNjcjN0      # echo -n "s3cr3t" | base64
```

**Production secret management options:**

| Tool | How it works |
|------|-------------|
| AWS Secrets Manager + CSI Driver | Mounts secrets as files/env vars from AWS |
| External Secrets Operator | Syncs AWS/GCP/Vault secrets into K8s Secrets |
| Sealed Secrets (Bitnami) | Encrypts secrets safe to commit to Git |
| HashiCorp Vault | Full secret lifecycle management |

---

## Health Checks & Readiness

Spring Boot Actuator exposes Kubernetes-native probes automatically when `management.endpoint.health.probes.enabled=true`:

| Endpoint | Kubernetes probe | Fails when |
|----------|-----------------|------------|
| `/actuator/health/liveness` | `livenessProbe` | App is in broken state; should restart |
| `/actuator/health/readiness` | `readinessProbe` | App can't serve traffic (DB down, warming up) |

```java
// Custom readiness indicator
@Component
public class WarmupReadinessIndicator implements ReadinessStateHealthIndicator {

    private final AtomicBoolean warmedUp = new AtomicBoolean(false);

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        // Pre-warm caches, connection pools, etc.
        warmupCaches();
        warmedUp.set(true);
    }

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        if (warmedUp.get()) {
            builder.up();
        } else {
            builder.down().withDetail("reason", "warming up");
        }
    }
}
```

---

## Observability: Logs, Metrics & Traces

### Structured Logging (Logback + JSON)

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="prod">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <includeMdcKeyName>userId</includeMdcKeyName>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="STDOUT"/>
    </root>
  </springProfile>
</configuration>
```

### Distributed Tracing (Micrometer Tracing + Zipkin)

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry.instrumentation</groupId>
  <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # 10% sampling in production
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

### Prometheus Metrics

```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final MeterRegistry meterRegistry;
    private final Counter orderCounter;

    @PostMapping("/orders")
    public ResponseEntity<OrderDto> createOrder(@Valid @RequestBody CreateOrderRequest req) {
        // Automatic HTTP metrics from Micrometer
        // Custom business metric:
        orderCounter.increment();
        return ResponseEntity.ok(orderService.create(req));
    }
}

@Configuration
public class MetricsConfig {
    @Bean
    public Counter orderCounter(MeterRegistry registry) {
        return Counter.builder("orders.created")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);
    }
}
```

---

## CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy Order Service

on:
  push:
    branches: [main]
    paths: ["services/order-service/**"]

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  IMAGE_NAME: order-service
  EKS_CLUSTER: prod-cluster
  K8S_NAMESPACE: orders

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: 'temurin' }
      - run: ./mvnw verify
        working-directory: services/order-service/

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsECR
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: services/order-service/
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsEKS
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/order-service \
            order-service=${{ env.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-push.outputs.image-tag }} \
            -n ${{ env.K8S_NAMESPACE }}

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/order-service \
            -n ${{ env.K8S_NAMESPACE }} \
            --timeout=300s
```

---

## Zero-Downtime Deployments

### Rolling Update (default)

```bash
# Kubernetes performs a rolling update automatically on image change
kubectl set image deployment/order-service order-service=<new-image>

# Monitor
kubectl rollout status deployment/order-service -n orders
kubectl rollout history deployment/order-service -n orders

# Rollback if something goes wrong
kubectl rollout undo deployment/order-service -n orders
kubectl rollout undo deployment/order-service --to-revision=3 -n orders
```

### Key settings for zero downtime

1. **`maxUnavailable: 0`** — never take pods out of rotation before new ones are ready
2. **`readinessProbe`** — Kubernetes only routes traffic to pods that pass readiness
3. **`preStop` sleep** — gives the load balancer time to drain connections before the pod stops accepting
4. **`terminationGracePeriodSeconds: 60`** — gives in-flight requests time to complete after SIGTERM
5. **`server.shutdown: graceful`** (Spring Boot) — stops accepting new requests but finishes existing ones

### Graceful shutdown sequence

```
Kubernetes sends SIGTERM
    ↓
preStop hook runs (sleep 5s — LB removes pod from rotation)
    ↓
Spring Boot stops accepting new requests
    ↓
In-flight requests complete (up to 30s)
    ↓
Application exits (code 0)
    ↓
Pod is marked Terminated
```

---

## Q&A

---

### Q1 — 🟢 What is the difference between `livenessProbe` and `readinessProbe` in Kubernetes?

<details><summary>Click to reveal answer</summary>

| Probe | Kubernetes action on failure | Use for |
|-------|------------------------------|---------|
| `livenessProbe` | **Restarts** the container | Detecting deadlocks, OOM, stuck threads — app is broken |
| `readinessProbe` | **Removes** the pod from Service endpoints (no traffic) | App is alive but not ready: DB not connected, cache warming, circuit breaker open |

**Spring Boot mapping:**
- `/actuator/health/liveness` → `ApplicationAvailability.LivenessState`
- `/actuator/health/readiness` → `ApplicationAvailability.ReadinessState`

Never use the same endpoint for both — a liveness failure causes a restart (pod storm risk), while readiness failure just stops traffic.

</details>

---

### Q2 — 🟡 Why do we add a `preStop` sleep and `terminationGracePeriodSeconds` to achieve zero downtime?

<details><summary>Click to reveal answer</summary>

When Kubernetes terminates a pod it does two things **simultaneously**:
1. Sends SIGTERM to the container process
2. Removes the pod IP from the Service `Endpoints` object (so no new traffic is routed)

The problem is that kube-proxy (which programs iptables/IPVS rules on each node) may take a few seconds to propagate the endpoint removal. During this window, the load balancer can still forward requests to the pod that is shutting down.

**Solution:**
- `preStop: sleep 5` — delays SIGTERM by 5 seconds, giving kube-proxy time to drain the pod from routing rules before the app starts refusing connections.
- `terminationGracePeriodSeconds: 60` — gives in-flight requests time to complete after SIGTERM is received.
- `server.shutdown: graceful` (Spring Boot) — the app stops accepting new connections but finishes active requests.

The order: preStop → SIGTERM → graceful shutdown → SIGKILL (if grace period expires).

</details>

---

### Q3 — 🔴 How would you implement a blue/green deployment with Kubernetes and a Spring Boot service?

<details><summary>Click to reveal answer</summary>

Blue/green keeps two identical production environments (blue = current, green = new). Traffic is switched atomically:

```yaml
# Blue deployment (currently live)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
  labels:
    app: order-service
    slot: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      slot: blue
  template:
    metadata:
      labels:
        app: order-service
        slot: blue
    spec:
      containers:
        - name: order-service
          image: order-service:1.2.3

---
# Green deployment (new version, not yet live)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
  labels:
    app: order-service
    slot: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      slot: green
  template:
    metadata:
      labels:
        app: order-service
        slot: green
    spec:
      containers:
        - name: order-service
          image: order-service:1.3.0

---
# Service initially points to blue
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    slot: blue   # ← switch to "green" to cut over
  ports:
    - port: 80
      targetPort: 8080
```

**Cutover steps:**
1. Deploy green and verify health (smoke tests, canary analysis)
2. `kubectl patch service order-service -p '{"spec":{"selector":{"slot":"green"}}}'`
3. Monitor error rates; if issues, patch back to `slot: blue` (instant rollback)
4. Scale down blue after confidence period

**Advantage over rolling:** instant full cutover and instant rollback; no mixed versions serving traffic simultaneously (important for incompatible API changes).

</details>
