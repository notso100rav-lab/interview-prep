# Java 8 & Functional Programming

> 💡 **Interviewer Tip:** Java 8 questions are nearly universal in backend interviews. Expect deep Stream API questions, lambda behavior, and Optional misuse scenarios. Candidates who truly understand functional programming stand out.

---

## Table of Contents
1. [Lambda Expressions](#lambda-expressions)
2. [Functional Interfaces](#functional-interfaces)
3. [Built-in Functional Interfaces](#built-in-functional-interfaces)
4. [Stream API](#stream-api)
5. [Collectors](#collectors)
6. [Optional](#optional)
7. [Method References](#method-references)
8. [Default & Static Interface Methods](#default--static-interface-methods)
9. [Parallel Streams](#parallel-streams)

---

## Lambda Expressions

### 🟢 Q1. What is a lambda expression in Java?

<details><summary>Click to reveal answer</summary>

A **lambda expression** is an anonymous function — it has parameters, a body, and a return type, but no name. It provides a concise way to implement functional interfaces.

**Syntax:** `(parameters) -> expression` or `(parameters) -> { statements; }`

```java
// Traditional anonymous class
Comparator<String> c1 = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};

// Lambda equivalent
Comparator<String> c2 = (a, b) -> a.length() - b.length();

// Various lambda forms:
Runnable r1 = () -> System.out.println("No params");
Consumer<String> c3 = s -> System.out.println(s);         // One param, no parens needed
BinaryOperator<Integer> add = (x, y) -> x + y;           // Multiple params
Function<String, Integer> len = s -> { return s.length(); }; // Block body
Predicate<Integer> isPos = n -> n > 0;

// With type inference
List<String> names = List.of("Charlie", "Alice", "Bob");
names.sort((a, b) -> a.compareTo(b));
// Or even simpler with method reference:
names.sort(String::compareTo);
```

**Effectively final variables:** Lambdas can capture local variables, but they must be effectively final:
```java
String prefix = "Hello"; // effectively final
Function<String, String> greet = name -> prefix + ", " + name; // OK

String mutable = "Hi";
mutable = "Hey"; // Now not effectively final
Function<String, String> bad = name -> mutable + name; // ❌ Compile error
```

</details>

---

### 🟡 Q2. What is a closure in Java lambdas?

<details><summary>Click to reveal answer</summary>

A **closure** is a lambda that captures variables from its enclosing scope. In Java, captured variables must be **effectively final**.

```java
class ClosureExample {

    public List<Supplier<Integer>> createCounters() {
        List<Supplier<Integer>> counters = new ArrayList<>();

        // ✅ Works — i is effectively final in each iteration
        for (int i = 0; i < 3; i++) {
            final int captured = i; // Must be final
            counters.add(() -> captured * 2);
        }
        return counters;
        // Returns: [() -> 0, () -> 2, () -> 4]
    }

    public void captureInstanceField() {
        // Instance fields CAN be mutated (not local variables)
        // this reference is captured
        String instanceField = this.name;
        Runnable r = () -> System.out.println(instanceField);
    }

    // Anti-pattern: "cheating" with array
    public Supplier<Integer> mutableCapture() {
        int[] counter = {0}; // Array is effectively final, element is mutable
        return () -> counter[0]++;  // Works but is an anti-pattern
    }
}
```

> 💡 **Interviewer Tip:** Ask "Why must captured variables be effectively final?" — Answer: Lambdas may execute in a different thread or at a different time. Mutable shared variables would create race conditions.

</details>

---

## Functional Interfaces

### 🟢 Q3. What is `@FunctionalInterface`?

<details><summary>Click to reveal answer</summary>

A **functional interface** is an interface with exactly **one abstract method** (SAM — Single Abstract Method). The `@FunctionalInterface` annotation enforces this at compile time.

```java
@FunctionalInterface
public interface StringTransformer {
    String transform(String input); // Single abstract method

    // Can have default methods
    default StringTransformer andThen(StringTransformer after) {
        return input -> after.transform(this.transform(input));
    }

    // Can have static methods
    static StringTransformer identity() {
        return input -> input;
    }

    // Can override Object methods
    @Override
    String toString();
}

// Usage
StringTransformer upper = s -> s.toUpperCase();
StringTransformer trim = s -> s.trim();
StringTransformer trimAndUpper = trim.andThen(upper);

System.out.println(trimAndUpper.transform("  hello  ")); // "HELLO"

// ❌ Compile error — two abstract methods breaks @FunctionalInterface
@FunctionalInterface
interface Invalid {
    void method1();
    void method2(); // Error!
}
```

</details>

---

## Built-in Functional Interfaces

### 🟢 Q4. Describe the core built-in functional interfaces in `java.util.function`.

<details><summary>Click to reveal answer</summary>

| Interface | Signature | Description |
|---|---|---|
| `Function<T, R>` | `R apply(T t)` | Transform T to R |
| `Predicate<T>` | `boolean test(T t)` | Test a condition |
| `Consumer<T>` | `void accept(T t)` | Consume value, no return |
| `Supplier<T>` | `T get()` | Produce a value |
| `BiFunction<T, U, R>` | `R apply(T t, U u)` | Two inputs, one output |
| `BiPredicate<T, U>` | `boolean test(T t, U u)` | Two inputs, boolean |
| `BiConsumer<T, U>` | `void accept(T t, U u)` | Two inputs, no return |
| `UnaryOperator<T>` | `T apply(T t)` | Function where T==R |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | BiFunction where T==U==R |

```java
// Function<T, R>
Function<String, Integer> strLength = String::length;
Function<Integer, String> intToStr = Object::toString;
Function<String, String> combined = strLength.andThen(intToStr); // Compose
// combined.apply("hello") -> 5 -> "5"

// Predicate<T>
Predicate<String> isLong = s -> s.length() > 10;
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> combined2 = isLong.and(startsWithA);
Predicate<String> either = isLong.or(startsWithA);
Predicate<String> notLong = isLong.negate();

// Consumer<T>
Consumer<String> printer = System.out::println;
Consumer<String> logger = s -> logger.log(s);
Consumer<String> printAndLog = printer.andThen(logger);

// Supplier<T>
Supplier<List<String>> listFactory = ArrayList::new;
Supplier<LocalDateTime> now = LocalDateTime::now;

// BiFunction<T, U, R>
BiFunction<String, Integer, String> repeat = (s, n) ->
    s.repeat(n);
System.out.println(repeat.apply("ab", 3)); // "ababab"

// UnaryOperator
UnaryOperator<String> trim = String::trim;
UnaryOperator<Integer> doubler = n -> n * 2;
```

</details>

---

### 🟡 Q5. What are primitive specializations of functional interfaces and why do they matter?

<details><summary>Click to reveal answer</summary>

Generic functional interfaces use boxed types (`Integer`, `Double`), causing autoboxing overhead. Primitive specializations avoid this:

```java
// ❌ Autoboxing: int -> Integer -> int
Function<Integer, Integer> boxed = n -> n * 2;
int result = boxed.apply(5); // Boxing and unboxing overhead

// ✅ No boxing with primitive specializations
IntUnaryOperator noBoxing = n -> n * 2;
int result2 = noBoxing.applyAsInt(5); // No boxing

// Key primitive specializations:
IntFunction<String>    intToStr = Integer::toString;
ToIntFunction<String>  strToInt = String::length;
IntToLongFunction      intToLong = n -> (long) n;
IntConsumer            printInt = System.out::println;
IntSupplier            randomInt = () -> new Random().nextInt();
IntPredicate           isEven = n -> n % 2 == 0;
IntBinaryOperator      sum = Integer::sum;

// LongFunction, DoubleFunction variants exist too
DoubleToIntFunction    round = n -> (int) Math.round(n);

// Practical example with streams
int[] numbers = {1, 2, 3, 4, 5};
int sum2 = IntStream.of(numbers)
    .filter(n -> n % 2 == 0)
    .map(n -> n * 2)
    .sum(); // No boxing at all!
```

✅ **Best Practice:** Use primitive streams (`IntStream`, `LongStream`, `DoubleStream`) and primitive functional interfaces for number-heavy operations.

</details>

---

## Stream API

### 🟢 Q6. What is the Java Stream API and what are intermediate vs terminal operations?

<details><summary>Click to reveal answer</summary>

The **Stream API** provides a pipeline-based approach to process sequences of elements. Streams are:
- **Not data structures** — they don't store data
- **Lazy** — intermediate operations execute only when a terminal operation is called
- **Potentially parallel** — can be parallelized with `.parallel()`
- **Consumed once** — cannot be reused after a terminal operation

```
Source → [filter] → [map] → [sorted] → [collect]
         intermediate        intermediate   terminal
         (lazy)              (lazy)         (triggers execution)
```

**Intermediate operations (lazy, return Stream):**
`filter`, `map`, `flatMap`, `distinct`, `sorted`, `peek`, `limit`, `skip`, `mapToInt`, `mapToObj`

**Terminal operations (eager, return result or void):**
`collect`, `forEach`, `reduce`, `count`, `findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch`, `min`, `max`, `toArray`, `sum` (primitive streams)

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave", "Eve");

// Pipeline: filter → map → sorted → collect
List<String> result = names.stream()
    .filter(name -> name.length() > 3)     // intermediate: Alice, Charlie, Dave
    .map(String::toUpperCase)               // intermediate: ALICE, CHARLIE, DAVE
    .sorted()                               // intermediate: ALICE, CHARLIE, DAVE
    .collect(Collectors.toList());          // terminal: triggers execution

System.out.println(result); // [ALICE, CHARLIE, DAVE]

// Count
long count = names.stream()
    .filter(n -> n.startsWith("A"))
    .count(); // 1

// Finding
Optional<String> first = names.stream()
    .filter(n -> n.length() > 4)
    .findFirst(); // Optional[Alice]
```

</details>

---

### 🟢 Q7. How does `filter()` work?

<details><summary>Click to reveal answer</summary>

`filter(Predicate<T>)` keeps only elements that satisfy the predicate:

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Filter even numbers
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList()); // [2, 4, 6, 8, 10]

// Chained filters (applied sequentially)
List<Integer> evenAndBig = numbers.stream()
    .filter(n -> n % 2 == 0)
    .filter(n -> n > 5)
    .collect(Collectors.toList()); // [6, 8, 10]

// Filtering objects
List<User> activeAdmins = users.stream()
    .filter(User::isActive)
    .filter(u -> u.getRole() == Role.ADMIN)
    .collect(Collectors.toList());

// With Predicate composition
Predicate<User> isActive = User::isActive;
Predicate<User> isAdmin = u -> u.getRole() == Role.ADMIN;
List<User> result = users.stream()
    .filter(isActive.and(isAdmin))
    .collect(Collectors.toList());
```

</details>

---

### 🟢 Q8. How does `map()` work and how does it differ from `flatMap()`?

<details><summary>Click to reveal answer</summary>

- `map(Function<T,R>)` — transforms each element 1:1
- `flatMap(Function<T, Stream<R>>)` — transforms each element to a stream, then flattens

```java
List<String> words = List.of("hello", "world", "java");

// map: String -> String (1:1 transformation)
List<String> uppercased = words.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList()); // [HELLO, WORLD, JAVA]

// map: String -> Integer
List<Integer> lengths = words.stream()
    .map(String::length)
    .collect(Collectors.toList()); // [5, 5, 4]

// ❌ map with split produces Stream<String[]> — nested!
List<String[]> nested = words.stream()
    .map(w -> w.split(""))
    .collect(Collectors.toList()); // [[h,e,l,l,o], [w,o,r,l,d], [j,a,v,a]]

// ✅ flatMap flattens: Stream<String[]> -> Stream<String>
List<String> chars = words.stream()
    .flatMap(w -> Arrays.stream(w.split("")))
    .distinct()
    .sorted()
    .collect(Collectors.toList()); // [a, d, e, h, j, l, o, r, v, w]

// Practical: flatten nested lists
List<List<Integer>> nested2 = List.of(
    List.of(1, 2, 3),
    List.of(4, 5),
    List.of(6, 7, 8, 9)
);
List<Integer> flat = nested2.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList()); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

// flatMap with Optional
List<Optional<String>> optionals = List.of(
    Optional.of("A"), Optional.empty(), Optional.of("B"));
List<String> present = optionals.stream()
    .flatMap(Optional::stream)  // Java 9+
    .collect(Collectors.toList()); // [A, B]
```

</details>

---

### 🟡 Q9. How does `reduce()` work?

<details><summary>Click to reveal answer</summary>

`reduce()` combines stream elements into a single result using an accumulator function.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

// reduce with identity and accumulator
int sum = numbers.stream()
    .reduce(0, (acc, n) -> acc + n); // 15
// Same as:
int sum2 = numbers.stream().reduce(0, Integer::sum);

// reduce without identity returns Optional (stream could be empty)
Optional<Integer> max = numbers.stream()
    .reduce((a, b) -> a > b ? a : b); // Optional[5]

// reduce with identity, accumulator, combiner (for parallel streams)
int parallelSum = numbers.parallelStream()
    .reduce(0, Integer::sum, Integer::sum);
// combiner merges partial results from parallel sub-tasks

// Practical: Building a string
String joined = Stream.of("Java", "is", "fun")
    .reduce("", (a, b) -> a.isEmpty() ? b : a + " " + b);
// "Java is fun"
// Better: use Collectors.joining()

// Reduce to find product
long product = LongStream.rangeClosed(1, 5)
    .reduce(1L, (a, b) -> a * b); // 120 (5!)

// Count words (using reduce)
Map<String, Long> wordCount = Arrays.stream("to be or not to be".split(" "))
    .reduce(
        new HashMap<>(),
        (map, word) -> {
            Map<String, Long> m = new HashMap<>(map);
            m.merge(word, 1L, Long::sum);
            return m;
        },
        (m1, m2) -> { m1.putAll(m2); return m1; }
    );
```

🚨 **Common Mistake:** Using `reduce()` when `collect()` is more appropriate for building collections. `reduce()` is ideal for scalar results (sum, max, min, product).

</details>

---

### 🟡 Q10. Explain `sorted()`, `distinct()`, `limit()`, and `skip()`.

<details><summary>Click to reveal answer</summary>

```java
List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5);

// distinct() — removes duplicates (uses equals/hashCode)
List<Integer> unique = nums.stream()
    .distinct()
    .collect(Collectors.toList()); // [3, 1, 4, 5, 9, 2, 6]

// sorted() — natural order
List<Integer> sorted = nums.stream()
    .sorted()
    .collect(Collectors.toList()); // [1, 1, 2, 3, 3, 4, 5, 5, 5, 6, 9]

// sorted(Comparator) — custom order
List<String> names = List.of("Charlie", "Alice", "Bob");
List<String> byLength = names.stream()
    .sorted(Comparator.comparingInt(String::length)
        .thenComparing(Comparator.naturalOrder()))
    .collect(Collectors.toList()); // [Bob, Alice, Charlie]

// limit(n) — take first n elements (short-circuiting)
List<Integer> first3 = nums.stream()
    .distinct()
    .sorted()
    .limit(3)
    .collect(Collectors.toList()); // [1, 2, 3]

// skip(n) — skip first n elements
List<Integer> after3 = nums.stream()
    .distinct()
    .sorted()
    .skip(3)
    .collect(Collectors.toList()); // [4, 5, 6, 9]

// Pagination: page 2, size 3
int page = 2, size = 3;
List<Integer> page2 = Stream.iterate(1, n -> n + 1)
    .limit(20L)
    .skip((long)(page - 1) * size)
    .limit(size)
    .collect(Collectors.toList()); // [4, 5, 6]
```

</details>

---

### 🟡 Q11. How do `anyMatch()`, `allMatch()`, and `noneMatch()` work?

<details><summary>Click to reveal answer</summary>

These are short-circuit terminal operations that evaluate predicates:

```java
List<Integer> nums = List.of(2, 4, 6, 8, 9, 10);

// anyMatch: true if ANY element matches
boolean hasOdd = nums.stream()
    .anyMatch(n -> n % 2 != 0); // true (9 is odd)

// allMatch: true if ALL elements match
boolean allEven = nums.stream()
    .allMatch(n -> n % 2 == 0); // false (9 is odd)

boolean allPositive = nums.stream()
    .allMatch(n -> n > 0); // true

// noneMatch: true if NO elements match
boolean noneNegative = nums.stream()
    .noneMatch(n -> n < 0); // true

// Short-circuit behavior (efficient)
boolean hasMillion = IntStream.range(1, Integer.MAX_VALUE)
    .anyMatch(n -> n == 1_000_000); // Stops at 1,000,000, not MAX_VALUE

// Empty stream behavior:
boolean anyMatchEmpty = Stream.empty().anyMatch(x -> true);  // false
boolean allMatchEmpty = Stream.empty().allMatch(x -> false); // true (vacuously true)
boolean noneMatchEmpty = Stream.empty().noneMatch(x -> true); // true

// Practical
List<User> users = getUserList();
boolean allVerified = users.stream().allMatch(User::isEmailVerified);
boolean anyBlocked = users.stream().anyMatch(u -> u.getStatus() == BLOCKED);
```

</details>

---

### 🔴 Q12. What is `peek()` and when should you use it?

<details><summary>Click to reveal answer</summary>

`peek(Consumer<T>)` is an intermediate operation that performs an action on each element without modifying it. It returns the same stream for chaining.

```java
// ✅ Legitimate use: debugging stream pipeline
List<String> result = names.stream()
    .filter(n -> n.length() > 3)
    .peek(n -> System.out.println("After filter: " + n))
    .map(String::toUpperCase)
    .peek(n -> System.out.println("After map: " + n))
    .collect(Collectors.toList());

// ✅ Legitimate use: logging in stream pipeline
List<Order> processed = orders.stream()
    .filter(Order::isValid)
    .peek(o -> auditLog.record(o.getId(), "processed"))
    .map(orderService::process)
    .collect(Collectors.toList());
```

🚨 **Common Mistake:** Using `peek()` to modify state or as a side-effect-heavy consumer. The behavior of `peek()` is not guaranteed to execute in certain terminal operations:

```java
// ❌ WRONG: peek may not execute with count() in some JVM optimizations
long count = orders.stream()
    .peek(this::processOrder) // May be skipped!
    .count();

// ✅ CORRECT: use forEach or collect for guaranteed execution
orders.stream()
    .forEach(this::processOrder);
long count2 = orders.size();
```

> 💡 **Interviewer Tip:** `peek()` is a debugging tool. If you find yourself using it heavily for business logic, reconsider your stream design.

</details>

---

### 🟡 Q13. How do you create streams from different sources?

<details><summary>Click to reveal answer</summary>

```java
// From Collection
List<String> list = List.of("a", "b", "c");
Stream<String> s1 = list.stream();
Stream<String> s2 = list.parallelStream();

// From array
String[] arr = {"x", "y", "z"};
Stream<String> s3 = Arrays.stream(arr);
IntStream intStream = Arrays.stream(new int[]{1, 2, 3});

// From values
Stream<String> s4 = Stream.of("hello", "world");
Stream<Object> empty = Stream.empty();

// Infinite streams
Stream<Integer> natural = Stream.iterate(1, n -> n + 1); // 1, 2, 3, ...
Stream<Integer> naturals2 = Stream.iterate(1, n -> n <= 100, n -> n + 1); // Java 9+

Stream<Double> randoms = Stream.generate(Math::random); // Infinite randoms
// Always use with limit() to avoid infinite loops!

// IntStream, LongStream, DoubleStream
IntStream range = IntStream.range(1, 11);     // 1 to 10
IntStream rangeClosed = IntStream.rangeClosed(1, 10); // 1 to 10 inclusive
LongStream longs = LongStream.of(1L, 2L, 3L);

// From Map
Map<String, Integer> map = Map.of("a", 1, "b", 2);
Stream<Map.Entry<String, Integer>> entries = map.entrySet().stream();
Stream<String> keys = map.keySet().stream();
Stream<Integer> values = map.values().stream();

// From files
try (Stream<String> lines = Files.lines(Path.of("file.txt"))) {
    lines.filter(l -> !l.isBlank()).forEach(System.out::println);
}

// From String
"hello world".chars()
    .mapToObj(c -> String.valueOf((char) c))
    .forEach(System.out::print);
```

</details>

---

## Collectors

### 🟡 Q14. Explain `Collectors.groupingBy()` with examples.

<details><summary>Click to reveal answer</summary>

`groupingBy()` groups stream elements by a classifier function, returning `Map<K, List<V>>`:

```java
List<Employee> employees = getEmployees();

// Basic grouping
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// Downstream collector: count instead of list
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// Downstream: average salary per department
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// Downstream: collect names (not employees)
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));

// Multi-level grouping (department → seniority → list)
Map<String, Map<String, List<Employee>>> nested = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getSeniority)
    ));

// Group into sets (deduplicate)
Map<String, Set<String>> skillsByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.flatMapping(e -> e.getSkills().stream(), Collectors.toSet())
    ));
```

</details>

---

### 🟡 Q15. What collectors are available for string joining and statistics?

<details><summary>Click to reveal answer</summary>

```java
List<String> words = List.of("Java", "Python", "Go", "Rust");

// Collectors.joining()
String joined1 = words.stream().collect(Collectors.joining());
// "JavaPythonGoRust"

String joined2 = words.stream().collect(Collectors.joining(", "));
// "Java, Python, Go, Rust"

String joined3 = words.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[Java, Python, Go, Rust]"

// Statistical collectors
List<Integer> nums = List.of(10, 20, 30, 40, 50);

IntSummaryStatistics stats = nums.stream()
    .collect(Collectors.summarizingInt(Integer::intValue));

System.out.println(stats.getCount()); // 5
System.out.println(stats.getSum());   // 150
System.out.println(stats.getMin());   // 10
System.out.println(stats.getMax());   // 50
System.out.println(stats.getAverage()); // 30.0

// toMap
Map<String, Integer> wordLengths = words.stream()
    .collect(Collectors.toMap(
        Function.identity(),  // key: the word itself
        String::length        // value: its length
    ));
// {Java=4, Python=6, Go=2, Rust=4}

// toMap with merge function (handle key conflicts)
Map<Integer, String> lengthToWord = words.stream()
    .collect(Collectors.toMap(
        String::length,
        Function.identity(),
        (existing, newVal) -> existing + "/" + newVal // merge duplicates
    ));
// {4=Java/Rust, 6=Python, 2=Go}

// partitioningBy
Map<Boolean, List<Integer>> partitioned = nums.stream()
    .collect(Collectors.partitioningBy(n -> n > 25));
// {false=[10, 20], true=[30, 40, 50]}
```

</details>

---

### 🔴 Q16. How do you create a custom Collector?

<details><summary>Click to reveal answer</summary>

A `Collector<T, A, R>` has 5 components: supplier, accumulator, combiner, finisher, characteristics.

```java
// Custom collector: collect to ImmutableList
public class ImmutableListCollector<T>
        implements Collector<T, List<T>, List<T>> {

    @Override
    public Supplier<List<T>> supplier() {
        return ArrayList::new; // Mutable accumulator
    }

    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return List::add; // Add each element
    }

    @Override
    public BinaryOperator<List<T>> combiner() {
        return (l1, l2) -> { l1.addAll(l2); return l1; }; // Merge for parallel
    }

    @Override
    public Function<List<T>, List<T>> finisher() {
        return Collections::unmodifiableList; // Convert to immutable
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.emptySet(); // Not IDENTITY_FINISH, UNORDERED, or CONCURRENT
    }
}

// Usage
List<String> immutable = Stream.of("a", "b", "c")
    .collect(new ImmutableListCollector<>());

// Simpler with Collector.of()
Collector<String, StringBuilder, String> stringBuilderCollector =
    Collector.of(
        StringBuilder::new,
        StringBuilder::append,
        StringBuilder::append,
        StringBuilder::toString,
        Collector.Characteristics.IDENTITY_FINISH // Not applicable here, just example
    );
```

> 💡 **Interviewer Tip:** Custom collectors are rarely needed in practice — most use cases are covered by `Collectors.toMap`, `groupingBy`, `joining`, or `teeing` (Java 12+). This question tests depth of Stream API knowledge.

</details>

---

## Optional

### 🟢 Q17. What is `Optional<T>` and what problem does it solve?

<details><summary>Click to reveal answer</summary>

`Optional<T>` is a container that may or may not contain a non-null value. It makes null-handling explicit in the API, reducing NullPointerExceptions.

```java
// ❌ Without Optional: callers may forget to null-check
public User findUserById(Long id) {
    return userRepository.findById(id); // Could return null
}
User user = findUserById(1L);
String email = user.getEmail(); // NullPointerException if user is null!

// ✅ With Optional: null-safety is explicit in the signature
public Optional<User> findUserById(Long id) {
    return Optional.ofNullable(userRepository.findById(id));
}

// Creating Optional
Optional<String> present = Optional.of("value");      // NPE if null!
Optional<String> nullable = Optional.ofNullable(null); // Safe
Optional<String> empty = Optional.empty();

// Checking
present.isPresent(); // true
empty.isEmpty();     // true (Java 11+)

// Getting value
String value = present.get(); // ❌ Throws NoSuchElementException if empty!
String safe = present.orElse("default");
String computed = empty.orElseGet(() -> computeDefault());
String orThrow = present.orElseThrow(() -> new NotFoundException("Not found"));

// Transforming
Optional<String> upper = present.map(String::toUpperCase);
Optional<Integer> length = present.map(String::length);
Optional<User> user = findUserById(1L);
Optional<String> email = user.map(User::getEmail); // Optional<String>

// Chaining Optionals
Optional<String> result = findUserById(1L)
    .map(User::getProfile)           // Optional<Profile>
    .flatMap(Profile::getAvatar)     // flatMap prevents Optional<Optional<String>>
    .map(Avatar::getUrl);

// ifPresent
user.ifPresent(u -> System.out.println("Found: " + u.getName()));

// ifPresentOrElse (Java 9+)
user.ifPresentOrElse(
    u -> System.out.println("Found: " + u.getName()),
    () -> System.out.println("User not found")
);
```

</details>

---

### 🟡 Q18. What are common Optional anti-patterns?

<details><summary>Click to reveal answer</summary>

```java
// ❌ Anti-pattern 1: Calling get() without checking
Optional<User> user = findUser(id);
User u = user.get(); // NoSuchElementException if empty!

// ✅ Use orElse, orElseGet, or orElseThrow
User u2 = user.orElseThrow(() -> new UserNotFoundException(id));

// ❌ Anti-pattern 2: Using Optional as method parameter
public void sendEmail(Optional<String> email) { ... } // BAD!

// ✅ Use nullable parameter or overloads
public void sendEmail(String email) { ... }

// ❌ Anti-pattern 3: Using Optional in fields
class User {
    private Optional<String> middleName; // BAD! Optional is not Serializable
}

// ✅ Use nullable field, return Optional from getter
class User {
    private String middleName; // Nullable
    public Optional<String> getMiddleName() {
        return Optional.ofNullable(middleName);
    }
}

// ❌ Anti-pattern 4: isPresent() + get() — verbose
if (user.isPresent()) {
    process(user.get());
}

// ✅ Use ifPresent() or map()
user.ifPresent(this::process);

// ❌ Anti-pattern 5: Optional.of() with potentially null value
Optional<String> opt = Optional.of(possiblyNull); // NPE if null!

// ✅ Use ofNullable() for potentially null values
Optional<String> opt2 = Optional.ofNullable(possiblyNull);

// ❌ Anti-pattern 6: Returning null instead of Optional.empty()
public Optional<User> findUser(Long id) {
    User user = repo.findById(id);
    if (user == null) return null; // DEFEATS THE PURPOSE!
    return Optional.of(user);
}

// ✅ Always return Optional.empty() from Optional-returning methods
public Optional<User> findUser(Long id) {
    return Optional.ofNullable(repo.findById(id));
}
```

</details>

---

## Method References

### 🟢 Q19. What are the four types of method references in Java 8?

<details><summary>Click to reveal answer</summary>

| Type | Syntax | Lambda Equivalent |
|---|---|---|
| Static method | `ClassName::staticMethod` | `x -> ClassName.staticMethod(x)` |
| Instance method of a specific instance | `instance::instanceMethod` | `x -> instance.instanceMethod(x)` |
| Instance method of an arbitrary instance | `ClassName::instanceMethod` | `x -> x.instanceMethod()` |
| Constructor | `ClassName::new` | `x -> new ClassName(x)` |

```java
// 1. Static method reference
Function<String, Integer> parseInt = Integer::parseInt; // Integer.parseInt(s)
List<String> nums = List.of("1", "2", "3");
nums.stream().map(Integer::parseInt).forEach(System.out::println);

// 2. Instance method of a specific instance
String prefix = "Hello, ";
Function<String, String> greet = prefix::concat; // prefix.concat(s)
// greet.apply("World") -> "Hello, World"

PrintStream printer = System.out;
Consumer<Object> print = printer::println; // System.out.println(x)

// 3. Instance method of an arbitrary instance (type's instance)
Function<String, String> upper = String::toUpperCase; // s -> s.toUpperCase()
Function<String, Integer> length = String::length;    // s -> s.length()
Predicate<String> isEmpty = String::isEmpty;           // s -> s.isEmpty()

List<String> words = List.of("Hello", "World");
words.stream().map(String::toLowerCase).forEach(System.out::println);

// 4. Constructor reference
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<String, StringBuilder> sbFactory = StringBuilder::new;
BiFunction<String, Integer, char[]> charArray = char[]::new; // length → new char[n]

// Practical examples
List<String> names = List.of("Alice", "Bob", "Charlie");
List<User> users = names.stream()
    .map(User::new)          // User(String name) constructor
    .collect(Collectors.toList());

// Comparator with method reference
List<Employee> emps = getEmployees();
emps.sort(Comparator.comparing(Employee::getSalary).reversed());
```

</details>

---

## Default & Static Interface Methods

### 🟢 Q20. What are default and static methods in interfaces (Java 8)?

<details><summary>Click to reveal answer</summary>

**Default methods** provide implementation in interfaces, enabling backward-compatible API evolution. **Static methods** are utility methods belonging to the interface.

```java
public interface Shape {

    // Abstract method — must be implemented
    double area();
    double perimeter();

    // Default method — provided implementation
    default String describe() {
        return String.format("Shape with area=%.2f, perimeter=%.2f",
            area(), perimeter());
    }

    // Default method for chaining/composition
    default Shape scale(double factor) {
        // Creates a new scaled shape using anonymous class
        double scaledArea = this.area() * factor * factor;
        return () -> scaledArea; // Won't work here but shows concept
    }

    // Static utility method
    static double areaOfCircle(double radius) {
        return Math.PI * radius * radius;
    }

    // Static factory method
    static Shape circle(double radius) {
        return new Circle(radius);
    }
}

// Class inheriting default method
class Rectangle implements Shape {
    private double width, height;

    @Override
    public double area() { return width * height; }

    @Override
    public double perimeter() { return 2 * (width + height); }

    // describe() inherited from Shape — no override needed
}

// Usage
Shape rect = new Rectangle(4, 5);
System.out.println(rect.describe()); // Uses default method

double circleArea = Shape.areaOfCircle(5.0); // Static method call
```

**Diamond problem resolution:**
```java
interface A { default void hello() { System.out.println("A"); } }
interface B extends A { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void hello() {
        B.super.hello(); // Explicit resolution required
    }
}
```

</details>

---

## Parallel Streams

### 🟡 Q21. What are parallel streams and what are their pitfalls?

<details><summary>Click to reveal answer</summary>

Parallel streams split work across multiple threads using the ForkJoinPool. Enable with `.parallelStream()` or `.parallel()`.

```java
// Simple parallel stream
List<Integer> numbers = IntStream.range(1, 1_000_001)
    .boxed()
    .collect(Collectors.toList());

// Sequential
long start = System.currentTimeMillis();
long sum = numbers.stream().mapToLong(Integer::longValue).sum();
System.out.println("Sequential: " + (System.currentTimeMillis() - start) + "ms");

// Parallel
start = System.currentTimeMillis();
long parallelSum = numbers.parallelStream().mapToLong(Integer::longValue).sum();
System.out.println("Parallel: " + (System.currentTimeMillis() - start) + "ms");
```

**Pitfalls:**

```java
// ❌ Pitfall 1: Shared mutable state — race condition!
List<Integer> results = new ArrayList<>();
numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .forEach(results::add); // ArrayList is NOT thread-safe — data corruption!

// ✅ Safe alternative
List<Integer> safeResults = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList()); // Thread-safe collection

// ❌ Pitfall 2: Ordered operations are slow in parallel
List<String> ordered = names.parallelStream()
    .sorted()         // Requires coordination — may be SLOWER than sequential!
    .collect(Collectors.toList());

// ❌ Pitfall 3: Using System.out.println (synchronized = bottleneck)
numbers.parallelStream().forEach(System.out::println); // Serialized by println

// ❌ Pitfall 4: Small datasets — overhead exceeds benefit
List<Integer> small = List.of(1, 2, 3, 4, 5);
small.parallelStream().map(n -> n * 2).collect(Collectors.toList());
// Fork/join overhead > computation time

// ❌ Pitfall 5: I/O-bound operations
urls.parallelStream()
    .map(url -> httpClient.get(url)) // Threads block on I/O — use CompletableFuture
    .collect(Collectors.toList());

// ✅ When parallel streams help:
// - CPU-bound operations on large datasets (100k+ elements)
// - No shared mutable state
// - No ordered dependencies between elements
long count = hugeList.parallelStream()
    .filter(this::expensivePredicate) // CPU-intensive
    .count();
```

> 💡 **Interviewer Tip:** Parallel streams use the common ForkJoinPool (default: CPU cores - 1 threads). Using them heavily in a web app can starve the thread pool and impact all requests.

</details>

---

### 🔴 Q22. How do you control the thread pool used by parallel streams?

<details><summary>Click to reveal answer</summary>

By default, parallel streams use `ForkJoinPool.commonPool()`. To use a custom pool:

```java
// Default: common pool (shared application-wide)
numbers.parallelStream().map(this::process).collect(Collectors.toList());

// Custom ForkJoinPool — isolates parallel stream from common pool
ForkJoinPool customPool = new ForkJoinPool(4); // 4 threads

try {
    List<Integer> result = customPool.submit(() ->
        numbers.parallelStream()
               .map(this::expensiveOperation)
               .collect(Collectors.toList())
    ).get();
} catch (InterruptedException | ExecutionException e) {
    Thread.currentThread().interrupt();
    throw new RuntimeException(e);
} finally {
    customPool.shutdown();
}

// Java 21+ Virtual Threads alternative (better for I/O)
List<String> results;
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    results = urls.stream()
        .map(url -> CompletableFuture.supplyAsync(() -> fetch(url), executor))
        .toList()
        .stream()
        .map(CompletableFuture::join)
        .collect(Collectors.toList());
}
```

</details>

---

### 🟡 Q23. How do Streams differ from Collections?

<details><summary>Click to reveal answer</summary>

| Feature | Collection | Stream |
|---|---|---|
| Storage | Stores elements in memory | Doesn't store elements |
| Reusability | Can iterate multiple times | Single-use — terminal op consumes it |
| Eagerness | Eagerly computed | Lazily evaluated |
| Modification | Supports add/remove | Cannot modify source |
| Size | Usually finite | Can be infinite |
| Focus | Data storage | Data processing |

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));

// Collection: multiple iterations
for (int i : list) { } // Iteration 1
for (int i : list) { } // Iteration 2 — OK!

// Stream: single-use only
Stream<Integer> stream = list.stream();
stream.forEach(System.out::println); // Works
stream.forEach(System.out::println); // ❌ IllegalStateException: stream already consumed!

// Stream: lazy evaluation
Stream<Integer> lazy = Stream.iterate(1, n -> n + 1) // Infinite!
    .filter(n -> n % 2 == 0)
    .limit(5); // Only 5 elements computed
List<Integer> evens = lazy.collect(Collectors.toList()); // [2, 4, 6, 8, 10]
```

</details>

---

### 🟡 Q24. What is `Collectors.teeing()` (Java 12)?

<details><summary>Click to reveal answer</summary>

`Collectors.teeing()` collects a stream with two collectors simultaneously and merges the results:

```java
// teeing(downstream1, downstream2, merger)
record MinMax(int min, int max) {}

List<Integer> numbers = List.of(5, 3, 8, 1, 9, 2, 7);

MinMax result = numbers.stream().collect(
    Collectors.teeing(
        Collectors.minBy(Integer::compareTo),    // Collector 1: find min
        Collectors.maxBy(Integer::compareTo),    // Collector 2: find max
        (min, max) -> new MinMax(                // Merge results
            min.orElseThrow(),
            max.orElseThrow()
        )
    )
);
System.out.println(result); // MinMax[min=1, max=9]

// Another example: sum and count simultaneously (for average)
record SumCount(long sum, long count) {
    double average() { return (double) sum / count; }
}

SumCount sc = numbers.stream().collect(
    Collectors.teeing(
        Collectors.summingLong(Integer::longValue),
        Collectors.counting(),
        SumCount::new
    )
);
System.out.println(sc.average()); // 5.0
```

</details>

---

### 🔴 Q25. How would you find the top 3 most frequent words in a string using streams?

<details><summary>Click to reveal answer</summary>

```java
public List<String> top3FrequentWords(String text) {
    return Arrays.stream(text.toLowerCase().split("\\W+"))
        .filter(w -> !w.isBlank())
        .collect(Collectors.groupingBy(
            Function.identity(),
            Collectors.counting()
        ))
        .entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
        .limit(3)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}

// Test
String text = "the quick brown fox jumps over the lazy dog the fox";
// Result: [the, fox, (any of the single-occurrence words)]
System.out.println(top3FrequentWords(text)); // [the, fox, quick]
```

**Step by step:**
1. Split text on non-word characters
2. Filter blanks
3. `groupingBy` + `counting` → `Map<String, Long>` (word → frequency)
4. Stream the entry set
5. Sort by value descending
6. Take top 3
7. Extract keys

</details>

---

### 🟡 Q26. Solve: given a list of transactions, find the sum of amounts for each currency.

<details><summary>Click to reveal answer</summary>

```java
record Transaction(String currency, double amount, String type) {}

List<Transaction> transactions = List.of(
    new Transaction("USD", 100.0, "CREDIT"),
    new Transaction("EUR", 200.0, "DEBIT"),
    new Transaction("USD", 50.0, "DEBIT"),
    new Transaction("GBP", 300.0, "CREDIT"),
    new Transaction("EUR", 150.0, "CREDIT")
);

// Sum by currency
Map<String, Double> sumByCurrency = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::currency,
        Collectors.summingDouble(Transaction::amount)
    ));
// {USD=150.0, EUR=350.0, GBP=300.0}

// Sum only CREDIT transactions by currency
Map<String, Double> creditSumByCurrency = transactions.stream()
    .filter(t -> "CREDIT".equals(t.type()))
    .collect(Collectors.groupingBy(
        Transaction::currency,
        Collectors.summingDouble(Transaction::amount)
    ));
// {USD=100.0, GBP=300.0, EUR=150.0}

// Get currency with highest total
Optional<Map.Entry<String, Double>> topCurrency = sumByCurrency.entrySet()
    .stream()
    .max(Map.Entry.comparingByValue());
// USD -> no, EUR=350.0 is max
topCurrency.ifPresent(e ->
    System.out.printf("Top: %s = %.2f%n", e.getKey(), e.getValue()));
// Top: EUR = 350.00
```

</details>

---

### 🟡 Q27. What are `mapToInt`, `mapToLong`, `mapToDouble` and why use them?

<details><summary>Click to reveal answer</summary>

These convert a `Stream<T>` to a primitive stream, avoiding boxing overhead and enabling primitive-specific operations (`sum`, `average`, `range`, `summaryStatistics`):

```java
List<Employee> employees = getEmployees();

// Stream<Employee> → IntStream (no boxing)
IntStream salaryStream = employees.stream()
    .mapToInt(Employee::getSalary);

int total = salaryStream.sum();       // Primitive sum
OptionalDouble avg = employees.stream()
    .mapToInt(Employee::getSalary)
    .average();

IntSummaryStatistics stats = employees.stream()
    .mapToInt(Employee::getSalary)
    .summaryStatistics();

// vs boxed sum (more verbose, boxing overhead)
Optional<Integer> sum2 = employees.stream()
    .map(Employee::getSalary)
    .reduce(Integer::sum); // Boxing overhead

// mapToObj: convert primitive stream back to object stream
IntStream.range(1, 6)
    .mapToObj(i -> "Item-" + i)
    .forEach(System.out::println);
// Item-1, Item-2, ..., Item-5

// Chaining primitive and object streams
double avgNameLength = employees.stream()
    .mapToInt(e -> e.getName().length())
    .average()
    .orElse(0.0);
```

</details>

---

### 🟡 Q28. What is `Stream.iterate()` vs `Stream.generate()`?

<details><summary>Click to reveal answer</summary>

```java
// Stream.iterate(seed, f): each element = f(previous)
// Always produces ordered, deterministic sequence
Stream<Integer> naturals = Stream.iterate(1, n -> n + 1);
naturals.limit(5).forEach(System.out::println); // 1, 2, 3, 4, 5

// Fibonacci sequence
Stream.iterate(new int[]{0, 1}, arr -> new int[]{arr[1], arr[0] + arr[1]})
    .limit(10)
    .map(arr -> arr[0])
    .forEach(System.out::println); // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34

// Java 9+: iterate with predicate (like a for loop)
Stream.iterate(1, n -> n <= 10, n -> n + 1)
    .forEach(System.out::println); // 1 to 10, then stops

// Stream.generate(Supplier): no seed, each element from supplier
// Not guaranteed to be ordered or related to previous
Stream<Double> randoms = Stream.generate(Math::random);
randoms.limit(5).forEach(System.out::println); // 5 random doubles

Stream<String> uuids = Stream.generate(() -> UUID.randomUUID().toString());
String firstUUID = uuids.findFirst().orElseThrow();

// Practical: generate test data
List<User> testUsers = Stream.generate(() ->
    new User("User-" + ThreadLocalRandom.current().nextInt(1000),
             "test@example.com"))
    .limit(100)
    .collect(Collectors.toList());
```

</details>

---

### 🔴 Q29. What is the difference between `findFirst()` and `findAny()`?

<details><summary>Click to reveal answer</summary>

- `findFirst()` — Returns the **first** element in encounter order. Consistent and deterministic.
- `findAny()` — Returns **any** element. In sequential streams behaves like `findFirst()`. In parallel streams, may return any element (faster).

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");

// Sequential: findFirst == findAny (usually)
Optional<String> first = names.stream()
    .filter(n -> n.startsWith("C"))
    .findFirst(); // Optional[Charlie]

Optional<String> any = names.stream()
    .filter(n -> n.startsWith("C"))
    .findAny(); // Optional[Charlie] — same in sequential

// Parallel: findAny may return non-first match (faster)
Optional<String> parallelAny = names.parallelStream()
    .filter(n -> n.length() > 3)
    .findAny(); // Could be Alice, Charlie, or Dave — non-deterministic

// ✅ Use findFirst() when order matters
Optional<String> firstLong = names.stream()
    .filter(n -> n.length() > 3)
    .findFirst(); // Always "Alice" (first in list matching filter)

// ✅ Use findAny() when any result is acceptable (better parallel performance)
boolean exists = largeList.parallelStream()
    .filter(this::matchesCriteria)
    .findAny()
    .isPresent();
```

</details>

---

### 🟡 Q30. Implement a stream pipeline to flatten, deduplicate, and sort employee skills.

<details><summary>Click to reveal answer</summary>

```java
record Employee(String name, List<String> skills) {}

List<Employee> employees = List.of(
    new Employee("Alice", List.of("Java", "Spring", "SQL")),
    new Employee("Bob",   List.of("Java", "Python", "Docker")),
    new Employee("Carol", List.of("Spring", "Kubernetes", "SQL"))
);

// All unique skills, sorted alphabetically
List<String> allSkills = employees.stream()
    .flatMap(e -> e.skills().stream())   // Flatten nested lists
    .distinct()                           // Remove duplicates
    .sorted()                             // Alphabetical order
    .collect(Collectors.toList());
// [Docker, Java, Kubernetes, Python, SQL, Spring]

// Skills held by more than one employee
Map<String, Long> skillFrequency = employees.stream()
    .flatMap(e -> e.skills().stream())
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

List<String> popularSkills = skillFrequency.entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .sorted()
    .collect(Collectors.toList());
// [Java, SQL, Spring]

// Employee(s) with the most skills
int maxSkills = employees.stream()
    .mapToInt(e -> e.skills().size())
    .max()
    .orElse(0);

List<String> mostSkilled = employees.stream()
    .filter(e -> e.skills().size() == maxSkills)
    .map(Employee::name)
    .collect(Collectors.toList());
// [Alice, Bob] — both have 3 skills
```

</details>

---

*End of Java 8 & Functional Programming*
