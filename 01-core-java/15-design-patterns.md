# Design Patterns (Backend-Relevant)

> 💡 **Interviewer Tip:** Design pattern questions test whether you understand *why* a pattern exists, not just its code structure. Always explain the problem a pattern solves before showing the solution. Bonus points for mentioning real-world Spring/backend usage.

---

## Table of Contents
1. [Creational Patterns](#creational-patterns)
   - Singleton, Factory Method, Abstract Factory, Builder, Prototype
2. [Structural Patterns](#structural-patterns)
   - Decorator, Proxy, Adapter
3. [Behavioral Patterns](#behavioral-patterns)
   - Strategy, Observer, Template Method, Chain of Responsibility, Command
4. [Pattern Q&A](#pattern-qa)

---

## Creational Patterns

### 🟡 Q1. Implement a thread-safe Singleton using double-checked locking.

<details><summary>Click to reveal answer</summary>

**Intent:** Ensure a class has only one instance and provide a global access point to it.

**When to use:** Shared resources (configuration, logging, thread pool, connection pool).

```java
public class DatabaseConnectionPool {

    // volatile ensures visibility across threads (prevents instruction reordering)
    private static volatile DatabaseConnectionPool instance;

    private final List<Connection> pool;

    private DatabaseConnectionPool() {
        // Private constructor prevents external instantiation
        this.pool = initializePool();
    }

    public static DatabaseConnectionPool getInstance() {
        if (instance == null) {                    // First check (no sync)
            synchronized (DatabaseConnectionPool.class) {
                if (instance == null) {            // Second check (with sync)
                    instance = new DatabaseConnectionPool();
                }
            }
        }
        return instance;
    }

    private List<Connection> initializePool() {
        // Initialize connection pool
        return new ArrayList<>();
    }

    public Connection getConnection() {
        // Return a connection from pool
        return pool.isEmpty() ? createNewConnection() : pool.remove(0);
    }
}
```

**Preferred alternative — Initialization-on-Demand Holder (thread-safe, lazy, no sync):**
```java
public class ConfigManager {

    private ConfigManager() {
        loadConfig();
    }

    // JVM guarantees class loading is thread-safe
    private static class Holder {
        private static final ConfigManager INSTANCE = new ConfigManager();
    }

    public static ConfigManager getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Simplest — Enum Singleton (Effective Java recommendation):**
```java
public enum AppConfig {
    INSTANCE;

    private final Properties properties = new Properties();

    AppConfig() {
        try (InputStream is = getClass().getResourceAsStream("/app.properties")) {
            properties.load(is);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public String get(String key) {
        return properties.getProperty(key);
    }
}

// Usage
String dbUrl = AppConfig.INSTANCE.get("db.url");
```

✅ **Best Practice:** Enum Singleton is serialization-safe and reflection-proof. Use it unless you need lazy initialization with complex setup.

**Spring equivalent:** `@Bean` with default singleton scope — Spring manages singleton lifecycle within the container.

</details>

---

### 🟡 Q2. Explain the Factory Method pattern with a full Java implementation.

<details><summary>Click to reveal answer</summary>

**Intent:** Define an interface for creating objects, but let subclasses decide which class to instantiate.

**When to use:** When the exact type of object to create varies, or to follow Open/Closed Principle.

```java
// Product interface
public interface Notification {
    void send(String recipient, String message);
}

// Concrete products
public class EmailNotification implements Notification {
    @Override
    public void send(String recipient, String message) {
        System.out.printf("Email to %s: %s%n", recipient, message);
        // SMTP logic here
    }
}

public class SMSNotification implements Notification {
    @Override
    public void send(String recipient, String message) {
        System.out.printf("SMS to %s: %s%n", recipient, message);
        // Twilio/AWS SNS logic here
    }
}

public class PushNotification implements Notification {
    @Override
    public void send(String recipient, String message) {
        System.out.printf("Push to %s: %s%n", recipient, message);
        // Firebase FCM logic here
    }
}

// Creator — defines the factory method
public abstract class NotificationService {

    // Factory method — subclasses override this
    protected abstract Notification createNotification();

    // Template method using the factory method
    public void notifyUser(String userId, String message) {
        User user = findUser(userId);
        Notification notification = createNotification();
        notification.send(user.getContact(), message);
        logNotification(userId, message);
    }

    private User findUser(String userId) { return new User(userId); }
    private void logNotification(String id, String msg) { /* log */ }
}

// Concrete creators
public class EmailNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new EmailNotification();
    }
}

public class SMSNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new SMSNotification();
    }
}

// Simple factory (not strictly Factory Method, but common):
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SMSNotification();
            case "PUSH"  -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Usage
NotificationFactory.create("EMAIL").send("alice@example.com", "Order shipped!");
```

**Real Spring usage:** `BeanFactory`, `FactoryBean<T>`, `@Bean` methods.

</details>

---

### 🔴 Q3. Implement the Abstract Factory pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Provide an interface for creating families of related objects without specifying concrete classes.

**When to use:** When you need families of related products (e.g., different DB dialects, different UI themes).

```java
// Abstract products
public interface Button { void render(); void onClick(); }
public interface TextField { void render(); String getValue(); }
public interface Dialog { void show(String message); }

// Concrete products — Material Design
public class MaterialButton implements Button {
    public void render() { System.out.println("Rendering Material Button"); }
    public void onClick() { System.out.println("Material ripple effect"); }
}
public class MaterialTextField implements TextField {
    public void render() { System.out.println("Rendering Material TextField"); }
    public String getValue() { return "material-input"; }
}

// Concrete products — Bootstrap
public class BootstrapButton implements Button {
    public void render() { System.out.println("Rendering Bootstrap Button"); }
    public void onClick() { System.out.println("Bootstrap click handler"); }
}
public class BootstrapTextField implements TextField {
    public void render() { System.out.println("Rendering Bootstrap TextField"); }
    public String getValue() { return "bootstrap-input"; }
}

// Abstract Factory interface
public interface UIComponentFactory {
    Button createButton();
    TextField createTextField();
    Dialog createDialog();
}

// Concrete factories
public class MaterialUIFactory implements UIComponentFactory {
    @Override public Button createButton()       { return new MaterialButton(); }
    @Override public TextField createTextField() { return new MaterialTextField(); }
    @Override public Dialog createDialog()       { return msg -> System.out.println("[Material] " + msg); }
}

public class BootstrapUIFactory implements UIComponentFactory {
    @Override public Button createButton()       { return new BootstrapButton(); }
    @Override public TextField createTextField() { return new BootstrapTextField(); }
    @Override public Dialog createDialog()       { return msg -> System.out.println("[Bootstrap] " + msg); }
}

// Client code uses factory without knowing concrete types
public class LoginForm {
    private final Button submitButton;
    private final TextField usernameField;
    private final TextField passwordField;

    public LoginForm(UIComponentFactory factory) {
        this.submitButton  = factory.createButton();
        this.usernameField = factory.createTextField();
        this.passwordField = factory.createTextField();
    }

    public void render() {
        usernameField.render();
        passwordField.render();
        submitButton.render();
    }
}

// Factory selection based on configuration
UIComponentFactory factory = "material".equals(System.getProperty("ui.theme"))
    ? new MaterialUIFactory()
    : new BootstrapUIFactory();

LoginForm form = new LoginForm(factory);
form.render();
```

**Real Spring usage:** `DataSource` creation — `HikariDataSource` vs `BasicDataSource` vs `EmbeddedDatabase` are like product families.

</details>

---

### 🟡 Q4. Implement the Builder pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Construct complex objects step-by-step, separating construction from representation.

**When to use:** Objects with many optional parameters, telescoping constructors, immutable objects.

```java
// The built object (immutable)
public final class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;
    private final boolean followRedirects;

    private HttpRequest(Builder builder) {
        this.method          = builder.method;
        this.url             = builder.url;
        this.headers         = Collections.unmodifiableMap(builder.headers);
        this.body            = builder.body;
        this.timeoutMs       = builder.timeoutMs;
        this.followRedirects = builder.followRedirects;
    }

    // Getters only (immutable)
    public String getMethod() { return method; }
    public String getUrl()    { return url; }
    public Map<String, String> getHeaders() { return headers; }
    public String getBody()   { return body; }

    @Override
    public String toString() {
        return String.format("HttpRequest{method=%s, url=%s, headers=%s}",
            method, url, headers);
    }

    // Static nested Builder
    public static class Builder {
        // Required parameters
        private final String method;
        private final String url;

        // Optional parameters with defaults
        private Map<String, String> headers = new HashMap<>();
        private String body = null;
        private int timeoutMs = 5000;
        private boolean followRedirects = true;

        public Builder(String method, String url) {
            if (method == null || url == null)
                throw new IllegalArgumentException("method and url are required");
            this.method = method;
            this.url    = url;
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this; // Fluent API
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int ms) {
            if (ms <= 0) throw new IllegalArgumentException("Timeout must be positive");
            this.timeoutMs = ms;
            return this;
        }

        public Builder followRedirects(boolean follow) {
            this.followRedirects = follow;
            return this;
        }

        public HttpRequest build() {
            validate();
            return new HttpRequest(this);
        }

        private void validate() {
            if ("POST".equals(method) && body == null) {
                throw new IllegalStateException("POST request requires a body");
            }
        }
    }
}

// Usage — fluent and readable
HttpRequest request = new HttpRequest.Builder("POST", "https://api.example.com/users")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\":\"Alice\"}")
    .timeout(3000)
    .build();

// Lombok @Builder in Spring projects:
@Builder
@Getter
@AllArgsConstructor
public class UserDto {
    private String name;
    private String email;
    @Builder.Default
    private String role = "USER";
}

UserDto dto = UserDto.builder()
    .name("Alice")
    .email("alice@example.com")
    .build();
```

</details>

---

### 🟡 Q5. Explain the Prototype pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Create new objects by copying an existing object (prototype), rather than instantiating from scratch.

**When to use:** When object creation is expensive, or when you need many similar objects with minor variations.

```java
// Prototype interface
public interface Cloneable {
    Object clone();
}

// Deep copy via copy constructor (preferred over Cloneable interface)
public class ReportConfig {
    private String title;
    private List<String> columns;
    private Map<String, Object> filters;
    private DateRange dateRange;

    // Copy constructor (deep copy)
    public ReportConfig(ReportConfig other) {
        this.title     = other.title; // String is immutable — safe
        this.columns   = new ArrayList<>(other.columns); // Deep copy list
        this.filters   = new HashMap<>(other.filters);   // Deep copy map
        this.dateRange = new DateRange(other.dateRange);  // Deep copy nested
    }

    public ReportConfig withTitle(String newTitle) {
        ReportConfig copy = new ReportConfig(this); // Clone
        copy.title = newTitle;                       // Modify copy
        return copy;
    }

    public ReportConfig withFilter(String key, Object value) {
        ReportConfig copy = new ReportConfig(this);
        copy.filters.put(key, value);
        return copy;
    }
}

// Usage — create variations of a base config
ReportConfig baseConfig = new ReportConfig("Sales Report",
    List.of("Date", "Product", "Amount"),
    Map.of("status", "ACTIVE"),
    DateRange.thisMonth());

// Clone and customize for different regions
ReportConfig usConfig = baseConfig
    .withTitle("US Sales Report")
    .withFilter("region", "US");

ReportConfig euConfig = baseConfig
    .withTitle("EU Sales Report")
    .withFilter("region", "EU");

// Java's Object.clone() approach (avoid — shallow by default)
public class Address implements Cloneable {
    private String street;
    private String city;

    @Override
    protected Address clone() {
        try {
            return (Address) super.clone(); // Shallow clone
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

🚨 **Common Mistake:** `Object.clone()` is a shallow copy — nested mutable objects are shared between original and copy.

</details>

---

## Structural Patterns

### 🟡 Q6. Implement the Decorator pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Attach additional behavior to objects dynamically by wrapping them, as an alternative to subclassing.

**When to use:** Adding optional features (caching, logging, compression, encryption) without modifying existing classes.

```java
// Component interface
public interface DataSource {
    void writeData(String data);
    String readData();
}

// Concrete component
public class FileDataSource implements DataSource {
    private final String filename;

    public FileDataSource(String filename) {
        this.filename = filename;
    }

    @Override
    public void writeData(String data) {
        System.out.println("Writing to file: " + filename);
        // File write logic
    }

    @Override
    public String readData() {
        System.out.println("Reading from file: " + filename);
        return "file-content";
    }
}

// Base decorator
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrappee;

    DataSourceDecorator(DataSource wrappee) {
        this.wrappee = wrappee;
    }

    @Override public void writeData(String data) { wrappee.writeData(data); }
    @Override public String readData()           { return wrappee.readData(); }
}

// Concrete decorators
public class EncryptionDecorator extends DataSourceDecorator {
    EncryptionDecorator(DataSource wrappee) { super(wrappee); }

    @Override
    public void writeData(String data) {
        wrappee.writeData(encrypt(data));
    }

    @Override
    public String readData() {
        return decrypt(wrappee.readData());
    }

    private String encrypt(String data) { return "ENC[" + data + "]"; }
    private String decrypt(String data) {
        return data.replaceAll("ENC\\[(.+)]", "$1");
    }
}

public class CompressionDecorator extends DataSourceDecorator {
    CompressionDecorator(DataSource wrappee) { super(wrappee); }

    @Override
    public void writeData(String data) {
        wrappee.writeData(compress(data));
    }

    @Override
    public String readData() {
        return decompress(wrappee.readData());
    }

    private String compress(String data)   { return "COMPRESSED(" + data + ")"; }
    private String decompress(String data) { return data.replace("COMPRESSED(", "").replace(")", ""); }
}

public class CachingDecorator extends DataSourceDecorator {
    private String cache;
    CachingDecorator(DataSource wrappee) { super(wrappee); }

    @Override
    public String readData() {
        if (cache == null) {
            cache = wrappee.readData();
        }
        return cache;
    }
}

// Usage — stack decorators
DataSource source = new CachingDecorator(
    new EncryptionDecorator(
        new CompressionDecorator(
            new FileDataSource("data.txt")
        )
    )
);
source.writeData("important data"); // Compressed → Encrypted → Written to file
String data = source.readData();    // Read from file → Decrypted → Decompressed → Cached
```

**Real Spring usage:** `HttpServletRequestWrapper`, `HttpServletResponseWrapper`, Spring Security filter chain wrapping HttpServletRequest.

</details>

---

### 🟡 Q7. Implement the Proxy pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Provide a surrogate that controls access to another object.

**Types:** Virtual Proxy (lazy init), Protection Proxy (access control), Remote Proxy, Caching Proxy.

```java
// Service interface
public interface UserService {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
}

// Real implementation
public class UserServiceImpl implements UserService {
    private final UserRepository repository;

    public UserServiceImpl(UserRepository repository) {
        this.repository = repository;
    }

    @Override
    public User findById(Long id) { return repository.findById(id).orElseThrow(); }
    @Override
    public List<User> findAll()   { return repository.findAll(); }
    @Override
    public void save(User user)   { repository.save(user); }
}

// Caching Proxy
public class CachedUserService implements UserService {
    private final UserService delegate;
    private final Map<Long, User> cache = new ConcurrentHashMap<>();

    public CachedUserService(UserService delegate) {
        this.delegate = delegate;
    }

    @Override
    public User findById(Long id) {
        return cache.computeIfAbsent(id, delegate::findById);
    }

    @Override
    public List<User> findAll() {
        return delegate.findAll();
    }

    @Override
    public void save(User user) {
        delegate.save(user);
        cache.put(user.getId(), user); // Update cache
    }
}

// Security Proxy
public class SecuredUserService implements UserService {
    private final UserService delegate;
    private final SecurityContext securityContext;

    public SecuredUserService(UserService delegate, SecurityContext ctx) {
        this.delegate = delegate;
        this.securityContext = ctx;
    }

    @Override
    public User findById(Long id) {
        return delegate.findById(id); // Anyone can read
    }

    @Override
    public List<User> findAll() {
        requireRole("ADMIN");
        return delegate.findAll();
    }

    @Override
    public void save(User user) {
        requireRole("ADMIN");
        delegate.save(user);
    }

    private void requireRole(String role) {
        if (!securityContext.hasRole(role)) {
            throw new AccessDeniedException("Role " + role + " required");
        }
    }
}

// Dynamic Proxy (JDK)
UserService dynamicProxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{UserService.class},
    (proxy, method, args) -> {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(realService, args);
        System.out.println("After: " + method.getName());
        return result;
    }
);
```

**Real Spring usage:** Spring AOP creates JDK dynamic proxies (for interfaces) or CGLIB proxies (for classes) to implement `@Transactional`, `@Cacheable`, `@Async`.

</details>

---

### 🟡 Q8. Implement the Adapter pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Convert the interface of a class into another interface that clients expect. Enables incompatible interfaces to work together.

**When to use:** Integrating legacy code, third-party libraries, or different API styles.

```java
// Target interface (what our system expects)
public interface PaymentProcessor {
    boolean processPayment(String cardNumber, double amount, String currency);
    boolean refund(String transactionId, double amount);
}

// Adaptee (legacy/third-party — incompatible interface)
public class LegacyPaymentGateway {
    public int chargeCard(long cardNum, int amountCents, String curr) {
        // Returns transaction ID or -1 on failure
        System.out.println("Legacy charge: " + cardNum + " " + amountCents + " " + curr);
        return 12345; // Transaction ID
    }

    public boolean reverseCharge(int transactionId, int amountCents) {
        System.out.println("Legacy reverse: " + transactionId);
        return true;
    }
}

// Adapter
public class LegacyPaymentAdapter implements PaymentProcessor {
    private final LegacyPaymentGateway legacy;

    public LegacyPaymentAdapter(LegacyPaymentGateway legacy) {
        this.legacy = legacy;
    }

    @Override
    public boolean processPayment(String cardNumber, double amount, String currency) {
        long cardNum = Long.parseLong(cardNumber.replaceAll("\\s+", ""));
        int cents = (int) Math.round(amount * 100);
        int txId = legacy.chargeCard(cardNum, cents, currency);
        return txId > 0;
    }

    @Override
    public boolean refund(String transactionId, double amount) {
        int txId = Integer.parseInt(transactionId);
        int cents = (int) Math.round(amount * 100);
        return legacy.reverseCharge(txId, cents);
    }
}

// Another example: adapting Iterator to Iterable
public class IteratorAdapter<T> implements Iterable<T> {
    private final Iterator<T> iterator;

    public IteratorAdapter(Iterator<T> iterator) {
        this.iterator = iterator;
    }

    @Override
    public Iterator<T> iterator() {
        return iterator;
    }
}

// Usage
LegacyPaymentGateway gateway = new LegacyPaymentGateway();
PaymentProcessor processor = new LegacyPaymentAdapter(gateway);

// Our code uses the clean PaymentProcessor interface
boolean success = processor.processPayment("4111 1111 1111 1111", 99.99, "USD");
```

</details>

---

## Behavioral Patterns

### 🟡 Q9. Implement the Strategy pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

**When to use:** Multiple variations of an algorithm, selecting behavior at runtime.

```java
// Strategy interface
public interface SortStrategy {
    <T extends Comparable<T>> void sort(List<T> data);
    String name();
}

// Concrete strategies
public class BubbleSortStrategy implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> data) {
        int n = data.size();
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (data.get(j).compareTo(data.get(j + 1)) > 0) {
                    T temp = data.get(j);
                    data.set(j, data.get(j + 1));
                    data.set(j + 1, temp);
                }
            }
        }
    }
    @Override public String name() { return "BubbleSort"; }
}

public class QuickSortStrategy implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> data) {
        Collections.sort(data); // Simplified — real quicksort logic would go here
    }
    @Override public String name() { return "QuickSort"; }
}

// Context
public class DataSorter {
    private SortStrategy strategy;

    public DataSorter(SortStrategy strategy) {
        this.strategy = strategy;
    }

    // Strategy can be changed at runtime
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public <T extends Comparable<T>> List<T> sort(List<T> data) {
        List<T> copy = new ArrayList<>(data);
        strategy.sort(copy);
        System.out.println("Sorted using: " + strategy.name());
        return copy;
    }
}

// Practical backend example: pricing strategy
public interface PricingStrategy {
    double calculatePrice(Product product, Customer customer);
}

public class RegularPricing implements PricingStrategy {
    public double calculatePrice(Product p, Customer c) { return p.getBasePrice(); }
}

public class PremiumDiscount implements PricingStrategy {
    public double calculatePrice(Product p, Customer c) {
        return p.getBasePrice() * 0.85; // 15% off
    }
}

public class SeasonalPricing implements PricingStrategy {
    private final double multiplier;
    public SeasonalPricing(double multiplier) { this.multiplier = multiplier; }
    public double calculatePrice(Product p, Customer c) {
        return p.getBasePrice() * multiplier;
    }
}

// Lambda as strategy (functional interface)
PricingStrategy vipPricing = (product, customer) ->
    product.getBasePrice() * (1 - customer.getLoyaltyDiscount());

// Usage
DataSorter sorter = new DataSorter(new BubbleSortStrategy());
List<Integer> sorted = sorter.sort(List.of(5, 3, 1, 4, 2));
sorter.setStrategy(new QuickSortStrategy()); // Switch strategy at runtime
```

**Real Spring usage:** `@ConditionalOnProperty` switching between bean implementations, multiple `AuthenticationProvider` implementations in Spring Security.

</details>

---

### 🟡 Q10. Implement the Observer pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Define a one-to-many dependency so that when one object changes state, all dependents are notified.

**When to use:** Event systems, pub/sub, model-view synchronization.

```java
// Observer interface
public interface EventListener<T> {
    void onEvent(T event);
}

// Observable subject
public class OrderService {
    private final Map<String, List<EventListener<?>>> listeners = new HashMap<>();

    public <T> void subscribe(String eventType, EventListener<T> listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }

    public <T> void unsubscribe(String eventType, EventListener<T> listener) {
        listeners.getOrDefault(eventType, List.of()).remove(listener);
    }

    @SuppressWarnings("unchecked")
    private <T> void emit(String eventType, T event) {
        listeners.getOrDefault(eventType, List.of())
            .forEach(l -> ((EventListener<T>) l).onEvent(event));
    }

    public Order placeOrder(OrderRequest request) {
        Order order = processOrder(request);
        emit("ORDER_PLACED", order);  // Notify all observers
        return order;
    }

    public void cancelOrder(Long orderId) {
        Order order = findOrder(orderId);
        order.cancel();
        emit("ORDER_CANCELLED", order);
    }

    private Order processOrder(OrderRequest request) { return new Order(request); }
    private Order findOrder(Long id)                 { return new Order(); }
}

// Concrete observers
public class EmailNotificationObserver implements EventListener<Order> {
    @Override
    public void onEvent(Order order) {
        System.out.println("Email sent for order: " + order.getId());
    }
}

public class InventoryObserver implements EventListener<Order> {
    @Override
    public void onEvent(Order order) {
        System.out.println("Inventory updated for order: " + order.getId());
        // Decrement stock
    }
}

public class AuditLogObserver implements EventListener<Order> {
    @Override
    public void onEvent(Order order) {
        System.out.println("Audit log: " + order.getId() + " at " + LocalDateTime.now());
    }
}

// Usage
OrderService orderService = new OrderService();
orderService.subscribe("ORDER_PLACED", new EmailNotificationObserver());
orderService.subscribe("ORDER_PLACED", new InventoryObserver());
orderService.subscribe("ORDER_PLACED", new AuditLogObserver());
orderService.subscribe("ORDER_CANCELLED", new EmailNotificationObserver());

orderService.placeOrder(new OrderRequest()); // Notifies all 3 observers
```

**Real Spring usage:** `ApplicationEventPublisher` / `@EventListener` / `ApplicationEvent` — Spring's built-in observer implementation.

```java
// Spring Event approach
@Component
public class OrderEventPublisher {
    @Autowired ApplicationEventPublisher publisher;

    public Order placeOrder(OrderRequest request) {
        Order order = processOrder(request);
        publisher.publishEvent(new OrderPlacedEvent(this, order));
        return order;
    }
}

@Component
public class OrderEmailHandler {
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        emailService.send(event.getOrder());
    }
}
```

</details>

---

### 🟡 Q11. Implement the Template Method pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Define the skeleton of an algorithm in a base class, deferring some steps to subclasses without changing the algorithm's structure.

**When to use:** Multiple classes share the same algorithm structure but differ in some steps.

```java
// Abstract class — defines the template method
public abstract class ReportGenerator {

    // Template method — defines the algorithm skeleton
    public final Report generate(ReportRequest request) {
        validateRequest(request);           // Common
        List<Object> rawData = fetchData(request); // Subclass-defined
        List<Object> processed = processData(rawData); // Subclass-defined
        Report report = formatReport(processed);    // Subclass-defined
        addHeader(report, request);         // Common
        addFooter(report);                  // Common (with hook)
        return report;
    }

    // Common steps
    private void validateRequest(ReportRequest request) {
        if (request.getStartDate().isAfter(request.getEndDate())) {
            throw new IllegalArgumentException("Start date must be before end date");
        }
    }

    private void addHeader(Report report, ReportRequest request) {
        report.setTitle(getReportTitle() + " - " + request.getStartDate());
    }

    // Hook method — subclasses can optionally override
    protected void addFooter(Report report) {
        report.setFooter("Generated at: " + LocalDateTime.now());
    }

    // Abstract steps — must be implemented by subclasses
    protected abstract String getReportTitle();
    protected abstract List<Object> fetchData(ReportRequest request);
    protected abstract List<Object> processData(List<Object> rawData);
    protected abstract Report formatReport(List<Object> data);
}

// Concrete implementation 1
public class SalesReportGenerator extends ReportGenerator {

    @Override
    protected String getReportTitle() { return "Sales Report"; }

    @Override
    protected List<Object> fetchData(ReportRequest request) {
        return salesRepository.findByDateRange(
            request.getStartDate(), request.getEndDate());
    }

    @Override
    protected List<Object> processData(List<Object> rawData) {
        return rawData.stream()
            .map(item -> aggregateSales(item))
            .collect(Collectors.toList());
    }

    @Override
    protected Report formatReport(List<Object> data) {
        return Report.asExcel(data);
    }

    @Override
    protected void addFooter(Report report) {
        super.addFooter(report);
        report.addNote("Confidential — Internal Use Only");
    }
}

// Concrete implementation 2
public class UserActivityReportGenerator extends ReportGenerator {

    @Override
    protected String getReportTitle() { return "User Activity Report"; }

    @Override
    protected List<Object> fetchData(ReportRequest request) {
        return auditLogRepository.findActivity(request.getStartDate(), request.getEndDate());
    }

    @Override
    protected List<Object> processData(List<Object> rawData) {
        return rawData.stream()
            .filter(log -> ((AuditLog) log).isSignificant())
            .collect(Collectors.toList());
    }

    @Override
    protected Report formatReport(List<Object> data) {
        return Report.asPdf(data);
    }
}
```

**Real Spring usage:** `JdbcTemplate` (executes common JDBC boilerplate, you provide SQL), `AbstractController`, `AbstractBeanFactory`.

</details>

---

### 🔴 Q12. Implement the Chain of Responsibility pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Pass a request along a chain of handlers. Each handler decides to process it or pass it to the next.

**When to use:** Request processing pipelines (auth → rate-limit → validation → business logic), middleware, filter chains.

```java
// Handler interface
public abstract class RequestHandler {
    private RequestHandler next;

    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next; // Enable chaining: a.setNext(b).setNext(c)
    }

    public abstract boolean handle(HttpRequest request, HttpResponse response);

    protected boolean passToNext(HttpRequest request, HttpResponse response) {
        if (next != null) {
            return next.handle(request, response);
        }
        return true; // End of chain — request approved
    }
}

// Concrete handlers
public class AuthenticationHandler extends RequestHandler {
    private final TokenService tokenService;

    public AuthenticationHandler(TokenService tokenService) {
        this.tokenService = tokenService;
    }

    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        String token = request.getHeader("Authorization");
        if (token == null || !tokenService.isValid(token)) {
            response.setStatus(401);
            response.setBody("Unauthorized");
            return false; // Stop chain
        }
        request.setAttribute("userId", tokenService.getUserId(token));
        return passToNext(request, response); // Continue chain
    }
}

public class RateLimitHandler extends RequestHandler {
    private final RateLimiter limiter;

    public RateLimitHandler(RateLimiter limiter) {
        this.limiter = limiter;
    }

    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        String userId = (String) request.getAttribute("userId");
        if (!limiter.allow(userId)) {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.setBody("Too Many Requests");
            return false;
        }
        return passToNext(request, response);
    }
}

public class ValidationHandler extends RequestHandler {
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        if (request.getBody() == null || request.getBody().isBlank()) {
            response.setStatus(400);
            response.setBody("Request body is required");
            return false;
        }
        return passToNext(request, response);
    }
}

public class BusinessLogicHandler extends RequestHandler {
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        // Process the actual business logic
        String result = processBusinessLogic(request.getBody());
        response.setStatus(200);
        response.setBody(result);
        return true;
    }
}

// Build the chain
RequestHandler chain = new AuthenticationHandler(tokenService);
chain.setNext(new RateLimitHandler(rateLimiter))
     .setNext(new ValidationHandler())
     .setNext(new BusinessLogicHandler());

// Handle request
chain.handle(incomingRequest, response);
```

**Real Spring usage:** Spring Security's `SecurityFilterChain`, Servlet `FilterChain`, Spring MVC `HandlerInterceptor` chain.

</details>

---

### 🟡 Q13. Implement the Command pattern.

<details><summary>Click to reveal answer</summary>

**Intent:** Encapsulate a request as an object, enabling undo/redo, queuing, and logging of requests.

**When to use:** Undo/redo functionality, transaction management, task queues, macro recording.

```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class TextEditor {
    private StringBuilder text = new StringBuilder();

    public void insertText(int pos, String content) {
        text.insert(pos, content);
    }

    public void deleteText(int pos, int length) {
        text.delete(pos, pos + length);
    }

    public String getText() { return text.toString(); }
}

// Concrete commands
public class InsertCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final String text;

    public InsertCommand(TextEditor editor, int position, String text) {
        this.editor   = editor;
        this.position = position;
        this.text     = text;
    }

    @Override
    public void execute() {
        editor.insertText(position, text);
    }

    @Override
    public void undo() {
        editor.deleteText(position, text.length()); // Reverse the insert
    }
}

public class DeleteCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final int length;
    private String deletedText; // Saved for undo

    public DeleteCommand(TextEditor editor, int position, int length) {
        this.editor   = editor;
        this.position = position;
        this.length   = length;
    }

    @Override
    public void execute() {
        deletedText = editor.getText().substring(position, position + length);
        editor.deleteText(position, length);
    }

    @Override
    public void undo() {
        editor.insertText(position, deletedText); // Restore deleted text
    }
}

// Invoker with undo/redo history
public class CommandHistory {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void execute(Command command) {
        command.execute();
        undoStack.push(command);
        redoStack.clear(); // New command clears redo history
    }

    public void undo() {
        if (!undoStack.isEmpty()) {
            Command cmd = undoStack.pop();
            cmd.undo();
            redoStack.push(cmd);
        }
    }

    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute();
            undoStack.push(cmd);
        }
    }
}

// Usage
TextEditor editor = new TextEditor();
CommandHistory history = new CommandHistory();

history.execute(new InsertCommand(editor, 0, "Hello"));
history.execute(new InsertCommand(editor, 5, " World"));
System.out.println(editor.getText()); // "Hello World"

history.undo();
System.out.println(editor.getText()); // "Hello"

history.redo();
System.out.println(editor.getText()); // "Hello World"
```

**Real Spring usage:** `@Transactional` rollback = undo command, Spring Batch `ItemProcessor` pipeline, `JdbcTemplate` wrapping SQL as commands.

</details>

---

## Pattern Q&A

### 🟢 Q14. When would you use Singleton vs Spring `@Bean`?

<details><summary>Click to reveal answer</summary>

```
| Concern         | Manual Singleton         | Spring @Bean              |
|-----------------|--------------------------|---------------------------|
| Lifecycle       | You manage it            | Spring container manages  |
| Testing         | Hard to mock             | Easy to mock with @MockBean|
| Dependency Injection | Manual              | Automatic via @Autowired  |
| Multiple instances | Hard                  | @Scope("prototype") easy  |
| Initialization  | Static / DCL             | Spring lazy/eager init    |
```

✅ **Best Practice:** In Spring applications, use `@Bean` / `@Component` singletons. Manual singletons are only appropriate in non-Spring contexts or framework-level code.

```java
// ❌ Manual singleton in Spring app — hard to test
public class EmailService {
    private static EmailService instance;
    public static EmailService getInstance() { ... }
}

// ✅ Spring managed bean — testable, injectable
@Service
public class EmailService {
    @Autowired private SmtpClient smtpClient;
}
```

</details>

---

### 🟡 Q15. How does Spring AOP relate to the Proxy pattern?

<details><summary>Click to reveal answer</summary>

Spring AOP wraps beans in **proxy objects** that intercept method calls to implement cross-cutting concerns.

```java
@Service
public class UserService {
    @Transactional  // Spring wraps this bean in a Proxy
    public void transferFunds(Long from, Long to, double amount) {
        // Spring's transaction proxy:
        // 1. Begins transaction before method
        // 2. Calls real method
        // 3. Commits or rolls back after method
        debit(from, amount);
        credit(to, amount);
    }

    @Cacheable("users")  // Caching Proxy
    public User findById(Long id) {
        return repository.findById(id).orElseThrow();
    }
}

// Behind the scenes, Spring creates:
// UserService proxy = Proxy.newProxyInstance(...) or CGLIB subclass
// Every @Transactional call goes through the proxy first
```

**JDK Proxy vs CGLIB:**
- **JDK Proxy:** Used when the bean implements an interface. Creates a proxy implementing the same interface.
- **CGLIB:** Used when no interface exists. Creates a subclass of the bean class.

> 💡 **Interviewer Tip:** `@Transactional` on a method called from within the same class bypasses the proxy — this is a common bug! `this.save()` doesn't go through the proxy.

</details>

---

### 🟡 Q16. What is the difference between Strategy and Template Method?

<details><summary>Click to reveal answer</summary>

Both define a family of algorithms, but differ in how variation is achieved:

| Aspect | Strategy | Template Method |
|---|---|---|
| Relationship | Composition | Inheritance |
| Algorithm | Entirely replaceable | Fixed skeleton, variable steps |
| Switching | At runtime | At compile time |
| Coupling | Loose | Tighter (subclass depends on parent) |

```java
// Template Method: inheritance — base class controls the skeleton
abstract class DataProcessor {
    final void process() {   // Fixed skeleton
        readData();          // Step 1: common
        processData();       // Step 2: varies
        writeData();         // Step 3: common
    }
    abstract void processData(); // Subclasses vary this step
}

// Strategy: composition — strategy object is swappable
class DataProcessor {
    private ProcessingStrategy strategy; // Injected

    void process() {
        readData();
        strategy.process(data); // Delegated to strategy
        writeData();
    }

    void setStrategy(ProcessingStrategy s) { this.strategy = s; } // Swap at runtime!
}
```

✅ **Best Practice:** Prefer Strategy over Template Method — composition over inheritance is more flexible and testable (easier to mock the strategy).

</details>

---

### 🔴 Q17. Explain the Decorator vs Proxy difference.

<details><summary>Click to reveal answer</summary>

Both wrap an object, but serve different purposes:

| Aspect | Decorator | Proxy |
|---|---|---|
| Intent | Add behavior | Control access |
| Who creates wrapped object | Client | Proxy itself |
| Interface | Same as wrapped | Same as wrapped |
| Transparency | Transparent to client | Client may be unaware of proxy |
| Layering | Multiple decorators stackable | Usually single proxy |

```java
// DECORATOR: Adding pizza toppings
Pizza pizza = new Mozzarella(new Pepperoni(new PlainPizza()));
// Client explicitly stacks decorators

// PROXY: Lazy loading of heavy object
class LazyImageProxy implements Image {
    private RealImage realImage; // Created only when needed

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage("huge-photo.jpg"); // Expensive!
        }
        realImage.display();
    }
    // Client doesn't know about lazy loading
}
```

In practice: Spring `@Cacheable` is more Proxy-like (controls access/caching), while `HttpServletRequestWrapper` is more Decorator-like (adds behavior).

</details>

---

### 🟡 Q18. What is the Builder pattern's advantage over constructor telescoping?

<details><summary>Click to reveal answer</summary>

**Telescoping constructor problem:** Multiple constructors with increasing parameters — unreadable and error-prone.

```java
// ❌ Telescoping constructors
class User {
    User(String name) { ... }
    User(String name, String email) { ... }
    User(String name, String email, int age) { ... }
    User(String name, String email, int age, String role) { ... }
    // ... grows rapidly

    // Usage — hard to read, what is the 4th param?
    User user = new User("Alice", "a@b.com", 30, "ADMIN");
}

// ✅ Builder pattern
User user = new User.Builder("Alice")
    .email("a@b.com")
    .age(30)
    .role("ADMIN")
    .build();
// Self-documenting, optional params skippable, validated in build()
```

**Advantages:**
1. Named "parameters" — clear what each value is
2. Optional parameters don't need overloads
3. Validation logic centralized in `build()`
4. Immutable objects possible (fields set once in constructor)
5. Easy to add new optional fields without breaking callers

</details>

---

### 🟢 Q19. What is the difference between Factory Method and Abstract Factory?

<details><summary>Click to reveal answer</summary>

| Aspect | Factory Method | Abstract Factory |
|---|---|---|
| Creates | One product | A family of related products |
| Structure | One factory method per subclass | One factory interface with multiple methods |
| Purpose | Let subclasses decide which class | Ensure products work together |

```java
// Factory Method: creates one type
interface LoggerFactory {
    Logger createLogger(); // One product
}

class FileLoggerFactory implements LoggerFactory {
    public Logger createLogger() { return new FileLogger(); }
}

// Abstract Factory: creates a family of related products
interface UIFactory {
    Button createButton();    // Product 1
    TextField createField();  // Product 2 (related to Button)
    Dialog createDialog();    // Product 3 (all same theme)
}
// All products from same factory are guaranteed to be compatible
```

</details>

---

### 🟡 Q20. When would you use the Command pattern vs Strategy?

<details><summary>Click to reveal answer</summary>

```
| Concern           | Command                     | Strategy                   |
|-------------------|-----------------------------|----------------------------|
| Encapsulates      | Request/action              | Algorithm                  |
| State             | Can store state (for undo)  | Usually stateless           |
| Undo/Redo         | Built-in support            | Not applicable              |
| Queuing           | Commands can be queued      | Not typical                 |
| Execution timing  | Deferred (execute later)    | Immediate (called by context)|
| Use case          | Text editor actions, batch jobs | Sorting, pricing, validation |
```

```java
// Command: encapsulates an action with undo capability
Command cmd = new TransferMoneyCommand(fromAccount, toAccount, 500.0);
commandHistory.execute(cmd);
// Later: commandHistory.undo() — reverses the transfer

// Strategy: encapsulates interchangeable algorithm
context.setStrategy(new PremiumPricingStrategy());
double price = context.calculatePrice(product); // Strategy runs immediately
```

</details>

---

*End of Design Patterns (Backend-Relevant)*
