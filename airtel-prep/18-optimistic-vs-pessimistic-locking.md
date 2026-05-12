# File 18: Optimistic vs Pessimistic Locking — Deep Pack

> Goal: own this end-to-end. From a 10-second intuitive answer up to a senior-IC discussion covering distributed locks, deadlocks, fencing tokens, retry storms, and how I've actually used both in the PayU lending stack.
>
> Anchored to real code: `SELECT FOR UPDATE` on the loan-disbursal row, JPA `@Version` on admin status updates, **Redisson `RLock` keyed by `digio:upinach:callback:{mandateId}`** for Digio webhook idempotency (STAR 2 in v2 behavioral pack), unique-constraint-as-lock on `ANachLogsEntity`.

---

## SECTION 0 — THE 10-SECOND ANSWER (memorize first)

> **Pessimistic** = "I expect a fight. Grab the lock upfront, hold it until I'm done."
>
> **Optimistic** = "I expect peace. Read freely; check at commit time that nothing changed; retry if it did."
>
> **Pick by contention rate.** High contention → pessimistic (avoid retry storms). Low contention → optimistic (avoid lock contention). **Both are wrong if you pick them backwards.**

---

## SECTION 1 — THE INTUITIVE MENTAL MODEL

### 1.1 The shared notebook analogy

Two students need to edit the same shared notebook:

**Pessimistic** (library reading room): Student A goes to the librarian and signs out the notebook. **Nobody else can touch it** until A signs it back in. B waits at the desk. Safe, no conflicts, but B is blocked.

**Optimistic** (Google Docs offline): A photocopies the notebook, walks away, edits her copy. B does the same, independently. When they return, they compare — if both edited the same page, **the second one to return has to redo their work** based on the latest version.

Pessimistic prevents conflict. Optimistic detects it.

### 1.2 The bank-account analogy (the one to use in interviews)

Two ATM transactions try to debit ₹500 from a ₹1000 account at the same instant:

**Pessimistic:**
```
T1: SELECT balance FROM account WHERE id=1 FOR UPDATE;  -- locks row, reads 1000
T1: UPDATE account SET balance = 500 WHERE id=1;        -- holds lock
T2: SELECT ... FOR UPDATE;                              -- BLOCKED, waits for T1
T1: COMMIT;                                             -- releases lock
T2: ... resumes, reads 500, deducts to 0;
```
Sequential, correct, but T2 waited.

**Optimistic:**
```
T1: SELECT balance, version FROM account WHERE id=1;    -- reads 1000, v=5
T2: SELECT balance, version FROM account WHERE id=1;    -- reads 1000, v=5 (no lock!)
T1: UPDATE account SET balance=500, version=6 WHERE id=1 AND version=5;  -- 1 row updated
T2: UPDATE account SET balance=500, version=6 WHERE id=1 AND version=5;  -- 0 rows updated!
T2: detects conflict, re-reads (balance=500, v=6), re-applies, succeeds.
```
Parallel, correct, but T2 had to do the work twice.

> **The pattern:** pessimistic pays **latency** (waiting), optimistic pays **retry CPU** (redoing work). Pick based on which cost is cheaper for your contention rate.

---

## SECTION 2 — PESSIMISTIC LOCKING — DEEP DIVE

### 2.1 SQL-level: `SELECT ... FOR UPDATE`

The canonical pessimistic primitive in RDBMSes. Acquires an exclusive row-level lock that lasts until the transaction commits or rolls back.

```sql
START TRANSACTION;

SELECT balance, status
FROM account
WHERE id = 12345
FOR UPDATE;             -- exclusive row lock acquired here

-- ... business logic, balance checks, validation ...

UPDATE account
SET balance = balance - 500,
    last_debit_at = NOW()
WHERE id = 12345;

COMMIT;                 -- lock released here
```

**While T1 holds the lock:**
- Other `SELECT ... FOR UPDATE` on the same row → **blocked** (wait for `innodb_lock_wait_timeout`, default 50s).
- Other `UPDATE` / `DELETE` on the same row → **blocked**.
- Plain `SELECT` (no `FOR UPDATE`) on the same row → **not blocked** in InnoDB (uses MVCC snapshot), but it sees the **pre-lock** version until T1 commits.

### 2.2 The lock modes you must know

| Mode | SQL syntax | Allows others to | Use when |
|---|---|---|---|
| **Exclusive (X)** | `SELECT ... FOR UPDATE` | Nothing (read with same mode blocked) | About to update |
| **Shared (S)** | `SELECT ... LOCK IN SHARE MODE` (MySQL) / `FOR SHARE` (PG) | Read with shared, not exclusive | Validating but not yet writing |
| **Update intent (UPDLOCK in SQL Server)** | `WITH (UPDLOCK)` | Reads, but blocks other update-intents | Read-then-maybe-update |

**Cross-question trap:** *"Does `SELECT ... FOR UPDATE` lock the table?"* → **No, it's a row-level lock** in InnoDB / Postgres for indexed lookups. **But** if your `WHERE` predicate doesn't use an index, InnoDB may lock every scanned row — effectively a table lock. **Always run `EXPLAIN` to confirm the lock is row-level.**

### 2.3 Gap locks and next-key locks (MySQL-specific)

InnoDB's default isolation is `REPEATABLE READ`. To prevent phantom reads, it locks not just the row but also the **gap** before it (range lock).

```sql
SELECT * FROM orders WHERE id BETWEEN 100 AND 200 FOR UPDATE;
```

This locks **every existing row in 100-200 AND the gaps between them**, so no concurrent insert can land in that range. **This is why high-contention range-scan-then-update workloads deadlock so often in MySQL.**

**Mitigations:**
- `READ COMMITTED` isolation → disables gap locks, but allows phantom reads. Trade-off.
- Use point lookups (`WHERE id = X`) when possible, not ranges.

### 2.4 JPA / Hibernate pessimistic

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findByIdForUpdate(@Param("id") Long id);
```

Or imperatively:
```java
Account account = entityManager.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE);
```

Lock modes:
- `PESSIMISTIC_READ` → SQL `LOCK IN SHARE MODE` (shared)
- `PESSIMISTIC_WRITE` → SQL `FOR UPDATE` (exclusive)
- `PESSIMISTIC_FORCE_INCREMENT` → exclusive + bump `@Version` even if no fields change (combines with optimistic)

**Timeout** (avoid indefinite waits):
```java
em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE,
    Map.of("javax.persistence.lock.timeout", 3000));  // 3 seconds
```

### 2.5 Distributed pessimistic — when DB-row-locks aren't enough

DB row locks work when the contended resource **is a DB row**. When the contended resource is across services, an external API, or doesn't have a DB row yet — you need a distributed lock.

**Options:**

| Lock backend | When to use | Caveat |
|---|---|---|
| **Redis (Redisson `RLock`)** | High-throughput, sub-ms locks, idempotency keys | Not perfectly safe under partition (Redlock critique) — fine for "best effort"  / coordination |
| **ZooKeeper / etcd** | Correctness-critical coordination, leader election | Higher latency (~10ms), operationally heavier |
| **DB advisory lock (`pg_advisory_lock`, MySQL `GET_LOCK`)** | When you already have a DB and want one-machine semantics | Tied to a single DB; doesn't scale beyond it |
| **DB unique constraint** | Idempotency / "create-or-fail" pattern | Not really a lock — just a "loser fails cleanly" mechanism |

#### Redisson `RLock` (the one I use in production)

```java
RLock lock = redisson.getLock("digio:upinach:callback:" + mandateId);
boolean acquired = lock.tryLock(2, 30, TimeUnit.SECONDS);
//                  ^waitTime ^leaseTime
if (!acquired) {
    log.warn("dedup-bypass: could not acquire lock for mandateId={}", mandateId);
    // graceful degradation: unique constraint on event_id is safety net
    return;
}
try {
    // critical section: process Digio callback
    processCallback(payload);
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

**Why `tryLock(waitTime, leaseTime)` instead of `lock()`:**
- `waitTime = 2s` → if I can't get the lock in 2s, give up (don't block the request thread forever).
- `leaseTime = 30s` → even if my pod dies holding the lock, Redis auto-releases after 30s (no permanent zombies).

> **Real anchor (STAR 2 — Digio NACH webhook):** the partner retries on any non-200, callbacks arrive faster than DB writes commit. Redisson `RLock` serializes per-mandate processing across pods. Graceful degradation: if Redis is down → log and proceed, the unique constraint on `event_id` in `ANachLogsEntity` is the safety net.

### 2.6 The Redlock controversy (Martin Kleppmann's critique)

`SET key value NX PX 30000` (the Redis-native lock primitive) and its multi-node version Redlock have **known correctness issues under partition + clock drift**:

1. Client A acquires lock, then **stalls** (GC pause, network hiccup, 30s passes).
2. Lock expires automatically.
3. Client B acquires the lock (legitimately, A's lease is gone).
4. Client A wakes up, **thinks it still holds the lock**, writes.
5. Two clients writing concurrently — invariant broken.

**The fix: fencing tokens.**

```
Lock returns: { token: 42, leaseTime: 30s }
Resource (e.g. file storage) refuses any write with a token < highest-seen.
A writes with token=42 → accepted, highest_seen=42.
B writes with token=43 → accepted, highest_seen=43.
A wakes up, retries write with token=42 → rejected (42 < 43).
```

**ZooKeeper / etcd give you fencing tokens natively** (zxid, mod-revision). Redis doesn't — Redisson uses some heuristics but isn't bulletproof. **For correctness-critical locks (money, leader election), use ZK/etcd. For best-effort coordination and idempotency keys, Redis is fine.**

### 2.7 Failure modes of pessimistic locking

| Failure mode | What goes wrong | Mitigation |
|---|---|---|
| **Deadlock** | T1 holds A waits B, T2 holds B waits A | Always acquire locks in same order, set `innodb_lock_wait_timeout`, retry on deadlock detection |
| **Lock contention / starvation** | Hot row, every txn waits | Reduce holding time (don't do external API calls inside lock), shard the resource, use optimistic |
| **Long-held lock** | One slow txn blocks others | Set query timeouts, never hold locks across user input or external API calls |
| **Dirty/orphan lock** | Process dies holding lock | DB recovers on commit/rollback; for distributed locks use lease/TTL |
| **Lock escalation** | Many row locks promoted to table lock | Some DBs (SQL Server) do this under memory pressure; MySQL/InnoDB doesn't escalate but can lock more than intended via gap locks |
| **Gap-lock deadlock** | Range-then-update workloads in MySQL REPEATABLE READ | Switch isolation to READ COMMITTED, or rewrite to point lookups |

### 2.8 Deadlock — what it looks like in practice

```
Txn1: BEGIN;
Txn1: SELECT ... FROM account WHERE id=1 FOR UPDATE;  -- acquires lock on row 1
Txn2: BEGIN;
Txn2: SELECT ... FROM account WHERE id=2 FOR UPDATE;  -- acquires lock on row 2
Txn1: SELECT ... FROM account WHERE id=2 FOR UPDATE;  -- waits for Txn2
Txn2: SELECT ... FROM account WHERE id=1 FOR UPDATE;  -- waits for Txn1
                                                     -- DEADLOCK
```

MySQL detects the cycle (every ~1s by default, configurable) and **kills the txn with fewer rows touched**, returning `ERROR 1213 (40001): Deadlock found when trying to get lock`.

**Application responsibility:** catch the deadlock error, retry the transaction (with exponential backoff). Don't crash; deadlocks are normal under load.

```java
@Retryable(value = CannotAcquireLockException.class,
           maxAttempts = 3,
           backoff = @Backoff(delay = 50, multiplier = 2))
@Transactional
public void debit(Long accountId, BigDecimal amount) {
    Account acc = accountRepo.findByIdForUpdate(accountId);
    // ... debit logic
}
```

**Prevention rule:** acquire locks **in a consistent order** across all code paths. If T1 always grabs accounts in ascending ID order, and T2 does the same, deadlock is mathematically impossible.

---

## SECTION 3 — OPTIMISTIC LOCKING — DEEP DIVE

### 3.1 SQL-level: version column

The canonical optimistic primitive. Every row has a `version` (integer) or `updated_at` (timestamp) that increments on every update.

```sql
-- Read (no lock)
SELECT id, balance, version
FROM account
WHERE id = 12345;
-- got: balance=1000, version=5

-- ... do business logic in app ...

-- Write (checks version; updates only if unchanged)
UPDATE account
SET balance = 500, version = 6
WHERE id = 12345 AND version = 5;

-- Inspect affected rows:
-- 1 → success, my write won.
-- 0 → conflict, someone else updated the row first. Re-read and retry.
```

**No DB lock held. No waiting. Failed writes return 0 rows affected, not an error.**

### 3.2 Timestamp-based version (the common alternative)

```sql
UPDATE account
SET balance = 500, updated_at = NOW()
WHERE id = 12345 AND updated_at = '2026-05-12 10:00:00.123';
```

**Avoid this if you can.** Two updates within the same clock tick collide. Use a monotonically-incrementing integer instead.

### 3.3 JPA / Hibernate optimistic (`@Version`)

This is the production-grade pattern. Hibernate does it transparently.

```java
@Entity
@Table(name = "account")
public class Account {

    @Id
    private Long id;

    private BigDecimal balance;

    @Version
    private Integer version;  // Hibernate manages this automatically

    // ... other fields
}
```

On `save()`:
```sql
UPDATE account
SET balance = ?, version = version + 1
WHERE id = ? AND version = ?;
```

If 0 rows affected → Hibernate throws **`OptimisticLockException`** (Spring wraps as `ObjectOptimisticLockingFailureException`).

```java
@Retryable(
    value = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 5,
    backoff = @Backoff(delay = 100, multiplier = 2)  // 100, 200, 400, 800, 1600 ms
)
@Transactional
public void updateApplicationStatus(Long appId, ApplicationStatus newStatus) {
    ApplicationBean app = applicationRepo.findById(appId).orElseThrow();
    app.setStatus(newStatus);
    applicationRepo.save(app);
    // if version mismatch, throws OptimisticLockException → @Retryable re-runs the whole method
}
```

**Critical:** the retry must **re-read** the row inside the new txn. Just retrying the save with the stale entity will keep failing.

### 3.4 Version field types — what to pick

| Type | Pro | Con |
|---|---|---|
| **`Integer` / `Long`** | Simple, monotonic, safe | Need DB migration to add |
| **`Timestamp`** | Already exists (`updated_at`) | Clock-tick collisions, timezone bugs |
| **`UUID`** | No incrementing needed | Larger storage, no "older than" semantics |
| **Hash of row contents** | No new column | Slow, false positives if column types serialize weirdly |

**My pick: `Integer` with `@Version`.** Boring, correct, well-supported.

### 3.5 The retry loop is the whole game

Optimistic locking is **only as good as your retry strategy.** Three things to get right:

1. **Bounded retries** (e.g. max 5). After that, fail loudly and let upstream decide.
2. **Exponential backoff** with jitter. Avoid thundering herd when many clients retry at the same instant.
3. **Idempotency of the underlying operation.** If the retry has side effects (sending an email, calling an external API), don't repeat them — do the side effect *after* the DB commit, or use an idempotency key.

```java
@Retryable(
    value = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 5,
    backoff = @Backoff(delay = 100, multiplier = 2, random = true)
)
@Transactional
public Result process(Long id) { ... }

@Recover
public Result recover(ObjectOptimisticLockingFailureException ex, Long id) {
    log.error("optimistic lock retries exhausted for id={}", id);
    alertingService.notify(...);
    throw new BusinessException("CONCURRENT_UPDATE_LIMIT_EXCEEDED");
}
```

### 3.6 Failure modes of optimistic locking

| Failure mode | What goes wrong | Mitigation |
|---|---|---|
| **Retry storm** | High contention → most txns conflict → most retry → load multiplies | Switch to pessimistic, or shard the resource, or add backoff with jitter |
| **Livelock** | Two clients keep beating each other in retry | Exponential backoff with random jitter |
| **Side-effect duplication** | Retry re-sends notification / re-calls API | Move side effects after commit; use idempotency keys |
| **Stale write reasoning** | App makes business decision based on `balance=1000`, conflict means decision was wrong | Re-read and re-evaluate, don't just re-write |
| **ABA problem** | Value changed A→B→A; version catches it, naive equality check wouldn't | Always use `@Version`, never compare full row state |

### 3.7 The ABA problem — why you need `@Version`, not equality

Imagine an alternative optimistic scheme: compare every column instead of a version.

```sql
UPDATE account SET balance = 500
WHERE id = 12345 AND balance = 1000;     -- "if balance is still 1000"
```

This **looks correct** but has a bug: between your read and your write, someone could have done:
1. `UPDATE account SET balance = 500`
2. `UPDATE account SET balance = 1000`

You'll see `balance = 1000` and think nothing changed. **But the row's history changed**, and any logic that depends on "this row hasn't been touched since I read it" is wrong.

`@Version` solves this because every write **increments** the version, regardless of whether the value cycled back.

---

## SECTION 4 — DECISION FRAMEWORK (when to pick which)

Walk through this **out loud** when asked. Senior-IC marker.

### 4.1 The four questions

1. **What's the contention rate?**
   - High (many txns target the same row often) → **pessimistic** wins (no retry overhead).
   - Low (most txns hit distinct rows) → **optimistic** wins (no lock overhead).

2. **What's the cost of retry?**
   - Cheap (idempotent in-memory work) → **optimistic** is fine.
   - Expensive (external API call, partner integration, notification) → **pessimistic** is safer.

3. **What's the holding time?**
   - Short (pure DB work, <100ms) → either works.
   - Long (involves I/O, user input, external call) → **never pessimistic** — you're holding a lock across uncontrolled latency.

4. **What's the consistency guarantee needed?**
   - Strict (money, identity, "only one can succeed") → **pessimistic** or **optimistic with idempotency**.
   - Soft (eventual is fine, last-writer-wins acceptable) → optimistic, or no lock at all.

### 4.2 The decision matrix

| Workload | Contention | Pick | Why |
|---|---|---|---|
| Loan disbursal (state transition `APPROVED → DISBURSED`) | **High** (retries from multiple sources) | **Pessimistic + idempotency key** | Wrong answer = double-disbursal; lock + dedup |
| User profile update from admin panel | Low (one admin, occasional edit) | **Optimistic `@Version`** | Conflict rare, retry cheap |
| Inventory decrement (e.g. limited offer) | High (flash sale) | **Pessimistic with short hold, OR atomic decrement** | `UPDATE stock SET qty=qty-1 WHERE qty>0` is even better than locking |
| Counter increment | High | **Atomic DB update** (`UPDATE ... SET count=count+1`) | No app-level lock needed |
| Bulk import / batch update | Low (rows distinct) | **No lock** + unique constraint | Concurrency is a feature, not a bug |
| Webhook idempotency | High (partner retries) | **Distributed lock + unique constraint** | Both, belt-and-suspenders |
| Application status flip (workflow) | Low (single admin/system action) | **Optimistic `@Version`** + state-machine validation | Re-read on conflict, re-validate transition |

### 4.3 The "atomic update" escape hatch

For many counter / increment / decrement workloads, **you don't need either lock**. The DB itself is atomic at the row level:

```sql
UPDATE stock SET qty = qty - 1 WHERE product_id = 42 AND qty > 0;
```

**Affected rows = 1** → succeeded. **Affected rows = 0** → out of stock. No app lock, no version, no retry.

> **Senior-IC marker:** when asked "how would you handle inventory with high contention?", **start with this**, not with `SELECT ... FOR UPDATE`. It's faster and simpler. The DB does the locking internally per row.

---

## SECTION 5 — REAL ANCHORS FROM THE LENDING STACK

### 5.1 Loan disbursal — **pessimistic + idempotency**

Disbursal is the highest-stakes write in the system. Multiple sources can trigger it: admin action, auto-disbursal cron, partner-initiated callback. We **must** ensure exactly one disbursal per approved application.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public DisbursalResult disburse(String applicationId, String idempotencyKey) {

    // Step 1 — idempotency check (already disbursed for this key?)
    if (disbursalLogRepo.existsByIdempotencyKey(idempotencyKey)) {
        return loadExistingResult(idempotencyKey);
    }

    // Step 2 — pessimistic lock on the application row
    ApplicationBean app = applicationRepo.findByIdForUpdate(applicationId);

    // Step 3 — state-machine guard
    if (app.getStatus() != ApplicationStatus.APPROVED) {
        throw new InvalidStateException(
            "Cannot disburse from status=" + app.getStatus());
    }

    // Step 4 — do the disbursal (external API)
    DisbursalResult result = lmsClient.initiateDisbursal(app);

    // Step 5 — update state + log the idempotency
    app.setStatus(ApplicationStatus.DISBURSED);
    applicationRepo.save(app);
    disbursalLogRepo.save(new DisbursalLog(idempotencyKey, result));

    return result;
}
```

**Why pessimistic here:**
- **Three layers of safety**: idempotency key, row lock, state-machine guard.
- The row lock prevents two threads / pods seeing `status=APPROVED` simultaneously.
- The external `lmsClient.initiateDisbursal` happens *inside* the lock — **but the application row is the only thing locked, and the LMS call itself is idempotency-keyed.** If a retry happens, LMS dedups.
- **Note the trade-off**: the LMS call extends the lock-hold time. If LMS is slow, our row is locked for that duration. Acceptable here because contention on a single application row is rare (each app is disbursed once).

### 5.2 Admin status flip — **optimistic `@Version`**

Admin panel allows updating application metadata (notes, custom tags, eligibility overrides). Contention is low (one admin per application at a time, usually).

```java
@Entity
@Table(name = "application")
public class ApplicationBean {
    @Id private String applicationId;
    private ApplicationStatus status;
    private String notes;
    @Version
    private Integer version;
}

@Service
public class ApplicationAdminService {

    @Retryable(value = ObjectOptimisticLockingFailureException.class,
               maxAttempts = 3,
               backoff = @Backoff(delay = 50, multiplier = 2))
    @Transactional
    public void updateNotes(String appId, String newNotes) {
        ApplicationBean app = applicationRepo.findById(appId).orElseThrow();
        app.setNotes(newNotes);
        applicationRepo.save(app);
        // Hibernate generates:
        //   UPDATE application SET notes=?, version=version+1
        //   WHERE applicationId=? AND version=?
    }
}
```

**Why optimistic here:**
- Contention is low; two admins editing the same application at the same instant is rare.
- Hold time would be long if pessimistic (admin reads form, edits for 5 minutes, clicks Save) — a pessimistic lock across the admin's coffee break would be catastrophic.
- Retry is cheap — re-read, re-apply notes, re-save.
- `@Version` catches the rare conflict (admin A and admin B editing simultaneously) cleanly.

### 5.3 Digio NACH webhook — **distributed lock + unique constraint**

> *STAR 2 in v2 behavioral pack.* Partner retries on any non-200; callbacks arrive faster than DB writes commit. Per-mandate processing must be serialized **across pods**.

```java
@Service
public class DigioNachCallbackHandler {

    private final RedissonClient redisson;
    private final ANachLogsRepository logsRepo;

    public ResponseEntity<Void> handleCallback(DigioPayload payload) {
        String mandateId = payload.getMandateId();
        String eventId = payload.getEventId();

        // Always 200 to Digio (ack first, process async)
        ResponseEntity<Void> ack = ResponseEntity.ok().build();

        // Layer 1: distributed lock — serialize per-mandate across all pods
        RLock lock = redisson.getLock("digio:upinach:callback:" + mandateId);
        boolean acquired = false;
        try {
            acquired = lock.tryLock(2, 30, TimeUnit.SECONDS);
            if (!acquired) {
                log.warn("dedup-bypass: could not lock mandateId={}, relying on unique constraint",
                         mandateId);
                // graceful degradation — fall through to unique constraint
            }

            // Layer 2: unique constraint — DB-level dedup safety net
            try {
                ANachLogsEntity entry = new ANachLogsEntity(eventId, payload);
                logsRepo.save(entry);  // throws DataIntegrityViolationException if duplicate
            } catch (DataIntegrityViolationException dup) {
                log.info("duplicate event_id={}, skipping", eventId);
                return ack;
            }

            // Layer 3: state-machine guard
            processCallback(payload);

        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
        return ack;
    }
}
```

**Three layers of safety:**
1. **Redisson `RLock`** — primary serialization across pods (fast, but lease-based, not 100% safe).
2. **Unique constraint on `event_id`** — DB-level guarantee. Even if Redis is down or the lock fails, no duplicate row exists.
3. **State-machine guard** — `if (already in target state) return`. Belt-and-suspenders.

**Why this layering:**
- The lock is a **performance optimization** — it prevents two pods from doing the same work concurrently.
- The unique constraint is the **correctness guarantee** — at-most-once persistence regardless of lock state.
- This pattern absorbs a 10x callback storm (Digio queue catch-up) without alerts.

### 5.4 Eligibility cache eviction — **no lock, use cache + write-site invalidation**

> *STAR 11 in v2 behavioral pack — the Meesho auto-disbursal cache bug.*

The lesson: **don't use locks where the actual problem is cache coherence.** I initially considered putting a lock around the eligibility-cache read; the right fix was defensive eviction at the entry point + write-site eviction hygiene + idempotency at the factory. **Locks are not a substitute for proper cache invalidation.**

---

## SECTION 6 — SIDE-BY-SIDE COMPARISON

### 6.1 The trade-off matrix

| Dimension | Pessimistic | Optimistic |
|---|---|---|
| **Contention model** | Assumes conflicts happen | Assumes conflicts are rare |
| **Locking** | Real DB-level lock | No lock; version check at commit |
| **Latency** | Higher (waiting) | Lower (no wait) |
| **Throughput under low contention** | Lower (lock overhead) | Higher |
| **Throughput under high contention** | Higher (no retry storm) | Lower (every txn retries) |
| **Deadlock risk** | Yes | No (no locks held) |
| **Retry needed** | No (lock holds until commit) | Yes (on version mismatch) |
| **Failure mode** | Deadlock, lock wait timeout | `OptimisticLockException`, retry |
| **Code complexity** | Lower (no retry loop) | Higher (retry + idempotent operation) |
| **Holding side effects** | Don't (long-held lock kills throughput) | Don't (you'll redo them on retry) |
| **DB ops impact** | Lock waits show in monitoring | Failed updates show as conflicts |
| **External API in critical section** | Risky (lock held during API call) | Safer (no lock; retry on conflict) |
| **Visibility / debuggability** | Easy (`SHOW ENGINE INNODB STATUS`) | Harder (failed updates look like no-ops without metrics) |

### 6.2 When each wins (the lookup table)

| If… | Pick |
|---|---|
| Money-moving state transitions | **Pessimistic** + idempotency |
| User-facing edits with long think-time | **Optimistic** |
| Inventory / counter decrements | **Atomic DB update** (`SET qty=qty-1 WHERE qty>0`) |
| Cross-service / cross-pod coordination | **Distributed lock** (Redisson / ZooKeeper) |
| Webhook idempotency | **Distributed lock + DB unique constraint** |
| Batch / bulk operations on distinct rows | **No lock**, parallelize |
| High-contention single hot row | **Pessimistic** + queue the work (don't make every client retry) |
| Low-contention with rare conflicts | **Optimistic `@Version`** |
| You don't know the contention rate | **Optimistic** — start optimistic, switch if retry rate is high |

### 6.3 Hybrid patterns

Real systems often combine both:

| Pattern | What it does | When to use |
|---|---|---|
| **Pessimistic outer + optimistic inner** | DB pessimistic lock + `@Version` on each sub-entity | Complex aggregates where outer integrity is critical |
| **Optimistic + idempotency key** | `@Version` to detect conflict + idempotency to make retry safe | Most modern microservice patterns |
| **Distributed lock + DB optimistic** | Redisson lock to coordinate; `@Version` as final safety | High-throughput multi-pod systems (my Digio pattern) |
| **`PESSIMISTIC_FORCE_INCREMENT`** | JPA: pessimistic lock that **also** bumps version | When you want both guarantees |

---

## SECTION 7 — CROSS-QUESTIONS WITH DETAILED ANSWERS

### CQ1. "What's the difference between optimistic and pessimistic locking?"

**Opener:** *"Pessimistic locks the row upfront; optimistic doesn't lock at all but verifies at commit time that nothing changed."*

"Pessimistic acquires a DB-level row lock (`SELECT ... FOR UPDATE`) at read time and holds it until commit; other writers block. Optimistic uses a version column — reads happen without any lock, but the update has `WHERE version = X`; if zero rows are affected, someone else won the race and the app retries. **The decision rule** is contention rate: high contention → pessimistic (no retry storm), low contention → optimistic (no lock overhead). In the lending stack: pessimistic on the application row during disbursal (`findByIdForUpdate`), optimistic with `@Version` on admin status flips."

---

### CQ2. "When would you pick pessimistic over optimistic?"

**Opener:** *"Four conditions: high contention, expensive retry, external side effects in the critical section, or strict serialization required."*

"Four:

1. **High contention** — many txns target the same row often; optimistic would retry until exhausted.
2. **Retry is expensive** — re-running the operation costs CPU, money, or wakes external systems.
3. **Side effects in the critical section** — if the retry would double-send a notification or double-call an API, lock instead.
4. **Strict serialization required** — money transfers, state-machine transitions where 'last writer wins' is wrong.

**My production example:** loan disbursal. Multiple sources can trigger it (admin, cron, partner callback), the operation calls an external LMS API and updates partner state, and we need exactly-once semantics. Pessimistic + idempotency key + state-machine guard — three layers."

---

### CQ3. "What's a deadlock and how do you prevent it?"

**Opener:** *"Two txns each hold a lock the other needs. MySQL detects the cycle, kills one. Prevent by acquiring locks in consistent order."*

"Classic 2-cycle: T1 holds row A, waits for B; T2 holds B, waits for A. The DB detects (via wait-for graph) and kills one — MySQL kills the txn with fewer rows touched, returns SQLState `40001`. **Three prevention rules:**

1. **Consistent lock order** — always acquire rows in ascending PK order; mathematically impossible to deadlock if all paths follow this.
2. **Short critical sections** — get in, do the work, get out. Don't hold locks across user input or slow API calls.
3. **Set timeouts** — `innodb_lock_wait_timeout=10` (vs default 50s) so a stuck waiter doesn't pile up.

**App-side:** catch `CannotAcquireLockException` / `DeadlockLoserDataAccessException` and retry with backoff. Deadlocks are normal under load; the app must be ready."

---

### CQ4. "What's the retry storm risk with optimistic locking?"

**Opener:** *"Under high contention, most txns fail and retry — load multiplies, latency tanks, throughput collapses."*

"With 10 clients hammering the same row:
- 1 succeeds, 9 fail version check.
- 9 retry; 1 succeeds, 8 fail.
- 8 retry; 1 succeeds, 7 fail.
- Total work to update 10 times: 1+2+3+...+10 = **55 attempts** instead of 10.

CPU load multiplies, DB connection pool exhausts, p99 latency spikes 5-10x. **This is why optimistic is wrong for high-contention workloads.** Mitigations: exponential backoff with jitter, cap retries (5 max, then fail loudly), or switch to pessimistic / atomic-update / queuing. **The metric to monitor:** optimistic retry rate. If >5% of attempts retry, you're picking the wrong tool."

---

### CQ5. "How do `@Version` and `OptimisticLockException` work in Hibernate?"

**Opener:** *"Hibernate adds the version column to every UPDATE; if zero rows match, it throws."*

"On every save:
```sql
UPDATE entity SET ..., version = ?+1 WHERE id = ? AND version = ?
```
Hibernate inspects `affected rows`. If 1 → success, in-memory version bumped. If 0 → throws `StaleObjectStateException` (JPA: `OptimisticLockException`). Spring wraps as `ObjectOptimisticLockingFailureException`. **Critical:** the next attempt must **re-read** the entity in a new transaction — the in-memory version is now stale. `@Transactional` + `@Retryable` on the method (not the save call) does this correctly. The retry boundary must encompass the read, the mutation, and the save — otherwise you keep retrying with the stale version and loop forever."

---

### CQ6. "How do you implement a distributed lock?"

**Opener:** *"Two real options: Redis (Redisson `RLock`) for performance, ZooKeeper/etcd for correctness. Pick by what failure mode you can tolerate."*

"**Redis (Redisson):** `tryLock(waitTime, leaseTime, TimeUnit)` — atomic acquire via `SET key value NX PX leaseTime` under the hood. Fast (~ms), simple, well-supported. Known caveats: lease-based, so a GC pause can let a second client legitimately acquire while you 'still hold' the lock — fixed with **fencing tokens** if the protected resource supports them.

**ZooKeeper / etcd:** sequential ephemeral znodes give you a built-in fencing token (zxid). Slower (~10-50ms), heavier ops, but bulletproof for correctness.

**My production usage:** Redisson `RLock` keyed by `digio:upinach:callback:{mandateId}` for webhook idempotency — paired with a DB unique constraint on `event_id` as the correctness fallback. The lock is a performance optimization (avoid duplicate work); the unique constraint is the correctness guarantee."

---

### CQ7. "What's a fencing token and why does it matter?"

**Opener:** *"A monotonically-increasing number issued with each lock acquisition. Resource refuses writes with a token older than the highest seen — prevents zombie writers."*

"Without fencing: client A acquires lock, GC pauses 30s, lease expires, client B legitimately acquires. A wakes, writes — two writers concurrently, invariant broken. **Fencing token:** lock returns a number (42). A's writes must include token=42. B acquires later, gets token=43. Both write to the same resource — resource tracks `highest_seen_token=43`, rejects A's write (42 < 43). **Native in ZooKeeper (zxid) and etcd (mod-revision). Not native in Redis** — Redisson uses some heuristics but isn't bulletproof. **For correctness-critical locks, use ZK/etcd.** For best-effort coordination (idempotency optimization), Redis is fine, paired with a DB unique constraint as the safety net."

---

### CQ8. "How do you handle an external API call inside a transaction?"

**Opener:** *"Don't, if you can avoid it. If you must, keep the lock window tight and have a compensation path."*

"External API calls inside `@Transactional` are dangerous:
- **Lock held during API latency** — if the API is slow, you've extended the lock for 10x its intended duration.
- **API succeeded but txn rolls back** — external system did the work, your DB doesn't reflect it. Permanent drift.
- **Timeouts can leave you in indeterminate state** — was the API call accepted or not?

**Three patterns to handle this:**

1. **Two-phase commit at the app level:** local DB txn to mark `IN_PROGRESS`, commit, call external API, then another txn to mark `COMPLETED` or `FAILED`.
2. **Saga with compensation:** call external API first (idempotency-keyed), then DB commit. On failure, the saga's compensating action calls the external API's reverse endpoint.
3. **Outbox pattern:** DB commit writes to an outbox table; separate poller calls the external API. Decouples the DB txn from external latency.

**What I do for disbursal (Section 5.1):** pessimistic lock + idempotency key. The LMS API call happens inside the txn, but the idempotency key on the LMS side ensures a retry won't double-disburse. The lock is acceptable here because we hold per-application (rare contention)."

---

### CQ9. "What's `PESSIMISTIC_FORCE_INCREMENT` in JPA?"

**Opener:** *"Pessimistic lock that also bumps `@Version`. For when you want both guarantees."*

"Default `PESSIMISTIC_WRITE` locks the row but doesn't touch the version. `PESSIMISTIC_FORCE_INCREMENT` does both — acquires the lock **and** bumps the version, even if no fields change. **Use case:** an aggregate root where the child entities are the things actually changing, but you want to signal 'something in this aggregate changed' to other readers. E.g. an order with line items — locking + bumping the order's version when a line item is added makes any cached `Order` instance go stale."

---

### CQ10. "Can you use `SELECT ... FOR UPDATE` without a transaction?"

**Opener:** *"No. The lock is released at txn end — without a txn, it's released immediately."*

"Outside a transaction, the lock is acquired and released within the single statement, giving you no protection. Always wrap in `@Transactional` or `START TRANSACTION ... COMMIT`. In MySQL with `autocommit=1`, a bare `SELECT ... FOR UPDATE` is functionally useless. **Senior-IC marker:** mention this — it's a common interview trap."

---

### CQ11. "What's the isolation level interaction with locking?"

**Opener:** *"Isolation level determines whether reads block writers and vice versa. Higher isolation = more locks = more contention."*

| Isolation | Reads block writers? | Phantom reads? | Notes |
|---|---|---|---|
| READ UNCOMMITTED | No | Yes | Dirty reads allowed — almost never used |
| READ COMMITTED | No (MVCC snapshot read) | Yes | PG default; cheap; no gap locks |
| REPEATABLE READ | No (MVCC) | No (in MySQL InnoDB, due to gap locks) | MySQL default |
| SERIALIZABLE | Effectively yes (range locks) | No | Strongest, slowest |

"**Real-world choice:** MySQL OLTP usually stays at REPEATABLE READ (default) for safety; high-throughput services drop to READ COMMITTED to **disable gap locks** and reduce deadlock rate. The trade-off is phantom reads, which the app must handle (or which don't matter for that workload). Postgres defaults to READ COMMITTED because it has MVCC for read consistency without locking."

---

### CQ12. "What if `tryLock` returns false?"

**Opener:** *"Decide based on the workload — bypass with graceful degradation, fail fast, or queue the work."*

"Three options:

1. **Graceful degradation** — log the failure, proceed without the lock, rely on a downstream safety net (DB unique constraint, state-machine check). My Digio webhook does this — if Redis is down, we still persist via the unique constraint.
2. **Fail fast** — return 503 / partner-specific 'try later' response. Use when there's no safety net.
3. **Queue the work** — push to a retry queue; process when the lock is available. Use for non-time-critical batch work.

**Anti-pattern:** spin-retry-in-place, hammering the lock. That's how you turn a brief contention spike into a full-on retry storm."

---

### CQ13. "How would you migrate a system from pessimistic to optimistic locking (or vice versa)?"

**Opener:** *"Measure first. Add the new pattern alongside the old. Migrate one workload at a time, with rollback at each step."*

"Five-step migration:

1. **Measure current behavior.** Pessimistic → record lock-wait time, deadlock rate. Optimistic → record conflict / retry rate. Establish a baseline.
2. **Add the new mechanism without removing the old.** Add `@Version` column; deploy code that maintains version but still uses `FOR UPDATE`. No behavior change yet.
3. **Switch one workload.** Pick the lowest-stakes workload (e.g. admin metadata update), switch to optimistic, watch metrics for 2 weeks.
4. **Iterate workload-by-workload.** Each workload is independent. Don't switch them all at once.
5. **Decommission the old path.** Drop `FOR UPDATE` from queries where it's no longer needed; drop `@Version` column from tables where pessimistic remained.

**What I'd never do:** big-bang switch on a Friday. Locking changes have subtle correctness implications; migrate slowly with metrics."

---

### CQ14. "What's the relationship between locking and CAP?"

**Opener:** *"Locking implies coordination, which implies consistency, which under partition costs availability."*

"Strong locks (single-leader pessimistic, linearizable distributed lock) are inherently CP — under partition, you cannot acquire the lock from the minority side, so writes stop. Optimistic locking is more partition-tolerant: each side can read and attempt to write; conflict detection happens at the merge point (which may be after partition heals). **The trade-off:** stronger locking = stronger consistency = lower availability under partition. **Real-world implication for distributed locks:** Redis-based locks (AP-ish) are best-effort under partition; ZooKeeper-based locks (CP) refuse to issue locks during a quorum loss. Pick by what failure mode you can absorb."

---

### CQ15. "What's a 'lost update' problem?"

**Opener:** *"Two txns read, both modify, second-to-write overwrites first's change as if it never happened."*

"Without any locking:
- T1 reads `balance=1000`.
- T2 reads `balance=1000`.
- T1: `UPDATE balance=900` (T1 deducted 100).
- T2: `UPDATE balance=800` (T2 deducted 200, but based on stale 1000 read).
- **Final balance: 800.** T1's update was lost — both should have left it at 700.

**Both pessimistic and optimistic solve this**, differently:
- Pessimistic: T2's `SELECT FOR UPDATE` blocks until T1 commits.
- Optimistic: T2's `UPDATE WHERE version=5` fails because T1 bumped version to 6. T2 retries with fresh read.

**What does NOT solve it:** REPEATABLE READ alone. RR gives you snapshot isolation for reads, not write conflict detection. **You need explicit locking.**"

---

### CQ16. "Is `UPDATE balance = balance - 100 WHERE id = X` atomic? Do I need a lock?"

**Opener:** *"Yes, it's atomic at the row level. You don't need an app-level lock for atomic updates."*

"Each `UPDATE` statement in InnoDB / Postgres acquires a brief row-level lock internally, performs the update, releases. **`balance = balance - 100` reads and writes in one operation — no chance of a stale read.** For pure counters / increments / decrements, this is the cheapest possible solution: no `FOR UPDATE`, no `@Version`, no retry. **Limitation:** doesn't help when the operation requires a check beyond the immediate column. E.g. 'deduct 100 if user is active' — that requires either `WHERE active=true` (atomic check) or a read + lock pattern."

---

### CQ17. "What if I need to coordinate across multiple rows?"

**Opener:** *"Lock all in consistent order, or use a single 'coordinator' row, or use a serializable transaction."*

"Three options:

1. **Consistent lock order** — always acquire rows in ascending PK order. Prevents the classic 2-cycle deadlock.
2. **Coordinator row** — designate one parent row (account, order, user) as the 'lock point.' Always lock the coordinator before touching any of its children. Used in DDD aggregate root pattern.
3. **SERIALIZABLE isolation** — DB does the coordination for you. Slowest, but correct without explicit locks. Postgres's SSI (serializable snapshot isolation) is actually efficient for moderate-contention workloads.

**Real example (PayU):** when processing a multi-loan repayment, we lock loans in ascending `loan_id` order across all code paths. Deadlocks went from a daily occurrence to zero."

---

### CQ18. "How do you debug a deadlock in production?"

**Opener:** *"`SHOW ENGINE INNODB STATUS` gives you the latest deadlock. For systemic issues, enable deadlock logging."*

"In MySQL:
```sql
SHOW ENGINE INNODB STATUS;
```
The `LATEST DETECTED DEADLOCK` section shows both transactions, the locks they hold, the locks they wait for, the SQL each was running. **Read it carefully** — usually one of:
- Two paths acquire same rows in different order → fix lock order.
- Range lock + insert collision → drop to READ COMMITTED or rewrite to point lookups.
- App-side retry causes second deadlock cycle → tune backoff.

**For systemic debugging:** enable `innodb_print_all_deadlocks=ON` to log every deadlock (not just the latest), then aggregate by SQL fingerprint in your log pipeline. **What I monitor in prod:** deadlock rate per minute; alert if >10/min, page if >100/min."

---

### CQ19. "Difference between `SELECT FOR UPDATE` and `SELECT FOR UPDATE NOWAIT`?"

**Opener:** *"`NOWAIT` doesn't wait — fails immediately if lock unavailable. Use for explicit short-circuit logic."*

"`SELECT FOR UPDATE`: waits up to `innodb_lock_wait_timeout` (default 50s). `SELECT FOR UPDATE NOWAIT`: returns immediately with an error if the lock isn't free. `SELECT FOR UPDATE SKIP LOCKED`: skips locked rows entirely — useful for **work-queue patterns** where multiple workers grab unique unlocked rows.

```sql
SELECT * FROM job_queue
WHERE status = 'PENDING'
ORDER BY created_at LIMIT 10
FOR UPDATE SKIP LOCKED;
```
Each worker grabs 10 unlocked jobs; concurrent workers don't fight over the same ones. **This pattern is gold for SQL-backed job queues.** Postgres 9.5+, MySQL 8+."

---

### CQ20. "When should I just use a queue instead of locking?"

**Opener:** *"When the work is asynchronous-tolerant and high-contention — serialize through a queue, eliminate the lock entirely."*

"If you find yourself with a hot row that every txn wants to update, the right answer is often **not** to optimize locking, but to remove the contention entirely:

- **Sharding** the resource (10 buckets instead of 1, hash the key).
- **Queueing** writes — every request enqueues a message, single consumer processes serially. Throughput is the consumer's rate, but no app-level locks needed.
- **CRDTs** for eventually-consistent counters / sets — concurrent operations always merge to the same result.
- **Batching** — accumulate updates, flush together.

**Example:** counting page views. Instead of `UPDATE views SET count = count+1` (deadlock heaven at scale), enqueue an event to Kafka, batch-aggregate by minute, single writer flushes to DB. **The lock is a symptom; the cause is wrong architecture.**"

---

## SECTION 8 — RAPID-FIRE Q BANK (20 one-liners)

1. **"Is `SELECT` blocked by `SELECT ... FOR UPDATE`?"** → No in InnoDB (MVCC snapshot read). Yes for `SELECT ... LOCK IN SHARE MODE`.
2. **"Is `@Version` a real DB lock?"** → No. It's a column with a check at write time. Zero locking overhead.
3. **"What error indicates an optimistic lock conflict?"** → `OptimisticLockException` (JPA) / `ObjectOptimisticLockingFailureException` (Spring).
4. **"What error indicates a pessimistic lock timeout?"** → `CannotAcquireLockException` (Spring) / SQLState `40001` for deadlock, `HY000` for lock wait timeout.
5. **"Default `innodb_lock_wait_timeout`?"** → 50 seconds. Almost always too high for OLTP — tune to 5-10s.
6. **"How does MySQL detect deadlocks?"** → Wait-for graph; checked on each lock wait.
7. **"Which txn does MySQL kill on deadlock?"** → The one with the smaller transaction (fewer rows modified).
8. **"Does READ COMMITTED prevent lost updates?"** → No. You need explicit locking or `@Version`.
9. **"What's `SKIP LOCKED` for?"** → Job queues, work distribution — skip rows other workers are processing.
10. **"Can you upgrade a shared lock to exclusive?"** → Yes (`LOCK IN SHARE MODE` → `FOR UPDATE`), but it's a common deadlock source if two txns try simultaneously.
11. **"Is Redisson's `RLock` reentrant?"** → Yes, by default.
12. **"What's a livelock?"** → Two clients keep beating each other in retries, neither making progress. Fix with backoff + jitter.
13. **"What's the cost of `@Version`?"** → One integer column + one `+1` per update. Negligible.
14. **"What's the cost of `SELECT FOR UPDATE`?"** → Lock overhead + contention. Measurable under heavy concurrent access.
15. **"What's a phantom read?"** → Re-running a range query in the same txn returns new rows because another txn inserted into the range.
16. **"How do you retry an optimistic conflict?"** → `@Retryable(OptimisticLockException, maxAttempts=5, backoff=...)` wrapping the entire `@Transactional` method (not just the save).
17. **"Best practice for distributed lock TTL?"** → Set lease longer than your worst-case work, with a sanity-check that you still hold the lock before committing. Use fencing if the resource supports.
18. **"Redis `SETNX` vs Redisson `RLock`?"** → `SETNX` is the primitive; Redisson wraps it with reentrancy, lease renewal (watchdog), pub-sub for fairness.
19. **"Why is gap locking a problem?"** → Lockable range is bigger than expected; range queries cause deadlocks under concurrent inserts.
20. **"What's the simplest atomic increment without any lock?"** → `UPDATE counter SET count = count + 1 WHERE id = ?`. Row-level atomic.

---

## SECTION 9 — CHEAT SHEET (visual recall)

### 9.1 The pick-one table

```
HIGH CONTENTION                          LOW CONTENTION
┌────────────────────┐                  ┌────────────────────┐
│   PESSIMISTIC      │                  │   OPTIMISTIC       │
│  SELECT FOR UPDATE │                  │   @Version         │
│  @Lock(WRITE)      │                  │   No DB lock       │
│  Redisson RLock    │                  │   Retry on conflict│
└────────────────────┘                  └────────────────────┘
        ↓                                       ↓
SHORT HOLDING                            LONG HOLDING (user think-time)
External API? Risky.                     Mandatory — no lock through I/O.
Lock-wait timeout matters.               Retry backoff matters.
```

### 9.2 The decision tree

```
Is the contention rate > 5%? ──Yes──→ Pessimistic
                              └─No──→ Is retry expensive (external side effects)? ──Yes──→ Pessimistic
                                                                                  └─No──→ Optimistic @Version
                                       
Multiple processes / pods coordinating? ──Yes──→ Distributed lock (Redisson/ZK)
                                                  + DB unique constraint as safety net
```

### 9.3 The "what error means what" table

| Error | Means | Action |
|---|---|---|
| `OptimisticLockException` | Version mismatch (someone won the race) | Re-read in fresh txn, re-apply, retry |
| `LockAcquisitionException` / `40001` deadlock | DB killed your txn | Retry with backoff |
| `LockTimeoutException` | Lock wait > timeout | Retry, or shorten holding times in offending paths |
| `tryLock` returns false | Distributed lock unavailable | Graceful degradation OR fail fast |
| `DataIntegrityViolationException` (unique key) | Duplicate insert | Treat as idempotent success or business conflict |

---

## SECTION 10 — REAL ANCHOR STORIES (use these in interviews)

### 10.1 "Tell me about a locking decision you made"

> **Use this when:** asked about concurrency, design, or DB choices.

"In the loan-disbursal path I deliberately chose **pessimistic + idempotency-keyed**. Multiple sources can trigger disbursal — admin UI, cron, partner webhook — and the operation makes an external LMS call. Three layers of safety: idempotency key (caller-provided), `SELECT ... FOR UPDATE` on the application row, and state-machine guard inside the txn. The lock prevents two pods from both seeing `status=APPROVED` at the same instant; the idempotency key prevents the LMS API from double-disbursing if a retry slips through. **Why not optimistic:** the retry cost is unacceptable — re-calling the LMS to discover 'already disbursed' is much more expensive than holding a row lock for ~200ms. And the contention rate, while low per application, is **bursty** during cron windows; pessimistic absorbs the burst cleanly. **What I'd never do here:** drop the lock and rely solely on idempotency. Idempotency makes retries safe; the lock makes them rare. Both layers earn their cost."

### 10.2 "Tell me about a time you used a distributed lock"

> **Anchored to STAR 2 (Digio NACH).**

"The Digio NACH webhook intake — partner retries on any non-200, callbacks arrive faster than DB writes commit, multiple pods running the consumer. I used **Redisson `RLock` keyed by `digio:upinach:callback:{mandateId}`** to serialize per-mandate work across pods. `tryLock(waitTime=2s, leaseTime=30s)` — bounded wait, auto-release on pod death. **Critical layering:** the lock is a *performance* optimization; the **DB unique constraint on `event_id`** is the *correctness* guarantee. If Redis is down, we log `dedup-bypass` and proceed — the unique constraint blocks duplicates regardless. **Result:** a 10x callback storm during a Digio queue catch-up absorbed without alerts. **What I learned about distributed locks:** they're best-effort under partition. Always pair with a DB-level invariant. Don't bet correctness on the lock alone."

### 10.3 "Tell me about a deadlock you debugged"

"Mandate cron — bulk repayment processing across multiple loans per merchant. Production deadlock rate climbed to ~100/hour. `SHOW ENGINE INNODB STATUS` showed two txns each holding one loan row and waiting for another — classic 2-cycle. Two code paths were acquiring loans in **different orders**: the cron sorted by `loan_id ASC`, the admin retrigger path sorted by `due_date ASC`. **Fix:** standardize lock acquisition to `loan_id ASC` everywhere, regardless of business sort order. Deadlock rate dropped to zero. Added an automated test that runs the same workload from both paths concurrently and asserts no deadlock occurs. **What I learned:** deadlock prevention is a *static property* of code, not a runtime concern. Lock order needs to be consistent across all code paths, enforced by review and ideally by abstraction (a `LoanLockingService` that always sorts before locking)."

---

## SECTION 11 — FINAL CHECKLIST (the night before)

- Read Section 0 (10-second answer) and Section 4 (decision framework) **word-for-word**.
- Memorize the **bank-account analogy in Section 1.2** — it's the cleanest way to explain both in 30 seconds.
- Drill **CQ1, CQ2, CQ3, CQ4, CQ6** — guaranteed to be asked.
- Pre-load **Section 5 anchors** — when asked "where have you used pessimistic locking?", you should not hesitate.
- Glance at **Section 9 cheat sheet** — visual recall under pressure.
- Skim **Section 6.2 lookup table** — answers "what would you pick for X workload?" without thinking.
- Be ready for **the trap:** *"why not just use SERIALIZABLE for everything?"* — answer: throughput collapse, distributed deadlocks, and you still need to choose a retry strategy. Isolation is not a substitute for explicit locking decisions.
- Close this file. Sleep.

> **The senior-IC marker:** when asked about locking, don't go "well it depends" — name the contention rate, name the holding time, name the retry cost, *then* pick. The senior signal is **deciding correctly out loud**, not knowing both options exist.
