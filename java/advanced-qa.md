# Java Advanced Interview Q&A — Senior Developer (SDE-2)

> **Target**: 5+ years Java Developer preparing for SDE-2 interviews at product companies
> **Total Questions**: 30 | Covers: Multithreading, JMM, Java 8/11/17, ClassLoaders
> **Strategy**: Each answer explains the WHY, gives compilable code, and ties to real-world systems.

---

# Section 1: Multithreading and Concurrency

---

## Q1: Explain the complete Thread lifecycle with all six states. [Medium]

A thread in Java transitions through **six states** defined in `Thread.State`:

```
NEW ──▶ RUNNABLE ──▶ BLOCKED ──▶ WAITING ──▶ TIMED_WAITING ──▶ TERMINATED
```

| State | How You Get Here | How You Leave |
|---|---|---|
| **NEW** | `new Thread()` — created but `start()` not called | Call `start()` |
| **RUNNABLE** | `start()` called; thread is eligible for CPU (may or may not be running) | Acquires lock → runs; or blocked/waiting |
| **BLOCKED** | Trying to enter a `synchronized` block but another thread holds the monitor | Monitor becomes available |
| **WAITING** | Called `Object.wait()`, `Thread.join()`, or `LockSupport.park()` without timeout | Another thread calls `notify()`/`notifyAll()`, or joined thread finishes |
| **TIMED_WAITING** | Called `Thread.sleep(ms)`, `wait(ms)`, `join(ms)`, `parkNanos()` | Timeout expires or notification received |
| **TERMINATED** | `run()` completes normally or throws an uncaught exception | — (final state) |

```java
public class ThreadLifecycleDemo {
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait(); // WAITING
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });

        System.out.println(t.getState()); // NEW
        t.start();
        Thread.sleep(100);
        System.out.println(t.getState()); // WAITING

        synchronized (lock) {
            lock.notify();
        }
        t.join();
        System.out.println(t.getState()); // TERMINATED
    }
}
```

**Why it matters**: Understanding states is critical for debugging thread dumps. When you see 200 threads in `BLOCKED` state in a production thread dump, you know there's lock contention.

> **Interview Tip**: Interviewers love the BLOCKED vs WAITING distinction. BLOCKED = waiting to *enter* a synchronized block. WAITING = already inside, voluntarily gave up the lock via `wait()`.

**Common follow-ups**: "What's the difference between `RUNNABLE` and actually running?" — Java doesn't distinguish; a RUNNABLE thread may be waiting for CPU time from the OS scheduler.

---

## Q2: Thread vs Runnable vs Callable — when do you use each? [Easy]

| Feature | `Thread` | `Runnable` | `Callable<V>` |
|---|---|---|---|
| Return value | No | No | Yes (`V`) |
| Checked exception | No | No | Yes (`throws Exception`) |
| Submitted to ExecutorService | No (directly) | Yes | Yes |
| Functional interface | No | Yes (`@FunctionalInterface`) | Yes |

```java
// 1. Runnable — fire-and-forget tasks
Runnable task = () -> System.out.println("Processing order #1234");
new Thread(task).start();

// 2. Callable — need a result or may throw
Callable<Double> priceCheck = () -> {
    // simulate API call to pricing service
    Thread.sleep(500);
    return 1299.99;
};

ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Double> future = executor.submit(priceCheck);
Double price = future.get(); // blocks until result is ready

// 3. Extending Thread — almost never recommended
class LegacyWorker extends Thread {
    @Override
    public void run() {
        // you lose the ability to extend any other class
    }
}
```

**Real-world**: At a payment company, you'd use `Callable` when calling a bank API that returns a transaction status, and `Runnable` for sending async notification emails.

> **Interview Tip**: Never say "I extend Thread." In production code, always implement `Runnable`/`Callable` and submit to an `ExecutorService`. Direct thread creation is an anti-pattern.

---

## Q3: synchronized — method level vs block level. What is a monitor lock? [Medium]

Every Java object has an intrinsic lock (also called a **monitor**). The `synchronized` keyword acquires this monitor.

```java
public class BankAccount {
    private double balance;
    private final Object withdrawLock = new Object();
    private final Object depositLock = new Object();

    // Method-level: locks on 'this' — entire method is critical section
    public synchronized void transferAll(BankAccount target) {
        target.balance += this.balance;
        this.balance = 0;
    }

    // Block-level: finer-grained control, better concurrency
    public void withdraw(double amount) {
        synchronized (withdrawLock) {  // only locks what's necessary
            if (balance >= amount) {
                balance -= amount;
            }
        }
        // other non-critical code runs without holding the lock
    }

    // Static synchronized locks on the Class object (BankAccount.class)
    public static synchronized void updateExchangeRate() {
        // only one thread across ALL instances can execute this
    }
}
```

**Why block-level is preferred**:
- **Reduced contention**: Only the critical section is locked, not the entire method.
- **Multiple locks**: You can use different lock objects for independent operations.
- **Lock on private objects**: Using `private final Object lock` prevents external code from interfering with your synchronization.

> **Interview Tip**: If the interviewer asks "Can two threads execute two different synchronized methods on the same object simultaneously?" — the answer is **No**, because both acquire the same monitor (the `this` reference). This is a very common gotcha.

**Common follow-ups**: "What happens if a synchronized method calls another synchronized method on the same object?" — It works because Java monitors are **reentrant**.

---

## Q4: What does `volatile` guarantee? When is it NOT enough? [Medium]

`volatile` provides two guarantees:

1. **Visibility**: A write to a volatile variable is immediately visible to all threads (flushed to main memory, not cached in CPU registers/L1 cache).
2. **Ordering**: Prevents instruction reordering around volatile reads/writes (establishes a happens-before relationship).

```java
public class GracefulShutdown {
    // Without volatile, the worker thread might cache 'running' and never see the change
    private volatile boolean running = true;

    public void stop() {
        running = false; // visible to worker thread immediately
    }

    public void work() {
        while (running) { // reads from main memory every time
            processNextItem();
        }
        System.out.println("Shutting down gracefully");
    }
}
```

**When volatile is NOT enough** — when you need atomicity:

```java
private volatile int counter = 0;

// BUG: counter++ is read-modify-write (3 operations), not atomic
public void increment() {
    counter++; // Thread A reads 5, Thread B reads 5, both write 6 → lost update!
}

// FIX: use AtomicInteger
private final AtomicInteger counter = new AtomicInteger(0);
public void increment() {
    counter.incrementAndGet(); // CAS-based, lock-free, atomic
}
```

**Rule of thumb**: Use `volatile` for flags (boolean stop signals). Use `AtomicXxx` or `synchronized` for compound operations.

> **Interview Tip**: The classic question is "Is `volatile` a replacement for `synchronized`?" — No. `volatile` guarantees visibility, not atomicity. `synchronized` guarantees both.

---

## Q5: ReentrantLock vs synchronized — when would you pick ReentrantLock? [Hard]

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class PaymentProcessor {
    private final ReentrantLock lock = new ReentrantLock(true); // fair lock

    // Feature 1: tryLock — don't block forever
    public boolean processPayment(Order order) {
        boolean acquired = false;
        try {
            acquired = lock.tryLock(2, TimeUnit.SECONDS); // wait max 2s
            if (acquired) {
                return chargeCard(order);
            } else {
                return fallbackToQueue(order); // degrade gracefully
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        } finally {
            if (acquired) lock.unlock(); // MUST unlock in finally
        }
    }

    // Feature 2: lockInterruptibly — respond to interrupts while waiting
    public void cancelableOperation() throws InterruptedException {
        lock.lockInterruptibly(); // throws if thread is interrupted while waiting
        try {
            doWork();
        } finally {
            lock.unlock();
        }
    }
}
```

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Acquire without blocking | No | `tryLock()`, `tryLock(timeout)` |
| Interruptible lock wait | No | `lockInterruptibly()` |
| Fairness policy | No (unfair) | Constructor param `new ReentrantLock(true)` |
| Multiple conditions | One wait-set per monitor | `lock.newCondition()` — multiple wait-sets |
| Must explicitly unlock | No (auto on block exit) | Yes — **always in `finally`** |
| ReadWrite separation | No | Use `ReentrantReadWriteLock` |

**When to use which**:
- **`synchronized`**: Simple mutual exclusion where you don't need timeout/fairness. Simpler, less error-prone.
- **`ReentrantLock`**: When you need tryLock, timed lock, fairness, multiple conditions, or ReadWrite locks.

> **Interview Tip**: Always mention that forgetting `unlock()` in a `finally` block is a common bug with ReentrantLock. This is the #1 reason `synchronized` is safer for simple cases.

---

## Q6: Explain ThreadPoolExecutor's 7 parameters. How would you configure it for a payment system? [Hard]

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                              // corePoolSize: threads always alive
    50,                              // maximumPoolSize: max threads under load
    60L, TimeUnit.SECONDS,           // keepAliveTime: idle non-core threads die after this
    new LinkedBlockingQueue<>(1000), // workQueue: tasks wait here when all core threads busy
    new CustomThreadFactory("payment-worker"), // threadFactory
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejectionPolicy
);
```

**Execution flow**:
1. Task arrives → core thread available? → Execute immediately.
2. All core threads busy → put in `workQueue`.
3. Queue full → create new thread up to `maximumPoolSize`.
4. Max threads reached AND queue full → `RejectedExecutionHandler` kicks in.

**Rejection Policies**:
| Policy | Behavior | Use Case |
|---|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` | When you must not silently drop tasks |
| `CallerRunsPolicy` | Caller thread executes the task | Natural back-pressure; slows down producer |
| `DiscardPolicy` | Silently drops the task | Non-critical tasks (metrics, logging) |
| `DiscardOldestPolicy` | Drops oldest queued task, retries | When latest data is more important |

**Real-world config for a payment gateway**:
```java
// CPU-bound (encryption, hashing): threads ≈ CPU cores
int cpuPool = Runtime.getRuntime().availableProcessors();

// I/O-bound (API calls to banks): threads = cores × (1 + waitTime/computeTime)
// If avg API call = 200ms, compute = 5ms → ratio ≈ 40 → but cap at practical limit
int ioPool = cpuPool * 10; // e.g., 80 threads on 8-core machine

// Bounded queue prevents OOM under traffic spikes
ThreadPoolExecutor paymentExecutor = new ThreadPoolExecutor(
    cpuPool * 2, ioPool,
    30L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(5000),
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure instead of dropping
);
```

> **Interview Tip**: Never use `Executors.newFixedThreadPool()` or `newCachedThreadPool()` in production. `newFixedThreadPool` uses an unbounded `LinkedBlockingQueue` (can cause OOM). `newCachedThreadPool` has no max thread limit (can create thousands of threads). Always configure `ThreadPoolExecutor` explicitly.

---

## Q7: CompletableFuture — thenApply vs thenCompose, and real-world parallel API calls [Hard]

**`thenApply`** = map (transforms the value, returns `CompletableFuture<U>`)
**`thenCompose`** = flatMap (chains another async operation, avoids `CompletableFuture<CompletableFuture<U>>`)

```java
// thenApply: synchronous transformation
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))
    .thenApply(user -> user.getName().toUpperCase()); // User → String

// thenCompose: another async call that depends on the first
CompletableFuture<Order> future = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))
    .thenCompose(user -> fetchLatestOrder(user.getId())); // User → CF<Order>
```

**Real-world: Calling multiple lending partner APIs in parallel at PayU**:
```java
public LoanOffer getBestLoanOffer(LoanRequest request) {
    CompletableFuture<LoanOffer> hdfcFuture = CompletableFuture
        .supplyAsync(() -> hdfcClient.getOffer(request), ioExecutor)
        .exceptionally(ex -> {
            log.warn("HDFC API failed: {}", ex.getMessage());
            return LoanOffer.empty(); // graceful degradation
        });

    CompletableFuture<LoanOffer> iciciFuture = CompletableFuture
        .supplyAsync(() -> iciciClient.getOffer(request), ioExecutor)
        .exceptionally(ex -> {
            log.warn("ICICI API failed: {}", ex.getMessage());
            return LoanOffer.empty();
        });

    CompletableFuture<LoanOffer> sbiOffer = CompletableFuture
        .supplyAsync(() -> sbiClient.getOffer(request), ioExecutor)
        .exceptionally(ex -> {
            log.warn("SBI API failed: {}", ex.getMessage());
            return LoanOffer.empty();
        });

    // Wait for ALL to complete, then pick the best
    return CompletableFuture.allOf(hdfcFuture, iciciFuture, sbiOffer)
        .thenApply(v -> Stream.of(hdfcFuture, iciciFuture, sbiOffer)
            .map(CompletableFuture::join)
            .filter(offer -> !offer.isEmpty())
            .min(Comparator.comparing(LoanOffer::getInterestRate))
            .orElse(LoanOffer.empty()))
        .get(5, TimeUnit.SECONDS); // hard timeout
}
```

**Key methods cheat sheet**:
| Method | Purpose |
|---|---|
| `supplyAsync(supplier, executor)` | Start async computation with result |
| `thenApply(fn)` | Transform result (sync) |
| `thenCompose(fn)` | Chain another CompletableFuture (async) |
| `thenCombine(other, fn)` | Combine results of two futures |
| `allOf(cf1, cf2, ...)` | Wait for all to complete |
| `anyOf(cf1, cf2, ...)` | Return as soon as any one completes |
| `exceptionally(fn)` | Handle exceptions, provide fallback |
| `handle((result, ex) -> ...)` | Handle both success and failure |
| `orTimeout(duration)` | Java 9+: auto-complete exceptionally on timeout |

> **Interview Tip**: Always pass a custom `Executor` to `supplyAsync`. The default `ForkJoinPool.commonPool()` is shared across the entire JVM — a slow API call there blocks Stream parallel operations and other CompletableFutures.

**Common follow-ups**: "What's the difference between `handle` and `exceptionally`?" — `handle` is called on BOTH success and failure. `exceptionally` only on failure.

---

## Q8: CountDownLatch vs CyclicBarrier vs Semaphore [Hard]

| Feature | CountDownLatch | CyclicBarrier | Semaphore |
|---|---|---|---|
| Reusable? | No (one-shot) | Yes (resets after each barrier trip) | Yes |
| Who waits? | One/many threads wait for others to count down | All threads wait for each other | Threads wait for permits |
| Use case | "Wait for N tasks to finish" | "All threads start phase 2 together" | "Limit concurrent access to N" |

```java
// CountDownLatch: Application startup — wait for all services to initialize
CountDownLatch latch = new CountDownLatch(3);

executor.submit(() -> { initDatabase();    latch.countDown(); });
executor.submit(() -> { initCache();       latch.countDown(); });
executor.submit(() -> { initMessageQueue();latch.countDown(); });

latch.await(30, TimeUnit.SECONDS); // main thread waits
startAcceptingRequests();

// CyclicBarrier: Parallel computation in rounds (e.g., simulation)
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All workers finished this round, merging results...");
});

for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        while (hasMoreRounds()) {
            computePartialResult();
            barrier.await(); // all 3 threads meet here before next round
        }
    });
}

// Semaphore: Rate limiting — max 5 concurrent DB connections
Semaphore dbPool = new Semaphore(5);

public ResultSet query(String sql) throws InterruptedException {
    dbPool.acquire();           // blocks if 5 threads already querying
    try {
        return executeQuery(sql);
    } finally {
        dbPool.release();       // return the permit
    }
}
```

> **Interview Tip**: "Can CountDownLatch be reused?" — No. Once the count reaches 0, it's done. Use `CyclicBarrier` or `Phaser` for reusable synchronization.

---

## Q9: What is ThreadLocal? What are its pitfalls? [Medium]

`ThreadLocal` gives each thread its own isolated copy of a variable.

```java
public class RequestContext {
    private static final ThreadLocal<String> currentUser = new ThreadLocal<>();
    private static final ThreadLocal<String> requestId = new ThreadLocal<>();

    public static void set(String user, String reqId) {
        currentUser.set(user);
        requestId.set(reqId);
    }

    public static String getUser() { return currentUser.get(); }
    public static String getRequestId() { return requestId.get(); }

    // THIS IS CRITICAL — must call in a finally block or filter
    public static void clear() {
        currentUser.remove();
        requestId.remove();
    }
}

// In a servlet filter:
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    try {
        RequestContext.set(extractUser(req), generateRequestId());
        chain.doFilter(req, res);
    } finally {
        RequestContext.clear(); // MUST clean up — thread goes back to pool!
    }
}
```

**Pitfalls**:
1. **Memory leak in thread pools**: If you don't call `remove()`, the value stays attached to the pooled thread and is visible to the next request — causing data leakage and memory leaks.
2. **Doesn't work with `CompletableFuture`/parallel streams**: The child thread gets `null`, not the parent's value. Use `InheritableThreadLocal` or pass context explicitly.
3. **Hidden state**: Makes code harder to test and reason about. Prefer dependency injection when possible.

> **Interview Tip**: If the interviewer asks about a real-world use — Spring's `SecurityContextHolder` uses `ThreadLocal` to store the authenticated user for the duration of the request. MDC (Mapped Diagnostic Context) in logging frameworks is another classic example.

---

## Q10: Deadlock — what causes it, how to detect, how to prevent? [Hard]

**Four conditions for deadlock** (all must hold simultaneously):
1. **Mutual Exclusion**: Resource can't be shared.
2. **Hold and Wait**: Thread holds one resource while waiting for another.
3. **No Preemption**: Resources can't be forcibly taken away.
4. **Circular Wait**: Thread A waits for B's resource, B waits for A's resource.

```java
// DEADLOCK EXAMPLE
public class DeadlockDemo {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {             // holds lockA
                sleep(100);                     // simulate work
                synchronized (lockB) {          // waits for lockB (held by t2)
                    System.out.println("T1");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {             // holds lockB
                sleep(100);
                synchronized (lockA) {          // waits for lockA (held by t1)
                    System.out.println("T2");
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

**FIX — consistent lock ordering**:
```java
// Always acquire locks in the same order: lockA first, then lockB
Thread t2Fixed = new Thread(() -> {
    synchronized (lockA) {   // same order as t1
        sleep(100);
        synchronized (lockB) {
            System.out.println("T2 — no deadlock");
        }
    }
});
```

**Detection**:
- **Thread dump**: `jstack <pid>` or `kill -3 <pid>` — JVM detects deadlocks and reports them.
- **JMX**: `ThreadMXBean.findDeadlockedThreads()`
- **VisualVM / JConsole**: GUI tools that highlight deadlocked threads.

**Prevention strategies**:
1. Lock ordering (as shown above).
2. Use `tryLock(timeout)` with `ReentrantLock`.
3. Avoid nested locks when possible.
4. Use higher-level concurrency utilities (`ConcurrentHashMap`, `BlockingQueue`).

> **Interview Tip**: If asked to write deadlock code, write it quickly and then *immediately* explain the fix. Shows you understand both the problem and solution.

---

## Q11: Implement the Producer-Consumer pattern using BlockingQueue [Medium]

```java
import java.util.concurrent.*;

public class ProducerConsumerDemo {
    private static final BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
    private static volatile boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // Producer: puts payment events into the queue
        executor.submit(() -> {
            int eventId = 0;
            while (running) {
                try {
                    String event = "PAYMENT_EVENT_" + (eventId++);
                    queue.put(event); // blocks if queue is full (back-pressure!)
                    System.out.println("Produced: " + event);
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });

        // Consumer 1 & 2: process events
        for (int i = 0; i < 2; i++) {
            final int consumerId = i;
            executor.submit(() -> {
                while (running || !queue.isEmpty()) {
                    try {
                        String event = queue.poll(1, TimeUnit.SECONDS);
                        if (event != null) {
                            System.out.printf("Consumer-%d processed: %s%n", consumerId, event);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }

        Thread.sleep(2000);
        running = false;
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

**BlockingQueue implementations**:
| Type | Bounded? | Ordering | Use Case |
|---|---|---|---|
| `ArrayBlockingQueue` | Yes (fixed) | FIFO | General-purpose bounded buffer |
| `LinkedBlockingQueue` | Optional | FIFO | Higher throughput (separate put/take locks) |
| `PriorityBlockingQueue` | Unbounded | Priority | Process high-priority tasks first |
| `SynchronousQueue` | Zero capacity | N/A | Direct handoff (used by `newCachedThreadPool`) |
| `DelayQueue` | Unbounded | Delay-based | Scheduled tasks, cache expiry |

> **Interview Tip**: When the interviewer says "implement Producer-Consumer," use `BlockingQueue` — not `wait()/notify()`. It demonstrates you write production-quality code, not textbook code.

---

# Section 2: Java Memory Model

---

## Q12: Explain Heap vs Stack memory with the object allocation flow. [Medium]

```
┌─────────────────────────────────────────────┐
│                 JVM MEMORY                   │
├─────────────────┬───────────────────────────┤
│   STACK          │        HEAP               │
│  (per thread)    │     (shared)              │
│                  │                           │
│  ┌────────────┐  │  ┌─────────────────────┐  │
│  │ Frame: main│──┼─▶│ Employee obj         │  │
│  │  emp (ref) │  │  │  name ──▶ "Shail"   │  │
│  │  salary=50k│  │  │  dept ──▶ "Engg"    │  │
│  └────────────┘  │  └─────────────────────┘  │
│  ┌────────────┐  │                           │
│  │Frame:calc()│  │  ┌──────────────────────┐ │
│  │  bonus=10k │  │  │ String pool          │ │
│  │  temp (ref)│──┼─▶│  "Shail", "Engg"    │ │
│  └────────────┘  │  └──────────────────────┘ │
└─────────────────┴───────────────────────────┘
```

| Aspect | Stack | Heap |
|---|---|---|
| Stores | Primitives, local variables, method frames, object references | Objects, instance variables, arrays |
| Thread safety | Thread-private (no sharing) | Shared across threads (need synchronization) |
| Lifetime | Method entry → method exit | Until GC collects (no more references) |
| Speed | Faster (LIFO push/pop) | Slower (complex allocation + GC) |
| Size | Small (default ~512KB–1MB per thread) | Large (configurable via `-Xmx`) |
| Error | `StackOverflowError` (deep recursion) | `OutOfMemoryError` |

```java
public void process() {
    int count = 10;                    // primitive → STACK
    String name = "Shail";             // reference → STACK, String object → HEAP (string pool)
    Employee emp = new Employee(name); // reference → STACK, Employee object → HEAP
    List<Order> orders = new ArrayList<>(); // ArrayList object + backing array → HEAP
}
// When process() returns: stack frame is popped, references are gone,
// objects become eligible for GC (if no other references exist)
```

> **Interview Tip**: "Where are static variables stored?" — In the **Metaspace** (since Java 8; previously PermGen). They live as long as the ClassLoader that loaded the class.

---

## Q13: Explain the Heap structure — Young Gen, Old Gen, Metaspace. [Medium]

```
HEAP (-Xms / -Xmx)
├── Young Generation (-Xmn)
│   ├── Eden Space (new objects allocated here)
│   ├── Survivor 0 (S0 / From)
│   └── Survivor 1 (S1 / To)
├── Old Generation (Tenured)
│   └── Long-lived objects promoted from Young Gen
│
NON-HEAP
├── Metaspace (class metadata, method bytecode — replaced PermGen in Java 8)
├── Code Cache (JIT-compiled native code)
└── Thread Stacks
```

**Object lifecycle through the heap**:
1. Object created → **Eden**.
2. Eden full → **Minor GC** (young GC): live objects copied to S0.
3. Next Minor GC → live objects from Eden + S0 copied to S1. S0 cleared. (S0 and S1 swap roles each cycle.)
4. Object survives N minor GCs (default threshold = 15) → **promoted to Old Gen**.
5. Old Gen full → **Major GC** (full GC): much more expensive, often stop-the-world.

**Tuning flags**:
```bash
java -Xms4g -Xmx4g           # initial and max heap
     -Xmn1g                   # young generation size
     -XX:SurvivorRatio=8      # Eden:S0:S1 = 8:1:1
     -XX:MaxTenuringThreshold=15
     -XX:MaxMetaspaceSize=256m
```

> **Interview Tip**: "Why is Minor GC fast?" — Because most objects die young (generational hypothesis). Minor GC only scans the small young generation, and most objects are already dead, so there's very little to copy.

---

## Q14: Compare GC algorithms — Serial, Parallel, CMS, G1, ZGC. Which to use when? [Hard]

| GC | Algorithm | Pause | Throughput | Best For |
|---|---|---|---|---|
| **Serial** (`-XX:+UseSerialGC`) | Mark-Copy (young) + Mark-Compact (old) | Long STW | Low | Client apps, small heaps (<100MB) |
| **Parallel** (`-XX:+UseParallelGC`) | Multi-threaded mark-copy + compact | Medium STW | High | Batch jobs, data pipelines (throughput > latency) |
| **CMS** (`-XX:+UseConcMarkSweepGC`) | Concurrent mark-sweep | Short STW | Medium | Legacy; deprecated in Java 9, removed in 14 |
| **G1** (`-XX:+UseG1GC`) ★ | Region-based, concurrent marking, mixed GC | Predictable | Good | Default since Java 9; heaps 4GB–16GB |
| **ZGC** (`-XX:+UseZGC`) | Colored pointers, load barriers | <10ms | Good | Ultra-low latency; heaps up to 16TB |
| **Shenandoah** | Brooks pointers, concurrent compaction | <10ms | Good | Alternative to ZGC (Red Hat) |

**G1 GC deep dive** (most commonly asked):
- Divides heap into ~2048 equal-sized **regions** (not contiguous young/old).
- Each region can be Eden, Survivor, Old, or Humongous (object > 50% of region size).
- **Mixed GC**: Collects young regions + some old regions with highest garbage ratio.
- Targets a max pause time: `-XX:MaxGCPauseMillis=200` (default).

```bash
# Production config for a payment microservice (8GB heap, G1)
java -Xms8g -Xmx8g
     -XX:+UseG1GC
     -XX:MaxGCPauseMillis=100
     -XX:G1HeapRegionSize=4m
     -XX:InitiatingHeapOccupancyPercent=45
     -Xlog:gc*:file=gc.log:time,level,tags
```

> **Interview Tip**: For SDE-2, know G1 well. If asked "how would you reduce GC pauses?", say: (1) Switch to G1/ZGC, (2) Reduce object allocation rate, (3) Tune `-XX:MaxGCPauseMillis`, (4) Increase heap to delay Full GC, (5) Analyze GC logs with tools like GCViewer or GCEasy.

---

## Q15: What causes memory leaks in Java? How do you detect and fix them? [Hard]

Despite garbage collection, Java can have memory leaks when objects are **unintentionally referenced**.

**Common causes**:

```java
// 1. Static collections that grow forever
public class LeakyCache {
    private static final Map<String, byte[]> cache = new HashMap<>();
    public static void add(String key, byte[] data) {
        cache.put(key, data); // never evicted → heap grows indefinitely
    }
    // FIX: Use WeakHashMap, Caffeine cache with maxSize, or explicit eviction
}

// 2. Unclosed resources
public void readFile(String path) {
    InputStream is = new FileInputStream(path); // never closed if exception
    // FIX: try-with-resources
}

// 3. ThreadLocal not cleaned up (covered in Q9)

// 4. Listeners/callbacks not deregistered
eventBus.register(this); // if 'this' is never unregistered, it can't be GC'd

// 5. Inner classes holding reference to outer class
public class Outer {
    private byte[] largeData = new byte[10_000_000];
    class Inner { // implicitly holds reference to Outer → largeData is retained
        void doWork() { }
    }
    // FIX: Use static inner class if you don't need outer reference
}
```

**Detection tools**:
1. **Heap dump**: `jmap -dump:format=b,file=heap.hprof <pid>` → Analyze with Eclipse MAT or VisualVM.
2. **JVM flags**: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/`
3. **Profilers**: async-profiler (allocation mode), YourKit, JFR (Java Flight Recorder).
4. **GC logs**: Look for steadily increasing old gen after each full GC.

> **Interview Tip**: A strong answer includes: "I'd enable `-XX:+HeapDumpOnOutOfMemoryError` in production, analyze the dump with Eclipse MAT, look at the dominator tree to find the largest retained objects, and trace back to the GC root to find the leak."

---

## Q16: Explain Strong, Weak, Soft, and Phantom references. [Medium]

| Type | GC Behavior | Use Case |
|---|---|---|
| **Strong** | Never collected while reachable | Default: `Object o = new Object()` |
| **Soft** (`SoftReference<T>`) | Collected only when JVM is low on memory | Memory-sensitive caches |
| **Weak** (`WeakReference<T>`) | Collected at next GC cycle (if no strong refs) | `WeakHashMap`, canonicalization maps |
| **Phantom** (`PhantomReference<T>`) | Enqueued after object is finalized, before memory is reclaimed | Pre-mortem cleanup (alternative to `finalize()`) |

```java
// Soft reference cache: values stay as long as memory allows
Map<String, SoftReference<ExpensiveObject>> cache = new ConcurrentHashMap<>();

public ExpensiveObject get(String key) {
    SoftReference<ExpensiveObject> ref = cache.get(key);
    ExpensiveObject obj = (ref != null) ? ref.get() : null;
    if (obj == null) {
        obj = loadExpensiveObject(key);
        cache.put(key, new SoftReference<>(obj));
    }
    return obj;
}

// WeakHashMap: entries automatically removed when key has no strong references
WeakHashMap<Connection, Metadata> connectionMetadata = new WeakHashMap<>();
// When the Connection object is GC'd, the entry is automatically removed
```

> **Interview Tip**: `WeakHashMap` is **NOT** a cache. It evicts entries when the *key* (not value) is weakly reachable. For caching, use `SoftReference` or a proper cache library like Caffeine.

---

## Q17: What are the different types of OutOfMemoryError and how do you debug each? [Medium]

| Error | Cause | Fix |
|---|---|---|
| `java.lang.OutOfMemoryError: Java heap space` | Heap exhausted (object allocation failed) | Increase `-Xmx`, fix memory leak, optimize object creation |
| `java.lang.OutOfMemoryError: Metaspace` | Too many classes loaded (common with heavy reflection, proxies, class generation) | Increase `-XX:MaxMetaspaceSize`, check for classloader leaks |
| `java.lang.OutOfMemoryError: GC overhead limit exceeded` | GC running >98% of time but recovering <2% of heap | Same as heap space — almost certainly a memory leak |
| `java.lang.OutOfMemoryError: unable to create native thread` | OS limit on threads reached | Reduce thread count, increase OS `ulimit -u`, reduce `-Xss` (stack size per thread) |
| `java.lang.OutOfMemoryError: Direct buffer memory` | NIO direct ByteBuffers exceeded `-XX:MaxDirectMemorySize` | Increase limit, ensure buffers are released |

**Debugging workflow**:
```bash
# Step 1: Enable heap dump on OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/logs/

# Step 2: Monitor live
jstat -gcutil <pid> 1000  # GC stats every second
jmap -histo <pid>         # live object histogram

# Step 3: Analyze heap dump
# Open in Eclipse MAT → Leak Suspects report → Dominator tree
```

> **Interview Tip**: If you mention "I've actually debugged an OOM in production," it shows real experience. Walk through the steps: got an alert, pulled the heap dump, opened MAT, found a `HashMap` with 2M entries growing unbounded, traced it to a missing cache eviction policy.

---

# Section 3: Java 8 Features

---

## Q18: Streams API — map, flatMap, reduce, groupingBy with real examples [Medium]

```java
List<Order> orders = getOrders();

// map: transform each element
List<String> customerNames = orders.stream()
    .map(Order::getCustomerName)
    .distinct()
    .collect(Collectors.toList());

// filter + map: pipeline
List<String> highValueCustomers = orders.stream()
    .filter(o -> o.getAmount() > 10_000)
    .map(Order::getCustomerName)
    .collect(Collectors.toList());

// flatMap: flatten nested collections
// Each Order has List<Item> items
List<Item> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream()) // Stream<List<Item>> → Stream<Item>
    .collect(Collectors.toList());

// reduce: aggregate to single value
double totalRevenue = orders.stream()
    .map(Order::getAmount)
    .reduce(0.0, Double::sum);

// groupingBy: group orders by status
Map<OrderStatus, List<Order>> ordersByStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));

// groupingBy + downstream collector: count per status
Map<OrderStatus, Long> countByStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()));

// partitioningBy: split into two groups (true/false)
Map<Boolean, List<Order>> partitioned = orders.stream()
    .collect(Collectors.partitioningBy(o -> o.getAmount() > 5000));
// {true: [high-value orders], false: [low-value orders]}

// toMap: with merge function for duplicate keys
Map<String, Double> revenueByCustomer = orders.stream()
    .collect(Collectors.toMap(
        Order::getCustomerName,
        Order::getAmount,
        Double::sum  // merge function: sum amounts for same customer
    ));
```

> **Interview Tip**: The most common mistake is forgetting the merge function in `toMap`. Without it, duplicate keys throw `IllegalStateException`. Always provide it unless you're sure keys are unique.

**Common follow-up**: "What's the difference between `map` and `flatMap`?" — `map` is one-to-one (transforms each element). `flatMap` is one-to-many (each element produces a stream, then all streams are merged into one).

---

## Q19: Parallel Streams — when to use and when NOT to use [Hard]

```java
// Good use case: CPU-intensive computation on large dataset, no shared state
long count = IntStream.range(0, 10_000_000)
    .parallel()
    .filter(n -> isPrime(n))
    .count();

// BAD use cases:
// 1. I/O operations — blocks ForkJoinPool threads, starving other parallel streams
orders.parallelStream()
    .forEach(o -> httpClient.post(o));  // TERRIBLE: blocks FJP threads on I/O

// 2. Small datasets — overhead of splitting > benefit of parallelism
List<String> names = List.of("a", "b", "c");
names.parallelStream().map(String::toUpperCase); // slower than sequential

// 3. Order-dependent operations
list.parallelStream().forEach(System.out::println); // order is non-deterministic
// Use forEachOrdered() if order matters (but negates most parallelism benefit)

// 4. Operations with shared mutable state
List<Integer> result = new ArrayList<>(); // NOT thread-safe
stream.parallel().forEach(result::add);   // RACE CONDITION
// FIX: use .collect(Collectors.toList()) instead
```

**How parallel streams work internally**:
- Uses `ForkJoinPool.commonPool()` (default: `availableProcessors() - 1` threads).
- Splits the source using `Spliterator`, processes chunks in parallel, merges results.
- `ArrayList` splits well (indexed access). `LinkedList` splits poorly (sequential access).

**Customizing the pool** (when you must use parallel streams for I/O):
```java
ForkJoinPool customPool = new ForkJoinPool(20);
customPool.submit(() ->
    orders.parallelStream()
        .map(this::callExternalApi)
        .collect(Collectors.toList())
).get();
```

> **Interview Tip**: The default answer should be "I avoid parallel streams in production." Use `CompletableFuture` with a dedicated `ExecutorService` for parallelism — it gives you control over pool size, timeouts, and error handling.

---

## Q20: Functional interfaces — built-in types and when to use each [Easy]

| Interface | Signature | Use Case | Example |
|---|---|---|---|
| `Function<T,R>` | `R apply(T t)` | Transform T → R | `user -> user.getName()` |
| `Predicate<T>` | `boolean test(T t)` | Filter/condition | `order -> order.getAmount() > 1000` |
| `Consumer<T>` | `void accept(T t)` | Side-effect operation | `msg -> logger.info(msg)` |
| `Supplier<T>` | `T get()` | Lazy creation/factory | `() -> new ArrayList<>()` |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs → one output | `(a, b) -> a + b` |
| `UnaryOperator<T>` | `T apply(T t)` | Transform T → same T | `s -> s.toUpperCase()` |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | Combine two T → T | `(a, b) -> a + b` in reduce |

```java
// Composing functions
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> cleanAndUpper = trim.andThen(upper);

System.out.println(cleanAndUpper.apply("  hello  ")); // "HELLO"

// Predicate composition
Predicate<Employee> isSenior = e -> e.getYears() > 5;
Predicate<Employee> isEngineer = e -> "Engineering".equals(e.getDept());
Predicate<Employee> seniorEngineer = isSenior.and(isEngineer);

List<Employee> result = employees.stream()
    .filter(seniorEngineer)
    .collect(Collectors.toList());
```

> **Interview Tip**: "What's the difference between `Function` and `UnaryOperator`?" — `UnaryOperator<T>` extends `Function<T,T>`. It's syntactic sugar for when input and output types are the same.

---

## Q21: Optional — proper usage and anti-patterns [Medium]

```java
// GOOD: return Optional from methods that may not have a result
public Optional<User> findByEmail(String email) {
    return Optional.ofNullable(userRepository.findByEmail(email));
}

// GOOD: chain operations
String city = findByEmail("shail@example.com")
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");

// GOOD: throw meaningful exception
User user = findByEmail(email)
    .orElseThrow(() -> new UserNotFoundException("No user with email: " + email));

// GOOD: conditional action
findByEmail(email).ifPresent(u -> sendWelcomeEmail(u));
```

**Anti-patterns** (what NOT to do):

```java
// BAD: Optional as method parameter
public void processUser(Optional<User> user) { } // use @Nullable or overloads

// BAD: Optional as field
class Order {
    private Optional<Discount> discount; // use null — Optional adds overhead
}

// BAD: Optional.get() without check
Optional<User> opt = findUser();
User u = opt.get(); // throws NoSuchElementException — defeats the purpose!

// BAD: isPresent() + get() — just use map/orElse
if (opt.isPresent()) {
    return opt.get().getName(); // verbose, non-functional
}
// GOOD:
return opt.map(User::getName).orElse("Anonymous");

// BAD: Optional.of(null) — throws NPE! Use Optional.ofNullable()
```

> **Interview Tip**: Brian Goetz (Java language architect) stated: "Optional was designed to be a return type for methods that may not have a result. It was NOT designed to be a field type, method parameter, or collection element."

---

## Q22: Method references — explain the 4 types with examples [Easy]

```java
// Type 1: Reference to a static method
// ClassName::staticMethod
Function<String, Integer> parser = Integer::parseInt;
// Equivalent: s -> Integer.parseInt(s)

// Type 2: Reference to an instance method of a particular object
// instance::instanceMethod
String prefix = "Order-";
Function<Integer, String> formatter = prefix::concat;
// Equivalent: id -> prefix.concat(String.valueOf(id))

// Type 3: Reference to an instance method of an arbitrary object of a particular type
// ClassName::instanceMethod (the stream element becomes 'this')
Function<String, String> upper = String::toUpperCase;
// Equivalent: s -> s.toUpperCase()

// Type 4: Reference to a constructor
// ClassName::new
Supplier<List<String>> listFactory = ArrayList::new;
// Equivalent: () -> new ArrayList<>()
Function<String, User> userFactory = User::new; // calls User(String) constructor
```

**In practice**:
```java
List<String> names = List.of("alice", "bob", "charlie");

// These are equivalent:
names.stream().map(s -> s.toUpperCase()).collect(Collectors.toList());
names.stream().map(String::toUpperCase).collect(Collectors.toList()); // Type 3
```

> **Interview Tip**: Type 3 (arbitrary object of a type) confuses people. Think of it as: the first argument becomes the object the method is called on. `Comparator<String> c = String::compareToIgnoreCase;` means `(s1, s2) -> s1.compareToIgnoreCase(s2)`.

---

## Q23: Default methods in interfaces — why were they added? What about the diamond problem? [Medium]

**Why added**: To evolve existing interfaces (like `Collection`, `List`) without breaking all implementing classes when Java 8 added `stream()`, `forEach()`, etc.

```java
public interface PaymentGateway {
    boolean charge(double amount);

    // Default method: new behavior without breaking existing implementations
    default boolean chargeWithRetry(double amount, int maxRetries) {
        for (int i = 0; i < maxRetries; i++) {
            if (charge(amount)) return true;
            sleep(1000 * (i + 1)); // exponential-ish backoff
        }
        return false;
    }
}
// Existing implementations (RazorpayGateway, StripeGateway) get chargeWithRetry() for free
```

**Diamond problem resolution**:
```java
interface A {
    default void hello() { System.out.println("A"); }
}
interface B {
    default void hello() { System.out.println("B"); }
}

// Compilation error! Must resolve conflict explicitly.
class C implements A, B {
    @Override
    public void hello() {
        A.super.hello(); // explicitly choose A's implementation
    }
}
```

**Rules**:
1. **Class wins**: A method in a class/abstract class takes priority over default methods.
2. **Sub-interface wins**: If `B extends A`, B's default wins.
3. **Compiler error**: If there's ambiguity and no class override, you must resolve it.

> **Interview Tip**: "Can interfaces have static methods in Java 8?" — Yes. Unlike default methods, static methods are NOT inherited by implementing classes. `List.of()`, `Map.of()` are examples.

---

# Section 4: Java 11/17 New Features

---

## Q24: Records — what problem do they solve? [Easy]

Records eliminate boilerplate for **immutable data carriers** (DTOs, value objects).

```java
// BEFORE (Java < 16): 50+ lines for a simple DTO
public class UserDTO {
    private final String name;
    private final String email;
    private final int age;

    public UserDTO(String name, String email, int age) { ... }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public int getAge() { return age; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// AFTER (Java 16+): 1 line
public record UserDTO(String name, String email, int age) { }
// Auto-generates: constructor, getters (name(), email(), age()),
// equals(), hashCode(), toString()
```

**Customization**:
```java
public record Money(BigDecimal amount, String currency) {
    // Compact constructor for validation
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
        currency = currency.toUpperCase();
    }

    // Custom method
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

**What Records can NOT do**:
- Cannot extend other classes (implicitly extend `java.lang.Record`).
- Cannot have mutable instance fields (all fields are `final`).
- Cannot be abstract.
- CAN implement interfaces.

> **Interview Tip**: Records are ideal for: API response DTOs, event payloads, map keys (because `equals`/`hashCode` are auto-generated based on all fields), and data transfer between layers.

---

## Q25: Sealed Classes — why restrict inheritance? [Medium]

Sealed classes let you **control which classes can extend** your class, enabling exhaustive pattern matching.

```java
// Only these three can extend Shape
public sealed class Shape permits Circle, Rectangle, Triangle {
    public abstract double area();
}

public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double area() { return Math.PI * radius * radius; }
}

public final class Rectangle extends Shape {
    private final double width, height;
    public Rectangle(double w, double h) { width = w; height = h; }
    public double area() { return width * height; }
}

public non-sealed class Triangle extends Shape {
    // non-sealed: anyone can extend Triangle
    private final double base, height;
    public Triangle(double b, double h) { base = b; height = h; }
    public double area() { return 0.5 * base * height; }
}
```

**Permitted subclass modifiers**:
- `final`: No further subclassing.
- `sealed`: Further restricts who can extend.
- `non-sealed`: Opens up the hierarchy again.

**Why it matters — exhaustive switch (Java 21 preview)**:
```java
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
    // No default needed — compiler knows all subtypes!
};
```

> **Interview Tip**: Sealed classes are Java's answer to algebraic data types (ADTs) from functional languages. They're perfect for modeling fixed sets of types: `PaymentResult` → `Success | Failure | Pending`.

---

## Q26: Pattern Matching for instanceof (Java 16) [Easy]

```java
// BEFORE: cast after instanceof
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// AFTER: binding variable in the same expression
if (obj instanceof String s) {
    System.out.println(s.length()); // 's' is already cast and scoped
}

// Works with negation too (scoping rules)
if (!(obj instanceof String s)) {
    return; // s is NOT in scope here
}
// s IS in scope here (because we know obj is a String if we reach this point)
System.out.println(s.length());

// Combine with logical operators
if (obj instanceof String s && s.length() > 5) {
    process(s); // safe: short-circuit ensures s is bound before length check
}
```

**Real-world use in equals()**:
```java
@Override
public boolean equals(Object o) {
    return (o instanceof Money m)
        && this.amount.equals(m.amount)
        && this.currency.equals(m.currency);
}
```

---

## Q27: Text Blocks, var, new String methods, and Switch Expressions [Easy]

**Text Blocks (Java 15)**:
```java
// BEFORE
String json = "{\n" +
    "  \"name\": \"Shail\",\n" +
    "  \"role\": \"SDE-2\"\n" +
    "}";

// AFTER
String json = """
        {
          "name": "Shail",
          "role": "SDE-2"
        }
        """;
// Indentation relative to closing """ determines leading whitespace trimming
```

**var (Java 10)** — local variable type inference:
```java
var list = new ArrayList<String>();       // inferred as ArrayList<String>
var stream = list.stream();               // inferred as Stream<String>
var entry = Map.entry("key", "value");    // inferred as Map.Entry<String,String>

// Can NOT use var for:
// - Fields, method parameters, return types
// - var x = null;   (can't infer type)
// - var arr = {1, 2, 3};  (array initializer)
```

**New String methods (Java 11+)**:
```java
" hello ".strip();       // "hello" (Unicode-aware, unlike trim())
" hello ".stripLeading();// "hello "
" hello ".stripTrailing();// " hello"
"".isBlank();            // true (empty or only whitespace)
"ha".repeat(3);          // "hahaha"
"line1\nline2".lines();  // Stream<String> of ["line1", "line2"]
"  ".isBlank();          // true
```

**Switch Expressions (Java 14)**:
```java
// BEFORE: statement with fall-through bugs
String result;
switch (status) {
    case "SUCCESS": result = "Completed"; break;  // forget break → bug
    case "PENDING": result = "In progress"; break;
    default: result = "Unknown";
}

// AFTER: expression with arrow syntax (no fall-through)
String result = switch (status) {
    case "SUCCESS" -> "Completed";
    case "PENDING" -> "In progress";
    case "FAILED", "ERROR" -> {
        logFailure(status);
        yield "Failed";  // 'yield' returns value from block
    }
    default -> "Unknown";
};
```

> **Interview Tip**: When using `var`, always name the variable descriptively. `var x = getResult()` is unreadable. `var paymentResult = getPaymentResult()` is fine because the name carries the type information.

---

# Section 5: ClassLoader

---

## Q28: Explain the ClassLoader hierarchy and delegation model. [Medium]

```
                     ┌────────────────────┐
                     │ Bootstrap ClassLoader │  (native C code, not a Java object)
                     │  loads: java.lang.*,   │
                     │  java.util.*, rt.jar   │
                     └────────┬───────────────┘
                              │ parent
                     ┌────────▼───────────────┐
                     │ Platform ClassLoader     │  (Java 9+; was Extension CL)
                     │  loads: javax.*, java.sql│
                     │  from jdk modules        │
                     └────────┬───────────────┘
                              │ parent
                     ┌────────▼───────────────┐
                     │ Application ClassLoader  │  (System ClassLoader)
                     │  loads: classpath classes │
                     │  your app code           │
                     └────────┬───────────────┘
                              │ parent
                     ┌────────▼───────────────┐
                     │ Custom ClassLoader       │  (e.g., Tomcat, OSGi, hot-reload)
                     └────────────────────────┘
```

**Delegation model (parent-first)**:
1. Custom CL asked to load `com.example.MyClass`.
2. Delegates to parent (Application CL).
3. Application CL delegates to Platform CL.
4. Platform CL delegates to Bootstrap CL.
5. Bootstrap can't find it → returns to Platform.
6. Platform can't find it → returns to Application.
7. Application finds it on classpath → loads it.

This prevents your code from overriding core Java classes (security).

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        // String is loaded by Bootstrap (returns null in Java)
        System.out.println(String.class.getClassLoader()); // null

        // Your class is loaded by Application ClassLoader
        System.out.println(ClassLoaderDemo.class.getClassLoader());
        // jdk.internal.loader.ClassLoaders$AppClassLoader

        // Walk the hierarchy
        ClassLoader cl = ClassLoaderDemo.class.getClassLoader();
        while (cl != null) {
            System.out.println(cl);
            cl = cl.getParent();
        }
    }
}
```

> **Interview Tip**: "Why does `String.class.getClassLoader()` return null?" — Because the Bootstrap ClassLoader is implemented in native code (C/C++), not in Java. It has no Java object representation.

---

## Q29: What is the class loading process? Explain Loading, Linking, and Initialization. [Medium]

```
Loading ──▶ Linking ──▶ Initialization
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
 Verify    Prepare    Resolve
```

**1. Loading**: Find the `.class` file (from disk, network, or generated at runtime) and create a `Class<?>` object in Metaspace.

**2. Linking**:
- **Verification**: Checks bytecode is valid (magic number `0xCAFEBABE`, valid opcodes, type safety). Prevents malicious bytecode.
- **Preparation**: Allocates memory for static fields and sets them to default values (`0`, `null`, `false`). Note: NOT the values from your code yet.
- **Resolution**: Symbolic references (class names as strings in the constant pool) are replaced with direct references (memory addresses). Can be lazy.

**3. Initialization**: Static initializers and static variable assignments execute in textual order. This is when `static { }` blocks and `static int x = 42;` run.

```java
public class InitOrder {
    static int x = 10;            // Preparation: x = 0; Initialization: x = 10
    static int y;                 // Preparation: y = 0
    static { y = x * 2; }        // Initialization: y = 20
    static final int Z = 30;     // Compile-time constant — inlined at compile time
}
```

> **Interview Tip**: "When is a class initialized?" — Only on first active use: `new`, static field access, static method call, reflection (`Class.forName()`), or if it's a superclass of an initialized class. Merely *loading* a class doesn't trigger initialization.

---

## Q30: When would you write a Custom ClassLoader? [Hard]

**Use cases**:
1. **Hot-reloading**: Load modified classes without restarting the JVM (used by Tomcat, Spring DevTools).
2. **Plugin systems**: Load plugins from external JARs at runtime (used by IDEs, OSGi).
3. **Encryption**: Load encrypted `.class` files, decrypt in memory.
4. **Isolation**: Load different versions of the same library in the same JVM.

```java
public class EncryptedClassLoader extends ClassLoader {
    private final Path classDir;
    private final byte[] encryptionKey;

    public EncryptedClassLoader(Path classDir, byte[] key, ClassLoader parent) {
        super(parent);
        this.classDir = classDir;
        this.encryptionKey = key;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            String fileName = name.replace('.', '/') + ".class";
            Path classFile = classDir.resolve(fileName);
            byte[] encrypted = Files.readAllBytes(classFile);
            byte[] decrypted = decrypt(encrypted, encryptionKey);
            return defineClass(name, decrypted, 0, decrypted.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }

    private byte[] decrypt(byte[] data, byte[] key) {
        // AES decryption logic
        return data;
    }
}

// Usage:
ClassLoader loader = new EncryptedClassLoader(
    Path.of("/secure/classes"), secretKey, getClass().getClassLoader()
);
Class<?> clazz = loader.loadClass("com.myapp.SecretAlgorithm");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

**Pitfall — ClassLoader leaks**:
Every class holds a reference to its ClassLoader, and the ClassLoader holds references to all classes it loaded. If any object from a custom ClassLoader is retained, the entire ClassLoader (and all its classes) can't be GC'd. This is the #1 cause of Metaspace leaks in application servers during redeployments.

> **Interview Tip**: Tomcat's ClassLoader architecture is a great real-world example. Each web application gets its own `WebappClassLoader` that loads classes from `WEB-INF/classes` and `WEB-INF/lib`, isolating applications from each other.

**Common follow-up**: "Can two classes with the same fully-qualified name exist in the JVM?" — Yes, if loaded by different ClassLoaders. The JVM identifies a class by its name **plus** its ClassLoader. This is how app servers run multiple versions of the same library.

---

# Bonus Questions

---

## Q31 (Bonus): What is the Java Memory Model (JMM) happens-before relationship? [Hard]

The JMM defines **happens-before** rules that guarantee visibility and ordering between threads:

| Rule | Guarantee |
|---|---|
| **Program order** | Each action in a thread happens-before every subsequent action in that thread |
| **Monitor lock** | An unlock on a monitor happens-before every subsequent lock on that monitor |
| **Volatile** | A write to a volatile field happens-before every subsequent read of that field |
| **Thread start** | `Thread.start()` happens-before any action in the started thread |
| **Thread join** | All actions in a thread happen-before another thread returns from `join()` on that thread |
| **Transitivity** | If A happens-before B, and B happens-before C, then A happens-before C |

```java
// Without happens-before, this is broken:
class Broken {
    int x = 0;
    boolean ready = false; // not volatile!

    // Thread 1
    void writer() {
        x = 42;
        ready = true;
    }

    // Thread 2
    void reader() {
        if (ready) {
            // Can print 0! Compiler/CPU may reorder: ready=true before x=42
            System.out.println(x);
        }
    }
}

// FIX: make 'ready' volatile — establishes happens-before
// The write to x (before volatile write) is guaranteed visible
// to the reader (after volatile read)
```

> **Interview Tip**: JMM is the theoretical foundation behind `volatile`, `synchronized`, and `final` field semantics. If the interviewer asks "why does double-checked locking need volatile?", the answer is: without it, the JMM allows the reader thread to see a partially constructed object due to instruction reordering.

---

## Q32 (Bonus): Double-checked locking — the classic singleton question [Medium]

```java
public class Singleton {
    // volatile is REQUIRED — prevents instruction reordering
    private static volatile Singleton instance;

    private Singleton() { }

    public static Singleton getInstance() {
        if (instance == null) {                    // first check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {             // second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Why `volatile` is needed**: Without it, `instance = new Singleton()` involves three steps:
1. Allocate memory.
2. Call constructor (initialize fields).
3. Assign reference to `instance`.

The JVM may reorder steps 2 and 3. Another thread could see a non-null `instance` with uninitialized fields.

**Better alternatives**:
```java
// Option 1: Enum singleton (recommended by Joshua Bloch)
public enum Singleton {
    INSTANCE;
    public void doWork() { }
}

// Option 2: Static holder (lazy, thread-safe, no synchronization)
public class Singleton {
    private Singleton() { }
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
        // initialized only when Holder class is loaded (first access)
    }
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

> **Interview Tip**: If asked about singleton, jump straight to the Enum approach or the static holder pattern. Only discuss double-checked locking if specifically asked — and make sure to mention `volatile`.

---

# Quick Reference: Interview Cheat Sheet

| Topic | Key Takeaway |
|---|---|
| `synchronized` vs `ReentrantLock` | Use synchronized for simple cases; ReentrantLock when you need tryLock/fairness/conditions |
| `volatile` vs `synchronized` | volatile = visibility only; synchronized = visibility + atomicity |
| Thread pool sizing | CPU-bound: cores; I/O-bound: cores × (1 + wait/compute ratio) |
| Parallel streams | Avoid in production; use CompletableFuture with custom executor |
| Optional | Return type only; never fields, params, or collections |
| Records | Immutable DTOs; replace boilerplate POJOs |
| GC for production | G1 (default, predictable); ZGC (ultra-low latency) |
| Memory leak detection | Heap dump + Eclipse MAT + dominator tree |
| ClassLoader | Parent-first delegation; custom for hot-reload/plugins |
| Deadlock prevention | Consistent lock ordering + tryLock with timeout |

---

*Last updated: March 2026 | Covers Java 8 through Java 21*
