# Chapter 1: Java Basics

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- JVM, JRE, and JDK architecture
- Java memory model: Stack vs Heap
- Primitive data types and wrapper classes
- Type casting (widening/narrowing/autoboxing)
- Operators and operator precedence
- Control flow (`if/else`, `switch`, loops)
- Garbage collection basics
- `static`, `final`, `this`, `super` keywords
- Pass-by-value vs pass-by-reference
- Java compilation and execution lifecycle

---

## 🔑 Key Concepts at a Glance

```
Java Source (.java)
       ↓ javac (compiler)
   Bytecode (.class)
       ↓ JVM (ClassLoader → Bytecode Verifier → Execution Engine)
  Machine Code (JIT compiled)
```

### JVM Memory Areas

```
┌─────────────────────────────────────────────────────────┐
│                        JVM Memory                        │
├─────────────────┬───────────────────────────────────────┤
│   Method Area   │  Class metadata, static vars, constants│
├─────────────────┼───────────────────────────────────────┤
│      Heap       │  Objects, instance variables           │
│  ┌───────────┐  │  ┌──────────────────────────────────┐ │
│  │  Young Gen│  │  │ Eden + S0 + S1 (Minor GC)        │ │
│  ├───────────┤  │  ├──────────────────────────────────┤ │
│  │  Old Gen  │  │  │ Long-lived objects (Major GC)    │ │
│  └───────────┘  │  └──────────────────────────────────┘ │
├─────────────────┼───────────────────────────────────────┤
│  Stack (per     │  Frames: local vars, operand stack,   │
│   thread)       │  method calls (LIFO)                  │
├─────────────────┼───────────────────────────────────────┤
│   PC Register   │  Current instruction per thread       │
├─────────────────┼───────────────────────────────────────┤
│  Native Method  │  JNI native method stack              │
│     Stack       │                                       │
└─────────────────┴───────────────────────────────────────┘
```

---

## 🧠 Theory Questions

### JVM / JRE / JDK Architecture

---

🟢 **Q1. What is the difference between JDK, JRE, and JVM?**

<details><summary>Click to reveal answer</summary>

| Component | Full Name | Contains | Purpose |
|-----------|-----------|---------|---------|
| **JVM** | Java Virtual Machine | Execution engine, GC, class loader | Runs bytecode |
| **JRE** | Java Runtime Environment | JVM + standard libraries | Run Java programs |
| **JDK** | Java Development Kit | JRE + compiler (`javac`) + tools | Develop Java programs |

**Relationship:** `JDK ⊃ JRE ⊃ JVM`

```
JDK
 ├── javac (compiler)
 ├── javadoc, jar, jdb, ...
 └── JRE
      ├── Java Standard Library (rt.jar / modules)
      └── JVM
           ├── Class Loader Subsystem
           ├── Runtime Data Areas
           └── Execution Engine (Interpreter + JIT + GC)
```

> 💡 **Interviewer Tip:** Interviewers often ask this to gauge fundamental understanding. Always clarify that JVM is platform-specific (different JVMs for different OS) while bytecode is platform-independent — this is the "Write Once, Run Anywhere" promise.

</details>

---

🟢 **Q2. What is bytecode? Why is it important?**

<details><summary>Click to reveal answer</summary>

Bytecode is the **intermediate representation** of Java programs compiled by `javac`. It is:
- **Platform-independent**: Not tied to any OS or CPU architecture
- **Not machine code**: Requires the JVM to interpret or JIT-compile it
- Stored in `.class` files

**Lifecycle:**
```
Developer writes → MyApp.java
javac compiles  → MyApp.class (bytecode)
JVM loads       → ClassLoader reads .class
JVM executes    → Interpreter or JIT compiler converts to native machine code
```

This is why Java achieves "Write Once, Run Anywhere" — the same `.class` file runs on Windows JVM, Linux JVM, or macOS JVM.

</details>

---

🟡 **Q3. Explain the role of the ClassLoader in the JVM.**

<details><summary>Click to reveal answer</summary>

The **ClassLoader** is responsible for dynamically loading `.class` files into the JVM at runtime. It operates in three phases: **Loading → Linking → Initialization**.

**ClassLoader Hierarchy:**
```
Bootstrap ClassLoader       → loads core Java classes (java.lang.*, rt.jar)
    ↑ parent
Extension ClassLoader       → loads ext/ directory (jre/lib/ext)
    ↑ parent
Application ClassLoader     → loads classpath (your classes, libraries)
    ↑ parent
Custom ClassLoader          → user-defined class loading
```

**Delegation Model (Parent-First):**
When a class needs to be loaded, the ClassLoader first delegates to its parent. Only if the parent cannot find it does the child attempt to load it. This prevents core Java classes from being overridden.

```java
// You can create a custom ClassLoader
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        return defineClass(name, classData, 0, classData.length);
    }
}
```

🚨 **Common Mistake:** Confusing `NoClassDefFoundError` (class existed at compile time, not at runtime) with `ClassNotFoundException` (class not found at all).

</details>

---

🟡 **Q4. What is the JIT compiler and how does it improve performance?**

<details><summary>Click to reveal answer</summary>

**JIT (Just-In-Time) compiler** compiles frequently-executed bytecode ("hot spots") into native machine code at runtime, caching the result for subsequent calls.

**Without JIT (Pure Interpretation):**
- Each bytecode instruction is interpreted one at a time
- Slow for repeated code paths

**With JIT:**
```
First N executions: interpreted (profiling occurs)
After threshold:    JIT compiles hot method → native code
Subsequent calls:   execute native code directly (much faster)
```

**HotSpot JVM has two compilers:**
- **C1 (Client Compiler):** Fast compilation, fewer optimizations (good for startup)
- **C2 (Server Compiler):** Slower compilation, aggressive optimizations (good for long-running apps)

**JVM flags:**
```bash
-XX:+PrintCompilation   # Show JIT compilation events
-XX:CompileThreshold=N  # Set invocation threshold (default: 10,000)
```

</details>

---

🔴 **Q5. Describe the Java Memory Model (JMM) and its happens-before guarantees.**

<details><summary>Click to reveal answer</summary>

The **Java Memory Model** defines how threads interact through memory and what behaviors are permitted during concurrent execution.

**Key Problem:** Without a memory model, JIT compilers and CPUs can reorder instructions for optimization, causing unexpected behavior in multithreaded programs.

**Happens-Before Relationships (guarantee visibility):**
1. **Program order rule:** Each action in a thread happens-before every action later in that thread
2. **Monitor lock rule:** `unlock()` happens-before subsequent `lock()` of the same monitor
3. **Volatile variable rule:** A write to a `volatile` field happens-before every subsequent read of that field
4. **Thread start rule:** `Thread.start()` happens-before any action in the started thread
5. **Thread join rule:** All actions in a thread happen-before `Thread.join()` returns
6. **Transitivity:** If A happens-before B, and B happens-before C, then A happens-before C

```java
// Without happens-before guarantee:
int x = 0;
boolean flag = false;

// Thread 1
x = 42;
flag = true;   // CPU may reorder: flag=true before x=42

// Thread 2
if (flag) {
    System.out.println(x);  // May print 0 due to reordering!
}

// With volatile (establishes happens-before):
volatile boolean flag = false;
// Now x=42 happens-before flag=true write, which happens-before flag read in Thread 2
```

> 💡 **Interviewer Tip:** This is an advanced topic. Mentioning happens-before in the context of `volatile` and `synchronized` shows deep JMM understanding and will impress senior interviewers.

</details>

---

### Memory Model: Stack vs Heap

---

🟢 **Q6. What is the difference between stack and heap memory in Java?**

<details><summary>Click to reveal answer</summary>

| Aspect | Stack | Heap |
|--------|-------|------|
| **Contents** | Local variables, method call frames, references | Objects, instance variables, arrays |
| **Allocation** | Compile-time (fixed size per frame) | Runtime (dynamic) |
| **Access** | LIFO (Last In, First Out) | Random access via reference |
| **Thread safety** | Each thread has its own stack | Shared across all threads |
| **Lifetime** | Until method returns | Until GC collects |
| **Size** | Small (typically 512KB–2MB) | Large (configurable, `-Xmx`) |
| **Overflow error** | `StackOverflowError` | `OutOfMemoryError` |
| **Speed** | Faster (simple pointer movement) | Slower (GC involved) |

```java
public void foo() {
    int x = 10;           // x lives on STACK
    String s = "hello";   // reference 's' on STACK, "hello" object on HEAP (String pool)
    Object obj = new Object(); // reference 'obj' on STACK, Object on HEAP
}
// When foo() returns: x, s, obj references are popped from stack
// The heap objects become eligible for GC if no other references exist
```

</details>

---

🟡 **Q7. What is the String Pool (String Intern Pool) and where does it live?**

<details><summary>Click to reveal answer</summary>

The **String Pool** is a special area in the **Heap** (since Java 7; was in PermGen/Method Area before Java 7) where the JVM stores interned string literals.

**How it works:**
```java
String s1 = "hello";          // Creates "hello" in String Pool
String s2 = "hello";          // Reuses same "hello" from pool
String s3 = new String("hello"); // Creates NEW object on heap (NOT in pool)

System.out.println(s1 == s2);   // true  (same reference)
System.out.println(s1 == s3);   // false (different objects)
System.out.println(s1.equals(s3)); // true (same content)

String s4 = s3.intern();       // Returns pool reference
System.out.println(s1 == s4);  // true
```

**Memory diagram:**
```
Heap:
  String Pool: ["hello"] ← s1, s2 point here
  Regular: [String@abc: "hello"] ← s3 points here
```

**Java version history:**
- **≤ Java 6:** String Pool in PermGen (fixed size, could cause `OutOfMemoryError: PermGen space`)
- **Java 7+:** String Pool moved to main Heap (can be GC'd)
- **Java 8+:** PermGen replaced by Metaspace (native memory)

</details>

---

### Primitive Types and Wrapper Classes

---

🟢 **Q8. List all Java primitive types with their sizes and default values.**

<details><summary>Click to reveal answer</summary>

| Type | Size | Range | Default (field) | Wrapper |
|------|------|-------|----------------|---------|
| `byte` | 8-bit | -128 to 127 | `0` | `Byte` |
| `short` | 16-bit | -32,768 to 32,767 | `0` | `Short` |
| `int` | 32-bit | -2³¹ to 2³¹-1 | `0` | `Integer` |
| `long` | 64-bit | -2⁶³ to 2⁶³-1 | `0L` | `Long` |
| `float` | 32-bit IEEE 754 | ~±3.4×10³⁸ | `0.0f` | `Float` |
| `double` | 64-bit IEEE 754 | ~±1.7×10³⁰⁸ | `0.0d` | `Double` |
| `char` | 16-bit Unicode | '\u0000' to '\uFFFF' | `'\u0000'` | `Character` |
| `boolean` | JVM-dependent | `true`/`false` | `false` | `Boolean` |

> 💡 **Note:** Default values only apply to class fields, NOT local variables. Using an uninitialized local variable causes a compile error.

```java
class Foo {
    int x;           // default: 0 ✓
    String s;        // default: null ✓

    void bar() {
        int y;
        System.out.println(y); // COMPILE ERROR: variable y might not have been initialized
    }
}
```

</details>

---

🟡 **Q9. What is autoboxing and unboxing? What are the performance implications?**

<details><summary>Click to reveal answer</summary>

**Autoboxing:** Automatic conversion of primitive to wrapper class.
**Unboxing:** Automatic conversion of wrapper class to primitive.

```java
// Autoboxing
Integer i = 42;          // equivalent to: Integer i = Integer.valueOf(42);
List<Integer> list = new ArrayList<>();
list.add(100);           // autoboxing: int → Integer

// Unboxing
int x = i;               // equivalent to: int x = i.intValue();
int sum = list.get(0) + 5; // unboxing: Integer → int
```

**Performance pitfall — unnecessary boxing in loops:**
```java
// BAD: Creates ~1000 Integer objects
Long sum = 0L;
for (long i = 0; i < 1000; i++) {
    sum += i; // unbox sum, add i, rebox result — 1000 autoboxing operations!
}

// GOOD: Use primitive
long sum = 0L;
for (long i = 0; i < 1000; i++) {
    sum += i;
}
```

**Integer cache (-128 to 127):**
```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (cached)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (not cached, different objects)
System.out.println(c.equals(d)); // true
```

🚨 **Common Mistake:** Using `==` to compare `Integer` objects instead of `.equals()`. Works for -128 to 127 due to cache, fails outside that range.

</details>

---

🟡 **Q10. Explain type casting in Java — widening vs narrowing.**

<details><summary>Click to reveal answer</summary>

**Widening (implicit):** Smaller type → Larger type. Safe, no data loss.
```
byte → short → int → long → float → double
                          char → int
```

**Narrowing (explicit):** Larger type → Smaller type. May lose data.

```java
// Widening (automatic)
int x = 100;
long y = x;       // automatic widening
double d = x;     // automatic widening

// Narrowing (must cast explicitly)
double pi = 3.14159;
int truncated = (int) pi;   // 3 (decimal part lost)

long big = 1_000_000_000_000L;
int overflow = (int) big;   // data loss! takes lower 32 bits

// Char ↔ int conversions
char c = 'A';
int ascii = c;       // widening: 65
char back = (char) 65; // narrowing: 'A'
```

**Casting with objects (ClassCastException):**
```java
Object obj = "hello";
String s = (String) obj;  // OK: actual type IS String
Integer n = (Integer) obj; // throws ClassCastException at runtime

// Safe casting with instanceof
if (obj instanceof String str) { // Java 16+ pattern matching
    System.out.println(str.length());
}
```

</details>

---

### Operators and Control Flow

---

🟢 **Q11. What is the difference between `==` and `.equals()`?**

<details><summary>Click to reveal answer</summary>

| | `==` | `.equals()` |
|-|------|------------|
| **Primitives** | Compares values | N/A (primitives have no methods) |
| **Objects** | Compares references (memory addresses) | Compares content (if properly overridden) |
| **null** | `null == null` is `true` | Calling on null throws `NullPointerException` |

```java
String s1 = new String("hello");
String s2 = new String("hello");

s1 == s2       // false (different objects on heap)
s1.equals(s2)  // true (same content)

// For primitives:
int a = 5, b = 5;
a == b          // true (value comparison)

// Null safety
String s = null;
s == null          // true
s.equals("hello")  // NullPointerException!
Objects.equals(s, "hello")  // false, null-safe
```

🚨 **Common Mistake:** Comparing strings with `==` in production code. This works accidentally for string literals (due to pool) but breaks for dynamically created strings.

</details>

---

🟡 **Q12. What is the difference between `&`, `&&`, `|`, `||`?**

<details><summary>Click to reveal answer</summary>

| Operator | Type | Short-circuit? | Use case |
|----------|------|---------------|---------|
| `&` | Bitwise AND / Non-short-circuit AND | ❌ No | Bitwise ops, always evaluates both |
| `&&` | Logical AND | ✅ Yes | Boolean conditions |
| `\|` | Bitwise OR / Non-short-circuit OR | ❌ No | Bitwise ops, always evaluates both |
| `\|\|` | Logical OR | ✅ Yes | Boolean conditions |

```java
// Short-circuit prevents NullPointerException
String s = null;
if (s != null && s.length() > 0) { ... } // Safe: short-circuits if s is null

// Non-short-circuit: BOTH sides always evaluated
int x = 5;
if (x > 3 | ++x > 4) { ... } // x becomes 6 regardless
if (x > 3 || ++x > 4) { ... } // x stays 6, right side NOT evaluated

// Bitwise on integers
int a = 0b1010;  // 10
int b = 0b1100;  // 12
int c = a & b;   // 0b1000 = 8 (bitwise AND)
int d = a | b;   // 0b1110 = 14 (bitwise OR)
int e = a ^ b;   // 0b0110 = 6 (bitwise XOR)
int f = ~a;      // bitwise NOT
int g = a << 1;  // 0b10100 = 20 (left shift)
int h = a >> 1;  // 0b0101 = 5 (right shift)
int i = -1 >>> 1; // unsigned right shift
```

</details>

---

🟢 **Q13. What is the enhanced switch expression (Java 14+)?**

<details><summary>Click to reveal answer</summary>

Java 14+ introduced **switch expressions** (finalized) with arrow syntax, eliminating fall-through bugs.

```java
// Old switch statement (fall-through prone)
int day = 3;
String name;
switch (day) {
    case 1: name = "Monday"; break;
    case 2: name = "Tuesday"; break;
    // Forgot break? Fall-through bug!
    default: name = "Other";
}

// New switch expression (Java 14+)
String name = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3, 4, 5 -> "Midweek";  // multiple labels
    default -> "Weekend";
};

// With yield for multi-statement blocks
String result = switch (day) {
    case 1 -> "Monday";
    default -> {
        String s = "Day " + day;
        yield s.toUpperCase();  // yield replaces return in switch expressions
    }
};

// Switch with strings, enums (Java 7+), patterns (Java 21+)
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area = switch (shape) {
    case Circle c    -> Math.PI * c.r() * c.r();
    case Rectangle r -> r.w() * r.h();
};
```

</details>

---

### `static`, `final`, `this`, `super`

---

🟡 **Q14. What are all the uses of the `static` keyword in Java?**

<details><summary>Click to reveal answer</summary>

`static` makes a member belong to the **class**, not to instances.

```java
public class Counter {
    // 1. Static variable (class variable) — shared across all instances
    private static int count = 0;

    // 2. Static method — can be called without creating an instance
    public static int getCount() {
        return count;
    }

    // 3. Static block — runs once when class is loaded
    static {
        System.out.println("Counter class loaded");
        count = 0;
    }

    // 4. Static nested class — doesn't have access to outer instance
    static class Helper {
        void help() { System.out.println("Helping!"); }
    }

    // 5. Static import (in import section)
    // import static java.lang.Math.PI;
    // Then use: double area = PI * r * r;
}
```

**Key rules:**
- Static methods **cannot** access instance variables or call instance methods directly
- Static methods **cannot** use `this` or `super`
- Static variables are initialized when the class is loaded
- Static methods can be inherited but **cannot be overridden** (only hidden)

🚨 **Common Mistake:** Thinking you can override static methods. Static methods are resolved at compile-time (method hiding), not runtime (polymorphism).

```java
class Parent {
    static void display() { System.out.println("Parent"); }
}
class Child extends Parent {
    static void display() { System.out.println("Child"); } // method HIDING
}
Parent ref = new Child();
ref.display(); // prints "Parent" — NOT polymorphic!
```

</details>

---

🟡 **Q15. What are all the uses of the `final` keyword?**

<details><summary>Click to reveal answer</summary>

`final` means **cannot be changed/extended/overridden** depending on context.

```java
// 1. final variable — constant (must be initialized once)
final int MAX = 100;
// MAX = 200; // COMPILE ERROR

// 2. final reference — reference is constant, object is mutable!
final List<String> list = new ArrayList<>();
list.add("hello");    // OK — mutating the object
// list = new ArrayList<>(); // COMPILE ERROR — cannot reassign reference

// 3. final method — cannot be overridden by subclasses
class Parent {
    final void show() { System.out.println("Parent"); }
}
class Child extends Parent {
    // void show() {} // COMPILE ERROR

    void demo() {
        // 4. final parameter — cannot be reassigned in method body
        // (useful to prevent accidental reassignment)
    }
    void demo(final int x) {
        // x = 5; // COMPILE ERROR
    }
}

// 5. final class — cannot be subclassed
final class Singleton { ... }
// class SubSingleton extends Singleton {} // COMPILE ERROR

// NOTE: String, Integer, and other wrapper classes are final
```

**Blank final variables:**
```java
class Config {
    final String host;  // blank final — must be set in constructor

    Config(String host) {
        this.host = host;  // set in constructor
    }
}
```

> 💡 **Interviewer Tip:** Always clarify that `final` on an object reference does NOT make the object immutable — it only prevents reassignment of the reference. True immutability requires more effort (final fields, no setters, defensive copies).

</details>

---

🟡 **Q16. Explain `this` and `super` keywords with examples.**

<details><summary>Click to reveal answer</summary>

**`this`** refers to the current object instance:

```java
class Person {
    String name;
    int age;

    // 1. Disambiguate field vs parameter
    Person(String name, int age) {
        this.name = name;   // this.name = field, name = parameter
        this.age = age;
    }

    // 2. Constructor chaining (must be first statement)
    Person(String name) {
        this(name, 0);  // calls Person(String, int)
    }

    // 3. Pass current instance as argument
    void register(Registry r) {
        r.add(this);
    }

    // 4. Return current instance (builder pattern)
    Person withName(String name) {
        this.name = name;
        return this;
    }
}
```

**`super`** refers to the parent class:

```java
class Animal {
    String name;
    Animal(String name) { this.name = name; }
    void speak() { System.out.println("..."); }
}

class Dog extends Animal {
    String breed;

    // 1. Call parent constructor (must be first statement)
    Dog(String name, String breed) {
        super(name);      // calls Animal(String)
        this.breed = breed;
    }

    // 2. Call parent's overridden method
    @Override
    void speak() {
        super.speak();    // calls Animal.speak()
        System.out.println("Woof!");
    }

    // 3. Access parent's field (if hidden by child's field)
    void info() {
        System.out.println(super.name); // explicitly accesses Animal.name
    }
}
```

🚨 **Common Mistake:** `this()` and `super()` must be the **first statement** in a constructor, so you cannot call both.

</details>

---

### Garbage Collection

---

🟡 **Q17. Explain how Java Garbage Collection works.**

<details><summary>Click to reveal answer</summary>

Java GC automatically reclaims memory occupied by objects that are no longer reachable.

**Reachability:**
An object is **reachable** if there's any path from a GC root to it.
GC roots include: stack variables, static variables, JNI references.

**Generational GC (HotSpot):**
```
Young Generation (Minor GC — frequent, fast)
├── Eden Space      → new objects born here
├── Survivor S0     → objects that survived 1+ GC
└── Survivor S1     → objects that survived 1+ GC

Old Generation (Major/Full GC — infrequent, slow)
└── Tenured         → objects that survived N minor GCs (N = tenuring threshold)

Metaspace (since Java 8)
└── Class metadata (was PermGen before Java 8)
```

**GC Algorithms:**
| GC | Flag | Description | Use case |
|----|------|-------------|---------|
| Serial GC | `-XX:+UseSerialGC` | Single-threaded, stop-the-world | Small apps, single CPU |
| Parallel GC | `-XX:+UseParallelGC` | Multi-threaded, throughput-focused | Batch jobs |
| G1 GC | `-XX:+UseG1GC` | Region-based, predictable pauses | Default Java 9+ |
| ZGC | `-XX:+UseZGC` | Sub-millisecond pauses | Low-latency (Java 15+) |
| Shenandoah | `-XX:+UseShenandoahGC` | Concurrent compaction | Low-latency (OpenJDK) |

**Making an object eligible for GC:**
```java
Object o = new Object();  // reachable
o = null;                  // now eligible for GC (if no other refs)

// Weak and Soft references for caching
WeakReference<MyObj> weakRef = new WeakReference<>(new MyObj());
// Gets collected when memory is needed
```

> 💡 **Interviewer Tip:** Never call `System.gc()` in production — it's a hint to the JVM, not a command, and can cause performance degradation.

</details>

---

🔴 **Q18. What is the difference between Minor GC, Major GC, and Full GC?**

<details><summary>Click to reveal answer</summary>

| Type | Scope | Frequency | STW Pause | Trigger |
|------|-------|-----------|-----------|---------|
| **Minor GC** | Young Generation only | Frequent | Short | Eden is full |
| **Major GC** | Old Generation | Less frequent | Longer | Old Gen threshold |
| **Full GC** | Entire Heap + Metaspace | Rare | Longest | Low memory, explicit `System.gc()`, promotion failure |

**Minor GC process:**
1. Eden is full → Minor GC triggered
2. Live objects in Eden copied to Survivor space (S0 or S1)
3. Objects that have survived N GCs promoted to Old Gen
4. Eden + old Survivor space cleared

**Stop-The-World (STW) pauses:**
All application threads are paused during certain GC phases. Modern GCs (G1, ZGC) minimize STW pauses by doing most work concurrently.

**Promotion failure** triggers Full GC:
When Old Gen has no space for promoted objects from Young Gen.

```bash
# Useful JVM flags for GC tuning
-Xms512m -Xmx2g           # Initial/max heap size
-XX:NewRatio=2             # Old:Young ratio (2:1)
-XX:MaxGCPauseMillis=200   # Target max GC pause (G1)
-XX:+PrintGCDetails        # Print GC logs
-Xlog:gc*                  # Modern GC logging (Java 9+)
```

</details>

---

### Pass-by-Value

---

🟡 **Q19. Is Java pass-by-value or pass-by-reference?**

<details><summary>Click to reveal answer</summary>

Java is **always pass-by-value**. However, for objects, the value being passed is the **reference (memory address)**, not a copy of the object.

```java
// Primitives: value is copied
void increment(int x) {
    x++;  // only modifies local copy
}
int a = 5;
increment(a);
System.out.println(a); // still 5

// Objects: reference is copied (but points to same object)
void addItem(List<String> list) {
    list.add("hello");  // modifies the SAME object
}
List<String> myList = new ArrayList<>();
addItem(myList);
System.out.println(myList); // ["hello"] — object was mutated

// Reassigning reference does NOT affect caller
void reassign(List<String> list) {
    list = new ArrayList<>();  // only changes local copy of reference
    list.add("bye");
}
List<String> myList2 = new ArrayList<>();
myList2.add("hi");
reassign(myList2);
System.out.println(myList2); // ["hi"] — original reference unchanged
```

**Mental model:** Java passes a copy of the reference (like copying a house address — you can go to the house and modify it, but you can't change where the original person's address book points).

</details>

---

## 🎭 Scenario-Based Questions

---

🟡 **S1. What will happen when this code runs?**

```java
public class Main {
    static int x = 10;

    public static void main(String[] args) {
        int x = 20;
        System.out.println(x);        // A
        System.out.println(Main.x);   // B
        {
            int y = 30;
            System.out.println(y);    // C
        }
        // System.out.println(y);     // D - would this compile?
    }
}
```

<details><summary>Click to reveal answer</summary>

- **A:** `20` — local variable `x` shadows the static field `x`
- **B:** `10` — explicitly accessing the static field via class name
- **C:** `30` — `y` is accessible within its block
- **D:** **Compile error** — `y` is out of scope outside its block

**Key concepts:** Variable shadowing, block scope, static vs instance variables.

</details>

---

🟡 **S2. What does this print?**

```java
public class Test {
    public static void main(String[] args) {
        Integer a = 100;
        Integer b = 100;
        Integer c = 200;
        Integer d = 200;

        System.out.println(a == b);  // 1
        System.out.println(c == d);  // 2
        System.out.println(a.equals(b)); // 3
    }
}
```

<details><summary>Click to reveal answer</summary>

1. **`true`** — Integer cache: values -128 to 127 are cached, so `a` and `b` point to the same cached `Integer` object.
2. **`false`** — 200 is outside the cache range, so `c` and `d` are two different `Integer` objects on the heap.
3. **`true`** — `.equals()` compares values, not references.

🚨 **Common Mistake:** Assuming `==` on Integer objects always works. This is a classic interview trap!

</details>

---

🟡 **S3. What's the output of this code?**

```java
class A {
    A() {
        System.out.println("A()");
        show();
    }
    void show() {
        System.out.println("A.show()");
    }
}

class B extends A {
    int x = 5;
    B() {
        System.out.println("B()");
    }
    @Override
    void show() {
        System.out.println("B.show(), x=" + x);
    }
}

public class Main {
    public static void main(String[] args) {
        new B();
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output:**
```
A()
B.show(), x=0
B()
```

**Explanation:**
1. `new B()` → implicit `super()` call → `A()` runs first
2. Inside `A()`, `show()` is called — but `show()` is overridden in `B` → **dynamic dispatch** calls `B.show()`
3. At this point, `B`'s instance variable `x` hasn't been initialized yet (instance initializers run after `super()`) — so `x = 0` (default)
4. `A()` finishes, then `B()` body runs → prints "B()"

> 💡 **Interviewer Tip:** This demonstrates why you should **never call overridable methods from constructors**. The object is not fully initialized when the parent constructor runs.

</details>

---

🔴 **S4. What's wrong with this code?**

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

<details><summary>Click to reveal answer</summary>

This is a **broken singleton** in a multithreaded environment. Two threads could simultaneously see `instance == null` and both create a new instance.

**Fix 1: Synchronized method (simple but slow):**
```java
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```

**Fix 2: Double-Checked Locking (better performance):**
```java
private static volatile Singleton instance; // volatile is critical!

public static Singleton getInstance() {
    if (instance == null) {                    // 1st check (no lock)
        synchronized (Singleton.class) {
            if (instance == null) {            // 2nd check (with lock)
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```
`volatile` prevents CPU instruction reordering of `instance = new Singleton()` (alloc memory → initialize → assign).

**Fix 3: Initialization-on-demand holder (best):**
```java
public class Singleton {
    private Singleton() {}

    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;  // class loaded lazily, thread-safe by JVM class loading
    }
}
```

**Fix 4: Enum Singleton (effective Java recommendation):**
```java
public enum Singleton {
    INSTANCE;
    public void doSomething() { ... }
}
```

</details>

---

🟡 **S5. What will happen?**

```java
public class Test {
    public static void main(String[] args) {
        String s = null;
        System.out.println(s + " world");  // A
        System.out.println(s.concat(" world")); // B
    }
}
```

<details><summary>Click to reveal answer</summary>

- **A:** Prints `"null world"` — String concatenation with `+` converts `null` to the string `"null"` (compiler uses `StringBuilder.append(Object)` which handles null)
- **B:** Throws `NullPointerException` — `.concat()` is a method call on `null` reference

</details>

---

🟡 **S6. What does this print?**

```java
public class Test {
    static {
        System.out.println("Static block 1");
    }

    int x = initX();

    static {
        System.out.println("Static block 2");
    }

    int initX() {
        System.out.println("Instance initializer");
        return 10;
    }

    Test() {
        System.out.println("Constructor");
    }

    public static void main(String[] args) {
        System.out.println("main start");
        new Test();
        new Test();
    }
}
```

<details><summary>Click to reveal answer</summary>

```
Static block 1
Static block 2
main start
Instance initializer
Constructor
Instance initializer
Constructor
```

**Order:** Static initializers (in order) → `main()` → for each `new Test()`: instance initializers (in order) → constructor

Static blocks run **once** when class is loaded; instance initializers run for **each** object creation.

</details>

---

🔴 **S7. Can you spot the memory leak?**

```java
public class EventBus {
    private static final List<EventListener> listeners = new ArrayList<>();

    public static void addListener(EventListener listener) {
        listeners.add(listener);
    }

    public static void fireEvent(Event event) {
        for (EventListener l : listeners) {
            l.onEvent(event);
        }
    }
}

// In code:
for (int i = 0; i < 1000; i++) {
    EventBus.addListener(new EventListener() {
        public void onEvent(Event e) { /* ... */ }
    });
}
```

<details><summary>Click to reveal answer</summary>

**Memory leak:** `listeners` is a static `List` that holds strong references to `EventListener` objects. Even if the code that created those listeners is done using them, the static list keeps them alive forever — GC cannot collect them.

**Fixes:**
1. **Remove listeners explicitly:**
```java
EventListener listener = new EventListener() { ... };
EventBus.addListener(listener);
// ... later ...
EventBus.removeListener(listener);
```

2. **Use WeakReferences:**
```java
private static final List<WeakReference<EventListener>> listeners = new ArrayList<>();
// Clean up dead references periodically
```

3. **Use WeakHashMap or dedicated event bus (Guava EventBus):**
```java
@Subscribe
public void handleEvent(Event e) { ... }
// EventBus manages lifecycle automatically
```

</details>

---

🟡 **S8. What happens when a `final` variable is assigned in a conditional?**

```java
public class Test {
    public static void main(String[] args) {
        final int x;
        boolean condition = true;
        if (condition) {
            x = 10;
        }
        // x = 20; // Would this compile?
        System.out.println(x); // Does this compile?
    }
}
```

<details><summary>Click to reveal answer</summary>

The code **compiles and runs fine**. The Java compiler performs **definite assignment analysis** — it can determine that `x` is definitely assigned before use.

- `x = 10` is fine (blank final assigned once)
- `x = 20` after would be a **compile error** (reassignment of final)
- `System.out.println(x)` compiles because compiler can prove `x` is always assigned (condition is a `boolean` variable, but since it's always `true`... actually the compiler treats non-constant booleans as potentially both true/false)

**Wait — actually:** Since `condition` is not a compile-time constant (it's a variable, not `final boolean condition = true`), the compiler sees this as a potential path where `x` is never assigned. Let's correct:

```java
if (condition) { x = 10; }
System.out.println(x); // COMPILE ERROR: x might not have been initialized
```

To fix:
```java
if (condition) { x = 10; } else { x = 0; } // now all paths assign x
```
OR use: `final boolean condition = true;` (constant — compiler evaluates at compile time)

</details>

---

🟡 **S9. What will this print?**

```java
public class Test {
    public static void main(String[] args) {
        int[] arr = {1, 2, 3, 4, 5};
        for (int x : arr) {
            x = x * 2;  // Does this modify the array?
        }
        System.out.println(Arrays.toString(arr));
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output:** `[1, 2, 3, 4, 5]`

The enhanced for loop creates a **copy** of each element in `x`. Modifying `x` does not modify the array. For primitives, changes to the loop variable have no effect on the original array.

**To modify:**
```java
for (int i = 0; i < arr.length; i++) {
    arr[i] = arr[i] * 2;  // modifies original array
}
// Now: [2, 4, 6, 8, 10]
```

**Note:** For arrays of objects, you CAN mutate the object through the loop variable (since you have a reference to the object), but you cannot reassign the reference.

</details>

---

🔴 **S10. What is the output of this tricky `finally` behavior?**

```java
public class Test {
    public static int getValue() {
        int x = 10;
        try {
            return x;
        } finally {
            x = 20;
            System.out.println("finally: x = " + x);
        }
    }

    public static void main(String[] args) {
        System.out.println("result: " + getValue());
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output:**
```
finally: x = 20
result: 10
```

**Explanation:** When `return x` is executed in the `try` block, the value `10` is **saved**. The `finally` block executes before actually returning (always), modifying `x` to `20` and printing it. But the **saved return value** is `10`, so that's what's returned.

🚨 If `finally` has its own `return` statement, it **overwrites** the try's return value:
```java
try {
    return 10;
} finally {
    return 20; // BAD PRACTICE — hides the return from try
}
// Returns 20
```

</details>

---

🟡 **S11. What will this print? (Tricky operator precedence)**

```java
public class Test {
    public static void main(String[] args) {
        int a = 5;
        int b = a++ + ++a;
        System.out.println("a=" + a + " b=" + b);

        int x = 10;
        int y = x-- - --x;
        System.out.println("x=" + x + " y=" + y);
    }
}
```

<details><summary>Click to reveal answer</summary>

**For `b = a++ + ++a`:**
- `a++`: uses `a=5`, then increments `a` to `6` → evaluates to `5`
- `++a`: increments `a` to `7`, evaluates to `7`
- `b = 5 + 7 = 12`, `a = 7`
- Output: `a=7 b=12`

**For `y = x-- - --x`:**
- `x--`: uses `x=10`, then decrements `x` to `9` → evaluates to `10`
- `--x`: decrements `x` to `8`, evaluates to `8`
- `y = 10 - 8 = 2`, `x = 8`
- Output: `x=8 y=2`

🚨 **Common Mistake:** Misunderstanding post vs pre increment/decrement. Never write production code like this — it's confusing and error-prone.

</details>

---

🟡 **S12. Why does this fail?**

```java
public class Test {
    public static void main(String[] args) {
        long x = 10000000000; // COMPILE ERROR?
        double d = 1/2;
        System.out.println(d);
    }
}
```

<details><summary>Click to reveal answer</summary>

1. `long x = 10000000000;` → **COMPILE ERROR**: `10000000000` is treated as an `int` literal, which overflows. Must use `L` suffix: `long x = 10000000000L;`

2. `double d = 1/2;` → **Prints `0.0`**: Both `1` and `2` are `int` literals, so integer division is performed first (result: `0`), then widened to `double`. For `0.5`, use: `1.0/2`, `1/2.0`, or `(double)1/2`.

</details>

---

🟡 **S13. What is the output?**

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(0.1 + 0.2);
        System.out.println(0.1 + 0.2 == 0.3);
    }
}
```

<details><summary>Click to reveal answer</summary>

```
0.30000000000000004
false
```

**Why?** IEEE 754 floating-point representation cannot exactly represent `0.1` or `0.2` in binary. The result of adding their approximations is slightly different from the approximation of `0.3`.

**Fix:** Use `BigDecimal` for precise decimal arithmetic:
```java
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
System.out.println(a.add(b));        // 0.3
System.out.println(a.add(b).compareTo(new BigDecimal("0.3")) == 0); // true

// OR use tolerance comparison for doubles:
double epsilon = 1e-9;
Math.abs(0.1 + 0.2 - 0.3) < epsilon  // true
```

🚨 **Never** use `double` for monetary calculations. Use `BigDecimal`.

</details>

---

🔴 **S14. What happens here? (Class loading order)**

```java
class Parent {
    static int x = initX();
    static int initX() {
        System.out.println("Parent.initX(), y=" + Child.y);
        return 10;
    }
}

class Child extends Parent {
    static int y = 20;
}

public class Main {
    public static void main(String[] args) {
        System.out.println("Parent.x = " + Parent.x);
        System.out.println("Child.y = " + Child.y);
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output:**
```
Parent.initX(), y=0
Parent.x = 10
Child.y = 20
```

**Why `y=0`?**
When `Parent` is loaded, `initX()` accesses `Child.y`. This triggers loading of `Child`, which first loads `Parent` (already being loaded — circular dependency). At this point, `Child.y` is still `0` (default) because `Child`'s static initialization hasn't run yet. After `Parent`'s static init finishes, `Child.y` is set to `20`.

</details>

---

🟡 **S15. What will this print?**

```java
public class Test {
    public static void main(String[] args) {
        Integer x = 1;
        Integer y = 2;
        Integer z = 3;
        Integer a = 300;
        Integer b = 300;
        Long c = 3L;

        System.out.println(z == (x + y));        // 1
        System.out.println(z.equals(x + y));     // 2
        System.out.println(c == (x + y));        // 3
        System.out.println(c.equals(x + y));     // 4
        System.out.println(c.equals((long)(x + y))); // 5
    }
}
```

<details><summary>Click to reveal answer</summary>

1. **`true`** — `(x + y)` unboxes to `int` sum `3`, `z == 3` unboxes `z`, comparing `int 3 == int 3`
2. **`true`** — `equals()` with `Integer` argument, value comparison
3. **`true`** — `(x + y)` unboxes to `int 3`, widened to `long 3`. `c == 3L` unboxes `c`, comparing `long 3 == long 3`
4. **`false`** — `c` is `Long`, `(x + y)` autoboxes to `Integer(3)`. `Long.equals(Integer)` returns false (different types)
5. **`true`** — `(long)(x+y)` casts int 3 to long 3, autoboxed to `Long(3)`. `Long.equals(Long)` compares values

> 💡 **Interviewer Tip:** This is a classic trick question testing knowledge of autoboxing, widening, and equals() behavior across numeric types.

</details>

---

🟡 **S16. Is this code thread-safe?**

```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

<details><summary>Click to reveal answer</summary>

**Not thread-safe.** `count++` is **not atomic** — it's actually three operations:
1. Read `count`
2. Add 1
3. Write back

Two threads can interleave these steps, causing lost updates.

**Fixes:**
```java
// Option 1: synchronized
public synchronized void increment() { count++; }

// Option 2: AtomicInteger (best for simple counters)
private AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }

// Option 3: volatile (NOT sufficient for increment — only solves visibility, not atomicity)
private volatile int count = 0;  // Still NOT thread-safe for count++!

// Option 4: LongAdder (best for high-contention scenarios)
private LongAdder count = new LongAdder();
public void increment() { count.increment(); }
public long getCount() { return count.sum(); }
```

</details>

---

🟡 **S17. What will happen?**

```java
public class Test {
    public static void main(String[] args) {
        try {
            System.exit(0);
        } finally {
            System.out.println("finally block"); // Will this run?
        }
    }
}
```

<details><summary>Click to reveal answer</summary>

`"finally block"` will **NOT** be printed. `System.exit(0)` terminates the JVM immediately — `finally` blocks are not executed when the JVM exits.

**Exceptions to "finally always runs":**
1. `System.exit()` — JVM terminates
2. `Runtime.halt()` — abrupt JVM termination
3. JVM crash (OutOfMemoryError, SIGSEGV, etc.)
4. Thread is killed (rare edge case)
5. Infinite loop in `try` block (finally never reached)

</details>

---

🟡 **S18. What's wrong?**

```java
public class Test {
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            Thread t = new Thread(new Runnable() {
                public void run() {
                    System.out.println(i); // COMPILE ERROR?
                }
            });
            t.start();
        }
    }
}
```

<details><summary>Click to reveal answer</summary>

**COMPILE ERROR:** `Variable 'i' used in lambda should be effectively final`.

Anonymous inner classes and lambdas can only access local variables that are **final** or **effectively final** (never reassigned). The loop variable `i` is reassigned each iteration, so it's not effectively final.

**Fix:** Capture a copy:
```java
for (int i = 0; i < 10; i++) {
    final int localI = i;  // effectively final copy
    new Thread(() -> System.out.println(localI)).start();
}
```

**Note:** Even if the compile error were fixed, the output order is non-deterministic since threads are scheduled by the OS.

</details>

---

🔴 **S19. What will this code do? (StackOverflowError)**

```java
public class Test {
    public static int factorial(int n) {
        return n * factorial(n - 1); // Missing base case!
    }

    public static void main(String[] args) {
        System.out.println(factorial(5));
    }
}
```

<details><summary>Click to reveal answer</summary>

This throws `StackOverflowError` at runtime. Without a base case, `factorial()` calls itself infinitely. Each call pushes a new stack frame (local variables + return address) onto the thread's stack until it's exhausted.

**Fixed version:**
```java
public static int factorial(int n) {
    if (n <= 1) return 1;  // base case
    return n * factorial(n - 1);
}
```

**Stack overflow vs heap OOM:**
- `StackOverflowError`: too deep recursion → stack exhausted
- `OutOfMemoryError: Java heap space`: too many objects on heap
- `OutOfMemoryError: Metaspace`: too many class definitions

**Alternative: Iterative factorial (avoids stack overflow for large n):**
```java
public static long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) result *= i;
    return result;
}
```

</details>

---

🟡 **S20. What does this output?**

```java
public class Test {
    public static void main(String[] args) {
        String s1 = "Java";
        String s2 = "Java";
        String s3 = new String("Java");
        String s4 = s3.intern();

        System.out.println(s1 == s2);     // A
        System.out.println(s1 == s3);     // B
        System.out.println(s1 == s4);     // C
        System.out.println(s3 == s4);     // D
    }
}
```

<details><summary>Click to reveal answer</summary>

- **A: `true`** — both literals point to the same String Pool entry
- **B: `false`** — `s3` is a new object on heap, not from pool
- **C: `true`** — `intern()` returns the pool reference for "Java", which is the same as `s1`
- **D: `false`** — `s3` is heap object, `s4` is pool reference, different addresses

</details>

---

🟢 **S21. What's the output? (Widening vs overloading)**

```java
public class Test {
    static void print(int x)    { System.out.println("int"); }
    static void print(long x)   { System.out.println("long"); }
    static void print(double x) { System.out.println("double"); }

    public static void main(String[] args) {
        print(5);
        print(5L);
        print(5.0);
        byte b = 5;
        print(b);
    }
}
```

<details><summary>Click to reveal answer</summary>

```
int
long
double
int
```

For `print(b)`: `byte` is widened to `int` (the most specific matching type). Java always chooses the most specific applicable overloaded method.

**Overload resolution priority:**
1. Exact match
2. Widening
3. Autoboxing
4. Varargs

</details>

---

🟡 **S22. What happens when you compare `null` with instanceof?**

```java
Object obj = null;
System.out.println(obj instanceof String);   // A
System.out.println(obj instanceof Object);   // B
```

<details><summary>Click to reveal answer</summary>

- **A: `false`** — `null instanceof AnyType` is always `false`. `null` has no type.
- **B: `false`** — same reason, even `null instanceof Object` is `false`.

This is by design: `instanceof` safely handles null without throwing `NullPointerException`, making it useful for null-safe type checks:
```java
if (obj instanceof String s) { // null-safe, never NPE
    System.out.println(s.length());
}
```

</details>

---

🟡 **S23. What's wrong with this use of `var`?**

```java
var list = new ArrayList<>();
list.add("hello");
list.add(42);
list.add(new Object());

for (var item : list) {
    String s = item; // Compile error?
}
```

<details><summary>Click to reveal answer</summary>

`var list = new ArrayList<>()` infers the type as `ArrayList<Object>` (because the diamond `<>` cannot infer a specific type without a left-hand side type). So `item` in the for loop is of type `Object`.

`String s = item` is a **compile error** — cannot assign `Object` to `String` without cast.

**Better practice:**
```java
var list = new ArrayList<String>();  // infers ArrayList<String>
// or
List<String> list = new ArrayList<>(); // explicit type
```

**`var` rules:**
- Only valid for local variables (not fields, parameters, return types)
- Must have an initializer (so type can be inferred)
- Cannot be `null` initializer (type unknown)
- Cannot be used in lambda parameters (Java 10-10; Java 11+ allows `(var x) -> x.length()`)

</details>

---

🔴 **S24. What is the output of this complex initialization order?**

```java
public class Test {
    static int a = 10;
    int b = a + 5;
    static { a = 20; }
    int c = a + b;

    Test() {
        System.out.println("a=" + a + " b=" + b + " c=" + c);
    }

    public static void main(String[] args) {
        new Test();
    }
}
```

<details><summary>Click to reveal answer</summary>

**Output:** `a=20 b=15 c=35`

**Order of initialization:**
1. Static variables and blocks (in textual order):
   - `a = 10` (static field)
   - `a = 20` (static block executes)
2. Instance variables (in textual order):
   - `b = a + 5 = 20 + 5 = 25`... 

Wait, let me re-trace:
1. Class loaded: static field `a = 10`, then static block runs `a = 20`. So `a = 20`.
2. `new Test()`: instance fields initialized in order:
   - `b = a + 5 = 20 + 5 = 25`
   - `c = a + b = 20 + 25 = 45`
3. Constructor runs: prints `a=20 b=25 c=45`

**Corrected Output:** `a=20 b=25 c=45`

**Key:** Static initialization (fields + blocks in textual order) runs once at class load time, before any instance is created.

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| Using `==` to compare strings | Use `.equals()` or `Objects.equals()` |
| `int overflow = 100 * 1000000` | Use `long`: `100L * 1000000` |
| `double d = 1/2` expecting `0.5` | Use `1.0/2` or `(double)1/2` |
| `System.gc()` in production | Let JVM manage GC |
| Calling overridable methods in constructors | Use factory methods or `final` methods |
| `count++` in multithreaded code | Use `AtomicInteger` or `synchronized` |
| Assuming `finally` always runs | `System.exit()` skips finally |
| `long x = 10000000000` | `long x = 10000000000L` |
| `float f = 3.14` | `float f = 3.14f` |
| Mutating loop variable in enhanced for loop | Use index-based loop for mutation |

---

## ✅ Best Practices

- ✅ Use `final` for variables that should not be reassigned
- ✅ Use `Objects.equals(a, b)` for null-safe comparison
- ✅ Use `BigDecimal` for monetary calculations
- ✅ Prefer `long` over `int` when values may exceed 2 billion
- ✅ Always specify the `L` suffix for long literals > `Integer.MAX_VALUE`
- ✅ Use `var` for improved readability but maintain explicit generics where clarity matters
- ✅ Design singletons using enum or initialization-on-demand holder idiom
- ✅ Use `AtomicInteger`/`LongAdder` for thread-safe counters
- ✅ Always define a base case for recursive methods

---

## 🔗 Related Sections

- [Chapter 2: OOP](./02-oops.md) — `static` and `final` in OOP design
- [Chapter 3: Multithreading](./03-multithreading-and-concurrency.md) — thread-safety, `volatile`, `synchronized`
- [Chapter 5: Exception Handling](./05-exception-handling.md) — `try/finally` traps
- [Chapter 7: Strings & Regex](./07-strings-and-regex.md) — String pool deep dive

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Section README](./README.md) | [Repository Root](../README.md) | [Chapter 2: OOP →](./02-oops.md) |
