# File 1: Flagship Projects Deep Dive

> Use this file to anchor every answer. Pick 2 flagships to lead with (Eklavya + GPay), keep ConfigNexus + Pariksha as backup depth, and reference Lending Stack only when asked about scale/reliability.

---

## 1. EKLAVYA — Multi-Agent AI Orchestrator for Merchant Lending

### What it does
LLM-powered conversational assistant that helps merchants navigate the loan lifecycle (eligibility → application → KYC → disbursement → repayment troubleshooting) on Web + WhatsApp. Replaced a brittle decision-tree FAQ bot.

### Architecture (one breath)
- **Spring Boot 3.2.5, Java 17, Spring AI 1.0.0-M6, AWS Bedrock** (Sonnet for reasoning, Haiku for routing/cheap calls).
- **Two-service split:**
  - `eklavya-agent` — sessions, channel adapters, routing, LLM calls, PII firewall, persistence (MySQL + Flyway, `eklavya_agent` DB).
  - `eklavya-backend` — read-only domain tools accessed via `RestClient` (no mutations from the agent).
- **Request flow:** `WebController/WhatsAppController` → `AgentOrchestrator` → `StageGroupRouter` → `IntentDisambiguator` → `SpecialistAgentFactory` → `SpecialistAgent.respond()`.
- **9 specialists** are NOT 9 services — they are prompt + tool-allowlist configs registered with `SpecialistAgentFactory` as a singleton bean.
- **Tool plane:** `ToolRegistry` (~17 tools) → `ToolPermissionScoper` (per-specialist allowlist) → `ConfirmationGate` (for sensitive tools) → `ToolExecutor`.
- **PII Firewall:** `PiiFirewall` scrubs (a) inbound user message, (b) outbound LLM response, (c) tool args + tool return JSON.
- **Multi-tenancy:** `ProgramConfigService` resolves program/channel-specific behaviour (prompts, tool allowlists, model overrides).

### Your contribution (own it)
- Designed the **single-agent-per-turn** orchestration model (vs. fan-out where N agents respond and we pick one).
- Built `ToolPermissionScoper` + `ConfirmationGate` so a specialist can't accidentally call a sibling's tools.
- Owned the **PII firewall** integration points (input/output/tool-JSON) and the failure mode (fail-closed: if firewall errors, we mask everything).
- Owned **persistence + session continuity** (conversation, turn, tool-invocation tables in MySQL via Flyway).

### Design decisions (have the why ready)
- **Single specialist per turn** vs fan-out → cost, latency, consistent voice, easier eval. Trade-off: one bad route hurts more, so `IntentDisambiguator` runs on Haiku to keep route cost low.
- **Read-only tools only** → state mutations (apply, pay, raise ticket) go through normal lending APIs after explicit confirmation, never auto-executed by the LLM. Trade-off: more turns, but auditable + safe.
- **No RAG for troubleshooting** — playbook content is embedded in the specialist prompt (versioned in code) instead of vector search. Trade-off: prompt size cost, but deterministic and easy to review/PR.
- **Bedrock over OpenAI** — Sonnet/Haiku tier mix gives latency vs reasoning trade-off; data residency + IAM (no extra vendor).

### Trade-offs you will be asked about
- "Why not LangChain/LlamaIndex?" → JVM stack alignment, ops familiarity, and we wanted explicit control over routing/tools, not a framework that hides them.
- "Why MySQL for sessions and not Redis?" → durability for compliance + audit; Redis used only for short-lived locks elsewhere.
- "How do you eval?" → frozen test conversations + tool-call assertions per turn (intent matched, allowed tool used, no PII leak).

### 60-second sound bite
> "Eklavya is a Spring Boot 3 / Java 17 multi-agent orchestrator on AWS Bedrock. A stage-aware router picks one of nine specialist configs, each with a scoped tool allow-list and a confirmation gate for sensitive actions. Every inbound, outbound, and tool payload passes through a PII firewall, and conversations persist in MySQL via Flyway. I designed the single-specialist-per-turn model and owned the tool permission and PII layers — it replaced a decision-tree FAQ bot and is now our merchant assistant on Web and WhatsApp."

### Follow-up Qs they will ask + how to answer
- **"How do you stop the LLM from calling random tools?"** → Tools are never exposed wholesale. Each specialist has a permission set in `ToolPermissionScoper`; `ToolExecutor` verifies the call against the set before invoking. Confirmation gate adds a human-in-the-loop step for irreversible actions.
- **"How do you handle PII?"** → `PiiFirewall` runs on three boundaries (in, out, tool-JSON). Detected PII is masked with type tokens; firewall failures fail-closed (mask all). We never log raw user text — only post-mask payloads.
- **"What about hallucinations on numerical answers?"** → Specialists are instructed to defer numbers to tools (`getOutstandingDues`, `getEligibility`). If the tool errors, the specialist must say so; never invent. Eval set asserts on tool calls, not on free text.
- **"What's the token / cost story?"** → Routing uses Haiku (cheap), specialist reasoning uses Sonnet, system prompts are version-controlled and reviewed for token bloat. Conversation history is windowed.
- **"Why not stream tokens?"** → Web channel does; WhatsApp doesn't support partial messages — different channel adapters handle it.

---

## 2. GOOGLE PAY TERM LOAN — Three-Layer Security Stack

### What it does
End-to-end backend integration for Google Pay's term loan product on India: merchant onboarding → eligibility → KYC → loan creation → disbursal → statement/repayment → reconciliation. Three security layers run in parallel because Google mandates them all.

### Architecture (one breath)
- **Inbound from Google → us:** Google sends signed JWTs in headers; our `JwtHelper` verifies via dual JWKS (v1 + v2 endpoints), enforces RS256, and binds the JWT subject to the application via `GooglePayContextHolder` (request-scoped).
- **Server-to-server (S2S) endpoints:** `GPayExternalController` exposes 5 PGP-encrypted endpoints (statement, repayment, settings, etc.). Body is PGP-encrypted + signed; we verify+decrypt with `PgpCryptoService` (Bouncy Castle) — passphrase-derived `PGPPrivateKey` cached by key-ID at init.
- **Onboarding endpoints:** 22+ on `/v1/gpay/*` authenticated by JWT only (no PGP — Google's contract).
- **Outbound to Google:** `GpayStatusUpdateImpl` notifies Google of stage changes (e.g. `APPLICATION_APPROVED`). HTTP 200 ≠ business success — we parse `acknowledged` key from the response (Google can return 200 with a business error). Exponential backoff retries on 5xx and on `acknowledged=false`.
- **Transport (mTLS):** `MTLSConnectionFactory` + `CertificateUtils` build a TLS 1.3 SSL context. PEM is converted to JKS in-memory at runtime (no on-disk JKS to leak). Google CA pinned in trust store. Lazy singleton + `volatile` double-checked init.
- **OAuth2 (service account):** `ServiceAccountCredentials.fromStream` for Google APIs that need OAuth (not the lending S2S itself).
- **Proxy:** `GPayProxyController` lets zipcredit-side services tunnel mTLS calls through Orchestration so we don't ship Google certs to every WAR.
- **Errors:** `GpayErrorResponse` matches Google's contract — `paymentIntegratorErrorCode`, `requestId`, epoch-millis timestamp; PGP-encrypted on the external path.

### Your contribution
- Built `PgpCryptoService` (sign+encrypt+armor outbound, verify+decrypt inbound). Base64URL armoring (Google's spec). Key-ID-keyed `PGPPrivateKey` cache so we don't re-derive from passphrase on every request.
- Built `MTLSConnectionFactory` (lazy singleton, TLS 1.3, in-memory JKS).
- Wrote `JwtHelper` (dual JWKS support for migration window) + `GooglePayContextHolder` (request scoped, cleared in filter `finally`).
- Built `GpayStatusUpdateImpl` retry semantics (HTTP 200 + `acknowledged` parsing).
- Built `GPayProxyController` so other services can call Google through a single mTLS chokepoint.

### Design decisions
- **Bouncy Castle over JCE** for PGP — JCE doesn't ship PGP primitives natively; Bouncy Castle is the de-facto standard.
- **Key cache by key-ID, not by alias** — Google rotates; cache survives multiple active keys during rotation window.
- **PEM → JKS in memory** — no disk artifact, simpler secret management (cert stored in secret manager as PEM).
- **mTLS proxy chokepoint** — single service holds Google certs; reduces blast radius and audit surface.
- **Dual JWKS (v1+v2)** — supports Google's JWKS endpoint migration without flag-day cutover.

### Trade-offs / honest limits
- Key rotation requires redeploy (no hot reload of PGP keys). Acceptable because Google notifies in advance and we use rolling deploys.
- `PgpCryptoService` is JVM-bound — not portable to our Python MCP. Acceptable: only Java services touch Google.
- `GPayProxyController` adds one network hop. Acceptable: all calls are async webhooks/notifications, not user-blocking.

### 60-second sound bite
> "I owned the Google Pay term loan backend at PayU. Three layers run together: Nimbus JWT with dual JWKS to verify Google's identity tokens; PGP sign-encrypt-armor with Bouncy Castle for server-to-server payloads, with the private key cached by key-ID; and TLS 1.3 mTLS via a lazy singleton connection factory that converts PEM to JKS in memory. Outbound notifications retry on 5xx **and** on `acknowledged=false` because Google can return HTTP 200 with a business error. We passed Google's security review first time and have had zero security incidents in production."

### Follow-up Qs + answers
- **"Why three layers — isn't mTLS enough?"** → JWT proves message-level identity (mTLS only proves channel). PGP proves payload integrity end-to-end across proxies. Each layer protects against different attacker classes.
- **"How do you rotate PGP keys without downtime?"** → New key added to keyring with new key-ID; both keys present for the rotation window; cache lookup is by key-ID so verify-side picks the right one. Send-side switches via config flag.
- **"Why parse `acknowledged` instead of HTTP status?"** → Google's contract: HTTP 200 means "we received it" not "we accepted it". Treating 200 as success caused duplicate notifications in early integration; parsing `acknowledged` fixed it.
- **"What's the proxy's failure mode?"** → Circuit breaker on the upstream call; retries with jitter; if proxy is down we don't notify Google — operational alert fires and a recon job replays from `gpay_notification_log`.
- **"Why request-scoped context vs. ThreadLocal?"** → Spring's request scope cleans up automatically; ThreadLocal leaks across thread-pool boundaries. We had a real incident with ThreadLocal in another integration; we don't repeat it.

---

## 3. CONFIGNEXUS — Multi-Tenant Config Governance Platform

### What it does
GitLab-MR-style platform for safely changing application configs and SQL across many lending services, with enforced separation of duties, SQL safety checks, blast-radius rollback, and an MCP wrapper so AI tools can use the same RBAC.

### Architecture (one breath)
- **Spring Boot 3.2, Java 17, MySQL, Flyway, MSAL Azure SSO + JWT**.
- **Two approval surfaces:**
  - Classic CR (`ChangeRequestController`) — single approver flow.
  - 3-level workflow (`ApprovalWorkflowController`) — Editor → Reviewer → Admin, with a separate **Deployer** role for segregation of duties (whoever approves cannot deploy).
- **Force-merge** path requires an incident ticket ID + business-impact justification, both stored on the audit row.
- **Deploy → `deploymentBatchId`** correlates every config row touched in one apply, used for rollback.
- **Config sync** — `ConfigSyncServiceImpl` compares source vs target with allow-listed tables (`SqlIdentifierValidator.sanitizeTableName`) and a `MAX_ROWS=500` bounded diff. Connections are ad-hoc or profile-based.
- **SQL execution** — `SqlValidationServiceImpl` blocks dangerous patterns (`TRUNCATE`, unbounded `UPDATE`/`DELETE`), runs `EXPLAIN` for DML, and auto-generates rollback SQL for some DDL.
- **Rollback** — `ConfigAudit` rows queried by `batchId` → `previewRollback` returns a time-bound (30-min) execution token; legacy fallback via `ChangeRequest` + `deploymentBatchId` + `ZipCreditJdbcService`.
- **Dynamic datasources** — `SSHTunnelManager` (JSch) opens a tunnel through bastion → `DynamicDataSourceFactory` builds a Hikari pool to `localhost:tunnelPort` with name `CNX-{serviceCode}`. Pool eviction on idle.
- **MCP wrapper (Python FastAPI)** — SSE + JSON-RPC + JWT pass-through via `contextvars.current_user_token`. **No direct DB access** — all calls go through the backend API so RBAC is enforced once.

### Your contribution
- Pushed for and shipped the 4-role split (Editor/Reviewer/Admin/Deployer) — see STAR story #5.
- Built the rollback path keyed by `deploymentBatchId` + `ConfigAudit` time-bound preview token.
- Built `SqlValidationServiceImpl` blocked-pattern list + EXPLAIN integration.
- Built the SSH tunnel + dynamic datasource factory with named Hikari pools per service.
- Built the MCP wrapper's JWT pass-through pattern using `contextvars`, so each MCP call inherits the caller's RBAC without server-side state.

### Design decisions
- **Backend-only RBAC** — MCP wrapper is intentionally dumb (no DB credentials, no SQL). Adding AI tooling cannot widen the security surface.
- **Time-bound rollback token (30 min)** — prevents stale/forged rollback executions; forces operator to be in the room.
- **Force-merge with mandatory incident ID** — escape hatch exists but every use is auditable.
- **Pool naming `CNX-{serviceCode}`** — observability: pool metrics in Micrometer are immediately attributable to a service without joining tags.

### Trade-offs
- 30-min rollback window can be too short for some incidents → operator can re-preview to refresh; we don't extend beyond 30 min by policy.
- 4-role split adds friction for small teams → mitigated by allowing one user to hold multiple roles in non-prod.
- SSH tunnel adds latency vs direct VPC peering → acceptable for governance traffic (low volume, not user-path).

### 60-second sound bite
> "ConfigNexus is the config-governance platform I built on Spring Boot 3 / Java 17. Changes flow like GitLab MRs but with enforced separation of duties — Editor, Reviewer, Admin, and a separate Deployer role. SQL is validated (blocked patterns, EXPLAIN for DML, auto-generated rollback), and every deploy gets a batch ID so we can preview-and-execute a precise rollback within a 30-minute window. We connect to target databases through SSH-tunnelled dynamic Hikari pools per service, and the MCP wrapper exposes the same RBAC to AI tooling by passing the caller's JWT through with contextvars — never giving the AI direct DB access."

### Follow-up Qs + answers
- **"Why not GitOps with PRs to a config repo?"** → We do for static config; ConfigNexus targets DB-backed runtime config + SQL operations where PR review is too slow and rollback needs row-level audit, not git revert.
- **"How is the rollback token tamper-proof?"** → Stored server-side keyed by hash, expires in 30 min, single-use, scoped to the batch + user. Token in response is opaque.
- **"What if the SSH tunnel dies mid-deploy?"** → `SSHTunnelManager` reconnects with backoff; the deploy transaction is per-statement, so we record progress in `ConfigAudit` and resume or rollback by batch ID.
- **"Why not give the MCP its own service account?"** → That would make the MCP a privilege escalation surface. With JWT pass-through, the AI tool can do exactly what the human caller can do — no more.
- **"How do you stop someone bypassing the workflow with a direct SQL client?"** → DB credentials are in the backend's secret manager and never issued to humans for these databases. Break-glass goes through the Force-Merge path with audit.

---

## 4. PARIKSHA — Feature Flag Platform at Scale

### What it does
Unleash-compatible feature flag stack used across the lending platform for gradual rollouts (per partner, per channel, per percent), kill switches, and A/B variants — without redeploys.

### Architecture (one breath)
- Custom Java SDK fork: `io.getpariksha:pariksha-client-java:1.0.2`.
- `ParikshaConfig` builds the `Unleash` bean with **synchronous fetch on init** (so first request after boot doesn't see "default off"), 10-second poll, hostname-based `instanceId` for impression attribution.
- `ParikshaFlagService.evaluateFlags(FlagEvaluationRequest)` builds an `UnleashContext` from the request map. Reserved keys: `userId`, `identifier`, `sessionId`, `appName`, `environment`. Everything else becomes a custom property (`channelCode`, `partner`, `programId`, …) used by Pariksha strategies/constraints.
- **Per-flag error → `enabled: false`** (fail-closed per flag). **Top-level SDK failure → empty response** (graceful degradation — caller treats all flags as off).
- BFF endpoint: `POST /v1/features/flags`, authenticated by `authenticateParikshaClient()` matching `x-pariksha-key` header to `pariksha.frontend.key` property.
- **Dual token model:** Frontend API token (browser, narrow scope) vs backend SDK key (server, full read). Mirrors Unleash's security model.
- Optional PostgreSQL impression tracking → `impression_data` table for downstream analytics.

### Your contribution
- Owned the server-side evaluator (`ParikshaFlagService`) and its error model.
- Built the BFF endpoint + frontend token authentication path (separate from backend SDK key — see STAR story #4).
- Wired PostgreSQL impression tracking and the decision to keep it optional via `@ConditionalOnProperty`.
- Pushed for the dual-token model after a security review flagged the SDK key as too broad to send to browsers.

### Design decisions
- **Synchronous init fetch** vs. async → eliminates a race where the first N requests after deploy get default values. Trade-off: boot is ~500 ms slower; acceptable.
- **Fail-closed per flag** vs. fail-open → in lending, accidentally enabling a feature is worse than disabling. Default `false` is safer.
- **Custom-property strategies** vs. user-bucketed only → partners and channels are first-class for us; bucketing by user alone wouldn't model rollout-per-partner cleanly.
- **BFF over direct frontend SDK** → see STAR story #4. Avoids exposing the backend SDK key.

### Trade-offs
- 10-second poll means flag flips have up to 10 s propagation delay → acceptable for rollouts; for kill switches we accept the worst-case window and document it.
- Impression table grows fast → partitioned by date; archived via cron.
- Dual token model adds a header + endpoint → small UX cost, large security win.

### 60-second sound bite
> "Pariksha is our Unleash-compatible feature flag stack with a custom Java SDK fork. I own the server-side evaluator: it builds an `UnleashContext` from the request — `userId`, `sessionId`, plus partner and channel as custom properties — and evaluates flags with strategies and constraints. Errors fail closed per flag and the SDK fails gracefully overall. We expose a BFF endpoint with a separate frontend API token so the browser never sees the backend SDK key, and we persist impressions to Postgres for analytics. It lets us roll out per partner, per channel, per percent without redeploys."

### Follow-up Qs + answers
- **"What about flag debt?"** → Quarterly cleanup pass + flag-age dashboard from impression data; flags older than 90 days require an owner or removal ticket.
- **"How do you test flag-gated code?"** → Unit tests pass an `UnleashContextProvider` mock; integration tests use a fake SDK that returns deterministic results per test profile.
- **"Why not LaunchDarkly?"** → Unleash is OSS, self-hostable (data residency), and we already had ops familiarity; cost wasn't the only driver.
- **"How do you prevent a misconfigured flag from breaking production?"** → Fail-closed default + canary partner first + impression data shows uptake within minutes; rollback is a single toggle in the admin UI.
- **"How does the frontend token's narrower scope work?"** → Frontend tokens are scoped to specific projects/environments in Pariksha admin; even if leaked, the blast radius is the frontend project, not the entire flag set.

---

## 5. SUPPORTING (Core Backend) Projects — Use For Depth Questions

### Orchestration BFF — Reliability + Caching
- **Redis distributed locks** (Redisson `RLock`) with key `digio:upinach:callback:{mandateId}` for Digio webhook dedup; **graceful degradation** — if Redis is down we still process but log a dedup-bypass warning.
- **Always-200 to Digio** pattern — Digio retries on any non-200; we ack first, persist `ANachLogsEntity`, then process.
- **OAuth2 method-level security** + standardized response via `ResponseHelper` (API v4 envelope).
- **Redis cache** (`RedisAuthCacheServiceImpl` + `CustomRedisCacheManager`) — drove **~20% API latency reduction** on auth-heavy endpoints.

### Loan Repayment — Read/Write Split + Strategy Pattern
- **Multi-datasource routing via AOP:** `@DataSource(FINFLUX_DB)` annotation → `DataSourceAspect` sets `DataSourceContextHolder` (ThreadLocal) → `TransactionRoutingDataSource` (extends `AbstractRoutingDataSource`) selects the right `DataSource`.
- **Read-write separation** via `DaoAsyncExecutorUtil` — `slaveDbExecutor` for reads, `masterDbExecutor` for writes; drove **~10x query improvement** on reporting paths.
- **Strategy + Registry** for partner report handlers keyed by `partner:reportName` (Swiggy, Meesho, etc.).
- **Chunked CSV processing** — 1000-row chunks, 500-row JDBC batch — bounds memory on big report files.
- **Mandate cron** uses `generateUniqueKeyForRetry` for idempotent debit retries (see STAR story #6).

### Digital Lending Suite — Pragmatic State Machine
- **Not** Spring State Machine framework. Pragmatic design: `ApplicationStage` enum (60+ states across NACH, CPV, KYC, approval, disbursement) + `ApplicationStatusServiceImpl.insertApplicationTracker` (deactivates conflicting dependent stages, triggers events) + `TriggerServiceImpl` with `partnerStageEventConfigMap` (channel + target stage → list of event configs).
- Drove **~30% onboarding-time reduction** by automating downstream triggers and partner-specific automations that were previously manual ops handoffs.
- Be precise in interview: "enum + tracker + trigger service" — don't claim Spring State Machine.

---

## How To Use This File In The Interview

1. Open with **Eklavya** if they ask "tell me about yourself" or "most challenging project". It's the strongest signal for staff-leaning IC work + AI/backend intersection.
2. Use **GPay** if they go deep on security, encryption, or integration design. It's the strongest signal for senior backend depth.
3. Use **ConfigNexus** if they ask about platform thinking, governance, RBAC, or how you handle "many services + many people".
4. Use **Pariksha** if they ask about rollouts, A/B, or how you ship safely.
5. Use **Lending Stack** for Redis, AOP, multi-datasource, idempotency, or "biggest production incident" depth.
6. **Never** lead with DLS-NACH or InsureX — only mention as supporting examples if asked specifically.
