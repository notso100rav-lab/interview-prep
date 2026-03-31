# Deploying Spring Boot on AWS

> Java Backend Engineer Interview Prep — Chapter 4.1

---

## Table of Contents
1. [EC2 Manual Deployment](#ec2-manual-deployment)
2. [ECS Fargate](#ecs-fargate)
3. [Elastic Beanstalk](#elastic-beanstalk)
4. [RDS PostgreSQL](#rds-postgresql)
5. [S3 File Upload](#s3-file-upload)
6. [SQS Consumer](#sqs-consumer)
7. [API Gateway → Lambda](#api-gateway--lambda)
8. [Q&A](#qa)

---

## EC2 Manual Deployment

### systemd Service Unit File

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Spring Boot Application
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/java \
  -Xms512m -Xmx1024m \
  -Dspring.profiles.active=prod \
  -Dspring.config.location=/opt/myapp/config/application-prod.yml \
  -jar /opt/myapp/myapp.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=myapp
EnvironmentFile=/opt/myapp/.env

[Install]
WantedBy=multi-user.target
```

```bash
# Deploy commands
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
sudo journalctl -u myapp -f          # tail logs
```

### EC2 User Data Bootstrap Script

```bash
#!/bin/bash
yum update -y
yum install -y java-17-amazon-corretto

# Create app directory
mkdir -p /opt/myapp/config

# Download artifact from S3
aws s3 cp s3://my-artifacts-bucket/myapp-1.0.0.jar /opt/myapp/myapp.jar

# Set up systemd service
cp /opt/myapp/myapp.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable myapp
systemctl start myapp
```

---

## ECS Fargate

### ECR Push Commands

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS \
    --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp --region us-east-1

# Build, tag, push
docker build -t myapp:latest .
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### ECS Task Definition (JSON)

```json
{
  "family": "myapp-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### ECS Service & Cluster CLI

```bash
# Create cluster
aws ecs create-cluster --cluster-name myapp-cluster

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Create service
aws ecs create-service \
  --cluster myapp-cluster \
  --service-name myapp-service \
  --task-definition myapp-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-123],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=myapp,containerPort=8080"

# Update service (rolling deploy)
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-service \
  --task-definition myapp-task:2 \
  --force-new-deployment
```

---

## Elastic Beanstalk

### Procfile

```
web: java -Xmx1024m -Dspring.profiles.active=prod -jar myapp.jar
```

### eb CLI Commands

```bash
# Initialize
eb init myapp --platform "Java 17 running on 64bit Amazon Linux 2" --region us-east-1

# Create environment
eb create myapp-prod \
  --instance-type t3.medium \
  --envvars SPRING_PROFILES_ACTIVE=prod,DB_URL=jdbc:postgresql://... \
  --single

# Deploy
mvn package -DskipTests
eb deploy myapp-prod

# Logs
eb logs myapp-prod

# SSH into instance
eb ssh myapp-prod

# Open app URL
eb open myapp-prod

# Set environment variables
eb setenv DB_PASSWORD=secret SPRING_PROFILES_ACTIVE=prod

# Scale
eb scale 3 --region us-east-1
```

### .ebextensions Configuration

```yaml
# .ebextensions/jvm.config
option_settings:
  aws:elasticbeanstalk:application:environment:
    SERVER_PORT: 5000
  aws:elasticbeanstalk:container:java:jvmOptions: "-Xmx1024m -XX:+UseG1GC"
```

---

## RDS PostgreSQL

### JDBC URL Format

```
jdbc:postgresql://<host>:<port>/<database>?ssl=true&sslmode=require
```

### application.properties

```properties
# Basic RDS configuration
spring.datasource.url=jdbc:postgresql://mydb.abc123.us-east-1.rds.amazonaws.com:5432/myapp
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# Connection pool (HikariCP)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000

# JPA
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
```

### AWS Secrets Manager Integration

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-secrets-manager-config</artifactId>
  <version>3.1.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  config:
    import: aws-secretsmanager:/myapp/prod/db-credentials
  datasource:
    url: jdbc:postgresql://${db.host}:5432/${db.name}
    username: ${db.username}
    password: ${db.password}
```

```java
// Programmatic Secrets Manager access
@Configuration
public class SecretsManagerConfig {

    @Bean
    public DataSource dataSource() {
        SecretsManagerClient client = SecretsManagerClient.builder()
            .region(Region.US_EAST_1)
            .build();

        GetSecretValueRequest request = GetSecretValueRequest.builder()
            .secretId("/myapp/prod/db-credentials")
            .build();

        String secretJson = client.getSecretValue(request).secretString();
        ObjectMapper mapper = new ObjectMapper();
        JsonNode secret = mapper.readTree(secretJson);

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(secret.get("jdbcUrl").asText());
        config.setUsername(secret.get("username").asText());
        config.setPassword(secret.get("password").asText());
        return new HikariDataSource(config);
    }
}
```

---

## S3 File Upload

### Maven Dependency

```xml
<dependency>
  <groupId>software.amazon.awssdk</groupId>
  <artifactId>s3</artifactId>
  <version>2.25.0</version>
</dependency>
```

### S3 Service with SDK v2

```java
@Service
@RequiredArgsConstructor
public class S3Service {

    private final S3Client s3Client;

    @Value("${aws.s3.bucket}")
    private String bucket;

    public String uploadFile(MultipartFile file, String key) throws IOException {
        PutObjectRequest request = PutObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .contentType(file.getContentType())
            .contentLength(file.getSize())
            .serverSideEncryption(ServerSideEncryption.AES256)
            .build();

        s3Client.putObject(request, RequestBody.fromInputStream(
            file.getInputStream(), file.getSize()));

        return String.format("https://%s.s3.amazonaws.com/%s", bucket, key);
    }

    public ResponseInputStream<GetObjectResponse> downloadFile(String key) {
        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .build();
        return s3Client.getObject(request);
    }

    public void deleteFile(String key) {
        s3Client.deleteObject(DeleteObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .build());
    }

    public String generatePresignedUrl(String key, Duration expiry) {
        try (S3Presigner presigner = S3Presigner.create()) {
            GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(expiry)
                .getObjectRequest(r -> r.bucket(bucket).key(key))
                .build();
            return presigner.presignGetObject(presignRequest).url().toString();
        }
    }
}
```

```java
@Configuration
public class S3Config {

    @Bean
    public S3Client s3Client(@Value("${aws.region}") String region) {
        return S3Client.builder()
            .region(Region.of(region))
            .credentialsProvider(DefaultCredentialsProvider.create())
            .build();
    }
}
```

---

## SQS Consumer

### Dependency

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-sqs</artifactId>
  <version>3.1.0</version>
</dependency>
```

### SQS Listener

```java
@Component
@Slf4j
public class OrderEventConsumer {

    @SqsListener(value = "${aws.sqs.order-queue}", acknowledgementMode = SqsAcknowledgementMode.MANUAL)
    public void handleOrderEvent(
            OrderEvent event,
            Acknowledgement acknowledgement,
            @Header("ApproximateReceiveCount") String receiveCount) {

        log.info("Processing order event: {}, attempt: {}", event.getOrderId(), receiveCount);

        try {
            processOrder(event);
            acknowledgement.acknowledge();
        } catch (RetryableException e) {
            log.warn("Retryable error for order {}, will retry", event.getOrderId());
            // Don't acknowledge — message returns to queue
        } catch (Exception e) {
            log.error("Fatal error processing order {}", event.getOrderId(), e);
            acknowledgement.acknowledge(); // Send to DLQ via redrive policy
        }
    }

    private void processOrder(OrderEvent event) {
        // business logic
    }
}
```

```java
// Sending messages to SQS
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final SqsTemplate sqsTemplate;

    @Value("${aws.sqs.order-queue}")
    private String queueUrl;

    public void publish(OrderEvent event) {
        sqsTemplate.send(to -> to
            .queue(queueUrl)
            .payload(event)
            .header("eventType", "ORDER_CREATED")
            .delaySeconds(0));
    }
}
```

```yaml
# application.yml
spring:
  cloud:
    aws:
      sqs:
        region: us-east-1
      credentials:
        access-key: ${AWS_ACCESS_KEY_ID}
        secret-key: ${AWS_SECRET_ACCESS_KEY}

aws:
  sqs:
    order-queue: https://sqs.us-east-1.amazonaws.com/123456789/order-queue
```

---

## API Gateway → Lambda

### Spring Cloud Function Setup

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-function-adapter-aws</artifactId>
  <version>4.1.0</version>
</dependency>
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-lambda-java-events</artifactId>
  <version>3.11.3</version>
</dependency>
```

```java
@SpringBootApplication
public class LambdaApplication {
    public static void main(String[] args) {
        SpringApplication.run(LambdaApplication.class, args);
    }

    @Bean
    public Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> handleRequest() {
        return event -> {
            String body = event.getBody();
            // process request
            return APIGatewayProxyResponseEvent.builder()
                .statusCode(200)
                .header("Content-Type", "application/json")
                .body("{\"message\": \"OK\"}")
                .build();
        };
    }
}
```

```java
// Handler class for Lambda entry point
public class FunctionHandler extends SpringBootRequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
}
```

```yaml
# serverless.yml (Serverless Framework)
service: myapp-lambda
provider:
  name: aws
  runtime: java17
  region: us-east-1
  memorySize: 512
  timeout: 30

functions:
  api:
    handler: com.example.FunctionHandler
    events:
      - http:
          path: /api/{proxy+}
          method: any
    environment:
      SPRING_PROFILES_ACTIVE: prod
```

---

## Q&A

### Q1 🟢 What are the main ways to deploy a Spring Boot app on AWS?

<details><summary>Click to reveal answer</summary>

The main deployment options are:

| Option | Best For | Management Overhead |
|--------|----------|-------------------|
| **EC2 + systemd** | Full control, custom AMIs | High |
| **ECS Fargate** | Containerized apps, serverless containers | Medium |
| **Elastic Beanstalk** | Managed PaaS, quick setup | Low |
| **Lambda + Spring Cloud Function** | Event-driven, low traffic | Very Low |
| **EKS** | Kubernetes orchestration | High |

</details>

---

### Q2 🟢 How do you configure Spring Boot to read credentials from AWS Secrets Manager?

<details><summary>Click to reveal answer</summary>

Add the AWS Secrets Manager config dependency and configure `spring.config.import`:

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-secrets-manager-config</artifactId>
  <version>3.1.0</version>
</dependency>
```

```yaml
spring:
  config:
    import: aws-secretsmanager:/myapp/prod/secrets
```

Secrets stored as JSON `{"db.password": "secret"}` become available as `${db.password}`. This avoids storing credentials in environment variables or config files.

</details>

---

### Q3 🟡 Explain the ECS task execution role vs. task role. What is each used for?

<details><summary>Click to reveal answer</summary>

- **Execution Role** (`ecsTaskExecutionRole`): Used by the **ECS agent** to pull the Docker image from ECR, write logs to CloudWatch, and fetch secrets from Secrets Manager / SSM Parameter Store before the container starts. The application itself never uses this role.

- **Task Role**: Used by the **application code running inside the container** to call AWS services (S3, SQS, DynamoDB, etc.) at runtime via the instance metadata credential endpoint `http://169.254.170.2`.

```json
// Task role policy example
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

</details>

---

### Q4 🟡 How does a Spring Boot app authenticate with AWS when running on EC2 or ECS?

<details><summary>Click to reveal answer</summary>

The AWS SDK uses the **Default Credential Provider Chain**, which searches in order:

1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. Java system properties
3. AWS credential profiles file (`~/.aws/credentials`)
4. **EC2 Instance Metadata Service (IMDS)** — for EC2 instances
5. **ECS Task Role** — for ECS containers via `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`

On EC2/ECS, the recommended approach is to attach an **IAM Role** to the instance/task. The SDK automatically fetches temporary credentials without any code changes:

```java
S3Client s3Client = S3Client.builder()
    .credentialsProvider(DefaultCredentialsProvider.create()) // uses chain
    .region(Region.US_EAST_1)
    .build();
```

</details>

---

### Q5 🟡 What is the difference between an SQS standard queue and a FIFO queue?

<details><summary>Click to reveal answer</summary>

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|------------|
| **Ordering** | Best-effort | Strict FIFO per message group |
| **Delivery** | At-least-once (duplicates possible) | Exactly-once processing |
| **Throughput** | Nearly unlimited | 300 msg/s (3,000 with batching) |
| **Deduplication** | No built-in | Yes (5-min dedup window) |
| **Use case** | High-volume, order not critical | Financial transactions, ordering |

For Spring Boot, FIFO queue URL ends in `.fifo`:

```yaml
aws:
  sqs:
    order-queue: https://sqs.us-east-1.amazonaws.com/123456789/orders.fifo
```

</details>

---

### Q6 🔴 How do you implement idempotent SQS message processing in Spring Boot?

<details><summary>Click to reveal answer</summary>

SQS delivers messages **at least once**, so you must handle duplicates:

```java
@Component
@RequiredArgsConstructor
public class IdempotentOrderConsumer {

    private final ProcessedMessageRepository processedRepo;

    @SqsListener("${aws.sqs.order-queue}")
    public void handleOrder(OrderEvent event,
                            @Header("MessageId") String messageId) {
        // Check if already processed
        if (processedRepo.existsByMessageId(messageId)) {
            log.info("Duplicate message {}, skipping", messageId);
            return;
        }

        // Process in transaction — save processed ID atomically
        processOrderAndMarkProcessed(event, messageId);
    }

    @Transactional
    public void processOrderAndMarkProcessed(OrderEvent event, String messageId) {
        orderService.process(event);
        processedRepo.save(new ProcessedMessage(messageId, Instant.now()));
    }
}
```

Key strategies:
1. **Database deduplication table** with unique constraint on `messageId`
2. **Redis SETNX** for high-throughput idempotency checks
3. **Natural idempotency** — design operations to be naturally idempotent (e.g., upserts)

</details>

---

### Q7 🟡 What is the Elastic Beanstalk Procfile used for?

<details><summary>Click to reveal answer</summary>

The `Procfile` tells Elastic Beanstalk which command to run to start your application. It's placed in the root of your deployment package:

```
web: java -Xmx1024m -Dspring.profiles.active=prod -jar myapp.jar
```

- The process type must be `web` for Beanstalk to route HTTP traffic to it
- `SERVER_PORT` must be `5000` (Beanstalk's default) or configured via `.ebextensions`
- Without a Procfile, Beanstalk looks for a `.jar` file automatically

</details>

---

### Q8 🟢 How do you upload a file to S3 from Spring Boot using SDK v2?

<details><summary>Click to reveal answer</summary>

```java
@Service
public class S3Service {
    private final S3Client s3Client;
    private final String bucket = "my-bucket";

    public String upload(MultipartFile file, String key) throws IOException {
        PutObjectRequest req = PutObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .contentType(file.getContentType())
            .build();

        s3Client.putObject(req,
            RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

        return "https://" + bucket + ".s3.amazonaws.com/" + key;
    }
}
```

Key SDK v2 differences from v1:
- Package is `software.amazon.awssdk` (not `com.amazonaws`)
- Immutable request builders instead of mutable request objects
- `RequestBody` instead of `InputStream` directly

</details>

---

### Q9 🔴 How would you configure Spring Boot to use RDS with connection pooling for high availability?

<details><summary>Click to reveal answer</summary>

```yaml
spring:
  datasource:
    url: jdbc:postgresql://mydb.cluster-abc.us-east-1.rds.amazonaws.com:5432/myapp
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000        # less than RDS wait_timeout (28800s default)
      keepalive-time: 300000       # send keepalive every 5 min
      connection-test-query: SELECT 1
      validation-timeout: 5000
```

For **RDS Multi-AZ failover**, use the **cluster endpoint** (not the instance endpoint). During failover (~30s), connections will fail until DNS updates. Configure retry:

```java
@Bean
public DataSourceHealthContributor dbHealthCheck(DataSource ds) {
    return new DataSourceHealthContributor(ds);
}
```

Also set `spring.datasource.hikari.keepalive-time` to prevent idle connections from being dropped by RDS's TCP timeout.

</details>

---

### Q10 🟡 What is the difference between EC2 Instance Connect and AWS Systems Manager Session Manager for accessing EC2?

<details><summary>Click to reveal answer</summary>

| Feature | EC2 Instance Connect | SSM Session Manager |
|---------|---------------------|---------------------|
| **Requires open SSH port** | Yes (port 22) | No |
| **IAM-based auth** | Yes (push temp key) | Yes |
| **Audit trail** | CloudTrail | CloudTrail + Session logs to S3/CW |
| **Works in private subnet** | No (needs internet) | Yes (via SSM endpoint) |
| **Recommended for prod** | No | Yes |

For security, prefer SSM Session Manager — no inbound security group rules needed:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

</details>

---

### Q11 🟡 How does ECS Fargate handle auto-scaling?

<details><summary>Click to reveal answer</summary>

ECS Fargate uses **Application Auto Scaling** with target tracking or step scaling policies:

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/myapp-cluster/myapp-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Create target tracking policy (CPU-based)
aws application-autoscaling put-scaling-policy \
  --policy-name cpu-tracking \
  --service-namespace ecs \
  --resource-id service/myapp-cluster/myapp-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

</details>

---

### Q12 🟢 What does the `SuccessExitStatus=143` mean in the systemd unit file for a Spring Boot app?

<details><summary>Click to reveal answer</summary>

Exit code **143** = 128 + 15 (SIGTERM). When systemd stops a service, it sends **SIGTERM** (signal 15). Java exits with code 143 when terminated by SIGTERM.

Without `SuccessExitStatus=143`, systemd would log the service as "failed" even during a clean shutdown, because any non-zero exit code is treated as failure by default.

Spring Boot handles SIGTERM for **graceful shutdown**:

```yaml
server:
  shutdown: graceful  # wait for in-flight requests before stopping
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

</details>

---

### Q13 🔴 Explain how Spring Cloud Function works with API Gateway and Lambda.

<details><summary>Click to reveal answer</summary>

Spring Cloud Function wraps Spring beans of type `Function<I, O>`, `Consumer<I>`, or `Supplier<O>` and exposes them as Lambda handlers.

**Request flow:**
```
Client → API Gateway (HTTP) → Lambda Invoke → SpringBootRequestHandler
  → DispatcherServlet (optional) → Your Function<> bean → Response
```

**Key classes:**
- `SpringBootRequestHandler<IN, OUT>` — entry point for Lambda
- `FunctionCatalog` — resolves which function to invoke
- `SPRING_CLOUD_FUNCTION_DEFINITION` env var — selects function by name

```java
// Multiple functions in one Lambda
@Bean
public Function<OrderRequest, OrderResponse> processOrder() {
    return request -> orderService.process(request);
}

@Bean
public Function<RefundRequest, RefundResponse> processRefund() {
    return request -> refundService.process(request);
}
```

Set `SPRING_CLOUD_FUNCTION_DEFINITION=processOrder` to route to a specific function.

**Cold start concern**: Spring context init takes ~2-5s. Mitigate with:
- Provisioned concurrency in Lambda
- `spring.main.lazy-initialization=true`
- GraalVM native image

</details>

---

### Q14 🟡 What is the Dead Letter Queue (DLQ) pattern in SQS and how do you set it up?

<details><summary>Click to reveal answer</summary>

A DLQ receives messages that fail processing after a maximum number of retries (`maxReceiveCount`). This prevents poison-pill messages from blocking the main queue.

```bash
# Create DLQ
aws sqs create-queue --queue-name order-queue-dlq

# Set redrive policy on main queue
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/order-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456789:order-queue-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

In Spring Boot, configure an alarm on DLQ depth:

```java
@SqsListener("${aws.sqs.order-dlq}")
public void handleDlqMessage(String rawMessage, @Header("MessageId") String messageId) {
    log.error("Message {} ended in DLQ: {}", messageId, rawMessage);
    alertingService.sendAlert("Message in DLQ: " + messageId);
}
```

</details>

---

### Q15 🟡 How do you pass environment variables to an ECS Fargate task securely?

<details><summary>Click to reveal answer</summary>

**Never** put plaintext secrets in the `environment` block. Use `secrets`:

```json
{
  "containerDefinitions": [{
    "environment": [
      { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
      { "name": "AWS_REGION", "value": "us-east-1" }
    ],
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/db-password"
      },
      {
        "name": "API_KEY",
        "valueFrom": "/myapp/prod/api-key"
      }
    ]
  }]
}
```

- `valueFrom` with Secrets Manager ARN: fetches full secret or `secret-arn:json-key::`
- `valueFrom` with SSM path: fetches Parameter Store value
- ECS agent fetches at task startup, injects as env variables
- Requires `secretsmanager:GetSecretValue` on the **execution role** (not task role)

</details>

---

### Q16 🔴 How would you design a blue-green deployment on ECS?

<details><summary>Click to reveal answer</summary>

**Architecture:**
```
ALB → Listener → Target Group BLUE (current v1)
                → Target Group GREEN (new v2)
```

**Steps:**
1. Deploy new version to GREEN target group
2. Run health checks against GREEN
3. Switch ALB listener rule to point to GREEN (zero-downtime)
4. Keep BLUE running for rollback
5. After validation, drain and deregister BLUE

```bash
# Switch traffic to green (CodeDeploy handles this automatically)
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --default-actions Type=forward,TargetGroupArn=arn:...:targetgroup/green/...

# Or use CodeDeploy AppSpec
# appspec.yaml for ECS blue-green:
```

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: myapp
          ContainerPort: 8080
Hooks:
  - BeforeAllowTraffic: ValidateFunctionArn
  - AfterAllowTraffic: SmokeTestFunctionArn
```

</details>

---

### Q17 🟢 How do you view logs from an ECS Fargate container?

<details><summary>Click to reveal answer</summary>

ECS Fargate uses the `awslogs` log driver to send container stdout/stderr to **CloudWatch Logs**:

```bash
# View logs via CLI
aws logs get-log-events \
  --log-group-name /ecs/myapp \
  --log-stream-name ecs/myapp/abc123def456 \
  --limit 100

# Tail logs (similar to tail -f)
aws logs tail /ecs/myapp --follow

# Or use ECS Exec (like SSH into running container)
aws ecs execute-command \
  --cluster myapp-cluster \
  --task abc123 \
  --container myapp \
  --interactive \
  --command "/bin/sh"
```

ECS Exec requires `enableExecuteCommand: true` on the service and the `ssmmessages` permissions on the task role.

</details>

---

### Q18 🟡 What is an IAM instance profile and how does it differ from an IAM role?

<details><summary>Click to reveal answer</summary>

- **IAM Role**: A set of permissions (policies) that can be assumed by AWS services or users
- **Instance Profile**: A container for an IAM role that is **attached to an EC2 instance**

An instance profile can hold exactly one IAM role. When you create an EC2 IAM role in the console, AWS automatically creates the instance profile with the same name.

```bash
# Create role and attach to instance
aws iam create-role --role-name MyEC2Role --assume-role-policy-document file://trust.json
aws iam create-instance-profile --instance-profile-name MyEC2Profile
aws iam add-role-to-instance-profile --instance-profile-name MyEC2Profile --role-name MyEC2Role
aws ec2 associate-iam-instance-profile --instance-id i-123 --iam-instance-profile Name=MyEC2Profile
```

The EC2 instance metadata service (IMDS) exposes temporary credentials at `http://169.254.169.254/latest/meta-data/iam/security-credentials/MyEC2Role`.

</details>

---

### Q19 🔴 How do you implement graceful shutdown for a Spring Boot app on ECS?

<details><summary>Click to reveal answer</summary>

When ECS stops a task, it sends **SIGTERM** and waits `stopTimeout` seconds before sending SIGKILL.

**Spring Boot config:**
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s   # must be < ECS stopTimeout
```

**ECS service config:**
```json
{
  "stopTimeout": 30
}
```

**ALB deregistration delay:** Set to > 0 to let in-flight requests complete before the target is removed from the target group:
```
deregistration_delay.timeout_seconds = 30
```

**Full shutdown sequence:**
1. ECS sends SIGTERM
2. Spring Boot stops accepting new requests (`/actuator/health` returns `OUT_OF_SERVICE`)
3. ALB health check fails → stops routing new traffic (~10s)
4. In-flight requests complete (up to 25s)
5. Spring context closes (DB connections, executors)
6. Process exits with code 143

</details>

---

### Q20 🟡 What is S3 Transfer Acceleration and when would you use it?

<details><summary>Click to reveal answer</summary>

S3 Transfer Acceleration uses **CloudFront edge locations** to speed up uploads to S3 over long distances. Uploads go to the nearest edge location and travel to S3 via AWS's optimized backbone network.

Enable via:
```bash
aws s3api put-bucket-accelerate-configuration \
  --bucket my-bucket \
  --accelerate-configuration Status=Enabled
```

Use accelerated endpoint:
```java
S3Client client = S3Client.builder()
    .accelerate(true)
    .region(Region.US_EAST_1)
    .build();
```

**Use when**: Users are geographically far from your S3 bucket region (e.g., European users uploading to `us-east-1`). Not beneficial for downloads or same-region transfers.

</details>

---

### Q21 🟢 What is the difference between S3 path-style and virtual-hosted-style URLs?

<details><summary>Click to reveal answer</summary>

- **Virtual-hosted style** (current standard): `https://bucket-name.s3.amazonaws.com/key`
- **Path-style** (deprecated): `https://s3.amazonaws.com/bucket-name/key`

AWS deprecated path-style URLs for new buckets. The SDK v2 uses virtual-hosted style by default. For local testing with MinIO or LocalStack:

```java
S3Client client = S3Client.builder()
    .endpointOverride(URI.create("http://localhost:9000"))
    .forcePathStyle(true)  // required for MinIO/LocalStack
    .region(Region.US_EAST_1)
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create("minioadmin", "minioadmin")))
    .build();
```

</details>

---

### Q22 🔴 How would you architect a Spring Boot application for multi-region AWS deployment?

<details><summary>Click to reveal answer</summary>

**Key components:**

1. **Route 53 with latency-based or geolocation routing** → routes users to nearest region
2. **CloudFront** → CDN for static assets, SSL termination
3. **ALB per region** → load balance across ECS tasks
4. **RDS Global Database** → primary in `us-east-1`, read replicas in `eu-west-1`
5. **DynamoDB Global Tables** → active-active, multi-region writes
6. **SQS/SNS per region** → decouple with event fan-out via EventBridge

**Application config per region:**
```yaml
# application-us-east-1.yml
spring:
  datasource:
    url: jdbc:postgresql://primary.cluster.us-east-1.rds.amazonaws.com:5432/myapp
  cloud:
    aws:
      region:
        static: us-east-1

# application-eu-west-1.yml
spring:
  datasource:
    url: jdbc:postgresql://reader.cluster.eu-west-1.rds.amazonaws.com:5432/myapp
  cloud:
    aws:
      region:
        static: eu-west-1
```

**Challenges**: data consistency (eventual vs strong), session management (use ElastiCache Global Datastore), distributed tracing across regions.

</details>

---

### Q23 🟡 How does the AWS SDK v2 handle retries and timeouts?

<details><summary>Click to reveal answer</summary>

The SDK v2 has built-in retry logic with exponential backoff + jitter:

```java
S3Client client = S3Client.builder()
    .region(Region.US_EAST_1)
    .overrideConfiguration(ClientOverrideConfiguration.builder()
        .retryPolicy(RetryPolicy.builder()
            .numRetries(3)
            .build())
        .apiCallTimeout(Duration.ofSeconds(30))
        .apiCallAttemptTimeout(Duration.ofSeconds(10))
        .build())
    .build();
```

- **`apiCallAttemptTimeout`**: max time for a single attempt
- **`apiCallTimeout`**: max total time including all retries
- Default retry mode: `LEGACY` (3 retries). Also available: `STANDARD`, `ADAPTIVE`

Retryable errors: throttling (429), transient server errors (500, 502, 503), connection errors.

</details>

---

### Q24 🟡 What is VPC endpoint and why would you use it for S3/SQS access?

<details><summary>Click to reveal answer</summary>

A **VPC endpoint** allows resources in a private VPC to access AWS services **without traversing the public internet**.

**Types:**
- **Gateway endpoint** (free): S3, DynamoDB
- **Interface endpoint** (paid): SQS, Secrets Manager, ECR, etc.

**Benefits:**
- Traffic stays within AWS network → lower latency, higher security
- No NAT Gateway required for private subnets → cost savings
- Resource policies can restrict access to only from the VPC

```bash
# Create S3 gateway endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-abc

# Create SQS interface endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.sqs \
  --subnet-ids subnet-abc subnet-def \
  --security-group-ids sg-123
```

</details>

---

### Q25 🔴 How do you implement a circuit breaker pattern for AWS service calls in Spring Boot?

<details><summary>Click to reveal answer</summary>

Use **Resilience4j** to wrap AWS SDK calls:

```xml
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
  <version>2.2.0</version>
</dependency>
```

```java
@Service
@RequiredArgsConstructor
public class ResilientS3Service {

    private final S3Client s3Client;
    private final CircuitBreakerRegistry circuitBreakerRegistry;

    @CircuitBreaker(name = "s3", fallbackMethod = "uploadFallback")
    @Retry(name = "s3")
    public String uploadFile(MultipartFile file, String key) throws IOException {
        // S3 upload logic
        return doUpload(file, key);
    }

    public String uploadFallback(MultipartFile file, String key, Exception ex) {
        log.error("S3 circuit breaker open, storing to local queue");
        localUploadQueue.enqueue(file, key);
        return "queued://" + key;
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      s3:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
  retry:
    instances:
      s3:
        maxAttempts: 3
        waitDuration: 500ms
        exponentialBackoffMultiplier: 2
```

</details>

---

*End of Chapter 4.1 — Deploying Spring Boot on AWS*
