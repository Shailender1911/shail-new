# File 2: Technical Q&A — Tied To Your Real Code

> Every answer ends with a class/file from YOUR codebase. Generic answers fail R1; project-grounded answers pass.
> Format: **Q → why they're asking → tight answer → your real example → follow-up they'll ask + how to handle.**

---

## SECTION A — CORE JAVA (12 Qs)

### A1. `==` vs `.equals()` — and where it actually matters
- **Why asked:** Tests if you understand reference vs value, and string pool behavior.
- **Answer:** `==` compares references; `.equals()` compares logical equality. Strings are pooled, so `==` may accidentally work on literals — never rely on it.
- **Your example:** In `DigioUtility` we compare incoming HMAC-SHA256 hex with computed hex using `equalsIgnoreCase`. `==` would have silently let a tampered webhook through during a string-pool coincidence. Security-relevant choice.
- **Follow-up:** "Why not constant-time compare for HMAC?" → Acknowledge: `MessageDigest.isEqual` is the right call for crypto comparisons; `equalsIgnoreCase` short-circuits and leaks timing. For non-crypto signatures we accept it; for new crypto code we use `MessageDigest.isEqual`.

### A2. HashMap internals (load factor, treeify, hash collisions)
- **Why asked:** Bread-and-butter R1 question.
- **Answer:** Array of buckets; index = `(n-1) & hash`; collisions → linked list, then red-black tree at threshold ≥8 (with capacity ≥64), reverts at ≤6. Default load factor 0.75 → resize doubles capacity and rehashes. Not thread-safe.
- **Your example:** In `ParikshaFlagService` we build `Map<String, Object>` of flag → result via streams. We size-hint the map to avoid resize churn when we know the flag count.
- **Follow-up:** "What's the worst-case complexity?" → O(log n) since Java 8 (treeified buckets), O(n) before that. "What if keys have a poor hashCode?" → Treeification mitigates degenerate cases; we still control input keys to be UUIDs/enums.

### A3. ConcurrentHashMap — how it's thread-safe
- **Why asked:** Senior question; tests beyond `synchronized HashMap`.
- **Answer:** Java 8+ uses CAS + `synchronized` per bucket head, not segment locks. Reads are lock-free; writes lock only the affected bin. `compute`/`computeIfAbsent` are atomic per key.
- **Your example:** `MTLSConnectionFactory` uses a `ConcurrentHashMap`-style cache for SSL contexts keyed by config hash, with `computeIfAbsent` to atomically build a context once per unique cert set. In Eklavya, the MCP wrapper's `_mcp_sessions` (Python equivalent) uses the same per-key atomicity idea.
- **Follow-up:** "Why not `Collections.synchronizedMap`?" → That serializes ALL reads + writes on one mutex; `ConcurrentHashMap` only serializes per-bucket writes and lets reads proceed. Order-of-magnitude better under contention.

### A4. `volatile` — what it actually guarantees
- **Why asked:** Memory model probe.
- **Answer:** Visibility (writes visible to other threads) + ordering (no reordering across the volatile access). It does **not** give atomicity for compound ops (no `volatile++`).
- **Your example:** `MTLSConnectionFactory` is a lazy singleton with `volatile SSLContext instance` and double-checked locking. Without `volatile`, a partially-constructed `SSLContext` could be observed by a second thread.
- **Follow-up:** "Why not just `synchronized` the whole getter?" → Hot path; we'd serialize every TLS handshake setup. DCL avoids the lock after init.

### A5. `synchronized` vs `ReentrantLock` vs `RLock` (distributed)
- **Why asked:** Locking maturity.
- **Answer:**
  - `synchronized`: monitor on object header, JVM-managed, can't try-with-timeout, can't fairness-tune.
  - `ReentrantLock`: explicit, supports `tryLock(timeout)`, fairness, multiple condition variables. Heavier API but more control.
  - Redisson `RLock`: distributed across JVMs, backed by Redis, supports lease-time + watchdog auto-extension.
- **Your example:** Orchestration uses Redisson `RLock` with key `digio:upinach:callback:{mandateId}` to dedup Digio webhook callbacks across pods. We use `tryLock(waitTime, leaseTime, TimeUnit)` so a dead lock holder doesn't freeze us forever.
- **Follow-up:** "What if Redis is down?" → Graceful degradation: we log + proceed without dedup, accept rare double-processing risk over total outage. The downstream `ANachLogsEntity` insert has a unique constraint as a safety net.

### A6. JVM memory + GC (G1 vs ZGC)
- **Why asked:** Production maturity.
- **Answer:** Heap regions: young (Eden + S0/S1), old, metaspace (class metadata), code cache. G1 is region-based, default since Java 9, low-pause target ~200 ms. ZGC: sub-10ms pauses, scales to multi-TB heaps, more memory overhead.
- **Your example:** Eklavya is on Java 17 — we evaluated ZGC for the agent service (long LLM calls hold large response strings) but stayed on G1 because heap is <8 GB and pause time is fine. Older orchestration on Java 8 uses G1 explicitly via `-XX:+UseG1GC`.
- **Follow-up:** "How do you size heap?" → Start from container memory request, leave ~25% for off-heap (Netty buffers, direct byte buffers for crypto), measure with `jstat`/Micrometer JVM metrics, tune `-Xmx`/`-Xms` equal to avoid runtime resize.

### A7. String immutability + String pool
- **Why asked:** Foundational + leads into security.
- **Answer:** Strings are immutable for thread-safety, hashCode caching, and security (a String passed to a method can't be mutated mid-flight). String pool (intern table) deduplicates literals.
- **Your example:** In JWT/PGP code we never use `String` for keys/passphrases beyond the parse boundary — we convert to `char[]` or `byte[]` so we can zero them out (Strings can't be wiped from the pool, so they linger in heap dumps).
- **Follow-up:** "Why is HashMap key often a String?" → Cached hashCode, immutable, safe key. But we use enums where the value space is fixed.

### A8. Java 8 Streams + Optional + Functional Interfaces
- **Why asked:** Idiom check.
- **Answer:** Streams = declarative pipeline (lazy, fused, parallel-capable); Optional = explicit "may be absent" without null; functional interfaces = single-abstract-method types `Function`/`Predicate`/`Consumer`/`Supplier`.
- **Your example:** `ParikshaFlagService` uses streams to map flag list → result map: `flags.stream().collect(toMap(name -> name, name -> evaluateOne(name, ctx)))`. We avoid `parallel()` here because Unleash SDK calls are short and parallelization adds overhead.
- **Follow-up:** "When NOT to use streams?" → Hot path with simple loop, when you need early break with side effects, or when debugging is hard (stack traces are uglier).

### A9. CompletableFuture — composition + error handling
- **Why asked:** Async maturity.
- **Answer:** `supplyAsync` returns a future on a pool; `thenApply`/`thenCompose` chain transformations; `allOf`/`anyOf` aggregate. Errors propagate via `exceptionally`/`handle`.
- **Your example:** `InsuranceServiceImpl.insurancePolicyCron` batches policy fetches in parallel: `CompletableFuture.allOf(futures).join()` after submitting each on `cronThreadPoolExecutor`. Each future has `.exceptionally(ex -> { log; return null; })` so one bad policy doesn't sink the batch.
- **Follow-up:** "Why not `parallelStream`?" → Custom executor (`InsuranceThreadPoolConfig`) — parallelStream uses common ForkJoinPool which is shared with the JVM and unsafe for blocking IO.

### A10. Exception handling (checked vs unchecked, custom hierarchies)
- **Why asked:** API design probe.
- **Answer:** Checked = caller MUST handle (compiler enforced) — good for recoverable; unchecked = programming bugs / unrecoverable. Modern Java + Spring favors unchecked + global handler.
- **Your example:** We have a custom hierarchy: `BaseException` → `BusinessException`, `ValidationException`, `IntegrationException`. `OrchestrationExceptionHandler` (`@ControllerAdvice`) maps each to a response code. For GPay we extend with `GpayErrorResponse` matching Google's contract (`paymentIntegratorErrorCode`, epoch ms timestamp, PGP-encrypted on external path).
- **Follow-up:** "Why not throw checked?" → Boilerplate explodes across layers; we'd lose lambda compatibility (most functional interfaces don't declare checked).

### A11. Thread pools — sizing + types
- **Why asked:** Reliability under load.
- **Answer:** `Executors` static factories are convenient but trap-laden (`newCachedThreadPool` = unbounded; `newFixedThreadPool` = unbounded queue). Use `ThreadPoolExecutor` directly with bounded queue + named threads + rejection policy.
- **Your example:** `InsuranceThreadPoolConfig.cronThreadPoolExecutor` is a bounded `ThreadPoolExecutor` with `LinkedBlockingQueue` capped, `CallerRunsPolicy` so over-flow back-pressures the producer instead of dropping. Eklavya's PII firewall has a similarly bounded executor for parallel scrubbing.
- **Follow-up:** "Sizing rule?" → CPU-bound: `nCPU + 1`. IO-bound: `nCPU * (1 + waitTime/serviceTime)`. We measure with Micrometer thread-pool metrics, not guess.

### A12. Deadlock — detection + prevention
- **Why asked:** Concurrency debugging.
- **Answer:** Two threads each holding a lock the other needs. Prevent by: consistent lock ordering, `tryLock` with timeout, lock striping, eliminating shared state when possible.
- **Your example:** Multi-datasource routing in loan-repayment is per-DB transactions only — we never hold two DB transactions at once. We considered XA across `arthmate_db` + `finflux_db`, ruled it out because deadlock surface explodes; instead we use the outbox-like pattern (write to one DB transactionally, async sync to the other).
- **Follow-up:** "How would you detect in prod?" → Thread dump (`jstack`), Micrometer JVM metrics (`jvm.threads.states`), and `ThreadMXBean.findDeadlockedThreads()` exposed via actuator if you want auto-alerts.

---

## SECTION B — SPRING BOOT (10 Qs)

### B1. Auto-configuration — what it actually does
- **Why asked:** Spring Boot fundamental.
- **Answer:** `@EnableAutoConfiguration` (transitively from `@SpringBootApplication`) reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (was `spring.factories` pre-2.7), then for each candidate class evaluates `@ConditionalOn*` (OnClass, OnMissingBean, OnProperty…) before instantiating.
- **Your example:** `ParikshaConfig` is wrapped in `@ConditionalOnProperty(prefix="pariksha", name="enabled", havingValue="true")` — feature is opt-in per environment. `TeamsNotificationServiceImpl` similarly conditional so non-prod doesn't spam Teams.
- **Follow-up:** "How do you debug auto-config?" → `--debug` prints the conditions report (positive/negative matches). Or `actuator/conditions`.

### B2. `@SpringBootApplication` — what's bundled
- **Why asked:** Trick question to see if you list all three.
- **Answer:** `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` (default base = current package).
- **Your example:** Eklavya overrides the default scan: `@SpringBootApplication(scanBasePackages = {"com.payu.eklavya", "com.payu.lending.client"})` so we pull in shared client beans without putting everything under one root package.
- **Follow-up:** "Why is default scan dangerous?" → If your `@SpringBootApplication` is in `com.example.app`, anything in sibling `com.example.shared` won't be picked up. We've been bitten and now make scan paths explicit.

### B3. DI — constructor vs field vs setter
- **Why asked:** Hygiene check.
- **Answer:** Prefer constructor — immutable fields, easy to test (no reflection), fail-fast at startup, works with `final`. Field injection hides dependencies and breaks testability.
- **Your example:** `ParikshaFlagService` and `GPayService` use Lombok `@RequiredArgsConstructor` to generate the constructor for `final` fields. No `@Autowired` on fields anywhere in new code.
- **Follow-up:** "Circular dependency?" → Refactor (extract a third bean) or last-resort `@Lazy` on one side. We treat the warning as a code smell, not a workaround target.

### B4. `@RestController` vs `@Controller`
- **Why asked:** REST vs MVC.
- **Answer:** `@RestController` = `@Controller` + `@ResponseBody` on every method — returns serialized objects (JSON via Jackson). `@Controller` returns view names for template engines.
- **Your example:** `FeatureFlagController`, `GPayExternalController`, `ApprovalWorkflowController` are all `@RestController`. Older zipcredit MVC pieces are `@Controller` with JSP — being phased out.
- **Follow-up:** "How do you customize JSON?" → `@JsonInclude(NON_NULL)`, `@JsonFormat` for dates, custom `ObjectMapper` bean for global serializers (we register a `JavaTimeModule` for `OffsetDateTime`).

### B5. `@ControllerAdvice` + `@ExceptionHandler`
- **Why asked:** API design.
- **Answer:** Cross-cutting exception → response mapping; keeps controllers clean.
- **Your example:** `OrchestrationExceptionHandler`, `LoanRepaymentExceptionHandler`, `LendingConnectorExceptionHandler` — each maps `BusinessException`/`ValidationException`/`IntegrationException` to a standard envelope. For GPay we have a separate handler returning `GpayErrorResponse` so external Google contract is preserved (different envelope from internal API v4).
- **Follow-up:** "What's the order of resolution?" → Most specific exception type wins; if multiple advices match, `@Order` decides.

### B6. `@Transactional` — propagation + isolation
- **Why asked:** DB maturity.
- **Answer:** Propagation: `REQUIRED` (default — join or create), `REQUIRES_NEW` (suspend + new), `NESTED` (savepoint), `SUPPORTS`/`NOT_SUPPORTED`/`MANDATORY`/`NEVER`. Isolation: `READ_COMMITTED` (typical), `REPEATABLE_READ`, `SERIALIZABLE`.
- **Your example:** We use `REQUIRED` everywhere by default. For mandate cron's idempotent retry we use `REQUIRES_NEW` so the retry log persists even if the outer transaction rolls back. We deliberately do **not** use `@Transactional` across microservices — instead, outbox-like webhook persistence (Orchestration's `ANachLogsEntity` always-200 pattern) gives us at-least-once with dedup.
- **Follow-up:** "Self-invocation gotcha?" → Calling `this.method()` bypasses the proxy; either inject self-reference or split classes. We've shipped both fixes.

### B7. Actuator
- **Why asked:** Production-ready check.
- **Answer:** `/health`, `/info`, `/metrics`, `/prometheus`, `/loggers`, `/conditions`, `/env`. Lock down with security; expose only what you need.
- **Your example:** ConfigNexus exposes `/health` and `/prometheus` (scraped by our Prometheus); `/loggers` requires admin role for runtime log-level changes. Eklavya additionally exposes a custom `/agent/health` that pings Bedrock with a 1-token Haiku call so kube readiness probe catches Bedrock IAM regressions.
- **Follow-up:** "K8s probe vs actuator health?" → Liveness should be cheap and never check downstreams (else cascades); readiness can check critical downstreams (DB, Bedrock).

### B8. Profiles
- **Why asked:** Environment hygiene.
- **Answer:** `spring.profiles.active` selects which `application-{profile}.yml` overlays merge with `application.yml`. Beans can be `@Profile("prod")`-scoped.
- **Your example:** Pariksha picks environment via `unleashContext.environment()` defaulting to `spring.profiles.active` — so flag context inherits env without extra wiring. ConfigNexus has `dev`/`stage`/`prod` profiles with different SSO tenants and tunnel hosts.
- **Follow-up:** "How do you avoid leaking prod creds to dev?" → Secrets in Vault per env; profile only chooses Vault path, never holds creds in YAML. CI gates check no `password:` in committed YAML.

### B9. Bean scopes
- **Why asked:** Stateful vs stateless.
- **Answer:** `singleton` (default), `prototype`, `request`, `session`, `application`, `websocket`. Singletons must be stateless or thread-safe.
- **Your example:** `SpecialistAgentFactory` in Eklavya is a singleton — it's a registry of immutable agent configs. `GooglePayContextHolder` is request-scoped and cleared in the JWT filter's `finally` to avoid leakage across requests.
- **Follow-up:** "Singleton with state?" → Only if state is thread-safe (concurrent collections, atomics) or is a cache with proper sync. Our `MTLSConnectionFactory` cache is `ConcurrentHashMap`-based.

### B10. AOP — concrete real-world example
- **Why asked:** Beyond hello-world.
- **Answer:** Aspects = cross-cutting (logging, security, datasource routing). `@Aspect` + pointcut + advice (`@Before`/`@Around`/`@After`). Backed by JDK proxies (interfaces) or CGLIB (classes).
- **Your example:** `DataSourceAspect` in loan-repayment is a real production AOP: pointcut on `@DataSource(FINFLUX_DB)` annotation; `@Around` advice sets `DataSourceContextHolder.set(FINFLUX_DB)` before, clears after. `TransactionRoutingDataSource` (extends `AbstractRoutingDataSource`) reads ThreadLocal to pick the actual `DataSource`. Powers the read-write split that drove ~10x query improvement on reports.
- **Follow-up:** "Self-invocation gotcha?" → Same as `@Transactional` — internal call bypasses proxy. We split into separate beans whenever an aspect would be needed on an internal call.

---

## SECTION C — MICROSERVICES + ARCHITECTURE (12 Qs)

### C1. Monolith vs microservices — and your honest split
- **Why asked:** Pragmatism check.
- **Answer:** Monolith wins for small teams + tight transactional bounds; microservices win when teams + deploy cadence + scaling profiles diverge. Modular monolith is often the right intermediate.
- **Your example:** zipcredit-backend's `dgl` is a modular monolith with multiple WARs (Spring 4.3, CXF) — we kept it because the consistency requirements are tight. Orchestration + loan-repayment + Eklavya are microservices because they have different teams, deploy cadences, and scale profiles (Eklavya is GPU-compute-bound via Bedrock, others are IO-bound).
- **Follow-up:** "When would you split the monolith?" → When two parts have meaningfully different scale, deploy cadence, or team ownership. Not because microservices are trendy.

### C2. Circuit breaker
- **Why asked:** Resilience pattern.
- **Answer:** Closed → open (after N failures in window) → half-open (probe) → closed. Prevents cascading failure. Resilience4j is the JVM standard post-Hystrix.
- **Your example:** `GpayStatusUpdateImpl` has retry-with-exponential-backoff on 5xx + on `acknowledged=false`; we additionally short-circuit if Google has been failing for >M seconds via a feature-flag-backed kill switch (Pariksha). Resilience4j is wired around the `RestClient` for the hot calls.
- **Follow-up:** "How do you tune thresholds?" → Start from p99 latency + error rate baseline; failure threshold ~50% in a 60s window for non-critical, tighter for money. Always pair with timeout (no breaker can save you from infinite waits).

### C3. Service discovery
- **Why asked:** Topology awareness.
- **Answer:** Client-side (Eureka, Consul) or server-side (k8s DNS + headless services). In k8s, DNS does most of the work — `service.namespace.svc.cluster.local` is your discovery.
- **Your example:** We use Kubernetes DNS + config-driven URLs in Helm values. We don't run Eureka. For external (Google, Digio, partners) we use config + secrets, not discovery — these are stable URLs.
- **Follow-up:** "Eureka vs k8s DNS?" → Eureka is heavier and adds an SPOF if not HA; k8s DNS is "free" if you're already on k8s. We chose simplicity.

### C4. API Gateway / BFF
- **Why asked:** Edge architecture.
- **Answer:** Gateway = AuthN, rate limit, routing, cross-cutting concerns at the edge (Spring Cloud Gateway / nginx / Kong). BFF = backend-for-frontend, shapes responses for one client (web vs mobile).
- **Your example:** Orchestration acts as a BFF — aggregates loan + KYC + repayment for one merchant view. The Pariksha frontend endpoint (`POST /v1/features/flags`) is a BFF pattern — lets the browser call without exposing the backend SDK key.
- **Follow-up:** "Auth at gateway vs service?" → Gateway terminates the public token (JWT validate, mTLS terminate) and forwards a trusted context (e.g. user id header signed internally). Services trust the context but enforce authorization (RBAC) themselves.

### C5. Saga vs Outbox vs 2PC
- **Why asked:** Distributed-tx maturity.
- **Answer:**
  - **2PC**: synchronous, blocking, coordinator SPOF; almost never used in microservices.
  - **Saga**: chain of local transactions with compensations; choreography (events) or orchestration (central coordinator).
  - **Outbox**: write business state + outbox row in same local tx; relay publishes; consumer is idempotent. Solves dual-write reliably.
- **Your example:** Loan-repayment money flow uses an outbox-like pattern — Digio webhook hits Orchestration, we **always 200**, persist `ANachLogsEntity` (the "outbox" row), then a worker processes and updates downstream systems with idempotency keys. We do not use 2PC anywhere.
- **Follow-up:** "Why not Spring Cloud Saga / Axon?" → Adds a framework-shaped runtime to debug; our outbox pattern is a few hundred lines and behaves predictably.

### C6. Kafka — delivery semantics, ordering, duplicates, poison
- **Why asked:** Streaming pattern probe (Airtel WILL ask).
- **Answer (be honest):** "Our current systems use HTTP-callback + persistence + retry rather than Kafka heavily. The patterns translate: at-least-once delivery + idempotent consumer + dedup keys. For ordering, partition by business key (`loanId`/`userId`/`mandateId`). Poison messages → DLQ + bounded retries with backoff + replay tooling."
- **Your example translation:** The `RLock` + `ANachLogsEntity` dedup pattern in Orchestration is logically identical to a Kafka idempotent consumer (`enable.idempotence=true` on producer + dedup-key check in consumer).
- **Follow-up:** "Exactly-once?" → Kafka transactions can give EOS within Kafka boundaries (read-process-write same broker). Across systems you fall back to at-least-once + idempotent consumer.

### C7. Idempotency in REST + webhooks
- **Why asked:** Production-grade integration.
- **Answer:** Provide an idempotency key (`Idempotency-Key` header or business key in payload). Server stores key → result for a TTL; replays return the stored result, not re-execute. For webhooks, dedup on the source's event id + ack first (always 200) + persist + then process.
- **Your example:** `MandateCronServiceImpl.generateUniqueKeyForRetry` builds a per-attempt key for Finflux retries; `findEligibleForDebitRetry` checks idempotency before re-debiting. Orchestration's Digio callback uses `RLock` keyed by `digio:upinach:callback:{mandateId}` plus `ANachLogsEntity` unique constraint as belt-and-braces.
- **Follow-up:** "Idempotency key TTL?" → Long enough to outlast any retry storm (24h typical), short enough to bound storage. We keep 7 days for money flows.

### C8. Caching — invalidation + stampede
- **Why asked:** "Two hard things in CS."
- **Answer:** Strategies: cache-aside (most common), read-through, write-through, write-behind. Invalidation: TTL + event-based eviction. Stampede: use singleflight (one fetch per key, others wait), jittered TTL to avoid synchronized expiry, soft-TTL with background refresh.
- **Your example:** Orchestration uses cache-aside with `RedisAuthCacheServiceImpl` + `CustomRedisCacheManager` for OAuth token caching — drove ~20% latency reduction. We use jittered TTL (`base + random(0, 10%)`) to spread cold-cache rebuilds. Redisson `RLock` for singleflight when we add a new heavy cache.
- **Follow-up:** "How do you invalidate on update?" → Two options: (a) write-through (update cache in same tx, risk of inconsistency on crash), (b) cache-aside + delete on write (simpler, one race window — read can repopulate stale; mitigated with very short TTL). We use (b).

### C9. Rate limiting — where + how
- **Why asked:** Defense + capacity.
- **Answer:** Two layers: (1) edge gateway for traffic shaping (token bucket per IP/API key), (2) service-level for resource protection (Redis counters per user/tenant/endpoint). Token bucket > fixed window (smoother).
- **Your example:** ConfigNexus rate-limits the SQL execution endpoint per user via a Redis counter (sliding window) since one user could fire many EXPLAINs. Pariksha frontend BFF is rate-limited per `x-pariksha-key` token at the gateway.
- **Follow-up:** "What's the failure mode?" → Fail-open for non-critical (limiter outage shouldn't kill the API), fail-closed for money/security flows.

### C10. SLO / SLA / SLI
- **Why asked:** Ops literacy.
- **Answer:** SLI = measurement (e.g. p99 latency, success rate). SLO = internal target (e.g. 99.9% success, p99 < 300ms). SLA = external promise with consequences. Error budget = 1 - SLO; spend it on releases.
- **Your example:** Our GPay notification flow has an internal SLO of 99.95% acked within 5 retries; we allocate the 0.05% error budget to deploy frequency and feature rollouts. The `acknowledged` parsing exists because HTTP 200 alone overstated our SLI.
- **Follow-up:** "What if you blow the budget?" → Freeze new features, stabilize, postmortem. We've done this once for orchestration after a Redis outage cascade.

### C11. Distributed tracing
- **Why asked:** Observability maturity.
- **Answer:** Trace = end-to-end request; span = unit of work. Propagate `traceparent`/`b3` headers across hops. OpenTelemetry is the standard; backends are Tempo/Jaeger/Zipkin/Datadog.
- **Your example:** Orchestration uses `micrometer-tracing-bridge-brave` + Sentry for error tracking. Eklavya additionally tags spans with `agent.specialist`, `agent.tools_called`, and `agent.tokens_used` so we can slice latency by specialist and by tool.
- **Follow-up:** "Sampling?" → Head-sampled in dev (100%), tail-sampled in prod (errors + slow requests always, 10% of others) to control cost.

### C12. mTLS + OAuth2 for inter-service
- **Why asked:** Security depth.
- **Answer:** mTLS = transport-layer mutual identity (cert-based). OAuth2 = application-layer authorization (token-based, can carry scopes/claims). Use both: mTLS gives "is the right service", OAuth2 gives "is allowed to do this action".
- **Your example:** GPay integration runs both — TLS 1.3 mTLS at transport (`MTLSConnectionFactory`), OAuth2 service account (`ServiceAccountCredentials.fromStream`) for Google APIs that need it, plus JWT identity verification on inbound. Three layers because each protects against a different attacker class.
- **Follow-up:** "Cert rotation?" → Cert lifetimes are 90 days; cert-manager / our internal PKI rotates and triggers a rolling restart that re-reads from secrets.

---

## SECTION D — DATABASE + JPA/HIBERNATE (8 Qs)

### D1. N+1 problem
- **Why asked:** Most common JPA bug.
- **Answer:** Loop over parents that lazy-load children → 1 + N queries. Fix: `JOIN FETCH` in JPQL, `@EntityGraph`, batch fetching (`@BatchSize`), or DTO projection.
- **Your example:** Loan-repayment loan-with-charges report originally was N+1 (loan list, then per-loan charges). Fix: `JOIN FETCH` in repository or DTO projection via constructor expression. We monitor with `hibernate.statistics.query_count` per request.
- **Follow-up:** "Why not always eager?" → Eager loads everything always — explodes payload + memory; a list of 1000 loans × all relations = OOM.

### D2. Lazy vs eager
- **Why asked:** ORM modeling.
- **Answer:** Lazy = proxy fetches on access; eager = fetched with parent. Default: `@OneToMany`/`@ManyToMany` lazy, `@OneToOne`/`@ManyToOne` eager (often wrong default). Prefer lazy + explicit fetch where needed.
- **Your example:** `PolicyDetails` (InsureX) keeps relations lazy; we use `@EntityGraph` on the audit-trail repository when we need related entities for the recon report.
- **Follow-up:** "LazyInitializationException?" → Session closed before access. Fix: open-session-in-view (controversial — leaks session into view layer), fetch eagerly for the use case, or use DTO projection at repository level. We prefer DTO projection.

### D3. `@Transactional` pitfalls
- **Why asked:** Common bug.
- **Answer:** Self-invocation bypasses proxy; checked exceptions don't roll back by default (need `rollbackFor`); `private`/`final` methods don't get advised; placement on interface vs class differs by proxy type.
- **Your example:** Loan-repayment uses `DataSourceAspect` aware of `@Transactional` interplay — we ensure datasource is set before the transaction starts. We never put `@Transactional` on a method that calls another `@Transactional` method on `this` — split into separate beans.
- **Follow-up:** "Why no `@Transactional` across services?" → Same answer as Saga/Outbox — distributed tx is fragile; outbox pattern is reliable + observable.

### D4. JPA entity lifecycle + Envers
- **Why asked:** Audit + history.
- **Answer:** Lifecycle: transient → managed (persist) → detached (close session) → removed. Envers (Hibernate Envers) auto-creates `_aud` tables on `@Audited` entities, captures revisions per transaction.
- **Your example:** ConfigNexus, loan-repayment, and InsureX all use `spring-data-envers`. `ConfigAudit` rows in ConfigNexus are written explicitly per change (deployment batch ID), in addition to Envers, because we need application-level fields that Envers doesn't track (rollback SQL, approver, deployer).
- **Follow-up:** "Performance?" → `_aud` writes double the IO; we mitigate by not auditing high-write tables (impressions, traces) and keeping audit on governance entities only.

### D5. SQL — joins, indexes, GROUP BY
- **Why asked:** SQL fluency.
- **Answer:** `INNER`/`LEFT`/`RIGHT`/`FULL OUTER`/`CROSS`. Index on join columns + WHERE filters; avoid functions on indexed columns (kills index). `GROUP BY` after `WHERE`, before `HAVING`. Composite index column order matters (most-selective first or matches WHERE prefix).
- **Your example:** ConfigNexus analytics queries for change-velocity per service had a missing index on `(service_code, created_at)`; query went from 4s → 80ms. UCIN loan reports use native SQL with explicit join hints because Hibernate's plan was suboptimal on Postgres.
- **Follow-up:** "Covering index?" → Index includes all columns the query needs → engine never touches the table. Trade-off: bigger index, slower writes.

### D6. Read-write separation
- **Why asked:** Scale pattern.
- **Answer:** Writes to primary, reads to replicas. ORM-level routing (Spring `AbstractRoutingDataSource`) or proxy-level (ProxySQL, Vitess). Replica lag means "read your writes" can fail — route some reads back to primary or use sticky sessions.
- **Your example:** `TransactionRoutingDataSource` + `DataSourceAspect` + `DataSourceContextHolder` does ORM-level routing in loan-repayment. `DaoAsyncExecutorUtil`'s `slaveDbExecutor` for reads, `masterDbExecutor` for writes. Drove ~10x improvement on reports.
- **Follow-up:** "Read-after-write consistency?" → For money flows we route critical reads to primary explicitly; for reports we accept a few seconds of replica lag and document it.

### D7. Connection pooling
- **Why asked:** Tuning maturity.
- **Answer:** HikariCP is default. Sizing: start from `~10 + cores * 2`; bigger isn't faster (DB CPU saturates first). Set `maxLifetime` < DB's `wait_timeout` to avoid stale connections. Tune `connectionTimeout` (acquire) separately from `idleTimeout`.
- **Your example:** ConfigNexus's `DynamicDataSourceFactory` builds named Hikari pools `CNX-{serviceCode}` — naming surfaces metrics per service in Micrometer. Pool size 10 default per service; tunable per profile.
- **Follow-up:** "Symptoms of pool starvation?" → `HikariPool... Connection not available` errors, p99 latency spike, DB CPU low — that's a pool-size problem, not a DB problem.

### D8. Flyway migrations
- **Why asked:** Schema-as-code.
- **Answer:** Versioned SQL files (`V{n}__{desc}.sql`), runs in order, tracked in `flyway_schema_history`. Use `R__` for repeatable (views, procs). Never edit applied versions; add a new version.
- **Your example:** ConfigNexus has `sql/migration/` with versioned migrations. Eklavya's `eklavya_agent` DB migrates via Flyway too. Older zipcredit uses bespoke `UpgradeScript` SQLs (pre-Flyway era) — we're migrating those to Flyway gradually.
- **Follow-up:** "Schema for zero-downtime deploy?" → Multi-step expand-contract: add column nullable → backfill → switch reads → switch writes → drop old. Never rename in place.

---

## SECTION E — CODING ROUND (5 problems)

### E1. Search in Rotated Sorted Array (Airtel actually asked this)
- **Pattern:** Modified binary search.
- **Approach:** At each mid, one half is sorted; check if target is in the sorted half.
- **Complexity:** O(log n) time, O(1) space.

```java
int search(int[] a, int target) {
    int lo = 0, hi = a.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        if (a[lo] <= a[mid]) {
            if (a[lo] <= target && target < a[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {
            if (a[mid] < target && target <= a[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

### E2. Two-Sum + Anagram + Find Duplicates (warmups)

```java
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (seen.containsKey(need)) return new int[]{seen.get(need), i};
        seen.put(nums[i], i);
    }
    return new int[0];
}

boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] freq = new int[26];
    for (int i = 0; i < s.length(); i++) {
        freq[s.charAt(i) - 'a']++;
        freq[t.charAt(i) - 'a']--;
    }
    for (int f : freq) if (f != 0) return false;
    return true;
}

List<Integer> findDuplicates(int[] nums) {
    List<Integer> dup = new ArrayList<>();
    for (int n : nums) {
        int idx = Math.abs(n) - 1;
        if (nums[idx] < 0) dup.add(idx + 1);
        else nums[idx] = -nums[idx];
    }
    return dup;
}
```

### E3. Design simple REST User service with JPA
- **Layers:** Controller → Service → Repository → Entity.
- **Talking points (live narration during coding):** "I'll use constructor injection via `@RequiredArgsConstructor`, `@RestController`, DTOs at the boundary so I'm not exposing JPA entities, validation via `@Valid` on the request DTO, `ResponseEntity` for status control, and a `@ControllerAdvice` for centralized error handling. For pagination I'd use `Pageable`. For the repo I'd extend `JpaRepository<User, Long>`."

```java
@RestController
@RequestMapping("/v1/users")
@RequiredArgsConstructor
class UserController {
    private final UserService service;

    @PostMapping
    ResponseEntity<UserResponse> create(@Valid @RequestBody UserRequest req) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(req));
    }

    @GetMapping("/{id}")
    UserResponse get(@PathVariable Long id) { return service.get(id); }

    @GetMapping
    Page<UserResponse> list(Pageable p) { return service.list(p); }
}

@Service
@RequiredArgsConstructor
class UserService {
    private final UserRepository repo;

    @Transactional
    UserResponse create(UserRequest req) {
        User u = repo.save(new User(req.email(), req.name()));
        return UserResponse.from(u);
    }

    @Transactional(readOnly = true)
    UserResponse get(Long id) {
        return repo.findById(id).map(UserResponse::from)
            .orElseThrow(() -> new NotFoundException("user " + id));
    }

    @Transactional(readOnly = true)
    Page<UserResponse> list(Pageable p) { return repo.findAll(p).map(UserResponse::from); }
}
```

### E4. SQL — second max salary + dense_rank (Airtel asked these)

```sql
-- Second max salary, no duplicates
SELECT MAX(salary) AS second_max
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Nth highest with DENSE_RANK
SELECT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rk
    FROM employees
) ranked
WHERE rk = 2;

-- Top employee per department
SELECT department_id, employee_id, salary
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn = 1;
```

### E5. Coding template (use this script live)
1. **Clarify** — input range, edge cases (empty, single, duplicates, negatives), expected output for ambiguous cases.
2. **Brute force** — state it out loud with complexity, even if you don't write it.
3. **Optimize** — what data structure or insight reduces it (hash, two-pointer, binary search, monotonic stack, DP).
4. **Code** — clean, named variables, no magic numbers.
5. **Test** — walk through 1 happy path + 1 edge case verbally.
6. **Complexity** — time + space, justify.
7. **Production hardening (bonus)** — overflow (`lo + (hi-lo)/2`), null checks at boundary, immutability where safe.

---

## How To Use This File In The Interview

- For each Q, lead with the **tight answer**, then drop the **project anchor** ("we did this in `ParikshaFlagService`...").
- The project anchor is what separates you from a tutorial-trained candidate — it's verifiable and shows you've shipped.
- If they go deeper, you have follow-ups pre-loaded.
- If they ask something not here, fall back to: clarify → reasoning out loud → trade-off → "in our system we'd do X because Y".
