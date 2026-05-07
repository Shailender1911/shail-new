# File 14: Airtel Digital — Top 5 System Designs (HLD + LLD)

> Five designs that map directly to Airtel Digital's product surface.
> Each has full HLD (clarifying Qs, capacity, API, data, arch, deep dives) + LLD (class design, sequence, schemas, algorithms).
> Read in the order they appear — that's also the priority order for your interview.

## Table of Contents

1. **[Airtel Money / UPI Wallet Platform](#1-airtel-money--upi-wallet-platform)** — your wheelhouse, lead with this
2. **[Airtel Xstream — OTT Streaming Platform](#2-airtel-xstream--ott-streaming-platform)** — CDN, video pipeline, scale
3. **[Telco-Scale Notification Service](#3-telco-scale-notification-service)** — OTP + bill + marketing at 400M subs
4. **[AI Customer Support Assistant](#4-ai-customer-support-assistant)** — pivot to your Eklavya
5. **[Mobile Recharge / Plan Management Platform](#5-mobile-recharge--plan-management-platform)** — Airtel core

---

## SHARED FRAMEWORK — START EVERY DESIGN WITH THIS

```
1. Clarify           → 5-7 questions; understand scale, consistency, scope
2. Requirements      → Functional (3-5) + Non-functional (latency, availability, durability)
3. Capacity          → QPS, storage/yr, bandwidth — show the math
4. API design        → 4-6 endpoints with method/path/key params
5. Data model        → Tables + indexes + partition strategy
6. HLD               → Components + dataflow (write path, read path)
7. Deep dives        → 2-3 areas they probe (hot key, consistency, failure)
8. LLD               → Key classes/interfaces + sequence + algorithms
9. Trade-offs        → Own them
10. Failure modes    → What breaks, how it recovers
11. 10x scale        → First bottleneck + mitigation
```

**Closing line:** *"Bottlenecks I'd watch first: X. To 10x: Y. Trade-offs I made: chose A over B because Z. Anywhere you want me to go deeper?"*

---

# 1. AIRTEL MONEY / UPI WALLET PLATFORM

> Your strongest design — direct fit with PayU + GPay + DLS NACH. Lead with this if they ask payments.

## 1.1 Clarifying Questions (ask out loud)

1. "Wallet only, or wallet + UPI + bill pay + recharge in one platform?" → Assume wallet + UPI for v1.
2. "Read:write ratio?" → Heavy reads (balance checks); ~5:1 read:write on hot wallet.
3. "Latency target for a payment?" → p99 < 2s end-to-end; balance check < 100ms.
4. "Consistency model — strong everywhere or eventual on reads?" → Strong on balance + tx; eventual on analytics/dashboards.
5. "Scale?" → 100M wallet users; 50M MAU; peak 10K txns/sec at festival.
6. "Single region or multi?" → Multi-region for DR; active-passive on writes; active-active on reads.
7. "Compliance?" → RBI Payments Bank licence; PCI-DSS scoped to card flows; full audit trail.

## 1.2 Requirements

### Functional
- Add money to wallet (UPI / Card / NetBanking).
- Send money (UPI / wallet-to-wallet).
- Pay merchant (UPI QR).
- Withdraw to bank.
- Transaction history + statements.

### Non-functional
- **Latency:** p99 payment 2s, balance check 100ms.
- **Availability:** 99.99% on payment; 99.9% on history.
- **Consistency:** Strong on balance + posted tx; eventual on analytics.
- **Durability:** Zero loss on confirmed tx.
- **Idempotent:** Every payment idempotent on `Idempotency-Key`.

## 1.3 Capacity Estimation

```
Users         : 100M
DAU           : 30M
Txn/user/day  : ~5
Daily txns    : 150M
Avg QPS       : 150M / 86400 ≈ 1,700 TPS
Peak QPS      : 5x avg ≈ 8,500 TPS (festival/payday)
Storage/yr    : 150M × 365 × 1KB ≈ 55 TB
Bandwidth     : 8500 × 5KB ≈ 40 MB/s ingress
```

## 1.4 API Design

```http
POST /v1/wallets                     # Provision wallet for user
GET  /v1/wallets/{userId}/balance    # Cached, sub-100ms
POST /v1/payments
  Headers: Idempotency-Key: <uuid>
  Body  : {sourceType, source, destType, dest, amount, purpose}
  Resp  : {paymentId, status, balance}
GET  /v1/payments/{paymentId}
GET  /v1/payments?userId=&from=&to=&page=
POST /v1/payments/{paymentId}/refund
POST /v1/webhooks/upi                # NPCI/PSP callbacks
```

## 1.5 Data Model

```sql
-- wallets (per-user; sharded by user_id)
CREATE TABLE wallets (
  user_id        BIGINT PRIMARY KEY,
  balance        DECIMAL(15,2) NOT NULL DEFAULT 0,
  status         ENUM('ACTIVE','FROZEN','CLOSED'),
  kyc_level      ENUM('MIN','FULL','VKYC'),
  daily_limit    DECIMAL(15,2),
  monthly_limit  DECIMAL(15,2),
  version        BIGINT NOT NULL DEFAULT 0,    -- optimistic lock
  updated_at     TIMESTAMP
);

-- payments (sharded by user_id)
CREATE TABLE payments (
  payment_id        BIGINT PRIMARY KEY,
  idempotency_key   VARCHAR(64),
  user_id           BIGINT,
  source_type       ENUM('WALLET','UPI','CARD','NB'),
  dest_type         ENUM('WALLET','UPI','MERCHANT','BANK'),
  amount            DECIMAL(15,2),
  status            ENUM('INITIATED','AUTHORIZED','CAPTURED','FAILED','REVERSED'),
  provider_txn_id   VARCHAR(64),
  created_at        TIMESTAMP,
  UNIQUE KEY uk_idem (user_id, idempotency_key),
  UNIQUE KEY uk_provider (provider_txn_id)
);

-- ledger (immutable, append-only; double-entry)
CREATE TABLE ledger (
  ledger_id     BIGINT PRIMARY KEY,
  payment_id    BIGINT,
  account_id    BIGINT,         -- user wallet id OR system account
  direction     ENUM('CREDIT','DEBIT'),
  amount        DECIMAL(15,2),
  balance_after DECIMAL(15,2),
  created_at    TIMESTAMP,
  KEY idx_account_time (account_id, created_at)
);

-- payment_events (outbox pattern)
CREATE TABLE payment_events (
  event_id      BIGINT PRIMARY KEY,
  payment_id    BIGINT,
  event_type    VARCHAR(32),
  payload       JSON,
  published     BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMP,
  KEY idx_unpub (published, created_at)
);
```

**Sharding:** `wallets`, `payments`, `ledger` all sharded by `user_id`. All-related-data colocated → no cross-shard tx for normal payments.

## 1.6 High-Level Design

```
            ┌──────────────────────────────────────────────────────────────────┐
            │                      Mobile / Web Clients                        │
            └──────────────┬─────────────────────────────────┬─────────────────┘
                           │                                 │
                  ┌────────▼─────────┐              ┌────────▼─────────┐
                  │   API Gateway    │              │   API Gateway    │
                  │ (TLS, Auth, RL)  │              │ (TLS, Auth, RL)  │
                  └────────┬─────────┘              └────────┬─────────┘
                           │                                 │
              ┌────────────▼────────────┐         ┌──────────▼────────┐
              │   Payment Service       │         │  Wallet Read       │
              │   (write path, async    │         │  Service           │
              │    coord, state mgmt)   │         │  (cached balance)  │
              └────┬─────────────┬──────┘         └────────┬───────────┘
                   │             │                         │
         ┌─────────▼──┐    ┌─────▼──────┐         ┌────────▼─────┐
         │ Idempot.    │    │ Provider   │         │ Redis Cluster│
         │ Cache       │    │ Adapters   │         │ (balance, idem)│
         │ (Redis)     │    │ NPCI, Banks│         └──────┬───────┘
         └─────────────┘    └─────┬──────┘                │
                                  │                       │
                            ┌─────▼─────────────┐         │
                            │ Wallet DB         │         │
                            │ (sharded by user) │◄────────┘
                            └─────┬─────────────┘
                                  │
                            ┌─────▼─────────────┐
                            │ Outbox Processor  │
                            │ → Kafka           │
                            └─────┬─────────────┘
                                  │
                ┌─────────────────┼──────────────────┐
                ▼                 ▼                  ▼
         ┌────────────┐    ┌────────────┐     ┌────────────┐
         │ Notification│    │ Analytics  │     │ Reconcil.  │
         │ Service     │    │ Pipeline   │     │ Service    │
         └─────────────┘    └────────────┘     └────────────┘
```

### Write Path (payment flow)
1. Client → Gateway → Payment Service.
2. Idempotency check: Redis `idem:{userId}:{key}`. Hit → return cached. Miss → continue.
3. Begin transaction on user's shard:
   - INSERT `payments` row with unique constraint on `(user_id, idempotency_key)`.
   - For wallet→wallet: SELECT wallets FOR UPDATE both rows (in deterministic order to avoid deadlock).
   - Validate balance + limits.
   - UPDATE balance with optimistic lock: `WHERE version = ?`.
   - INSERT `ledger` debit + credit rows (double entry).
   - INSERT `payment_events` outbox row.
4. Commit.
5. Return `{paymentId, status, newBalance}`.
6. Outbox processor (separate worker) reads unpublished events → Kafka → downstream consumers (notifications, analytics, recon).

### Read Path (balance check)
1. Client → Gateway → Wallet Read Service.
2. Redis cache `balance:{userId}` (TTL 60s with refresh-ahead for hot users).
3. Miss → DB read replica → cache → return.

## 1.7 Deep Dive 1 — Idempotency (the heart)

```
Two-layer protection:

Layer 1 (fast path): Redis SETNX
  KEY:   idem:{userId}:{idempotencyKey}
  VALUE: {paymentId, status, response} (cached after first success)
  TTL:   24h
  → Returns same response without DB hit.

Layer 2 (correctness): DB unique constraint
  UNIQUE (user_id, idempotency_key) on payments
  → On race, second tx gets unique violation; service catches, reads existing row, returns its response.
```

**Why both?** Redis can lag/flush; unique constraint is the source of truth.

## 1.8 Deep Dive 2 — Concurrent Wallet Updates

**Problem:** Two simultaneous outbound payments from the same wallet.

**Option A — Pessimistic lock:** `SELECT ... FOR UPDATE` on wallet row. Simple, correct. Bottleneck: lock holds during external provider call → no, we never hold lock across external calls.

**Option B — Optimistic lock with version:** `UPDATE wallets SET balance=?, version=version+1 WHERE user_id=? AND version=?`. If 0 rows updated → retry from re-read.

**My pick:** Optimistic for normal flow (low contention per user); pessimistic for wallet-to-wallet transfers within same shard (need both rows locked atomically, deterministic order).

## 1.9 Deep Dive 3 — UPI Provider Integration

**External flow (UPI initiation):**
```
User → Wallet Service → NPCI gateway → Bank PSP → Bank → response back
```

**Key constraints:**
- **NEVER hold a DB transaction across the external NPCI call.** Long external calls inside tx = connection pool starvation = system collapse.
- **Pass our idempotency key to NPCI.** They have their own idempotency too — belt and braces.
- **Timeout policy:** sync wait up to 25s; if NPCI returns no response, mark `STATUS_UNKNOWN`, NEVER retry, recon next morning.
- **Webhook fallback:** Final status often arrives via callback even after sync timeout — match on `provider_txn_id` (unique).

## 1.10 Deep Dive 4 — Reconciliation

EOD job downloads NPCI/PSP settlement file → joins against our `payments` + `ledger` → flags:

```
- Provider has txn we don't       → "missing-on-our-side"   → INSERT into our DB with reconciliation flag, manual review
- We have txn provider doesn't   → "orphan-on-our-side"     → query provider; if not found, REVERSE
- Amount mismatch                → manual escalation, hold further txns for that pair
```

Without recon, drift accumulates and money goes missing. **Non-negotiable.**

## 1.11 LLD — Key Classes (Java/Spring)

```java
// Service interface
public interface PaymentService {
    PaymentResponse initiate(PaymentRequest req, String idempotencyKey);
    PaymentResponse get(long paymentId);
    void handleProviderCallback(ProviderCallback cb);
}

// Implementation skeleton
@Service
@RequiredArgsConstructor
public class PaymentServiceImpl implements PaymentService {
    private final IdempotencyManager idempotency;
    private final WalletRepository wallets;
    private final PaymentRepository payments;
    private final LedgerRepository ledger;
    private final OutboxRepository outbox;
    private final ProviderRouter providerRouter;
    private final RateLimiter rateLimiter;
    private final FraudCheckService fraud;

    public PaymentResponse initiate(PaymentRequest req, String idemKey) {
        rateLimiter.check(req.userId(), "payment");
        var cached = idempotency.lookup(req.userId(), idemKey);
        if (cached.isPresent()) return cached.get();

        fraud.preCheck(req);

        // STAGE 1 — book intent (atomic, on user's shard)
        var payment = bookPayment(req, idemKey);

        // STAGE 2 — call external provider (NO DB tx held)
        var providerResp = providerRouter.route(req).execute(payment);

        // STAGE 3 — finalize (atomic again)
        return finalizePayment(payment, providerResp);
    }

    @Transactional
    private Payment bookPayment(PaymentRequest req, String idemKey) {
        try {
            var wallet = wallets.lockForUpdate(req.userId());
            assertEnoughBalance(wallet, req.amount());
            assertWithinLimits(wallet, req.amount());
            var p = payments.insert(Payment.initiated(req, idemKey));
            ledger.insert(LedgerEntry.debit(wallet, req.amount(), p.id()));
            wallet.debit(req.amount());
            wallets.save(wallet);
            outbox.enqueue(PaymentEvents.initiated(p));
            return p;
        } catch (DataIntegrityViolationException dup) {
            return payments.findByIdempotency(req.userId(), idemKey).orElseThrow();
        }
    }

    @Transactional
    private PaymentResponse finalizePayment(Payment p, ProviderResponse resp) {
        if (resp.isSuccess()) {
            payments.updateStatus(p.id(), CAPTURED, resp.providerTxnId());
            outbox.enqueue(PaymentEvents.captured(p));
        } else if (resp.isUnknown()) {
            payments.updateStatus(p.id(), STATUS_UNKNOWN);
            outbox.enqueue(PaymentEvents.unknown(p));   // → recon picks up
        } else {
            // Reverse: credit wallet back
            var wallet = wallets.lockForUpdate(p.userId());
            wallet.credit(p.amount());
            wallets.save(wallet);
            ledger.insert(LedgerEntry.credit(wallet, p.amount(), p.id()));
            payments.updateStatus(p.id(), FAILED);
            outbox.enqueue(PaymentEvents.failed(p));
        }
        return PaymentResponse.from(p);
    }
}
```

### Idempotency Manager

```java
@Component
@RequiredArgsConstructor
public class IdempotencyManager {
    private final RedisTemplate<String, PaymentResponse> redis;
    private final PaymentRepository payments;

    public Optional<PaymentResponse> lookup(long userId, String key) {
        var cached = redis.opsForValue().get(redisKey(userId, key));
        if (cached != null) return Optional.of(cached);

        // Layer 2: DB
        return payments.findByIdempotency(userId, key)
            .map(PaymentResponse::from)
            .map(r -> { cache(userId, key, r); return r; });
    }

    public void cache(long userId, String key, PaymentResponse resp) {
        redis.opsForValue().set(redisKey(userId, key), resp, Duration.ofHours(24));
    }

    private String redisKey(long userId, String key) {
        return "idem:" + userId + ":" + key;
    }
}
```

### Provider Router (Strategy + Factory)

```java
public interface PaymentProviderAdapter {
    ProviderType type();
    ProviderResponse execute(Payment p);
    boolean supports(PaymentRequest req);
}

@Component
@RequiredArgsConstructor
public class ProviderRouter {
    private final List<PaymentProviderAdapter> adapters;
    private final CircuitBreakerRegistry cbRegistry;

    public PaymentProviderAdapter route(PaymentRequest req) {
        // Pick by destination type + circuit-breaker state + per-user merchant routing
        return adapters.stream()
            .filter(a -> a.supports(req))
            .filter(a -> cbRegistry.circuitBreaker(a.type().name()).getState() != OPEN)
            .findFirst()
            .orElseThrow(() -> new NoProviderAvailableException(req));
    }
}

@Component
class NpciUpiAdapter implements PaymentProviderAdapter { ... }
@Component
class CardAdapter      implements PaymentProviderAdapter { ... }
@Component
class WalletInternalAdapter implements PaymentProviderAdapter { ... }
```

### Sequence — Wallet → UPI Payment

```
Client      Gateway      PaymentSvc    Idempotency    DB(shard)    NPCI         Outbox      Kafka
  │            │             │             │             │           │             │           │
  │──POST────▶│             │             │             │           │             │           │
  │            │──forward──▶│             │             │           │             │           │
  │            │             │──lookup───▶│             │           │             │           │
  │            │             │◀──miss─────│             │           │             │           │
  │            │             │──BEGIN TX──────────────▶│             │           │             │           │
  │            │             │  lockForUpdate(wallet)  │             │           │             │           │
  │            │             │  insert payment INITIATED            │           │             │           │
  │            │             │  insert ledger debit                 │           │             │           │
  │            │             │  update wallet balance               │           │             │           │
  │            │             │  insert outbox initiated              │           │             │           │
  │            │             │──COMMIT─────────────────│             │           │             │           │
  │            │             │──route+execute─────────────────────▶│             │           │           │
  │            │             │                                       │ NPCI call │             │           │
  │            │             │◀──response─────────────────────────────│             │           │
  │            │             │──BEGIN TX (status update)──────────▶│             │           │             │
  │            │             │  payments→CAPTURED                   │             │           │             │
  │            │             │  outbox captured                     │             │           │             │
  │            │             │──COMMIT─────────────────│             │           │             │           │
  │            │             │──cache(idem)─────────▶│             │           │             │           │
  │◀──response─│◀────────────│             │             │           │             │           │
  │            │             │             │             │           │             │──poll───▶│
  │            │             │             │             │           │             │──publish▶│
                                                                                                ↓
                                                                                         Notification, Analytics, Recon
```

## 1.12 Trade-Offs (own these)

- **Sync provider call vs async** → sync for UX (2s budget); ugly failure modes (timeouts) handled by `STATUS_UNKNOWN` + recon.
- **Outbox over direct Kafka publish** → survives crash mid-publish; one DB tx vs two systems.
- **Optimistic locking on wallet** → low contention per user; pessimistic only for two-wallet transfers.
- **Strong consistency on balance** → never overdraft; trade-off is throughput per wallet shard, mitigated by user-level sharding.

## 1.13 Failure Modes

| Scenario | Behavior |
|---|---|
| Provider 200 but we crash before persist | Webhook + recon catch it |
| Provider timeout (no response) | `STATUS_UNKNOWN` → recon resolves |
| Redis idem cache down | Falls through to DB unique constraint |
| Wallet shard down | API returns `RETRY_LATER` for that shard's users; others unaffected |
| Outbox processor lag | Notifications delayed; payments still succeed |
| Webhook retry after our update | Match on `provider_txn_id`; dedup → no-op |

## 1.14 Scale to 10x

- **First bottleneck:** hot shard (festival heavy users) → re-shard or move to user-bucketed virtual shards.
- **Read replicas + Redis fan-out** for balance checks (>90% hit rate target).
- **Provider routing** with multi-provider failover (NPCI primary; secondary aggregator on circuit open).
- **Outbox processor** sharded per partition; back-pressure via bounded queue.

---

# 2. AIRTEL XSTREAM — OTT STREAMING PLATFORM

> Most likely after payment. Big architecture canvas — perfect for showing breadth.

## 2.1 Clarifying Questions

1. "Live + VOD or VOD only?" → Both (Xstream has live channels + on-demand).
2. "DRM required?" → Yes — premium content. Widevine + FairPlay + PlayReady.
3. "Adaptive bitrate?" → Yes — HLS / DASH with multiple renditions.
4. "Recommendation in scope?" → Yes — homepage feed.
5. "Scale?" → 50M users; 10M concurrent peak (cricket); 100K concurrent live for big match.
6. "Geographic scope?" → India primary; some international for diaspora.
7. "Latency budget?" → VOD start <2s; live latency 30-60s acceptable (LL-HLS could push to 5-8s).

## 2.2 Requirements

### Functional
- Browse catalog (search, categories, recommendations).
- Stream VOD with adaptive bitrate.
- Stream live channels.
- Resume playback (last-watched position).
- Download for offline (DRM-protected).
- Watch history + ratings.

### Non-functional
- VOD start time p95 < 2s.
- Live glass-to-glass latency < 60s (standard HLS); < 8s with LL-HLS.
- Video availability 99.99%.
- Catalog API p99 < 200ms.

## 2.3 Capacity

```
Users               : 50M
DAU                 : 15M
Avg session         : 45 min
Concurrent peak     : 10M (cricket)
Avg bitrate served  : 2 Mbps (mix of qualities)
Egress at peak      : 10M × 2 Mbps = 20 Tbps  → CDN MUST handle this
Catalog QPS         : 10M browse + 5M searches per day → ~200 QPS avg, 5K peak
```

**Conclusion:** Origin storage trivial; CDN egress is the hard problem.

## 2.4 API Design

```http
GET  /v1/catalog/feed?userId=&page=        # personalized homepage
GET  /v1/catalog/{contentId}                # detail page
GET  /v1/catalog/search?q=
GET  /v1/playback/manifest/{contentId}      # HLS/DASH manifest URL + license URL
GET  /v1/playback/license                   # DRM license issuance
POST /v1/playback/heartbeat                 # every 30s {contentId, position}
GET  /v1/users/{userId}/continue-watching
POST /v1/ratings {contentId, rating}
GET  /v1/live/channels
GET  /v1/live/{channelId}/manifest
```

## 2.5 Data Model

```sql
-- Content catalog (Postgres for joins; Elasticsearch for search)
CREATE TABLE content (
  content_id    BIGINT PRIMARY KEY,
  type          ENUM('MOVIE','EPISODE','LIVE'),
  title         TEXT,
  metadata      JSONB,
  duration_sec  INT,
  release_date  DATE,
  rights_until  TIMESTAMP,
  geo_allowed   TEXT[],     -- country codes
  status        ENUM('DRAFT','PUBLISHED','UNLISTED')
);

CREATE TABLE content_renditions (
  rendition_id  BIGINT PRIMARY KEY,
  content_id    BIGINT,
  bitrate_kbps  INT,
  resolution    TEXT,
  codec         TEXT,
  cdn_path      TEXT,
  drm_key_id    TEXT
);

-- Playback events (Cassandra — time-series, write-heavy)
-- partition: user_id; clustering: started_at DESC
CREATE TABLE playback_events (
  user_id        BIGINT,
  started_at     TIMESTAMP,
  content_id     BIGINT,
  position_sec   INT,
  duration_sec   INT,
  device         TEXT,
  PRIMARY KEY ((user_id), started_at)
) WITH CLUSTERING ORDER BY (started_at DESC);

-- Recommendation cache (Redis) — populated by ML pipeline
-- key: rec:user:{userId}:home  → JSON list of contentIds
-- TTL: 1h
```

## 2.6 High-Level Design

```
                            ┌─────────────────────────┐
                            │   Content Ingest        │
                            │   (Editor/CMS upload)   │
                            └──────────┬──────────────┘
                                       │
                            ┌──────────▼──────────────┐
                            │   Transcoding Pipeline   │
                            │  (Mediaconvert/ffmpeg)   │
                            │  → multiple renditions   │
                            │  → DRM packaging         │
                            └──────────┬──────────────┘
                                       │
                            ┌──────────▼──────────────┐
                            │   Origin Storage (S3)   │
                            │  HLS/DASH segments       │
                            └──────────┬──────────────┘
                                       │
                            ┌──────────▼──────────────┐
                            │   CDN (CloudFront/      │
                            │   Akamai/Airtel CDN)    │
                            │   edge caching          │
                            └──────────┬──────────────┘
                                       │
        ┌─────────────────────┬────────┴────────┬─────────────────┐
        │                     │                 │                 │
   ┌────▼───────┐    ┌────────▼────────┐  ┌─────▼──────┐   ┌──────▼────────┐
   │Mobile App  │    │ Smart TV        │  │ Web        │   │ Set-Top Box   │
   └────┬───────┘    └────────┬────────┘  └─────┬──────┘   └──────┬────────┘
        └─────────────────────┴─────────────────┴─────────────────┘
                                       │
                                       │  (Catalog/Playback API calls)
                                       │
                            ┌──────────▼──────────────┐
                            │   API Gateway           │
                            └──────────┬──────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
   ┌────▼───────────┐         ┌────────▼────────┐         ┌──────────▼──────┐
   │Catalog Service  │         │Playback Service │         │ License Service │
   │  ES + Postgres  │         │ Manifest gen    │         │   DRM keys       │
   │  Recs (Redis)   │         │ Heartbeats      │         │   Widevine etc.  │
   └────┬───────────┘         └────────┬────────┘         └──────────────────┘
        │                              │
        │                       ┌──────▼──────┐
        │                       │ Cassandra   │
        │                       │ (playback)  │
        │                       └──────┬──────┘
        │                              │
        │                       ┌──────▼──────┐
        │                       │ Kafka:      │
        │                       │ play.events │
        │                       └──────┬──────┘
        │                              │
        │              ┌───────────────┼─────────────────┐
        │              ▼               ▼                 ▼
        │       ┌────────────┐  ┌──────────────┐  ┌──────────────┐
        └──────▶│ML/Recs     │  │ Analytics    │  │ Concurrent   │
                │ Pipeline   │  │ (BigQuery)   │  │ Stream Limit │
                └────────────┘  └──────────────┘  └──────────────┘
```

### Playback Flow
1. Client requests manifest: `GET /playback/manifest/{contentId}` → Playback Service.
2. Service checks entitlement (subscription, geo, parental), generates signed manifest URL pointing to CDN, plus DRM license URL.
3. Client fetches manifest from CDN (cached at edge).
4. Client picks rendition via ABR algorithm; downloads encrypted segments from CDN.
5. Client requests DRM license from License Service → License Service issues per-device license bound to content + user.
6. Client decrypts and plays.
7. Heartbeat every 30s with position → Playback Service → Cassandra + Kafka (`play.events`).

## 2.7 Deep Dive 1 — Adaptive Bitrate (ABR)

- **Multiple renditions** generated at upload: 240p (400 kbps), 480p (1 Mbps), 720p (2.5 Mbps), 1080p (5 Mbps), optionally 4K.
- **HLS Master Playlist** lists all renditions; client (e.g. ExoPlayer) measures bandwidth + buffer health → switches.
- **Server-side recommendations:** start with mid-tier (480p) for fast start; client adapts.
- **Origin → CDN sequencing:** segments are 2-6s long; LL-HLS uses partial segments (~250ms) for low-latency.

## 2.8 Deep Dive 2 — DRM + License Flow

```
Client                                Playback Svc        License Svc           DRM Vendor (Widevine)
  │                                       │                    │                        │
  │──GET /manifest/{contentId}─────────▶│                    │                        │
  │                                       │ check entitlement  │                        │
  │                                       │ generate signed URL│                        │
  │◀──manifest URL + licenseUrl──────────│                    │                        │
  │                                       │                    │                        │
  │──GET manifest from CDN──────▶ CDN ───▶                    │                        │
  │◀──manifest playlist─────────                              │                        │
  │                                                             │                        │
  │──fetch encrypted segments from CDN ────▶                   │                        │
  │                                                             │                        │
  │──POST /license {challenge}───────────────────────────────▶│                        │
  │                                                             │  validate user/content │
  │                                                             │──forward challenge───▶│
  │                                                             │◀──license response───│
  │◀──license────────────────────────────────────────────────│                        │
  │ (decrypt + play)                                            │                        │
```

- License is **per device per content**, time-bound.
- License renewals happen mid-playback via heartbeat events.

## 2.9 Deep Dive 3 — Live Streaming

- Encoder pushes RTMP to ingest endpoint → packager produces HLS segments → push to CDN.
- For 1M+ concurrent live viewers: CDN is mandatory; origin shielding (mid-tier cache) reduces origin load.
- **DVR window** (rewind live by N minutes) → keep last N minutes of segments at origin; older ones evicted.
- **Sub-10s LL-HLS** uses chunked transfer + partial segments + HTTP/2 push.

## 2.10 Deep Dive 4 — Personalized Catalog Feed

```
Offline pipeline (daily / hourly):
  Watch events (Cassandra) → Spark/Flink → embed users + content → ANN search
  → top-K candidates per user → write to Redis: rec:user:{userId}:home
  TTL: 1h

Online (request time):
  GET /catalog/feed?userId
  → fetch rec:user:{userId}:home  (Redis hit ~95%)
  → on miss, fall back to "popular this week" from Redis
  → enrich with metadata (Postgres)
  → return rails (Continue Watching, Recs, Trending, etc.)
```

## 2.11 Deep Dive 5 — Concurrent Stream Enforcement

Subscription tier allows N concurrent streams (e.g. 2). When user starts 3rd stream:
- Increment counter `streams:{userId}` in Redis (TTL refreshed by heartbeats).
- On exceed → return 403; UI prompts "stop another device".
- Graceful: heartbeat-driven counter — if a device crashes, counter expires after 90s.

## 2.12 LLD — Key Classes

```java
public interface PlaybackService {
    ManifestResponse getManifest(long userId, long contentId, DeviceInfo device);
    void heartbeat(long userId, PlaybackHeartbeat hb);
}

@Service
@RequiredArgsConstructor
public class PlaybackServiceImpl implements PlaybackService {
    private final EntitlementService entitlements;
    private final CatalogService catalog;
    private final ManifestGenerator manifestGen;
    private final ConcurrencyLimiter concurrency;
    private final HeartbeatRecorder recorder;
    private final SignedUrlSigner urlSigner;

    public ManifestResponse getManifest(long userId, long contentId, DeviceInfo device) {
        var entitlement = entitlements.check(userId, contentId, device.geo());
        if (!entitlement.allowed()) throw new ForbiddenException(entitlement.reason());

        if (!concurrency.tryAcquire(userId, device.deviceId())) {
            throw new TooManyStreamsException();
        }

        var content = catalog.get(contentId);
        var manifestUrl = manifestGen.signedManifestUrl(content, device);
        var licenseUrl  = manifestGen.licenseUrl(content, userId, device);
        return new ManifestResponse(manifestUrl, licenseUrl, content.renditions());
    }

    public void heartbeat(long userId, PlaybackHeartbeat hb) {
        recorder.record(userId, hb);            // Cassandra
        concurrency.refresh(userId, hb.deviceId());  // extend Redis TTL
    }
}

public interface CatalogService {
    ContentDetail get(long contentId);
    List<ContentDetail> feed(long userId, FeedRequest req);
    SearchResult search(String q, SearchOptions opts);
}

@Component
@RequiredArgsConstructor
public class FeedServiceImpl implements CatalogService {
    private final RedisTemplate<String, List<Long>> redis;
    private final ContentRepository contentRepo;
    private final RecommendationFallback fallback;

    public List<ContentDetail> feed(long userId, FeedRequest req) {
        var cached = redis.opsForValue().get("rec:user:" + userId + ":home");
        var ids = cached != null ? cached : fallback.popular();
        return contentRepo.findAllById(ids);
    }
}

@Component
public class ConcurrencyLimiter {
    private final RedisTemplate<String, String> redis;
    private final SubscriptionService subscriptions;

    public boolean tryAcquire(long userId, String deviceId) {
        int max = subscriptions.maxConcurrent(userId);
        var key = "streams:" + userId;
        var newCount = redis.opsForHash().put(key, deviceId, now());
        redis.expire(key, Duration.ofSeconds(120));
        long active = redis.opsForHash().keys(key).size();
        return active <= max;
    }

    public void refresh(long userId, String deviceId) {
        redis.opsForHash().put("streams:" + userId, deviceId, now());
        redis.expire("streams:" + userId, Duration.ofSeconds(120));
    }
}
```

## 2.13 Trade-Offs

- **CDN-heavy architecture** — expensive at scale but unavoidable. Trade-off: vendor coupling; mitigated with multi-CDN routing.
- **Cassandra for playback events** — write-heavy, time-series, no joins → perfect fit. Trade-off: limited query flexibility; offset by Kafka stream for analytics.
- **Eventually-consistent recommendations** — daily refresh; trade-off: stale for new users (cold start) → fallback to popularity.
- **Pre-generated renditions** — storage 5x originals; trade-off: instant ABR vs JIT transcoding (which would add latency + GPU cost on origin).

## 2.14 Failure Modes

| Scenario | Behavior |
|---|---|
| CDN edge cache miss | Origin shield absorbs; client sees slightly higher TTFB |
| DRM license service down | Client cannot start; existing streams continue (license cached) |
| Heartbeat service down | Concurrent stream counter eventually expires → user can re-start; resume position last-known |
| Cassandra outage | Heartbeats buffered locally on app server (with cap); resume on recovery |
| Recommendation cache stale | Fallback to popularity; user sees less personalized but still good content |

## 2.15 Scale to 10x

- Multi-CDN routing (geo + cost-based); origin shielding.
- Manifest signing on edge workers (Lambda@Edge / CloudFront Functions).
- Sharded Cassandra for playback; partition by `(user_id, week)` if needed.
- Recommendation pipeline → online inference for hot users.

---

# 3. TELCO-SCALE NOTIFICATION SERVICE

> Airtel sends OTPs, bill reminders, recharge confirmations, marketing pushes — at 400M scale.

## 3.1 Clarifying Questions

1. "Channels?" → SMS, Email, Push (FCM/APNS), In-app, IVR-call-based OTP.
2. "OTP latency budget?" → p99 SMS delivery <10s.
3. "Marketing scale?" → 50M-message campaigns in 30 min.
4. "Priority tiers?" → Transactional (OTP, payment confirm) vs Marketing.
5. "Compliance?" → DLT scrubbing (TRAI), DND, frequency cap, opt-out.
6. "Multi-tenant?" → Internal teams (ops, marketing, finance) all use it.

## 3.2 Requirements

### Functional
- Send via specified channels with template rendering.
- Per-user preferences + opt-out.
- Frequency capping.
- Delivery tracking (sent/delivered/failed).
- Retry on transient failure.
- Provider failover.
- Audit + retention.

### Non-functional
- OTP p99 <10s end-to-end.
- 99.99% on transactional path.
- Marketing best-effort.
- Durable + auditable.

## 3.3 Capacity

```
Users                : 400M (Airtel scale)
SMS / day            : 100M (transactional + marketing)
Peak SMS rate        : 50M / 30 min ≈ 28K TPS (campaign)
OTPs / sec sustained : 5K (festival)
OTPs / sec peak      : 50K
Push / day           : 200M
Storage retention    : 90 days hot + 7 yr cold
```

## 3.4 API Design

```http
POST /v1/notifications
  Body: {userId, channels[], templateId, params{}, priority, idempotencyKey}
  Resp: {notificationId, status}

POST /v1/notifications/bulk
  Body: {templateId, audience: {filter|userIds[]}, params, scheduleAt?}

GET  /v1/notifications/{id}
GET  /v1/users/{userId}/preferences
PUT  /v1/users/{userId}/preferences
POST /v1/notifications/webhook       # carrier delivery receipts
```

## 3.5 Data Model

```sql
CREATE TABLE notifications (
  id              BIGINT PRIMARY KEY,
  user_id         BIGINT,
  template_id     BIGINT,
  params          JSONB,
  priority        ENUM('OTP','TRANSACTIONAL','MARKETING'),
  idempotency_key VARCHAR(64),
  status          ENUM('QUEUED','SENT','DELIVERED','FAILED','SUPPRESSED'),
  created_at      TIMESTAMP,
  UNIQUE (user_id, idempotency_key),
  KEY idx_user_time (user_id, created_at)
);

CREATE TABLE notification_attempts (
  id               BIGINT,
  notification_id  BIGINT,
  channel          ENUM('SMS','EMAIL','PUSH','IVR'),
  provider         VARCHAR(32),
  status           ENUM('SENT','DELIVERED','FAILED'),
  error            TEXT,
  attempted_at     TIMESTAMP
) PARTITION BY RANGE (attempted_at);

CREATE TABLE templates (
  id           BIGINT PRIMARY KEY,
  channel      VARCHAR(16),
  body         TEXT,
  variables    TEXT[],
  dlt_id       VARCHAR(32),       -- TRAI DLT registration
  category     VARCHAR(32),
  active       BOOLEAN
);

CREATE TABLE user_preferences (
  user_id     BIGINT PRIMARY KEY,
  sms_opt_in  BOOLEAN DEFAULT TRUE,
  email_opt_in BOOLEAN DEFAULT TRUE,
  push_opt_in BOOLEAN DEFAULT TRUE,
  dnd         BOOLEAN DEFAULT FALSE,
  daily_cap   INT DEFAULT 5
);

-- Frequency counter: Redis sliding window, key freq:{userId}:{templateCategory}
```

## 3.6 High-Level Design

```
                                  Producers (OTP, Marketing, Bill, Recharge)
                                            │
                                  ┌─────────▼──────────┐
                                  │  Ingest API        │
                                  │  (validate, dedup) │
                                  └─────────┬──────────┘
                                            │
                          ┌─────────────────┼─────────────────┐
                          ▼                 ▼                 ▼
                    ┌───────────┐    ┌───────────┐    ┌───────────┐
                    │OTP topic  │    │Trans topic│    │Mktg topic │
                    │(Kafka,    │    │(Kafka,    │    │(Kafka,    │
                    │ many parts│    │ medium)   │    │ low prio) │
                    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
                          │                │                │
                  ┌───────▼─────────┐ ┌────▼──────────┐ ┌───▼────────┐
                  │OTP Workers      │ │Trans Workers  │ │Mktg Workers│
                  │(autoscale 0-N)  │ │               │ │(rate-shaped│
                  │                 │ │               │ │+batched)   │
                  └─────────┬───────┘ └────┬──────────┘ └─────┬──────┘
                            │              │                  │
                       ┌────▼──────────────▼──────────────────▼───┐
                       │  Channel Adapters (SMS/Email/Push/IVR)    │
                       │  + Template Engine + Frequency Cap +     │
                       │  DLT scrub + DND check + Provider Router │
                       └─────────────────────┬─────────────────────┘
                                             │
                  ┌─────────────────┬────────┴────────┬─────────────────┐
                  ▼                 ▼                 ▼                 ▼
           ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
           │SMS Aggr 1 │    │ SES        │    │ FCM/APNS  │    │ IVR vendor│
           └─────┬─────┘    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
                 │                │                │                │
                 └────────────────┴────────┬───────┴────────────────┘
                                           ▼
                                     Carrier Webhooks
                                     (delivery receipts)
                                           │
                                  ┌────────▼─────────┐
                                  │ Receipt Collector│
                                  │ → update attempts│
                                  │ → trigger retry  │
                                  └──────────────────┘
```

## 3.7 Deep Dive 1 — Priority Isolation (OTP vs Marketing)

- **Separate Kafka topics** with separate consumer groups.
- **OTP workers autoscale 0→100** based on lag; marketing workers run at flat low rate.
- **Marketing is rate-shaped** at producer side (token bucket per campaign) so it never starves OTP.
- **Independent dead-letter topics** per priority — OTP failures alert pager; marketing failures go to weekly review.

## 3.8 Deep Dive 2 — Provider Routing + Failover

- **SMS providers (3+):** Karix (primary), Gupshup (secondary), Aggregator C (fallback).
- **Per-provider circuit breaker** in Redis: `cb:provider:{name}` with failure count + half-open probes.
- **Per-provider rate limit** to respect aggregator throughput.
- **Delivery rate window** per provider: if delivered/sent < 80% in last 5 min → mark unhealthy, route to next.

## 3.9 Deep Dive 3 — Frequency Cap + DND

```
Pre-send checks (in order):
  1. dnd_check(userId, templateCategory) → reject SUPPRESSED
  2. user_pref(userId, channel)          → reject SUPPRESSED if opted out
  3. dlt_scrub(template)                 → reject if not registered (TRAI)
  4. freq_cap(userId, category, daily)   → Redis sliding window
       INCR freq:{userId}:{category}; EXPIRE 24h
       if count > cap → SUPPRESSED
  5. quiet_hours(userId, now)            → 9pm-9am block for marketing only
```

## 3.10 Deep Dive 4 — OTP Hot Path

OTP needs <10s. Optimizations:
- Skip Kafka for OTP — direct sync HTTP from auth service to OTP-worker pool (avoid one queue hop).
- Per-pod local rate limiter (no Redis round-trip).
- Pre-rendered template (no DB hit).
- Provider selection: pick lowest-latency-region provider.
- Voice fallback: if SMS not delivered in 30s, trigger IVR call.

## 3.11 Deep Dive 5 — Marketing Campaign at 50M Scale

```
campaign launch:
  1. Audience builder runs against subscriber DB (BigQuery): COPY ids to Redis stream.
  2. Worker pool reads stream at controlled rate (e.g. 28K TPS).
  3. For each id: hydrate template params (often: just user name) → push to provider.
  4. Provider response → notification_attempts.
  5. Carrier webhook → delivery_receipt → analytics rollup.

Campaign control:
  - Pause/resume → flip Redis flag, workers respect.
  - Max-error-rate kill switch → if error > 5% in 1 min, halt + alert.
  - Per-provider bucket so one stuck provider doesn't backlog campaign.
```

## 3.12 LLD — Key Classes

```java
public interface NotificationService {
    NotificationResponse send(SendRequest req);
    BulkResponse sendBulk(BulkRequest req);
    Notification get(long id);
}

@Service
@RequiredArgsConstructor
public class NotificationServiceImpl implements NotificationService {
    private final IdempotencyManager idempotency;
    private final NotificationRepository repo;
    private final KafkaTemplate<String, NotificationMessage> kafka;
    private final TemplateService templates;
    private final UserPreferenceService prefs;

    public NotificationResponse send(SendRequest req) {
        var existing = idempotency.lookup(req.userId(), req.idempotencyKey());
        if (existing.isPresent()) return existing.get();

        templates.validate(req.templateId());
        var n = repo.save(Notification.queued(req));
        idempotency.cache(req.userId(), req.idempotencyKey(), n);

        var topic = topicFor(req.priority());
        kafka.send(topic, String.valueOf(req.userId()), NotificationMessage.from(n));

        return NotificationResponse.from(n);
    }
}

public interface ChannelAdapter {
    Channel channel();
    DeliveryResult send(NotificationContext ctx);
}

@Component
@RequiredArgsConstructor
public class SmsChannelAdapter implements ChannelAdapter {
    private final SmsProviderRouter router;
    private final TemplateEngine engine;
    private final DltService dlt;

    public Channel channel() { return Channel.SMS; }

    public DeliveryResult send(NotificationContext ctx) {
        var template = ctx.template();
        var rendered = engine.render(template, ctx.params());
        if (!dlt.isRegistered(template.dltId())) {
            return DeliveryResult.suppressed("DLT_NOT_REGISTERED");
        }
        var provider = router.pick(ctx.userId());
        try {
            var resp = provider.send(ctx.toPhoneNumber(), rendered);
            return DeliveryResult.sent(provider.name(), resp.providerMsgId());
        } catch (ProviderException ex) {
            router.markFailure(provider.name());
            return DeliveryResult.failed(ex.getMessage());
        }
    }
}

@Component
@RequiredArgsConstructor
public class FrequencyCapService {
    private final RedisTemplate<String, Long> redis;
    private final UserPreferenceService prefs;

    public boolean allow(long userId, String category) {
        var key = "freq:" + userId + ":" + category;
        var count = redis.opsForValue().increment(key, 1);
        if (count == 1) redis.expire(key, Duration.ofHours(24));
        var cap = prefs.dailyCap(userId);
        return count <= cap;
    }
}

@Component
@RequiredArgsConstructor
public class NotificationWorker {
    private final List<ChannelAdapter> adapters;
    private final NotificationRepository repo;
    private final NotificationAttemptRepository attempts;
    private final FrequencyCapService freqCap;
    private final UserPreferenceService prefs;

    @KafkaListener(topics = "notifications.transactional",
                   concurrency = "20",
                   containerFactory = "transactionalContainer")
    public void onMessage(NotificationMessage msg) {
        var n = repo.findById(msg.notificationId()).orElseThrow();

        for (var channel : msg.channels()) {
            if (!prefs.optedIn(n.userId(), channel)) {
                attempts.save(Attempt.suppressed(n, channel, "OPT_OUT"));
                continue;
            }
            if (n.priority() != Priority.OTP && !freqCap.allow(n.userId(), n.category())) {
                attempts.save(Attempt.suppressed(n, channel, "FREQ_CAP"));
                continue;
            }
            var adapter = pick(adapters, channel);
            var ctx = NotificationContext.of(n, channel, prefs.contact(n.userId(), channel));
            var result = adapter.send(ctx);
            attempts.save(Attempt.from(n, channel, result));
        }
    }
}
```

## 3.13 Trade-Offs

- **Kafka skipped for OTP** → lower latency; trade-off: less observability/replay; mitigated by direct attempt log.
- **Per-pod local rate limit** for OTP → fast; trade-off: less precise globally; OTP rate is small relative to capacity.
- **Frequency cap in Redis** → fast but if Redis dies we may over-send; mitigation: cache fail-closed for marketing, fail-open for OTP.

## 3.14 Failure Modes

| Scenario | Behavior |
|---|---|
| Primary SMS provider down | Circuit breaker opens; route to secondary |
| All SMS providers down | OTP fallback to IVR voice call |
| Kafka partition lag growing | Auto-scale workers; alert at 5-min lag SLO |
| Carrier webhook missed | Polling fallback queries provider for status |
| User device offline (push) | FCM stores up to 28 days; APNS limited |

## 3.15 Scale to 10x

- Partition Kafka by `userId` for ordering per user.
- Pre-render template once per campaign for bulk sends.
- Region-local SMS providers for international.
- Dedicate workers per channel; horizontal autoscale on lag.

---

# 4. AI CUSTOMER SUPPORT ASSISTANT

> Pivot to your Eklavya. Airtel + Google Cloud have a public GenAI partnership — this is a strong fit.

## 4.1 Clarifying Questions

1. "Channels?" → Web chat, mobile in-app, WhatsApp, IVR (text-to-speech).
2. "Use cases?" → Bill enquiry, recharge help, plan recommendation, network issue troubleshooting, agent handoff.
3. "Multi-language?" → Hindi + English + 10 regional.
4. "Hallucination tolerance?" → Zero on numerical answers (bill amount, plan price). Tools mandatory for facts.
5. "Personalization?" → Use subscriber profile (plan, recent recharge, network area).
6. "Scale?" → 10M sessions/day; 5M concurrent during outage events.

## 4.2 Requirements

### Functional
- NL conversation in subscriber's language.
- Plan + bill enquiry via tools (no hallucination on numbers).
- Recharge plan recommendation.
- Troubleshoot network issues with conversational diagnostics.
- Escalate to human agent with full context.
- PII firewall on input + output.

### Non-functional
- p95 latency per turn <3s.
- Tool calls deterministic + auditable.
- Conversation history persisted 90 days.

## 4.3 Capacity

```
Sessions/day        : 10M
Avg turns/session   : 5
Total turns/day     : 50M
Avg LLM cost/turn   : $0.005
Daily LLM spend     : $250K (must optimize)
Concurrency peak    : 5M (outage events)
```

## 4.4 API Design

```http
POST /v1/chat/sessions                       # create session
POST /v1/chat/sessions/{id}/messages         # send user message; stream response
GET  /v1/chat/sessions/{id}                  # history
POST /v1/chat/sessions/{id}/handoff          # escalate to human
POST /v1/chat/feedback                       # thumbs up/down per turn
```

## 4.5 Data Model

```sql
CREATE TABLE conversations (
  id            BIGINT PRIMARY KEY,
  user_id       BIGINT,
  channel       VARCHAR(16),
  language      VARCHAR(8),
  status        ENUM('ACTIVE','HANDED_OFF','CLOSED'),
  created_at    TIMESTAMP,
  closed_at     TIMESTAMP,
  KEY idx_user_time (user_id, created_at)
);

CREATE TABLE turns (
  id              BIGINT PRIMARY KEY,
  conversation_id BIGINT,
  role            ENUM('USER','ASSISTANT','TOOL'),
  content         TEXT,           -- post-PII-mask
  raw_content_id  VARCHAR(64),    -- ref to encrypted blob (PII-bearing) in vault
  tools_called    JSONB,          -- list of tool invocations
  intent          VARCHAR(64),
  specialist      VARCHAR(64),
  tokens_in       INT,
  tokens_out      INT,
  cost_usd        DECIMAL(10,6),
  latency_ms      INT,
  created_at      TIMESTAMP
);

CREATE TABLE tool_invocations (
  id           BIGINT PRIMARY KEY,
  turn_id      BIGINT,
  tool_name    VARCHAR(64),
  args         JSONB,             -- post-PII-mask
  result       JSONB,             -- post-PII-mask
  duration_ms  INT,
  status       VARCHAR(16),
  created_at   TIMESTAMP
);
```

## 4.6 High-Level Design

```
                                Channels (Web / WhatsApp / Mobile / IVR)
                                          │
                                ┌─────────▼─────────┐
                                │ Channel Adapters  │
                                │ (normalize to     │
                                │  ChatRequest)     │
                                └─────────┬─────────┘
                                          │
                                ┌─────────▼──────────┐
                                │  Session Manager   │
                                │  (Redis + Postgres)│
                                └─────────┬──────────┘
                                          │
                                ┌─────────▼──────────┐
                                │  PII Firewall (in) │
                                └─────────┬──────────┘
                                          │
                                ┌─────────▼──────────┐
                                │ Intent Router      │
                                │ (Haiku — cheap)    │
                                │ → specialist       │
                                └─────────┬──────────┘
                                          │
                                ┌─────────▼──────────┐
                                │ Specialist Agent   │
                                │ (Sonnet, prompt +  │
                                │  tool allowlist)   │
                                └─────┬──────┬───────┘
                                      │      │
                            ┌─────────▼─┐  ┌─▼──────────┐
                            │Tool       │  │LLM Response│
                            │Executor   │  │            │
                            └────┬──────┘  └─────┬──────┘
                                 │               │
                       ┌─────────▼──────┐ ┌──────▼──────┐
                       │Tool Registry   │ │PII Firewall │
                       │ - getBill      │ │  (out)      │
                       │ - getPlans     │ └──────┬──────┘
                       │ - getNetwork   │        │
                       │ - createTicket │        │
                       │   (gated)      │        │
                       └─────────┬──────┘        │
                                 │               │
                       ┌─────────▼──────┐        │
                       │Backend APIs    │        │
                       │ (read-only)    │        │
                       └────────────────┘        │
                                                  │
                       ┌──────────────────────────▼──────┐
                       │ Conversation Persistence        │
                       │ (Postgres + Object Store for raw)│
                       └─────────────────────────────────┘
                                          │
                                ┌─────────▼──────────┐
                                │ Eval Harness +     │
                                │ Quality Pipeline   │
                                └────────────────────┘
```

## 4.7 Deep Dive 1 — Single-Specialist-Per-Turn

(Same architecture choice as your Eklavya — drop this verbatim.)

- Cheaper, lower latency, consistent voice.
- Routing accuracy matters more than specialist quality.
- Routing on cheap model (Haiku); specialist on expensive (Sonnet).

## 4.8 Deep Dive 2 — Tool Permission Scoping

```
SpecialistConfig:
  name: "BillingSpecialist"
  prompt: "You help users with billing enquiries..."
  allowedTools: [getBill, getPaymentHistory, getInvoicePdfUrl]
  confirmationGated: [createTicket]   -- requires explicit user confirm

ToolExecutor pre-call check:
  if (!specialist.allowedTools.contains(toolName)) reject;
  if (specialist.confirmationGated.contains(toolName) && !ctx.confirmed) askConfirm;
```

## 4.9 Deep Dive 3 — PII Firewall (3 boundaries)

```
1. Inbound user message:
     scan + mask → store masked in turns.content
     store raw in encrypted blob with retention policy

2. Outbound LLM response:
     scan + mask → never let model echo PII it inferred (defense vs prompt-leak)

3. Tool args + return JSON:
     scan + mask before logging
     model never sees full PAN/Aadhaar — only last-4 / masked

Failure mode:
  PII firewall errors → fail-closed (mask everything) — never let unmasked PII leak.
```

## 4.10 Deep Dive 4 — Hallucination Defense for Numbers

```
Specialist prompt:
  "Never make up bill amounts, plan prices, or dates.
   Always call the appropriate tool. If the tool fails, say
   'I'm having trouble fetching that' — do NOT guess."

Eval set:
  Frozen test conversations with assertions:
    - Specialist matched
    - Allowed tool was called
    - No PII in output
    - Numerical answers came from tool, not LLM
```

## 4.11 Deep Dive 5 — Cost Control

- **Routing on Haiku** (~10x cheaper than Sonnet).
- **Sliding window history** — keep last 10 turns; older turns summarized.
- **Tool result truncation** — return only fields needed by the prompt.
- **Pre-emptive caching** — common questions ("what's my plan") cached per user with TTL.
- **Streaming** on Web channel; non-streamed on WhatsApp.

## 4.12 LLD — Key Classes

```java
public interface AgentOrchestrator {
    AgentResponse respond(AgentRequest req);
}

@Service
@RequiredArgsConstructor
public class AgentOrchestratorImpl implements AgentOrchestrator {
    private final SessionManager sessions;
    private final PiiFirewall pii;
    private final IntentRouter router;
    private final SpecialistAgentFactory specialists;
    private final ConversationPersistence persistence;
    private final ProgramConfigService programConfig;

    public AgentResponse respond(AgentRequest req) {
        var session = sessions.getOrCreate(req);
        var maskedInput = pii.maskInput(req.userMessage());
        var program = programConfig.lookup(req.channel(), req.programId());
        var routedSpecialistName = router.route(maskedInput, session.history(), program);
        var specialist = specialists.get(routedSpecialistName);
        var raw = specialist.respond(maskedInput, session.history(), program);
        var maskedOutput = pii.maskOutput(raw.content());
        persistence.recordTurn(session, req, maskedInput, raw, maskedOutput);
        return AgentResponse.of(maskedOutput, raw.toolCalls());
    }
}

public interface SpecialistAgent {
    String name();
    SpecialistResponse respond(String userMsg, List<Turn> history, ProgramConfig program);
}

@Component
@RequiredArgsConstructor
public class BillingSpecialist implements SpecialistAgent {
    private final BedrockClient bedrock;
    private final ToolExecutor toolExecutor;
    private final ToolPermissionScoper scoper;
    private final ConfirmationGate gate;

    public SpecialistResponse respond(String userMsg, List<Turn> history, ProgramConfig pc) {
        var systemPrompt = buildPrompt(pc);
        var allowedTools = scoper.toolsFor("billing");
        var llmRequest = LlmRequest.builder()
            .system(systemPrompt)
            .messages(history.stream().map(Turn::toMessage).toList())
            .userMessage(userMsg)
            .tools(allowedTools)
            .model(pc.specialistModel())  // sonnet
            .build();

        var llmResp = bedrock.chat(llmRequest);
        for (var toolCall : llmResp.toolCalls()) {
            scoper.assertAllowed("billing", toolCall.name());
            if (gate.requiresConfirm(toolCall)) return SpecialistResponse.askConfirm(toolCall);
            var result = toolExecutor.execute(toolCall);
            llmResp = bedrock.chat(llmRequest.withToolResult(toolCall, result));
        }
        return SpecialistResponse.of(llmResp.content(), llmResp.toolCalls());
    }
}

public interface PiiFirewall {
    String maskInput(String userMessage);
    String maskOutput(String llmResponse);
    JsonNode maskJson(JsonNode payload);
}

@Component
@RequiredArgsConstructor
public class RegexPiiFirewall implements PiiFirewall {
    private static final Map<String, Pattern> patterns = Map.of(
        "PAN",     Pattern.compile("[A-Z]{5}[0-9]{4}[A-Z]"),
        "AADHAAR", Pattern.compile("\\b\\d{4}\\s?\\d{4}\\s?\\d{4}\\b"),
        "PHONE",   Pattern.compile("\\b[6-9]\\d{9}\\b"),
        "EMAIL",   Pattern.compile("[\\w.-]+@[\\w.-]+\\.\\w+")
    );

    public String maskInput(String msg) {
        try {
            var out = msg;
            for (var entry : patterns.entrySet()) {
                out = entry.getValue().matcher(out).replaceAll("[" + entry.getKey() + "]");
            }
            return out;
        } catch (Exception e) {
            // Fail-closed: mask everything if regex fails
            return "[REDACTED]";
        }
    }
    // similar for output and json
}
```

## 4.13 Trade-Offs

- **No RAG for troubleshooting** → playbooks in prompt (versioned in code), reviewable in PR. Trade-off: prompt cost; benefit: deterministic, easy to audit.
- **Read-only tools** → mutations (raise ticket) are confirmation-gated. Slower UX; safer.
- **MySQL for sessions over Redis** → durability + audit > speed.
- **Haiku for routing** → cost vs accuracy; eval on routing test set guides model choice.

## 4.14 Failure Modes

| Scenario | Behavior |
|---|---|
| LLM API timeout | Fall back to FAQ-style canned response + "let me get a human" |
| Tool call fails | Specialist instructed: "I'm having trouble — would you like a human agent?" |
| PII firewall errors | Fail-closed: full redaction; alert ops |
| Session storage down | New sessions fail; existing sessions degraded (no history); alert |
| Cost spike | Per-user rate limit + per-program kill switch |

## 4.15 Scale to 10x

- Specialist sharding by intent for warm caches.
- Pre-cache common turns (e.g. "what's my balance" answered without LLM).
- Distill specialists into smaller models offline; serve from your own infra.
- Multi-region Bedrock for latency.

---

# 5. MOBILE RECHARGE / PLAN MANAGEMENT PLATFORM

> Airtel's bread and butter. High QPS, payment-integrated, telecom operator integration.

## 5.1 Clarifying Questions

1. "Self-recharge only or recharge-for-others?" → Both.
2. "Operators?" → Airtel + cross-operator (Jio, VI, BSNL).
3. "Plan catalog dynamism?" → Plans change weekly; some are user-targeted.
4. "Latency target?" → p95 recharge initiation <3s; status check <500ms.
5. "Scale?" → 50M recharges/month; peak 10K TPS festival.
6. "Reconciliation with operator?" → Daily settlement files.

## 5.2 Requirements

### Functional
- Browse plans (geo + circle + user-targeted).
- Validate mobile number → fetch operator + circle (LRN/MNP).
- Initiate recharge with payment.
- Track recharge status (queued → submitted → confirmed/failed).
- Recharge history.
- Auto-pay setup.
- Refund on failure.

### Non-functional
- p95 recharge initiation <3s.
- 99.9% on payment + recharge composite.
- Idempotent on `Idempotency-Key`.

## 5.3 Capacity

```
Recharges/month  : 50M
Daily peak       : 5M
Daily avg TPS    : ~60
Festival peak    : 10K TPS
Storage/yr       : 600M × 1KB ≈ 600GB
```

## 5.4 API Design

```http
GET  /v1/plans?msisdn=&circle=         # plan catalog (cached aggressively)
POST /v1/recharge
  Headers: Idempotency-Key
  Body: {msisdn, planId, paymentMethod, autopay?}
  Resp: {rechargeId, paymentRedirect?, status}
GET  /v1/recharge/{id}                 # status polling
GET  /v1/recharge/history?userId=
POST /v1/autopay
POST /v1/recharge/{id}/refund
POST /v1/operators/webhook              # operator confirmation
```

## 5.5 Data Model

```sql
CREATE TABLE recharges (
  id              BIGINT PRIMARY KEY,
  user_id         BIGINT,
  msisdn          VARCHAR(15),
  operator        VARCHAR(16),
  circle          VARCHAR(16),
  plan_id         BIGINT,
  amount          DECIMAL(10,2),
  status          ENUM('INITIATED','PAYMENT_DONE','SUBMITTED','CONFIRMED','FAILED','REFUNDED'),
  payment_id      BIGINT,
  operator_ref    VARCHAR(64),
  idempotency_key VARCHAR(64),
  created_at      TIMESTAMP,
  UNIQUE (user_id, idempotency_key)
);

CREATE TABLE plans (
  plan_id      BIGINT PRIMARY KEY,
  operator     VARCHAR(16),
  circle       VARCHAR(16),
  category     VARCHAR(32),    -- VOICE, DATA, COMBO, OTT
  price        DECIMAL(10,2),
  validity     INT,
  benefits     JSONB,          -- daily/total data, calls, sms, ott bundle
  active_from  TIMESTAMP,
  active_to    TIMESTAMP,
  active       BOOLEAN
);

CREATE TABLE plan_targeting (
  plan_id     BIGINT,
  segment     VARCHAR(64),    -- pulled from subscriber pipeline
  PRIMARY KEY (plan_id, segment)
);

CREATE TABLE operator_routing (
  operator     VARCHAR(16),
  circle       VARCHAR(16),
  primary_aggregator   VARCHAR(32),
  fallback_aggregator  VARCHAR(32)
);
```

## 5.6 High-Level Design

```
                              Mobile App / Web / IVR
                                       │
                            ┌──────────▼──────────┐
                            │  API Gateway        │
                            └──────────┬──────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
       ┌─────────────┐         ┌──────────────┐         ┌─────────────┐
       │ Plan Catalog│         │ Recharge Svc │         │ History/    │
       │ Service     │         │ (orchestrate)│         │ Analytics   │
       │ (cached agg)│         └──────┬───────┘         └─────────────┘
       └─────┬───────┘                │
             │                        │
       ┌─────▼──────┐         ┌───────▼────────┐
       │ Catalog DB │         │ Payment Svc    │  (calls Airtel Money or PG)
       │ Redis cache│         └───────┬────────┘
       └────────────┘                 │
                                      ▼
                            ┌────────────────────┐
                            │ MNP Lookup Service │
                            │ (LRN cache)        │
                            └────────┬───────────┘
                                     │
                            ┌────────▼───────────┐
                            │ Operator Adapter   │
                            │  Strategy/Factory  │
                            │  - Airtel internal │
                            │  - Jio (aggregator)│
                            │  - VI (aggregator) │
                            │  - BSNL (aggreg.)  │
                            └────────┬───────────┘
                                     │
                            ┌────────▼───────────┐
                            │ Recharge DB        │
                            │ + Outbox           │
                            └────────┬───────────┘
                                     │
                            ┌────────▼───────────┐
                            │ Kafka: recharge.events │
                            └────────┬───────────┘
                                     │
                ┌────────────────────┼────────────────────┐
                ▼                    ▼                    ▼
          ┌──────────┐        ┌──────────────┐    ┌──────────────┐
          │ Notify   │        │ Recon Service│    │ Settlement   │
          │ (SMS+Push)        │ (operator EOD│    │ Pipeline     │
          └──────────┘        │  files)      │    └──────────────┘
                              └──────────────┘
```

### Recharge Flow
1. Client → `POST /recharge` → Recharge Service.
2. **MNP lookup**: validate `msisdn` → operator + circle (cached aggressively).
3. **Idempotency check**: `(user_id, idempotency_key)` in Redis + DB.
4. **Plan validation**: plan exists, active, valid for circle, eligible for user.
5. **Payment**: call Payment Service (sync). If autopay → use stored mandate; else redirect.
6. On payment success → status `PAYMENT_DONE` → submit to operator via adapter.
7. Operator may respond synchronously (rare) or via webhook (typical) → status `CONFIRMED` / `FAILED`.
8. On `FAILED` → trigger refund through Payment Service (idempotent on rechargeId).
9. Outbox publishes events; notification + analytics + recon consume.

## 5.7 Deep Dive 1 — Plan Catalog Caching

```
Catalog request: GET /plans?msisdn=...
  1. MNP lookup (Redis msisdn:lrn cache, TTL 7 days; on miss → MNP service).
  2. Build cache key: plans:{operator}:{circle}:{user_segment_bucket}
  3. Redis lookup → hit returns instantly.
  4. Miss → DB query joining plans + plan_targeting → cache (TTL 5 min) → return.

Hot key protection:
  - Jittered TTL.
  - Per-pod local cache (Caffeine, 1 min).

Plan changes propagation:
  - Editor publishes via admin → invalidate cache by pattern (Redis SCAN).
  - Or: short TTL + accept up-to-5-min staleness.
```

## 5.8 Deep Dive 2 — Operator Adapter (Strategy + Factory)

```java
public interface OperatorAdapter {
    Operator type();
    boolean supports(String operator, String circle);
    OperatorResponse submit(Recharge r);
    OperatorStatus query(String operatorRef);
    void handleWebhook(Map<String, String> payload);
}
```

- One concrete per operator + aggregator.
- Routing per `(operator, circle)` from `operator_routing` table.
- Per-operator circuit breaker; if open → primary→fallback or queue + retry.

## 5.9 Deep Dive 3 — Webhook Handling (Operator → Us)

Same pattern as Digio in your code:
- HMAC validation.
- Always 200 ack to operator.
- Persist to `recharge_events` (unique on operator event id).
- Worker processes → updates recharge status via state machine.

## 5.10 Deep Dive 4 — State Machine (your bread and butter)

```
INITIATED ──payment_success──▶ PAYMENT_DONE ──submit_to_op──▶ SUBMITTED
                  │                                                │
                  │                                                ▼
                  │                                  ┌──CONFIRMED (success)
                  │                                  │
                  │                                  └──FAILED (operator rejected)
                  │                                            │
                  ▼                                            ▼
             FAILED (payment fail)                          REFUNDED (auto)
                  │
                  ▼
              REFUNDED
```

- Tracker table records every transition with timestamp.
- Trigger service fires events on transitions (notification, recon).
- Same architecture pattern as your DLS state machine.

## 5.11 Deep Dive 5 — Reconciliation

EOD operator settlement file → match against `recharges` table:
- Operator confirms but we marked FAILED → status correction + audit log + notification.
- We marked CONFIRMED but operator denies → reverse + refund.
- Amount mismatches → escalate.

## 5.12 LLD — Key Classes

```java
public interface RechargeService {
    RechargeResponse initiate(RechargeRequest req, String idempotencyKey);
    RechargeStatus get(long rechargeId);
    void handleOperatorCallback(OperatorCallback cb);
}

@Service
@RequiredArgsConstructor
public class RechargeServiceImpl implements RechargeService {
    private final IdempotencyManager idempotency;
    private final MnpLookup mnp;
    private final PlanCatalogService planCatalog;
    private final PaymentService payments;
    private final OperatorRouter operatorRouter;
    private final RechargeRepository repo;
    private final OutboxRepository outbox;

    public RechargeResponse initiate(RechargeRequest req, String idemKey) {
        var existing = idempotency.lookup(req.userId(), idemKey);
        if (existing.isPresent()) return existing.get();

        var msisdnInfo = mnp.lookup(req.msisdn());
        var plan = planCatalog.validate(req.planId(), msisdnInfo);

        // Stage 1: book
        var recharge = bookRecharge(req, plan, msisdnInfo, idemKey);

        // Stage 2: pay (sync)
        var paymentResp = payments.charge(req.userId(), plan.price(),
                                          req.paymentMethod(), recharge.id());
        if (!paymentResp.isSuccess()) {
            markFailed(recharge, "PAYMENT_FAILED");
            return RechargeResponse.failed(recharge);
        }

        // Stage 3: submit to operator (async friendly)
        var operatorResp = operatorRouter.route(msisdnInfo).submit(recharge);
        return finalizeRecharge(recharge, operatorResp);
    }

    @Transactional
    private Recharge bookRecharge(RechargeRequest req, Plan plan,
                                  MsisdnInfo info, String idemKey) {
        try {
            return repo.save(Recharge.initiated(req, plan, info, idemKey));
        } catch (DataIntegrityViolationException e) {
            return repo.findByIdempotency(req.userId(), idemKey).orElseThrow();
        }
    }

    @Transactional
    private RechargeResponse finalizeRecharge(Recharge r, OperatorResponse resp) {
        if (resp.isAccepted()) {
            r.setStatus(SUBMITTED);
            r.setOperatorRef(resp.operatorRef());
            outbox.enqueue(RechargeEvents.submitted(r));
        } else {
            r.setStatus(FAILED);
            outbox.enqueue(RechargeEvents.failed(r));
            // refund triggered downstream via outbox event consumer
        }
        repo.save(r);
        return RechargeResponse.from(r);
    }

    public void handleOperatorCallback(OperatorCallback cb) {
        var r = repo.findByOperatorRef(cb.operatorRef()).orElseThrow();
        if (r.status() != SUBMITTED) return;  // dedup / late callback
        r.setStatus(cb.success() ? CONFIRMED : FAILED);
        outbox.enqueue(cb.success() ? RechargeEvents.confirmed(r) : RechargeEvents.failed(r));
        repo.save(r);
    }
}

public interface OperatorAdapter {
    Operator type();
    boolean supports(String operator, String circle);
    OperatorResponse submit(Recharge r);
}

@Component
class AirtelInternalAdapter implements OperatorAdapter {
    public boolean supports(String op, String c) { return "AIRTEL".equals(op); }
    public OperatorResponse submit(Recharge r) { /* internal API */ }
}

@Component
class CrossOperatorAggregatorAdapter implements OperatorAdapter {
    private final CircuitBreaker breaker;
    public boolean supports(String op, String c) { return !"AIRTEL".equals(op); }
    public OperatorResponse submit(Recharge r) {
        return breaker.executeSupplier(() -> aggregator.submit(r));
    }
}
```

## 5.13 Sequence Diagram

```
Client    Gateway    Recharge      MNP      PlanCat    Payment    Operator    Outbox    Kafka
  │         │          │             │         │          │           │          │         │
  │─POST───▶│─────────▶│             │         │          │           │          │         │
  │         │          │─lookup msisdn▶         │          │           │          │         │
  │         │          │◀──op,circle──         │          │           │          │         │
  │         │          │──validate plan────────▶│          │           │          │         │
  │         │          │◀──ok─────────         │          │           │          │         │
  │         │          │─book (DB tx)          │          │           │          │         │
  │         │          │─charge───────────────────────────▶│           │          │         │
  │         │          │◀──payment ok──────────────────────│           │          │         │
  │         │          │─submit──────────────────────────────────────▶│          │         │
  │         │          │◀──ack/operatorRef──────────────────────────│          │         │
  │         │          │─update status SUBMITTED                     │          │         │
  │         │          │─outbox event                                 │          │         │
  │◀─resp───│◀─────────│                                              │          │─poll──▶│
  │         │          │                                              │──webhook─│         │
  │         │          │◀──op callback (CONFIRMED)                    │          │         │
  │         │          │─update CONFIRMED                             │          │         │
  │         │          │─outbox notify                                │          │         │
```

## 5.14 Trade-Offs

- **Sync payment, async operator** → user gets confirmation fast; recharge confirmation arrives via push/SMS.
- **Per-(operator, circle) routing** → flexible but complex. Mitigated by config-table-driven routing.
- **Cache plans aggressively** → minor staleness vs hot-path speed.
- **Strong consistency on recharge state** → no double recharge ever.

## 5.15 Failure Modes

| Scenario | Behavior |
|---|---|
| Payment fails | Recharge → FAILED; user sees "payment failed" |
| Operator rejects post-payment | FAILED + auto-refund triggered via outbox |
| Operator timeout (no webhook) | Status query cron after 5 min; if still unknown after T, treat as FAILED + refund |
| MNP service down | Use last-known cache; longer TTL fallback |
| Plan changed mid-recharge | Validated at booking; subsequent change doesn't affect in-flight |

## 5.16 Scale to 10x

- Shard recharges by `userId` like payments.
- Plan catalog read replicas + edge cache (CloudFront).
- Per-operator dedicated worker pools so one slow operator doesn't backlog all.
- Festival mode: pre-warm caches, scale workers, raise rate limits.

---

# CLOSING TIPS — INTERVIEW DELIVERY

### Open every design with the same script
> "Before I jump in, can I clarify three things — scale, consistency requirements, and read:write ratio?"

### Use the 7-step skeleton out loud
- Mention each step as you reach it ("OK, capacity-wise…", "moving to the data model…", "let me deep-dive on idempotency…").

### Always volunteer trade-offs
- Don't wait for them to ask. "I'm choosing X over Y because Z; the trade-off is W, which is acceptable here because…"

### Pivot to YOUR projects when relevant
- "We solved this exact pattern in [Eklavya / GPay / DLS NACH] with [class/approach]…"
- This is the senior signal: real shipped patterns, not textbook.

### Closing line for ANY design
> "Bottlenecks I'd watch first: [X]. To 10x: [Y]. Trade-offs I made: chose [A over B because Z]. Anywhere you want me to go deeper?"

That last sentence — "anywhere you want me to go deeper" — invites them to drive the conversation. It's the closer.
