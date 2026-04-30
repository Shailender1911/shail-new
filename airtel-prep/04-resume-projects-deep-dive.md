# File 4: Resume Projects Deep Dive + All Follow-Up Questions

> The interviewer reads your resume first. Every bullet here will be probed.
> Cross-reference: GPay/Eklavya/ConfigNexus/Pariksha deep dives are in `01-flagship-projects-deep-dive.md`.

---

## DEFENSIVE NARRATIVE — READ FIRST

Three things on your resume need careful framing because the implementation details don't fully back the claim. Have these ready:

| Resume claim | Reality | What you say if pressed |
|---|---|---|
| "Kafka for async processing & webhook delivery" (DLS-NACH, InsureX) | HTTP-callback + persistence + cron retry pattern; Kafka used selectively for partner notification fan-out | "We used Kafka for the partner notification fan-out where multiple downstream consumers needed the same event. The webhook ingest path itself was HTTP-callback + DB persistence + idempotent worker because Digio retries on any non-200." |
| "State machine" (DLS) | Enum + tracker table + trigger service (not Spring State Machine framework) | "Pragmatic state machine — `ApplicationStage` enum with 60+ states, `ApplicationStatusServiceImpl.insertApplicationTracker` deactivates conflicting stages, `TriggerServiceImpl` fires partner-stage event configs. Not a framework — built from primitives because we needed audit and partner-specific automation that frameworks don't model cleanly." |
| "Multi-vendor support (ICICI Lombard, Acko)" (InsureX) | ICICI is the production integration, Acko was scaffolded with the factory pattern as a stub | "Architected for multi-vendor with `InsuranceVendorFactory`; ICICI is in production; Acko was scaffolded to validate the abstraction. Factory keeps adding a new vendor to a few config + impl class additions." |

**Rule:** Lead with the architecture/trade-off; only get into "exactly which Kafka topic" if they push. If they push hard on Kafka internals, pivot: "the platform team owned Kafka cluster ops; I owned producer/consumer code, idempotency, and the dedup pattern."

---

## PROJECT 1 — DLS NACH SERVICE

### One-liner
Microservice owning NACH mandate lifecycle (creation, activation, debit retry) for UPI, API, and Physical NACH types via Digio platform integration.

### Stack
Spring Boot, Java, MySQL, Redis (Redisson), Kafka (selective), Digio API, HMAC-SHA256, JPA/Hibernate.

### Architecture (one breath)
- `MandateController` → `MandateService` → vendor strategies via `MandateStrategyFactory` (UPI / API / Physical).
- Each strategy implements `MandateStrategy` interface — `createMandate`, `processCallback`, `validateSignature`.
- Outbound to Digio: REST client with retry + circuit breaker.
- Inbound webhook: `DigioCallbackController` → HMAC-SHA256 validation (`DigioUtility.validateSignature`) → always-200 ack → persist `ANachLogsEntity` → async process → state transition.
- Multi-tenancy: partner-scoped configs (Digio API keys, callback URLs per partner).
- Backfill jobs: read from upstream sources, replay through normal mandate creation path.

### Your contribution (own it)
- Designed strategy + factory split so a new NACH type drops in as one new strategy class + factory entry — no changes to controller or core service.
- Built HMAC-SHA256 callback validation (security-critical — webhook source trust).
- Built the always-200-ack + DB persistence + async processing pattern (idempotency-safe under Digio's aggressive retry policy).
- Built the multi-tenant backfill pipeline.

### Design decisions + why
- **Strategy + Factory over polymorphism via inheritance** — strategies are stateless, swappable per request, and per-tenant override is a config flag away.
- **HMAC-SHA256 with `equalsIgnoreCase`** — chose hex string compare; if I redid it today I'd use `MessageDigest.isEqual` for constant-time. Honest answer if asked.
- **Always-200 to Digio** — Digio retries on any non-200; ack first, persist, process async. This decouples our processing latency from their retry policy.
- **Kafka used for partner notification fan-out** — when a mandate state changes, multiple downstream services need the event (lending stack, comms, analytics). Kafka over HTTP fan-out for delivery guarantees.

### Trade-offs you owned
- Strategy pattern adds one indirection — accepted for the modularity.
- Always-200 means we ack before validating signature — mitigated by HMAC validation as the first step inside the async worker; bad signatures get dropped + alerted, not retried.
- HMAC `equalsIgnoreCase` is timing-leaky — acceptable for non-money-flow signature; flagged as a tech-debt item.

### Metrics
- "Reduced mandate creation failures via retry workflow" — defendable by pointing at retry config + DLQ pattern.
- Throughput: don't volunteer specific numbers; if pushed, frame as "we sized for X mandates/sec based on partner demand and the bottleneck was Digio's rate limit, not our service."

### 60-sec pitch
> "DLS NACH is a microservice I built from scratch for NACH mandate lifecycle on Digio. Three NACH types — UPI, API, Physical — each behind a Strategy interface, dispatched by a Factory keyed on type + partner config. Inbound Digio webhooks are HMAC-SHA256 validated, always 200 acked, persisted, then processed async — Digio retries on any non-200 so the ack-first pattern is critical. Outbound state changes fan out via Kafka to downstream consumers. Multi-tenant backfill replays through the normal creation path so we don't have two code paths to maintain."

### Follow-up questions — categorized

**Architecture probes**
- Q: "Why Strategy + Factory and not just an `if/switch` on NACH type?"
  → Strategies are stateless beans, easy to test in isolation, easy to add a fourth type without touching the controller. Spring DI gives us per-strategy config (Digio endpoint per type) cleanly.
- Q: "What happens when a new NACH type is added?"
  → New strategy class implementing `MandateStrategy`, registered in `MandateStrategyFactory` (Spring auto-wires beans into the registry), config additions for endpoint + credentials. No core service changes.
- Q: "Why Digio and not direct bank integration?"
  → Bank integrations are 20+ separate certifications; Digio aggregates. Trade-off: vendor lock + dependency on their SLA. Mitigated by abstract `MandateStrategy` so we could add a second aggregator later.

**Code-level probes**
- Q: "Walk me through the HMAC validation."
  → Receive payload + `X-Digio-Signature` header → compute HMAC-SHA256 over raw body using shared secret → hex-encode → compare. We use `equalsIgnoreCase`; the right call is `MessageDigest.isEqual` for constant-time — that's tech debt I'd fix.
- Q: "Where is the shared secret stored?"
  → Per-partner secret in our secret manager; loaded at startup, never logged. Rotation is a config update + restart.
- Q: "How do you handle replay attacks?"
  → Webhook payload includes Digio's event id + timestamp; we dedup on event id (unique constraint on `ANachLogsEntity.event_id`). Stale-timestamp check rejects events older than N minutes.

**Failure mode probes**
- Q: "Digio sends a duplicate callback. What happens?"
  → HMAC validates → persist attempts → unique constraint on event_id rejects → we still 200 (Digio doesn't care, treats as ack). Worker doesn't process the duplicate because the row is already-processed.
- Q: "Digio is down. What happens to our outbound mandate creation?"
  → Retry with exponential backoff (3–5 attempts), then mark `MANDATE_CREATION_FAILED` in tracker, surface to ops, and the customer sees a "try again later" message. No silent loss.
- Q: "Our DB is down when callback arrives."
  → We can't ack-and-persist atomically. Two options: return 500 (Digio retries) — accepted; or queue in-memory (risk of loss on pod restart) — rejected. We chose 500 + Digio retry.
- Q: "What if our async worker crashes mid-processing?"
  → State stays at `RECEIVED`; a sweeper cron picks up rows in `RECEIVED` older than X minutes and re-queues. Idempotent processing makes the re-pick safe.

**Trade-off probes**
- Q: "Why not message queue for the inbound side too?"
  → HTTP webhook from Digio is the contract — we can't change it. Internally we DO queue (DB-as-queue with status flag); we considered Kafka there too but DB-queue gave us auditability + point-in-time recovery for free.
- Q: "Strategy pattern feels heavy for 3 types — overengineering?"
  → Multi-tenant means it's actually 3 types × N partners, each with their own credential and behaviour quirks. Without strategies we'd have nested `if`s on type + partner — we'd be refactoring out of that within months.

**Scale/perf probes**
- Q: "How would you scale this 10x?"
  → Bottleneck is Digio's rate limit, not us. Within our service: stateless, horizontally scalable; DB is the next limit; partition `ANachLogsEntity` by month + index on `event_id` + `mandate_id`.
- Q: "How do you observe production?"
  → Per-strategy metrics (creation success rate, callback latency), Digio API SLA dashboard, alert on `MANDATE_CREATION_FAILED` rate.

**Honest pivots if pushed beyond your comfort zone**
- Kafka deep internals → "The platform team owned Kafka cluster ops; I owned the producer config (idempotence enabled, acks=all), consumer group, and the at-least-once + idempotent consumer pattern. For Kafka tuning at the broker level I'd lean on that team."
- HMAC timing attack → "Yes, `equalsIgnoreCase` is timing-leaky. For new code I'd use `MessageDigest.isEqual`. For this case the attacker can't iteratively probe because we 200 either way and validate inside the async worker, so the practical attack surface is small — but I'd still fix it."

---

## PROJECT 2 — INSUREX SERVICE

### One-liner
Insurance policy management service with multi-vendor support (ICICI Lombard live, Acko scaffolded), two-phase async API flow (Policy + Certificate of Insurance), and audit trail for compliance.

### Stack
Spring Boot, Java, MySQL, JPA/Hibernate (with Envers), CompletableFuture, Kafka (for vendor webhook fan-out), `spring-data-envers`, Apache PDFBox.

### Architecture (one breath)
- `PolicyController` → `InsuranceServiceImpl` → vendor selected via `InsuranceVendorFactory` → vendor impl (`IciciInsuranceServiceImpl`).
- **Two-phase flow:** Phase 1 — issue policy (sync vendor call → policy id returned, status `POLICY_ISSUED`); Phase 2 — fetch COI (Certificate of Insurance, often async at vendor — they push back via webhook or we poll).
- **Cron retry:** `InsuranceServiceImpl.insurancePolicyCron` runs periodically over `STATUS_PENDING_COI` rows, batches by `cronThreadPoolExecutor`, uses `CompletableFuture.allOf().join()` for parallel vendor calls.
- **Audit trail:** `PolicyDetails` is `@Audited` (Hibernate Envers) — every mutation creates a `_aud` row with revision id, supporting compliance asks.
- **Webhook ingestion (vendor → us):** signature validation → ack → persist → process — same pattern as DLS NACH.
- **Kafka:** vendor webhook events fan out to comms, lending, analytics consumers.

### Your contribution
- Designed `InsuranceVendorFactory` so adding a vendor is one impl class + factory registration.
- Built the two-phase Policy + COI flow including state model and cron retry.
- Wired Envers audit on `PolicyDetails` after a compliance review flagged we couldn't reconstruct historical state.
- Built the bounded-pool parallel cron with `CompletableFuture.allOf` + per-future `.exceptionally` so one vendor failure doesn't sink the batch.

### Design decisions + why
- **Two-phase flow** — many insurers issue the policy synchronously but generate the PDF/COI later (sometimes minutes); coupling them in one synchronous flow would block the user-facing request for too long.
- **`CompletableFuture` over `parallelStream`** — custom `cronThreadPoolExecutor` (`InsuranceThreadPoolConfig`) so cron load is isolated from the rest of the JVM (`parallelStream` uses the JVM-wide common ForkJoinPool, unsafe for blocking IO).
- **Envers over hand-rolled audit** — covers all field changes automatically; we add application-level audit only where Envers can't (e.g. who-acted-on-behalf-of in admin tools).
- **Factory + interface for vendor abstraction** — even if Acko is a stub today, the abstraction proved its worth when we discussed adding HDFC ERGO; the conversation was scope-only, no architecture redesign.

### Trade-offs you owned
- Acko stub means the abstraction is unverified by a second real integration — we accept the risk; the Strategy + Factory shape is the same as DLS NACH, which gives us empirical confidence.
- Envers writes double the IO on `PolicyDetails` mutations — acceptable since policy mutation rate is low.
- Cron polling for COI is wasteful when vendors don't push — we considered listening only to webhooks but vendors' webhook reliability isn't 100%; cron is the safety net.

### Metrics
- Audit completeness — Envers gives us 100% field-level auditability.
- COI fetch success rate — tracked via cron success/fail counts.

### 60-sec pitch
> "InsureX manages insurance policies with a vendor-agnostic architecture. `InsuranceVendorFactory` dispatches to the right vendor — ICICI Lombard is live, Acko is scaffolded. The flow is two-phase: policy issuance is synchronous, COI fetch is async because vendors generate PDFs out-of-band — handled by a cron that uses `CompletableFuture.allOf` on a bounded thread pool so one vendor's slowness doesn't block the others. Audit trail is via Hibernate Envers on `PolicyDetails` so compliance can reconstruct history at any revision."

### Follow-up questions — categorized

**Architecture probes**
- Q: "Why two-phase instead of one synchronous flow?"
  → Vendor COI generation can take minutes to hours. Synchronous would either timeout or hold a connection pool slot uselessly. Two-phase decouples — user gets policy immediately, COI arrives when ready, and we notify via comms.
- Q: "What's in `InsuranceVendorFactory`?"
  → A `Map<VendorCode, InsuranceVendorService>` populated by Spring autowiring all `InsuranceVendorService` beans + their `@Component` qualifier. Lookup is O(1) and adding a vendor needs no factory code change.
- Q: "How do you handle vendor-specific fields?"
  → Common fields on `PolicyDetails`; vendor-specific JSON in a `vendor_metadata` column. We don't model every vendor's quirks as columns — proliferates the schema.

**Code-level probes**
- Q: "Walk me through the cron."
  → Read `STATUS_PENDING_COI` rows in batch (pageable). For each, submit a `CompletableFuture.supplyAsync(() -> vendorService.fetchCoi(...), cronThreadPoolExecutor)`. Each future has `.exceptionally(ex -> { log; return null; })`. After submission, `CompletableFuture.allOf(futures).join()` to wait. Failed ones stay in `PENDING_COI` for next run.
- Q: "Why not `parallelStream`?"
  → `parallelStream` uses the common ForkJoinPool — shared with the JVM. Blocking IO on a shared pool causes pool starvation system-wide. Custom executor isolates us.
- Q: "How is the executor sized?"
  → Bounded queue + `CallerRunsPolicy` so overflow back-pressures the cron (caller thread runs the task) instead of dropping work or growing unbounded.

**Failure mode probes**
- Q: "ICICI returns a 500 mid-cron run."
  → That future fails, `.exceptionally` catches it, returns null, logged + metric incremented. Other futures continue. Row stays in `PENDING_COI` — next cron picks it up. After N retries, mark `COI_FETCH_FAILED` and alert ops.
- Q: "Cron fires twice (overlapping runs)."
  → We lock the cron run with a Redis distributed lock (Redisson `RLock`) keyed by cron name. If a second instance starts, it sees the lock and skips. Even without the lock, Envers + idempotent vendor calls would prevent damage; the lock saves us from wasted vendor API calls.
- Q: "Vendor webhook for COI arrives late, after we've moved policy to `COI_FETCH_FAILED`."
  → Webhook handler still updates the row to `COI_RECEIVED` based on policy id — the failed status doesn't lock us out. Audit trail (Envers) shows the transition.

**Trade-off probes**
- Q: "Why `@Audited` everywhere — isn't it overkill?"
  → Only `PolicyDetails` is `@Audited`, not every entity. Compliance specifically asked for full revision history on the policy record itself. Other tables get application-level audit on selected fields.
- Q: "Why MySQL and not a document store for vendor metadata?"
  → JSON column in MySQL gives us flexibility without splitting persistence. Adding Mongo would mean dual-tx complexity for marginal gain.

**Honest pivots**
- "Acko is more scaffolded than live — how confident are you in the abstraction?" → Be direct: "Acko is a stub. The factory + strategy shape is the same as DLS NACH where it does have multiple live integrations. So I'm confident in the abstraction by analogy, not by parallel proof in InsureX itself."

---

## PROJECT 3 — DIGITAL LENDING SUITE (DLS)

### One-liner
End-to-end loan-application orchestration platform with stage-driven workflow, read-write split for reporting, and partner-specific automation that drove a 30% reduction in onboarding time.

### Stack
Spring Boot, Java, MySQL (read replicas), Hibernate, Spring AOP, Redis, Apache PDFBox.

### Architecture (one breath)
- **Pragmatic state machine:** `ApplicationStage` enum (60+ stages — `LEAD_CREATED`, `KYC_INITIATED`, `KYC_VERIFIED`, `BUREAU_PULLED`, `OFFER_GENERATED`, `OFFER_ACCEPTED`, `NACH_INITIATED`, `NACH_ACTIVE`, `DISBURSEMENT_INITIATED`, `DISBURSED`, `LOAN_CLOSED`, plus failure states).
- `ApplicationStatusServiceImpl.insertApplicationTracker(applicationId, stage)` — deactivates conflicting dependent stages, inserts the new tracker row, fires triggers via `TriggerServiceImpl`.
- `TriggerServiceImpl.partnerStageEventConfigMap` — `(partnerCode, targetStage) → List<EventConfig>` driving partner-specific automation (auto-NACH initiation after KYC verified, auto-bureau pull on lead creation, etc.).
- **Read-write split:** `@DataSource(SLAVE_DB)` AOP annotation → `DataSourceAspect` sets `DataSourceContextHolder` (ThreadLocal) → `TransactionRoutingDataSource` (extends `AbstractRoutingDataSource`) selects the actual `DataSource` per query.
- **Batch + rate-limit:** Reporting paths use chunked CSV/JDBC batch (1000-row chunks, 500-row JDBC batch); per-partner rate limits at the API layer.

### Your contribution
- Designed and shipped the enum + tracker + trigger model — explicitly chose this over Spring State Machine framework.
- Built `DataSourceAspect` + `TransactionRoutingDataSource` for read-write split — drove the 10x query improvement on reports.
- Designed `partnerStageEventConfigMap` so a new partner's automation flow is config-driven, not code-driven.
- Owned the rate limiter + batch processing on the reporting endpoints.

### Design decisions + why
- **Why not Spring State Machine?** — it forces you into its lifecycle, makes audit and partner-specific events second-class, and the framework's state-action coupling didn't map to our reality where the same stage transition fires different events for different partners. Pragmatic enum + tracker + trigger gave us audit, partner config, and re-entrancy without the framework tax.
- **Why AOP for datasource routing instead of two `JpaRepository` instances?** — the same business logic often does both (read pre-existing → mutate); two repository hierarchies would force the service layer to know which is which. Annotation + AOP keeps the service layer clean.
- **Why ThreadLocal `DataSourceContextHolder`?** — `AbstractRoutingDataSource` resolves the target lookup key via `determineCurrentLookupKey()` which has no input parameter. ThreadLocal is the standard way to thread the context through.
- **Read replicas only for reads** — writes always go to master to keep replica lag from corrupting state.

### Trade-offs you owned
- ThreadLocal context can leak across threadpool boundaries — we set + clear in `@Around` advice, never rely on auto-cleanup. Tested this explicitly in async paths.
- Replica lag means "read your writes" can fail on slave — for money flows we explicitly route reads to master via `@DataSource(MASTER_DB)`. Documented per endpoint.
- 60+ stages is a lot — but it's the actual state space; collapsing them would lose audit fidelity. Refactor would be UI-side grouping (CATEGORY -> STAGE), not enum-side.

### Metrics
- **30% onboarding time reduction** — defendable: partner-specific triggers automated steps that were previously manual ops handoffs (auto-bureau pull, auto-NACH init after KYC). Before/after partner onboarding median time is the source.
- **10x query improvement** — defendable: read-write split + targeted indexes on `application_tracker(application_id, stage, active)`; report queries that were 4s went to ~300-400ms.
- **40% server load reduction** — batch processing + rate limiting on reports. Defendable as "moved heavy reporting off live request threads onto chunked async with backpressure."

### 60-sec pitch
> "DLS is our loan-application orchestration platform. The state machine is pragmatic — `ApplicationStage` enum with 60+ states, a tracker table for audit, and a trigger service driven by `(partnerCode, targetStage) → eventConfigs` so a new partner's automation is config, not code. I deliberately chose this over Spring State Machine because the framework's coupling didn't model partner-specific events well. The read path uses an `AbstractRoutingDataSource` keyed by a ThreadLocal that an AOP aspect sets from a `@DataSource` annotation — that drove a 10x improvement on reporting queries. Partner triggers cut onboarding time 30%."

### Follow-up questions — categorized

**Architecture probes**
- Q: "Walk me through one application's full flow."
  → Lead created → tracker `LEAD_CREATED` → trigger fires bureau pull → `BUREAU_PULLED` → trigger fires offer generation → `OFFER_GENERATED` → user accepts → `OFFER_ACCEPTED` → KYC → NACH → disbursement. Each transition deactivates conflicting dependent stages (e.g., re-doing KYC deactivates downstream NACH/disbursement for that revision).
- Q: "Why an enum and not a database-driven state list?"
  → Enums are compile-time-safe (typo in a stage name fails build, not runtime); we get exhaustive switch checking; transitions are reviewable in code. Configurable per-partner *behaviour* lives in DB; the state space itself is code.
- Q: "What's the trigger service doing?"
  → `partnerStageEventConfigMap.get((partnerCode, targetStage))` returns a list of `EventConfig` (event type, async/sync, retry policy). Each event is dispatched — could be HTTP call, Kafka event, internal service invocation.
- Q: "Is this Spring State Machine?"
  → No. I deliberately didn't use it. Spring State Machine couples states to actions in a way that didn't model partner-specific behaviour — same stage means different things per partner. Pragmatic enum + tracker + trigger gave us the abstraction we needed without framework coupling.

**Code-level probes**
- Q: "Show me the AOP for datasource routing."
  → `@Around("@annotation(dataSource)")` advice; reads the annotation value; sets `DataSourceContextHolder.set(value)`; proceed; in `finally` clear the context. `TransactionRoutingDataSource.determineCurrentLookupKey()` returns the ThreadLocal value.
- Q: "What if `@DataSource` is missing?"
  → Default lookup key is `MASTER`. Fail-safe to writes-target so we never accidentally read stale and corrupt state.
- Q: "How does this interact with `@Transactional`?"
  → Aspect order matters — datasource aspect runs *before* `@Transactional` so the routing DataSource is set before the transaction is opened. Spring `@Order(Ordered.HIGHEST_PRECEDENCE)` on the datasource aspect ensures this.

**State machine probes**
- Q: "What if a callback arrives for a stage we've already moved past?"
  → `insertApplicationTracker` checks current active stage; if the incoming stage is for an already-superseded version we ignore + log. Idempotent on event id.
- Q: "How do you handle concurrent stage transitions for the same application?"
  → Pessimistic lock on `application_tracker` row (`SELECT ... FOR UPDATE`) within the transition method. Concurrency at the application level is rare (one application is one user) but possible across cron + webhook.
- Q: "How do you test the state machine?"
  → Stage-transition tests per partner config + property-based tests for "any reachable state should be reachable from `LEAD_CREATED`" + ops-driven scenarios from real production sequences.

**Read-write split probes**
- Q: "What if the slave has lag and we read stale?"
  → For money flows we annotate with `@DataSource(MASTER_DB)` so they always go to master. For analytics/reporting, lag is acceptable + we document the SLO.
- Q: "Why not separate JPA repositories per datasource?"
  → Service layer logic is identical; doubling repositories doubles code and makes the read/write decision visible in every method signature. Annotation-driven is cleaner and lets us flip a method's source without API change.

**Trade-off probes**
- Q: "60+ stages sounds like a lot. Aren't you going to drown in stage management?"
  → It's the actual state space; KYC alone is 8-10 sub-stages because regulatory steps have to be auditable. The DSL we built (enum + config-driven triggers) keeps the *behaviour* manageable; the state count just reflects domain complexity.
- Q: "Why not make every stage transition emit a Kafka event?"
  → We do for partner notification fan-out. For internal triggers we use direct method dispatch — Kafka adds latency + ops surface that doesn't earn its keep for in-process transitions.

**Scale probes**
- Q: "10x more applications — what breaks first?"
  → DB writes (tracker inserts) before the service. Mitigation: partition `application_tracker` by `application_id` modulo, async-batch tracker writes for non-critical transitions (analytics events, not money events), index review.
- Q: "What about hotspotting on a single big partner?"
  → Per-partner rate limiting at the API layer; Kafka partition by partner+application id so one partner's spike doesn't backlog other partners' consumers.

---

## PROJECT 4 — LENDING REVAMP

### One-liner
Replaced legacy HTML-to-PDF document generation with native Java (Apache PDFBox), and led Hyperverge integration for Video KYC ensuring RBI compliance.

### Stack
Java, Apache PDFBox, Spring Boot, Hyperverge SDK, REST.

### Architecture (one breath)
- **PDF generation:** `PdfDocumentService` → `PdfTemplateRegistry` (template per document type — sanction letter, KFS, etc.) → PDFBox `PDDocument` build with structured layout (no HTML round-trip).
- Templates parameterized via a context map (loan id, customer name, EMI schedule, terms).
- Output streamed to S3; URL persisted on the loan record.
- **Video KYC:** `HypervergeService` wraps Hyperverge's REST API — initiate session → pass session token to user → poll/webhook for completion → fetch verified PAN/Aadhaar/face-match results → persist to KYC record with audit trail.
- RBI-compliance: video session retained per regulatory window, geo-tagging captured, agent-and-customer face-match required.

### Your contribution
- Designed the PDFBox template abstraction — adding a new document = one template class + registry entry.
- Built the Hyperverge wrapper API surface so other services don't talk to Hyperverge directly (single integration chokepoint).
- Drove the migration from HTML→PDF to native — drove 30% reduction in document generation time.

### Design decisions + why
- **Why PDFBox over the legacy HTML→PDF (wkhtmltopdf / iText / similar)?** — HTML round-trip was slow, fragile (CSS parser quirks per template), and hard to test. PDFBox is verbose but deterministic — the same input gives byte-identical output.
- **Why a wrapper API for Hyperverge?** — vendor abstraction (we could swap to a competitor); single auth/credential surface; logs and metrics in our domain.
- **Templates as code, not as data** — versionable, PR-reviewable, type-safe context map. Trade-off: template changes need a deploy. Acceptable for low-frequency template updates.

### Trade-offs you owned
- PDFBox is verbose — accepted for determinism + speed.
- Templates as code means non-engineers can't tweak — acceptable, those changes are rare and risky enough to want code review.
- Hyperverge polling vs webhook — webhook is preferred but we keep polling as a safety net for missed webhooks.

### Metrics
- **30% PDF generation time reduction** — defendable: HTML round-trip overhead eliminated; PDFBox runs in-memory. Before/after on the same document set.

### 60-sec pitch
> "Lending Revamp had two main pieces. I replaced the legacy HTML-to-PDF pipeline with native Java using Apache PDFBox — drove a 30% speedup because we eliminated the HTML round-trip and the fragility of CSS parsing. Templates live in code, parameterized via a context map, registered in a `PdfTemplateRegistry`. Second, I led the Hyperverge integration for Video KYC — wrapped their SDK behind our own service so other internal services have one chokepoint and we control auth, logging, and audit. RBI compliance was baked in: session retention, geo-tagging, face-match verification."

### Follow-up questions — categorized

**Architecture probes**
- Q: "Why not iText?"
  → iText's commercial licensing for production use is restrictive (AGPL or paid). PDFBox is Apache 2.0 — straightforward.
- Q: "How do you handle template versioning?"
  → Templates are classes; class name encodes version (`SanctionLetterV2`); registry maps `(documentType, version) → template`. Old loans rendered with their original template version are reproducible.
- Q: "Why a registry pattern?"
  → Same shape as the strategy registries in DLS NACH and InsureX — gives us O(1) lookup, Spring autowiring of all template beans, and a uniform shape across services.

**Code-level probes**
- Q: "How does PDFBox build a page?"
  → `PDDocument` → `PDPage` → `PDPageContentStream` for text/graphics commands; explicit font registration; explicit positioning. Verbose but deterministic.
- Q: "How do you test PDF output?"
  → Snapshot tests — render with a known context, hash the bytes, compare. Plus content extraction tests — extract text via PDFBox, assert key fields are present.

**Hyperverge probes**
- Q: "What's the Video KYC flow?"
  → User clicks "Start KYC" → backend initiates Hyperverge session → returns session token to user's app → user goes through video flow in Hyperverge SDK → on completion, Hyperverge webhooks us → we fetch result + persist + transition KYC stage.
- Q: "What's the webhook signature validation?"
  → HMAC-SHA256 over payload using Hyperverge shared secret — same pattern as DLS NACH.
- Q: "What if the webhook is missed?"
  → Cron polls pending KYC sessions older than X minutes against Hyperverge `getSessionStatus`. Defense in depth.

**Compliance probes**
- Q: "What does RBI require for Video KYC?"
  → Live agent presence, face-match between video and submitted ID, geo-location, session retention for the regulated window, audit trail. Hyperverge handles most; we persist results + retention metadata.
- Q: "How do you handle the data retention policy?"
  → Hyperverge stores the video; we persist the verified result + a reference. Retention policy enforced at Hyperverge tier per contract; we hold our DB record per RBI window.

---

## WORK EXPERIENCE BULLETS — DEEP DIVE

### Senior Software Engineer (Dec 2024 – Present)

#### Bullet: Lending Partner Integrations (Google Pay, PhonePe, BharatPe, Paytm, Swiggy)
- **Google Pay** — full deep dive in `01-flagship-projects-deep-dive.md` (JWT + PGP + mTLS + OAuth2). Lead with this if asked about partner integrations.
- **PhonePe / BharatPe / Paytm** — typically OAuth2 + signed payload + webhook callback patterns; less crypto-heavy than GPay. If pressed, frame as "applied the same security stack pattern as GPay scaled appropriately — JWT verification + signed callbacks + idempotent webhook handlers."
- **Swiggy / Meesho (loan-repayment side)** — partner-specific report ingestion via Strategy + Registry keyed on `partner:reportName`; chunked CSV processing.

**Follow-ups:**
- Q: "Which integration is the hardest?" → GPay, hands down. Three-layer security stack, dual JWKS migration, PGP key-ID-keyed cache.
- Q: "Common patterns across partners?" → Strategy + Factory for per-partner code, idempotent webhook ingestion, audit trail per call, secret-per-partner in secret manager.
- Q: "How do you onboard a new partner?" → Implement the partner-specific service interface, register in the factory, add credentials to secret manager, configure routes/rate limits, add monitors. ~1-2 weeks excluding partner-side cert/cred provisioning.

#### Bullet: Redis Caching → 20% API Latency Reduction
- Already covered in `01-flagship-projects-deep-dive.md` under Orchestration BFF. Anchor classes: `RedisAuthCacheServiceImpl`, `CustomRedisCacheManager`.

**Follow-ups:**
- Q: "Cache invalidation strategy?" → Cache-aside + delete-on-write. Short TTL as safety net.
- Q: "Stampede protection?" → Jittered TTL (`base + random(0, 10%)`); singleflight via Redisson `RLock` for hot keys.
- Q: "What did you cache?" → OAuth tokens (highest hit rate), partner config, eligibility metadata. Not user PII, not money state.
- Q: "How did you measure 20%?" → Before/after p50 + p95 latency on the auth-heavy endpoints over a representative week.

#### Bullet: GenAI for Development (Cursor, ChatGPT, Gemini)
- Frame this as **productivity**, not **replacement**: "Used Cursor and ChatGPT for boilerplate generation, code review augmentation, and exploring unfamiliar libraries; reviewed every output — never merged AI code without reading + testing."

**Follow-ups:**
- Q: "Concrete example where AI helped?" → Generated initial Strategy interface skeletons for new partner integrations; generated test fixtures from sample webhook payloads; explored unfamiliar Bouncy Castle PGP API surface.
- Q: "Any AI mistakes you caught?" → Several. Hallucinated method names on third-party SDKs; subtly wrong null-handling. The review step is non-negotiable.
- Q: "How does Eklavya tie to this?" → Eklavya is the production-side: applying LLMs as a product, not just a dev tool. Same discipline — never let the LLM make irreversible decisions; gate sensitive actions; fail-closed.

### Software Engineer (May 2022 – Nov 2024)

#### Bullet: State Machine → 30% Onboarding Time Reduction
- Covered fully under DLS deep dive. Lead with `ApplicationStage` enum + `partnerStageEventConfigMap`.

#### Bullet: Limit Renewals Automation (Application Creation, Bureau Pull, Penny Drop, NACH)
- **What it does:** Existing customers renewing credit limits — instead of full re-application, automate idempotent re-runs of the same pipeline gated by what's already valid (KYC, NACH).
- **Architecture:** Cron + on-demand triggers → `LimitRenewalService` → orchestrates `ApplicationService.create` (idempotent on customer + cycle), `BureauService.pull` (fresh CIBIL), `PennyDropService.verify` (account validity), `NachService.register` (new mandate if needed). Each step is idempotent.
- **Your contribution:** Designed the orchestration flow, the idempotency keys per step, and the partial-failure recovery (if Penny Drop fails, NACH attempt is skipped — don't burn a NACH attempt on an unverified account).

**Follow-ups:**
- Q: "How do you handle a customer who renews twice in a window?"
  → Idempotency key per `(customerId, renewalCycleId)`; second call short-circuits with the existing renewal status.
- Q: "What if Bureau Pull fails — do you abort?"
  → No. Cache the last-known bureau response with TTL; if we have a recent valid pull (within RBI window) we reuse. If not and live pull fails, mark renewal blocked + retry tomorrow.

#### Bullet: Webhook Reliability → 20% Improvement
- **What it does:** Replaced ad-hoc webhook handlers with a unified ingest layer that has signature validation, dedup, ack-first, async processing, and dynamic retry/backoff config per partner.
- **Architecture:** Generic `WebhookIngressController` → partner-specific `WebhookHandler` strategy → unified `WebhookEventLog` for dedup + audit.
- **Dynamic config:** Retry count, backoff, timeout, dedup window per `partnerCode + eventType` in DB → hot-reloadable via config refresh.
- **Your contribution:** Designed the unified ingress + dynamic config layer; pulled the legacy handlers onto this pattern partner by partner.

**Follow-ups:**
- Q: "How did you measure 20% improvement?"
  → Webhook delivery success rate (ratio of successful processing / total received) over a representative window before vs after migration. Improvements came mostly from idempotent retries replacing one-shot processing that dropped on transient failure.
- Q: "What's a 'dynamic configuration'?"
  → Retry policy, timeout, dedup window stored in DB keyed by partner + event type; reloaded on a cron / via Pariksha flag. We can tune per-partner without redeploy.
- Q: "How do you avoid retry storms?"
  → Exponential backoff with jitter; bounded max retries; DLQ for the last failure with manual replay tooling.

---

## OLDER COMPANIES — KEEP TIGHT

### Mobileum (Jul 2021 – May 2022)

#### Bullet: Multi-tenant Discount Schemes → 25% Config Error Reduction
- **What:** Telecom discount engine that previously had per-tenant config sprawl with inconsistent shapes.
- **Action:** Designed a uniform discount schema + validation layer + tenant-scoped overrides; CI-validated configs before deploy.
- **Result:** Configuration errors (deploy failures + runtime mis-applications) dropped 25%.

**Follow-ups:**
- Q: "What was the schema?" → Discount type (flat / percent / tiered), eligibility (product, geo, customer segment), validity window, stacking rules. Validation enforced mutually exclusive fields and required fields per type.
- Q: "How did you migrate existing configs?" → One-time migration script + linter that flagged non-conforming configs; teams fixed them over a sprint.

#### Bullet: Credit-Debit Engine, 1M+ Transactions, 20% Improvement
- **What:** Core ledger handling prepaid balances and adjustments at telecom scale.
- **Action:** Optimized hot-path queries (composite index on `(account_id, txn_time)`), batch-flush write path, removed N+1 on adjacent metadata loads.
- **Result:** Throughput improvement; defendable as "p95 latency improved 20% under same load."

**Follow-ups:**
- Q: "How did you find the bottleneck?" → DB slow query log + APM (whatever Mobileum used); top 3 queries accounted for ~70% of DB CPU. Indexed those.
- Q: "How do you ensure no double debit?" → Transaction-level idempotency key; unique constraint on `(account_id, external_txn_id)`. Same primitive as your fintech work.

### Comviva (Sep 2019 – Jun 2021)

#### Bullet: 40% Response Time Improvement via VPN Call Flow Optimization
- **What:** Telecom service VPN integration — the original flow had unnecessary round-trips.
- **Action:** Profiled, identified redundant lookups, batched calls, cached static config.
- **Result:** Response time dropped ~40%.

**Follow-ups:**
- Q: "What did you cache and what didn't you?" → Static config (service catalogues, VPN endpoint metadata) — yes; dynamic state — no.
- Q: "How long ago was this and what would you do today?" → 5+ years ago. Today: profile with proper APM, cache with explicit invalidation, SLO-driven optimization rather than ad-hoc.

#### Bullet: REST API Development for Billing/Prepaid
- Standard CRUD + integration glue; don't over-claim. If asked, frame as "first production exposure to REST design — taught me API contract discipline that I apply now."

---

## CROSS-CUTTING DEFENSIVE ANSWERS — REHEARSE

### "Walk me through the most complex thing on your resume."
> "Two contenders. The Google Pay term loan integration — three-layer security stack with JWT, PGP, mTLS, all composed correctly, and we passed Google's security review on the first attempt. And Eklavya, our multi-agent AI orchestrator on Spring Boot 3 / Java 17 / AWS Bedrock with PII firewall and per-specialist tool scoping. I'd lead with whichever one connects to what you're building at Airtel."

### "Your resume mentions Kafka — tell me how you used it."
> "Kafka was used for partner notification fan-out — when a mandate or policy state changed, multiple downstream consumers (lending stack, comms, analytics) needed the event with delivery guarantees. Producer config: `acks=all`, `enable.idempotence=true`. Partition key: business id (loanId / mandateId / customerId) so per-entity ordering is preserved. Consumer: at-least-once + idempotent processing via dedup on event id. The webhook ingest path itself was HTTP + DB persistence, not Kafka — the contract with Digio/vendors is HTTP and they retry on any non-200, so ack-first-and-persist fits better than queue-on-receipt."

### "Your resume mentions a state machine — what library did you use?"
> "Pragmatic state machine, no framework. `ApplicationStage` enum with 60+ states, `application_tracker` table for audit, and a trigger service driven by `(partnerCode, targetStage) → eventConfigs`. I deliberately didn't use Spring State Machine because its state-action coupling didn't model partner-specific behaviour cleanly — same target stage means different downstream events for different partners. The pragmatic version gave us audit, partner config, and re-entrancy without the framework tax."

### "Your resume mentions multi-vendor for InsureX — how live are the vendors?"
> "ICICI Lombard is the production integration, fully live. Acko was scaffolded with the factory pattern as a stub to validate the abstraction. The same factory + strategy shape is in production with multiple live integrations in DLS NACH (UPI, API, Physical), so I'm confident in the abstraction by parallel — Acko going live is implementation, not architecture."

### "Tell me a metric you're proud of and how you measured it."
> "10x query improvement on reporting paths in DLS via read-write split. Baseline: representative reporting queries averaged 4s on master under normal load. After: same queries on slave with composite index on `application_tracker(application_id, stage, active)` averaged ~300-400ms. Measured with the slow query log + application metrics over a representative week. The split also kept reporting load off the master, which had the side effect of improving write throughput too."

### "What was your biggest production incident and what did you do?"
> Use STAR 2 from `03-behavioral-managerial-star-stories.md` — Digio NACH callback storms, Redisson RLock + always-200 + ANachLogsEntity dedup.

### "What would you do differently in your projects today?"
- DLS NACH HMAC: switch to `MessageDigest.isEqual` for constant-time compare.
- InsureX: drive Acko to actual production integration to verify the abstraction.
- Lending Revamp: explore native PDF rendering libraries that have shipped since (e.g. Apache PDFBox 3.x improvements, or template-as-data with type-safe binding).
- Webhook reliability: today I'd push harder to consolidate on a single ingress framework with built-in retry, dedup, signature validation rather than per-partner handlers.
- Read-write split: today I'd consider proxy-level (ProxySQL / Vitess) over ORM-level for transparency to the application.

---

## ANTI-PATTERNS (don't do these)

- **Don't say "I built it alone"** for any large project. You collaborated. Frame as "I owned X part of this; the team owned Y; we shipped together."
- **Don't oversell Acko or claim Spring State Machine.** You'll get caught and lose senior signal.
- **Don't volunteer specific Kafka throughput numbers** unless you have hard data. Frame qualitatively + offer the partition strategy.
- **Don't stay shallow when probed.** They're testing depth. If your honest answer is "I touched the surface; the platform team owned the deep ops," say that — it's a senior answer.
- **Don't speak negatively about Digio, vendors, or PayU.** Frame all challenges as engineering trade-offs you navigated.

---

## 30-SECOND PROJECT SHORT-PITCHES (memorize)

| Project | One-line pitch |
|---|---|
| **Eklavya** | "Multi-agent AI orchestrator on Spring Boot 3 / Java 17 / AWS Bedrock — stage-aware router, 9 specialists, 17 read-only tools, PII firewall, MySQL persistence." |
| **Google Pay** | "End-to-end term loan integration — JWT + PGP + mTLS + OAuth2 — passed Google's security review first time." |
| **ConfigNexus** | "GitLab-MR-style config governance with Editor/Reviewer/Admin/Deployer separation, SQL safety, batch-id-keyed rollback, MCP wrapper with JWT pass-through." |
| **Pariksha** | "Unleash-compatible feature flag stack with custom Java SDK fork, frontend BFF + dual-token model, Postgres impression tracking." |
| **DLS NACH** | "From-scratch microservice for NACH lifecycle — Strategy + Factory for UPI/API/Physical, HMAC-validated webhook ingest, always-200 + ack-first." |
| **InsureX** | "Insurance policy management — vendor factory (ICICI live, Acko scaffolded), two-phase Policy + COI flow, parallel cron with `CompletableFuture.allOf`, Envers audit." |
| **DLS** | "Loan orchestration with pragmatic state machine (60+ enum states + tracker + trigger service), AOP-driven read-write split (10x improvement), partner-specific automation (30% onboarding reduction)." |
| **Lending Revamp** | "Native PDFBox PDF generation (30% faster than legacy HTML) + Hyperverge Video KYC wrapper for RBI compliance." |

Pick 2-3 to lead with based on what they probe. If "tell me about your most complex project" — Eklavya or GPay. If "tell me about a backend system you built end to end" — DLS NACH or InsureX. If "tell me about scale" — DLS read-write split + 1M+ transaction Mobileum engine.
