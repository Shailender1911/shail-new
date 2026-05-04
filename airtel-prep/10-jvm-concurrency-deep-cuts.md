# File 10: JVM + Concurrency Deep Cuts

> If they go deeper than the basics, this is where you fight.
> Production scenarios over textbook definitions.

---

## SECTION 1 — JVM MEMORY (must know)

### Heap layout
```
Heap
├── Young Generation
│   ├── Eden  (most allocations)
│   ├── S0    (survivor)
│   └── S1    (survivor — copy collection swaps S0/S1)
└── Old Generation (long-lived objects after N survivor passes)

Metaspace (off-heap, native) — class metadata
Code Cache — JIT-compiled code
Direct Buffer Pool — Netty / NIO direct ByteBuffers
Stack (per thread) — frames, local primitives
```

### GC algorithms — when to use which
- **Serial GC** — single-threaded; for tiny apps / containers <100 MB.
- **Parallel GC** — throughput-focused; old default; high pauses.
- **G1 GC** — region-based; default since Java 9; low-pause target ~200 ms; sweet spot for 4-32 GB heaps. **(Our default.)**
- **ZGC** — sub-10 ms pauses; scales to TB heaps; more memory overhead. Use for latency-critical large heaps.
- **Shenandoah** — concurrent compaction; similar goals to ZGC; OpenJDK alt.

### How to talk about heap sizing
- Set `-Xms` = `-Xmx` (avoid runtime resize cost).
- Leave ~25% of container memory for off-heap (direct buffers, metaspace, native libs, code cache).
- For a 4 GB container: `-Xmx3g`, leave 1 GB for off-heap + OS.
- Monitor: Micrometer `jvm.memory.used`, `jvm.gc.pause`, `jvm.gc.live.data.size`.

---

## SECTION 2 — PRODUCTION OOM TYPES (each has different fix)

### `java.lang.OutOfMemoryError: Java heap space`
- **Cause:** heap too small for working set; or memory leak.
- **Diagnosis:** heap dump (`-XX:+HeapDumpOnOutOfMemoryError`); analyze with Eclipse MAT — look for dominator tree.
- **Common leaks:** static collections growing forever; ThreadLocal not cleared on thread reuse; listener-not-removed pattern; classloader leaks.

### `java.lang.OutOfMemoryError: GC overhead limit exceeded`
- **Cause:** JVM spending >98% time in GC, recovering <2% heap. Death spiral.
- **Fix:** more heap OR fix the leak.

### `java.lang.OutOfMemoryError: Direct buffer memory`
- **Cause:** Netty / NIO direct buffer leak (off-heap ByteBuffers not released).
- **Fix:** `-XX:MaxDirectMemorySize`; ensure `ByteBuf.release()` is called; track with Netty's leak detector.

### `java.lang.OutOfMemoryError: Metaspace`
- **Cause:** classloader leak (e.g. dynamic class generation, hot redeploy in dev).
- **Fix:** `-XX:MaxMetaspaceSize`; find the loader keeping classes alive.

### `java.lang.OutOfMemoryError: unable to create new native thread`
- **Cause:** thread leak — too many threads created and not terminated.
- **Diagnosis:** `jstack <pid>` → look for thread count + repeating thread names.
- **Common cause:** unbounded `Executors.newCachedThreadPool` under sustained load.

---

## SECTION 3 — THREAD DUMP READING

### Get a dump
```bash
jstack <pid> > thread.dump
# or
kill -3 <pid>   # JVM writes dump to stdout
```

### What to look for (in this order)
1. **`BLOCKED` threads** — waiting for monitor; show what they're waiting for.
2. **`WAITING (on object monitor)`** — `wait()` / `park()`.
3. **Deadlock detection** — `jstack` reports "Found one Java-level deadlock" automatically if it sees a cycle.
4. **Hot stack patterns** — same stack across many threads = bottleneck (often DB connection or external HTTP).
5. **Custom thread names** — proves you've named your pools (we name ours `cron-executor-1`, `pii-firewall-2`, `CNX-{serviceCode}-pool`).

### Real production patterns
- "All 50 Tomcat threads stuck in `socketRead`" → downstream (DB / API) is slow; need timeout + circuit breaker.
- "Custom executor at 100% capacity, queue full, `CallerRunsPolicy` triggered" → backpressure working as designed; check upstream rate.
- "Many threads in `LockSupport.park` on `AbstractQueuedSynchronizer`" → contention on a `ReentrantLock` or `BlockingQueue`.

---

## SECTION 4 — `ThreadLocal` PITFALLS (your real story)

### The trap
- ThreadLocal holds a reference per thread. In a thread pool, the thread is reused; the ThreadLocal value persists across requests unless cleaned.
- Memory leak: large object in ThreadLocal × N pool threads = N × size.

### How we use it correctly (your `DataSourceContextHolder`)
```java
@Around("@annotation(dataSource)")
public Object route(ProceedingJoinPoint pjp, DataSource dataSource) throws Throwable {
  DataSourceContextHolder.set(dataSource.value());
  try {
    return pjp.proceed();
  } finally {
    DataSourceContextHolder.clear();   // <-- CRITICAL
  }
}
```

### Real incident anecdote (in your back pocket)
> "We had a real incident in another integration where a request-scoped object lived in ThreadLocal but wasn't cleared in the `finally`. Under load the next request on that thread saw the previous request's data — including a JWT subject mismatch. Fixed by moving to Spring's request scope and adding the `finally` clear discipline."

---

## SECTION 5 — CONCURRENCY DEEP CUTS

### `synchronized` vs `ReentrantLock` vs `StampedLock` vs `RLock`
| Tool | Scope | Use when |
|---|---|---|
| `synchronized` | Single JVM, one method/block | Simple mutex, no timeout/fairness needs |
| `ReentrantLock` | Single JVM, explicit | Need `tryLock(timeout)`, fairness, multiple Conditions |
| `StampedLock` | Single JVM | Read-heavy with optimistic-read; advanced |
| `Redisson RLock` | Distributed across JVMs | Cross-pod coordination (your Digio dedup pattern) |

### CAS — Compare-And-Swap
- Hardware-level atomic operation: `CAS(addr, expected, new) → bool success`.
- Underpins `AtomicInteger`, `ConcurrentHashMap`, `LongAdder`.
- ABA problem: value goes A → B → A; CAS thinks unchanged. `AtomicStampedReference` / `AtomicMarkableReference` solve it.

### `LongAdder` vs `AtomicLong`
- `AtomicLong`: single CAS variable, contention-prone under load.
- `LongAdder`: striped — multiple cells, threads update local cell, sum on read. **Use for high-contention counters.**

### `ForkJoinPool` (and why we avoid it for blocking IO)
- Work-stealing pool — threads steal from each other's deques.
- Designed for compute-bound, recursively splittable tasks (mergesort-like).
- **Common ForkJoinPool** is JVM-wide; `parallelStream()` uses it.
- Block on it (HTTP call, DB query) → starvation cascades JVM-wide.
- **Our rule:** custom `ThreadPoolExecutor` for blocking IO (e.g. `cronThreadPoolExecutor` in `InsuranceThreadPoolConfig`).

### Java 21 Virtual Threads (one-line awareness)
- Lightweight, JVM-scheduled threads (M:N onto carrier threads).
- Goal: "thread per request" without thread cost.
- Caveats: pinning on `synchronized` blocks (use `ReentrantLock` instead); `ThreadLocal` works but be careful with massive thread counts.
- We're on Java 17 → not yet using; aware of the model.

---

## SECTION 6 — `volatile` DEEP CUTS

### Happens-before relationships (JMM)
- Write to `volatile` happens-before subsequent read of same variable.
- Unlocking `synchronized` happens-before subsequent lock acquisition.
- `Thread.start()` happens-before any action in the started thread.
- `Thread.join()` happens-after all actions in the joined thread.

### Double-checked locking (your `MTLSConnectionFactory` pattern)
```java
private volatile SSLContext instance;

public SSLContext get() {
  SSLContext local = instance;
  if (local == null) {
    synchronized (this) {
      local = instance;
      if (local == null) {
        local = build();
        instance = local;
      }
    }
  }
  return local;
}
```
- `volatile` is essential — without it, a partially-constructed `SSLContext` reference could leak.
- The local variable trick avoids re-reading `volatile` (faster after init).

---

## SECTION 7 — GC TUNING TALKING POINTS

### Real flags we'd tune (G1)
```
-Xms3g -Xmx3g                     # fixed heap
-XX:+UseG1GC                      # G1 (default 9+, explicit anyway)
-XX:MaxGCPauseMillis=200          # target pause
-XX:+HeapDumpOnOutOfMemoryError   # always
-XX:HeapDumpPath=/var/log/heap    # somewhere persistent
-XX:+ExitOnOutOfMemoryError       # let k8s restart pod
-Xlog:gc*:file=/var/log/gc.log    # GC logging
```

### When to switch to ZGC
- p99 GC pause matters more than throughput.
- Heap > 16 GB.
- Acceptable to spend 10-15% more memory.

---

## SECTION 8 — INTERVIEW Q&A (real production framing)

### Q: "Walk me through diagnosing a memory leak in production."
> "First — confirm the leak vs heap pressure. Look at `jvm.memory.used` over time; a leak shows monotonic growth across GCs. Heap pressure shows up as growing pause times.
>
> If it's a leak, get a heap dump (`-XX:+HeapDumpOnOutOfMemoryError` for crash dumps; `jmap -dump:format=b,file=heap.bin <pid>` for live). Open in Eclipse MAT. Look at the dominator tree — what's holding the most memory? Common culprits: static collections, ThreadLocal not cleared, classloader leaks (especially in hot-redeploy environments), listener-not-removed.
>
> Real example: in [scenario] we found a static `Map<String, Cache>` that grew unbounded because keys had a UUID component — every request created a new entry. Fixed with bounded `Caffeine` cache + size limit."

### Q: "How do you tune a thread pool?"
> "Start from the workload. CPU-bound: pool size ≈ `nCPU + 1`. IO-bound: `nCPU * (1 + waitTime/serviceTime)` — typically 2-4× CPU count.
>
> But sizing alone isn't enough — bound the queue, name the threads, set rejection policy. We use `ThreadPoolExecutor` directly (not `Executors.newCachedThreadPool` — unbounded; not `Executors.newFixedThreadPool` — unbounded queue), with `LinkedBlockingQueue` capped, named threads via `ThreadFactoryBuilder`, and `CallerRunsPolicy` so overflow back-pressures the caller instead of dropping work.
>
> Metrics matter: `executor.queue.size`, `executor.active`, `executor.rejected.count`. Alert on rejection rate > 0.
>
> Concrete example: `cronThreadPoolExecutor` in `InsuranceThreadPoolConfig` — bounded queue, named threads, CallerRunsPolicy. Survived a 10x cron-event spike during a vendor catch-up without dropping work."

### Q: "What's the difference between `submit()` and `execute()`?"
> "`execute(Runnable)` is fire-and-forget. `submit()` returns a `Future` for result/exception. `submit(Runnable)` wraps the Runnable's exception inside the Future — without calling `Future.get()`, you'd silently lose exceptions. We always either use `submit + get` or wrap Runnables with explicit logging."

### Q: "How would you implement a producer-consumer with backpressure?"
> "`BlockingQueue` (e.g. `LinkedBlockingQueue` with bounded capacity). Producer calls `put()` — blocks when queue is full (backpressure). Consumer calls `take()` — blocks when empty. Multiple consumers compete on `take()`. For shutdown: producer signals end (poison-pill or `volatile boolean done`); consumer drains remaining items."

### Q: "What's a race condition you've debugged?"
> Use the **Digio NACH callback storm** story (STAR 2 from File 03) — duplicate state transitions because callbacks arrived faster than DB writes committed. Fixed with Redisson `RLock` + `ANachLogsEntity` unique constraint.
