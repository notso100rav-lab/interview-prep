# Web Development (Servlets & JSP)

> 💡 **Interviewer Tip:** While modern Java backends use Spring MVC, understanding Servlets & JSP shows you grasp the underlying plumbing that frameworks build on. Interviewers use these questions to gauge depth of knowledge.

---

## Table of Contents
1. [Servlet Lifecycle](#servlet-lifecycle)
2. [HttpServletRequest & HttpServletResponse](#httpservletrequest--httpservletresponse)
3. [Filters & Listeners](#filters--listeners)
4. [Session Management](#session-management)
5. [RequestDispatcher](#requestdispatcher)
6. [Servlet Container & Tomcat](#servlet-container--tomcat)
7. [JSP Basics](#jsp-basics)

---

## Servlet Lifecycle

### 🟢 Q1. What is a Servlet and what is its role in web development?

<details><summary>Click to reveal answer</summary>

A **Servlet** is a Java class that handles HTTP requests and produces responses. It runs inside a **Servlet Container** (e.g., Tomcat, Jetty) and follows a well-defined lifecycle.

```java
import jakarta.servlet.http.*;
import jakarta.servlet.*;

public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.setContentType("text/html");
        resp.getWriter().println("<h1>Hello, World!</h1>");
    }
}
```

**Servlet registration (web.xml):**
```xml
<servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.example.HelloServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>HelloServlet</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```

**Or annotation-based (Servlet 3.0+):**
```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet { ... }
```

Role: Servlets serve as the **Controller** layer in MVC — receiving requests, calling service/business logic, and writing responses.

</details>

---

### 🟢 Q2. Describe the Servlet lifecycle in detail.

<details><summary>Click to reveal answer</summary>

The Servlet lifecycle is managed by the container and consists of three phases:

```
Client Request
     ↓
Servlet Container
     ↓
[First Request Only]
  1. load class
  2. instantiate
  3. init()
     ↓
[Every Request]
  4. service()
     ↓ (dispatches to)
  doGet() / doPost() / doPut() / doDelete() etc.
     ↓
[Shutdown]
  5. destroy()
```

```java
@WebServlet("/lifecycle")
public class LifecycleServlet extends HttpServlet {

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        // Called ONCE: initialize resources (DB connections, caches)
        System.out.println("Servlet initialized");
        String dbUrl = config.getInitParameter("dbUrl");
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // Called for EVERY request; default implementation routes to doGet/doPost
        super.service(req, resp);
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.getWriter().println("Handling GET");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.getWriter().println("Handling POST");
    }

    @Override
    public void destroy() {
        // Called ONCE: release resources (close connections, flush data)
        System.out.println("Servlet destroyed");
    }
}
```

**Key facts:**
- Servlet is instantiated **once** (singleton per container)
- `init()` is called **once** after instantiation
- `service()` is called for **every request** (potentially by multiple threads)
- `destroy()` is called **once** before servlet is removed

🚨 **Common Mistake:** Storing request-specific state in instance variables — Servlets are NOT thread-safe by default!

</details>

---

### 🟡 Q3. Is a Servlet thread-safe? How do you handle concurrency?

<details><summary>Click to reveal answer</summary>

**No**, Servlets are NOT thread-safe by default. The container uses a **single servlet instance** to serve multiple concurrent requests across multiple threads.

```java
// ❌ NOT THREAD-SAFE — instance variable shared across threads
@WebServlet("/counter")
public class UnsafeServlet extends HttpServlet {
    private int counter = 0; // Shared mutable state!

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        counter++; // Race condition!
        resp.getWriter().println("Count: " + counter);
    }
}

// ✅ THREAD-SAFE — use local variables or thread-safe types
@WebServlet("/counter")
public class SafeServlet extends HttpServlet {
    private final AtomicInteger counter = new AtomicInteger(0); // Thread-safe

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        int current = counter.incrementAndGet();
        resp.getWriter().println("Count: " + current);
    }
}
```

**Best practices for thread safety:**
1. Use **local variables** for request-specific state
2. Use **thread-safe collections** (`ConcurrentHashMap`, `AtomicInteger`)
3. Make instance variables **immutable** (final, read-only)
4. Use **synchronization** only for critical sections (avoid if possible)

> 💡 **Interviewer Tip:** The old `SingleThreadModel` interface made each thread get its own instance, but it was deprecated in Servlet 2.4 due to performance issues.

</details>

---

### 🟢 Q4. What is the `init-param` vs `context-param` difference?

<details><summary>Click to reveal answer</summary>

- **`init-param`** – Specific to a single servlet, accessed via `ServletConfig`
- **`context-param`** – Available to the entire web application, accessed via `ServletContext`

```xml
<!-- web.xml -->
<context-param>
    <param-name>appName</param-name>
    <param-value>MyApp</param-value>
</context-param>

<servlet>
    <servlet-name>MyServlet</servlet-name>
    <servlet-class>com.example.MyServlet</servlet-class>
    <init-param>
        <param-name>maxConnections</param-name>
        <param-value>10</param-value>
    </init-param>
</servlet>
```

```java
@WebServlet(value = "/config",
    initParams = @WebInitParam(name = "timeout", value = "30"))
public class ConfigServlet extends HttpServlet {

    @Override
    public void init() throws ServletException {
        // Access servlet-specific init param
        String timeout = getServletConfig().getInitParameter("timeout");

        // Access application-wide context param
        String appName = getServletContext().getInitParameter("appName");

        System.out.println("Timeout: " + timeout + ", App: " + appName);
    }
}
```

</details>

---

## HttpServletRequest & HttpServletResponse

### 🟢 Q5. What are the key methods of `HttpServletRequest`?

<details><summary>Click to reveal answer</summary>

```java
@Override
protected void doPost(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

    // URL components
    String method     = req.getMethod();          // "POST"
    String requestURI = req.getRequestURI();      // "/app/users"
    String contextPath = req.getContextPath();    // "/app"
    String servletPath = req.getServletPath();    // "/users"
    String queryString = req.getQueryString();    // "page=2&sort=name"

    // Parameters (query string or form body)
    String name  = req.getParameter("name");          // Single value
    String[] ids = req.getParameterValues("ids");     // Multiple values
    Map<String, String[]> params = req.getParameterMap();

    // Headers
    String contentType = req.getContentType();
    String authHeader  = req.getHeader("Authorization");
    Enumeration<String> headerNames = req.getHeaderNames();

    // Client info
    String remoteAddr = req.getRemoteAddr(); // Client IP
    String remoteHost = req.getRemoteHost();

    // Session
    HttpSession session = req.getSession();         // Create if absent
    HttpSession existing = req.getSession(false);   // Return null if absent

    // Attributes (request-scoped objects)
    req.setAttribute("user", currentUser);
    User user = (User) req.getAttribute("user");

    // Body reading
    BufferedReader reader = req.getReader(); // Text body
    ServletInputStream stream = req.getInputStream(); // Binary body
}
```

</details>

---

### 🟢 Q6. What are the key methods of `HttpServletResponse`?

<details><summary>Click to reveal answer</summary>

```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

    // Status code
    resp.setStatus(HttpServletResponse.SC_OK);         // 200
    resp.setStatus(HttpServletResponse.SC_NOT_FOUND);  // 404

    // Content type
    resp.setContentType("application/json");
    resp.setCharacterEncoding("UTF-8");

    // Headers
    resp.setHeader("Cache-Control", "no-cache, no-store");
    resp.addHeader("X-Custom-Header", "value");
    resp.setIntHeader("Content-Length", 1024);

    // Response body
    PrintWriter writer = resp.getWriter();      // Text
    writer.println("{\"message\":\"OK\"}");

    // Or binary
    ServletOutputStream out = resp.getOutputStream();
    out.write(imageBytes);

    // Redirect (302 by default)
    resp.sendRedirect("/login");
    resp.sendRedirect("https://external.com");

    // Error response
    resp.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied");

    // Cookies
    Cookie cookie = new Cookie("sessionId", "abc123");
    cookie.setMaxAge(3600);
    cookie.setHttpOnly(true);
    resp.addCookie(cookie);
}
```

🚨 **Common Mistake:** Calling `getWriter()` and `getOutputStream()` on the same response — this throws `IllegalStateException`. Use one or the other.

</details>

---

### 🟡 Q7. How do you handle file uploads in a Servlet?

<details><summary>Click to reveal answer</summary>

```java
@WebServlet("/upload")
@MultipartConfig(
    maxFileSize = 5 * 1024 * 1024,    // 5MB per file
    maxRequestSize = 20 * 1024 * 1024, // 20MB total
    fileSizeThreshold = 1024 * 1024    // 1MB before writing to disk
)
public class FileUploadServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        // Get text field
        String description = req.getParameter("description");

        // Get uploaded file part
        Part filePart = req.getPart("file");
        String fileName = filePart.getSubmittedFileName();
        long fileSize = filePart.getSize();

        // Read file content
        try (InputStream inputStream = filePart.getInputStream()) {
            byte[] content = inputStream.readAllBytes();
            // Process or save file
            Files.write(Path.of("/uploads/" + fileName), content);
        }

        // Multiple files
        Collection<Part> fileParts = req.getParts();
        for (Part part : fileParts) {
            if (part.getContentType() != null) { // Is a file
                part.write("/uploads/" + part.getSubmittedFileName());
            }
        }

        resp.sendRedirect("/success");
    }
}
```

</details>

---

## Filters & Listeners

### 🟡 Q8. What is a Servlet Filter and how does it work?

<details><summary>Click to reveal answer</summary>

A **Filter** intercepts requests and responses, performing pre/post-processing. Filters form a chain — each filter calls `chain.doFilter()` to pass control to the next filter or the servlet.

```java
@WebFilter("/*")  // Applied to all requests
public class LoggingFilter implements Filter {

    @Override
    public void init(FilterConfig config) throws ServletException {
        System.out.println("LoggingFilter initialized");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpReq = (HttpServletRequest) request;
        long startTime = System.currentTimeMillis();

        System.out.printf("[REQUEST] %s %s%n",
            httpReq.getMethod(), httpReq.getRequestURI());

        // Pass request to next filter or servlet
        chain.doFilter(request, response);

        // Post-processing (after response)
        long duration = System.currentTimeMillis() - startTime;
        System.out.printf("[RESPONSE] %dms%n", duration);
    }

    @Override
    public void destroy() {
        System.out.println("LoggingFilter destroyed");
    }
}
```

**Authentication Filter example:**
```java
@WebFilter("/api/*")
public class AuthFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;

        String token = request.getHeader("Authorization");
        if (token == null || !tokenService.isValid(token)) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid token");
            return; // Stop the chain
        }

        chain.doFilter(req, resp);
    }
}
```

**Filter ordering in web.xml:**
```xml
<filter-mapping>
    <filter-name>LoggingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
<filter-mapping>
    <filter-name>AuthFilter</filter-name>
    <url-pattern>/api/*</url-pattern>
</filter-mapping>
```

</details>

---

### 🟡 Q9. What are Servlet Listeners and what types exist?

<details><summary>Click to reveal answer</summary>

Listeners respond to lifecycle events in the servlet container. Main types:

| Listener Interface | Event Scope | Use Case |
|---|---|---|
| `ServletContextListener` | Application start/stop | Initialize app-wide resources |
| `ServletContextAttributeListener` | Context attribute change | Monitoring |
| `HttpSessionListener` | Session created/destroyed | Track active sessions |
| `HttpSessionAttributeListener` | Session attribute change | Audit trail |
| `ServletRequestListener` | Request start/end | Logging, MDC setup |
| `HttpSessionBindingListener` | Object bound to session | Cleanup when session data changes |

```java
@WebListener
public class AppStartupListener implements ServletContextListener {

    private DatabaseConnectionPool pool;

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        // Application is starting
        pool = DatabaseConnectionPool.create();
        sce.getServletContext().setAttribute("dbPool", pool);
        System.out.println("Application started, DB pool initialized");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        // Application is shutting down
        if (pool != null) pool.close();
        System.out.println("Application stopped, DB pool closed");
    }
}

@WebListener
public class SessionTrackingListener implements HttpSessionListener {

    private static final AtomicInteger activeSessions = new AtomicInteger(0);

    @Override
    public void sessionCreated(HttpSessionEvent se) {
        int count = activeSessions.incrementAndGet();
        System.out.println("Session created. Active sessions: " + count);
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        int count = activeSessions.decrementAndGet();
        System.out.println("Session destroyed. Active sessions: " + count);
    }

    public static int getActiveSessions() {
        return activeSessions.get();
    }
}
```

</details>

---

## Session Management

### 🟢 Q10. What is `HttpSession` and how does it work?

<details><summary>Click to reveal answer</summary>

`HttpSession` stores user-specific state on the server between requests. The server assigns each session a unique ID, sent to the client via a `JSESSIONID` cookie (or URL rewriting).

```java
@WebServlet("/login")
public class LoginServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        String username = req.getParameter("username");
        String password = req.getParameter("password");

        if (authService.authenticate(username, password)) {
            // Create session
            HttpSession session = req.getSession(true);
            session.setAttribute("username", username);
            session.setAttribute("role", "USER");
            session.setMaxInactiveInterval(30 * 60); // 30 minutes

            resp.sendRedirect("/dashboard");
        } else {
            resp.sendError(401, "Invalid credentials");
        }
    }
}

@WebServlet("/dashboard")
public class DashboardServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        HttpSession session = req.getSession(false); // Don't create new
        if (session == null) {
            resp.sendRedirect("/login");
            return;
        }

        String username = (String) session.getAttribute("username");
        String sessionId = session.getId();
        long creationTime = session.getCreationTime();
        long lastAccessed = session.getLastAccessedTime();

        resp.getWriter().println("Welcome, " + username);
    }
}

@WebServlet("/logout")
public class LogoutServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        HttpSession session = req.getSession(false);
        if (session != null) {
            session.invalidate(); // Destroy the session
        }
        resp.sendRedirect("/login");
    }
}
```

> 💡 **Interviewer Tip:** Ask about session clustering in distributed environments — sessions stored in-memory won't work across multiple server instances. Solutions: sticky sessions, distributed cache (Redis), or JWT.

</details>

---

### 🟡 Q11. What are Cookies and how do they differ from Session?

<details><summary>Click to reveal answer</summary>

| Feature | Session | Cookie |
|---|---|---|
| Storage | Server-side | Client-side (browser) |
| Security | More secure | Less secure (can be tampered) |
| Size limit | No fixed limit | ~4KB |
| Expiry | Server timeout | Explicit expiry date |
| Accessibility | Server only | Client and server |

```java
// Creating and reading cookies
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

    // Create cookie
    Cookie userPref = new Cookie("theme", "dark");
    userPref.setMaxAge(7 * 24 * 60 * 60); // 7 days
    userPref.setPath("/");                  // Available to all paths
    userPref.setHttpOnly(true);             // Not accessible via JavaScript
    userPref.setSecure(true);               // HTTPS only
    // userPref.setSameSite("Strict");      // CSRF protection (Servlet 6.0+)
    resp.addCookie(userPref);

    // Read cookies
    Cookie[] cookies = req.getCookies();
    if (cookies != null) {
        for (Cookie cookie : cookies) {
            if ("theme".equals(cookie.getName())) {
                String theme = cookie.getValue();
                System.out.println("Theme: " + theme);
            }
        }
    }

    // Delete cookie (set maxAge to 0)
    Cookie deleteCookie = new Cookie("theme", "");
    deleteCookie.setMaxAge(0);
    deleteCookie.setPath("/");
    resp.addCookie(deleteCookie);
}
```

✅ **Best Practice:** Always set `HttpOnly` and `Secure` flags on session cookies to prevent XSS and man-in-the-middle attacks.

</details>

---

### 🟡 Q12. What is URL rewriting and why is it a fallback for sessions?

<details><summary>Click to reveal answer</summary>

URL rewriting appends the session ID to every URL when cookies are disabled. The session ID is embedded as a query parameter (`jsessionid`).

```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

    HttpSession session = req.getSession();

    // encodeURL adds ;jsessionid=... if cookies are disabled
    String url = resp.encodeURL("/products");
    // Becomes: /products;jsessionid=ABC123

    String redirectUrl = resp.encodeRedirectURL("/login");
    // Becomes: /login;jsessionid=ABC123

    PrintWriter out = resp.getWriter();
    out.println("<a href='" + url + "'>Products</a>");
}
```

🚨 **Security Risk:** URL rewriting is insecure because:
1. Session ID appears in browser history
2. Session ID is logged in server access logs
3. Session ID can be shared if users copy/paste URLs

✅ **Best Practice:** Disable URL rewriting in modern applications. Use cookies with `HttpOnly` and `Secure` flags, or JWT tokens in Authorization headers.

</details>

---

## RequestDispatcher

### 🟡 Q13. What is `RequestDispatcher` and what is the difference between `forward()` and `sendRedirect()`?

<details><summary>Click to reveal answer</summary>

`RequestDispatcher` routes requests internally within the server.

| Feature | `forward()` | `sendRedirect()` |
|---|---|---|
| Client-visible | No — same request | Yes — new request (302) |
| URL change | No — URL stays same | Yes — URL changes |
| Request attributes | Preserved | Lost (new request) |
| Server round trips | 1 | 2 |
| POST data | Preserved | Lost (becomes GET) |
| Scope | Same web application | Any URL (even external) |

```java
@WebServlet("/process")
public class ProcessServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        String action = req.getParameter("action");

        if ("view".equals(action)) {
            // FORWARD — internal routing, URL stays /process
            req.setAttribute("data", fetchData());
            RequestDispatcher dispatcher = req.getRequestDispatcher("/WEB-INF/view.jsp");
            dispatcher.forward(req, resp);

        } else if ("redirect".equals(action)) {
            // REDIRECT — client makes new GET request, URL changes to /home
            resp.sendRedirect("/home");

        } else if ("include".equals(action)) {
            // INCLUDE — includes another resource's output in current response
            RequestDispatcher rd = req.getRequestDispatcher("/header.jsp");
            rd.include(req, resp);
            resp.getWriter().println("<p>Main content</p>");
        }
    }
}
```

> 💡 **Interviewer Tip:** The **Post-Redirect-Get (PRG) pattern** uses `sendRedirect()` after a POST to prevent form resubmission on browser refresh.

</details>

---

### 🟢 Q14. What is the Post-Redirect-Get (PRG) pattern?

<details><summary>Click to reveal answer</summary>

PRG solves the duplicate form submission problem: if a user submits a form (POST) and then hits F5 (refresh), the browser re-sends the POST request.

**Solution:** After processing POST, redirect (302) to a GET endpoint.

```java
@WebServlet("/orders")
public class OrderServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        // Process the form submission
        String productId = req.getParameter("productId");
        int quantity = Integer.parseInt(req.getParameter("quantity"));

        Order order = orderService.placeOrder(productId, quantity);

        // PRG: Store result in session, then REDIRECT to GET
        req.getSession().setAttribute("lastOrder", order);
        resp.sendRedirect("/orders/confirmation"); // New GET request
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        // GET: Safe to refresh — just reads from session
        HttpSession session = req.getSession(false);
        Order order = session != null ?
            (Order) session.getAttribute("lastOrder") : null;

        req.setAttribute("order", order);
        req.getRequestDispatcher("/WEB-INF/confirmation.jsp")
           .forward(req, resp);
    }
}
```

</details>

---

## Servlet Container & Tomcat

### 🟢 Q15. What is a Servlet Container?

<details><summary>Click to reveal answer</summary>

A **Servlet Container** (also called a Web Container) is a runtime environment that manages Servlet lifecycle, URL mapping, threading, and HTTP communication. Examples: Apache Tomcat, Jetty, Undertow.

Responsibilities:
1. **Lifecycle management** – Loads, initializes, and destroys servlets
2. **URL mapping** – Routes requests to the correct servlet
3. **Thread management** – Creates threads to handle concurrent requests
4. **Security** – Authentication realms, SSL termination
5. **Session management** – Creates and expires `HttpSession`
6. **JSP compilation** – Compiles `.jsp` to servlet classes

**Tomcat directory structure:**
```
tomcat/
├── bin/          ← startup.sh, shutdown.sh
├── conf/
│   ├── server.xml   ← connectors, hosts, port config
│   ├── web.xml      ← global servlet configuration
│   └── context.xml  ← datasource configuration
├── lib/          ← servlet-api.jar, jsp-api.jar
├── logs/
├── webapps/
│   └── myapp/    ← your WAR deployed here
│       ├── WEB-INF/
│       │   ├── web.xml
│       │   ├── classes/
│       │   └── lib/
│       └── index.jsp
└── work/         ← compiled JSP classes
```

</details>

---

### 🟡 Q16. What is `web.xml` and what can you configure in it?

<details><summary>Click to reveal answer</summary>

`web.xml` is the **Deployment Descriptor** — the configuration file for a Java web application.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" version="6.0">

    <!-- Application display name -->
    <display-name>My Web App</display-name>

    <!-- Context parameters (app-wide) -->
    <context-param>
        <param-name>appVersion</param-name>
        <param-value>1.0.0</param-value>
    </context-param>

    <!-- Servlet declaration -->
    <servlet>
        <servlet-name>FrontController</servlet-name>
        <servlet-class>com.example.FrontControllerServlet</servlet-class>
        <init-param>
            <param-name>configFile</param-name>
            <param-value>/WEB-INF/mvc-config.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup> <!-- Eager init, lower = earlier -->
    </servlet>
    <servlet-mapping>
        <servlet-name>FrontController</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <!-- Filter -->
    <filter>
        <filter-name>CharsetFilter</filter-name>
        <filter-class>com.example.CharsetFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>CharsetFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

    <!-- Session config -->
    <session-config>
        <session-timeout>30</session-timeout> <!-- minutes -->
        <cookie-config>
            <http-only>true</http-only>
            <secure>true</secure>
        </cookie-config>
    </session-config>

    <!-- Error pages -->
    <error-page>
        <error-code>404</error-code>
        <location>/WEB-INF/error/404.jsp</location>
    </error-page>
    <error-page>
        <exception-type>java.lang.Exception</exception-type>
        <location>/WEB-INF/error/generic.jsp</location>
    </error-page>

    <!-- Welcome files -->
    <welcome-file-list>
        <welcome-file>index.html</welcome-file>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```

</details>

---

### 🟡 Q17. What does `load-on-startup` do?

<details><summary>Click to reveal answer</summary>

`load-on-startup` controls **eager initialization** of servlets.

- **Not set or negative** – Servlet is initialized lazily (on first request)
- **0 or positive integer** – Servlet is initialized at application startup; lower numbers initialize first

```xml
<servlet>
    <servlet-name>FrontController</servlet-name>
    <servlet-class>com.example.FrontControllerServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet>
    <servlet-name>DataInitServlet</servlet-name>
    <servlet-class>com.example.DataInitServlet</servlet-class>
    <load-on-startup>0</load-on-startup> <!-- Initializes before FrontController -->
</servlet>
```

**Use cases for eager initialization:**
- Pre-warming caches
- Establishing database connection pools
- Validating configuration on startup (fail fast)

```java
@WebServlet(value = "/app", loadOnStartup = 1)
public class AppInitServlet extends HttpServlet {

    @Override
    public void init() throws ServletException {
        // Runs at startup — fail fast if config is invalid
        String apiKey = getServletContext().getInitParameter("apiKey");
        if (apiKey == null || apiKey.isBlank()) {
            throw new ServletException("API key is required in context-param");
        }
        System.out.println("App initialized successfully");
    }
}
```

</details>

---

### 🟡 Q18. What is the difference between WAR and JAR in web deployment?

<details><summary>Click to reveal answer</summary>

| Format | Stands For | Content | Deployment |
|---|---|---|---|
| `.jar` | Java Archive | Classes, resources, `META-INF/MANIFEST.MF` | Executable or library |
| `.war` | Web Application Archive | Classes, JSPs, static resources, `WEB-INF/web.xml` | Servlet container |
| `.ear` | Enterprise Archive | Multiple WARs and JARs | Full Java EE server |

**WAR structure:**
```
myapp.war
├── index.html
├── css/style.css
├── WEB-INF/
│   ├── web.xml
│   ├── classes/
│   │   └── com/example/MyServlet.class
│   └── lib/
│       └── dependency.jar
```

**Modern approach — Executable JAR (Spring Boot):**
```
myapp.jar
├── BOOT-INF/
│   ├── classes/   ← your application classes
│   └── lib/       ← all dependencies including embedded Tomcat
├── META-INF/
└── org/springframework/boot/loader/
```

Spring Boot embeds Tomcat inside the JAR, so no separate container is needed:
```bash
java -jar myapp.jar  # Starts embedded Tomcat on port 8080
```

</details>

---

## JSP Basics

### 🟢 Q19. What is JSP and how does it work?

<details><summary>Click to reveal answer</summary>

**JavaServer Pages (JSP)** is a technology that lets you embed Java code in HTML templates. The container compiles JSPs into Servlet classes.

**JSP Lifecycle:**
1. First request: JSP is translated to Java Servlet class
2. Servlet is compiled to bytecode
3. Compiled servlet handles all subsequent requests

```jsp
<%-- hello.jsp --%>
<%@ page language="java" contentType="text/html; charset=UTF-8" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>

<!DOCTYPE html>
<html>
<head><title>Hello JSP</title></head>
<body>

    <%-- Scriptlet (Java code - avoid in modern JSP) --%>
    <%
        String name = (String) request.getAttribute("name");
        if (name == null) name = "World";
    %>

    <%-- Expression (outputs value) --%>
    <h1>Hello, <%= name %>!</h1>

    <%-- EL Expression (preferred) --%>
    <h1>Hello, ${name}!</h1>

    <%-- JSTL tag --%>
    <c:if test="${not empty users}">
        <ul>
            <c:forEach var="user" items="${users}">
                <li>${user.name} - ${user.email}</li>
            </c:forEach>
        </ul>
    </c:if>

    <%-- Declaration (fields and methods) --%>
    <%!
        private String formatName(String name) {
            return name.toUpperCase();
        }
    %>

    <%-- Directive --%>
    <%@ include file="/WEB-INF/footer.jsp" %>

</body>
</html>
```

</details>

---

### 🟢 Q20. What are JSP implicit objects?

<details><summary>Click to reveal answer</summary>

JSP provides 9 implicit objects available in scriptlets and EL:

| Object | Type | Description |
|---|---|---|
| `request` | `HttpServletRequest` | Current request |
| `response` | `HttpServletResponse` | Current response |
| `session` | `HttpSession` | User session |
| `application` | `ServletContext` | Application context |
| `out` | `JspWriter` | Output stream |
| `config` | `ServletConfig` | Servlet config |
| `pageContext` | `PageContext` | Page-level context |
| `page` | `Object` | `this` — current servlet instance |
| `exception` | `Throwable` | Error page only |

```jsp
<%-- Accessing implicit objects in scriptlets --%>
<%
    String username = (String) session.getAttribute("username");
    String appName = application.getInitParameter("appName");
    out.println("Welcome, " + username);
%>

<%-- Accessing in EL (preferred) --%>
<p>Username: ${sessionScope.username}</p>
<p>Request param: ${param.id}</p>
<p>Cookie: ${cookie.theme.value}</p>
<p>Header: ${header['User-Agent']}</p>
```

**EL scope resolution order:** page → request → session → application

</details>

---

### 🟡 Q21. What is JSTL and why should you prefer it over scriptlets?

<details><summary>Click to reveal answer</summary>

**JSTL (JavaServer Pages Standard Tag Library)** provides custom tags for common JSP operations, replacing scriptlets with clean markup.

```jsp
<%-- AVOID: Scriptlets (Java code in JSP) --%>
<%
    List<User> users = (List<User>) request.getAttribute("users");
    if (users != null && !users.isEmpty()) {
        for (User user : users) {
            out.println("<li>" + user.getName() + "</li>");
        }
    }
%>

<%-- PREFER: JSTL + EL --%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<%@ taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>

<c:if test="${not empty users}">
    <c:forEach var="user" items="${users}" varStatus="status">
        <tr class="${status.even ? 'even' : 'odd'}">
            <td>${status.index + 1}</td>
            <td>${fn:escapeXml(user.name)}</td>
            <td>
                <fmt:formatDate value="${user.createdAt}"
                                pattern="yyyy-MM-dd"/>
            </td>
        </tr>
    </c:forEach>
</c:if>

<c:choose>
    <c:when test="${user.role == 'ADMIN'}">
        <a href="/admin">Admin Panel</a>
    </c:when>
    <c:otherwise>
        <p>Regular user</p>
    </c:otherwise>
</c:choose>
```

**Why avoid scriptlets:**
- Mixes Java and HTML (violation of SoC)
- Hard to test and maintain
- Difficult to read for front-end developers
- Encourages putting business logic in views

</details>

---

### 🟡 Q22. What is the difference between `<%@ include %>` and `<jsp:include>`?

<details><summary>Click to reveal answer</summary>

| Feature | `<%@ include file="..." %>` | `<jsp:include page="..." />` |
|---|---|---|
| Type | Directive (static) | Action (dynamic) |
| When processed | Translation time | Request time |
| Included content | Merged into JSP source | Separate servlet call |
| File changes | Requires recompile | Picked up dynamically |
| Variable sharing | Yes (same scope) | No (separate request) |

```jsp
<%-- Static include — merged at translation time --%>
<%-- Changes to header.jsp require redeployment --%>
<%@ include file="/WEB-INF/includes/header.jsp" %>

<%-- Dynamic include — called at runtime --%>
<%-- Changes to footer.jsp are immediately visible --%>
<jsp:include page="/WEB-INF/includes/footer.jsp">
    <jsp:param name="year" value="2024"/>
</jsp:include>
```

✅ **Best Practice:** Use `<%@ include %>` for static content (header, footer) that rarely changes. Use `<jsp:include>` for dynamic content that may vary per request.

</details>

---

### 🟢 Q23. What is the MVC pattern in a Servlet/JSP application?

<details><summary>Click to reveal answer</summary>

In Servlet/JSP MVC:
- **Model** – Java beans / service layer (business logic)
- **View** – JSP pages (presentation)
- **Controller** – Servlet (routes requests, calls model, sets attributes)

```java
// CONTROLLER: Servlet
@WebServlet("/users")
public class UserController extends HttpServlet {

    private final UserService userService = new UserServiceImpl();

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        // 1. Controller receives request
        String pageStr = req.getParameter("page");
        int page = pageStr != null ? Integer.parseInt(pageStr) : 1;

        // 2. Call Model (service layer)
        List<User> users = userService.findAll(page, 10);
        long totalCount = userService.count();

        // 3. Set attributes for View
        req.setAttribute("users", users);
        req.setAttribute("totalCount", totalCount);
        req.setAttribute("currentPage", page);

        // 4. Forward to View (JSP)
        req.getRequestDispatcher("/WEB-INF/views/users.jsp")
           .forward(req, resp);
    }
}
```

```jsp
<%-- VIEW: users.jsp --%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<html>
<body>
    <h2>Users (Total: ${totalCount})</h2>
    <c:forEach var="user" items="${users}">
        <p>${user.name} - ${user.email}</p>
    </c:forEach>
    <a href="/users?page=${currentPage + 1}">Next Page</a>
</body>
</html>
```

```java
// MODEL: Service and Entity
public class User {
    private Long id;
    private String name;
    private String email;
    // getters, setters
}
```

</details>

---

### 🟡 Q24. What is a Front Controller pattern and how does it apply to servlets?

<details><summary>Click to reveal answer</summary>

The **Front Controller** pattern routes ALL requests through a single entry-point servlet, which then delegates to specific handlers. Spring MVC's `DispatcherServlet` implements this pattern.

```java
@WebServlet("/")
public class FrontControllerServlet extends HttpServlet {

    private Map<String, RequestHandler> handlers = new HashMap<>();

    @Override
    public void init() throws ServletException {
        // Register handlers
        handlers.put("GET:/users",    new GetUsersHandler());
        handlers.put("GET:/users/*",  new GetUserHandler());
        handlers.put("POST:/users",   new CreateUserHandler());
        handlers.put("DELETE:/users", new DeleteUserHandler());
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        String method = req.getMethod();
        String path = req.getPathInfo() != null ?
            req.getPathInfo() : req.getServletPath();

        String key = method + ":" + path;
        RequestHandler handler = handlers.get(key);

        if (handler == null) {
            resp.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        handler.handle(req, resp);
    }
}

interface RequestHandler {
    void handle(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException;
}
```

**Benefits:**
- Centralized request processing (security, logging, auth)
- Consistent error handling
- Single point for cross-cutting concerns

</details>

---

### 🟡 Q25. How do you prevent Cross-Site Scripting (XSS) in JSPs?

<details><summary>Click to reveal answer</summary>

XSS occurs when user-controlled data is rendered as HTML without escaping.

```jsp
<%-- ❌ VULNERABLE: Directly outputs user input as HTML --%>
<p>Welcome, ${param.username}!</p>

<%-- If username = "<script>alert('XSS')</script>" this runs! --%>
```

**Prevention techniques:**

```jsp
<%-- ✅ SAFE: JSTL fn:escapeXml() --%>
<%@ taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>
<p>Welcome, ${fn:escapeXml(param.username)}!</p>

<%-- ✅ SAFE: c:out with default escapeXml=true --%>
<p>Welcome, <c:out value="${param.username}"/></p>
```

```java
// ✅ Server-side: Apache Commons Text
import org.apache.commons.text.StringEscapeUtils;

String safe = StringEscapeUtils.escapeHtml4(userInput);

// ✅ Or with OWASP Java Encoder
import org.owasp.encoder.Encode;
String safe = Encode.forHtml(userInput);
String safeAttr = Encode.forHtmlAttribute(userInput);
String safeJs = Encode.forJavaScript(userInput);
```

**Content Security Policy (CSP) Header:**
```java
// Filter to add CSP header
resp.setHeader("Content-Security-Policy",
    "default-src 'self'; script-src 'self' 'nonce-" + nonce + "'");
```

</details>

---

### 🔴 Q26. Explain how session fixation attacks work and how to prevent them.

<details><summary>Click to reveal answer</summary>

**Session Fixation:** An attacker forces a known session ID on the victim's browser. When the victim logs in, the attacker's known session ID is now authenticated.

**Attack flow:**
1. Attacker obtains a valid session ID (e.g., from `JSESSIONID` URL parameter)
2. Tricks victim into using that session ID (via URL)
3. Victim logs in — session is now privileged
4. Attacker uses same session ID to access victim's account

**Prevention:** Invalidate old session and create new one upon login:

```java
@WebServlet("/login")
public class LoginServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        String username = req.getParameter("username");
        String password = req.getParameter("password");

        if (authService.authenticate(username, password)) {

            // ✅ CRITICAL: Invalidate old session and create new one
            HttpSession oldSession = req.getSession(false);
            Map<String, Object> attributes = new HashMap<>();

            if (oldSession != null) {
                // Copy needed attributes
                Enumeration<String> names = oldSession.getAttributeNames();
                while (names.hasMoreElements()) {
                    String name = names.nextElement();
                    attributes.put(name, oldSession.getAttribute(name));
                }
                oldSession.invalidate(); // Destroy old session (and old ID)
            }

            // Create new session with new ID
            HttpSession newSession = req.getSession(true);
            attributes.forEach(newSession::setAttribute);
            newSession.setAttribute("username", username);
            newSession.setAttribute("authenticated", true);

            resp.sendRedirect("/dashboard");
        }
    }
}
```

</details>

---

*End of Web Development (Servlets & JSP)*
