# Annotations & Reflection

> 💡 **Interviewer Tip:** Annotation and Reflection questions reveal how well you understand Java's metaprogramming capabilities. Interviewers look for understanding of how frameworks like Spring, Hibernate, and JUnit use these features under the hood — and for awareness of the performance and security implications.

---

## Table of Contents
1. [Built-in Annotations](#built-in-annotations)
2. [Custom Annotations](#custom-annotations)
3. [Meta-Annotations](#meta-annotations)
4. [Reflection API](#reflection-api)
5. [Dynamic Proxies](#dynamic-proxies)
6. [Annotation Processors](#annotation-processors)
7. [Theory Questions](#theory-questions)
8. [Scenario-Based Questions](#scenario-based-questions)

---

## Quick Reference

| Feature | Package | Use Case |
|---|---|---|
| Built-in annotations | `java.lang` | `@Override`, `@Deprecated`, `@SuppressWarnings` |
| Meta-annotations | `java.lang.annotation` | Define annotation behavior |
| Reflection | `java.lang.reflect` | Inspect/invoke at runtime |
| Dynamic proxies | `java.lang.reflect` | AOP, decorators, mocking |
| Annotation processing | `javax.annotation.processing` | Compile-time code generation |

---

## Theory Questions

### Built-in Annotations

**Q1.** 🟢 What are the standard Java built-in annotations?

<details><summary>Click to reveal answer</summary>

**Compiler annotations:**

| Annotation | Purpose |
|---|---|
| `@Override` | Ensures method overrides a superclass/interface method |
| `@Deprecated` | Marks element as obsolete; compiler warns on usage |
| `@SuppressWarnings` | Suppresses specific compiler warnings |
| `@SafeVarargs` | Suppresses unchecked warnings for varargs with generics |
| `@FunctionalInterface` | Ensures interface has exactly one abstract method |

```java
public class Example {

    @Override
    public String toString() { return "Example"; }

    @Deprecated(since = "2.0", forRemoval = true)
    public void oldMethod() { }

    @SuppressWarnings({"unchecked", "deprecation"})
    public void suppressedMethod() {
        List rawList = new ArrayList(); // unchecked
        oldMethod(); // deprecation
    }

    @SafeVarargs
    public final <T> List<T> asList(T... elements) {
        return Arrays.asList(elements);
    }

    @FunctionalInterface
    interface Transformer<T, R> {
        R transform(T input);
        // Only ONE abstract method allowed
    }
}
```

</details>

---

**Q2.** 🟡 What is `@FunctionalInterface` and what does it enforce?

<details><summary>Click to reveal answer</summary>

`@FunctionalInterface` is a marker annotation that enforces at **compile time** that the interface has exactly one abstract method (SAM — Single Abstract Method). The interface can still have:
- Multiple `default` methods
- Multiple `static` methods
- Methods from `Object` (`equals`, `hashCode`, `toString`)

```java
@FunctionalInterface
interface Validator<T> {
    boolean validate(T value); // single abstract method

    // These are OK:
    default Validator<T> and(Validator<T> other) {
        return value -> this.validate(value) && other.validate(value);
    }

    static <T> Validator<T> alwaysTrue() {
        return value -> true;
    }

    // This would FAIL compilation:
    // boolean anotherAbstract(T value); // Error: multiple abstract methods
}

// Can be used as lambda target
Validator<String> notEmpty = s -> !s.isEmpty();
Validator<String> notNull = s -> s != null;
Validator<String> combined = notNull.and(notEmpty);
```

🚨 **Common Mistake:** The annotation is optional — any interface with one abstract method IS a functional interface. But omitting it means the compiler won't catch accidentally added abstract methods.

</details>

---

### Custom Annotations

**Q3.** 🟡 How do you create a custom annotation in Java?

<details><summary>Click to reveal answer</summary>

Custom annotations are declared with `@interface`. They can have elements (similar to methods) with optional default values.

```java
import java.lang.annotation.*;

@Documented                           // include in Javadoc
@Retention(RetentionPolicy.RUNTIME)   // available at runtime
@Target({ElementType.METHOD, ElementType.TYPE}) // can annotate methods and classes
public @interface Audit {
    String action();                 // required element (no default)
    String description() default ""; // optional with default
    AuditLevel level() default AuditLevel.INFO; // enum default

    // Arrays are allowed
    String[] tags() default {};

    // Annotation elements can be annotations too
    // (used in @Repeatable groups)
}

public enum AuditLevel { INFO, WARNING, CRITICAL }

// Usage:
@Audit(action = "DELETE_USER", level = AuditLevel.CRITICAL, tags = {"admin", "destructive"})
public void deleteUser(Long userId) { ... }
```

**Annotation element rules:**
- Return types: primitive, `String`, `Class`, `enum`, annotation, or arrays thereof
- No `null` defaults — use `""` or special sentinel values instead
- No generic types

</details>

---

**Q4.** 🟡 What are the `RetentionPolicy` options and when is each used?

<details><summary>Click to reveal answer</summary>

| `RetentionPolicy` | Where Available | Use Case |
|---|---|---|
| `SOURCE` | Only in source code | IDE hints, code generation, `@Override` |
| `CLASS` | In `.class` file but NOT at runtime | Bytecode tools, static analysis |
| `RUNTIME` | In `.class` file AND available via Reflection | Frameworks (Spring, JUnit, Hibernate) |

```java
@Retention(RetentionPolicy.SOURCE)
@interface Todo { String value(); } // stripped by compiler

@Retention(RetentionPolicy.CLASS)  // default if @Retention omitted
@interface BytecodeHint { }

@Retention(RetentionPolicy.RUNTIME)
@interface Transactional { boolean readOnly() default false; }

// RUNTIME annotations can be read via reflection:
Method m = MyService.class.getMethod("doWork");
Transactional tx = m.getAnnotation(Transactional.class);
if (tx != null && !tx.readOnly()) {
    beginWriteTransaction();
}
```

> 💡 **Interviewer Tip:** Ask "why does `@Override` have `SOURCE` retention?" — because it's only useful for the compiler, not at runtime. Whereas `@RequestMapping` (Spring) has `RUNTIME` because Spring reads it reflectively to configure routing.

</details>

---

**Q5.** 🟡 What are `ElementType` targets for annotations?

<details><summary>Click to reveal answer</summary>

```java
@Target({
    ElementType.TYPE,             // class, interface, enum, record
    ElementType.FIELD,            // instance/static fields
    ElementType.METHOD,           // methods
    ElementType.PARAMETER,        // method parameters
    ElementType.CONSTRUCTOR,      // constructors
    ElementType.LOCAL_VARIABLE,   // local variables (SOURCE only useful)
    ElementType.ANNOTATION_TYPE,  // other annotations (meta-annotations)
    ElementType.PACKAGE,          // package-info.java
    ElementType.TYPE_PARAMETER,   // generic type parameters <T>
    ElementType.TYPE_USE,         // any type use (Java 8+)
    ElementType.MODULE,           // module declarations (Java 9+)
    ElementType.RECORD_COMPONENT  // record components (Java 16+)
})
```

**`TYPE_USE` examples:**
```java
// With TYPE_USE target:
List<@NotNull String> names;
@NonNull String name = (@NonNull String) obj;
```

</details>

---

**Q6.** 🟡 What is `@Repeatable` and how does it work?

<details><summary>Click to reveal answer</summary>

`@Repeatable` (Java 8+) allows the same annotation to appear multiple times on the same element. You need a **container annotation** to hold the array.

```java
// Step 1: Create the container annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Schedules {
    Schedule[] value(); // must have value() returning array of the repeatable annotation
}

// Step 2: Create the repeatable annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(Schedules.class) // link to container
@interface Schedule {
    String cron();
    String timezone() default "UTC";
}

// Step 3: Use multiple times
@Schedule(cron = "0 0 8 * * MON-FRI")      // weekday morning
@Schedule(cron = "0 0 10 * * SAT", timezone = "America/New_York") // Saturday
public void sendReport() { }

// Reading via reflection
Method m = MyClass.class.getMethod("sendReport");
Schedule[] schedules = m.getAnnotationsByType(Schedule.class); // handles unwrapping
for (Schedule s : schedules) {
    System.out.println(s.cron() + " @ " + s.timezone());
}

// Or get the container
Schedules container = m.getAnnotation(Schedules.class);
```

</details>

---

### Meta-Annotations

**Q7.** 🟢 What are meta-annotations?

<details><summary>Click to reveal answer</summary>

Meta-annotations are annotations that annotate **other annotations**. They control how annotation types behave.

| Meta-annotation | Purpose |
|---|---|
| `@Retention` | How long the annotation is kept |
| `@Target` | Where the annotation can be applied |
| `@Documented` | Include in Javadoc |
| `@Inherited` | Subclasses inherit the annotation |
| `@Repeatable` | Allow multiple instances on one element |

```java
// @Inherited example
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Category {
    String value();
}

@Category("Service")
class BaseService { }

class UserService extends BaseService { }

// UserService INHERITS @Category from BaseService:
UserService.class.getAnnotation(Category.class); // returns @Category("Service")
// But ONLY works on class-level, NOT method/field annotations
```

🚨 **Common Mistake:** `@Inherited` only affects class-level annotations and only through `class` inheritance — not through interface implementation.

</details>

---

### Reflection API

**Q8.** 🟡 What is the Reflection API and what can it do?

<details><summary>Click to reveal answer</summary>

The Reflection API (`java.lang.reflect`) allows runtime inspection and manipulation of classes, methods, fields, and constructors.

**Capabilities:**
- Inspect class structure (fields, methods, constructors, annotations)
- Invoke methods dynamically (including private ones)
- Create instances without knowing the class at compile time
- Access and modify fields (including private and final)
- Create arrays of any type at runtime

```java
Class<?> clazz = Class.forName("com.example.UserService");

// Inspect
String name = clazz.getName();
Class<?> superclass = clazz.getSuperclass();
Class<?>[] interfaces = clazz.getInterfaces();
Annotation[] annotations = clazz.getAnnotations();

// Fields
Field[] fields = clazz.getDeclaredFields(); // all, including private
Field field = clazz.getDeclaredField("name");
field.setAccessible(true); // bypass access control
Object value = field.get(instance);
field.set(instance, "new value");

// Methods
Method[] methods = clazz.getDeclaredMethods();
Method method = clazz.getDeclaredMethod("processUser", Long.class, String.class);
method.setAccessible(true);
Object result = method.invoke(instance, 42L, "Alice");

// Constructors
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class);
ctor.setAccessible(true);
Object obj = ctor.newInstance("arg");
```

</details>

---

**Q9.** 🟡 How do you get a `Class` object in Java?

<details><summary>Click to reveal answer</summary>

```java
// 1. Class literal (compile-time, no exception)
Class<String> c1 = String.class;
Class<int[]> c2 = int[].class;
Class<Void> c3 = void.class;

// 2. getClass() on instance (returns runtime type)
String s = "hello";
Class<?> c4 = s.getClass(); // java.lang.String

// 3. Class.forName() (runtime, may throw ClassNotFoundException)
Class<?> c5 = Class.forName("java.lang.String");
Class<?> c6 = Class.forName("[Ljava.lang.String;"); // String[]

// 4. ClassLoader
Class<?> c7 = Thread.currentThread().getContextClassLoader()
                   .loadClass("com.example.MyClass");

// 5. From a type token
public <T> Class<T> getClass(T obj) {
    return (Class<T>) obj.getClass();
}
```

**Difference between `getClass()` and `.class`:**
```java
List<String> list = new ArrayList<>();
list.getClass(); // class java.util.ArrayList (runtime type)
List.class;      // interface java.util.List (compile-time type)
```

</details>

---

**Q10.** 🟡 What is `getDeclaredMethods()` vs `getMethods()`?

<details><summary>Click to reveal answer</summary>

| Method | Returns | Includes private? | Includes inherited? |
|---|---|---|---|
| `getMethods()` | `public` methods only | No | Yes (from superclass and interfaces) |
| `getDeclaredMethods()` | All access methods | Yes | No (only declared in this class) |

```java
public class Parent {
    public void parentPublic() {}
    private void parentPrivate() {}
}

public class Child extends Parent {
    public void childPublic() {}
    private void childPrivate() {}
    protected void childProtected() {}
}

Class<Child> c = Child.class;

// getMethods(): public only, includes inherited
// [childPublic, parentPublic, Object.equals, Object.hashCode, Object.toString, ...]
Method[] methods = c.getMethods();

// getDeclaredMethods(): all access, this class only
// [childPublic, childPrivate, childProtected]
Method[] declared = c.getDeclaredMethods();

// To get private method from parent class
Method parentPrivate = Parent.class.getDeclaredMethod("parentPrivate");
parentPrivate.setAccessible(true);
parentPrivate.invoke(childInstance);
```

</details>

---

**Q11.** 🔴 What are the performance implications of Reflection?

<details><summary>Click to reveal answer</summary>

Reflection is significantly slower than direct code because:
1. **No JIT optimization** — reflective calls bypass most JIT optimizations
2. **Dynamic dispatch overhead** — argument boxing, type checking, security checks
3. **`setAccessible(true)`** — bypasses access control but adds overhead
4. **`Class.forName()`** — class loading and caching cost

**Benchmark (approximate):**
| Operation | Relative Cost |
|---|---|
| Direct method call | 1x |
| Cached reflective call | 10–50x |
| Uncached `Class.forName()` + invoke | 100–1000x |

**Optimization strategies:**
```java
// BAD: looking up method every call
public Object invoke(Object target, String methodName, Object... args) throws Exception {
    return target.getClass().getMethod(methodName, ...).invoke(target, args);
}

// GOOD: cache Method objects
private final Map<String, Method> methodCache = new ConcurrentHashMap<>();

public Object invoke(Object target, String methodName, Object... args) throws Exception {
    Method method = methodCache.computeIfAbsent(methodName,
        name -> findMethod(target.getClass(), name));
    return method.invoke(target, args);
}

// BETTER: use MethodHandle (Java 7+) - JIT-friendly, ~direct call speed
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle mh = lookup.findVirtual(MyClass.class, "myMethod",
    MethodType.methodType(String.class, int.class));
String result = (String) mh.invoke(instance, 42); // nearly as fast as direct call
```

</details>

---

**Q12.** 🔴 How do `MethodHandle` (Java 7+) and `VarHandle` (Java 9+) improve on Reflection?

<details><summary>Click to reveal answer</summary>

**`MethodHandle`** — a typed, directly executable reference to a method, constructor, or field that can be fully optimized by the JIT.

```java
// MethodHandle - faster than reflection, JIT-optimizable
MethodHandles.Lookup lookup = MethodHandles.lookup();

// Method handle for instance method
MethodHandle mh = lookup.findVirtual(String.class, "substring",
    MethodType.methodType(String.class, int.class, int.class));
String sub = (String) mh.invoke("Hello, World!", 0, 5); // "Hello"

// Constructor handle
MethodHandle ctor = lookup.findConstructor(ArrayList.class,
    MethodType.methodType(void.class, int.class));
List<String> list = (List<String>) ctor.invoke(10);

// Static method
MethodHandle parseInt = lookup.findStatic(Integer.class, "parseInt",
    MethodType.methodType(int.class, String.class));
int n = (int) parseInt.invoke("42");
```

**`VarHandle`** — provides atomic and memory-ordered access to variables (fields, array elements).

```java
class Counter {
    private volatile int count;
    private static final VarHandle COUNT;
    static {
        try {
            COUNT = MethodHandles.lookup().findVarHandle(Counter.class, "count", int.class);
        } catch (ReflectiveOperationException e) { throw new ExceptionInInitializerError(e); }
    }

    public boolean compareAndSet(int expected, int newValue) {
        return COUNT.compareAndSet(this, expected, newValue); // atomic CAS
    }
}
```

</details>

---

**Q13.** 🔴 How does Reflection handle generic type information?

<details><summary>Click to reveal answer</summary>

Due to **type erasure**, generic type information is partially lost at runtime. However, it's preserved in some places — method signatures, field types, superclass generics.

```java
public class UserRepository extends GenericRepository<User, Long> {
    private List<String> tags;

    public List<User> findByIds(List<Long> ids) { return null; }
}

// Get generic supertype
Type genericSuper = UserRepository.class.getGenericSuperclass();
// ParameterizedType: GenericRepository<User, Long>
if (genericSuper instanceof ParameterizedType pt) {
    Type[] typeArgs = pt.getActualTypeArguments();
    // typeArgs[0] = User.class, typeArgs[1] = Long.class
}

// Get generic field type
Field tagsField = UserRepository.class.getDeclaredField("tags");
Type genericType = tagsField.getGenericType();
// ParameterizedType: List<String>
if (genericType instanceof ParameterizedType pt) {
    Type elementType = pt.getActualTypeArguments()[0]; // String.class
}

// Get generic return type
Method method = UserRepository.class.getMethod("findByIds", List.class);
Type returnType = method.getGenericReturnType(); // List<User>

// Method parameter generic types
Type paramType = method.getGenericParameterTypes()[0]; // List<Long>
```

**Super type token pattern (used by Jackson, Guice):**
```java
// Capture generic type info via anonymous class
Type type = new TypeReference<List<User>>(){}.getType();
// Works because anonymous class's generic supertype is preserved
```

</details>

---

### Dynamic Proxies

**Q14.** 🟡 What is a Java Dynamic Proxy and how does it work?

<details><summary>Click to reveal answer</summary>

A dynamic proxy is a class created at **runtime** that implements one or more interfaces and delegates all method calls to an `InvocationHandler`. Used by Spring AOP, Hibernate lazy loading, mock frameworks.

```java
public interface UserService {
    User findById(Long id);
    void save(User user);
}

// InvocationHandler intercepts all method calls
public class LoggingInvocationHandler implements InvocationHandler {
    private final Object target;

    public LoggingInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.nanoTime();
        System.out.println("Calling " + method.getName() + " with " + Arrays.toString(args));
        try {
            Object result = method.invoke(target, args); // delegate to real object
            long elapsed = System.nanoTime() - start;
            System.out.println("Returned in " + elapsed + "ns: " + result);
            return result;
        } catch (InvocationTargetException e) {
            throw e.getCause(); // unwrap to get original exception
        }
    }
}

// Create proxy
UserServiceImpl realService = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{ UserService.class },
    new LoggingInvocationHandler(realService)
);

// All calls on proxy go through InvocationHandler
proxy.findById(42L); // logged and timed
```

🚨 **Common Mistake:** JDK dynamic proxies only work with **interfaces**. For class-based proxying, you need CGLIB or ByteBuddy (which Spring uses when no interface is available).

</details>

---

**Q15.** 🔴 How does Spring AOP use dynamic proxies?

<details><summary>Click to reveal answer</summary>

Spring creates proxy beans that intercept method calls to apply cross-cutting concerns (transactions, security, logging).

```
Original Call                Spring Proxy                 Target Bean
-----------     --------->   ----------------             --------
service.save()  -----------> TransactionProxy.save()  --> RealService.save()
                             (begin transaction)
                             (call real method)
                             (commit/rollback)
```

**Two proxy mechanisms:**
1. **JDK Dynamic Proxy** — used when the bean implements an interface
2. **CGLIB Proxy** — used when no interface (subclasses the class)

```java
@Service
public class UserService {  // No interface - Spring uses CGLIB

    @Transactional  // Spring wraps in transaction proxy
    public void transferMoney(Long from, Long to, BigDecimal amount) {
        debit(from, amount);
        credit(to, amount);
    }

    // Self-invocation problem:
    @Transactional
    public void methodA() {
        methodB(); // This does NOT go through the proxy!
                   // Spring AOP only intercepts external calls
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() { ... } // NOT called through proxy from methodA
}
```

🚨 **Common Mistake:** Self-invocation (calling another `@Transactional` method from within the same bean) bypasses the proxy — the transaction annotation is ignored.

</details>

---

**Q16.** 🔴 What is the security risk of `setAccessible(true)`?

<details><summary>Click to reveal answer</summary>

`setAccessible(true)` bypasses Java's access control — it can access `private` fields and methods, break encapsulation, and expose sensitive data.

**Java 9+ Module System mitigations:**
```java
// Before Java 9: setAccessible(true) always worked
field.setAccessible(true); // could access any private field

// Java 9+: setAccessible(true) may throw InaccessibleObjectException
// if the module doesn't export the package
// --add-opens java.base/java.lang=ALL-UNNAMED (JVM arg to re-enable)

// Checking reflective access
try {
    field.setAccessible(true);
} catch (InaccessibleObjectException e) {
    // Module system blocked access
}
```

**Security implications:**
```java
// Could read private fields (e.g., password hash in user objects)
Field passwordField = User.class.getDeclaredField("passwordHash");
passwordField.setAccessible(true);
String hash = (String) passwordField.get(userInstance);

// Could access private encryption keys
Field keyField = CryptoService.class.getDeclaredField("privateKey");
keyField.setAccessible(true);
PrivateKey key = (PrivateKey) keyField.get(cryptoServiceInstance);

// Mitigation: SecurityManager (deprecated in Java 17)
// Java 9+ modules provide better protection
// Mark sensitive classes/fields with module encapsulation
```

✅ **Best Practice:** Minimize use of `setAccessible(true)` in application code. It's primarily for frameworks and testing utilities.

</details>

---

**Q17.** 🟡 How do you read annotations at runtime using Reflection?

<details><summary>Click to reveal answer</summary>

```java
// Class-level annotation
@Audit(action = "USER_SERVICE")
public class UserService {

    @Audit(action = "DELETE", level = AuditLevel.CRITICAL)
    @Deprecated
    public void deleteUser(Long id) { }
}

// Reading annotations
Class<UserService> clazz = UserService.class;

// Check if annotation present
boolean hasAudit = clazz.isAnnotationPresent(Audit.class);

// Get single annotation
Audit classAudit = clazz.getAnnotation(Audit.class);
if (classAudit != null) {
    System.out.println(classAudit.action()); // "USER_SERVICE"
}

// Get all annotations
Annotation[] all = clazz.getAnnotations(); // includes inherited
Annotation[] declared = clazz.getDeclaredAnnotations(); // only on this class

// Method-level
Method method = clazz.getMethod("deleteUser", Long.class);
Audit methodAudit = method.getAnnotation(Audit.class);
System.out.println(methodAudit.level()); // AuditLevel.CRITICAL

// Parameter annotations
Method m = clazz.getMethod("someMethod", String.class, int.class);
Annotation[][] paramAnnotations = m.getParameterAnnotations();
// paramAnnotations[0] = annotations on first parameter
// paramAnnotations[1] = annotations on second parameter

// Repeatable annotations
Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
```

</details>

---

**Q18.** 🟡 How do you implement an annotation-based validation framework?

<details><summary>Click to reveal answer</summary>

```java
// Define validation annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface NotNull { String message() default "Field must not be null"; }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface MinLength {
    int value();
    String message() default "Too short";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Range {
    int min(); int max();
}

// Object to validate
public class RegistrationForm {
    @NotNull
    @MinLength(3)
    private String username;

    @NotNull
    @MinLength(8)
    private String password;

    @Range(min = 0, max = 150)
    private int age;
}

// Validator implementation
public class BeanValidator {
    public List<String> validate(Object obj) throws IllegalAccessException {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = obj.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            Object value = field.get(obj);

            if (field.isAnnotationPresent(NotNull.class) && value == null) {
                errors.add(field.getName() + ": " +
                    field.getAnnotation(NotNull.class).message());
            }

            if (field.isAnnotationPresent(MinLength.class) && value instanceof String s) {
                int min = field.getAnnotation(MinLength.class).value();
                if (s.length() < min) {
                    errors.add(field.getName() + ": must be at least " + min + " chars");
                }
            }

            if (field.isAnnotationPresent(Range.class) && value instanceof Number n) {
                Range range = field.getAnnotation(Range.class);
                int v = n.intValue();
                if (v < range.min() || v > range.max()) {
                    errors.add(field.getName() + ": must be between " + range.min() + " and " + range.max());
                }
            }
        }
        return errors;
    }
}
```

</details>

---

### Annotation Processors

**Q19.** 🔴 What is an Annotation Processor and how does it work?

<details><summary>Click to reveal answer</summary>

Annotation Processors run at **compile time** (during `javac`) to generate code, validate annotations, or produce resources. They use the `javax.annotation.processing` API.

```java
@SupportedAnnotationTypes("com.example.GenerateBuilder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (TypeElement annotation : annotations) {
            for (Element element : roundEnv.getElementsAnnotatedWith(annotation)) {
                if (element.getKind() == ElementKind.CLASS) {
                    generateBuilder((TypeElement) element);
                }
            }
        }
        return true; // annotation claimed
    }

    private void generateBuilder(TypeElement classElement) {
        String className = classElement.getSimpleName() + "Builder";
        // Use processingEnv.getFiler() to write source files
        try {
            JavaFileObject file = processingEnv.getFiler()
                .createSourceFile(className, classElement);
            try (Writer writer = file.openWriter()) {
                writer.write("public class " + className + " { ... }");
            }
        } catch (IOException e) {
            processingEnv.getMessager().printMessage(
                Diagnostic.Kind.ERROR, e.getMessage(), classElement);
        }
    }
}
```

**Registration:** Add to `META-INF/services/javax.annotation.processing.Processor`

**Examples in the wild:**
- **Lombok** — generates getters, setters, builders at compile time
- **MapStruct** — generates type-safe mapper implementations
- **Dagger** — generates dependency injection code
- **AutoValue** — generates immutable value classes

</details>

---

**Q20.** 🟡 How does Lombok work internally?

<details><summary>Click to reveal answer</summary>

Lombok uses a non-standard approach: it **modifies the AST (Abstract Syntax Tree)** during compilation rather than generating separate source files. This is possible because it hooks into javac's internal APIs.

```java
// What you write:
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private String email;
}

// What Lombok generates (added to the class's AST):
// - getters for all fields
// - setters for non-final fields
// - equals(), hashCode(), toString()
// - Builder inner class with fluent API
// - Default constructor
// - All-args constructor
```

**Why Lombok is controversial:**
- Uses internal javac APIs (`com.sun.tools.javac.*`) that can break with Java upgrades
- IDEs need a plugin to understand Lombok-generated code
- Debugging can be confusing (generated code not visible in source)
- But: massive productivity gain for boilerplate-heavy classes

</details>

---

**Q21.** 🟡 What is the difference between annotations and XML configuration in frameworks?

<details><summary>Click to reveal answer</summary>

| Aspect | Annotations | XML Configuration |
|---|---|---|
| Location | Close to code | Separate file |
| Compile-time checking | Partial (type safety for values) | None |
| Verbosity | Less verbose | More verbose |
| Separation of concerns | Configuration mixed with code | Separated |
| Dynamic reconfiguration | Requires recompile | Can change without recompile |
| Discoverability | Easy (near the code) | Requires separate file lookup |

```java
// Spring Bean - annotation style
@Component
@Scope("singleton")
@Lazy
public class UserRepository {
    @Autowired
    private DataSource dataSource;
}

// Spring Bean - XML style (equivalent)
// <bean id="userRepository" class="com.example.UserRepository"
//       scope="singleton" lazy-init="true">
//     <property name="dataSource" ref="dataSource"/>
// </bean>
```

**Modern recommendation:** Annotations for most configuration, external files (YAML/Properties) for environment-specific values (URLs, passwords).

</details>

---

**Q22.** 🟡 How do you find all classes annotated with a specific annotation in a package?

<details><summary>Click to reveal answer</summary>

Java's reflection API doesn't support "find all classes" directly — you need to scan the classpath.

```java
// Option 1: Reflections library (3rd party)
Reflections reflections = new Reflections("com.mypackage");
Set<Class<?>> annotated = reflections.getTypesAnnotatedWith(MyAnnotation.class);

// Option 2: Manual classpath scanning
public Set<Class<?>> findAnnotatedClasses(String packageName, Class<? extends Annotation> annotation)
        throws IOException, ClassNotFoundException {
    Set<Class<?>> result = new HashSet<>();
    String path = packageName.replace('.', '/');
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    Enumeration<URL> resources = cl.getResources(path);

    while (resources.hasMoreElements()) {
        URL resource = resources.nextElement();
        File dir = new File(resource.getFile());
        for (File file : dir.listFiles(f -> f.getName().endsWith(".class"))) {
            String className = packageName + '.' + file.getName().replace(".class", "");
            Class<?> clazz = Class.forName(className);
            if (clazz.isAnnotationPresent(annotation)) {
                result.add(clazz);
            }
        }
    }
    return result;
}

// Option 3: Spring's ClassPathScanningCandidateComponentProvider
ClassPathScanningCandidateComponentProvider scanner =
    new ClassPathScanningCandidateComponentProvider(false);
scanner.addIncludeFilter(new AnnotationTypeFilter(MyAnnotation.class));
Set<BeanDefinition> candidates = scanner.findCandidateComponents("com.mypackage");
```

</details>

---

**Q23.** 🟡 Explain how JUnit 5 uses annotations internally.

<details><summary>Click to reveal answer</summary>

JUnit 5 reads annotations via reflection to discover and configure tests:

```java
// JUnit 5 annotations (simplified internal processing):
// @Test - marks method as a test
// @BeforeEach - run before each test method
// @ParameterizedTest + @ValueSource - run test with multiple values

// Internal JUnit processing (simplified):
public class JUnitTestRunner {
    public void runTestClass(Class<?> testClass) throws Exception {
        Object instance = testClass.getDeclaredConstructor().newInstance();

        List<Method> beforeEachMethods = Arrays.stream(testClass.getDeclaredMethods())
            .filter(m -> m.isAnnotationPresent(BeforeEach.class))
            .toList();

        List<Method> testMethods = Arrays.stream(testClass.getDeclaredMethods())
            .filter(m -> m.isAnnotationPresent(Test.class))
            .toList();

        for (Method testMethod : testMethods) {
            // Run @BeforeEach
            for (Method before : beforeEachMethods) {
                before.invoke(instance);
            }
            // Run test (catch and report failures)
            try {
                testMethod.invoke(instance);
                System.out.println("PASS: " + testMethod.getName());
            } catch (InvocationTargetException e) {
                System.out.println("FAIL: " + testMethod.getName() + " - " + e.getCause());
            }
        }
    }
}
```

</details>

---

**Q24.** 🟡 How do you create an annotation with `String[]` elements?

<details><summary>Click to reveal answer</summary>

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Roles {
    String[] value() default {};         // empty array default
    String[] exclude() default {};
}

// Usage - single value shortcut
@Roles("ADMIN")    // shorthand when only 'value' is set
class AdminController { }

// Usage - multiple values
@Roles(value = {"ADMIN", "MODERATOR"}, exclude = {"GUEST"})
class ManagementController { }

// Reading
Roles roles = ManagementController.class.getAnnotation(Roles.class);
String[] allowed = roles.value();  // ["ADMIN", "MODERATOR"]
String[] excluded = roles.exclude(); // ["GUEST"]
```

**Note:** When an annotation has only one element named `value`, you can omit the `value =` in the usage.

</details>

---

**Q25.** 🔴 Explain the ClassLoader hierarchy and how Reflection interacts with it.

<details><summary>Click to reveal answer</summary>

```
Bootstrap ClassLoader (C code, loads java.lang.*, java.io.*, etc.)
    ↑ parent
Extension/Platform ClassLoader (loads javax.*, java.sql.*, etc.)
    ↑ parent
Application/System ClassLoader (loads classpath classes)
    ↑ parent
Custom ClassLoaders (e.g., web app loaders in Tomcat)
```

**Reflection and ClassLoaders:**
```java
// Class.forName uses the calling class's ClassLoader by default
Class<?> c1 = Class.forName("com.example.MyClass");

// Explicit ClassLoader
Class<?> c2 = Class.forName("com.example.MyClass", true,
    Thread.currentThread().getContextClassLoader()); // TCCL

// Loading from a different classloader - useful for plugins
URL[] urls = { new URL("file:///plugins/myplugin.jar") };
URLClassLoader pluginLoader = new URLClassLoader(urls, getClass().getClassLoader());
Class<?> pluginClass = pluginLoader.loadClass("com.plugin.MyPlugin");
```

**TCCL (Thread Context ClassLoader):** Frameworks like JNDI, JDBC, and Spring use TCCL to load classes in the correct application context, especially in container environments (Tomcat, JBoss).

</details>

---

**Q26.** 🟡 What is annotation inheritance and how does `@Inherited` work?

<details><summary>Click to reveal answer</summary>

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface ServiceType {
    String value();
}

@ServiceType("business")
class BaseService { }

class OrderService extends BaseService { } // inherits @ServiceType

class ReportService extends BaseService {
    // Overrides — this class's @ServiceType "replaces" inherited one if present
}

// Lookup
ServiceType type = OrderService.class.getAnnotation(ServiceType.class);
// Returns @ServiceType("business") — inherited

// BUT: @Inherited does NOT work for:
// - Interface annotations
// - Method/field annotations
// - Implementation of annotated interface
```

</details>

---

**Q27.** 🟡 How do frameworks use `@interface` to create DSL-like APIs?

<details><summary>Click to reveal answer</summary>

```java
// JPA entities defined purely through annotations
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_email", columnList = "email", unique = true),
    @Index(name = "idx_lastname", columnList = "last_name")
})
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "email", nullable = false, length = 255)
    @NotNull
    @Email
    private String email;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders;
}

// Spring MVC routing DSL via annotations
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    @GetMapping("/{id}")
    @ResponseStatus(HttpStatus.OK)
    @Cacheable(value = "users", key = "#id")
    public UserDto getUser(@PathVariable Long id,
                           @RequestParam(defaultValue = "false") boolean includeOrders) {
        return userService.findById(id);
    }
}
```

</details>

---

**Q28.** 🟡 What is the difference between `getAnnotation()` and `getDeclaredAnnotation()`?

<details><summary>Click to reveal answer</summary>

| Method | Returns |
|---|---|
| `getAnnotation(Class)` | Annotation if present (including `@Inherited` annotations) |
| `getDeclaredAnnotation(Class)` | Only annotations directly declared on this element |
| `getAnnotations()` | All annotations (including inherited) |
| `getDeclaredAnnotations()` | Only annotations directly declared |
| `getAnnotationsByType(Class)` | All annotations of type (handles `@Repeatable`) |
| `getDeclaredAnnotationsByType(Class)` | Same but only declared |

```java
@ServiceType("base")
class Parent { }

class Child extends Parent { }

// getAnnotation follows @Inherited
ServiceType t1 = Child.class.getAnnotation(ServiceType.class); // @ServiceType("base")

// getDeclaredAnnotation does NOT follow @Inherited
ServiceType t2 = Child.class.getDeclaredAnnotation(ServiceType.class); // null
```

</details>

---

**Q29.** 🔴 What is the Proxy pattern vs Dynamic Proxy in Java?

<details><summary>Click to reveal answer</summary>

**Proxy pattern** (design pattern): A structural pattern where a proxy object controls access to the real object.

**Java Dynamic Proxy**: A runtime implementation of the proxy pattern using `java.lang.reflect.Proxy`.

```java
// Static Proxy - manually written
public class CachingUserService implements UserService {
    private final UserService delegate;
    private final Map<Long, User> cache = new HashMap<>();

    public CachingUserService(UserService delegate) { this.delegate = delegate; }

    @Override
    public User findById(Long id) {
        return cache.computeIfAbsent(id, delegate::findById);
    }
}

// Dynamic Proxy - generated at runtime
UserService cachingProxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{ UserService.class },
    (proxy, method, args) -> {
        if ("findById".equals(method.getName())) {
            Long id = (Long) args[0];
            return cache.computeIfAbsent(id, k -> {
                try { return (User) method.invoke(realService, k); }
                catch (Exception e) { throw new RuntimeException(e); }
            });
        }
        return method.invoke(realService, args);
    }
);
```

**Dynamic proxy advantages:** Works for any interface without writing boilerplate; apply cross-cutting concerns generically.

</details>

---

**Q30.** 🟡 What is reflection's role in dependency injection frameworks?

<details><summary>Click to reveal answer</summary>

Dependency injection frameworks use reflection extensively:

1. **Discovery:** Scan classpath for `@Component`, `@Service`, `@Repository` classes
2. **Instantiation:** Create instances via `Constructor.newInstance()`
3. **Injection:** Find `@Autowired` fields/constructors, inject dependencies
4. **AOP:** Create proxies for `@Transactional`, `@Cacheable`, etc.

```java
// Simplified Spring-like DI container
public class SimpleContainer {
    private final Map<Class<?>, Object> beans = new HashMap<>();

    public <T> void register(Class<T> clazz) throws Exception {
        // Find injectable constructor (or no-arg)
        Constructor<?> ctor = Arrays.stream(clazz.getDeclaredConstructors())
            .filter(c -> c.isAnnotationPresent(Inject.class))
            .findFirst()
            .orElse(clazz.getDeclaredConstructor());

        // Resolve constructor parameters
        Object[] params = Arrays.stream(ctor.getParameterTypes())
            .map(this::getBean)
            .toArray();

        Object instance = ctor.newInstance(params);

        // Field injection
        for (Field field : clazz.getDeclaredFields()) {
            if (field.isAnnotationPresent(Inject.class)) {
                field.setAccessible(true);
                field.set(instance, getBean(field.getType()));
            }
        }

        beans.put(clazz, instance);
    }

    public <T> T getBean(Class<T> type) {
        return type.cast(beans.get(type));
    }
}
```

</details>

---

**Q31–Q42.** Additional theory questions (brief answers):

**Q31.** 🟢 Can annotations have methods with throws? → No; annotation elements cannot declare exceptions.

**Q32.** 🟡 What happens if you annotate a constructor with `@Deprecated`? → Compiler warns on invocation; visible in IDE.

**Q33.** 🟡 Can annotation elements return `null`? → No; `null` is not a valid element value. Use special sentinels.

**Q34.** 🔴 What is `Module.getAnnotation()` in Java 9+ modules? → Annotations can be placed on `module-info.java` and read via `Module.getAnnotation()`.

**Q35.** 🟡 What is `AnnotatedElement` interface? → Superinterface for `Class`, `Method`, `Field`, `Constructor`, etc. — all things that can have annotations.

**Q36.** 🟡 How to get annotation from a method in a superclass? → `getDeclaredMethods()` on each class in the hierarchy; annotations on methods are NOT inherited.

**Q37.** 🔴 What is `MethodHandles.privateLookupIn()`? → Java 9+ way to get a lookup with private access to a class (replaces `setAccessible(true)` for modules).

**Q38.** 🟢 Can you annotate an annotation? → Yes, using meta-annotations (`@Retention`, `@Target`, etc.) or custom annotations with `@Target(ElementType.ANNOTATION_TYPE)`.

**Q39.** 🟡 What is `@Contended` annotation? → JVM hint to add padding around a field to avoid false sharing on cache lines (in `sun.misc` / `jdk.internal.vm.annotation`).

**Q40.** 🟡 How does Jackson use Reflection? → Scans fields/methods for `@JsonProperty`, `@JsonIgnore`, etc.; uses `Method.invoke()` or `MethodHandle` for serialization.

**Q41.** 🔴 What is the "annotation tax" on method parameters? → Annotation info stored in class file; many annotations on many parameters = larger class files.

**Q42.** 🟡 What is `Class.isInstance()` vs `instanceof`? → `isInstance()` is the reflective equivalent: `String.class.isInstance(obj)` == `obj instanceof String`.

---

## Scenario-Based Questions

**S1.** 🟡 You need to build a simple ORM that maps objects to database tables using annotations. How would you design it?

<details><summary>Click to reveal answer</summary>

```java
// Define ORM annotations
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.TYPE)
@interface Table { String name(); }

@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.FIELD)
@interface Column { String name() default ""; boolean primaryKey() default false; }

// Entity
@Table(name = "products")
public class Product {
    @Column(name = "id", primaryKey = true)
    private Long id;

    @Column(name = "product_name")
    private String name;

    @Column // uses field name as column name
    private double price;
}

// ORM query builder
public class SimpleOrm {
    public String buildInsertSql(Object entity) throws IllegalAccessException {
        Class<?> clazz = entity.getClass();
        Table table = clazz.getAnnotation(Table.class);
        if (table == null) throw new IllegalArgumentException("Not an ORM entity");

        List<String> columns = new ArrayList<>();
        List<String> values = new ArrayList<>();

        for (Field field : clazz.getDeclaredFields()) {
            Column col = field.getAnnotation(Column.class);
            if (col == null || col.primaryKey()) continue;

            field.setAccessible(true);
            String colName = col.name().isEmpty() ? field.getName() : col.name();
            columns.add(colName);
            values.add("'" + field.get(entity) + "'");
        }

        return "INSERT INTO " + table.name() +
               " (" + String.join(", ", columns) + ")" +
               " VALUES (" + String.join(", ", values) + ")";
    }
}

// Output: INSERT INTO products (product_name, price) VALUES ('Widget', '9.99')
```

</details>

---

**S2.** 🔴 How would you implement method-level security using a custom annotation and proxy?

<details><summary>Click to reveal answer</summary>

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface RequiresRole {
    String[] value();
}

public class SecurityProxy implements InvocationHandler {
    private final Object target;
    private final Supplier<Set<String>> currentUserRoles;

    public SecurityProxy(Object target, Supplier<Set<String>> currentUserRoles) {
        this.target = target;
        this.currentUserRoles = currentUserRoles;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        RequiresRole annotation = method.getAnnotation(RequiresRole.class);
        if (annotation != null) {
            Set<String> userRoles = currentUserRoles.get();
            boolean authorized = Arrays.stream(annotation.value())
                .anyMatch(userRoles::contains);

            if (!authorized) {
                throw new SecurityException("Access denied to " + method.getName() +
                    ". Required roles: " + Arrays.toString(annotation.value()) +
                    ", User roles: " + userRoles);
            }
        }
        try {
            return method.invoke(target, args);
        } catch (InvocationTargetException e) {
            throw e.getCause();
        }
    }

    public static <T> T wrap(T target, Class<?>[] interfaces, Supplier<Set<String>> roleSupplier) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            interfaces,
            new SecurityProxy(target, roleSupplier));
    }
}

// Usage
interface AdminService {
    @RequiresRole({"ADMIN", "SUPERUSER"})
    void deleteAllUsers();

    @RequiresRole("VIEWER")
    List<User> listUsers();
}

AdminService secured = SecurityProxy.wrap(
    new AdminServiceImpl(),
    new Class[]{AdminService.class},
    () -> currentUser.getRoles()
);
```

</details>

---

**S3.** 🟡 You need to deep-clone an object without implementing `Cloneable` or a copy constructor. How?

<details><summary>Click to reveal answer</summary>

```java
// Reflection-based deep clone
public class ReflectionCloner {
    public static <T> T deepClone(T original) throws Exception {
        Class<?> clazz = original.getClass();

        // Create new instance (requires no-arg constructor)
        T clone = (T) clazz.getDeclaredConstructor().newInstance();

        for (Field field : getAllFields(clazz)) {
            field.setAccessible(true);
            Object value = field.get(original);

            if (value == null || isPrimitive(field.getType())) {
                field.set(clone, value); // primitives/null: direct copy
            } else if (value instanceof Collection<?> coll) {
                // Deep clone collection
                Collection<Object> newColl = (Collection<Object>) value.getClass()
                    .getDeclaredConstructor().newInstance();
                for (Object item : coll) {
                    newColl.add(item.getClass().isPrimitive() ? item : deepClone(item));
                }
                field.set(clone, newColl);
            } else {
                field.set(clone, deepClone(value)); // recursive clone
            }
        }
        return clone;
    }

    private static List<Field> getAllFields(Class<?> clazz) {
        List<Field> fields = new ArrayList<>();
        while (clazz != null && clazz != Object.class) {
            fields.addAll(Arrays.asList(clazz.getDeclaredFields()));
            clazz = clazz.getSuperclass();
        }
        return fields;
    }

    private static boolean isPrimitive(Class<?> type) {
        return type.isPrimitive() || type == String.class ||
               Number.class.isAssignableFrom(type) || type == Boolean.class;
    }
}
```

> 💡 **Note:** For production use, prefer serialization-based clone or dedicated libraries (Kryo, Orika).

</details>

---

**S4.** 🟡 How would you implement a plugin system using Reflection and ClassLoaders?

<details><summary>Click to reveal answer</summary>

```java
// Plugin interface
public interface Plugin {
    String getName();
    void execute(Map<String, Object> context);
}

// Plugin loader
public class PluginLoader {
    private final Path pluginDir;
    private final Map<String, Plugin> loadedPlugins = new ConcurrentHashMap<>();

    public PluginLoader(Path pluginDir) { this.pluginDir = pluginDir; }

    public void loadPlugin(String jarFileName) throws Exception {
        Path jarPath = pluginDir.resolve(jarFileName);
        URL[] urls = { jarPath.toUri().toURL() };

        // Each plugin gets its own classloader for isolation
        URLClassLoader loader = new URLClassLoader(urls, getClass().getClassLoader());

        // Find plugin descriptor
        Properties props = new Properties();
        try (InputStream is = loader.getResourceAsStream("plugin.properties")) {
            if (is == null) throw new IllegalStateException("No plugin.properties");
            props.load(is);
        }

        String mainClass = props.getProperty("plugin.class");
        Class<?> pluginClass = loader.loadClass(mainClass);

        if (!Plugin.class.isAssignableFrom(pluginClass)) {
            throw new IllegalStateException(mainClass + " does not implement Plugin");
        }

        Plugin plugin = (Plugin) pluginClass.getDeclaredConstructor().newInstance();
        loadedPlugins.put(plugin.getName(), plugin);
        System.out.println("Loaded plugin: " + plugin.getName());
    }

    public void executePlugin(String name, Map<String, Object> context) {
        Plugin plugin = loadedPlugins.get(name);
        if (plugin == null) throw new IllegalArgumentException("Plugin not found: " + name);
        plugin.execute(context);
    }
}
```

</details>

---

**S5.** 🔴 How would you implement a caching annotation processor (compile-time)?

<details><summary>Click to reveal answer</summary>

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Cacheable {
    String cacheName();
    int ttlSeconds() default 300;
}

// Runtime processor (using AOP/Proxy - not compile-time for simplicity)
public class CachingHandler implements InvocationHandler {
    private final Object target;
    private final Map<String, Map<String, CachedEntry>> caches = new ConcurrentHashMap<>();

    record CachedEntry(Object value, long expiresAt) {
        boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Cacheable cacheable = method.getAnnotation(Cacheable.class);
        if (cacheable == null) return method.invoke(target, args);

        String cacheKey = Arrays.toString(args);
        Map<String, CachedEntry> cache = caches.computeIfAbsent(
            cacheable.cacheName(), k -> new ConcurrentHashMap<>());

        CachedEntry entry = cache.get(cacheKey);
        if (entry != null && !entry.isExpired()) {
            return entry.value(); // cache hit
        }

        // Cache miss - invoke real method
        Object result;
        try { result = method.invoke(target, args); }
        catch (InvocationTargetException e) { throw e.getCause(); }

        long expiresAt = System.currentTimeMillis() + (cacheable.ttlSeconds() * 1000L);
        cache.put(cacheKey, new CachedEntry(result, expiresAt));
        return result;
    }
}
```

</details>

---

**S6–S30.** Additional scenario topics (brief):

**S6.** 🟢 How to check if a class is abstract using reflection? → `Modifier.isAbstract(clazz.getModifiers())`

**S7.** 🟡 How to invoke a varargs method reflectively? → Pass `Object[]` with last element as `Object[]` for varargs.

**S8.** 🟡 How does Mockito use reflection/proxies? → Creates CGLIB proxy of the class, records `when()` calls, plays back on `verify()`.

**S9.** 🔴 How to access JDK internal APIs in Java 9+? → Use `--add-opens` JVM arg or `MethodHandles.privateLookupIn()`.

**S10.** 🟡 How to detect if code is running in a JAR? → `getClass().getProtectionDomain().getCodeSource().getLocation()`.

**S11.** 🟡 How to get all implemented interfaces recursively? → Walk `getInterfaces()` recursively and `getSuperclass()`.

**S12.** 🔴 How to create a type-safe event bus using annotations? → Register handlers with `@EventHandler`, use reflection to find matching method for event type.

**S13.** 🟡 How to serialize an annotated class to JSON using custom logic? → Scan `@JsonField` annotated fields, build JSON string via reflection.

**S14.** 🟡 When should you avoid reflection? → Performance-critical paths, security-sensitive code; prefer interfaces/design patterns instead.

**S15.** 🟡 How do annotation processors run during incremental builds? → They may run on only changed files; processors must be idempotent.

---

## 🔗 Related Topics
- [Design Patterns](15-design-patterns.md) — Proxy pattern, Decorator pattern
- [Java Testing](12-java-testing.md) — JUnit annotations
- [Spring Framework](18-spring-framework-and-hibernate-intro.md) — Spring annotations

---

*✅ Best Practice: Use annotations for declarative, static metadata. Use Reflection sparingly in frameworks. Cache `Method`/`Field` objects. Prefer `MethodHandle` over raw reflection for performance.*

*🚨 Common Mistake: Forgetting `setAccessible(true)` when accessing private members, or forgetting it can throw `InaccessibleObjectException` in Java 9+ modules.*
