# Spring Framework & Hibernate Introduction

> 💡 **Interviewer Tip:** Spring questions appear in nearly every Java backend interview. Interviewers want to see you understand the *why* — why IoC, why AOP, why JPA — not just the annotations. Connect every concept to the problem it solves.

---

## Table of Contents
1. [IoC Container & Dependency Injection](#ioc-container--dependency-injection)
2. [Spring Beans](#spring-beans)
3. [Spring MVC Request Lifecycle](#spring-mvc-request-lifecycle)
4. [Hibernate & ORM](#hibernate--orm)
5. [Entity Relationships](#entity-relationships)
6. [JPA vs Hibernate](#jpa-vs-hibernate)
7. [HQL & JPQL Basics](#hql--jpql-basics)

---

## IoC Container & Dependency Injection

### 🟢 Q1. What is Inversion of Control (IoC)?

<details><summary>Click to reveal answer</summary>

**Inversion of Control** is a design principle where the control of object creation and lifecycle is transferred from application code to a framework/container.

**Traditional approach (you control dependencies):**
```java
// You create and manage all objects
public class OrderService {
    private UserRepository userRepository;
    private EmailService emailService;

    public OrderService() {
        // You create dependencies — tightly coupled!
        this.userRepository = new UserRepositoryImpl(new DatabaseConfig());
        this.emailService = new SmtpEmailService(new SmtpConfig("smtp.gmail.com", 587));
    }
}
```

**With IoC (Spring controls dependencies):**
```java
// Spring creates and injects dependencies
@Service
public class OrderService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    // Spring provides these — you don't create them
    public OrderService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

**Benefits of IoC:**
- **Loose coupling** — `OrderService` doesn't know which implementation it gets
- **Testability** — Easy to inject mock dependencies in tests
- **Configurability** — Swap implementations without changing business logic
- **Lifecycle management** — Spring handles creation, initialization, destruction

> 💡 **Interviewer Tip:** "Don't call us, we'll call you" — The Hollywood Principle. Your code doesn't create/find its dependencies; the container provides them.

</details>

---

### 🟢 Q2. What are the three types of Dependency Injection in Spring?

<details><summary>Click to reveal answer</summary>

**1. Constructor Injection (Recommended)**
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final EmailService emailService;

    // Spring injects via constructor
    // @Autowired is optional when there's only one constructor (Spring 4.3+)
    public OrderService(OrderRepository orderRepository,
                        PaymentService paymentService,
                        EmailService emailService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.emailService = emailService;
    }
}
```

✅ **Why constructor injection is preferred:**
- Dependencies are **immutable** (final fields)
- **Fail-fast** — NPE impossible, missing deps caught at startup
- **Testable** — Create without Spring: `new OrderService(mockRepo, mockPayment, mockEmail)`
- Makes circular dependencies visible at compile time

---

**2. Setter Injection (For optional dependencies)**
```java
@Service
public class ReportService {
    private OrderRepository orderRepository;
    private AuditLogger auditLogger; // Optional

    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Autowired(required = false) // Optional dependency
    public void setAuditLogger(AuditLogger auditLogger) {
        this.auditLogger = auditLogger;
    }
}
```

---

**3. Field Injection (Convenient but problematic)**
```java
@Service
public class UserService {
    @Autowired // ❌ Not recommended for production code
    private UserRepository userRepository;

    @Autowired
    private EmailService emailService;
}
```

🚨 **Why field injection is problematic:**
- Dependencies are **mutable** (not final) — can be set to null accidentally
- **Hard to test** — requires Spring context or reflection to inject mocks
- **Hides coupling** — class may have too many dependencies without it being obvious
- Framework-dependent — can't use without `@Autowired`

```java
// Testing with field injection — awkward
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service; // Mockito uses reflection to inject

    // vs constructor injection — clean:
    // UserService service = new UserService(mockRepo, mockEmail);
}
```

</details>

---

### 🟡 Q3. What is the difference between `ApplicationContext` and `BeanFactory`?

<details><summary>Click to reveal answer</summary>

Both are IoC containers, but `ApplicationContext` is a superset of `BeanFactory`:

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Bean instantiation | Lazy (on first request) | Eager (at startup, by default) |
| Internationalization (i18n) | ❌ | ✅ MessageSource |
| Event publishing | ❌ | ✅ ApplicationEventPublisher |
| AOP integration | ❌ | ✅ Built-in |
| Environment abstraction | ❌ | ✅ PropertySources |
| `@Autowired` / `@Component` scanning | Limited | ✅ Full |
| ApplicationContextAware | ❌ | ✅ |

```java
// BeanFactory — lightweight, only basic DI
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));
// Rarely used directly in modern Spring

// ApplicationContext — full-featured Spring container
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

// SpringApplication.run() returns ApplicationContext (Spring Boot)
ConfigurableApplicationContext ctx = SpringApplication.run(MyApp.class, args);

// Getting beans programmatically (prefer injection over this)
OrderService orderService = ctx.getBean(OrderService.class);
OrderService byName = (OrderService) ctx.getBean("orderService");

// ApplicationContext features
ctx.getBean(MessageSource.class).getMessage("welcome.message", null, Locale.ENGLISH);
ctx.publishEvent(new OrderPlacedEvent(ctx, order));
ctx.getEnvironment().getProperty("spring.datasource.url");
```

**Common ApplicationContext implementations:**
- `AnnotationConfigApplicationContext` — Java config (`@Configuration`)
- `ClassPathXmlApplicationContext` — XML config (legacy)
- `WebApplicationContext` — Web applications (Spring MVC)
- `AnnotationConfigServletWebServerApplicationContext` — Spring Boot default

</details>

---

### 🟡 Q4. How does Spring resolve `@Autowired` when multiple beans of the same type exist?

<details><summary>Click to reveal answer</summary>

When multiple beans of the same type exist, Spring uses these resolution strategies (in order):

```java
// Multiple implementations of PaymentService
@Service("creditCardPayment")
public class CreditCardPaymentService implements PaymentService { }

@Service("paypalPayment")
public class PayPalPaymentService implements PaymentService { }

@Service
@Primary // Preferred when no qualifier specified
public class DefaultPaymentService implements PaymentService { }
```

**Resolution strategies:**

```java
// 1. @Primary — marks the preferred bean
@Service
@Primary
public class CreditCardPaymentService implements PaymentService { }

@Autowired
private PaymentService paymentService; // Gets CreditCardPaymentService

// 2. @Qualifier — explicit selection
@Autowired
@Qualifier("paypalPayment")
private PaymentService paymentService; // Gets PayPalPaymentService

// 3. Field name matching — if field name matches bean name
@Autowired
private PaymentService creditCardPayment; // Gets CreditCardPaymentService (name match)

// 4. Inject all implementations as a list
@Autowired
private List<PaymentService> allPaymentServices; // All 3!

// 5. Inject as a Map (beanName → bean)
@Autowired
private Map<String, PaymentService> paymentServices;
// {"creditCardPayment": ..., "paypalPayment": ..., "defaultPaymentService": ...}

// 6. Custom qualifier annotation
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface PayPal { }

@Service @PayPal
public class PayPalPaymentService implements PaymentService { }

@Autowired @PayPal
private PaymentService paymentService;
```

</details>

---

## Spring Beans

### 🟢 Q5. What are Spring Bean scopes?

<details><summary>Click to reveal answer</summary>

Bean scope defines the lifecycle and number of bean instances Spring creates:

| Scope | Instances | Lifecycle | Use Case |
|---|---|---|---|
| `singleton` | 1 per container (default) | Container | Stateless services, repositories |
| `prototype` | New each `getBean()` | You manage | Stateful beans, command objects |
| `request` | 1 per HTTP request | Request | Request-specific data (web only) |
| `session` | 1 per HTTP session | Session | User session data (web only) |
| `application` | 1 per ServletContext | Application | App-wide state (web only) |
| `websocket` | 1 per WebSocket session | WebSocket | WebSocket-specific (web only) |

```java
// Singleton (default) — one instance shared everywhere
@Service // @Scope("singleton") is default
public class UserService {
    // Must be stateless — shared by all threads!
}

// Prototype — new instance each time
@Component
@Scope("prototype")
public class ReportBuilder {
    private List<String> rows = new ArrayList<>(); // Stateful — OK for prototype

    public void addRow(String row) { rows.add(row); }
    public Report build() { return new Report(rows); }
}

// Inject prototype into singleton — requires scoped proxy!
@Service
public class ReportService {

    // ❌ WRONG: Spring injects prototype once, always gets same instance
    @Autowired
    private ReportBuilder reportBuilder;

    // ✅ OPTION 1: Scoped proxy
    @Autowired
    @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
    private ReportBuilder reportBuilder; // Always gets new instance

    // ✅ OPTION 2: ApplicationContext.getBean()
    @Autowired
    private ApplicationContext ctx;

    public Report generateReport(ReportRequest req) {
        ReportBuilder builder = ctx.getBean(ReportBuilder.class); // New instance
        req.getItems().forEach(builder::addRow);
        return builder.build();
    }

    // ✅ OPTION 3: Provider<T> (Spring/JSR-330)
    @Autowired
    private ObjectProvider<ReportBuilder> reportBuilderProvider;

    public Report generate() {
        ReportBuilder builder = reportBuilderProvider.getObject(); // New each time
        return builder.build();
    }
}

// Request scope (web applications)
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String requestId = UUID.randomUUID().toString();
    public String getRequestId() { return requestId; }
}
```

</details>

---

### 🟡 Q6. Explain the Spring Bean lifecycle.

<details><summary>Click to reveal answer</summary>

Spring beans go through a well-defined lifecycle managed by the container:

```
1. Instantiation    → Spring creates bean instance
2. Property population → @Autowired dependencies injected
3. BeanNameAware    → setBeanName() called
4. BeanFactoryAware → setBeanFactory() called
5. ApplicationContextAware → setApplicationContext() called
6. @PostConstruct   → Init method called
7. InitializingBean → afterPropertiesSet() called
8. @Bean(initMethod) → Custom init method called
────────────── BEAN IS READY FOR USE ──────────────
9. @PreDestroy      → Cleanup method called
10. DisposableBean  → destroy() called
11. @Bean(destroyMethod) → Custom destroy method called
```

```java
@Component
public class DatabaseConnectionPool
        implements InitializingBean, DisposableBean,
                   BeanNameAware, ApplicationContextAware {

    private DataSource dataSource;
    private String beanName;

    // Step 3: BeanNameAware
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("Bean name: " + name);
    }

    // Step 5: ApplicationContextAware
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("Context set: " + ctx.getDisplayName());
    }

    // Step 6: @PostConstruct (preferred — no Spring interface dependency)
    @PostConstruct
    public void init() {
        System.out.println("@PostConstruct: Initializing pool");
        dataSource = createPool();
    }

    // Step 7: InitializingBean (Spring-specific)
    @Override
    public void afterPropertiesSet() {
        System.out.println("afterPropertiesSet: Post-init checks");
        validatePool();
    }

    // Step 9: @PreDestroy (preferred)
    @PreDestroy
    public void cleanup() {
        System.out.println("@PreDestroy: Releasing connections");
        dataSource.close();
    }

    // Step 10: DisposableBean
    @Override
    public void destroy() {
        System.out.println("destroy: Final cleanup");
    }
}

// @Bean with init/destroy methods (for third-party classes you can't annotate)
@Configuration
public class AppConfig {

    @Bean(initMethod = "start", destroyMethod = "stop")
    public SomeThirdPartyService thirdPartyService() {
        return new SomeThirdPartyService();
    }
}
```

</details>

---

### 🟢 Q7. What is `@Configuration` and how does it differ from `@Component`?

<details><summary>Click to reveal answer</summary>

Both mark a class as a Spring-managed bean source, but with a key difference:

```java
// @Configuration: Full proxy — @Bean methods intercepted by CGLIB
@Configuration
public class AppConfig {

    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl(dataSource());
    }

    @Bean
    public OrderService orderService() {
        // Calling userRepository() again — returns SAME bean (singleton)!
        // CGLIB proxy intercepts the call
        return new OrderService(userRepository());
    }

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

// @Component with @Bean: Lite mode — NO CGLIB proxy
@Component
public class AppConfig2 {

    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl(dataSource());
    }

    @Bean
    public OrderService orderService() {
        // ❌ Calls userRepository() as a regular Java method — creates NEW instance!
        // Not singleton behavior!
        return new OrderService(userRepository());
    }
}
```

✅ **Best Practice:** Use `@Configuration` for bean factory classes (not `@Component`). The CGLIB proxy ensures singleton behavior when `@Bean` methods call each other.

**Key annotations:**
```java
@Configuration   // Defines beans — proxy mode
@Component       // Marks class as component — auto-detected by scanning
@Service         // @Component + semantic hint (service layer)
@Repository      // @Component + enables PersistenceExceptionTranslation
@Controller      // @Component + marks MVC controller
@RestController  // @Controller + @ResponseBody on all methods
```

</details>

---

## Spring MVC Request Lifecycle

### 🟡 Q8. Describe the Spring MVC request lifecycle step by step.

<details><summary>Click to reveal answer</summary>

```
HTTP Request
     ↓
1. DispatcherServlet (Front Controller)
     ↓
2. HandlerMapping (find which controller handles this URL)
     ↓
3. HandlerAdapter (execute the handler — controller method)
     ↓ ←→ HandlerInterceptor.preHandle()
4. Controller method executes
     ↓ → HandlerInterceptor.postHandle()
5. ViewResolver (resolve logical view name → actual view)
     ↓
6. View renders (JSP, Thymeleaf, JSON via HttpMessageConverter)
     ↓ → HandlerInterceptor.afterCompletion()
7. HTTP Response sent to client
```

```java
// 1. DispatcherServlet configured in web.xml or auto-configured by Spring Boot
// Spring Boot auto-configures DispatcherServlet on "/"

// 2. HandlerMapping — maps URL to controller
// RequestMappingHandlerMapping reads @RequestMapping annotations

// 3. @RestController — handled by RequestMappingHandlerAdapter
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired private OrderService orderService;

    // 4. Controller method
    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(
            @PathVariable Long id,
            @RequestHeader("Authorization") String auth,
            @RequestParam(defaultValue = "false") boolean includeItems) {

        Order order = orderService.findById(id);
        OrderDto dto = includeItems ? OrderDto.withItems(order) : OrderDto.basic(order);
        return ResponseEntity.ok(dto);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderDto createOrder(@Valid @RequestBody CreateOrderRequest request,
                                 BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            throw new ValidationException(bindingResult);
        }
        return OrderDto.from(orderService.create(request));
    }
}

// 5. ViewResolver — for REST, Jackson's MappingJackson2HttpMessageConverter
//    serializes return value to JSON (HttpMessageConverter, not ViewResolver)

// HandlerInterceptor — cross-cutting concerns
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse resp,
                              Object handler) {
        req.setAttribute("startTime", System.currentTimeMillis());
        log.info("Request: {} {}", req.getMethod(), req.getRequestURI());
        return true; // Continue processing
    }

    @Override
    public void postHandle(HttpServletRequest req, HttpServletResponse resp,
                            Object handler, ModelAndView mav) {
        // Called after controller, before view rendering
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse resp,
                                 Object handler, Exception ex) {
        long duration = System.currentTimeMillis() - (Long)req.getAttribute("startTime");
        log.info("Completed in {}ms, status: {}", duration, resp.getStatus());
    }
}

// Register interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new RequestLoggingInterceptor())
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/health");
    }
}
```

</details>

---

### 🟡 Q9. What is `@RequestMapping` and its shortcuts?

<details><summary>Click to reveal answer</summary>

`@RequestMapping` maps HTTP requests to controller methods. Shortcut annotations provide more readable alternatives:

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    // Full @RequestMapping
    @RequestMapping(method = RequestMethod.GET, produces = "application/json")
    public List<Product> getAll() { return productService.findAll(); }

    // Shortcut annotations (same as above)
    @GetMapping                                     // GET /api/v1/products
    @GetMapping("/{id}")                            // GET /api/v1/products/{id}
    @PostMapping                                    // POST /api/v1/products
    @PutMapping("/{id}")                            // PUT /api/v1/products/{id}
    @PatchMapping("/{id}")                          // PATCH /api/v1/products/{id}
    @DeleteMapping("/{id}")                         // DELETE /api/v1/products/{id}

    // Full example
    @GetMapping(value = "/{id}",
                produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
    public ResponseEntity<ProductDto> getProduct(
            @PathVariable Long id,
            @RequestParam(value = "currency", defaultValue = "USD") String currency,
            @RequestHeader(value = "Accept-Language", defaultValue = "en") String lang) {

        Product product = productService.findById(id);
        ProductDto dto = productMapper.toDto(product, currency, lang);
        return ResponseEntity.ok()
            .header("Cache-Control", "max-age=300")
            .body(dto);
    }

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(HttpStatus.CREATED)
    public ProductDto createProduct(@Valid @RequestBody CreateProductRequest request,
                                     UriComponentsBuilder ucb) {
        Product product = productService.create(request);
        URI location = ucb.path("/api/v1/products/{id}").buildAndExpand(product.getId()).toUri();
        // Include Location header as per REST convention
        return productMapper.toDto(product);
    }
}
```

**Parameter binding annotations:**
```java
@GetMapping("/search")
public List<Product> search(
    @PathVariable Long categoryId,          // /categories/{categoryId}/products
    @RequestParam String query,             // ?query=laptop
    @RequestParam(required = false, defaultValue = "10") int limit,
    @RequestHeader("X-Client-Id") String clientId,
    @CookieValue("sessionId") String sessionId,
    @RequestBody ProductSearchRequest body,
    @ModelAttribute ProductFilter filter,   // Form data / query params → object
    @AuthenticationPrincipal UserDetails user
) { ... }
```

</details>

---

### 🟡 Q10. What is `@ControllerAdvice` and `@ExceptionHandler`?

<details><summary>Click to reveal answer</summary>

`@ControllerAdvice` is a cross-cutting concern that applies to all (or selected) controllers. `@ExceptionHandler` handles specific exceptions globally.

```java
@RestControllerAdvice  // @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "Invalid value",
                (e1, e2) -> e1
            ));
        return new ErrorResponse("Validation failed", errors);
    }

    // Handle not found
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex) {
        return new ErrorResponse(ex.getMessage());
    }

    // Handle access denied
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex) {
        return new ErrorResponse("Access denied");
    }

    // Handle all other exceptions
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex, HttpServletRequest request) {
        String requestId = UUID.randomUUID().toString();
        log.error("Unhandled exception [requestId={}] for {} {}",
            requestId, request.getMethod(), request.getRequestURI(), ex);
        return new ErrorResponse("Internal server error. RequestId: " + requestId);
    }

    // @InitBinder — configure data binding for all controllers
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, true));
    }

    // @ModelAttribute — add data to model for all controllers
    @ModelAttribute("appVersion")
    public String appVersion() {
        return "1.2.3";
    }
}

// Error response DTO
public record ErrorResponse(String message, Map<String, String> fieldErrors) {
    public ErrorResponse(String message) { this(message, Map.of()); }
}
```

</details>

---

## Hibernate & ORM

### 🟢 Q11. What is ORM and what problem does Hibernate solve?

<details><summary>Click to reveal answer</summary>

**ORM (Object-Relational Mapping)** bridges the gap between Java objects and relational database tables — the **impedance mismatch** problem.

**Without ORM (JDBC):**
```java
// Manual mapping — verbose, error-prone
String sql = "SELECT id, name, email, age, created_at FROM users WHERE id = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setLong(1, id);
ResultSet rs = ps.executeQuery();
if (rs.next()) {
    user = new User();
    user.setId(rs.getLong("id"));
    user.setName(rs.getString("name"));
    user.setEmail(rs.getString("email"));
    user.setAge(rs.getInt("age"));
    user.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
}
// Manual for every entity — 50 entities = 50 result set mappings!
```

**With Hibernate:**
```java
// Automatic mapping — just annotate your class
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    private int age;

    @Column(name = "created_at")
    private LocalDateTime createdAt;
}

// Query — Hibernate generates SQL automatically
User user = session.get(User.class, id);
// SELECT u.id, u.name, u.email, u.age, u.created_at FROM users u WHERE u.id = ?
```

**Hibernate benefits:**
- Automatic SQL generation
- Database portability (switch MySQL to PostgreSQL with config change)
- Caching (first-level, second-level)
- Lazy loading of associations
- Transaction management integration

</details>

---

### 🟢 Q12. Explain the core JPA annotations: `@Entity`, `@Table`, `@Column`, `@Id`.

<details><summary>Click to reveal answer</summary>

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity  // Marks class as JPA entity (must have no-arg constructor)
@Table(
    name = "products",           // Maps to "products" table
    schema = "inventory",        // Optional: specific schema
    indexes = {
        @Index(name = "idx_products_sku", columnList = "sku", unique = true),
        @Index(name = "idx_products_category", columnList = "category_id")
    }
)
public class Product {

    @Id  // Primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY) // AUTO_INCREMENT
    // Other strategies:
    // GenerationType.SEQUENCE — uses DB sequence (PostgreSQL default)
    // GenerationType.UUID — generates UUID (Java/DB)
    // GenerationType.TABLE — portable but slow (avoids — deprecated in Jakarta Persistence 3.2)
    private Long id;

    @Column(
        name = "product_name",  // Maps to "product_name" column (default: field name)
        nullable = false,       // NOT NULL constraint
        length = 255,           // VARCHAR(255)
        unique = false
    )
    private String name;

    @Column(
        precision = 10,  // Total digits
        scale = 2        // Decimal places: DECIMAL(10, 2)
    )
    private BigDecimal price;

    @Column(name = "sku", nullable = false, unique = true, length = 50)
    private String sku;

    @Column(name = "description", columnDefinition = "TEXT") // Custom column definition
    private String description;

    @Column(name = "in_stock", nullable = false)
    private boolean inStock = true;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // Lifecycle callbacks
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = createdAt;
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // Transient — not persisted
    @Transient
    private String temporaryDisplayName;

    // Enum mapping
    @Enumerated(EnumType.STRING) // Stores "ACTIVE", not 0
    @Column(nullable = false)
    private ProductStatus status = ProductStatus.ACTIVE;

    // LOB types
    @Lob
    @Column(name = "thumbnail")
    private byte[] thumbnail; // Binary large object

    // Version for optimistic locking
    @Version
    private Long version;

    // Required no-arg constructor for JPA
    protected Product() {}

    public Product(String name, BigDecimal price, String sku) {
        this.name = name;
        this.price = price;
        this.sku = sku;
    }

    // Getters...
}
```

</details>

---

### 🟡 Q13. What is the difference between `@GeneratedValue` strategies?

<details><summary>Click to reveal answer</summary>

```java
// IDENTITY: Uses auto-increment column — no pre-generation
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
// Pros: Simple, widely supported
// Cons: Hibernate can't batch inserts (needs DB to return ID after each insert)

// SEQUENCE: Uses DB sequences — default for PostgreSQL
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE,
                generator = "product_seq")
@SequenceGenerator(
    name = "product_seq",
    sequenceName = "products_id_seq",
    allocationSize = 50  // Fetch 50 IDs at once (batch-friendly)
)
private Long id;
// Pros: Batch inserts, pre-generation, better performance
// Cons: PostgreSQL/Oracle specific (MySQL uses IDENTITY)

// AUTO: Let Hibernate choose (SEQUENCE for PostgreSQL, IDENTITY for MySQL)
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;

// UUID: Generate universally unique ID
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id; // java.util.UUID

// Or with custom generator
@Id
@Column(name = "id", updatable = false, nullable = false)
private String id = UUID.randomUUID().toString(); // Self-assign

// Composite key
@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
}

@Embeddable
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals, hashCode required!
}
```

</details>

---

## Entity Relationships

### 🟡 Q14. Explain `@OneToMany` and `@ManyToOne` with a full example.

<details><summary>Click to reveal answer</summary>

```java
// ONE Order has MANY OrderItems
// MANY OrderItems belong to ONE Order

@Entity
@Table(name = "orders")
public class Order {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerEmail;
    private LocalDateTime placedAt;

    // ONE order → MANY items
    // mappedBy = field in the owning side (OrderItem.order)
    // CascadeType.ALL: operations on Order cascade to items (save, delete, etc.)
    // orphanRemoval = true: removing item from collection deletes it from DB
    @OneToMany(
        mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY // Default for collections — load only when accessed
    )
    private List<OrderItem> items = new ArrayList<>();

    // Helper method to maintain bidirectional consistency
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this); // Set back-reference!
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Entity
@Table(name = "order_items")
public class OrderItem {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // MANY items → ONE order (OWNING side — has the foreign key column)
    @ManyToOne(fetch = FetchType.LAZY) // Default for @ManyToOne is EAGER — override!
    @JoinColumn(
        name = "order_id",        // FK column in order_items table
        nullable = false,
        foreignKey = @ForeignKey(name = "fk_order_items_order")
    )
    private Order order;

    @Column(nullable = false)
    private String productName;

    @Column(nullable = false)
    private int quantity;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;

    // Transient computed field
    @Transient
    public BigDecimal getTotalPrice() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}

// Usage
Order order = new Order();
order.setCustomerEmail("alice@example.com");
order.setPlacedAt(LocalDateTime.now());

OrderItem item1 = new OrderItem("Java Book", 2, new BigDecimal("29.99"));
OrderItem item2 = new OrderItem("Spring Sticker", 5, new BigDecimal("2.99"));

order.addItem(item1); // Maintains bidirectionality
order.addItem(item2);

orderRepository.save(order); // Cascades to save items too
```

🚨 **Common Mistake:** `FetchType.EAGER` on `@ManyToOne` is the default — but cascades through associations can cause N+1 problems. Always prefer `LAZY` and use JOIN FETCH in JPQL when needed.

</details>

---

### 🟡 Q15. How does `@ManyToMany` work and what are its pitfalls?

<details><summary>Click to reveal answer</summary>

```java
// Students and Courses: Many students can enroll in many courses

@Entity
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    // CascadeType.ALL would delete courses when student is deleted — wrong!
    @JoinTable(
        name = "student_courses",          // Junction table
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>(); // Set to avoid duplicates

    public void enrollIn(Course course) {
        courses.add(course);
        course.getStudents().add(this); // Maintain bidirectionality
    }

    public void unenroll(Course course) {
        courses.remove(course);
        course.getStudents().remove(this);
    }
}

@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses") // Non-owning side
    private Set<Student> students = new HashSet<>();
}
```

**Pitfall — extra data in join table:**
```java
// If you need extra columns in the join table (e.g., enrollmentDate, grade)
// → Convert @ManyToMany to two @OneToMany relationships with an intermediate entity

@Entity
@Table(name = "enrollment")
public class Enrollment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne
    @JoinColumn(name = "course_id")
    private Course course;

    // Extra columns
    private LocalDate enrollmentDate;
    private String grade;
}
```

🚨 **Common Mistake:** Using `@ManyToMany` with `CascadeType.REMOVE` or `CascadeType.ALL` — deleting one side deletes all associations AND the other side!

</details>

---

### 🟡 Q16. What is the N+1 query problem and how do you solve it?

<details><summary>Click to reveal answer</summary>

**N+1 problem:** Loading N entities, then executing 1 additional query per entity to load a related association. Results in N+1 total queries.

```java
// Entity setup
@Entity
public class Author {
    @Id Long id;
    String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    List<Book> books;
}

// ❌ N+1 PROBLEM:
List<Author> authors = em.createQuery("SELECT a FROM Author a", Author.class).getResultList();
// Query 1: SELECT * FROM authors (returns 100 authors)

for (Author author : authors) {
    System.out.println(author.getBooks().size()); // Triggers lazy load!
    // Query 2: SELECT * FROM books WHERE author_id = 1
    // Query 3: SELECT * FROM books WHERE author_id = 2
    // ...
    // Query 101: SELECT * FROM books WHERE author_id = 100
    // TOTAL: 101 queries!
}
```

**Solutions:**

```java
// ✅ Solution 1: JOIN FETCH (JPQL)
List<Author> authors = em.createQuery(
    "SELECT DISTINCT a FROM Author a JOIN FETCH a.books", Author.class)
    .getResultList();
// Single query: SELECT a.*, b.* FROM authors a JOIN books b ON b.author_id = a.id
// TOTAL: 1 query!

// ✅ Solution 2: @EntityGraph
@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = "books")
    List<Author> findAll(); // Generates JOIN FETCH

    @EntityGraph(attributePaths = {"books", "books.reviews"}) // Multiple levels
    Optional<Author> findById(Long id);
}

// ✅ Solution 3: @BatchSize (reduces N+1 to N/batch queries)
@Entity
public class Author {
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    @BatchSize(size = 25) // Load 25 authors' books in one query
    List<Book> books;
}

// ✅ Solution 4: DTO projections (when you don't need full entities)
public interface AuthorSummary {
    String getName();
    long getBookCount();
}

@Query("SELECT a.name AS name, COUNT(b) AS bookCount " +
       "FROM Author a LEFT JOIN a.books b GROUP BY a.name")
List<AuthorSummary> findAuthorSummaries();
```

> 💡 **Interviewer Tip:** Detecting N+1 in production: Use Hibernate statistics (`spring.jpa.properties.hibernate.generate_statistics=true`) or `spring-boot-starter-actuator` with DataSource proxy.

</details>

---

## JPA vs Hibernate

### 🟢 Q17. What is the difference between JPA and Hibernate?

<details><summary>Click to reveal answer</summary>

| Aspect | JPA | Hibernate |
|---|---|---|
| What it is | Specification (API/interface) | Implementation (provider) |
| Package | `jakarta.persistence.*` | `org.hibernate.*` |
| Standardized | Yes (Jakarta EE standard) | No (proprietary) |
| Portability | Can switch providers | Tied to Hibernate |
| Extra features | Basic ORM features | Hibernate-specific extensions |

```java
// JPA standard — portable across providers
import jakarta.persistence.*;

EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();
User user = em.find(User.class, 1L);         // JPA API
em.persist(new User("Alice"));               // JPA API
em.getTransaction().commit();

// Hibernate-specific (not portable)
import org.hibernate.*;
import org.hibernate.cfg.Configuration;

SessionFactory factory = new Configuration().configure().buildSessionFactory();
Session session = factory.openSession();
session.beginTransaction();
User user = session.get(User.class, 1L);     // Hibernate API (similar to JPA)
session.save(new User("Alice"));              // Hibernate API
session.getTransaction().commit();

// Hibernate extras not in JPA:
// @Cache (second-level cache with @NaturalId)
// @Filter (dynamic query filters)
// @Loader (custom SQL for load)
// StatelessSession (bulk operations without 1st-level cache)
// MultiTenancy support
```

**Spring Data JPA:** Yet another layer that wraps JPA:
```java
// Spring Data JPA = Spring + JPA (Hibernate under the hood)
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring Data generates: SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // Custom JPQL
    @Query("SELECT u FROM User u WHERE u.role = :role AND u.active = true")
    List<User> findActiveByRole(@Param("role") String role);

    // Native SQL
    @Query(value = "SELECT * FROM users WHERE YEAR(created_at) = :year",
           nativeQuery = true)
    List<User> findByCreationYear(@Param("year") int year);
}
```

</details>

---

### 🟡 Q18. What is the Hibernate first-level and second-level cache?

<details><summary>Click to reveal answer</summary>

**First-Level Cache (Session/EntityManager scope):**
- Automatic, always enabled
- Scoped to a single Session/EntityManager
- Ensures same object identity within a transaction

```java
EntityManager em = factory.createEntityManager();
em.getTransaction().begin();

User user1 = em.find(User.class, 1L); // SQL: SELECT FROM users WHERE id=1
User user2 = em.find(User.class, 1L); // NO SQL — returned from 1st-level cache!

assertSame(user1, user2); // true — same instance

em.getTransaction().commit();
em.close(); // Cache cleared!

// New EntityManager — cache is gone
EntityManager em2 = factory.createEntityManager();
User user3 = em2.find(User.class, 1L); // SQL executed again
```

**Second-Level Cache (SessionFactory/EntityManagerFactory scope):**
- Opt-in, shared across sessions
- Survives session close
- Requires external provider (Ehcache, Caffeine, Redis)

```java
// application.yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax.cache.provider: org.ehcache.jsr107.EhcacheCachingProvider

// Enable caching on entity
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Hibernate annotation
public class Category {
    @Id Long id;
    String name;

    @OneToMany
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // Also cache collections
    private List<Product> products;
}

// Cache strategies:
// READ_ONLY: Best performance, no updates after creation
// NONSTRICT_READ_WRITE: Updates occasionally
// READ_WRITE: Updates require locking
// TRANSACTIONAL: Full transaction support (JTA)

// Spring's @Cacheable (application-level cache, simpler):
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }
}
```

</details>

---

### 🟡 Q19. What is optimistic vs pessimistic locking in JPA?

<details><summary>Click to reveal answer</summary>

**Optimistic Locking:** Assumes conflicts are rare. Detects conflicts at commit time.
**Pessimistic Locking:** Assumes conflicts are frequent. Locks row for duration of transaction.

```java
// OPTIMISTIC LOCKING — @Version
@Entity
public class Product {
    @Id Long id;
    String name;
    int stock;

    @Version  // Auto-managed version field
    private Long version;
}

// How it works:
// Thread 1: SELECT id, name, stock, version FROM products WHERE id=1
//           → stock=10, version=5
// Thread 2: SELECT id, name, stock, version FROM products WHERE id=1
//           → stock=10, version=5
// Thread 1: UPDATE products SET stock=9, version=6 WHERE id=1 AND version=5 → 1 row updated ✅
// Thread 2: UPDATE products SET stock=9, version=6 WHERE id=1 AND version=5 → 0 rows! → OptimisticLockException!

// Handling OptimisticLockException:
public void decrementStock(Long productId) {
    int retries = 3;
    while (retries-- > 0) {
        try {
            Product p = em.find(Product.class, productId);
            p.decrementStock();
            em.flush();
            return;
        } catch (OptimisticLockException e) {
            em.clear(); // Clear stale state
            if (retries == 0) throw new ConcurrentModificationException(e);
        }
    }
}

// PESSIMISTIC LOCKING — locks row in DB
public Product findAndLock(Long id) {
    return em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE);
    // Equivalent to: SELECT ... FROM products WHERE id=? FOR UPDATE
    // Other transactions blocked until this transaction commits
}

// JPQL with lock
Product p = em.createQuery(
    "SELECT p FROM Product p WHERE p.id = :id", Product.class)
    .setParameter("id", id)
    .setLockMode(LockModeType.PESSIMISTIC_WRITE)
    .getSingleResult();
```

> 💡 **Interviewer Tip:** Optimistic locking is preferred for high-read, low-conflict scenarios. Pessimistic locking is better when conflicts are likely (e.g., ticket booking, inventory reservation).

</details>

---

## HQL & JPQL Basics

### 🟡 Q20. What is JPQL and how does it differ from SQL?

<details><summary>Click to reveal answer</summary>

**JPQL (Java Persistence Query Language)** is an object-oriented query language that operates on entity objects, not database tables. Hibernate's HQL is nearly identical.

| Feature | SQL | JPQL/HQL |
|---|---|---|
| Operates on | Tables, columns | Entities, fields |
| Case sensitive | No (usually) | Class names yes, keywords no |
| Joins | Table JOINs | Association traversal |
| Result | ResultSet (rows) | Entity objects or projections |
| Portable | Not portable | Portable across JPA providers |

```java
// BASIC QUERIES

// SQL:  SELECT * FROM users WHERE email = 'alice@example.com'
// JPQL: (operates on User entity, not "users" table)
List<User> users = em.createQuery(
    "SELECT u FROM User u WHERE u.email = :email", User.class)
    .setParameter("email", "alice@example.com")
    .getResultList();

// JOIN FETCH (resolve N+1)
// SQL: SELECT u.*, o.* FROM users u JOIN orders o ON o.user_id = u.id WHERE u.active = true
List<User> users2 = em.createQuery(
    "SELECT DISTINCT u FROM User u JOIN FETCH u.orders WHERE u.active = true", User.class)
    .getResultList();

// AGGREGATE FUNCTIONS
Long count = em.createQuery(
    "SELECT COUNT(u) FROM User u WHERE u.role = :role", Long.class)
    .setParameter("role", "ADMIN")
    .getSingleResult();

// PROJECTION (select specific fields — avoids loading full entity)
List<Object[]> results = em.createQuery(
    "SELECT u.name, u.email FROM User u WHERE u.active = true")
    .getResultList();

// DTO projection (type-safe)
List<UserSummary> summaries = em.createQuery(
    "SELECT new com.example.dto.UserSummary(u.id, u.name, u.email) " +
    "FROM User u WHERE u.active = true", UserSummary.class)
    .getResultList();

// SUBQUERY
List<Order> bigOrders = em.createQuery(
    "SELECT o FROM Order o WHERE o.total > " +
    "(SELECT AVG(o2.total) FROM Order o2)", Order.class)
    .getResultList();

// UPDATE / DELETE (bulk operations — bypass 1st-level cache)
int updated = em.createQuery(
    "UPDATE Product p SET p.status = 'DISCONTINUED' WHERE p.stock = 0")
    .executeUpdate();

// NAMED QUERY (compiled at startup — slight performance benefit)
@Entity
@NamedQuery(
    name = "User.findByRole",
    query = "SELECT u FROM User u WHERE u.role = :role ORDER BY u.name"
)
public class User { ... }

List<User> admins = em.createNamedQuery("User.findByRole", User.class)
    .setParameter("role", "ADMIN")
    .getResultList();
```

</details>

---

### 🟡 Q21. What is Spring Data JPA's query derivation?

<details><summary>Click to reveal answer</summary>

Spring Data JPA automatically generates queries from repository method names:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // findBy + Property: WHERE email = ?
    Optional<User> findByEmail(String email);

    // findBy + Property + And + Property
    List<User> findByActiveAndRole(boolean active, String role);

    // findBy + Property + Or + Property
    List<User> findByNameOrEmail(String name, String email);

    // Greater than, less than
    List<User> findByAgeGreaterThan(int age);
    List<User> findByAgeBetween(int min, int max);

    // Like, StartingWith, EndingWith, Containing
    List<User> findByNameContainingIgnoreCase(String keyword);
    List<User> findByEmailEndingWith(String domain);

    // IsNull, IsNotNull
    List<User> findByDeletedAtIsNull();
    List<User> findByAvatarUrlIsNotNull();

    // OrderBy
    List<User> findByActiveOrderByNameAsc(boolean active);
    List<User> findByRoleOrderByCreatedAtDesc(String role);

    // First, Top (limit results)
    Optional<User> findFirstByRoleOrderByCreatedAtDesc(String role);
    List<User> findTop10ByActiveOrderByCreatedAtDesc(boolean active);

    // Count, Exists, Delete
    long countByRole(String role);
    boolean existsByEmail(String email);
    void deleteByDeletedAtBefore(LocalDateTime cutoff);

    // Nested property (join)
    List<User> findByDepartmentName(String deptName);
    // Generates: ... JOIN department d WHERE d.name = ?

    // With Pageable
    Page<User> findByRole(String role, Pageable pageable);
    Slice<User> findByActive(boolean active, Pageable pageable);

    // With @Modifying for update/delete
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.lastLoginAt = :now WHERE u.id = :id")
    int updateLastLogin(@Param("id") Long id, @Param("now") LocalDateTime now);
}

// Using the repository
@Service
public class UserService {
    @Autowired UserRepository userRepository;

    public Page<User> getActiveAdmins(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return userRepository.findByRoleAndActive("ADMIN", true, pageable);
    }
}
```

</details>

---

### 🔴 Q22. How do you handle transactions in Spring?

<details><summary>Click to reveal answer</summary>

Spring manages transactions declaratively with `@Transactional` via AOP proxy.

```java
@Service
@Transactional // Default: all public methods are transactional
public class OrderService {

    @Autowired private OrderRepository orderRepository;
    @Autowired private InventoryService inventoryService;
    @Autowired private PaymentService paymentService;

    // Method-level override
    @Transactional(
        propagation = Propagation.REQUIRED,         // Join existing or create new
        isolation = Isolation.READ_COMMITTED,        // Prevent dirty reads
        timeout = 30,                               // 30 second timeout
        rollbackFor = {PaymentException.class},     // Rollback on checked exception
        noRollbackFor = {AuditException.class},     // Don't rollback on this
        readOnly = false                            // Allows write operations
    )
    public Order placeOrder(OrderRequest request) {
        inventoryService.reserve(request.getItems()); // In same transaction
        Payment payment = paymentService.charge(request.getPaymentToken(), request.getTotal());
        return orderRepository.save(new Order(request, payment));
        // If any exception: entire transaction rolled back
    }

    @Transactional(readOnly = true) // Optimization: no dirty checking, read-only DB connection
    public List<Order> findUserOrders(Long userId) {
        return orderRepository.findByUserId(userId);
    }
}

// Propagation behaviors:
// REQUIRED (default): Join existing transaction or create new
// REQUIRES_NEW: Always create new transaction (suspend existing)
// SUPPORTS: Join if exists, run non-transactionally if not
// NOT_SUPPORTED: Always run non-transactionally (suspend existing)
// MANDATORY: Must run in existing transaction (throw if not)
// NEVER: Must NOT run in transaction (throw if exists)
// NESTED: Execute in nested transaction (savepoint)

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String event) {
        // Runs in its OWN transaction — always committed, even if outer tx rolls back
        auditRepository.save(new AuditLog(event, LocalDateTime.now()));
    }
}

// ❌ Common mistake: Self-invocation bypasses proxy!
@Service
public class ProductService {
    @Transactional
    public void updateAll(List<Product> products) {
        products.forEach(p -> this.updateOne(p)); // this.updateOne bypasses proxy!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateOne(Product product) { // Propagation ignored!
        productRepository.save(product);
    }
}

// ✅ Fix: inject self or use separate service
@Service
public class ProductService {
    @Autowired ProductService self; // Spring injects the proxy!

    @Transactional
    public void updateAll(List<Product> products) {
        products.forEach(p -> self.updateOne(p)); // Goes through proxy ✅
    }
}
```

</details>

---

### 🟡 Q23. What is Lazy vs Eager loading and when would you use each?

<details><summary>Click to reveal answer</summary>

```java
// LAZY: Load association only when accessed — DEFAULT for collections
@OneToMany(fetch = FetchType.LAZY) // Default — preferred
List<OrderItem> items; // Loaded only when items is accessed

// EAGER: Load association immediately with parent — DEFAULT for @ManyToOne
@ManyToOne(fetch = FetchType.EAGER) // Default for @ManyToOne — consider overriding
User user; // Always loaded with OrderItem, even when not needed

// ❌ EAGER problem: Always loads even when unnecessary
// Loading 1000 orders → loads 1000 users → loads 1000 addresses → ...

// ✅ Best practice: Default to LAZY, use JOIN FETCH when you need the data
@OneToMany(fetch = FetchType.LAZY) // All collections: LAZY
@ManyToOne(fetch = FetchType.LAZY) // Also override to LAZY
@ManyToMany(fetch = FetchType.LAZY) // All collections: LAZY

// Load with JOIN FETCH when you know you need the data:
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);

    @Query("SELECT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.userId = :userId")
    List<Order> findByUserIdWithDetails(@Param("userId") Long userId);
}

// LazyInitializationException: Accessing LAZY association outside Session
@Service
public class BadService {
    @Autowired OrderRepository orderRepo;

    public List<OrderItem> getItems(Long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow(); // Session closes!
        return order.getItems(); // ❌ LazyInitializationException — Session closed!
    }
}

// Fix: Use @Transactional to keep session open
@Transactional(readOnly = true)
public List<OrderItem> getItems(Long orderId) {
    Order order = orderRepo.findById(orderId).orElseThrow();
    return order.getItems(); // ✅ Session still open
}
```

</details>

---

### 🔴 Q24. How does Spring Boot auto-configure JPA/Hibernate?

<details><summary>Click to reveal answer</summary>

Spring Boot's auto-configuration reduces boilerplate:

```yaml
# application.yaml — Spring Boot automatically configures Hibernate
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    # HikariCP configured automatically
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2

  jpa:
    hibernate:
      ddl-auto: validate   # validate|update|create|create-drop|none
      # validate: Validates schema on startup — RECOMMENDED for production
      # update: Auto-updates schema — use ONLY for development
      # create: Creates schema on startup, drops on close — tests only
    show-sql: false         # Log SQL queries (true for debugging only)
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          batch_size: 50    # Batch inserts for better performance
          order_inserts: true
          order_updates: true
        cache:
          use_second_level_cache: false
    open-in-view: false     # IMPORTANT: disable OSIV to prevent lazy-loading surprises
```

```java
// What Spring Boot auto-configures:
// 1. DataSource (HikariCP)
// 2. EntityManagerFactory (backed by Hibernate)
// 3. JpaTransactionManager
// 4. Spring Data JPA repositories

// Disable auto-configuration to customize
@SpringBootApplication(exclude = {HibernateJpaAutoConfiguration.class})
public class MyApp { }

// Custom EntityManagerFactory
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource ds) {
    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setDataSource(ds);
    factory.setPackagesToScan("com.example.domain");
    HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
    adapter.setDatabase(Database.POSTGRESQL);
    factory.setJpaVendorAdapter(adapter);
    Properties props = new Properties();
    props.put("hibernate.format_sql", "true");
    factory.setJpaProperties(props);
    return factory;
}
```

> 💡 **Interviewer Tip:** `spring.jpa.open-in-view=true` (Spring Boot default before 2.x) keeps the Hibernate Session open for the whole HTTP request — enabling lazy loading in views but causing long-lived connections. **Disable it** (`false`) in production and use `@Transactional` boundaries properly.

</details>

---

### 🟡 Q25. What is the difference between `save()`, `persist()`, and `merge()` in JPA?

<details><summary>Click to reveal answer</summary>

```java
// EntityManager (JPA native)
// persist: Attaches NEW entity to session — must be transient (not have an ID)
User newUser = new User("Alice");
em.persist(newUser); // Makes user "managed" — changes tracked until commit
// After commit: newUser.getId() is populated

// merge: Copies state of DETACHED entity back to managed entity
User detached = userRepository.findById(1L); // Loaded, then session closed
detached.setName("Updated Name");
User managed = em.merge(detached); // Returns a NEW managed instance
// detached is still detached! Use 'managed' after merge.

// Spring Data JPA repository (simpler API)
// save() does: persist if new (no ID), merge if existing (has ID)
User newUser2 = userRepository.save(new User("Bob"));     // Calls persist
User existingUser = userRepository.save(updatedUser);    // Calls merge

// How Spring Data determines "new":
// 1. @Id field is null/0 → persist
// 2. @Id field has value → merge
// 3. Implement Persistable<ID> for custom logic
@Entity
public class Order implements Persistable<String> {
    @Id
    private String id = UUID.randomUUID().toString(); // ID set before save

    @Transient
    private boolean isNew = true; // Manually track

    @Override
    public String getId() { return id; }

    @Override
    public boolean isNew() { return isNew; }

    @PostPersist @PostLoad
    void markNotNew() { isNew = false; }
}
```

🚨 **Common Mistake:** Assuming changes to a detached entity are automatically persisted. You must call `save()` or `merge()` for changes on detached entities to be tracked.

</details>

---

### 🟡 Q26. What is `@Transactional` `readOnly = true` and when does it help?

<details><summary>Click to reveal answer</summary>

```java
@Service
public class ReportService {

    @Transactional(readOnly = true) // Performance optimization for reads
    public List<OrderSummary> generateMonthlyReport(YearMonth month) {
        return orderRepository.findSummaryByMonth(month);
    }
}
```

**Benefits of `readOnly = true`:**

1. **Hibernate optimization:** Disables dirty checking (no need to detect changes to entities) — faster and less memory
2. **Database optimization:** Some drivers/DBs use read-only connections with read replicas
3. **Connection routing:** Spring's routing DataSource can direct reads to read replicas
4. **Transaction flush mode:** Set to `NEVER` — Hibernate won't flush to DB during the transaction

```java
// Routing reads to read replica using readOnly
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {

        return new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                // Route to replica if current transaction is read-only
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "REPLICA"
                    : "PRIMARY";
            }

            {
                setTargetDataSources(Map.of("PRIMARY", primary, "REPLICA", replica));
                setDefaultTargetDataSource(primary);
            }
        };
    }
}

// Service
@Transactional(readOnly = true) // Routes to read replica
public List<Product> getProducts() {
    return productRepository.findAll(); // Hits read replica!
}

@Transactional // Routes to primary (write)
public Product createProduct(ProductRequest req) {
    return productRepository.save(new Product(req)); // Hits primary!
}
```

</details>

---

*End of Spring Framework & Hibernate Introduction*
