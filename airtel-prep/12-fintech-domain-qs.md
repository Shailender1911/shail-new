# File 12: FinTech Domain Questions

> Airtel has Airtel Payments Bank + Airtel Money + Airtel Finance — they may probe domain depth.
> Lead with what you've shipped at PayU; this file fills the vocabulary gaps.

---

## SECTION 1 — INDIAN PAYMENT RAILS (must know)

### NEFT / RTGS / IMPS / UPI / NACH — when each is used

| Rail | Use case | Settlement | Limits |
|---|---|---|---|
| **NEFT** | Bank → bank, no SLA | Hourly batches, T+0 | No limit |
| **RTGS** | High-value, real-time | Real-time | ≥ ₹2 lakh |
| **IMPS** | 24x7 instant transfer | Real-time | ≤ ₹5 lakh |
| **UPI** | Account-linked instant payments | Real-time | ≤ ₹1 lakh (varies by bank) |
| **NACH** | Bulk debit/credit; recurring mandates | T+1 | Per mandate |

### NACH deep dive (your DLS NACH expertise)
- **NPCI's batch rail** for recurring debits (loan EMI, SIP, insurance premium).
- **Mandate** = customer authorizes a creditor to debit their account up to a limit on a frequency.
- **Mandate types we handle:**
  - **UPI NACH** — mandate authorized via UPI app; modern.
  - **API NACH** — net-banking redirect; stable.
  - **Physical NACH** — paper form; legacy, declining.
- **Lifecycle:** create → submit to bank → ACTIVE / REJECTED → debit attempts (per cycle) → ACK / NACK responses.
- **Digio** is our aggregator (you don't talk to NPCI directly).

### UPI deep dive
- **Players:** payer's PSP (PhonePe, GPay, Paytm) ↔ NPCI ↔ payee's PSP ↔ payee's bank.
- **VPA** (Virtual Payment Address): `kumar@oksbi`, `9876543210@ybl`.
- **Transaction id (RRN / UPI ref):** unique per transaction; idempotency anchor.
- **Collect vs Push:** push = payer initiates; collect = payee requests, payer approves.
- **UPI AutoPay:** UPI-equivalent of NACH for recurring (you've worked on UPI NACH variant).

---

## SECTION 2 — KYC + COMPLIANCE

### KYC types (RBI-defined)
- **Full KYC** — paper or digital with verified ID + address proof + photo.
- **eKYC (Aadhaar OTP)** — instant, OTP on Aadhaar-linked mobile. ₹1L wallet limit, certain product limits.
- **Video KYC (V-KYC)** — agent-led video session with liveness check + ID match. RBI-mandated for many lending flows. (Hyperverge in your stack.)
- **Re-KYC** — periodic refresh (RBI mandates 2/8/10 years based on risk).

### KYC stages (your DLS state machine sub-states)
1. PAN verification (NSDL)
2. Aadhaar verification (UIDAI eKYC)
3. Address verification (Aadhaar OR self-declared)
4. Liveness check (selfie + face match against ID)
5. Bank account verification (Penny Drop)
6. Final approval

### Penny Drop
- Send ₹1 to the customer's bank account; bank returns name on account.
- Match against KYC name → confirms account validity AND ownership.
- Replaced cancelled-cheque-uploads.

### CIBIL / Bureau pull
- **CIBIL, Experian, Equifax, CRIF** — RBI-licensed credit bureaus.
- **Hard pull** = full report, affects score; **Soft pull** = no impact.
- **Score:** 300-900; lenders typically reject < 650.
- **Cache window:** RBI mandates max 30 days for re-decisioning; after that re-pull.

---

## SECTION 3 — MONEY-FLOW SAFETY (your wheelhouse)

### Idempotency for payments (THE topic for FinTech R1)
- **Client-supplied `Idempotency-Key`** header per payment attempt.
- **Server stores key → result for 24h+.**
- **Replay** returns stored result, doesn't re-execute.
- **Retry policy:** retry only when "definitely failed" (network error before send). Timeout after send → mark `STATUS_UNKNOWN`, recon next morning.
- **Belt-and-braces:** unique constraint on `(merchant_id, idempotency_key)` AND on `(provider, provider_txn_id)`.

### Two-phase commit on money is bad
- 2PC blocks. Coordinator SPOF. Doesn't survive network partitions.
- **What we do instead:** outbox pattern + idempotent consumers + reconciliation cron.

### Reconciliation — non-negotiable for any money system
- EOD: download provider settlement file → diff against our DB → flag discrepancies → ops alerts → manual resolution path.
- Discrepancy types: provider has a txn we don't (orphan); we have a txn provider doesn't (missing); amount mismatch.
- Without recon, drift accumulates silently.

### Settlement T+0 / T+1 / T+2
- T+0 = same-day; T+1 = next business day. Most card payments are T+2; UPI is T+0.
- Money flow: customer → acquirer → network → issuer; settlement goes the other way.

---

## SECTION 4 — SECURITY (FinTech-specific)

### PCI-DSS scope reduction (interview talking point)
- PAN (card number) = Primary Account Number. Anything storing/processing PAN is in scope → expensive audits.
- **Strategy:** tokenize at the edge. We accept card via PG iframe → PG returns a token → we store the token, never PAN.
- Reduces our PCI scope to: a few audited services that touch tokens (not PAN).

### Encryption at rest
- DB-level: TDE (Transparent Data Encryption).
- Field-level: KMS-managed keys for sensitive columns (Aadhaar, PAN). Key rotation per RBI guidance.

### Encryption in transit
- TLS 1.2+ everywhere. TLS 1.3 preferred (you've shipped TLS 1.3 mTLS in `MTLSConnectionFactory`).

### HMAC vs digital signature
- **HMAC** (e.g. HMAC-SHA256): symmetric — same secret on both sides. Faster. Used for webhook signatures (Digio, Hyperverge).
- **Digital signature** (RSA/ECDSA): asymmetric — sign with private, verify with public. Used in JWT (RS256), PGP signing (you've shipped both).

### JWT pitfalls (interview classic)
- **`none` algorithm acceptance** — never accept; pin allowed algs explicitly.
- **Algorithm confusion (RS256 → HS256)** — attacker uses your public RSA key as HMAC secret. Pin alg, not just verify.
- **Long-lived tokens without revocation** — pair with short TTL + refresh token.
- **PII in payload** — JWT is base64, NOT encrypted. Don't put secrets in claims.

---

## SECTION 5 — RBAC (your ConfigNexus expertise)

### Common patterns
- **RBAC** (Role-Based) — user → role → permissions. Simple, common.
- **ABAC** (Attribute-Based) — policies evaluate user/resource/env attributes. More flexible, more complex.
- **Separation of Duties (SoD)** — same user cannot both approve and execute. **Your ConfigNexus 4-role split.**
- **Least privilege** — users get minimum needed; expanded only when needed.
- **Time-bound elevation** — break-glass / sudo with audit + expiry.

### Audit trail requirements (compliance)
- Who did what, when, on what resource, with what before/after state.
- Tamper-evident (append-only, ideally).
- Retention per regulation (often 7 years for financial data).
- **Your code:** Hibernate Envers `@Audited` + `ConfigAudit` for application-level events.

---

## SECTION 6 — REGULATORY VOCAB (drop naturally)

| Term | Means |
|---|---|
| **RBI** | Reserve Bank of India — banking regulator |
| **NPCI** | National Payments Corporation — runs UPI, IMPS, NACH |
| **SEBI** | Stock market regulator |
| **IRDAI** | Insurance regulator |
| **PA / PG** | Payment Aggregator / Payment Gateway |
| **NBFC** | Non-Banking Financial Company (most lenders) |
| **DPDPA / DPDP Act** | India's data privacy law (2023) |
| **PMLA** | Prevention of Money Laundering Act |
| **AML / CFT** | Anti Money Laundering / Counter Financing of Terrorism |
| **FRM** | Fraud Risk Management |
| **APR** | Annual Percentage Rate (interest + fees combined) |
| **EMI** | Equated Monthly Installment |
| **DPD** | Days Past Due (delinquency tracking) |
| **NPA** | Non-Performing Asset (>90 DPD) |
| **DSC** | Digital Signature Certificate |
| **eSign** | Aadhaar-based digital signing |
| **CIBIL TU** | TransUnion CIBIL — credit bureau |

---

## SECTION 7 — INTERVIEW Q&A

### Q: "How do you make sure a customer is not double-charged?"
> *Use Design 4 from File 05 — Idempotent Payment API.* Lead with: client supplies `Idempotency-Key`, server stores key → result, unique constraints belt-and-braces, recon job catches drift. Specifically: "We treat the timeout-after-send as STATUS_UNKNOWN, not retry — that's how we got bitten before. Anything we're unsure about goes to recon next morning."

### Q: "Walk me through KYC for a loan application."
> "Customer enters basic details → backend validates PAN with NSDL → Aadhaar eKYC for identity + address (OTP-based) → Penny Drop verifies bank account ownership → if RBI requires Video KYC for the product, agent-led video session with liveness + ID match (we use Hyperverge) → Bureau pull (CIBIL) for credit decisioning → final approval. Each step is idempotent and stage-gated — re-running is safe; conflicting state transitions are deactivated by `ApplicationStatusServiceImpl.insertApplicationTracker`."

### Q: "How do you handle a webhook from a payment provider?"
> "Always-200-first pattern. Provider's retry policy is aggressive — anything non-200 triggers re-delivery. We HMAC-validate the signature, persist the event with unique constraint on event id, ack 200, then process async. Concrete: Digio webhook handler in Orchestration uses Redisson `RLock` `digio:upinach:callback:{mandateId}` for in-flight dedup + `ANachLogsEntity` unique constraint as belt-and-braces. Eliminates duplicate state transitions."

### Q: "What's tokenization and why does it matter?"
> "Replace sensitive data (card PAN) with a non-sensitive token at the edge. We never store PAN; we store the token issued by the payment gateway. Drastically reduces our PCI-DSS audit scope — instead of auditing every service that might touch a card, only the tiny tokenization boundary needs PCI compliance. Same idea applies to PII more broadly."

### Q: "Why is reconciliation important?"
> "In any money flow, drift is inevitable — provider has a txn we don't, or vice versa, or amounts mismatch. Without recon, drift accumulates silently and the business can't trust its own numbers. EOD recon downloads the provider settlement file, diffs against our DB, flags discrepancies for manual resolution. It's not optional — it's how we sleep at night."

### Q: "How do you design for compliance audit?"
> "Three layers. **Field-level audit** via Hibernate Envers (`@Audited` on `PolicyDetails`, etc.) — every change has a revision id. **Action-level audit** via application-side `ConfigAudit` rows — captures who/when/what for governance events that Envers can't model (approvers, deployers, justifications). **Immutability** — audit rows are append-only, never updated or deleted. Retention is per regulation (7 years for most financial data)."

### Q: "What's a good idempotency key?"
> "Client-generated UUID per intent. Same `(merchant_id, idempotency_key)` means 'same intent — return the prior result'. Bad keys: random per retry (defeats the purpose), business identifiers like loan id alone (collides across attempts). Good keys: client UUID per user action, scoped to merchant."

### Q: "What happens if a customer's NACH debit fails?"
> "Bank returns NACK with reason code (insufficient funds, account closed, etc.). We mark the attempt FAILED, route to a retry strategy based on reason (insufficient funds → retry next day; account closed → escalate to ops). `MandateCronServiceImpl` runs the retry pipeline with `generateUniqueKeyForRetry` so each retry is idempotent at the bank end. Max retries bounded; after that → DPD tracking + collections handoff."

---

## REGULATORY-COMPLIANCE OPENING (if they ask "how does compliance shape your design")

> "Three things drive my design when compliance is in scope. First — every state change is audited; we use Hibernate Envers plus application-level `ConfigAudit` for fields Envers can't model. Second — separation of duties on sensitive operations; ConfigNexus has Editor/Reviewer/Admin/Deployer split with mandatory incident ticket for break-glass force-merge. Third — encryption everywhere; field-level KMS for PAN/Aadhaar at rest, TLS 1.3 in transit, and for partner integrations like GPay we layer JWT identity, PGP payload, and mTLS transport. The discipline is: if it can be audited later, it must be auditable now."
