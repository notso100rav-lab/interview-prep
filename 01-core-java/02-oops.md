# Chapter 2: Object-Oriented Programming (Design-Focused)

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- The four pillars: Encapsulation, Inheritance, Polymorphism, Abstraction
- SOLID principles (with Java examples and violations)
- Composition vs Inheritance
- Interface vs Abstract Class (evolution through Java versions)
- Covariant return types
- Diamond problem and resolution
- Method overloading vs overriding
- Real-world OOP design scenarios
- Design patterns referenced in interviews

---

## 🔑 Key Concepts at a Glance

```
OOP Pillars:
┌─────────────────────────────────────────────────────────────┐
│  ENCAPSULATION  → Hide internal state, expose via methods   │
│  INHERITANCE    → IS-A relationship, code reuse             │
│  POLYMORPHISM   → Same interface, different behavior        │
│  ABSTRACTION    → Expose essential, hide complexity         │
└─────────────────────────────────────────────────────────────┘

SOLID:
  S - Single Responsibility Principle
  O - Open/Closed Principle
  L - Liskov Substitution Principle
  I - Interface Segregation Principle
  D - Dependency Inversion Principle
```

---

## 🧠 Theory Questions

### The Four Pillars

---

🟢 **Q1. What is encapsulation and why is it important?**

<details><summary>Click to reveal answer</summary>

**Encapsulation** is the bundling of data (fields) and methods that operate on that data within a single class, while restricting direct access to some components.

**Why important:**
- **Data hiding:** Protects internal state from invalid modifications
- **Maintainability:** Internal implementation can change without affecting callers
- **Flexibility:** Can add validation, logging, caching in accessors

```java
// BAD: No encapsulation
public class BankAccount {
    public double balance;  // Anyone can set balance = -999999
}

// GOOD: Encapsulated
public class BankAccount {
    private double balance;
    private String accountNumber;

    public BankAccount(String accountNumber, double initialBalance) {
        if (initialBalance < 0) throw new IllegalArgumentException("Initial balance cannot be negative");
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    public double getBalance() { return balance; }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit must be positive");
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance) throw new InsufficientFundsException("Insufficient funds");
        balance -= amount;
    }
    // No setter for balance — it can only change via deposit/withdraw
}
```

**Levels of access control:**

| Modifier | Same Class | Same Package | Subclass | World |
|----------|-----------|-------------|---------|-------|
| `private` | ✅ | ❌ | ❌ | ❌ |
| (default) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

> 💡 **Interviewer Tip:** Go beyond "getters and setters." Emphasize that true encapsulation means designing a class around **behaviors** (deposit, withdraw) not just data access.

</details>

---

🟡 **Q2. What is inheritance and what are its drawbacks?**

<details><summary>Click to reveal answer</summary>

**Inheritance** is a mechanism where a subclass acquires properties and behaviors of a superclass (IS-A relationship).

```java
public abstract class Vehicle {
    protected String brand;
    protected int year;

    public Vehicle(String brand, int year) {
        this.brand = brand;
        this.year = year;
    }

    public abstract double fuelCost(int km);  // abstract: must override

    public String getInfo() {  // concrete: can inherit as-is
        return brand + " (" + year + ")";
    }
}

public class Car extends Vehicle {
    private double fuelEfficiency; // km per liter

    public Car(String brand, int year, double fuelEfficiency) {
        super(brand, year);
        this.fuelEfficiency = fuelEfficiency;
    }

    @Override
    public double fuelCost(int km) {
        return km / fuelEfficiency * 1.5; // fuel price = 1.5/L
    }
}
```

**Drawbacks of inheritance:**
1. **Tight coupling:** Subclass depends on superclass implementation details
2. **Fragile base class:** Changing superclass can break subclasses unexpectedly
3. **Single inheritance limit:** Java allows only one superclass per class
4. **Violated LSP:** Subclasses may not be truly substitutable
5. **Deep hierarchies:** Hard to understand, debug, and maintain

> 💡 **Effective Java (Bloch):** "Favor composition over inheritance." Inheritance is powerful but creates tight coupling. Use it only when a true IS-A relationship exists AND the subclass truly extends (not replaces) behavior.

</details>

---

🟡 **Q3. What are the types of polymorphism in Java?**

<details><summary>Click to reveal answer</summary>

Java supports two types:

**1. Compile-time (Static) Polymorphism — Method Overloading:**
```java
public class Calculator {
    public int add(int a, int b)       { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
    // resolved at compile time based on argument types/count
}
```

**2. Runtime (Dynamic) Polymorphism — Method Overriding:**
```java
abstract class Shape {
    abstract double area();
}

class Circle extends Shape {
    double r;
    Circle(double r) { this.r = r; }
    @Override double area() { return Math.PI * r * r; }
}

class Rectangle extends Shape {
    double w, h;
    Rectangle(double w, double h) { this.w = w; this.h = h; }
    @Override double area() { return w * h; }
}

// Runtime polymorphism:
Shape s = new Circle(5);
s.area();  // calls Circle.area() — resolved at runtime via vtable
s = new Rectangle(3, 4);
s.area();  // calls Rectangle.area()

// Useful for polymorphic processing:
List<Shape> shapes = List.of(new Circle(3), new Rectangle(4, 5));
double totalArea = shapes.stream().mapToDouble(Shape::area).sum();
```

**Dynamic Dispatch mechanism:**
The JVM uses a **virtual method table (vtable)** per class. When a method is called on a reference, the JVM looks up the actual object's class vtable at runtime.

</details>

---

🟡 **Q4. What is abstraction and how is it achieved in Java?**

<details><summary>Click to reveal answer</summary>

**Abstraction** hides implementation details and exposes only essential functionality.

**Two mechanisms:**
1. **Abstract classes** — partial abstraction
2. **Interfaces** — pure abstraction (before Java 8; now can have default/static methods)

```java
// Abstract class: partial abstraction
public abstract class DataProcessor {
    // Template method pattern — defines algorithm skeleton
    public final void process() {
        readData();
        transformData();
        writeData();
    }

    protected abstract void readData();
    protected abstract void transformData();
    protected abstract void writeData();

    // Concrete method — shared behavior
    protected void log(String message) {
        System.out.println("[" + getClass().getSimpleName() + "] " + message);
    }
}

public class CsvProcessor extends DataProcessor {
    @Override protected void readData() { log("Reading CSV"); }
    @Override protected void transformData() { log("Transforming CSV data"); }
    @Override protected void writeData() { log("Writing results"); }
}

// Interface: pure abstraction (contract)
public interface PaymentGateway {
    PaymentResult charge(Customer customer, Amount amount);
    RefundResult refund(String transactionId, Amount amount);
    PaymentStatus getStatus(String transactionId);
}

// Implementations hide complexity from callers:
public class StripePaymentGateway implements PaymentGateway {
    // Hides Stripe API, HTTP calls, retries, etc.
    @Override
    public PaymentResult charge(Customer customer, Amount amount) {
        // ... 200 lines of Stripe-specific code
    }
}
```

</details>

---

### Interface vs Abstract Class

---

🟡 **Q5. What is the difference between an interface and an abstract class? When to use each?**

<details><summary>Click to reveal answer</summary>

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Instantiation | ❌ Cannot | ❌ Cannot |
| Inheritance | Single (`extends`) | Multiple (`implements`) |
| Fields | Any (instance, static, final) | `public static final` only |
| Constructors | ✅ Yes | ❌ No |
| Methods | Abstract + concrete | Abstract + `default` + `static` (Java 8+) |
| Access modifiers | Any | `public` (implicit) |
| IS-A relationship | Strong (shared state/behavior) | Contract/capability |
| Use case | Shared base implementation | Define a contract/role |

**Java 8+ interface features:**
```java
public interface Sortable {
    // Abstract method — must implement
    int compareTo(Object other);

    // Default method — optional to override
    default void sort(List<?> list) {
        Collections.sort((List) list);
    }

    // Static method — called on interface
    static Comparator<Integer> naturalOrder() {
        return Integer::compareTo;
    }

    // Private method (Java 9+) — helper for default methods
    private void validate(Object obj) {
        if (obj == null) throw new NullPointerException();
    }
}
```

**When to use Abstract Class:**
- Sharing code among closely related classes
- Classes need non-public fields or constructors
- Template Method pattern (define algorithm skeleton)
- Need to maintain state across subclasses

**When to use Interface:**
- Unrelated classes need same behavior (e.g., `Comparable`, `Serializable`)
- Specifying a contract without implementation
- Multiple inheritance of type is needed
- Defining a capability/role (e.g., `Flyable`, `Printable`)

```java
// Combines both effectively:
public interface Repository<T, ID> {
    T findById(ID id);
    void save(T entity);
    void delete(ID id);
}

public abstract class BaseRepository<T, ID> implements Repository<T, ID> {
    protected final DataSource dataSource;

    protected BaseRepository(DataSource ds) { this.dataSource = ds; }

    // Shared implementation
    @Override
    public void delete(ID id) {
        T entity = findById(id);
        if (entity != null) performDelete(id);
    }

    protected abstract void performDelete(ID id);
}
```

</details>

---

🔴 **Q6. What is the diamond problem and how does Java resolve it?**

<details><summary>Click to reveal answer</summary>

The **diamond problem** occurs when a class inherits from two classes (or interfaces) that both define the same method, creating ambiguity.

```
     A
    / \
   B   C
    \ /
     D
```

**Java classes avoid this** because Java only supports single class inheritance.

**With interfaces (Java 8+ default methods):**
```java
interface A {
    default void greet() { System.out.println("Hello from A"); }
}

interface B extends A {
    default void greet() { System.out.println("Hello from B"); }
}

interface C extends A {
    default void greet() { System.out.println("Hello from C"); }
}

class D implements B, C {
    // MUST override greet() — compiler forces resolution
    @Override
    public void greet() {
        B.super.greet();  // Explicitly choose B's implementation
        // or C.super.greet();
        // or provide your own implementation
    }
}
```

**Resolution rules (in order):**
1. **Classes win over interfaces:** If a class defines the method, it wins
2. **More specific interfaces win:** If one interface extends another, the more specific one wins
3. **Must explicitly override:** If still ambiguous, compiler error — must override

```java
interface Printer {
    default void print() { System.out.println("Printer"); }
}
interface Scanner {
    default void print() { System.out.println("Scanner"); }
}

class MultiFunctionDevice implements Printer, Scanner {
    // COMPILER ERROR if we don't override:
    @Override
    public void print() {
        Printer.super.print(); // explicit resolution
    }
}
```

</details>

---

### SOLID Principles

---

🟡 **Q7. Explain the Single Responsibility Principle (SRP) with a violation and fix.**

<details><summary>Click to reveal answer</summary>

**SRP:** A class should have only **one reason to change** — one responsibility.

**Violation:**
```java
// BAD: UserService does TOO much
public class UserService {
    public void createUser(User user) { /* save to DB */ }
    public void sendWelcomeEmail(User user) { /* send email via SMTP */ }
    public String generateUserReport(List<User> users) { /* create CSV/PDF */ }
    public void validateUser(User user) { /* validate fields */ }
    public void logUserActivity(User user, String action) { /* write to log file */ }
}
// This class changes if: DB schema changes, email template changes,
// report format changes, validation rules change, or log format changes.
```

**Fix:**
```java
// Each class has ONE reason to change
public class UserRepository {
    public void save(User user) { /* DB persistence only */ }
    public User findById(long id) { /* DB query only */ }
}

public class UserEmailService {
    public void sendWelcomeEmail(User user) { /* email only */ }
}

public class UserReportGenerator {
    public String generateReport(List<User> users) { /* report format only */ }
}

public class UserValidator {
    public void validate(User user) throws ValidationException { /* validation only */ }
}

// Orchestrate in a thin service/facade:
public class UserService {
    private final UserRepository repository;
    private final UserEmailService emailService;
    private final UserValidator validator;

    public void createUser(User user) {
        validator.validate(user);
        repository.save(user);
        emailService.sendWelcomeEmail(user);
    }
}
```

> 💡 **Interviewer Tip:** SRP doesn't mean "do only one thing." It means "have only one stakeholder" or "one axis of change." A `UserService` can do many user-related things as long as all changes come from user business logic requirements.

</details>

---

🟡 **Q8. Explain the Open/Closed Principle (OCP).**

<details><summary>Click to reveal answer</summary>

**OCP:** Classes should be **open for extension** but **closed for modification**. Add new behavior by adding new code, not changing existing code.

**Violation:**
```java
// BAD: Must modify calculateDiscount() for every new discount type
public class DiscountCalculator {
    public double calculateDiscount(Order order, String discountType) {
        if (discountType.equals("PERCENTAGE")) {
            return order.getTotal() * 0.10;
        } else if (discountType.equals("FIXED")) {
            return 5.0;
        } else if (discountType.equals("BOGO")) {
            // new type added: modify existing class!
            return order.getTotal() * 0.50;
        }
        return 0;
    }
}
```

**Fix — use polymorphism/strategy pattern:**
```java
// Open for extension via new implementations
public interface DiscountStrategy {
    double calculate(Order order);
}

public class PercentageDiscount implements DiscountStrategy {
    private final double percentage;
    public PercentageDiscount(double percentage) { this.percentage = percentage; }
    @Override public double calculate(Order order) { return order.getTotal() * percentage; }
}

public class FixedDiscount implements DiscountStrategy {
    private final double amount;
    public FixedDiscount(double amount) { this.amount = amount; }
    @Override public double calculate(Order order) { return amount; }
}

// New discount type: just add new class, don't modify DiscountCalculator
public class BuyOneGetOneDiscount implements DiscountStrategy {
    @Override public double calculate(Order order) { return order.getTotal() * 0.5; }
}

// Closed for modification
public class DiscountCalculator {
    public double calculateDiscount(Order order, DiscountStrategy strategy) {
        return strategy.calculate(order);
    }
}
```

</details>

---

🔴 **Q9. Explain the Liskov Substitution Principle (LSP) with a violation.**

<details><summary>Click to reveal answer</summary>

**LSP:** Objects of a subclass should be substitutable for objects of its superclass without altering the correctness of the program.

**Classic violation — Square extends Rectangle:**
```java
// BAD: Square IS-A Rectangle geometrically, but NOT in code
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // must keep equal sides
    }

    @Override
    public void setHeight(int height) {
        this.width = height;
        this.height = height;
    }
}

// Code that works for Rectangle but breaks with Square:
void resizeAndVerify(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50;  // FAILS for Square! (area = 100)
}
```

**Fix — don't force the inheritance:**
```java
// Use interface instead, don't inherit
public interface Shape {
    int area();
}

public class Rectangle implements Shape {
    private final int width, height;
    Rectangle(int w, int h) { this.width = w; this.height = h; }
    @Override public int area() { return width * height; }
}

public class Square implements Shape {
    private final int side;
    Square(int side) { this.side = side; }
    @Override public int area() { return side * side; }
}
```

**LSP requirements:**
- Preconditions cannot be strengthened in a subtype (subclass can't be more restrictive)
- Postconditions cannot be weakened in a subtype (subclass can't do less)
- Invariants of the supertype must be preserved
- No new exceptions should be thrown that the supertype doesn't throw

🚨 **Common violation pattern:** Overriding methods to throw `UnsupportedOperationException` (like `ArrayList.subList` → unmodifiable list) violates LSP.

</details>

---

🟡 **Q10. Explain the Interface Segregation Principle (ISP).**

<details><summary>Click to reveal answer</summary>

**ISP:** Clients should not be forced to depend on interfaces they do not use. Many specific interfaces are better than one fat interface.

**Violation:**
```java
// BAD: Fat interface forcing implementors to stub methods
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeCode();
    void manageTeam();
}

// Robot doesn't eat or sleep or attend meetings!
public class Robot implements Worker {
    @Override public void work() { /* robot works */ }
    @Override public void eat() { throw new UnsupportedOperationException(); }
    @Override public void sleep() { throw new UnsupportedOperationException(); }
    @Override public void attendMeeting() { throw new UnsupportedOperationException(); }
    @Override public void writeCode() { /* robot codes */ }
    @Override public void manageTeam() { throw new UnsupportedOperationException(); }
}
```

**Fix:**
```java
public interface Workable { void work(); }
public interface Feedable { void eat(); }
public interface Sleepable { void sleep(); }
public interface Codeable { void writeCode(); }
public interface TeamManager { void manageTeam(); }
public interface MeetingAttendee { void attendMeeting(); }

// Human employee implements relevant interfaces
public class HumanEmployee implements Workable, Feedable, Sleepable, Codeable, MeetingAttendee {
    @Override public void work() { /* ... */ }
    @Override public void eat() { /* ... */ }
    @Override public void sleep() { /* ... */ }
    @Override public void writeCode() { /* ... */ }
    @Override public void attendMeeting() { /* ... */ }
}

// Robot only implements what applies
public class Robot implements Workable, Codeable {
    @Override public void work() { /* ... */ }
    @Override public void writeCode() { /* ... */ }
}
```

</details>

---

🟡 **Q11. Explain the Dependency Inversion Principle (DIP).**

<details><summary>Click to reveal answer</summary>

**DIP:** 
1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on details. Details should depend on abstractions.

**Violation:**
```java
// BAD: High-level OrderService depends on concrete low-level EmailService
public class OrderService {
    private EmailService emailService = new EmailService(); // direct dependency!

    public void placeOrder(Order order) {
        // process order...
        emailService.sendConfirmation(order); // tightly coupled
    }
}

public class EmailService {
    public void sendConfirmation(Order order) { /* send email */ }
}
```

**Fix — inject the abstraction:**
```java
// Abstraction (interface)
public interface NotificationService {
    void sendOrderConfirmation(Order order);
}

// Concrete implementations depend on the abstraction
public class EmailNotificationService implements NotificationService {
    @Override public void sendOrderConfirmation(Order order) { /* email logic */ }
}

public class SmsNotificationService implements NotificationService {
    @Override public void sendOrderConfirmation(Order order) { /* SMS logic */ }
}

// High-level module depends on abstraction, NOT on concrete class
public class OrderService {
    private final NotificationService notificationService;

    // Dependency injected (via constructor)
    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        // process order...
        notificationService.sendOrderConfirmation(order);
    }
}

// Wiring (e.g., Spring does this automatically with @Autowired)
NotificationService ns = new EmailNotificationService();
OrderService os = new OrderService(ns);
// Easy to swap: new OrderService(new SmsNotificationService());
```

> 💡 **Interviewer Tip:** DIP is the foundation of Dependency Injection (DI) frameworks like Spring. Understanding DIP helps explain why Spring's `@Autowired` is designed around interfaces.

</details>

---

### Composition vs Inheritance

---

🟡 **Q12. When should you prefer composition over inheritance?**

<details><summary>Click to reveal answer</summary>

**Inheritance (IS-A):** Use when the subclass truly IS a type of the superclass and extends its behavior.

**Composition (HAS-A):** Use when you want to reuse functionality without tight coupling.

**Classic example — favor composition:**
```java
// BAD with inheritance: Stack extends Vector
// Stack IS-A Vector? Not really — Stack shouldn't expose Vector's random-access methods!
java.util.Stack<E> extends Vector<E>  // broken abstraction — can call stack.get(0), etc.

// GOOD with composition:
public class Stack<E> {
    private final ArrayDeque<E> deque = new ArrayDeque<>();  // HAS-A

    public void push(E item) { deque.push(item); }
    public E pop() { return deque.pop(); }
    public E peek() { return deque.peek(); }
    public boolean isEmpty() { return deque.isEmpty(); }
    // Only exposes Stack-appropriate operations
}
```

**Benefits of composition:**
1. **Loose coupling:** Implementation details hidden
2. **Flexibility:** Can change the composed object at runtime
3. **No fragile base class problem**
4. **No violation of LSP**

```java
// Strategy pattern uses composition
public class Sorter {
    private SortStrategy strategy;  // HAS-A strategy

    public Sorter(SortStrategy strategy) { this.strategy = strategy; }

    // Can change strategy at runtime!
    public void setStrategy(SortStrategy strategy) { this.strategy = strategy; }

    public void sort(int[] arr) { strategy.sort(arr); }
}

// Choose inheritance when:
// 1. There is a true IS-A relationship
// 2. You want polymorphism (runtime method dispatch)
// 3. You need to extend, not replace, superclass behavior
public abstract class HttpServlet {
    // Template method
    public final void service(HttpRequest req, HttpResponse res) {
        if ("GET".equals(req.getMethod())) doGet(req, res);
        else if ("POST".equals(req.getMethod())) doPost(req, res);
    }

    protected void doGet(HttpRequest req, HttpResponse res) { /* default: 405 */ }
    protected void doPost(HttpRequest req, HttpResponse res) { /* default: 405 */ }
}
```

</details>

---

### Method Overloading vs Overriding

---

🟡 **Q13. What is the difference between method overloading and method overriding?**

<details><summary>Click to reveal answer</summary>

| Feature | Overloading | Overriding |
|---------|------------|-----------|
| **Class** | Same class (or subclass) | Subclass only |
| **Signature** | Different parameters | Same signature |
| **Return type** | Can differ | Same or covariant |
| **Access modifier** | Can differ | Cannot be more restrictive |
| **Exception** | Can differ | Cannot throw broader checked exceptions |
| **Resolution** | Compile-time (static) | Runtime (dynamic) |
| **`static` methods** | Can overload | Cannot override (hiding) |
| **`final` methods** | Can overload | Cannot override |

```java
class Animal {
    // Original method
    public void eat(String food) {
        System.out.println("Animal eats " + food);
    }
}

class Dog extends Animal {
    // OVERLOADING: same name, different parameter types
    public void eat(int amount) {  // different signature
        System.out.println("Dog eats " + amount + " grams");
    }

    // OVERRIDING: same signature
    @Override
    public void eat(String food) {
        System.out.println("Dog eats " + food + " enthusiastically");
    }
}
```

**Covariant return types (Java 5+):**
```java
class Animal {
    public Animal create() { return new Animal(); }
}
class Dog extends Animal {
    @Override
    public Dog create() { return new Dog(); }  // covariant: Dog is a subtype of Animal
}
```

**Rules for overriding:**
```java
class Parent {
    protected Object process(String s) throws IOException { return null; }
}
class Child extends Parent {
    @Override
    // ✅ can narrow return type (String is-a Object)
    // ✅ can widen access (public is wider than protected)
    // ✅ can throw narrower checked exception (FileNotFoundException is-a IOException)
    // ✅ can add unchecked exceptions
    public String process(String s) throws FileNotFoundException {
        return "processed";
    }
}
```

</details>

---

🔴 **Q14. What is the `@Override` annotation and why should you always use it?**

<details><summary>Click to reveal answer</summary>

`@Override` is a marker annotation that instructs the compiler to verify that the method is actually overriding a superclass method or implementing an interface method.

**Why always use it:**
```java
class Animal {
    public void speak() { System.out.println("..."); }
}

class Dog extends Animal {
    // BUG: typo — this does NOT override Animal.speak()
    // Creates a NEW overloaded method instead!
    public void speek() { System.out.println("Woof"); }

    // With @Override:
    @Override
    public void speek() { // COMPILE ERROR: no method to override!
        System.out.println("Woof");
    }
}
```

**Cases where @Override helps:**
1. Catches typos in method names
2. Catches wrong parameter types
3. Detects when the superclass method was removed/renamed
4. When implementing interface methods (Java 6+)
5. Documents intent clearly for readers

```java
// Interface implementation
public class ArrayList<E> implements List<E> {
    @Override
    public boolean add(E e) { ... }  // @Override ensures we're implementing List.add()
}
```

> 💡 **Interviewer Tip:** Always use `@Override`. There is literally no downside. Not using it is considered bad practice in code reviews.

</details>

---

### Real-World OOP Design Scenarios

---

🔴 **Q15. Design a parking lot system using OOP principles.**

<details><summary>Click to reveal answer</summary>

```java
// Enums for types
public enum VehicleType { MOTORCYCLE, CAR, BUS }
public enum SpotType { SMALL, MEDIUM, LARGE }
public enum SpotStatus { AVAILABLE, OCCUPIED }

// Vehicle hierarchy
public abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;

    public Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }

    public abstract SpotType requiredSpotType();
    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
}

public class Motorcycle extends Vehicle {
    public Motorcycle(String plate) { super(plate, VehicleType.MOTORCYCLE); }
    @Override public SpotType requiredSpotType() { return SpotType.SMALL; }
}

public class Car extends Vehicle {
    public Car(String plate) { super(plate, VehicleType.CAR); }
    @Override public SpotType requiredSpotType() { return SpotType.MEDIUM; }
}

public class Bus extends Vehicle {
    public Bus(String plate) { super(plate, VehicleType.BUS); }
    @Override public SpotType requiredSpotType() { return SpotType.LARGE; }
}

// Parking spot
public class ParkingSpot {
    private final int id;
    private final SpotType type;
    private SpotStatus status;
    private Vehicle currentVehicle;

    public ParkingSpot(int id, SpotType type) {
        this.id = id;
        this.type = type;
        this.status = SpotStatus.AVAILABLE;
    }

    public boolean canFit(Vehicle v) {
        return status == SpotStatus.AVAILABLE && type == v.requiredSpotType();
    }

    public void park(Vehicle v) {
        if (!canFit(v)) throw new IllegalStateException("Cannot park here");
        currentVehicle = v;
        status = SpotStatus.OCCUPIED;
    }

    public Vehicle leave() {
        Vehicle v = currentVehicle;
        currentVehicle = null;
        status = SpotStatus.AVAILABLE;
        return v;
    }

    public int getId() { return id; }
    public SpotType getType() { return type; }
    public SpotStatus getStatus() { return status; }
}

// Ticket
public class ParkingTicket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final Instant entryTime;

    public ParkingTicket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = UUID.randomUUID().toString();
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = Instant.now();
    }

    public Duration getDuration() { return Duration.between(entryTime, Instant.now()); }
    public ParkingSpot getSpot() { return spot; }
    // getters...
}

// Pricing strategy (OCP — open for new strategies)
public interface PricingStrategy {
    double calculateFee(Duration duration, SpotType spotType);
}

public class HourlyPricing implements PricingStrategy {
    @Override
    public double calculateFee(Duration duration, SpotType spotType) {
        long hours = Math.max(1, duration.toHours());
        double ratePerHour = switch (spotType) {
            case SMALL -> 1.0;
            case MEDIUM -> 2.0;
            case LARGE -> 4.0;
        };
        return hours * ratePerHour;
    }
}

// Parking lot (orchestrates everything)
public class ParkingLot {
    private final String name;
    private final List<ParkingSpot> spots;
    private final Map<String, ParkingTicket> activeTickets; // licensePlate -> ticket
    private final PricingStrategy pricing;

    public ParkingLot(String name, int small, int medium, int large, PricingStrategy pricing) {
        this.name = name;
        this.spots = new ArrayList<>();
        this.activeTickets = new HashMap<>();
        this.pricing = pricing;
        // Initialize spots
        int id = 1;
        for (int i = 0; i < small; i++) spots.add(new ParkingSpot(id++, SpotType.SMALL));
        for (int i = 0; i < medium; i++) spots.add(new ParkingSpot(id++, SpotType.MEDIUM));
        for (int i = 0; i < large; i++) spots.add(new ParkingSpot(id++, SpotType.LARGE));
    }

    public Optional<ParkingTicket> park(Vehicle vehicle) {
        Optional<ParkingSpot> spot = spots.stream()
            .filter(s -> s.canFit(vehicle))
            .findFirst();

        if (spot.isEmpty()) return Optional.empty();

        spot.get().park(vehicle);
        ParkingTicket ticket = new ParkingTicket(vehicle, spot.get());
        activeTickets.put(vehicle.getLicensePlate(), ticket);
        return Optional.of(ticket);
    }

    public double leave(String licensePlate) {
        ParkingTicket ticket = activeTickets.remove(licensePlate);
        if (ticket == null) throw new IllegalArgumentException("No ticket found");

        double fee = pricing.calculateFee(ticket.getDuration(), ticket.getSpot().getType());
        ticket.getSpot().leave();
        return fee;
    }

    public long availableSpots(SpotType type) {
        return spots.stream()
            .filter(s -> s.getType() == type && s.getStatus() == SpotStatus.AVAILABLE)
            .count();
    }
}
```

**Design decisions to discuss in interview:**
- OOP pillars demonstrated: encapsulation (spot status), inheritance (Vehicle hierarchy), polymorphism (pricing strategy), abstraction (interfaces)
- SOLID: SRP (each class has one job), OCP (add new pricing/vehicle types), DIP (ParkingLot depends on PricingStrategy interface)
- Thread safety: In production, use `ConcurrentHashMap` and synchronized spot access

</details>

---

## 🎭 Scenario-Based Questions

---

🟡 **S1. What's the output of this polymorphism scenario?**

```java
class Animal {
    String name = "Animal";
    void sound() { System.out.println("Generic sound"); }
}

class Dog extends Animal {
    String name = "Dog";  // field hiding
    @Override
    void sound() { System.out.println("Woof"); }
}

public class Main {
    public static void main(String[] args) {
        Animal a = new Dog();
        System.out.println(a.name);   // A
        a.sound();                     // B

        Dog d = (Dog) a;
        System.out.println(d.name);   // C
        d.sound();                     // D
    }
}
```

<details><summary>Click to reveal answer</summary>

- **A:** `"Animal"` — field access is **NOT** polymorphic. Fields are resolved by the reference type, not the object type.
- **B:** `"Woof"` — method calls ARE polymorphic. `sound()` is resolved by the actual object type (`Dog`).
- **C:** `"Dog"` — `d` is of type `Dog`, so `d.name` accesses `Dog.name`
- **D:** `"Woof"` — still calls `Dog.sound()`

🚨 **Key insight:** Java has **no field polymorphism**. Only methods are dispatched dynamically. This is why you should always make fields private and access via methods.

</details>

---

🟡 **S2. Does this violate LSP?**

```java
public interface FileReader {
    String read(String path) throws IOException;
}

public class SecureFileReader implements FileReader {
    @Override
    public String read(String path) throws IOException, SecurityException {
        // SecurityException is unchecked, so this compiles fine
        checkPermissions(path);
        return Files.readString(Path.of(path));
    }
}
```

<details><summary>Click to reveal answer</summary>

**Technically compiles** (unchecked exceptions don't violate the override rule), but **may violate LSP** depending on usage.

If callers have code like:
```java
FileReader reader = new SecureFileReader();
try {
    String content = reader.read("/etc/passwd");
} catch (IOException e) {
    // handle IO error
}
// SecurityException is unchecked — caller might not handle it!
```

**LSP concern:** Callers coded against `FileReader` interface don't expect `SecurityException`. This breaks substitutability if not documented.

**Better approach:**
```java
public class SecureFileReader implements FileReader {
    @Override
    public String read(String path) throws IOException {
        if (!hasPermission(path)) {
            throw new AccessDeniedException(path);  // subtype of IOException
        }
        return Files.readString(Path.of(path));
    }
}
```

</details>

---

🔴 **S3. What design pattern should be used here, and why?**

```java
// Problem: Need to export reports in PDF, CSV, or Excel format
// Current (bad) approach:
public class ReportExporter {
    public void export(Report report, String format) {
        if (format.equals("PDF")) {
            // 50 lines of PDF logic
        } else if (format.equals("CSV")) {
            // 50 lines of CSV logic
        } else if (format.equals("EXCEL")) {
            // 50 lines of Excel logic
        }
        // Adding new format = modify this class (OCP violation)
    }
}
```

<details><summary>Click to reveal answer</summary>

**Strategy Pattern** — encapsulate the algorithm (export format) behind an interface.

```java
// Strategy interface
public interface ExportStrategy {
    void export(Report report, OutputStream output) throws IOException;
    String getContentType();
    String getFileExtension();
}

// Concrete strategies
public class PdfExportStrategy implements ExportStrategy {
    @Override
    public void export(Report report, OutputStream output) throws IOException {
        // PDF generation logic (e.g., using iText or Apache PDFBox)
    }
    @Override public String getContentType() { return "application/pdf"; }
    @Override public String getFileExtension() { return ".pdf"; }
}

public class CsvExportStrategy implements ExportStrategy {
    @Override
    public void export(Report report, OutputStream output) throws IOException {
        // CSV generation logic
    }
    @Override public String getContentType() { return "text/csv"; }
    @Override public String getFileExtension() { return ".csv"; }
}

// Context — closed for modification, open for extension
public class ReportExporter {
    private final Map<String, ExportStrategy> strategies;

    public ReportExporter(List<ExportStrategy> strategies) {
        this.strategies = strategies.stream()
            .collect(Collectors.toMap(
                s -> s.getFileExtension(),
                s -> s
            ));
    }

    public void export(Report report, String format, OutputStream output) throws IOException {
        ExportStrategy strategy = strategies.get(format);
        if (strategy == null) throw new UnsupportedFormatException(format);
        strategy.export(report, output);
    }
}

// Adding Excel support: just create ExcelExportStrategy and register it — no existing code changes!
```

</details>

---

🟡 **S4. What's wrong with this inheritance hierarchy?**

```java
public class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
    public void eat() {
        System.out.println("Eating...");
    }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}
```

<details><summary>Click to reveal answer</summary>

This violates **LSP**. A `Penguin` IS-A `Bird`, but cannot be substituted wherever a `Bird` is expected when `fly()` is called.

```java
void makeBirdFly(Bird bird) {
    bird.fly();  // throws exception if bird is a Penguin!
}
```

**Fix 1: Restructure the hierarchy:**
```java
public interface Eatable { void eat(); }
public interface Flyable { void fly(); }

public abstract class Bird implements Eatable { }

public class FlyingBird extends Bird implements Flyable {
    @Override public void fly() { System.out.println("Flying"); }
}

public class Penguin extends Bird {
    @Override public void eat() { System.out.println("Eating fish"); }
    // No fly() — penguins don't implement Flyable
}

public class Eagle extends FlyingBird {
    @Override public void eat() { System.out.println("Eating prey"); }
}
```

**Fix 2: Composition with capability check:**
```java
public class Bird {
    private final boolean canFly;
    Bird(boolean canFly) { this.canFly = canFly; }
    public boolean canFly() { return canFly; }
    public void fly() {
        if (!canFly) throw new UnsupportedOperationException();
        System.out.println("Flying");
    }
}
```

> 💡 This is the classic "penguin problem" in OOP. The lesson is: just because something IS-A in the real world doesn't mean inheritance is the right model in code.

</details>

---

🟡 **S5. Can a constructor be private? What's the use case?**

<details><summary>Click to reveal answer</summary>

Yes, constructors can be `private`. Use cases:

**1. Singleton pattern:**
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}  // prevent external instantiation
    public static Singleton getInstance() { return INSTANCE; }
}
```

**2. Factory method pattern (control instantiation):**
```java
public class Connection {
    private final String url;
    private Connection(String url) { this.url = url; }  // private

    public static Connection create(String url) {
        validate(url);
        return new Connection(url);
    }

    public static Connection fromEnvironment() {
        return new Connection(System.getenv("DB_URL"));
    }
}
```

**3. Utility class (prevent instantiation):**
```java
public final class MathUtils {
    private MathUtils() { throw new AssertionError("Utility class"); }
    public static int add(int a, int b) { return a + b; }
}
```

**4. Builder pattern (inner builder creates outer class):**
```java
public class Person {
    private final String name;
    private final int age;

    private Person(Builder b) { // private constructor
        this.name = b.name;
        this.age = b.age;
    }

    public static class Builder {
        private String name;
        private int age;
        public Builder name(String n) { this.name = n; return this; }
        public Builder age(int a) { this.age = a; return this; }
        public Person build() { return new Person(this); }
    }
}
// Usage: new Person.Builder().name("Alice").age(30).build();
```

</details>

---

🔴 **S6. What is method hiding vs method overriding?**

```java
class Parent {
    static void staticMethod() { System.out.println("Parent static"); }
    void instanceMethod() { System.out.println("Parent instance"); }
}

class Child extends Parent {
    static void staticMethod() { System.out.println("Child static"); }
    @Override void instanceMethod() { System.out.println("Child instance"); }
}

Parent obj = new Child();
obj.staticMethod();    // A
obj.instanceMethod();  // B
```

<details><summary>Click to reveal answer</summary>

- **A:** `"Parent static"` — Static methods are **hidden**, not overridden. Resolution is based on the **reference type** at compile time. `obj` is of type `Parent`, so `Parent.staticMethod()` is called.
- **B:** `"Child instance"` — Instance methods are **overridden**. Resolution is based on the **actual object type** at runtime (`Child`), so `Child.instanceMethod()` is called.

**Summary:**
| | Static Method | Instance Method |
|-|---------------|----------------|
| **In subclass with same signature** | Method hiding | Method overriding |
| **Resolution** | Compile-time (reference type) | Runtime (object type) |
| **Polymorphic?** | ❌ No | ✅ Yes |
| **@Override** | ❌ Not applicable | ✅ Required/recommended |

</details>

---

🟡 **S7. Can you override `equals()` without overriding `hashCode()`? What happens?**

<details><summary>Click to reveal answer</summary>

You **can** compile and run, but it **breaks the contract** — particularly for any hash-based collections.

**The Contract:**
- If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` MUST be `true`
- If `a.hashCode() == b.hashCode()`, `a.equals(b)` may be `true` or `false`

**Breaking the contract:**
```java
public class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
    // NO hashCode override!
}

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
p1.equals(p2)  // true ✓

Set<Point> set = new HashSet<>();
set.add(p1);
set.contains(p2);  // false! p1 and p2 have different hashCodes (from Object)
                   // They're in different buckets — never found!

Map<Point, String> map = new HashMap<>();
map.put(p1, "origin-ish");
map.get(p2);  // null! Same problem.
```

**Correct implementation:**
```java
@Override
public int hashCode() {
    return Objects.hash(x, y);  // consistent with equals
}

// OR: auto-generate with IDE (IntelliJ: Alt+Insert → equals/hashCode)
// OR: use Java 16+ records (generates equals/hashCode automatically)
record Point(int x, int y) {}
```

🚨 **Rule:** Always override `hashCode` when you override `equals`.

</details>

---

🟡 **S8. Design a simple Observer pattern for a stock price notification system.**

<details><summary>Click to reveal answer</summary>

```java
// Observer interface
public interface StockObserver {
    void onPriceChange(String symbol, double oldPrice, double newPrice);
}

// Subject interface
public interface StockSubject {
    void addObserver(StockObserver observer);
    void removeObserver(StockObserver observer);
    void notifyObservers(String symbol, double oldPrice, double newPrice);
}

// Concrete Subject
public class StockMarket implements StockSubject {
    private final List<StockObserver> observers = new CopyOnWriteArrayList<>(); // thread-safe
    private final Map<String, Double> prices = new ConcurrentHashMap<>();

    @Override
    public void addObserver(StockObserver observer) { observers.add(observer); }

    @Override
    public void removeObserver(StockObserver observer) { observers.remove(observer); }

    @Override
    public void notifyObservers(String symbol, double oldPrice, double newPrice) {
        observers.forEach(o -> o.onPriceChange(symbol, oldPrice, newPrice));
    }

    public void updatePrice(String symbol, double newPrice) {
        double oldPrice = prices.getOrDefault(symbol, 0.0);
        prices.put(symbol, newPrice);
        if (oldPrice != newPrice) {
            notifyObservers(symbol, oldPrice, newPrice);
        }
    }
}

// Concrete Observers
public class AlertObserver implements StockObserver {
    private final double threshold;
    public AlertObserver(double threshold) { this.threshold = threshold; }

    @Override
    public void onPriceChange(String symbol, double oldPrice, double newPrice) {
        if (Math.abs(newPrice - oldPrice) / oldPrice > threshold) {
            System.out.printf("ALERT: %s changed by >%.0f%%: %.2f -> %.2f%n",
                symbol, threshold * 100, oldPrice, newPrice);
        }
    }
}

public class TradingBot implements StockObserver {
    @Override
    public void onPriceChange(String symbol, double oldPrice, double newPrice) {
        if (newPrice < oldPrice * 0.95) {
            System.out.printf("BOT: Buying %s at %.2f (5%% dip)%n", symbol, newPrice);
        }
    }
}

// Usage
StockMarket market = new StockMarket();
market.addObserver(new AlertObserver(0.05));  // 5% threshold
market.addObserver(new TradingBot());

market.updatePrice("AAPL", 150.0);
market.updatePrice("AAPL", 142.0);  // triggers both observers
```

</details>

---

🔴 **S9. Explain the difference between compile-time and runtime polymorphism with a tricky example.**

```java
class A {
    public String name() { return "A"; }
}
class B extends A {
    @Override
    public String name() { return "B"; }
}
class C extends B {
    @Override
    public String name() { return "C"; }
}

A obj = new C();
System.out.println(obj.name());

// What if we add:
A obj2 = new B();
System.out.println(((A)obj2).name());
```

<details><summary>Click to reveal answer</summary>

**First call:** `"C"` — The actual object is `C`, so `C.name()` is called regardless of reference type `A`. Runtime polymorphism.

**Second call:** `"B"` — Casting `obj2` to `A` doesn't change the actual object — it's still a `B`. Casting only changes the **reference type** (what methods are visible at compile time). Runtime dispatch still uses the actual type `B`.

**Key insight:** In Java, casting a reference **never changes the object**. It only affects what methods are visible to the compiler. Runtime dispatch always uses the actual runtime type.

```java
// Compile-time: what methods are visible
// Runtime: which implementation is called

A ref = new C();  // reference type: A, actual type: C
ref.name();       // compile: sees A.name() (OK, A has it)
                  // runtime: calls C.name() → "C"

// ref.methodOnlyInC()  // COMPILE ERROR: A doesn't have this method
((C)ref).methodOnlyInC(); // OK after cast — compiler sees C type
```

</details>

---

🟡 **S10. What's the output? (Abstract class and interface interaction)**

```java
interface Greeting {
    default String greet() { return "Hello from Interface"; }
}

abstract class Base {
    public String greet() { return "Hello from Abstract Class"; }
}

class Derived extends Base implements Greeting {
    // Does NOT override greet()
}

public class Main {
    public static void main(String[] args) {
        Derived d = new Derived();
        System.out.println(d.greet());

        Greeting g = new Derived();
        System.out.println(g.greet());
    }
}
```

<details><summary>Click to reveal answer</summary>

Both print: `"Hello from Abstract Class"`

**Why?** Java's resolution rule: **Class methods win over interface default methods**. `Base.greet()` (concrete class method) takes priority over `Greeting.greet()` (default interface method) — even when accessed through the interface reference.

This applies to the second call too — the actual object is `Derived` which inherits from `Base`, and the class hierarchy wins.

</details>

---

🟡 **S11. Can you instantiate an abstract class? What about through anonymous class?**

<details><summary>Click to reveal answer</summary>

**Cannot** directly instantiate: `new AbstractClass()` → compile error.

**Can** instantiate via anonymous class:
```java
abstract class Shape {
    abstract double area();
    void describe() { System.out.println("Area: " + area()); }
}

// Anonymous class — provides implementation for abstract methods
Shape triangle = new Shape() {  // valid! Creates anonymous subclass
    @Override
    double area() { return 0.5 * 3 * 4; }
};
triangle.describe();  // "Area: 6.0"

// Anonymous classes are useful for:
// 1. Short-lived implementations (event listeners, etc.)
// 2. Callbacks before lambdas were available
// 3. Implementing interfaces inline:
Comparator<String> comp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) { return a.compareToIgnoreCase(b); }
};
// Modern equivalent (lambda):
Comparator<String> comp2 = (a, b) -> a.compareToIgnoreCase(b);
```

</details>

---

🔴 **S12. What is the Template Method Pattern and how does it use abstract classes?**

<details><summary>Click to reveal answer</summary>

**Template Method Pattern** defines the skeleton of an algorithm in a base class, deferring some steps to subclasses. The algorithm's structure is fixed; specific steps can be customized.

```java
// Abstract class defines the template
public abstract class DataMigration {

    // Template method — final so subclasses can't change the algorithm
    public final void migrate() {
        connect();
        readSourceData();
        transformData();
        validateData();
        writeTargetData();
        disconnect();
        logSummary();
    }

    // Hooks — can optionally override
    protected void connect() { System.out.println("Connecting to default source..."); }
    protected void disconnect() { System.out.println("Disconnecting..."); }
    protected void logSummary() { System.out.println("Migration complete."); }

    // Abstract steps — must override
    protected abstract List<Record> readSourceData();
    protected abstract List<Record> transformData();
    protected abstract void validateData();
    protected abstract void writeTargetData();
}

// Concrete implementation for MySQL to PostgreSQL migration
public class MySQLToPostgresMigration extends DataMigration {
    @Override
    protected List<Record> readSourceData() {
        System.out.println("Reading from MySQL...");
        return mysqlJdbc.query("SELECT * FROM orders");
    }

    @Override
    protected List<Record> transformData() {
        System.out.println("Transforming data types (MySQL → PostgreSQL)...");
        return convertTimestampFormats(sourceData);
    }

    @Override
    protected void validateData() {
        System.out.println("Validating referential integrity...");
    }

    @Override
    protected void writeTargetData() {
        System.out.println("Bulk inserting into PostgreSQL...");
        postgresJdbc.batchInsert(transformedData);
    }
}
```

**Why abstract class (not interface)?**
- Template method needs a concrete, non-overridable `migrate()` method
- Abstract class can have state (JDBC connections, counters)
- Abstract class can provide default implementations of hooks

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| Using `==` for object comparison | Override `equals()` and `hashCode()` |
| Overriding `equals()` without `hashCode()` | Always override both together |
| Deep inheritance hierarchies (5+ levels) | Prefer composition, shallow hierarchies |
| Making all fields public for simplicity | Encapsulate with private fields + methods |
| Calling overridable methods in constructors | Use `final` methods or factory methods |
| `UnsupportedOperationException` in overrides | Restructure the hierarchy to avoid LSP violations |
| Fat interfaces with unrelated methods | Apply ISP — split into focused interfaces |
| Missing `@Override` annotation | Always use `@Override` when overriding |

---

## ✅ Best Practices

- ✅ Follow SOLID principles — use them as a design checklist
- ✅ Prefer composition over inheritance by default
- ✅ Use interfaces to define contracts/capabilities
- ✅ Use abstract classes for shared behavior with state
- ✅ Always use `@Override` on overriding methods
- ✅ Override both `equals()` and `hashCode()` together
- ✅ Design for the "open for extension, closed for modification" goal
- ✅ Make fields private unless there's a strong reason not to
- ✅ Consider using records (Java 16+) for simple value objects

---

## 🔗 Related Sections

- [Chapter 1: Java Basics](./01-java-basics.md) — `static`, `final`, `this`, `super`
- [Chapter 3: Multithreading](./03-multithreading-and-concurrency.md) — thread-safe OOP design
- [Chapter 6: Collections](./06-collections-framework.md) — `Comparable`, `Comparator`
- [Chapter 8: Advanced Java](./08-advanced-java.md) — Generics, Records, Sealed Classes

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 1: Java Basics](./01-java-basics.md) | [Section README](./README.md) | [Chapter 3: Multithreading →](./03-multithreading-and-concurrency.md) |
