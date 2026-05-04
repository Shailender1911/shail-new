# File 7: Mock Interview Script — R1 + Managerial Simulation

> Practice this with someone tomorrow morning. Or read it out loud solo. Time yourself.
> R1 ≈ 60 min, Managerial ≈ 45 min.

---

## ROUND 1 — TECHNICAL SCREEN (60 min)

### Phase A: Intro + Resume Walk (8 min)

**Interviewer:** "Tell me about yourself."

**You (90-120 sec):** *Use the opening from the cheatsheet — Eklavya + GPay as headline.*

**Interviewer:** "Walk me through your most challenging project."

**You (3 min):** Lead with **Eklavya**. Architecture (45s) → your contribution (45s) → key design decision: single-specialist-per-turn over fan-out (45s) → trade-off you owned (45s).

**Likely follow-ups:**
- "How do you stop the LLM from calling random tools?" → `ToolPermissionScoper` per specialist + `ConfirmationGate` for sensitive actions.
- "PII handling?" → `PiiFirewall` on input/output/tool-JSON; fail-closed.
- "Why MySQL not Redis for sessions?" → Durability + audit trail.

**If they go deep on Eklavya, you can spend 8-10 min here. If they pivot, follow.**

---

### Phase B: Core Java (12 min)

**Q1: "Explain HashMap internals."**

**You (60 sec):** "Array of buckets, index = `(n-1) & hash`. Collisions go into a linked list, treeified to red-black at threshold 8 with capacity ≥ 64, untreeified at 6. Default load factor 0.75 — resize doubles capacity and rehashes. Not thread-safe — for concurrent use, ConcurrentHashMap, which uses CAS plus per-bucket synchronized in Java 8+; reads are lock-free."

**Likely follow-up:** "Worst-case complexity?" → "O(log n) since Java 8 due to treeify; O(n) before. We control input keys to be UUIDs or enums to avoid pathological cases. Real example — `ParikshaFlagService` uses size-hinted maps so we don't churn on resize."

**Q2: "When would you use `volatile`?"**

**You (60 sec):** "Visibility + ordering, not atomicity. Concrete example: `MTLSConnectionFactory` — lazy singleton SSLContext with `volatile` field plus double-checked locking. Without volatile, a partially constructed SSLContext could be observed by a second thread reading without the lock."

**Q3: "Explain CompletableFuture."**

**You (60 sec):** "Async composition. `supplyAsync` runs on a pool, `thenApply`/`thenCompose` chain transformations, `allOf`/`anyOf` aggregate, `exceptionally`/`handle` for errors. Concrete example: `InsuranceServiceImpl.insurancePolicyCron` — batches policy fetches in parallel on a custom `cronThreadPoolExecutor`, each future has `.exceptionally` so one bad policy doesn't sink the batch, then `CompletableFuture.allOf(futures).join()` to wait. We never use `parallelStream` for blocking IO because it shares the JVM common ForkJoinPool."

**Q4: "Tell me about a deadlock you've debugged."**

**You (90 sec):** *If you have a real one, lead with that. Otherwise:* "Deadlock specifically — no, but a related issue: in loan-repayment we considered XA across `arthmate_db` and `finflux_db`, evaluated the deadlock surface, and rejected XA in favor of an outbox-like pattern — write to one DB transactionally, async sync to the other. Multi-datasource routing in our codebase is per-DB transactions only — we never hold two DB transactions at once. For detection in production we'd use `jstack`, `jvm.threads.states`, and `ThreadMXBean.findDeadlockedThreads`."

---

### Phase C: Spring Boot (10 min)

**Q1: "What does `@SpringBootApplication` actually do?"**

**You (45 sec):** "Combines `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Auto-config reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` and applies `@ConditionalOn*` checks before instantiating. Concrete example: `ParikshaConfig` is wrapped in `@ConditionalOnProperty(prefix=\"pariksha\", name=\"enabled\", havingValue=\"true\")` — opt-in per environment."

**Q2: "Constructor vs field injection — which and why?"**

**You (45 sec):** "Constructor. Immutable fields with `final`, fail-fast at startup, easy to test without reflection. We use Lombok `@RequiredArgsConstructor` everywhere — `ParikshaFlagService`, `GPayService`. No `@Autowired` on fields in new code. Circular dependencies show up at startup as a code smell, not get hidden by `@Lazy` workaround."

**Q3: "How does `@Transactional` work?"**

**You (90 sec):** "Spring proxies the bean and intercepts the method to open/commit/rollback a transaction. Default propagation is REQUIRED — join existing or create new. Common gotchas: self-invocation (`this.method()`) bypasses the proxy; checked exceptions don't trigger rollback by default — need `rollbackFor`; only public methods are advised by default. In our codebase: `MandateCronServiceImpl` retry uses `REQUIRES_NEW` so the retry log persists even if the outer transaction rolls back. We deliberately don't use `@Transactional` across microservices — outbox-like pattern instead."

**Q4: "Real example of AOP you've shipped."**

**You (90 sec):** "`DataSourceAspect` in loan-repayment. Pointcut on `@DataSource(SLAVE_DB)` annotation; `@Around` advice sets a ThreadLocal `DataSourceContextHolder`; `TransactionRoutingDataSource` extends `AbstractRoutingDataSource` and reads ThreadLocal in `determineCurrentLookupKey`. Powers our read-write split — drove ~10x improvement on reporting queries. The aspect runs at `Ordered.HIGHEST_PRECEDENCE` so the routing DataSource is set before the transaction is opened."

---

### Phase D: Microservices + Architecture (12 min)

**Q1: "Monolith vs microservices — when?"**

**You (60 sec):** "Pragmatically — monolith wins for tight transactional bounds and small teams; microservices win when teams, deploy cadence, and scale profiles diverge. Modular monolith is often the right intermediate. In our shop: zipcredit-backend is a modular monolith with multiple WARs because consistency is tight. Orchestration, loan-repayment, and Eklavya are microservices because they have different teams and scale profiles — Eklavya is GPU-compute-bound via Bedrock, others are IO-bound."

**Q2: "How do you handle distributed transactions?"**

**You (90 sec):** "Outbox pattern by default. Saga when explicit compensation is needed. 2PC almost never. Concrete example: Orchestration's Digio webhook handler — we always 200 to Digio first, persist `ANachLogsEntity` (the outbox row), then async process. The dual-write problem is solved by ack-first-and-persist + idempotent worker. The same pattern translates to Kafka: producer with `acks=all` + `enable.idempotence=true`, consumer with dedup on event id. We don't use Spring Cloud Saga — adds runtime to debug; outbox is a few hundred lines and behaves predictably."

**Q3: "How do you make webhooks idempotent?"**

**You (60 sec):** "Source provides an event id; we dedup on it via unique constraint plus a Redis distributed lock for in-flight protection. Concrete: `ANachLogsEntity` has a unique constraint on `event_id`; Orchestration uses Redisson `RLock` with key `digio:upinach:callback:{mandateId}` to serialize per-mandate processing across pods. Always 200 to Digio so they don't retry-storm us. Graceful degradation: if Redis is down, we proceed without lock and rely on the unique constraint as the safety net."

**Q4: "Cache stampede protection?"**

**You (60 sec):** "Three layers. Cache-aside is the default; jittered TTL — `base + random(0, 10%)` — prevents synchronized expiry; for hot keys we use singleflight via Redisson `RLock` so only one process rebuilds, others wait. Soft-TTL with background refresh for the hottest keys. We added jittered TTL after a deploy-time cold-cache stampede taught us the lesson — drove 20% latency reduction overall via `RedisAuthCacheServiceImpl`."

---

### Phase E: Coding Round (15 min)

**Likely problems (rank order):**
1. Search in rotated sorted array (binary search)
2. Two Sum / valid anagram / find duplicates (warmup)
3. SQL: 2nd highest salary
4. Design a REST endpoint with JPA

**Live narration template:**

> "OK, so let me clarify first — [restate problem]. Inputs are [X], output [Y], edges I'm thinking are [empty/single/duplicates/negatives]. *(pause for confirmation)*
>
> Brute force would be [approach]; that's O([X]). I think we can do better with [data structure / pattern] — let me code that up."

*Narrate as you code. State variables clearly. After:*

> "Let me trace this with [happy case]: [walk through]. And edge case [empty array]: [walk through]. Time complexity [X], space [Y]. For production hardening, I'd watch for [overflow / null / immutability]."

---

### Phase F: Wrap-up (3 min)

**Interviewer:** "Any questions for me?"

**You (ask 1-2):**
1. "What does the on-call rotation look like for this team, and what are the top reliability risks today?"
2. "What does success for this role look like at the 6-month mark?"

---

## ROUND 2 — MANAGERIAL (45 min)

### Phase A: Intro + Why Airtel (8 min)

**Manager:** "Tell me about yourself."

**You:** *Same opening as R1 but lean into outcomes and breadth, less into technical depth.*

**Manager:** "Why are you looking to leave PayU?"

**You (60 sec):** *Use the resume STAR 7 script.*
> "PayU has been a strong run — five years, promoted to senior, depth across lending, AI orchestration with Eklavya, and governance with ConfigNexus. The org is in a good place. What I'm looking for next is consumer-scale impact — where the work I do touches a much larger user base, and where reliability, personalization, and security all compose at scale."

**Manager:** "Why Airtel specifically?"

**You (60 sec):**
> "Airtel sits at the intersection of telecom, payments, digital services, and AI initiatives — at a user base scale where the engineering bar on latency, reliability, and security is genuinely different. The work I've done — multi-agent AI orchestration with Eklavya, security-heavy integrations like GPay, governance platforms — applies directly. I want to contribute to India-scale systems and grow into staff-level platform ownership."

---

### Phase B: System Design (15 min)

**Likely prompts (rank order):**
1. Design a notification service (OTP-grade reliability)
2. Design a payment system / how would you avoid double-charging
3. Design a feature flag system → **pivot to Pariksha**
4. Design a config governance / approval workflow → **pivot to ConfigNexus**
5. Design a chatbot / AI assistant for X → **pivot to Eklavya**

**Your script (regardless of prompt):**

> "Before I jump in — can I clarify scale, consistency requirements, and read:write ratio?"
>
> *(pause)*
>
> "OK. Functional reqs are [3 items]. Non-functional: [latency / availability / consistency / durability]. Capacity-wise, at [X QPS] we'd need [Y storage/year]. API surface: [POST / GET / webhook]. Data model: [primary tables + indexes]. High level: [ingest → queue → workers → store]. Two areas worth deep diving: [hot key handling] and [failure mode]. *(do these.)*
>
> Bottlenecks I'd watch first: [X]. Scaling 10x: [partition by Y]. Trade-offs I made: chose [A over B because Z]. Anywhere you want me to go deeper?"

---

### Phase C: Behavioral (15 min)

**Likely prompts:**

**1. "Tell me about a time you disagreed with someone."**
→ STAR 4: Pariksha frontend token. PM wanted backend SDK key in browser → security risk → I brought 2 options + trade-offs → picked Option A → shipped on schedule + passed security review. **Closing learning:** "No alone isn't engineering input. Bring 2 viable options + trade-offs."

**2. "Tell me about a production incident."**
→ STAR 2: Digio NACH callback storms. Redisson `RLock` + always-200 + `ANachLogsEntity` dedup + graceful degradation. **Closing learning:** "Always design webhook intake as two-phase: ack + persist first, process second."

**3. "Tell me about a mistake."**
→ STAR 6: Mandate cron retry without idempotency. Owned the incident, refunded affected merchants, fixed with `generateUniqueKeyForRetry` + `findEligibleForDebitRetry`. **Closing learning:** "Assume duplication in any async integration. Every retry must be idempotent at the protocol level."

**4. "Tell me about leading without authority."**
→ STAR 5: ConfigNexus 4-role split. Walked through 3 outage scenarios; pushed for Editor/Reviewer/Admin/Deployer; brought Force-Merge as escape hatch with audit. Adopted unanimously. **Closing:** "Pair governance with operability — escape hatch + audit + low-friction default."

**5. "What's your biggest weakness / area of growth?"**
→ "I default to ownership-mode in incidents — I want the bug fixed, the recon done, the post-mortem written. The growth area is delegation: pulling in others earlier so the whole team builds the muscle, not just me. I've started doing this consciously on Eklavya — pairing with juniors on the PII firewall extensions instead of doing it myself."

---

### Phase D: Trade-off Discussion (5 min)

**Manager:** "We have a service that's getting slow under load. Walk me through how you'd approach it."

**You (3 min):**
> "First — I want metrics, not guesses. Look at p50/p95/p99 latency, error rate, throughput. Identify the breaking point — is it CPU, DB, downstream, network? APM tooling and a flame graph for hot paths.
>
> Once I know the bottleneck, the toolkit is layered. Cache the hot path with cache-aside + jittered TTL — that's how we got 20% latency reduction in Orchestration. If it's DB-bound, read-write split — that drove 10x in loan-repayment via `TransactionRoutingDataSource`. If it's downstream-bound, circuit breaker + bulkheads. If it's a hot key, partition or replicate.
>
> Reliability under load isn't one mechanism — it's layered: SLO + timeouts + retries with jitter + bulkheads + rate limits + feature flags for shedding. We use Pariksha to kill non-critical features instead of dropping all traffic."

---

### Phase E: Vision + Wrap-up (2 min)

**Manager:** "Where do you see yourself in 3-5 years?"

**You (45 sec):**
> "Strong individual contributor with staff-level influence — owning a platform area end-to-end with reliability and architectural ownership. Mentor 2-3 strong engineers. Set standards the broader team adopts. Ship work that gets pointed to as 'the way we do this here.' Not the people-manager track — IC depth + influence."

**Manager:** "Any questions?"

**You:**
1. "How does the team approach trade-off conversations — feature speed vs reliability vs technical debt? Who drives the call?"
2. "What's the team's biggest technical bet for the next 6 months?"

---

## TIMING DRILL — TONIGHT

Set a 60-min timer. Open this file. Walk through R1 phases A-F end-to-end out loud, no skipping. Then 45-min for managerial.

If you stumble on a question, mark it. Re-rehearse the answer. Run again.

---

## RED FLAG ANSWERS (if you say these, STOP and rephrase)

| Don't say | Say instead |
|---|---|
| "I don't know." | "I haven't shipped that, but I'd reason about it as…" |
| "It depends." (alone) | "It depends on X. If X, I'd do A; if Y, I'd do B; for our context I'd lean A because Z." |
| "We just used Spring." | "We chose Spring because [X]; we considered [Y] but ruled it out because [Z]." |
| "It worked fine." | "Outcome was [metric]; what I learned was [Y]." |
| "Kafka." (vague) | "Kafka with `acks=all`, `enable.idempotence=true`, partition by business key, at-least-once consumer with dedup on event id." |
| "Microservices are better." | "We chose microservices for [X]; we run a modular monolith for [Y] where consistency is tight." |
