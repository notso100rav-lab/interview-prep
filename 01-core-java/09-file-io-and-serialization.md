# File I/O & Serialization

> 💡 **Interviewer Tip:** File I/O questions test your understanding of Java's layered I/O architecture. Interviewers look for knowledge of when to choose streams vs readers/writers, NIO vs classic I/O, and the subtleties of Java serialization (especially `serialVersionUID` and the `transient` keyword).

---

## Table of Contents
1. [InputStream & OutputStream](#inputstream--outputstream)
2. [Reader & Writer](#reader--writer)
3. [NIO (Path, Files, Channel, Buffer)](#nio-path-files-channel-buffer)
4. [Serialization & Deserialization](#serialization--deserialization)
5. [Theory Questions](#theory-questions)
6. [Scenario-Based Questions](#scenario-based-questions)

---

## Quick Reference

| Category | Classes | Use Case |
|---|---|---|
| Byte Streams | `InputStream`, `OutputStream` | Binary data (images, audio) |
| Character Streams | `Reader`, `Writer` | Text data with encoding |
| Buffered | `BufferedInputStream`, `BufferedReader` | Performance via buffering |
| NIO Channels | `FileChannel`, `SocketChannel` | Non-blocking, high performance |
| NIO Path/Files | `Path`, `Files` | Modern file operations |
| Serialization | `ObjectInputStream/OutputStream` | Object persistence |

---

## Theory Questions

### InputStream & OutputStream

**Q1.** 🟢 What is the difference between `InputStream`/`OutputStream` and `Reader`/`Writer`?

<details><summary>Click to reveal answer</summary>

`InputStream`/`OutputStream` are **byte-oriented** streams — they read and write raw bytes (0–255). They are used for binary data like images, audio files, or network protocols.

`Reader`/`Writer` are **character-oriented** streams — they handle Unicode characters with character encoding/decoding. They are suitable for text data.

```java
// Byte stream - binary data
try (FileInputStream fis = new FileInputStream("image.png");
     FileOutputStream fos = new FileOutputStream("copy.png")) {
    byte[] buffer = new byte[4096];
    int bytesRead;
    while ((bytesRead = fis.read(buffer)) != -1) {
        fos.write(buffer, 0, bytesRead);
    }
}

// Character stream - text data with encoding
try (FileReader fr = new FileReader("text.txt", StandardCharsets.UTF_8);
     FileWriter fw = new FileWriter("output.txt", StandardCharsets.UTF_8)) {
    char[] buffer = new char[1024];
    int charsRead;
    while ((charsRead = fr.read(buffer)) != -1) {
        fw.write(buffer, 0, charsRead);
    }
}
```

> **Rule of thumb:** Use byte streams for binary data; use character streams for text to avoid character encoding issues.

</details>

---

**Q2.** 🟢 What does `read()` return when end-of-stream is reached?

<details><summary>Click to reveal answer</summary>

`read()` returns **`-1`** when the end of the stream is reached. This is why the idiomatic loop is:

```java
int byteRead;
while ((byteRead = inputStream.read()) != -1) {
    // process byteRead (0–255)
}
```

🚨 **Common Mistake:** Storing the result in a `byte` variable instead of `int`. A `byte` can be `-1` (0xFF truncated), causing premature loop exit:

```java
// WRONG - byte b can never hold -1 to signal EOF correctly
byte b;
while ((b = (byte) inputStream.read()) != -1) { ... } // Bug!

// CORRECT
int b;
while ((b = inputStream.read()) != -1) { ... }
```

</details>

---

**Q3.** 🟡 What is the purpose of `BufferedInputStream`? How does it improve performance?

<details><summary>Click to reveal answer</summary>

`BufferedInputStream` wraps another `InputStream` and maintains an **internal byte array buffer** (default 8192 bytes). Instead of making a system call to the OS for every single byte, it reads a large chunk into its buffer at once. Subsequent `read()` calls are served from the buffer, drastically reducing system call overhead.

```java
// Without buffering: potentially millions of system calls
try (FileInputStream fis = new FileInputStream("large.dat")) {
    int b;
    while ((b = fis.read()) != -1) { } // each read() = 1 syscall
}

// With buffering: reads 8KB chunks, very few syscalls
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("large.dat"))) {
    int b;
    while ((b = bis.read()) != -1) { } // each read() served from buffer
}
```

| | Without Buffer | With Buffer (8KB) |
|---|---|---|
| File size 1MB | ~1,048,576 syscalls | ~128 syscalls |
| Relative speed | 1x (baseline) | ~100–1000x faster |

✅ **Best Practice:** Always wrap file streams with buffered wrappers unless you're already reading in large chunks.

</details>

---

**Q4.** 🟡 What is the decorator pattern in Java I/O?

<details><summary>Click to reveal answer</summary>

Java I/O uses the **Decorator pattern** extensively. Base stream classes (`InputStream`, `OutputStream`) define the interface, and decorator classes wrap existing streams to add functionality without changing the interface.

```java
// Layered decorators
InputStream in = new FileInputStream("data.gz");          // raw bytes
in = new BufferedInputStream(in);                          // add buffering
in = new GZIPInputStream(in);                              // add decompression
in = new CipherInputStream(in, cipher);                    // add decryption

// Reading from the top-level stream transparently reads,
// decrypts, decompresses, and buffers from the file.
```

**Key decorator classes:**

| Decorator | Adds |
|---|---|
| `BufferedInputStream/OutputStream` | Buffering |
| `DataInputStream/OutputStream` | Primitive type reading/writing |
| `ObjectInputStream/OutputStream` | Object serialization |
| `GZIPInputStream/OutputStream` | GZIP compression |
| `CipherInputStream/OutputStream` | Encryption |

</details>

---

**Q5.** 🟡 Explain `DataInputStream` vs `ObjectInputStream`.

<details><summary>Click to reveal answer</summary>

**`DataInputStream`** reads Java primitive types and Strings written by `DataOutputStream`. It does NOT handle complex objects.

**`ObjectInputStream`** reads fully serialized Java objects (written by `ObjectOutputStream`). It handles object graphs, inheritance, and references.

```java
// DataOutputStream/InputStream - primitives only
try (DataOutputStream dos = new DataOutputStream(new FileOutputStream("prims.dat"))) {
    dos.writeInt(42);
    dos.writeDouble(3.14);
    dos.writeUTF("hello");
}

try (DataInputStream dis = new DataInputStream(new FileInputStream("prims.dat"))) {
    int i = dis.readInt();       // 42
    double d = dis.readDouble(); // 3.14
    String s = dis.readUTF();   // "hello"
}

// ObjectOutputStream/InputStream - full objects
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("obj.dat"))) {
    oos.writeObject(new ArrayList<>(List.of(1, 2, 3)));
}

try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("obj.dat"))) {
    List<Integer> list = (List<Integer>) ois.readObject();
}
```

</details>

---

**Q6.** 🟢 What is `System.in`, `System.out`, and `System.err`?

<details><summary>Click to reveal answer</summary>

- **`System.in`** — `InputStream` connected to standard input (keyboard by default)
- **`System.out`** — `PrintStream` connected to standard output (console by default)  
- **`System.err`** — `PrintStream` connected to standard error (console by default, unbuffered)

`System.out` is buffered; `System.err` is not (errors are flushed immediately).

```java
// Reading from stdin
try (BufferedReader reader = new BufferedReader(new InputStreamReader(System.in))) {
    String line = reader.readLine();
    System.out.println("You typed: " + line);
}

// Redirecting
System.setOut(new PrintStream(new FileOutputStream("output.log")));
System.out.println("This goes to output.log now");
```

</details>

---

### Reader & Writer

**Q7.** 🟡 What is `InputStreamReader` and why is it important?

<details><summary>Click to reveal answer</summary>

`InputStreamReader` is a **bridge** from byte streams to character streams. It reads bytes from an `InputStream` and decodes them to characters using a specified (or default) charset.

```java
// Default charset (platform-dependent — DANGEROUS!)
InputStreamReader isr = new InputStreamReader(System.in);

// Explicit charset — always prefer this
InputStreamReader isr = new InputStreamReader(System.in, StandardCharsets.UTF_8);
```

🚨 **Common Mistake:** Relying on the default charset. The default varies by OS/JVM setting, causing `mojibake` (garbled text) when moving between systems.

```java
// Always specify charset explicitly
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("file.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

</details>

---

**Q8.** 🟡 Compare `FileReader` with `InputStreamReader(new FileInputStream(...))`.

<details><summary>Click to reveal answer</summary>

| | `FileReader` | `InputStreamReader(FileInputStream)` |
|---|---|---|
| Charset control | Limited (Java 11+: constructor with charset) | Full control |
| Flexibility | Less flexible | More flexible |
| Pre-Java 11 | Uses platform default charset | Can specify any charset |

Before Java 11, `FileReader` always used the platform default charset. Since Java 11, `FileReader` has a constructor accepting a `Charset`.

```java
// Java 11+ - both are equivalent
FileReader fr = new FileReader("file.txt", StandardCharsets.UTF_8);

InputStreamReader isr = new InputStreamReader(
    new FileInputStream("file.txt"), StandardCharsets.UTF_8);
```

✅ **Best Practice:** Always specify charset explicitly regardless of which approach you use.

</details>

---

**Q9.** 🟢 What does `BufferedReader.readLine()` return?

<details><summary>Click to reveal answer</summary>

`readLine()` reads a line of text (terminated by `\n`, `\r`, or `\r\n`) and returns it **without the line terminator**. Returns `null` when end-of-stream is reached.

```java
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt", StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line); // line does NOT contain \n
    }
}

// Java 8+ - stream-based reading
try (BufferedReader reader = Files.newBufferedReader(Path.of("file.txt"))) {
    reader.lines().forEach(System.out::println);
}
```

</details>

---

**Q10.** 🟡 Explain `PrintWriter` vs `BufferedWriter`.

<details><summary>Click to reveal answer</summary>

| | `PrintWriter` | `BufferedWriter` |
|---|---|---|
| Purpose | Formatted text output | Buffered character output |
| Auto-flush | Optional (constructor param) | Manual `flush()` required |
| Exception handling | Swallows exceptions (check `checkError()`) | Throws `IOException` |
| Convenience methods | `print()`, `println()`, `printf()` | `write()`, `newLine()` |

```java
// PrintWriter - convenient but swallows exceptions
try (PrintWriter pw = new PrintWriter(new FileWriter("out.txt"))) {
    pw.println("Hello, World!");
    pw.printf("Pi = %.4f%n", Math.PI);
    if (pw.checkError()) System.err.println("Write error occurred");
}

// BufferedWriter - explicit, throws IOException
try (BufferedWriter bw = new BufferedWriter(new FileWriter("out.txt"))) {
    bw.write("Hello, World!");
    bw.newLine(); // platform-appropriate line separator
    bw.flush();   // explicit flush
}
```

</details>

---

### NIO (Path, Files, Channel, Buffer)

**Q11.** 🟡 What is the difference between Java IO and Java NIO?

<details><summary>Click to reveal answer</summary>

| Feature | Java IO (java.io) | Java NIO (java.nio) |
|---|---|---|
| Orientation | Stream-oriented | Buffer-oriented |
| Blocking | Always blocking | Blocking or non-blocking |
| Channels | No | Yes (bidirectional) |
| Selectors | No | Yes (multiplexing) |
| API style | Imperative | More complex, powerful |
| File API | `File` class | `Path`, `Files`, `FileSystem` |

**NIO2** (Java 7+, `java.nio.file`) is the modern file API:

```java
// Old IO - java.io.File
File file = new File("/path/to/file.txt");
boolean exists = file.exists();

// NIO2 - java.nio.file.Path + Files
Path path = Path.of("/path/to/file.txt");
boolean exists = Files.exists(path);

// NIO2 one-liners
String content = Files.readString(path, StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(path);
Files.writeString(path, "content", StandardOpenOption.CREATE);
```

</details>

---

**Q12.** 🟡 What is a `Buffer` in NIO and what are its key properties?

<details><summary>Click to reveal answer</summary>

A `Buffer` is a fixed-capacity container for primitive data used with NIO channels. Key properties:

- **capacity** — total number of elements the buffer can hold (fixed)
- **limit** — index of the first element that should not be read/written
- **position** — index of the next element to read/write
- **mark** — saved position (optional)

**Invariant:** `0 ≤ mark ≤ position ≤ limit ≤ capacity`

```java
ByteBuffer buffer = ByteBuffer.allocate(8);
// capacity=8, limit=8, position=0

buffer.put((byte) 1);
buffer.put((byte) 2);
// position=2, limit=8

buffer.flip();  // prepare for reading: limit=position, position=0
// position=0, limit=2

byte b1 = buffer.get(); // position=1
byte b2 = buffer.get(); // position=2

buffer.clear(); // reset: position=0, limit=capacity
// OR
buffer.compact(); // move unread data to start, position after unread data
```

🚨 **Common Mistake:** Forgetting to call `flip()` before reading from a buffer you just wrote to.

</details>

---

**Q13.** 🟡 What is a `FileChannel` and how does it differ from `FileInputStream`?

<details><summary>Click to reveal answer</summary>

`FileChannel` is a channel for reading, writing, mapping, and locking files. Key advantages:

| Feature | `FileInputStream` | `FileChannel` |
|---|---|---|
| Direction | Read only | Read + Write |
| Memory-mapped files | No | Yes (`map()`) |
| File locking | No | Yes (`lock()`, `tryLock()`) |
| Scatter/Gather | No | Yes |
| Transfer between channels | No | Yes (`transferTo/From()`) |

```java
// FileChannel for reading
try (FileChannel fc = FileChannel.open(Path.of("file.dat"), StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (fc.read(buffer) != -1) {
        buffer.flip();
        // process buffer.array() from 0 to buffer.limit()
        buffer.clear();
    }
}

// Memory-mapped file (fastest for large files)
try (FileChannel fc = FileChannel.open(Path.of("large.dat"), StandardOpenOption.READ)) {
    MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_ONLY, 0, fc.size());
    // Access file content directly from virtual memory
    byte firstByte = mbb.get(0);
}

// Efficient file copy using transferTo
try (FileChannel src = FileChannel.open(Path.of("src.dat"), StandardOpenOption.READ);
     FileChannel dst = FileChannel.open(Path.of("dst.dat"),
         StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
    src.transferTo(0, src.size(), dst);
}
```

</details>

---

**Q14.** 🟢 How do you read all lines from a file using `Files` in NIO2?

<details><summary>Click to reveal answer</summary>

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.stream.Stream;

Path path = Path.of("data.txt");

// Method 1: readAllLines (loads entire file into memory)
List<String> lines = Files.readAllLines(path, StandardCharsets.UTF_8);

// Method 2: readString (Java 11+, entire content as String)
String content = Files.readString(path, StandardCharsets.UTF_8);

// Method 3: lines() stream (lazy, good for large files)
try (Stream<String> stream = Files.lines(path, StandardCharsets.UTF_8)) {
    stream.filter(line -> line.contains("ERROR"))
          .forEach(System.out::println);
}

// Method 4: newBufferedReader (traditional but NIO2-backed)
try (BufferedReader br = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
    br.lines().forEach(System.out::println);
}
```

✅ **Best Practice:** Use `Files.lines()` with try-with-resources for large files — it's lazy and properly closes the underlying file handle.

</details>

---

**Q15.** 🟡 What is `WatchService` in NIO2?

<details><summary>Click to reveal answer</summary>

`WatchService` monitors a directory for file system changes (create, modify, delete) using OS-level notifications (inotify on Linux, kqueue on macOS, ReadDirectoryChangesW on Windows).

```java
Path dir = Path.of("/path/to/watch");
try (WatchService watcher = FileSystems.getDefault().newWatchService()) {
    dir.register(watcher,
        StandardWatchEventKinds.ENTRY_CREATE,
        StandardWatchEventKinds.ENTRY_MODIFY,
        StandardWatchEventKinds.ENTRY_DELETE);

    while (true) {
        WatchKey key = watcher.take(); // blocks until event
        for (WatchEvent<?> event : key.pollEvents()) {
            WatchEvent.Kind<?> kind = event.kind();
            Path filename = (Path) event.context();
            System.out.printf("Event: %s on %s%n", kind, filename);
        }
        boolean valid = key.reset();
        if (!valid) break; // directory no longer accessible
    }
}
```

</details>

---

**Q16.** 🟡 How do you copy/move/delete files using `Files`?

<details><summary>Click to reveal answer</summary>

```java
Path src  = Path.of("source.txt");
Path dst  = Path.of("destination.txt");
Path dest = Path.of("/other/dir/");

// Copy
Files.copy(src, dst);                                      // fails if dst exists
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING); // overwrite
Files.copy(src, dst, StandardCopyOption.COPY_ATTRIBUTES);  // preserve metadata

// Move (atomic on same filesystem)
Files.move(src, dst, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dst, StandardCopyOption.ATOMIC_MOVE);

// Delete
Files.delete(src);          // throws NoSuchFileException if not found
Files.deleteIfExists(src);  // returns false if not found (no exception)

// Recursive delete (no built-in — use walk)
Files.walk(Path.of("dirToDelete"))
     .sorted(Comparator.reverseOrder()) // delete children before parents
     .forEach(p -> {
         try { Files.delete(p); } catch (IOException e) { throw new RuntimeException(e); }
     });
```

</details>

---

### Serialization & Deserialization

**Q17.** 🟢 What is Java Serialization and how do you make a class serializable?

<details><summary>Click to reveal answer</summary>

Java Serialization converts an object's state into a byte stream that can be persisted or transmitted, and later reconstructed (deserialized) back into an object.

To make a class serializable:
1. Implement the `java.io.Serializable` marker interface (no methods to implement)
2. Ensure all non-transient fields are also serializable

```java
import java.io.*;

public class Employee implements Serializable {
    private static final long serialVersionUID = 1L; // highly recommended!

    private String name;
    private int age;
    private transient String password; // excluded from serialization
    private static String company;     // static fields are NOT serialized

    // constructors, getters, setters...
}

// Serialization
try (ObjectOutputStream oos = new ObjectOutputStream(
        new BufferedOutputStream(new FileOutputStream("employee.ser")))) {
    oos.writeObject(new Employee("Alice", 30, "secret", "Acme"));
}

// Deserialization
try (ObjectInputStream ois = new ObjectInputStream(
        new BufferedInputStream(new FileInputStream("employee.ser")))) {
    Employee emp = (Employee) ois.readObject();
    // emp.getPassword() == null (transient)
    // emp.getCompany() == current static value (not deserialized)
}
```

</details>

---

**Q18.** 🟡 What is `serialVersionUID` and what happens if it's missing?

<details><summary>Click to reveal answer</summary>

`serialVersionUID` is a version identifier used during deserialization to verify that the sender and receiver have compatible class definitions.

If not explicitly defined, the JVM **auto-generates** one based on the class structure (fields, methods, etc.). This is dangerous because any class change (adding a field, changing a method signature) changes the auto-generated UID, causing `InvalidClassException` when trying to deserialize old data.

```java
// BAD: no explicit serialVersionUID
public class User implements Serializable {
    private String name; // if you add a field later, old .ser files break!
}

// GOOD: explicit serialVersionUID
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    // Future fields can be added; old .ser files will still deserialize
    // (new fields will be null/default for old data)
}
```

**What happens on mismatch:**
```
java.io.InvalidClassException: User; local class incompatible:
stream classdesc serialVersionUID = -3847918283, local class serialVersionUID = 1
```

✅ **Best Practice:** Always declare `serialVersionUID` explicitly. Increment it when making **incompatible** changes.

</details>

---

**Q19.** 🟡 What is the `transient` keyword?

<details><summary>Click to reveal answer</summary>

The `transient` keyword marks a field to be **excluded from serialization**. When deserializing, transient fields receive their **default values** (`null` for objects, `0` for numbers, `false` for booleans).

```java
public class Session implements Serializable {
    private static final long serialVersionUID = 1L;

    private String userId;
    private transient String authToken;    // sensitive — don't serialize
    private transient Connection dbConn;   // can't serialize (Connection isn't Serializable)
    private transient int cachedResult;   // derived — can recompute after load

    // Custom readObject to reinitialize transient fields
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject(); // reads non-transient fields
        this.cachedResult = computeResult(); // reinitialize derived field
        // authToken must be re-authenticated — left null intentionally
    }
}
```

**Use `transient` for:**
- Sensitive data (passwords, tokens)
- Non-serializable fields (database connections, sockets)
- Cached/derived fields that can be recomputed

</details>

---

**Q20.** 🔴 What is `Externalizable` and how does it differ from `Serializable`?

<details><summary>Click to reveal answer</summary>

`Externalizable` extends `Serializable` and gives **complete control** over the serialization format. Unlike `Serializable` (which uses reflection), `Externalizable` calls `writeExternal()` and `readExternal()` methods you define.

```java
public class Point implements Externalizable {
    private int x, y;

    // Required public no-arg constructor for Externalizable
    public Point() {}
    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(x);
        out.writeInt(y);
        // We control exactly what gets written
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        this.x = in.readInt();
        this.y = in.readInt();
    }
}
```

| Feature | `Serializable` | `Externalizable` |
|---|---|---|
| Control | JVM controls format | Developer controls format |
| Performance | Slower (reflection) | Faster (direct) |
| No-arg constructor | Not required | **Required** |
| Default behavior | All non-transient fields | Nothing (explicit only) |
| Inheritance | Inherited | Must reimplement |

</details>

---

**Q21.** 🔴 What are the security pitfalls of Java serialization?

<details><summary>Click to reveal answer</summary>

Java deserialization is one of the most dangerous attack vectors in Java applications (**CVE-2015-4852**, **CVE-2016-4438**, etc.). Attackers can craft malicious byte streams that execute arbitrary code when deserialized.

**Why it's dangerous:**
1. `readObject()` is called on the class — which runs its constructor-equivalent logic
2. In complex object graphs (common libraries), this can trigger gadget chains
3. The Java deserialization mechanism trusts the byte stream

**Mitigations:**
```java
// 1. Use ObjectInputStream with a filter (Java 9+)
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(info -> {
    if (info.serialClass() != null &&
        !ALLOWED_CLASSES.contains(info.serialClass().getName())) {
        return ObjectInputFilter.Status.REJECTED;
    }
    return ObjectInputFilter.Status.ALLOWED;
});

// 2. Global filter (Java 9+)
// In JVM args: -Djdk.serialFilter=com.myapp.*;!*

// 3. Never deserialize untrusted data
// 4. Consider alternatives: JSON (Jackson), Protocol Buffers, Avro
```

🚨 **Common Mistake:** Deserializing data received from users, message queues, or network without validation. This is a top-10 OWASP vulnerability ("Insecure Deserialization").

</details>

---

**Q22.** 🔴 How do you customize serialization with `readObject`/`writeObject`?

<details><summary>Click to reveal answer</summary>

You can add private `writeObject()` and `readObject()` methods to a `Serializable` class to customize behavior while still using the default serialization mechanism for most fields.

```java
public class SecureUser implements Serializable {
    private static final long serialVersionUID = 1L;
    private String username;
    private String encryptedPassword; // stored encrypted

    private transient String plainPassword; // never serialized directly

    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject(); // serialize non-transient fields normally
        // Custom: encrypt and write password
        oos.writeObject(encrypt(plainPassword));
    }

    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject(); // deserialize non-transient fields
        // Custom: decrypt the password
        String encrypted = (String) ois.readObject();
        this.plainPassword = decrypt(encrypted);
    }

    private void readObjectNoData() throws ObjectStreamException {
        // Called when no data available (e.g., from older serialized form)
        this.username = "unknown";
    }
}
```

**Other customization hooks:**

| Method | When Called |
|---|---|
| `writeObject()` | During serialization |
| `readObject()` | During deserialization |
| `readObjectNoData()` | When class added to hierarchy |
| `writeReplace()` | Replace object before serialization |
| `readResolve()` | Replace object after deserialization |

</details>

---

**Q23.** 🟡 How does serialization handle inheritance?

<details><summary>Click to reveal answer</summary>

```java
public class Animal implements Serializable {
    private static final long serialVersionUID = 1L;
    String name;
}

public class Dog extends Animal {
    private static final long serialVersionUID = 1L;
    String breed;
}

// Dog is serializable because it extends Serializable Animal
// Both 'name' and 'breed' are serialized
```

**If a superclass is NOT Serializable:**
```java
public class Vehicle { // NOT Serializable
    int speed;
    public Vehicle() { this.speed = 0; } // must have no-arg constructor!
}

public class Car extends Vehicle implements Serializable {
    private static final long serialVersionUID = 1L;
    String model;
}

// During deserialization:
// - Vehicle's no-arg constructor IS called (to initialize 'speed')
// - Car's serialized fields are restored
// - Vehicle MUST have accessible no-arg constructor
```

</details>

---

**Q24.** 🟡 Explain `readResolve()` and its use in Singleton serialization.

<details><summary>Click to reveal answer</summary>

When deserializing, Java creates a **new object** bypassing constructors. This breaks the Singleton pattern. `readResolve()` lets you replace the deserialized object with an existing instance.

```java
public class DatabaseConfig implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final DatabaseConfig INSTANCE = new DatabaseConfig();

    private DatabaseConfig() {}

    public static DatabaseConfig getInstance() { return INSTANCE; }

    // Without readResolve, deserialization creates a 2nd instance!
    protected Object readResolve() {
        return INSTANCE; // replace deserialized object with singleton
    }
}

// Test
DatabaseConfig cfg = DatabaseConfig.getInstance();
// Serialize then deserialize
DatabaseConfig deserialized = (DatabaseConfig) ois.readObject();
System.out.println(cfg == deserialized); // true (with readResolve)
                                          // false (without readResolve)
```

✅ **Best Practice:** Use `enum` for singletons — Java guarantees enum instances are never duplicated during deserialization:
```java
public enum DatabaseConfig {
    INSTANCE;
    // enum singleton is serialization-safe by default
}
```

</details>

---

**Q25.** 🟢 What fields are NOT serialized?

<details><summary>Click to reveal answer</summary>

Fields that are **not** serialized:
1. `static` fields — belong to the class, not the instance
2. `transient` fields — explicitly excluded
3. Fields in non-serializable superclasses — initialized via their no-arg constructor

```java
public class Example implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;              // ✅ serialized
    private int age;                  // ✅ serialized
    private static int instanceCount; // ❌ static - NOT serialized
    private transient String token;   // ❌ transient - NOT serialized
    private Thread thread;            // ❌ Thread is not Serializable -> exception!
}
```

🚨 **Common Mistake:** Trying to serialize a class that has non-serializable, non-transient fields. This throws `NotSerializableException` at runtime.

</details>

---

**Q26.** 🟡 What is object graph serialization and how does Java handle circular references?

<details><summary>Click to reveal answer</summary>

Java serialization handles **object graphs** — groups of objects connected by references, including circular references. It maintains a **handle table** mapping already-serialized objects to references.

```java
public class Node implements Serializable {
    private static final long serialVersionUID = 1L;
    String data;
    Node next;

    // Circular reference
    static Node createCircle() {
        Node a = new Node("A");
        Node b = new Node("B");
        a.next = b;
        b.next = a; // circular!
        return a;
    }
}

// Java handles this correctly - each object is written once,
// subsequent references write a "handle" (back-reference)
Node a = Node.createCircle();
// Serialize and deserialize - circular structure is preserved!
```

**How:** During serialization, Java assigns a handle to each object. If the same object is encountered again, it writes the handle instead of re-serializing.

</details>

---

**Q27.** 🟡 How do you handle versioning of serialized objects?

<details><summary>Click to reveal answer</summary>

**Compatible changes** (same `serialVersionUID`):
- Adding fields — old data deserializes; new field gets default value
- Removing fields — data is silently ignored during deserialization
- Adding classes in the hierarchy

**Incompatible changes** (should change `serialVersionUID`):
- Changing field types
- Changing class hierarchy significantly
- Changing a non-serializable superclass to serializable

```java
// Version 1
public class Order implements Serializable {
    private static final long serialVersionUID = 1L;
    private String orderId;
    private double amount;
}

// Version 2 - compatible (just added field)
public class Order implements Serializable {
    private static final long serialVersionUID = 1L; // keep same!
    private String orderId;
    private double amount;
    private String currency; // new field - old .ser files will have null here
}

// Version 3 - incompatible (changed type)
public class Order implements Serializable {
    private static final long serialVersionUID = 2L; // change UID!
    private String orderId;
    private BigDecimal amount; // changed type from double - incompatible!
    private String currency;
}
```

</details>

---

**Q28.** 🟡 What is the `Files.walk()` method and how is it used?

<details><summary>Click to reveal answer</summary>

`Files.walk()` lazily traverses a directory tree using depth-first search, returning a `Stream<Path>`.

```java
// List all .java files recursively
try (Stream<Path> paths = Files.walk(Path.of("src"))) {
    paths.filter(p -> p.toString().endsWith(".java"))
         .forEach(System.out::println);
}

// Calculate total size of files in a directory
long totalSize;
try (Stream<Path> paths = Files.walk(Path.of("project"))) {
    totalSize = paths
        .filter(Files::isRegularFile)
        .mapToLong(p -> {
            try { return Files.size(p); }
            catch (IOException e) { return 0L; }
        })
        .sum();
}

// Find files modified in last 24 hours
Instant oneDayAgo = Instant.now().minus(Duration.ofDays(1));
try (Stream<Path> paths = Files.walk(Path.of("."))) {
    paths.filter(Files::isRegularFile)
         .filter(p -> {
             try {
                 return Files.getLastModifiedTime(p).toInstant().isAfter(oneDayAgo);
             } catch (IOException e) { return false; }
         })
         .forEach(System.out::println);
}
```

✅ **Best Practice:** Always use try-with-resources with `Files.walk()` to close the underlying directory stream.

</details>

---

**Q29.** 🟢 What is `Path.of()` vs `Paths.get()`?

<details><summary>Click to reveal answer</summary>

Both create a `Path` object. `Path.of()` was added in Java 11 as a factory method directly on the `Path` interface, which is cleaner.

```java
// Java 7+ - Paths.get (utility class)
Path p1 = Paths.get("/home/user/file.txt");
Path p2 = Paths.get("/home", "user", "file.txt"); // joins parts

// Java 11+ - Path.of (cleaner, same result)
Path p3 = Path.of("/home/user/file.txt");
Path p4 = Path.of("/home", "user", "file.txt");

// Path operations
Path base = Path.of("/home/user");
Path resolved = base.resolve("documents/file.txt"); // /home/user/documents/file.txt
Path relativized = base.relativize(resolved);        // documents/file.txt
Path normalized = Path.of("/home/user/../user/./file.txt").normalize(); // /home/user/file.txt
```

</details>

---

**Q30.** 🟡 How do you use `Files.newBufferedWriter()` and what are `OpenOption`s?

<details><summary>Click to reveal answer</summary>

```java
Path path = Path.of("output.txt");

// Default: CREATE, TRUNCATE_EXISTING, WRITE
try (BufferedWriter writer = Files.newBufferedWriter(path, StandardCharsets.UTF_8)) {
    writer.write("Hello");
    writer.newLine();
}

// Append mode
try (BufferedWriter writer = Files.newBufferedWriter(path, StandardCharsets.UTF_8,
        StandardOpenOption.CREATE, StandardOpenOption.APPEND)) {
    writer.write("Appended line");
}

// Write atomically - write to temp, then rename
Path temp = path.resolveSibling(path.getFileName() + ".tmp");
try (BufferedWriter writer = Files.newBufferedWriter(temp)) {
    writer.write("atomic content");
}
Files.move(temp, path, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
```

**Common `StandardOpenOption` values:**

| Option | Meaning |
|---|---|
| `READ` | Open for reading |
| `WRITE` | Open for writing |
| `CREATE` | Create if not exists |
| `CREATE_NEW` | Fail if exists |
| `APPEND` | Append to end |
| `TRUNCATE_EXISTING` | Clear before writing |
| `SYNC` | Sync metadata on write |
| `DSYNC` | Sync data on write |

</details>

---

**Q31.** 🟡 What is a `SequenceInputStream`?

<details><summary>Click to reveal answer</summary>

`SequenceInputStream` concatenates multiple `InputStream`s, reading them one after another as if they were a single stream.

```java
InputStream is1 = new FileInputStream("part1.dat");
InputStream is2 = new FileInputStream("part2.dat");
InputStream is3 = new FileInputStream("part3.dat");

// Concatenate streams
try (SequenceInputStream combined = new SequenceInputStream(
        new SequenceInputStream(is1, is2), is3)) {
    byte[] data = combined.readAllBytes();
}

// With Enumeration
List<InputStream> streams = List.of(is1, is2, is3);
Enumeration<InputStream> e = Collections.enumeration(streams);
try (SequenceInputStream seq = new SequenceInputStream(e)) {
    // reads is1, then is2, then is3 seamlessly
}
```

</details>

---

**Q32.** 🟡 What is `PipedInputStream` and `PipedOutputStream`?

<details><summary>Click to reveal answer</summary>

`PipedInputStream`/`PipedOutputStream` create a **pipe** between two threads — one writes to the output, the other reads from the input. They are connected and share an internal circular buffer.

```java
PipedOutputStream pos = new PipedOutputStream();
PipedInputStream pis = new PipedInputStream(pos, 8192); // 8KB buffer

// Writer thread
Thread writer = new Thread(() -> {
    try (pos) {
        pos.write("Hello from writer!".getBytes(StandardCharsets.UTF_8));
    } catch (IOException e) { e.printStackTrace(); }
});

// Reader thread
Thread reader = new Thread(() -> {
    try (pis) {
        String data = new String(pis.readAllBytes(), StandardCharsets.UTF_8);
        System.out.println("Read: " + data);
    } catch (IOException e) { e.printStackTrace(); }
});

writer.start(); reader.start();
writer.join(); reader.join();
```

🚨 **Common Mistake:** Using pipe streams on a single thread — it will deadlock because writing blocks when the buffer is full, and reading blocks when the buffer is empty.

</details>

---

**Q33.** 🟢 What is try-with-resources and why is it critical for I/O?

<details><summary>Click to reveal answer</summary>

Try-with-resources (Java 7+) automatically calls `close()` on resources that implement `AutoCloseable`, even if an exception occurs.

```java
// WRONG - resource leak if exception thrown
FileInputStream fis = new FileInputStream("file.txt");
int data = fis.read(); // if this throws, close() is never called
fis.close();

// WRONG - close() not called if the try block throws
try {
    FileInputStream fis2 = new FileInputStream("file.txt");
    int data = fis2.read();
    fis2.close();
} catch (IOException e) { }

// CORRECT - always closes
try (FileInputStream fis = new FileInputStream("file.txt")) {
    int data = fis.read();
} // fis.close() is always called

// Multiple resources - closed in reverse order
try (FileInputStream fis = new FileInputStream("in.txt");
     FileOutputStream fos = new FileOutputStream("out.txt")) {
    fos.write(fis.readAllBytes());
} // fos.close(), then fis.close()
```

✅ **Best Practice:** ALWAYS use try-with-resources for any I/O resource. Never manually call `close()` in a finally block — it's error-prone and verbose.

</details>

---

**Q34.** 🟡 What are `ByteArrayInputStream` and `ByteArrayOutputStream`?

<details><summary>Click to reveal answer</summary>

In-memory stream implementations that use byte arrays as the backing store — no I/O to disk or network.

```java
// ByteArrayOutputStream - write to in-memory buffer
ByteArrayOutputStream baos = new ByteArrayOutputStream();
try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
    oos.writeObject(someObject);
}
byte[] serializedBytes = baos.toByteArray(); // get the bytes

// ByteArrayInputStream - read from in-memory buffer
ByteArrayInputStream bais = new ByteArrayInputStream(serializedBytes);
try (ObjectInputStream ois = new ObjectInputStream(bais)) {
    Object obj = ois.readObject();
}

// Use case: deep clone via serialization
public static <T extends Serializable> T deepClone(T obj) throws IOException, ClassNotFoundException {
    ByteArrayOutputStream baos2 = new ByteArrayOutputStream();
    new ObjectOutputStream(baos2).writeObject(obj);
    return (T) new ObjectInputStream(new ByteArrayInputStream(baos2.toByteArray())).readObject();
}
```

</details>

---

**Q35.** 🟡 What is `RandomAccessFile` and when would you use it?

<details><summary>Click to reveal answer</summary>

`RandomAccessFile` allows reading and writing at **arbitrary positions** in a file using a file pointer, unlike sequential streams.

```java
try (RandomAccessFile raf = new RandomAccessFile("data.bin", "rw")) {
    // Write at specific position
    raf.seek(100); // move to byte 100
    raf.writeInt(42);

    // Read from beginning
    raf.seek(0);
    int firstInt = raf.readInt();

    // File length
    long length = raf.length();

    // Append without truncating
    raf.seek(raf.length());
    raf.write("appended".getBytes());
}
```

**Use cases:**
- Database file storage (fixed-size records)
- Updating specific records in binary files
- Reading file headers without reading entire file
- Multi-threaded file access (each thread seeks to its region)

**Modes:** `"r"` (read-only), `"rw"` (read-write), `"rws"` (sync content+metadata), `"rwd"` (sync content)

</details>

---

**Q36.** 🔴 Explain `java.nio.file.attribute` for reading file metadata.

<details><summary>Click to reveal answer</summary>

```java
Path path = Path.of("file.txt");

// Basic attributes
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
System.out.println("Size: " + attrs.size());
System.out.println("Created: " + attrs.creationTime());
System.out.println("Modified: " + attrs.lastModifiedTime());
System.out.println("Is directory: " + attrs.isDirectory());
System.out.println("Is symlink: " + attrs.isSymbolicLink());

// POSIX attributes (Unix/Linux)
PosixFileAttributes posixAttrs = Files.readAttributes(path, PosixFileAttributes.class);
System.out.println("Owner: " + posixAttrs.owner().getName());
System.out.println("Group: " + posixAttrs.group().getName());
System.out.println("Permissions: " + PosixFilePermissions.toString(posixAttrs.permissions()));

// Setting permissions
Set<PosixFilePermission> perms = PosixFilePermissions.fromString("rw-r--r--");
Files.setPosixFilePermissions(path, perms);

// Windows ACL (if on Windows)
// AclFileAttributeView acl = Files.getFileAttributeView(path, AclFileAttributeView.class);
```

</details>

---

**Q37.** 🟡 What are `Scanner` and `PrintWriter` and when do you use them?

<details><summary>Click to reveal answer</summary>

```java
// Scanner - parse structured text/tokens
try (Scanner scanner = new Scanner(Path.of("numbers.txt"), StandardCharsets.UTF_8)) {
    while (scanner.hasNextInt()) {
        int n = scanner.nextInt();
        System.out.println(n);
    }
}

// Scanner with custom delimiter
try (Scanner scanner = new Scanner("a,b,c,d").useDelimiter(",")) {
    while (scanner.hasNext()) {
        System.out.println(scanner.next());
    }
}

// PrintWriter - formatted text output
try (PrintWriter pw = new PrintWriter(
        Files.newBufferedWriter(Path.of("report.txt"), StandardCharsets.UTF_8))) {
    pw.printf("%-20s %10s%n", "Name", "Score");
    pw.printf("%-20s %10.2f%n", "Alice", 95.5);
    pw.printf("%-20s %10.2f%n", "Bob", 87.3);
}
```

🚨 **Common Mistake:** Using `Scanner` for large files — it's slow for large-volume parsing. Use `BufferedReader` + `split()` for performance-critical file parsing.

</details>

---

**Q38.** 🟡 What is the difference between `flush()` and `close()`?

<details><summary>Click to reveal answer</summary>

| | `flush()` | `close()` |
|---|---|---|
| Effect | Writes buffered data to underlying stream | Writes buffered data AND releases resource |
| Stream usable after | Yes | No (throws IOException if used after) |
| Typical use | When you need data written but keep stream open | When done with stream |

```java
try (BufferedWriter bw = new BufferedWriter(new FileWriter("out.txt"))) {
    bw.write("line 1");
    bw.newLine();
    bw.flush(); // data written to file NOW (still open)

    bw.write("line 2"); // can still write
    bw.newLine();
    // close() is called implicitly by try-with-resources
    // which also flushes remaining data
}

// Network scenario - flush is critical
try (PrintWriter pw = new PrintWriter(socket.getOutputStream(), true)) { // autoFlush=true
    pw.println("COMMAND"); // flushed immediately with autoFlush=true
}
```

</details>

---

**Q39.** 🟡 What is `StreamTokenizer`?

<details><summary>Click to reveal answer</summary>

`StreamTokenizer` parses a character stream into tokens (words, numbers, quoted strings, whitespace, end-of-line) — more lightweight than full lexers.

```java
try (Reader reader = new BufferedReader(new FileReader("config.txt"))) {
    StreamTokenizer st = new StreamTokenizer(reader);
    st.eolIsSignificant(false); // treat EOL as whitespace

    while (st.nextToken() != StreamTokenizer.TT_EOF) {
        switch (st.ttype) {
            case StreamTokenizer.TT_NUMBER -> System.out.println("Number: " + st.nval);
            case StreamTokenizer.TT_WORD   -> System.out.println("Word: " + st.sval);
            case '"'                       -> System.out.println("String: " + st.sval);
        }
    }
}
```

</details>

---

**Q40.** 🔴 Explain asynchronous I/O with `AsynchronousFileChannel`.

<details><summary>Click to reveal answer</summary>

`AsynchronousFileChannel` (Java 7+) performs file I/O asynchronously — read/write operations return immediately, and results are delivered via callbacks or `Future`.

```java
// Future-based async read
try (AsynchronousFileChannel afc = AsynchronousFileChannel.open(
        Path.of("large.dat"), StandardOpenOption.READ)) {

    ByteBuffer buffer = ByteBuffer.allocate(1024);
    Future<Integer> future = afc.read(buffer, 0);

    // Do other work while reading...
    doOtherWork();

    int bytesRead = future.get(); // wait for completion
    buffer.flip();
    // process buffer
}

// Callback-based async read
AsynchronousFileChannel afc = AsynchronousFileChannel.open(
        Path.of("large.dat"), StandardOpenOption.READ);

ByteBuffer buffer = ByteBuffer.allocate(1024);
afc.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        attachment.flip();
        System.out.println("Read " + result + " bytes");
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        System.err.println("Read failed: " + exc.getMessage());
    }
});
```

</details>

---

**Q41.** 🟡 What is `Files.mismatch()` (Java 12+)?

<details><summary>Click to reveal answer</summary>

`Files.mismatch()` compares two files and returns the position of the first byte that differs, or `-1L` if the files are identical.

```java
Path file1 = Path.of("original.txt");
Path file2 = Path.of("copy.txt");

long mismatchPos = Files.mismatch(file1, file2);

if (mismatchPos == -1L) {
    System.out.println("Files are identical");
} else {
    System.out.println("Files differ at byte position: " + mismatchPos);
}
```

More efficient than reading both files fully and comparing — it can short-circuit on the first difference.

</details>

---

**Q42.** 🟡 How do you create a temporary file or directory in Java?

<details><summary>Click to reveal answer</summary>

```java
// Create temp file (deleted when JVM exits if deleteOnExit called)
Path tempFile = Files.createTempFile("prefix-", ".suffix");
tempFile.toFile().deleteOnExit();

// Create temp file in specific directory
Path tempInDir = Files.createTempFile(Path.of("work"), "tmp-", ".dat");

// Create temp directory
Path tempDir = Files.createTempDirectory("myapp-");

// Using try-with-resources pattern for cleanup
Path temp = Files.createTempFile("data-", ".json");
try {
    Files.writeString(temp, jsonData);
    processFile(temp);
} finally {
    Files.deleteIfExists(temp);
}
```

</details>

---

## Scenario-Based Questions

**S1.** 🟡 You need to copy a 10GB file as efficiently as possible in Java. What approach do you use?

<details><summary>Click to reveal answer</summary>

```java
// Option 1: Files.copy() - delegates to OS-level copy (fastest in practice)
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);

// Option 2: FileChannel.transferTo (zero-copy if OS supports it)
try (FileChannel from = FileChannel.open(src, StandardOpenOption.READ);
     FileChannel to   = FileChannel.open(dst, StandardOpenOption.WRITE,
                                         StandardOpenOption.CREATE,
                                         StandardOpenOption.TRUNCATE_EXISTING)) {
    long transferred = 0;
    long size = from.size();
    while (transferred < size) {
        transferred += from.transferTo(transferred, size - transferred, to);
    }
}

// Option 3: Memory-mapped (good for multiple passes over the file)
try (FileChannel fc = FileChannel.open(src, StandardOpenOption.READ)) {
    MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_ONLY, 0, fc.size());
    Files.write(dst, mbb.array()); // not ideal for 10GB (MBB limited to ~2GB)
}
```

**Recommendation:** `Files.copy()` first (uses OS optimizations), then `FileChannel.transferTo()` for explicit control.

</details>

---

**S2.** 🟡 A serialized object is sent over the network from one service to another. The receiving service has a newer version of the class with an extra field. What happens?

<details><summary>Click to reveal answer</summary>

It depends on the `serialVersionUID`:

**Same `serialVersionUID` (compatible change):**
- Deserialization succeeds
- The new field is initialized to its **default value** (`null`, `0`, `false`)
- Old data for removed fields is silently ignored

**Different `serialVersionUID`:**
- `InvalidClassException` is thrown
- Deserialization fails

**Best approach for evolving serialized classes:**
```java
// Service V1 sends this:
public class UserEvent implements Serializable {
    private static final long serialVersionUID = 1L;
    private String userId;
    private String action;
}

// Service V2 receives with newer class:
public class UserEvent implements Serializable {
    private static final long serialVersionUID = 1L; // same UID = compatible
    private String userId;
    private String action;
    private Instant timestamp; // new field - will be null for old events

    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        if (timestamp == null) timestamp = Instant.EPOCH; // safe default
    }
}
```

✅ **Best Practice:** For inter-service communication, prefer JSON (Jackson) or Protocol Buffers over Java serialization — much easier to version.

</details>

---

**S3.** 🔴 You're implementing a file-based cache that needs to handle concurrent reads and writes safely. Sketch the design.

<details><summary>Click to reveal answer</summary>

```java
public class FileCache {
    private final Path cacheDir;
    private final ConcurrentHashMap<String, ReadWriteLock> locks = new ConcurrentHashMap<>();

    public FileCache(Path cacheDir) throws IOException {
        this.cacheDir = cacheDir;
        Files.createDirectories(cacheDir);
    }

    private ReadWriteLock getLock(String key) {
        return locks.computeIfAbsent(key, k -> new ReentrantReadWriteLock());
    }

    public Optional<byte[]> get(String key) throws IOException {
        Path file = cacheDir.resolve(sanitize(key));
        ReadWriteLock lock = getLock(key);
        lock.readLock().lock();
        try {
            if (!Files.exists(file)) return Optional.empty();
            return Optional.of(Files.readAllBytes(file));
        } finally {
            lock.readLock().unlock();
        }
    }

    public void put(String key, byte[] value) throws IOException {
        Path file = cacheDir.resolve(sanitize(key));
        Path temp = cacheDir.resolve(sanitize(key) + ".tmp." + Thread.currentThread().getId());
        ReadWriteLock lock = getLock(key);
        lock.writeLock().lock();
        try {
            // Write to temp, then atomic move
            Files.write(temp, value);
            Files.move(temp, file, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
        } finally {
            Files.deleteIfExists(temp);
            lock.writeLock().unlock();
        }
    }

    private String sanitize(String key) {
        // Prevent path traversal attacks
        return key.replaceAll("[^a-zA-Z0-9_-]", "_");
    }
}
```

Key points: `ReadWriteLock` per key, atomic move for safe concurrent writes, path sanitization.

</details>

---

**S4.** 🟡 How would you implement a streaming CSV parser that handles files larger than available RAM?

<details><summary>Click to reveal answer</summary>

```java
public class StreamingCsvParser {

    public Stream<String[]> parse(Path csvFile) throws IOException {
        BufferedReader reader = Files.newBufferedReader(csvFile, StandardCharsets.UTF_8);
        return reader.lines()
                     .onClose(() -> {
                         try { reader.close(); }
                         catch (IOException e) { throw new UncheckedIOException(e); }
                     })
                     .map(this::parseLine);
    }

    private String[] parseLine(String line) {
        // Simple CSV parser (use Apache Commons CSV for production)
        List<String> fields = new ArrayList<>();
        boolean inQuotes = false;
        StringBuilder field = new StringBuilder();
        for (char c : line.toCharArray()) {
            if (c == '"') {
                inQuotes = !inQuotes;
            } else if (c == ',' && !inQuotes) {
                fields.add(field.toString());
                field = new StringBuilder();
            } else {
                field.append(c);
            }
        }
        fields.add(field.toString());
        return fields.toArray(String[]::new);
    }
}

// Usage - processes 100GB file with constant memory
try (Stream<String[]> rows = parser.parse(Path.of("huge.csv"))) {
    rows.skip(1) // skip header
        .filter(row -> "ACTIVE".equals(row[2]))
        .forEach(row -> processRow(row));
}
```

</details>

---

**S5.** 🔴 A service deserializes objects from an external queue. What security measures must you implement?

<details><summary>Click to reveal answer</summary>

```java
// 1. Allowlist-based ObjectInputFilter
ObjectInputFilter allowlistFilter = ObjectInputFilter.Config.createFilter(
    "com.mycompany.messages.*;java.util.*;java.lang.*;!*"
);
// Sets global filter
ObjectInputFilter.Config.setSerialFilter(allowlistFilter);

// 2. Per-stream filter with size limits
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(info -> {
    // Reject unknown classes
    if (info.serialClass() != null) {
        String name = info.serialClass().getName();
        if (!ALLOWED_CLASSES.contains(name)) {
            return ObjectInputFilter.Status.REJECTED;
        }
    }
    // Reject if too many objects (possible DoS)
    if (info.depth() > 10) return ObjectInputFilter.Status.REJECTED;
    if (info.references() > 1000) return ObjectInputFilter.Status.REJECTED;
    if (info.streamBytes() > 1_000_000) return ObjectInputFilter.Status.REJECTED;
    return ObjectInputFilter.Status.ALLOWED;
});

// 3. Better: avoid Java serialization entirely
// Use JSON with Jackson (with polymorphism disabled)
ObjectMapper mapper = new ObjectMapper();
mapper.disableDefaultTyping(); // prevent polymorphic deserialization attacks
MyMessage msg = mapper.readValue(jsonBytes, MyMessage.class);
```

**Key mitigations:**
1. Allowlist permitted classes
2. Limit object graph depth, reference count, stream size
3. Prefer JSON/Protobuf over Java serialization for external data
4. Never deserialize from untrusted sources without validation

</details>

---

**S6.** 🟡 How would you tail a log file in Java (like `tail -f`)?

<details><summary>Click to reveal answer</summary>

```java
public class LogTailer {
    public static void tailFile(Path path) throws IOException, InterruptedException {
        try (RandomAccessFile raf = new RandomAccessFile(path.toFile(), "r")) {
            // Jump to end of file
            long fileSize = raf.length();
            raf.seek(fileSize);

            while (true) {
                String line = raf.readLine();
                if (line != null) {
                    System.out.println(line);
                } else {
                    // Check if file was rotated (size decreased)
                    if (raf.length() < raf.getFilePointer()) {
                        raf.seek(0); // file was rotated, read from beginning
                    }
                    Thread.sleep(200); // poll interval
                }
            }
        }
    }

    // Better: use WatchService to avoid polling
    public static void tailWithWatchService(Path path) throws IOException, InterruptedException {
        try (RandomAccessFile raf = new RandomAccessFile(path.toFile(), "r");
             WatchService watcher = FileSystems.getDefault().newWatchService()) {

            raf.seek(raf.length());
            path.getParent().register(watcher, StandardWatchEventKinds.ENTRY_MODIFY);

            while (true) {
                WatchKey key = watcher.poll(1, TimeUnit.SECONDS);
                if (key != null) {
                    for (WatchEvent<?> event : key.pollEvents()) {
                        if (path.getFileName().equals(event.context())) {
                            String line;
                            while ((line = raf.readLine()) != null) {
                                System.out.println(line);
                            }
                        }
                    }
                    key.reset();
                }
            }
        }
    }
}
```

</details>

---

**S7.** 🟡 You need to read a large binary file and process records in parallel. How do you split the work?

<details><summary>Click to reveal answer</summary>

```java
public class ParallelRecordProcessor {
    private static final int RECORD_SIZE = 128; // fixed-size records

    public void processInParallel(Path file, int threadCount) throws Exception {
        long fileSize = Files.size(file);
        long totalRecords = fileSize / RECORD_SIZE;
        long recordsPerThread = totalRecords / threadCount;

        try (FileChannel fc = FileChannel.open(file, StandardOpenOption.READ)) {
            List<Future<?>> futures = new ArrayList<>();
            ExecutorService executor = Executors.newFixedThreadPool(threadCount);

            for (int t = 0; t < threadCount; t++) {
                final long startRecord = t * recordsPerThread;
                final long endRecord = (t == threadCount - 1) ? totalRecords : startRecord + recordsPerThread;
                final long startByte = startRecord * RECORD_SIZE;
                final long byteCount = (endRecord - startRecord) * RECORD_SIZE;

                futures.add(executor.submit(() -> {
                    // Each thread gets its own mapped region
                    MappedByteBuffer mbb = fc.map(
                        FileChannel.MapMode.READ_ONLY, startByte, byteCount);
                    byte[] record = new byte[RECORD_SIZE];
                    while (mbb.hasRemaining()) {
                        mbb.get(record);
                        processRecord(record);
                    }
                }));
            }

            for (Future<?> f : futures) f.get();
            executor.shutdown();
        }
    }
}
```

</details>

---

**S8.** 🟢 How do you read a properties file in Java?

<details><summary>Click to reveal answer</summary>

```java
// Method 1: java.util.Properties
Properties props = new Properties();
try (InputStream is = Files.newInputStream(Path.of("config.properties"))) {
    props.load(is);
}
String host = props.getProperty("db.host", "localhost"); // with default

// Method 2: Load from classpath
try (InputStream is = MyClass.class.getResourceAsStream("/config.properties")) {
    if (is == null) throw new IllegalStateException("config.properties not found in classpath");
    props.load(is);
}

// Method 3: XML properties
try (InputStream is = Files.newInputStream(Path.of("config.xml"))) {
    props.loadFromXML(is);
}

// Writing properties
Properties output = new Properties();
output.setProperty("key", "value");
try (OutputStream os = Files.newOutputStream(Path.of("output.properties"))) {
    output.store(os, "Application configuration");
}
```

</details>

---

**S9.** 🟡 Describe how `ObjectOutputStream` handles the serialization of an object graph with shared references.

<details><summary>Click to reveal answer</summary>

`ObjectOutputStream` maintains a **reference table** (essentially an identity map). When serializing an object:
1. Check if the object is already in the table
2. If yes → write a **back-reference** (handle) instead of re-serializing
3. If no → serialize the object fully and add it to the table

```java
String shared = "shared string";
List<String> list = new ArrayList<>();
list.add(shared);
list.add(shared); // same reference!

// Serialized form contains "shared string" once, with a back-reference for the second entry

ByteArrayOutputStream baos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(baos);
oos.writeObject(list);
oos.writeObject(shared); // back-reference again!

// CAUTION: reset() clears the reference table
oos.reset(); // now next writeObject will serialize fresh
oos.writeObject(shared); // serialized again (larger but allows GC of old objects)
```

**`reset()` use case:** Long-running serialization sessions where you want the stream to release references to previously-written objects, allowing GC.

</details>

---

**S10.** 🟡 How would you efficiently write millions of records to a file?

<details><summary>Click to reveal answer</summary>

```java
public void writeMillionRecords(Path outputPath, Stream<Record> records) throws IOException {
    // Key: use buffered writer + batch writes + efficient formatting
    try (BufferedWriter writer = new BufferedWriter(
            new OutputStreamWriter(
                Files.newOutputStream(outputPath, StandardOpenOption.CREATE, StandardOpenOption.WRITE),
                StandardCharsets.UTF_8),
            65536)) { // large buffer

        StringBuilder batch = new StringBuilder(65536);
        final int[] count = {0};

        records.forEach(record -> {
            batch.append(record.toCSVLine()).append('\n');
            count[0]++;
            if (count[0] % 10_000 == 0) {
                try {
                    writer.write(batch.toString());
                    batch.setLength(0); // reset without allocating
                } catch (IOException e) { throw new UncheckedIOException(e); }
            }
        });

        // Write remaining
        if (batch.length() > 0) writer.write(batch.toString());
    }
}
```

**Optimizations:**
1. Large buffer (64KB) — fewer system calls
2. Batch string building — fewer write calls
3. `StringBuilder.setLength(0)` — reuse buffer without GC pressure
4. Stream API for memory-efficient source iteration

</details>

---

**S11–S30.** Additional scenario topics (brief):

**S11.** 🟢 How to check if a file exists without race condition? → `Files.exists()` is racy; use try-open-catch instead.

**S12.** 🟡 How to implement file locking? → `FileChannel.lock()` / `FileChannel.tryLock()`.

**S13.** 🟡 How to read a file from classpath in a JAR? → `getClass().getResourceAsStream("/path")`.

**S14.** 🟡 How to serialize to JSON instead of binary? → Use Jackson `ObjectMapper.writeValue(file, object)`.

**S15.** 🔴 How to handle `OutOfMemoryError` when reading a file? → Use streaming (`Files.lines()`), never `readAllBytes()` for large files.

**S16.** 🟡 How to detect file encoding? → Use ICU4J or read BOM bytes: `EF BB BF` = UTF-8, `FF FE` = UTF-16 LE.

**S17.** 🟢 How to list files in a directory? → `Files.list(dir)` returns a `Stream<Path>`.

**S18.** 🟡 How to write a ZIP file? → `ZipOutputStream` wrapping `FileOutputStream`, add `ZipEntry` for each file.

**S19.** 🔴 How does Java handle symbolic links in NIO? → `Files.isSymbolicLink()`, `Files.readSymbolicLink()`, most methods follow links by default; use `LinkOption.NOFOLLOW_LINKS` to not follow.

**S20.** 🟡 How to implement a `Serializable` class that has a `Logger` field? → Mark the `Logger` as `transient`; reinitialize in `readObject()`.

---

## 🔗 Related Topics
- [Java Collections Framework](04-collections-framework.md)
- [Java Concurrency](06-concurrency.md)
- [Java Security](16-java-security.md)

---

*🚨 Common Mistake: Using `File` class instead of `Path`/`Files` for new code. The `java.nio.file` API is superior in every way.*

*✅ Best Practice: Always close streams, always specify charset, always buffer I/O, never deserialize untrusted data.*
