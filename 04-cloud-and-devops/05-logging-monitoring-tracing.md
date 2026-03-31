# Logging, Monitoring & Tracing

> Java Backend Engineer Interview Prep — Chapter 4.5

---

## Table of Contents
1. [SLF4J + Logback](#slf4j--logback)
2. [Structured JSON Logging](#structured-json-logging)
3. [MDC (Mapped Diagnostic Context)](#mdc-mapped-diagnostic-context)
4. [ELK Stack](#elk-stack)
5. [Distributed Tracing](#distributed-tracing)
6. [Micrometer Tracing + Zipkin](#micrometer-tracing--zipkin)
7. [Q&A](#qa)

---

## SLF4J + Logback

### Dependencies

```xml
<!-- Included in spring-boot-starter — no additional dep needed -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
  <!-- Includes spring-boot-starter-logging → logback-classic + slf4j-api -->
</dependency>

<!-- For JSON logging -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>7.4</version>
</dependency>
```

### logback-spring.xml — Console + File Appenders

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <!-- Import Spring Boot defaults -->
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <!-- Properties from Spring Environment -->
  <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="myapp"/>
  <springProperty scope="context" name="LOG_PATH" source="logging.file.path" defaultValue="logs"/>
  <springProperty scope="context" name="LOG_LEVEL" source="logging.level.root" defaultValue="INFO"/>

  <!-- ── Console Appender ──────────────────────────────────────── -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId:-}] [%X{correlationId:-}] - %msg%n</pattern>
      <charset>UTF-8</charset>
    </encoder>
  </appender>

  <!-- ── Rolling File Appender ─────────────────────────────────── -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_PATH}/${APP_NAME}.log</file>
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId:-}] - %msg%n</pattern>
      <charset>UTF-8</charset>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>${LOG_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
      <maxFileSize>100MB</maxFileSize>
      <maxHistory>30</maxHistory>
      <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
  </appender>

  <!-- ── JSON Appender (for production) ───────────────────────── -->
  <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>traceId</includeMdcKeyName>
      <includeMdcKeyName>spanId</includeMdcKeyName>
      <includeMdcKeyName>correlationId</includeMdcKeyName>
      <includeMdcKeyName>userId</includeMdcKeyName>
      <customFields>{"app":"${APP_NAME}","env":"${SPRING_PROFILES_ACTIVE:-local}"}</customFields>
      <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
        <maxDepthPerCause>10</maxDepthPerCause>
        <rootCauseFirst>true</rootCauseFirst>
      </throwableConverter>
    </encoder>
  </appender>

  <!-- ── Async Wrapper (performance) ──────────────────────────── -->
  <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <discardingThreshold>0</discardingThreshold>
    <queueSize>1024</queueSize>
    <appender-ref ref="FILE"/>
  </appender>

  <!-- ── Log Levels Per Package ────────────────────────────────── -->
  <logger name="com.example.myapp" level="DEBUG" additivity="false">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="ASYNC_FILE"/>
  </logger>

  <logger name="org.springframework.web" level="WARN"/>
  <logger name="org.hibernate.SQL" level="DEBUG"/>
  <logger name="org.hibernate.type.descriptor.sql" level="TRACE"/>
  <logger name="com.zaxxer.hikari" level="WARN"/>

  <!-- ── Profile-Specific Configuration ───────────────────────── -->
  <springProfile name="local,test">
    <root level="DEBUG">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>

  <springProfile name="prod">
    <root level="INFO">
      <appender-ref ref="JSON_CONSOLE"/>
      <appender-ref ref="ASYNC_FILE"/>
    </root>
  </springProfile>

  <springProfile name="!prod">
    <root level="${LOG_LEVEL}">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>

</configuration>
```

### application.yml Logging Config

```yaml
logging:
  level:
    root: INFO
    com.example.myapp: DEBUG
    org.springframework.security: WARN
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
  file:
    path: /var/log/myapp
    name: /var/log/myapp/application.log
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId:-}] - %msg%n"
  logback:
    rollingpolicy:
      max-file-size: 100MB
      max-history: 30
      total-size-cap: 3GB
```

### Using SLF4J in Code

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrderService {

    // Declare as static final — one logger per class
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    // With Lombok @Slf4j annotation (alternative)
    // @Slf4j annotates the class and creates: Logger log = LoggerFactory.getLogger(...)

    public Order createOrder(OrderRequest request) {
        log.debug("Creating order for user: {}, items: {}", request.getUserId(), request.getItemCount());

        try {
            Order order = processOrder(request);
            log.info("Order created successfully: orderId={}, userId={}, total={}",
                order.getId(), request.getUserId(), order.getTotal());
            return order;

        } catch (InsufficientInventoryException e) {
            log.warn("Insufficient inventory for order: itemId={}, requested={}",
                e.getItemId(), e.getQuantityRequested());
            throw e;

        } catch (Exception e) {
            log.error("Unexpected error creating order for userId={}", request.getUserId(), e);
            throw new OrderProcessingException("Failed to create order", e);
        }
    }
}
```

---

## Structured JSON Logging

### LogstashEncoder Configuration

```xml
<!-- logback-spring.xml — JSON encoder -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <!-- Standard fields: @timestamp, @version, message, logger_name, thread_name, level -->

    <!-- Include MDC fields -->
    <includeMdcKeyName>traceId</includeMdcKeyName>
    <includeMdcKeyName>spanId</includeMdcKeyName>
    <includeMdcKeyName>correlationId</includeMdcKeyName>
    <includeMdcKeyName>userId</includeMdcKeyName>
    <includeMdcKeyName>requestId</includeMdcKeyName>

    <!-- Static custom fields -->
    <customFields>{"service":"myapp","env":"prod","version":"1.0.0"}</customFields>

    <!-- Rename fields -->
    <fieldNames>
      <timestamp>@timestamp</timestamp>
      <message>message</message>
      <logger>logger</logger>
      <thread>thread</thread>
      <levelValue>[ignore]</levelValue>  <!-- exclude level value -->
    </fieldNames>

    <!-- Short stack traces -->
    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
      <maxDepthPerCause>20</maxDepthPerCause>
      <maxLength>4096</maxLength>
      <rootCauseFirst>true</rootCauseFirst>
      <exclude>sun\.reflect\..*</exclude>
      <exclude>net\.sf\.cglib\..*</exclude>
    </throwableConverter>
  </encoder>
</appender>
```

### JSON Log Output Example

```json
{
  "@timestamp": "2024-01-15T10:30:45.123Z",
  "message": "Order created successfully: orderId=12345, userId=USER-001, total=99.99",
  "logger": "com.example.OrderService",
  "thread": "http-nio-8080-exec-1",
  "level": "INFO",
  "traceId": "67a9b2c4d5e6f7a8",
  "spanId": "1a2b3c4d5e6f",
  "correlationId": "req-abc123",
  "userId": "USER-001",
  "service": "myapp",
  "env": "prod",
  "version": "1.0.0"
}
```

### Structured Arguments (StructuredArguments)

```java
import static net.logstash.logback.argument.StructuredArguments.*;

@Service
@Slf4j
public class OrderService {

    public Order createOrder(OrderRequest request) {
        // Key-value pairs in JSON output
        log.info("Order created",
            keyValue("orderId", order.getId()),
            keyValue("userId", request.getUserId()),
            keyValue("total", order.getTotal()),
            keyValue("itemCount", request.getItems().size())
        );

        // Array of key-values
        log.info("Processing payment",
            entries(Map.of(
                "paymentMethod", request.getPaymentMethod(),
                "currency", request.getCurrency()
            ))
        );
    }
}
```

---

## MDC (Mapped Diagnostic Context)

### OncePerRequestFilter for Correlation ID

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String REQUEST_ID_KEY = "requestId";
    private static final String CORRELATION_ID_KEY = "correlationId";
    private static final String USER_ID_KEY = "userId";
    private static final String METHOD_KEY = "method";
    private static final String PATH_KEY = "path";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String correlationId = Optional
            .ofNullable(request.getHeader(CORRELATION_ID_HEADER))
            .orElse(UUID.randomUUID().toString());

        String requestId = UUID.randomUUID().toString();

        try {
            // Set MDC values — available in all log statements for this thread
            MDC.put(CORRELATION_ID_KEY, correlationId);
            MDC.put(REQUEST_ID_KEY, requestId);
            MDC.put(METHOD_KEY, request.getMethod());
            MDC.put(PATH_KEY, request.getRequestURI());

            // Add to response so callers can correlate
            response.setHeader(CORRELATION_ID_HEADER, correlationId);
            response.setHeader("X-Request-ID", requestId);

            // If authenticated, add userId to MDC
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth != null && auth.isAuthenticated() && !(auth instanceof AnonymousAuthenticationToken)) {
                MDC.put(USER_ID_KEY, auth.getName());
            }

            filterChain.doFilter(request, response);

        } finally {
            // ALWAYS clear MDC — thread pool reuses threads!
            MDC.remove(CORRELATION_ID_KEY);
            MDC.remove(REQUEST_ID_KEY);
            MDC.remove(METHOD_KEY);
            MDC.remove(PATH_KEY);
            MDC.remove(USER_ID_KEY);
            // Or: MDC.clear(); — clears all keys
        }
    }
}
```

### MDC in Async Code

```java
@Service
@Slf4j
public class AsyncOrderService {

    @Async
    public CompletableFuture<Void> processOrderAsync(Order order) {
        // MDC is NOT automatically propagated to new threads!
        // Use MDC.getCopyOfContextMap() to capture and restore
        Map<String, String> mdcContext = MDC.getCopyOfContextMap();

        return CompletableFuture.runAsync(() -> {
            // Restore MDC in the new thread
            if (mdcContext != null) {
                MDC.setContextMap(mdcContext);
            }
            try {
                log.info("Processing order async: {}", order.getId()); // has MDC values
                doProcess(order);
            } finally {
                MDC.clear();
            }
        });
    }
}
```

```java
// MDC-propagating TaskDecorator for @Async
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setTaskDecorator(runnable -> {
            Map<String, String> mdcContext = MDC.getCopyOfContextMap();
            return () -> {
                try {
                    if (mdcContext != null) MDC.setContextMap(mdcContext);
                    runnable.run();
                } finally {
                    MDC.clear();
                }
            };
        });
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### Logback Pattern with MDC

```xml
<!-- Reference MDC values with %X{key:-default} -->
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [traceId=%X{traceId:-NONE}] [corrId=%X{correlationId:-}] - %msg%n</pattern>
```

---

## ELK Stack

### Architecture Overview

```
Spring Boot App → Log File → Filebeat → Logstash → Elasticsearch → Kibana
                           (shipper)   (transform)    (store)       (visualize)

OR (simpler):
Spring Boot App → stdout/JSON → Logstash → Elasticsearch → Kibana
```

### Filebeat Configuration

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
    json.keys_under_root: true
    json.add_error_key: true
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after

processors:
  - add_host_metadata: ~
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"

output.logstash:
  hosts: ["logstash:5044"]

# Or send directly to Elasticsearch
output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "myapp-logs-%{+yyyy.MM.dd}"
```

### Logstash Pipeline

```ruby
# logstash/pipeline/myapp.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON log
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # Parse timestamp
  date {
    match => ["@timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Add geoIP for client IPs
  if [clientIp] {
    geoip {
      source => "clientIp"
      target => "geoip"
    }
  }

  # Remove noise
  if [level] == "DEBUG" and [env] == "prod" {
    drop { }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "myapp-logs-%{+YYYY.MM.dd}"
    document_type => "_doc"
  }
  # Debug: also print to stdout
  stdout { codec => rubydebug }
}
```

### Docker Compose for ELK

```yaml
# docker-compose-elk.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    ports:
      - "5601:5601"
    networks:
      - elk

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.12.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log/myapp:/var/log/myapp:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    user: root
    networks:
      - elk

volumes:
  elasticsearch_data:

networks:
  elk:
    driver: bridge
```

---

## Distributed Tracing

### Concepts

```
Service A (traceId=abc, spanId=001)
  ↓ HTTP call → X-B3-TraceId: abc, X-B3-SpanId: 002
Service B (traceId=abc, spanId=002)  — same trace, new span
  ↓ HTTP call → X-B3-TraceId: abc, X-B3-SpanId: 003
Service C (traceId=abc, spanId=003)
```

- **Trace**: End-to-end request across all services (same `traceId`)
- **Span**: Single unit of work within a service (unique `spanId`)
- **Parent Span**: The calling span's ID (`parentSpanId`)
- **Context Propagation**: Pass trace IDs via HTTP headers (B3, W3C TraceContext)

### Spring Boot 3.x — Micrometer Tracing

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0    # 1.0 = 100% (use 0.1 for 10% in production)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

spring:
  application:
    name: order-service
```

```java
// Automatic tracing for HTTP, DB, messaging — no code changes needed
// Manual span creation:
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;

    public Order createOrder(OrderRequest request) {
        Span span = tracer.nextSpan().name("create-order").start();
        span.tag("userId", request.getUserId());
        span.tag("itemCount", String.valueOf(request.getItems().size()));

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            return processOrder(request);
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Micrometer Tracing + Zipkin

### Adding TraceId to Log Pattern

```xml
<!-- logback-spring.xml — Spring Boot 3.x auto-populates traceId/spanId in MDC -->
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId:-},%X{spanId:-}] - %msg%n</pattern>
```

```yaml
# application.yml
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
    # Output: INFO [order-service,67a9b2c4d5e6f7a8,1a2b3c4d5e6f]
```

### Zipkin Docker Setup

```bash
# Run Zipkin locally
docker run -d -p 9411:9411 --name zipkin openzipkin/zipkin:latest

# With persistent storage in Elasticsearch
docker run -d -p 9411:9411 \
  -e STORAGE_TYPE=elasticsearch \
  -e ES_HOSTS=http://elasticsearch:9200 \
  --name zipkin \
  openzipkin/zipkin:latest
```

### RestTemplate/WebClient with Trace Propagation

```java
// Spring Boot 3.x auto-instruments RestTemplate and WebClient
// Just inject them as beans and trace context propagates automatically

@Configuration
public class HttpClientConfig {

    // @Autowired ObservationRegistry — auto-injected by Spring Boot
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
        // Micrometer auto-instruments this bean for tracing
    }
}

@Service
@RequiredArgsConstructor
public class InventoryClient {

    private final RestTemplate restTemplate;

    public InventoryResponse checkInventory(String itemId) {
        // traceId and spanId automatically propagated via HTTP headers
        return restTemplate.getForObject(
            "http://inventory-service/api/inventory/{itemId}",
            InventoryResponse.class,
            itemId
        );
    }
}
```

### Feign Client Tracing

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@FeignClient(name = "inventory-service", url = "${inventory.service.url}")
public interface InventoryClient {
    @GetMapping("/api/inventory/{itemId}")
    InventoryResponse getInventory(@PathVariable String itemId);
    // Trace context automatically propagated by Micrometer + Feign integration
}
```

---

## Q&A

### Q1 🟢 What is the difference between SLF4J and Logback?

<details><summary>Click to reveal answer</summary>

- **SLF4J** (Simple Logging Facade for Java): An **API/abstraction** — defines the `Logger` interface and logging methods. No actual logging implementation. Code depends only on SLF4J.

- **Logback**: The **implementation** — actually writes log records to console, files, etc. Implements the SLF4J API.

```java
// Code uses SLF4J API only
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

Logger log = LoggerFactory.getLogger(MyClass.class);
log.info("Hello");  // calls SLF4J API → Logback implementation
```

**Why this separation?**
- Switch logging implementation without changing application code
- Library authors use SLF4J; end users choose Logback vs Log4j2 vs JUL

Spring Boot defaults to Logback. To switch to Log4j2:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

</details>

---

### Q2 🟡 What is MDC and why do you need to clear it?

<details><summary>Click to reveal answer</summary>

**MDC (Mapped Diagnostic Context)** is a thread-local map for storing contextual data (correlationId, userId) that automatically appears in every log statement without passing it explicitly.

```java
MDC.put("correlationId", "req-abc123");
log.info("Processing order");      // log includes correlationId automatically
log.info("Validating payment");    // same — no need to pass correlationId manually
MDC.clear();
```

**Why clear MDC?** Web servers use **thread pools** — the same thread handles multiple requests sequentially. If you don't clear MDC after each request, the next request's logs will have the previous request's correlationId!

```java
// Always clear in finally block
try {
    MDC.put("correlationId", uuid);
    filterChain.doFilter(request, response);
} finally {
    MDC.clear();   // or MDC.remove("correlationId")
}
```

Also important in async code — MDC is per-thread, so async tasks in a thread pool DON'T inherit the calling thread's MDC automatically. Use `MDC.getCopyOfContextMap()` to propagate.

</details>

---

### Q3 🟡 Explain the ELK stack and what each component does.

<details><summary>Click to reveal answer</summary>

| Component | Role |
|-----------|------|
| **Elasticsearch** | Distributed search + analytics engine. Stores and indexes logs. Full-text search with inverted index. |
| **Logstash** | Data pipeline — collect, parse, transform, and forward logs. Heavy, but powerful filters. |
| **Kibana** | Visualization UI — dashboards, log search (Discover), alerting. |
| **Filebeat** | Lightweight log shipper (runs on each server, tails log files, ships to Logstash/ES) |

**Modern alternatives to Logstash:**
- **Fluentd** / **Fluent Bit** — lighter weight, more efficient
- **Vector** — Rust-based, very fast

**Flow:**
```
App → stdout → Filebeat (tail logs)
                  ↓
              Logstash (filter: parse JSON, enrich, drop noise)
                  ↓
            Elasticsearch (index: myapp-logs-2024.01.15)
                  ↓
               Kibana (search, dashboards, alerts)
```

</details>

---

### Q4 🔴 How do you propagate trace context across microservices?

<details><summary>Click to reveal answer</summary>

Trace context is propagated via **HTTP headers**. The calling service adds headers; the called service extracts them and continues the trace.

**B3 propagation (Zipkin standard):**
```
X-B3-TraceId: 67a9b2c4d5e6f7a8
X-B3-SpanId: 1a2b3c4d5e6f
X-B3-ParentSpanId: 0000000000000001
X-B3-Sampled: 1
```

**W3C TraceContext (modern standard):**
```
traceparent: 00-67a9b2c4d5e6f7a8b9c0d1e2f3a4b5c6-1a2b3c4d5e6f-01
tracestate: myapp=somevendordata
```

With **Micrometer Tracing + Brave**, propagation is automatic for:
- `RestTemplate` (interceptor added)
- `WebClient` (filter added)
- `Feign` (interceptor added)
- Kafka (header propagation)
- Spring Security (context preservation)

**Across message queues (SQS/Kafka):**
```java
// Send: embed trace context in message headers
@SqsListener("order-queue")
public void consume(OrderEvent event, MessageHeaders headers) {
    // Micrometer automatically extracts trace context from SQS message attributes
    // when using spring-cloud-aws-sqs with micrometer integration
}
```

</details>

---

### Q5 🟢 What is the `%X{traceId:-}` pattern in Logback?

<details><summary>Click to reveal answer</summary>

`%X{key}` outputs the value of `key` from the **MDC** (Mapped Diagnostic Context).

- `%X{traceId}` — outputs the MDC value for "traceId", or empty string if not set
- `%X{traceId:-}` — outputs "traceId" or empty string (explicit default)
- `%X{traceId:-N/A}` — outputs "traceId" or "N/A" if not set

```xml
<pattern>
  %d{HH:mm:ss} %-5level [traceId=%X{traceId:-NONE}] - %msg%n
</pattern>
```

Spring Boot 3.x with Micrometer Tracing automatically puts `traceId` and `spanId` into MDC, so this pattern works out of the box. Log output:

```
10:30:45 INFO  [traceId=67a9b2c4d5e6f7a8] - Order created: 12345
```

</details>

---

### Q6 🔴 How do you implement distributed tracing for Kafka messages?

<details><summary>Click to reveal answer</summary>

```java
// Producer: inject trace context into Kafka message headers
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    private final Tracer tracer;

    public void publish(OrderEvent event) {
        Span span = tracer.nextSpan().name("kafka-publish").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("topic", "order-events");
            span.tag("orderId", event.getOrderId());

            // Spring Kafka with Micrometer auto-propagates trace context
            // via KafkaHeaders in message headers
            kafkaTemplate.send("order-events", event.getOrderId(), event);
        } finally {
            span.end();
        }
    }
}

// Consumer: extract trace context from Kafka message headers
@KafkaListener(topics = "order-events")
public void consume(
        ConsumerRecord<String, OrderEvent> record,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {

    // Spring Kafka + Micrometer auto-extracts trace context from headers
    // traceId is already set in MDC
    log.info("Consuming order event: {}, traceId from MDC: {}",
        record.value().getOrderId(), MDC.get("traceId"));

    processEvent(record.value());
}
```

```yaml
spring:
  kafka:
    producer:
      properties:
        # Enable micrometer observation
        spring.kafka.producer.observation-enabled: true
    consumer:
      properties:
        spring.kafka.consumer.observation-enabled: true
```

</details>

---

### Q7 🟡 What is sampling in distributed tracing and why is it important?

<details><summary>Click to reveal answer</summary>

**Sampling** determines what percentage of requests have their traces collected and sent to Zipkin/Jaeger. 

**Why not 100%?**
- Sending every trace adds ~1-5ms latency per request
- Storage costs for high-traffic services (1000 req/s × 86400 = 86.4M traces/day)
- Most traces are identical (healthy traffic is boring)

```yaml
management:
  tracing:
    sampling:
      probability: 0.1    # 10% sampling in production
      # 1.0 for development/debugging
      # 0.01 for very high-traffic services
```

**Sampling strategies:**

1. **Probabilistic** (default): random X% of requests
2. **Rate-limiting**: max N traces per second
3. **Adaptive**: adjust rate based on traffic volume
4. **Head-based**: decision made at first span (all or nothing)
5. **Tail-based**: decision after request completes (sample errors always)

```java
// Always sample errors regardless of probability
@Bean
public Sampler sampler() {
    return new Sampler() {
        @Override
        public boolean isSampled(long traceId) {
            // Custom logic — always sample, let Micrometer filter
            return true;
        }
    };
}
```

</details>

---

### Q8 🟡 How does logback-spring.xml differ from logback.xml?

<details><summary>Click to reveal answer</summary>

| Feature | `logback.xml` | `logback-spring.xml` |
|---------|--------------|---------------------|
| **Loaded by** | Logback directly (early) | Spring Boot (later) |
| **Spring features** | ❌ No `<springProfile>`, `<springProperty>` | ✅ Full Spring support |
| **Property access** | Only Logback properties | Spring Environment properties |
| **Recommended** | No (use `-spring` variant) | Yes |

```xml
<!-- logback-spring.xml — Spring Boot features available -->
<springProperty scope="context" name="APP_NAME" source="spring.application.name"/>

<springProfile name="prod">
  <!-- Only active in prod profile -->
  <root level="INFO">
    <appender-ref ref="JSON_CONSOLE"/>
  </root>
</springProfile>

<springProfile name="!prod">
  <!-- Active in all non-prod profiles -->
  <root level="DEBUG">
    <appender-ref ref="CONSOLE"/>
  </root>
</springProfile>
```

`<springProperty>` reads from Spring's `Environment` (application.yml, env vars, etc.).
`<springProfile>` activates config based on active Spring profiles.
Neither is available in plain `logback.xml`.

</details>

---

### Q9 🔴 How would you implement request logging with timing in Spring Boot?

<details><summary>Click to reveal answer</summary>

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)   // after CorrelationIdFilter
@Slf4j
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Set<String> EXCLUDED_PATHS = Set.of(
        "/actuator/health", "/actuator/prometheus"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        if (EXCLUDED_PATHS.contains(request.getRequestURI())) {
            chain.doFilter(request, response);
            return;
        }

        long startTime = System.currentTimeMillis();
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper wrappedResponse = new ContentCachingResponseWrapper(response);

        try {
            chain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            int status = wrappedResponse.getStatus();

            if (status >= 400) {
                log.warn("HTTP {} {} → {} ({}ms) | body={}",
                    request.getMethod(), request.getRequestURI(),
                    status, duration,
                    getRequestBody(wrappedRequest));
            } else {
                log.info("HTTP {} {} → {} ({}ms)",
                    request.getMethod(), request.getRequestURI(),
                    status, duration);
            }

            wrappedResponse.copyBodyToResponse();
        }
    }

    private String getRequestBody(ContentCachingRequestWrapper request) {
        byte[] content = request.getContentAsByteArray();
        if (content.length == 0) return "";
        String body = new String(content, StandardCharsets.UTF_8);
        return body.length() > 500 ? body.substring(0, 500) + "..." : body;
    }
}
```

</details>

---

### Q10 🟡 What is the difference between a trace and a log, and how do they complement each other?

<details><summary>Click to reveal answer</summary>

| Aspect | Logs | Traces |
|--------|------|--------|
| **Purpose** | Record events with details | Track request flow across services |
| **Granularity** | Individual events | Request lifecycle (spans) |
| **Correlation** | Via traceId in log fields | Built-in parent-child span structure |
| **Structure** | Text/JSON per event | Span tree with timing |
| **Storage** | Elasticsearch, S3 | Zipkin, Jaeger, AWS X-Ray |
| **Query** | Full-text search | Trace/span lookup, latency analysis |

**How they complement each other:**

1. Trace tells you: "The /api/orders call took 500ms — 450ms was spent in the DB query span"
2. Log tells you: "The DB query was `SELECT * FROM orders WHERE user_id=? LIMIT 100` and returned 85 rows"

With `traceId` in logs:
```
Kibana: search traceId=abc123 → see all logs for that request
Zipkin: lookup traceId=abc123 → see timing waterfall across all services
```

</details>

---

### Q11 🔴 How do you configure log levels at runtime without restarting the application?

<details><summary>Click to reveal answer</summary>

Spring Boot Actuator provides a `/actuator/loggers` endpoint:

```bash
# View all loggers and their levels
curl http://localhost:8080/actuator/loggers

# View specific logger
curl http://localhost:8080/actuator/loggers/com.example.myapp

# Change log level at runtime (POST)
curl -X POST http://localhost:8080/actuator/loggers/com.example.myapp \
  -H 'Content-Type: application/json' \
  -d '{"configuredLevel": "DEBUG"}'

# Reset to default (null removes override)
curl -X POST http://localhost:8080/actuator/loggers/com.example.myapp \
  -H 'Content-Type: application/json' \
  -d '{"configuredLevel": null}'
```

```yaml
# Secure the endpoint in production
management:
  endpoints:
    web:
      exposure:
        include: health,info,loggers,prometheus
  endpoint:
    loggers:
      enabled: true

# Add security
spring:
  security:
    user:
      name: actuator-user
      password: ${ACTUATOR_PASSWORD}
```

This is invaluable for production debugging — enable DEBUG for a specific package temporarily without restart.

</details>

---

### Q12 🟡 What is the Logstash Logback Encoder and how does it differ from a standard pattern encoder?

<details><summary>Click to reveal answer</summary>

**Pattern Encoder** (default): Outputs human-readable text:
```
2024-01-15 10:30:45 INFO  OrderService - Order created: 12345
```

**LogstashEncoder** (`logstash-logback-encoder`): Outputs structured JSON:
```json
{
  "@timestamp": "2024-01-15T10:30:45.123Z",
  "@version": "1",
  "message": "Order created: 12345",
  "logger_name": "com.example.OrderService",
  "thread_name": "http-nio-exec-1",
  "level": "INFO",
  "level_value": 20000,
  "traceId": "abc123",
  "correlationId": "req-xyz",
  "service": "order-service"
}
```

**Benefits of JSON logging:**
- Machine-parseable — Logstash/Elasticsearch/Splunk can index all fields
- No regex parsing needed
- All MDC fields automatically included
- Stack traces as structured data, not multi-line strings
- Consistent schema across all services

**Configuration:**
```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <customFields>{"service":"myapp","team":"backend"}</customFields>
</encoder>
```

</details>

---

*End of Chapter 4.5 — Logging, Monitoring & Tracing*
