# REST API Design & Validation

> 💡 **Interviewer Tip:** REST API design questions blend Spring MVC knowledge with HTTP fundamentals and Bean Validation. Expect code-based questions on `@Valid` vs `@Validated`, custom validators, API versioning trade-offs, and proper use of `ResponseEntity`.

---

## Table of Contents
1. [Core REST Annotations](#core-rest-annotations)
2. [Bean Validation](#bean-validation)
3. [API Versioning Strategies](#api-versioning-strategies)
4. [Interview Questions](#interview-questions)

---

## Core REST Annotations

| Annotation | Purpose |
|-----------|---------|
| `@RestController` | `@Controller` + `@ResponseBody` — serializes return values to JSON/XML |
| `@RequestMapping` | Maps HTTP requests to handler methods (class or method level) |
| `@GetMapping` | Shorthand for `@RequestMapping(method = GET)` |
| `@PostMapping` | Shorthand for `@RequestMapping(method = POST)` |
| `@PutMapping` | Shorthand for `@RequestMapping(method = PUT)` |
| `@PatchMapping` | Shorthand for `@RequestMapping(method = PATCH)` |
| `@DeleteMapping` | Shorthand for `@RequestMapping(method = DELETE)` |
| `@PathVariable` | Binds URI template variable to method parameter |
| `@RequestParam` | Binds query string parameter to method parameter |
| `@RequestBody` | Deserializes request body (JSON/XML) to Java object |
| `@RequestHeader` | Binds HTTP header to method parameter |
| `@ResponseStatus` | Sets default HTTP status code for a controller method |

---

## Interview Questions

### 🟢 Easy

---

**Q1. What is the difference between `@Controller` and `@RestController`?**

<details><summary>Click to reveal answer</summary>

`@RestController` is a convenience annotation that combines `@Controller` and `@ResponseBody`:

```java
// Traditional @Controller — used for view-based MVC (returns view names)
@Controller
public class WebController {

    @GetMapping("/dashboard")
    public String dashboard(Model model) {
        model.addAttribute("user", getCurrentUser());
        return "dashboard"; // returns view name (e.g., dashboard.html)
    }

    @GetMapping("/api/users")
    @ResponseBody // required to serialize return value to response body
    public List<User> getUsers() {
        return userService.findAll();
    }
}

// @RestController — every method is implicitly @ResponseBody
@RestController
@RequestMapping("/api/v1")
public class UserController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll(); // automatically serialized to JSON
    }

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

```java
// Equivalent definitions:
@RestController
// is exactly the same as:
@Controller
@ResponseBody
```

✅ **Best Practice:** Always use `@RestController` for REST APIs. Use `@Controller` only when your controller also serves HTML views (mixed MVC + API).

</details>

---

**Q2. What is the difference between `@PathVariable` and `@RequestParam`?**

<details><summary>Click to reveal answer</summary>

```java
@RestController
@RequestMapping("/api/v1")
public class ProductController {

    // @PathVariable — part of the URL path: /products/42
    @GetMapping("/products/{id}")
    public Product getById(@PathVariable Long id) {
        return productService.findById(id);
    }

    // @PathVariable with name mapping
    @GetMapping("/categories/{categoryId}/products/{productId}")
    public Product getByCategory(
            @PathVariable("categoryId") Long catId,
            @PathVariable Long productId) {
        return productService.findByCategoryAndId(catId, productId);
    }

    // @RequestParam — query string: /products?page=0&size=10&sort=name
    @GetMapping("/products")
    public Page<Product> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String sort,
            @RequestParam(required = false) String search) {
        return productService.findAll(page, size, sort, search);
    }

    // Optional RequestParam with Optional type
    @GetMapping("/products/search")
    public List<Product> search(
            @RequestParam Optional<String> name,
            @RequestParam Optional<BigDecimal> maxPrice) {
        return productService.search(name.orElse(""), maxPrice.orElse(null));
    }
}
```

| | `@PathVariable` | `@RequestParam` |
|---|---|---|
| Location | URL path segment | Query string |
| Example | `/users/42` | `/users?id=42` |
| Required by default | Yes | Yes (can set `required=false`) |
| Use for | Resource identification | Filtering, pagination, options |

> 💡 **Interviewer Tip:** REST convention — use path variables for resource IDs (nouns), query params for filtering/sorting/pagination.

</details>

---

**Q3. How does `@RequestBody` work and what media type does it expect?**

<details><summary>Click to reveal answer</summary>

`@RequestBody` uses a `HttpMessageConverter` (typically `MappingJackson2HttpMessageConverter`) to deserialize the request body into a Java object.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    // Spring reads Content-Type header and picks appropriate converter
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User createUser(@RequestBody @Valid CreateUserRequest request) {
        return userService.create(request);
    }

    @PutMapping("/{id}")
    public User updateUser(
            @PathVariable Long id,
            @RequestBody @Valid UpdateUserRequest request) {
        return userService.update(id, request);
    }

    // Optional request body (e.g., for PATCH with partial update)
    @PatchMapping("/{id}")
    public User patchUser(
            @PathVariable Long id,
            @RequestBody(required = false) Map<String, Object> updates) {
        return userService.patch(id, updates);
    }
}
```

```java
// DTO with validation
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    String name,

    @Email(message = "Must be a valid email")
    @NotBlank
    String email,

    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {}
```

🚨 **Common Mistake:** Forgetting `Content-Type: application/json` header in requests. Without it, Spring returns `415 Unsupported Media Type`.

```bash
# Correct request:
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com","password":"secret123"}'
```

</details>

---

**Q4. What does `ResponseEntity` provide and when should you use it?**

<details><summary>Click to reveal answer</summary>

`ResponseEntity` gives full control over the HTTP response: status code, headers, and body.

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @GetMapping("/{id}")
    public ResponseEntity<Product> getById(@PathVariable Long id) {
        return productService.findById(id)
            .map(product -> ResponseEntity.ok(product))
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<Product> create(@RequestBody @Valid CreateProductRequest req) {
        Product created = productService.create(req);
        URI location = URI.create("/api/v1/products/" + created.getId());
        return ResponseEntity
            .created(location)  // 201 Created + Location header
            .body(created);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build(); // 204 No Content
    }

    @GetMapping
    public ResponseEntity<List<Product>> list(
            @RequestParam(defaultValue = "0") int page) {
        List<Product> products = productService.findAll(page);
        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(productService.count()))
            .header("X-Page", String.valueOf(page))
            .body(products);
    }

    // Custom status
    @PostMapping("/bulk")
    public ResponseEntity<BulkResult> bulkCreate(
            @RequestBody List<CreateProductRequest> requests) {
        BulkResult result = productService.bulkCreate(requests);
        HttpStatus status = result.hasErrors() ? HttpStatus.MULTI_STATUS : HttpStatus.CREATED;
        return ResponseEntity.status(status).body(result);
    }
}
```

✅ **Best Practice:** Use `ResponseEntity` when you need to:
- Return different status codes based on logic (201 vs 200)
- Add custom response headers
- Return `void` responses with a specific status (204)
- Handle "not found" → 404 without throwing exceptions

</details>

---

**Q5. What are the core Bean Validation annotations?**

<details><summary>Click to reveal answer</summary>

```java
public class UserRegistrationRequest {

    // Null checks
    @NotNull(message = "ID must not be null")
    private Long id;

    @NotBlank(message = "Username must not be blank") // checks null AND empty/whitespace
    private String username;

    @NotEmpty(message = "Email must not be empty")   // checks null AND empty (not whitespace)
    private String email;

    // String constraints
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;

    @Email(message = "Must be a valid email address")
    private String emailAddress;

    @Pattern(regexp = "^[A-Za-z0-9+_.-]+@(.+)$", message = "Custom email regex")
    private String contactEmail;

    // Numeric constraints
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120, message = "Age must be realistic")
    private int age;

    @Positive(message = "Price must be positive")
    private BigDecimal price;

    @PositiveOrZero(message = "Quantity cannot be negative")
    private int quantity;

    @DecimalMin(value = "0.0", inclusive = false)
    @DecimalMax(value = "100.0")
    private BigDecimal discountPercent;

    // Date constraints
    @Past(message = "Birth date must be in the past")
    private LocalDate birthDate;

    @Future(message = "Expiry date must be in the future")
    private LocalDate expiryDate;

    @PastOrPresent
    private LocalDateTime lastLogin;

    // Boolean
    @AssertTrue(message = "You must accept the terms")
    private boolean termsAccepted;

    // Collection
    @NotEmpty
    @Size(max = 5)
    private List<@NotBlank String> tags;

    // Nested validation
    @Valid
    @NotNull
    private AddressRequest address;
}
```

```java
// Nested object also needs @Valid to cascade validation
public class AddressRequest {
    @NotBlank
    private String street;

    @NotBlank
    @Size(min = 2, max = 2)
    private String stateCode;

    @Pattern(regexp = "\\d{5}(-\\d{4})?")
    private String zipCode;
}
```

</details>

---

### 🟡 Medium

---

**Q6. What is the difference between `@Valid` and `@Validated`?**

<details><summary>Click to reveal answer</summary>

Both trigger Bean Validation, but they differ in capability:

| Feature | `@Valid` | `@Validated` |
|---------|---------|-------------|
| Source | JSR-303/380 (standard) | Spring-specific |
| Validation groups | ❌ Not supported | ✅ Supported |
| Cascaded validation | ✅ (`@Valid` on fields) | ✅ |
| Method-level validation | Limited | ✅ (on service classes) |

```java
// Validation groups
public interface OnCreate {}
public interface OnUpdate {}

public class UserRequest {
    @Null(groups = OnCreate.class)     // must be null on create
    @NotNull(groups = OnUpdate.class)  // must be present on update
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String name;

    @NotBlank(groups = OnCreate.class)
    @Email
    private String email;
}

@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @PostMapping
    public User create(
            @Validated(OnCreate.class) @RequestBody UserRequest req) {
        // id must be null, email is required
        return userService.create(req);
    }

    @PutMapping("/{id}")
    public User update(
            @PathVariable Long id,
            @Validated(OnUpdate.class) @RequestBody UserRequest req) {
        // id must not be null
        return userService.update(id, req);
    }
}
```

```java
// @Validated on service for method-level validation
@Service
@Validated
public class ProductService {

    public Product findById(@NotNull @Positive Long id) {
        // Spring validates parameters — throws ConstraintViolationException if invalid
        return repository.findById(id).orElseThrow();
    }

    public Product create(@Valid @NotNull CreateProductRequest req) {
        return repository.save(mapper.toEntity(req));
    }
}
```

🚨 **Common Mistake:** Using `@Valid` with validation groups — it simply ignores the groups. Use `@Validated(GroupName.class)` for group-specific validation.

</details>

---

**Q7. How do you create a custom Bean Validation constraint?**

<details><summary>Click to reveal answer</summary>

Creating a custom validator requires two things: the annotation and the `ConstraintValidator` implementation.

```java
// 1. Define the annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = PhoneNumberValidator.class)
public @interface ValidPhoneNumber {
    String message() default "Must be a valid phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String countryCode() default "US";
}

// 2. Implement ConstraintValidator
public class PhoneNumberValidator
        implements ConstraintValidator<ValidPhoneNumber, String> {

    private String countryCode;

    @Override
    public void initialize(ValidPhoneNumber annotation) {
        this.countryCode = annotation.countryCode();
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true; // null handled by @NotNull separately

        // Use libphonenumber or simple regex
        String pattern = switch (countryCode) {
            case "US" -> "^\\+?1?\\s?\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}$";
            case "UK" -> "^\\+?44\\s?\\d{10}$";
            default   -> "^\\+?\\d{7,15}$";
        };

        return value.matches(pattern);
    }
}

// 3. Use the annotation
public class ContactRequest {
    @ValidPhoneNumber(countryCode = "US")
    private String phoneNumber;

    @ValidPhoneNumber(countryCode = "UK", message = "Invalid UK phone number")
    private String ukPhone;
}
```

**Class-level validator (cross-field validation):**
```java
// Validates that password and confirmPassword match
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String password() default "password";
    String confirmPassword() default "confirmPassword";
}

public class PasswordMatchValidator
        implements ConstraintValidator<PasswordMatch, Object> {

    private String passwordField;
    private String confirmField;

    @Override
    public void initialize(PasswordMatch annotation) {
        this.passwordField = annotation.password();
        this.confirmField = annotation.confirmPassword();
    }

    @Override
    public boolean isValid(Object obj, ConstraintValidatorContext ctx) {
        try {
            Object password = new BeanWrapperImpl(obj).getPropertyValue(passwordField);
            Object confirm  = new BeanWrapperImpl(obj).getPropertyValue(confirmField);

            boolean valid = Objects.equals(password, confirm);
            if (!valid) {
                ctx.disableDefaultConstraintViolation();
                ctx.buildConstraintViolationWithTemplate(ctx.getDefaultConstraintMessageTemplate())
                   .addPropertyNode(confirmField)
                   .addConstraintViolation();
            }
            return valid;
        } catch (Exception e) {
            return false;
        }
    }
}

// Usage
@PasswordMatch
public class RegistrationRequest {
    @NotBlank @Size(min = 8)
    private String password;

    @NotBlank
    private String confirmPassword;
}
```

</details>

---

**Q8. What are the different API versioning strategies and their trade-offs?**

<details><summary>Click to reveal answer</summary>

**Strategy 1: URL Path Versioning (most common)**
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserDtoV1 getUser(@PathVariable Long id) {
        return mapper.toDtoV1(userService.findById(id));
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserDtoV2 getUser(@PathVariable Long id) {
        return mapper.toDtoV2(userService.findById(id));
    }
}
```

**Strategy 2: Request Header Versioning**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(value = "/{id}", headers = "X-API-Version=1")
    public UserDtoV1 getUserV1(@PathVariable Long id) {
        return mapper.toDtoV1(userService.findById(id));
    }

    @GetMapping(value = "/{id}", headers = "X-API-Version=2")
    public UserDtoV2 getUserV2(@PathVariable Long id) {
        return mapper.toDtoV2(userService.findById(id));
    }
}
```

**Strategy 3: Media Type (Content Negotiation) Versioning**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(value = "/{id}",
                produces = "application/vnd.company.app-v1+json")
    public UserDtoV1 getUserV1(@PathVariable Long id) {
        return mapper.toDtoV1(userService.findById(id));
    }

    @GetMapping(value = "/{id}",
                produces = "application/vnd.company.app-v2+json")
    public UserDtoV2 getUserV2(@PathVariable Long id) {
        return mapper.toDtoV2(userService.findById(id));
    }
}
```

**Strategy 4: Query Parameter Versioning**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(value = "/{id}", params = "version=1")
    public UserDtoV1 getUserV1(@PathVariable Long id) {
        return mapper.toDtoV1(userService.findById(id));
    }

    @GetMapping(value = "/{id}", params = "version=2")
    public UserDtoV2 getUserV2(@PathVariable Long id) {
        return mapper.toDtoV2(userService.findById(id));
    }
}
```

**Trade-offs:**

| Strategy | Pros | Cons |
|---------|------|------|
| URL path | Simple, cacheable, visible | Pollutes URLs, not RESTfully pure |
| Header | Clean URLs, RESTfully correct | Not browser-friendly, harder to test |
| Media type | Most RESTful, standard | Complex, poor browser support |
| Query param | Easy to test in browser | Not cacheable-friendly, clutters URLs |

> 💡 **Interviewer Tip:** For most enterprise APIs, URL path versioning wins because of simplicity and cacheability. Media type versioning is theoretically "more correct" per REST principles.

</details>

---

**Q9. How does content negotiation work in Spring MVC?**

<details><summary>Click to reveal answer</summary>

Content negotiation determines the response format based on the client's `Accept` header.

```java
@RestController
@RequestMapping("/api/v1/reports")
public class ReportController {

    // Serves both JSON and XML based on Accept header
    @GetMapping(value = "/{id}",
                produces = {
                    MediaType.APPLICATION_JSON_VALUE,
                    MediaType.APPLICATION_XML_VALUE
                })
    public Report getReport(@PathVariable Long id) {
        return reportService.findById(id);
    }

    // Only serves specific CSV format
    @GetMapping(value = "/{id}/export",
                produces = "text/csv")
    public ResponseEntity<Resource> exportCsv(@PathVariable Long id) {
        byte[] csv = reportService.exportToCsv(id);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION,
                    "attachment; filename=\"report-" + id + ".csv\"")
            .contentType(MediaType.parseMediaType("text/csv"))
            .body(new ByteArrayResource(csv));
    }
}
```

```java
// Configuration for XML support
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(false)               // disable ?format=json
            .favorPathExtension(false)           // disable .json extension
            .ignoreAcceptHeader(false)           // use Accept header (default)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

```xml
<!-- pom.xml — add for XML support -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

```java
// DTO that serializes to both JSON and XML
@JacksonXmlRootElement(localName = "report")
public class Report {
    @JsonProperty("reportId")
    @JacksonXmlProperty(localName = "report-id")
    private Long id;

    private String title;
    private LocalDate generatedAt;
}
```

</details>

---

**Q10. How do you handle `@RequestHeader` and what are common use cases?**

<details><summary>Click to reveal answer</summary>

```java
@RestController
@RequestMapping("/api/v1")
public class ApiController {

    // Single header — required by default
    @GetMapping("/data")
    public ResponseEntity<Data> getData(
            @RequestHeader("X-API-Key") String apiKey,
            @RequestHeader("X-Request-ID") String requestId) {

        auditService.log(requestId);
        return ResponseEntity.ok(dataService.fetch(apiKey));
    }

    // Optional header with default
    @GetMapping("/products")
    public List<Product> getProducts(
            @RequestHeader(value = "Accept-Language",
                           defaultValue = "en") String language,
            @RequestHeader(value = "X-Tenant-ID",
                           required = false) String tenantId) {
        return productService.findAll(language, tenantId);
    }

    // All headers as a map
    @GetMapping("/debug")
    public Map<String, String> debugHeaders(
            @RequestHeader Map<String, String> headers) {
        return headers;
    }

    // HttpEntity gives access to both headers and body
    @PostMapping("/events")
    public ResponseEntity<String> processEvent(
            HttpEntity<EventPayload> entity) {
        HttpHeaders headers = entity.getHeaders();
        String signature = headers.getFirst("X-Signature");
        EventPayload payload = entity.getBody();

        if (!signatureService.verify(payload, signature)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        eventService.process(payload);
        return ResponseEntity.accepted().build();
    }
}
```

**Common real-world header uses:**
- `X-API-Key` — API authentication
- `X-Request-ID` — distributed tracing
- `X-Tenant-ID` — multi-tenancy routing
- `Authorization` — Bearer token / Basic auth
- `X-Forwarded-For` — original client IP behind proxy

</details>

---

**Q11. How do you implement HATEOAS in a Spring Boot REST API?**

<details><summary>Click to reveal answer</summary>

HATEOAS (Hypermedia As The Engine Of Application State) adds links to responses so clients can navigate the API.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
// Model class extends RepresentationModel
public class UserModel extends RepresentationModel<UserModel> {
    private Long id;
    private String name;
    private String email;

    // constructors, getters, setters
}

@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping("/{id}")
    public EntityModel<UserModel> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        UserModel model = mapper.toModel(user);

        return EntityModel.of(model,
            // Self link
            linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
            // Related links
            linkTo(methodOn(UserController.class).listOrders(id)).withRel("orders"),
            linkTo(methodOn(UserController.class).updateUser(id, null)).withRel("update"),
            linkTo(methodOn(UserController.class).deleteUser(id)).withRel("delete")
        );
    }

    @GetMapping
    public CollectionModel<EntityModel<UserModel>> listUsers() {
        List<EntityModel<UserModel>> users = userService.findAll().stream()
            .map(user -> {
                UserModel model = mapper.toModel(user);
                return EntityModel.of(model,
                    linkTo(methodOn(UserController.class)
                        .getUser(user.getId())).withSelfRel()
                );
            })
            .collect(Collectors.toList());

        return CollectionModel.of(users,
            linkTo(methodOn(UserController.class).listUsers()).withSelfRel()
        );
    }
}
```

```json
// Response with HATEOAS links:
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self": { "href": "http://localhost:8080/api/v1/users/1" },
    "orders": { "href": "http://localhost:8080/api/v1/users/1/orders" },
    "update": { "href": "http://localhost:8080/api/v1/users/1" },
    "delete": { "href": "http://localhost:8080/api/v1/users/1" }
  }
}
```

> 💡 **Interviewer Tip:** HATEOAS is the highest level of Richardson Maturity Model (Level 3). Most real-world APIs stop at Level 2 (HTTP verbs + resource naming). Know the concept but acknowledge it's rarely fully implemented.

</details>

---

### 🔴 Hard

---

**Q12. How do you implement a custom `HandlerMethodArgumentResolver`?**

<details><summary>Click to reveal answer</summary>

`HandlerMethodArgumentResolver` allows you to inject custom objects into controller method parameters.

```java
// Custom annotation
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentUser {}

// The resolver
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    private final JwtTokenService tokenService;
    private final UserRepository userRepository;

    public CurrentUserArgumentResolver(JwtTokenService ts, UserRepository ur) {
        this.tokenService = ts;
        this.userRepository = ur;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class)
            && parameter.getParameterType().equals(User.class);
    }

    @Override
    public Object resolveArgument(
            MethodParameter parameter,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            WebDataBinderFactory binderFactory) throws Exception {

        HttpServletRequest request =
            webRequest.getNativeRequest(HttpServletRequest.class);

        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            throw new UnauthorizedException("Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);
        Long userId = tokenService.extractUserId(token);
        return userRepository.findById(userId)
            .orElseThrow(() -> new UnauthorizedException("User not found"));
    }
}

// Registration
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired private JwtTokenService tokenService;
    @Autowired private UserRepository userRepository;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver(tokenService, userRepository));
    }
}

// Usage in controller — clean and elegant
@RestController
@RequestMapping("/api/v1")
public class ProfileController {

    @GetMapping("/me")
    public UserProfile getProfile(@CurrentUser User user) {
        return profileService.getProfile(user);
    }

    @PostMapping("/orders")
    public Order createOrder(
            @CurrentUser User user,
            @RequestBody @Valid CreateOrderRequest req) {
        return orderService.create(user, req);
    }
}
```

</details>

---

**Q13. How do you implement pagination and sorting in a REST API?**

<details><summary>Click to reveal answer</summary>

Spring Data's `Pageable` integrates seamlessly with Spring MVC.

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @Autowired private ProductRepository repository;

    // Spring auto-resolves Pageable from query params:
    // GET /api/v1/products?page=0&size=20&sort=price,asc&sort=name,desc
    @GetMapping
    public Page<ProductDto> list(
            @PageableDefault(size = 20, sort = "createdAt",
                             direction = Sort.Direction.DESC)
            Pageable pageable) {
        return repository.findAll(pageable).map(mapper::toDto);
    }

    // Custom paginated response
    @GetMapping("/search")
    public ResponseEntity<PagedResponse<ProductDto>> search(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String category,
            Pageable pageable) {

        Page<Product> page = repository.search(name, category, pageable);

        PagedResponse<ProductDto> response = PagedResponse.<ProductDto>builder()
            .content(page.getContent().stream().map(mapper::toDto).toList())
            .page(page.getNumber())
            .size(page.getSize())
            .totalElements(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .last(page.isLast())
            .build();

        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(page.getTotalElements()))
            .body(response);
    }
}

// Enable pageable in configuration
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        PageableHandlerMethodArgumentResolver resolver =
            new PageableHandlerMethodArgumentResolver();
        resolver.setMaxPageSize(100); // prevent large page requests
        resolver.setFallbackPageable(PageRequest.of(0, 20));
        resolvers.add(resolver);
    }
}
```

```java
// Custom pageable-aware repository query
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p WHERE " +
           "(:name IS NULL OR LOWER(p.name) LIKE LOWER(CONCAT('%', :name, '%'))) AND " +
           "(:category IS NULL OR p.category = :category)")
    Page<Product> search(
        @Param("name") String name,
        @Param("category") String category,
        Pageable pageable);
}
```

```java
// PagedResponse DTO
@Builder
public class PagedResponse<T> {
    private List<T> content;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private boolean last;
}
```

</details>

---

**Q14. How do you implement rate limiting in a Spring Boot REST API?**

<details><summary>Click to reveal answer</summary>

```java
// Using Bucket4j (token bucket algorithm)
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Cache<String, Bucket> buckets;

    public RateLimitingFilter(CacheManager cacheManager) {
        this.buckets = cacheManager.getCache("rateLimits");
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        String clientId = extractClientId(request);
        Bucket bucket = buckets.get(clientId, this::createNewBucket);

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        if (probe.isConsumed()) {
            response.setHeader("X-Rate-Limit-Remaining",
                String.valueOf(probe.getRemainingTokens()));
            chain.doFilter(request, response);
        } else {
            long waitForRefill = probe.getNanosToWaitForRefill() / 1_000_000_000;
            response.setHeader("X-Rate-Limit-Retry-After-Seconds",
                String.valueOf(waitForRefill));
            response.sendError(HttpStatus.TOO_MANY_REQUESTS.value(),
                "Rate limit exceeded. Try again in " + waitForRefill + " seconds.");
        }
    }

    private Bucket createNewBucket() {
        // 100 requests per minute
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100,
                Refill.intervally(100, Duration.ofMinutes(1))))
            .build();
    }

    private String extractClientId(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        return apiKey != null ? apiKey : request.getRemoteAddr();
    }
}
```

```java
// Annotation-based rate limiting with AOP
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int requests() default 100;
    long duration() default 60;
    TimeUnit unit() default TimeUnit.SECONDS;
}

@RestController
public class SearchController {

    @GetMapping("/api/v1/search")
    @RateLimit(requests = 10, duration = 1, unit = TimeUnit.MINUTES)
    public List<SearchResult> search(@RequestParam String query) {
        return searchService.search(query);
    }
}
```

</details>

---

**Q15. How do you handle file upload and download in a Spring Boot REST API?**

<details><summary>Click to reveal answer</summary>

```java
@RestController
@RequestMapping("/api/v1/files")
public class FileController {

    @Value("${app.upload.dir}")
    private String uploadDir;

    // Single file upload
    @PostMapping("/upload")
    public ResponseEntity<FileUploadResponse> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam(required = false) String description) {

        if (file.isEmpty()) {
            return ResponseEntity.badRequest().build();
        }

        // Validate file type
        String contentType = file.getContentType();
        if (!isAllowedType(contentType)) {
            return ResponseEntity.status(HttpStatus.UNSUPPORTED_MEDIA_TYPE).build();
        }

        String filename = UUID.randomUUID() + "_" +
            StringUtils.cleanPath(file.getOriginalFilename());

        try {
            Path targetPath = Paths.get(uploadDir).resolve(filename);
            Files.copy(file.getInputStream(), targetPath,
                StandardCopyOption.REPLACE_EXISTING);

            FileUploadResponse response = new FileUploadResponse(
                filename, file.getSize(), contentType,
                "/api/v1/files/download/" + filename
            );
            return ResponseEntity.status(HttpStatus.CREATED).body(response);

        } catch (IOException e) {
            throw new StorageException("Failed to store file", e);
        }
    }

    // Multiple files
    @PostMapping("/upload/multiple")
    public List<FileUploadResponse> uploadMultiple(
            @RequestParam("files") List<MultipartFile> files) {
        return files.stream()
            .map(this::processFile)
            .collect(Collectors.toList());
    }

    // File download
    @GetMapping("/download/{filename:.+}")
    public ResponseEntity<Resource> downloadFile(
            @PathVariable String filename,
            HttpServletRequest request) {

        Path filePath = Paths.get(uploadDir).resolve(filename).normalize();
        Resource resource = new UrlResource(filePath.toUri());

        if (!resource.exists()) {
            return ResponseEntity.notFound().build();
        }

        String contentType = determineContentType(request, resource);

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType))
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + resource.getFilename() + "\"")
            .body(resource);
    }

    private boolean isAllowedType(String contentType) {
        return contentType != null && (
            contentType.startsWith("image/") ||
            contentType.equals("application/pdf") ||
            contentType.equals("application/octet-stream")
        );
    }
}
```

```yaml
# application.yml — multipart config
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 50MB
      enabled: true
```

🚨 **Common Mistake:** Using the original filename directly from `getOriginalFilename()` in file paths — this is a **path traversal security vulnerability**. Always sanitize with `StringUtils.cleanPath()` and store files with generated names.

</details>

---

**Q16. How do you implement request/response logging in Spring Boot?**

<details><summary>Click to reveal answer</summary>

```java
// Using CommonsRequestLoggingFilter (built-in)
@Configuration
public class RequestLoggingConfig {

    @Bean
    public CommonsRequestLoggingFilter requestLoggingFilter() {
        CommonsRequestLoggingFilter filter = new CommonsRequestLoggingFilter();
        filter.setIncludeQueryString(true);
        filter.setIncludePayload(true);
        filter.setMaxPayloadLength(10000);
        filter.setIncludeHeaders(true);
        filter.setAfterMessagePrefix("REQUEST DATA: ");
        return filter;
    }
}
```

```java
// Custom logging filter with correlation IDs
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestResponseLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        String requestId = Optional
            .ofNullable(request.getHeader("X-Request-ID"))
            .orElse(UUID.randomUUID().toString());

        // Add to MDC for log correlation
        MDC.put("requestId", requestId);
        response.setHeader("X-Request-ID", requestId);

        long startTime = System.currentTimeMillis();
        try {
            log.info("Request: {} {} from {}",
                request.getMethod(), request.getRequestURI(), request.getRemoteAddr());
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Response: {} {} → {} in {}ms",
                request.getMethod(), request.getRequestURI(),
                response.getStatus(), duration);
            MDC.clear();
        }
    }
}
```

</details>

---

**Q17. What is the difference between `@RequestMapping` `consumes` and `produces`?**

<details><summary>Click to reveal answer</summary>

```java
@RestController
@RequestMapping("/api/v1")
public class DataController {

    // consumes — restricts which Content-Type the method accepts (request body format)
    // produces — restricts which Accept types the method serves (response body format)

    @PostMapping(
        value = "/data",
        consumes = MediaType.APPLICATION_JSON_VALUE,    // client must send JSON
        produces = MediaType.APPLICATION_JSON_VALUE     // response will be JSON
    )
    public DataResponse processJson(@RequestBody DataRequest request) {
        return service.process(request);
    }

    @PostMapping(
        value = "/data",
        consumes = MediaType.APPLICATION_XML_VALUE,     // client must send XML
        produces = MediaType.APPLICATION_XML_VALUE      // response will be XML
    )
    public DataResponse processXml(@RequestBody DataRequest request) {
        return service.process(request);
    }

    // Accept multiple input types, produce only JSON
    @PostMapping(
        value = "/events",
        consumes = {MediaType.APPLICATION_JSON_VALUE,
                    "application/vnd.mycompany.event+json"},
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public EventResult processEvent(@RequestBody EventRequest request) {
        return eventService.process(request);
    }

    // Negate: accept anything EXCEPT XML
    @PostMapping(
        value = "/upload",
        consumes = "!" + MediaType.APPLICATION_XML_VALUE
    )
    public UploadResult upload(@RequestBody byte[] data) {
        return storageService.store(data);
    }
}
```

- **`consumes`** maps to the request's `Content-Type` header
- **`produces`** maps to the request's `Accept` header
- If client sends a `Content-Type` not matching `consumes` → `415 Unsupported Media Type`
- If client sends an `Accept` not matching `produces` → `406 Not Acceptable`

</details>

---

**Q18. How do you validate that an enum value in a request is valid?**

<details><summary>Click to reveal answer</summary>

```java
// Option 1: Custom @ValidEnum annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = EnumValidator.class)
public @interface ValidEnum {
    Class<? extends Enum<?>> enumClass();
    String message() default "must be a valid enum value";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class EnumValidator implements ConstraintValidator<ValidEnum, String> {

    private Set<String> validValues;

    @Override
    public void initialize(ValidEnum annotation) {
        validValues = Arrays.stream(annotation.enumClass().getEnumConstants())
            .map(Enum::name)
            .collect(Collectors.toSet());
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true;
        boolean valid = validValues.contains(value.toUpperCase());
        if (!valid) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate(
                "must be one of: " + validValues
            ).addConstraintViolation();
        }
        return valid;
    }
}

// Usage
public class OrderRequest {
    @ValidEnum(enumClass = OrderStatus.class)
    private String status;

    @ValidEnum(enumClass = PaymentMethod.class,
               message = "Invalid payment method")
    private String paymentMethod;
}

// Option 2: Use JsonCreator for deserialization-time validation
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED;

    @JsonCreator
    public static OrderStatus fromString(String value) {
        try {
            return OrderStatus.valueOf(value.toUpperCase());
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException(
                "Invalid status: " + value + ". Valid values: " +
                Arrays.toString(OrderStatus.values()));
        }
    }
}
```

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@PathVariable` | URL segment (`/users/{id}`) |
| `@RequestParam` | Query string (`?page=0&size=10`) |
| `@Valid` | Triggers Bean Validation, no group support |
| `@Validated` | Supports validation groups, Spring-specific |
| `ResponseEntity` | Full control over status, headers, body |
| API versioning | URL path (most common), header, media type |
| `consumes` | Restricts accepted `Content-Type` |
| `produces` | Restricts served `Accept` type |
| Custom validator | Implement `ConstraintValidator<A, T>` |
