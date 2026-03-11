# Java Core — Interview Q&A for SDE-2 (5+ Years Experience)

> Target companies: Razorpay, CRED, Groww, PhonePe, Swiggy, and similar product-based startups.
> Total Questions: 35+

---

## Table of Contents

1. [OOP Concepts](#1-oop-concepts)
2. [Collections Framework Internals](#2-collections-framework-internals)
3. [Strings](#3-strings)
4. [Exception Handling](#4-exception-handling)
5. [Generics](#5-generics)
6. [Serialization](#6-serialization)
7. [Equals and HashCode Contract](#7-equals-and-hashcode-contract)
8. [Immutability](#8-immutability)

---

# 1. OOP Concepts

---

## Q1: Explain the four pillars of OOP with a single real-world Java example. [Medium]

**Encapsulation** — hiding internal state and requiring all interaction through methods.
**Abstraction** — exposing only relevant behavior, hiding implementation details.
**Inheritance** — acquiring properties and behavior from a parent.
**Polymorphism** — one interface, many implementations.

```java
// Abstraction — expose what, hide how
abstract class Payment {
    // Encapsulation — private state, controlled access
    private String txnId;
    private double amount;

    public Payment(String txnId, double amount) {
        this.txnId = txnId;
        this.amount = amount;
    }

    public double getAmount() { return amount; }
    public String getTxnId() { return txnId; }

    // Abstract method — subclass decides the "how"
    public abstract boolean process();
}

// Inheritance — UpiPayment IS-A Payment
class UpiPayment extends Payment {
    private String vpa;

    public UpiPayment(String txnId, double amount, String vpa) {
        super(txnId, amount);
        this.vpa = vpa;
    }

    // Polymorphism (runtime) — same method, different behavior
    @Override
    public boolean process() {
        System.out.println("Processing UPI: " + vpa + " ₹" + getAmount());
        return true;
    }
}

class CardPayment extends Payment {
    private String last4;

    public CardPayment(String txnId, double amount, String last4) {
        super(txnId, amount);
        this.last4 = last4;
    }

    @Override
    public boolean process() {
        System.out.println("Processing Card ending " + last4 + " ₹" + getAmount());
        return true;
    }
}

// Client code — works with the abstraction, not the implementation
public class PaymentService {
    public void settle(Payment payment) {   // polymorphic reference
        payment.process();
    }
}
```

> **Interview Tip:** Interviewers at Razorpay/PhonePe love payment-domain examples. Use domain-relevant models wherever possible.

---

## Q2: Compile-time vs Runtime Polymorphism — how does the JVM resolve each? [Medium]

| Aspect | Compile-time (Static) | Runtime (Dynamic) |
|---|---|---|
| Mechanism | Method **overloading** | Method **overriding** |
| Binding | Early binding by compiler | Late binding by JVM via vtable |
| Resolved using | Reference type, number/type of args | Actual object type at runtime |

```java
class Calculator {
    // Compile-time — overloading (same name, different signatures)
    int add(int a, int b)           { return a + b; }
    double add(double a, double b)  { return a + b; }

    // The compiler picks the right method based on argument types
}

class Animal {
    void speak() { System.out.println("..."); }
}

class Dog extends Animal {
    @Override
    void speak() { System.out.println("Bark!"); }  // runtime polymorphism
}

// At runtime, JVM checks the actual object
Animal a = new Dog();
a.speak();  // prints "Bark!" — resolved at RUNTIME
```

**Common follow-up:** *"Can we override static methods?"*
No. Static methods are resolved by reference type at compile time (method hiding, not overriding).

> **Key Takeaway:** Overloading = compile time, Overriding = runtime. The JVM uses a virtual method table (vtable) for runtime dispatch.

---

## Q3: Abstract Class vs Interface — what changed in Java 8 and beyond? [Medium]

| Feature | Abstract Class | Interface (Java 8+) |
|---|---|---|
| Methods | Abstract + concrete | Abstract + `default` + `static` (Java 9: `private`) |
| State | Instance variables | Only `public static final` constants |
| Constructor | Yes | No |
| Multiple inheritance | No (single extends) | Yes (multiple implements) |
| Access modifiers | Any | Methods are `public` by default |

**When to use which:**
- **Abstract class** — when classes share **state** and **behavior** (e.g., `AbstractPayment` with `txnId`).
- **Interface** — when unrelated classes share a **capability** (e.g., `Retryable`, `Auditable`).

```java
public interface Retryable {
    int maxRetries();

    // Java 8 default method — backward-compatible evolution
    default boolean shouldRetry(int attempt) {
        return attempt < maxRetries();
    }

    // Java 9 private helper (implementation detail)
    private void logRetry(int attempt) {
        System.out.println("Retry attempt: " + attempt);
    }

    // Static factory-style utility
    static Retryable withMaxRetries(int n) {
        return () -> n;  // functional interface usage
    }
}
```

> **Interview Tip:** Always mention Java 8 `default`/`static` methods and Java 9 `private` methods. Many candidates miss these.

---

## Q4: What is the Diamond Problem and how does Java handle it? [Hard]

The diamond problem arises when a class inherits from two sources that provide the same method.

**With classes** — Java prevents this entirely: only single class inheritance is allowed.

**With interfaces (Java 8+)** — the problem reappears with `default` methods.

```java
interface PaymentGateway {
    default String status() { return "Gateway: OK"; }
}

interface FraudChecker {
    default String status() { return "Fraud: CLEAR"; }
}

// Compiler ERROR if we don't resolve ambiguity
class PaymentProcessor implements PaymentGateway, FraudChecker {
    @Override
    public String status() {
        // Must explicitly choose or combine
        return PaymentGateway.super.status() + " | " + FraudChecker.super.status();
    }
}
```

**Resolution rules:**
1. **Class wins** — a concrete method in a class beats any default method.
2. **Sub-interface wins** — a more specific interface's default beats a less specific one.
3. **Must override** — if ambiguity remains, the implementing class must provide its own.

> **Key Takeaway:** Java forces you to resolve diamond conflicts explicitly. The `InterfaceName.super.method()` syntax lets you delegate to a specific parent.

---

# 2. Collections Framework Internals

---

## Q5: Explain HashMap internal working in detail — hashing, buckets, collisions, and Java 8 treeification. [Hard]

This is THE most asked question at product companies. Know it cold.

**Internal structure:**
- An array of `Node<K,V>[]` called `table` (default initial capacity: 16).
- Each index is a **bucket** that holds a linked list (or tree) of entries.

**put(key, value) flow:**

```
1. Compute hash:  hash = key.hashCode() ^ (key.hashCode() >>> 16)
   (upper bits are mixed into lower bits to reduce collisions)
2. Find bucket:   index = hash & (capacity - 1)
   (bitwise AND — works because capacity is always a power of 2)
3. If bucket is empty → insert new Node
4. If bucket has entries → walk the chain:
   a. If key matches (hash equal AND equals() returns true) → replace value
   b. Else → append to end of chain (Java 8: tail insertion; Java 7 was head insertion)
5. If chain length ≥ TREEIFY_THRESHOLD (8) AND table size ≥ 64 → convert to Red-Black tree
6. If size > capacity × loadFactor (0.75) → resize (double the table, rehash all entries)
```

```java
// Simplified view of what happens internally
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// Bucket index
int index = hash & (table.length - 1);
```

**Java 8 treeification:**
- When a single bucket has ≥ 8 nodes AND the table size is ≥ 64, the linked list converts to a balanced **Red-Black Tree**.
- Lookup in that bucket goes from O(n) → O(log n).
- When the tree shrinks to ≤ 6 nodes (UNTREEIFY_THRESHOLD), it converts back to a linked list.

**Rehashing:**
- Triggered when `size > capacity * 0.75`.
- Table doubles in size. Each entry is re-placed: since capacity is `2^n`, each entry either stays at the same index or moves to `index + oldCapacity`.

**Common follow-up:** *"Why is initial capacity a power of 2?"*
So that `hash & (capacity - 1)` works as a fast modulus operation. If capacity is 16 (binary `10000`), then `capacity - 1` is `01111`, and AND keeps only the lower bits.

**Common follow-up:** *"What if two keys have the same hashCode AND equals returns true?"*
The value is replaced; no duplicate key exists.

**Common follow-up:** *"Can null be a key?"*
Yes, exactly one `null` key is allowed; it always maps to bucket 0.

> **Interview Tip:** Draw the bucket array and a chain on paper during on-site rounds. Mention treeification thresholds (8 to tree, 6 back to list) and the hash perturbation (XOR with unsigned right shift) — it shows deep understanding.

---

## Q6: How does ConcurrentHashMap work? How is it different from `Collections.synchronizedMap()`? [Hard]

| Feature | `synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|
| Locking | Single lock on entire map | Per-bucket (segment-free since Java 8) |
| Read concurrency | Blocked during writes | Lock-free reads (volatile + CAS) |
| Null keys/values | Allowed | **Not allowed** (ambiguity with `null` return) |
| Iterator | Fail-fast | **Weakly consistent** (fail-safe) |
| Performance | Poor under contention | Excellent under contention |

**Java 8+ internals:**
- No more `Segment` array (Java 7 approach). Uses a `Node[]` table like `HashMap`.
- **Reads** use `volatile` reads — no locking at all.
- **Writes** use **CAS (Compare-And-Swap)** on the bucket head. If CAS fails (contention), it falls back to `synchronized` on the specific bucket's head node.
- Treeification works the same way (threshold 8).

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic compound operations — this is why CHM is preferred
map.putIfAbsent("orders", 0);
map.compute("orders", (key, val) -> val + 1);       // atomic read-modify-write
map.merge("orders", 1, Integer::sum);                // atomic merge
```

**Common follow-up:** *"Why are null keys/values not allowed?"*
Because `get(key)` returning `null` is ambiguous — does the key not exist, or is the value `null`? In a concurrent context, you can't call `containsKey()` and `get()` atomically to distinguish.

> **Key Takeaway:** ConcurrentHashMap uses CAS + per-bucket synchronization (not segment locks). Always use atomic compound operations (`compute`, `merge`, `putIfAbsent`) to avoid race conditions.

---

## Q7: ArrayList vs LinkedList — time complexity and when to use which. [Easy]

| Operation | ArrayList | LinkedList |
|---|---|---|
| `get(i)` | **O(1)** — direct index | O(n) — traverse from head |
| `add(e)` (end) | **O(1)** amortized | O(1) — append to tail |
| `add(i, e)` (middle) | O(n) — shift elements | O(n) — traverse + O(1) insert |
| `remove(i)` | O(n) — shift elements | O(n) — traverse + O(1) unlink |
| Memory | Compact (contiguous array) | High overhead (Node objects, pointers) |
| Cache locality | **Excellent** | Poor (nodes scattered in heap) |

```java
// ArrayList — backed by Object[]
// When array is full, grows by 50%: newCapacity = oldCapacity + (oldCapacity >> 1)
List<String> orders = new ArrayList<>(1000); // pre-size if you know the count

// LinkedList — doubly-linked list, also implements Deque
Deque<String> queue = new LinkedList<>();
queue.offerFirst("task1");
queue.pollLast();
```

**The practical answer:** Use `ArrayList` almost always. `LinkedList` wins only when you need a `Deque` (double-ended queue) and even then `ArrayDeque` is usually faster.

> **Interview Tip:** Mention **CPU cache locality** — `ArrayList`'s contiguous memory layout is much more cache-friendly than `LinkedList`'s scattered nodes. This is the real-world performance difference.

---

## Q8: TreeMap vs HashMap vs LinkedHashMap — when to use which? [Medium]

| Feature | HashMap | LinkedHashMap | TreeMap |
|---|---|---|---|
| Ordering | None | **Insertion order** (or access order) | **Sorted** (natural or Comparator) |
| Underlying structure | Hash table | Hash table + doubly-linked list | Red-Black Tree |
| `get`/`put` | O(1) | O(1) | O(log n) |
| Null key | 1 allowed | 1 allowed | **Not allowed** (needs comparison) |
| Use case | General purpose | LRU cache, ordered iteration | Range queries, sorted data |

```java
// LinkedHashMap as an LRU Cache
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true);  // true = access-order
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}

// TreeMap — sorted + range operations
TreeMap<Integer, String> scores = new TreeMap<>();
scores.put(85, "Alice");
scores.put(92, "Bob");
scores.put(78, "Charlie");

scores.headMap(90);   // {78=Charlie, 85=Alice} — all entries with key < 90
scores.tailMap(85);   // {85=Alice, 92=Bob}     — all entries with key >= 85
```

> **Key Takeaway:** `HashMap` for speed, `LinkedHashMap` for ordered iteration / LRU cache, `TreeMap` for sorted keys and range queries. The LRU cache pattern is a very common interview question.

---

## Q9: How does HashSet work internally? [Easy]

`HashSet` is backed by a `HashMap` where each element is stored as a **key** and the value is a dummy constant object (`PRESENT`).

```java
// Simplified internal view
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // null means key was new
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
}
```

This is why:
- Elements must have proper `hashCode()` and `equals()`.
- `HashSet` allows one `null` (because `HashMap` allows one null key).
- Ordering is not guaranteed.

> **Interview Tip:** If asked "How does HashSet ensure uniqueness?", answer: "It delegates to HashMap — uniqueness is enforced by the key semantics of HashMap using `hashCode()` and `equals()`."

---

## Q10: What are fail-fast vs fail-safe iterators? [Medium]

| Feature | Fail-Fast | Fail-Safe |
|---|---|---|
| Behavior on modification | Throws `ConcurrentModificationException` | No exception; works on a snapshot or weakly consistent view |
| Examples | `ArrayList`, `HashMap`, `HashSet` | `CopyOnWriteArrayList`, `ConcurrentHashMap` |
| Mechanism | Checks `modCount` on each `next()` call | Iterates over a copy or allows concurrent changes |

```java
// Fail-fast — modifying the list while iterating
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) {
        list.remove(s);  // throws ConcurrentModificationException!
    }
}

// Safe way — use Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) {
        it.remove();  // safe!
    }
}

// Fail-safe — CopyOnWriteArrayList
List<String> cowList = new CopyOnWriteArrayList<>(List.of("a", "b", "c"));
for (String s : cowList) {
    cowList.remove(s);  // no exception — iterates over snapshot
}
```

**Common follow-up:** *"Is fail-fast guaranteed?"*
No. The Javadoc says it's on a "best-effort basis." It uses an internal `modCount` counter; if the count changes between `next()` calls, it throws. But it's not a guaranteed concurrency check.

> **Key Takeaway:** Fail-fast detects structural modification and fails early. Fail-safe trades consistency for safety. Use `Iterator.remove()` or fail-safe collections in concurrent scenarios.

---

## Q11: Comparable vs Comparator — when to use which? [Easy]

| Aspect | Comparable | Comparator |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T o1, T o2)` |
| Modifies class? | Yes (class implements it) | No (external) |
| # of orderings | One (natural ordering) | Unlimited |

```java
// Comparable — natural ordering built into the class
public class Employee implements Comparable<Employee> {
    private String name;
    private int salary;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.salary, other.salary);  // natural order: by salary
    }
}

// Comparator — external, flexible, composable (Java 8+)
List<Employee> employees = getEmployees();

employees.sort(Comparator
    .comparing(Employee::getSalary).reversed()
    .thenComparing(Employee::getName));

// Comparator for null-safe sorting
Comparator<Employee> nullSafe = Comparator.nullsLast(
    Comparator.comparing(Employee::getName)
);
```

> **Interview Tip:** Mention `Comparator.comparing()`, `thenComparing()`, `reversed()`, and `nullsFirst()`/`nullsLast()` from Java 8. These show you write modern Java, not pre-Java-8 anonymous classes.

---

# 3. Strings

---

## Q12: What is the String Pool and how does `intern()` work? [Medium]

The **String Pool** (also called the intern pool) is a special memory area in the **heap** (moved from PermGen in Java 7) where the JVM stores unique string literals.

```java
String a = "hello";           // goes to pool
String b = "hello";           // reuses same pool reference
String c = new String("hello"); // creates new object on heap (NOT in pool)

System.out.println(a == b);   // true  — same pool reference
System.out.println(a == c);   // false — different objects

String d = c.intern();        // explicitly puts c's value into the pool (or returns existing)
System.out.println(a == d);   // true  — d now points to the pool instance
```

**How `intern()` works:**
1. Checks if an equal string exists in the pool.
2. If yes, returns the pool reference.
3. If no, adds this string to the pool and returns it.

**Common follow-up:** *"How many objects are created by `new String("abc")`?"*
Up to 2: one in the pool (for the literal `"abc"` if not already there) and one on the heap (the `new` call).

> **Key Takeaway:** Literals go to the pool automatically. `new String()` creates a separate heap object. Use `intern()` carefully — it can cause memory issues if you intern too many unique strings.

---

## Q13: Why is String immutable in Java? What are the benefits? [Medium]

**String is immutable** because its internal `char[]` (or `byte[]` since Java 9) is `final` and private, with no methods that modify it.

**Benefits:**

1. **String Pool / Caching** — safe to share across references because no one can modify the content.
2. **Thread Safety** — immutable objects are inherently thread-safe; no synchronization needed.
3. **Security** — class names, file paths, network URLs, and database connection strings are Strings. If they were mutable, an attacker could change a validated path after the security check.
4. **Hash Code Caching** — `hashCode()` is computed once and cached. Makes String excellent as `HashMap` key.

```java
// hashCode is cached after first computation
public final class String {
    private int hash;  // cached, default 0

    public int hashCode() {
        int h = hash;
        if (h == 0 && !hashIsZero) {
            h = computeHashCode();  // expensive — done once
            hash = h;
        }
        return h;
    }
}
```

5. **ClassLoader safety** — classes are loaded by fully-qualified name (a String). Mutability would break the class loading mechanism.

> **Interview Tip:** Hit all five benefits: pool, thread-safety, security, hash caching, classloader. Most candidates stop at one or two.

---

## Q14: String vs StringBuilder vs StringBuffer — when to use which? [Easy]

| Feature | String | StringBuilder | StringBuffer |
|---|---|---|---|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes (immutable) | **No** | Yes (synchronized) |
| Performance | Slow for concatenation | **Fastest** | Slower (sync overhead) |
| Since | JDK 1.0 | JDK 1.5 | JDK 1.0 |

```java
// BAD — creates ~10,000 intermediate String objects
String result = "";
for (int i = 0; i < 10_000; i++) {
    result += i;  // each += creates a new String
}

// GOOD — single mutable buffer, O(n) total
StringBuilder sb = new StringBuilder(50_000);  // pre-size!
for (int i = 0; i < 10_000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

**Approximate performance (10,000 iterations):**
- `String +=` : ~200ms (JVM may optimize small cases, but large loops are terrible)
- `StringBuilder` : ~1ms
- `StringBuffer` : ~2ms (synchronization overhead)

**When to use what:**
- **String** — small, known-at-compile-time concatenations (`"Hello, " + name`; the compiler optimizes this).
- **StringBuilder** — loops, dynamic string building (most common).
- **StringBuffer** — only when multiple threads build the same string (extremely rare).

> **Key Takeaway:** `StringBuilder` in 99% of cases. `StringBuffer` is a legacy class; you'll almost never need it. The JVM's `javac` compiler converts `"a" + "b"` into `StringBuilder` calls anyway (or uses `invokedynamic` `StringConcatFactory` since Java 9).

---

## Q15: Can String be used in switch statements? [Easy]

Yes, since **Java 7**. Internally, the compiler converts it to a `switch` on `hashCode()` with `equals()` verification.

```java
String method = "UPI";

switch (method) {
    case "UPI":
        System.out.println("Processing UPI");
        break;
    case "CARD":
        System.out.println("Processing Card");
        break;
    default:
        System.out.println("Unknown method");
}

// Java 14+ enhanced switch
String message = switch (method) {
    case "UPI"  -> "Processing UPI";
    case "CARD" -> "Processing Card";
    default     -> "Unknown method";
};
```

**What happens internally:**
1. Compiler generates `switch(method.hashCode())`.
2. Inside each case, it adds an `if (method.equals("UPI"))` check (because different strings can have the same hashCode).
3. `NullPointerException` is thrown if the string is `null`.

> **Interview Tip:** Mention the null caveat — many candidates forget that passing `null` to a String switch throws NPE.

---

# 4. Exception Handling

---

## Q16: Explain the exception hierarchy — checked vs unchecked. [Medium]

```
                    Throwable
                   /         \
              Error          Exception
             /    \         /          \
    StackOverflow  OOM  IOException   RuntimeException
    VirtualMachine       SQLException    /    |      \
    Error (etc.)         (checked)   NPE  ClassCast  IllegalArgument
                                     (unchecked — all RuntimeException subclasses)
```

| Type | Checked | Unchecked |
|---|---|---|
| Superclass | `Exception` (not `RuntimeException`) | `RuntimeException`, `Error` |
| Compile-time check | Must be caught or declared | No obligation |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |
| Indicates | Recoverable conditions | Programming bugs / unrecoverable |

```java
// Checked — caller must handle
public byte[] readFile(String path) throws IOException {
    return Files.readAllBytes(Path.of(path));
}

// Unchecked — programming error, no need to declare
public void transfer(double amount) {
    if (amount <= 0) {
        throw new IllegalArgumentException("Amount must be positive: " + amount);
    }
}
```

**When to use which:**
- **Checked** — when the caller can reasonably recover (file not found → try another path, network timeout → retry).
- **Unchecked** — when it's a programming error (null input, invalid argument, broken invariant).

> **Key Takeaway:** Modern Java (and frameworks like Spring) lean heavily toward **unchecked** exceptions. Checked exceptions force every caller to handle them, which leads to catch-and-swallow anti-patterns.

---

## Q17: Explain try-with-resources and AutoCloseable. [Medium]

`try-with-resources` (Java 7+) ensures that resources are **automatically closed** in the correct order, even if exceptions occur.

```java
// Before Java 7 — verbose and error-prone
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("data.csv"));
    return br.readLine();
} finally {
    if (br != null) {
        br.close();  // what if THIS throws? original exception is lost!
    }
}

// Java 7+ — clean and correct
try (BufferedReader br = new BufferedReader(new FileReader("data.csv"))) {
    return br.readLine();
}
// br.close() is called automatically — even on exception

// Multiple resources — closed in REVERSE declaration order
try (
    Connection conn = dataSource.getConnection();
    PreparedStatement ps = conn.prepareStatement(sql);
    ResultSet rs = ps.executeQuery()
) {
    while (rs.next()) { /* process */ }
}
// Close order: rs → ps → conn
```

**AutoCloseable vs Closeable:**
- `AutoCloseable` — `close()` can throw `Exception` (broader).
- `Closeable` extends `AutoCloseable` — `close()` throws `IOException` only.

```java
public class DatabaseConnectionPool implements AutoCloseable {
    @Override
    public void close() {
        // release all pooled connections
    }
}
```

**Suppressed exceptions:** If both the `try` block and `close()` throw, the `close()` exception is **suppressed** (attached to the primary exception via `getSuppressed()`).

> **Interview Tip:** Mention suppressed exceptions — it shows you understand the subtleties. Access them with `exception.getSuppressed()`.

---

## Q18: What is exception chaining? Why is it important? [Medium]

Exception chaining preserves the **root cause** when wrapping a low-level exception in a higher-level one.

```java
public class PaymentService {
    public void processPayment(Order order) {
        try {
            gateway.charge(order);
        } catch (SocketTimeoutException e) {
            // Wrap low-level exception in domain-specific one
            // The cause chain is preserved!
            throw new PaymentProcessingException(
                "Payment failed for order " + order.getId(), e  // ← cause
            );
        }
    }
}

// Custom exception with chaining
public class PaymentProcessingException extends RuntimeException {
    public PaymentProcessingException(String message, Throwable cause) {
        super(message, cause);
    }
}

// In logs, you see the full chain:
// PaymentProcessingException: Payment failed for order ORD-123
//   Caused by: java.net.SocketTimeoutException: connect timed out
```

**Why it matters:**
- Without chaining, you lose the root cause: `catch (Exception e) { throw new RuntimeException("failed"); }` — **terrible** because the original stack trace disappears.
- With chaining, debugging is straightforward — you always know what actually went wrong.

> **Key Takeaway:** Always pass the original exception as the `cause`. Never swallow or discard exceptions.

---

## Q19: Custom exceptions — best practices. [Easy]

```java
// 1. Extend RuntimeException for unchecked (preferred for most cases)
public class InsufficientBalanceException extends RuntimeException {
    private final String accountId;
    private final BigDecimal requested;
    private final BigDecimal available;

    public InsufficientBalanceException(String accountId,
                                         BigDecimal requested,
                                         BigDecimal available) {
        super(String.format("Account %s: requested ₹%s but only ₹%s available",
                accountId, requested, available));
        this.accountId = accountId;
        this.requested = requested;
        this.available = available;
    }

    // Getters for programmatic access
    public String getAccountId() { return accountId; }
    public BigDecimal getRequested() { return requested; }
    public BigDecimal getAvailable() { return available; }
}

// 2. Extend Exception for checked (when caller MUST handle)
public class RetryablePaymentException extends Exception {
    private final int retryAfterSeconds;

    public RetryablePaymentException(String msg, int retryAfterSeconds, Throwable cause) {
        super(msg, cause);
        this.retryAfterSeconds = retryAfterSeconds;
    }

    public int getRetryAfterSeconds() { return retryAfterSeconds; }
}
```

**Best practices:**
- Carry context (account ID, amounts, etc.) — not just a message string.
- Always provide a constructor that accepts `Throwable cause`.
- Name exceptions clearly: `InsufficientBalanceException`, not `MyException`.
- Prefer unchecked unless the caller genuinely can (and should) recover.
- Don't create exception hierarchies deeper than 2 levels without good reason.

> **Interview Tip:** Show you think about exception design — carry context fields, provide cause constructors, and name exceptions as `<Problem>Exception`.

---

# 5. Generics

---

## Q20: What is type erasure? Why does it matter? [Hard]

**Type erasure** is the process by which the Java compiler removes all generic type information at compile time. At runtime, generics don't exist.

```java
// What you write:
List<String> names = new ArrayList<>();
List<Integer> numbers = new ArrayList<>();

// What exists at runtime (after erasure):
List names = new ArrayList();    // just raw List
List numbers = new ArrayList();  // same raw List

// This is why:
System.out.println(names.getClass() == numbers.getClass());  // true!
```

**Consequences of type erasure:**

```java
// 1. Cannot use instanceof with generic type
if (obj instanceof List<String>) { }    // COMPILE ERROR
if (obj instanceof List<?>) { }         // OK — unbounded wildcard

// 2. Cannot create generic arrays
T[] array = new T[10];                  // COMPILE ERROR
Object[] array = new Object[10];        // workaround

// 3. Cannot overload methods differing only in generic type
void process(List<String> list) { }
void process(List<Integer> list) { }    // COMPILE ERROR — same erasure

// 4. Cannot use primitives as type parameters
List<int> numbers;                      // COMPILE ERROR
List<Integer> numbers;                  // OK — autoboxing
```

**Why does Java use erasure?** Backward compatibility with pre-generics code (Java < 5). A `List<String>` must work with legacy code expecting a raw `List`.

> **Key Takeaway:** Generics are a compile-time safety mechanism. At runtime, `List<String>` and `List<Integer>` are both just `List`. This is why you can't do `new T()` or `T.class`.

---

## Q21: Explain bounded types — `extends` vs `super`. [Medium]

```java
// Upper bound: T must be Number or a subclass of Number
public <T extends Number> double sum(List<T> list) {
    double total = 0;
    for (T item : list) {
        total += item.doubleValue();  // Number methods are available
    }
    return total;
}

sum(List.of(1, 2, 3));        // Integer extends Number ✓
sum(List.of(1.5, 2.5));       // Double extends Number ✓
sum(List.of("a", "b"));       // String doesn't extend Number ✗

// Multiple bounds
public <T extends Comparable<T> & Serializable> T findMax(List<T> list) {
    return list.stream().max(Comparator.naturalOrder()).orElseThrow();
}
```

**`extends`** = upper bound — "T must be this type or a subtype."
**`super`** = lower bound — "T must be this type or a supertype." (used with wildcards)

> **Key Takeaway:** Use `extends` when you need to **read** values and call methods on them. Use `super` when you need to **write** values. This leads to the PECS rule (next question).

---

## Q22: What is the PECS rule? (Producer Extends, Consumer Super) [Hard]

PECS tells you which wildcard to use when designing generic APIs:

- **Producer** (you read FROM it) → `? extends T`
- **Consumer** (you write INTO it) → `? super T`

```java
// PRODUCER — we READ items from source → use extends
public static <T> void copy(List<? extends T> source,    // producer
                              List<? super T> destination) { // consumer
    for (T item : source) {
        destination.add(item);
    }
}

// Why "extends" for producer?
List<? extends Number> numbers = List.of(1, 2.5, 3L);
Number n = numbers.get(0);   // ✓ Safe to READ as Number
numbers.add(42);              // ✗ COMPILE ERROR — can't add (could be List<Double>)

// Why "super" for consumer?
List<? super Integer> target = new ArrayList<Number>();
target.add(42);               // ✓ Safe to WRITE Integer
Integer i = target.get(0);    // ✗ COMPILE ERROR — could return Number or Object
```

**Real-world example from the JDK:**

```java
// Collections.sort — the Comparator consumes T (compares them)
public static <T> void sort(List<T> list, Comparator<? super T> c) { }

// A Comparator<Number> can compare Integers — super is correct here
Comparator<Number> byValue = Comparator.comparingDouble(Number::doubleValue);
List<Integer> ints = List.of(3, 1, 2);
Collections.sort(ints, byValue);  // works because Comparator<? super Integer>
```

> **Interview Tip:** If you remember one thing: "PE-CS" — **P**roducer **E**xtends, **C**onsumer **S**uper. Joshua Bloch coined this in *Effective Java*, and interviewers at top companies love it.

---

## Q23: Generic methods vs generic classes — when to use which? [Medium]

```java
// GENERIC CLASS — type parameter lives at the class level
public class Repository<T> {
    private final List<T> items = new ArrayList<>();

    public void save(T item) { items.add(item); }
    public T findById(int index) { return items.get(index); }
}

Repository<Order> orderRepo = new Repository<>();
orderRepo.save(new Order());    // T is bound to Order for this instance


// GENERIC METHOD — type parameter lives at the method level
public class Utils {
    // <T> is declared before return type — independent of any class-level generic
    public static <T extends Comparable<T>> T max(T a, T b) {
        return a.compareTo(b) >= 0 ? a : b;
    }
}

Utils.max(10, 20);         // T inferred as Integer
Utils.max("abc", "xyz");   // T inferred as String
```

**When to use which:**
- **Generic class** — when the type parameter is part of the class's identity and used across multiple methods (e.g., `List<E>`, `Repository<T>`).
- **Generic method** — when the type parameter is relevant only to a single method (e.g., utility methods, factory methods).

> **Key Takeaway:** Prefer generic methods for standalone utility operations. Use generic classes when the type is fundamental to the object's behavior.

---

# 6. Serialization

---

## Q24: Serializable vs Externalizable — what's the difference? [Medium]

| Feature | Serializable | Externalizable |
|---|---|---|
| Interface | Marker (no methods) | Has `writeExternal()` and `readExternal()` |
| Control | JVM handles everything automatically | Full manual control |
| Default constructor | Not required | **Required** (public no-arg) |
| Performance | Slower (uses reflection) | Faster (no reflection) |
| `transient` respected? | Yes | Not applicable (you control everything) |

```java
// Serializable — easy but less control
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password;  // excluded from serialization
}

// Externalizable — full control
public class User implements Externalizable {
    private String name;
    private String password;

    public User() { }  // MUST have public no-arg constructor

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);
        // password intentionally not written — full control
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        this.name = in.readUTF();
    }
}
```

> **Key Takeaway:** Use `Serializable` for simplicity. Use `Externalizable` when you need full control over the serialized format (rare in modern Java where JSON/Protobuf is preferred).

---

## Q25: What is `serialVersionUID` and why does it matter? [Easy]

`serialVersionUID` is a version identifier for a `Serializable` class. It's used during deserialization to verify that the sender and receiver have compatible class versions.

```java
public class Order implements Serializable {
    private static final long serialVersionUID = 1L;  // explicit version

    private String orderId;
    private double amount;
    // If you add a field later (e.g., String status), and the UID is the same,
    // deserialization still works — the new field gets a default value (null).
    // If the UID is DIFFERENT, you get InvalidClassException.
}
```

**What happens if you don't declare it?**
The JVM **auto-generates** one based on class structure (fields, methods, etc.). If you add a single field, the auto-generated UID changes, and old serialized data becomes incompatible → `InvalidClassException`.

**Best practice:** Always declare `serialVersionUID` explicitly. Change it only when you intentionally break backward compatibility.

> **Interview Tip:** If asked "What happens when you add a field to a serialized class?", the answer depends on whether `serialVersionUID` matches. If it matches → new fields get default values. If it doesn't → `InvalidClassException`.

---

## Q26: Custom serialization with `readObject` / `writeObject`. [Hard]

You can customize the serialization process by providing these `private` methods:

```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;
    private String accountId;
    private BigDecimal balance;
    private transient String encryptedToken;  // transient — not serialized by default

    // Custom serialization
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();  // serialize non-transient fields normally
        // Manually serialize the transient field in an encrypted form
        oos.writeUTF(encrypt(encryptedToken));
    }

    // Custom deserialization
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();  // deserialize non-transient fields normally
        // Manually restore the transient field
        this.encryptedToken = decrypt(ois.readUTF());
    }

    // Validation after deserialization
    private void readResolve() throws ObjectStreamException {
        // Enforce invariants (e.g., for singletons)
        if (balance.compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidObjectException("Balance cannot be negative");
        }
        return this;
    }

    private String encrypt(String token) { /* ... */ return token; }
    private String decrypt(String token) { /* ... */ return token; }
}
```

**Important methods:**
- `writeObject` / `readObject` — customize field-by-field serialization.
- `readResolve` — replace the deserialized object (used for singletons to ensure only one instance exists).
- `writeReplace` — substitute the object before serialization.

> **Key Takeaway:** Custom serialization is useful for security (encrypting sensitive fields), validation (checking invariants after deserialization), and singleton patterns (`readResolve`).

---

# 7. Equals and HashCode Contract

---

## Q27: Why must you override both `equals()` and `hashCode()`? [Hard]

**The contract (from the Javadoc):**
1. If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` must be `true`.
2. If `a.hashCode() == b.hashCode()`, `a.equals(b)` is NOT necessarily `true` (hash collisions are allowed).

**What breaks when you violate the contract:**

```java
public class Money {
    private int amount;
    private String currency;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;
        return amount == m.amount && currency.equals(m.currency);
    }

    // If we DON'T override hashCode():
}

Money a = new Money(100, "INR");
Money b = new Money(100, "INR");

a.equals(b);  // true ✓

Set<Money> set = new HashSet<>();
set.add(a);
set.contains(b);  // FALSE! Because hashCode() is different (from Object identity)
                   // HashMap/HashSet look in the wrong bucket entirely
```

**What happens with only `hashCode` overridden (no `equals`):**
Two equal-meaning objects could end up in the same bucket but `equals()` (which defaults to `==`) returns `false`, so HashMap treats them as different keys → duplicates in the map.

```java
// Correct implementation
@Override
public int hashCode() {
    return Objects.hash(amount, currency);  // uses same fields as equals
}
```

> **Key Takeaway:** `hashCode` determines the bucket; `equals` determines identity within that bucket. Override both, using the **same fields**.

---

## Q28: Best practices for implementing `equals()` and `hashCode()`. [Medium]

```java
public class Transaction {
    private final String txnId;
    private final BigDecimal amount;
    private final LocalDateTime timestamp;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                          // 1. Reference check
        if (!(o instanceof Transaction that)) return false;  // 2. Type check (Java 16 pattern matching)
        return Objects.equals(txnId, that.txnId)             // 3. Field-by-field comparison
            && Objects.equals(amount, that.amount)
            && Objects.equals(timestamp, that.timestamp);
    }

    @Override
    public int hashCode() {
        return Objects.hash(txnId, amount, timestamp);       // Same fields as equals
    }
}
```

**Rules:**
1. Use `instanceof` (not `getClass()`) for type checking — it handles subclasses correctly in most cases.
2. Use `Objects.equals()` for null-safe field comparison.
3. Use `Objects.hash()` for simple hash computation.
4. Use the **same fields** in both methods.
5. Prefer **immutable fields** in equals/hashCode — if a field changes after the object is added to a `HashSet`, the object becomes "lost" (it's in the wrong bucket).

**Common follow-up:** *"Can you use mutable fields in equals/hashCode?"*
Technically yes, but it's dangerous. If a field used in `hashCode` changes after the object is put into a `HashMap`, the map can't find it anymore — it looks in the new bucket, but the object is still in the old one.

> **Interview Tip:** Mention `instanceof` pattern matching (Java 16) and `Objects.hash()` — it shows you write modern, clean code.

---

# 8. Immutability

---

## Q29: How do you create an immutable class in Java? Step by step. [Hard]

```java
public final class Money {                         // 1. Class is final — cannot be subclassed

    private final int amount;                       // 2. All fields are final
    private final String currency;
    private final List<String> tags;                // mutable type — needs defensive copy

    public Money(int amount, String currency, List<String> tags) {
        this.amount = amount;                       // 3. Set all fields via constructor
        this.currency = currency;
        this.tags = List.copyOf(tags);              // 4. Defensive copy on input!
    }

    public int getAmount() { return amount; }       // 5. No setters
    public String getCurrency() { return currency; }

    public List<String> getTags() {
        return tags;                                 // List.copyOf already returns unmodifiable
    }

    // If using pre-Java-10:
    // return Collections.unmodifiableList(new ArrayList<>(tags));
}
```

**The five rules:**
1. Declare the class `final` (no subclass can break immutability).
2. Make all fields `private final`.
3. No setter methods.
4. **Defensive copy** mutable inputs in the constructor.
5. **Defensive copy** (or return unmodifiable views) mutable outputs in getters.

**Common follow-up:** *"What about Date fields?"*

```java
public final class Event {
    private final Date eventDate;

    public Event(Date eventDate) {
        this.eventDate = new Date(eventDate.getTime());  // defensive copy IN
    }

    public Date getEventDate() {
        return new Date(eventDate.getTime());             // defensive copy OUT
    }
}
// Better: use java.time.LocalDate (already immutable) instead of java.util.Date
```

> **Key Takeaway:** Immutability is not just "make fields final." The tricky part is handling mutable fields (collections, Date, arrays) via defensive copying. In modern Java, prefer `java.time` classes and `List.copyOf()`.

---

## Q30: What are Java Records? How do they relate to immutability? [Medium]

Records (Java 16+) are **immutable data carriers** with auto-generated `equals()`, `hashCode()`, `toString()`, and accessors.

```java
// Old way — 50+ lines of boilerplate
public final class Point {
    private final int x;
    private final int y;
    // constructor, getters, equals, hashCode, toString...
}

// Record — 1 line
public record Point(int x, int y) { }

// Usage
Point p = new Point(3, 4);
p.x();              // 3 — accessor (NOT getX())
p.y();              // 4
p.toString();       // Point[x=3, y=4]
p.equals(new Point(3, 4));  // true
```

**What Records give you automatically:**
- `final` class (cannot be extended).
- `private final` fields for each component.
- All-args constructor.
- Accessors named like the components (`x()`, not `getX()`).
- `equals()`, `hashCode()`, and `toString()` based on all components.

**What Records DON'T do:**
- They don't make mutable component types immutable:

```java
// DANGER — the List inside is still mutable!
public record Order(String id, List<String> items) { }

Order o = new Order("ORD-1", new ArrayList<>(List.of("A", "B")));
o.items().add("C");  // mutates the internal list!

// Fix — use compact constructor for defensive copy
public record Order(String id, List<String> items) {
    public Order {                              // compact constructor
        items = List.copyOf(items);             // defensive copy → truly immutable
    }
}
```

**Common follow-up:** *"Can Records have methods?"*
Yes. You can add instance methods, static methods, and implement interfaces. You cannot add mutable instance fields.

> **Interview Tip:** Records are NOT just "less boilerplate." They are a semantic declaration: "this class is a transparent, immutable data carrier." If you need mutability, encapsulation of internals, or non-data behavior, use a regular class.

---

# Bonus Questions (Commonly Asked Follow-ups)

---

## Q31: What is the difference between `==` and `.equals()`? [Easy]

- `==` compares **references** (are they the same object in memory?).
- `.equals()` compares **logical equality** (are they meaningfully the same?).

```java
String a = new String("hello");
String b = new String("hello");

a == b;       // false — different objects on heap
a.equals(b);  // true  — same content

// With Integer caching (-128 to 127):
Integer x = 127;
Integer y = 127;
x == y;       // true — cached by Integer.valueOf()

Integer p = 128;
Integer q = 128;
p == q;       // false — outside cache range, different objects
p.equals(q);  // true
```

> **Interview Tip:** The Integer cache range (-128 to 127) is a classic trap question. Always use `.equals()` for object comparison.

---

## Q32: What is the `transient` keyword? [Easy]

`transient` marks a field to be **excluded from serialization**.

```java
public class Session implements Serializable {
    private String userId;
    private transient String authToken;    // not serialized
    private transient int loginAttempts;   // not serialized

    // After deserialization:
    // userId   → restored
    // authToken → null (default for Object)
    // loginAttempts → 0 (default for int)
}
```

**Use cases:** passwords, tokens, cached/derived values, non-serializable fields (like `Thread` or `Socket`).

**Common follow-up:** *"Can you serialize a transient field?"*
Not automatically, but you can manually with custom `writeObject()`/`readObject()` (see Q26).

---

## Q33: What is the difference between `final`, `finally`, and `finalize()`? [Easy]

| Keyword | Purpose |
|---|---|
| `final` | Variable: constant. Method: can't override. Class: can't extend. |
| `finally` | Block that always executes after try/catch (for cleanup). |
| `finalize()` | **Deprecated (Java 9+)**. Called by GC before reclaiming an object. |

```java
final int MAX = 100;          // constant — cannot reassign

try {
    riskyOperation();
} catch (Exception e) {
    log(e);
} finally {
    cleanup();                 // always runs (even if catch re-throws)
}

// finalize() — NEVER use in modern Java
@Override
protected void finalize() throws Throwable {
    // unreliable, slow, deprecated — use try-with-resources or Cleaner instead
}
```

> **Interview Tip:** If `finalize()` comes up, mention that it's deprecated since Java 9 and that `java.lang.ref.Cleaner` (Java 9+) is the modern alternative. This shows you keep up with Java evolution.

---

## Q34: What is the difference between shallow copy and deep copy? [Medium]

```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Employee implements Cloneable {
    String name;
    Address address;    // reference type — the tricky part

    // SHALLOW COPY — copies references, not objects
    @Override
    protected Employee clone() throws CloneNotSupportedException {
        return (Employee) super.clone();
        // address field still points to the SAME Address object
    }

    // DEEP COPY — creates new copies of referenced objects
    public Employee deepClone() {
        Employee copy = new Employee();
        copy.name = this.name;                      // String is immutable, safe
        copy.address = new Address(this.address.city); // new Address object
        return copy;
    }
}

Employee original = new Employee();
original.name = "Alice";
original.address = new Address("Mumbai");

Employee shallow = original.clone();
shallow.address.city = "Delhi";
System.out.println(original.address.city);  // "Delhi" — OOPS! original changed

Employee deep = original.deepClone();
deep.address.city = "Bangalore";
System.out.println(original.address.city);  // "Delhi" — original is safe
```

**Modern alternatives to `Cloneable`:**
- Copy constructor: `new Employee(existingEmployee)`
- Static factory: `Employee.copyOf(existingEmployee)`
- Serialization-based deep copy (heavy but thorough)

> **Key Takeaway:** `Cloneable` and `clone()` are considered broken by design (see *Effective Java*). Prefer copy constructors or factory methods.

---

## Q35: What is method hiding vs method overriding? [Medium]

```java
class Parent {
    static void greet() { System.out.println("Parent static"); }
    void hello()        { System.out.println("Parent instance"); }
}

class Child extends Parent {
    static void greet() { System.out.println("Child static"); }   // HIDING (not overriding)

    @Override
    void hello()        { System.out.println("Child instance"); } // OVERRIDING
}

Parent ref = new Child();
ref.greet();   // "Parent static" — resolved by REFERENCE type (compile time)
ref.hello();   // "Child instance" — resolved by OBJECT type (runtime)
```

| Aspect | Method Overriding | Method Hiding |
|---|---|---|
| Applies to | Instance methods | Static methods |
| Resolution | Runtime (dynamic dispatch) | Compile time (reference type) |
| `@Override` | Yes | No (not an override) |
| Polymorphism? | Yes | No |

> **Interview Tip:** Interviewers sometimes show code with `static` methods and polymorphic references to trick you. Static methods are resolved by the reference type, not the object type.

---

## Q36: Explain `Optional` — why it was introduced and best practices. [Medium]

`Optional<T>` (Java 8) is a container that may or may not hold a non-null value. It was introduced to combat `NullPointerException` and make APIs more expressive.

```java
// BAD — returning null
public User findUser(String id) {
    // ... might return null, caller must remember to check
    return null;
}

// GOOD — returning Optional
public Optional<User> findUser(String id) {
    return Optional.ofNullable(userMap.get(id));
}

// Usage
findUser("USR-1")
    .map(User::getName)
    .filter(name -> name.startsWith("A"))
    .ifPresentOrElse(
        name -> System.out.println("Found: " + name),
        ()   -> System.out.println("Not found")
    );

// Getting value safely
String name = findUser("USR-1")
    .map(User::getName)
    .orElse("Unknown");

// Throw if absent
User user = findUser("USR-1")
    .orElseThrow(() -> new UserNotFoundException("USR-1"));
```

**Best practices:**
- Use `Optional` as a **return type** — never as a field, method parameter, or collection element.
- Never call `optional.get()` without checking — use `orElse()`, `orElseGet()`, or `orElseThrow()`.
- Never do `Optional.of(null)` — it throws NPE. Use `Optional.ofNullable()`.
- Don't use `Optional` just to avoid `if-null` checks in private methods — it adds overhead.

> **Key Takeaway:** `Optional` is for API boundaries: "this method might not return a value." It's not a replacement for null everywhere.

---

## Quick Reference: Complexity Cheat Sheet

| Collection | get | add | remove | contains | Ordered | Sorted | Thread-safe |
|---|---|---|---|---|---|---|---|
| ArrayList | O(1) | O(1)* | O(n) | O(n) | Yes | No | No |
| LinkedList | O(n) | O(1) | O(n) | O(n) | Yes | No | No |
| HashMap | O(1) | O(1) | O(1) | O(1) | No | No | No |
| LinkedHashMap | O(1) | O(1) | O(1) | O(1) | Yes | No | No |
| TreeMap | O(log n) | O(log n) | O(log n) | O(log n) | Yes | Yes | No |
| HashSet | — | O(1) | O(1) | O(1) | No | No | No |
| TreeSet | — | O(log n) | O(log n) | O(log n) | Yes | Yes | No |
| ConcurrentHashMap | O(1) | O(1) | O(1) | O(1) | No | No | Yes |
| CopyOnWriteArrayList | O(1) | O(n) | O(n) | O(n) | Yes | No | Yes |

\* amortized

---

> **Final Interview Tips:**
> 1. Always relate answers to **real-world systems** — payment processing, order management, user sessions.
> 2. Know **Java version changes** — interviewers at CRED/Groww expect you to know what changed in Java 8, 11, 16, 17.
> 3. Don't just memorize — understand **why** (why is HashMap O(1)? why is String immutable?).
> 4. For **Collections**, be ready to draw diagrams and trace through `put()` / `get()` operations.
> 5. Practice writing `equals()`/`hashCode()`, immutable classes, and custom exceptions from scratch — these are common "write code on the spot" questions.
