# Exception Handling with @ControllerAdvice

> 💡 **Interviewer Tip:** Exception handling questions test whether a candidate can build production-grade APIs. Interviewers look for knowledge of `@ControllerAdvice` vs `@RestControllerAdvice`, RFC 7807 Problem Details, and the ability to produce consistent, informative error responses across the entire API.

---

## Table of Contents
1. [Global vs Local Exception Handling](#global-vs-local-exception-handling)
2. [Error Response Standards](#error-response-standards)
3. [Interview Questions](#interview-questions)

---

## Global vs Local Exception Handling

| Approach | Scope | Annotation |
|---------|-------|-----------|
| Global | All controllers | `@ControllerAdvice` / `@RestControllerAdvice` |
| Local | Single controller | `@ExceptionHandler` inside the controller class |
| Specific | Specific controllers | `@ControllerAdvice(assignableTypes = ...)` |

## Error Response Standards

**RFC 7807 Problem Details for HTTP APIs:**
```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request body contains invalid fields",
  "instance": "/api/v1/users",
  "timestamp": "2024-01-15T10:30:00Z",
  "errors": [
    {"field": "email", "message": "must be a valid email address"},
    {"field": "name", "message": "must not be blank"}
  ]
}
```

---

## Interview Questions

### 🟢 Easy

---

**Q1. What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?**

<details><summary>Click to reveal answer</summary>

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`

```java
// @ControllerAdvice — for both MVC (view-based) and REST controllers
// Return values from @ExceptionHandler need @ResponseBody to be JSON
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseBody  // explicit — needed to serialize response to JSON
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }
}

// @RestControllerAdvice — specifically for REST APIs
// @ResponseBody is implicit on all @ExceptionHandler methods
@RestControllerAdvice
public class GlobalRestExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage()); // auto-serialized to JSON
    }
}
```

> 💡 **Interviewer Tip:** In practice, always use `@RestControllerAdvice` for REST APIs. Reserve `@ControllerAdvice` for applications that mix MVC views and REST endpoints and need different error rendering per endpoint type.

</details>

---

**Q2. How does `@ExceptionHandler` work?**

<details><summary>Click to reveal answer</summary>

`@ExceptionHandler` marks a method to handle exceptions thrown by request-handling methods. It can be placed inside a `@Controller` (local) or `@ControllerAdvice` (global).

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handle a single exception type
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {

        return ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }

    // Handle multiple exception types
    @ExceptionHandler({
        IllegalArgumentException.class,
        IllegalStateException.class
    })
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleBadRequest(RuntimeException ex) {
        return ErrorResponse.of(HttpStatus.BAD_REQUEST, ex.getMessage());
    }

    // Handle all unhandled exceptions (catch-all)
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        // Log the full stack trace but return a safe message to the client
        log.error("Unhandled exception", ex);
        return ErrorResponse.of(HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred");
    }
}
```

**Exception handler resolution order:**
1. Most-specific exception type wins
2. If both local (in controller) and global (in advice) handlers exist, **local wins**
3. Among global handlers, most-specific exception class wins

```java
// Local handler takes priority over global for same exception type
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    // This handles ResourceNotFoundException for OrderController only
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleOrderNotFound(ResourceNotFoundException ex) {
        return Map.of("error", "Order not found: " + ex.getMessage());
    }
}
```

</details>

---

**Q3. What is `@ResponseStatus` and when should you use it?**

<details><summary>Click to reveal answer</summary>

`@ResponseStatus` sets the HTTP status code for a response. It can be applied to:
1. Exception classes — default status when that exception is thrown
2. Controller methods — override default 200 OK
3. `@ExceptionHandler` methods — set status for that handler

```java
// 1. On exception class — Spring returns this status whenever the exception is thrown
@ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "Resource not found")
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}

// Drawback: only works if no @ExceptionHandler intercepts it first
// The "reason" appears as response body (plain text), not JSON

// 2. On controller method
@RestController
public class UserController {

    @PostMapping("/users")
    @ResponseStatus(HttpStatus.CREATED) // returns 201 instead of 200
    public User createUser(@RequestBody @Valid CreateUserRequest req) {
        return userService.create(req);
    }

    @DeleteMapping("/users/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT) // returns 204
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}

// 3. On @ExceptionHandler method (preferred over on exception class)
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(404, ex.getMessage());
    }
}
```

🚨 **Common Mistake:** Annotating exception classes with `@ResponseStatus(reason = "...")` in REST APIs. The `reason` attribute writes a plain-text body, not JSON. Use `@ExceptionHandler` with a custom DTO instead.

</details>

---

**Q4. What is a good structure for an error response DTO?**

<details><summary>Click to reveal answer</summary>

```java
// Basic error response DTO
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private String path;
    private Instant timestamp;
    private List<FieldError> errors; // for validation failures

    @Builder
    public ErrorResponse(int status, String error, String message,
                         String path, Instant timestamp, List<FieldError> errors) {
        this.status = status;
        this.error = error;
        this.message = message;
        this.path = path;
        this.timestamp = timestamp != null ? timestamp : Instant.now();
        this.errors = errors;
    }

    public static ErrorResponse of(HttpStatus status, String message) {
        return ErrorResponse.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .message(message)
            .build();
    }

    @Value
    public static class FieldError {
        String field;
        Object rejectedValue;
        String message;
    }
}
```

```java
// RFC 7807 Problem Details (Spring 6 / Boot 3 has built-in support)
public class ProblemDetail {
    private URI type;
    private String title;
    private int status;
    private String detail;
    private URI instance;
    // extensible with additional properties
}

// Spring Boot 3 built-in ProblemDetail
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest req) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setInstance(URI.create(req.getRequestURI()));
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```

```yaml
# Enable RFC 7807 responses for built-in Spring exceptions
spring:
  mvc:
    problemdetails:
      enabled: true
```

</details>

---

### 🟡 Medium

---

**Q5. How do you handle `MethodArgumentNotValidException` (validation errors)?**

<details><summary>Click to reveal answer</summary>

`MethodArgumentNotValidException` is thrown when `@Valid` / `@Validated` fails on `@RequestBody`.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationErrors(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {

        // Extract field-level errors
        List<ErrorResponse.FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(err -> new ErrorResponse.FieldError(
                err.getField(),
                err.getRejectedValue(),
                err.getDefaultMessage()
            ))
            .collect(Collectors.toList());

        // Extract global (class-level) errors
        List<String> globalErrors = ex.getBindingResult()
            .getGlobalErrors()
            .stream()
            .map(err -> err.getObjectName() + ": " + err.getDefaultMessage())
            .collect(Collectors.toList());

        return ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Request body contains " + fieldErrors.size() + " validation error(s)")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .errors(fieldErrors)
            .build();
    }

    // For @RequestParam / @PathVariable validation (@Validated on controller)
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleConstraintViolations(
            ConstraintViolationException ex, HttpServletRequest request) {

        List<ErrorResponse.FieldError> errors = ex.getConstraintViolations()
            .stream()
            .map(v -> new ErrorResponse.FieldError(
                extractFieldName(v.getPropertyPath()),
                v.getInvalidValue(),
                v.getMessage()
            ))
            .collect(Collectors.toList());

        return ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Constraint Violation")
            .message("Request parameters failed validation")
            .path(request.getRequestURI())
            .errors(errors)
            .build();
    }

    // For missing @RequestParam
    @ExceptionHandler(MissingServletRequestParameterException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleMissingParam(
            MissingServletRequestParameterException ex) {
        return ErrorResponse.of(HttpStatus.BAD_REQUEST,
            "Required parameter '" + ex.getParameterName() + "' is missing");
    }

    private String extractFieldName(Path path) {
        String fullPath = path.toString();
        int lastDot = fullPath.lastIndexOf('.');
        return lastDot >= 0 ? fullPath.substring(lastDot + 1) : fullPath;
    }
}
```

```json
// Response for validation failure:
{
  "status": 400,
  "error": "Validation Failed",
  "message": "Request body contains 2 validation error(s)",
  "path": "/api/v1/users",
  "timestamp": "2024-01-15T10:30:00Z",
  "errors": [
    {
      "field": "email",
      "rejectedValue": "not-an-email",
      "message": "must be a valid email address"
    },
    {
      "field": "name",
      "rejectedValue": "",
      "message": "must not be blank"
    }
  ]
}
```

</details>

---

**Q6. What is `ResponseEntityExceptionHandler` and when should you extend it?**

<details><summary>Click to reveal answer</summary>

`ResponseEntityExceptionHandler` is a base class provided by Spring MVC that handles common Spring MVC exceptions and returns `ResponseEntity` responses.

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // Override built-in Spring MVC exception handlers to customize response format

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        List<ErrorResponse.FieldError> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> new ErrorResponse.FieldError(
                fe.getField(), fe.getRejectedValue(), fe.getDefaultMessage()))
            .collect(Collectors.toList());

        ErrorResponse body = ErrorResponse.builder()
            .status(status.value())
            .error("Validation Failed")
            .errors(errors)
            .path(((ServletWebRequest) request).getRequest().getRequestURI())
            .build();

        return ResponseEntity.status(status).body(body);
    }

    @Override
    protected ResponseEntity<Object> handleNoHandlerFoundException(
            NoHandlerFoundException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        ErrorResponse body = ErrorResponse.of(HttpStatus.NOT_FOUND,
            "No handler found for " + ex.getHttpMethod() + " " + ex.getRequestURL());
        return ResponseEntity.status(status).body(body);
    }

    @Override
    protected ResponseEntity<Object> handleHttpMessageNotReadable(
            HttpMessageNotReadableException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        String message = "Malformed JSON request";
        if (ex.getCause() instanceof JsonParseException jpe) {
            message = "JSON parse error: " + jpe.getOriginalMessage();
        } else if (ex.getCause() instanceof InvalidFormatException ife) {
            message = "Invalid value for field '" + ife.getPath().get(0).getFieldName() + "'";
        }

        return ResponseEntity.status(status).body(ErrorResponse.of(HttpStatus.BAD_REQUEST, message));
    }

    // Add your custom handlers too
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }
}
```

**Spring MVC exceptions handled by `ResponseEntityExceptionHandler`:**
- `MethodArgumentNotValidException` → 400
- `NoHandlerFoundException` → 404
- `HttpMessageNotReadableException` → 400
- `HttpRequestMethodNotSupportedException` → 405
- `HttpMediaTypeNotSupportedException` → 415
- `HttpMediaTypeNotAcceptableException` → 406
- `MissingServletRequestParameterException` → 400
- `TypeMismatchException` → 400

</details>

---

**Q7. How do you implement business exception hierarchy?**

<details><summary>Click to reveal answer</summary>

```java
// Base application exception
public abstract class ApplicationException extends RuntimeException {
    private final String errorCode;
    private final HttpStatus httpStatus;

    protected ApplicationException(String errorCode, String message, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    protected ApplicationException(String errorCode, String message, HttpStatus status, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = status;
    }

    public String getErrorCode() { return errorCode; }
    public HttpStatus getHttpStatus() { return httpStatus; }
}

// Specific exceptions
public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resourceType, Object id) {
        super("RESOURCE_NOT_FOUND",
              resourceType + " with id " + id + " not found",
              HttpStatus.NOT_FOUND);
    }
}

public class BusinessRuleException extends ApplicationException {
    public BusinessRuleException(String message) {
        super("BUSINESS_RULE_VIOLATION", message, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}

public class ConflictException extends ApplicationException {
    public ConflictException(String message) {
        super("CONFLICT", message, HttpStatus.CONFLICT);
    }
}

public class ExternalServiceException extends ApplicationException {
    public ExternalServiceException(String service, String message, Throwable cause) {
        super("EXTERNAL_SERVICE_ERROR",
              "External service '" + service + "' failed: " + message,
              HttpStatus.BAD_GATEWAY, cause);
    }
}

// Single handler for all application exceptions
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ApplicationException.class)
    public ResponseEntity<ErrorResponse> handleApplicationException(
            ApplicationException ex, HttpServletRequest request) {

        log.warn("Application exception [{}]: {}", ex.getErrorCode(), ex.getMessage());

        ErrorResponse body = ErrorResponse.builder()
            .status(ex.getHttpStatus().value())
            .error(ex.getHttpStatus().getReasonPhrase())
            .code(ex.getErrorCode())
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .build();

        return ResponseEntity.status(ex.getHttpStatus()).body(body);
    }
}

// Usage in service layer
@Service
public class UserService {

    public User findById(Long id) {
        return repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    public User createUser(CreateUserRequest req) {
        if (repository.existsByEmail(req.email())) {
            throw new ConflictException("Email already registered: " + req.email());
        }
        // ... create user
    }
}
```

✅ **Best Practice:** Build a clear exception hierarchy rooted in a custom base class. This allows a single `@ExceptionHandler(ApplicationException.class)` to handle all business exceptions with consistent formatting.

</details>

---

**Q8. How do you handle 404 errors for unmapped URLs in Spring Boot?**

<details><summary>Click to reveal answer</summary>

By default, Spring Boot forwards unmapped requests to `/error` rather than throwing `NoHandlerFoundException`. To throw it:

```yaml
# application.yml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
  web:
    resources:
      add-mappings: false   # disable static resource handler that catches all requests
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @Override
    protected ResponseEntity<Object> handleNoHandlerFoundException(
            NoHandlerFoundException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        String path = ex.getRequestURL();
        String method = ex.getHttpMethod();

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .error("Endpoint Not Found")
            .message("No endpoint available for " + method + " " + path)
            .path(path)
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }

    // Handle method not allowed (wrong HTTP verb)
    @Override
    protected ResponseEntity<Object> handleHttpRequestMethodNotSupported(
            HttpRequestMethodNotSupportedException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        String supported = ex.getSupportedMethods() != null
            ? String.join(", ", ex.getSupportedMethods())
            : "unknown";

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.METHOD_NOT_ALLOWED.value())
            .error("Method Not Allowed")
            .message("HTTP method " + ex.getMethod() +
                     " is not supported. Supported: " + supported)
            .build();

        return ResponseEntity.status(HttpStatus.METHOD_NOT_ALLOWED)
            .header(HttpHeaders.ALLOW, supported)
            .body(body);
    }
}
```

</details>

---

**Q9. How do you scope `@ControllerAdvice` to specific controllers?**

<details><summary>Click to reveal answer</summary>

```java
// Target by annotation
@ControllerAdvice(annotations = RestController.class)
public class RestExceptionHandler {
    // Handles exceptions from @RestController classes only
}

// Target by base package
@ControllerAdvice(basePackages = "com.example.api.v1")
public class V1ExceptionHandler {
    // Handles only controllers in com.example.api.v1
}

// Target by type (specific controllers)
@ControllerAdvice(assignableTypes = {
    UserController.class,
    OrderController.class
})
public class UserOrderExceptionHandler {
    // Handles exceptions from UserController and OrderController only
}

// Multiple criteria (OR logic)
@ControllerAdvice(
    basePackages = "com.example.api",
    annotations = {RestController.class}
)
public class ApiExceptionHandler { }

// Ordering when multiple advisors exist
@ControllerAdvice
@Order(1)  // lower number = higher priority
public class SpecificExceptionHandler {
    @ExceptionHandler(SpecificException.class)
    public ResponseEntity<ErrorResponse> handle(SpecificException ex) { /* ... */ return null; }
}

@ControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)  // catch-all
public class GenericExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handle(Exception ex) { /* ... */ return null; }
}
```

> 💡 **Interviewer Tip:** Use `@Order` to control which advice takes precedence when multiple `@ControllerAdvice` beans handle the same exception type. `@Order(1)` fires before `@Order(2)`.

</details>

---

**Q10. How do you handle exceptions thrown during `@Async` method execution?**

<details><summary>Click to reveal answer</summary>

`@ControllerAdvice` does **not** catch exceptions from `@Async` methods because they run in a separate thread. You need an `AsyncUncaughtExceptionHandler`.

```java
@Service
public class EmailService {

    @Async
    public void sendWelcomeEmail(String email) {
        // If this throws, @ControllerAdvice cannot catch it
        emailClient.send(email, "Welcome!", "Welcome to our platform!");
    }

    @Async
    public CompletableFuture<String> generateReportAsync(Long userId) {
        // CompletableFuture exceptions CAN be handled at the call site
        try {
            String report = reportService.generate(userId);
            return CompletableFuture.completedFuture(report);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}

// Configure async exception handler
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomAsyncExceptionHandler.class);

    @Override
    public void handleUncaughtException(
            Throwable ex, Method method, Object... params) {
        log.error("Async method [{}] threw exception: {}",
            method.getName(), ex.getMessage(), ex);

        // Send alert, store in dead-letter queue, etc.
        alertService.sendAlert("Async failure in " + method.getName(), ex);
    }
}

// Handling CompletableFuture exceptions at call site
@RestController
public class ReportController {

    @GetMapping("/api/v1/reports/{userId}")
    public CompletableFuture<ResponseEntity<String>> getReport(
            @PathVariable Long userId) {
        return emailService.generateReportAsync(userId)
            .thenApply(report -> ResponseEntity.ok(report))
            .exceptionally(ex -> {
                log.error("Report generation failed", ex);
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("Report generation failed: " + ex.getMessage());
            });
    }
}
```

</details>

---

### 🔴 Hard

---

**Q11. How do you implement RFC 7807 Problem Details with Spring Boot 3?**

<details><summary>Click to reveal answer</summary>

Spring Boot 3 / Spring Framework 6 has native RFC 7807 support via `ProblemDetail`.

```java
// Enable built-in problem details for Spring MVC exceptions
```
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

```java
// Custom global exception handler using ProblemDetail
@RestControllerAdvice
public class ProblemDetailExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleResourceNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        problem.setTitle("Resource Not Found");
        problem.setDetail(ex.getMessage());
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setType(URI.create("https://api.example.com/errors/not-found"));

        // Extension properties
        problem.setProperty("timestamp", Instant.now());
        problem.setProperty("errorCode", ex.getErrorCode());

        return problem;
    }

    @ExceptionHandler(BusinessRuleException.class)
    public ResponseEntity<ProblemDetail> handleBusinessRule(
            BusinessRuleException ex, HttpServletRequest request) {

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        problem.setTitle("Business Rule Violation");
        problem.setDetail(ex.getMessage());
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("errorCode", ex.getErrorCode());

        return ResponseEntity.unprocessableEntity()
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem);
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        ProblemDetail problem = ex.updateAndGetBody(
            getMessageSource(), getLocale(request));
        problem.setTitle("Validation Failed");

        List<Map<String, Object>> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of(
                "field", (Object) fe.getField(),
                "message", fe.getDefaultMessage(),
                "rejectedValue", fe.getRejectedValue() != null
                    ? fe.getRejectedValue() : "null"
            ))
            .collect(Collectors.toList());

        problem.setProperty("errors", errors);
        return ResponseEntity.badRequest()
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem);
    }
}
```

```json
// Response: Content-Type: application/problem+json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with id 42 not found",
  "instance": "/api/v1/users/42",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "errorCode": "RESOURCE_NOT_FOUND"
}
```

</details>

---

**Q12. How do you handle exceptions from Spring Security filters in your global handler?**

<details><summary>Click to reveal answer</summary>

Spring Security filters run **before** the `DispatcherServlet`, so `@ControllerAdvice` cannot intercept them. You need `AuthenticationEntryPoint` and `AccessDeniedHandler`.

```java
// Custom entry point for 401 Unauthorized
@Component
public class JwtAuthEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    public JwtAuthEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        ErrorResponse error = ErrorResponse.builder()
            .status(401)
            .error("Unauthorized")
            .message("Authentication is required: " + authException.getMessage())
            .path(request.getRequestURI())
            .build();

        objectMapper.writeValue(response.getOutputStream(), error);
    }
}

// Custom handler for 403 Forbidden
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    public CustomAccessDeniedHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) throws IOException {

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);

        ErrorResponse error = ErrorResponse.builder()
            .status(403)
            .error("Forbidden")
            .message("You do not have permission to access this resource")
            .path(request.getRequestURI())
            .build();

        objectMapper.writeValue(response.getOutputStream(), error);
    }
}

// Wire them into SecurityFilterChain
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired private JwtAuthEntryPoint authEntryPoint;
    @Autowired private CustomAccessDeniedHandler accessDeniedHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)    // 401
                .accessDeniedHandler(accessDeniedHandler)    // 403
            )
            // ... other config
            .build();
    }
}
```

</details>

---

**Q13. How do you write tests for `@ControllerAdvice` exception handlers?**

<details><summary>Click to reveal answer</summary>

```java
// Unit test using MockMvc
@WebMvcTest(UserController.class)
class UserControllerExceptionHandlingTest {

    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;

    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.findById(99L))
            .thenThrow(new ResourceNotFoundException("User", 99L));

        mockMvc.perform(get("/api/v1/users/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.status").value(404))
            .andExpect(jsonPath("$.error").value("Not Found"))
            .andExpect(jsonPath("$.message").value("User with id 99 not found"))
            .andExpect(jsonPath("$.path").value("/api/v1/users/99"))
            .andExpect(jsonPath("$.timestamp").exists());
    }

    @Test
    void shouldReturn400ForValidationErrors() throws Exception {
        String invalidBody = """
            {
              "email": "not-an-email",
              "name": ""
            }
            """;

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidBody))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.status").value(400))
            .andExpect(jsonPath("$.errors").isArray())
            .andExpect(jsonPath("$.errors", hasSize(2)))
            .andExpect(jsonPath("$.errors[*].field",
                containsInAnyOrder("email", "name")));
    }

    @Test
    void shouldReturn405ForWrongMethod() throws Exception {
        mockMvc.perform(delete("/api/v1/users")) // DELETE on collection not supported
            .andExpect(status().isMethodNotAllowed())
            .andExpect(jsonPath("$.status").value(405));
    }
}

// Integration test
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ExceptionHandlingIntegrationTest {

    @Autowired TestRestTemplate restTemplate;

    @Test
    void globalHandlerResolvesNotFound() {
        ResponseEntity<ErrorResponse> response =
            restTemplate.getForEntity("/api/v1/users/99999", ErrorResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
        assertThat(response.getBody().getStatus()).isEqualTo(404);
        assertThat(response.getBody().getMessage()).contains("99999");
    }
}
```

</details>

---

**Q14. How do you handle database constraint violation exceptions?**

<details><summary>Click to reveal answer</summary>

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Spring Data wraps most DB exceptions in DataAccessException subclasses
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrityViolation(
            DataIntegrityViolationException ex, HttpServletRequest request) {

        String message = "Data integrity violation";
        String code = "DATA_INTEGRITY_VIOLATION";

        // Inspect the root cause for more detail
        Throwable rootCause = ex.getRootCause();
        if (rootCause instanceof org.hibernate.exception.ConstraintViolationException cve) {
            String constraintName = cve.getConstraintName();
            message = mapConstraintToMessage(constraintName);
            code = "CONSTRAINT_VIOLATION";
        }

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.CONFLICT.value())
            .error("Conflict")
            .code(code)
            .message(message)
            .path(request.getRequestURI())
            .build();

        return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
    }

    private String mapConstraintToMessage(String constraintName) {
        if (constraintName == null) return "Duplicate or invalid data";
        return switch (constraintName.toLowerCase()) {
            case "users_email_key"  -> "Email address is already registered";
            case "users_phone_key"  -> "Phone number is already in use";
            case "orders_fk_user"   -> "Referenced user does not exist";
            default -> "Data constraint violation: " + constraintName;
        };
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleOptimisticLock(OptimisticLockingFailureException ex) {
        return ErrorResponse.builder()
            .status(HttpStatus.CONFLICT.value())
            .error("Conflict")
            .code("OPTIMISTIC_LOCK_FAILURE")
            .message("The resource was modified by another request. Please retry.")
            .build();
    }
}
```

🚨 **Common Mistake:** Leaking raw database error messages (containing table/column names, SQL details) to the API client. Always map them to business-friendly messages.

</details>

---

## Quick Reference Summary

| Concept | Key Point |
|---------|-----------|
| `@RestControllerAdvice` | Global handler + `@ResponseBody` implicit |
| `@ExceptionHandler` | Most specific exception type wins; local > global |
| `ResponseEntityExceptionHandler` | Base class for handling Spring MVC built-in exceptions |
| RFC 7807 | `ProblemDetail` — standard error format with `type`, `title`, `status`, `detail` |
| Validation errors | `MethodArgumentNotValidException` → extract `BindingResult.getFieldErrors()` |
| Security exceptions | Use `AuthenticationEntryPoint` (401) and `AccessDeniedHandler` (403) |
| `@ResponseStatus(reason)` | Avoid in REST APIs — returns plain text, not JSON |
| `@Async` exceptions | Use `AsyncUncaughtExceptionHandler` — `@ControllerAdvice` won't catch them |
| DB constraints | Handle `DataIntegrityViolationException`; never leak SQL details to client |
