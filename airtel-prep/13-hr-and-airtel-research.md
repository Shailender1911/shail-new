# File 13: HR + Airtel Research

> Managerial round + HR call. Have these scripts ready.

---

## SECTION 1 — AIRTEL TECH STACK + PRODUCTS (research-backed)

### What Airtel does (quick framing)
- **Bharti Airtel** — India's #2 telecom (after Jio); ~400M+ subscribers in India.
- **Airtel Payments Bank** — RBI-licensed payments bank; Airtel Money wallet, Aadhaar-enabled banking, micro-ATMs.
- **Airtel Money** — wallet + UPI + bill pay; competing with PhonePe/GPay/Paytm in payments adjacent space.
- **Airtel Finance / Airtel Flexi Credit** — lending arm (personal loans, BNPL via partnerships).
- **Airtel Black** — bundled telecom + DTH + broadband + fiber.
- **Airtel Xstream** — content (OTT app + Fiber).
- **Airtel Business** — B2B (enterprise connectivity, cloud, IoT, security).
- **Wynk Music**, **Airtel Thanks** (consumer app), **Airtel ADS** (digital advertising / data monetization).

### Tech bets (from public reporting)
- **AI / GenAI:** Airtel partnered with Google Cloud for GenAI initiatives; building enterprise AI offerings; consumer AI use cases (network ops, customer support).
- **5G rollout:** ongoing cap-ex; standalone 5G infrastructure.
- **Network ops automation:** ML-driven network optimization, fault prediction.
- **Data platform:** subscriber-scale data infra; targeting personalization + enterprise resale (privacy-aware).
- **Cloud-native migration:** moving legacy stacks to k8s + microservices.

### Likely Java backend stack (educated guess; ask in interview to confirm)
- Spring Boot, Java 11/17.
- MySQL / PostgreSQL / Cassandra (subscriber-scale).
- Kafka (event streaming at telco scale).
- Redis.
- Kubernetes on cloud (Azure / GCP / hybrid).
- Observability: Prometheus, Grafana, ELK, Jaeger.

### Why this matters in your interview
- Mention Airtel Payments Bank / Airtel Money — shows you know they're a payments player. Connect to your PayU/lending background.
- Mention 5G + AI — connects to Eklavya AI work + the scale story.
- Don't overclaim familiarity — frame as "I've researched, here's why it lines up with my background."

---

## SECTION 2 — "WHY AIRTEL?" — POLISHED ANSWER

> "Three reasons specifically.
>
> First, **scale**. Airtel sits at ~400 million subscribers across telecom, payments bank, and lending — a footprint where engineering decisions on latency, reliability, and security have to compose at India scale. The work I've done — Eklavya for AI orchestration, GPay's three-layer security, ConfigNexus for governance — applies directly at that footprint.
>
> Second, **product surface**. Airtel Payments Bank, Airtel Money, and Airtel Finance are adjacent to my PayU domain — payments, lending, KYC, money flow safety — but at a different distribution scale. I'd be applying depth I already have to a much larger user base.
>
> Third, **technical bets**. Airtel's GenAI partnership with Google Cloud and the network-side ML push align with where I want to grow — AI as a product capability, not just a dev tool. My Eklavya work is exactly that pattern: LLMs in production, gated by tool scoping and PII firewalls."

---

## SECTION 3 — "WHY ARE YOU LEAVING PAYU?"

### The script (memorize)
> "PayU has been a strong run — five years, promoted from SDE to senior, depth across lending, AI orchestration with Eklavya, and governance with ConfigNexus. The org is in a good place. What I'm looking for next is **consumer-scale impact** — where the work I do touches a much larger user base, and where reliability, security, and personalization compose at scale. PayU's footprint is meaningful but it's primarily B2B and partner-driven; I want my next chapter at the consumer edge."

### What NOT to say
- "Pay is too low" (negotiate at the right time, not unprompted).
- "My manager / team / company" anything negative.
- "I've outgrown the role" (sounds arrogant).
- "I want better work-life balance" (signals lower commitment).

### What to convey
- Growth-oriented, not running-from.
- Consumer-scale + AI + reliability all in one frame.
- PayU was good; Airtel is the next chapter.

---

## SECTION 4 — SALARY NEGOTIATION

### When to bring it up
- **Don't volunteer first.** Wait for them to ask.
- If asked early: "I'd want to first understand the role and team better; happy to discuss numbers once we both feel it's a fit."

### When they push for a number
- **Anchor on total comp**, not base alone (ESOPs, bonus, joining bonus, perks).
- **Have a research-backed range:** Senior SDE at Airtel Java backend ~₹35–55 LPA total comp depending on level (varies; check Levels.fyi / AmbitionBox / Glassdoor for fresh data).
- **Frame as a range:** "Based on my research and current comp + the level I'd be joining at, I'd be looking at total comp in the X-Y range. Open to specifics once we discuss the role's level mapping."

### If they ask current CTC
- Be honest. "Current total comp is around ₹A LPA — base ₹B, variable ₹C, ESOPs at ₹D over vest period."
- Then immediately frame the next move: "For this role I'd be looking at ~25-40% increase given the scope and level — open to discuss specifics."

### Common levers (don't leave on table)
- **Joining bonus** — offset notice period overlap or unvested stock.
- **ESOPs / RSUs** — vest schedule (4 years with 1-year cliff is standard).
- **Variable / bonus target %** — 10-20% typical.
- **Notice period buyout** — if Airtel can buy out your PayU notice (often 60-90 days at PayU).
- **Designation** — Senior vs Lead vs Staff matters for next move.

### Don't accept on the call
- "Thanks for the offer. Let me review the full package and get back in 24-48 hours."
- Always negotiate at least once. They expect it. Asking for 10-15% more is normal.

---

## SECTION 5 — NOTICE PERIOD

### Standard answer
> "PayU's standard notice is [60/90 — verify yours] days. I can explore early-release with my current employer if needed; alternatively, the offer side can consider buyout. Happy to align on what works."

### What to actually do
- **Verify your notice period** in your offer letter / HR portal before the call.
- **Buyout options** — Airtel may offer 30-60 days buyout (essentially paying your salary to PayU to skip notice).
- **Garden leave** — uncommon in India for ICs but mentionable.

---

## SECTION 6 — LOGISTICS / LOCATION / WFH

### Location
- Airtel HQ: Gurugram (Bharti Crescent). Engineering offices in Bangalore, Pune, Chennai.
- Be ready: "I'm currently in [your city], open to relocating; would prefer hybrid if available."

### WFH / Hybrid
- Airtel has been on hybrid (typically 3 days office, 2 WFH) — verify.
- Don't push hard for full remote unless that's a deal-breaker.

---

## SECTION 7 — "ANY QUESTIONS FOR ME?" — TIERED LIST

### For technical interviewer (R1)
1. "What's the team's tech stack — Java version, Spring Boot version, primary databases?"
2. "What's the on-call rotation, and what are the top reliability risks for this team today?"
3. "What's a recent technical decision the team is proud of, and one you wish you'd done differently?"

### For hiring manager (R2)
1. "What does success in this role look like at the 6-month mark?"
2. "How does the team approach trade-off conversations — feature speed vs reliability vs technical debt? Who drives the call?"
3. "What's the team's biggest technical bet for the next 6-12 months?"
4. "How does career progression work here for senior ICs — staff track exists?"

### For HR
1. "What's the typical interview pipeline timeline from here?"
2. "What's the work-pattern — hybrid days expected?"
3. "What benefits stand out vs market — health, ESOPs, learning budget?"

### Don't ask (in early rounds)
- Salary specifics (let them lead).
- "When can I leave early?" / vacation policy specifics.
- Negative-framed questions ("Have you had layoffs?").

---

## SECTION 8 — STRENGTHS / WEAKNESSES (HR classics)

### "What's your biggest strength?"
> "Designing systems with explicit trade-offs. Two examples: GPay three-layer security — I picked Bouncy Castle over JCE for PGP, accepted the redeploy-for-key-rotation cost, and got the security review through first time. Eklavya — I picked single-specialist-per-turn over fan-out LLM voting, accepted the route-correctness sensitivity, and got the cost/latency profile we needed. The pattern is the same: name the trade-off, own the choice, document it for the team."

### "What's your biggest weakness / area of growth?"
> "I default to ownership-mode in incidents — I want the bug fixed, the recon done, the post-mortem written, and the prevention shipped. The growth area is **delegation** — pulling in others earlier so the whole team builds the muscle, not just me. I've started doing this consciously: pairing with juniors on Eklavya extensions instead of doing them myself. It's slower up front, faster team capability long-term."

### "Tell me about a time you failed."
> *Use STAR 6 (Mandate cron retry without idempotency).* Owned the incident, refunded customers, fixed with `generateUniqueKeyForRetry`, wrote the post-mortem that became internal reference. Closing learning: "Assume duplication in any async integration. Every retry must be idempotent at the protocol level."

---

## SECTION 9 — CULTURE-FIT QUESTIONS (HR / managerial)

### "How do you handle stress / tight deadlines?"
> "Triage and communicate. First — what's the actual ask, what's the immovable, what's negotiable. Communicate trade-offs to stakeholders early — false confidence is worse than asking for time. Then execute focused. The Digio NACH callback storm was a tight one — we had production duplicate state transitions, I owned the response, communicated the fix path hourly, and shipped within a day. Stress is fine when the team is informed; surprise is the killer."

### "How do you handle disagreement with a manager?"
> "Bring data and options, not just objections. The Pariksha frontend token situation was that — PM wanted backend SDK key in browser; I disagreed; I came back with two options, side-by-side trade-offs, and let them pick. Disagreement is fine; surprise outcomes aren't. Once a decision is made, I commit to it — disagree-and-commit is the real principle."

### "What kind of team / culture do you do well in?"
> "High autonomy + high accountability. Teams that ship in small slices, write postmortems, do code review seriously, and treat reliability as part of the product. I work less well in teams with heavy process for its own sake or unclear ownership."

### "Tell me about a time you mentored someone."
> *Use a real example or pivot to ConfigNexus 4-role split (STAR 5) — convincing a team without authority is mentoring-adjacent.* Or: "On Eklavya, I paired with a junior on the PII firewall extensions — instead of writing the new detector myself, walked through the design constraints, let them implement, reviewed. It took 2x my time, but they own that area now."

---

## SECTION 10 — RED FLAGS THEY MIGHT SURFACE

### "Why didn't you do more leadership at PayU?"
> "I've led without authority on several specific projects — ConfigNexus's separation-of-duties redesign, Pariksha's frontend token model, the Eklavya orchestration architecture. PayU's IC track favors depth + cross-team influence, which is what I've built. I'm interested in continuing on the IC track here — staff-level depth and influence rather than people management."

### "5 years at PayU — why no earlier moves?"
> "Promotion + scope kept evolving. SDE in 2022 → senior in late 2024 → flagship projects on Eklavya, ConfigNexus, GPay. Each was a meaningful new scope. The decision to look outside now is about consumer scale, not stagnation at PayU."

### "Tell me about a project that didn't go well."
> *Use STAR 6 (mandate retry).* Real, owned, learned, fixed. Avoid blaming others.

### "What if we offer a lower role / lower comp than your expectation?"
> "Open to discussing. The role + team + scope matter to me too — if there's a strong match on those, comp is a conversation. If the role is meaningfully below where I'm at currently, that's harder."

---

## SECTION 11 — QUICK-ANSWER MATRIX FOR HR ROUND

| Q | Tight answer |
|---|---|
| Years of experience | 5+ years (Comviva → Mobileum → PayU) |
| Current title | Senior Software Engineer at PayU |
| Current notice period | [Verify before call — typical 60/90 days] |
| Reason for change | Consumer-scale impact, applying lending + AI depth at India scale |
| Expected CTC | "Open to discussion based on level mapping; market range for senior Java backend at this scale is X-Y." |
| Joining bonus needed? | "Helpful to offset notice period overlap; happy to align on specifics." |
| Hybrid OK? | Yes |
| Relocate to Gurugram / Bangalore? | "Yes, open." (or your honest answer) |
| Marital / dependents | Polite but minimal. "Personal details I'd keep separate from professional discussion." |

---

## ONE-LINE NOTE TO YOURSELF

You've been at PayU 5 years. You've shipped flagship work. You're not asking for a chance — you're evaluating whether Airtel is the right next chapter. Walk in with that posture.
