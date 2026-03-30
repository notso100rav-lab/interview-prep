# Spring Security & JWT

> **Security is non-negotiable in production systems.**  
> This guide covers Spring Security 6.x architecture, JWT implementation end-to-end, and method-level authorization — exactly what senior Java interviews test.

---

## Table of Contents
1. [SecurityFilterChain Configuration](#securityfilterchain-configuration)
2. [Authentication vs Authorization](#authentication-vs-authorization)
3. [UserDetails / UserDetailsService](#userdetails--userdetailsservice)
4. [PasswordEncoder](#passwordencoder)
5. [JWT Deep Dive](#jwt-deep-dive)
6. [JWT Implementation with JJWT](#jwt-implementation-with-jjwt)
7. [JwtAuthenticationFilter](#jwtauthenticationfilter)
8. [SecurityContext and SecurityContextHolder](#securitycontext-and-securitycontextholder)
9. [Method-Level Security](#method-level-security)
10. [CORS Configuration](#cors-configuration)
11. [CSRF](#csrf)
12. [Refresh Token Pattern](#refresh-token-pattern)
13. [Full Implementation Reference](#full-implementation-reference)
14. [Q&A](#qa)

---

## SecurityFilterChain Configuration

Spring Security 6.x uses the **lambda DSL** with `SecurityFilterChain` beans. The old `WebSecurityConfigurerAdapter` is removed.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                          // Stateless JWT — no CSRF
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()        // Public endpoints
                .requestMatchers("/api/admin/**").hasRole("ADMIN")  // Role-based
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .anyRequest().authenticated()                       // Everything else: auth required
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Filter Order

Spring Security installs a chain of filters. JWT filter is inserted **before** `UsernamePasswordAuthenticationFilter`:

```
Request → DelegatingFilterProxy → FilterChainProxy → [
    DisableEncodeUrlFilter,
    SecurityContextHolderFilter,
    HeaderWriterFilter,
    CorsFilter,
    CsrfFilter,
    ... (logout, oauth2 etc.) ...
    JwtAuthenticationFilter,        ← our custom filter
    UsernamePasswordAuthenticationFilter,
    ExceptionTranslationFilter,
    AuthorizationFilter
] → Controller
```

---

## Authentication vs Authorization

| Concept | Question answered | Spring component |
|---|---|---|
| **Authentication** | *Who are you?* | `AuthenticationManager`, `AuthenticationProvider` |
| **Authorization** | *What are you allowed to do?* | `AuthorizationManager`, `@PreAuthorize` |

```
Request
  ↓
[Authentication] → Validate credentials → Set SecurityContext
  ↓
[Authorization] → Check roles/permissions → Allow or 403/401
  ↓
Controller method executes
```

---

## UserDetails / UserDetailsService

### `UserDetails` Implementation

```java
@Entity
@Table(name = "app_user")
public class AppUser implements UserDetails {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    private Role role; // enum: USER, ADMIN, MODERATOR

    private boolean enabled = true;
    private boolean accountNonLocked = true;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }

    @Override public String getPassword()   { return passwordHash; }
    @Override public String getUsername()   { return email; }
    @Override public boolean isEnabled()    { return enabled; }
    @Override public boolean isAccountNonExpired()    { return true; }
    @Override public boolean isAccountNonLocked()     { return accountNonLocked; }
    @Override public boolean isCredentialsNonExpired() { return true; }
}
```

### `UserDetailsService` Implementation

```java
@Service
@RequiredArgsConstructor
public class AppUserDetailsService implements UserDetailsService {

    private final AppUserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
    }
}
```

---

## PasswordEncoder

Always hash passwords before storage. Never store plain text.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // cost factor: 4-31, default 10
}
```

### Usage

```java
@Service
public class UserService {

    private final AppUserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public AppUser register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new EmailAlreadyExistsException(request.email());
        }

        AppUser user = new AppUser();
        user.setEmail(request.email());
        user.setPasswordHash(passwordEncoder.encode(request.password())); // Hash here
        user.setRole(Role.USER);

        return userRepository.save(user);
    }

    public boolean verifyPassword(String rawPassword, String storedHash) {
        return passwordEncoder.matches(rawPassword, storedHash); // BCrypt compare
    }
}
```

### Other PasswordEncoder Options

```java
// Delegating encoder — supports multiple algorithms, upgrades old hashes
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // Stores as: {bcrypt}$2a$10$...
}

// Argon2 (memory-hard, stronger than BCrypt)
new Argon2PasswordEncoder(16, 32, 1, 65536, 10);

// PBKDF2
new Pbkdf2PasswordEncoder("secret", 16, 310000, Pbkdf2PasswordEncoder.SecretKeyFactoryAlgorithm.PBKDF2WithHmacSHA256);
```

---

## JWT Deep Dive

### Structure

A JWT consists of three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header
.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwiZXhwIjoxNzA5MDAwMDAwfQ  ← Payload
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

### Header (decoded)

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload / Claims (decoded)

```json
{
  "sub": "user@example.com",
  "iat": 1709000000,
  "exp": 1709003600,
  "roles": ["ROLE_USER"],
  "userId": 42
}
```

| Claim | Name | Description |
|---|---|---|
| `sub` | Subject | Identifies the principal (e.g., user email or ID) |
| `iat` | Issued At | Unix timestamp of token issuance |
| `exp` | Expiration | Unix timestamp when token expires |
| `nbf` | Not Before | Token not valid before this time |
| `iss` | Issuer | Who issued the token |
| `aud` | Audience | Intended recipient |
| `jti` | JWT ID | Unique token identifier (for revocation) |

### Signature

```
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  secretKey
)
```

The signature **cannot be faked** without the secret key, but the payload is **not encrypted** — only encoded. Never put sensitive data in the payload.

---

## JWT Implementation with JJWT

**Dependencies:**
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

### `JwtUtil` Class

```java
@Component
public class JwtUtil {

    @Value("${app.jwt.secret}")
    private String secretKey;

    @Value("${app.jwt.expiration-ms:3600000}")       // 1 hour default
    private long accessTokenExpirationMs;

    @Value("${app.jwt.refresh-expiration-ms:604800000}") // 7 days default
    private long refreshTokenExpirationMs;

    // ─── Key derivation ─────────────────────────────────────────────────────

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // ─── Token generation ────────────────────────────────────────────────────

    public String generateAccessToken(UserDetails userDetails) {
        Map<String, Object> extraClaims = new HashMap<>();
        extraClaims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList());
        return buildToken(extraClaims, userDetails, accessTokenExpirationMs);
    }

    public String generateRefreshToken(UserDetails userDetails) {
        return buildToken(new HashMap<>(), userDetails, refreshTokenExpirationMs);
    }

    private String buildToken(Map<String, Object> extraClaims,
                               UserDetails userDetails,
                               long expirationMs) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date(System.currentTimeMillis()))
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    // ─── Token parsing ───────────────────────────────────────────────────────

    public Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public String extractUsername(String token) {
        return extractAllClaims(token).getSubject();
    }

    public Date extractExpiration(String token) {
        return extractAllClaims(token).getExpiration();
    }

    // ─── Validation ──────────────────────────────────────────────────────────

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}
```

### `application.yml`

```yaml
app:
  jwt:
    secret: dGhpcy1pcy1hLXZlcnktc2VjdXJlLXNlY3JldC1rZXktZm9yLWRlbW8tb25seQ==
    expiration-ms: 3600000       # 1 hour
    refresh-expiration-ms: 604800000  # 7 days
```

> **Secret key requirement:** For HS256, the secret must be at least 256 bits (32 bytes). Generate securely:
> ```bash
> openssl rand -base64 32
> ```

---

## JwtAuthenticationFilter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        final String authHeader = request.getHeader(AUTHORIZATION_HEADER);

        // 1. Skip filter if no Bearer token present
        if (authHeader == null || !authHeader.startsWith(BEARER_PREFIX)) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(BEARER_PREFIX.length());

        try {
            final String userEmail = jwtUtil.extractUsername(jwt);

            // 2. Only authenticate if not already authenticated
            if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);

                // 3. Validate token
                if (jwtUtil.isTokenValid(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                    // 4. Set authentication in SecurityContext
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (ExpiredJwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Token expired\"}");
            return;
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Invalid token\"}");
            return;
        }

        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getServletPath();
        return path.startsWith("/api/auth/"); // Skip JWT check for auth endpoints
    }
}
```

---

## SecurityContext and SecurityContextHolder

The `SecurityContext` stores the current authenticated principal for the duration of a request.

```java
// Reading the current user anywhere in the application
public AppUser getCurrentUser() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

    if (authentication == null || !authentication.isAuthenticated()) {
        throw new UnauthorizedException("No authenticated user");
    }

    return (AppUser) authentication.getPrincipal();
}

// In a controller — using @AuthenticationPrincipal
@GetMapping("/profile")
public ResponseEntity<ProfileDTO> getProfile(
        @AuthenticationPrincipal AppUser currentUser) {
    return ResponseEntity.ok(profileService.getProfile(currentUser.getId()));
}
```

### Storage Strategy

By default, `SecurityContextHolder` uses `ThreadLocal` storage — the security context is tied to the current request thread.

```java
// Available strategies:
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_THREADLOCAL);  // default
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL); // for child threads
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_GLOBAL); // rare: shared across all threads
```

---

## Method-Level Security

Enabled via `@EnableMethodSecurity` (replaces `@EnableGlobalMethodSecurity` in Spring Security 6):

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,  // @PreAuthorize, @PostAuthorize (default: true)
    securedEnabled = true,  // @Secured
    jsr250Enabled = true    // @RolesAllowed
)
public class SecurityConfig { ... }
```

### `@PreAuthorize`

Evaluated **before** method execution. Most powerful — supports SpEL:

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // Simple role check
    @GetMapping("/admin/all")
    @PreAuthorize("hasRole('ADMIN')")
    public List<Order> getAllOrders() { ... }

    // Multiple roles
    @DeleteMapping("/{id}")
    @PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
    public void deleteOrder(@PathVariable Long id) { ... }

    // Method parameter in SpEL
    @GetMapping("/{userId}/orders")
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public List<Order> getUserOrders(@PathVariable Long userId) { ... }

    // Authority check (without ROLE_ prefix)
    @PutMapping("/{id}/approve")
    @PreAuthorize("hasAuthority('ORDER_APPROVE')")
    public Order approveOrder(@PathVariable Long id) { ... }

    // Bean method call in SpEL
    @PostMapping
    @PreAuthorize("@orderSecurityService.canCreateOrder(authentication, #request)")
    public Order createOrder(@RequestBody OrderRequest request) { ... }
}
```

### `@PostAuthorize`

Evaluated **after** method execution — can check the return value:

```java
// Only return the order if it belongs to the current user or user is ADMIN
@GetMapping("/{id}")
@PostAuthorize("returnObject.userId == authentication.principal.id or hasRole('ADMIN')")
public Order getOrder(@PathVariable Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

### `@Secured`

Simpler but less powerful — no SpEL:

```java
@Secured({"ROLE_ADMIN", "ROLE_MODERATOR"})
public void moderateContent(Long contentId) { ... }
```

### `@RolesAllowed` (JSR-250)

Similar to `@Secured`, from the `jakarta.annotation` package:

```java
@RolesAllowed("ADMIN")
public void adminOperation() { ... }
```

### Custom Security Expression

```java
@Component("orderSecurityService")
public class OrderSecurityService {

    public boolean canCreateOrder(Authentication auth, OrderRequest request) {
        AppUser user = (AppUser) auth.getPrincipal();
        // Custom business logic: e.g., check subscription, fraud signals
        return user.isSubscriptionActive() && !user.isFlaggedForFraud();
    }
}
```

---

## CORS Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();

        configuration.setAllowedOrigins(List.of(
            "https://myapp.com",
            "https://www.myapp.com"
        ));
        // OR use patterns for dev:
        // configuration.setAllowedOriginPatterns(List.of("http://localhost:*"));

        configuration.setAllowedMethods(List.of(
            "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
        ));

        configuration.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Requested-With",
            "Accept"
        ));

        configuration.setExposedHeaders(List.of("Authorization")); // Headers client can read
        configuration.setAllowCredentials(true);    // Allow cookies/auth headers
        configuration.setMaxAge(3600L);             // Preflight cache duration (seconds)

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

Reference in `SecurityFilterChain`:

```java
.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

---

## CSRF

**Why disable CSRF for JWT APIs:**

CSRF (Cross-Site Request Forgery) attacks work by exploiting browser-based session cookies — the browser automatically sends cookies on cross-origin requests. JWT tokens stored in `Authorization` headers are **not** sent automatically by browsers, so CSRF is not a concern for pure JWT-based APIs.

```java
// Stateless JWT API — disable CSRF
http.csrf(csrf -> csrf.disable());

// If you have a mix (e.g., form login + JWT), disable only for API paths:
http.csrf(csrf -> csrf
    .ignoringRequestMatchers("/api/**")
);
```

**Keep CSRF enabled if:**
- You use session-based authentication (Spring MVC with form login)
- You store JWT in a cookie (`httpOnly` cookie — then CSRF matters again)

---

## Refresh Token Pattern

Short-lived access tokens + long-lived refresh tokens prevent indefinite exposure if an access token is stolen.

```
Access Token:  15 min - 1 hour lifetime
Refresh Token: 7 - 30 days lifetime (stored securely, revocable)
```

### Refresh Token Entity

```java
@Entity
public class RefreshToken {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String token; // UUID or opaque random string

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private AppUser user;

    @Column(nullable = false)
    private Instant expiresAt;

    private boolean revoked = false;

    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }
}
```

### Refresh Token Service

```java
@Service
@Transactional
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final AppUserRepository userRepository;
    private final JwtUtil jwtUtil;

    @Value("${app.jwt.refresh-expiration-ms:604800000}")
    private long refreshExpirationMs;

    public RefreshToken createRefreshToken(String email) {
        AppUser user = userRepository.findByEmail(email).orElseThrow();

        // Revoke existing tokens for this user (optional: single-session policy)
        refreshTokenRepository.revokeAllByUser(user);

        RefreshToken token = new RefreshToken();
        token.setToken(UUID.randomUUID().toString());
        token.setUser(user);
        token.setExpiresAt(Instant.now().plusMillis(refreshExpirationMs));
        token.setRevoked(false);

        return refreshTokenRepository.save(token);
    }

    public TokenPair refreshAccessToken(String refreshTokenValue) {
        RefreshToken refreshToken = refreshTokenRepository
            .findByTokenAndRevokedFalse(refreshTokenValue)
            .orElseThrow(() -> new InvalidTokenException("Refresh token not found or revoked"));

        if (refreshToken.isExpired()) {
            refreshToken.setRevoked(true);
            throw new InvalidTokenException("Refresh token expired");
        }

        AppUser user = refreshToken.getUser();
        String newAccessToken = jwtUtil.generateAccessToken(user);

        // Rotate refresh token (optional but recommended)
        refreshToken.setRevoked(true);
        RefreshToken newRefreshToken = createRefreshToken(user.getEmail());

        return new TokenPair(newAccessToken, newRefreshToken.getToken());
    }

    public void revokeRefreshToken(String token) {
        refreshTokenRepository.findByToken(token)
            .ifPresent(rt -> rt.setRevoked(true));
    }
}
```

### Auth Controller

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final RefreshTokenService refreshTokenService;
    private final UserService userService;

    @PostMapping("/register")
    public ResponseEntity<MessageResponse> register(@Valid @RequestBody RegisterRequest request) {
        userService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new MessageResponse("User registered successfully"));
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password())
        );

        AppUser user = (AppUser) userDetailsService.loadUserByUsername(request.email());
        String accessToken = jwtUtil.generateAccessToken(user);
        RefreshToken refreshToken = refreshTokenService.createRefreshToken(request.email());

        return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken.getToken()));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@Valid @RequestBody RefreshRequest request) {
        TokenPair tokens = refreshTokenService.refreshAccessToken(request.refreshToken());
        return ResponseEntity.ok(new AuthResponse(tokens.accessToken(), tokens.refreshToken()));
    }

    @PostMapping("/logout")
    public ResponseEntity<MessageResponse> logout(@Valid @RequestBody RefreshRequest request) {
        refreshTokenService.revokeRefreshToken(request.refreshToken());
        SecurityContextHolder.clearContext();
        return ResponseEntity.ok(new MessageResponse("Logged out successfully"));
    }
}
```

---

## Full Implementation Reference

### Request/Response DTOs

```java
public record RegisterRequest(
    @Email @NotBlank String email,
    @NotBlank @Size(min = 8) String password,
    @NotBlank String firstName,
    @NotBlank String lastName
) {}

public record LoginRequest(
    @Email @NotBlank String email,
    @NotBlank String password
) {}

public record RefreshRequest(@NotBlank String refreshToken) {}

public record AuthResponse(String accessToken, String refreshToken) {}

public record MessageResponse(String message) {}
```

### Exception Handling for Security

```java
@Component
public class CustomAuthEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().write("""
            {"error": "Unauthorized", "message": "%s"}
            """.formatted(authException.getMessage()));
    }
}

@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.getWriter().write("""
            {"error": "Forbidden", "message": "Insufficient privileges"}
            """);
    }
}
```

Reference in `SecurityFilterChain`:

```java
.exceptionHandling(ex -> ex
    .authenticationEntryPoint(customAuthEntryPoint)
    .accessDeniedHandler(customAccessDeniedHandler)
)
```

---

## Q&A

---

🟢 **Q1: What is the difference between authentication and authorization?**

<details><summary>Click to reveal answer</summary>

- **Authentication** answers: *"Who are you?"* — verifying the identity of the caller (username/password, token, certificate).
- **Authorization** answers: *"What can you do?"* — determining whether the authenticated user has permission to perform an action.

In Spring Security:
- Authentication is handled by `AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService`.
- Authorization is handled by `AuthorizationManager`, `@PreAuthorize`, and `SecurityFilterChain` request matchers.

An unauthenticated request returns **401 Unauthorized**.  
An authenticated-but-not-authorized request returns **403 Forbidden**.

</details>

---

🟢 **Q2: What is `UserDetails` and why does your `User` entity implement it?**

<details><summary>Click to reveal answer</summary>

`UserDetails` is a Spring Security interface representing the authenticated principal. It provides:

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

Your `User` entity implements `UserDetails` so Spring Security can directly use it as the principal after authentication, eliminating a separate DTO adapter class. The `getAuthorities()` method returns the user's roles/permissions as `GrantedAuthority` objects.

**Alternatively**, you can use a separate `UserDetailsAdapter` class to avoid coupling domain entities to the security framework.

</details>

---

🟢 **Q3: What is `BCryptPasswordEncoder` and what is the cost factor?**

<details><summary>Click to reveal answer</summary>

`BCryptPasswordEncoder` uses the BCrypt adaptive hashing algorithm. It automatically:
- Generates a random salt per password.
- Applies the algorithm `2^strength` times (cost factor).
- Embeds the salt in the resulting hash string.

```java
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
// strength: 4–31. Higher = slower hashing = more resistant to brute force.
// Default is 10. Use 12 for new apps (benchmarks to ~250ms on modern hardware).

String hash = encoder.encode("myPassword");
// "$2a$12$randomSaltHere.hashedPasswordHere"

boolean matches = encoder.matches("myPassword", hash); // true
```

**Why it's adaptive:** As CPUs get faster, you increase the cost factor and re-hash stored passwords on next login — no migration needed.

</details>

---

🟡 **Q4: Explain the structure of a JWT token.**

<details><summary>Click to reveal answer</summary>

A JWT has three Base64URL-encoded parts separated by dots: `header.payload.signature`.

**Header:**
```json
{"alg": "HS256", "typ": "JWT"}
```

**Payload (claims):**
```json
{
  "sub": "user@example.com",
  "iat": 1709000000,
  "exp": 1709003600,
  "roles": ["ROLE_USER"]
}
```

**Signature:**
```
HMACSHA256(base64url(header) + "." + base64url(payload), secretKey)
```

**Key facts:**
- The payload is **encoded, not encrypted** — anyone can decode it. Never put passwords or PII in JWT.
- The signature ensures **integrity** — if the payload is tampered with, the signature won't match.
- The server validates the signature without a database lookup — that's what makes JWT stateless.

</details>

---

🟡 **Q5: What does `OncePerRequestFilter` guarantee?**

<details><summary>Click to reveal answer</summary>

`OncePerRequestFilter` ensures that `doFilterInternal` is called **exactly once per request**, even in servlet environments where a request might be dispatched multiple times (e.g., via `RequestDispatcher.forward()` or error dispatches).

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        // Guaranteed to run once per HTTP request
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // Return true to completely skip this filter for certain paths
        return request.getServletPath().startsWith("/api/auth/");
    }
}
```

Without this guarantee, a `Filter` implementing `javax.servlet.Filter` could be called twice for error-dispatched requests, causing duplicate security context population or duplicate logging.

</details>

---

🟡 **Q6: How do you set the SecurityContext after validating a JWT?**

<details><summary>Click to reveal answer</summary>

After validating the JWT and loading the `UserDetails`, create a `UsernamePasswordAuthenticationToken` and set it on the `SecurityContextHolder`:

```java
UsernamePasswordAuthenticationToken authToken =
    new UsernamePasswordAuthenticationToken(
        userDetails,           // principal
        null,                  // credentials (null = already authenticated)
        userDetails.getAuthorities()  // roles
    );

// Attach request details (IP, session ID) — optional but useful for auditing
authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

// Store in SecurityContext for this request
SecurityContextHolder.getContext().setAuthentication(authToken);
```

After this line, any downstream code (filters, controllers, service methods) can access the current user via `SecurityContextHolder.getContext().getAuthentication()`.

</details>

---

🟡 **Q7: Why do we disable CSRF for JWT REST APIs?**

<details><summary>Click to reveal answer</summary>

CSRF attacks work because browsers automatically include cookies on cross-origin requests. A malicious site can trigger requests to your API and the browser sends the session cookie without the user's knowledge.

JWT tokens stored in the `Authorization: Bearer <token>` header are **not** automatically sent by browsers — they require explicit JavaScript code to attach. Therefore:

- A malicious cross-origin site cannot trigger an authenticated request to your API.
- CSRF protection is unnecessary and adds overhead.

```java
http.csrf(csrf -> csrf.disable()); // Safe for Bearer token APIs
```

**Exception:** If you store JWTs in `httpOnly` cookies (to prevent XSS access), CSRF protection becomes relevant again because the browser will auto-send the cookie.

</details>

---

🔴 **Q8: What is the refresh token rotation pattern and why is it important?**

<details><summary>Click to reveal answer</summary>

Refresh token rotation means issuing a **new refresh token every time** the current one is used, and immediately revoking the old one.

**Why it matters:** If a refresh token is stolen, the attacker and the legitimate user will both try to use it. With rotation:

1. Legitimate user refreshes → gets new access + refresh token, old refresh token revoked.
2. Attacker tries to use the (now-revoked) old token → **rejected**.

```java
public TokenPair refreshAccessToken(String refreshTokenValue) {
    RefreshToken token = repository.findByTokenAndRevokedFalse(refreshTokenValue)
        .orElseThrow(() -> new InvalidTokenException("Revoked or invalid"));

    if (token.isExpired()) {
        token.setRevoked(true);
        throw new InvalidTokenException("Expired");
    }

    // Revoke used token (rotation)
    token.setRevoked(true);

    // Issue new tokens
    String newAccessToken = jwtUtil.generateAccessToken(token.getUser());
    RefreshToken newRefresh = createRefreshToken(token.getUser().getEmail());

    return new TokenPair(newAccessToken, newRefresh.getToken());
}
```

**Reuse detection:** If an already-revoked refresh token is presented, it likely indicates token theft — revoke all tokens for that user immediately.

</details>

---

🟡 **Q9: What is the difference between `hasRole('ADMIN')` and `hasAuthority('ROLE_ADMIN')`?**

<details><summary>Click to reveal answer</summary>

They are equivalent in effect, but `hasRole` automatically prepends the `ROLE_` prefix:

```java
// These are equivalent:
@PreAuthorize("hasRole('ADMIN')")           // Checks for ROLE_ADMIN
@PreAuthorize("hasAuthority('ROLE_ADMIN')") // Checks for ROLE_ADMIN literally

// Fine-grained authorities (not roles) — use hasAuthority:
@PreAuthorize("hasAuthority('ORDER_APPROVE')")  // Checks for ORDER_APPROVE (no prefix)
```

When you add authorities to `UserDetails`:
```java
// Role-based:
return List.of(new SimpleGrantedAuthority("ROLE_ADMIN")); // hasRole('ADMIN') works

// Permission-based:
return List.of(
    new SimpleGrantedAuthority("ROLE_USER"),
    new SimpleGrantedAuthority("ORDER_APPROVE"),  // hasAuthority('ORDER_APPROVE')
    new SimpleGrantedAuthority("REPORT_VIEW")
);
```

</details>

---

🟡 **Q10: What happens if the JWT secret key is too short?**

<details><summary>Click to reveal answer</summary>

For HS256, the secret must be **at least 256 bits (32 bytes)**. JJWT 0.12.x enforces this:

```java
// Too short — throws WeakKeyException at runtime:
Keys.hmacShaKeyFor("short-secret".getBytes()); // ❌

// Correct — at least 32 bytes:
Keys.hmacShaKeyFor(Decoders.BASE64.decode("dGhpcy1pcy1hLXZlcnktc2VjdXJlLXNlY3JldA==")); // ✅
```

**Generate a proper key:**
```bash
openssl rand -base64 32
```

**Store in environment variable, not in source code:**
```yaml
app:
  jwt:
    secret: ${JWT_SECRET}  # Injected from environment
```

Using a weak secret allows attackers to brute-force the signing key and forge arbitrary tokens.

</details>

---

🔴 **Q11: How would you implement JWT token revocation (logout/blacklisting)?**

<details><summary>Click to reveal answer</summary>

JWTs are stateless — they're valid until expiry by design. To support revocation, you need server-side state:

**Option 1: Refresh token revocation (recommended)**
- Short access token TTL (15 min).
- Logout only revokes the refresh token.
- Access tokens remain valid until expiry (acceptable window).

**Option 2: Token blacklist in Redis**

```java
@Service
public class TokenBlacklistService {

    private final StringRedisTemplate redisTemplate;
    private final JwtUtil jwtUtil;

    public void blacklist(String token) {
        Date expiry = jwtUtil.extractExpiration(token);
        long ttlSeconds = (expiry.getTime() - System.currentTimeMillis()) / 1000;
        if (ttlSeconds > 0) {
            redisTemplate.opsForValue()
                .set("blacklist:" + token, "revoked", Duration.ofSeconds(ttlSeconds));
        }
    }

    public boolean isBlacklisted(String token) {
        return Boolean.TRUE.equals(redisTemplate.hasKey("blacklist:" + token));
    }
}
```

Check in `JwtAuthenticationFilter`:
```java
if (jwtUtil.isTokenValid(jwt, userDetails) && !blacklistService.isBlacklisted(jwt)) {
    // Set security context
}
```

**Trade-off:** Redis lookup per request adds latency. Use short-lived access tokens to minimize the blacklist size.

**Option 3: Token versioning**

Add a `version` field to the user record. Include it in the JWT. On logout, increment the version — all existing tokens with old version are rejected.

</details>

---

🟡 **Q12: How does `@PostAuthorize` differ from `@PreAuthorize`?**

<details><summary>Click to reveal answer</summary>

- **`@PreAuthorize`**: Evaluated **before** the method executes. If it fails, the method is never called. Use for access control based on method arguments or the current user.

- **`@PostAuthorize`**: Evaluated **after** the method returns. Can check `returnObject`. If it fails, a 403 is returned but the method **has already executed** (including side effects like database writes).

```java
// Safe: check before execution
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public List<Order> getOrders(Long userId) { ... }

// Useful for ownership check on the returned entity
@PostAuthorize("returnObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
public Document getDocument(Long id) {
    return documentRepository.findById(id).orElseThrow(); // Always loads from DB
}
```

**Warning:** `@PostAuthorize` is suitable for read operations only. Never use it on write operations (the write has already happened when authorization fails).

</details>

---

🔴 **Q13: How do you handle CORS when both Spring MVC and Spring Security are configured?**

<details><summary>Click to reveal answer</summary>

CORS **must be handled before Spring Security** processes the request — otherwise, preflight OPTIONS requests (which lack an Authorization header) get rejected with 401 before CORS headers are added.

**Correct approach:** Configure CORS in Spring Security's filter chain via `CorsConfigurationSource`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource())) // ← Configure here
        .csrf(csrf -> csrf.disable())
        // ...
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myapp.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

**Do NOT** configure CORS only via `@CrossOrigin` or `WebMvcConfigurer.addCorsMappings()` when Spring Security is present — those are processed by the DispatcherServlet after security filters, which is too late for OPTIONS preflights.

</details>

---

🟡 **Q14: What is `DaoAuthenticationProvider` and what is its role?**

<details><summary>Click to reveal answer</summary>

`DaoAuthenticationProvider` is Spring Security's standard `AuthenticationProvider` that authenticates users by:
1. Loading `UserDetails` via `UserDetailsService.loadUserByUsername()`.
2. Comparing the provided password with the stored hash via `PasswordEncoder.matches()`.
3. Checking account status flags (`isEnabled()`, `isAccountNonLocked()`, etc.).

```java
@Bean
public AuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService); // Where to load users
    provider.setPasswordEncoder(passwordEncoder());      // How to verify passwords
    return provider;
}
```

It is registered with the `AuthenticationManager`, which is called by `authenticationManager.authenticate(new UsernamePasswordAuthenticationToken(email, password))` during login.

</details>

---

🟡 **Q15: What is `SessionCreationPolicy.STATELESS` and why is it critical for JWT APIs?**

<details><summary>Click to reveal answer</summary>

`SessionCreationPolicy.STATELESS` tells Spring Security to **never create or use an HTTP session** for storing the security context.

```java
http.sessionManagement(session ->
    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

**Why this matters for JWT:**
- Without this, Spring Security might create a `JSESSIONID` cookie even when using JWT.
- The client could then authenticate via session instead of JWT, creating a security inconsistency.
- With STATELESS, every request must carry a valid JWT — true stateless authentication.

**Effect:** `SecurityContextHolder` is cleared after every request and rebuilt from the JWT on every incoming request.

</details>

---

🔴 **Q16: How would you implement multi-tenancy in a JWT-secured Spring Boot application?**

<details><summary>Click to reveal answer</summary>

Embed tenant information in the JWT claims and use it to route database operations:

```java
// 1. Add tenantId claim during token generation
private String buildToken(UserDetails userDetails, AppUser user, long expirationMs) {
    return Jwts.builder()
        .subject(userDetails.getUsername())
        .claim("tenantId", user.getTenantId())  // ← tenant claim
        .claim("roles", getAuthorities(userDetails))
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + expirationMs))
        .signWith(getSigningKey())
        .compact();
}

// 2. Extract tenantId in filter and set in TenantContext
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(...) {
        String jwt = extractBearerToken(request);
        if (jwt != null) {
            Claims claims = jwtUtil.extractAllClaims(jwt);
            String tenantId = claims.get("tenantId", String.class);
            TenantContext.setCurrentTenant(tenantId); // ThreadLocal storage
        }
        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear(); // CRITICAL: always clean up
        }
    }
}

// 3. Use tenantId in repository queries
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByTenantId(String tenantId);
}
```

</details>

---

🟢 **Q17: What is the `@AuthenticationPrincipal` annotation?**

<details><summary>Click to reveal answer</summary>

`@AuthenticationPrincipal` injects the currently authenticated principal directly into a controller method parameter, replacing verbose `SecurityContextHolder` lookups.

```java
// Verbose approach
@GetMapping("/profile")
public ProfileDTO getProfile() {
    AppUser user = (AppUser) SecurityContextHolder
        .getContext()
        .getAuthentication()
        .getPrincipal();
    return profileService.getProfile(user.getId());
}

// Clean approach with @AuthenticationPrincipal
@GetMapping("/profile")
public ProfileDTO getProfile(@AuthenticationPrincipal AppUser currentUser) {
    return profileService.getProfile(currentUser.getId());
}

// With SpEL to extract a specific field
@GetMapping("/profile")
public ProfileDTO getProfile(
        @AuthenticationPrincipal(expression = "id") Long userId) {
    return profileService.getProfile(userId);
}
```

Returns `null` if no authentication exists (unauthenticated request). Use `@AuthenticationPrincipal(errorOnInvalidType = true)` to throw instead.

</details>

---

🟡 **Q18: What is the difference between `permitAll()` and `anonymous()` in Spring Security?**

<details><summary>Click to reveal answer</summary>

- **`permitAll()`**: Allows all requests to the matched path — authenticated AND unauthenticated (anonymous). No security check performed.
- **`anonymous()`**: Allows only **unauthenticated (anonymous)** users. Authenticated users are still subject to further authorization rules.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/public/**").permitAll()      // Anyone can access
    .requestMatchers("/api/login").anonymous()           // Only if NOT logged in
    .anyRequest().authenticated()
);
```

**Practical use:** `anonymous()` is useful for login/register pages — if the user is already authenticated, redirect them away. `permitAll()` is for truly public resources (static files, health checks, public API endpoints).

</details>

---

🔴 **Q19: How would you test Spring Security configuration in a unit/integration test?**

<details><summary>Click to reveal answer</summary>

**Unit test with `@WebMvcTest` and `@WithMockUser`:**

```java
@WebMvcTest(ProductController.class)
@Import(SecurityConfig.class)
class ProductControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @MockBean
    private JwtUtil jwtUtil;

    @MockBean
    private AppUserDetailsService userDetailsService;

    @Test
    void unauthenticatedRequest_shouldReturn401() throws Exception {
        mockMvc.perform(get("/api/products/1"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void authenticatedUser_canReadProduct() throws Exception {
        given(productService.findById(1L)).willReturn(testProduct());

        mockMvc.perform(get("/api/products/1"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void regularUser_cannotAccessAdminEndpoint() throws Exception {
        mockMvc.perform(delete("/api/admin/products/1"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void admin_canAccessAdminEndpoint() throws Exception {
        mockMvc.perform(delete("/api/admin/products/1"))
            .andExpect(status().isOk());
    }

    @Test
    void validJwtToken_shouldAuthenticate() throws Exception {
        String token = "valid.jwt.token";
        AppUser user = testUser();

        given(jwtUtil.extractUsername(token)).willReturn(user.getEmail());
        given(userDetailsService.loadUserByUsername(user.getEmail())).willReturn(user);
        given(jwtUtil.isTokenValid(token, user)).willReturn(true);

        mockMvc.perform(get("/api/products/1")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
}
```

**Custom security annotation for tests:**
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "admin@test.com", roles = "ADMIN")
public @interface WithMockAdmin { }

// Usage
@Test
@WithMockAdmin
void adminCanDeleteProduct() throws Exception { ... }
```

</details>

---

🟡 **Q20: What is `@EnableMethodSecurity` and what did it replace?**

<details><summary>Click to reveal answer</summary>

`@EnableMethodSecurity` (Spring Security 5.6+, default in 6.x) replaces `@EnableGlobalMethodSecurity`.

```java
// Old (Spring Security 5.x) — deprecated
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)

// New (Spring Security 6.x) — use this
@EnableMethodSecurity(
    prePostEnabled = true,  // Default: true (enables @PreAuthorize, @PostAuthorize, etc.)
    securedEnabled = true,  // Default: false (enables @Secured)
    jsr250Enabled = true    // Default: false (enables @RolesAllowed)
)
```

**Key improvements in `@EnableMethodSecurity`:**
- Uses `AuthorizationManager` instead of `AccessDecisionManager` (more extensible).
- `@PreAuthorize` is now an `@Inherited` annotation — works on interface methods.
- Better support for method security on `@Bean` methods.
- Proxyless support via `mode = AdviceMode.ASPECTJ` for direct method calls.

</details>

---

🟢 **Q21: What HTTP status codes should a security layer return and when?**

<details><summary>Click to reveal answer</summary>

| Scenario | Status Code | Spring Security Component |
|---|---|---|
| No credentials provided | 401 Unauthorized | `AuthenticationEntryPoint` |
| Invalid/expired token | 401 Unauthorized | `AuthenticationEntryPoint` |
| Valid token, wrong role | 403 Forbidden | `AccessDeniedHandler` |
| Invalid credentials (login) | 401 Unauthorized | `AuthenticationFailureHandler` |
| Successful login | 200 OK | `AuthenticationSuccessHandler` |

```java
// Custom entry point for 401
@Component
public class JwtAuthEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res,
                         AuthenticationException ex) throws IOException {
        res.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401
        res.setContentType("application/json");
        res.getWriter().write("{\"error\": \"Unauthorized\"}");
    }
}

// Custom handler for 403
@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest req, HttpServletResponse res,
                       AccessDeniedException ex) throws IOException {
        res.setStatus(HttpServletResponse.SC_FORBIDDEN); // 403
        res.setContentType("application/json");
        res.getWriter().write("{\"error\": \"Forbidden\"}");
    }
}
```

</details>

---

🔴 **Q22: How do you secure sensitive endpoints differently for microservices vs monoliths?**

<details><summary>Click to reveal answer</summary>

**Monolith:**
- Full Spring Security stack: JWT validation, UserDetailsService, method security.
- Single security context per request.
- All token validation logic lives in one place.

**Microservices:**
- **API Gateway** (Spring Cloud Gateway / Kong) validates the JWT once at the edge.
- Downstream services receive a propagated identity header (e.g., `X-User-Id`, `X-Roles`).
- Internal services can trust these headers (secured network, mTLS).
- Each service can optionally re-validate the JWT for defense-in-depth.

```java
// API Gateway: validate JWT, add headers
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("product-service", r -> r
            .path("/api/products/**")
            .filters(f -> f.filter(jwtValidationFilter)) // Validate here
            .uri("lb://product-service"))
        .build();
}

// Downstream service: trust propagated headers OR re-validate
@GetMapping("/products/{id}")
public Product getProduct(
        @PathVariable Long id,
        @RequestHeader("X-User-Id") Long userId,
        @RequestHeader("X-User-Roles") String roles) {
    // No JWT parsing needed — gateway already authenticated
}
```

**Spring Security for inter-service calls:** Use OAuth2 client credentials flow or mTLS for service-to-service authentication rather than user-scoped JWT tokens.

</details>

---

🟡 **Q23: What is the purpose of `WebAuthenticationDetailsSource` in the JWT filter?**

<details><summary>Click to reveal answer</summary>

`WebAuthenticationDetailsSource` builds a `WebAuthenticationDetails` object that captures metadata from the HTTP request:
- Remote IP address
- Session ID (if any)

```java
authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
```

These details are attached to the `Authentication` object and available throughout the request for:
- **Audit logging**: Log which IP made the authenticated request.
- **Fraud detection**: Flag requests from unexpected IPs.
- **Custom authorization logic**: IP-based access control in `@PreAuthorize` or custom `AuthorizationManager`.

```java
// Access in security logic
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
WebAuthenticationDetails details = (WebAuthenticationDetails) auth.getDetails();
String remoteIp = details.getRemoteAddress();
```

It's optional but recommended for production audit trails.

</details>

---

🔴 **Q24: How do you implement role hierarchy in Spring Security?**

<details><summary>Click to reveal answer</summary>

Role hierarchy allows higher roles to inherit permissions of lower roles:

```
ROLE_ADMIN > ROLE_MODERATOR > ROLE_USER
```

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_ADMIN > ROLE_MODERATOR
        ROLE_MODERATOR > ROLE_USER
        """);
}

@Bean
public DefaultWebSecurityExpressionHandler expressionHandler() {
    DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy());
    return handler;
}
```

Register with HTTP security:
```java
http.authorizeHttpRequests(auth -> auth
    .expressionHandler(expressionHandler())
    // Now ADMIN can access USER-only endpoints automatically
    .requestMatchers("/api/user/**").hasRole("USER")
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
);
```

**With method security** — inject into `MethodSecurityExpressionHandler`:

```java
@Bean
public MethodSecurityExpressionHandler methodSecurityExpressionHandler() {
    DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy());
    return handler;
}
```

Now `@PreAuthorize("hasRole('USER')")` automatically passes for ADMIN and MODERATOR users.

</details>

---

🟡 **Q25: What are the security risks of storing JWT in localStorage vs httpOnly cookies?**

<details><summary>Click to reveal answer</summary>

| Storage | XSS Risk | CSRF Risk | Recommendation |
|---|---|---|---|
| `localStorage` | ❌ High (JS can read it) | ✅ Low (not auto-sent) | Avoid for sensitive apps |
| `sessionStorage` | ❌ High (JS can read it) | ✅ Low | Avoid for sensitive apps |
| `httpOnly` cookie | ✅ Low (JS cannot read) | ❌ High (auto-sent) | Use with CSRF protection |
| In-memory (JS var) | ✅ Low (lost on refresh) | ✅ Low | Best security, UX trade-off |

**Recommended approach for SPAs:**
- Store the **access token in memory** (JS variable — lost on page refresh).
- Store the **refresh token in an httpOnly cookie** with `Secure` and `SameSite=Strict` flags.
- On page load, silently call `/api/auth/refresh` to get a new access token.

```java
// Set httpOnly cookie for refresh token
ResponseCookie cookie = ResponseCookie.from("refreshToken", refreshToken.getToken())
    .httpOnly(true)
    .secure(true)
    .sameSite("Strict")
    .path("/api/auth/refresh")
    .maxAge(Duration.ofDays(7))
    .build();

response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```

</details>

---

*Last updated: 2025 | Spring Boot 3.x | Spring Security 6.x | JJWT 0.12.x | Java 21*
