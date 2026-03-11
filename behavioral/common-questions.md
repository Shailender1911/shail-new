# Common Behavioral Interview Questions for SDE-2

> Organized by category. For each question, think of which STAR story from `star-stories.md` you can adapt.

---

## Category 1: Leadership & Ownership

| # | Question | Suggested Story |
|---|----------|-----------------|
| 1 | Tell me about a time you led a project | Story 1 (Partner Integrations) |
| 2 | Describe a system you designed from scratch | Story 2 (NACH Service) |
| 3 | How do you make technical decisions? | Story 2 or Story 4 |
| 4 | Tell me about a time you took initiative | Story 8 (Automating Limit Renewals) |
| 5 | How do you prioritize when you have multiple tasks? | Adapt Story 1 (5 partners, prioritized by business impact) |

### How to answer Leadership questions at SDE-2 level:
- Show you can lead WITHOUT formal authority
- Demonstrate ownership beyond your assigned tasks
- Show you think about the team, not just your code
- Mention how you unblocked others or shared knowledge

---

## Category 2: Problem Solving & Technical Depth

| # | Question | Suggested Story |
|---|----------|-----------------|
| 6 | Walk me through your debugging process | Story 6 (Production Incident) |
| 7 | Tell me about a performance issue you solved | Story 3 (Redis Caching) |
| 8 | Describe the most complex technical challenge you've faced | Story 2 (NACH Service) or Story 4 (InsureX) |
| 9 | How do you approach learning new technologies? | Mention CompletableFuture for InsureX, Kafka integration |
| 10 | Tell me about a time you had to optimize for scale | Story 3 (Redis) + Story 8 (Rate Limiting) |

### How to answer Technical Depth questions:
- Start with the PROBLEM, not the solution
- Explain WHY you chose your approach over alternatives
- Mention trade-offs you considered
- Always include measurable results

---

## Category 3: Collaboration & Communication

| # | Question | Suggested Story |
|---|----------|-----------------|
| 11 | How do you handle disagreements with teammates? | Story 5 (Sync vs Async Webhooks) |
| 12 | Tell me about a cross-team project | Story 7 (Partner Onboarding State Machine) |
| 13 | How do you communicate technical concepts to non-technical people? | Adapt Story 7 (dashboard for all stakeholders) |
| 14 | Describe a time you gave difficult feedback | Adapt Story 5 |
| 15 | How do you handle working with a difficult team member? | Adapt Story 5 (focus on empathy + data) |

### How to answer Collaboration questions:
- Show EMPATHY first, solution second
- Demonstrate you listen before you speak
- Show how you find common ground
- Never badmouth anyone, even in hypothetical scenarios

---

## Category 4: Failure & Growth

| # | Question | Suggested Story |
|---|----------|-----------------|
| 16 | Tell me about your biggest mistake | Story 6 (Production Incident) |
| 17 | Describe a time a project didn't go as planned | Adapt Story 4 (unclear requirements) |
| 18 | How do you handle failure? | Story 6 (own it, fix fast, prevent recurrence) |
| 19 | What have you learned from a challenging experience? | Story 6 (EXPLAIN ANALYZE checklist) |
| 20 | Tell me about a time you received critical feedback | Create a story: code review feedback that changed your approach |

### How to answer Failure questions:
- OWN the mistake, don't deflect
- Show SPEED in response
- Emphasize SYSTEMIC fixes, not just one-time patches
- End with what you LEARNED and how you CHANGED

---

## Category 5: Process & Methodology

| # | Question | Suggested Story |
|---|----------|-----------------|
| 21 | How do you approach code reviews? | Mention specific things you look for: N+1 queries, error handling, test coverage |
| 22 | What's your testing strategy? | Discuss unit + integration + E2E approach |
| 23 | How do you handle tech debt? | Mention how you introduced EXPLAIN ANALYZE checks |
| 24 | Describe your ideal development process | Agile, sprint planning, CI/CD, code reviews |
| 25 | How do you stay updated with technology? | Mention Cursor AI, GenAI tools, side projects |

---

## Category 6: SDE-2 Specific Questions

These questions specifically assess whether you're ready for a Senior/SDE-2 role:

### 26. "How do you mentor junior engineers?"
**Framework answer:**
- I do thorough code reviews with explanations, not just "fix this"
- I pair program on complex features for the first implementation
- I document architectural decisions in ADRs (Architecture Decision Records)
- I create reusable templates and examples for common patterns
- *Example:* When a junior engineer was implementing a new partner integration, I walked them through the Strategy pattern we use and had them implement it with me for the first integration, then independently for the second.

### 27. "How do you estimate timelines for complex projects?"
**Framework answer:**
- Break down into subtasks (backend, frontend, testing, deployment)
- Add buffer for unknowns (usually 20-30%)
- Identify dependencies and blockers early
- Communicate early if timeline is at risk
- *Example:* For the InsureX service, I estimated 6 weeks but communicated to the PM that vendor API readiness could add 2 weeks. It did, but because I flagged it early, the PM adjusted the launch timeline proactively.

### 28. "Describe a time you had to balance quality vs speed"
**Framework answer:**
- Discuss the trade-off explicitly with stakeholders
- Propose an MVP approach: ship the core, iterate on polish
- Never compromise on security or data integrity
- *Example:* For a partner integration deadline, I proposed shipping with synchronous webhook delivery (simpler, faster to build) and migrating to Kafka-based delivery in the next sprint. The PM agreed, and we hit the deadline without compromising reliability for the critical path.

### 29. "How do you decide when to refactor vs rewrite?"
**Framework answer:**
- Refactor when: core logic is sound but code is messy, can be done incrementally
- Rewrite when: fundamental architecture doesn't support new requirements
- Always: measure the cost of not refactoring (bug rate, development velocity)
- *Example:* We refactored the webhook delivery system incrementally (adding retry logic, dead-letter queues) rather than rewriting because the core HTTP delivery worked well — we just needed reliability improvements.

### 30. "What makes good API design?"
**Framework answer:**
- Consistent naming and versioning (/v1/loans, not /getLoan)
- Proper HTTP status codes (201 for creation, 404 for not found, 409 for conflict)
- Pagination for list endpoints
- Idempotency for POST/PUT operations
- Clear error messages with error codes
- Rate limiting and authentication
- *Example:* In the NACH service, I designed idempotent mandate creation APIs so that retries from partners wouldn't create duplicate mandates.

---

## Managerial Round Questions (What cost you the Tide interview)

These are specifically for the final/hiring manager round:

### 31. "What's your management style?" (if asked about leading)
"I lead by enabling, not directing. I set clear goals and context, then trust the team to find the best approach. I'm available for guidance but I don't micromanage. I believe in code reviews as a teaching tool, not a gatekeeping mechanism."

### 32. "How do you handle conflict between team members?"
"I listen to both sides separately first, then bring them together to find common ground. I focus on the technical merits, not personal preferences. If we can't agree, I suggest we prototype both approaches and let the data decide."

### 33. "How would you onboard a new team member?"
"Week 1: Environment setup, codebase walkthrough, first small PR. Week 2-3: Pair on a medium feature. Week 4: Independent feature with thorough review. I also share our team's decision logs and architecture documents upfront."

### 34. "What would you do differently if you could restart your current project?"
"I would invest more in observability from day one. We added monitoring and alerting reactively after incidents, but having comprehensive dashboards from the start would have caught issues earlier and reduced our incident response time."

### 35. "How do you measure your team's/your own success?"
"For the team: deployment frequency, change failure rate, time to recovery (DORA metrics). For myself: impact of systems I build (reliability, performance), how many team members I've helped grow, and whether my architectural decisions stand the test of time."

---

## Quick Prep Checklist Before Any Interview

- [ ] Review your 8 STAR stories (re-read `star-stories.md`)
- [ ] Practice "Tell me about yourself" (2-minute version)
- [ ] Research the company: recent news, tech blog, engineering culture
- [ ] Prepare "Why this company?" with 3 specific reasons
- [ ] Prepare 2-3 questions to ask the interviewer:
  - "What does the engineering team look like? How are teams structured?"
  - "What's the biggest technical challenge the team is facing right now?"
  - "How does the team handle incidents and post-mortems?"
  - "What does career growth look like for an SDE-2 at [company]?"
