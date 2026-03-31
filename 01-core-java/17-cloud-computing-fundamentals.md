# Cloud Computing Fundamentals

> 💡 **Interviewer Tip:** Cloud questions in Java backend interviews focus on *conceptual understanding* and *design decisions*, not memorizing AWS console navigation. Candidates should be able to choose the right service for a use case and explain trade-offs.

---

## Table of Contents
1. [Cloud Service Models](#cloud-service-models)
2. [AWS Core Services](#aws-core-services)
3. [12-Factor App Methodology](#12-factor-app-methodology)
4. [Microservices vs Monolith](#microservices-vs-monolith)
5. [Containerization](#containerization)
6. [Serverless Computing](#serverless-computing)
7. [Infrastructure Concepts](#infrastructure-concepts)

---

## Cloud Service Models

### 🟢 Q1. What are IaaS, PaaS, and SaaS? Provide examples of each.

<details><summary>Click to reveal answer</summary>

The three primary cloud service models define *how much* the provider manages vs. how much you manage:

```
Responsibility         IaaS          PaaS           SaaS
─────────────────────────────────────────────────────────
Application            YOU           YOU            Provider
Data                   YOU           YOU            Provider
Runtime                YOU           Provider       Provider
Middleware             YOU           Provider       Provider
OS                     YOU           Provider       Provider
Virtualization         Provider      Provider       Provider
Servers                Provider      Provider       Provider
Storage                Provider      Provider       Provider
Networking             Provider      Provider       Provider
```

**IaaS (Infrastructure as a Service)**
- Provider gives you: virtual machines, storage, networking
- You manage: OS, runtime, middleware, applications
- Examples: **AWS EC2**, Google Compute Engine, Azure VMs, DigitalOcean Droplets
- Use when: Maximum control needed, custom OS configs, legacy app migration

**PaaS (Platform as a Service)**
- Provider gives you: managed runtime, OS, middleware, scaling
- You manage: application code and data only
- Examples: **AWS Elastic Beanstalk**, Google App Engine, Azure App Service, Heroku
- Use when: Focus on code, not infrastructure; standard web apps

**SaaS (Software as a Service)**
- Provider gives you: fully managed software application
- You manage: data and configuration only
- Examples: **Gmail**, Salesforce, Slack, GitHub, Jira, Snowflake
- Use when: Off-the-shelf functionality, no custom development needed

```
Real-world analogy (Pizza):
IaaS:  You have the kitchen, oven, and ingredients. You cook everything.
PaaS:  You order pizza dough and toppings. Restaurant handles the oven and cooking.
SaaS:  You order a fully made pizza delivered to your door.
```

> 💡 **Interviewer Tip:** Know where CaaS (Container as a Service — ECS, Kubernetes) and FaaS (Function as a Service — Lambda) fit in. They blur IaaS/PaaS lines.

</details>

---

### 🟡 Q2. What is the shared responsibility model in cloud security?

<details><summary>Click to reveal answer</summary>

AWS's shared responsibility model divides security duties:

**AWS Responsible For (Security OF the Cloud):**
- Physical data centers (access, power, cooling)
- Hardware (servers, networking equipment)
- Hypervisor and virtualization layer
- Global network infrastructure

**Customer Responsible For (Security IN the Cloud):**
- Data encryption (at rest and in transit)
- IAM policies and access management
- Operating system patches (EC2)
- Network configuration (Security Groups, NACLs)
- Application-level security
- Compliance with regulations

```
Service Type    | AWS Manages More  | You Manage Less
─────────────────────────────────────────────────────
EC2 (IaaS)     | Hardware, hypervisor | OS, patches, app
RDS (PaaS)     | +OS, DB engine patches | App, data, access
DynamoDB (SaaS)| +Everything except data | Data, IAM, encryption
Lambda (FaaS)  | +Runtime, scaling | Function code, IAM
```

**Java application example:**
```yaml
# What YOU must configure for an EC2-hosted Spring Boot app:
- Install and patch JDK
- Configure security groups (allow port 8080, restrict SSH)
- Set up IAM role with least-privilege permissions
- Enable encryption for EBS volumes
- Configure CloudWatch logging
- Manage SSL/TLS certificates
- Apply OS security patches

# What AWS handles:
- Physical server security
- Hypervisor isolation between your EC2 and others
- Network DDoS protection (AWS Shield Standard)
```

</details>

---

## AWS Core Services

### 🟡 Q3. Describe the key AWS compute services and when to use each.

<details><summary>Click to reveal answer</summary>

**EC2 (Elastic Compute Cloud)**
- Virtual machines (instances) in the cloud
- Full OS access, any language/runtime
- Use for: Long-running processes, custom OS config, stateful apps

```bash
# EC2 instance types:
t3.micro    # 2 vCPU, 1 GB RAM — dev/testing
t3.large    # 2 vCPU, 8 GB RAM — small Spring Boot app
c5.4xlarge  # 16 vCPU, 32 GB RAM — CPU-intensive batch processing
r5.4xlarge  # 16 vCPU, 128 GB RAM — memory-intensive (Redis, JVM with large heap)
```

**ECS (Elastic Container Service)**
- Managed Docker container orchestration
- Fargate: serverless containers (no EC2 management)
- Use for: Containerized microservices, consistent deployments

**Lambda**
- Serverless functions, event-driven
- Auto-scales to 0, pay-per-invocation
- Use for: Infrequent workloads, event handlers, ETL, API backends

**Elastic Beanstalk**
- PaaS — deploy code, AWS manages everything
- Supports Java (Tomcat, Corretto)
- Use for: Quick deployments without infrastructure concerns

```java
// Lambda function in Java
public class OrderProcessorHandler implements RequestHandler<SQSEvent, Void> {

    private final OrderService orderService = new OrderService();

    @Override
    public Void handleRequest(SQSEvent event, Context context) {
        event.getRecords().forEach(record -> {
            Order order = parseOrder(record.getBody());
            orderService.process(order);
            context.getLogger().log("Processed order: " + order.getId());
        });
        return null;
    }
}
```

</details>

---

### 🟡 Q4. Explain AWS storage services: S3, EBS, EFS, and when to use each.

<details><summary>Click to reveal answer</summary>

| Service | Type | Access | Use Case |
|---|---|---|---|
| S3 | Object storage | HTTP API | Files, images, backups, static websites |
| EBS | Block storage | Attached to single EC2 | EC2 root volume, databases |
| EFS | File storage (NFS) | Multiple EC2 instances | Shared files across instances |
| RDS | Managed relational DB | JDBC | PostgreSQL, MySQL, Oracle |
| DynamoDB | NoSQL (key-value/document) | SDK/API | High-throughput, flexible schema |

**S3 (Simple Storage Service):**
```java
// Upload to S3
S3Client s3 = S3Client.builder().region(Region.US_EAST_1).build();

// Upload file
PutObjectRequest putRequest = PutObjectRequest.builder()
    .bucket("my-bucket")
    .key("uploads/user-123/profile.jpg")
    .contentType("image/jpeg")
    .serverSideEncryption(ServerSideEncryption.AES256)
    .build();
s3.putObject(putRequest, RequestBody.fromBytes(imageBytes));

// Generate pre-signed URL (temporary access)
S3Presigner presigner = S3Presigner.builder().region(Region.US_EAST_1).build();
GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
    .signatureDuration(Duration.ofHours(1))
    .getObjectRequest(GetObjectRequest.builder()
        .bucket("my-bucket")
        .key("uploads/user-123/profile.jpg")
        .build())
    .build();
PresignedGetObjectRequest presigned = presigner.presignGetObject(presignRequest);
String downloadUrl = presigned.url().toString();

// Spring Boot with @Value
@Value("${aws.s3.bucket}")
private String bucketName;

// S3 storage classes:
// STANDARD         — frequent access, highest cost
// STANDARD_IA      — infrequent access, lower cost
// GLACIER          — archival, retrieval takes hours
// INTELLIGENT_TIERING — auto-moves between tiers
```

</details>

---

### 🟡 Q5. What are SQS and SNS, and how do they differ?

<details><summary>Click to reveal answer</summary>

**SQS (Simple Queue Service)** — Message queue (point-to-point)
- Messages sit in queue until consumed and deleted
- One consumer processes each message
- Decouples producers from consumers
- Types: Standard (at-least-once, unordered) and FIFO (exactly-once, ordered)

**SNS (Simple Notification Service)** — Pub/sub (one-to-many)
- Publishers send messages to a topic
- All subscribers receive the message simultaneously
- Subscribers: SQS, Lambda, HTTP endpoints, email, SMS

```
SQS:  Producer → [Queue] → Consumer (one-to-one)
SNS:  Publisher → [Topic] → Multiple Subscribers (fan-out)
```

```java
// Sending to SQS
SqsClient sqs = SqsClient.builder().region(Region.US_EAST_1).build();

SendMessageRequest request = SendMessageRequest.builder()
    .queueUrl("https://sqs.us-east-1.amazonaws.com/123/order-queue")
    .messageBody(objectMapper.writeValueAsString(new OrderEvent(orderId, "PLACED")))
    .delaySeconds(0)
    .messageGroupId("orders")   // FIFO only
    .messageDeduplicationId(orderId.toString())  // FIFO only
    .build();
sqs.sendMessage(request);

// Spring Cloud AWS — SQS listener
@SqsListener("order-queue")
public void processOrder(@Payload OrderEvent event,
                          @Header("SenderId") String senderId) {
    log.info("Processing order {} from {}", event.getOrderId(), senderId);
    orderService.process(event);
}

// SNS fan-out pattern:
// 1. Order placed → SNS topic: "order-events"
// 2. SQS queue for email service (subscriber 1)
// 3. SQS queue for inventory service (subscriber 2)
// 4. SQS queue for analytics (subscriber 3)
// All receive the same message simultaneously
```

**Common pattern — SNS → SQS fan-out:**
- Decouples event producers from consumers
- SQS provides retry, buffering, and backpressure for each consumer

</details>

---

### 🟡 Q6. What is AWS RDS and what advantages does it offer over self-managed DB?

<details><summary>Click to reveal answer</summary>

**RDS (Relational Database Service)** is a managed relational database service.

**Supported engines:** PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Aurora (AWS proprietary)

**Advantages over self-managed:**

| Feature | Self-managed (EC2) | RDS |
|---|---|---|
| Backups | Manual, you configure | Automated daily + point-in-time |
| Failover | Manual setup | Automatic Multi-AZ failover |
| Patching | You apply patches | AWS applies minor patches |
| Read scaling | Manual replica setup | Read replicas with 1-click |
| Monitoring | Install tools yourself | CloudWatch metrics built-in |
| Encryption | Configure manually | Enable with checkbox |

```yaml
# Spring Boot application.properties for RDS
spring.datasource.url=jdbc:postgresql://${RDS_HOSTNAME}:${RDS_PORT}/${RDS_DB_NAME}
spring.datasource.username=${RDS_USERNAME}
spring.datasource.password=${RDS_PASSWORD}

# Connection pooling (HikariCP)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
```

```java
// RDS IAM authentication (no hardcoded passwords)
@Bean
public DataSource dataSource() {
    RdsIamAuthTokenGenerator tokenGenerator = RdsIamAuthTokenGenerator.builder()
        .credentials(DefaultCredentialsProvider.create())
        .region("us-east-1")
        .build();

    String token = tokenGenerator.getAuthToken(
        GetIamAuthTokenRequest.builder()
            .hostname(rdsHost)
            .port(5432)
            .userName("app_user")
            .build()
    );

    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:postgresql://" + rdsHost + ":5432/" + dbName);
    ds.setUsername("app_user");
    ds.setPassword(token); // IAM token as password
    ds.addDataSourceProperty("ssl", "true");
    return ds;
}
```

</details>

---

### 🟡 Q7. What is AWS API Gateway and how does it work with Lambda?

<details><summary>Click to reveal answer</summary>

**API Gateway** is a fully managed service for creating, publishing, and securing REST, HTTP, and WebSocket APIs.

```
Client → API Gateway → Lambda → DynamoDB
                     → ECS/EC2
                     → HTTP backend
```

**Features:**
- Request/response transformation
- Authentication (IAM, Cognito, Lambda authorizers)
- Rate limiting and throttling
- CORS configuration
- Stage management (dev, staging, prod)
- Caching

```java
// Lambda handler integrated with API Gateway
public class GetUserHandler
        implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    private final ObjectMapper mapper = new ObjectMapper();
    private final UserService userService = new UserService();

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {

        // Extract path parameters
        String userId = event.getPathParameters().get("userId");

        // Extract query parameters
        String includeProfile = event.getQueryStringParameters()
            .getOrDefault("includeProfile", "false");

        // Extract headers
        String authorization = event.getHeaders().get("Authorization");

        try {
            User user = userService.findById(Long.parseLong(userId));

            return new APIGatewayProxyResponseEvent()
                .withStatusCode(200)
                .withHeaders(Map.of("Content-Type", "application/json"))
                .withBody(mapper.writeValueAsString(user));

        } catch (UserNotFoundException e) {
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(404)
                .withBody("{\"error\":\"User not found\"}");
        } catch (Exception e) {
            context.getLogger().log("Error: " + e.getMessage());
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(500)
                .withBody("{\"error\":\"Internal server error\"}");
        }
    }
}
```

</details>

---

### 🟢 Q8. What is CloudWatch and how do you use it for a Java application?

<details><summary>Click to reveal answer</summary>

**CloudWatch** is AWS's monitoring and observability service — logs, metrics, alarms, dashboards.

```java
// Publishing custom metrics to CloudWatch
@Service
public class MetricsService {

    private final CloudWatchClient cloudWatch = CloudWatchClient.builder()
        .region(Region.US_EAST_1)
        .build();

    public void recordOrderProcessingTime(long durationMs) {
        MetricDatum datum = MetricDatum.builder()
            .metricName("OrderProcessingTime")
            .value((double) durationMs)
            .unit(StandardUnit.MILLISECONDS)
            .dimensions(Dimension.builder()
                .name("Environment")
                .value("production")
                .build())
            .timestamp(Instant.now())
            .build();

        cloudWatch.putMetricData(PutMetricDataRequest.builder()
            .namespace("MyApp/Orders")
            .metricData(datum)
            .build());
    }
}

// Spring Boot Actuator + CloudWatch (automatic metrics)
// application.yaml
management:
  metrics:
    export:
      cloudwatch:
        namespace: MyApp
        step: 1m
    tags:
      application: ${spring.application.name}
      environment: ${env:production}

// Structured logging with MDC (Mapped Diagnostic Context)
// Enables filtering logs in CloudWatch Insights
@RestController
public class OrderController {

    @PostMapping("/orders")
    public ResponseEntity<Order> placeOrder(@RequestBody OrderRequest req) {
        MDC.put("userId", req.getUserId().toString());
        MDC.put("requestId", UUID.randomUUID().toString());
        try {
            log.info("Placing order"); // Includes userId and requestId in log
            Order order = orderService.place(req);
            return ResponseEntity.ok(order);
        } finally {
            MDC.clear();
        }
    }
}

// CloudWatch Logs Insights query:
// fields @timestamp, @message, userId
// | filter @message like /ERROR/
// | sort @timestamp desc
// | limit 100
```

</details>

---

## 12-Factor App Methodology

### 🟡 Q9. Explain all 12 factors of the 12-Factor App methodology.

<details><summary>Click to reveal answer</summary>

The 12-Factor App is a methodology for building scalable, maintainable, cloud-native applications.

**Factor I: Codebase** — One codebase tracked in version control, many deploys
```bash
# One Git repo = one app
# Different environments (dev, staging, prod) = same code, different configs
git clone https://github.com/company/order-service.git
```

**Factor II: Dependencies** — Explicitly declare and isolate dependencies
```xml
<!-- Maven pom.xml declares all dependencies — no implicit system dependencies -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.0</version>
</dependency>
```

**Factor III: Config** — Store config in the environment, not code
```yaml
# ❌ WRONG: Config in code
database.url=jdbc:postgresql://prod-db:5432/mydb

# ✅ RIGHT: Config from environment variables
spring.datasource.url=${DATABASE_URL}
spring.datasource.password=${DB_PASSWORD}

# Or via AWS Parameter Store / Secrets Manager
```

**Factor IV: Backing Services** — Treat backing services as attached resources
```yaml
# DB, queue, cache are resources attached via URL — swap without code changes
DATABASE_URL=jdbc:postgresql://localhost:5432/dev
DATABASE_URL=jdbc:postgresql://rds-prod.amazonaws.com:5432/prod
REDIS_URL=redis://localhost:6379
REDIS_URL=redis://elasticache-prod.amazonaws.com:6379
```

**Factor V: Build, Release, Run** — Strictly separate build and run stages
```
Code → [BUILD: compile, assets] → [RELEASE: build + config] → [RUN: execute]
         mvn package              docker tag                    java -jar
         docker build             helm upgrade
```

**Factor VI: Processes** — Execute app as stateless processes
```java
// ❌ WRONG: Storing state in instance (breaks horizontal scaling)
@RestController
public class CartController {
    private Map<String, Cart> carts = new HashMap<>(); // In-memory state!
}

// ✅ RIGHT: State in backing service (Redis, DB)
@RestController
public class CartController {
    @Autowired private CartRepository cartRepository; // External state store
}
```

**Factor VII: Port Binding** — Export services via port binding
```java
// Spring Boot embeds Tomcat and binds to a port — self-contained
// application.yaml
server.port=${PORT:8080}  // Reads PORT env variable, defaults to 8080
```

**Factor VIII: Concurrency** — Scale out via the process model
```
Horizontal scaling: Run more instances (not bigger instances)
web process: 3 instances (handle HTTP)
worker process: 5 instances (consume SQS messages)
Each process stateless → can scale independently
```

**Factor IX: Disposability** — Maximize robustness with fast startup and graceful shutdown
```java
// Fast startup: use Spring's lazy initialization
spring.main.lazy-initialization=true

// Graceful shutdown: complete in-flight requests
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s

// Shutdown hook: release resources
@PreDestroy
public void cleanup() {
    sqsConsumer.stop();
    connectionPool.close();
}
```

**Factor X: Dev/Prod Parity** — Keep dev, staging, and prod as similar as possible
```yaml
# Use same DB engine in dev and prod (not H2 in dev, PostgreSQL in prod)
# TestContainers for local/CI testing with real PostgreSQL
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
```

**Factor XI: Logs** — Treat logs as event streams
```java
// ✅ Write logs to stdout/stderr — let the platform capture them
// Don't write to log files yourself
// Platform (CloudWatch, ELK, Datadog) collects stdout

// Structured logging (JSON) for machine-parseable logs
@Bean
public LogstashEncoder logEncoder() {
    return new LogstashEncoder(); // Outputs JSON to stdout
}
// Output: {"@timestamp":"2024-01-15T10:30:00Z","level":"INFO","message":"Order placed","orderId":"123"}
```

**Factor XII: Admin Processes** — Run admin/management tasks as one-off processes
```java
// Database migrations as one-off process (Flyway/Liquibase)
// runs once on startup, separate from main app
spring.flyway.enabled=true

// Admin CLI commands as separate Spring Boot applications
@SpringBootApplication
public class DataMigrationJob {
    public static void main(String[] args) {
        SpringApplication.run(DataMigrationJob.class, args);
    }

    @Bean
    public CommandLineRunner migrate(UserService service) {
        return args -> service.backfillMissingProfiles();
    }
}
```

</details>

---

## Microservices vs Monolith

### 🟡 Q10. What are the trade-offs between microservices and a monolithic architecture?

<details><summary>Click to reveal answer</summary>

**Monolith:** Single deployable unit containing all functionality

```
[Monolith]
├── UserModule
├── OrderModule
├── PaymentModule
├── InventoryModule
└── NotificationModule
Single process, single DB, single deployment
```

**Microservices:** Many small, independently deployable services

```
[User Service] → DB1 (PostgreSQL)
[Order Service] → DB2 (PostgreSQL)
[Payment Service] → DB3 (PostgreSQL)
[Inventory Service] → DB4 (MongoDB)
[Notification Service] → (stateless)
Each independently deployed and scaled
```

| Concern | Monolith | Microservices |
|---|---|---|
| Development speed (early) | ✅ Fast | ❌ Slower setup |
| Deployment | ✅ Simple | ❌ Complex (K8s, service mesh) |
| Scaling | ❌ Scale everything | ✅ Scale bottleneck only |
| Fault isolation | ❌ One crash = all down | ✅ Failures are contained |
| Technology diversity | ❌ One stack | ✅ Best tool per service |
| Data consistency | ✅ ACID transactions | ❌ Eventual consistency |
| Testing | ✅ Easier (one app) | ❌ Integration tests complex |
| Latency | ✅ In-process calls | ❌ Network calls add latency |
| Team autonomy | ❌ Coordinated releases | ✅ Independent teams/releases |
| Observability | ✅ Single log stream | ❌ Distributed tracing needed |

```
When to choose Monolith:
- Small team (< 5 engineers)
- MVP / startup (move fast)
- Domain boundaries unclear
- Low traffic

When to choose Microservices:
- Large team (> 10 engineers), multiple teams
- Clear domain boundaries
- Independent scaling requirements
- High availability requirements
- Team autonomy important
```

> 💡 **Interviewer Tip:** The right answer is often "Start with a modular monolith, extract services when you have clear boundaries and scaling needs." Premature microservices is a common antipattern.

</details>

---

### 🔴 Q11. How do microservices communicate with each other?

<details><summary>Click to reveal answer</summary>

**Synchronous Communication:**
- **REST (HTTP)** — Simple, universal, but creates tight temporal coupling
- **gRPC** — Binary protocol (Protocol Buffers), strongly typed, faster than REST
- **GraphQL** — Client specifies needed fields, reduces over-fetching

**Asynchronous Communication:**
- **Message queues (SQS, RabbitMQ, ActiveMQ)** — Decoupled, guaranteed delivery
- **Event streaming (Kafka, Kinesis)** — High throughput, event replay
- **Pub/Sub (SNS, Redis Pub/Sub)** — Fan-out to multiple consumers

```java
// Synchronous: REST with RestTemplate / WebClient
@Service
public class OrderService {

    private final WebClient inventoryClient = WebClient.builder()
        .baseUrl("http://inventory-service:8080")
        .build();

    public boolean checkInventory(Long productId, int quantity) {
        return inventoryClient.get()
            .uri("/products/{id}/availability?qty={qty}", productId, quantity)
            .retrieve()
            .bodyToMono(AvailabilityResponse.class)
            .map(AvailabilityResponse::isAvailable)
            .block(Duration.ofSeconds(5)); // Timeout!
    }
}

// Asynchronous: Event-driven with SQS
@Service
public class OrderService {

    @Autowired private SqsTemplate sqsTemplate;

    public Order placeOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        // Fire and forget — inventory service processes asynchronously
        sqsTemplate.send("inventory-updates-queue",
            new InventoryUpdateEvent(order.getItems()));

        return order; // Return immediately, don't wait for inventory update
    }
}

// Service discovery (how services find each other)
// Kubernetes: services by DNS name (http://inventory-service)
// AWS: ECS Service Discovery or ALB/NLB
// Consul/Eureka: service registry
@LoadBalanced  // Spring Cloud — load-balanced RestTemplate
@Bean
public RestTemplate restTemplate() { return new RestTemplate(); }
```

**Circuit Breaker pattern (Resilience4j):**
```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallback")
@TimeLimiter(name = "inventoryService")
public CompletableFuture<Boolean> checkInventory(Long productId, int qty) {
    return CompletableFuture.supplyAsync(() ->
        inventoryClient.check(productId, qty));
}

public CompletableFuture<Boolean> fallback(Long productId, int qty, Exception e) {
    log.warn("Inventory service unavailable, assuming available", e);
    return CompletableFuture.completedFuture(true); // Optimistic fallback
}
```

</details>

---

## Containerization

### 🟢 Q12. What is containerization and how does Docker work?

<details><summary>Click to reveal answer</summary>

**Containerization** packages an application with its dependencies into a standardized unit (container) that runs consistently across environments.

**Container vs VM:**
```
VM: Hardware → Hypervisor → [OS + App]  [OS + App]
    Heavy (GBs), minutes to start

Container: Hardware → OS → Container Runtime → [App] [App] [App]
           Lightweight (MBs), seconds to start
           Shares host OS kernel
```

**Dockerfile for Spring Boot:**
```dockerfile
# Multi-stage build (smaller final image)
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q  # Cache dependencies
COPY src ./src
RUN mvn package -DskipTests -q

# Runtime image (no Maven, no sources)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
EXPOSE 8080

# Create non-root user (security best practice)
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copy JAR from builder stage
COPY --from=builder /app/target/*.jar app.jar

# JVM tuning for containers
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

```yaml
# docker-compose.yml for local development
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=jdbc:postgresql://db:5432/myapp
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres-data:
```

</details>

---

### 🟡 Q13. What is Kubernetes and what problems does it solve?

<details><summary>Click to reveal answer</summary>

**Kubernetes (K8s)** is a container orchestration platform that automates deployment, scaling, and management of containerized applications.

**Problems Kubernetes solves:**
- Container scheduling across multiple nodes
- Self-healing (restart failed containers)
- Horizontal scaling (auto-scale based on CPU/memory)
- Service discovery and load balancing
- Rolling updates with zero downtime
- Configuration and secrets management

```yaml
# Kubernetes Deployment for Spring Boot app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

</details>

---

## Serverless Computing

### 🟡 Q14. What is serverless computing and what are its trade-offs?

<details><summary>Click to reveal answer</summary>

**Serverless** means you write code (functions), and the cloud provider handles all infrastructure: servers, scaling, patching, availability.

**Key characteristics:**
- No server management
- Auto-scales to 0 (and to thousands)
- Pay per invocation (not idle time)
- Stateless execution

**AWS Lambda specifics:**
- Timeout: Max 15 minutes
- Memory: 128 MB to 10 GB
- Ephemeral storage: /tmp up to 10 GB
- Cold start: ~100ms-2s (JVM languages suffer more)

```java
// Lambda with SnapStart (Java cold start optimization, Java 11+)
@SpringBootApplication
public class OrderLambdaApplication implements ApplicationRunner {

    @Autowired private OrderService orderService;

    @Override
    public void run(ApplicationArguments args) { }
}

// Handler using Spring Cloud Function
@Configuration
public class FunctionConfig {

    @Bean
    public Function<OrderRequest, OrderResponse> processOrder(OrderService service) {
        return request -> {
            Order order = service.place(request);
            return new OrderResponse(order.getId(), order.getStatus());
        };
    }
}
```

**Trade-offs:**

| Concern | Serverless | Always-on Server |
|---|---|---|
| Cost (low traffic) | ✅ Pay per request | ❌ Paying for idle time |
| Cost (high traffic) | ❌ Can exceed EC2 cost | ✅ Flat rate |
| Cold starts | ❌ Latency spike | ✅ Always warm |
| Execution time | ❌ Max 15 min | ✅ Unlimited |
| State | ❌ Stateless | ✅ Can hold state |
| Debugging | ❌ Harder | ✅ Easier |
| Vendor lock-in | ❌ Lambda-specific | ✅ Portable |

✅ **Best for serverless:** Event handlers, API backends with variable traffic, ETL/batch jobs, scheduled tasks, webhooks.

</details>

---

## Infrastructure Concepts

### 🟢 Q15. What is the difference between AWS Regions and Availability Zones?

<details><summary>Click to reveal answer</summary>

**Region** — A geographic area containing multiple isolated data centers
- Examples: `us-east-1` (N. Virginia), `eu-west-1` (Ireland), `ap-southeast-1` (Singapore)
- Each region is isolated from other regions (data sovereignty)
- Choose region based on: latency to users, data residency laws, service availability, cost

**Availability Zone (AZ)** — One or more discrete data centers within a region
- Each region has 2-6 AZs (e.g., `us-east-1a`, `us-east-1b`, `us-east-1c`)
- AZs are physically separate (different power, cooling, flood zone)
- Connected by low-latency, high-bandwidth networking
- Failure of one AZ should not affect others

```
us-east-1 (Region: N. Virginia)
├── us-east-1a (AZ: Data Center 1, 2)
├── us-east-1b (AZ: Data Center 3, 4)
├── us-east-1c (AZ: Data Center 5)
└── us-east-1d (AZ: Data Center 6)
```

**Multi-AZ architecture for high availability:**
```yaml
# RDS Multi-AZ: automatic failover to standby in another AZ
aws_db_instance:
  multi_az: true  # Synchronous replication to standby AZ
  # If primary AZ fails, RDS promotes standby in ~60-120 seconds

# ECS: spread tasks across AZs
placement_constraints:
  - type: spread
    field: attribute:ecs.availability-zone

# ALB: spans multiple AZs automatically
aws_lb:
  subnets: [subnet-az1, subnet-az2, subnet-az3]

# Spring Boot: no code changes needed — architect at infra level
```

**Edge Locations (CDN):**
- CloudFront CDN points of presence
- 400+ globally (more than regions)
- Caches static content close to users
- Not full AWS infrastructure — only for CloudFront, Route 53, WAF

</details>

---

### 🟡 Q16. What is Auto Scaling and how do you configure it for a Java application?

<details><summary>Click to reveal answer</summary>

**Auto Scaling** automatically adjusts the number of running instances based on demand.

**Types:**
- **Target tracking** — Maintain a target metric (e.g., 70% CPU)
- **Step scaling** — Add N instances when metric crosses threshold
- **Scheduled scaling** — Scale up before known traffic spikes
- **Predictive scaling** — ML-based forecast scaling

```yaml
# AWS Auto Scaling Group configuration
aws_autoscaling_group:
  min_size: 2
  max_size: 20
  desired_capacity: 4
  health_check_type: ELB  # Use load balancer health checks
  health_check_grace_period: 300  # Wait 5 min for Spring Boot to start

# Target tracking policy: maintain 70% CPU
aws_autoscaling_policy:
  policy_type: TargetTrackingScaling
  target_tracking_configuration:
    predefined_metric_specification:
      predefined_metric_type: ASGAverageCPUUtilization
    target_value: 70.0
```

```java
// Spring Boot Actuator health endpoint (required for ALB health checks)
// application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true  # /actuator/health/liveness and /actuator/health/readiness

// Custom health indicator
@Component
public class DatabaseHealthIndicator extends AbstractHealthIndicator {

    @Autowired private DataSource dataSource;

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement("SELECT 1")) {
            stmt.execute();
            builder.up().withDetail("database", "Available");
        } catch (SQLException e) {
            builder.down().withDetail("error", e.getMessage());
        }
    }
}
```

</details>

---

### 🟡 Q17. What is a CDN and how does it improve application performance?

<details><summary>Click to reveal answer</summary>

**CDN (Content Delivery Network)** is a geographically distributed network of servers that caches content close to users.

```
Without CDN:
User (Tokyo) → Request → Origin server (US East) → Response (200ms RTT)

With CDN:
User (Tokyo) → Request → Edge Location (Tokyo) → Cache HIT → Response (5ms)
                                                 → Cache MISS → Origin → Cache → Response (210ms, then 5ms after)
```

**AWS CloudFront + Spring Boot:**
```java
// Configure S3 + CloudFront for static assets
// Spring Boot serves API, S3+CloudFront serves frontend/images

// Set cache-control headers for different content types
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS)
                .cachePublic());

        registry.addResourceHandler("/api/**")
            .setCacheControl(CacheControl.noCache()
                .noStore()
                .mustRevalidate());
    }
}

// API responses — cache-control headers
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(@PathVariable Long id) {
    Product product = productService.findById(id);
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES).cachePublic())
        .eTag(String.valueOf(product.getVersion()))
        .body(product);
}
```

**What to cache at CDN:**
- ✅ Static files (JS, CSS, images, fonts)
- ✅ Public API responses (product catalog, public content)
- ❌ Personalized content (user dashboard, cart)
- ❌ Session-based content

</details>

---

### 🟡 Q18. What is a Load Balancer and what types does AWS provide?

<details><summary>Click to reveal answer</summary>

A **Load Balancer** distributes incoming traffic across multiple targets (EC2, ECS tasks, Lambda).

**AWS Load Balancer types:**

| Type | Layer | Protocol | Use Case |
|---|---|---|---|
| ALB (Application) | L7 (HTTP/HTTPS) | HTTP, HTTPS, WebSocket | Web apps, microservices, path-based routing |
| NLB (Network) | L4 (TCP/UDP) | TCP, UDP, TLS | High-performance, static IP, gaming |
| CLB (Classic) | L4/L7 | Legacy | Avoid — deprecated |
| GWLB (Gateway) | L3 | IP packets | Security appliances |

```yaml
# ALB configuration for Spring Boot microservices
# Path-based routing
Listener: HTTPS:443
Rules:
  - /api/users/*   → Target Group: user-service  (3 instances)
  - /api/orders/*  → Target Group: order-service (5 instances)
  - /api/products/*→ Target Group: product-service (2 instances)
  - default        → Target Group: frontend-service

# Health check
health_check:
  path: /actuator/health
  interval: 30
  timeout: 10
  healthy_threshold: 2
  unhealthy_threshold: 3
```

```java
// Spring Boot: Handle X-Forwarded-For header (real client IP behind ALB)
// application.yaml
server:
  forward-headers-strategy: framework  # Spring Boot 2.2+

// Or explicitly in code
@GetMapping("/users")
public String getClientIp(HttpServletRequest request) {
    String forwarded = request.getHeader("X-Forwarded-For");
    String clientIp = forwarded != null ? forwarded.split(",")[0].trim()
                                        : request.getRemoteAddr();
    return clientIp;
}
```

</details>

---

### 🔴 Q19. What is Infrastructure as Code (IaC) and why is it important?

<details><summary>Click to reveal answer</summary>

**Infrastructure as Code** manages cloud infrastructure using code and version control, rather than manual console clicks.

**Benefits:**
- **Reproducibility** — Recreate entire environments reliably
- **Version control** — Track infrastructure changes like code
- **Code review** — Peer review infrastructure changes
- **Automation** — CI/CD for infrastructure
- **Documentation** — Code is self-documenting

**Tools:** Terraform (multi-cloud), AWS CloudFormation (AWS-native), AWS CDK (Java/TypeScript), Pulumi

```java
// AWS CDK (Java) — define infrastructure in Java!
public class OrderServiceStack extends Stack {

    public OrderServiceStack(App app, String id) {
        super(app, id);

        // VPC
        Vpc vpc = Vpc.Builder.create(this, "OrderVpc")
            .maxAzs(3)
            .build();

        // RDS
        DatabaseInstance db = DatabaseInstance.Builder.create(this, "OrderDB")
            .engine(DatabaseInstanceEngine.postgres(
                PostgresInstanceEngineProps.builder()
                    .version(PostgresEngineVersion.VER_15_3)
                    .build()))
            .vpc(vpc)
            .instanceType(InstanceType.of(InstanceClass.T3, InstanceSize.SMALL))
            .multiAz(true)
            .deletionProtection(true)
            .build();

        // ECS Cluster + Fargate Service
        Cluster cluster = Cluster.Builder.create(this, "OrderCluster")
            .vpc(vpc)
            .build();

        ApplicationLoadBalancedFargateService service =
            ApplicationLoadBalancedFargateService.Builder.create(this, "OrderService")
                .cluster(cluster)
                .cpu(512)
                .memoryLimitMiB(1024)
                .desiredCount(3)
                .taskImageOptions(ApplicationLoadBalancedTaskImageOptions.builder()
                    .image(ContainerImage.fromRegistry("myapp/order-service:latest"))
                    .containerPort(8080)
                    .build())
                .build();

        // Auto-scaling
        ScalableTaskCount scaling = service.getService()
            .autoScaleTaskCount(EnableScalingProps.builder()
                .minCapacity(2)
                .maxCapacity(10)
                .build());

        scaling.scaleOnCpuUtilization("CpuScaling",
            CpuUtilizationScalingProps.builder()
                .targetUtilizationPercent(70)
                .build());
    }
}
```

</details>

---

### 🟢 Q20. What is a VPC and what networking concepts should a Java developer know?

<details><summary>Click to reveal answer</summary>

**VPC (Virtual Private Cloud)** is a logically isolated section of AWS where you launch resources.

**Key networking concepts:**

```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24) — Internet-accessible
│   ├── Load Balancer
│   └── NAT Gateway (for private subnet internet access)
└── Private Subnet (10.0.2.0/24) — Not directly internet-accessible
    ├── EC2 instances (Spring Boot apps)
    ├── RDS (database)
    └── ElastiCache (Redis)
```

**Security Groups vs NACLs:**

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance/ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Default | Deny all inbound | Allow all |

```java
// Spring Boot security group design:
// ALB Security Group: Allow 443 from 0.0.0.0/0
// App Security Group: Allow 8080 from ALB Security Group only
// DB Security Group: Allow 5432 from App Security Group only

// Connecting to RDS in private subnet:
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(System.getenv("DATABASE_URL"));
        // RDS endpoint is private: order-db.xxxxx.us-east-1.rds.amazonaws.com
        // Only reachable from within VPC private subnet
        ds.setConnectionTimeout(30_000);
        ds.setMaximumPoolSize(10);
        return ds;
    }
}
```

</details>

---

*End of Cloud Computing Fundamentals*
