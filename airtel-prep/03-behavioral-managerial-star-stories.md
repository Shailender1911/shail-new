# File 3: Behavioral + Managerial STAR Stories

> Memorize the OPENING + 3 STAR stories cold (Eklavya, Digio incident, GPay security). The rest are backup.
> Speak in 90-120 second blocks. Always end with **outcome + what you learned/owned**.

---

## OPENING — "Tell me about yourself" (2-min spoken script)

**Use this when they say "walk me through your background" or "tell me about yourself".**

> "I'm Shailender, a senior software engineer at PayU Payments — promoted from SDE to senior in late 2024 — with about 5 years of backend experience focused on lending and FinTech.
>
> Most of my recent work sits at two ends of the stack. On the AI side, I designed and built **Eklavya**, our multi-agent orchestrator for merchant lending — Spring Boot 3, Java 17, Spring AI on AWS Bedrock. It's a stage-aware router with 9 specialist agent configs, 17 read-only tools, a PII firewall on every boundary, and conversation persistence in MySQL. It replaced our decision-tree FAQ bot and now serves merchants on Web and WhatsApp.
>
> On the security side, I owned the **Google Pay term loan integration** — a three-layer stack: Nimbus JWT with dual JWKS, PGP sign-encrypt with Bouncy Castle, and TLS 1.3 mTLS via a lazy connection factory. We passed Google's security review on the first attempt and have had zero security incidents.
>
> Beyond those two, I've shipped **ConfigNexus**, a config-governance platform with GitLab-MR-style approval workflows and SQL safety; **Pariksha**, our Unleash-compatible feature flag stack; and the core **Orchestration BFF** with Redis-based webhook deduplication and read-write split datasources that drove a ~10x query improvement on reporting paths.
>
> I'm looking at Airtel because I want to apply that same depth at consumer scale — where reliability, personalization pipelines, and security have to land at a much larger distribution footprint than payments alone."

**Alt opening — if they ask "what are you most proud of?"**
> "Two things. First, designing Eklavya — the orchestrator model, the per-specialist tool permission scoping, and the PII firewall on input/output/tool-JSON. Second, the GPay security stack — getting JWT, PGP, and mTLS to compose cleanly, with a key-ID-keyed PGP cache and an in-memory PEM-to-JKS conversion so we never wrote certs to disk. Both shipped to production and both have passed external review."

**Airtel-specific bridge (slot anywhere in opening):**
> "Airtel operates at consumer scale where session count, latency budgets, and reliability all matter together. My GPay integration work and Eklavya's session+tool layer are the closest analogs in my experience — high-throughput, security-sensitive, and operationally observable from day one."

---

## THE 8 STAR STORIES — Memorize 3-4

### STAR 1. Most challenging / most impactful — **Eklavya**

**Situation.** Our merchant lending product had a v1 conversational helper that was a decision-tree FAQ bot — brittle, didn't carry context, drove a ~25% drop-off in onboarding because merchants couldn't get past stage-specific blockers.

**Task.** Design a real LLM-driven assistant that could (a) understand the merchant's current loan stage, (b) call lending APIs to get real data, (c) never leak PII, and (d) never make irreversible mutations on its own.

**Action.**
- Designed a **stage-aware router** (`StageGroupRouter`) that picks one of nine specialist configs per turn. I deliberately picked **single-specialist-per-turn** over a fan-out-and-vote model — cheaper, lower latency, consistent voice.
- Built **`ToolPermissionScoper`** so each specialist has a tool allow-list. The orchestrator can never accidentally call a sibling's tools, even if the LLM hallucinates a tool name.
- Built **`ConfirmationGate`** for sensitive actions — anything user-impacting requires an explicit human confirmation turn.
- Drove the **PII Firewall** integration on three boundaries: inbound user message, outbound LLM response, and the tool argument + return JSON. Failure mode is fail-closed (mask everything if the firewall errors).
- Picked **Bedrock Sonnet for reasoning + Haiku for routing** — different latency/cost tier per call type.
- Picked **MySQL + Flyway** for conversation persistence over Redis because we need durability for compliance and audit, not just speed.

**Result.** Replaced v1; serves Web + WhatsApp; reduced funnel drop-off through state-aware guidance. Architecture is now the template for additional channels and partners. Internally it's our reference for "how to add an LLM safely into a regulated pipeline."

**Key tension.** Fan-out multi-LLM voting was tempting for accuracy, but I argued — and the team agreed — that we'd pay 3× cost and 2× latency for a small accuracy bump, and the eval showed routing was the actual bottleneck, not specialist choice quality.

---

### STAR 2. Production incident — **Digio NACH callback storms**

**Situation.** Our UPI NACH integration with Digio started producing duplicate state transitions in production. Symptoms: same mandate moving through `INITIATED → ACTIVE` twice, occasionally with downstream side effects (duplicate notifications). Digio retries on any non-200 response, and callbacks were arriving faster than our DB writes were committing.

**Task.** Eliminate duplicates without changing Digio's contract, and make the system safe under callback storms.

**Action.**
- Introduced **Redisson `RLock`** with key `digio:upinach:callback:{mandateId}` to serialize processing per mandate across pods.
- Switched to the **always-200-to-Digio** pattern — ack the webhook first, persist `ANachLogsEntity` (idempotency-keyed), then process asynchronously.
- Added a **graceful degradation** path — if Redis is down, log a `dedup-bypass` warning and proceed; the `ANachLogsEntity` unique constraint on the source event id catches duplicates as a safety net.
- Used `tryLock(waitTime, leaseTime, TimeUnit)` so a dead lock holder can't freeze us forever.

**Result.** Duplicate state transitions dropped to zero. Callback storms — including a 10x spike during a Digio queue catch-up — were absorbed without alerts. The pattern became our standard for any new partner webhook integration.

**Learning.** Always design webhook intake as a two-phase pipeline: ack + persist first (cheap, fast, idempotent), process second. The source's retry policy is your enemy if you skip the ack.

---

### STAR 3. Technical decision you're proud of — **GPay three-layer security stack**

**Situation.** Google mandated four security primitives together for the term loan integration — JWT identity, PGP payload encryption, mTLS transport, and OAuth2 service accounts. Most teams that integrate this fail their first security review.

**Task.** Design and ship the integration so that all four compose cleanly, are operable in production, and pass Google's external review on first attempt.

**Action.**
- Built **`PgpCryptoService`** with cached `PGPPrivateKey` keyed by **key-ID** — survives Google's key rotation window without flag-day cutover. Used Bouncy Castle for primitives (JCE doesn't ship PGP). Base64URL armoring per Google's spec.
- Built **`MTLSConnectionFactory`** as a lazy singleton with `volatile` double-checked init. PEM is converted to JKS **in memory** at runtime — no JKS on disk, simpler secret rotation. TLS 1.3 enforced explicitly.
- Built **`JwtHelper`** with **dual JWKS (v1 + v2)** support so we could ride out Google's JWKS endpoint migration. Mocked-mode flag for non-prod so devs aren't dependent on real Google certs.
- Built **`GpayStatusUpdateImpl`** with retry on 5xx **and** on `acknowledged=false` — because Google can return HTTP 200 with a business error in the body. Treating 200 alone as success caused duplicate notifications during early integration; parsing `acknowledged` fixed it.
- Built **`GPayProxyController`** — single mTLS chokepoint so we don't ship Google certs to every downstream WAR; reduces blast radius.

**Trade-off owned.** Bouncy Castle key rotation requires redeploy (no hot-reload of PGP keys). Acceptable because Google notifies us in advance and we use rolling deploys. I chose this over a more complex hot-reload mechanism because the rotation cadence didn't justify the operational complexity.

**Result.** Passed Google's security review on first attempt. Zero security incidents in production. The PGP cache + mTLS proxy patterns were adopted internally for two other partner integrations.

---

### STAR 4. Conflict / disagreement — **Pariksha frontend token auth**

**Situation.** PM wanted to expose Pariksha flag evaluation directly from the browser using the backend SDK key — said it would be "faster and simpler than building a BFF endpoint."

**Task.** Push back on the security risk without becoming the team blocker, and ship a working solution on schedule.

**Action.**
- Diagnosed why this was unsafe: the backend SDK key grants access to **all** flags and admin operations. Putting it in the browser is equivalent to leaking admin credentials to every visitor.
- Brought **two options** to the meeting, not just a "no":
  - **Option A** — proxy frontend calls through a backend BFF endpoint with a separate, narrowly-scoped frontend token (`x-pariksha-key`). Aligns with Unleash's own security model.
  - **Option B** — use Pariksha's frontend API token model directly (also narrower scope), but requires more SDK glue.
- Showed PM the blast radius of each option side by side. PM picked Option A.
- Shipped `POST /v1/features/flags` BFF endpoint with `authenticateParikshaClient()` matching the frontend key, on schedule.

**Result.** Shipped on time, passed security review, no functional regressions for the frontend team. PM specifically appreciated being shown trade-offs rather than getting a flat "no" — that pattern continued in subsequent design conversations.

**Learning.** "No" alone is not engineering input. Always bring two viable options + the trade-off so the decision-maker stays in the driver's seat.

---

### STAR 5. Mentoring / leading without authority — **ConfigNexus separation of duties**

**Situation.** The team was initially going to merge the Approver + Deployer roles in ConfigNexus to "keep the workflow simple." I argued this would fail compliance and create real production risk.

**Task.** Convince the team — without formal authority — to adopt a 4-role separation (Editor / Reviewer / Admin / Deployer).

**Action.**
- Walked through three concrete production-outage scenarios where one person who can both approve and deploy would be able to push a change without independent oversight (single point of failure for governance).
- Pushed for the **4-role split** with a separate **Deployer** role — whoever approved cannot deploy.
- Acknowledged the friction and brought the **Force-Merge** path as the escape hatch — it's allowed but requires (a) an incident ticket ID and (b) a business-impact justification, both stored on the audit row.
- Coded the audit trail and the Force-Merge UI flow myself so the team didn't see it as additional work for them.

**Result.** Adopted unanimously. The audit trail has since withstood multiple compliance reviews and one internal audit. Force-Merge has been used twice in 18 months — both for genuine incidents, both fully audit-trail compliant.

**Learning.** When you're pushing for governance, ship the operability story (escape hatch, audit, low-friction default) along with the principle. Otherwise the principle gets traded away the first time it's inconvenient.

---

### STAR 6. Biggest mistake / retrospective — **Mandate cron retry without idempotency**

**Situation.** Early `MandateCronServiceImpl` retry path reused the same transaction id on retries. Under Finflux intermittent failures, we double-booked mandates — real merchants saw duplicate debit attempts.

**Task.** Stop the bleeding, fix the data, and prevent recurrence.

**Action.**
- Owned the incident response: stopped the cron immediately on detection.
- Wrote a reconciliation query against Finflux + our local DB to identify the duplicate-debit set, refunded affected merchants.
- Introduced **`generateUniqueKeyForRetry`** — per-attempt unique key so retries never collide with prior attempts.
- Added **`findEligibleForDebitRetry`** with explicit idempotency checks before the retry call.
- Wrote a postmortem that became internal reference for "async integration retry design".

**Result.** Zero duplicate debits since the fix. The unique-key-per-retry pattern is now standard in any new partner integration we build.

**Learning.** Assume duplication in any async integration. Every retry must be idempotent at the protocol level — not just "we hope the partner deduplicates."

---

### STAR 7. Why leaving PayU / Why Airtel

**Why leaving PayU (non-negative phrasing):**
> "PayU has been a strong run — five years, promotion to senior, depth across lending, AI orchestration with Eklavya, and governance with ConfigNexus. The org is in a good place. What I'm looking for next is consumer-scale impact — where the work I do touches a much larger user base, and where reliability, personalization, and security all compose at scale. PayU's footprint is meaningful but it's primarily B2B and partner-driven; I want my next chapter to be at the consumer edge."

**Why Airtel:**
> "Airtel sits at the intersection of telecom, payments (Airtel Payments Bank), digital services, and now AI initiatives — the user base is at a scale where the engineering bar on latency, reliability, and security is genuinely different. The kind of work I've done — multi-agent AI orchestration, security-heavy integrations like GPay, governance platforms — applies directly. Plus Airtel's recent push on engineering quality and India-scale platforms is a place I want to contribute to."

---

### STAR 8. 3-5 year vision

> "I want to be a strong individual contributor with staff-level influence — owning a platform area end-to-end (architecture, reliability, mentoring), with a clear track record of measurable customer outcomes. Not necessarily a people-manager track. I want to mentor 2-3 strong engineers, set reliability standards that the broader team adopts, and ship work that gets pointed to as 'the way we do this here.' The kind of work I did with ConfigNexus governance and Eklavya's architecture is the shape — I want to do that in 3-5 more domains."

---

## MANAGERIAL TECHNICAL — TRADE-OFF DISCUSSIONS (8 topics)

> R2/managerial rounds are less "what's a HashMap" and more "walk me through how you'd design X under these constraints". Have an opinion + a real-world anchor for each.

### M1. Strong vs Eventual Consistency
- **Opinion:** Strong on the write path for money/money-adjacent state; eventual on the read path for analytics, dashboards, search.
- **Real anchor:** Loan-repayment write path is single-DB transactional (strong); reporting paths read from replicas with documented lag (eventual). The split is explicit, not accidental.
- **When asked "could you do everything strong?"** → Yes, but you trade write throughput, replica utility, and operational complexity. We chose to make consistency a per-flow design decision.

### M2. Distributed Transactions: 2PC vs Saga vs Outbox
- **Opinion:** **Outbox** is the default for cross-service writes in our shop. **Saga** when you need explicit compensation steps. **2PC** almost never — coordinator SPOF, blocking, doesn't compose with any modern infra.
- **Real anchor:** Orchestration's Digio webhook → `ANachLogsEntity` → async processor is logically an outbox. The dual-write problem (webhook ack + DB persist) is solved by ack-first, persist-as-the-outbox-row, process-from-outbox.
- **When asked "what if your outbox processor falls behind?"** → Bounded queue lag SLO + alert; can scale horizontally because each row is keyed by business id (no cross-row ordering needed within the SLO).

### M3. Kafka — Duplicates, Ordering, Poison Messages
- **Opinion:** Always-design-for-at-least-once + idempotent consumer + business-key partitioning. EOS is for read-process-write within Kafka, not for cross-system flows.
- **Patterns:**
  - **Duplicates:** consumer-side dedup via `(topic, partition, offset)` or business id; idempotent processing.
  - **Ordering:** partition by `loanId` / `userId` / `mandateId` so per-entity order is preserved; cross-entity order is rarely needed.
  - **Poison:** bounded retries with exponential backoff → DLQ topic; replay tooling to reprocess after fix; alert on DLQ growth rate.
- **Real anchor:** Our HTTP-callback + `RLock` + `ANachLogsEntity` dedup pattern is the same idea, just over HTTP instead of Kafka. The translation to Kafka would be straightforward.

### M4. Idempotency Protocol Design
- **Opinion:** Idempotency key is a contract, not an implementation detail. Document it, version it, test it.
- **Pattern:**
  - Client supplies `Idempotency-Key` header (or business key in payload).
  - Server stores key → result with TTL (24h+ for money, 7d for our flows).
  - Replays return the stored result, do not re-execute.
  - For webhooks: ack first + persist source event id (unique constraint) + process.
- **Real anchor:** `MandateCronServiceImpl.generateUniqueKeyForRetry` + Orchestration's `RLock` + `ANachLogsEntity`.

### M5. Caching — Invalidation + Stampede
- **Opinion:** Cache-aside is the default; jittered TTL + singleflight prevents the dragon (stampede on cold cache).
- **Patterns:**
  - **Cache-aside** + delete-on-write (accept short stale window over write-through complexity).
  - **Jittered TTL** (`base + random(0, 10%)`) prevents synchronized expiry storms.
  - **Singleflight** via `Redisson RLock` for heavy-cost rebuilds — one process rebuilds, others wait.
  - **Soft TTL** + background refresh for hot keys you can't afford a miss on.
- **Real anchor:** `RedisAuthCacheServiceImpl` + `CustomRedisCacheManager` in Orchestration drove ~20% latency reduction. We added jittered TTL after a deploy-time cold-cache stampede taught us the lesson.

### M6. Rate Limiting — Where + How
- **Opinion:** Two layers — gateway (traffic shaping per IP/API key) + service (resource protection per user/tenant/endpoint). Token bucket > fixed window.
- **Real anchor:** ConfigNexus rate-limits SQL execution per user via Redis sliding window. Pariksha frontend BFF rate-limited per `x-pariksha-key` at the gateway.
- **Failure mode:** Fail-open for non-critical (limiter outage shouldn't kill the API), fail-closed for money/security flows.

### M7. `@Transactional` in microservices — and why we don't
- **Opinion:** Don't reach for it across services. Use outbox + idempotent consumer + saga for compensation.
- **Real anchor:** We deliberately do not use distributed transactions in loan-repayment. Money flow is outbox-pattern; recon job catches the rare drift.
- **When asked "what about Spring Cloud Saga / Axon?"** → Adds runtime to debug; our outbox is a few hundred lines and behaves predictably. Framework overhead not justified.

### M8. Reliability under load — your toolkit
- **Opinion:** Layer the defenses — they compose. No single mechanism saves you.
- **Toolkit:**
  - **SLO + error budget** to drive priority.
  - **Timeouts** on every outbound call (network + read separately).
  - **Retries with jitter + bounded** (never infinite, never aligned).
  - **Circuit breaker** with health-aware probes.
  - **Bulkheads** (isolated thread pools per downstream).
  - **Rate limits** at edge + per-resource.
  - **Feature flags (Pariksha)** for shedding load — kill a non-critical feature instead of dropping all traffic.
  - **Graceful degradation** — example: Orchestration runs without Redis dedup if Redis is down (`ANachLogsEntity` unique constraint catches duplicates).

---

## QUESTIONS YOU SHOULD ASK THEM (Closing — 3 only)

> Ask 2-3 maximum. Showing prep + curiosity, not interrogating.

1. **"What does the on-call rotation look like for this team, and what are the top reliability risks today?"** — Signals you take ops seriously.
2. **"What does success look like for the person in this role at the 6-month mark?"** — Forces them to articulate what they actually want.
3. **"How does the team approach trade-off conversations — feature speed vs reliability vs technical debt? Who drives the call?"** — Signals you understand engineering management dynamics.

**Optional 4th if vibe is good:**
4. **"What's a recent technical decision the team made that you're proud of, and one you wish you'd done differently?"** — Invites them to be candid; tells you a lot about engineering culture.

---

## ENERGY + LOGISTICS (the night before / morning of)

- **Join 5-10 minutes early.** Test camera, mic, screen-share. Have water.
- **Have 2-3 metrics ready** to drop naturally:
  - "~20% API latency reduction via Redis caching in Orchestration"
  - "~10x query improvement via read-write split in loan-repayment"
  - "~30% onboarding time reduction via automated downstream triggers in DLS"
  - "passed Google security review first time + zero security incidents"
- **Speak to trade-offs, not features.** Anyone can list features. Trade-off articulation is what separates senior from mid.
- **For coding round:** clarify → brute → optimize → code → test → complexity. Narrate the whole way.
- **If you don't know an answer:** say so, then reason out loud from first principles. "I haven't shipped X, but I'd think about it as Y because Z" is a senior answer.
- **End every story with the outcome + what you learned.** Without those two beats, the STAR is incomplete.

---

## ANTI-PATTERNS TO AVOID

- **Don't lead with InsureX or DLS-NACH.** They're small services and weaken your seniority signal.
- **Don't overclaim Kafka.** Be honest: HTTP-callback + persistence + retry is your real pattern; map it to Kafka semantics if asked.
- **Don't say "Spring State Machine".** Your DLS state machine is enum + tracker + trigger service — say that.
- **Don't bash PayU.** Phrase the move as growth-toward, not away-from.
- **Don't fabricate metrics you can't defend.** The four real metrics above are enough.
- **Don't memorize this file word-for-word.** Memorize the structure + the project anchors. Speak naturally.

---

## FINAL CHECK — 60-SECOND SELF-PITCH (rehearse 3x out loud tonight)

> "Five years backend, currently senior engineer at PayU. Two flagships: Eklavya — multi-agent AI orchestrator on Spring Boot 3, Java 17, AWS Bedrock, with PII firewall and per-specialist tool scoping; and Google Pay term loan — three-layer security stack with JWT, PGP, and TLS 1.3 mTLS, passed Google's security review first time. Beyond those, ConfigNexus governance platform and Pariksha feature flags. Looking at Airtel for consumer-scale impact where reliability, security, and personalization compose at a much larger footprint than payments alone."

You're ready. Sleep early.
