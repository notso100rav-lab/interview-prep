# Chapter 4: Java Networking

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- TCP vs UDP
- `java.net` package overview
- `Socket` and `ServerSocket` (TCP)
- `DatagramSocket` (UDP)
- `URL` and `URI`
- `HttpURLConnection`
- `HttpClient` (Java 11+)
- HTTP methods and status codes
- REST vs SOAP
- Connection pooling and timeouts
- Non-blocking I/O with `java.nio`

---

## 🔑 Key Concepts at a Glance

```
Networking Layers:
  Application (HTTP, FTP, SMTP)
  Transport   (TCP, UDP) ← Java networking operates here and above
  Network     (IP)
  Data Link   (Ethernet)
  Physical    (cables, WiFi)

Java Networking API:
  java.net    → classic blocking I/O (Socket, ServerSocket, URL, HttpURLConnection)
  java.nio    → non-blocking, selector-based I/O (Netty is built on this)
  java.net.http → modern HTTP client (Java 11+, async, HTTP/2)
```

---

## 🧠 Theory Questions

### TCP vs UDP

---

🟢 **Q1. What is the difference between TCP and UDP?**

<details><summary>Click to reveal answer</summary>

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | Guaranteed delivery, ordering, retransmit | Best-effort, may lose/reorder |
| **Speed** | Slower (overhead) | Faster (no overhead) |
| **Flow control** | ✅ Yes | ❌ No |
| **Header size** | 20–60 bytes | 8 bytes |
| **Use cases** | HTTP, HTTPS, FTP, SMTP, SSH | DNS, video streaming, VoIP, gaming |
| **Java class** | `Socket`, `ServerSocket` | `DatagramSocket`, `DatagramPacket` |

**TCP 3-way handshake:**
```
Client                Server
  │──── SYN ──────────►│
  │◄─── SYN-ACK ───────│
  │──── ACK ──────────►│
  │    Connection Established    │
```

**When to use UDP:**
- Real-time applications where latency > reliability (video call, gaming)
- Simple query/response (DNS)
- Broadcast/multicast
- When application implements its own reliability (QUIC protocol)

</details>

---

🟢 **Q2. What is the difference between `URL` and `URI`?**

<details><summary>Click to reveal answer</summary>

| | URI | URL |
|-|-----|-----|
| **Full name** | Uniform Resource Identifier | Uniform Resource Locator |
| **Purpose** | Identifies a resource | Locates a resource (specifies HOW to access) |
| **Relationship** | Superset | Subset of URI |
| **Includes** | URN, URL | Always has scheme + location |

```
URI  = scheme ":" ["//" authority] path ["?" query] ["#" fragment]
URL  = URI that specifies how to access the resource (http://, ftp://, etc.)
URN  = URI that names a resource without location (urn:isbn:0451450523)

Examples:
  URL: https://example.com/api/users?page=1#top
  URN: urn:isbn:0451450523 (identifies a book, no access method)
  URI: Both URL and URN are URIs
```

```java
import java.net.*;

// URL — can open connections
URL url = new URL("https://api.example.com/users");
URLConnection conn = url.openConnection();

// URI — more general, better for construction/manipulation
URI uri = new URI("https", "api.example.com", "/users", "page=1", "section-1");
System.out.println(uri); // https://api.example.com/users?page=1#section-1

// Convert URI to URL
URL urlFromUri = uri.toURL();

// URI for resource identifiers in REST:
URI resourceUri = URI.create("/api/v1/users/123");
```

</details>

---

### Sockets

---

🟡 **Q3. How do you create a simple TCP client-server in Java?**

<details><summary>Click to reveal answer</summary>

**Server (accepts one connection at a time — not production-ready):**
```java
import java.net.*;
import java.io.*;

public class EchoServer {
    public static void main(String[] args) throws IOException {
        int port = 8080;

        // Bind to port
        try (ServerSocket serverSocket = new ServerSocket(port)) {
            System.out.println("Server listening on port " + port);

            // Accept clients in a loop
            while (true) {
                Socket clientSocket = serverSocket.accept(); // blocks until client connects
                System.out.println("Client connected: " + clientSocket.getInetAddress());

                // Handle each client in a separate thread (basic multithreaded server)
                new Thread(() -> handleClient(clientSocket)).start();
            }
        }
    }

    private static void handleClient(Socket socket) {
        try (socket;  // Java 9+ try-with-resources on variable
             BufferedReader in = new BufferedReader(
                 new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true)) {

            String line;
            while ((line = in.readLine()) != null) {
                System.out.println("Received: " + line);
                out.println("Echo: " + line);  // echo back
            }
        } catch (IOException e) {
            System.err.println("Error handling client: " + e.getMessage());
        }
    }
}
```

**Client:**
```java
public class EchoClient {
    public static void main(String[] args) throws IOException {
        try (Socket socket = new Socket("localhost", 8080);
             BufferedReader in = new BufferedReader(
                 new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader stdin = new BufferedReader(
                 new InputStreamReader(System.in))) {

            // Set timeouts
            socket.setSoTimeout(5000);        // read timeout: 5s
            socket.setTcpNoDelay(true);       // disable Nagle's algorithm

            String userInput;
            while ((userInput = stdin.readLine()) != null) {
                out.println(userInput);
                System.out.println("Server: " + in.readLine());
            }
        }
    }
}
```

**Production-ready server uses `ExecutorService`:**
```java
ExecutorService pool = Executors.newFixedThreadPool(50);
while (!serverSocket.isClosed()) {
    Socket client = serverSocket.accept();
    pool.submit(() -> handleClient(client));
}
```

</details>

---

🟡 **Q4. How do you implement UDP communication in Java?**

<details><summary>Click to reveal answer</summary>

```java
// UDP Server
public class UdpServer {
    public static void main(String[] args) throws IOException {
        try (DatagramSocket socket = new DatagramSocket(9090)) {
            byte[] buffer = new byte[1024];
            System.out.println("UDP Server listening on port 9090");

            while (true) {
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
                socket.receive(packet);  // blocks until packet received

                String message = new String(packet.getData(), 0, packet.getLength());
                System.out.println("Received from " + packet.getAddress() + ": " + message);

                // Echo back to sender
                String response = "ACK: " + message;
                byte[] responseData = response.getBytes();
                DatagramPacket responsePacket = new DatagramPacket(
                    responseData, responseData.length,
                    packet.getAddress(), packet.getPort()
                );
                socket.send(responsePacket);
            }
        }
    }
}

// UDP Client
public class UdpClient {
    public static void main(String[] args) throws IOException {
        try (DatagramSocket socket = new DatagramSocket()) {
            InetAddress serverAddress = InetAddress.getByName("localhost");
            socket.setSoTimeout(2000);  // timeout for receive

            String message = "Hello UDP";
            byte[] data = message.getBytes();
            DatagramPacket packet = new DatagramPacket(data, data.length, serverAddress, 9090);
            socket.send(packet);

            // Receive response
            byte[] buffer = new byte[1024];
            DatagramPacket response = new DatagramPacket(buffer, buffer.length);
            socket.receive(response);
            System.out.println("Server: " + new String(response.getData(), 0, response.getLength()));
        }
    }
}
```

**Key UDP characteristics:**
- No connection establishment — just send packets
- Each packet is independent
- Must handle packet loss yourself if needed
- `DatagramPacket` contains data, destination IP, and port

</details>

---

### HttpURLConnection and HttpClient

---

🟡 **Q5. How do you make an HTTP GET request using `HttpURLConnection`?**

<details><summary>Click to reveal answer</summary>

```java
import java.net.*;
import java.io.*;
import java.nio.charset.StandardCharsets;

public class HttpExample {

    // GET request
    public static String get(String urlString) throws IOException {
        URL url = new URL(urlString);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();

        try {
            conn.setRequestMethod("GET");
            conn.setRequestProperty("Accept", "application/json");
            conn.setRequestProperty("Authorization", "Bearer " + getToken());
            conn.setConnectTimeout(5000);  // connection timeout
            conn.setReadTimeout(10000);    // read timeout

            int responseCode = conn.getResponseCode();
            System.out.println("Response Code: " + responseCode);

            if (responseCode == HttpURLConnection.HTTP_OK) {
                try (BufferedReader br = new BufferedReader(
                        new InputStreamReader(conn.getInputStream(), StandardCharsets.UTF_8))) {
                    StringBuilder response = new StringBuilder();
                    String line;
                    while ((line = br.readLine()) != null) {
                        response.append(line);
                    }
                    return response.toString();
                }
            } else {
                // Read error stream for 4xx/5xx responses
                try (BufferedReader br = new BufferedReader(
                        new InputStreamReader(conn.getErrorStream(), StandardCharsets.UTF_8))) {
                    // handle error
                }
                throw new IOException("HTTP error: " + responseCode);
            }
        } finally {
            conn.disconnect();
        }
    }

    // POST request with JSON body
    public static String post(String urlString, String jsonBody) throws IOException {
        URL url = new URL(urlString);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();

        try {
            conn.setRequestMethod("POST");
            conn.setRequestProperty("Content-Type", "application/json; charset=UTF-8");
            conn.setRequestProperty("Accept", "application/json");
            conn.setDoOutput(true);  // enable writing request body
            conn.setConnectTimeout(5000);
            conn.setReadTimeout(10000);

            // Write request body
            try (OutputStream os = conn.getOutputStream()) {
                byte[] input = jsonBody.getBytes(StandardCharsets.UTF_8);
                os.write(input, 0, input.length);
            }

            int responseCode = conn.getResponseCode();
            try (BufferedReader br = new BufferedReader(
                    new InputStreamReader(conn.getInputStream(), StandardCharsets.UTF_8))) {
                StringBuilder response = new StringBuilder();
                String line;
                while ((line = br.readLine()) != null) response.append(line);
                return response.toString();
            }
        } finally {
            conn.disconnect();
        }
    }
}
```

</details>

---

🟡 **Q6. How does Java 11's `HttpClient` improve on `HttpURLConnection`?**

<details><summary>Click to reveal answer</summary>

`java.net.http.HttpClient` (Java 11+) provides a modern, fluent API with async support and HTTP/2.

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;

// Build client once, reuse
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .version(HttpClient.Version.HTTP_2)
    .build();

// Synchronous GET
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Accept", "application/json")
    .header("Authorization", "Bearer " + token)
    .timeout(Duration.ofSeconds(10))
    .GET()
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println("Status: " + response.statusCode());
System.out.println("Body: " + response.body());

// Asynchronous GET
CompletableFuture<HttpResponse<String>> futureResponse =
    client.sendAsync(request, HttpResponse.BodyHandlers.ofString());

futureResponse
    .thenApply(HttpResponse::body)
    .thenAccept(body -> System.out.println("Async response: " + body))
    .exceptionally(ex -> { System.err.println("Error: " + ex); return null; });

// POST with JSON body
HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"Alice\"}"))
    .build();

HttpResponse<String> postResponse = client.send(postRequest, HttpResponse.BodyHandlers.ofString());

// Stream response body
HttpResponse<Stream<String>> streamResponse =
    client.send(request, HttpResponse.BodyHandlers.ofLines());
streamResponse.body().forEach(System.out::println);
```

**Comparison:**

| Feature | `HttpURLConnection` | `HttpClient` (Java 11) |
|---------|--------------------|-----------------------|
| API style | Verbose, stateful | Fluent, immutable |
| Async | ❌ | ✅ `sendAsync()` |
| HTTP/2 | ❌ | ✅ |
| WebSocket | ❌ | ✅ |
| Connection pooling | Manual | ✅ Automatic |
| Thread-safe? | ❌ Not reusable | ✅ Yes, reuse the client |

</details>

---

### HTTP Protocol

---

🟢 **Q7. What are the main HTTP methods and their semantics?**

<details><summary>Click to reveal answer</summary>

| Method | Purpose | Idempotent? | Safe? | Request Body? |
|--------|---------|-------------|-------|---------------|
| `GET` | Retrieve resource | ✅ | ✅ | No (discouraged) |
| `POST` | Create resource / non-idempotent action | ❌ | ❌ | Yes |
| `PUT` | Replace entire resource | ✅ | ❌ | Yes |
| `PATCH` | Partial update | ❌ (can be) | ❌ | Yes |
| `DELETE` | Delete resource | ✅ | ❌ | Rarely |
| `HEAD` | Like GET but no body (check metadata) | ✅ | ✅ | No |
| `OPTIONS` | List supported methods (CORS preflight) | ✅ | ✅ | No |

**Idempotent:** Calling N times has same effect as calling once.
**Safe:** Does not modify server state.

```
GET    /users         → list all users
GET    /users/123     → get user 123
POST   /users         → create new user
PUT    /users/123     → replace user 123 completely
PATCH  /users/123     → update specific fields of user 123
DELETE /users/123     → delete user 123
```

</details>

---

🟢 **Q8. What are common HTTP status codes?**

<details><summary>Click to reveal answer</summary>

| Code | Category | Meaning |
|------|---------|---------|
| **2xx** | **Success** | |
| 200 | OK | Success (GET, PUT, PATCH response) |
| 201 | Created | Resource created (POST response), include Location header |
| 204 | No Content | Success, no body (DELETE response) |
| **3xx** | **Redirection** | |
| 301 | Moved Permanently | Resource has new URL (cached) |
| 302 | Found | Temporary redirect |
| 304 | Not Modified | Client cache is fresh (use ETag/If-None-Match) |
| **4xx** | **Client Errors** | |
| 400 | Bad Request | Invalid syntax, validation failure |
| 401 | Unauthorized | Authentication required (not logged in) |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported |
| 409 | Conflict | Resource conflict (duplicate, version mismatch) |
| 422 | Unprocessable Entity | Semantic validation error |
| 429 | Too Many Requests | Rate limited |
| **5xx** | **Server Errors** | |
| 500 | Internal Server Error | Generic server error |
| 502 | Bad Gateway | Upstream server error |
| 503 | Service Unavailable | Server overloaded or down |
| 504 | Gateway Timeout | Upstream server timeout |

> 💡 **Interviewer Tip:** Distinguish 401 (not authenticated) vs 403 (authenticated but unauthorized). Many developers confuse these.

</details>

---

### REST vs SOAP

---

🟡 **Q9. What is the difference between REST and SOAP?**

<details><summary>Click to reveal answer</summary>

| Aspect | REST | SOAP |
|--------|------|------|
| **Style** | Architectural style | Protocol |
| **Transport** | HTTP (usually) | HTTP, SMTP, TCP, etc. |
| **Format** | JSON, XML, any | XML only |
| **Contract** | OpenAPI/Swagger (optional) | WSDL (mandatory) |
| **State** | Stateless | Can be stateful |
| **Operations** | HTTP methods (GET/POST/PUT/DELETE) | SOAP actions in body |
| **Error handling** | HTTP status codes | SOAP Fault element |
| **WS-* standards** | No | Yes (WS-Security, WS-ReliableMessaging) |
| **Performance** | Lighter, faster | Heavier (XML overhead) |
| **Use case** | Public APIs, microservices | Enterprise, banking, strict contracts |

**REST constraints (Richardson Maturity Model):**
- Level 0: HTTP as transport (like RPC)
- Level 1: Resources (URLs for each resource)
- Level 2: HTTP verbs (GET, POST, PUT, DELETE)
- Level 3: Hypermedia/HATEOAS (links to next actions in response)

**SOAP example:**
```xml
<soap:Envelope>
  <soap:Header>...</soap:Header>
  <soap:Body>
    <GetUserRequest>
      <UserId>123</UserId>
    </GetUserRequest>
  </soap:Body>
</soap:Envelope>
```

**REST equivalent:**
```
GET /api/users/123
Accept: application/json
```

**Java for SOAP:** JAX-WS (`@WebService`, `@WebMethod`)
**Java for REST:** JAX-RS (Jersey, RESTEasy), Spring MVC

</details>

---

🟡 **Q10. What are the key principles of RESTful API design?**

<details><summary>Click to reveal answer</summary>

**1. Stateless:** Each request contains all info needed. No server-side session.

**2. Uniform Interface:**
```
GET    /users          → list users
GET    /users/{id}     → get specific user
POST   /users          → create user
PUT    /users/{id}     → update user
DELETE /users/{id}     → delete user

# Sub-resources:
GET    /users/{id}/orders       → user's orders
POST   /users/{id}/orders       → create order for user
GET    /users/{id}/orders/{oid} → specific order
```

**3. Resource naming (nouns, not verbs):**
```
BAD:  GET /getUsers, POST /createUser, DELETE /deleteUser?id=1
GOOD: GET /users, POST /users, DELETE /users/1
```

**4. Versioning:**
```
/api/v1/users      → version in URL (simple, widely used)
Accept: application/vnd.myapp.v2+json  → content negotiation
X-API-Version: 2   → custom header
```

**5. Pagination:**
```json
GET /users?page=2&size=20

{
  "data": [...],
  "pagination": {
    "page": 2,
    "size": 20,
    "total": 150,
    "links": {
      "self": "/users?page=2&size=20",
      "next": "/users?page=3&size=20",
      "prev": "/users?page=1&size=20"
    }
  }
}
```

**6. Error responses:**
```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Email is required",
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/api/users"
}
```

</details>

---

### Timeouts and Connection Pooling

---

🟡 **Q11. What types of timeouts should you configure for network connections?**

<details><summary>Click to reveal answer</summary>

**Types of timeouts:**

```java
// 1. Connection timeout: time to establish TCP connection
conn.setConnectTimeout(5000);   // 5 seconds

// 2. Read/Socket timeout: time waiting for data after connection established
conn.setReadTimeout(30000);     // 30 seconds (for slow responses)

// 3. Connection request timeout (from pool): time to get a connection from pool
// (Apache HttpClient example)
RequestConfig config = RequestConfig.custom()
    .setConnectionRequestTimeout(5000)   // wait for pool connection
    .setConnectTimeout(5000)              // TCP connect
    .setSocketTimeout(30000)              // read data
    .build();
```

**Rules of thumb:**
- Connect timeout: 3–10 seconds (TCP handshake should be fast)
- Read timeout: depends on operation (30s for normal, longer for file uploads)
- Always set timeouts — default is often infinite (hangs forever!)

```java
// Java 11 HttpClient
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .timeout(Duration.ofSeconds(30))  // overall request timeout
    .build();
```

</details>

---

🔴 **Q12. What is connection pooling and why is it important?**

<details><summary>Click to reveal answer</summary>

**Connection pooling** reuses TCP connections instead of creating a new one for each request. Creating a TCP connection involves a 3-way handshake (latency) and resource allocation.

**Without pooling:**
```
Request 1: connect → send → receive → disconnect  (slow)
Request 2: connect → send → receive → disconnect  (slow again)
```

**With pooling:**
```
Startup: create N connections
Request 1: get connection → send → receive → return to pool  (fast)
Request 2: get connection → send → receive → return to pool  (fast)
```

**Java HTTP connection pooling:**
```java
// Java 11 HttpClient — automatic connection pooling
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)  // HTTP/2 multiplexes multiple requests on 1 connection
    .build();
// Reuse client across requests — don't create a new one per request!

// Apache HttpClient 5 — explicit pool configuration
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(100);         // max total connections
cm.setDefaultMaxPerRoute(20); // max per host

CloseableHttpClient httpClient = HttpClients.custom()
    .setConnectionManager(cm)
    .build();
```

**JDBC connection pool (HikariCP — most popular):**
```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost/mydb");
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);

HikariDataSource ds = new HikariDataSource(config);
```

> 💡 **Interviewer Tip:** Connection pools are critical for performance. In Spring Boot, HikariCP is the default JDBC pool. Knowing pool sizing principles (CPU cores × 2 for CPU-bound, more for I/O-bound) shows production experience.

</details>

---

### java.nio and Non-Blocking I/O

---

🔴 **Q13. What is the difference between blocking (java.io) and non-blocking (java.nio) I/O?**

<details><summary>Click to reveal answer</summary>

**Blocking I/O (java.io):**
- Each connection needs a dedicated thread
- Thread blocks (sleeps) while waiting for data
- Scales poorly (C10K problem)

**Non-Blocking I/O (java.nio):**
- Single thread can handle many connections using `Selector`
- Thread polls for ready channels (read/write ready)
- Scales to millions of connections

```java
// Blocking I/O: one thread per connection
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket client = serverSocket.accept();
    new Thread(() -> handleClient(client)).start();  // 10,000 connections = 10,000 threads!
}

// Non-Blocking NIO: one selector thread for many channels
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);  // non-blocking!
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // blocks until at least one channel is ready
    Iterator<SelectionKey> keys = selector.selectedKeys().iterator();

    while (keys.hasNext()) {
        SelectionKey key = keys.next();
        keys.remove();

        if (key.isAcceptable()) {
            // Accept new connection
            SocketChannel client = serverChannel.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // Read data from channel
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            int bytesRead = client.read(buffer);
            if (bytesRead == -1) {
                client.close();
            } else {
                buffer.flip();
                // process data
            }
        }
    }
}
```

**In practice:** Use Netty (built on NIO) instead of raw NIO — it handles the complexity.

```
Netty Architecture:
  EventLoopGroup (boss) → accepts connections
  EventLoopGroup (worker) → handles I/O on channels
  ChannelPipeline → chain of handlers (encode/decode, business logic)
```

</details>

---

## 🎭 Scenario-Based Questions

---

🟡 **S1. How would you implement a simple HTTP health check endpoint?**

<details><summary>Click to reveal answer</summary>

```java
// Simple HTTP server using Java's built-in com.sun.net.httpserver (avoid in production)
// or using HttpClient to call an external service

// Using Java 18+ SimpleFileServer or custom handler
public class HealthCheckServer {
    public static void main(String[] args) throws IOException {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

        // Health check handler
        server.createContext("/health", exchange -> {
            String response = "{\"status\":\"UP\",\"timestamp\":\"" +
                Instant.now() + "\"}";
            exchange.getResponseHeaders().set("Content-Type", "application/json");
            exchange.sendResponseHeaders(200, response.length());
            try (OutputStream os = exchange.getResponseBody()) {
                os.write(response.getBytes());
            }
        });

        server.setExecutor(Executors.newFixedThreadPool(4));
        server.start();
        System.out.println("Health check available at http://localhost:8080/health");
    }
}

// More realistic: use Spring Boot Actuator
// application.properties:
// management.endpoints.web.exposure.include=health,info,metrics
// GET /actuator/health → {"status":"UP"}
```

</details>

---

🟡 **S2. How do you handle HTTP redirects in `HttpURLConnection`?**

<details><summary>Click to reveal answer</summary>

```java
// HttpURLConnection does NOT automatically follow redirects from HTTP to HTTPS
// (for security reasons — different protocols)
HttpURLConnection conn = (HttpURLConnection) new URL(url).openConnection();
conn.setInstanceFollowRedirects(false);  // disable auto-follow

int responseCode = conn.getResponseCode();
if (responseCode == HttpURLConnection.HTTP_MOVED_PERM ||
    responseCode == HttpURLConnection.HTTP_MOVED_TEMP ||
    responseCode == 307 || responseCode == 308) {

    String newUrl = conn.getHeaderField("Location");
    conn.disconnect();

    // Follow redirect manually
    conn = (HttpURLConnection) new URL(newUrl).openConnection();
    responseCode = conn.getResponseCode();
}

// Java 11 HttpClient handles redirects automatically:
HttpClient client = HttpClient.newBuilder()
    .followRedirects(HttpClient.Redirect.ALWAYS)  // ALWAYS, NORMAL, NEVER
    .build();
// NORMAL: follows all redirects except from HTTPS to HTTP
// ALWAYS: follows all (less secure)
```

</details>

---

🟡 **S3. What happens when a `Socket` connection is refused?**

<details><summary>Click to reveal answer</summary>

```java
try {
    Socket socket = new Socket("localhost", 8080);
} catch (ConnectException e) {
    // "Connection refused" — server not listening on that port
    System.err.println("Server not available: " + e.getMessage());
} catch (SocketTimeoutException e) {
    // connect timeout expired before connection established
    System.err.println("Connection timed out: " + e.getMessage());
} catch (UnknownHostException e) {
    // hostname cannot be resolved to IP
    System.err.println("Unknown host: " + e.getMessage());
} catch (IOException e) {
    // Other network errors
    System.err.println("Network error: " + e.getMessage());
}
```

**Common causes of `ConnectException`:**
1. Server not started
2. Firewall blocking the port
3. Wrong port number
4. Wrong hostname/IP

**Retry pattern with exponential backoff:**
```java
int maxRetries = 5;
long delay = 1000;

for (int i = 0; i < maxRetries; i++) {
    try {
        Socket socket = new Socket("localhost", 8080);
        // success
        break;
    } catch (ConnectException e) {
        if (i < maxRetries - 1) {
            Thread.sleep(delay);
            delay *= 2;  // exponential backoff
        } else {
            throw e;  // give up
        }
    }
}
```

</details>

---

🔴 **S4. How would you implement a simple HTTP connection pool from scratch?**

<details><summary>Click to reveal answer</summary>

```java
public class SimpleConnectionPool {
    private final String host;
    private final int port;
    private final int maxConnections;
    private final BlockingQueue<Socket> pool;

    public SimpleConnectionPool(String host, int port, int maxConnections) throws IOException {
        this.host = host;
        this.port = port;
        this.maxConnections = maxConnections;
        this.pool = new ArrayBlockingQueue<>(maxConnections);

        // Pre-create connections
        for (int i = 0; i < maxConnections; i++) {
            pool.offer(createConnection());
        }
    }

    private Socket createConnection() throws IOException {
        Socket socket = new Socket(host, port);
        socket.setKeepAlive(true);
        socket.setSoTimeout(30000);
        return socket;
    }

    // Borrow a connection (blocks if none available)
    public Socket borrow() throws InterruptedException, IOException {
        Socket socket = pool.poll(5, TimeUnit.SECONDS);
        if (socket == null) throw new IOException("Connection pool exhausted");

        // Check if connection is still valid
        if (socket.isClosed() || !socket.isConnected()) {
            socket = createConnection();
        }
        return socket;
    }

    // Return connection to pool
    public void returnConnection(Socket socket) {
        if (socket != null && !socket.isClosed()) {
            pool.offer(socket);
        }
    }

    public void shutdown() {
        pool.forEach(socket -> {
            try { socket.close(); } catch (IOException ignored) {}
        });
    }
}

// Usage:
SimpleConnectionPool pool = new SimpleConnectionPool("api.example.com", 443, 10);
Socket conn = pool.borrow();
try {
    // use connection
} finally {
    pool.returnConnection(conn);  // MUST return in finally!
}
```

> 💡 In production, use Apache HttpClient, OkHttp, or Java 11 HttpClient which handle pooling, health checks, and keep-alive automatically.

</details>

---

🟡 **S5. What is the difference between `setConnectTimeout` and `setReadTimeout`?**

<details><summary>Click to reveal answer</summary>

```
Timeline of an HTTP request:
                                         
Client                                                Server
  │                                                     │
  ├──[connect timeout]──────────────────────────────────►│
  │         ← TCP 3-way handshake (SYN, SYN-ACK, ACK)   │
  │                                                      │
  ├──[HTTP request sent]─────────────────────────────────►│
  │                                                      │
  │◄────[read timeout]─── waiting for first byte of response ───┤
```

- **`setConnectTimeout(ms)`:** Maximum time to establish the TCP connection. Throws `SocketTimeoutException` if can't connect within this time.

- **`setReadTimeout(ms)`:** Maximum time to wait for data AFTER the connection is established. Applies to each read operation. If server processes slowly and doesn't send any data for this duration, throws `SocketTimeoutException`.

```java
conn.setConnectTimeout(5000);   // 5s to establish TCP connection
conn.setReadTimeout(30000);     // 30s to receive data once connected

// Example: Server hangs after receiving request, never sends response
// After 30s with no data: SocketTimeoutException (read timeout)

// Example: Server not listening on port
// After 5s: SocketTimeoutException (connect timeout)
// Or immediately: ConnectException (connection refused)
```

</details>

---

🟡 **S6. How do you handle SSL/HTTPS in Java?**

<details><summary>Click to reveal answer</summary>

```java
// HTTPS with HttpURLConnection — works out of the box for trusted certs
URL url = new URL("https://api.example.com/data");
HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();
conn.setRequestMethod("GET");
// Java's default SSLSocketFactory uses the JVM's cacerts truststore

// Custom truststore (for self-signed certificates in dev/testing)
KeyStore trustStore = KeyStore.getInstance("JKS");
trustStore.load(new FileInputStream("mytruststore.jks"), "password".toCharArray());

TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
tmf.init(trustStore);

SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, tmf.getTrustManagers(), null);

conn.setSSLSocketFactory(sslContext.getSocketFactory());

// Java 11 HttpClient with custom SSL
HttpClient client = HttpClient.newBuilder()
    .sslContext(sslContext)
    .build();

// ⚠️ DO NOT disable SSL verification in production:
// (This is only for testing/debugging)
TrustManager[] trustAll = new TrustManager[]{
    new X509TrustManager() {
        public X509Certificate[] getAcceptedIssuers() { return null; }
        public void checkClientTrusted(X509Certificate[] certs, String authType) {}
        public void checkServerTrusted(X509Certificate[] certs, String authType) {}
    }
};
// NEVER use in production — disables security!
```

🚨 **Never disable SSL certificate validation in production code.** It makes your application vulnerable to man-in-the-middle attacks.

</details>

---

🟡 **S7. How do you parse query parameters from a URL?**

<details><summary>Click to reveal answer</summary>

```java
// Manual parsing
public static Map<String, String> parseQueryParams(String query) {
    if (query == null || query.isEmpty()) return Collections.emptyMap();

    Map<String, String> params = new LinkedHashMap<>();
    String[] pairs = query.split("&");
    for (String pair : pairs) {
        int idx = pair.indexOf("=");
        if (idx > 0) {
            String key = URLDecoder.decode(pair.substring(0, idx), StandardCharsets.UTF_8);
            String value = URLDecoder.decode(pair.substring(idx + 1), StandardCharsets.UTF_8);
            params.put(key, value);
        }
    }
    return params;
}

// Usage:
URI uri = URI.create("https://example.com/search?q=java+17&page=2&sort=date");
Map<String, String> params = parseQueryParams(uri.getQuery());
// params = {q: "java 17", page: "2", sort: "date"}

// URL encoding:
String encoded = URLEncoder.encode("hello world & foo=bar", StandardCharsets.UTF_8);
// "hello+world+%26+foo%3Dbar"

// In Spring MVC (framework):
@GetMapping("/search")
public ResponseEntity<?> search(@RequestParam String q,
                                 @RequestParam(defaultValue = "1") int page) {
    // Spring auto-decodes query params
}
```

</details>

---

🔴 **S8. What is HTTP Keep-Alive and why is it important?**

<details><summary>Click to reveal answer</summary>

**HTTP Keep-Alive** (persistent connections) reuses the same TCP connection for multiple HTTP requests, instead of closing and reopening for each request.

**Without Keep-Alive (HTTP/1.0 default):**
```
Request 1: TCP connect → HTTP GET → HTTP response → TCP close
Request 2: TCP connect → HTTP GET → HTTP response → TCP close
(Each request: ~100ms TCP handshake overhead)
```

**With Keep-Alive (HTTP/1.1 default):**
```
TCP connect
Request 1: HTTP GET → HTTP response
Request 2: HTTP GET → HTTP response  ← reuse connection
Request 3: HTTP GET → HTTP response  ← reuse connection
TCP close (after idle timeout or explicit Connection: close)
```

**HTTP/2 multiplexing (even better):**
```
Single TCP connection
Request 1 ──────────────►
Request 2 ──────────────►  (concurrent, not sequential!)
Request 3 ──────────────►
◄── Response 2
◄── Response 1
◄── Response 3
```

```java
// HttpURLConnection: Keep-Alive is automatic in Java
// Check: System.setProperty("http.keepAlive", "false") to disable

// Java 11 HttpClient: defaults to HTTP/2 with multiplexing
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .build();
// Multiple requests reuse the same connection automatically

// Response headers indicate Keep-Alive:
// Connection: keep-alive
// Keep-Alive: timeout=5, max=1000
```

> 💡 **Interviewer Tip:** This is why you should create ONE `HttpClient` instance and reuse it — recreating it defeats connection reuse.

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| Not setting timeouts | Always set `connectTimeout` and `readTimeout` |
| Using `==` to compare `URL` objects | Use `url.equals()` or compare `toString()` |
| Disabling SSL verification in production | Use proper certificate management |
| Creating new `HttpClient` per request | Reuse a single `HttpClient` instance |
| Not closing connections | Use try-with-resources |
| Ignoring HTTP error status codes | Check `getResponseCode()` and handle 4xx/5xx |
| Not URL-encoding query parameters | Use `URLEncoder.encode()` |
| Using `HttpURLConnection` for new code | Prefer Java 11 `HttpClient` |
| Blocking I/O in high-throughput servers | Use NIO or Netty |
| Using `com.sun.net.httpserver` in production | Use Netty, Jetty, or Undertow |

---

## ✅ Best Practices

- ✅ Always set connection and read timeouts
- ✅ Reuse `HttpClient` instances (don't create per-request)
- ✅ Use connection pooling for external HTTP calls
- ✅ Handle redirects explicitly if needed
- ✅ Always close/return connections (try-with-resources)
- ✅ Use `CompletableFuture` with `sendAsync()` for concurrent HTTP calls
- ✅ Prefer JSON over XML for REST APIs
- ✅ Use TLS 1.2+ and validate SSL certificates
- ✅ Implement retry logic with exponential backoff and jitter
- ✅ Log request IDs for distributed tracing

---

## 🔗 Related Sections

- [Chapter 3: Multithreading](./03-multithreading-and-concurrency.md) — async HTTP with `CompletableFuture`
- [Chapter 5: Exception Handling](./05-exception-handling.md) — handling network exceptions

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 3: Multithreading](./03-multithreading-and-concurrency.md) | [Section README](./README.md) | [Chapter 5: Exception Handling →](./05-exception-handling.md) |
