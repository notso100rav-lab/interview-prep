# Prometheus & Grafana

> Java Backend Engineer Interview Prep — Chapter 4.6

---

## Table of Contents
1. [Spring Boot Actuator Endpoints](#spring-boot-actuator-endpoints)
2. [Micrometer Fundamentals](#micrometer-fundamentals)
3. [Custom Business Metrics](#custom-business-metrics)
4. [Prometheus Configuration](#prometheus-configuration)
5. [PromQL Queries](#promql-queries)
6. [Grafana Setup & Dashboards](#grafana-setup--dashboards)
7. [Alerting](#alerting)
8. [Q&A](#qa)

---

## Spring Boot Actuator Endpoints

### Dependencies

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,env
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized      # always | never | when-authorized
      show-components: always
      probes:
        enabled: true                    # /liveness and /readiness
    prometheus:
      enabled: true
    metrics:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:local}
      region: ${AWS_REGION:us-east-1}
    distribution:
      percentiles-histogram:
        http.server.requests: true         # enable histogram for HTTP metrics
        spring.data.repository.invocations: true
      percentiles:
        http.server.requests: 0.5, 0.90, 0.95, 0.99
      slo:                                 # service-level objectives
        http.server.requests: 50ms, 100ms, 200ms, 500ms
  server:
    port: 8081    # optional: expose actuator on separate port
```

### Health Endpoint Response

```bash
curl http://localhost:8080/actuator/health | jq
```

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.8"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 107374182400,
        "free": 53687091200,
        "threshold": 10485760
      }
    },
    "livenessState": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
```

### Prometheus Metrics Endpoint

```bash
curl http://localhost:8080/actuator/prometheus | head -60
```

```
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{application="myapp",area="heap",id="G1 Eden Space"} 1.2345678E8
jvm_memory_used_bytes{application="myapp",area="heap",id="G1 Survivor Space"} 5242880.0
jvm_memory_used_bytes{application="myapp",area="nonheap",id="Metaspace"} 7.6543E7

# HELP http_server_requests_seconds Duration of HTTP server request handling
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{application="myapp",exception="None",method="GET",outcome="SUCCESS",status="200",uri="/api/orders"} 1250
http_server_requests_seconds_sum{application="myapp",exception="None",method="GET",outcome="SUCCESS",status="200",uri="/api/orders"} 62.5

# HELP http_server_requests_seconds_max Maximum time of HTTP server requests
http_server_requests_seconds_max{application="myapp",exception="None",method="GET",...} 2.345
```

### Custom Health Indicator

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient client;

    @Override
    public Health health() {
        try {
            boolean up = client.ping();
            if (up) {
                return Health.up()
                    .withDetail("url", client.getBaseUrl())
                    .withDetail("responseTime", client.getLastPingMs() + "ms")
                    .build();
            }
            return Health.down()
                .withDetail("reason", "Payment gateway not responding")
                .build();
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("url", client.getBaseUrl())
                .build();
        }
    }
}
```

---

## Micrometer Fundamentals

### MeterRegistry — Injecting and Using

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry registry;   // auto-injected by Spring Boot

    // Counter: monotonically increasing value
    // Use for: requests, errors, events, processed items
    private final Counter ordersCreatedCounter;
    private final Counter ordersFailedCounter;

    @PostConstruct
    public void initMetrics() {
        // Register counters
        ordersCreatedCounter = Counter.builder("orders.created")
            .description("Total number of orders created")
            .tag("env", "prod")
            .register(registry);

        ordersFailedCounter = Counter.builder("orders.failed")
            .description("Total number of failed order creations")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        try {
            Order order = doCreateOrder(request);
            ordersCreatedCounter.increment();
            return order;
        } catch (Exception e) {
            ordersFailedCounter.increment();
            throw e;
        }
    }
}
```

### Counter

```java
@Component
public class MetricsService {

    private final MeterRegistry registry;

    // Simple increment
    public void recordApiCall(String endpoint, String method) {
        registry.counter("api.calls.total",
            "endpoint", endpoint,
            "method", method)
            .increment();
    }

    // Increment by value
    public void recordBytesReceived(long bytes) {
        registry.counter("bytes.received.total").increment(bytes);
    }
}
```

### Timer

```java
// Timer: measures duration + count
// Use for: method execution time, HTTP calls, DB queries

@Service
@RequiredArgsConstructor
public class PaymentService {

    private final MeterRegistry registry;

    public PaymentResult processPayment(PaymentRequest request) {
        // Option 1: record lambda
        return registry.timer("payment.processing",
                "method", request.getPaymentMethod(),
                "currency", request.getCurrency())
            .record(() -> doProcessPayment(request));
    }

    public void processAsync() throws InterruptedException {
        // Option 2: manual timing
        Timer.Sample sample = Timer.start(registry);
        try {
            doWork();
        } finally {
            sample.stop(registry.timer("async.work.duration"));
        }
    }

    public void annotationBased() {
        // Option 3: @Timed annotation (requires AspectJ)
    }
}

// @Timed annotation
@Service
public class InventoryService {

    @Timed(value = "inventory.check", description = "Time to check inventory",
           percentiles = {0.5, 0.95, 0.99}, histogram = true)
    public InventoryResult checkInventory(String itemId) {
        return inventoryRepository.findByItemId(itemId);
    }
}
```

### Gauge

```java
// Gauge: current value (can go up or down)
// Use for: queue depth, active connections, cache size, thread count

@Component
public class QueueMetrics {

    private final BlockingQueue<Order> orderQueue;
    private final MeterRegistry registry;

    @PostConstruct
    public void registerMetrics() {
        // Gauge — automatically samples queue.size() when Prometheus scrapes
        Gauge.builder("order.queue.size", orderQueue, Collection::size)
            .description("Number of orders waiting to be processed")
            .tag("queue", "main-order-queue")
            .register(registry);

        // Gauge with strong reference (prevents GC)
        Gauge.builder("active.sessions", this, MetricsService::getActiveSessions)
            .description("Number of active user sessions")
            .register(registry);
    }

    private double getActiveSessions() {
        return sessionStore.getActiveSessions();
    }
}
```

### DistributionSummary

```java
// DistributionSummary: records value distribution (not time)
// Use for: request size, response body size, batch size

@Component
public class RequestMetrics {

    private final DistributionSummary requestBodySize;

    public RequestMetrics(MeterRegistry registry) {
        requestBodySize = DistributionSummary.builder("http.request.size")
            .description("HTTP request body size in bytes")
            .baseUnit("bytes")
            .publishPercentiles(0.5, 0.90, 0.99)
            .publishPercentileHistogram()
            .register(registry);
    }

    public void recordRequestSize(int bytes) {
        requestBodySize.record(bytes);
    }
}
```

---

## Custom Business Metrics

### Active Users Gauge

```java
@Component
@RequiredArgsConstructor
public class BusinessMetrics {

    private final MeterRegistry registry;
    private final UserSessionRepository sessionRepository;
    private final OrderRepository orderRepository;

    @PostConstruct
    public void registerGauges() {
        // Active users: current value polled each scrape
        Gauge.builder("business.active_users", sessionRepository,
                repo -> repo.countActiveSessions())
            .description("Number of currently active users")
            .register(registry);

        // Pending orders in queue
        Gauge.builder("business.orders_pending", orderRepository,
                repo -> repo.countByStatus("PENDING"))
            .description("Number of orders pending processing")
            .tag("status", "pending")
            .register(registry);
    }
}
```

### Order Processing Timer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderProcessor {

    private final MeterRegistry registry;
    private final Timer orderProcessingTimer;
    private final Counter ordersProcessed;
    private final Counter ordersFailed;

    @PostConstruct
    public void init() {
        this.orderProcessingTimer = Timer.builder("business.order.processing.duration")
            .description("Time to process an order end-to-end")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .sla(Duration.ofMillis(500), Duration.ofSeconds(1), Duration.ofSeconds(5))
            .register(registry);

        this.ordersProcessed = registry.counter("business.orders.processed",
            "outcome", "success");
        this.ordersFailed = registry.counter("business.orders.processed",
            "outcome", "failure");
    }

    public void process(Order order) {
        Timer.Sample sample = Timer.start(registry);
        try {
            doProcess(order);
            sample.stop(Timer.builder("business.order.processing.duration")
                .tag("type", order.getType())
                .tag("outcome", "success")
                .register(registry));
            ordersProcessed.increment();
        } catch (Exception e) {
            sample.stop(Timer.builder("business.order.processing.duration")
                .tag("type", order.getType())
                .tag("outcome", "failure")
                .register(registry));
            ordersFailed.increment();
            throw e;
        }
    }
}
```

### Revenue Tracking

```java
@Service
public class RevenueMetrics {

    private final DistributionSummary orderRevenue;
    private final Counter totalRevenue;

    public RevenueMetrics(MeterRegistry registry) {
        orderRevenue = DistributionSummary.builder("business.order.revenue")
            .description("Revenue per order in USD cents")
            .baseUnit("cents")
            .publishPercentiles(0.5, 0.90, 0.99)
            .register(registry);

        totalRevenue = Counter.builder("business.revenue.total")
            .description("Total revenue in USD cents")
            .baseUnit("cents")
            .register(registry);
    }

    public void recordOrder(Order order) {
        long amountInCents = order.getTotalAmount().multiply(BigDecimal.valueOf(100)).longValue();
        orderRevenue.record(amountInCents);
        totalRevenue.increment(amountInCents);
    }
}
```

---

## Prometheus Configuration

### prometheus.yml — Scrape Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s          # default: scrape every 15s
  evaluation_interval: 15s      # evaluate rules every 15s
  scrape_timeout: 10s

rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:

  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Spring Boot app
  - job_name: 'spring-boot-myapp'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['myapp:8080']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'myapp-prod'

  # Kubernetes service discovery (EKS)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['production']
    relabel_configs:
      # Only scrape pods with annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom path if annotated
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      # Use custom port if annotated
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      # Add pod metadata as labels
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

  # Node exporter (host metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # PostgreSQL exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Docker Compose for Prometheus + Grafana

```yaml
# docker-compose-monitoring.yml
services:
  prometheus:
    image: prom/prometheus:v2.49.0
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'        # POST /-/reload to reload config
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: https://grafana.myapp.com
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "9093:9093"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

---

## PromQL Queries

### HTTP Request Rate

```promql
# Request rate (requests per second) over last 5 minutes
rate(http_server_requests_seconds_count{application="myapp"}[5m])

# By endpoint and status
rate(http_server_requests_seconds_count{application="myapp", uri="/api/orders"}[5m])

# Error rate (5xx responses only)
rate(http_server_requests_seconds_count{application="myapp", status=~"5.."}[5m])

# Error percentage
sum(rate(http_server_requests_seconds_count{application="myapp", status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count{application="myapp"}[5m]))
* 100
```

### Latency Percentiles

```promql
# P99 latency (requires histogram enabled)
histogram_quantile(0.99,
  sum by (le, uri) (
    rate(http_server_requests_seconds_bucket{application="myapp"}[5m])
  )
)

# P95 and P50 for comparison
histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket[5m])))
histogram_quantile(0.50, sum by (le) (rate(http_server_requests_seconds_bucket[5m])))

# Average latency
rate(http_server_requests_seconds_sum{application="myapp"}[5m])
/
rate(http_server_requests_seconds_count{application="myapp"}[5m])
```

### JVM Metrics

```promql
# JVM heap usage percentage
jvm_memory_used_bytes{application="myapp", area="heap"}
/
jvm_memory_max_bytes{application="myapp", area="heap"}
* 100

# GC pause time rate
rate(jvm_gc_pause_seconds_sum{application="myapp"}[5m])

# GC pause count
rate(jvm_gc_pause_seconds_count[5m])

# Thread count
jvm_threads_live_threads{application="myapp"}

# Active DB connections
hikaricp_connections_active{application="myapp"}

# DB connection pool utilization
hikaricp_connections_active / hikaricp_connections_max
```

### Business Metrics

```promql
# Order processing rate (per minute)
rate(business_orders_processed_total{outcome="success"}[1m]) * 60

# Order failure rate
rate(business_orders_processed_total{outcome="failure"}[5m])
/
rate(business_orders_processed_total[5m])

# Revenue per minute (in dollars)
rate(business_revenue_total_cents_total[1m]) * 60 / 100

# P99 order processing time
histogram_quantile(0.99,
  rate(business_order_processing_duration_seconds_bucket[5m])
)

# Active users
business_active_users

# Increase in orders over last 1 hour
increase(business_orders_processed_total{outcome="success"}[1h])
```

### Infrastructure Queries

```promql
# CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Disk usage
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100

# Network I/O
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

---

## Grafana Setup & Dashboards

### Add Prometheus Data Source (via Provisioning)

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus-ds
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      timeInterval: "15s"
      exemplarTraceIdDestinations:
        - name: traceId
          datasourceUid: tempo-ds   # link to Tempo/Zipkin for traces
```

### Dashboard Provisioning

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: Spring Boot Dashboards
    orgId: 1
    folder: Spring Boot
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

### Dashboard JSON (Key Panels)

```json
{
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
      "targets": [{
        "expr": "sum(rate(http_server_requests_seconds_count{application='myapp'}[5m]))",
        "legendFormat": "req/s"
      }]
    },
    {
      "title": "Error Rate %",
      "type": "gauge",
      "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
      "targets": [{
        "expr": "sum(rate(http_server_requests_seconds_count{status=~'5..'}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100",
        "legendFormat": "Error %"
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 1},
              {"color": "red", "value": 5}
            ]
          },
          "unit": "percent",
          "max": 100
        }
      }
    },
    {
      "title": "P99 Latency",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.99, sum by (le)(rate(http_server_requests_seconds_bucket{application='myapp'}[5m]))) * 1000",
        "legendFormat": "P99"
      }, {
        "expr": "histogram_quantile(0.95, sum by (le)(rate(http_server_requests_seconds_bucket[5m]))) * 1000",
        "legendFormat": "P95"
      }],
      "fieldConfig": {"defaults": {"unit": "ms"}}
    },
    {
      "title": "JVM Memory",
      "type": "timeseries",
      "targets": [{
        "expr": "jvm_memory_used_bytes{area='heap'} / 1024 / 1024",
        "legendFormat": "Heap Used (MB)"
      }, {
        "expr": "jvm_memory_max_bytes{area='heap'} / 1024 / 1024",
        "legendFormat": "Heap Max (MB)"
      }]
    }
  ]
}
```

### Dashboard Variables

```json
{
  "templating": {
    "list": [
      {
        "name": "application",
        "type": "query",
        "query": "label_values(http_server_requests_seconds_count, application)",
        "refresh": 2,
        "multi": true
      },
      {
        "name": "instance",
        "type": "query",
        "query": "label_values(http_server_requests_seconds_count{application='$application'}, instance)",
        "refresh": 2
      },
      {
        "name": "interval",
        "type": "interval",
        "options": ["1m", "5m", "10m", "30m", "1h"],
        "current": "5m"
      }
    ]
  }
}
```

---

## Alerting

### Prometheus Alert Rules

```yaml
# prometheus/alert_rules.yml
groups:
  - name: spring-boot-alerts
    interval: 30s
    rules:

      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))
          > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High HTTP error rate: {{ $value | humanizePercentage }}"
          description: "Error rate above 5% for 2 minutes on {{ $labels.application }}"
          runbook: "https://wiki.myapp.com/runbooks/high-error-rate"
          dashboard: "https://grafana.myapp.com/d/spring-boot/myapp"

      # High latency
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(http_server_requests_seconds_bucket[5m]))
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency is {{ $value | humanizeDuration }}"
          description: "P99 request latency > 2s for 5 minutes"

      # Instance down
      - alert: InstanceDown
        expr: up{job="spring-boot-myapp"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "Spring Boot instance has been unreachable for more than 1 minute"

      # JVM memory
      - alert: HighJvmMemoryUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"}
          > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "JVM heap usage > 85% for {{ $labels.application }}"

      # DB connection pool saturation
      - alert: DbConnectionPoolSaturated
        expr: |
          hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool > 90% utilized"
          description: "{{ $labels.pool }} pool on {{ $labels.application }} is nearly exhausted"
```

### Alertmanager Configuration

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@myapp.com'
  smtp_auth_username: 'alerts@myapp.com'
  smtp_auth_password: '${SMTP_PASSWORD}'

route:
  group_by: ['alertname', 'application']
  group_wait: 10s          # wait before sending first alert
  group_interval: 5m       # wait before sending new alerts for same group
  repeat_interval: 4h      # resend if still firing
  receiver: 'default'

  routes:
    - match:
        severity: critical
      receiver: pagerduty
      group_wait: 0s        # immediate for critical

    - match:
        severity: warning
      receiver: slack
      group_wait: 30s

    - match_re:
        alertname: 'InstanceDown|HighErrorRate'
      receiver: oncall

receivers:
  - name: default
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'

  - name: pagerduty
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_KEY}'
        description: '{{ template "pagerduty.description" . }}'

  - name: slack
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#backend-alerts'
        title: |
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Dashboard:* {{ .Annotations.dashboard }}
          {{ end }}
        send_resolved: true

  - name: oncall
    webhook_configs:
      - url: '${ONCALL_WEBHOOK_URL}'

inhibit_rules:
  # Don't alert for individual instances if the whole service is down
  - source_match:
      alertname: 'ServiceDown'
    target_match:
      alertname: 'InstanceDown'
    equal: ['application']
```

### Grafana Alerts (UI-based)

```yaml
# Grafana alert rule (exported JSON format)
{
  "title": "High Error Rate",
  "condition": "B",
  "data": [
    {
      "refId": "A",
      "queryType": "",
      "datasourceUid": "prometheus-ds",
      "model": {
        "expr": "sum(rate(http_server_requests_seconds_count{status=~'5..'}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100",
        "intervalMs": 1000,
        "maxDataPoints": 43200,
        "refId": "A"
      }
    },
    {
      "refId": "B",
      "queryType": "",
      "datasourceUid": "__expr__",
      "model": {
        "conditions": [{"evaluator": {"params": [5], "type": "gt"}}],
        "refId": "B",
        "type": "classic_conditions"
      }
    }
  ],
  "noDataState": "NoData",
  "execErrState": "Error",
  "for": "2m",
  "annotations": {
    "summary": "Error rate is {{ $values.A.Value | printf '%.2f' }}%"
  },
  "labels": {"severity": "critical", "team": "backend"}
}
```

---

## Q&A

### Q1 🟢 What is Micrometer and what is its relationship to Prometheus?

<details><summary>Click to reveal answer</summary>

**Micrometer** is a metrics **facade/API** for the JVM — like SLF4J but for metrics. It provides a vendor-neutral API (`Counter`, `Timer`, `Gauge`) and publishes metrics to various backends.

**Prometheus** is one of the many backends Micrometer supports (others: Datadog, CloudWatch, InfluxDB, New Relic).

```
Your Code (Counter/Timer/Gauge)
    ↓ Micrometer API
  MeterRegistry
    ↓
PrometheusMeterRegistry → /actuator/prometheus endpoint
    ↓ HTTP scrape
  Prometheus (time-series DB)
    ↓
  Grafana (visualization)
```

By using Micrometer, you can switch from Prometheus to Datadog by just changing the dependency — no code changes.

</details>

---

### Q2 🟢 What is the difference between Counter, Gauge, and Timer in Micrometer?

<details><summary>Click to reveal answer</summary>

| Metric | Direction | Use Case | Example |
|--------|-----------|----------|---------|
| **Counter** | Only increases | Count events | Total requests, total errors |
| **Gauge** | Up or down | Current state | Queue depth, active connections |
| **Timer** | - | Duration + count | Request duration, method execution |
| **DistributionSummary** | - | Value distribution | Request size, batch size |

```java
// Counter — never decreases
Counter.builder("requests.total").register(registry).increment();

// Gauge — sampled at scrape time
Gauge.builder("queue.size", queue, Collection::size).register(registry);

// Timer — measures duration
Timer timer = Timer.builder("http.requests").register(registry);
timer.record(() -> doWork());

// DistributionSummary — records non-time values
DistributionSummary.builder("request.size.bytes").register(registry).record(1024);
```

</details>

---

### Q3 🟡 Explain `rate()` vs `increase()` in PromQL.

<details><summary>Click to reveal answer</summary>

Both work on **counter** metrics (monotonically increasing):

- **`rate(metric[5m])`**: Average per-second rate over the time window
  - Returns: requests per second
  - Use for: dashboards showing throughput, alerting on rate

- **`increase(metric[5m])`**: Total increase in the counter over the time window
  - Returns: total count increase (not per second)
  - Equivalent to: `rate(metric[5m]) * 5 * 60`
  - Use for: "how many errors occurred in the last 5 minutes?"

```promql
# Request rate: 125 req/s
rate(http_server_requests_seconds_count[5m])

# Total requests in last 5 min: 37,500
increase(http_server_requests_seconds_count[5m])

# Error rate alert (> 5% = 0.05)
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
/
rate(http_server_requests_seconds_count[5m])
> 0.05
```

Rule of thumb: use `rate()` for dashboards and alerts, `increase()` for totals in a specific window.

</details>

---

### Q4 🟡 What does `histogram_quantile()` compute and what does it require?

<details><summary>Click to reveal answer</summary>

`histogram_quantile(φ, histogram_metric)` computes the **φ-th percentile** (e.g., 0.99 = P99) from histogram buckets.

**Requirements:**
1. Histogram metric must have `_bucket` suffix (Prometheus histogram type)
2. Buckets must cover the expected value range
3. The metric must be configured with `percentiles-histogram: true` in Micrometer

```yaml
# Spring Boot config to enable histogram
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms, 1s, 2s
```

```promql
# P99 latency in milliseconds
histogram_quantile(0.99,
  sum by (le, uri) (
    rate(http_server_requests_seconds_bucket{application="myapp"}[5m])
  )
) * 1000

# P50/P95/P99 together
histogram_quantile(0.5, sum by(le)(rate(http_server_requests_seconds_bucket[5m])))
histogram_quantile(0.95, sum by(le)(rate(http_server_requests_seconds_bucket[5m])))
histogram_quantile(0.99, sum by(le)(rate(http_server_requests_seconds_bucket[5m])))
```

**Important**: `histogram_quantile` results are approximate (depends on bucket boundaries). For exact percentiles use summary type (but summaries can't be aggregated across instances).

</details>

---

### Q5 🔴 How do you add custom labels (tags) to all metrics in Spring Boot?

<details><summary>Click to reveal answer</summary>

**Option 1: Configuration (global tags)**
```yaml
management:
  metrics:
    tags:
      application: myapp
      environment: ${spring.profiles.active:local}
      region: ${AWS_REGION:us-east-1}
      version: ${APP_VERSION:unknown}
```

**Option 2: MeterFilter bean**
```java
@Bean
MeterFilter commonTagsFilter() {
    return MeterFilter.commonTags(
        "application", "myapp",
        "team", "backend",
        "datacenter", System.getenv("DATACENTER")
    );
}
```

**Option 3: Per-metric tags**
```java
Counter.builder("orders.processed")
    .tag("type", orderType)
    .tag("region", region)
    .register(registry)
    .increment();
```

**Cardinality warning**: Don't use high-cardinality values as tags (userId, orderId) — this creates millions of time series and blows up Prometheus storage. Good tags: status, type, region, environment. Bad tags: userId, requestId.

</details>

---

### Q6 🔴 How do you configure Prometheus scraping in Kubernetes?

<details><summary>Click to reveal answer</summary>

Add annotations to your Pod spec:
```yaml
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
```

Configure Prometheus with pod service discovery:
```yaml
# prometheus.yml
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
```

**Better approach — Prometheus Operator with ServiceMonitor:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
  namespaceSelector:
    matchNames: [production]
```

</details>

---

### Q7 🟡 What is the difference between Prometheus summaries and histograms?

<details><summary>Click to reveal answer</summary>

| Feature | Summary | Histogram |
|---------|---------|-----------|
| **Percentiles** | Exact (client-side) | Approximate (server-side) |
| **Aggregation** | ❌ Cannot aggregate across instances | ✅ Can aggregate with `histogram_quantile` |
| **Storage** | Client stores percentile windows | Fixed number of buckets |
| **Configuration** | Percentiles in app config | Buckets in Prometheus query |
| **Use case** | Single-instance, exact quantiles | Multi-instance, flexible quantiles |

**Histograms** are almost always preferred in microservices because you can aggregate across multiple instances:

```promql
# Works across all instances (histogram)
histogram_quantile(0.99,
  sum by (le) (rate(http_server_requests_seconds_bucket[5m]))
)

# Summary P99 — one per instance, can't sum across instances
http_server_requests_seconds{quantile="0.99"}
```

In Micrometer, histograms are enabled with `percentiles-histogram: true`.

</details>

---

### Q8 🟡 How do you set up a Grafana dashboard variable for multi-instance monitoring?

<details><summary>Click to reveal answer</summary>

Dashboard variables allow users to filter/switch between applications, instances, or environments dynamically.

```
Grafana → Dashboard → Settings → Variables → Add variable:
```

```json
{
  "name": "application",
  "type": "query",
  "datasource": "Prometheus",
  "query": "label_values(http_server_requests_seconds_count, application)",
  "refresh": "On time range change",
  "multi": true,
  "includeAll": true
}
```

Use in panel queries:
```promql
# Use variable with $application
rate(http_server_requests_seconds_count{application=~"$application"}[5m])

# Multi-value regex pattern (when multi: true)
rate(http_server_requests_seconds_count{application=~"$application", instance=~"$instance"}[5m])
```

Common variables: `application`, `instance`, `environment`, `namespace`, `interval`

</details>

---

### Q9 🔴 How do you avoid high cardinality issues with custom metrics?

<details><summary>Click to reveal answer</summary>

High cardinality = too many unique label combinations → millions of time series → Prometheus OOM.

**Bad — high cardinality tags:**
```java
// ❌ Never do this — creates one time series per user/order
Counter.builder("orders.processed")
    .tag("userId", request.getUserId())     // millions of users
    .tag("orderId", order.getId())           // billions of orders
    .register(registry).increment();
```

**Good — low cardinality tags:**
```java
// ✅ Bounded set of values
Counter.builder("orders.processed")
    .tag("type", order.getType())           // "STANDARD"|"PREMIUM"|"EXPRESS" (3 values)
    .tag("status", order.getStatus())       // "SUCCESS"|"FAILED" (2 values)
    .tag("country", order.getCountry())     // ~200 values max
    .register(registry).increment();
```

**Rule of thumb:**
- Tag value cardinality should be < 100 ideally, < 1000 max
- Never tag with: userId, sessionId, orderId, requestId, IP addresses
- Do tag with: HTTP status, endpoint group, payment method, country, outcome

**Detect high cardinality:**
```promql
# Find metrics with too many series
count by (__name__) ({__name__=~".+"}) > 10000
```

</details>

---

### Q10 🟡 What are recording rules in Prometheus and when should you use them?

<details><summary>Click to reveal answer</summary>

Recording rules pre-compute expensive PromQL queries and store results as new time series. This speeds up dashboard load time and alert evaluation.

```yaml
# prometheus/recording_rules.yml
groups:
  - name: myapp.rules
    interval: 1m
    rules:
      # Pre-compute error rate (expensive to compute on every dashboard load)
      - record: job:http_error_rate:ratio5m
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))

      # Pre-compute P99 per endpoint
      - record: job:http_p99_latency:5m
        expr: |
          histogram_quantile(0.99,
            sum by (le, uri) (rate(http_server_requests_seconds_bucket[5m]))
          )

      # Total requests per minute
      - record: job:http_requests:rate1m
        expr: sum by (application, uri) (rate(http_server_requests_seconds_count[1m])) * 60
```

**Use recording rules when:**
- A query takes > 1s to compute
- The same expensive query appears in multiple dashboards
- Alert rules reference the same complex expression

**Use in dashboards/alerts:**
```promql
# Instead of: expensive computation
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))

# Use pre-computed recording rule: fast!
job:http_error_rate:ratio5m
```

</details>

---

*End of Chapter 4.6 — Prometheus & Grafana*
