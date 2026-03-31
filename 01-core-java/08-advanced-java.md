# Chapter 8: Advanced Java

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- Generics: type parameters, type erasure, wildcards, bounded types
- Enums: advanced usage, abstract methods, EnumSet/EnumMap
- Inner classes: static nested, inner, anonymous, local
- `var` keyword (Java 10+) and type inference
- Records (Java 16+)
- Sealed classes and interfaces (Java 17+)
- Pattern matching (Java 16+)
- Java Platform Module System (JPMS, Java 9+)
- Text blocks (Java 15+)

---

## 🔑 Key Concepts at a Glance

```
Generics:
  T          → type parameter (any type)
  ? extends T → upper bounded wildcard (T or subtype) — Producer
  ? super T   → lower bounded wildcard (T or supertype) — Consumer
  Type erasure → generics removed at runtime (byte code has Object/bounds)

Modern Java (Java 9-21):
  var          → local variable type inference (Java 10)
  Records      → immutable data classes (Java 16)
  Sealed       → restrict class hierarchy (Java 17)
  Pattern Match→ instanceof with binding (Java 16)
  Text Blocks  → multiline strings (Java 15)
  JPMS         → explicit module declarations (Java 9)
```

---

## 🧠 Theory Questions

### Generics

---

🟡 **Q1. What are generics in Java and what problem do they solve?**

<details><summary>Click to reveal answer</summary>

Generics provide **compile-time type safety** and **eliminate the need for casting**.

**Without generics:**
```java
// Pre-Java 5 — all collections hold Object
List list = new ArrayList();
list.add("hello");
list.add(42);  // compiles fine!

String s = (String) list.get(0);  // explicit cast, may fail at runtime
String s2 = (String) list.get(1); // ClassCastException at runtime!
```

**With generics:**
```java
List<String> list = new ArrayList<>();
list.add("hello");
list.add(42);  // COMPILE ERROR — type-safe!

String s = list.get(0);  // no cast needed — compiler knows it's String
```

**Generic class:**
```java
public class Pair<A, B> {
    private final A first;
    private final B second;

    public Pair(A first, B second) {
        this.first = first;
        this.second = second;
    }

    public A getFirst() { return first; }
    public B getSecond() { return second; }

    public static <A, B> Pair<A, B> of(A a, B b) {
        return new Pair<>(a, b);
    }

    // Type-specific operations:
    public Pair<B, A> swap() {
        return new Pair<>(second, first);
    }
}

Pair<String, Integer> pair = Pair.of("hello", 42);
String s = pair.getFirst();    // no cast
Integer i = pair.getSecond();  // no cast
Pair<Integer, String> swapped = pair.swap();
```

**Generic method:**
```java
// Generic method (type parameter declared before return type)
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

int m = max(3, 7);          // T=Integer
String s = max("apple", "banana");  // T=String
```

</details>

---

🔴 **Q2. What is type erasure and what are its implications?**

<details><summary>Click to reveal answer</summary>

**Type erasure** is the process where the compiler removes all generic type information at compile time. The compiled bytecode uses `Object` (or the upper bound) instead of type parameters.

```java
// Source code:
public class Box<T> {
    private T value;
    public T getValue() { return value; }
    public void setValue(T value) { this.value = value; }
}

// After type erasure (what the JVM sees):
public class Box {
    private Object value;  // T → Object
    public Object getValue() { return value; }
    public void setValue(Object value) { this.value = value; }
}

// With bounded type parameter:
public <T extends Comparable<T>> T max(T a, T b) { ... }
// After erasure:
public Comparable max(Comparable a, Comparable b) { ... }  // T → upper bound
```

**Implications of type erasure:**

**1. Cannot use `instanceof` with generic type:**
```java
List<String> list = new ArrayList<>();
// list instanceof List<String>  // COMPILE ERROR
list instanceof List<?>          // OK — unbounded wildcard
list instanceof List             // OK — raw type
```

**2. Cannot create generic arrays:**
```java
T[] arr = new T[10];      // COMPILE ERROR
T[] arr = (T[]) new Object[10];  // OK (but unchecked cast warning)
```

**3. Cannot overload methods that differ only by type parameter:**
```java
// COMPILE ERROR: Both erase to the same method signature
void process(List<String> list) { ... }
void process(List<Integer> list) { ... }
// After erasure: both become process(List list)
```

**4. Generic type info at runtime (via reflection):**
```java
// Field/method type info IS preserved via reflection:
class Container<T> {
    List<T> items;
}
Field field = Container.class.getDeclaredField("items");
Type genericType = field.getGenericType();  // ParameterizedType: List<T>
// Libraries like Gson/Jackson use this to deserialize generics
```

**Workaround — pass Class<T>:**
```java
public <T> T parseJson(String json, Class<T> type) {
    return objectMapper.readValue(json, type);
}
User user = parseJson(json, User.class);
```

</details>

---

🟡 **Q3. What are wildcards in generics? Explain `?`, `? extends T`, and `? super T`.**

<details><summary>Click to reveal answer</summary>

**Unbounded wildcard `?`:**
```java
// Can read from any List (as Object), cannot write (except null)
public static void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);  // OK — item is Object
    }
    // list.add("hello");  // COMPILE ERROR — unknown type
    list.add(null);        // null is OK
}
```

**Upper bounded wildcard `? extends T` (Producer):**
```java
// "? extends Number" = Number or any subtype (Integer, Double, etc.)
public static double sum(List<? extends Number> numbers) {
    double total = 0;
    for (Number n : numbers) {
        total += n.doubleValue();  // OK — can call Number methods
    }
    // numbers.add(42);  // COMPILE ERROR — can't add (might be List<Double>!)
    return total;
}

List<Integer> ints = List.of(1, 2, 3);
List<Double> doubles = List.of(1.5, 2.5, 3.5);
sum(ints);     // works!
sum(doubles);  // works!
```

**Lower bounded wildcard `? super T` (Consumer):**
```java
// "? super Integer" = Integer or any supertype (Number, Object)
public static void addNumbers(List<? super Integer> list) {
    for (int i = 1; i <= 5; i++) {
        list.add(i);  // OK — can add Integer (subtype of bound)
    }
    // Object obj = list.get(0);  // Can only get as Object
}

List<Number> numbers = new ArrayList<>();
List<Object> objects = new ArrayList<>();
addNumbers(numbers);  // works!
addNumbers(objects);  // works!
```

**PECS: Producer Extends, Consumer Super**
```
If you are getting items OUT of a collection (producing): use ? extends T
If you are putting items IN to a collection (consuming):  use ? super T
If you are doing both:                                    use exact type T

// Classic example — Collections.copy:
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) {  // src produces (extends)
        dest.add(item);   // dest consumes (super)
    }
}
```

> 💡 **Interviewer Tip:** PECS is a commonly tested concept. Practice saying: "upper bounded for reading from a collection, lower bounded for writing to a collection."

</details>

---

### Enums

---

🟡 **Q4. What are the advanced features of Java enums?**

<details><summary>Click to reveal answer</summary>

Enums in Java are **full classes** — they can have fields, methods, and implement interfaces.

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6),
    EARTH  (5.976e+24, 6.37814e6),
    MARS   (6.421e+23, 3.3972e6);

    private final double mass;   // instance field
    private final double radius;

    // Constructor (always private implicitly)
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    static final double G = 6.67300E-11;

    // Instance method
    double surfaceGravity() {
        return G * mass / (radius * radius);
    }

    double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}

// Usage:
double earthWeight = 75.0;
double mass = earthWeight / Planet.EARTH.surfaceGravity();
for (Planet p : Planet.values()) {
    System.out.printf("Weight on %s = %6.2f%n", p, p.surfaceWeight(mass));
}
```

**Abstract methods in enums:**
```java
public enum Operation {
    ADD {
        @Override
        public double apply(double x, double y) { return x + y; }
        @Override
        public String symbol() { return "+"; }
    },
    SUBTRACT {
        @Override
        public double apply(double x, double y) { return x - y; }
        @Override
        public String symbol() { return "-"; }
    },
    MULTIPLY {
        @Override
        public double apply(double x, double y) { return x * y; }
        @Override
        public String symbol() { return "*"; }
    };

    public abstract double apply(double x, double y);
    public abstract String symbol();

    @Override
    public String toString() {
        return symbol();
    }
}

// Usage:
double result = Operation.ADD.apply(3, 4);  // 7.0
```

**Enum implementing interface:**
```java
public interface Describable {
    String describe();
}

public enum Season implements Describable {
    SPRING, SUMMER, FALL, WINTER;

    @Override
    public String describe() {
        return switch (this) {
            case SPRING -> "Warm and rainy";
            case SUMMER -> "Hot and sunny";
            case FALL -> "Cool and colorful";
            case WINTER -> "Cold and snowy";
        };
    }
}
```

**Enum built-in methods:**
```java
Season.SUMMER.name()     // "SUMMER" (always the enum constant name)
Season.SUMMER.ordinal()  // 1 (0-based index)
Season.valueOf("FALL")   // Season.FALL (throws IllegalArgumentException if not found)
Season.values()          // Season[] of all constants

// EnumSet and EnumMap — optimized for enums
EnumSet<Season> warmSeasons = EnumSet.of(Season.SPRING, Season.SUMMER);
EnumSet<Season> allSeasons = EnumSet.allOf(Season.class);
EnumSet<Season> except = EnumSet.complementOf(warmSeasons);  // FALL, WINTER

EnumMap<Season, String> descriptions = new EnumMap<>(Season.class);
descriptions.put(Season.SUMMER, "Hot");
```

</details>

---

### Inner Classes

---

🟡 **Q5. What are the different types of inner classes in Java?**

<details><summary>Click to reveal answer</summary>

**1. Static Nested Class:**
```java
public class Outer {
    private static int staticField = 10;
    private int instanceField = 20;

    public static class StaticNested {
        // Can access Outer's static members
        void display() { System.out.println(staticField); }
        // Cannot access instanceField without an Outer instance
    }
}

// Create without Outer instance:
Outer.StaticNested nested = new Outer.StaticNested();
```

**2. Inner Class (Non-static):**
```java
public class Outer {
    private int x = 10;

    public class Inner {
        // Implicitly holds reference to Outer.this
        void display() { System.out.println(x); }  // can access Outer's instance members
        void explicit() { System.out.println(Outer.this.x); }
    }
}

// Must create through Outer instance:
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
inner.display();
```

**3. Local Class:**
```java
public void doSomething() {
    int localVar = 10;  // must be effectively final to use in local class

    class LocalHelper {
        void help() { System.out.println(localVar); }
    }

    new LocalHelper().help();
}
```

**4. Anonymous Class:**
```java
// One-off implementation without naming the class
Comparator<String> byLength = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return Integer.compare(a.length(), b.length());
    }
};

// Equivalent lambda (Java 8+):
Comparator<String> byLength2 = (a, b) -> Integer.compare(a.length(), b.length());

// Anonymous class with state (can't use lambda):
Runnable counter = new Runnable() {
    private int count = 0;
    @Override
    public void run() {
        count++;
        System.out.println("Count: " + count);
    }
};
```

**Memory leak concern with inner classes:**
```java
// Inner class holds implicit reference to outer class
// If inner class escapes (e.g., passed to long-lived thread), outer class can't be GC'd

// Example: Android Activity leak
class Activity {
    class AsyncTask extends Thread {  // holds reference to Activity!
        public void run() { /* ... long operation */ }
    }
}
// If Activity is destroyed but AsyncTask is still running, Activity leaks!

// Fix: Use static nested class + weak reference
static class SafeAsyncTask extends Thread {
    private final WeakReference<Activity> activityRef;
    SafeAsyncTask(Activity a) { activityRef = new WeakReference<>(a); }
}
```

</details>

---

### var Keyword

---

🟢 **Q6. What is `var` in Java and what are its rules?**

<details><summary>Click to reveal answer</summary>

`var` (Java 10+) is local variable type inference — the compiler infers the type from the initializer.

```java
// Without var
ArrayList<Map<String, List<Integer>>> data = new ArrayList<Map<String, List<Integer>>>();

// With var — less verbose
var data = new ArrayList<Map<String, List<Integer>>>();

// More examples
var list = new ArrayList<String>();           // ArrayList<String>
var map = new HashMap<String, Integer>();     // HashMap<String, Integer>
var text = "Hello World";                     // String
var i = 42;                                   // int
var d = 3.14;                                 // double
var entry = map.entrySet().iterator().next(); // Map.Entry<String, Integer>

// With streams
var result = list.stream()
    .filter(s -> s.startsWith("A"))
    .collect(Collectors.toList());  // List<String>
```

**Rules — where `var` CAN be used:**
```java
var x = 10;                    // ✅ local variable with initializer
for (var item : list) { }     // ✅ enhanced for loop
for (var i = 0; i < 10; i++) { }  // ✅ for loop
try (var in = new FileInputStream("file")) { }  // ✅ try-with-resources
```

**Rules — where `var` CANNOT be used:**
```java
class Foo {
    var x = 10;           // ❌ class field — not allowed

    var method() { }      // ❌ return type — not allowed

    void method(var x) { }  // ❌ method parameter — not allowed
}

var x;             // ❌ no initializer
var x = null;      // ❌ null initializer (type unknown)
var x = { 1, 2 }; // ❌ array initializer without type

// Lambda with var (Java 11+):
Consumer<String> c = (var s) -> System.out.println(s);  // ✅ allows annotations
Consumer<String> c2 = (@NonNull var s) -> System.out.println(s);  // ✅ with annotation
```

🚨 **Common Mistake:** `var` makes code less readable when the type isn't obvious from the right-hand side. Prefer explicit types when the type is not immediately clear.

</details>

---

### Records

---

🟡 **Q7. What are Java Records and when should you use them?**

<details><summary>Click to reveal answer</summary>

Records (Java 16+) are concise, immutable data classes. The compiler automatically generates `constructor`, `getters`, `equals()`, `hashCode()`, and `toString()`.

```java
// Traditional immutable class (boilerplate):
public final class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) { this.x = x; this.y = y; }

    public int x() { return x; }
    public int y() { return y; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }

    @Override
    public int hashCode() { return Objects.hash(x, y); }

    @Override
    public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}

// Record — equivalent, one line:
public record Point(int x, int y) {}

// Usage:
Point p1 = new Point(3, 4);
p1.x()           // 3 (accessor, not getX())
p1.y()           // 4
p1.toString()    // "Point[x=3, y=4]"
p1.equals(new Point(3, 4))  // true
```

**Record features:**
```java
public record Person(String name, int age) {

    // Compact canonical constructor (validate/normalize)
    public Person {
        Objects.requireNonNull(name, "name cannot be null");
        if (age < 0) throw new IllegalArgumentException("age must be non-negative");
        name = name.trim();  // can modify parameter before assignment
    }

    // Custom constructor (must delegate to canonical)
    public Person(String name) {
        this(name, 0);
    }

    // Instance methods
    public boolean isAdult() { return age >= 18; }

    // Static methods
    public static Person unknown() { return new Person("Unknown", -1); }

    // Can have static fields (not instance fields)
    static final String DEFAULT_NAME = "John";
}

// Records can implement interfaces
public record Temperature(double celsius) implements Comparable<Temperature> {
    @Override
    public int compareTo(Temperature other) {
        return Double.compare(this.celsius, other.celsius);
    }

    public double fahrenheit() { return celsius * 9 / 5 + 32; }
}

// Records as DTOs in Spring:
public record UserRequest(
    @NotBlank String username,
    @Email String email,
    @Min(18) int age
) {}

@PostMapping("/users")
public ResponseEntity<Void> createUser(@Valid @RequestBody UserRequest req) { ... }
```

**What records can't do:**
- Cannot extend other classes (implicitly extend `java.lang.Record`)
- Cannot have non-static instance fields (except component fields)
- Components are final (immutable)
- Cannot declare `abstract` records

</details>

---

### Sealed Classes

---

🔴 **Q8. What are sealed classes and interfaces in Java?**

<details><summary>Click to reveal answer</summary>

Sealed classes (Java 17+) restrict which classes can extend or implement them, enabling **exhaustive pattern matching**.

```java
// Sealed class — only these classes can extend it:
public sealed class Shape
    permits Circle, Rectangle, Triangle {}

// Each permitted class must be final, sealed, or non-sealed
public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double area() { return Math.PI * radius * radius; }
}

public final class Rectangle extends Shape {
    private final double width, height;
    public Rectangle(double width, double height) { this.width = width; this.height = height; }
    public double area() { return width * height; }
}

public non-sealed class Triangle extends Shape {
    // non-sealed: can be extended by any class
    private final double base, height;
    public Triangle(double base, double height) { this.base = base; this.height = height; }
    public double area() { return 0.5 * base * height; }
}
```

**Power: Exhaustive pattern matching (Java 21+):**
```java
// Without sealed: switch needs default
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t -> 0.5 * t.base() * t.height();
    default -> throw new IllegalArgumentException("Unknown shape");
};

// With sealed: no default needed — compiler knows all possibilities
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
    // No default needed — compiler verifies exhaustiveness!
};
```

**Sealed interfaces:**
```java
// Sealed interface for ADT (Algebraic Data Type) — Result type
public sealed interface Result<T>
    permits Result.Success, Result.Failure {}

public record Success<T>(T value) implements Result<T> {}
public record Failure<T>(String error, Throwable cause) implements Result<T> {}

// Usage:
Result<String> result = callService();
String message = switch (result) {
    case Success<String> s -> "Got: " + s.value();
    case Failure<String> f -> "Error: " + f.error();
};
```

**Use cases:**
- Domain model hierarchies (payment types, event types)
- Error handling (Result types)
- AST nodes (compiler/parser)
- State machine states

</details>

---

### Pattern Matching

---

🟡 **Q9. What is pattern matching for `instanceof` (Java 16+)?**

<details><summary>Click to reveal answer</summary>

Pattern matching eliminates the verbose cast-after-instanceof pattern.

```java
// Old way:
if (obj instanceof String) {
    String s = (String) obj;  // redundant cast!
    System.out.println(s.length());
}

// New way (Java 16+):
if (obj instanceof String s) {  // binding variable 's'
    System.out.println(s.length());  // s is String, no cast!
}

// Works with null safety (instanceof always false for null):
Object obj = null;
if (obj instanceof String s) {  // false, no NPE
    System.out.println(s);
}

// In switch (Java 21):
Object obj = ...;
String result = switch (obj) {
    case Integer i -> "int: " + i;
    case String s when s.length() > 5 -> "long string: " + s;  // guarded pattern
    case String s -> "short string: " + s;
    case null -> "null";
    default -> "other: " + obj;
};

// With records and deconstruction (Java 21+):
record Point(int x, int y) {}

Object o = new Point(3, 4);
if (o instanceof Point(int x, int y)) {
    System.out.println("x=" + x + " y=" + y);  // deconstruct record components
}
```

</details>

---

### Java Module System (JPMS)

---

🔴 **Q10. What is the Java Platform Module System (JPMS) and why was it introduced?**

<details><summary>Click to reveal answer</summary>

JPMS (Java 9+) introduces strong encapsulation at the package level, improving on classpath-based isolation.

**Problems before JPMS:**
- Classpath hell (JAR conflicts, version incompatibilities)
- No way to enforce `public` means "public within my library only"
- The entire JDK was available to all code (security issue)
- Hard to detect missing dependencies at compile time

**Module declaration (`module-info.java`):**
```java
// src/com.myapp/module-info.java
module com.myapp {
    requires java.base;          // implicit — always available
    requires java.sql;           // uses JDBC
    requires spring.context;     // uses Spring

    requires transitive com.mylib;  // any module that requires com.myapp
                                    // also gets com.mylib transitively

    exports com.myapp.api;       // public to all modules
    exports com.myapp.internal to com.myapp.tests;  // qualified export

    opens com.myapp.dto;         // allows deep reflection (for Jackson, JPA)
    opens com.myapp.dto to com.fasterxml.jackson.databind;  // qualified open

    uses com.myapp.api.Plugin;   // declares use of a service
    provides com.myapp.api.Plugin with com.myapp.impl.MyPlugin;  // provides implementation
}
```

**Module types:**
```
Named module:   has module-info.java — fully module-aware
Automatic module: JAR on module path without module-info.java
                  (name derived from JAR filename, exports everything)
Unnamed module: classpath — all classpath code — legacy support
```

**Implications:**
```java
// BEFORE JPMS: could access internal JDK classes
com.sun.misc.Unsafe unsafe = Unsafe.getUnsafe();  // worked pre-Java 9

// AFTER JPMS: module encapsulation prevents this
// InaccessibleObjectException: module java.base does not open java.lang to ...

// Libraries that use reflection need: --add-opens at startup
// spring.boot.jar requires:
// --add-opens java.base/java.lang=ALL-UNNAMED

// Or in module-info.java:
opens com.mypackage to hibernate.core;  // allow Hibernate to reflect on entities
```

**Key commands:**
```bash
# Compile with module path
javac --module-path mods -d out src/module-info.java src/**/*.java

# Run
java --module-path mods --module com.myapp/com.myapp.Main

# List modules
java --list-modules

# Module dependencies
jdeps --module-path mods --module com.myapp
```

> 💡 **Interviewer Tip:** In practice, most Spring Boot applications still use classpath (unnamed module). JPMS is used mainly for JDK structuring and library development. Knowing the concept and why it was introduced is more important than detailed usage.

</details>

---

### Text Blocks

---

🟢 **Q11. What are text blocks and how do they work?**

<details><summary>Click to reveal answer</summary>

Text blocks (Java 15+) are multi-line string literals with no escape characters needed.

```java
// Old way — hard to read
String json = "{\n" +
    "  \"name\": \"Alice\",\n" +
    "  \"age\": 30\n" +
    "}";

// Text block — much cleaner
String json = """
    {
      "name": "Alice",
      "age": 30
    }
    """;
// Incidental whitespace (the 4 spaces indent) is stripped automatically
// Closing """ position determines the indent to strip

// SQL
String sql = """
    SELECT u.id, u.name, u.email
    FROM users u
    JOIN orders o ON u.id = o.user_id
    WHERE o.status = 'ACTIVE'
    ORDER BY u.name
    """;

// HTML
String html = """
    <html>
        <body>
            <p>Hello, %s!</p>
        </body>
    </html>
    """.formatted("World");  // formatted() works with text blocks

// Escape sequences in text blocks:
String escapes = """
    Line 1\nLine 2    ← \n still works
    Line 3\s           ← \s prevents trailing whitespace stripping
    Line 4\\          ← \\ literal backslash
    """;

// No trailing newline:
String noTrailing = """
    hello""";  // closing """ on same line as last content
```

</details>

---

## 🎭 Scenario-Based Questions

---

🔴 **S1. What happens with type erasure here?**

```java
public class Container<T> {
    private T value;

    public boolean isString() {
        return value instanceof String;  // A
    }

    public void checkType() {
        if (value instanceof T) { ... }  // B — compile error?
    }

    @SuppressWarnings("unchecked")
    public T[] createArray(int size) {
        return (T[]) new Object[size];   // C — safe?
    }
}
```

<details><summary>Click to reveal answer</summary>

- **A:** `value instanceof String` — **compiles and works**. `instanceof` checks the actual runtime type of the object, not the generic type. Fine.

- **B:** `value instanceof T` — **COMPILE ERROR**. `T` is erased to `Object` at runtime. Cannot check `instanceof` with a type parameter. Compiler catches this.

- **C:** `(T[]) new Object[size]` — **compiles with warning**, works at runtime as long as the array stays as `Object[]` internally (never returned as a typed array to external code). If caller tries to store it as `T[]` where T is known, they'll get `ClassCastException`.

**Correct workaround for creating generic arrays:**
```java
public class Container<T> {
    private final Class<T> type;

    public Container(Class<T> type) { this.type = type; }

    @SuppressWarnings("unchecked")
    public T[] createArray(int size) {
        return (T[]) Array.newInstance(type, size);  // creates actual T[] via reflection
    }

    public boolean isInstance(Object obj) {
        return type.isInstance(obj);  // runtime check
    }
}

Container<String> c = new Container<>(String.class);
String[] arr = c.createArray(10);  // String[] — safe!
```

</details>

---

🟡 **S2. What does this print? (Generics and inheritance)**

```java
List<Integer> ints = new ArrayList<>();
ints.add(1); ints.add(2);

List<Number> nums = ints;  // A — compiles?
nums.add(3.14);             // B — compiles?
```

<details><summary>Click to reveal answer</summary>

- **A:** **COMPILE ERROR** — `List<Integer>` is NOT a `List<Number>` even though `Integer extends Number`. Generics are **invariant**. If this were allowed:
  ```java
  List<Number> nums = ints;  // if allowed
  nums.add(3.14);            // would add Double to a List<Integer>!
  ints.get(2);               // ClassCastException! List<Integer> contains a Double
  ```

- **B:** Would compile if A compiled, but causes type safety violation.

**Fix using wildcards:**
```java
List<? extends Number> nums = ints;  // OK — covariant read-only view
nums.add(3.14);                       // COMPILE ERROR — can't add (PECS: extends = Producer only)

Number first = nums.get(0);           // OK — reads as Number

// To both read and write:
void process(List<Number> list) { ... }  // requires exact type
process(ints);                           // COMPILE ERROR

// Or use bounded wildcards:
void sumAll(List<? extends Number> list) {  // reads, doesn't write
    list.stream().mapToDouble(Number::doubleValue).sum();
}
sumAll(ints);  // OK!
```

</details>

---

🟡 **S3. Implement a generic `Optional`-like type using records and sealed classes.**

<details><summary>Click to reveal answer</summary>

```java
// Sealed interface for type-safe optional
public sealed interface Maybe<T> permits Maybe.Some, Maybe.None {

    record Some<T>(T value) implements Maybe<T> {
        public Some {
            Objects.requireNonNull(value, "value cannot be null");
        }
    }

    record None<T>() implements Maybe<T> {}

    // Factory methods
    static <T> Maybe<T> of(T value) {
        return value != null ? new Some<>(value) : new None<>();
    }

    static <T> Maybe<T> empty() { return new None<>(); }

    // Pattern matching behavior
    default T orElse(T defaultValue) {
        return switch (this) {
            case Some<T> s -> s.value();
            case None<T> n -> defaultValue;
        };
    }

    default <U> Maybe<U> map(java.util.function.Function<T, U> mapper) {
        return switch (this) {
            case Some<T> s -> Maybe.of(mapper.apply(s.value()));
            case None<T> n -> Maybe.empty();
        };
    }

    default boolean isPresent() {
        return this instanceof Some<T>;
    }
}

// Usage:
Maybe<String> name = Maybe.of("Alice");
Maybe<String> noName = Maybe.empty();

String result = name.map(String::toUpperCase).orElse("UNKNOWN");  // "ALICE"
String result2 = noName.orElse("UNKNOWN");  // "UNKNOWN"

// Pattern matching in switch:
switch (name) {
    case Maybe.Some<String> s -> System.out.println("Got: " + s.value());
    case Maybe.None<String> n -> System.out.println("Nothing");
}
```

</details>

---

🟡 **S4. What's wrong with this generic method?**

```java
public static <T> void swap(T[] arr, int i, int j) {
    T temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
}

// Called as:
int[] ints = {1, 2, 3};
swap(ints, 0, 1);  // Works?
```

<details><summary>Click to reveal answer</summary>

**COMPILE ERROR** for `swap(ints, 0, 1)`.

`int[]` is a primitive array. Generics only work with reference types. `T` cannot be `int` — it would need to be `Integer`.

**Fix 1: Use Integer array:**
```java
Integer[] ints = {1, 2, 3};
swap(ints, 0, 1);  // OK
```

**Fix 2: Overload for primitive arrays:**
```java
public static void swap(int[] arr, int i, int j) {
    int temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
}
```

**Fix 3: Generic version works for object arrays:**
```java
String[] strs = {"a", "b", "c"};
swap(strs, 0, 1);  // OK — String is a reference type
```

**Why can't generics work with primitives?**
Type erasure: `T` is replaced with `Object` at runtime. `int` is not an `Object`. This is why Java has wrapper classes (`Integer`, `Long`, etc.) and why generic collections can't hold primitive arrays.

</details>

---

🔴 **S5. Implement a type-safe heterogeneous container.**

<details><summary>Click to reveal answer</summary>

Joshua Bloch's "type-safe heterogeneous container" pattern uses `Class<T>` objects as keys.

```java
public class TypeSafeMap {
    private final Map<Class<?>, Object> map = new HashMap<>();

    public <T> void put(Class<T> type, T value) {
        map.put(Objects.requireNonNull(type), type.cast(value));
    }

    @SuppressWarnings("unchecked")
    public <T> T get(Class<T> type) {
        return type.cast(map.get(type));  // type-safe cast via Class.cast()
    }

    public <T> boolean contains(Class<T> type) {
        return map.containsKey(type);
    }
}

// Usage:
TypeSafeMap container = new TypeSafeMap();
container.put(String.class, "Hello");
container.put(Integer.class, 42);
container.put(Boolean.class, true);

String s = container.get(String.class);   // "Hello" — no cast!
Integer i = container.get(Integer.class); // 42 — type-safe!

// For generic types, use TypeToken pattern (Guava):
// TypeToken<List<String>> token = new TypeToken<List<String>>() {};
```

</details>

---

🟡 **S6. What is the output of this enum code?**

```java
enum Status {
    ACTIVE, INACTIVE, PENDING;

    static {
        System.out.println("Status enum loaded");
    }

    Status() {
        System.out.println("Creating: " + name());
    }
}

public class Main {
    public static void main(String[] args) {
        System.out.println("Before first use");
        Status s = Status.ACTIVE;
        System.out.println("After first use: " + s);
        System.out.println(Status.ACTIVE == Status.ACTIVE);
    }
}
```

<details><summary>Click to reveal answer</summary>

```
Before first use
Creating: ACTIVE
Creating: INACTIVE
Creating: PENDING
Status enum loaded
After first use: ACTIVE
true
```

**Explanation:**
1. "Before first use" — printed before enum class is loaded
2. Accessing `Status.ACTIVE` triggers class loading
3. Enum constants initialized in declaration order: ACTIVE, INACTIVE, PENDING (constructor prints each)
4. Static block runs after all constants initialized
5. "After first use: ACTIVE" — `Status.toString()` returns the name
6. `Status.ACTIVE == Status.ACTIVE` is `true` — enum constants are singletons (only one instance per constant)

Enum singletons are guaranteed by JVM class loading — making enum the safest singleton pattern.

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| `instanceof` with generic type parameter | Use `Class<T>` for runtime type checks |
| Creating generic arrays `new T[n]` | Use `(T[]) new Object[n]` or `Array.newInstance()` |
| Assigning `List<Integer>` to `List<Number>` | Use `List<? extends Number>` |
| Forgetting PECS — writing to `? extends` | `? extends` for reading, `? super` for writing |
| Using `var` when type isn't obvious | Use explicit type for clarity |
| Records with mutable component types | Record is shallow-immutable; deep immutability needs defensive copies |
| `name()` vs `toString()` on enums | `name()` always returns the enum constant name; `toString()` can be overridden |
| Using ordinal for persistence | Use `name()` — ordinal changes if enum order changes |
| Not handling `null` in switch with sealed classes | Add `case null` or check before switch |
| Missing module requires | Add `requires` directive and recompile |

---

## ✅ Best Practices

- ✅ Use records for simple, immutable data transfer objects
- ✅ Use sealed classes to model closed type hierarchies
- ✅ Use PECS for generic method parameters
- ✅ Pass `Class<T>` tokens when you need runtime type info with generics
- ✅ Use `EnumSet`/`EnumMap` for enum-keyed collections
- ✅ Use `var` only when the type is obvious from the right-hand side
- ✅ Prefer named constants in enums over `ordinal()`
- ✅ Use text blocks for multi-line string literals (SQL, JSON, HTML)
- ✅ Use pattern matching `instanceof` instead of explicit casts
- ✅ Use `@SuppressWarnings("unchecked")` with explanation when necessary

---

## 🔗 Related Sections

- [Chapter 2: OOP](./02-oops.md) — abstract classes, interfaces, polymorphism
- [Chapter 6: Collections](./06-collections-framework.md) — generic collections, wildcards in practice
- [Chapter 1: Java Basics](./01-java-basics.md) — `var`, type casting

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 7: Strings & Regex](./07-strings-and-regex.md) | [Section README](./README.md) | *(Section Complete)* |
