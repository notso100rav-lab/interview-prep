# Chapter 7: Strings & Regular Expressions

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- String immutability and why it matters
- String pool and `intern()`
- `StringBuilder` vs `StringBuffer` vs String concatenation
- String methods: `charAt`, `substring`, `indexOf`, `split`, `trim`, `strip`
- String comparison pitfalls
- Regular expressions: Pattern, Matcher, groups, flags
- Common string manipulation interview problems
- Java 15+ Text Blocks

---

## 🔑 Key Concepts at a Glance

```
String Memory Model:
  String Pool (Heap, Java 7+):  ["hello"] ← s1, s2 refer here
  Regular Heap:                 [String@abc] ← new String("hello")

  "abc" + "def"  →  "abcdef" (compile-time constant folding)
  s1 + s2        →  StringBuilder internally (runtime concatenation)
  
  s1 + s2 in loop → Creates N StringBuilders! Use StringBuilder.
```

---

## 🧠 Theory Questions

### String Immutability

---

🟡 **Q1. Why is `String` immutable in Java? What are the benefits?**

<details><summary>Click to reveal answer</summary>

`String` is immutable — once created, its value cannot change. Implemented with:
- `private final char[] value` (pre-Java 9) or `private final byte[] value` (Java 9+ compact strings)
- No setter methods exposed

**Benefits of immutability:**

**1. String Pool (caching):**
```java
// If strings were mutable, pool sharing would be dangerous:
String s1 = "hello";
String s2 = "hello";
// Both point to same pool object
s1.append("!"); // if mutation were possible, s2 would ALSO change!

// Immutability makes sharing safe:
String s1 = "hello";
String s2 = "hello";
// s2 still "hello" regardless of s1 operations
```

**2. Thread safety:**
```java
// Strings are inherently thread-safe — no synchronization needed
String key = "database.url";
// Multiple threads can read 'key' simultaneously without issues
```

**3. Safe as HashMap/HashSet key:**
```java
Map<String, User> cache = new HashMap<>();
String key = "user123";
cache.put(key, user);
// key.value can never change, so hashCode() is always consistent
```

**4. Security:**
```java
// File paths, class names, network hosts — can't be changed after construction
void readFile(String path) {
    // Even if caller retains reference to 'path', they can't change it
    // after passing it to this method
    SecurityManager.checkRead(path);
    // ... read file
}
```

**5. Caching `hashCode()`:**
```java
// String caches its hashCode after first computation:
private int hash; // default 0

public int hashCode() {
    int h = hash;
    if (h == 0 && !hashIsZero) {  // compute once
        h = computeHash();
        hash = h;
    }
    return h;
}
```

> 💡 **Interviewer Tip:** Most candidates say "thread safety" and stop. Mentioning String pool, HashMap key safety, and hashCode caching shows deeper understanding.

</details>

---

🟡 **Q2. What is the String pool and how does `intern()` work?**

<details><summary>Click to reveal answer</summary>

**String Pool:** A special area in the heap where the JVM stores string literals to reuse them.

```java
// Literals → automatically go to pool
String s1 = "hello";
String s2 = "hello";
System.out.println(s1 == s2);  // true — same pool object

// new String() → always creates heap object (NOT in pool)
String s3 = new String("hello");
System.out.println(s1 == s3);  // false — different objects

// intern() → returns the pool version (adds to pool if not present)
String s4 = s3.intern();
System.out.println(s1 == s4);  // true — s4 IS s1 (same pool object)
System.out.println(s3 == s4);  // false — s3 is heap, s4 is pool

// Compile-time constant folding → result goes to pool
String s5 = "hel" + "lo";      // compile-time: "hello" → pool
System.out.println(s1 == s5);  // true

// Runtime concatenation → NOT in pool
String a = "hel";
String s6 = a + "lo";           // runtime: new String("hello") on heap
System.out.println(s1 == s6);  // false
```

**`intern()` use case — memory optimization:**
```java
// When reading millions of strings from file/DB, many may be duplicates
List<String> cities = loadCitiesFromDB();  // ["New York", "London", "New York", ...]
// After intern: all "New York" strings share the same pool object
cities.replaceAll(String::intern);  // may save significant memory
// Modern JVMs: deduplicate strings automatically with G1 (-XX:+UseStringDeduplication)
```

</details>

---

### StringBuilder vs StringBuffer

---

🟡 **Q3. What is the difference between `String`, `StringBuilder`, and `StringBuffer`?**

<details><summary>Click to reveal answer</summary>

| | `String` | `StringBuilder` | `StringBuffer` |
|-|----------|----------------|---------------|
| Mutability | ❌ Immutable | ✅ Mutable | ✅ Mutable |
| Thread-safe | ✅ (inherently) | ❌ | ✅ (synchronized) |
| Performance | Slow for repeated concat | Fastest | Slower (sync overhead) |
| Since | Java 1.0 | Java 5 | Java 1.0 |
| Use case | Constants, simple ops | Single-thread string building | Multi-thread string building (rare) |

```java
// String concatenation in loop — TERRIBLE performance
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;  // creates 10000 intermediate String objects!
    // JVM may optimize simple cases, but don't rely on it
}

// StringBuilder — correct approach
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);  // in-place mutation, no new objects
}
String result = sb.toString();

// StringBuilder API
StringBuilder sb2 = new StringBuilder("hello");
sb2.append(" world");           // "hello world"
sb2.insert(5, ",");             // "hello, world"
sb2.delete(5, 6);               // "hello world"
sb2.reverse();                  // "dlrow olleh"
sb2.replace(0, 5, "bye");       // "bye olleh" (after reverse)
sb2.deleteCharAt(0);            // delete at index
int idx = sb2.indexOf("o");     // find index
sb2.charAt(0);                  // char at index
sb2.length();                   // current length
sb2.capacity();                 // buffer capacity (default 16)
sb2.setCharAt(0, 'X');          // mutate character at index

// Chaining:
String r = new StringBuilder()
    .append("Hello")
    .append(", ")
    .append("World")
    .append("!")
    .toString();
```

**Compiler optimization — string concatenation with `+`:**
```java
// Simple concatenation (literals/constants): compiled to single string
String s = "Hello" + " " + "World";  // → "Hello World" at compile time

// Dynamic concatenation: compiler uses StringBuilder internally
String a = "Hello";
String b = "World";
String s = a + " " + b;
// Compiled as: new StringBuilder().append(a).append(" ").append(b).toString()
// So: s = a + " " + b is fine (one StringBuilder created)

// In loops: compiler creates StringBuilder per ITERATION — bad!
for (int i = 0; i < n; i++) {
    result = result + items[i];  // new StringBuilder each iteration!
}
```

</details>

---

### String Methods

---

🟢 **Q4. Explain key String methods that appear in interviews.**

<details><summary>Click to reveal answer</summary>

```java
String s = "  Hello, World!  ";

// Basic operations
s.length()              // 18
s.isEmpty()             // false (has whitespace)
s.isBlank()             // false (Java 11+: true only if all whitespace)
"".isEmpty()            // true
"  ".isBlank()          // true (Java 11+)

// Case
s.toUpperCase()         // "  HELLO, WORLD!  "
s.toLowerCase()         // "  hello, world!  "

// Whitespace
s.trim()                // "Hello, World!" (removes ASCII whitespace: space, tab, etc.)
s.strip()               // "Hello, World!" (Java 11+: Unicode-aware)
s.stripLeading()        // "Hello, World!  " (Java 11+)
s.stripTrailing()       // "  Hello, World!" (Java 11+)

// Search
s.contains("World")     // true
s.startsWith("  Hello") // true
s.endsWith("!  ")       // true
s.indexOf("o")          // 7 (first occurrence)
s.lastIndexOf("o")      // 12 (last occurrence)
s.indexOf("o", 8)       // 12 (search from index 8)

// Extraction
s.charAt(2)             // 'H'
s.substring(2, 7)       // "Hello" (from index 2, up to but not including 7)
s.substring(9)          // "World!  " (from index 9 to end)

// Transformation
s.replace("World", "Java")     // "  Hello, Java!  "
s.replaceAll("\\s+", " ")      // replace all whitespace runs with single space
s.replaceFirst("\\w+", "Hi")   // replace first word

// Split
"a,b,,c".split(",")             // ["a", "b", "", "c"] — empty string included
"a,b,,c".split(",", -1)         // ["a", "b", "", "c"] — same, limit -1 keeps trailing empty
"a,b,,c".split(",", 2)          // ["a", "b,,c"] — max 2 parts
"  a  b  c  ".trim().split("\\s+")  // ["a", "b", "c"]

// Join
String.join(", ", "a", "b", "c")   // "a, b, c"
String.join("-", List.of("x","y"))  // "x-y"

// Char/Bytes
char[] chars = s.toCharArray()
byte[] bytes = s.getBytes(StandardCharsets.UTF_8)
String.valueOf(42)     // "42"
String.valueOf(true)   // "true"
String.valueOf('c')    // "c"
String.format("Hello %s, you are %d", "Alice", 30)  // "Hello Alice, you are 30"

// Java 11+ methods
"line1\nline2\nline3".lines()  // Stream<String>
"hello".repeat(3)               // "hellohellohello"
"abc".chars()                   // IntStream of char codes

// Java 12+
"   ".isBlank()                 // true
```

</details>

---

### Regular Expressions

---

🟡 **Q5. How do you use `Pattern` and `Matcher` in Java?**

<details><summary>Click to reveal answer</summary>

```java
import java.util.regex.*;

// Compile once, reuse (Pattern compilation is expensive)
Pattern emailPattern = Pattern.compile(
    "^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$"
);

// Test if entire string matches
String email = "user@example.com";
boolean isValid = emailPattern.matcher(email).matches();

// Find matches within string
Pattern wordPattern = Pattern.compile("\\b\\w{4}\\b");  // 4-letter words
Matcher matcher = wordPattern.matcher("This Java code test");

List<String> found = new ArrayList<>();
while (matcher.find()) {
    found.add(matcher.group());  // entire match
    System.out.println("Found '" + matcher.group() + "' at position " + matcher.start());
}
// Output: "This", "Java", "code", "test"

// Capturing groups
Pattern datePattern = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");
Matcher dateMatcher = datePattern.matcher("Event on 2024-01-15");

if (dateMatcher.find()) {
    String fullMatch = dateMatcher.group(0);   // "2024-01-15"
    String year = dateMatcher.group(1);        // "2024"
    String month = dateMatcher.group(2);       // "01"
    String day = dateMatcher.group(3);         // "15"
}

// Named groups (Java 7+)
Pattern namedPattern = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})");
Matcher namedMatcher = namedPattern.matcher("2024-01-15");
if (namedMatcher.matches()) {
    String year = namedMatcher.group("year");   // "2024"
}

// String.matches() — matches entire string (anchored)
"hello123".matches("[a-z]+\\d+")   // true
"hello123!".matches("[a-z]+\\d+")  // false (! at end)

// String.replaceAll() — uses regex
"hello world".replaceAll("\\s+", "_")    // "hello_world"
"hello 123 world".replaceAll("\\d+", "X")  // "hello X world"

// Split by regex
"one,  two,   three".split(",\\s*")  // ["one", "two", "three"]
```

**Pattern flags:**
```java
// Case-insensitive
Pattern.compile("hello", Pattern.CASE_INSENSITIVE);
Pattern.compile("(?i)hello");  // inline flag

// Multi-line (^ and $ match line boundaries)
Pattern.compile("^\\w+", Pattern.MULTILINE);
Pattern.compile("(?m)^\\w+");

// Dotall (. matches newlines too)
Pattern.compile(".*", Pattern.DOTALL);
Pattern.compile("(?s).*");

// Comments and whitespace ignored
Pattern.compile("(?x)  \\d+  \\.  \\d+  # decimal number");
```

</details>

---

🟢 **Q6. What are the most important regex patterns to know?**

<details><summary>Click to reveal answer</summary>

```
Metacharacters:
  .    any character (except newline by default)
  \d   digit [0-9]
  \D   non-digit
  \w   word character [a-zA-Z0-9_]
  \W   non-word character
  \s   whitespace [\t\n\f\r ]
  \S   non-whitespace
  \b   word boundary
  ^    start of string (or line with MULTILINE)
  $    end of string (or line with MULTILINE)

Quantifiers:
  *    0 or more (greedy)
  +    1 or more (greedy)
  ?    0 or 1 (greedy)
  *?   0 or more (lazy/reluctant)
  +?   1 or more (lazy)
  {n}  exactly n
  {n,} n or more
  {n,m} n to m (inclusive)

Groups:
  (abc)     capturing group
  (?:abc)   non-capturing group
  (?=abc)   positive lookahead
  (?!abc)   negative lookahead
  (?<=abc)  positive lookbehind
  (?<!abc)  negative lookbehind

Common patterns:
```

```java
// Email (simplified)
"^[\\w.%+\\-]+@[\\w.\\-]+\\.[a-zA-Z]{2,}$"

// Phone (US format)
"^\\+?1?[-. ]?\\(?[0-9]{3}\\)?[-. ]?[0-9]{3}[-. ]?[0-9]{4}$"

// URL
"https?://[\\w.\\-]+(:[0-9]+)?(/[\\w.\\-/%?&=]*)?"

// IP address
"^(25[0-5]|2[0-4]\\d|[01]?\\d\\d?)\\.(25[0-5]|2[0-4]\\d|[01]?\\d\\d?)\\.(25[0-5]|2[0-4]\\d|[01]?\\d\\d?)\\.(25[0-5]|2[0-4]\\d|[01]?\\d\\d?)$"

// Java identifier
"[a-zA-Z_$][a-zA-Z0-9_$]*"

// Number (integer or decimal)
"^-?\\d+(\\.\\d+)?$"

// Whitespace-only string
"^\\s*$"

// Alphanumeric
"^[a-zA-Z0-9]+$"

// Date YYYY-MM-DD
"^\\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\\d|3[01])$"

// Extract all numbers from string:
Pattern.compile("\\d+").matcher("abc 123 def 456").results()  // Java 9+
    .map(MatchResult::group)
    .collect(Collectors.toList())  // ["123", "456"]
```

</details>

---

## 🎭 Scenario-Based Questions

---

🟡 **S1. What's the output?**

```java
String s1 = "Hello";
String s2 = "Hello";
String s3 = new String("Hello");
String s4 = s3.intern();
String s5 = "Hel" + "lo";
String part = "Hel";
String s6 = part + "lo";

System.out.println(s1 == s2);      // A
System.out.println(s1 == s3);      // B
System.out.println(s1 == s4);      // C
System.out.println(s1 == s5);      // D
System.out.println(s1 == s6);      // E
System.out.println(s1.equals(s6)); // F
```

<details><summary>Click to reveal answer</summary>

- **A: `true`** — Both literals, same pool object
- **B: `false`** — s3 created on heap with `new`
- **C: `true`** — `intern()` returns pool reference
- **D: `true`** — `"Hel" + "lo"` is compile-time constant → goes to pool
- **E: `false`** — `part + "lo"` is runtime concatenation → new String on heap
- **F: `true`** — `equals()` compares content, not reference

</details>

---

🟡 **S2. How do you reverse a string efficiently?**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: StringBuilder (simplest, most commonly expected in interviews)
public static String reverse(String s) {
    return new StringBuilder(s).reverse().toString();
}

// Method 2: Two-pointer (in-place on char array)
public static String reverseManual(String s) {
    char[] chars = s.toCharArray();
    int left = 0, right = chars.length - 1;
    while (left < right) {
        char temp = chars[left];
        chars[left++] = chars[right];
        chars[right--] = temp;
    }
    return new String(chars);
}

// Method 3: Stream
public static String reverseStream(String s) {
    return s.chars()
        .mapToObj(c -> String.valueOf((char) c))
        .reduce("", (a, b) -> b + a);  // O(n²) due to String creation — not recommended
}

// Reverse words in a sentence:
public static String reverseWords(String s) {
    String[] words = s.trim().split("\\s+");
    StringBuilder sb = new StringBuilder();
    for (int i = words.length - 1; i >= 0; i--) {
        sb.append(words[i]);
        if (i > 0) sb.append(" ");
    }
    return sb.toString();
}
// "Hello World Java" → "Java World Hello"
```

</details>

---

🟡 **S3. How do you check if a string is a palindrome?**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: Two-pointer (most efficient)
public static boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) return false;
        left++;
        right--;
    }
    return true;
}

// Method 2: StringBuilder.reverse()
public static boolean isPalindromeSimple(String s) {
    return s.equals(new StringBuilder(s).reverse().toString());
}

// Case-insensitive, alphanumeric only (common interview variant):
public static boolean isPalindromeClean(String s) {
    String cleaned = s.toLowerCase().replaceAll("[^a-z0-9]", "");
    return cleaned.equals(new StringBuilder(cleaned).reverse().toString());
}
// "A man, a plan, a canal: Panama" → true
```

</details>

---

🟡 **S4. How do you count character frequencies in a string?**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: HashMap
public static Map<Character, Integer> charFrequency(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    for (char c : s.toCharArray()) {
        freq.merge(c, 1, Integer::sum);  // atomic merge
        // or: freq.put(c, freq.getOrDefault(c, 0) + 1);
    }
    return freq;
}

// Method 2: Stream (Java 8+)
Map<Character, Long> freq = s.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

// Method 3: Array (for ASCII only — O(1) space relative to alphabet size)
public static int[] charFrequencyArray(String s) {
    int[] freq = new int[128]; // ASCII
    for (char c : s.toCharArray()) freq[c]++;
    return freq;
}

// Find first non-repeating character:
public static char firstNonRepeating(String s) {
    Map<Character, Integer> freq = new LinkedHashMap<>();  // maintain insertion order
    for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);
    for (Map.Entry<Character, Integer> e : freq.entrySet()) {
        if (e.getValue() == 1) return e.getKey();
    }
    return '\0';  // none found
}
```

</details>

---

🟡 **S5. How do you check if two strings are anagrams?**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: Sort and compare — O(n log n)
public static boolean isAnagram(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    char[] a = s1.toCharArray();
    char[] b = s2.toCharArray();
    Arrays.sort(a);
    Arrays.sort(b);
    return Arrays.equals(a, b);
}

// Method 2: Character frequency — O(n) time, O(1) space (fixed alphabet)
public static boolean isAnagramOptimal(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    int[] count = new int[26];
    for (int i = 0; i < s1.length(); i++) {
        count[s1.charAt(i) - 'a']++;
        count[s2.charAt(i) - 'a']--;
    }
    for (int c : count) if (c != 0) return false;
    return true;
}
// "listen" vs "silent" → true
// "hello" vs "world" → false

// Group anagrams together (interview problem):
public static Map<String, List<String>> groupAnagrams(String[] words) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String word : words) {
        char[] chars = word.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);  // sorted chars as key
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(word);
    }
    return groups;
}
// ["eat", "tea", "tan", "ate", "nat", "bat"]
// → {aet: [eat, tea, ate], ant: [tan, nat], abt: [bat]}
```

</details>

---

🟡 **S6. How do you find all occurrences of a substring?**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: indexOf loop
public static List<Integer> findAll(String text, String pattern) {
    List<Integer> positions = new ArrayList<>();
    int index = 0;
    while ((index = text.indexOf(pattern, index)) != -1) {
        positions.add(index);
        index++;  // advance by 1 to find overlapping matches
        // advance by pattern.length() for non-overlapping
    }
    return positions;
}
// findAll("aababc", "ab") → [1, 3] (non-overlapping)

// Method 2: Regex (for pattern matching)
public static List<Integer> findAllRegex(String text, String pattern) {
    List<Integer> positions = new ArrayList<>();
    Matcher m = Pattern.compile(Pattern.quote(pattern)).matcher(text);
    while (m.find()) {
        positions.add(m.start());
    }
    return positions;
}

// Method 3: KMP algorithm (O(n+m) — for interview bonus points)
public static List<Integer> kmpSearch(String text, String pattern) {
    int n = text.length(), m = pattern.length();
    int[] lps = buildLPS(pattern);
    List<Integer> results = new ArrayList<>();

    int i = 0, j = 0;
    while (i < n) {
        if (text.charAt(i) == pattern.charAt(j)) { i++; j++; }
        if (j == m) {
            results.add(i - j);
            j = lps[j - 1];
        } else if (i < n && text.charAt(i) != pattern.charAt(j)) {
            if (j != 0) j = lps[j - 1];
            else i++;
        }
    }
    return results;
}

private static int[] buildLPS(String pattern) {
    int m = pattern.length();
    int[] lps = new int[m];
    int len = 0, i = 1;
    while (i < m) {
        if (pattern.charAt(i) == pattern.charAt(len)) {
            lps[i++] = ++len;
        } else if (len != 0) {
            len = lps[len - 1];
        } else {
            lps[i++] = 0;
        }
    }
    return lps;
}
```

</details>

---

🔴 **S7. Implement a method to check if a string has all unique characters.**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: HashSet — O(n) time, O(min(n,m)) space (m = charset size)
public static boolean hasUniqueChars(String s) {
    Set<Character> seen = new HashSet<>();
    for (char c : s.toCharArray()) {
        if (!seen.add(c)) return false;  // add returns false if already present
    }
    return true;
}

// Method 2: Bit manipulation for lowercase a-z — O(n) time, O(1) space
public static boolean hasUniqueCharsOptimal(String s) {
    if (s.length() > 26) return false;  // pigeonhole principle

    int checker = 0;
    for (char c : s.toCharArray()) {
        int bit = 1 << (c - 'a');
        if ((checker & bit) != 0) return false;  // bit already set
        checker |= bit;
    }
    return true;
}

// Method 3: Sort and compare adjacent
public static boolean hasUniqueCharsSort(String s) {
    char[] chars = s.toCharArray();
    Arrays.sort(chars);
    for (int i = 1; i < chars.length; i++) {
        if (chars[i] == chars[i - 1]) return false;
    }
    return true;
}
```

</details>

---

🟡 **S8. How do you count words in a string?**

<details><summary>Click to reveal answer</summary>

```java
// Simple: split and count
public static int countWords(String s) {
    if (s == null || s.isBlank()) return 0;
    return s.trim().split("\\s+").length;
}

// Using streams
public static long countWordsStream(String s) {
    if (s == null || s.isBlank()) return 0;
    return Arrays.stream(s.trim().split("\\s+")).count();
}

// Count specific word occurrences (case-insensitive)
public static int countOccurrences(String text, String word) {
    String lowerText = text.toLowerCase();
    String lowerWord = word.toLowerCase();
    int count = 0;
    int index = 0;
    while ((index = lowerText.indexOf(lowerWord, index)) != -1) {
        count++;
        index += lowerWord.length();
    }
    return count;
}

// Regex for word boundary matching (more accurate)
public static int countWordRegex(String text, String word) {
    Pattern p = Pattern.compile("\\b" + Pattern.quote(word) + "\\b",
        Pattern.CASE_INSENSITIVE);
    Matcher m = p.matcher(text);
    int count = 0;
    while (m.find()) count++;
    return count;
}
```

</details>

---

🔴 **S9. What is the longest common prefix of a list of strings?**

<details><summary>Click to reveal answer</summary>

```java
// Method 1: Horizontal scanning
public static String longestCommonPrefix(String[] strs) {
    if (strs == null || strs.length == 0) return "";

    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (!strs[i].startsWith(prefix)) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    }
    return prefix;
}
// ["flower", "flow", "flight"] → "fl"
// ["dog", "car", "racecar"] → ""

// Method 2: Vertical scanning (character by character)
public static String longestCommonPrefixVertical(String[] strs) {
    if (strs == null || strs.length == 0) return "";

    for (int i = 0; i < strs[0].length(); i++) {
        char c = strs[0].charAt(i);
        for (int j = 1; j < strs.length; j++) {
            if (i == strs[j].length() || strs[j].charAt(i) != c) {
                return strs[0].substring(0, i);
            }
        }
    }
    return strs[0];
}
```

</details>

---

🟡 **S10. How do you efficiently validate and parse structured strings?**

<details><summary>Click to reveal answer</summary>

```java
// CSV parsing (simplified — use Apache Commons CSV in production)
public static List<String[]> parseCsv(String csv) {
    return Arrays.stream(csv.split("\n"))
        .map(line -> line.split(",(?=(?:[^\"]*\"[^\"]*\")*[^\"]*$)", -1))
        .collect(Collectors.toList());
}

// JSON-like key=value parsing
public static Map<String, String> parseKeyValue(String input) {
    Map<String, String> result = new HashMap<>();
    Pattern p = Pattern.compile("(\\w+)=(\\w+)");
    Matcher m = p.matcher(input);
    while (m.find()) {
        result.put(m.group(1), m.group(2));
    }
    return result;
}
// "name=Alice age=30 city=NYC" → {name:Alice, age:30, city:NYC}

// Template string substitution
public static String substitute(String template, Map<String, String> vars) {
    Pattern p = Pattern.compile("\\$\\{(\\w+)\\}");
    Matcher m = p.matcher(template);
    StringBuilder sb = new StringBuilder();
    while (m.find()) {
        String varName = m.group(1);
        String replacement = vars.getOrDefault(varName, m.group(0));  // keep if not found
        m.appendReplacement(sb, Matcher.quoteReplacement(replacement));
    }
    m.appendTail(sb);
    return sb.toString();
}
// template: "Hello, ${name}! You have ${count} messages."
// vars: {name: "Alice", count: "5"}
// result: "Hello, Alice! You have 5 messages."
```

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| `s1 == s2` for string comparison | Use `s1.equals(s2)` or `Objects.equals()` |
| String concatenation in loops | Use `StringBuilder` |
| `s.split(",")` missing edge cases | Test with empty values and trailing delimiters |
| `String.matches()` — partial match | `matches()` anchors to full string; use `Pattern.find()` for partial |
| Using regex without compiling `Pattern` | Compile `Pattern` once and reuse |
| `s.substring()` — expecting copy | `substring()` returns a new String (not a view in Java 7+) |
| Null-check before `.equals()` | Use `"literal".equals(variable)` or `Objects.equals()` |
| `trim()` for Unicode whitespace | Use `strip()` (Java 11+) for Unicode-aware trimming |
| Regex special chars unescaped | Use `Pattern.quote(str)` for literal matching |
| `s.toCharArray()` mutation thinking changes String | `toCharArray()` returns a COPY |

---

## ✅ Best Practices

- ✅ Use `Objects.equals(s1, s2)` for null-safe comparison
- ✅ Use `StringBuilder` for building strings in loops
- ✅ Compile `Pattern` once as a static field
- ✅ Use `strip()` instead of `trim()` (Java 11+) for Unicode support
- ✅ Use `String.format()` or text blocks for complex strings
- ✅ Use `Pattern.quote()` when treating a string as a literal pattern
- ✅ Prefer `split("\\s+")` after `trim()` for splitting on whitespace
- ✅ Use `String.join()` or `Collectors.joining()` for building delimited strings
- ✅ Use text blocks (Java 15+) for multi-line strings (SQL, JSON, HTML)
- ✅ Validate regex patterns with unit tests (edge cases matter!)

---

## 🔗 Related Sections

- [Chapter 1: Java Basics](./01-java-basics.md) — String pool memory model
- [Chapter 6: Collections](./06-collections-framework.md) — String as HashMap key
- [Chapter 8: Advanced Java](./08-advanced-java.md) — Java 15+ text blocks

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 6: Collections](./06-collections-framework.md) | [Section README](./README.md) | [Chapter 8: Advanced Java →](./08-advanced-java.md) |
