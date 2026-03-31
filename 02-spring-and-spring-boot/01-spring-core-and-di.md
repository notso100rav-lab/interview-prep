# Spring Core & Dependency Injection

**Topics:** `@Component` · `@Service` · `@Repository` · `@Controller` · `@Autowired` · `@Qualifier` · `@Primary` · Constructor vs Setter vs Field Injection · Circular Dependencies · ApplicationContext  
**Questions:** 22 scenarios  
**Difficulty:** 🟢 Easy → 🔴 Hard

---

## Table of Contents

1. [Stereotype Annotations](#stereotype-annotations)
2. [Injection Types](#injection-types)
3. [Qualifier and Primary](#qualifier-and-primary)
4. [Circular Dependencies](#circular-dependencies)
5. [ApplicationContext & BeanFactory](#applicationcontext--beanfactory)
6. [Advanced Scenarios](#advanced-scenarios)

---

## Stereotype Annotations

### Q1 🟢 What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?

<details><summary>Click to reveal answer</summary>

All four are **specializations of `@Component`** — they are all discovered by Spring's component scan. The semantic difference matters for readability and for certain framework behaviors:

| Annotation | Layer | Extra Behavior |
|------------|-------|---------------|
| `@Component` | Any | Generic Spring-managed bean |
| `@Service` | Business/Service | No extra behavior; communicates intent |
| `@Repository` | Data Access | **Exception translation** — wraps persistence exceptions into `DataAccessException` hierarchy |
| `@Controller` | Web/MVC | Marks the class as a Spring MVC controller; works with `@RequestMapping` |
| `@RestController` | Web/MVC | `@Controller` + `@ResponseBody` |

```java
@Component
public class GenericHelper { }

@Service
public class OrderService {
    // business logic
}

@Repository
public class OrderRepository {
    // Spring translates SQLException -> DataAccessException here
}

@Controller
public class OrderController {
    @GetMapping("/orders")
    public String listOrders(Model model) { return "orders"; }
}
```

> 💡 **Interviewer Tip:** Interviewers love asking why `@Repository` is special. The **exception translation** point is the answer they want. Without it, a JPA `PersistenceException` would propagate directly; with it, Spring converts it to a more meaningful `DataAccessException` subclass.

✅ **Best Practice:** Always use the most specific stereotype — it communicates intent and, for `@Repository`, gives you real runtime benefits.

</details>

---

### Q2 🟢 What does `@ComponentScan` do, and what is its default scanning path?

<details><summary>Click to reveal answer</summary>

`@ComponentScan` tells Spring where to look for stereotype-annotated classes. Without it, no beans would be auto-detected.

**Default behavior with Spring Boot:**

`@SpringBootApplication` includes `@ComponentScan` with no explicit `basePackages`, so Spring scans the **package of the annotated class and all its sub-packages**.

```java
// com.example.myapp.Application.java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
// Scans: com.example.myapp and everything below it
```

**Explicitly specifying packages:**

```java
@ComponentScan(basePackages = {
    "com.example.orders",
    "com.example.payments"
})
```

**Using type-safe base package classes:**

```java
@ComponentScan(basePackageClasses = {OrderService.class, PaymentService.class})
```

🚨 **Common Mistake:** Placing `Application.java` in a sub-package like `com.example.myapp.config` will cause Spring to miss beans in sibling packages like `com.example.myapp.service`. Always put the main class at the **root package**.

</details>

---

### Q3 🟡 Your `@Service` bean is not being picked up by Spring. What are the most common reasons and how do you debug it?

<details><summary>Click to reveal answer</summary>

**Common causes and fixes:**

1. **Class is outside the component scan path**
   ```java
   // Fix: move Application.java up one level, or add explicit scan
   @ComponentScan(basePackages = "com.example")
   ```

2. **Missing stereotype annotation** — the class has no `@Component`/`@Service`/etc.

3. **Bean is in a `@Configuration` that is not imported**
   ```java
   @Configuration
   @Import(SomeExternalConfig.class) // explicitly import non-scanned configs
   public class AppConfig { }
   ```

4. **Conditional annotation preventing registration**
   ```java
   @ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
   @Service
   public class FeatureService { } // only registered if property is set
   ```

5. **The class is `abstract`** — Spring cannot instantiate abstract classes.

**Debugging steps:**
```java
// Print all registered beans at startup
@SpringBootApplication
public class Application implements CommandLineRunner {
    @Autowired ApplicationContext ctx;
    
    @Override
    public void run(String... args) {
        Arrays.stream(ctx.getBeanDefinitionNames())
              .sorted()
              .forEach(System.out::println);
    }
}
```

Or enable debug logging:
```yaml
# application.yml
logging:
  level:
    org.springframework: DEBUG
```

> 💡 **Interviewer Tip:** Mentioning `ApplicationContext.getBeanDefinitionNames()` and debug logging shows real debugging experience.

</details>

---

## Injection Types

### Q4 🟡 Compare constructor injection, setter injection, and field injection. Which should you prefer and why?

<details><summary>Click to reveal answer</summary>

**Field Injection** (`@Autowired` on field):
```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // ❌ avoid
}
```
Problems: Cannot be used without Spring (unit testing is painful), hides dependencies, allows creation of objects in invalid state.

---

**Setter Injection** (`@Autowired` on setter):
```java
@Service
public class OrderService {
    private PaymentService paymentService;
    
    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```
Use when: the dependency is **optional** or can change after construction.

---

**Constructor Injection** (preferred):
```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    
    // @Autowired is optional when there is a single constructor (Spring 4.3+)
    public OrderService(PaymentService paymentService,
                        InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
}
```

**Why constructor injection wins:**
| Criteria | Field | Setter | Constructor |
|----------|-------|--------|-------------|
| Immutability (`final`) | ❌ | ❌ | ✅ |
| Testable without Spring | ❌ | ✅ | ✅ |
| Forces dependencies explicit | ❌ | Partial | ✅ |
| Detects circular deps at startup | ❌ | ❌ | ✅ |
| Null safety | ❌ | Risky | ✅ |

✅ **Best Practice:** Use constructor injection everywhere. With Lombok, it's concise:
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
}
```

> 💡 **Interviewer Tip:** Spring's own team recommends constructor injection. Mentioning `@RequiredArgsConstructor` (Lombok) shows you know real-world Spring development.

</details>

---

### Q5 🟡 When would you use setter injection over constructor injection?

<details><summary>Click to reveal answer</summary>

Setter injection is appropriate for **optional dependencies** that have defaults:

```java
@Service
public class NotificationService {
    private MessageFormatter formatter = new DefaultFormatter(); // default

    @Autowired(required = false)
    public void setFormatter(MessageFormatter formatter) {
        this.formatter = formatter;
    }
}
```

Also used when dealing with **legacy frameworks** or **circular dependencies** that cannot be refactored (though fixing the design is always preferred).

Another valid use: **framework-style beans** where you need to configure the bean after construction but before use.

🚨 **Common Mistake:** Using setter injection for mandatory dependencies. If the bean won't function without the dependency, it must be in the constructor.

</details>

---

### Q6 🟢 What happens if Spring finds multiple beans of the same type and you try to `@Autowire` one?

<details><summary>Click to reveal answer</summary>

Spring throws `NoUniqueBeanDefinitionException`:

```
NoUniqueBeanDefinitionException: No qualifying bean of type 'PaymentProcessor' 
available: expected single matching bean but found 2: stripeProcessor,paypalProcessor
```

**Solutions:**

```java
// Solution 1: @Qualifier
@Autowired
@Qualifier("stripeProcessor")
private PaymentProcessor paymentProcessor;

// Solution 2: @Primary on the preferred implementation
@Primary
@Service("stripeProcessor")
public class StripePaymentProcessor implements PaymentProcessor { }

// Solution 3: Inject all implementations
@Autowired
private List<PaymentProcessor> allProcessors;

// Solution 4: Inject as Map (bean name -> bean)
@Autowired
private Map<String, PaymentProcessor> processorMap;
// processorMap.get("stripeProcessor")

// Solution 5: Name the variable to match the bean name
@Autowired
private PaymentProcessor stripeProcessor; // Spring matches by field name
```

> 💡 **Interviewer Tip:** Injecting `List<T>` and `Map<String, T>` is a powerful pattern for strategy/plugin architectures. Show that you know this.

</details>

---

## Qualifier and Primary

### Q7 🟡 Explain the difference between `@Primary` and `@Qualifier`. When would you use each?

<details><summary>Click to reveal answer</summary>

**`@Primary`** — marks one bean as the default when multiple candidates exist. Applied at the **bean definition** side:

```java
@Service
@Primary
public class StripePaymentProcessor implements PaymentProcessor {
    // This is the default when @Autowired PaymentProcessor is used
}

@Service
public class PayPalPaymentProcessor implements PaymentProcessor { }
```

```java
@Autowired
PaymentProcessor processor; // gets StripePaymentProcessor (the @Primary one)
```

---

**`@Qualifier`** — specifies exactly which bean to inject. Applied at the **injection point** side:

```java
@Autowired
@Qualifier("payPalPaymentProcessor")
PaymentProcessor processor; // explicitly gets PayPal
```

**Custom qualifiers** (cleaner):
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
public @interface PayPal { }

@Service
@PayPal
public class PayPalPaymentProcessor implements PaymentProcessor { }

// Usage:
@Autowired @PayPal
PaymentProcessor processor;
```

**When to use which:**
- `@Primary`: you have one clear default implementation for most cases.
- `@Qualifier`: injection points need different implementations.

🚨 **Common Mistake:** Using string-based `@Qualifier("beanName")` is fragile — if you rename the class, the qualifier silently breaks. Custom qualifier annotations are type-safe.

</details>

---

### Q8 🟡 You have a `@Configuration` class that declares a `@Bean` with the same type as a `@Service`. Which one takes priority?

<details><summary>Click to reveal answer</summary>

By default in Spring Boot 2.x, bean definition **overriding is allowed** — the last-registered bean wins. In Spring Boot 2.1+, **overriding is disabled by default** and throws a `BeanDefinitionOverrideException`.

```yaml
# application.yml - to allow override (not recommended)
spring:
  main:
    allow-bean-definition-overriding: true
```

**Priority rules when overriding is allowed:**
- `@Bean` in a `@Configuration` class typically overrides `@Component`-scanned beans because configuration classes are processed first.
- Beans from `@ComponentScan` come after explicit `@Bean` definitions in some cases.

✅ **Best Practice:** Avoid bean name collisions entirely. Use `@ConditionalOnMissingBean` for override-intended patterns:

```java
@Configuration
public class CacheConfig {
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }
}
```

This allows application code to replace the default without disabling the override protection.

</details>

---

## Circular Dependencies

### Q9 🔴 What is a circular dependency in Spring, and how does Spring handle it?

<details><summary>Click to reveal answer</summary>

A **circular dependency** occurs when bean A depends on bean B, and bean B depends on bean A (directly or transitively).

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;
    public ServiceA(ServiceB serviceB) { this.serviceB = serviceB; }
}

@Service
public class ServiceB {
    private final ServiceA serviceA;
    public ServiceB(ServiceA serviceA) { this.serviceA = serviceA; } 
    // 💥 UnsatisfiedDependencyException with constructor injection
}
```

**With constructor injection:** Spring detects the cycle at startup and throws `BeanCurrentlyInCreationException`.

**With field/setter injection (legacy behavior):** Spring resolves cycles using a **three-level cache**:
1. `singletonObjects` — fully initialized beans
2. `earlySingletonObjects` — early-exposed but not fully initialized beans
3. `singletonFactories` — factories that can create early references

Spring exposes a **partially constructed proxy** to break the cycle. This works but is fragile.

**Correct solutions:**

```java
// Solution 1: Refactor — extract shared logic into a third service
@Service
public class SharedService { /* logic both A and B needed */ }

@Service
public class ServiceA {
    private final SharedService sharedService;
    // No dependency on ServiceB
}

// Solution 2: Use @Lazy to delay initialization
@Service
public class ServiceA {
    private final ServiceB serviceB;
    public ServiceA(@Lazy ServiceB serviceB) { this.serviceB = serviceB; }
}

// Solution 3: Use ApplicationContext.getBean() (pull-based, not recommended)
@Service
public class ServiceA {
    @Autowired ApplicationContext ctx;
    
    public void doWork() {
        ServiceB b = ctx.getBean(ServiceB.class); // lazy lookup
    }
}
```

> 💡 **Interviewer Tip:** The "right" answer is that circular dependencies indicate a **design problem**. Always mention refactoring as the first option. Mention `@Lazy` as a tactical fix when refactoring isn't immediately feasible.

🚨 **Common Mistake:** In Spring Boot 2.6+, circular dependencies via field injection are also **detected and rejected by default**. Set `spring.main.allow-circular-references=true` only as a last resort.

</details>

---

### Q10 🔴 You inherit a codebase with circular dependencies everywhere. Spring Boot 2.6 now rejects them. How do you approach fixing this without a massive rewrite?

<details><summary>Click to reveal answer</summary>

**Step 1: Identify all cycles**
```bash
# Spring's error message shows the cycle chain:
# BeanA -> BeanB -> BeanC -> BeanA
```

**Step 2: Tactical fix with `@Lazy` per cycle** (buys time)
```java
@Service
public class ServiceA {
    public ServiceA(@Lazy ServiceB serviceB) { ... }
}
```

**Step 3: Strategic refactoring patterns**

**Extract a mediator/event:**
```java
// Instead of A calling B and B calling A, use events
@Service
public class ServiceA {
    private final ApplicationEventPublisher eventPublisher;
    
    public void process() {
        eventPublisher.publishEvent(new OrderProcessedEvent(this));
    }
}

@Service
public class ServiceB {
    @EventListener
    public void onOrderProcessed(OrderProcessedEvent event) {
        // react without depending on ServiceA
    }
}
```

**Interface segregation:**
```java
// Split ServiceB into two: the part A needs and the full service
public interface ServiceBReader { Data read(); }

@Service
public class ServiceA {
    private final ServiceBReader reader; // only depends on interface
}
```

**Step 4: Enforce going forward**
```yaml
spring:
  main:
    allow-circular-references: false # keep this as false
```

</details>

---

## ApplicationContext & BeanFactory

### Q11 🟡 What is the difference between `BeanFactory` and `ApplicationContext`?

<details><summary>Click to reveal answer</summary>

`BeanFactory` is the core IoC container. `ApplicationContext` extends it with enterprise features:

| Feature | BeanFactory | ApplicationContext |
|---------|-------------|-------------------|
| Basic DI | ✅ | ✅ |
| Lazy loading by default | ✅ | ❌ (eager by default) |
| Internationalization (i18n) | ❌ | ✅ |
| Event publication | ❌ | ✅ |
| AOP integration | Limited | ✅ |
| @PostConstruct / @PreDestroy | ❌ | ✅ |
| `Environment` / profiles | ❌ | ✅ |
| `MessageSource` | ❌ | ✅ |

```java
// BeanFactory (rarely used directly)
BeanFactory factory = new XmlBeanFactory(new FileSystemResource("beans.xml"));

// ApplicationContext (always use this)
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// Spring Boot creates this automatically
ApplicationContext ctx = SpringApplication.run(Application.class, args);
```

✅ **Best Practice:** Always use `ApplicationContext` in production. `BeanFactory` is mostly a historical artifact exposed for library authors.

</details>

---

### Q12 🟢 How do you programmatically retrieve a bean from the Spring context?

<details><summary>Click to reveal answer</summary>

```java
@Component
public class DynamicBeanLookup {
    private final ApplicationContext applicationContext;

    public DynamicBeanLookup(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    public void example() {
        // By type
        PaymentProcessor processor = applicationContext.getBean(PaymentProcessor.class);
        
        // By name
        PaymentProcessor stripe = (PaymentProcessor) applicationContext.getBean("stripePaymentProcessor");
        
        // By name and type (safest)
        PaymentProcessor paypal = applicationContext.getBean("payPalPaymentProcessor", PaymentProcessor.class);
        
        // Check if bean exists
        boolean exists = applicationContext.containsBean("stripePaymentProcessor");
        
        // Get all beans of a type
        Map<String, PaymentProcessor> all = applicationContext.getBeansOfType(PaymentProcessor.class);
    }
}
```

🚨 **Common Mistake:** Calling `getBean()` everywhere is the **Service Locator anti-pattern**. It hides dependencies and makes code harder to test. Only use it for dynamic/runtime bean selection (e.g., strategy pattern based on runtime data).

</details>

---

## Advanced Scenarios

### Q13 🔴 How does Spring AOP work, and how does it interact with DI?

<details><summary>Click to reveal answer</summary>

Spring AOP uses **proxies** — when you inject a `@Service` that has `@Transactional` or custom AOP advice, Spring injects a **proxy** around your bean, not the bean itself.

Two proxy mechanisms:
1. **JDK Dynamic Proxy** — works only if the bean implements an interface
2. **CGLIB Proxy** — subclasses the concrete class (default in Spring Boot)

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        // Spring wraps this in a proxy that manages the transaction
    }
    
    public void processPayment(Order order) {
        this.placeOrder(order); // 🚨 Calls real method, bypasses proxy!
    }
}
```

The self-invocation pitfall: `this.placeOrder()` bypasses the proxy entirely.

**Fix using self-injection (not ideal but works):**
```java
@Service
public class OrderService {
    @Autowired
    private OrderService self; // inject proxy of self
    
    public void processPayment(Order order) {
        self.placeOrder(order); // goes through proxy ✅
    }
}
```

**Better fix: extract to separate bean:**
```java
@Service
public class PaymentService {
    private final OrderService orderService; // proxy injected here
    
    public void processPayment(Order order) {
        orderService.placeOrder(order); // goes through proxy ✅
    }
}
```

> 💡 **Interviewer Tip:** The self-invocation pitfall is one of the most common senior-level Spring interview questions. It applies to `@Transactional`, `@Cacheable`, `@Async`, and any AOP advice.

</details>

---

### Q14 🟡 What is the difference between `@Bean` and `@Component`?

<details><summary>Click to reveal answer</summary>

| | `@Component` | `@Bean` |
|-|-------------|---------|
| Where used | On class | On method in `@Configuration` |
| Spring-managed class? | Must be (annotation on the class itself) | No — method can create any class |
| Third-party classes | ❌ Can't annotate | ✅ Perfect for third-party beans |
| Full configuration semantics | N/A | ✅ Calls between `@Bean` methods are intercepted by CGLIB |

```java
// @Component — you own the class
@Service
public class OrderService { }

// @Bean — creating a third-party object you don't own
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

**The CGLIB proxy trick for `@Configuration`:**
```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }
    
    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource()); // Calls dataSource() but CGLIB intercepts
        // and returns the SAME singleton DataSource, not a new one
    }
}
```

🚨 **Common Mistake:** If you use `@Configuration(proxyBeanMethods = false)` or `@Component` on the config class, calling `dataSource()` directly creates a NEW instance each time.

</details>

---

### Q15 🟡 How do you conditionally register a Spring bean based on an environment property?

<details><summary>Click to reveal answer</summary>

**Option 1: `@ConditionalOnProperty`** (Spring Boot)
```java
@Service
@ConditionalOnProperty(
    name = "feature.cache.enabled",
    havingValue = "true",
    matchIfMissing = false
)
public class RedisCacheService implements CacheService { }
```

**Option 2: `@Profile`**
```java
@Service
@Profile("production")
public class ProductionEmailService implements EmailService { }

@Service
@Profile("!production")
public class MockEmailService implements EmailService { }
```

**Option 3: `@Conditional` with custom condition**
```java
public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getEnvironment()
            .getProperty("os.name", "").toLowerCase().contains("linux");
    }
}

@Bean
@Conditional(OnLinuxCondition.class)
public FileWatcher linuxFileWatcher() { return new InotifyFileWatcher(); }
```

**Option 4: `@ConditionalOnMissingBean`** (most common in auto-configuration)
```java
@Bean
@ConditionalOnMissingBean(EmailService.class)
public EmailService defaultEmailService() {
    return new SmtpEmailService();
}
```

</details>

---

### Q16 🔴 Explain how Spring resolves a `@Autowired` dependency. Walk through the resolution algorithm.

<details><summary>Click to reveal answer</summary>

Spring's autowiring resolution follows these steps in order:

**1. Find all beans matching the declared type**
```java
@Autowired
PaymentProcessor processor; // Find all beans of type PaymentProcessor
```

**2. If exactly one match → inject it**

**3. If multiple matches → apply narrowing rules:**
- First, check for `@Primary` on candidate beans
- Then, check `@Qualifier` at the injection point
- Then, match by **bean name** (field name or parameter name)

```java
// Step-by-step example:
// Registered: stripeProcessor (@Primary), paypalProcessor, squareProcessor

@Autowired
PaymentProcessor processor;
// → stripeProcessor (only @Primary candidate)

@Autowired
@Qualifier("squareProcessor")
PaymentProcessor processor;
// → squareProcessor (explicit qualifier wins over @Primary)

@Autowired
PaymentProcessor paypalProcessor; // field name matches bean name
// → paypalProcessor (name match, but @Primary might still win — @Qualifier is explicit)
```

**4. If zero matches:**
- If `@Autowired(required = true)` (default) → `NoSuchBeanDefinitionException`
- If `@Autowired(required = false)` → field stays null

**5. Optional injection:**
```java
@Autowired
Optional<PaymentProcessor> optionalProcessor;

@Autowired(required = false)
PaymentProcessor processor; // null if not found
```

> 💡 **Interviewer Tip:** Few candidates can articulate the full resolution algorithm. Knowing that name-matching is the tiebreaker after `@Primary` sets you apart.

</details>

---

### Q17 🟡 What is `@DependsOn` and when is it needed?

<details><summary>Click to reveal answer</summary>

`@DependsOn` forces Spring to initialize specified beans before the annotated bean, even when there's no injection-based dependency.

**Use case: Static resource initialization**
```java
@Service
@DependsOn("databaseMigrationService")  // Flyway/Liquibase must run first
public class OrderService {
    // OrderService assumes DB schema is up to date
}

@Service("databaseMigrationService")
public class DatabaseMigrationService implements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        // Run Flyway migrations
    }
}
```

**Another use case: Shared static state**
```java
@Component
@DependsOn("configurationLoader")
public class BusinessRuleEngine {
    // Uses static configuration loaded by ConfigurationLoader
}
```

🚨 **Common Mistake:** `@DependsOn` only controls **creation order**, not destruction order (which is reversed). Also, it doesn't inject the dependency — it just ensures it's created first.

✅ **Best Practice:** If you need `@DependsOn`, it often means your design has hidden coupling. Consider whether explicit injection would be better.

</details>

---

### Q18 🟢 How do you inject a `String` or `int` property value from configuration into a Spring bean?

<details><summary>Click to reveal answer</summary>

```java
@Service
public class StripePaymentService {
    
    @Value("${stripe.api.key}")
    private String apiKey;
    
    @Value("${stripe.timeout.ms:5000}") // default value: 5000
    private int timeoutMs;
    
    @Value("${app.features.enabled:false}")
    private boolean featureEnabled;
    
    // Constructor injection with @Value (testable)
    public StripePaymentService(
            @Value("${stripe.api.key}") String apiKey,
            @Value("${stripe.timeout.ms:5000}") int timeoutMs) {
        this.apiKey = apiKey;
        this.timeoutMs = timeoutMs;
    }
}
```

**Better approach for multiple properties — `@ConfigurationProperties`:**
```java
@ConfigurationProperties(prefix = "stripe")
@Component
public class StripeProperties {
    private String apiKey;
    private int timeoutMs = 5000;
    private boolean enabled = true;
    // getters + setters (or record in Java 16+)
}

// application.yml
// stripe:
//   api-key: sk_test_xxx
//   timeout-ms: 3000
```

> 💡 **Interviewer Tip:** `@ConfigurationProperties` is strongly preferred over many `@Value` annotations. It groups related config, supports validation with `@Validated`, and IDE auto-completion.

</details>

---

### Q19 🔴 What is the Spring `Environment` abstraction? How does property resolution work?

<details><summary>Click to reveal answer</summary>

The `Environment` abstraction represents the current application's runtime environment. It has two key capabilities: **profiles** and **property sources**.

**Property sources (in priority order, highest first):**
1. Devtools global settings
2. `@TestPropertySource` (in tests)
3. Command-line arguments (`--server.port=9090`)
4. `SPRING_APPLICATION_JSON`
5. `ServletConfig`/`ServletContext` init parameters
6. JNDI
7. Java System Properties (`System.getProperties()`)
8. OS environment variables
9. `application-{profile}.properties` outside the jar
10. `application.properties` outside the jar
11. `application-{profile}.properties` inside the jar
12. `application.properties` inside the jar
13. `@PropertySource` on `@Configuration` classes
14. Default properties

```java
@Service
public class ConfigService {
    @Autowired
    private Environment env;
    
    public void inspect() {
        String dbUrl = env.getProperty("spring.datasource.url");
        String dbUrlOrDefault = env.getProperty("spring.datasource.url", "jdbc:h2:mem:test");
        boolean isProd = env.acceptsProfiles(Profiles.of("production"));
        String[] activeProfiles = env.getActiveProfiles();
    }
}
```

✅ **Best Practice:** For Docker/Kubernetes deployments, use environment variables (item 8 in the list) — they naturally override packaged configuration and follow 12-factor app principles.

</details>

---

### Q20 🟡 You have a `@Configuration` class. Explain `@Import`, `@ImportResource`, and when you'd use each.

<details><summary>Click to reveal answer</summary>

**`@Import`** — imports another `@Configuration` class or `ImportSelector`/`ImportBeanDefinitionRegistrar`:
```java
@Configuration
@Import({SecurityConfig.class, CacheConfig.class})
public class AppConfig { }

// Useful for modular configuration:
@Configuration
@Import(DataSourceConfig.class)
public class RepositoryConfig { }
```

**`@ImportResource`** — imports legacy XML bean definitions:
```java
@Configuration
@ImportResource("classpath:legacy-beans.xml")
public class MigrationConfig {
    // New Java config + legacy XML beans together
}
```

**`ImportSelector`** — programmatically select which configurations to import:
```java
public class ConditionalConfigSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        if (isCloud()) {
            return new String[]{CloudCacheConfig.class.getName()};
        }
        return new String[]{LocalCacheConfig.class.getName()};
    }
}
```

> 💡 **Interviewer Tip:** `ImportSelector` is how Spring Boot's auto-configuration works internally. Understanding this shows you can write your own Spring Boot starters.

</details>

---

### Q21 🟡 How do you register a bean dynamically at runtime in Spring?

<details><summary>Click to reveal answer</summary>

```java
// Option 1: BeanDefinitionRegistry (during context refresh)
public class DynamicBeanRegistrar implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        GenericBeanDefinition definition = new GenericBeanDefinition();
        definition.setBeanClass(DynamicService.class);
        definition.setScope(BeanDefinition.SCOPE_SINGLETON);
        registry.registerBeanDefinition("dynamicService", definition);
    }
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) { }
}

// Option 2: ApplicationContext.registerBean() (Spring 5+)
@Component
public class DynamicRegistration implements ApplicationContextAware {
    private GenericApplicationContext context;
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.context = (GenericApplicationContext) applicationContext;
    }
    
    public void registerAtRuntime(String name, Class<?> type) {
        context.registerBean(name, type);
    }
}
```

🚨 **Common Mistake:** Registering beans after the context is fully started can cause issues with beans that have already resolved their dependencies. This pattern is most reliable at startup time.

</details>

---

### Q22 🔴 What is a `BeanPostProcessor`, and how is it different from a `BeanFactoryPostProcessor`?

<details><summary>Click to reveal answer</summary>

**`BeanFactoryPostProcessor`** — runs **after bean definitions are loaded** but **before any beans are instantiated**. Used to modify bean definitions.

```java
@Component
public class PropertyDecryptionPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Can inspect/modify bean definitions
        String[] names = beanFactory.getBeanDefinitionNames();
        for (String name : names) {
            BeanDefinition bd = beanFactory.getBeanDefinition(name);
            // modify property values, change scope, etc.
        }
    }
}
```

**`BeanPostProcessor`** — runs **during bean instantiation**, wrapping each bean before and after initialization:

```java
@Component
public class AuditBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Called BEFORE @PostConstruct / afterPropertiesSet()
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Called AFTER @PostConstruct / afterPropertiesSet()
        // This is where AOP proxies are created!
        if (bean instanceof PaymentProcessor) {
            return Proxy.newProxyInstance(...); // wrap with proxy
        }
        return bean;
    }
}
```

**Execution order:**
```
Bean definitions loaded
   ↓
BeanFactoryPostProcessor.postProcessBeanFactory()
   ↓
Bean instantiation (constructor)
   ↓
Dependency injection
   ↓
BeanPostProcessor.postProcessBeforeInitialization()
   ↓
@PostConstruct / InitializingBean.afterPropertiesSet()
   ↓
BeanPostProcessor.postProcessAfterInitialization()  ← AOP proxies created here
   ↓
Bean ready for use
```

> 💡 **Interviewer Tip:** Spring's `@Autowired`, `@Transactional`, `@Cacheable`, and `@Async` all work through `BeanPostProcessor` implementations. Understanding this is key to understanding Spring internals.

</details>

---

## 🔗 Related Topics

- [Bean Lifecycle & Scopes](./02-bean-lifecycle-and-scopes.md) — what happens after DI
- [Spring Boot Auto-Configuration](./03-spring-boot-auto-configuration.md) — how `@ConditionalOnMissingBean` works
- [Transactions & Propagation](./06-transactions-and-propagation.md) — the self-invocation pitfall in detail
- [Spring Security & JWT](./11-spring-security-jwt.md) — security filter chain as BeanPostProcessor pattern

---

*Next: [Bean Lifecycle & Scopes →](./02-bean-lifecycle-and-scopes.md)*
