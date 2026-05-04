# File 6: One-Page Cheatsheet — Night-Before Skim

> Print this. Read it 10 minutes before joining. Nothing else.

---

## OPENING SCRIPT (memorized, 90 sec)

> "I'm Shailender, senior software engineer at PayU, ~5 years backend in lending FinTech. Two flagships: **Eklavya** — multi-agent AI orchestrator on Spring Boot 3 / Java 17 / AWS Bedrock with PII firewall and per-specialist tool scoping; and **Google Pay term loan** — three-layer security stack with JWT, PGP, and TLS 1.3 mTLS, passed Google's security review first time. Beyond those, ConfigNexus governance platform and Pariksha feature flags. Looking at Airtel for consumer-scale impact."

---

## THE 4 NUMBERS (drop naturally)

- **20%** API latency reduction → Orchestration Redis caching (`RedisAuthCacheServiceImpl`)
- **10x** query improvement → loan-repayment read-write split (`TransactionRoutingDataSource` + `DataSourceAspect`)
- **30%** onboarding time reduction → DLS state machine + partner triggers
- **First-time pass + zero security incidents** → GPay security review

---

## CLASS NAMES TO DROP (proves you've shipped)

- `AgentOrchestrator`, `StageGroupRouter`, `SpecialistAgentFactory`, `ToolRegistry`, `PiiFirewall`
- `JwtHelper`, `PgpCryptoService`, `MTLSConnectionFactory`, `GpayStatusUpdateImpl`, `GpayErrorResponse`
- `ParikshaConfig`, `ParikshaFlagService`, `FeatureFlagController`
- `ChangeRequestController`, `ApprovalWorkflowController`, `SqlValidationServiceImpl`, `SSHTunnelManager`
- `DataSourceAspect`, `TransactionRoutingDataSource`, `DataSourceContextHolder`
- `MandateStrategyFactory`, `DigioUtility`, `ANachLogsEntity`, `MandateCronServiceImpl.generateUniqueKeyForRetry`
- `InsuranceVendorFactory`, `InsuranceServiceImpl.insurancePolicyCron`
- `ApplicationStage`, `ApplicationStatusServiceImpl.insertApplicationTracker`, `TriggerServiceImpl.partnerStageEventConfigMap`

---

## DEFENSIVE PHRASES (when stuck)

- **Don't know:** "I haven't shipped that, but I'd reason about it as…" → first principles → trade-off.
- **Pressed on Kafka internals:** "Platform team owned cluster ops; I owned producer/consumer code, idempotence, and dedup pattern."
- **Pressed on state machine:** "Pragmatic enum + tracker + trigger, deliberately not Spring State Machine — partner-specific behaviour didn't fit the framework."
- **Pressed on Acko:** "ICICI live, Acko scaffolded; abstraction proven by parallel in DLS NACH."
- **HMAC `equalsIgnoreCase` flagged:** "Yes, timing-leaky. For new code I'd use `MessageDigest.isEqual`. Tech debt I'd fix."

---

## TOP 15 RAPID-FIRE FACTS

1. **HashMap default load factor:** 0.75; treeify ≥8 (cap ≥64); untreeify ≤6.
2. **ConcurrentHashMap:** Java 8+ uses CAS + per-bucket synchronized; reads lock-free.
3. **`volatile`:** visibility + ordering, NOT atomicity.
4. **Spring bean default scope:** singleton.
5. **`@SpringBootApplication`** = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.
6. **`@Transactional` self-invocation:** bypasses proxy; split beans or inject self.
7. **JPA N+1:** lazy collection in loop; fix with `JOIN FETCH` / `@EntityGraph` / DTO projection.
8. **Hibernate Envers:** auto `_aud` tables on `@Audited`.
9. **TLS 1.3:** 0-RTT possible; only forward-secret ciphers.
10. **PGP cache key:** by **key-ID**, not alias (survives rotation window).
11. **JWT alg:** RS256 verified via JWKS (never `none`).
12. **Outbox pattern:** business write + outbox row in same tx; relay publishes; consumer idempotent.
13. **Token bucket > fixed window** (smoother under burst).
14. **Cache-aside + jittered TTL** prevents stampede.
15. **Always-200 to webhook source** + persist + async process.

---

## CODING TEMPLATE (live narration)

1. **Clarify** — input range, edges, expected output for ambiguous cases.
2. **Brute** — say complexity out loud (don't write).
3. **Optimize** — name the data structure / pattern (hash, two-pointer, binary search, monotonic stack, DP).
4. **Code** — clean, named vars, no magic numbers, `lo + (hi-lo)/2` not `(lo+hi)/2`.
5. **Test** — happy + 1 edge.
6. **Complexity** — time + space + justify.
7. **Hardening** — overflow, null, immutability.

---

## STAR TRIGGERS (memorize 3)

| Prompt | Story | Anchor |
|---|---|---|
| "Most challenging / impactful" | **Eklavya** | Single-specialist-per-turn, PII firewall on 3 boundaries, `ToolPermissionScoper`, MySQL persistence over Redis |
| "Production incident" | **Digio NACH callback storm** | Redisson `RLock` `digio:upinach:callback:{mandateId}` + always-200 + `ANachLogsEntity` dedup + graceful degrade |
| "Technical decision proud of" | **GPay 3-layer security** | JWT dual JWKS + PGP key-ID cache + TLS 1.3 mTLS in-memory PEM-to-JKS; passed first time |

Backup: **Pariksha frontend token** (conflict), **ConfigNexus 4-role split** (lead w/o authority), **Mandate retry idempotency** (mistake).

---

## SYSTEM DESIGN OPENING

"Before I jump in — can I clarify scale, consistency, and read:write ratio?"

**7 steps:** Clarify → Reqs (functional/NFR) → Capacity → API → Data model → Arch → Deep dives + trade-offs.

**Closing line:** "Bottlenecks I'd watch first: X. To 10x: Y. Trade-offs I made: chose A over B because Z. Anywhere you want me to go deeper?"

---

## QUESTIONS TO ASK THEM (closing — ASK 2 only)

1. "On-call rotation + top reliability risks for this team today?"
2. "Success criteria for this role at the 6-month mark?"
3. "How does the team approach trade-off conversations — speed vs reliability vs tech debt?"

---

## ANTI-PATTERNS — DON'T

- Lead with InsureX or DLS-NACH.
- Claim "Spring State Machine".
- Volunteer specific Kafka throughput numbers.
- Stay shallow when probed — say "I touched the surface; team owned deep ops" instead.
- Speak negatively about PayU, vendors, or anyone.
- Apologize for not knowing — reason from first principles instead.

---

## LOGISTICS

- Join 5-10 min early. Camera + mic + screen-share tested.
- Water nearby. Resume open in another tab. This file in another.
- Phone on silent.
- Speak to **trade-offs**, not features.
- End every story with **outcome + what you learned**.

---

## 60-SEC SELF-PITCH (rehearse 3x out loud)

> "Five years backend, currently senior at PayU. Two flagships: Eklavya — multi-agent AI orchestrator on Spring Boot 3 / Java 17 / AWS Bedrock, with PII firewall and per-specialist tool scoping; and Google Pay term loan — three-layer security stack with JWT, PGP, and TLS 1.3 mTLS, passed Google's security review first time. Beyond those, ConfigNexus governance platform and Pariksha feature flags. Looking at Airtel for consumer-scale impact where reliability, security, and personalization compose at a much larger footprint than payments alone."

You're ready.
