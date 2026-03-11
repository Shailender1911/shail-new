# Spring Data JPA Interview Questions & Answers

> Comprehensive guide covering JPA/Hibernate internals, N+1 problem, transactions, locking, and performance.
> Real-world examples drawn from fintech/lending platforms.

---

## Table of Contents

1. [JPA/Hibernate Basics](#1-jpahibernate-basics)
2. [N+1 Problem (Most Asked)](#2-n1-problem-most-asked)
3. [Transactions](#3-transactions)
4. [Locking](#4-locking)
5. [Performance & Advanced](#5-performance--advanced)

---

## 1. JPA/Hibernate Basics

---

### Q1. Explain the JPA Entity Lifecycle. [Medium]

An entity transitions through four states:

```
  new Entity()          em.persist()         em.detach() / close session
      │                     │                        │
      ▼                     ▼                        ▼
 ┌──────────┐        ┌───────────┐            ┌───────────┐
 │ TRANSIENT │──────▶│  MANAGED  │──────────▶│  DETACHED  │
 └──────────┘        └───────────┘            └───────────┘
                       │       ▲                    │
                       │       │ em.merge()         │
                       │       └────────────────────┘
                       │
                       │ em.remove()
                       ▼
                  ┌───────────┐
                  │  REMOVED  │
                  └───────────┘
```

| State | In Persistence Context? | In Database? | Description |
|---|---|---|---|
| **Transient** | ❌ | ❌ | Plain Java object, `new Entity()`, JPA doesn't know about it |
| **Managed** | ✅ | ✅ (on flush/commit) | Tracked by EntityManager; changes auto-synced via dirty checking |
| **Detached** | ❌ | ✅ | Was managed, now disconnected (session closed / `em.detach()`) |
| **Removed** | ✅ | Pending DELETE | Marked for deletion, removed on flush/commit |

```java
@Transactional
public void entityLifecycleDemo() {
    // TRANSIENT: just a Java object
    Loan loan = new Loan();
    loan.setAmount(new BigDecimal("50000"));
    loan.setStatus(LoanStatus.PENDING);

    // MANAGED: JPA tracks this entity now
    entityManager.persist(loan); // INSERT scheduled (not executed yet)

    // Still MANAGED: dirty checking detects this change
    loan.setStatus(LoanStatus.APPROVED); // UPDATE scheduled automatically

    // REMOVED: DELETE scheduled
    entityManager.remove(loan);

    // On @Transactional method exit → flush → commit
    // SQL: INSERT → UPDATE → DELETE (optimized by Hibernate)
}

// DETACHED scenario
public Loan getDetachedLoan(Long id) {
    Loan loan = entityManager.find(Loan.class, id); // MANAGED
    entityManager.detach(loan);                       // DETACHED
    loan.setAmount(new BigDecimal("100000"));          // change NOT tracked
    return loan;
}

public void reattach(Loan detachedLoan) {
    Loan managed = entityManager.merge(detachedLoan); // MANAGED again, changes synced
}
```

> **Interview Tip:** A common mistake is modifying a detached entity and expecting it to be saved. Always use `merge()` or re-fetch within a transaction.

---

### Q2. Explain First-Level Cache vs Second-Level Cache. [Hard]

**First-Level Cache (L1) — Session/EntityManager cache:**
- Enabled by default, cannot be disabled
- Scoped to a **single transaction/session**
- Guarantees **repeatable reads** within a transaction
- Cleared when session closes or `em.clear()` is called

```java
@Transactional
public void firstLevelCacheDemo() {
    // SQL: SELECT * FROM loan WHERE id = 1
    Loan loan1 = loanRepository.findById(1L).orElseThrow();

    // NO SQL — served from L1 cache (same session, same ID)
    Loan loan2 = loanRepository.findById(1L).orElseThrow();

    System.out.println(loan1 == loan2); // true — same object reference!
}
```

**Second-Level Cache (L2) — SessionFactory-wide cache:**
- Shared across all sessions/transactions
- Must be explicitly enabled and configured
- Providers: Ehcache, Caffeine, Hazelcast, Redis (via Redisson)
- Caches entities, collections, and query results separately

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "loans")
public class Loan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;

    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "loanRepayments")
    @OneToMany(mappedBy = "loan")
    private List<Repayment> repayments;
}
```

| Cache Concurrency Strategy | Description | Use Case |
|---|---|---|
| `READ_ONLY` | Never modified after creation | Reference data (countries, currencies) |
| `NONSTRICT_READ_WRITE` | Eventual consistency, no locking | Rarely updated data |
| `READ_WRITE` | Uses soft locks for consistency | Frequently read, occasionally updated |
| `TRANSACTIONAL` | Full JTA transaction support | Critical financial data |

**Query Cache:**
```java
public interface LoanRepository extends JpaRepository<Loan, Long> {

    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    List<Loan> findByStatus(LoanStatus status);
}
```

> **Anti-Pattern:** Don't cache highly mutable data in L2 — the overhead of cache invalidation can be worse than hitting the database.

| Aspect | L1 Cache | L2 Cache |
|---|---|---|
| Scope | Single session/transaction | All sessions (application-wide) |
| Default | Always enabled | Must be configured |
| Invalidation | On session close | On entity update/evict |
| Shared | ❌ | ✅ |
| Stores | Entity object references | Dehydrated state (arrays, not objects) |

---

### Q3. How does Hibernate Dirty Checking work? [Medium]

When an entity is loaded into the Persistence Context (L1 cache), Hibernate takes a **snapshot** of its state (field values). At flush time, it compares the current state with the snapshot. If they differ, an UPDATE is generated.

```java
@Transactional
public void updateLoanStatus(Long loanId) {
    Loan loan = loanRepository.findById(loanId).orElseThrow();
    // Hibernate snapshots: { id=1, amount=50000, status=PENDING }

    loan.setStatus(LoanStatus.APPROVED);
    // No explicit save() needed!

    // At flush (before commit):
    // Current:  { id=1, amount=50000, status=APPROVED }
    // Snapshot: { id=1, amount=50000, status=PENDING }
    // Diff detected → generates: UPDATE loan SET status='APPROVED' WHERE id=1
}
```

**How the snapshot comparison works internally:**
1. `DefaultFlushEntityEventListener` iterates all managed entities.
2. For each entity, it calls `EntityPersister.findDirty()`.
3. This compares every field of the current state vs the loaded snapshot (using `==` for primitives, `.equals()` for objects).
4. If differences found → an `EntityUpdateAction` is queued.
5. On `flush()` → all queued actions are executed as SQL.

**Performance consideration:**
```java
// If you only read entities and don't modify them, use read-only to skip dirty checking
@Transactional(readOnly = true)
public List<Loan> getActiveLoans() {
    return loanRepository.findByStatus(LoanStatus.ACTIVE);
    // No dirty checking performed — better performance for read operations
}
```

> **Interview Tip:** "Setting `@Transactional(readOnly = true)` not only skips dirty checking but also provides hints to the JDBC driver and database for read optimization."

---

### Q4. What's the difference between flush() and commit()? [Easy]

| Operation | `flush()` | `commit()` |
|---|---|---|
| What it does | Synchronizes persistence context → database (executes SQL) | Ends the transaction, makes changes permanent |
| Reversible? | ✅ Changes can still be rolled back | ❌ Permanent |
| When called | Explicitly or automatically (before queries, at TX end) | At transaction boundary |
| Relationship | `commit()` always triggers a `flush()` first | `commit()` = `flush()` + end transaction |

```java
@Transactional
public void flushExample() {
    Loan loan = new Loan();
    loan.setAmount(new BigDecimal("50000"));
    entityManager.persist(loan);

    // SQL not executed yet — just queued

    entityManager.flush();
    // NOW: INSERT INTO loan (...) VALUES (...) is executed
    // BUT: transaction is still open, can be rolled back

    // loan.getId() is now available (generated by DB)
    log.info("Loan ID after flush: {}", loan.getId());

    // At method exit: commit() is called by @Transactional → changes are permanent
}
```

**Flush modes:**
- `AUTO` (default) — Hibernate flushes before queries to ensure consistent results
- `COMMIT` — Only flush on commit (better performance, risk of stale query results)

```java
entityManager.setFlushMode(FlushModeType.COMMIT); // delay flush for batch performance
```

---

### Q5. What is the difference between save() and saveAndFlush()? [Easy]

```java
public interface JpaRepository<T, ID> {
    <S extends T> S save(S entity);           // persist or merge, flush on commit
    <S extends T> S saveAndFlush(S entity);   // persist or merge + immediate flush
}
```

```java
// save() — SQL deferred until flush/commit
Loan loan = new Loan();
loan.setAmount(new BigDecimal("75000"));
Loan saved = loanRepository.save(loan);
// saved.getId() may be null (depends on ID generation strategy)

// saveAndFlush() — SQL executed immediately
Loan flushed = loanRepository.saveAndFlush(loan);
// flushed.getId() is guaranteed to be populated
```

**When to use `saveAndFlush()`:**
- When you need the generated ID immediately (for a response or dependent insert)
- When you need to catch database constraints before the transaction ends
- Batch inserts with `flush()` + `clear()` pattern (see Performance section)

---

## 2. N+1 Problem (Most Asked)

---

### Q6. What is the N+1 Problem? Explain with a clear example. [Hard]

The N+1 problem occurs when fetching a parent entity triggers N additional queries to fetch related child entities — one query for the parent list + N queries for each parent's children.

**Scenario: In a lending platform, fetch all active loans with their repayment schedules.**

```java
@Entity
public class Loan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String borrowerName;
    private BigDecimal amount;

    @OneToMany(mappedBy = "loan", fetch = FetchType.LAZY) // LAZY is default for collections
    private List<Repayment> repayments;
}

@Entity
public class Repayment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;
    private LocalDate dueDate;
    private RepaymentStatus status;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "loan_id")
    private Loan loan;
}
```

```java
// ❌ N+1 PROBLEM
@Transactional(readOnly = true)
public List<LoanWithRepaymentsDTO> getAllActiveLoans() {
    List<Loan> loans = loanRepository.findByStatus(LoanStatus.ACTIVE);
    // SQL 1: SELECT * FROM loan WHERE status = 'ACTIVE'
    // Returns 100 loans

    return loans.stream().map(loan -> {
        List<Repayment> repayments = loan.getRepayments(); // triggers lazy load
        // SQL 2:  SELECT * FROM repayment WHERE loan_id = 1
        // SQL 3:  SELECT * FROM repayment WHERE loan_id = 2
        // SQL 4:  SELECT * FROM repayment WHERE loan_id = 3
        // ...
        // SQL 101: SELECT * FROM repayment WHERE loan_id = 100
        return new LoanWithRepaymentsDTO(loan, repayments);
    }).toList();
}
// TOTAL: 1 + 100 = 101 queries! 💀
```

**Why is this bad?**
- 101 database round-trips instead of 1 or 2
- Network latency multiplied 101 times
- Database connection held for much longer
- In production with 10,000 loans → 10,001 queries

---

### Q7. What are the solutions to the N+1 Problem? [Hard]

#### Solution 1: JOIN FETCH (JPQL)

```java
public interface LoanRepository extends JpaRepository<Loan, Long> {

    @Query("SELECT l FROM Loan l JOIN FETCH l.repayments WHERE l.status = :status")
    List<Loan> findByStatusWithRepayments(@Param("status") LoanStatus status);
}
// SQL: SELECT l.*, r.* FROM loan l
//      INNER JOIN repayment r ON l.id = r.loan_id
//      WHERE l.status = 'ACTIVE'
// TOTAL: 1 query ✅
```

**Caveat:** JOIN FETCH with pagination doesn't work in a single query — Hibernate fetches all data and paginates in memory with a warning:
```
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

**Fix for pagination + JOIN FETCH:**
```java
// Step 1: Fetch paginated IDs
@Query("SELECT l.id FROM Loan l WHERE l.status = :status")
Page<Long> findIdsByStatus(@Param("status") LoanStatus status, Pageable pageable);

// Step 2: Fetch full entities with collections using the IDs
@Query("SELECT l FROM Loan l JOIN FETCH l.repayments WHERE l.id IN :ids")
List<Loan> findByIdsWithRepayments(@Param("ids") List<Long> ids);
```

#### Solution 2: @EntityGraph

```java
public interface LoanRepository extends JpaRepository<Loan, Long> {

    // Named EntityGraph
    @EntityGraph(attributePaths = {"repayments"})
    List<Loan> findByStatus(LoanStatus status);
    // SQL: SELECT l.*, r.* FROM loan l
    //      LEFT OUTER JOIN repayment r ON l.id = r.loan_id
    //      WHERE l.status = ?
    // TOTAL: 1 query ✅
}

// Or define on the entity
@Entity
@NamedEntityGraph(
    name = "Loan.withRepaymentsAndBorrower",
    attributeNodes = {
        @NamedAttributeNode("repayments"),
        @NamedAttributeNode("borrower")
    }
)
public class Loan { ... }

// Usage
@EntityGraph(value = "Loan.withRepaymentsAndBorrower")
List<Loan> findByStatus(LoanStatus status);
```

#### Solution 3: @BatchSize

Instead of N queries, Hibernate batches them into N/batchSize queries.

```java
@Entity
public class Loan {

    @OneToMany(mappedBy = "loan")
    @BatchSize(size = 25) // fetch 25 loan's repayments at a time
    private List<Repayment> repayments;
}
// With 100 loans:
// SQL 1: SELECT * FROM loan WHERE status = 'ACTIVE'
// SQL 2: SELECT * FROM repayment WHERE loan_id IN (1,2,3,...,25)
// SQL 3: SELECT * FROM repayment WHERE loan_id IN (26,27,...,50)
// SQL 4: SELECT * FROM repayment WHERE loan_id IN (51,52,...,75)
// SQL 5: SELECT * FROM repayment WHERE loan_id IN (76,77,...,100)
// TOTAL: 5 queries (instead of 101) ✅
```

Global batch size:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25
```

#### Solution 4: DTO Projection (Best for Read-Only)

```java
public record LoanRepaymentProjection(
    Long loanId,
    String borrowerName,
    BigDecimal loanAmount,
    BigDecimal repaymentAmount,
    LocalDate dueDate,
    String repaymentStatus
) {}

@Query("""
    SELECT new com.lending.dto.LoanRepaymentProjection(
        l.id, l.borrowerName, l.amount,
        r.amount, r.dueDate, r.status
    )
    FROM Loan l JOIN l.repayments r
    WHERE l.status = :status
    ORDER BY l.id, r.dueDate
    """)
List<LoanRepaymentProjection> findLoanRepaymentsFlat(@Param("status") LoanStatus status);
// TOTAL: 1 query, no entity overhead, no dirty checking ✅
```

#### Comparison of Solutions

| Solution | Queries | Pagination | Cartesian Product Risk | Best For |
|---|---|---|---|---|
| JOIN FETCH | 1 | ❌ (in-memory) | ⚠️ With multiple collections | Single collection fetch |
| @EntityGraph | 1 | ❌ (in-memory) | ⚠️ With multiple collections | Declarative, Spring Data |
| @BatchSize | N/batch | ✅ Works | ❌ None | Multiple collections, pagination |
| DTO Projection | 1 | ✅ Works | ❌ None | Read-only views, APIs |

> **Interview Tip:** "My go-to is `@BatchSize(25)` globally + DTO projections for API responses. JOIN FETCH for targeted fetching within a specific use case."

---

### Q8. How to detect N+1 problems? [Medium]

#### 1. Enable SQL logging (development only)

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true # shows which entity triggered the query
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE  # show bind parameters (Hibernate 6+)
```

#### 2. Use p6spy (detailed logging with execution time)

```xml
<dependency>
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>datasource-proxy-spring-boot-starter</artifactId>
    <version>1.9.1</version>
</dependency>
```

#### 3. Hibernate Statistics

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

```java
// Programmatic check in tests
@Test
void shouldNotHaveNPlusOne() {
    Statistics stats = entityManager.unwrap(Session.class)
            .getSessionFactory().getStatistics();
    stats.clear();

    loanService.getAllActiveLoans();

    long queryCount = stats.getQueryExecutionCount();
    assertThat(queryCount).isLessThanOrEqualTo(2); // expect 1-2 queries, not 101
}
```

#### 4. Use db-util library for automatic detection

```java
@SpringBootTest
class LoanServiceTest {

    @Autowired
    private SQLStatementCountValidator validator;

    @Test
    void shouldFetchLoansEfficiently() {
        validator.reset();
        loanService.getAllActiveLoans();
        validator.assertSelectCount(1); // fails if more than 1 SELECT
    }
}
```

---

## 3. Transactions

---

### Q9. Explain @Transactional — where to place it and how the proxy works. [Hard]

**Where to place `@Transactional`:**
- ✅ **Service layer** — business logic boundaries
- ❌ NOT on repositories — Spring Data already wraps them
- ❌ NOT on controllers — too coarse, mixes concerns
- ✅ `readOnly = true` for read operations — enables optimizations

**Proxy mechanism:**

```
Controller → calls → $Proxy(LoanService) → calls → Actual LoanService

$Proxy does:
  1. Begin transaction (get connection, setAutoCommit(false))
  2. Delegate to actual method
  3a. No exception → commit
  3b. RuntimeException → rollback
  3c. Checked exception → commit (by default!)
```

```java
@Service
public class LoanService {

    private final LoanRepository loanRepository;
    private final AuditService auditService;

    @Transactional // wraps in a transaction
    public Loan approveLoan(Long loanId, String approvedBy) {
        Loan loan = loanRepository.findById(loanId)
                .orElseThrow(() -> new ResourceNotFoundException("Loan", loanId));
        loan.setStatus(LoanStatus.APPROVED);
        loan.setApprovedBy(approvedBy);
        loan.setApprovedAt(Instant.now());
        // dirty checking → UPDATE on commit
        auditService.log("LOAN_APPROVED", loanId, approvedBy);
        return loan;
        // commit here — both loan update and audit log in same transaction
    }

    @Transactional(readOnly = true)
    public Page<Loan> searchLoans(LoanSearchCriteria criteria, Pageable pageable) {
        return loanRepository.findAll(criteria.toSpecification(), pageable);
    }
}
```

**Key rules about proxy-based @Transactional:**

```java
// ❌ RULE 1: Self-invocation bypasses proxy
@Service
public class LoanService {

    @Transactional
    public void processLoan(Long id) {
        validate(id);       // ← NOT transactional (direct call, no proxy)
        approveLoan(id);    // ← NOT in its own transaction
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validate(Long id) { ... }

    @Transactional
    public void approveLoan(Long id) { ... }
}

// ❌ RULE 2: @Transactional on private/protected methods is silently ignored
@Transactional
private void doWork() { } // IGNORED — proxy can't override private methods

// ❌ RULE 3: Checked exceptions don't trigger rollback by default
@Transactional
public void riskyOperation() throws BusinessException {
    // ...
    throw new BusinessException("Oops"); // transaction COMMITS, not rolls back!
}

// ✅ FIX:
@Transactional(rollbackFor = Exception.class)
public void riskyOperation() throws BusinessException { ... }
```

> **Interview Tip:** Always mention: (1) proxy-based — self-invocation doesn't work, (2) only public methods, (3) checked exceptions don't rollback by default.

---

### Q10. Explain Transaction Propagation levels with real examples. [Hard]

| Propagation | Existing TX? | Behavior | Real-World Example |
|---|---|---|---|
| `REQUIRED` (default) | Yes → Join it; No → Create new | Most common, safe default | Service methods |
| `REQUIRES_NEW` | Always creates new (suspends existing) | Independent transaction | Audit logging (must persist even if parent fails) |
| `NESTED` | Yes → Savepoint; No → Create new | Partial rollback to savepoint | Batch item processing |
| `SUPPORTS` | Yes → Join; No → Run without TX | Flexible | Read operations that work with or without TX |
| `NOT_SUPPORTED` | Yes → Suspend; No → Run without TX | Forces non-transactional | Heavy reads that shouldn't hold TX |
| `MANDATORY` | Yes → Join; No → THROW exception | Enforces calling code has TX | Internal methods that require TX context |
| `NEVER` | Yes → THROW exception; No → Run without TX | Enforces no TX | Sanity check |

```java
// REQUIRED (default) — most service methods
@Service
public class LoanService {

    @Transactional // REQUIRED by default
    public void approveLoan(Long loanId) {
        Loan loan = loanRepository.findById(loanId).orElseThrow();
        loan.setStatus(LoanStatus.APPROVED);
        notificationService.sendApproval(loanId); // joins same transaction
    }
}

// REQUIRES_NEW — audit log must be saved even if main transaction fails
@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAction(String action, Long entityId, String userId) {
        AuditLog log = new AuditLog(action, entityId, userId, Instant.now());
        auditRepository.save(log);
        // This runs in its OWN transaction
        // Even if the caller's transaction rolls back, this audit log is saved
    }
}

// Use case flow:
@Transactional
public void processPayment(PaymentRequest request) {
    try {
        paymentGateway.charge(request);          // in main TX
        orderService.updateStatus(request);       // in main TX
    } catch (PaymentException e) {
        // main TX will rollback, but audit log is already committed
        throw e;
    } finally {
        auditService.logAction("PAYMENT_ATTEMPT", request.getOrderId(), request.getUserId());
    }
}

// NESTED — batch processing with partial rollback
@Transactional
public BatchResult processBatch(List<LoanApplication> applications) {
    BatchResult result = new BatchResult();
    for (LoanApplication app : applications) {
        try {
            processOne(app); // each item in a savepoint
            result.addSuccess(app.getId());
        } catch (Exception e) {
            result.addFailure(app.getId(), e.getMessage());
            // only this item's savepoint rolls back, batch continues
        }
    }
    return result;
}

@Transactional(propagation = Propagation.NESTED)
public void processOne(LoanApplication app) {
    // runs within a savepoint of the parent transaction
    validateCredit(app);
    calculateEmi(app);
    // if this throws, savepoint rolls back but parent TX continues
}

// MANDATORY — enforce that caller provides a transaction
@Transactional(propagation = Propagation.MANDATORY)
public void debitAccount(Long accountId, BigDecimal amount) {
    // This MUST be called within an existing transaction
    // If called without TX → throws IllegalTransactionStateException
    Account account = accountRepository.findById(accountId).orElseThrow();
    account.debit(amount);
}
```

> **Interview Tip:** "In a lending platform, I use `REQUIRED` for 90% of cases, `REQUIRES_NEW` for audit logging and notifications that must persist regardless, and `NESTED` for batch processing where individual failures shouldn't fail the entire batch."

---

### Q11. Explain Transaction Isolation levels and read anomalies. [Hard]

**Read Anomalies:**

| Anomaly | Description | Example |
|---|---|---|
| **Dirty Read** | Reading uncommitted data from another TX | TX1 updates balance to 0, TX2 reads 0, TX1 rolls back (balance was never 0) |
| **Non-Repeatable Read** | Same query returns different data because another TX committed an UPDATE | TX1 reads balance=1000, TX2 updates to 500 and commits, TX1 reads again → 500 |
| **Phantom Read** | Same query returns different ROWS because another TX committed an INSERT/DELETE | TX1 counts 10 loans, TX2 inserts a new loan and commits, TX1 counts again → 11 |

**Isolation Levels vs Anomalies:**

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | ⚠️ Possible | ⚠️ Possible | ⚠️ Possible | Fastest |
| `READ_COMMITTED` (default in PostgreSQL) | ✅ Prevented | ⚠️ Possible | ⚠️ Possible | Fast |
| `REPEATABLE_READ` (default in MySQL InnoDB) | ✅ Prevented | ✅ Prevented | ⚠️ Possible | Medium |
| `SERIALIZABLE` | ✅ Prevented | ✅ Prevented | ✅ Prevented | Slowest |

```java
// Scenario: In a lending platform, checking credit limit before disbursement
// Must read committed data to avoid disbursing based on a rolled-back credit update

@Transactional(isolation = Isolation.READ_COMMITTED)
public void disburseLoan(Long loanId) {
    Loan loan = loanRepository.findById(loanId).orElseThrow();
    CreditLimit limit = creditService.getCurrentLimit(loan.getBorrowerId());
    if (loan.getAmount().compareTo(limit.getAvailable()) > 0) {
        throw new InsufficientCreditException(loan.getAmount(), limit.getAvailable());
    }
    disbursementService.transfer(loan);
}

// Scenario: Financial report that needs consistent snapshot
@Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
public FinancialReport generateMonthlyReport(YearMonth month) {
    BigDecimal totalDisbursed = loanRepository.sumDisbursedInMonth(month);
    BigDecimal totalRepaid = repaymentRepository.sumPaidInMonth(month);
    BigDecimal totalOutstanding = loanRepository.sumOutstanding();
    // All three queries see the SAME snapshot — consistent report
    return new FinancialReport(totalDisbursed, totalRepaid, totalOutstanding);
}

// Scenario: Account balance deduction — must be serialized
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    Account from = accountRepository.findById(fromAccountId).orElseThrow();
    Account to = accountRepository.findById(toAccountId).orElseThrow();
    from.debit(amount);
    to.credit(amount);
    // SERIALIZABLE ensures no concurrent transfer can read stale balance
}
```

> **Anti-Pattern:** Don't blindly use `SERIALIZABLE` — it dramatically reduces throughput due to locking. Use optimistic locking (`@Version`) for most concurrency scenarios.

---

### Q12. Explain transaction timeout and rollbackFor. [Medium]

```java
@Transactional(
    timeout = 30,                             // seconds — rolls back if exceeds 30s
    rollbackFor = Exception.class,             // rollback on ALL exceptions (including checked)
    noRollbackFor = EmailDeliveryException.class // except this one
)
public void processLoanApplication(LoanApplicationRequest request) {
    Loan loan = createLoan(request);
    creditCheck(loan);       // may throw CreditCheckException (checked)
    calculateEmi(loan);
    try {
        emailService.sendConfirmation(loan); // EmailDeliveryException should NOT rollback
    } catch (EmailDeliveryException e) {
        log.warn("Email failed, will retry later: {}", e.getMessage());
    }
}
```

**Default rollback behavior:**
- `RuntimeException` (unchecked) → **Rollback**
- `Error` → **Rollback**
- `Exception` (checked) → **Commit** (this surprises many developers!)

```java
// ❌ Common mistake: checked exception → TX commits despite the error
@Transactional
public void riskyOperation() throws IOException {
    repository.save(entity);
    fileService.upload(file); // throws IOException
    // IOException is checked → TX COMMITS, entity is saved
    // But the file upload failed — inconsistent state!
}

// ✅ Fix: explicitly include checked exceptions
@Transactional(rollbackFor = { IOException.class, BusinessException.class })
public void riskyOperation() throws IOException { ... }

// ✅ Better fix: wrap in unchecked exception
@Transactional
public void riskyOperation() {
    try {
        fileService.upload(file);
    } catch (IOException e) {
        throw new FileUploadException("Upload failed", e); // RuntimeException → rollback
    }
}
```

---

## 4. Locking

---

### Q13. Explain Optimistic Locking with @Version. [Hard]

Optimistic locking assumes conflicts are rare. It detects concurrent modifications at commit time using a version column.

**How it works:**
1. Entity has a `@Version` field (int/long/timestamp).
2. Every UPDATE includes `WHERE version = ?` and increments the version.
3. If another transaction already incremented it, the WHERE matches 0 rows → `OptimisticLockException`.

```java
@Entity
public class Loan {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private BigDecimal amount;
    private LoanStatus status;

    @Version
    private Long version; // Hibernate manages this automatically
}
```

```
Transaction A: SELECT * FROM loan WHERE id = 1   → version = 5
Transaction B: SELECT * FROM loan WHERE id = 1   → version = 5

Transaction A: UPDATE loan SET status='APPROVED', version=6 WHERE id=1 AND version=5
  → 1 row updated ✅ (version is now 6)

Transaction B: UPDATE loan SET status='REJECTED', version=6 WHERE id=1 AND version=5
  → 0 rows updated ❌ → OptimisticLockException!
```

```java
@Service
public class LoanService {

    @Transactional
    public Loan updateLoanAmount(Long loanId, BigDecimal newAmount) {
        Loan loan = loanRepository.findById(loanId).orElseThrow();
        loan.setAmount(newAmount);
        return loanRepository.save(loan);
        // If concurrent modification: OptimisticLockException thrown at flush/commit
    }
}

// Handle in controller/advice
@RestControllerAdvice
public class LockingExceptionHandler {

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ProblemDetail handleOptimisticLock(OptimisticLockingFailureException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        pd.setTitle("Concurrent Modification");
        pd.setDetail("This record was modified by another user. Please refresh and try again.");
        return pd;
    }
}

// Retry pattern for optimistic lock
@Retryable(
    retryFor = OptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional
public Loan updateWithRetry(Long loanId, BigDecimal newAmount) {
    Loan loan = loanRepository.findById(loanId).orElseThrow();
    loan.setAmount(newAmount);
    return loanRepository.save(loan);
}
```

**When to use Optimistic Locking:**
- Read-heavy workloads with rare conflicts
- Web applications (user editing forms)
- Short-lived transactions
- When you can afford to retry on conflict

---

### Q14. Explain Pessimistic Locking with @Lock. [Hard]

Pessimistic locking locks the database row when reading, preventing other transactions from reading/writing until the lock is released.

```java
public interface LoanRepository extends JpaRepository<Loan, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT l FROM Loan l WHERE l.id = :id")
    Optional<Loan> findByIdForUpdate(@Param("id") Long id);
    // SQL: SELECT * FROM loan WHERE id = ? FOR UPDATE

    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT l FROM Loan l WHERE l.id = :id")
    Optional<Loan> findByIdWithSharedLock(@Param("id") Long id);
    // SQL: SELECT * FROM loan WHERE id = ? FOR SHARE
}
```

| Lock Mode | SQL | Other TX can READ? | Other TX can WRITE? | Use Case |
|---|---|---|---|---|
| `PESSIMISTIC_READ` | `FOR SHARE` | ✅ Yes | ❌ Blocked | Ensure data doesn't change while you read |
| `PESSIMISTIC_WRITE` | `FOR UPDATE` | ❌ Blocked | ❌ Blocked | Exclusive access for updates |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + version++ | ❌ Blocked | ❌ Blocked | Pessimistic + version bump |

```java
// Real-world: Disbursement with exclusive lock to prevent double-disbursement
@Service
public class DisbursementService {

    @Transactional(timeout = 10)
    public DisbursementResult disburseLoan(Long loanId) {
        // Lock the row — no other thread can disburse the same loan concurrently
        Loan loan = loanRepository.findByIdForUpdate(loanId)
                .orElseThrow(() -> new ResourceNotFoundException("Loan", loanId));

        if (loan.getStatus() != LoanStatus.APPROVED) {
            throw new InvalidStateException("Loan is not in APPROVED state: " + loan.getStatus());
        }

        // Safe: we hold an exclusive lock on this row
        loan.setStatus(LoanStatus.DISBURSED);
        loan.setDisbursedAt(Instant.now());
        bankingService.transfer(loan.getBorrowerAccountId(), loan.getAmount());

        return new DisbursementResult(loan.getId(), loan.getAmount());
    }
}
```

**Pessimistic lock with timeout:**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "5000")) // 5 seconds
@Query("SELECT l FROM Loan l WHERE l.id = :id")
Optional<Loan> findByIdForUpdateWithTimeout(@Param("id") Long id);
```

**When to use Pessimistic Locking:**
- Critical financial operations (payment processing, balance deductions)
- High-contention resources
- When conflicts are frequent and retries are expensive
- When you must guarantee no concurrent modification

---

### Q15. Optimistic vs Pessimistic Locking — comparison and when to use each. [Medium]

| Aspect | Optimistic | Pessimistic |
|---|---|---|
| **Mechanism** | Version check at commit time | DB row lock at read time |
| **Lock duration** | No lock held | Lock held until TX ends |
| **Conflict detection** | At commit (late) | At read (early) |
| **Throughput** | Higher (no locking) | Lower (blocks other TX) |
| **Conflict cost** | Retry entire operation | Wait (or timeout) |
| **Deadlock risk** | None | Possible |
| **Best for** | Low contention, read-heavy | High contention, critical writes |

```
Real-world decision tree:

Is it a financial transaction (money movement)?
  → YES → Pessimistic (PESSIMISTIC_WRITE)

Is it a user editing a form (low conflict probability)?
  → YES → Optimistic (@Version)

Are conflicts frequent?
  → YES → Pessimistic
  → NO  → Optimistic

Can you afford retries?
  → YES → Optimistic with @Retryable
  → NO  → Pessimistic
```

> **Interview Tip:** "In a lending platform, I use optimistic locking for loan applications (users editing forms — rare conflicts) and pessimistic locking for disbursement processing (must prevent double-disbursement — financial risk)."

---

## 5. Performance & Advanced

---

### Q16. Explain Lazy vs Eager Loading — LazyInitializationException. [Hard]

**Default fetch types:**
| Mapping | Default | Recommended |
|---|---|---|
| `@ManyToOne` | `EAGER` | Change to `LAZY` |
| `@OneToOne` | `EAGER` | Change to `LAZY` |
| `@OneToMany` | `LAZY` | Keep `LAZY` |
| `@ManyToMany` | `LAZY` | Keep `LAZY` |

```java
@Entity
public class Loan {

    @ManyToOne(fetch = FetchType.LAZY)  // override default EAGER
    @JoinColumn(name = "borrower_id")
    private Borrower borrower;

    @OneToMany(mappedBy = "loan", fetch = FetchType.LAZY) // default
    private List<Repayment> repayments;
}
```

**LazyInitializationException:**

Occurs when you access a lazy-loaded association **outside** a Hibernate session (after the transaction ends).

```java
// ❌ LazyInitializationException
@Transactional
public Loan getLoan(Long id) {
    return loanRepository.findById(id).orElseThrow();
}

// In controller (outside transaction):
Loan loan = loanService.getLoan(1L);
loan.getRepayments().size(); // 💥 LazyInitializationException — session is closed!
```

**Solutions:**

```java
// Solution 1: Fetch eagerly in the query (JOIN FETCH)
@Query("SELECT l FROM Loan l JOIN FETCH l.repayments WHERE l.id = :id")
Optional<Loan> findByIdWithRepayments(@Param("id") Long id);

// Solution 2: @EntityGraph
@EntityGraph(attributePaths = {"repayments", "borrower"})
Optional<Loan> findById(Long id);

// Solution 3: DTO projection — don't return entities to the controller
@Transactional(readOnly = true)
public LoanDetailDTO getLoanDetail(Long id) {
    Loan loan = loanRepository.findByIdWithRepayments(id).orElseThrow();
    return LoanDetailDTO.from(loan); // map inside the transaction
}

// Solution 4: Hibernate.initialize() — force load within session
@Transactional(readOnly = true)
public Loan getLoanWithRepayments(Long id) {
    Loan loan = loanRepository.findById(id).orElseThrow();
    Hibernate.initialize(loan.getRepayments()); // triggers SELECT
    return loan;
}
```

**Open Session in View (OSIV):**
```yaml
spring:
  jpa:
    open-in-view: false  # ← DISABLE THIS IN PRODUCTION
```

> **Anti-Pattern:** Open Session in View keeps the Hibernate session open through the entire HTTP request (including view rendering). This hides N+1 problems, holds DB connections longer, and makes it hard to track where queries are generated. **Always disable OSIV** and fetch what you need in the service layer.

---

### Q17. How does pagination work in Spring Data JPA? [Medium]

```java
// Repository — already supports pagination
public interface LoanRepository extends JpaRepository<Loan, Long> {

    Page<Loan> findByStatus(LoanStatus status, Pageable pageable);

    Slice<Loan> findByBorrowerId(Long borrowerId, Pageable pageable);

    @Query("SELECT l FROM Loan l WHERE l.amount > :minAmount")
    Page<Loan> findLargeLoans(@Param("minAmount") BigDecimal minAmount, Pageable pageable);
}

// Service
@Transactional(readOnly = true)
public Page<LoanSummaryDTO> getActiveLoans(int page, int size, String sortBy) {
    Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, sortBy));
    Page<Loan> loans = loanRepository.findByStatus(LoanStatus.ACTIVE, pageable);
    return loans.map(LoanSummaryDTO::from);
}

// Controller
@GetMapping("/loans")
public Page<LoanSummaryDTO> getLoans(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy) {
    return loanService.getActiveLoans(page, size, sortBy);
}
```

**Page vs Slice:**

| Feature | `Page<T>` | `Slice<T>` |
|---|---|---|
| Total count query | ✅ Runs `SELECT COUNT(*)` | ❌ No count query |
| Total elements | ✅ `getTotalElements()` | ❌ Not available |
| Total pages | ✅ `getTotalPages()` | ❌ Not available |
| Has next | ✅ `hasNext()` | ✅ `hasNext()` (fetches size+1) |
| Performance | Slower (extra COUNT query) | Faster |
| Use case | UI with page numbers | Infinite scroll / "Load more" |

**Multi-field sorting:**
```java
Pageable pageable = PageRequest.of(0, 20,
    Sort.by(Sort.Order.desc("status"), Sort.Order.asc("createdAt")));
// SQL: ... ORDER BY status DESC, created_at ASC LIMIT 20 OFFSET 0
```

> **Interview Tip:** "For large tables (millions of rows), avoid `Page` because `COUNT(*)` is expensive. Use `Slice` or keyset pagination instead."

**Keyset pagination (cursor-based) for large datasets:**
```java
@Query("""
    SELECT l FROM Loan l
    WHERE l.status = :status AND l.id > :lastId
    ORDER BY l.id ASC
    """)
List<Loan> findNextPage(@Param("status") LoanStatus status,
                        @Param("lastId") Long lastId,
                        Pageable pageable);
```

---

### Q18. How does Spring Data JPA Auditing work? [Medium]

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .map(Authentication::getName);
    }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    @Version
    private Long version;
}

@Entity
public class Loan extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;
    private LoanStatus status;
}
```

**Spring Data Envers (history tracking):**

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-envers</artifactId>
</dependency>
```

```java
@Entity
@Audited // Hibernate Envers — tracks all changes
public class Loan extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;
    private LoanStatus status;
}

// Repository with revision support
public interface LoanRepository extends JpaRepository<Loan, Long>,
                                        RevisionRepository<Loan, Long, Integer> {
}

// Usage — find all revisions of a loan
Revisions<Integer, Loan> revisions = loanRepository.findRevisions(loanId);
revisions.forEach(revision -> {
    Loan loan = revision.getEntity();
    RevisionMetadata<Integer> metadata = revision.getMetadata();
    log.info("Revision {}: status={}, by={}, at={}",
            metadata.getRevisionNumber().orElse(0),
            loan.getStatus(),
            loan.getUpdatedBy(),
            metadata.getRevisionInstant().orElse(null));
});
```

---

### Q19. How to write custom queries — @Query, Specifications, Criteria API? [Medium]

#### 1. Derived Query Methods

```java
public interface LoanRepository extends JpaRepository<Loan, Long> {

    List<Loan> findByStatusAndAmountGreaterThan(LoanStatus status, BigDecimal amount);
    // SELECT * FROM loan WHERE status = ? AND amount > ?

    Optional<Loan> findFirstByBorrowerIdOrderByCreatedAtDesc(Long borrowerId);
    // SELECT * FROM loan WHERE borrower_id = ? ORDER BY created_at DESC LIMIT 1

    long countByStatus(LoanStatus status);

    boolean existsByBorrowerIdAndStatus(Long borrowerId, LoanStatus status);

    @Modifying
    @Query("UPDATE Loan l SET l.status = :status WHERE l.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") LoanStatus status);
}
```

#### 2. @Query (JPQL and Native)

```java
// JPQL
@Query("SELECT l FROM Loan l WHERE l.borrower.creditScore > :minScore AND l.status = :status")
List<Loan> findQualifiedLoans(@Param("minScore") int minScore,
                               @Param("status") LoanStatus status);

// Native SQL
@Query(value = """
    SELECT l.*, b.name as borrower_name
    FROM loan l
    JOIN borrower b ON l.borrower_id = b.id
    WHERE l.amount > :amount
    AND l.created_at > :since
    ORDER BY l.amount DESC
    """, nativeQuery = true)
List<Loan> findLargeRecentLoans(@Param("amount") BigDecimal amount,
                                 @Param("since") LocalDateTime since);

// DTO projection with JPQL
@Query("""
    SELECT new com.lending.dto.LoanSummary(l.id, l.amount, l.status, b.name)
    FROM Loan l JOIN l.borrower b
    WHERE l.status IN :statuses
    """)
List<LoanSummary> findLoanSummaries(@Param("statuses") List<LoanStatus> statuses);
```

#### 3. Specifications (dynamic queries)

```java
// Specification builder
public class LoanSpecifications {

    public static Specification<Loan> hasStatus(LoanStatus status) {
        return (root, query, cb) -> status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<Loan> amountBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min != null && max != null) return cb.between(root.get("amount"), min, max);
            if (min != null) return cb.greaterThanOrEqualTo(root.get("amount"), min);
            return cb.lessThanOrEqualTo(root.get("amount"), max);
        };
    }

    public static Specification<Loan> borrowerNameContains(String name) {
        return (root, query, cb) -> name == null ? null :
                cb.like(cb.lower(root.join("borrower").get("name")),
                        "%" + name.toLowerCase() + "%");
    }

    public static Specification<Loan> createdAfter(LocalDate date) {
        return (root, query, cb) -> date == null ? null :
                cb.greaterThanOrEqualTo(root.get("createdAt"), date.atStartOfDay());
    }
}

// Repository must extend JpaSpecificationExecutor
public interface LoanRepository extends JpaRepository<Loan, Long>,
                                        JpaSpecificationExecutor<Loan> { }

// Service — compose specs dynamically
@Transactional(readOnly = true)
public Page<Loan> searchLoans(LoanSearchRequest request, Pageable pageable) {
    Specification<Loan> spec = Specification
            .where(LoanSpecifications.hasStatus(request.getStatus()))
            .and(LoanSpecifications.amountBetween(request.getMinAmount(), request.getMaxAmount()))
            .and(LoanSpecifications.borrowerNameContains(request.getBorrowerName()))
            .and(LoanSpecifications.createdAfter(request.getCreatedAfter()));

    return loanRepository.findAll(spec, pageable);
}
```

> **Interview Tip:** "I prefer Specifications for dynamic search/filter APIs. They're composable, testable, and type-safe. For static queries, `@Query` with JPQL is simpler."

---

### Q20. Explain batch inserts and bulk operations for performance. [Hard]

**Problem:** Inserting 10,000 entities one-by-one is extremely slow.

```java
// ❌ Slow — 10,000 individual INSERTs
@Transactional
public void importRepayments(List<RepaymentData> data) {
    data.forEach(d -> {
        Repayment r = new Repayment(d.getLoanId(), d.getAmount(), d.getDueDate());
        repaymentRepository.save(r); // 10,000 INSERTs, 10,000 flushes
    });
}
```

**Solution 1: JDBC Batch with Hibernate**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          batch_versioned_data: true
        order_inserts: true
        order_updates: true
```

```java
// ✅ Batch insert with flush+clear pattern
@Transactional
public void importRepaymentsBatch(List<RepaymentData> dataList) {
    int batchSize = 50;
    for (int i = 0; i < dataList.size(); i++) {
        RepaymentData d = dataList.get(i);
        Repayment r = new Repayment(d.getLoanId(), d.getAmount(), d.getDueDate());
        entityManager.persist(r);

        if (i > 0 && i % batchSize == 0) {
            entityManager.flush();  // execute batched INSERTs
            entityManager.clear();  // free memory (detach all managed entities)
        }
    }
    entityManager.flush();
    entityManager.clear();
}
```

**Important:** For batch inserts, use `SEQUENCE` or `TABLE` ID generation (not `IDENTITY`). `IDENTITY` strategy disables JDBC batching because Hibernate needs the generated ID immediately.

```java
@Entity
public class Repayment {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "repayment_seq")
    @SequenceGenerator(name = "repayment_seq", sequenceName = "repayment_id_seq", allocationSize = 50)
    private Long id;
}
```

**Solution 2: JdbcTemplate for ultra-fast bulk inserts**

```java
@Repository
public class RepaymentBulkRepository {

    private final JdbcTemplate jdbcTemplate;

    public RepaymentBulkRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public void bulkInsert(List<RepaymentData> dataList) {
        String sql = "INSERT INTO repayment (loan_id, amount, due_date, status) VALUES (?, ?, ?, ?)";
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                RepaymentData d = dataList.get(i);
                ps.setLong(1, d.getLoanId());
                ps.setBigDecimal(2, d.getAmount());
                ps.setDate(3, Date.valueOf(d.getDueDate()));
                ps.setString(4, "PENDING");
            }

            @Override
            public int getBatchSize() {
                return dataList.size();
            }
        });
    }
}
```

**Solution 3: Bulk UPDATE/DELETE (skip dirty checking)**

```java
// Bulk update — bypasses entity lifecycle (no events, no dirty checking)
@Modifying(clearAutomatically = true) // clear L1 cache after modification
@Query("UPDATE Loan l SET l.status = :newStatus WHERE l.status = :oldStatus AND l.dueDate < :cutoff")
int markOverdueLoans(@Param("newStatus") LoanStatus newStatus,
                     @Param("oldStatus") LoanStatus oldStatus,
                     @Param("cutoff") LocalDate cutoff);

// Bulk delete
@Modifying(clearAutomatically = true)
@Query("DELETE FROM AuditLog a WHERE a.createdAt < :before")
int purgeOldAuditLogs(@Param("before") Instant before);
```

> **Anti-Pattern:** Using `saveAll()` without configuring batch size — Spring Data calls `save()` in a loop, which generates individual INSERTs. Configure `hibernate.jdbc.batch_size` and use `SEQUENCE` ID generation.

---

### Q21. How to use interface-based projections in Spring Data JPA? [Easy]

```java
// Closed projection — only selected fields
public interface LoanSummaryView {
    Long getId();
    BigDecimal getAmount();
    LoanStatus getStatus();

    @Value("#{target.amount.multiply(target.interestRate)}")
    BigDecimal getInterestAmount(); // computed via SpEL
}

// Usage — Spring generates a proxy, only selected columns fetched
public interface LoanRepository extends JpaRepository<Loan, Long> {

    List<LoanSummaryView> findByStatus(LoanStatus status);
    // SQL: SELECT id, amount, status, interest_rate FROM loan WHERE status = ?
    // (only needed columns, no full entity load)

    <T> List<T> findByBorrowerId(Long borrowerId, Class<T> type);
    // Dynamic projection — choose at runtime!
}

// Service
public List<LoanSummaryView> getActiveLoanSummaries() {
    return loanRepository.findByStatus(LoanStatus.ACTIVE);
}

// Or dynamically:
public <T> List<T> getLoansByBorrower(Long borrowerId, Class<T> projectionType) {
    return loanRepository.findByBorrowerId(borrowerId, projectionType);
}
```

> **Interview Tip:** "Interface projections are great for read-only API responses. They reduce memory footprint and network transfer by only loading needed columns."

---

### Q22. How to handle soft deletes in Spring Data JPA? [Medium]

```java
@Entity
@SQLRestriction("deleted = false") // Hibernate 6.3+ (replaces @Where)
public class Loan {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private BigDecimal amount;
    private LoanStatus status;

    private boolean deleted = false;

    @Column(name = "deleted_at")
    private Instant deletedAt;

    @Column(name = "deleted_by")
    private String deletedBy;

    public void softDelete(String deletedBy) {
        this.deleted = true;
        this.deletedAt = Instant.now();
        this.deletedBy = deletedBy;
    }
}
```

```java
// Repository — @SQLRestriction automatically filters deleted entities
public interface LoanRepository extends JpaRepository<Loan, Long> {

    // This automatically includes WHERE deleted = false
    List<Loan> findByStatus(LoanStatus status);

    // Override to include deleted
    @Query("SELECT l FROM Loan l WHERE l.id = :id")
    @SQLRestriction("") // bypass restriction for this query
    Optional<Loan> findByIdIncludingDeleted(@Param("id") Long id);
}

// Service
@Transactional
public void deleteLoan(Long loanId, String userId) {
    Loan loan = loanRepository.findById(loanId).orElseThrow();
    loan.softDelete(userId); // sets deleted=true, doesn't remove from DB
}
```

> **Anti-Pattern:** Hard deleting financial records. In a lending platform, you should **always** soft-delete loans, repayments, and transactions for audit and compliance reasons.

---

### Q23. Explain Spring Data JPA repository hierarchy. [Easy]

```
Repository<T, ID>                    (marker interface, no methods)
    │
    └── CrudRepository<T, ID>        (save, findById, findAll, delete, count, existsById)
            │
            └── ListCrudRepository   (returns List instead of Iterable)
            │
            └── PagingAndSortingRepository  (findAll(Pageable), findAll(Sort))
                    │
                    └── JpaRepository<T, ID>  (flush, saveAndFlush, deleteInBatch, getById)

JpaSpecificationExecutor<T>          (findAll(Specification), count(Specification))
QueryByExampleExecutor<T>            (findAll(Example))
```

```java
// Minimal — just basic CRUD
public interface LoanRepository extends CrudRepository<Loan, Long> { }

// Full JPA features — most common choice
public interface LoanRepository extends JpaRepository<Loan, Long>,
                                        JpaSpecificationExecutor<Loan> { }

// Custom repository for complex logic
public interface LoanRepositoryCustom {
    List<LoanAnalytics> calculateAnalytics(AnalyticsCriteria criteria);
}

public class LoanRepositoryCustomImpl implements LoanRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<LoanAnalytics> calculateAnalytics(AnalyticsCriteria criteria) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        // ... complex criteria query
    }
}

// Combine standard + custom
public interface LoanRepository extends JpaRepository<Loan, Long>,
                                        JpaSpecificationExecutor<Loan>,
                                        LoanRepositoryCustom { }
```

---

## Quick Reference — Common Annotations

| Annotation | Purpose |
|---|---|
| `@Entity` | Marks class as JPA entity |
| `@Table` | Customize table name |
| `@Id` + `@GeneratedValue` | Primary key with auto-generation |
| `@Column` | Customize column mapping |
| `@ManyToOne` / `@OneToMany` | Relationship mapping |
| `@JoinColumn` | Foreign key column |
| `@Version` | Optimistic locking |
| `@CreatedDate` / `@LastModifiedDate` | Audit timestamps |
| `@Transactional` | Declarative transaction |
| `@Modifying` | For UPDATE/DELETE queries |
| `@Query` | Custom JPQL or native SQL |
| `@EntityGraph` | Fetch plan to avoid N+1 |
| `@BatchSize` | Batch lazy loading |
| `@Lock` | Pessimistic locking |
| `@SQLRestriction` | Row-level filter (soft deletes) |
| `@Audited` (Envers) | Track entity history |
| `@Cache` | Second-level cache config |

---

## Performance Checklist

- [ ] Set `spring.jpa.open-in-view=false`
- [ ] All `@ManyToOne` and `@OneToOne` set to `fetch = FetchType.LAZY`
- [ ] Configure `hibernate.default_batch_fetch_size=25`
- [ ] Use DTO projections for API responses instead of returning entities
- [ ] Use `@Transactional(readOnly = true)` for read operations
- [ ] Enable Hibernate statistics in dev to detect N+1
- [ ] Use `@EntityGraph` or `JOIN FETCH` for known eager-fetch scenarios
- [ ] Configure JDBC batch size for bulk operations
- [ ] Use `SEQUENCE` ID generation (not `IDENTITY`) for batch inserts
- [ ] Index foreign keys and frequently-queried columns
- [ ] Use pagination (`Pageable`) for list endpoints — never return unbounded lists
- [ ] Avoid returning JPA entities directly in REST responses — use DTOs

---

*Last updated: March 2026*
