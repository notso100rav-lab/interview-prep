# Spring Boot Auto-Configuration

> 💡 **Interviewer Tip:** Auto-configuration is the heart of Spring Boot's "convention over configuration" philosophy. Interviewers test whether you understand *how* it works under the hood, not just *that* it works. Be ready to explain `@Conditional` annotations and how to write custom auto-configurations.

---

## Table of Contents
1. [The @SpringBootApplication Meta-Annotation](#the-springbootapplication-meta-annotation)
2. [How Auto-Configuration Works](#how-auto-configuration-works)
3. [Conditional Annotations](#conditional-annotations)
4. [Configuration Properties](#configuration-properties)
5. [Interview Questions](#interview-questions)

---

## The @SpringBootApplication Meta-Annotation

```java
@SpringBootApplication
// is equivalent to:
@Configuration           // Marks this as a Spring configuration class
@EnableAutoConfiguration // Triggers auto-configuration mechanism
@ComponentScan           // Scans current package and sub-packages
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

## How Auto-Configuration Works

**Spring Boot 2.x:** Reads `META-INF/spring.factories`
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration,\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**Spring Boot 3.x:** Reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
```
com.example.MyAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

## Conditional Annotations

| Annotation | Activates when |
|-----------|----------------|
| `@ConditionalOnClass` | Class is on classpath |
| `@ConditionalOnMissingClass` | Class is NOT on classpath |
| `@ConditionalOnBean` | Bean of type exists |
| `@ConditionalOnMissingBean` | Bean of type does NOT exist |
| `@ConditionalOnProperty` | Property has specific value |
| `@ConditionalOnWebApplication` | Application is a web application |
| `@ConditionalOnExpression` | SpEL expression is true |
| `@ConditionalOnResource` | Resource exists on classpath |

---

## Interview Questions

### 🟢 Easy

---

**Q1. What does `@SpringBootApplication` do?**

<details><summary>Click to reveal answer</summary>

`@SpringBootApplication` is a meta-annotation composed of three annotations:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration    // which extends @Configuration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { ... })
public @interface SpringBootApplication { }
```

- **`@Configuration`** (via `@SpringBootConfiguration`) — Marks the class as a source of bean definitions
- **`@EnableAutoConfiguration`** — Tells Spring Boot to auto-configure beans based on classpath, beans, and properties
- **`@ComponentScan`** — Scans the package of the annotated class and all sub-packages for Spring components

```java
@SpringBootApplication(
    scanBasePackages = {"com.example.api", "com.example.shared"},
    exclude = {DataSourceAutoConfiguration.class}  // disable specific auto-config
)
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

✅ **Best Practice:** Place `@SpringBootApplication` in the **root package** of your application so that `@ComponentScan` naturally picks up all sub-packages without needing explicit `scanBasePackages`.

</details>

---

**Q2. What is the difference between `application.properties` and `application.yml`?**

<details><summary>Click to reveal answer</summary>

Both serve the same purpose — externalized configuration — but differ in syntax:

```properties
# application.properties — flat key=value
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.jpa.hibernate.ddl-auto=validate
logging.level.com.example=DEBUG
```

```yaml
# application.yml — hierarchical YAML
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
  jpa:
    hibernate:
      ddl-auto: validate
logging:
  level:
    com.example: DEBUG
```

**Lists in both formats:**
```properties
# application.properties
app.allowed-origins[0]=https://example.com
app.allowed-origins[1]=https://api.example.com
```

```yaml
# application.yml
app:
  allowed-origins:
    - https://example.com
    - https://api.example.com
```

✅ **Best Practice:** YAML is preferred for complex, hierarchical configurations. Properties is fine for simple configs. **Never use both in the same project** — properties file takes precedence over YAML if both exist for the same profile.

> 💡 **Interviewer Tip:** A common follow-up is "what if both exist?" — `application.properties` is loaded after `application.yml`, so it overrides YAML values.

</details>

---

**Q3. What are Spring profiles and how do you activate them?**

<details><summary>Click to reveal answer</summary>

Spring profiles allow different configurations for different environments (dev, staging, prod).

**Defining profile-specific configs:**
```yaml
# application.yml (shared)
app:
  name: MyApp

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
spring:
  datasource:
    url: jdbc:h2:mem:devdb

---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
spring:
  datasource:
    url: jdbc:postgresql://prod-server:5432/mydb
```

**Activating profiles:**
```bash
# Environment variable (recommended for production)
SPRING_PROFILES_ACTIVE=prod

# JVM argument
java -jar app.jar --spring.profiles.active=prod

# Multiple profiles
java -jar app.jar --spring.profiles.active=prod,metrics

# application.properties (for default profile)
spring.profiles.active=dev
```

**`@Profile` on beans:**
```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://prod:5432/mydb");
        return ds;
    }

    @Bean
    @Profile("!prod")  // all profiles except prod
    public DataSource testDataSource() {
        return new EmbeddedDatabaseBuilder().build();
    }
}
```

</details>

---

**Q4. How do you access externalized configuration properties in Spring Boot?**

<details><summary>Click to reveal answer</summary>

**Method 1: `@Value`**
```java
@Service
public class EmailService {
    @Value("${app.email.from}")
    private String fromAddress;

    @Value("${app.email.timeout:5000}") // default value
    private int timeoutMs;

    @Value("${app.email.enabled:true}")
    private boolean enabled;
}
```

**Method 2: `@ConfigurationProperties` (preferred for groups)**
```java
@ConfigurationProperties(prefix = "app.email")
@Component
public class EmailProperties {
    private String from;
    private int timeout = 5000;
    private boolean enabled = true;
    private List<String> recipients = new ArrayList<>();
    private Smtp smtp = new Smtp();

    // getters and setters required

    public static class Smtp {
        private String host;
        private int port = 587;
        // getters and setters
    }
}
```

```yaml
app:
  email:
    from: noreply@example.com
    timeout: 3000
    enabled: true
    recipients:
      - admin@example.com
      - ops@example.com
    smtp:
      host: smtp.gmail.com
      port: 587
```

```java
@SpringBootApplication
@EnableConfigurationProperties(EmailProperties.class) // or use @Component on the properties class
public class MyApplication { }
```

✅ **Best Practice:** Prefer `@ConfigurationProperties` over `@Value` for:
- Groups of related properties
- Type-safe binding with validation
- IDE auto-completion (with `spring-configuration-metadata`)

```java
// Add validation to configuration properties
@ConfigurationProperties(prefix = "app.email")
@Validated
public class EmailProperties {
    @NotBlank
    private String from;

    @Min(1000) @Max(30000)
    private int timeout = 5000;
}
```

</details>

---

**Q5. How do you disable a specific auto-configuration?**

<details><summary>Click to reveal answer</summary>

**Method 1: `@SpringBootApplication(exclude = ...)`**
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class,
    JpaRepositoriesAutoConfiguration.class
})
public class MyApplication { }
```

**Method 2: `@EnableAutoConfiguration(exclude = ...)`**
```java
@Configuration
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
public class AppConfig { }
```

**Method 3: `application.properties`**
```properties
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

**Method 4: Providing your own bean (`@ConditionalOnMissingBean`)**
```java
// The auto-config only creates a DataSource if you haven't provided one
@Configuration
public class MyDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // Your custom DataSource — auto-configuration backs off
        return new MyCustomDataSource();
    }
}
```

> 💡 **Interviewer Tip:** Method 4 (providing your own bean) is the most elegant and how Spring Boot is *designed* to be customized. The `@ConditionalOnMissingBean` pattern is central to understanding auto-configuration.

</details>

---

### 🟡 Medium

---

**Q6. Explain how `@ConditionalOnClass` and `@ConditionalOnMissingBean` work together in auto-configuration.**

<details><summary>Click to reveal answer</summary>

This combination is the foundation of Spring Boot's "intelligent defaults" — auto-configure something if the library is present, but back off if the user provides their own.

```java
// Spring Boot's DataSourceAutoConfiguration (simplified)
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
// Only active if DataSource class is on classpath (i.e., JDBC is a dependency)
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, ... })
public class DataSourceAutoConfiguration {

    @Configuration
    @Conditional(EmbeddedDatabaseCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    // Only creates the embedded DB if user hasn't defined their own DataSource
    protected static class EmbeddedDatabaseConfiguration {
        @Bean
        @ConditionalOnMissingBean
        public DataSource dataSource(DataSourceProperties properties) {
            return properties.initializeDataSourceBuilder().build();
        }
    }
}
```

**Reading the logic:**
1. `@ConditionalOnClass(DataSource.class)` → "Is there a JDBC driver on classpath?"
2. `@ConditionalOnMissingBean(DataSource.class)` → "Has the user NOT defined their own DataSource?"
3. Only if both are true → Spring Boot creates a DataSource for you

```java
// Your custom auto-configuration
@AutoConfiguration
@ConditionalOnClass(RedisClient.class)       // Lettuce is on classpath
@ConditionalOnProperty(
    prefix = "app.cache",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true                     // active even if property not set
)
public class RedisCacheAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(CacheManager.class)  // user hasn't provided one
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory).build();
    }
}
```

</details>

---

**Q7. How do you create a custom Spring Boot auto-configuration?**

<details><summary>Click to reveal answer</summary>

```java
// 1. Create the auto-configuration class
@AutoConfiguration  // Spring Boot 3.x (use @Configuration in Boot 2.x)
@ConditionalOnClass(NotificationClient.class)
@EnableConfigurationProperties(NotificationProperties.class)
public class NotificationAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public NotificationClient notificationClient(NotificationProperties props) {
        return NotificationClient.builder()
            .apiKey(props.getApiKey())
            .baseUrl(props.getBaseUrl())
            .build();
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnBean(NotificationClient.class)
    public NotificationService notificationService(NotificationClient client) {
        return new NotificationService(client);
    }
}

// 2. Create the properties class
@ConfigurationProperties(prefix = "notification")
public class NotificationProperties {
    private String apiKey;
    private String baseUrl = "https://api.notifications.com";

    // getters and setters
    public String getApiKey() { return apiKey; }
    public void setApiKey(String k) { this.apiKey = k; }
    public String getBaseUrl() { return baseUrl; }
    public void setBaseUrl(String u) { this.baseUrl = u; }
}
```

```
// 3. Register in Spring Boot 3.x:
// src/main/resources/META-INF/spring/
//   org.springframework.boot.autoconfigure.AutoConfiguration.imports

com.example.notification.NotificationAutoConfiguration
```

```
# Or for Spring Boot 2.x:
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.notification.NotificationAutoConfiguration
```

```java
// 4. Add ordering if needed
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
// or: @AutoConfigureAfter, @AutoConfigureBefore (Boot 2.x)
public class MyAutoConfiguration { }
```

✅ **Best Practice:** Always use `@ConditionalOnMissingBean` on your auto-configured beans so users can override them by providing their own bean definition.

</details>

---

**Q8. What is the Spring Boot configuration property hierarchy (precedence order)?**

<details><summary>Click to reveal answer</summary>

Spring Boot loads properties from many sources, with later sources overriding earlier ones:

```
Priority (lowest → highest):
 1. Default properties (@SpringApplication.setDefaultProperties)
 2. @PropertySource annotations on @Configuration classes
 3. application.properties / application.yml
 4. application-{profile}.properties / application-{profile}.yml
 5. OS environment variables
 6. JVM system properties (java -Dproperty=value)
 7. JNDI attributes (java:comp/env)
 8. ServletContext/ServletConfig init parameters
 9. Command-line arguments (--property=value)
10. @TestPropertySource (in tests)
```

```bash
# Priority example:
# application.properties has: server.port=8080
# Command-line overrides:
java -jar app.jar --server.port=9090
# Result: server.port=9090

# Environment variable (overrides application.properties)
SERVER_PORT=9090 java -jar app.jar
# Spring auto-converts SERVER_PORT → server.port
```

**Relaxed binding rules:**
```properties
# All of these bind to the same property:
app.myValue=hello
app.my-value=hello      # kebab-case (preferred)
app.my_value=hello      # snake_case
APP_MY_VALUE=hello      # environment variable style
```

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String myValue; // binds to all of the above
}
```

> 💡 **Interviewer Tip:** The hierarchy question often comes up in the context of "how do you override a production config without changing code?" — Answer: use environment variables or command-line arguments (highest priority after test annotations).

</details>

---

**Q9. How does `@ConditionalOnProperty` work?**

<details><summary>Click to reveal answer</summary>

`@ConditionalOnProperty` activates a bean or configuration only when specific properties have specific values.

```java
// Basic usage — active only when property exists and equals "true"
@Bean
@ConditionalOnProperty("app.feature.notifications.enabled")
public NotificationService notificationService() {
    return new NotificationService();
}
```

```java
// With havingValue — active when property equals specific value
@Configuration
@ConditionalOnProperty(
    prefix = "app.cache",
    name = "provider",
    havingValue = "redis"
)
public class RedisCacheConfig {
    @Bean
    public CacheManager redisCacheManager() { /* ... */ return null; }
}

@Configuration
@ConditionalOnProperty(
    prefix = "app.cache",
    name = "provider",
    havingValue = "caffeine"
)
public class CaffeineCacheConfig {
    @Bean
    public CacheManager caffeineCacheManager() { /* ... */ return null; }
}
```

```yaml
# application.yml
app:
  cache:
    provider: redis     # activates RedisCacheConfig
  feature:
    notifications:
      enabled: true     # activates NotificationService
```

```java
// matchIfMissing — active even when property is not set
@Bean
@ConditionalOnProperty(
    prefix = "app",
    name = "metrics.enabled",
    havingValue = "true",
    matchIfMissing = true  // enabled by default; set to false to disable
)
public MetricsCollector metricsCollector() {
    return new MetricsCollector();
}
```

🚨 **Common Mistake:** `@ConditionalOnProperty("app.enabled")` checks if the property EXISTS and is not `false`. It's different from `havingValue = "true"`. If the property is set to `"yes"`, it still passes the existence check.

</details>

---

**Q10. What is Spring Cloud Config and how does it relate to Spring Boot externalized configuration?**

<details><summary>Click to reveal answer</summary>

Spring Cloud Config provides centralized, externalized configuration management for distributed systems. Spring Boot connects to it via a config server.

```
Architecture:
[Git Repo / Vault / etc.] ← stores configs
        ↓
[Config Server] ← serves configs via REST API
        ↓
[Spring Boot Services] ← fetch configs at startup
```

```yaml
# bootstrap.yml (Spring Boot 2.x) or application.yml (Spring Boot 3.x)
spring:
  application:
    name: order-service
  config:
    import: "configserver:http://config-server:8888"
  cloud:
    config:
      username: configuser
      password: ${CONFIG_SERVER_PASSWORD}
      fail-fast: true   # fail startup if config server unavailable
      retry:
        max-attempts: 5
        initial-interval: 1000
```

```yaml
# Config server serves profile-specific files:
# order-service.yml           → shared config
# order-service-prod.yml      → prod overrides
# order-service-staging.yml   → staging overrides
```

```java
// Config server setup
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config-server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/configs
          default-label: main
          search-paths: '{application}'
```

✅ **Best Practice:** Use `@RefreshScope` on beans whose properties need to be refreshed at runtime without restarting:
```java
@Component
@RefreshScope
public class FeatureFlags {
    @Value("${feature.new-checkout-flow:false}")
    private boolean newCheckoutFlow;
}
// POST /actuator/refresh triggers re-initialization of @RefreshScope beans
```

</details>

---

**Q11. What does `@EnableAutoConfiguration` actually do internally?**

<details><summary>Click to reveal answer</summary>

`@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`, which:

1. Reads all auto-configuration class names from `spring.factories` / `AutoConfiguration.imports`
2. Filters them based on `@Conditional` annotations
3. Imports the surviving configurations

```java
// Simplified view of what Spring does:
public class AutoConfigurationImportSelector implements DeferredImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 1. Load all candidate configuration class names
        List<String> configurations = loadCandidateConfigurations(metadata);

        // 2. Remove duplicates and apply exclusions
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(metadata);
        configurations.removeAll(exclusions);

        // 3. Filter by @Conditional evaluation
        configurations = filter(configurations, autoConfigurationMetadata);

        // 4. Fire event for reporting
        fireAutoConfigurationImportEvents(configurations, exclusions);

        return StringUtils.toStringArray(configurations);
    }
}
```

**Debugging auto-configuration decisions:**
```java
// Add to your main class or use actuator
@SpringBootApplication
public class DebugApp {
    public static void main(String[] args) {
        // Enable auto-configuration report
        System.setProperty("logging.level.org.springframework.boot.autoconfigure", "DEBUG");
        SpringApplication.run(DebugApp.class, args);
    }
}
```

```properties
# application.properties — see why auto-configs were/weren't applied
debug=true
```

This outputs the **Conditions Evaluation Report**:
```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
DataSourceAutoConfiguration matched:
  - @ConditionalOnClass found required classes 'javax.sql.DataSource'
  - @ConditionalOnMissingBean (types: DataSource) did not find any beans

Negative matches:
-----------------
MongoAutoConfiguration:
  Did not match:
    - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'
```

</details>

---

**Q12. How do you use multiple profiles simultaneously in Spring Boot?**

<details><summary>Click to reveal answer</summary>

```java
// Activating multiple profiles
// application.properties / command line:
spring.profiles.active=prod,metrics,eu-region

// or programmatically:
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("metrics");
app.run(args);
```

```java
// Profile groups (Spring Boot 2.4+)
```
```yaml
# application.yml
spring:
  profiles:
    group:
      prod:
        - prod-db
        - prod-cache
        - prod-security
      local:
        - local-db
        - local-mock-services
```

```bash
# Activating 'prod' group activates prod-db, prod-cache, prod-security
java -jar app.jar --spring.profiles.active=prod
```

```java
// Combining @Profile conditions
@Bean
@Profile({"prod", "staging"})  // active in prod OR staging
public AuditLogger strictAuditLogger() { return new StrictAuditLogger(); }

@Bean
@Profile("!prod")  // active in all profiles EXCEPT prod
public MockEmailService mockEmail() { return new MockEmailService(); }

@Bean
@Profile({"!prod", "!staging"})  // active only in dev/local
public SwaggerConfig swaggerConfig() { return new SwaggerConfig(); }
```

```java
// Injecting different implementations per profile
public interface PaymentGateway { void processPayment(Payment p); }

@Component
@Profile("prod")
public class StripeGateway implements PaymentGateway {
    @Override
    public void processPayment(Payment p) { /* real Stripe API */ }
}

@Component
@Profile({"dev", "test"})
public class MockPaymentGateway implements PaymentGateway {
    @Override
    public void processPayment(Payment p) { /* no-op mock */ }
}
```

</details>

---

### 🔴 Hard

---

**Q13. What is the difference between `@SpringBootConfiguration`, `@Configuration`, and `@TestConfiguration`?**

<details><summary>Click to reveal answer</summary>

```java
// @Configuration — standard Spring configuration
@Configuration
public class AppConfig {
    @Bean
    public SomeService someService() { return new SomeService(); }
}

// @SpringBootConfiguration — extends @Configuration, marks the primary config
// There should be ONLY ONE @SpringBootConfiguration per application
// @SpringBootApplication implicitly includes it
@SpringBootConfiguration  // same as @Configuration but signals "this is the main config"
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// @TestConfiguration — adds beans WITHOUT replacing the main context
@TestConfiguration
public class TestConfig {
    @Bean
    public MockExternalService externalService() {
        return new MockExternalService(); // override for tests
    }
}
```

**Key difference in test context:**

```java
// @Configuration in test = replaces the full context (or requires full setup)
// @TestConfiguration = adds to / overrides beans in the existing context

@SpringBootTest
class OrderServiceTest {

    @TestConfiguration  // adds MockPaymentGateway to the existing app context
    static class TestConfig {
        @Bean
        @Primary
        public PaymentGateway paymentGateway() {
            return new MockPaymentGateway();
        }
    }

    @Autowired
    private OrderService orderService; // uses MockPaymentGateway

    @Test
    void testPlaceOrder() { /* ... */ }
}
```

> 💡 **Interviewer Tip:** `@TestConfiguration` is often confused with `@MockBean`. Use `@MockBean` for Mockito-based mocking; use `@TestConfiguration` when you need a full custom bean implementation (e.g., an in-memory service).

</details>

---

**Q14. How does Spring Boot handle configuration metadata and IDE auto-completion?**

<details><summary>Click to reveal answer</summary>

Spring Boot uses `spring-configuration-metadata.json` for IDE auto-completion in `application.properties` and `application.yml`.

**Auto-generated via annotation processor:**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

```java
@ConfigurationProperties(prefix = "app.payment")
public class PaymentProperties {
    /** Maximum retry attempts for failed payments. */
    private int maxRetries = 3;

    /** Payment gateway API key. */
    private String apiKey;

    /** Supported currencies (ISO 4217). */
    private List<String> supportedCurrencies = List.of("USD", "EUR");

    // getters and setters...
}
```

The annotation processor generates `META-INF/spring-configuration-metadata.json`:
```json
{
  "properties": [
    {
      "name": "app.payment.max-retries",
      "type": "java.lang.Integer",
      "description": "Maximum retry attempts for failed payments.",
      "defaultValue": 3
    },
    {
      "name": "app.payment.api-key",
      "type": "java.lang.String",
      "description": "Payment gateway API key."
    }
  ]
}
```

**Manual metadata (for non-POJO classes):**
```json
// src/main/resources/META-INF/additional-spring-configuration-metadata.json
{
  "properties": [
    {
      "name": "app.feature.dark-mode",
      "type": "java.lang.Boolean",
      "description": "Enable dark mode for the UI.",
      "defaultValue": false
    }
  ],
  "hints": [
    {
      "name": "app.payment.supported-currencies",
      "values": [
        {"value": "USD", "description": "US Dollar"},
        {"value": "EUR", "description": "Euro"},
        {"value": "GBP", "description": "British Pound"}
      ]
    }
  ]
}
```

</details>

---

**Q15. What is `SpringApplication` and what customizations does it allow?**

<details><summary>Click to reveal answer</summary>

`SpringApplication` is the entry point that bootstraps a Spring Boot application. It's highly customizable before `run()` is called.

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(MyApplication.class);

    // Customize banner
    app.setBannerMode(Banner.Mode.OFF);

    // Add custom properties
    Map<String, Object> defaultProps = new HashMap<>();
    defaultProps.put("server.port", 8080);
    defaultProps.put("spring.profiles.default", "dev");
    app.setDefaultProperties(defaultProps);

    // Add additional sources
    app.addPrimarySources(Set.of(AdditionalConfig.class));

    // Add listeners
    app.addListeners(new ApplicationStartingEventListener());

    // Set web application type
    app.setWebApplicationType(WebApplicationType.NONE); // or REACTIVE

    // Disable headless mode (for GUI apps)
    app.setHeadless(false);

    ApplicationContext ctx = app.run(args);
}
```

**Application Events (in order):**
```java
@Component
public class AppLifecycleLogger {

    @EventListener(ApplicationStartingEvent.class)
    public void onStarting(ApplicationStartingEvent e) {
        System.out.println("Starting..."); // Very early, no context yet
    }

    @EventListener(ApplicationEnvironmentPreparedEvent.class)
    public void onEnvPrepared(ApplicationEnvironmentPreparedEvent e) {
        System.out.println("Environment ready");
    }

    @EventListener(ApplicationContextInitializedEvent.class)
    public void onContextInit(ApplicationContextInitializedEvent e) {
        System.out.println("Context initialized");
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady(ApplicationReadyEvent e) {
        System.out.println("Application is ready to serve requests!");
    }

    @EventListener(ApplicationFailedEvent.class)
    public void onFailed(ApplicationFailedEvent e) {
        System.out.println("Application failed: " + e.getException().getMessage());
    }
}
```

✅ **Best Practice:** Use `ApplicationReadyEvent` (not `@PostConstruct`) for tasks that should run after the entire application is fully started and ready to serve.

</details>

---

**Q16. How do `@Import`, `@ImportResource`, and `@ImportAutoConfiguration` differ?**

<details><summary>Click to reveal answer</summary>

```java
// @Import — imports specific @Configuration classes or ImportSelector implementations
@Configuration
@Import({
    SecurityConfig.class,       // another @Configuration class
    JpaConfig.class,            // another @Configuration class
    MyImportSelector.class      // implements ImportSelector for conditional imports
})
public class MainConfig { }

// Custom ImportSelector
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // Dynamically decide which classes to import
        boolean isProduction = /* check environment */ false;
        return isProduction
            ? new String[]{ProductionConfig.class.getName()}
            : new String[]{DevelopmentConfig.class.getName()};
    }
}
```

```java
// @ImportResource — imports XML-based Spring configurations
@Configuration
@ImportResource("classpath:legacy-beans.xml")
public class HybridConfig { }
```

```java
// @ImportAutoConfiguration — used in TESTS to selectively apply auto-configurations
// Useful for slice tests
@DataJpaTest
// @DataJpaTest internally uses @ImportAutoConfiguration to import only JPA-related configs
public class UserRepositoryTest { }

// Custom slice test
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@AutoConfigureWith
@ImportAutoConfiguration({
    ValidationAutoConfiguration.class,
    WebMvcAutoConfiguration.class
})
public @interface WebValidationTest { }
```

```java
// @EnableXxx annotations typically use @Import under the hood
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)  // ← @Import inside @EnableCaching
public @interface EnableCaching { }
```

</details>

---

**Q17. What are the Spring Boot Actuator endpoints and how do you secure them?**

<details><summary>Click to reveal answer</summary>

Spring Boot Actuator exposes production-ready endpoints for monitoring and management.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,beans,conditions
        # use '*' to expose all (not recommended in production)
  endpoint:
    health:
      show-details: when-authorized
    env:
      enabled: true
  server:
    port: 8081  # separate port for management (best practice)
```

**Securing actuator endpoints:**
```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        http
            .requestMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeRequests(auth -> auth
                .requestMatchers(EndpointRequest.to(HealthEndpoint.class)).permitAll()
                .requestMatchers(EndpointRequest.to(InfoEndpoint.class)).permitAll()
                .anyRequest().hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

**Custom health indicator:**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            conn.isValid(2);
            return Health.up()
                .withDetail("database", "PostgreSQL")
                .withDetail("status", "Connected")
                .build();
        } catch (SQLException e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

✅ **Best Practice:** Run actuator on a separate management port (`management.server.port=8081`) and restrict it to internal network access only — never expose sensitive endpoints to the public internet.

</details>

---

**Q18. How do you write a conditional annotation (`@Conditional`) from scratch?**

<details><summary>Click to reveal answer</summary>

```java
// 1. Create the condition class
public class OnLinuxCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String os = context.getEnvironment().getProperty("os.name", "");
        return os.toLowerCase().contains("linux");
    }
}

// 2. Create the meta-annotation (optional but clean)
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnLinuxCondition.class)
public @interface ConditionalOnLinux { }

// 3. Use it
@Bean
@ConditionalOnLinux
public FileWatcher linuxFileWatcher() {
    return new InotifyFileWatcher(); // Linux-specific implementation
}
```

**More complex example — condition reads annotation attributes:**
```java
// Custom condition with annotation attributes
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnFeatureFlagCondition.class)
public @interface ConditionalOnFeatureFlag {
    String flag();
    String defaultValue() default "false";
}

public class OnFeatureFlagCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // Read annotation attributes
        Map<String, Object> attrs =
            metadata.getAnnotationAttributes(ConditionalOnFeatureFlag.class.getName());

        String flag = (String) attrs.get("flag");
        String defaultValue = (String) attrs.get("defaultValue");

        String value = context.getEnvironment()
            .getProperty("feature.flags." + flag, defaultValue);

        return Boolean.parseBoolean(value);
    }
}

// Usage
@Bean
@ConditionalOnFeatureFlag(flag = "new-payment-flow", defaultValue = "false")
public NewPaymentService newPaymentService() {
    return new NewPaymentService();
}
```

```yaml
# application.yml
feature:
  flags:
    new-payment-flow: true
```

</details>

---

**Q19. What is the difference between `@PropertySource` and `@TestPropertySource`?**

<details><summary>Click to reveal answer</summary>

```java
// @PropertySource — loads additional properties files in main code
@Configuration
@PropertySource("classpath:custom.properties")
@PropertySource(value = "classpath:optional.properties", ignoreResourceNotFound = true)
public class AppConfig { }

// Multiple sources
@Configuration
@PropertySources({
    @PropertySource("classpath:app.properties"),
    @PropertySource("classpath:db.properties")
})
public class MultiConfig { }

// Factory for custom formats (not just .properties)
@Configuration
@PropertySource(
    value = "classpath:app.yaml",
    factory = YamlPropertySourceFactory.class
)
public class YamlConfig { }
```

```java
// Custom PropertySourceFactory for YAML
public class YamlPropertySourceFactory implements PropertySourceFactory {
    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource resource)
            throws IOException {
        YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
        factory.setResources(resource.getResource());
        Properties props = factory.getObject();
        return new PropertiesPropertySource(
            resource.getResource().getFilename(), props);
    }
}
```

```java
// @TestPropertySource — overrides properties in tests (highest priority)
@SpringBootTest
@TestPropertySource(properties = {
    "app.payment.max-retries=1",
    "app.feature.dark-mode=true"
})
class PaymentServiceTest { }

// Or load from a test-specific file
@SpringBootTest
@TestPropertySource(locations = "classpath:test-application.properties")
class IntegrationTest { }
```

🚨 **Common Mistake:** `@PropertySource` does NOT work with YAML files by default — you need a custom `PropertySourceFactory`. Use `application.yml` instead for auto-loaded YAML config.

</details>

---

**Q20. How do you test auto-configuration classes?**

<details><summary>Click to reveal answer</summary>

Spring Boot provides `ApplicationContextRunner` for testing auto-configurations without spinning up a full application context.

```java
class NotificationAutoConfigurationTest {

    private final ApplicationContextRunner contextRunner =
        new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(NotificationAutoConfiguration.class));

    @Test
    void configuresNotificationClientWhenClassPresent() {
        contextRunner
            .withPropertyValues(
                "notification.api-key=test-key",
                "notification.base-url=https://api.test.com"
            )
            .run(context -> {
                assertThat(context).hasSingleBean(NotificationClient.class);
                assertThat(context).hasSingleBean(NotificationService.class);

                NotificationClient client = context.getBean(NotificationClient.class);
                assertThat(client.getApiKey()).isEqualTo("test-key");
            });
    }

    @Test
    void backsOffWhenBeanAlreadyDefined() {
        contextRunner
            .withUserConfiguration(CustomNotificationConfig.class) // user-defined bean
            .run(context -> {
                assertThat(context).hasSingleBean(NotificationClient.class);
                // Should use user's bean, not auto-configured one
                assertThat(context.getBean(NotificationClient.class))
                    .isInstanceOf(CustomNotificationClient.class);
            });
    }

    @Test
    void doesNotConfigureWhenPropertyDisabled() {
        contextRunner
            .withPropertyValues("app.notifications.enabled=false")
            .run(context -> {
                assertThat(context).doesNotHaveBean(NotificationService.class);
            });
    }

    @Configuration
    static class CustomNotificationConfig {
        @Bean
        public NotificationClient notificationClient() {
            return new CustomNotificationClient();
        }
    }
}
```

✅ **Best Practice:** Always test your auto-configurations with `ApplicationContextRunner`. Test three scenarios:
1. Happy path — configuration is applied correctly
2. Back-off — user's bean prevents auto-configured bean
3. Exclusion — property/class condition prevents configuration

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| `@SpringBootApplication` | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| Auto-config loading | `spring.factories` (Boot 2.x) / `AutoConfiguration.imports` (Boot 3.x) |
| `@ConditionalOnMissingBean` | Auto-config backs off if user defines their own bean |
| Profile activation | `spring.profiles.active` env var (recommended for prod) |
| Config precedence | Command-line args > env vars > application-{profile}.properties > application.properties |
| Relaxed binding | `SERVER_PORT` → `server.port` automatically |
| Testing auto-config | Use `ApplicationContextRunner` — no full Spring context needed |
| Actuator security | Separate management port + role-based access |
