# File 20: Redis Deep Pack — Internals, Clients, Patterns, Anti-Patterns

> Goal: own Redis end-to-end at interview depth. From "why is single-threaded fast" through "Jedis vs Lettuce vs Redisson with a verdict" through "how would you build a token-bucket rate limiter."
>
> Anchored to your PayU usage: `RedisConfig.getCacheManagerConfigs()` cache managers, Redisson `RLock` for Digio webhook (STAR 2), `@Cacheable` on `ApplicationDBService.selectApplication`, defensive eviction at `checkEligibilityAsync` (STAR 11). Cross-refs files 17, 18, 19.

---

## SECTION 0 — THE 10-SECOND MENTAL MODEL

> *"Redis is an in-memory key-value store with rich data structures (strings, lists, sets, sorted sets, hashes, streams, HyperLogLog, geo, bitmaps), optional persistence (RDB + AOF), master-replica replication, Sentinel for HA or Cluster for horizontal sharding, and single-threaded command execution that gives you atomic ops for free. It excels at caching, locking, rate limiting, sessions, leaderboards, pub/sub, and as a fast queue. It is never the source of truth for money."*

---

## SECTION 1 — INTERNALS

### 1.1 Single-threaded event loop — and why it's still fast

Redis serves all commands from a **single thread** that runs an event loop (epoll on Linux, kqueue on BSD). Sounds slow. Isn't, for three reasons:

1. **No locks, no contention.** Every command is atomic by design. No mutex overhead, no deadlock potential.
2. **CPU cache friendly.** One thread → one core's L1/L2 cache stays hot.
3. **Memory-bound, not CPU-bound.** The bottleneck is RAM bandwidth, not CPU cycles. Adding threads doesn't help when you're waiting on memory.

**What a single Redis instance pushes:** ~100K-200K ops/sec on commodity hardware; ~1M+ ops/sec with pipelining. That's faster than most things you'd put in front of it.

**Multi-threading since 6.0:** I/O threads can do socket read/write in parallel; **command execution stays single-threaded.** This unblocks the network stack on high-connection servers.

### 1.2 Data structures (the user-facing primitives)

| Type | Big-O reads | Big-O writes | Use cases |
|---|---|---|---|
| **String** (binary-safe, up to 512MB) | O(1) | O(1) | Cache values, counters (INCR), bit flags |
| **List** (linked list or quicklist) | O(N) by index, O(1) at ends | O(1) at ends | Queues, recent-items, timelines |
| **Set** (unordered unique) | O(1) member check | O(1) add/remove | Tags, unique visitors, deduplication |
| **Sorted set** (skiplist + hashtable) | O(log N) rank/range | O(log N) add | **Leaderboards, time-windowed counters, priority queues** |
| **Hash** (field-value map) | O(1) field get | O(1) field set | Object storage, sessions |
| **Stream** (5.0+, append-only log) | O(log N) range | O(1) append | Event log with consumer groups |
| **HyperLogLog** (probabilistic count) | O(1) cardinality | O(1) add | Unique-visitor count at scale, ~0.8% error |
| **Bitmap** (string used as bit array) | O(1) bit test | O(1) bit set | Per-day active-user flags, presence |
| **Geo** (uses sorted set internally) | O(log N) | O(log N) | "Nearby" queries |

**The senior-IC marker:** when interviewer says "build a leaderboard," **immediately** say "sorted set with ZADD / ZREVRANGE." Not "MySQL ORDER BY." Not "in-memory tree." Sorted set is the answer.

### 1.3 Encoding internals (why memory tuning matters)

Redis silently switches between encodings based on size/count thresholds. Small collections use compact memory layouts; large ones switch to standard hash tables / linked lists.

| Type | Small encoding | Threshold | Large encoding |
|---|---|---|---|
| Hash | `listpack` (was `ziplist` pre-7.0) | ≤ 128 fields, all values ≤ 64 bytes | `hashtable` |
| List | `listpack` / `quicklist` | similar | `quicklist` (linked listpacks) |
| Set | `intset` (small int-only) or `listpack` | ≤ 128 entries | `hashtable` |
| Sorted set | `listpack` | ≤ 128 entries, values ≤ 64 bytes | `skiplist + hashtable` |
| String | `embstr` (≤ 44 bytes) or `int` (if numeric) | thresholds | `raw` (separate alloc) |

**Why this matters in production:** a hash with 100 fields of small values uses ~5x less RAM than 100 small hashes with 1 field each. **Batch small fields into bigger hashes** for memory efficiency — this is how Twitter's tweet cache used to be designed.

**How to check:** `OBJECT ENCODING key`.

### 1.4 Memory management

- **jemalloc** allocator (faster fragmentation handling than glibc malloc).
- **Fragmentation ratio** = `used_memory_rss / used_memory`. > 1.5 means fragmentation. Restart or `MEMORY PURGE`.
- **`maxmemory`** policy enforces eviction when full (see Section 4).
- **Expiration sampling:** every 100ms, sample 20 random keys with TTL; expire any that should be gone. Plus passive expiration on touch.

### 1.5 Persistence on disk

Two mechanisms (Section 2 deep dive). At a high level:
- **RDB**: periodic snapshots; fast restart; loss window = since last snapshot.
- **AOF**: every write appended; replay on restart; loss window = fsync policy.
- **Hybrid (4.0+)**: AOF rewrite starts with an RDB-format prefix for compactness.

---

## SECTION 2 — PERSISTENCE DEEP DIVE

### 2.1 RDB (Redis Database snapshot)

**What it is:** periodic point-in-time snapshot of the entire dataset.

**Trigger:** time-based (`save 900 1` = save if 1+ key changed in 900s) + manual (`SAVE` synchronous, `BGSAVE` async fork).

**How fork() works:**
1. Redis calls `fork()`. OS uses copy-on-write — child gets a virtual copy of parent's memory.
2. Child writes RDB file to disk.
3. Parent continues serving traffic; only modified pages get copied.

**Pros:** fast restart (single file load), small file, useful for backups.

**Cons:** loss window = time since last snapshot. fork() can cause latency spikes on large datasets (10GB+ instance can pause for seconds).

**Tuning:** disable `save` rules and trigger BGSAVE only from cron if you can tolerate longer loss windows; reduces fork pressure.

### 2.2 AOF (Append-Only File)

**What it is:** every write command appended to a log file. On restart, Redis replays the log.

**Three fsync policies:**

| Policy | Latency | Data-loss window |
|---|---|---|
| `appendfsync always` | Highest (fsync on every write) | ~0 |
| `appendfsync everysec` | Low (fsync once per second) | **~1 second** (typical choice) |
| `appendfsync no` | Lowest (OS flushes at will) | Up to 30 seconds (Linux default) |

**AOF rewrite** (compaction): the log grows forever. `BGREWRITEAOF` triggers a fork + emit minimal commands to reconstruct current state. Auto-triggered at `auto-aof-rewrite-percentage` (default 100, i.e. doubled).

**Hybrid mode (4.0+):** rewrite produces an RDB-format prefix + AOF tail with commands since the rewrite. **Best of both:** fast restart (RDB load) + low loss window (AOF tail). **Use this by default.**

### 2.3 The decision matrix

| Use case | Pick | Why |
|---|---|---|
| Cache only (can rebuild from source) | RDB or no persistence | Loss window doesn't matter; minimize fsync overhead |
| Session store | AOF `everysec` + RDB hourly | ~1s loss acceptable; faster restart |
| Distributed lock backing | AOF `everysec` minimum | Lock state must survive crash |
| Source-of-truth-ish (rare for Redis) | AOF `always` + RDB hourly | Trade throughput for durability |
| Lightning fast (low durability needed) | `appendfsync no` + frequent BGSAVE | Accept up to 30s loss |

**My production default:** **AOF `everysec` + hybrid mode + BGSAVE hourly + replication to a replica.** Same trade-off as PayU's lending stack.

### 2.4 Persistence interaction with replication

- The **replica's RDB/AOF is independent** of the master's. You can have RDB on master, AOF on replica.
- **Initial replication** sends a full RDB dump from master to replica, then streams the replication backlog.
- **Diskless replication** (since 2.8.18): master streams RDB directly to replica socket without writing to disk first. Useful for fast SSDs vs slow disks.

---

## SECTION 3 — REPLICATION + HA + CLUSTER

### 3.1 Master-replica replication

- **Async replication** — master sends commands; replica applies. Lag = replica's processing time.
- **PSYNC** (partial resync) — replica reconnects, sends last offset; master streams missed commands from backlog (sized by `repl-backlog-size`).
- **Full sync** required if replica is offline longer than backlog covers — RDB dump again.

**Lag impact:** if you read from a replica, you may see stale data (file 18 CQ4 — eventual consistency).

### 3.2 Redis Sentinel (HA without sharding)

**Purpose:** monitor master + replicas, auto-failover when master is down.

**Architecture:**
- Run 3+ Sentinel processes (quorum-based decisions; odd number prevents split-brain).
- Sentinels gossip via pub/sub; agree on master state.
- On master failure: quorum of Sentinels declare `+sdown`, then `+odown`, then elect a leader Sentinel, leader promotes a replica to master, others reconfigure.

**Client discovery:** clients ask Sentinel `SENTINEL get-master-addr-by-name mymaster` → returns current master IP. Jedis / Lettuce / Redisson support this natively.

**Failover time:** typically 10-30s (Sentinel down-after-milliseconds + election).

**When to use:** moderate dataset (≤ ~100GB), high availability requirement, no need for horizontal sharding.

### 3.3 Redis Cluster (horizontal sharding)

**Purpose:** shard data across multiple nodes for capacity beyond one machine.

**Architecture:**
- **16384 hash slots** total.
- Each master owns a range of slots.
- Key → slot: `CRC16(key) mod 16384`. Hash tags `{userId}:profile` force same slot for related keys.
- Each master has 0+ replicas (usually 1).
- **Gossip protocol** between nodes: `PING/PONG` every second; full topology view exchanged.

**Client handling:**
- Smart client knows the slot → node map.
- `MOVED` redirect: client cached wrong node; client updates and retries.
- `ASK` redirect: slot is migrating; one-time redirect.

**Limitations:**
- **No cross-slot multi-key ops** unless keys share hash tag. `MGET k1 k2 k3` only works if all on same slot.
- **No Lua scripts across slots.** Same hash-tag restriction.
- **No multi-DB** (only DB 0).

**Failover:** masters elect a replica when peer masters detect a master is down (`FAIL` flag). Similar to Sentinel but built into the cluster.

**When to use:** dataset > 100GB, write throughput > one node, geo-distributed sharding.

### 3.4 Sentinel vs Cluster decision

| | Sentinel | Cluster |
|---|---|---|
| Sharding | No | Yes (16384 slots) |
| Capacity per cluster | Single-node RAM | N nodes × RAM each |
| HA | Yes (via Sentinel quorum) | Yes (built-in) |
| Multi-key ops | All allowed | Only same-slot via hash tag |
| Lua scripts | Multi-key allowed | Same-slot only |
| Operational complexity | Lower | Higher |
| Failover speed | 10-30s | Similar |

**My choice in production:** **Sentinel** for moderate scale (most PayU services). Cluster when capacity exceeds one node, which is rare unless you're storing huge session stores or rich data per key.

---

## SECTION 4 — EVICTION POLICIES (when memory is full)

Set via `maxmemory-policy`.

| Policy | Behavior | When to use |
|---|---|---|
| `noeviction` (default) | Writes fail with OOM error | **Dangerous** — only if Redis is source-of-truth and you want loud failure |
| `allkeys-lru` | Evict least-recently-used across all keys | **Standard cache** — best general-purpose |
| `allkeys-lfu` | Evict least-frequently-used (4.0+) | Better than LRU when access patterns are stable; learns hot keys |
| `volatile-lru` | LRU on keys with TTL only; ignore non-TTL keys | When mixing cache (TTL'd) + permanent data (no TTL) |
| `volatile-lfu` | LFU on keys with TTL | Same; better learning |
| `allkeys-random` | Evict random | Bizarre choice; almost never right |
| `volatile-random` | Evict random with TTL | Same |
| `volatile-ttl` | Evict keys closest to expiry | When you want predictable expiry |

**Sampling:** LRU/LFU are **approximated** — Redis samples 5 keys (`maxmemory-samples`) and evicts the worst. Higher = closer to true LRU but slower. Default 5 is fine.

**Production default:** **`allkeys-lru`** for cache; **`noeviction`** with alerting if Redis is critical-state (locks, rate-limit counters).

**The trap:** `noeviction` + write-heavy workload + memory pressure = writes start failing. Application thinks "Redis is up" because reads work; writes silently bounce. **Always monitor `evicted_keys` and `OOM` errors.**

---

## SECTION 5 — THREADING MODEL (6.0+)

### 5.1 The model in 7.x

| Component | Thread count | Notes |
|---|---|---|
| Command execution | **1** (still single-threaded) | The classic atomic-ops property is preserved |
| I/O (socket read/write) | **N** (configurable via `io-threads`) | 6.0+; helps high-connection servers |
| Background tasks | A few | `BGSAVE`, AOF rewrite, expiration sampling, `UNLINK` (async DEL) |
| Replication backlog | 1 | Sender thread |

**`io-threads` tuning:** start with 4-8 for typical workloads. Helps when network is the bottleneck (lots of small clients).

### 5.2 Background ops you should know

- **`BGSAVE`** — fork + write RDB. Non-blocking command-wise; fork can stall.
- **`BGREWRITEAOF`** — similar; fork + write AOF.
- **`UNLINK key`** vs **`DEL key`** — UNLINK is async (background); use for big keys (>10K elements) to avoid blocking the event loop.
- **`FLUSHDB ASYNC`** / **`FLUSHALL ASYNC`** — async flush; doesn't block.

---

## SECTION 6 — PUB/SUB VS STREAMS

### 6.1 Pub/Sub

**Fire-and-forget messaging.**

```
PUBLISH channel msg
SUBSCRIBE channel
```

- No persistence; if subscriber is down, message is **lost**.
- Slow subscribers don't block publishers (server drops if subscriber buffer fills past `client-output-buffer-limit`).
- Pattern subscriptions via `PSUBSCRIBE chan.*`.

**When to use:** real-time notifications where occasional loss is fine (chat, presence, cache invalidation broadcasts).

**When NOT to use:** anything requiring delivery guarantees. Use Streams or Kafka.

### 6.2 Streams (5.0+)

**Persistent append-only log with consumer groups.**

```
XADD stream * field1 v1 field2 v2          -- append, auto-ID
XREADGROUP GROUP mygroup c1 COUNT 10 STREAMS stream >  -- consume
XACK stream mygroup msg-id                  -- ack processed
XPENDING stream mygroup                     -- see unacked
XCLAIM stream mygroup c2 60000 msg-id       -- re-claim from dead consumer
```

- **Consumer groups** for parallel consumption (each consumer gets disjoint messages).
- **At-least-once delivery** via XACK + XPENDING.
- **Retention** by `MAXLEN` (count) or `MINID` (oldest ID).
- **Replay** by reading from older ID.

**When to use:** at-least-once delivery; consumer groups for parallelism; some Kafka-like patterns.

**When NOT to use:** infinite retention; very high throughput (millions/sec). Use Kafka.

### 6.3 The decision

| Need | Pick |
|---|---|
| Best-effort real-time, no replay | Pub/Sub |
| At-least-once + consumer groups + replay (limited) | Streams |
| Infinite retention + multiple downstream consumers | **Kafka** (not Redis) |

---

## SECTION 7 — TRANSACTIONS + LUA

### 7.1 MULTI / EXEC (transactional pipeline)

```
MULTI
SET key1 v1
INCR key2
EXEC
```

- Commands queued; executed atomically as a batch.
- **Not a true transaction** — no rollback on error. If `INCR` fails (e.g. type mismatch), `SET` still ran.
- All-or-nothing? **No.** Each command runs independently inside EXEC.
- **WATCH for optimistic concurrency:**
  ```
  WATCH key1
  val = GET key1
  // compute new val
  MULTI
  SET key1 newVal
  EXEC          // returns nil if key1 changed since WATCH; retry on app side
  ```

**Use case:** compare-and-swap. Same pattern as file 18 optimistic locking, applied to Redis.

### 7.2 Lua scripting

**Atomic execution of multiple commands in a single script.**

```lua
-- Atomic decrement-if-positive
local v = tonumber(redis.call('GET', KEYS[1]))
if v and v > 0 then
    redis.call('DECR', KEYS[1])
    return 1
else
    return 0
end
```

- Script runs **atomically** — no other commands interleave.
- Same key-slot constraint in Cluster mode.
- **`EVALSHA`** caches the script by SHA; subsequent calls reuse.
- **Limit:** keep scripts < 5ms; longer scripts block the event loop.

**Use case:** any compound op that needs atomicity — rate limiter, leaderboard updates, conditional puts. **Lua > MULTI/EXEC** for anything with conditionals.

---

## SECTION 8 — PIPELINING

**Pipelining** = sending multiple commands without waiting for each response. Server reads all, processes, sends all responses.

```java
Pipeline p = jedis.pipelined();
for (int i = 0; i < 1000; i++) {
    p.set("k" + i, "v" + i);
}
List<Object> results = p.syncAndReturnAll();
```

**Why fast:** removes N round-trips (`N × RTT`) and replaces with 1. **10-50x throughput gain** for small commands.

**Not the same as transactions.** Pipelining is just batching network I/O; each command can still be interleaved with another client's commands. For atomicity use MULTI or Lua.

**Limitations:**
- Memory: server buffers responses until pipeline is drained.
- Latency: first response only after all sent.
- Cluster mode: must group commands by slot.

---

## SECTION 9 — JAVA REDIS CLIENTS DEEP DIVE (your explicit ask)

### 9.1 The three clients

| | **Jedis** | **Lettuce** | **Redisson** |
|---|---|---|---|
| **Style** | Sync, blocking | Async, non-blocking (Netty) | High-level distributed objects (on top of Lettuce/Netty) |
| **Thread safety** | One connection per thread (needs pool) | Single connection, multiplexed, thread-safe | Like Lettuce + higher abstractions |
| **API level** | Low-level (GET, SET, ZADD) | Low-level + reactive (Mono/Flux) | High-level (`RLock`, `RMap`, `RAtomicLong`, `RBucket`, `RRateLimiter`, `RBloomFilter`, `RScheduledExecutor`) |
| **Cluster support** | Yes (since 3.x) | Yes (native) | Yes (best-in-class) |
| **Sentinel support** | Yes | Yes | Yes |
| **Reactive** | No (blocking only) | Yes (Reactor + RxJava) | Yes (Reactor + RxJava + async API) |
| **Maintained** | Yes (Redis Labs) | Yes (Redis-cofounder, default in Spring Boot 2+) | Yes (commercial backing; Pro tier paywalled) |
| **Maturity** | Oldest, simplest | Production-grade default | Production-grade, opinionated |
| **Connection pool** | Required (Apache Commons Pool) | Single connection (no pool needed for most ops) | Internal pool managed |
| **Spring Boot default** | Available | **Default since 2.0** | Drop-in replacement |
| **Pipelining** | Manual | Automatic batching | Automatic |
| **Transaction support** | MULTI/EXEC manual | MULTI/EXEC manual | High-level abstractions (`RTransaction`) |
| **Distributed primitives** | None native | None native | **RLock**, **RReadWriteLock**, **RSemaphore**, **RCountDownLatch**, **RTopic** (pub/sub abstraction), **RRateLimiter**, **RBloomFilter**, **RBucket**, **RAtomicLong**, **RScheduledExecutorService**, **RMap with cache + TTL** |
| **License** | MIT | Apache 2.0 | Apache 2.0 (Pro = commercial) |

### 9.2 Hands-on examples — same task, three clients

**Task: atomic GET-then-SET if value matches expected (CAS).**

**Jedis (manual WATCH/MULTI/EXEC):**
```java
try (Jedis jedis = pool.getResource()) {
    while (true) {
        jedis.watch("counter");
        String current = jedis.get("counter");
        if (!"42".equals(current)) {
            jedis.unwatch();
            break;
        }
        Transaction tx = jedis.multi();
        tx.set("counter", "43");
        List<Object> result = tx.exec();
        if (result != null) break;  // null = WATCH failed, retry
    }
}
```

**Lettuce (async, manual WATCH/MULTI/EXEC):**
```java
StatefulRedisConnection<String, String> conn = client.connect();
RedisAsyncCommands<String, String> async = conn.async();
async.watch("counter").thenCompose(__ ->
    async.get("counter")
).thenCompose(current -> {
    if (!"42".equals(current)) return async.unwatch();
    return async.multi()
        .thenCompose(_x -> async.set("counter", "43"))
        .thenCompose(_x -> async.exec());
});
```

**Redisson (high-level):**
```java
RAtomicLong counter = redisson.getAtomicLong("counter");
boolean swapped = counter.compareAndSet(42, 43);
```

**Verdict:** Redisson hides the complexity. Same op in 1 line vs 10.

### 9.3 Distributed lock — same comparison

**Task: acquire distributed lock keyed by mandate ID, lease 30s, wait up to 2s.**

**Jedis (DIY with `SET key value NX PX`):**
```java
try (Jedis jedis = pool.getResource()) {
    String lockId = UUID.randomUUID().toString();
    String result = jedis.set("lock:" + key, lockId, SetParams.setParams().nx().px(30000));
    boolean acquired = "OK".equals(result);
    if (acquired) {
        try { /* work */ }
        finally {
            // Atomic release with Lua to check ownership
            String lua = "if redis.call('GET', KEYS[1]) == ARGV[1] " +
                         "then return redis.call('DEL', KEYS[1]) else return 0 end";
            jedis.eval(lua, 1, "lock:" + key, lockId);
        }
    }
}
```
- No retry-on-busy.
- No lease renewal (watchdog).
- No fencing.
- Manual ownership check on release.

**Lettuce (same DIY pattern):** code is identical except async API.

**Redisson:**
```java
RLock lock = redisson.getLock("digio:upinach:callback:" + mandateId);
if (lock.tryLock(2, 30, TimeUnit.SECONDS)) {
    try { /* work */ }
    finally { if (lock.isHeldByCurrentThread()) lock.unlock(); }
}
```
- **Reentrant** by default.
- **Watchdog** auto-renews lease while holder is alive (no premature expiry on long ops).
- **Cluster-aware**.
- **Fair locks** (`getFairLock`) and **MultiLock** (Redlock implementation) available.

**Verdict:** Redisson wins decisively for anything beyond plain GET/SET.

### 9.4 The verdict (the user's explicit question)

> **For caching only → Lettuce (Spring Boot default). For distributed locks, rate limiting, leaderboards, semaphores, atomic counters, scheduled tasks across pods, or anything beyond GET/SET → Redisson on top.** Jedis only if you have a legacy reason or specifically want sync-blocking for simplicity.

**Why this is the senior answer:**

1. **Lettuce is the Spring Boot 2+ default.** If you start a new Spring Boot project today and add `spring-boot-starter-data-redis`, you get Lettuce automatically. Good for caching, sessions, simple Redis use.
2. **Redisson is best-in-class for distributed primitives.** If your service needs `RLock` (you almost always do, eventually), `RRateLimiter`, `RBloomFilter` — Redisson saves you weeks of building these by hand, badly.
3. **Jedis is showing its age.** Connection-per-thread is brittle (pool sizing is a constant pain); no native async; the API hasn't evolved much. Use it only if a legacy codebase is locked in.

**The PayU stack uses Redisson** for exactly this reason: the Digio webhook needs `RLock` (STAR 2), the cache managers (`RedisConfig.getCacheManagerConfigs()`) use Redisson's TTL + max-idle config, and we use `RBucket` / `RAtomicLong` for various counters. **Lettuce alone would force us to build distributed primitives by hand.**

**Spring Boot configuration to switch from Lettuce to Redisson:**

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.27.x</version>
</dependency>
```

That's it. The starter auto-replaces the Lettuce connection factory with Redisson's; `RedisTemplate` continues to work for cache use; `RedissonClient` is injectable for distributed primitives. **Best of both: Spring Boot ergonomics + Redisson power.**

### 9.5 When Jedis still makes sense

- Legacy codebase already on Jedis; migration cost > value.
- Very simple use case (cache only, sync, low concurrency); team explicitly wants blocking simplicity.
- Specific Redis module integration (older Jedis sometimes ships module support faster).

### 9.6 When Lettuce alone makes sense (without Redisson)

- Pure caching with `@Cacheable`; no distributed locks or counters needed.
- Reactive stack (Spring WebFlux); want native `Mono` / `Flux` integration.
- Team strongly prefers building distributed primitives themselves (rare, usually wrong).

---

## SECTION 10 — PATTERNS AT SCALE

### 10.1 Cache patterns

| Pattern | How it works | Pros | Cons |
|---|---|---|---|
| **Cache-aside** | App checks cache; on miss, fetches from DB, populates cache | Simple; cache failures don't break writes | Stale risk if writes bypass cache |
| **Write-through** | App writes to cache + DB synchronously | Cache always fresh | Write latency = max(cache, DB); both must succeed |
| **Write-behind** | App writes to cache; cache flushes to DB async | Fast writes | Data loss if cache crashes before flush |
| **Refresh-ahead** | Cache proactively refreshes near TTL expiry | No miss-storms | More cache traffic |

**Production defaults:**
- Read-heavy with stale tolerance → **cache-aside**.
- Write-heavy, no loss tolerance → **write-through**.
- Money-moving decisions → **defensive eviction at the entry point** (STAR 11 — file 17 CQ13).

### 10.2 Distributed locking — Redisson `RLock`

Already covered in file 18 (Section 2.5) and Section 9.3 above. **Key gotcha: pair with a DB-level invariant (unique constraint or state check) — the lock is performance, the invariant is correctness.**

### 10.3 Rate limiting — token bucket (atomic Lua)

(See file 19 Section 3.4 for full Lua script.)

**Redisson's `RRateLimiter`:**
```java
RRateLimiter limiter = redisson.getRateLimiter("api:merchant:" + merchantId);
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);
if (limiter.tryAcquire()) { /* allow */ } else { /* reject 429 */ }
```

One line vs 30 lines of Lua. **Use Redisson unless you need custom burst behavior.**

### 10.4 Leaderboards — sorted sets

```
ZADD leaderboard 1500 "user:42"        -- update score
ZREVRANGE leaderboard 0 9 WITHSCORES   -- top 10
ZRANK leaderboard "user:42"            -- rank of user
ZRANGEBYSCORE leaderboard 1000 2000    -- range query
```

**100M users?** Single sorted set works to ~100M entries in one Redis instance. Shard by user-id-prefix if you need more.

### 10.5 Session store

```
HSET session:abc123 user_id 42 role admin last_seen 1715000000
EXPIRE session:abc123 1800   -- 30 min sliding window
```

Slide TTL on access. Mark active sessions. Boom — session store in 2 commands. No DB table needed.

### 10.6 Distributed counter

```
INCR pageviews:2026-05-12
INCRBY revenue:merchant:42 1500
```

Atomic; no app-level lock needed (file 18 CQ16).

### 10.7 Pub/Sub at scale

- Avoid > 10K subscribers per channel (broadcast cost is O(N)).
- Use **shard-channels** by hash or by region.
- For durability, switch to Streams or Kafka.

### 10.8 Cache stampede prevention

**Problem:** popular key expires; 1000 concurrent requests miss; all hit DB; DB overwhelmed.

**Solutions:**
1. **Probabilistic early expiration** — refresh `random()` % of the time before TTL hits, smoothing the curve.
2. **Mutex on miss** — only one thread fetches from DB, others wait. Redisson `RReadWriteLock` shines here.
3. **Stale-while-revalidate** — serve stale data while one thread refreshes. Built into `RMapCache` in Redisson.

---

## SECTION 11 — ANTI-PATTERNS (what NOT to do)

| Anti-pattern | Why bad | Use instead |
|---|---|---|
| `KEYS *` in prod | O(N) full scan; blocks event loop (single-threaded!) | `SCAN` with cursor |
| Large keys (>1MB hash/list/set) | Single command blocks event loop | Split into smaller keys |
| Hot keys (>50% traffic on one key) | One server pegged; rest idle | Shard with `key:{shardId}`; read replicas |
| `BLPOP` / `BRPOP` on main thread | Blocks until pop; defeats single-threaded model | Use timeout; or use Streams |
| `SAVE` in prod | Synchronous fork; blocks entire instance | `BGSAVE` (async) |
| `DEL` on big keys | Blocks while freeing memory | `UNLINK` (async free) |
| `EXPIRE` without renewal in sliding-window scenario | Session/lock expires unexpectedly | Refresh TTL on access (or use watchdog) |
| Treating Redis as source-of-truth for money | RDB loss window, AOF `everysec` 1s gap, master failover edge cases | Pair with durable DB |
| Long-running Lua scripts (>5ms) | Blocks all other clients | Break into chunks; offload to app code |
| Pub/Sub for at-least-once delivery | No persistence; subscriber down = message lost | Use Streams or Kafka |
| `noeviction` + write-heavy load | Writes silently fail when full | Use LRU/LFU + monitor `evicted_keys` |
| `SET key val NX PX 30000` as a lock | No fencing, no watchdog, no reentrancy | Redisson `RLock` |
| Storing JSON blobs as strings + parsing N times | Inefficient; full re-parse on every read | Use Hash for fields, or RedisJSON module |
| One field per hash | Loses memory-tuning benefit of ziplist/listpack encoding | Batch fields into single hash |

**The "one bad command" rule:** any single Redis command should complete in **< 1ms** ideally, **< 5ms** worst-case. Anything longer blocks every other client. Profile with `SLOWLOG GET 10`.

---

## SECTION 12 — PAYU REDIS ANCHOR (real production usage)

### 12.1 Cache managers (from `RedisConfig.getCacheManagerConfigs()`)

```java
configs.put(A_USER_APPLICATION_CHACHE_MANAGER, getCacheConfig(120000, 120000));         // 120s TTL, 120s max-idle
configs.put(A_CONFIG_CACHE_MANAGER, getCacheConfig(72000000, 28800000));               // 20h TTL, 8h max-idle
configs.put(A_CONFIG_REF_CACHE_MANAGER, getCacheConfig(75600000, 32400000));           // 21h TTL, 9h max-idle
configs.put(A_APPLICATION_STAGE_TRACKER_CACHE_MANAGER, getCacheConfig(79200000, 3600000));  // 22h TTL, 1h max-idle
```

**Why these specific values:**
- **Application cache (120s)** — frequent reads; tight TTL because the underlying `ApplicationBean` mutates often. STAR 11 lesson: TTL is not coherence; we evict at every write site.
- **Config cache (20h)** — partner config changes rarely; long TTL acceptable; refreshed on explicit invalidation event.
- **Stage tracker (22h)** — stage history is append-only; once written, never mutates; long TTL is safe.

### 12.2 `@Cacheable` annotations

```java
@Cacheable(value = A_USER_APPLICATION_CHACHE_MANAGER, key = "#applicationId + '_' + #tenantId")
public ApplicationBean selectApplication(String applicationId, Integer tenantId) { ... }

@Cacheable(value = A_CONFIG_CACHE_MANAGER, key = "#key")
public ConfigBean getConfig(String key) { ... }
```

### 12.3 Distributed lock (STAR 2 — Digio webhook)

```java
RLock lock = redisson.getLock("digio:upinach:callback:" + mandateId);
boolean acquired = lock.tryLock(2, 30, TimeUnit.SECONDS);
// ... process callback inside critical section ...
```

(Full pattern in file 18 Section 5.3.)

### 12.4 Defensive eviction (STAR 11 — Meesho cache bug)

```java
public Response checkEligibilityAsync(String applicationId, ...) {
    // The fix that came out of the Meesho auto-disbursal cache bug:
    cacheEvictByKey.evictApplicationCache(applicationId);
    applicationBean = applicationService.selectApplication(applicationId, tenantId);
    // ... eligibility decision based on fresh data
}
```

**Lesson:** any read that drives a money-moving decision evicts its own cache at entry. Belt-and-suspenders against missed write-site evictions elsewhere.

### 12.5 What we don't use Redis for (and why)

- **Source of truth for money state** — never. Backed by MySQL.
- **Long event log** — Kafka, not Redis Streams. Retention requirements (7 years) exceed Redis economics.
- **Search** — Elasticsearch, not Redis. RediSearch module exists but not the right shape for our workload.

---

## SECTION 13 — CROSS-QUESTIONS WITH DETAILED ANSWERS

### CQ1. "Why is Redis single-threaded and still fast?"

**Opener:** *"Memory-bound, not CPU-bound. Single thread eliminates locks and keeps the CPU cache hot."*

"Redis serves all commands from one thread on an event loop (epoll). Three reasons it's fast despite the single-thread model:

1. **No locks, no contention.** Every command is atomic by design — no mutex overhead, no deadlock.
2. **CPU cache friendly.** One thread → one core's L1/L2 cache stays warm.
3. **Memory-bound bottleneck.** Adding threads doesn't help when you're waiting on memory bandwidth.

A single instance pushes 100K-200K ops/sec on commodity hardware, 1M+ with pipelining. **Faster than most things you'd put in front of it.** Since 6.0, I/O threads parallelize the socket layer, but command execution stays single-threaded — preserving atomicity."

---

### CQ2. "RDB vs AOF — when to pick which?"

**Opener:** *"RDB for fast restart + backups; AOF for durability. Default to hybrid (both) since 4.0."*

(See Section 2.3 decision matrix.)

---

### CQ3. "What happens if Redis runs out of memory?"

**Opener:** *"Depends on `maxmemory-policy`. `noeviction` = writes fail; `allkeys-lru` = evict cold keys."*

"Eight policies (Section 4). The dangerous default for many setups is `noeviction` — writes fail with `OOM command not allowed when used memory > 'maxmemory'`. **Reads continue working, so monitoring 'Redis is up' lies.** Always monitor `evicted_keys` count and OOM-error rate. **Production default: `allkeys-lru` for cache, with alerting on eviction rate spike.**"

---

### CQ4. "How does Redis Cluster shard data?"

**Opener:** *"16384 hash slots. Each master owns a range. Key → slot via CRC16(key) mod 16384."*

(See Section 3.3.)

---

### CQ5. "How does Sentinel decide to fail over?"

**Opener:** *"Quorum-based. 3+ Sentinels watch master; quorum of Sentinels declare it down; leader Sentinel promotes a replica."*

"Each Sentinel runs `down-after-milliseconds` ping check on master. After timeout → `+sdown` (subjectively down). When **quorum** of Sentinels agree → `+odown` (objectively down). Sentinels elect a leader (Raft-ish algorithm). Leader picks the best replica (lowest replication offset diff, highest priority) → promotes it. Others reconfigure to replicate from new master.

**Time to failover:** 10-30s typical. **Reads can continue from replicas during failover; writes block until promotion completes.**"

---

### CQ6. "Pub/Sub vs Streams — when to pick which?"

**Opener:** *"Pub/Sub for best-effort real-time; Streams for at-least-once + consumer groups. Kafka for infinite retention."*

(See Section 6.3.)

---

### CQ7. "What's the difference between WATCH and MULTI?"

**Opener:** *"WATCH is optimistic concurrency setup; MULTI begins the queued transaction. Together they implement CAS."*

"`WATCH key` marks the key. If anyone modifies it before `EXEC`, the transaction aborts (EXEC returns nil). `MULTI` opens a queue; commands accumulate; `EXEC` atomically runs them. Combined: read with WATCH, compute, MULTI + writes + EXEC. If EXEC = nil → retry. **Same pattern as `@Version` in JPA (file 18) but for Redis values.**"

---

### CQ8. "When would you use Lua over MULTI?"

**Opener:** *"Whenever you need conditional logic. Lua is atomic and allows if/else; MULTI just queues."*

"MULTI runs a fixed list of commands. Lua runs **arbitrary logic atomically**. If your op is 'decrement IF positive, ELSE return 0,' you need Lua. If your op is just 'do these three commands together,' MULTI is fine.

**Lua caveat:** keep scripts < 5ms. They block the event loop. Cache via `EVALSHA` for repeated calls.

**Real example: token-bucket rate limiter** (file 19 Section 3.4) — read tokens, compute refill, conditionally decrement. Lua is the only sane way."

---

### CQ9. "Jedis vs Lettuce vs Redisson — your pick?"

(See Section 9.4 — the headline question.)

---

### CQ10. "How do you handle a hot key?"

**Opener:** *"Shard the key by suffix; or use read replicas; or front it with an L1 cache in app memory."*

"Three options:

1. **Shard the key.** `counter:{shardId}` for shardId in 0..N. Aggregator reads all N and sums. Spreads load across slots / nodes.
2. **Read replicas.** Route reads to replicas (with stale-tolerance). Writes still go to master, but writes are usually <5% of traffic.
3. **App-side L1 cache.** Caffeine in-process; refresh every 100ms. Reduces Redis reads by 100-1000x.

**Detection:** `redis-cli --hotkeys` (in 4.0+) samples hot keys. Or use a histogram of command timings in your observability stack."

---

### CQ11. "How do you detect a big key in production?"

**Opener:** *"`redis-cli --bigkeys` for sampling; `MEMORY USAGE key` for spot check; alert on `memory_used_per_key`."*

"`--bigkeys` walks all keys (with SCAN) and reports the biggest per type. **One-shot diagnostic, not continuous.** For ongoing detection, sample with SCAN + MEMORY USAGE in a cron, alert if any single key > 1MB.

**Big keys are dangerous because:** any command on them blocks the event loop. A 100K-element hash with HGETALL can take 100ms on the single thread, freezing everything else."

---

### CQ12. "What's `allkeys-lru` and when is it wrong?"

**Opener:** *"Standard cache eviction policy. Wrong when access pattern is bimodal (most keys cold but few hot are also rare)."*

"`allkeys-lru` evicts the least-recently-used keys. Right for most caches. Wrong cases:

- **Bimodal access** — most keys are cold, but a few hot keys are also accessed rarely. LRU evicts them when memory pressure hits. Switch to **LFU**, which learns access frequency.
- **TTL-mixed** — mixing cache keys (TTL'd) with permanent state. LRU may evict your locks. Use `volatile-lru` or split into separate instances.
- **Source-of-truth Redis** (rare) — eviction is destruction. Switch to `noeviction` and **alert when memory hits 90%.**"

---

### CQ13. "Why never `KEYS *` in prod?"

**Opener:** *"O(N) scan that blocks the single-threaded event loop. Use SCAN with cursor instead."*

"`KEYS *` walks every key in the database. On a 10M-key Redis, that's seconds of blocking — and during those seconds, **every other client is frozen.** `SCAN cursor MATCH pattern COUNT 100` is incremental; gives back partial results; safe to call concurrently with writes (you may see some keys twice or miss new ones, but the server isn't blocked)."

---

### CQ14. "Redis as a queue — when does it fall apart?"

**Opener:** *"At-least-once + consumer groups + durability needs. Streams help but Kafka wins past low millions of events/day."*

"Simple FIFO with LPUSH/RPOP works for low-throughput, best-effort queues. **Fails at:**

- **At-least-once delivery** — RPOPLPUSH gives ack-and-resume but no consumer groups.
- **Multiple consumers in parallel** — without consumer groups, each consumer competes for the same items.
- **Replay / retention** — once popped, gone.

**Streams (5.0+) fix some of this** — XREADGROUP, XACK, XPENDING, XCLAIM. Works for at-least-once + consumer groups + bounded replay.

**Past tens of thousands of msg/s or weeks of retention → Kafka.** Redis Streams isn't a Kafka replacement; it's a Kafka-shaped tool for moderate workloads."

---

### CQ15. "How does Redis persistence interact with replication?"

**Opener:** *"Replica RDB/AOF is independent of master. Initial sync sends full RDB; subsequent updates streamed."*

(See Section 2.4.)

---

### CQ16. "What's a fencing token and does Redis support it?"

**Opener:** *"A monotonically-increasing token issued with each lock. Resource refuses writes with token < highest seen. Redis doesn't natively support it; ZooKeeper does."*

(Same as file 18 Section 2.6 and CQ7.)

---

### CQ17. "How does Redisson's watchdog work?"

**Opener:** *"Background thread renews the lock lease while the holder is alive. Prevents premature expiry on long ops."*

"When you `tryLock(waitTime, /*no leaseTime arg*/)`, Redisson uses a default lease (30s) + starts a **watchdog timer** in a background thread. Every 10s, watchdog checks: is the lock still held by this thread? If yes, renew the lease to another 30s. If the JVM dies or the thread releases, the timer stops and the lease expires naturally.

**Why it matters:** without it, your `lock(30s)` would expire mid-work if the work takes 31s, and another client would acquire — classic split-brain. Watchdog automatically extends.

**Caveat:** if you pass an explicit `leaseTime` (e.g. `tryLock(2, 30, SECONDS)`), watchdog is **disabled** — you opted into manual lease management."

---

### CQ18. "Can Redis lose data? When?"

**Opener:** *"Yes, in three ways: AOF fsync gap, RDB snapshot gap, master failover with replication lag."*

"Three scenarios:

1. **AOF `everysec`**: data written in the last 0-1 second isn't fsynced; lost on crash.
2. **RDB only**: data since last snapshot is lost on crash.
3. **Master failover**: replica was N ms behind; that N ms of writes is lost when replica is promoted.

**Mitigations:**
- AOF `always` (slow) + replication for #1.
- Hybrid mode + frequent BGSAVE for #2.
- Wait for replica ack on critical writes (`WAIT N timeout`) for #3 — but it slows you down.

**The right answer for money:** don't use Redis as source-of-truth. Pair with MySQL/Postgres. (File 17, file 18 anchors.)"

---

### CQ19. "How does EXPIRE actually evict — active or passive?"

**Opener:** *"Both. Lazy expiration on access + active sampling every 100ms."*

"**Lazy:** every time a key is accessed, Redis checks if it's expired. If yes, evict and return null/false.

**Active:** every 100ms, Redis samples 20 random keys with TTL. Any expired → evict. Repeat if > 25% of sample were expired (catch a wave).

**Hot-key with TTL never gets active-sweep evicted** because lazy catches it first. Cold-key with TTL eventually gets caught by active sampling.

**Memory implication:** an expired-but-not-yet-evicted key still consumes RAM until either lazy or active reaches it. **Heavy TTL churn = transient extra RAM.**"

---

### CQ20. "What's keyspace notifications and when would you use them?"

**Opener:** *"Pub/Sub events on key operations (set, expire, eviction). Useful for cache invalidation broadcasts; not for durable workflows."*

"Enable via `notify-keyspace-events Ex` (E = events, x = expired). Then subscribe to `__keyevent@0__:expired` and you get events when keys expire.

**Use cases:**
- Cross-pod cache invalidation broadcasts.
- Session-timeout side effects.
- Trigger cleanup when a temp key expires.

**Caveats:**
- Pub/Sub semantics → if subscriber is down, event is lost.
- Performance impact on heavy-write workloads.
- **Don't use for durable workflows** — Kafka/Streams better."

---

### CQ21. "How would you build a leaderboard for 100M users?"

**Opener:** *"Sorted set with ZADD; shard by user-id-prefix if needed; ZREVRANGE for top-N; Lua for atomic score updates with conditional logic."*

"Single sorted set scales to ~100M entries (~30GB on a beefy instance). Operations:

```
ZADD leaderboard 1500 "user:42"          -- O(log N)
ZREVRANGE leaderboard 0 9 WITHSCORES     -- top 10, O(log N + count)
ZREVRANK leaderboard "user:42"           -- O(log N)
ZINCRBY leaderboard 100 "user:42"        -- atomic score increment
```

**For 1B users:** shard by user_id hash prefix into 10 sorted sets. Top-N is a merge across shards (read all, merge-sort by score). Acceptable for top-1000-ish queries; expensive for arbitrary rank lookup.

**Alternative for very large scale:** **Redis Cluster** with sorted set per slot, app-side aggregation."

---

### CQ22. "How would you build a token-bucket rate limiter atomically?"

**Opener:** *"Lua script holding the entire CAS — read tokens, compute refill, decrement if available, return verdict."*

(See file 19 Section 3.4 for the full Lua script.)

---

### CQ23. "How does Redis cluster handle resharding?"

**Opener:** *"Online slot migration. Source serves the slot until handoff; clients see ASK redirects mid-migration."*

"`CLUSTER SETSLOT slot MIGRATING target-node` on source + `IMPORTING source-node` on target. While migrating:

- Read on source for an existing key in that slot → served.
- Read on source for a key that doesn't exist locally → **`ASK` redirect** to target.
- Once `MIGRATE` moves the key, source has gap; target has key.

When all keys moved: `CLUSTER SETSLOT slot NODE target-node` finalizes. Source no longer serves; target announces ownership via gossip; clients update their slot map.

**Migration is online but slow** — `MIGRATE` is per-key. Tools like `redis-cli --cluster reshard` parallelize."

---

### CQ24. "What's MOVED vs ASK redirect?"

**Opener:** *"MOVED is permanent (slot is on different node); ASK is one-time (slot is migrating)."*

"`MOVED 12345 10.0.0.5:6379` → 'slot 12345 is owned by 10.0.0.5; update your map permanently.'

`ASK 12345 10.0.0.5:6379` → 'slot 12345 is migrating; for THIS request, ask the new node; my map is still correct for other keys.'

**Client behavior:** smart clients (Jedis/Lettuce/Redisson) handle both transparently. Update map on MOVED; one-time redirect with ASKING prefix on ASK."

---

### CQ25. "Why doesn't Redis support cross-slot multi-key ops in cluster mode?"

**Opener:** *"Atomicity guarantees would require distributed transactions across nodes. Use hash tags to colocate or split the workload."*

"In standalone Redis, MGET/MULTI/Lua are atomic because there's one thread. In cluster, keys live on different nodes — atomic across nodes requires 2PC or similar, which Redis explicitly doesn't do.

**Workarounds:**
- **Hash tags:** `{userId}:profile`, `{userId}:settings` → both hash on `userId` → same slot.
- **App-side aggregation:** read N keys from N nodes; combine; accept non-atomic.
- **Don't use cluster** if you need cross-key atomicity at scale; use Sentinel + bigger node.

**The principle:** cluster mode trades cross-key atomicity for horizontal scale. Most workloads don't need cross-key atomicity if you design the key naming well."

---

## SECTION 14 — RAPID-FIRE Q BANK (30 one-liners)

1. **"Default Redis port?"** → 6379.
2. **"Is `INCR` atomic?"** → Yes. Single-threaded core makes it free.
3. **"What's `SETEX`?"** → `SET key value` + `EXPIRE key seconds` in one atomic command.
4. **"What's `SETNX`?"** → SET if not exists. The primitive behind locks. Use `SET key val NX PX ms` for atomic NX + expiry.
5. **"Default max client connections?"** → 10000.
6. **"What's `OBJECT ENCODING`?"** → Reports the internal encoding (ziplist, listpack, hashtable, etc.).
7. **"What's `INFO`?"** → Massive metrics dump; basis for monitoring.
8. **"What's `SLOWLOG`?"** → Records commands that exceeded `slowlog-log-slower-than` (microseconds).
9. **"What's `MONITOR`?"** → Streams every command in real time. Debugging only — kills throughput.
10. **"What's `CLIENT LIST`?"** → Lists all connected clients with stats.
11. **"What's `CLIENT KILL`?"** → Kill a client connection by address or ID.
12. **"What's `CONFIG SET`?"** → Change config at runtime without restart. Useful for `maxmemory-policy` flips.
13. **"What's `WAIT`?"** → Block until N replicas have ack'd current writes. Synchronous replication primitive.
14. **"`UNLINK` vs `DEL`?"** → UNLINK is async (background free); DEL is sync. Always UNLINK for big keys.
15. **"`MGET` vs `GET`?"** → MGET is pipelined N gets in one round-trip.
16. **"`HMSET` vs `HSET`?"** → Same (HMSET deprecated in 4.0; HSET takes multiple field-value pairs).
17. **"`ZINCRBY` use case?"** → Atomic score increment for leaderboards.
18. **"`SUBSCRIBE` blocking?"** → Yes; that connection becomes subscribe-only.
19. **"`PSUBSCRIBE`?"** → Pattern subscribe; e.g. `chan.*`.
20. **"`EXPIRE` on a non-existent key?"** → Returns 0; no-op.
21. **"`PERSIST`?"** → Remove TTL from a key; make it permanent.
22. **"`TTL`?"** → Seconds remaining; -1 if no TTL; -2 if key doesn't exist.
23. **"Max value size?"** → 512MB per key.
24. **"Max databases?"** → 16 by default (DB 0-15); changed via `databases` config; **not supported in cluster mode**.
25. **"`SELECT`?"** → Switch DB; deprecated in cluster mode.
26. **"`FLUSHDB` vs `FLUSHALL`?"** → DB-scoped vs all-DBs scoped.
27. **"`DEBUG SLEEP n`?"** → Blocks for n seconds. **Never in prod.**
28. **"`CLUSTER NODES`?"** → Lists all cluster nodes + slot assignments.
29. **"`MEMORY USAGE key`?"** → Bytes used by a specific key.
30. **"`MEMORY DOCTOR`?"** → AI-style health report on memory issues.

---

## SECTION 15 — FINAL CHECKLIST (the night before)

- Memorize **Section 0** (10-second answer) word-for-word.
- Drill **Section 1.2 data structures table** — every interview opens with "what data structures does Redis have?"
- Memorize the **Section 4 eviction policies** — at least know `allkeys-lru` is the cache default and `noeviction` is dangerous.
- Drill **Section 9 (clients comparison + verdict)** — the user's explicit ask. Be able to say "Redisson on top of Lettuce for distributed primitives" in one sentence.
- Pre-load **CQ1, CQ2, CQ3, CQ4, CQ5, CQ9, CQ10, CQ13** — most common interview questions.
- Cross-reference **file 18** (locking) for Redisson `RLock` deep dive.
- Cross-reference **file 17** (SQL vs NoSQL) for "when Redis vs MySQL" framing.
- Cross-reference **file 19** (payment gateway) for production usage patterns.
- Close this file. Sleep.

> **The senior-IC marker:** when asked "tell me about Redis," **don't start with the data structures** — start with the **mental model** ("in-memory KV store, single-threaded core, rich data structures, optional persistence, never source of truth for money"). Then drill into specifics as the interviewer probes. The mental model is the senior signal; the trivia is the junior signal.
