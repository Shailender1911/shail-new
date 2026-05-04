# File 5: System Design Pack

> Managerial round WILL have a system design Q. Pick one of these 5 to lead with based on the prompt.
> Format per design: **Clarifying Qs → Functional/Non-functional reqs → Capacity → API → Data model → High-level arch → Deep dives → Trade-offs → Failure modes → Bottlenecks → How you'd scale 10x.**
> Speak the framework out loud; that's what they're scoring.

---

## DESIGN FRAMEWORK — 7-STEP TEMPLATE (memorize this script)

1. **Clarify** — scope, scale, constraints (always ask: read:write ratio, latency budget, consistency requirements, single-region or multi).
2. **Functional reqs** — 3-5 must-haves; **Non-functional** — latency, availability, consistency, durability.
3. **Capacity estimation** — QPS, storage/year, bandwidth. Show the math.
4. **API design** — 3-5 endpoints with method + path + key params.
5. **Data model** — primary tables/entities + key indexes + partition strategy.
6. **High-level architecture** — components + flow (2-3 sentences each).
7. **Deep dives + trade-offs** — pick 2-3 areas they'll probe (caching, sharding, consistency, hot-path) and own the trade-off.

**Closing line for any design:** "Bottlenecks I'd watch first: [DB writes / cache stampede / X service]. Scaling 10x: [partition by Y / add Z layer]. Trade-offs I made: [chose A over B because Z]."

---

## DESIGN 1 — URL SHORTENER (bit.ly)

### Clarifying Qs (ask these out loud)
- "What's the read:write ratio?" → typically 100:1, reads dominate.
- "Custom aliases or only auto?" → assume auto-generated for v1, custom is v2.
- "Expiry?" → optional TTL per URL.
- "Analytics?" → click counts, geographic, referrer — async.
- "Scale?" → assume 100M URLs/year, 100B reads/year.

### Reqs
- **Functional:** Shorten URL → return short code; resolve short → original URL; analytics on clicks.
- **Non-functional:** p99 redirect latency <50 ms; 99.99% availability on read; eventual consistency on analytics; durable on write.

### Capacity
- 100M new URLs/year → ~3.2/sec sustained, ~30/sec peak.
- 100B reads/year → ~3,170/sec sustained, ~30K/sec peak.
- Storage: 100M × ~500 bytes (URL + metadata) ≈ 50 GB/year.
- Cache hit rate target ~90% on hot URLs.

### API
- `POST /v1/urls` body `{longUrl, customAlias?, ttl?}` → `{shortCode, shortUrl, expiresAt}`.
- `GET /{shortCode}` → 301 redirect to long URL.
- `GET /v1/urls/{shortCode}/analytics` → click count, top referrers.

### Data model
- `urls` table: `short_code` (PK, varchar(8)), `long_url` (text), `created_at`, `expires_at`, `user_id`, `is_active`.
- `clicks` table (analytics): `short_code`, `clicked_at`, `referrer`, `country`. Partitioned by `clicked_at` (monthly).
- Index on `short_code` (PK), `(user_id, created_at)` for "my URLs".

### High-level architecture
- **Write path:** Client → Gateway → URL Service → Code Generator → DB write → Cache write (write-through).
- **Read path:** Client → Gateway → URL Service → Cache (Redis); on miss → DB → Cache; return 301.
- **Analytics:** Read service emits async `click` event → Kafka → Click Processor → analytics DB (separate from URL DB).
- **CDN:** edge caching of 301 responses for top URLs.

### Code generation strategy (interview gold)
- **Option A — Hash (MD5/SHA) + base62 truncate:** simple but collision-prone at scale.
- **Option B — Counter + base62 encode:** monotonic id → base62 (6-7 chars covers 56B+ URLs). Single counter is bottleneck.
- **Option C — Pre-generated key range per server:** counter service hands out ranges of 1000 ids; servers consume locally; survives counter outage briefly.
- **My pick:** Option C. Trade-off: ids aren't strictly monotonic across all servers (acceptable).

### Deep dives
- **Caching:** Cache-aside on Redis with TTL = URL TTL (or 24h default). Hot keys (top 1%) get longer TTL. Negative caching (404s for nonexistent codes, 5-min TTL) prevents cache penetration attacks.
- **Sharding:** When DB outgrows single node, shard `urls` table by `short_code` hash. Click table sharded by `short_code` so analytics joins are local.
- **Hot URL problem:** A viral URL gets 1M reads/sec. CDN edge cache absorbs most; cache replicas + read replicas; if Redis hot-key, use client-side cache (small LRU) on app server.

### Trade-offs
- **Eventually consistent analytics** vs strong: chose eventual — analytics doesn't need real-time, async via Kafka.
- **301 vs 302:** 301 is permanent → browsers + CDNs cache aggressively → fewer hits to our service. Trade-off: harder to revoke a URL (CDN keeps cached 301). Mitigation: short CDN TTL.
- **Custom alias collision:** Check + reserve atomically (Redis SETNX or DB unique constraint with retry).

### Failure modes
- **DB down on write:** Reject new URLs (read still works from cache + read replicas).
- **Cache down on read:** Fall through to DB; latency spikes; circuit-break if DB also overloads.
- **Counter service down (Option C):** Servers have 1000-id buffer; survive minutes; alert ops.

### Bottlenecks + 10x scale
- **First bottleneck:** Redis read throughput (one shard) → shard Redis by hash(short_code).
- **Next:** DB write throughput → write batching + counter ranges per server.
- **Next:** Analytics ingestion → multiple Kafka partitions + Click Processor scale-out.

---

## DESIGN 2 — RATE LIMITER

### Clarifying Qs
- "Per-IP, per-user, per-API key?" → mix; assume per-API-key + per-endpoint.
- "Limits?" → e.g. 100 req/min per key per endpoint.
- "Distributed across N servers?" → yes, must be globally consistent.
- "Fairness — burst allowed?" → yes, token bucket.
- "On limit hit: reject or queue?" → reject with 429 + Retry-After header.

### Reqs
- **Functional:** Reject requests over limit; expose remaining quota in headers.
- **Non-functional:** Limiter overhead <5ms p99; available even under attack; fail-open for non-critical, fail-closed for money flows.

### Algorithm choice (interview gold)
- **Fixed window** — simplest; spike at window boundary (worst case 2x limit in adjacent windows).
- **Sliding window log** — exact; stores every request timestamp; memory-heavy.
- **Sliding window counter** — interpolation between two adjacent windows; good approximation.
- **Token bucket** — supports burst; conceptually clean; widely used.
- **Leaky bucket** — smooths output; harder to implement distributed.

**My pick:** Token bucket. Bursty traffic (login flurries, retries) handled gracefully; refill rate models steady-state limit.

### High-level architecture
- **In-memory token bucket per key** on each server is fast but doesn't share state across pods.
- **Distributed:** Redis-backed token bucket. Atomic Lua script: read tokens, refill based on time elapsed, check + decrement. Single round-trip, atomic.
- **Caching the limit config:** Limits per (apiKey, endpoint) cached locally with short TTL; admin updates invalidate via pub/sub.

### Lua script (Redis token bucket)
```lua
-- KEYS[1] = bucket key
-- ARGV[1] = max tokens, ARGV[2] = refill rate (tokens/sec), ARGV[3] = now (epoch ms), ARGV[4] = requested tokens

local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or tonumber(ARGV[1])
local last = tonumber(bucket[2]) or tonumber(ARGV[3])
local elapsed = (tonumber(ARGV[3]) - last) / 1000.0
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[2]))

if tokens < tonumber(ARGV[4]) then
  redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', ARGV[3])
  redis.call('EXPIRE', KEYS[1], 3600)
  return 0
end

tokens = tokens - tonumber(ARGV[4])
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', ARGV[3])
redis.call('EXPIRE', KEYS[1], 3600)
return 1
```

### Deep dives
- **Hot key:** A single API key gets DDoS'd. Redis single key bottlenecks. Mitigation: key-level sharding (`limit:{key}:{endpoint}:{shard0..N}`); aggregate counts across shards. Trade-off: slight over-allowance.
- **Two-tier limiting:** Local in-memory bucket per pod (fast, approximate) + Redis for global truth (slower, exact). Local rejects 95% of overages locally; only spillover hits Redis. Drastically reduces Redis QPS.
- **Distributed clock skew:** Use Redis server time (`TIME` command) in the script, not client time. Eliminates clock drift.

### Trade-offs
- **Strict accuracy vs throughput:** Two-tier is approximate (allows slight over) but scales 100x better. For non-money APIs, accept the approximation.
- **Fail-open vs fail-closed on Redis outage:** Fail-open for non-critical (limiter outage shouldn't break the API); fail-closed for money APIs (better to reject than over-allow).

### Failure modes
- **Redis cluster failover (~5s):** Local in-memory bucket carries the rate during outage; over-allowance acceptable for that window.
- **Lua script bug:** Wraps every change in tests; deploy via Redis script SHA so old + new can coexist briefly.

### 10x scale
- Redis cluster + key-level sharding for hot APIs.
- Move heavy clients to dedicated rate limiter shards.
- Push to edge (gateway-side) for IP-based limits to avoid hitting backend at all.

---

## DESIGN 3 — NOTIFICATION SERVICE (Email/SMS/Push)

### Clarifying Qs
- "Synchronous or fire-and-forget?" → fire-and-forget; user shouldn't wait.
- "Channels?" → email, SMS, push (mobile + web), in-app.
- "Templating?" → yes, parameterized templates per channel.
- "Delivery guarantee?" → at-least-once; idempotent on `notificationId`.
- "Scale?" → 10M notifications/day; 1B retained for 90 days for audit.
- "Priorities?" → yes — transactional (OTP, payment confirm) vs marketing.

### Reqs
- **Functional:** Send notification via specified channel(s); template rendering; delivery tracking; retries on failure; deduplication.
- **Non-functional:** Transactional p99 <2s end-to-end; marketing best-effort; durable; auditable.

### API
- `POST /v1/notifications` body `{userId, channels[], templateId, params{}, priority, idempotencyKey}` → `{notificationId, status}`.
- `GET /v1/notifications/{id}` → status, attempts, last error.
- Webhook from carriers (Twilio, FCM, SES) → `POST /v1/notifications/{id}/delivery-receipt`.

### Data model
- `notifications`: `id` (PK), `user_id`, `template_id`, `params` (JSON), `priority`, `idempotency_key` (unique), `created_at`, `status`.
- `notification_attempts`: `notification_id`, `channel`, `provider`, `attempt_no`, `status`, `error`, `attempted_at`. Partitioned by `attempted_at`.
- `templates`: `id`, `channel`, `subject`, `body`, `variables[]`, `version`, `active`.
- `user_preferences`: `user_id`, `channel`, `enabled`, `frequency_cap`.

### High-level architecture
- **Ingest:** API → validate + dedup (idempotency key) → enqueue per priority.
- **Two queues:** `transactional-queue` (Kafka, low-latency consumers, more partitions); `marketing-queue` (lower priority, batched).
- **Channel workers:** Email Worker (SES/SendGrid), SMS Worker (Twilio/AWS SNS), Push Worker (FCM/APNS), In-App Worker (writes to user inbox + WebSocket push).
- **Template service:** Renders template + params; cached.
- **Preference service:** Checks user opt-out and frequency caps.
- **Receipt collector:** Carrier webhooks → update `notification_attempts` → trigger retries on failure.

### Deep dives
- **Priority isolation:** Transactional and marketing on separate Kafka topics with separate consumer groups. Marketing pause shouldn't affect OTP delivery.
- **Provider fallback:** Primary email = SES; on quota or error rate spike, fallback to SendGrid. Per-provider circuit breaker. State stored per-provider in Redis.
- **Frequency capping:** Sliding window counter per user per template (Redis); reject if cap exceeded; record as "suppressed" for analytics.
- **Idempotency:** `idempotencyKey` unique constraint on `notifications` table; second submission returns existing notification id.
- **Personalization at scale:** Templates are precompiled (Mustache/Pebble); param injection only — never raw concatenation (XSS via email is real).
- **Order of operations for OTP (transactional, time-sensitive):** Sync send (fire async, but tight p99 SLO); SMS first (fastest channel); fallback to voice/email after T seconds.

### Trade-offs
- **At-least-once + idempotent receiver vs exactly-once:** At-least-once is realistic; carriers + retries cause duplicates; client (or template) should be idempotent on view.
- **In-process render vs separate template service:** Separate service for centralized template versioning, A/B, multi-language. Trade-off: extra hop.
- **Push token freshness:** Tokens expire; on FCM "invalid token" response, immediately invalidate in DB and skip future sends to that token.

### Failure modes
- **SES throttle spike:** Provider error → circuit break SES; fallback to SendGrid; alert ops.
- **Carrier webhook missed:** After T minutes without receipt, query carrier API for status (poll fallback).
- **Kafka outage:** Producer buffers locally; if buffer fills, fail-fast on transactional, drop on marketing.

### 10x scale
- Partition Kafka by `userId` for ordering per user; shard channel workers per partition.
- Pre-render template + cache fully-formed payload for marketing campaigns (millions to same template).
- Region-local SMS providers for international scale.

---

## DESIGN 4 — IDEMPOTENT PAYMENT API (your wheelhouse)

### Clarifying Qs
- "Charging customer or moving money internally?" → both; let's do customer charge to merchant.
- "Idempotency contract?" → client-supplied `Idempotency-Key`; server stores result for 24h+.
- "PCI-DSS scope?" → assume tokenized cards (we don't store PAN).
- "Payment providers?" → 2+ (failover + per-merchant routing).
- "Latency?" → user-blocking, p99 <3s; auth + capture in single call.

### Reqs
- **Functional:** Create payment; refund; webhooks for async events; status query; reconcile.
- **Non-functional:** No double-charge under any retry scenario; durable audit; consistent across provider + our DB.

### API
- `POST /v1/payments` headers `Idempotency-Key: <uuid>` body `{amount, currency, customerToken, merchantId, captureMode}` → `{paymentId, status}`.
- `POST /v1/payments/{id}/refund` body `{amount}` headers `Idempotency-Key`.
- `GET /v1/payments/{id}` → status + history.
- `POST /v1/payments/webhook` (from provider).

### Data model
- `payments`: `id` (PK), `idempotency_key` (unique with `merchant_id`), `merchant_id`, `amount`, `currency`, `status`, `provider`, `provider_txn_id`, `created_at`, `updated_at`.
- `payment_attempts`: `payment_id`, `attempt_no`, `provider`, `status`, `provider_response` (JSON), `attempted_at`.
- `payment_events`: `payment_id`, `event_type`, `event_data`, `created_at` (outbox table).
- Index: `(merchant_id, idempotency_key)` unique; `(provider_txn_id, provider)` unique; `(merchant_id, created_at)` for queries.

### High-level architecture
- **Write path:** API → Idempotency Check (Redis + DB) → Persist `payments` (status `INITIATED`) → Provider Adapter → Update status → Persist `payment_events` (outbox) → Return.
- **Outbox processor:** Reads `payment_events` → publishes to Kafka → downstream consumers (analytics, settlement, ledger).
- **Webhook ingestion:** Provider → signature validate → ack 200 → persist event → async update payment status.
- **Reconciliation cron:** Reads provider EOD report → diffs against our DB → flags discrepancies → ops alerts.

### The "no double charge" deep dive (the heart of the question)
1. **Client supplies `Idempotency-Key`** (UUID). Required header.
2. **Server checks Redis** for `idem:{merchant}:{key}` — if hit, return cached response (the second-call short-circuit).
3. **If miss, atomically insert** into `payments` with `(merchant_id, idempotency_key)` unique constraint. If unique violation → another concurrent request won; loop back, read existing row, return.
4. **State machine:** `INITIATED` → `PROVIDER_CALLED` → `AUTHORIZED`/`DECLINED` → `CAPTURED`/`VOIDED`. Each transition is a DB update with `WHERE current_status = expected`.
5. **Provider call wrapped in retry only if response is "definitely failed"** (network timeout BEFORE provider received it). If timeout AFTER provider received it (no response but provider may have charged) → mark `STATUS_UNKNOWN`, do NOT retry, surface to recon.
6. **Provider's own idempotency** (Stripe `Idempotency-Key`, Razorpay equivalent) — pass our key through. Belt and braces.
7. **Webhook arrives later** → match on `provider_txn_id` (also unique) → update status; if conflicting status (e.g. provider says CAPTURED but we marked VOIDED), alert + manual recon.
8. **Refund is separately idempotent** with its own key.

### Trade-offs (own these)
- **Sync provider call vs async:** Sync for user UX (2-3s latency budget). Trade-off: failure modes are uglier (timeout = unknown state). Mitigation: STATUS_UNKNOWN + recon.
- **Outbox pattern over direct Kafka publish:** Single DB transaction for `payments` + `payment_events` row; outbox processor publishes. Survives crash mid-send.
- **`@Transactional` boundary:** Around DB write only — provider HTTP call is NOT in the transaction (long external call inside tx is a deadlock waiting to happen).

### Failure modes
- **Provider responds 200 but we crash before persisting:** Provider state ≠ our state. Webhook + recon catch it.
- **Provider call times out:** STATUS_UNKNOWN state — recon job queries provider next morning to resolve.
- **Redis idempotency cache down:** Fall through to DB unique constraint — slower but correct.
- **Webhook retried by provider after we've already updated:** dedup on `(provider, event_id)` — second arrival is no-op.

### 10x scale
- Shard `payments` by `merchant_id` (most queries are per-merchant).
- Provider routing: per-merchant primary + fallback; circuit breaker per provider.
- Async capture for non-time-sensitive flows (auth now, capture in batch overnight).

---

## DESIGN 5 — DISTRIBUTED CACHE (Redis-like)

### Clarifying Qs
- "Cache or KV store?" → cache (TTL'd, lossy).
- "Eviction?" → LRU.
- "Consistency?" → eventual; cache is best-effort.
- "Persistence?" → optional snapshots; not the primary use.
- "Scale?" → 10TB working set; 1M ops/sec.

### Reqs
- **Functional:** GET, SET (with TTL), DELETE, expire.
- **Non-functional:** p99 <1ms; horizontal scale to TB; survive single-node failure with bounded data loss.

### Architecture
- **Sharding:** Consistent hashing across N shards (N = 100 to start, virtual nodes for balance). Client hashes key → routes to owner shard.
- **Replication:** Each shard has 1 primary + 2 replicas. Async replication.
- **Failure:** Heartbeat from coordinator (Zookeeper/etcd); on primary fail, promote replica.
- **Eviction:** Per-shard LRU; approximate (sample-based) for performance.
- **Persistence (optional):** AOF (append-only file) + periodic RDB snapshots.

### Deep dives
- **Hot key problem:** One key receives 10x normal QPS; that shard saturates. Mitigation: client-side caching (small local LRU per app pod) absorbs reads; or replicate hot keys to all shards; or split with a salted suffix (`key:{0..N}`).
- **Cache stampede on cold-cache:** Soft-TTL + early refresh; singleflight (Redisson `RLock`-equivalent) so only one process rebuilds.
- **Consistent hashing rebalance:** Virtual nodes (~150 per physical) → adding a shard moves only ~1/N of keys, not 50%.
- **Network partition:** AP-leaning — accept stale reads during partition; client reconnects to whoever's reachable. CP-leaning systems (etcd) lose availability instead.

### Trade-offs
- **Async replication:** Faster writes; replica lag → potential data loss on primary crash. Acceptable for cache.
- **In-memory only:** Speed → cost (RAM is expensive); bounded size via LRU.
- **Single-threaded per shard (Redis-style):** Simpler concurrency model; one slow query blocks others. Mitigation: keep ops O(1); reject `KEYS *`.

### Failure modes
- **Shard primary dies mid-write:** Last few writes lost (async replication). Acceptable for cache.
- **Coordinator network partition:** Split-brain risk → use majority quorum (3+ nodes).
- **Memory pressure:** LRU + eviction warnings + alert when hit rate drops.

### 10x scale
- Add shards (consistent hashing absorbs).
- Tier (hot in RAM, warm on SSD).
- Per-region clusters with cross-region async replication for disaster recovery.

---

## DESIGN MENU — WHICH ONE FOR WHICH PROMPT

| If they ask… | Lead with… |
|---|---|
| "Design a URL shortener" | Design 1 |
| "Design rate limiting / API gateway protection" | Design 2 |
| "Design a notification / messaging system" | Design 3 |
| "Design a payment system / order system / how would you avoid double charging" | Design 4 (your wheelhouse — own it) |
| "Design a caching layer / Redis from scratch" | Design 5 |
| "Design a feature flag system" | Pivot to your Pariksha experience — "I built one. Here's the architecture." |
| "Design a multi-tenant config system" | Pivot to ConfigNexus |
| "Design a chatbot / AI assistant for X" | Pivot to Eklavya |

---

## CHEAT-SHEET CAPACITY ESTIMATIONS (memorize)

| Quantity | Approx |
|---|---|
| Day | 86,400 sec ≈ **100K sec** |
| Month | 2.5M sec |
| Year | 31M sec ≈ **30M sec** |
| 1M req/day | ~12 req/sec |
| 1B req/day | ~12K req/sec |
| 1KB × 1M = 1GB | |
| 1KB × 1B = 1TB | |

**Read:write ratios:** Social feed ~100:1, payments ~5:1, URL shortener ~100:1, search ~1000:1.

**Latency budgets:**
- L1 cache: 1 ns
- L2 cache: 10 ns
- RAM: 100 ns
- SSD random read: 100 µs
- HDD random read: 10 ms
- Same DC network round trip: 0.5 ms
- Cross-region round trip: 50-150 ms

---

## SCRIPT — START AND END A SYSTEM DESIGN ANSWER

**Opening:** "Before I jump in, can I clarify three things — scale, consistency requirements, and read:write ratio?" *Pause for answers.*

**Then:** "OK, so we're solving for X at Y scale with Z consistency. I'll walk through reqs, capacity, API, data model, architecture, then dive into the parts I think matter most — feel free to redirect me anytime."

**Closing every design:** "Bottlenecks I'd watch first: [DB writes / cache stampede / hot key]. To 10x: [partition by Y / add layer Z]. Trade-offs I made: [chose A over B because Z]. Anywhere you want me to go deeper?"

That last sentence — "anywhere you want me to go deeper" — invites them to drive the conversation, which is the senior signal.
