# Java Testing (JUnit 5 & Mockito)

> 💡 **Interviewer Tip:** Testing questions reveal how a candidate thinks about correctness, maintainability, and collaboration. Be ready to discuss *why* you write tests, not just *how*.

---

## Table of Contents
1. [JUnit 5 Fundamentals](#junit-5-fundamentals)
2. [JUnit 5 Annotations](#junit-5-annotations)
3. [Assertions](#assertions)
4. [Mockito Basics](#mockito-basics)
5. [Advanced Mockito](#advanced-mockito)
6. [TDD Basics](#tdd-basics)
7. [Testing Best Practices](#testing-best-practices)

---

## JUnit 5 Fundamentals

### 🟢 Q1. What is JUnit 5, and how does it differ from JUnit 4?

<details><summary>Click to reveal answer</summary>

JUnit 5 is the fifth major version of the JUnit framework. It is composed of three modules:

- **JUnit Platform** – Foundation for launching testing frameworks on the JVM
- **JUnit Jupiter** – New programming and extension model for writing tests
- **JUnit Vintage** – Support for running JUnit 3/4 tests

Key differences from JUnit 4:

| Feature | JUnit 4 | JUnit 5 |
|---|---|---|
| Import | `org.junit.*` | `org.junit.jupiter.api.*` |
| `@Before` | `@Before` | `@BeforeEach` |
| `@After` | `@After` | `@AfterEach` |
| `@BeforeClass` | `@BeforeClass` | `@BeforeAll` |
| `@AfterClass` | `@AfterClass` | `@AfterAll` |
| Extension | `@RunWith` | `@ExtendWith` |
| Nested tests | Not supported natively | `@Nested` |
| Parameterized | Separate runner | Built-in `@ParameterizedTest` |

```java
// JUnit 5 minimal test class
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    @Test
    void addsTwoNumbers() {
        Calculator calc = new Calculator();
        assertEquals(5, calc.add(2, 3));
    }
}
```

</details>

---

### 🟢 Q2. What is the JUnit 5 test lifecycle?

<details><summary>Click to reveal answer</summary>

By default, JUnit 5 creates a **new instance** of the test class for each test method (unlike JUnit 4 which also did this, but some frameworks differ). The lifecycle order is:

1. `@BeforeAll` – Runs once before all tests in the class (must be `static`)
2. `@BeforeEach` – Runs before each test method
3. `@Test` – The actual test
4. `@AfterEach` – Runs after each test method
5. `@AfterAll` – Runs once after all tests (must be `static`)

```java
import org.junit.jupiter.api.*;

class LifecycleTest {

    @BeforeAll
    static void initAll() {
        System.out.println("Before ALL tests");
    }

    @BeforeEach
    void init() {
        System.out.println("Before EACH test");
    }

    @Test
    void testOne() {
        System.out.println("Test ONE");
    }

    @Test
    void testTwo() {
        System.out.println("Test TWO");
    }

    @AfterEach
    void tearDown() {
        System.out.println("After EACH test");
    }

    @AfterAll
    static void tearDownAll() {
        System.out.println("After ALL tests");
    }
}
// Output order: initAll → init → testOne → tearDown → init → testTwo → tearDown → tearDownAll
```

> 💡 **Interviewer Tip:** Ask candidates what happens if `@BeforeAll` is NOT `static` — it causes an error unless `@TestInstance(Lifecycle.PER_CLASS)` is used.

</details>

---

## JUnit 5 Annotations

### 🟢 Q3. What is `@TestInstance` and when would you use it?

<details><summary>Click to reveal answer</summary>

`@TestInstance` controls how JUnit creates test class instances.

- `@TestInstance(Lifecycle.PER_METHOD)` – Default. New instance per test.
- `@TestInstance(Lifecycle.PER_CLASS)` – Single instance for all tests.

Use `PER_CLASS` when:
- You want to share expensive state between tests
- You want `@BeforeAll` / `@AfterAll` to be non-static

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DatabaseIntegrationTest {

    private Connection connection;

    @BeforeAll
    void openConnection() {  // No 'static' required!
        connection = DataSource.openConnection();
    }

    @Test
    void queryUsers() {
        assertNotNull(connection.query("SELECT * FROM users"));
    }

    @AfterAll
    void closeConnection() {
        connection.close();
    }
}
```

🚨 **Common Mistake:** With `PER_CLASS`, shared mutable state between tests can cause test ordering issues. Tests should remain independent.

</details>

---

### 🟡 Q4. How does `@Nested` work and why is it useful?

<details><summary>Click to reveal answer</summary>

`@Nested` allows you to group related tests inside inner classes, creating a hierarchical test structure. This improves readability and enables setup per group.

```java
class OrderServiceTest {

    OrderService orderService = new OrderService();

    @Nested
    class WhenOrderIsEmpty {

        @Test
        void shouldReturnZeroTotal() {
            Order order = new Order();
            assertEquals(0.0, orderService.calculateTotal(order));
        }

        @Test
        void shouldNotAllowCheckout() {
            Order order = new Order();
            assertThrows(EmptyOrderException.class,
                () -> orderService.checkout(order));
        }
    }

    @Nested
    class WhenOrderHasItems {

        Order order;

        @BeforeEach
        void setUp() {
            order = new Order();
            order.addItem(new Item("Book", 29.99));
        }

        @Test
        void shouldCalculateTotalCorrectly() {
            assertEquals(29.99, orderService.calculateTotal(order), 0.001);
        }

        @Test
        void shouldAllowCheckout() {
            assertDoesNotThrow(() -> orderService.checkout(order));
        }
    }
}
```

✅ **Best Practice:** Use `@Nested` to document behavior in a BDD-style (Given/When/Then) structure.

</details>

---

### 🟡 Q5. Explain `@ParameterizedTest` and the available source annotations.

<details><summary>Click to reveal answer</summary>

`@ParameterizedTest` runs a single test with multiple inputs. Sources include:

- `@ValueSource` – Array of primitives or strings
- `@CsvSource` – Inline CSV rows
- `@CsvFileSource` – External CSV file
- `@MethodSource` – Factory method returning a `Stream`
- `@EnumSource` – All or selected enum constants
- `@NullAndEmptySource` – Null and empty string

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import java.util.stream.Stream;

class ValidatorTest {

    @ParameterizedTest
    @ValueSource(strings = {"racecar", "level", "madam"})
    void shouldIdentifyPalindromes(String word) {
        assertTrue(StringUtils.isPalindrome(word));
    }

    @ParameterizedTest
    @CsvSource({
        "Alice, 30, true",
        "Bob,   17, false",
        "Charlie, 18, true"
    })
    void shouldCheckAdultEligibility(String name, int age, boolean expected) {
        assertEquals(expected, Eligibility.isAdult(age));
    }

    @ParameterizedTest
    @MethodSource("provideInvalidEmails")
    void shouldRejectInvalidEmails(String email) {
        assertFalse(EmailValidator.isValid(email));
    }

    static Stream<String> provideInvalidEmails() {
        return Stream.of("notanemail", "@missing.com", "no-at-sign");
    }

    @ParameterizedTest
    @NullAndEmptySource
    void shouldHandleNullAndEmpty(String input) {
        assertTrue(StringUtils.isBlank(input));
    }
}
```

</details>

---

### 🟢 Q6. What does `@ExtendWith` do in JUnit 5?

<details><summary>Click to reveal answer</summary>

`@ExtendWith` registers extensions that hook into the JUnit 5 lifecycle. The most common usage is integrating with Mockito:

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    @Test
    void shouldFindUserById() {
        when(userRepository.findById(1L))
            .thenReturn(Optional.of(new User(1L, "Alice")));

        User result = userService.findById(1L);

        assertEquals("Alice", result.getName());
    }
}
```

Other common extensions:
- `SpringExtension.class` – Spring context integration
- `MockitoExtension.class` – Mockito lifecycle management
- Custom extensions implementing `BeforeEachCallback`, `AfterEachCallback`, etc.

</details>

---

### 🟢 Q7. What are `@Disabled` and `@Tag` used for?

<details><summary>Click to reveal answer</summary>

- `@Disabled` – Skips a test or entire class (with optional reason)
- `@Tag` – Categorizes tests for filtered test runs

```java
@Tag("integration")
class PaymentIntegrationTest {

    @Test
    @Tag("slow")
    void processPaymentEndToEnd() {
        // ...
    }

    @Test
    @Disabled("Payment gateway sandbox is down — re-enable after INFRA-456")
    void processRefund() {
        // ...
    }
}
```

Run only tagged tests with Maven:
```bash
mvn test -Dgroups="integration"
mvn test -DexcludedGroups="slow"
```

✅ **Best Practice:** Use tags to separate unit tests (`@Tag("unit")`) from integration tests (`@Tag("integration")`) and run them separately in CI/CD pipelines.

</details>

---

## Assertions

### 🟢 Q8. What is `assertAll()` and why is it better than multiple `assertEquals()` calls?

<details><summary>Click to reveal answer</summary>

`assertAll()` executes all assertions and reports **all** failures at once. With sequential `assertEquals()`, the test stops at the first failure.

```java
@Test
void shouldMapUserCorrectly() {
    User user = userMapper.fromDto(new UserDto("Alice", "alice@example.com", 30));

    // ❌ BAD: Stops at first failure — you won't know what else failed
    assertEquals("Alice", user.getName());
    assertEquals("alice@example.com", user.getEmail());
    assertEquals(30, user.getAge());

    // ✅ GOOD: Reports ALL failures
    assertAll("user fields",
        () -> assertEquals("Alice", user.getName()),
        () -> assertEquals("alice@example.com", user.getEmail()),
        () -> assertEquals(30, user.getAge())
    );
}
```

🚨 **Common Mistake:** Using `assertAll` with side-effecting lambdas — each lambda should be an independent assertion.

</details>

---

### 🟢 Q9. How does `assertThrows()` work?

<details><summary>Click to reveal answer</summary>

`assertThrows()` verifies that a specific exception is thrown by the tested code, and returns the exception for further inspection.

```java
@Test
void shouldThrowWhenDividingByZero() {
    Calculator calc = new Calculator();

    ArithmeticException ex = assertThrows(
        ArithmeticException.class,
        () -> calc.divide(10, 0)
    );

    assertEquals("/ by zero", ex.getMessage());
}

@Test
void shouldThrowIllegalArgumentForNegativeAge() {
    UserService service = new UserService();

    assertThrows(
        IllegalArgumentException.class,
        () -> service.createUser("Alice", -1)
    );
}
```

Related: `assertDoesNotThrow()` verifies no exception is thrown:

```java
@Test
void shouldNotThrowForValidInput() {
    assertDoesNotThrow(() -> service.createUser("Alice", 25));
}
```

</details>

---

### 🟡 Q10. How do you test asynchronous code with JUnit 5?

<details><summary>Click to reveal answer</summary>

Use `assertTimeout()` or `assertTimeoutPreemptively()` for time-bound assertions:

```java
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

@Test
void shouldCompleteWithinTimeout() {
    assertTimeout(Duration.ofSeconds(2), () -> {
        Thread.sleep(100); // simulating async work
        return "done";
    });
}

// assertTimeoutPreemptively aborts if timeout is exceeded
@Test
void shouldNotBlockTooLong() {
    assertTimeoutPreemptively(Duration.ofMillis(500), () -> {
        fastService.process();
    });
}

// For CompletableFuture:
@Test
void shouldResolveCompletableFuture() throws Exception {
    CompletableFuture<String> future = asyncService.fetchData();
    String result = future.get(2, TimeUnit.SECONDS);
    assertEquals("expected", result);
}
```

> 💡 **Interviewer Tip:** `assertTimeout` runs in the same thread. `assertTimeoutPreemptively` uses a separate thread, which means ThreadLocal variables won't be inherited.

</details>

---

## Mockito Basics

### 🟢 Q11. What is Mockito and what problem does it solve?

<details><summary>Click to reveal answer</summary>

Mockito is a Java mocking framework that allows you to create test doubles (mocks) for dependencies. This lets you test a class in isolation without instantiating its real collaborators.

**Without Mockito:** Tests become integration tests if they rely on real database, network, or file system access.

**With Mockito:** You control the behavior of dependencies.

```java
// Class under test
class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }

    public Order placeOrder(OrderRequest request) {
        Order order = repository.save(new Order(request));
        emailService.sendConfirmation(order);
        return order;
    }
}

// Test with Mockito
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository repository;
    @Mock EmailService emailService;
    @InjectMocks OrderService orderService;

    @Test
    void shouldPlaceOrderAndSendConfirmation() {
        OrderRequest request = new OrderRequest("Book", 1);
        Order savedOrder = new Order(1L, "Book", 1);

        when(repository.save(any(Order.class))).thenReturn(savedOrder);

        Order result = orderService.placeOrder(request);

        assertEquals(savedOrder.getId(), result.getId());
        verify(emailService).sendConfirmation(savedOrder);
    }
}
```

</details>

---

### 🟢 Q12. What is the difference between `@Mock`, `@Spy`, and `@InjectMocks`?

<details><summary>Click to reveal answer</summary>

| Annotation | Behavior |
|---|---|
| `@Mock` | Creates a fully mocked object. All methods return default values (null, 0, false) unless stubbed. |
| `@Spy` | Wraps a real object. Unstubbed methods call the real implementation. |
| `@InjectMocks` | Creates an instance of the class under test and injects `@Mock`/`@Spy` fields. |

```java
@ExtendWith(MockitoExtension.class)
class SpyVsMockTest {

    @Mock
    List<String> mockedList;   // Completely fake

    @Spy
    List<String> spiedList = new ArrayList<>();  // Real ArrayList

    @Test
    void mockBehavior() {
        mockedList.add("one");
        // Real add() never called — size() returns 0 (default)
        assertEquals(0, mockedList.size());  // passes!
    }

    @Test
    void spyBehavior() {
        spiedList.add("one");
        // Real add() IS called
        assertEquals(1, spiedList.size());  // passes!
    }

    @Test
    void spyWithStubbing() {
        // Override specific method on spy
        doReturn(100).when(spiedList).size();
        spiedList.add("one");
        assertEquals(100, spiedList.size());
    }
}
```

🚨 **Common Mistake:** Using `when(spy.method()).thenReturn(x)` on a spy calls the real method before stubbing. Use `doReturn(x).when(spy).method()` instead.

</details>

---

### 🟡 Q13. What is `@Captor` and `ArgumentCaptor`?

<details><summary>Click to reveal answer</summary>

`ArgumentCaptor` captures arguments passed to a mock method, allowing assertions on them.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock UserRepository repository;
    @Captor ArgumentCaptor<User> userCaptor;
    @InjectMocks UserService userService;

    @Test
    void shouldSaveUserWithCorrectData() {
        userService.register("Alice", "alice@example.com");

        verify(repository).save(userCaptor.capture());

        User captured = userCaptor.getValue();
        assertEquals("Alice", captured.getName());
        assertEquals("alice@example.com", captured.getEmail());
        assertNotNull(captured.getCreatedAt());
    }

    @Test
    void shouldCaptureMutipleInvocations() {
        userService.register("Alice", "a@example.com");
        userService.register("Bob", "b@example.com");

        verify(repository, times(2)).save(userCaptor.capture());

        List<User> allCaptured = userCaptor.getAllValues();
        assertEquals("Alice", allCaptured.get(0).getName());
        assertEquals("Bob", allCaptured.get(1).getName());
    }
}
```

✅ **Best Practice:** Use `ArgumentCaptor` when you need to verify the *content* of arguments, not just that the method was called.

</details>

---

### 🟢 Q14. How do you stub methods with `when().thenReturn()` and `doThrow()`?

<details><summary>Click to reveal answer</summary>

```java
@ExtendWith(MockitoExtension.class)
class StubExamplesTest {

    @Mock PaymentGateway gateway;

    @Test
    void shouldReturnSuccessOnPayment() {
        when(gateway.charge("card-123", 99.99))
            .thenReturn(PaymentResult.SUCCESS);

        PaymentResult result = gateway.charge("card-123", 99.99);
        assertEquals(PaymentResult.SUCCESS, result);
    }

    @Test
    void shouldHandlePaymentFailure() {
        when(gateway.charge(anyString(), anyDouble()))
            .thenThrow(new PaymentException("Card declined"));

        assertThrows(PaymentException.class,
            () -> gateway.charge("card-999", 500.0));
    }

    @Test
    void shouldReturnDifferentValuesOnConsecutiveCalls() {
        when(gateway.getStatus())
            .thenReturn("PENDING")
            .thenReturn("PROCESSING")
            .thenReturn("COMPLETE");

        assertEquals("PENDING",    gateway.getStatus());
        assertEquals("PROCESSING", gateway.getStatus());
        assertEquals("COMPLETE",   gateway.getStatus());
    }

    // For void methods, use doThrow:
    @Test
    void shouldThrowOnVoidMethod() {
        doThrow(new RuntimeException("Send failed"))
            .when(gateway).sendReceipt(anyString());

        assertThrows(RuntimeException.class,
            () -> gateway.sendReceipt("alice@example.com"));
    }
}
```

</details>

---

## Advanced Mockito

### 🟡 Q15. How do you use `verify()` to assert interactions?

<details><summary>Click to reveal answer</summary>

`verify()` asserts that a mock method was (or wasn't) called with specific arguments.

```java
@Test
void verificationExamples() {
    orderService.placeOrder(request);

    // Called exactly once (default)
    verify(repository).save(any(Order.class));

    // Called exactly N times
    verify(emailService, times(1)).sendConfirmation(any());

    // Never called
    verify(auditLogger, never()).log(anyString());

    // Called at least N times
    verify(cache, atLeast(1)).evict(anyString());

    // Called at most N times
    verify(rateLimiter, atMost(3)).check();

    // No more interactions with the mock
    verifyNoMoreInteractions(repository);

    // No interactions at all
    verifyNoInteractions(fraudDetector);
}
```

> 💡 **Interviewer Tip:** `verifyNoMoreInteractions()` can make tests brittle — it fails when new interactions are added. Use it sparingly and only when interaction completeness is critical.

</details>

---

### 🟡 Q16. What are Mockito argument matchers?

<details><summary>Click to reveal answer</summary>

Argument matchers allow flexible matching instead of exact values:

```java
import static org.mockito.ArgumentMatchers.*;

// Built-in matchers
when(repo.findById(anyLong())).thenReturn(Optional.of(user));
when(service.process(anyString(), any(Order.class))).thenReturn(true);
when(cache.get(eq("specific-key"))).thenReturn(value);
when(validator.validate(argThat(u -> u.getAge() >= 18))).thenReturn(true);

// Custom matcher with lambda
verify(emailService).send(argThat(email ->
    email.getTo().equals("alice@example.com") &&
    email.getSubject().contains("Order Confirmation")
));
```

🚨 **Common Mistake:** Mixing matchers and exact values. If you use ANY matcher, ALL arguments must use matchers:

```java
// ❌ WRONG - mixing matcher with exact value
when(service.find(anyLong(), "exact")).thenReturn(result);

// ✅ CORRECT - use eq() for exact matching alongside matchers
when(service.find(anyLong(), eq("exact"))).thenReturn(result);
```

</details>

---

### 🔴 Q17. How do you mock static methods and constructors with Mockito?

<details><summary>Click to reveal answer</summary>

Since Mockito 3.4+, static methods can be mocked with `mockStatic()`:

```java
// Requires mockito-inline dependency
import org.mockito.MockedStatic;

@Test
void shouldMockStaticMethod() {
    try (MockedStatic<UUID> mockedUUID = mockStatic(UUID.class)) {
        UUID fixed = UUID.fromString("123e4567-e89b-12d3-a456-426614174000");
        mockedUUID.when(UUID::randomUUID).thenReturn(fixed);

        String id = userService.generateId();
        assertEquals("123e4567-e89b-12d3-a456-426614174000", id);
    }
    // Static mock is restored after try-with-resources
}

// Mocking constructors with MockedConstruction
@Test
void shouldMockConstructor() {
    try (MockedConstruction<PaymentGateway> mocked =
            mockConstruction(PaymentGateway.class,
                (mock, context) -> when(mock.charge(anyDouble()))
                    .thenReturn(PaymentResult.SUCCESS))) {

        PaymentService service = new PaymentService(); // Uses mocked gateway
        assertTrue(service.pay(100.0));
    }
}
```

✅ **Best Practice:** Prefer dependency injection over static methods to avoid needing `mockStatic`. Static mocking is a code smell workaround.

</details>

---

### 🟡 Q18. What is `doAnswer()` and when would you use it?

<details><summary>Click to reveal answer</summary>

`doAnswer()` provides custom logic when a mocked method is called, useful for:
- Void methods that modify arguments
- Callbacks and async patterns
- Complex return value computation

```java
@Test
void shouldSimulateCallbackWithDoAnswer() {
    doAnswer(invocation -> {
        Runnable callback = invocation.getArgument(1);
        callback.run(); // Invoke the callback immediately
        return null;
    }).when(asyncProcessor).process(any(), any(Runnable.class));

    AtomicBoolean callbackInvoked = new AtomicBoolean(false);
    asyncProcessor.process(request, () -> callbackInvoked.set(true));

    assertTrue(callbackInvoked.get());
}

@Test
void shouldModifyArgumentInVoidMethod() {
    doAnswer(invocation -> {
        List<String> list = invocation.getArgument(0);
        list.add("injected");
        return null;
    }).when(populator).populate(anyList());

    List<String> result = new ArrayList<>();
    populator.populate(result);
    assertTrue(result.contains("injected"));
}
```

</details>

---

### 🟡 Q19. Explain `@MockBean` vs `@Mock` in a Spring Boot context.

<details><summary>Click to reveal answer</summary>

| Annotation | Context | Package |
|---|---|---|
| `@Mock` | Plain Mockito — no Spring context | `org.mockito` |
| `@MockBean` | Spring Boot Test — replaces bean in ApplicationContext | `org.springframework.boot.test.mock.mockito` |

```java
// Plain Mockito — fast, no Spring context
@ExtendWith(MockitoExtension.class)
class UserServiceUnitTest {
    @Mock UserRepository repo;
    @InjectMocks UserService userService;
}

// Spring Boot Test — slower, full/partial context
@SpringBootTest
class UserControllerIntegrationTest {

    @Autowired MockMvc mockMvc;

    @MockBean UserService userService; // Replaces real bean in context

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L))
            .thenReturn(new User(1L, "Alice"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

✅ **Best Practice:** Prefer `@Mock` + `@InjectMocks` for unit tests. Use `@MockBean` only when you need the Spring context (e.g., integration/slice tests).

</details>

---

## TDD Basics

### 🟢 Q20. What is TDD and what is the Red-Green-Refactor cycle?

<details><summary>Click to reveal answer</summary>

**Test-Driven Development (TDD)** is a development practice where you write tests *before* writing the implementation.

**Red-Green-Refactor cycle:**
1. 🔴 **Red** – Write a failing test that describes desired behavior
2. 🟢 **Green** – Write the minimal code to make the test pass
3. 🔵 **Refactor** – Clean up code without breaking the test

```java
// STEP 1: Red — write failing test
@Test
void shouldCalculateFibonacci() {
    assertEquals(0, fibonacci(0));
    assertEquals(1, fibonacci(1));
    assertEquals(1, fibonacci(2));
    assertEquals(5, fibonacci(5));
}

// STEP 2: Green — minimal implementation
public int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// STEP 3: Refactor — optimize if needed (e.g., memoization)
private Map<Integer, Integer> memo = new HashMap<>();
public int fibonacci(int n) {
    if (n <= 1) return n;
    return memo.computeIfAbsent(n, k -> fibonacci(k - 1) + fibonacci(k - 2));
}
```

> 💡 **Interviewer Tip:** Ask "What are the benefits of TDD?" — Candidates should mention: built-in regression safety, better design (testable = loosely coupled), living documentation.

</details>

---

### 🟡 Q21. What is the difference between unit tests, integration tests, and end-to-end tests?

<details><summary>Click to reveal answer</summary>

This is commonly explained with the **Test Pyramid**:

```
       /\
      /E2E\        ← Few, slow, expensive
     /------\
    /Integr. \     ← Moderate number
   /----------\
  /   Unit     \   ← Many, fast, cheap
 /--------------\
```

| Type | Scope | Speed | Dependencies |
|---|---|---|---|
| Unit | Single class/method | Milliseconds | Mocked |
| Integration | Multiple layers (e.g., Service + DB) | Seconds | Real or test containers |
| E2E | Full application stack | Minutes | Real environment |

```java
// Unit test — isolated, mocked
@ExtendWith(MockitoExtension.class)
class UserServiceUnitTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;
    // Fast, no DB
}

// Integration test — with real DB (e.g., H2 or TestContainers)
@SpringBootTest
@Transactional
class UserRepositoryIntegrationTest {
    @Autowired UserRepository repo;
    // Uses actual JPA/Hibernate
}

// E2E test — full HTTP stack
@SpringBootTest(webEnvironment = RANDOM_PORT)
class UserApiE2ETest {
    @Autowired TestRestTemplate restTemplate;
    // Actual HTTP calls
}
```

</details>

---

## Testing Best Practices

### 🟢 Q22. What makes a good unit test? (FIRST principles)

<details><summary>Click to reveal answer</summary>

**FIRST** is a mnemonic for good unit test qualities:

- **F**ast – Runs in milliseconds, no I/O
- **I**solated – No shared state between tests; each test is independent
- **R**epeatable – Same result every time, regardless of environment or order
- **S**elf-validating – Pass or fail automatically, no manual inspection
- **T**imely – Written alongside (or before) the code

```java
// ✅ Good unit test following FIRST
@Test
void shouldHashPasswordWithBCrypt() {
    // Fast: No DB, no network
    PasswordEncoder encoder = new BCryptPasswordEncoder();

    // Isolated: Creates fresh objects
    String raw = "securePassword123";
    String hashed = encoder.encode(raw);

    // Self-validating: Asserts result
    assertNotEquals(raw, hashed);
    assertTrue(encoder.matches(raw, hashed));

    // Repeatable: BCrypt is deterministic in matching
    assertTrue(encoder.matches(raw, hashed));
}
```

</details>

---

### 🟡 Q23. What is the AAA pattern in testing?

<details><summary>Click to reveal answer</summary>

**Arrange-Act-Assert (AAA)** is a pattern for structuring test methods for clarity:

- **Arrange** – Set up the test data and dependencies
- **Act** – Execute the code under test
- **Assert** – Verify the expected outcome

```java
@Test
void shouldApplyDiscountForPremiumCustomers() {
    // Arrange
    Customer customer = new Customer("Alice", CustomerType.PREMIUM);
    Order order = new Order(List.of(
        new Item("Laptop", 1000.0),
        new Item("Mouse", 50.0)
    ));
    DiscountService discountService = new DiscountService();

    // Act
    double discountedTotal = discountService.applyDiscount(customer, order);

    // Assert
    assertEquals(945.0, discountedTotal, 0.001); // 10% off for premium
}
```

✅ **Best Practice:** Add blank lines between Arrange/Act/Assert sections to visually separate them. Each test should have only ONE Act section.

</details>

---

### 🟡 Q24. How should you name test methods?

<details><summary>Click to reveal answer</summary>

Good test names describe **what** is being tested, **under what condition**, and **what the expected outcome** is.

Common naming patterns:

```java
// Pattern 1: should_ExpectedBehavior_WhenCondition
@Test void should_ReturnUser_WhenValidIdProvided() {}
@Test void should_ThrowException_WhenUserNotFound() {}

// Pattern 2: methodName_Condition_ExpectedResult
@Test void findById_ValidId_ReturnsUser() {}
@Test void findById_InvalidId_ThrowsNotFoundException() {}

// Pattern 3: Plain English (popular in BDD)
@Test void returnsUserWhenIdExists() {}
@Test void throwsNotFoundExceptionForMissingUser() {}

// Pattern 4: Given-When-Then style
@Test void givenPremiumUser_whenPlacingOrder_thenAppliesDiscount() {}
```

🚨 **Common Mistake:** Generic names like `test1()`, `testSuccess()`, or `testFindById()`. These don't communicate intent.

> 💡 **Interviewer Tip:** Ask candidates to name a test for "creating a user with a duplicate email should throw DuplicateEmailException" — good answers reveal how they communicate intent through code.

</details>

---

### 🔴 Q25. What is test coverage and what are its limitations?

<details><summary>Click to reveal answer</summary>

**Test coverage** measures what percentage of code lines, branches, or paths are executed by tests. Common tools: JaCoCo, Cobertura.

```xml
<!-- JaCoCo Maven configuration -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**Limitations of coverage metrics:**

1. **100% coverage ≠ bug-free** — You can cover every line without testing meaningful scenarios
2. **Doesn't measure assertion quality** — A test with no assertions can achieve 100% coverage
3. **Doesn't test edge cases** — Hitting a line once doesn't mean all branches are tested
4. **Can incentivize bad tests** — Teams write tests *just* to hit a coverage number

```java
// 100% coverage but useless test:
@Test
void coverageButNoAssertions() {
    calculator.add(1, 2); // Covers the line, but asserts nothing!
}

// ✅ Meaningful test:
@Test
void additionReturnsCorrectSum() {
    assertEquals(3, calculator.add(1, 2));
    assertEquals(0, calculator.add(-1, 1));
    assertEquals(Integer.MAX_VALUE, calculator.add(Integer.MAX_VALUE, 0));
}
```

</details>

---

### 🟡 Q26. What is mutation testing?

<details><summary>Click to reveal answer</summary>

Mutation testing evaluates the quality of tests by deliberately introducing small bugs (mutations) into the code and checking if tests catch them. Tool: **PITest** for Java.

Common mutations:
- Changing `>` to `>=`
- Negating conditionals
- Changing `+` to `-`
- Removing return statements

```xml
<!-- PITest Maven plugin -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <configuration>
        <targetClasses>
            <param>com.example.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.*Test</param>
        </targetTests>
    </configuration>
</plugin>
```

```java
// Original code
public boolean isAdult(int age) {
    return age >= 18;  // Mutation: change to age > 18
}

// Weak test — doesn't catch mutation
@Test void testAdult() { assertTrue(isAdult(20)); }

// ✅ Strong test — catches the boundary mutation
@Test void testAdultBoundaries() {
    assertFalse(isAdult(17));
    assertTrue(isAdult(18)); // This catches the >= vs > mutation!
    assertTrue(isAdult(19));
}
```

</details>

---

### 🟡 Q27. How do you test exception messages and causes?

<details><summary>Click to reveal answer</summary>

```java
@Test
void shouldThrowWithCorrectMessageAndCause() {
    // assertThrows returns the exception
    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> userService.createUser(null, "email@test.com")
    );

    // Assert on message
    assertEquals("Name must not be null", ex.getMessage());
    // Or use contains for partial match
    assertTrue(ex.getMessage().contains("null"));
}

@Test
void shouldWrapOriginalException() {
    // Test wrapped exceptions (e.g., service wrapping repository exception)
    when(repository.save(any())).thenThrow(new DataIntegrityViolationException("duplicate"));

    ServiceException ex = assertThrows(
        ServiceException.class,
        () -> userService.createUser("Alice", "alice@test.com")
    );

    assertNotNull(ex.getCause());
    assertInstanceOf(DataIntegrityViolationException.class, ex.getCause());
    assertTrue(ex.getCause().getMessage().contains("duplicate"));
}
```

</details>

---

### 🟡 Q28. What are TestContainers and when would you use them?

<details><summary>Click to reveal answer</summary>

TestContainers is a Java library that provides lightweight, throwaway Docker containers for integration testing. Instead of H2 in-memory DB, you test against a real PostgreSQL/MySQL/MongoDB container.

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired UserRepository userRepository;

    @Test
    void shouldPersistAndRetrieveUser() {
        User user = userRepository.save(new User("Alice", "alice@example.com"));
        Optional<User> found = userRepository.findById(user.getId());

        assertTrue(found.isPresent());
        assertEquals("Alice", found.get().getName());
    }
}
```

✅ **Best Practice:** Use `@Container` with `static` for class-level containers (shared across tests) or instance-level for isolated containers per test.

</details>

---

### 🔴 Q29. How do you test time-dependent code?

<details><summary>Click to reveal answer</summary>

Avoid `System.currentTimeMillis()` or `LocalDateTime.now()` directly — they're not testable. Use a `Clock` abstraction.

```java
// ✅ Testable design using Clock
class TokenService {
    private final Clock clock;

    TokenService(Clock clock) {
        this.clock = clock;
    }

    public boolean isExpired(Instant expiresAt) {
        return Instant.now(clock).isAfter(expiresAt);
    }
}

// Production usage
TokenService service = new TokenService(Clock.systemUTC());

// Test with fixed clock
@Test
void shouldDetectExpiredToken() {
    Instant fixedNow = Instant.parse("2024-01-15T12:00:00Z");
    Clock fixedClock = Clock.fixed(fixedNow, ZoneOffset.UTC);
    TokenService service = new TokenService(fixedClock);

    Instant expiredAt = fixedNow.minusSeconds(60);
    assertTrue(service.isExpired(expiredAt));
}

@Test
void shouldDetectValidToken() {
    Instant fixedNow = Instant.parse("2024-01-15T12:00:00Z");
    Clock fixedClock = Clock.fixed(fixedNow, ZoneOffset.UTC);
    TokenService service = new TokenService(fixedClock);

    Instant futureExpiry = fixedNow.plusSeconds(3600);
    assertFalse(service.isExpired(futureExpiry));
}
```

</details>

---

### 🟡 Q30. What is the difference between `thenReturn()` and `thenAnswer()`?

<details><summary>Click to reveal answer</summary>

- `thenReturn(value)` – Returns a fixed, pre-defined value
- `thenAnswer(invocation -> ...)` – Computes return value dynamically at call time

```java
@Test
void thenReturnVsThenAnswer() {
    List<String> mutableList = new ArrayList<>();

    // thenReturn captures the value at stubbing time
    when(service.getItems()).thenReturn(mutableList);
    mutableList.add("item");
    assertEquals(1, service.getItems().size()); // returns same reference

    // thenAnswer computes each time called
    AtomicInteger callCount = new AtomicInteger(0);
    when(idGenerator.nextId()).thenAnswer(inv -> callCount.incrementAndGet());

    assertEquals(1, idGenerator.nextId());
    assertEquals(2, idGenerator.nextId());
    assertEquals(3, idGenerator.nextId());
}

// Practical use: returning argument back
@Test
void shouldReturnSavedEntity() {
    when(repository.save(any(User.class)))
        .thenAnswer(invocation -> {
            User user = invocation.getArgument(0);
            user.setId(UUID.randomUUID());
            return user;
        });

    User saved = repository.save(new User("Alice"));
    assertNotNull(saved.getId()); // ID was set by the answer
}
```

> 💡 **Interviewer Tip:** `thenAnswer` is particularly useful for testing that the saved entity has auto-generated fields set correctly.

</details>

---

### 🔴 Q31. How would you structure a test for a class with multiple dependencies?

<details><summary>Click to reveal answer</summary>

```java
// Class under test with multiple dependencies
class CheckoutService {
    private final CartRepository cartRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    // Constructor injection (testable!)
    CheckoutService(CartRepository cartRepository,
                    InventoryService inventoryService,
                    PaymentService paymentService,
                    OrderRepository orderRepository,
                    NotificationService notificationService) {
        this.cartRepository = cartRepository;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }

    public Order checkout(Long cartId, String paymentToken) {
        Cart cart = cartRepository.findById(cartId)
            .orElseThrow(() -> new CartNotFoundException(cartId));
        inventoryService.reserve(cart.getItems());
        Payment payment = paymentService.charge(paymentToken, cart.getTotal());
        Order order = orderRepository.save(new Order(cart, payment));
        notificationService.sendConfirmation(order);
        return order;
    }
}

@ExtendWith(MockitoExtension.class)
class CheckoutServiceTest {

    @Mock CartRepository cartRepository;
    @Mock InventoryService inventoryService;
    @Mock PaymentService paymentService;
    @Mock OrderRepository orderRepository;
    @Mock NotificationService notificationService;

    @InjectMocks CheckoutService checkoutService;

    // Shared test fixtures
    private Cart testCart;
    private final Long CART_ID = 1L;

    @BeforeEach
    void setUp() {
        testCart = new Cart(CART_ID, List.of(new CartItem("Book", 2, 29.99)));
        when(cartRepository.findById(CART_ID)).thenReturn(Optional.of(testCart));
    }

    @Test
    void shouldCompleteCheckoutSuccessfully() {
        Payment payment = new Payment("pay-123", testCart.getTotal());
        Order expectedOrder = new Order(testCart, payment);

        when(paymentService.charge(anyString(), anyDouble())).thenReturn(payment);
        when(orderRepository.save(any())).thenReturn(expectedOrder);

        Order result = checkoutService.checkout(CART_ID, "tok-abc");

        assertNotNull(result);
        verify(inventoryService).reserve(testCart.getItems());
        verify(notificationService).sendConfirmation(expectedOrder);
    }

    @Test
    void shouldThrowWhenCartNotFound() {
        when(cartRepository.findById(999L)).thenReturn(Optional.empty());

        assertThrows(CartNotFoundException.class,
            () -> checkoutService.checkout(999L, "tok-abc"));

        verifyNoInteractions(paymentService, orderRepository, notificationService);
    }
}
```

</details>

---

### 🟢 Q32. What is the difference between `assertEquals` and `assertSame`?

<details><summary>Click to reveal answer</summary>

- `assertEquals(expected, actual)` – Checks **value equality** using `.equals()`
- `assertSame(expected, actual)` – Checks **reference equality** using `==`

```java
@Test
void equalityVsIdentity() {
    String a = new String("hello");
    String b = new String("hello");

    assertEquals(a, b);   // ✅ Passes — same content
    assertNotSame(a, b);  // ✅ Passes — different objects

    String c = a;
    assertSame(a, c);     // ✅ Passes — same reference
}

@Test
void cacheReturnsIdenticalObject() {
    UserCache cache = new UserCache();
    User first = cache.get(1L);
    User second = cache.get(1L);

    // assertSame verifies the cache works (same instance returned)
    assertSame(first, second);
}
```

🚨 **Common Mistake:** Using `assertSame` for value comparisons with `String` or `Integer` — it works sometimes due to JVM string/integer pooling, but is unreliable.

</details>

---

*End of Java Testing (JUnit 5 & Mockito)*
