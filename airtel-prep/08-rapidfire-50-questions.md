# File 8: 50 Rapid-Fire Questions — One-Line Answers

> Warm-up tomorrow morning. Read all 50 in 5 minutes. Re-read any that don't roll off the tongue.

---

## CORE JAVA (15)

1. **`==` vs `.equals()`?** → Reference vs logical equality.
2. **HashMap thread-safe?** → No. Use ConcurrentHashMap.
3. **HashMap load factor default?** → 0.75; treeify ≥8 with capacity ≥64.
4. **HashMap capacity always?** → Power of 2 (so `(n-1) & hash` works).
5. **String immutable why?** → Thread-safety, hashCode caching, security (key in HashMap).
6. **String pool location?** → Heap since Java 7 (was PermGen before).
7. **`final` vs `finally` vs `finalize`?** → Modifier / try-block / GC hook (deprecated).
8. **Checked vs unchecked exception?** → Compiler-enforced vs not. Modern Java + Spring favors unchecked.
9. **`volatile` guarantees?** → Visibility + ordering, NOT atomicity.
10. **`AtomicInteger.incrementAndGet`?** → CAS-based atomic increment.
11. **`synchronized` vs `ReentrantLock`?** → Implicit JVM monitor vs explicit, with `tryLock`/timeout/fairness.
12. **`ExecutorService` vs `Thread`?** → Pool + lifecycle vs raw thread; always pool in production.
13. **`CompletableFuture.allOf` vs `anyOf`?** → Wait for all vs first.
14. **Default GC in Java 17?** → G1.
15. **Java 17 vs 8 — biggest wins?** → Records, sealed classes, pattern matching, ZGC, removed permgen long ago.

---

## SPRING / SPRING BOOT (10)

16. **Bean default scope?** → Singleton.
17. **`@RestController` = ?** → `@Controller` + `@ResponseBody`.
18. **`@Component` vs `@Service` vs `@Repository`?** → All `@Component`; `@Repository` adds exception translation.
19. **DI types?** → Constructor (preferred), setter, field (avoid).
20. **`@Transactional` on private method?** → Doesn't work — proxy can't intercept.
21. **`@SpringBootApplication` =?** → `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.
22. **Profiles activate how?** → `spring.profiles.active=prod` (env / arg / property).
23. **Filter vs Interceptor?** → Servlet level (before DispatcherServlet) vs Spring MVC level (around handler).
24. **Bean lifecycle order?** → Constructor → DI → `@PostConstruct` → init → use → `@PreDestroy` → destroy.
25. **Circular dependency fix?** → Refactor (extract third bean) — `@Lazy` is workaround, not solution.

---

## REST + HTTP (5)

26. **Idempotent HTTP methods?** → GET, PUT, DELETE, HEAD, OPTIONS. POST is not.
27. **PUT vs PATCH?** → PUT replaces; PATCH partial update.
28. **Stateless REST means?** → Server doesn't keep client session; each request self-contained.
29. **HTTP 401 vs 403?** → Not authenticated vs authenticated but not authorized.
30. **CORS solves?** → Browser same-origin policy for cross-origin AJAX.

---

## DATABASE / SQL (8)

31. **Index improves what?** → Reads on indexed columns; slows writes.
32. **B-tree index good for?** → Range queries + equality; not for `LIKE '%foo%'`.
33. **Composite index `(a, b)` — does it help WHERE b=?** → No; only WHERE a=? or WHERE a=? AND b=?.
34. **`INNER` vs `LEFT JOIN`?** → Only matching rows vs all left + matching right (nulls otherwise).
35. **N+1 problem fix?** → `JOIN FETCH`, `@EntityGraph`, DTO projection.
36. **JPA lazy default for `@OneToMany`?** → LAZY.
37. **JPA lazy default for `@ManyToOne`?** → EAGER (often wrong; override to LAZY).
38. **`SERIALIZABLE` isolation cost?** → Highest correctness, lowest concurrency; rarely used.

---

## DISTRIBUTED / MICROSERVICES (7)

39. **CAP — pick two?** → C + A + P; pick CP or AP under partition.
40. **Eventual consistency means?** → Replicas converge given no new writes; reads may be stale briefly.
41. **2PC vs Saga?** → Synchronous coordinator (rare) vs chain of local txns + compensations.
42. **Outbox pattern solves?** → Dual write — DB + message bus reliability.
43. **Idempotency key TTL?** → Long enough to outlast retries (24h+ for money).
44. **Circuit breaker states?** → Closed → open → half-open → closed.
45. **Service mesh — example?** → Istio / Linkerd. Sidecar handles mTLS, retries, observability.

---

## SECURITY (5)

46. **JWT structure?** → Header.Payload.Signature (Base64URL encoded).
47. **JWT alg `none` — safe?** → No. Reject `none`; pin to `RS256`/`ES256`.
48. **OAuth2 vs OIDC?** → Authorization vs identity (OIDC sits on OAuth2).
49. **mTLS vs TLS?** → Both sides present cert; mutual identity at transport.
50. **HMAC vs encryption?** → HMAC = integrity + authenticity; encryption = confidentiality. Different goals.

---

## BONUS — 10 INDUSTRY-SPECIFIC (FinTech / Telecom)

51. **NACH?** → National Automated Clearing House — bulk debit/credit mandates.
52. **UPI flow?** → Initiator → PSP → NPCI → beneficiary PSP → bank; idempotent on UPI ref.
53. **PCI-DSS scope?** → Anything touching PAN. We tokenize — never store PAN.
54. **KYC types?** → eKYC (Aadhaar OTP), Video KYC, paper-based.
55. **CIBIL pull window?** → RBI mandates re-pull within 30 days for re-decisioning; cache valid pulls.
56. **Idempotency for payments?** → Client-supplied `Idempotency-Key`; server caches result for 24h+.
57. **Provider timeout — retry?** → Only if "definitely failed" (network error before send). If unknown — recon.
58. **Settlement T+1?** → Money settles next business day after capture.
59. **3DS?** → Cardholder auth challenge for online card payments.
60. **DLQ?** → Dead Letter Queue — failed messages after N retries; manual replay tooling.

---

## HOW TO USE THIS FILE

- Read all 60 in sequence at 9:30 AM tomorrow.
- Mark any you stumble on with a star.
- Re-read starred ones at 10:30 AM.
- This is muscle memory, not deep thinking — these should fire in <5 sec each in the interview.
