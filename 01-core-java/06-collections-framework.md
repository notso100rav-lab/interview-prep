# Chapter 6: Collections Framework

> **Section 1: Core Java** | Java Backend Engineer Interview Prep

---

## 📚 Topics Covered

- `List`: `ArrayList` vs `LinkedList`
- `Map`: `HashMap` internals (hashing, buckets, treeification, resizing)
- `ConcurrentHashMap` internals
- `TreeMap`, `LinkedHashMap`, `IdentityHashMap`
- `Set` implementations: `HashSet`, `TreeSet`, `LinkedHashSet`
- `Queue` and `Deque`: `PriorityQueue`, `ArrayDeque`
- `Collections` utility class
- Fail-fast vs fail-safe iterators
- `Comparable` vs `Comparator`
- Immutable/unmodifiable collections

---

## 🔑 Key Concepts at a Glance

```
Collections Framework:

Collection<E>
├── List<E>          (ordered, duplicates allowed, index access)
│   ├── ArrayList    (dynamic array, O(1) get, O(n) add-middle)
│   ├── LinkedList   (doubly linked, O(1) add/remove ends, O(n) get)
│   └── Vector       (synchronized ArrayList, legacy)
├── Set<E>           (no duplicates, no guaranteed order)
│   ├── HashSet      (HashMap backed, O(1) avg ops)
│   ├── LinkedHashSet (insertion order maintained)
│   └── TreeSet      (sorted, O(log n) ops, red-black tree)
└── Queue<E>
    ├── PriorityQueue (heap-based, natural ordering or Comparator)
    └── Deque<E>
        ├── ArrayDeque  (resizable array, faster than Stack/LinkedList)
        └── LinkedList  (also implements Deque)

Map<K,V>             (key-value pairs, keys unique)
├── HashMap          (O(1) avg, no order, null keys allowed)
├── LinkedHashMap    (insertion/access order maintained)
├── TreeMap          (sorted by key, O(log n), red-black tree)
├── ConcurrentHashMap (thread-safe HashMap, segment locking)
├── Hashtable        (synchronized, legacy, no null)
└── WeakHashMap      (weak references to keys, GC-friendly)
```

---

## 🧠 Theory Questions

### HashMap Internals

---

🔴 **Q1. Explain the internal implementation of `HashMap` in detail.**

<details><summary>Click to reveal answer</summary>

`HashMap` uses an array of **linked lists** (or trees for large buckets) to store entries.

**ASCII Diagram:**
```
HashMap internal structure (default capacity 16):

Index  Bucket (LinkedList or Tree)
  0  →  null
  1  →  [Entry{key="cat", val=3, hash=h1, next=Entry{...}}]
  2  →  null
  3  →  [Entry{key="dog", val=1, hash=h2, next=null}]
  4  →  null
  ...
  7  →  [Entry{key="ant", val=2, hash=h3} → Entry{key="bee", val=5, hash=h4}]
         (hash collision at index 7 — two entries in same bucket)
  ...
 15  →  null

When bucket size > TREEIFY_THRESHOLD (8):
  7  →  [TreeNode: balanced red-black tree for O(log n) lookup]
```

**Step-by-step `put(key, value)`:**
```java
// Simplified HashMap.put() logic
public V put(K key, V value) {
    // 1. Compute hash
    int hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    // XOR with upper bits improves distribution for power-of-2 table sizes

    // 2. Determine bucket index
    int index = hash & (capacity - 1);  // faster than hash % capacity (capacity is power of 2)

    // 3. Check for existing key in bucket (linked list traversal)
    for (Entry<K,V> e = table[index]; e != null; e = e.next) {
        if (e.hash == hash && (e.key == key || key.equals(e.key))) {
            V old = e.value;
            e.value = value;  // update existing
            return old;
        }
    }

    // 4. Add new entry at head of list
    table[index] = new Entry<>(hash, key, value, table[index]);
    size++;

    // 5. Resize if load factor exceeded
    if (size > capacity * loadFactor) {
        resize();  // doubles capacity, rehashes all entries
    }
    return null;
}
```

**Key parameters:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| Initial capacity | 16 | Initial array size (always power of 2) |
| Load factor | 0.75 | Resize when `size > capacity * loadFactor` |
| Treeify threshold | 8 | Convert bucket to red-black tree |
| Untreeify threshold | 6 | Convert tree back to list |
| Min treeify capacity | 64 | Only treeify if overall table >= 64 |

**Why load factor 0.75?**
- Lower → fewer collisions, more memory
- Higher → more collisions, less memory
- 0.75 is the empirical sweet spot for time-space tradeoff

**Resizing (rehashing):**
```java
void resize() {
    int newCapacity = capacity * 2;
    Entry[] newTable = new Entry[newCapacity];
    // Rehash all existing entries into new table
    for (Entry<K,V> entry : table) {
        while (entry != null) {
            Entry<K,V> next = entry.next;
            int newIndex = entry.hash & (newCapacity - 1);
            entry.next = newTable[newIndex];
            newTable[newIndex] = entry;
            entry = next;
        }
    }
    table = newTable;
    capacity = newCapacity;
}
```

> 💡 **Interviewer Tip:** Explaining HashMap internals with hash computation, bucket index, collision handling, and treeification will impress most interviewers. Add that Java 8+ uses red-black trees for large buckets (O(log n) worst case vs O(n)).

</details>

---

🟡 **Q2. What is the contract between `equals()` and `hashCode()` in `HashMap`?**

<details><summary>Click to reveal answer</summary>

**Contract:**
1. If `a.equals(b)` → `a.hashCode() == b.hashCode()` (MANDATORY)
2. If `a.hashCode() == b.hashCode()` → `a.equals(b)` MAY be true (hash collision is OK)

**Why it matters for HashMap:**
```java
// HashMap lookup for get(key):
int index = hash(key) & (capacity - 1);  // find bucket using hashCode
// Then find exact match in bucket using equals()

// If hashCode() is inconsistent with equals():
public class BadKey {
    int x;
    BadKey(int x) { this.x = x; }

    @Override
    public boolean equals(Object o) {
        return o instanceof BadKey bk && this.x == bk.x;
    }
    // NO hashCode override!
}

BadKey k1 = new BadKey(5);
BadKey k2 = new BadKey(5);
k1.equals(k2)  // true (same x)

Map<BadKey, String> map = new HashMap<>();
map.put(k1, "hello");
map.get(k2);  // NULL — k1 and k2 hash to different buckets!
              // HashMap can't find k2 even though equals() says they're equal
```

**Correct implementation:**
```java
public class Point {
    int x, y;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);  // consistent with equals
    }
}

// Objects.hash() under the hood:
// return 31 * (31 + x) + y (polynomial rolling hash)

// Java 16+ Records auto-generate equals/hashCode:
record Point(int x, int y) {}  // hashCode automatically consistent with equals
```

**hashCode() best practices:**
- Include all fields used in `equals()`
- Use `Objects.hash(field1, field2, ...)` for simplicity
- For performance-critical code, consider caching hashCode (if object is immutable)

</details>

---

🔴 **Q3. What happens when two keys have the same `hashCode()` but different `equals()`?**

<details><summary>Click to reveal answer</summary>

This is a **hash collision**. Both keys map to the same bucket index. HashMap handles this by storing them as a chain (linked list) or tree in that bucket.

```java
// Example of collision (contrived)
class AlwaysHashSame {
    String value;
    AlwaysHashSame(String v) { this.value = v; }

    @Override
    public int hashCode() { return 42; }  // ALL instances hash to same bucket!

    @Override
    public boolean equals(Object o) {
        return o instanceof AlwaysHashSame a && value.equals(a.value);
    }
}

Map<AlwaysHashSame, Integer> map = new HashMap<>();
map.put(new AlwaysHashSame("a"), 1);
map.put(new AlwaysHashSame("b"), 2);
map.put(new AlwaysHashSame("c"), 3);

// All three entries in the SAME bucket:
// Bucket 42%16 → [Entry("a",1) → Entry("b",2) → Entry("c",3)]
// get("a") must traverse the chain: O(n) instead of O(1)

// Performance degradation:
// 10 entries in same bucket → O(10) per lookup
// Java 8+ mitigates with treeification:
// 8+ entries in same bucket → red-black tree → O(log n)
```

**Real-world collision attack (Hash DoS):**
Attackers could craft inputs that all hash to the same bucket, degrading HashMap from O(1) to O(n) — a DoS attack. Java 8+ randomizes hash seeds for String in some contexts to mitigate this.

</details>

---

🔴 **Q4. How does `ConcurrentHashMap` differ from `HashMap` and `Hashtable`?**

<details><summary>Click to reveal answer</summary>

| Feature | `HashMap` | `Hashtable` | `ConcurrentHashMap` |
|---------|-----------|------------|---------------------|
| Thread-safe | ❌ | ✅ (full sync) | ✅ (segment sync) |
| Null keys | ✅ 1 null key | ❌ | ❌ |
| Null values | ✅ | ❌ | ❌ |
| Performance | Best (no sync) | Worst (full sync) | Good (partial sync) |
| Iterator | Fail-fast | Fail-safe | Fail-safe |
| Java version | 1.2 | 1.0 (legacy) | 1.5+ |

**ConcurrentHashMap Java 7 (segment locking):**
```
16 segments, each with its own lock
Segment 0: [bucket0...bucket3]
Segment 1: [bucket4...bucket7]
...
Segment 15: [bucket60...bucket63]

Two threads can write to different segments simultaneously
```

**ConcurrentHashMap Java 8+ (node-level locking):**
```java
// Key changes in Java 8:
// - Dropped Segment locking
// - Uses CAS (Compare-And-Swap) for adds to empty buckets
// - Uses synchronized on first node of bucket for collision chains
// - Red-black tree for large buckets (same as HashMap)

// Put operation (simplified):
void put(K key, V value) {
    // Find bucket
    int hash = spread(key.hashCode());
    int index = hash & (n - 1);

    Node<K,V> bucket = table[index];

    if (bucket == null) {
        // CAS for empty bucket — no lock needed!
        if (casTabAt(table, index, null, new Node<>(hash, key, value))) return;
    } else {
        // Synchronized on bucket head for non-empty bucket
        synchronized (bucket) {
            // insert or update
        }
    }
}
```

**Atomic operations in ConcurrentHashMap:**
```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic compute — replaces get + check + put pattern
map.compute("key", (k, v) -> v == null ? 1 : v + 1);

// Atomic computeIfAbsent — only adds if absent
map.computeIfAbsent("key", k -> expensiveComputation(k));

// Atomic merge
map.merge("key", 1, Integer::sum);  // add 1 to existing, or init to 1

// compare-and-swap style
map.putIfAbsent("key", defaultValue);
map.replace("key", oldValue, newValue);  // conditional replace
```

> 💡 **Interviewer Tip:** Always mention that `ConcurrentHashMap` is not a silver bullet for all concurrency issues. Compound operations (check-then-act) still need synchronization unless you use `compute()`, `computeIfAbsent()`, or `merge()`.

</details>

---

### Map Implementations

---

🟡 **Q5. When would you use `TreeMap`, `LinkedHashMap`, or `HashMap`?**

<details><summary>Click to reveal answer</summary>

| Feature | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---------|-----------|----------------|----------|
| **Order** | None | Insertion order (or access order) | Sorted by key |
| **Performance** | O(1) avg | O(1) avg | O(log n) |
| **Null keys** | 1 allowed | 1 allowed | ❌ (keys must be comparable) |
| **Use case** | General-purpose | LRU cache, preserve insertion order | Range queries, sorted data |
| **Interface** | `Map` | `Map` | `SortedMap`, `NavigableMap` |

```java
// HashMap — no order guarantee
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("banana", 2); hashMap.put("apple", 1); hashMap.put("cherry", 3);
// Iteration order: random (hash-based)

// LinkedHashMap — insertion order
Map<String, Integer> linkedMap = new LinkedHashMap<>();
linkedMap.put("banana", 2); linkedMap.put("apple", 1); linkedMap.put("cherry", 3);
// Iteration: banana, apple, cherry (insertion order)

// LinkedHashMap — access order (LRU cache pattern)
Map<String, Integer> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > 100;  // evict when > 100 entries
    }
};

// TreeMap — sorted by key
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("banana", 2); treeMap.put("apple", 1); treeMap.put("cherry", 3);
// Iteration: apple, banana, cherry (alphabetical)

// TreeMap navigation methods
TreeMap<Integer, String> navMap = new TreeMap<>();
navMap.floorKey(5);       // largest key <= 5
navMap.ceilingKey(5);     // smallest key >= 5
navMap.headMap(5);        // all keys < 5
navMap.tailMap(5);        // all keys >= 5
navMap.subMap(3, 7);      // keys in range [3, 7)
```

</details>

---

### List Implementations

---

🟡 **Q6. What is the difference between `ArrayList` and `LinkedList`? When to use each?**

<details><summary>Click to reveal answer</summary>

| Operation | `ArrayList` | `LinkedList` |
|-----------|------------|--------------|
| `get(i)` | O(1) | O(n) |
| `add(end)` | O(1) amortized | O(1) |
| `add(middle)` | O(n) (shift elements) | O(n) (traverse) + O(1) (insert) |
| `remove(middle)` | O(n) (shift elements) | O(n) (traverse) + O(1) (remove) |
| `contains(obj)` | O(n) | O(n) |
| Memory | Compact (array) | Higher (node objects + pointers) |
| Cache efficiency | ✅ Excellent (contiguous) | ❌ Poor (scattered nodes) |

**Internal structure:**
```
ArrayList:
  [0]=a [1]=b [2]=c [3]=d [4]=null [5]=null  (capacity > size)
  // add at [2]: shift c,d right → [0]=a [1]=b [2]=new [3]=c [4]=d

LinkedList:
  head → [a|→b] ↔ [b|→a,→c] ↔ [c|→b,→d] ↔ [d|→c] ← tail
  // add before c: O(1) once you have the node reference
```

**When to use `ArrayList`:**
- Most use cases (better cache performance and memory efficiency)
- Frequent random access by index
- Iteration-heavy workloads
- Infrequent insertions/deletions in the middle

**When to use `LinkedList`:**
- Frequent insertions/deletions at the **head** or **tail** (queue/deque operations)
- Implement `Queue` or `Deque` (though `ArrayDeque` is usually better)
- When you need `ListIterator.add()` in the middle and have iterator reference

**In practice:** `ArrayList` wins in almost all cases due to cache locality. Even for frequent insertions, the constant factor difference means `ArrayList` is often faster.

```java
// For queue: use ArrayDeque, not LinkedList
Deque<String> queue = new ArrayDeque<>();
queue.addFirst("first");  // O(1)
queue.addLast("last");    // O(1)
queue.removeFirst();      // O(1)
```

</details>

---

### Fail-Fast vs Fail-Safe Iterators

---

🟡 **Q7. What is the difference between fail-fast and fail-safe iterators?**

<details><summary>Click to reveal answer</summary>

**Fail-fast iterators** throw `ConcurrentModificationException` if the collection is modified during iteration (other than through the iterator itself).

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
Iterator<String> it = list.iterator();

// Modify list while iterating:
list.add("d");  // structural modification → modCount changes

while (it.hasNext()) {
    System.out.println(it.next());  // ConcurrentModificationException!
}
```

**How fail-fast works:**
```
ArrayList maintains: modCount (modification counter)
Iterator stores: expectedModCount = modCount at creation time
Each next()/remove() call: checks modCount == expectedModCount
If not equal: throw ConcurrentModificationException
```

**Fail-safe iterators** work on a snapshot/copy, so modifications don't affect ongoing iteration.

```java
// CopyOnWriteArrayList: creates a new array on each modification
List<String> cowList = new CopyOnWriteArrayList<>(List.of("a", "b", "c"));
Iterator<String> it = cowList.iterator();  // iterator has snapshot

cowList.add("d");  // creates a new array, doesn't affect existing iterator

while (it.hasNext()) {
    System.out.println(it.next());  // prints a, b, c (original snapshot)
}

// ConcurrentHashMap's iterators are also weakly consistent (fail-safe)
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
// iterator may or may not reflect modifications made after iterator creation
```

**Removing during iteration — the right way:**
```java
// Using Iterator.remove() — safe for fail-fast
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.startsWith("b")) {
        it.remove();  // safe — updates modCount and expectedModCount together
    }
}

// Using removeIf() — Java 8+, cleanest
list.removeIf(s -> s.startsWith("b"));

// Using stream filter
list = list.stream().filter(s -> !s.startsWith("b")).collect(Collectors.toList());
```

| | Fail-Fast | Fail-Safe |
|-|-----------|-----------|
| Collections | `ArrayList`, `HashMap`, `HashSet` | `CopyOnWriteArrayList`, `ConcurrentHashMap` |
| Modification during iteration | `ConcurrentModificationException` | Works (on snapshot) |
| Memory | Efficient | Extra copy (COW) |
| Use case | Single-threaded iteration | Multi-threaded, rare writes |

</details>

---

### Comparable vs Comparator

---

🟡 **Q8. What is the difference between `Comparable` and `Comparator`?**

<details><summary>Click to reveal answer</summary>

| | `Comparable<T>` | `Comparator<T>` |
|-|----------------|----------------|
| Package | `java.lang` | `java.util` |
| Method | `int compareTo(T o)` | `int compare(T o1, T o2)` |
| Implemented by | The class itself | Separate class/lambda |
| Order defined | Natural ordering | Custom ordering |
| Multiple orderings | ❌ One per class | ✅ Multiple comparators |
| Modifying the class | Required | Not required |

```java
// Comparable — natural ordering (defined in the class)
public class Student implements Comparable<Student> {
    String name;
    double gpa;

    @Override
    public int compareTo(Student other) {
        return Double.compare(this.gpa, other.gpa);  // sort by GPA ascending
    }
}

List<Student> students = new ArrayList<>(...);
Collections.sort(students);  // uses compareTo (natural order)
TreeSet<Student> byGpa = new TreeSet<>(students);  // needs Comparable

// Comparator — custom ordering (external)
Comparator<Student> byName = Comparator.comparing(s -> s.name);
Comparator<Student> byGpaDesc = Comparator.comparingDouble(Student::getGpa).reversed();
Comparator<Student> byNameThenGpa = Comparator.comparing(Student::getName)
    .thenComparingDouble(Student::getGpa);

students.sort(byNameThenGpa);  // sort using external comparator
TreeSet<Student> byName2 = new TreeSet<>(byName);

// Null handling
Comparator<String> nullsFirst = Comparator.nullsFirst(Comparator.naturalOrder());
Comparator<String> nullsLast = Comparator.nullsLast(Comparator.reverseOrder());
```

**`compareTo` / `compare` return value:**
- Negative: `this < other` (or `o1 < o2`)
- Zero: equal
- Positive: `this > other` (or `o1 > o2`)

**Consistency with `equals`:** `compareTo` returning 0 should ideally mean `equals` returns `true` (required for `TreeSet`/`TreeMap` correctness, but not enforced).

</details>

---

### Set Implementations

---

🟡 **Q9. How does `HashSet` work internally?**

<details><summary>Click to reveal answer</summary>

`HashSet` is backed by a `HashMap` where elements are stored as **keys** with a constant dummy value.

```java
// HashSet internal implementation (simplified)
public class HashSet<E> {
    private static final Object PRESENT = new Object();  // dummy value
    private final HashMap<E, Object> map = new HashMap<>();

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // returns true if key was new
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }

    public int size() { return map.size(); }
}
```

**This means:**
- Same O(1) average performance as HashMap
- Same `equals()`/`hashCode()` requirements
- At most one `null` element allowed

**Set implementations:**
```java
// HashSet — fastest, no order
Set<String> hash = new HashSet<>();

// LinkedHashSet — insertion order
Set<String> linked = new LinkedHashSet<>();

// TreeSet — sorted, O(log n), needs Comparable or Comparator
Set<String> sorted = new TreeSet<>();
((TreeSet<String>)sorted).floor("m");    // largest element <= "m"
((TreeSet<String>)sorted).ceiling("m");  // smallest element >= "m"

// EnumSet — highly optimized for enum types (bit vector internally)
EnumSet<DayOfWeek> weekdays = EnumSet.of(MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY);

// Immutable set (Java 9+)
Set<String> immutable = Set.of("a", "b", "c");
// throws UnsupportedOperationException on modification
```

</details>

---

### Queue and Deque

---

🟡 **Q10. What is `PriorityQueue` and how does it work?**

<details><summary>Click to reveal answer</summary>

`PriorityQueue` is a **min-heap** (smallest element at top by default) that orders elements by natural ordering or a `Comparator`.

```java
// Min-heap (default — smallest element at head)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5); minHeap.offer(1); minHeap.offer(3);
System.out.println(minHeap.poll());  // 1 (smallest)
System.out.println(minHeap.poll());  // 3
System.out.println(minHeap.poll());  // 5

// Max-heap (using reversed comparator)
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5); maxHeap.offer(1); maxHeap.offer(3);
System.out.println(maxHeap.poll());  // 5 (largest)

// Priority queue of objects
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);

// Key operations:
pq.offer(e);     // add (enqueue), O(log n)
pq.peek();       // view head without removing, O(1)
pq.poll();       // remove head, O(log n)
pq.contains(e);  // O(n) — no fast lookup
pq.size();       // O(1)

// Common interview problems using PriorityQueue:
// - Kth largest element
// - Merge K sorted lists
// - Top K frequent elements
// - Dijkstra's algorithm
// - Huffman coding
```

**Internal structure (binary heap as array):**
```
Heap:   [1, 3, 5, 7, 4]
Array:  [_, 1, 3, 5, 7, 4]  (1-indexed internally)
         parent(i) = i/2
         leftChild(i) = 2i
         rightChild(i) = 2i+1
```

</details>

---

### Collections Utility

---

🟡 **Q11. What is the difference between `Collections.unmodifiableList()` and `List.of()`?**

<details><summary>Click to reveal answer</summary>

| Feature | `Collections.unmodifiableList()` | `List.of()` (Java 9+) |
|---------|----------------------------------|----------------------|
| Mutation of wrapper | ❌ throws `UnsupportedOperationException` | ❌ throws `UnsupportedOperationException` |
| Backed by original list | ✅ Yes — changes to original VISIBLE | ❌ No — truly immutable |
| Null elements | ✅ Allowed (if original has them) | ❌ throws `NullPointerException` |
| Serializable | ✅ | ✅ |
| Random access efficient | Depends on backing | ✅ |

```java
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));

// unmodifiableList — view of mutable list
List<String> unmod = Collections.unmodifiableList(mutable);
unmod.add("d");    // UnsupportedOperationException
mutable.add("d");  // WORKS — and unmod NOW shows "d" (it's a view!)
System.out.println(unmod.size());  // 4

// List.of() — truly immutable, fixed content
List<String> immutable = List.of("a", "b", "c");
immutable.add("d");  // UnsupportedOperationException
// Content can never change

// List.copyOf() — immutable copy
List<String> copy = List.copyOf(mutable);  // snapshot, independent
mutable.add("e");
System.out.println(copy.size());  // still 4

// Defensive copy pattern:
public class ImmutableContainer {
    private final List<String> items;

    public ImmutableContainer(List<String> items) {
        this.items = List.copyOf(items);  // defensive copy
    }

    public List<String> getItems() {
        return items;  // already immutable, safe to return directly
    }
}
```

</details>

---

## 🎭 Scenario-Based Questions

---

🔴 **S1. Why does this HashMap behave unexpectedly?**

```java
public class MutableKey {
    int value;
    MutableKey(int v) { this.value = v; }

    @Override
    public boolean equals(Object o) {
        return o instanceof MutableKey mk && this.value == mk.value;
    }

    @Override
    public int hashCode() { return value; }
}

Map<MutableKey, String> map = new HashMap<>();
MutableKey key = new MutableKey(5);
map.put(key, "hello");

key.value = 10;  // MUTATE the key!

System.out.println(map.get(key));        // A
System.out.println(map.containsKey(key)); // B
System.out.println(map.size());           // C
```

<details><summary>Click to reveal answer</summary>

- **A:** `null` — key was put with hash for `value=5` (bucket 5 % 16 = 5). After mutation, hash is for `value=10` (bucket 10), where no entry exists.
- **B:** `false` — same reason, looks in wrong bucket.
- **C:** `1` — entry still exists, just lost!

**This is a classic bug:** Never use mutable objects as HashMap keys. The key's hashCode() changes after mutation, making the entry unreachable.

**Best practices for Map keys:**
- Use immutable objects (String, Integer, UUID, LocalDate)
- If mutable, ensure hashCode/equals are based on immutable state
- Use `final` fields that form the key's identity

```java
// Good: String keys are immutable
Map<String, User> userByName = new HashMap<>();
// Good: record (immutable by nature)
Map<Point, String> grid = new HashMap<>();  // record Point(int x, int y) {}
```

</details>

---

🟡 **S2. What's the output?**

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c", "d"));
for (String s : list) {
    if (s.equals("b")) {
        list.remove(s);
    }
}
System.out.println(list);
```

<details><summary>Click to reveal answer</summary>

This throws `ConcurrentModificationException`!

The enhanced for loop uses an `Iterator`. When `list.remove(s)` is called (not `iterator.remove()`), it modifies `modCount`. The iterator's next `hasNext()` or `next()` call detects the change and throws.

**Fixes:**
```java
// 1. Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();  // safe
}

// 2. removeIf (Java 8+) — cleanest
list.removeIf(s -> s.equals("b"));

// 3. Collect items to remove, then removeAll
list.removeAll(List.of("b"));

// 4. Stream filter
list = list.stream().filter(s -> !s.equals("b")).collect(Collectors.toList());
```

</details>

---

🟡 **S3. How do you make `HashMap` thread-safe? Compare your options.**

<details><summary>Click to reveal answer</summary>

```java
// Option 1: ConcurrentHashMap (BEST for most cases)
Map<String, Integer> map = new ConcurrentHashMap<>();
// - Segment/node-level locking (not whole map)
// - Atomic compute operations
// - Good read performance
map.compute("key", (k, v) -> v == null ? 1 : v + 1);

// Option 2: Collections.synchronizedMap (wraps HashMap, worst performance)
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// - Every operation locks the entire map
// - Must synchronize externally for compound operations
synchronized (syncMap) {
    if (!syncMap.containsKey("key")) {
        syncMap.put("key", 1);  // must synchronize compound operations
    }
}

// Option 3: CopyOnWriteMap (no standard impl, but concept)
// - Write creates a copy of backing array
// - Read requires no lock
// - Good for read-heavy, write-rare scenarios

// Option 4: Hashtable (legacy, avoid)
Map<String, Integer> hashtable = new Hashtable<>();
// - Whole-map synchronization on every operation
// - No null keys or values

// Comparison:
// ConcurrentHashMap: best general-purpose concurrent map
// synchronizedMap: ok for occasional concurrent access, simple use cases
// Hashtable: legacy, don't use in new code
```

**Choosing between them:**
- Write-heavy + read-heavy: `ConcurrentHashMap`
- Read-heavy + write-rare: consider `CopyOnWriteArrayList` pattern for data structures
- Single-threaded or external synchronization: plain `HashMap`

</details>

---

🔴 **S4. Design an LRU Cache using `LinkedHashMap`.**

<details><summary>Click to reveal answer</summary>

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        // accessOrder=true: most recently accessed entry at end
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    public V get(K key) {
        // LinkedHashMap.get() with accessOrder moves accessed entry to end
        return super.getOrDefault(key, null);
    }

    public void put(K key, V value) {
        super.put(key, value);
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;  // evict LRU (head) when over capacity
    }
}

// Usage:
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");
cache.get(1);     // access 1 → moves to end
cache.put(4, "four");  // evicts 2 (LRU)
System.out.println(cache.containsKey(2));  // false — evicted

// Thread-safe version:
Map<K, V> threadSafeCache = Collections.synchronizedMap(new LRUCache<>(100));
// OR: ConcurrentLinkedHashMap (Guava)

// Alternative: implement with HashMap + Doubly LinkedList (O(1) operations)
public class LRUCacheManual<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null);  // dummy
    private final Node<K, V> tail = new Node<>(null, null);  // dummy

    // ... full implementation with doubly linked list manipulation
}
```

</details>

---

🟡 **S5. What's the output?**

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(5, "five");
map.put(2, "two");
map.put(8, "eight");
map.put(1, "one");
map.put(4, "four");

System.out.println(map);                    // A
System.out.println(map.firstKey());         // B
System.out.println(map.lastKey());          // C
System.out.println(map.headMap(4));         // D
System.out.println(map.tailMap(4));         // E
System.out.println(map.floorKey(3));        // F
System.out.println(map.ceilingKey(3));      // G
```

<details><summary>Click to reveal answer</summary>

TreeMap always maintains keys in sorted (ascending) order:

- **A:** `{1=one, 2=two, 4=four, 5=five, 8=eight}`
- **B:** `1`
- **C:** `8`
- **D:** `{1=one, 2=two}` (exclusive of 4 — headMap is [min, fromKey))
- **E:** `{4=four, 5=five, 8=eight}` (inclusive of 4 — tailMap is [fromKey, max])
- **F:** `2` (largest key <= 3, since there's no key 3)
- **G:** `4` (smallest key >= 3, since there's no key 3)

</details>

---

🟡 **S6. How do you find duplicates in a list efficiently?**

<details><summary>Click to reveal answer</summary>

```java
List<Integer> list = Arrays.asList(1, 2, 3, 2, 4, 3, 5, 1);

// Method 1: HashSet — O(n) time, O(n) space
public static <T> Set<T> findDuplicates(List<T> list) {
    Set<T> seen = new HashSet<>();
    Set<T> duplicates = new LinkedHashSet<>();  // preserves order of first duplicate
    for (T item : list) {
        if (!seen.add(item)) {  // add returns false if already present
            duplicates.add(item);
        }
    }
    return duplicates;
}
// Result: [2, 3, 1]

// Method 2: Stream and collectors
Map<Integer, Long> freq = list.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
Set<Integer> duplicates = freq.entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .collect(Collectors.toSet());
// Result: {1, 2, 3}

// Method 3: Check all duplicates (using frequency)
list.stream()
    .filter(x -> Collections.frequency(list, x) > 1)
    .distinct()
    .collect(Collectors.toList());
// INEFFICIENT: Collections.frequency is O(n), total O(n²)
```

</details>

---

🔴 **S7. What is the performance of various `PriorityQueue` operations, and how would you implement "Kth largest element"?**

<details><summary>Click to reveal answer</summary>

**PriorityQueue complexities:**
| Operation | Time |
|-----------|------|
| `offer(e)` | O(log n) |
| `peek()` | O(1) |
| `poll()` | O(log n) |
| `contains(e)` | O(n) |
| Build from collection | O(n) |

**Kth Largest Element:**
```java
// Approach 1: Min-heap of size K — O(n log k)
public int findKthLargest(int[] nums, int k) {
    // Min-heap of size k keeps the k largest elements
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll();  // remove smallest — keeps k largest
        }
    }

    return minHeap.peek();  // smallest of the k largest = kth largest
}

// Example: nums=[3,2,1,5,6,4], k=2
// After processing: heap=[5,6] → peek=5 → 2nd largest is 5 ✓

// Approach 2: Max-heap — O(n + k log n)
public int findKthLargestMaxHeap(int[] nums, int k) {
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
    for (int n : nums) maxHeap.offer(n);
    for (int i = 0; i < k - 1; i++) maxHeap.poll();
    return maxHeap.peek();
}

// Approach 3: QuickSelect — O(n) average
public int quickSelect(int[] nums, int k) {
    return select(nums, 0, nums.length - 1, nums.length - k);
}
private int select(int[] nums, int lo, int hi, int k) {
    int p = partition(nums, lo, hi);
    if (p == k) return nums[p];
    return p < k ? select(nums, p + 1, hi, k) : select(nums, lo, p - 1, k);
}
```

</details>

---

🟡 **S8. How does `Arrays.asList()` differ from `List.of()`?**

<details><summary>Click to reveal answer</summary>

```java
// Arrays.asList() — fixed-size list backed by array
List<String> asList = Arrays.asList("a", "b", "c");
asList.set(0, "A");         // OK — mutation of elements allowed
asList.add("d");             // UnsupportedOperationException — can't change size!
asList.remove("a");          // UnsupportedOperationException — can't change size!

// List.of() — truly immutable
List<String> listOf = List.of("a", "b", "c");
listOf.set(0, "A");          // UnsupportedOperationException
listOf.add("d");             // UnsupportedOperationException
listOf.contains(null);       // NullPointerException (null not allowed)

// Comparison:
// Arrays.asList: fixed-size, mutable elements, backed by array (changes propagate!)
// List.of: fully immutable, null not allowed, no backing array concern

// The "backed by array" gotcha:
String[] arr = {"a", "b", "c"};
List<String> list = Arrays.asList(arr);
arr[0] = "X";
System.out.println(list.get(0));  // "X" — changes propagate!

// To get a truly independent mutable list:
List<String> mutableList = new ArrayList<>(Arrays.asList("a", "b", "c"));
// or
List<String> mutableList2 = new ArrayList<>(List.of("a", "b", "c"));
```

</details>

---

## 🚨 Common Mistakes Summary

| Mistake | Correct Approach |
|---------|-----------------|
| Using mutable objects as HashMap keys | Use immutable keys (String, Integer, records) |
| `==` to compare elements in collections | Use `.equals()` |
| Modifying collection during iteration | Use `Iterator.remove()` or `removeIf()` |
| Not overriding `hashCode()` with `equals()` | Always override both |
| `Arrays.asList()` thinking it's fully mutable | Can't add/remove, only set |
| `Collections.unmodifiableList()` thinking it's immutable | Original list can still be modified |
| Using `LinkedList` assuming O(1) middle insert | Still O(n) to find position |
| `new HashMap<>(capacity)` without considering load factor | `new HashMap<>(expectedSize / 0.75 + 1)` |
| `Collections.synchronizedMap` for compound operations | Use `ConcurrentHashMap.compute()` |
| ConcurrentModificationException in enhanced for | Use `removeIf()` or `Iterator.remove()` |

---

## ✅ Best Practices

- ✅ Use `HashMap` by default; switch to `LinkedHashMap`/`TreeMap` when ordering needed
- ✅ Use `ConcurrentHashMap` for thread-safe maps; avoid `synchronizedMap`
- ✅ Always override both `equals()` and `hashCode()` together
- ✅ Use immutable keys in maps (String, Integer, records)
- ✅ Use `List.of()` / `Set.of()` / `Map.of()` for immutable collections
- ✅ Use `List.copyOf()` for defensive copies
- ✅ Use `ArrayDeque` instead of `Stack` or `LinkedList` for queue/stack
- ✅ Use `EnumSet`/`EnumMap` when keys are enum values
- ✅ Pre-size `HashMap` when size is known: `new HashMap<>(size * 4 / 3 + 1)`
- ✅ Use `computeIfAbsent()` for map-of-lists patterns

---

## 🔗 Related Sections

- [Chapter 1: Java Basics](./01-java-basics.md) — `equals()` and `hashCode()` basics
- [Chapter 2: OOP](./02-oops.md) — `Comparable` and `Comparator` design
- [Chapter 3: Multithreading](./03-multithreading-and-concurrency.md) — `ConcurrentHashMap`, thread-safe collections

---

## 🔗 Navigation

| ← Previous | Home | Next → |
|-----------|------|--------|
| [Chapter 5: Exception Handling](./05-exception-handling.md) | [Section README](./README.md) | [Chapter 7: Strings & Regex →](./07-strings-and-regex.md) |
