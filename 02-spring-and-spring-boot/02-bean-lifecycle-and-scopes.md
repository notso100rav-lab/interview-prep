# Bean Lifecycle & Scopes

> 💡 **Interviewer Tip:** Bean lifecycle and scopes are foundational Spring concepts tested in nearly every senior Java backend interview. Expect questions that combine scope with lifecycle callbacks and edge cases like prototype beans injected into singletons.

---

## Table of Contents
1. [Bean Lifecycle Overview](#bean-lifecycle-overview)
2. [Bean Scopes](#bean-scopes)
3. [Interview Questions](#interview-questions)

---

## Bean Lifecycle Overview

The full Spring bean lifecycle follows this order:

```
Instantiation
    → Populate Properties (DI)
    → BeanNameAware.setBeanName()
    → BeanFactoryAware.setBeanFactory()
    → ApplicationContextAware.setApplicationContext()
    → BeanPostProcessor.postProcessBeforeInitialization()
    → @PostConstruct / InitializingBean.afterPropertiesSet()
    → init-method (XML or @Bean(initMethod=...))
    → BeanPostProcessor.postProcessAfterInitialization()
    → [Bean in Use]
    → @PreDestroy / DisposableBean.destroy()
    → destroy-method
```

## Bean Scopes

| Scope | Description | Default |
|-------|-------------|---------|
| `singleton` | One instance per Spring container | ✅ Yes |
| `prototype` | New instance per injection/getBean call | No |
| `request` | One instance per HTTP request (web) | No |
| `session` | One instance per HTTP session (web) | No |
| `application` | One instance per ServletContext (web) | No |
| `websocket` | One instance per WebSocket session | No |

---

## Interview Questions

### 🟢 Easy

---

**Q1. What are the main phases of a Spring bean's lifecycle?**

<details><summary>Click to reveal answer</summary>

The Spring bean lifecycle consists of these phases:

1. **Instantiation** – Spring creates the bean using reflection
2. **Property Population** – Dependencies are injected (DI/autowiring)
3. **Aware interfaces** – Spring calls `setBeanName()`, `setBeanFactory()`, `setApplicationContext()` if implemented
4. **BeanPostProcessor (before init)** – `postProcessBeforeInitialization()` is called
5. **Initialization** – `@PostConstruct`, `InitializingBean.afterPropertiesSet()`, or custom `init-method`
6. **BeanPostProcessor (after init)** – `postProcessAfterInitialization()` is called
7. **Bean in Use** – the bean serves requests
8. **Destruction** – `@PreDestroy`, `DisposableBean.destroy()`, or custom `destroy-method`

```java
@Component
public class MyBean implements InitializingBean, DisposableBean,
        BeanNameAware, ApplicationContextAware {

    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2. InitializingBean.afterPropertiesSet()");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("3. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("4. DisposableBean.destroy()");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("BeanNameAware: " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("ApplicationContextAware set");
    }
}
```

✅ **Best Practice:** Prefer `@PostConstruct` / `@PreDestroy` over implementing `InitializingBean` / `DisposableBean` to avoid tight coupling to Spring interfaces.

</details>

---

**Q2. What is the default bean scope in Spring?**

<details><summary>Click to reveal answer</summary>

The default scope is **singleton** — Spring creates exactly one instance per `ApplicationContext` and returns the same instance for every injection or `getBean()` call.

```java
@Component
// same as @Scope("singleton")
public class UserService {
    // shared instance across the entire application
}
```

```java
@SpringBootTest
class SingletonTest {
    @Autowired UserService s1;
    @Autowired UserService s2;

    @Test
    void sameInstance() {
        assertSame(s1, s2); // passes — same object reference
    }
}
```

> 💡 **Interviewer Tip:** Interviewers often ask "is singleton in Spring the same as the GoF Singleton pattern?" — Answer: **No**. Spring singleton is per-container; you can have multiple containers each with their own instance. GoF singleton is per classloader.

</details>

---

**Q3. What does `@PostConstruct` do and when is it called?**

<details><summary>Click to reveal answer</summary>

`@PostConstruct` marks a method to be called **after** the bean is fully initialized (all dependencies injected) but **before** the bean is put into service.

```java
@Component
public class DatabaseConnectionPool {

    @Value("${db.poolSize:10}")
    private int poolSize;

    private List<Connection> pool;

    @PostConstruct
    public void init() {
        // Safe: poolSize is already injected
        pool = new ArrayList<>(poolSize);
        for (int i = 0; i < poolSize; i++) {
            pool.add(createConnection());
        }
        System.out.println("Pool initialized with " + poolSize + " connections");
    }
}
```

🚨 **Common Mistake:** Trying to use injected dependencies in a constructor — they won't be available yet if using field injection:

```java
@Component
public class BadExample {
    @Autowired
    private SomeService service;

    public BadExample() {
        service.doSomething(); // NullPointerException! service is not yet injected
    }

    @PostConstruct
    public void init() {
        service.doSomething(); // ✅ Safe here
    }
}
```

</details>

---

**Q4. What is the difference between `@PostConstruct` and an `init-method`?**

<details><summary>Click to reveal answer</summary>

Both serve the same purpose but differ in mechanism:

| Feature | `@PostConstruct` | `init-method` |
|---------|-----------------|---------------|
| Source | JSR-250 annotation | Spring-specific |
| Coupling | Minimal (just annotation) | XML or `@Bean(initMethod=...)` |
| Location | On the method | Specified externally |

```java
// Using @PostConstruct
@Component
public class ServiceA {
    @PostConstruct
    public void setup() { /* ... */ }
}

// Using @Bean initMethod
@Configuration
public class AppConfig {
    @Bean(initMethod = "setup", destroyMethod = "cleanup")
    public ServiceB serviceB() {
        return new ServiceB();
    }
}

public class ServiceB {
    public void setup() { /* called after construction */ }
    public void cleanup() { /* called on destruction */ }
}
```

✅ **Best Practice:** Use `@PostConstruct` for annotation-based config; use `initMethod` when configuring third-party classes you can't annotate.

</details>

---

**Q5. How do you define a prototype-scoped bean?**

<details><summary>Click to reveal answer</summary>

Use `@Scope("prototype")` or `@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)`:

```java
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();

    public void addItem(Item item) {
        items.add(item);
    }
}
```

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ReportGenerator {
    // Each request for this bean gets a brand-new instance
}
```

Key behaviors of prototype scope:
- A new instance is created **every time** the bean is requested
- Spring does **not** manage the destruction lifecycle of prototype beans
- `@PreDestroy` is **never called** on prototype beans

```java
@SpringBootTest
class PrototypeTest {
    @Autowired ApplicationContext ctx;

    @Test
    void differentInstances() {
        ShoppingCart c1 = ctx.getBean(ShoppingCart.class);
        ShoppingCart c2 = ctx.getBean(ShoppingCart.class);
        assertNotSame(c1, c2); // passes — different objects
    }
}
```

</details>

---

### 🟡 Medium

---

**Q6. What is the "prototype bean in a singleton" problem and how do you solve it?**

<details><summary>Click to reveal answer</summary>

When a singleton bean holds a reference to a prototype bean via `@Autowired`, the prototype bean is only injected **once** — at the time the singleton is created. Every call to the singleton uses the same prototype instance, defeating the purpose of prototype scope.

```java
// ❌ PROBLEM: Cart is always the same instance
@Service
public class OrderService {
    @Autowired
    private ShoppingCart cart; // Injected once, never refreshed

    public void processOrder() {
        cart.clear(); // mutates the same cart every time!
    }
}
```

**Solutions:**

**1. ApplicationContext lookup (lookup method injection)**
```java
@Service
public class OrderService {
    @Autowired
    private ApplicationContext ctx;

    public void processOrder() {
        ShoppingCart cart = ctx.getBean(ShoppingCart.class); // fresh each time
        cart.clear();
    }
}
```

**2. `@Lookup` annotation (Spring proxy)**
```java
@Service
public abstract class OrderService {

    @Lookup
    public abstract ShoppingCart getCart(); // Spring overrides this

    public void processOrder() {
        ShoppingCart cart = getCart(); // fresh each time via CGLIB
        cart.clear();
    }
}
```

**3. ObjectFactory or ObjectProvider**
```java
@Service
public class OrderService {
    @Autowired
    private ObjectProvider<ShoppingCart> cartProvider;

    public void processOrder() {
        ShoppingCart cart = cartProvider.getObject(); // fresh each time
    }
}
```

✅ **Best Practice:** Prefer `ObjectProvider` — it's the most idiomatic and testable solution.

</details>

---

**Q7. What is `BeanPostProcessor` and how does it differ from `BeanFactoryPostProcessor`?**

<details><summary>Click to reveal answer</summary>

| Aspect | `BeanPostProcessor` | `BeanFactoryPostProcessor` |
|--------|--------------------|-----------------------------|
| Operates on | Bean instances | Bean **definitions** (metadata) |
| Timing | After bean creation | Before bean creation |
| Use case | Wrap beans, add proxies | Modify property placeholders, change bean def |

```java
// BeanPostProcessor - intercepts every bean creation
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean; // must return the bean (or a proxy)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After init: " + beanName);
        return bean;
    }
}
```

```java
// BeanFactoryPostProcessor - modifies bean definitions
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        BeanDefinition bd = factory.getBeanDefinition("myService");
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE); // Change scope programmatically
    }
}
```

> 💡 **Interviewer Tip:** `@PropertySourcesPlaceholderConfigurer` and `PropertyPlaceholderConfigurer` are well-known `BeanFactoryPostProcessor` implementations that resolve `${...}` placeholders.

</details>

---

**Q8. What are web-aware bean scopes and how do you use them?**

<details><summary>Click to reveal answer</summary>

Web-aware scopes (`request`, `session`, `application`) are only available in a web-aware `ApplicationContext` (like Spring MVC).

```java
// Request scope: new instance per HTTP request
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String requestId = UUID.randomUUID().toString();

    public String getRequestId() { return requestId; }
}

// Session scope: new instance per HTTP session
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession {
    private String username;
    private List<String> recentSearches = new ArrayList<>();

    public void setUsername(String u) { this.username = u; }
    public String getUsername() { return username; }
}
```

The `proxyMode = ScopedProxyMode.TARGET_CLASS` is **critical** when injecting short-lived scoped beans into longer-lived beans:

```java
@Service
public class SearchService {
    @Autowired
    private UserSession userSession; // This is a CGLIB proxy, not the actual session bean

    public void search(String query) {
        // Spring resolves the actual session-scoped bean at runtime
        String user = userSession.getUsername();
    }
}
```

🚨 **Common Mistake:** Forgetting `proxyMode` when injecting request/session beans into a singleton — Spring will throw an error because it can't inject a shorter-scoped bean directly into a singleton.

</details>

---

**Q9. What is `@Lazy` and when should you use it?**

<details><summary>Click to reveal answer</summary>

`@Lazy` delays bean initialization until the bean is first requested, rather than at application startup.

```java
// Eager (default) - created at startup even if never used
@Component
public class HeavyService { /* expensive init */ }

// Lazy - only created when first needed
@Component
@Lazy
public class HeavyReportService {
    @PostConstruct
    public void init() {
        System.out.println("Initialized only when first requested");
    }
}

// Lazily injected into a singleton
@Service
public class DashboardService {
    @Autowired
    @Lazy
    private HeavyReportService reportService; // Not created until this field is accessed
}
```

When to use `@Lazy`:
- Expensive beans that may never be used in certain deployment modes
- Breaking circular dependency (as a last resort)
- Improving startup time for large applications

🚨 **Common Mistake:** Using `@Lazy` everywhere "for performance" — it shifts initialization cost to first request, potentially causing runtime latency spikes. Use selectively.

```java
// @Lazy on @Configuration makes ALL @Bean methods in the class lazy
@Configuration
@Lazy
public class ThirdPartyIntegrationConfig {
    @Bean
    public SlackClient slackClient() { /* expensive */ return new SlackClient(); }

    @Bean
    public TwilioClient twilioClient() { /* expensive */ return new TwilioClient(); }
}
```

</details>

---

**Q10. How do you implement the `ApplicationContextAware` interface and when is it useful?**

<details><summary>Click to reveal answer</summary>

`ApplicationContextAware` allows a bean to receive a reference to the `ApplicationContext` that manages it.

```java
@Component
public class BeanLocator implements ApplicationContextAware {

    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        BeanLocator.context = applicationContext;
    }

    public static <T> T getBean(Class<T> beanClass) {
        return context.getBean(beanClass);
    }

    public static <T> T getBean(String name, Class<T> beanClass) {
        return context.getBean(name, beanClass);
    }
}
```

Use cases:
- Service locator pattern (e.g., strategy pattern with runtime bean selection)
- Accessing prototype beans from singleton context
- Dynamic bean lookup

```java
// Strategy pattern using ApplicationContextAware
@Component
public class PaymentStrategyFactory implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public PaymentStrategy getStrategy(String type) {
        return switch (type) {
            case "CARD"   -> ctx.getBean(CardPaymentStrategy.class);
            case "PAYPAL" -> ctx.getBean(PayPalPaymentStrategy.class);
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}
```

✅ **Best Practice:** Avoid using `ApplicationContextAware` as a general service locator — it couples your code to Spring. Prefer constructor injection and `ObjectProvider` for most use cases.

</details>

---

**Q11. What happens to prototype bean destruction callbacks?**

<details><summary>Click to reveal answer</summary>

Spring does **not** call `@PreDestroy` or `DisposableBean.destroy()` on prototype beans. Once Spring hands off the prototype instance, it loses track of it — destruction is the **caller's responsibility**.

```java
@Component
@Scope("prototype")
public class DatabaseMigrationTask implements DisposableBean {

    @PostConstruct
    public void init() {
        System.out.println("Task created"); // ✅ Called
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Task cleaned up"); // ❌ NEVER called by Spring
    }

    @Override
    public void destroy() {
        System.out.println("DisposableBean.destroy()"); // ❌ NEVER called by Spring
    }
}
```

**Workaround using a custom `BeanPostProcessor`:**

```java
@Component
public class DestructionAwareBeanPostProcessor implements BeanPostProcessor {
    private final List<Object> prototypeBeans = new ArrayList<>();

    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean.getClass().isAnnotationPresent(TrackDestruction.class)) {
            prototypeBeans.add(bean);
        }
        return bean;
    }

    @PreDestroy
    public void destroyPrototypes() {
        // Manually call destroy on tracked prototype beans
        for (Object bean : prototypeBeans) {
            if (bean instanceof DisposableBean db) {
                try { db.destroy(); } catch (Exception e) { /* log */ }
            }
        }
    }
}
```

> 💡 **Interviewer Tip:** This is a nuanced lifecycle detail. Knowing that prototype destruction is not managed by Spring demonstrates deep framework knowledge.

</details>

---

**Q12. What is bean definition overriding and how do you control it?**

<details><summary>Click to reveal answer</summary>

Bean definition overriding occurs when two bean definitions exist for the same name. By default in Spring Boot 2.1+, overriding is **disabled** and throws an exception.

```java
// Bean 1
@Configuration
public class PrimaryConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(); // primary
    }
}

// Bean 2 — same name, will override or throw
@Configuration
public class TestConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().build(); // test override
    }
}
```

```yaml
# application.properties — enable overriding (use with caution)
spring.main.allow-bean-definition-overriding=true
```

**Using `@Primary` for precedence:**
```java
@Bean
@Primary
public DataSource primaryDataSource() {
    return new HikariDataSource();
}

@Bean
public DataSource secondaryDataSource() {
    return new HikariDataSource();
}

// Spring injects the @Primary one when there's ambiguity
@Autowired
private DataSource dataSource; // gets primaryDataSource
```

**Using `@Qualifier` for explicit selection:**
```java
@Autowired
@Qualifier("secondaryDataSource")
private DataSource secondary;
```

🚨 **Common Mistake:** Relying on bean override behavior in tests without `allow-bean-definition-overriding=true`. Use `@MockBean` or `@TestConfiguration` instead for clean test overrides.

</details>

---

### 🔴 Hard

---

**Q13. Describe the full execution order when a bean implements multiple lifecycle interfaces simultaneously.**

<details><summary>Click to reveal answer</summary>

```java
@Component
public class FullLifecycleBean implements
        BeanNameAware,
        BeanFactoryAware,
        ApplicationContextAware,
        InitializingBean,
        DisposableBean {

    @Autowired
    private SomeDependency dependency; // injected before any callbacks

    @Override
    public void setBeanName(String name) {
        System.out.println("1. BeanNameAware.setBeanName: " + name);
    }

    @Override
    public void setBeanFactory(BeanFactory factory) {
        System.out.println("2. BeanFactoryAware.setBeanFactory");
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("3. ApplicationContextAware.setApplicationContext");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("4. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("5. InitializingBean.afterPropertiesSet");
    }

    // init-method (if declared) would be "6. init-method"

    @PreDestroy
    public void preDestroy() {
        System.out.println("7. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("8. DisposableBean.destroy");
    }
    // destroy-method (if declared) would be "9. destroy-method"
}
```

**Execution order:**
1. Constructor called
2. Dependency injection (fields, setters)
3. `BeanNameAware.setBeanName()`
4. `BeanFactoryAware.setBeanFactory()`
5. `ApplicationContextAware.setApplicationContext()`
6. `BeanPostProcessor.postProcessBeforeInitialization()` (all registered processors)
7. `@PostConstruct`
8. `InitializingBean.afterPropertiesSet()`
9. `@Bean(initMethod="...")`
10. `BeanPostProcessor.postProcessAfterInitialization()` (all registered processors)
11. **[Bean in use]**
12. `@PreDestroy`
13. `DisposableBean.destroy()`
14. `@Bean(destroyMethod="...")`

> 💡 **Interviewer Tip:** The key insight is that `BeanPostProcessor` wraps the `@PostConstruct`/`afterPropertiesSet()` calls — it's not a separate phase from initialization, it's a wrapper around it.

</details>

---

**Q14. How does Spring handle circular dependencies, and does scope affect this?**

<details><summary>Click to reveal answer</summary>

Spring handles circular dependencies in singleton beans using a **three-level cache**:
- **Level 1:** `singletonObjects` — fully initialized beans
- **Level 2:** `earlySingletonObjects` — early-exposed (partially initialized) beans
- **Level 3:** `singletonFactories` — factories that produce early bean references

```java
// Spring can resolve this circular dependency via early reference exposure
@Service
public class ServiceA {
    @Autowired private ServiceB serviceB; // ← wants ServiceB
}

@Service
public class ServiceB {
    @Autowired private ServiceA serviceA; // ← wants ServiceA (circular!)
}
```

**Spring Boot 2.6+ disables circular dependency detection by default:**
```yaml
spring.main.allow-circular-references=true  # opt-in
```

**Scope and circular dependency:**

Circular dependencies with prototype beans **cannot** be resolved by Spring:
```java
@Component
@Scope("prototype")
public class A {
    @Autowired B b; // Spring can't early-expose prototype beans
}

@Component
@Scope("prototype")
public class B {
    @Autowired A a; // BeanCurrentlyInCreationException!
}
```

**Fix — use `@Lazy` to break the cycle:**
```java
@Service
public class ServiceA {
    @Autowired
    @Lazy // ServiceB is created lazily, breaking the cycle
    private ServiceB serviceB;
}
```

**Better fix — redesign to eliminate the cycle:**
```java
// Extract shared functionality into a third service
@Service
public class SharedService { /* common logic */ }

@Service
public class ServiceA {
    @Autowired private SharedService sharedService;
}

@Service
public class ServiceB {
    @Autowired private SharedService sharedService;
}
```

🚨 **Common Mistake:** Using `@Lazy` as a permanent fix for circular dependencies. It's a code smell — circular dependencies usually indicate a design problem.

</details>

---

**Q15. How does `@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)` work internally?**

<details><summary>Click to reveal answer</summary>

When `proxyMode = ScopedProxyMode.TARGET_CLASS` is used, Spring creates a CGLIB proxy that:
1. Implements the same interface / extends the same class as the scoped bean
2. Holds a reference to the `ApplicationContext`
3. On every method call, resolves the **actual** scoped bean from the context

```java
@Component
@Scope(
    value = WebApplicationContext.SCOPE_REQUEST,
    proxyMode = ScopedProxyMode.TARGET_CLASS
)
public class RequestScopedCart {
    private List<String> items = new ArrayList<>();

    public void addItem(String item) { items.add(item); }
    public List<String> getItems() { return items; }
}

@Service
public class CheckoutService {
    @Autowired
    private RequestScopedCart cart; // Actually a CGLIB proxy!

    public Receipt checkout() {
        // Every call to cart.getItems() goes through the proxy
        // The proxy looks up the real cart from the current HTTP request scope
        return new Receipt(cart.getItems());
    }
}
```

**Proxy types:**
- `ScopedProxyMode.TARGET_CLASS` — CGLIB proxy (subclasses the target, works for classes)
- `ScopedProxyMode.INTERFACES` — JDK dynamic proxy (requires interface)
- `ScopedProxyMode.NO` — no proxy (default, fails if injected into longer-lived bean)

```java
// Inspecting the injected bean
@Service
public class DiagnosticService {
    @Autowired
    private RequestScopedCart cart;

    public void diagnose() {
        System.out.println(cart.getClass().getName());
        // Prints something like: com.example.RequestScopedCart$$EnhancerBySpringCGLIB$$...
        System.out.println(AopUtils.isAopProxy(cart)); // true
        System.out.println(AopUtils.isCglibProxy(cart)); // true
    }
}
```

</details>

---

**Q16. What happens when you use `@Autowired` constructor injection vs field injection in terms of lifecycle?**

<details><summary>Click to reveal answer</summary>

Constructor injection wires dependencies **during instantiation** (phase 1), while field/setter injection happens **after** construction (phase 2).

```java
// Field injection — dependency available AFTER construction
@Service
public class FieldInjectedService {
    @Autowired
    private Repository repo; // null during constructor!

    public FieldInjectedService() {
        // repo is null here — instantiation happens before DI
        repo.findAll(); // NullPointerException!
    }
}

// Constructor injection — dependency available DURING construction
@Service
public class ConstructorInjectedService {
    private final Repository repo;

    @Autowired  // optional in Spring 4.3+ if single constructor
    public ConstructorInjectedService(Repository repo) {
        this.repo = repo; // guaranteed non-null (unless Spring can't satisfy it)
        repo.findAll(); // ✅ Safe!
    }
}
```

**Advantages of constructor injection:**
1. Dependencies are guaranteed non-null (fail-fast at startup)
2. `final` fields — immutable, thread-safe
3. Easier unit testing (no Spring context needed)
4. Circular dependencies are detectable at startup

```java
// Clean constructor injection with Lombok
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;     // final = mandatory
    private final PaymentService paymentService;       // final = mandatory
    private final Optional<DiscountService> discount; // optional dep
}
```

✅ **Best Practice:** Always prefer constructor injection for mandatory dependencies. Use `@Autowired` on field only for optional beans in test configuration or when absolutely needed.

</details>

---

**Q17. How do you create a custom bean scope in Spring?**

<details><summary>Click to reveal answer</summary>

Implement the `Scope` interface and register it with the `ConfigurableBeanFactory`.

```java
public class ThreadScope implements Scope {

    private final ThreadLocal<Map<String, Object>> threadBeans =
            ThreadLocal.withInitial(HashMap::new);

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scope = threadBeans.get();
        return scope.computeIfAbsent(name, k -> objectFactory.getObject());
    }

    @Override
    public Object remove(String name) {
        return threadBeans.get().remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        // optionally store and invoke on thread cleanup
    }

    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }

    @Override
    public String getConversationId() {
        return Thread.currentThread().getName();
    }
}
```

```java
// Register the custom scope
@Configuration
public class ThreadScopeConfig {

    @Bean
    public static CustomScopeConfigurer customScopeConfigurer() {
        CustomScopeConfigurer configurer = new CustomScopeConfigurer();
        configurer.addScope("thread", new ThreadScope());
        return configurer;
    }
}

// Use the custom scope
@Component
@Scope("thread")
public class ThreadLocalContext {
    private String currentUser;

    public void setCurrentUser(String user) { this.currentUser = user; }
    public String getCurrentUser() { return currentUser; }
}
```

</details>

---

**Q18. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?**

<details><summary>Click to reveal answer</summary>

All four are **stereotype annotations** — they all trigger component scanning and register a bean. The differences are semantic and functional:

```
@Component
    ├── @Service    (business logic layer)
    ├── @Repository (data access layer)
    └── @Controller (presentation layer)
         └── @RestController (@Controller + @ResponseBody)
```

```java
@Component  // generic Spring-managed component
public class UtilityComponent { }

@Service    // business service — no extra behavior vs @Component
public class UserService { }

@Repository // DAO — adds exception translation (DataAccessException)
public class UserRepository { }

@Controller // Spring MVC — handles web requests
public class UserController { }

@RestController // @Controller + @ResponseBody
public class UserRestController { }
```

**The only functional difference** is `@Repository`: Spring's `PersistenceExceptionTranslationPostProcessor` translates persistence-specific exceptions (e.g., Hibernate `HibernateException`) into Spring's `DataAccessException` hierarchy.

```java
@Repository
public class JdbcUserRepository {
    // If a database error occurs, Spring wraps it in DataAccessException
    // Instead of: org.hibernate.exception.ConstraintViolationException
    // You get:    org.springframework.dao.DataIntegrityViolationException
}
```

> 💡 **Interviewer Tip:** Candidates often say "@Service and @Component are identical." That's technically true today, but `@Service` communicates **intent** and allows future AOP or behavior additions targeted specifically at service beans.

</details>

---

**Q19. How does Spring's singleton scope ensure thread safety?**

<details><summary>Click to reveal answer</summary>

**It doesn't.** Spring singleton means one instance per container — it does **not** mean thread-safe. Thread safety is the developer's responsibility.

```java
// ❌ NOT thread-safe singleton
@Service
public class CounterService {
    private int count = 0; // shared mutable state!

    public void increment() {
        count++; // race condition — not atomic!
    }

    public int getCount() {
        return count;
    }
}

// ✅ Thread-safe using AtomicInteger
@Service
public class SafeCounterService {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}

// ✅ Thread-safe using synchronized
@Service
public class SynchronizedService {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }
}
```

**Rules for thread-safe Spring singletons:**
1. **No mutable instance state** — prefer stateless services
2. If state is needed, use `ThreadLocal`, `AtomicXxx`, or `synchronized`
3. Inject only thread-safe collaborators (other stateless singletons, connection pools, etc.)

```java
// ✅ Ideal singleton — completely stateless
@Service
public class UserService {
    private final UserRepository repo; // final, thread-safe (stateless repo)

    public UserService(UserRepository repo) { this.repo = repo; }

    public User findById(Long id) {
        return repo.findById(id).orElseThrow(); // no shared mutable state
    }
}
```

</details>

---

**Q20. Explain how Spring uses CGLIB and JDK proxies for AOP and how it relates to bean lifecycle.**

<details><summary>Click to reveal answer</summary>

Spring AOP creates proxies during the `postProcessAfterInitialization` phase via `AbstractAutoProxyCreator` (a `BeanPostProcessor`).

```java
// Original bean
@Service
public class OrderService {
    public void placeOrder(Order order) {
        // business logic
    }
}

// With @Transactional — Spring wraps it in a proxy
@Service
@Transactional
public class OrderService {
    public void placeOrder(Order order) {
        // Spring generates a proxy that:
        // 1. Begins transaction
        // 2. Calls this method
        // 3. Commits or rolls back
    }
}
```

**JDK Proxy vs CGLIB:**

```java
// JDK Proxy — requires interface
public interface OrderService { void placeOrder(Order order); }

@Service
@Transactional
public class OrderServiceImpl implements OrderService {
    @Override
    public void placeOrder(Order order) { /* ... */ }
}
// Spring uses JDK proxy by default if interface present

// CGLIB — subclasses the target class (no interface needed)
@Service
@Transactional
public class ProductService { // no interface!
    public void createProduct(Product p) { /* ... */ }
}
// Spring uses CGLIB proxy
```

**Configuration:**
```java
// Force CGLIB even with interface
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class AppConfig { }
```

**Self-invocation problem (critical interview topic):**
```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        validateOrder(order);         // calls this.validateOrder — bypasses proxy!
        processPayment(order);
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void validateOrder(Order order) {
        // This @Transactional is IGNORED when called from placeOrder
        // because self-invocation bypasses the proxy
    }
}
```

🚨 **Common Mistake:** Calling `@Transactional` methods within the same class — Spring cannot intercept self-invocations through a proxy. The solution is to inject the bean's own proxy (`@Autowired private OrderService self`) or refactor into a separate class.

</details>

---

**Q21. How does Spring detect and instantiate beans, and what is the role of `BeanDefinition`?**

<details><summary>Click to reveal answer</summary>

Spring's bean creation pipeline:

1. **Scanning** — `ClassPathBeanDefinitionScanner` scans classpath for annotated classes
2. **Registration** — Creates `BeanDefinition` objects stored in `BeanDefinitionRegistry`
3. **BeanFactoryPostProcessor** — Modifies `BeanDefinition`s (e.g., `PropertySourcesPlaceholderConfigurer`)
4. **Instantiation** — Creates bean instances from `BeanDefinition`
5. **BeanPostProcessor** — Post-processes instances

```java
// BeanDefinition contains all metadata about a bean:
BeanDefinition bd = new RootBeanDefinition(MyService.class);
bd.setScope(BeanDefinition.SCOPE_SINGLETON);
bd.setLazyInit(false);
bd.getPropertyValues().add("timeout", 30);
bd.setInitMethodName("init");
bd.setDestroyMethodName("cleanup");

// Register programmatically
GenericApplicationContext ctx = new GenericApplicationContext();
ctx.registerBeanDefinition("myService", bd);
ctx.refresh();
```

```java
// Programmatic bean registration in Spring Boot
@Configuration
public class DynamicBeanConfig {

    @Autowired
    private GenericApplicationContext context;

    @PostConstruct
    public void registerDynamicBeans() {
        for (String tenantId : getTenants()) {
            context.registerBeanDefinition(
                "tenantService-" + tenantId,
                BeanDefinitionBuilder
                    .genericBeanDefinition(TenantService.class)
                    .addConstructorArgValue(tenantId)
                    .setScope("singleton")
                    .getBeanDefinition()
            );
        }
    }
}
```

> 💡 **Interviewer Tip:** Understanding `BeanDefinition` separates candidates who know Spring's API surface from those who deeply understand its internals. It's used for frameworks built on top of Spring (like Spring Data's repository factory).

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| Default scope | `singleton` — one per `ApplicationContext` |
| Prototype | New instance per request, no destruction callback |
| `@PostConstruct` | After DI, before use — prefer over `InitializingBean` |
| `@PreDestroy` | Before destruction — not called for prototype beans |
| Proxy modes | CGLIB for classes, JDK for interfaces |
| Self-invocation | Bypasses proxy → `@Transactional` / AOP ignored |
| Scoped proxies | Required when injecting short-lived into long-lived beans |
| Thread safety | Spring singleton ≠ thread-safe; use stateless design |
