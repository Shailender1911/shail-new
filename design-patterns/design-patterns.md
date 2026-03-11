# Design Patterns — SDE-2 Interview Guide (Lending/Fintech Domain)

---

## CREATIONAL PATTERNS

---

### 1. Singleton Pattern

**Problem it solves:** Ensure a class has exactly one instance and provide a global access point to it. Useful for shared resources like configuration, connection pools, or caches.

**UML (text):**
```
+----------------------------+
|        Singleton           |
+----------------------------+
| - instance: Singleton      |
+----------------------------+
| - Singleton()              |
| + getInstance(): Singleton |
+----------------------------+
```

**Java — Thread-safe Enum Singleton (recommended):**

```java
public enum AppConfig {
    INSTANCE;

    private final Properties properties = new Properties();

    AppConfig() {
        try (InputStream is = getClass().getResourceAsStream("/config.properties")) {
            properties.load(is);
        } catch (IOException e) {
            throw new RuntimeException("Failed to load config", e);
        }
    }

    public String get(String key) {
        return properties.getProperty(key);
    }
}

// Usage
String dbUrl = AppConfig.INSTANCE.get("db.url");
```

**Java — Double-checked Locking (classic approach):**

```java
public class ConnectionPoolManager {
    private static volatile ConnectionPoolManager instance;
    private final HikariDataSource dataSource;

    private ConnectionPoolManager() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/lending_db");
        config.setMaximumPoolSize(10);
        this.dataSource = new HikariDataSource(config);
    }

    public static ConnectionPoolManager getInstance() {
        if (instance == null) {                    // first check (no lock)
            synchronized (ConnectionPoolManager.class) {
                if (instance == null) {            // second check (with lock)
                    instance = new ConnectionPoolManager();
                }
            }
        }
        return instance;
    }

    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

**Why `volatile`?** Without it, the JVM can reorder instructions — thread B could see a partially constructed object.

**Lending domain use case:** Configuration manager, connection pool, rate limiter for partner API calls.

**Spring Framework usage:** All Spring beans are singletons by default (`@Scope("singleton")`). The Spring container manages the lifecycle — you almost never need to write Singleton manually in Spring.

**When to use:** Shared resources that are expensive to create (DB pool, cache, config).  
**When NOT to use:** When you need multiple instances, or when the singleton holds mutable shared state that causes contention. Singletons can hide dependencies and make testing harder.

---

### 2. Factory Method Pattern

**Problem it solves:** Delegate object creation to subclasses or a factory. The client code doesn't know the exact class being instantiated — it works with the interface.

**UML (text):**
```
+------------------+          +------------------+
|  NachProcessor   |<---------|  NachFactory     |
|  <<interface>>   |          +------------------+
+------------------+          | + create(type):  |
| + process()      |          |   NachProcessor  |
+------------------+          +------------------+
        ^
        |
   +----+--------+----------+
   |             |            |
+----------+ +----------+ +------------+
| UpiNach  | | ApiNach  | | PhysicalNach|
+----------+ +----------+ +------------+
```

**Java:**

```java
public interface NachProcessor {
    MandateResult processMandate(MandateRequest request);
    String getType();
}

@Component
public class UpiNachProcessor implements NachProcessor {
    @Override
    public MandateResult processMandate(MandateRequest request) {
        // Call UPI NACH API
        return new MandateResult(Status.SUCCESS, "UPI mandate registered");
    }

    @Override
    public String getType() { return "UPI_NACH"; }
}

@Component
public class ApiNachProcessor implements NachProcessor {
    @Override
    public MandateResult processMandate(MandateRequest request) {
        // Call API NACH endpoint
        return new MandateResult(Status.SUCCESS, "API mandate registered");
    }

    @Override
    public String getType() { return "API_NACH"; }
}

@Component
public class PhysicalNachProcessor implements NachProcessor {
    @Override
    public MandateResult processMandate(MandateRequest request) {
        // Generate physical NACH form
        return new MandateResult(Status.PENDING, "Physical form generated");
    }

    @Override
    public String getType() { return "PHYSICAL_NACH"; }
}

@Component
public class NachProcessorFactory {
    private final Map<String, NachProcessor> processors;

    public NachProcessorFactory(List<NachProcessor> processorList) {
        this.processors = processorList.stream()
            .collect(Collectors.toMap(NachProcessor::getType, Function.identity()));
    }

    public NachProcessor getProcessor(String type) {
        NachProcessor processor = processors.get(type);
        if (processor == null) {
            throw new IllegalArgumentException("Unknown NACH type: " + type);
        }
        return processor;
    }
}

// Client code
@Service
public class MandateService {
    private final NachProcessorFactory factory;

    public MandateResult registerMandate(MandateRequest request) {
        NachProcessor processor = factory.getProcessor(request.getNachType());
        return processor.processMandate(request);
    }
}
```

**Lending domain use case:** Different NACH types (UPI NACH, API NACH, Physical NACH), different insurance vendors (ICICI Lombard, Acko), different disbursement channels.

**Spring Framework usage:** `BeanFactory` is a factory for Spring beans. `FactoryBean<T>` lets you customize bean creation.

**When to use:** When exact types are determined at runtime, or you want to decouple creation from usage.  
**When NOT to use:** When there's only one implementation and no foreseeable need for variants.

---

### 3. Abstract Factory Pattern

**Problem it solves:** Create families of related objects without specifying concrete classes. Ensures objects from the same family are used together.

**UML (text):**
```
+---------------------+
| LendingUIFactory     |
| <<interface>>        |
+---------------------+
| + createForm()       |
| + createAgreement()  |
| + createSigner()     |
+---------------------+
         ^
    +----+----+
    |         |
+--------+ +--------+
| ICICI  | | HDFC   |
| Factory| | Factory|
+--------+ +--------+
```

**Java:**

```java
public interface LoanApplicationForm {
    byte[] generate(LoanApplication app);
}

public interface LoanAgreement {
    byte[] generateAgreement(LoanApplication app);
}

public interface DigitalSigner {
    SignedDocument sign(byte[] document, String aadhaarNumber);
}

public interface LendingDocumentFactory {
    LoanApplicationForm createApplicationForm();
    LoanAgreement createAgreement();
    DigitalSigner createSigner();
}

public class IciciDocumentFactory implements LendingDocumentFactory {
    public LoanApplicationForm createApplicationForm() { return new IciciApplicationForm(); }
    public LoanAgreement createAgreement() { return new IciciLoanAgreement(); }
    public DigitalSigner createSigner() { return new IciciDigitalSigner(); }
}

public class HdfcDocumentFactory implements LendingDocumentFactory {
    public LoanApplicationForm createApplicationForm() { return new HdfcApplicationForm(); }
    public LoanAgreement createAgreement() { return new HdfcLoanAgreement(); }
    public DigitalSigner createSigner() { return new HdfcDigitalSigner(); }
}

// Client code uses the factory — never knows about ICICI or HDFC specifics
public class LoanDocumentService {
    private final LendingDocumentFactory factory;

    public LoanDocumentService(LendingDocumentFactory factory) {
        this.factory = factory;
    }

    public SignedDocument processDocuments(LoanApplication app) {
        byte[] form = factory.createApplicationForm().generate(app);
        byte[] agreement = factory.createAgreement().generateAgreement(app);
        return factory.createSigner().sign(agreement, app.getAadhaarNumber());
    }
}
```

**Lending domain use case:** Each lending partner (ICICI, HDFC) has its own set of forms, agreements, and signing mechanisms. Abstract Factory ensures you always use a consistent set.

**When to use:** When you have families of related objects that must be used together.  
**When NOT to use:** When object families aren't related or compatibility between them isn't a concern.

---

### 4. Builder Pattern

**Problem it solves:** Construct complex objects step by step. Avoids telescoping constructors (constructors with many parameters).

**UML (text):**
```
+---------------------------+
|     LoanApplication       |
+---------------------------+
| - customerId: String      |
| - amount: BigDecimal      |
| - tenure: int             |
| - interestRate: double    |
| - product: LoanProduct    |
| - documents: List<Doc>    |
+---------------------------+
| + builder(): Builder      |
+---------------------------+
       |
+---------------------------+
|     Builder (static)      |
+---------------------------+
| + customerId(String)      |
| + amount(BigDecimal)      |
| + tenure(int)             |
| + build(): LoanApplication|
+---------------------------+
```

**Java (manual builder):**

```java
public class LoanApplication {
    private final String customerId;
    private final BigDecimal amount;
    private final int tenureMonths;
    private final double interestRate;
    private final LoanProduct product;
    private final List<Document> documents;

    private LoanApplication(Builder builder) {
        this.customerId = builder.customerId;
        this.amount = builder.amount;
        this.tenureMonths = builder.tenureMonths;
        this.interestRate = builder.interestRate;
        this.product = builder.product;
        this.documents = builder.documents;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String customerId;
        private BigDecimal amount;
        private int tenureMonths;
        private double interestRate;
        private LoanProduct product;
        private List<Document> documents = new ArrayList<>();

        public Builder customerId(String customerId) {
            this.customerId = customerId;
            return this;
        }

        public Builder amount(BigDecimal amount) {
            this.amount = amount;
            return this;
        }

        public Builder tenureMonths(int tenureMonths) {
            this.tenureMonths = tenureMonths;
            return this;
        }

        public Builder interestRate(double interestRate) {
            this.interestRate = interestRate;
            return this;
        }

        public Builder product(LoanProduct product) {
            this.product = product;
            return this;
        }

        public Builder addDocument(Document document) {
            this.documents.add(document);
            return this;
        }

        public LoanApplication build() {
            Objects.requireNonNull(customerId, "customerId is required");
            Objects.requireNonNull(amount, "amount is required");
            if (tenureMonths <= 0) throw new IllegalArgumentException("tenure must be positive");
            return new LoanApplication(this);
        }
    }
}

// Usage
LoanApplication app = LoanApplication.builder()
    .customerId("CUST-12345")
    .amount(new BigDecimal("500000"))
    .tenureMonths(36)
    .interestRate(12.5)
    .product(LoanProduct.PERSONAL_LOAN)
    .addDocument(new Document("AADHAAR", aadhaarFile))
    .addDocument(new Document("PAN", panFile))
    .build();
```

**Lombok shortcut:**

```java
@Builder
@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class LoanApplication {
    private final String customerId;
    private final BigDecimal amount;
    private final int tenureMonths;
    private final double interestRate;
    private final LoanProduct product;
    @Singular private final List<Document> documents;
}
```

**Lending domain use case:** Loan applications have 10-15 fields, some optional, some requiring validation. Builder provides readable construction with validation in `build()`.

**Spring Framework usage:** `UriComponentsBuilder`, `ResponseEntity.ok().body(...)`, `WebClient.builder()`.

**When to use:** Objects with many fields, especially when some are optional.  
**When NOT to use:** Simple objects with 2-3 fields — a constructor suffices.

---

## STRUCTURAL PATTERNS

---

### 5. Adapter Pattern

**Problem it solves:** Make incompatible interfaces work together. Wraps a third-party or legacy class to match the interface your code expects.

**UML (text):**
```
+------------------+     +------------------+     +-------------------+
| PaymentGateway   |<----| RazorpayAdapter  |---->| RazorpaySDK       |
| <<interface>>    |     +------------------+     | (3rd party)       |
+------------------+     | + charge()       |     +-------------------+
| + charge()       |     | + refund()       |     | + createPayment() |
| + refund()       |     +------------------+     | + initiateRefund()|
+------------------+                              +-------------------+
```

**Java:**

```java
public interface PaymentGateway {
    PaymentResponse charge(String orderId, BigDecimal amount, String currency);
    RefundResponse refund(String transactionId, BigDecimal amount);
}

// Third-party SDK with its own API shape
public class RazorpayClient {
    public RzpPayment createPayment(RzpPaymentRequest request) { /* ... */ }
    public RzpRefund initiateRefund(String paymentId, int amountInPaise) { /* ... */ }
}

@Component
public class RazorpayAdapter implements PaymentGateway {
    private final RazorpayClient razorpayClient;

    public RazorpayAdapter(RazorpayClient razorpayClient) {
        this.razorpayClient = razorpayClient;
    }

    @Override
    public PaymentResponse charge(String orderId, BigDecimal amount, String currency) {
        RzpPaymentRequest rzpRequest = new RzpPaymentRequest();
        rzpRequest.setOrderId(orderId);
        rzpRequest.setAmountInPaise(amount.multiply(new BigDecimal(100)).intValue());
        rzpRequest.setCurrency(currency);

        RzpPayment rzpPayment = razorpayClient.createPayment(rzpRequest);

        return new PaymentResponse(rzpPayment.getId(), mapStatus(rzpPayment.getStatus()));
    }

    @Override
    public RefundResponse refund(String transactionId, BigDecimal amount) {
        int amountInPaise = amount.multiply(new BigDecimal(100)).intValue();
        RzpRefund rzpRefund = razorpayClient.initiateRefund(transactionId, amountInPaise);
        return new RefundResponse(rzpRefund.getId(), mapRefundStatus(rzpRefund.getStatus()));
    }

    private PaymentStatus mapStatus(String rzpStatus) {
        return switch (rzpStatus) {
            case "captured" -> PaymentStatus.SUCCESS;
            case "failed" -> PaymentStatus.FAILED;
            default -> PaymentStatus.PENDING;
        };
    }
}
```

**Lending domain use case:** Each payment partner (Razorpay, PayU, Cashfree) has a different SDK API. Adapters convert their APIs to your unified `PaymentGateway` interface.

**Spring Framework usage:** `HandlerAdapter` in Spring MVC adapts different handler types to a common invocation interface.

**When to use:** Integrating with third-party APIs, legacy systems, or any code with incompatible interfaces.  
**When NOT to use:** When you control both interfaces and can simply change one.

---

### 6. Decorator Pattern

**Problem it solves:** Add behavior to objects dynamically without modifying the original class. Wraps the original object and adds new functionality before/after delegating.

**UML (text):**
```
+---------------------+
| LoanService         |
| <<interface>>       |
+---------------------+
| + applyLoan()       |
+---------------------+
        ^       ^
        |       |
+----------+ +----------------------+
| BaseLoan | | LoggingDecorator     |
| Service  | +----------------------+
+----------+ | - wrapped: LoanSvc  |
             | + applyLoan()       |
             +----------------------+
```

**Java:**

```java
public interface NotificationService {
    void send(String userId, String message);
}

@Component("smsNotification")
public class SmsNotificationService implements NotificationService {
    @Override
    public void send(String userId, String message) {
        // Send SMS via vendor API
        smsClient.send(getUserPhone(userId), message);
    }
}

public class LoggingNotificationDecorator implements NotificationService {
    private final NotificationService delegate;
    private final Logger log = LoggerFactory.getLogger(getClass());

    public LoggingNotificationDecorator(NotificationService delegate) {
        this.delegate = delegate;
    }

    @Override
    public void send(String userId, String message) {
        log.info("Sending notification to user={}, message={}", userId, message);
        long start = System.currentTimeMillis();
        delegate.send(userId, message);
        log.info("Notification sent in {}ms", System.currentTimeMillis() - start);
    }
}

public class RetryNotificationDecorator implements NotificationService {
    private final NotificationService delegate;
    private final int maxRetries;

    public RetryNotificationDecorator(NotificationService delegate, int maxRetries) {
        this.delegate = delegate;
        this.maxRetries = maxRetries;
    }

    @Override
    public void send(String userId, String message) {
        int attempt = 0;
        while (attempt < maxRetries) {
            try {
                delegate.send(userId, message);
                return;
            } catch (Exception e) {
                attempt++;
                if (attempt >= maxRetries) throw e;
            }
        }
    }
}

// Composing decorators
NotificationService service = new RetryNotificationDecorator(
    new LoggingNotificationDecorator(
        new SmsNotificationService()
    ), 3
);
```

**Lending domain use case:** Add retry, logging, metrics, rate-limiting, or encryption behaviors to partner API calls without modifying the API client code.

**Spring Framework usage:** `BufferedInputStream` wrapping `FileInputStream` (Java I/O). Spring's `HandlerInterceptor` and `Filter` chains act as decorators around request handling.

**When to use:** When you need to add/remove behaviors independently and combine them flexibly.  
**When NOT to use:** When the decorator chain becomes too deep and hard to debug. Consider AOP instead.

---

### 7. Proxy Pattern

**Problem it solves:** Provide a surrogate or placeholder that controls access to another object. Common uses: lazy loading, access control, logging, caching.

**UML (text):**
```
+------------------+
| DataService      |
| <<interface>>    |
+------------------+
| + getData()      |
+------------------+
      ^       ^
      |       |
+--------+ +------------------+
| Real   | | CachingProxy     |
| Service| +------------------+
+--------+ | - real: DataSvc  |
           | - cache: Map     |
           | + getData()      |
           +------------------+
```

**Java:**

```java
public interface CreditScoreService {
    CreditScore fetchScore(String panNumber);
}

@Component("realCreditScoreService")
public class ExternalCreditScoreService implements CreditScoreService {
    @Override
    public CreditScore fetchScore(String panNumber) {
        // Expensive external API call to CIBIL/Experian
        return externalApi.getCreditScore(panNumber);
    }
}

@Component
@Primary
public class CachingCreditScoreProxy implements CreditScoreService {
    private final CreditScoreService realService;
    private final Cache<String, CreditScore> cache;

    public CachingCreditScoreProxy(
            @Qualifier("realCreditScoreService") CreditScoreService realService) {
        this.realService = realService;
        this.cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofHours(24))
            .build();
    }

    @Override
    public CreditScore fetchScore(String panNumber) {
        return cache.get(panNumber, key -> realService.fetchScore(key));
    }
}
```

**Lending domain use case:** Credit bureau calls are expensive and rate-limited. A caching proxy avoids redundant calls within 24 hours.

**Spring Framework usage:**
- **Spring AOP** creates dynamic proxies for `@Transactional`, `@Cacheable`, `@Async`.
- **JPA Lazy Loading:** `@OneToMany(fetch = FetchType.LAZY)` returns a proxy collection that loads data on first access.
- **Feign clients** create proxies from interface definitions.

**When to use:** Caching, access control, lazy initialization, remote proxies.  
**When NOT to use:** When direct access is simpler and there's no cross-cutting concern.

---

### 8. Facade Pattern

**Problem it solves:** Provide a simplified interface to a complex subsystem. Hides the internal complexity behind a clean API.

**UML (text):**
```
+--------------------+
| LoanFacade         |
+--------------------+         +-----------+  +----------+  +--------+
| + applyForLoan()   |-------->| KYCService|  | CreditSvc|  | LoanSvc|
| + getStatus()      |         +-----------+  +----------+  +--------+
+--------------------+         | FraudSvc  |  | DocSvc   |
                               +-----------+  +----------+
```

**Java:**

```java
@Service
public class LoanApplicationFacade {
    private final KycService kycService;
    private final CreditScoreService creditService;
    private final FraudCheckService fraudService;
    private final DocumentService documentService;
    private final LoanUnderwritingService underwritingService;
    private final NotificationService notificationService;

    public LoanApplicationResult applyForLoan(LoanApplicationRequest request) {
        // Step 1: KYC verification
        KycResult kyc = kycService.verify(request.getPanNumber(), request.getAadhaarNumber());
        if (!kyc.isVerified()) {
            return LoanApplicationResult.rejected("KYC verification failed");
        }

        // Step 2: Credit score check
        CreditScore score = creditService.fetchScore(request.getPanNumber());
        if (score.getScore() < 650) {
            return LoanApplicationResult.rejected("Credit score below threshold");
        }

        // Step 3: Fraud check
        FraudResult fraud = fraudService.check(request.getCustomerId());
        if (fraud.isFlagged()) {
            return LoanApplicationResult.rejected("Fraud check failed");
        }

        // Step 4: Document verification
        documentService.verifyDocuments(request.getDocuments());

        // Step 5: Underwriting
        LoanOffer offer = underwritingService.evaluate(request, score);

        // Step 6: Notify customer
        notificationService.send(request.getCustomerId(),
            "Your loan is approved! Offer: " + offer.getAmount());

        return LoanApplicationResult.approved(offer);
    }
}

// Client code — simple, clean
@RestController
public class LoanController {
    private final LoanApplicationFacade facade;

    @PostMapping("/api/loans/apply")
    public ResponseEntity<LoanApplicationResult> apply(@RequestBody LoanApplicationRequest req) {
        return ResponseEntity.ok(facade.applyForLoan(req));
    }
}
```

**Lending domain use case:** Applying for a loan involves KYC, credit check, fraud detection, document verification, underwriting, and notification. The facade orchestrates this workflow — the controller just calls `applyForLoan()`.

**Spring Framework usage:** `JdbcTemplate` is a facade over raw JDBC (Connection, Statement, ResultSet). `RestTemplate` is a facade over HTTP client complexity.

**When to use:** When a subsystem has many interacting components and clients need a simple entry point.  
**When NOT to use:** When the facade becomes a God class with too many responsibilities — consider splitting into multiple facades.

---

## BEHAVIORAL PATTERNS

---

### 9. Strategy Pattern

**Problem it solves:** Define a family of algorithms, encapsulate each one, and make them interchangeable at runtime. Eliminates conditional statements for selecting behavior.

**UML (text):**
```
+-----------------------+       +-------------------+
| MandateExecutor       |------>| ExecutionStrategy  |
+-----------------------+       | <<interface>>      |
| - strategy: Strategy  |       +-------------------+
| + execute()           |       | + execute(mandate) |
+-----------------------+       +-------------------+
                                        ^
                                   +----+----+--------+
                                   |         |        |
                              +--------+ +------+ +----------+
                              | UPI    | | API  | | Physical |
                              | Strat  | | Strat| | Strat    |
                              +--------+ +------+ +----------+
```

**Java:**

```java
public interface NachExecutionStrategy {
    MandateResult execute(Mandate mandate);
    NachType getSupportedType();
}

@Component
public class UpiNachStrategy implements NachExecutionStrategy {
    @Override
    public MandateResult execute(Mandate mandate) {
        // UPI NACH: collect via UPI mandate
        UpiMandateRequest upiReq = buildUpiRequest(mandate);
        return upiGateway.executeMandateCollection(upiReq);
    }

    @Override
    public NachType getSupportedType() { return NachType.UPI_NACH; }
}

@Component
public class ApiNachStrategy implements NachExecutionStrategy {
    @Override
    public MandateResult execute(Mandate mandate) {
        // API NACH: direct API-based debit
        ApiDebitRequest apiReq = buildApiRequest(mandate);
        return nachApiClient.executeDebit(apiReq);
    }

    @Override
    public NachType getSupportedType() { return NachType.API_NACH; }
}

@Component
public class PhysicalNachStrategy implements NachExecutionStrategy {
    @Override
    public MandateResult execute(Mandate mandate) {
        // Physical NACH: generate form, scan, and submit
        byte[] form = generateNachForm(mandate);
        return physicalProcessingService.submit(form);
    }

    @Override
    public NachType getSupportedType() { return NachType.PHYSICAL_NACH; }
}

@Service
public class MandateExecutionService {
    private final Map<NachType, NachExecutionStrategy> strategies;

    public MandateExecutionService(List<NachExecutionStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(NachExecutionStrategy::getSupportedType, Function.identity()));
    }

    public MandateResult execute(Mandate mandate) {
        NachExecutionStrategy strategy = strategies.get(mandate.getNachType());
        if (strategy == null) {
            throw new UnsupportedNachTypeException(mandate.getNachType());
        }
        return strategy.execute(mandate);
    }
}
```

**Lending domain use case:** EMI collection can happen via UPI NACH, API NACH, or Physical NACH. Each has entirely different APIs and flows but the service layer picks the right strategy at runtime.

**Spring Framework usage:** Spring's `ResourceLoader` uses strategy to resolve resources (`classpath:`, `file:`, `http:`).

**When to use:** When you have multiple algorithms for the same task and the choice is made at runtime.  
**When NOT to use:** When there's only one algorithm and no realistic chance of adding another.

---

### 10. Observer Pattern

**Problem it solves:** Define a one-to-many dependency where when one object changes state, all dependents are notified automatically. Decouples event producers from consumers.

**UML (text):**
```
+--------------------+        +--------------------+
| EventPublisher     |------->| EventListener      |
+--------------------+  1..*  | <<interface>>      |
| + publish(event)   |        +--------------------+
+--------------------+        | + onEvent(event)   |
                              +--------------------+
                                      ^
                              +-------+-------+
                              |               |
                        +-----------+  +-----------+
                        | Notifier  |  | Analytics |
                        | Listener  |  | Listener  |
                        +-----------+  +-----------+
```

**Java (Spring Events):**

```java
// Event
public class LoanDisbursedEvent {
    private final String loanId;
    private final String customerId;
    private final BigDecimal amount;
    private final LocalDateTime disbursedAt;

    public LoanDisbursedEvent(String loanId, String customerId,
                               BigDecimal amount, LocalDateTime disbursedAt) {
        this.loanId = loanId;
        this.customerId = customerId;
        this.amount = amount;
        this.disbursedAt = disbursedAt;
    }

    // getters
}

// Publisher
@Service
public class DisbursementService {
    private final ApplicationEventPublisher eventPublisher;

    public void disburse(LoanApplication app) {
        // ... perform disbursement logic ...
        transferFunds(app);

        eventPublisher.publishEvent(new LoanDisbursedEvent(
            app.getLoanId(), app.getCustomerId(),
            app.getAmount(), LocalDateTime.now()
        ));
    }
}

// Listener 1: Send notification
@Component
public class DisbursementNotificationListener {
    @EventListener
    public void onLoanDisbursed(LoanDisbursedEvent event) {
        smsService.send(event.getCustomerId(),
            "Your loan of ₹" + event.getAmount() + " has been disbursed!");
    }
}

// Listener 2: Update analytics
@Component
public class DisbursementAnalyticsListener {
    @EventListener
    public void onLoanDisbursed(LoanDisbursedEvent event) {
        analyticsService.recordDisbursement(event.getLoanId(), event.getAmount());
    }
}

// Listener 3: Trigger EMI schedule creation (async)
@Component
public class EmiScheduleListener {
    @Async
    @EventListener
    public void onLoanDisbursed(LoanDisbursedEvent event) {
        emiService.createSchedule(event.getLoanId());
    }
}
```

**Lending domain use case:** When a loan is disbursed, multiple things happen: customer notification, analytics update, EMI schedule creation, partner webhook, audit log. Observer decouples all of these from the disbursement logic.

**Spring Framework usage:** `ApplicationEventPublisher` + `@EventListener` is the built-in observer. For distributed systems, Kafka consumers act as observers to Kafka events.

**When to use:** When multiple components need to react to state changes, and you want to decouple them.  
**When NOT to use:** When the order of listener execution matters critically — observers execute in undefined order (unless you use `@Order`).

---

### 11. Template Method Pattern

**Problem it solves:** Define the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the structure.

**UML (text):**
```
+-----------------------------+
| AbstractLoanProcessor       |
+-----------------------------+
| + processLoan() [final]     |
|   1. validateApplication()  |  <- abstract
|   2. checkCreditScore()     |  <- abstract
|   3. calculateOffer()       |  <- abstract
|   4. sendNotification()     |  <- concrete (common)
+-----------------------------+
            ^
       +----+----+
       |         |
+----------+ +----------+
| Personal | | Business |
| LoanProc | | LoanProc |
+----------+ +----------+
```

**Java:**

```java
public abstract class AbstractLoanProcessor {

    // Template method — defines the algorithm structure
    public final LoanResult processLoan(LoanApplication app) {
        validateApplication(app);
        CreditScore score = checkCreditScore(app);
        LoanOffer offer = calculateOffer(app, score);
        sendNotification(app, offer);
        return new LoanResult(app.getLoanId(), offer);
    }

    protected abstract void validateApplication(LoanApplication app);
    protected abstract CreditScore checkCreditScore(LoanApplication app);
    protected abstract LoanOffer calculateOffer(LoanApplication app, CreditScore score);

    // Common step — same for all loan types
    protected void sendNotification(LoanApplication app, LoanOffer offer) {
        notificationService.send(app.getCustomerId(),
            "Loan offer: ₹" + offer.getAmount() + " at " + offer.getRate() + "%");
    }
}

@Service
public class PersonalLoanProcessor extends AbstractLoanProcessor {

    @Override
    protected void validateApplication(LoanApplication app) {
        if (app.getAge() < 21 || app.getAge() > 60) {
            throw new ValidationException("Age must be between 21 and 60");
        }
        if (app.getMonthlyIncome().compareTo(new BigDecimal("25000")) < 0) {
            throw new ValidationException("Minimum income: ₹25,000");
        }
    }

    @Override
    protected CreditScore checkCreditScore(LoanApplication app) {
        return cibilService.fetchScore(app.getPanNumber());
    }

    @Override
    protected LoanOffer calculateOffer(LoanApplication app, CreditScore score) {
        double rate = score.getScore() > 750 ? 10.5 : 14.0;
        BigDecimal maxAmount = app.getMonthlyIncome().multiply(new BigDecimal("20"));
        return new LoanOffer(maxAmount, rate, 36);
    }
}

@Service
public class BusinessLoanProcessor extends AbstractLoanProcessor {

    @Override
    protected void validateApplication(LoanApplication app) {
        if (app.getBusinessVintage() < 3) {
            throw new ValidationException("Minimum 3 years of business vintage required");
        }
    }

    @Override
    protected CreditScore checkCreditScore(LoanApplication app) {
        CreditScore personal = cibilService.fetchScore(app.getPanNumber());
        CreditScore business = cibilService.fetchCommercialScore(app.getGstNumber());
        return CreditScore.combined(personal, business);
    }

    @Override
    protected LoanOffer calculateOffer(LoanApplication app, CreditScore score) {
        double rate = score.getScore() > 700 ? 12.0 : 16.0;
        BigDecimal maxAmount = app.getAnnualTurnover().multiply(new BigDecimal("0.25"));
        return new LoanOffer(maxAmount, rate, 48);
    }
}
```

**Lending domain use case:** Personal loans and business loans follow the same high-level flow (validate → credit check → offer → notify) but each step has different rules.

**Spring Framework usage:** `AbstractController`, `JdbcTemplate` (template for query execution), `JmsTemplate`, `RestTemplate`.

**When to use:** When subclasses share the same algorithm structure but differ in individual steps.  
**When NOT to use:** When the algorithm has too many steps that vary — leads to deep inheritance. Consider Strategy instead.

---

### 12. State Pattern

**Problem it solves:** Allow an object to change its behavior when its internal state changes. The object will appear to change its class. Avoids massive switch/if-else blocks for state transitions.

**UML (text):**
```
+--------------------+        +------------------+
| LoanApplication    |------->| LoanState        |
+--------------------+  curr  | <<interface>>    |
| - state: LoanState |        +------------------+
| + submit()         |        | + submit(ctx)    |
| + approve()        |        | + approve(ctx)   |
| + disburse()       |        | + reject(ctx)    |
| + reject()         |        | + disburse(ctx)  |
+--------------------+        +------------------+
                                      ^
                           +----------+----------+
                           |          |          |
                      +---------+ +--------+ +----------+
                      | Draft   | |Pending | |Approved  |
                      | State   | |State   | |State     |
                      +---------+ +--------+ +----------+
```

**Java:**

```java
public interface LoanState {
    default void submit(LoanContext ctx)   { throw new InvalidStateTransitionException("Cannot submit in " + ctx.getStateName()); }
    default void approve(LoanContext ctx)  { throw new InvalidStateTransitionException("Cannot approve in " + ctx.getStateName()); }
    default void reject(LoanContext ctx)   { throw new InvalidStateTransitionException("Cannot reject in " + ctx.getStateName()); }
    default void disburse(LoanContext ctx) { throw new InvalidStateTransitionException("Cannot disburse in " + ctx.getStateName()); }
}

public class DraftState implements LoanState {
    @Override
    public void submit(LoanContext ctx) {
        ctx.performValidation();
        ctx.setState(new PendingApprovalState());
        ctx.setStateName("PENDING_APPROVAL");
    }
}

public class PendingApprovalState implements LoanState {
    @Override
    public void approve(LoanContext ctx) {
        ctx.runCreditCheck();
        ctx.generateOffer();
        ctx.setState(new ApprovedState());
        ctx.setStateName("APPROVED");
    }

    @Override
    public void reject(LoanContext ctx) {
        ctx.recordRejectionReason();
        ctx.setState(new RejectedState());
        ctx.setStateName("REJECTED");
    }
}

public class ApprovedState implements LoanState {
    @Override
    public void disburse(LoanContext ctx) {
        ctx.transferFunds();
        ctx.createEmiSchedule();
        ctx.setState(new DisbursedState());
        ctx.setStateName("DISBURSED");
    }

    @Override
    public void reject(LoanContext ctx) {
        ctx.recordRejectionReason();
        ctx.setState(new RejectedState());
        ctx.setStateName("REJECTED");
    }
}

public class LoanContext {
    private LoanState state;
    private String stateName;

    public LoanContext() {
        this.state = new DraftState();
        this.stateName = "DRAFT";
    }

    public void submit()   { state.submit(this); }
    public void approve()  { state.approve(this); }
    public void reject()   { state.reject(this); }
    public void disburse() { state.disburse(this); }

    void setState(LoanState state) { this.state = state; }
    // ... other methods
}

// Usage
LoanContext loan = new LoanContext();
loan.submit();    // DRAFT -> PENDING_APPROVAL
loan.approve();   // PENDING_APPROVAL -> APPROVED
loan.disburse();  // APPROVED -> DISBURSED
loan.submit();    // throws InvalidStateTransitionException
```

**State transition diagram:**
```
DRAFT --submit--> PENDING_APPROVAL --approve--> APPROVED --disburse--> DISBURSED
                                   \                  \
                                    --reject--> REJECTED
```

**Lending domain use case:** A loan application moves through states: DRAFT → PENDING → APPROVED/REJECTED → DISBURSED. Each state allows different actions. State pattern makes invalid transitions impossible at compile time.

**Spring Framework usage:** Spring Statemachine project provides a framework for state machines with transitions, guards, and actions.

**When to use:** When an object's behavior changes based on its state and you have complex transition rules.  
**When NOT to use:** When there are only 2-3 states with simple transitions — a simple enum + if-else may suffice.

---

### 13. Command Pattern

**Problem it solves:** Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

**UML (text):**
```
+-------------------+     +------------------+
| CommandInvoker    |---->| Command          |
+-------------------+     | <<interface>>    |
| + execute(cmd)    |     +------------------+
| + enqueue(cmd)    |     | + execute()      |
+-------------------+     | + undo()         |
                          +------------------+
                                  ^
                          +-------+-------+
                          |               |
                   +------------+  +------------+
                   |Disburse Cmd|  |WaiveOff Cmd|
                   +------------+  +------------+
```

**Java:**

```java
public interface LoanCommand {
    CommandResult execute();
    void undo();
    String getDescription();
}

public class DisbursementCommand implements LoanCommand {
    private final String loanId;
    private final BigDecimal amount;
    private final DisbursementService disbursementService;
    private String transactionId;

    public DisbursementCommand(String loanId, BigDecimal amount,
                                DisbursementService disbursementService) {
        this.loanId = loanId;
        this.amount = amount;
        this.disbursementService = disbursementService;
    }

    @Override
    public CommandResult execute() {
        this.transactionId = disbursementService.transfer(loanId, amount);
        return new CommandResult(true, "Disbursed ₹" + amount + " for loan " + loanId);
    }

    @Override
    public void undo() {
        disbursementService.reverseTransfer(transactionId, amount);
    }

    @Override
    public String getDescription() {
        return "DISBURSE " + loanId + " ₹" + amount;
    }
}

public class WaiveOffCommand implements LoanCommand {
    private final String loanId;
    private final BigDecimal waiveAmount;
    private final WaiveOffService waiveOffService;

    // constructor...

    @Override
    public CommandResult execute() {
        waiveOffService.applyWaiver(loanId, waiveAmount);
        return new CommandResult(true, "Waived ₹" + waiveAmount + " for loan " + loanId);
    }

    @Override
    public void undo() {
        waiveOffService.reverseWaiver(loanId, waiveAmount);
    }

    @Override
    public String getDescription() {
        return "WAIVE_OFF " + loanId + " ₹" + waiveAmount;
    }
}

@Service
public class LoanCommandProcessor {
    private final List<LoanCommand> commandHistory = new ArrayList<>();

    @Transactional
    public CommandResult process(LoanCommand command) {
        CommandResult result = command.execute();
        if (result.isSuccess()) {
            commandHistory.add(command);
            auditLog.record(command.getDescription());
        }
        return result;
    }

    @Transactional
    public void undoLast() {
        if (!commandHistory.isEmpty()) {
            LoanCommand last = commandHistory.remove(commandHistory.size() - 1);
            last.undo();
            auditLog.record("UNDO: " + last.getDescription());
        }
    }
}

// Usage
LoanCommand cmd = new DisbursementCommand("LOAN-123", new BigDecimal("500000"), disbursementService);
commandProcessor.process(cmd);

// Later, if needed:
commandProcessor.undoLast();  // reverses the disbursement
```

**Lending domain use case:** Financial operations (disbursement, waive-off, fee reversal) need audit trails and sometimes rollback. Command pattern encapsulates each operation with its undo logic and provides a history.

**Spring Framework usage:** Spring Batch uses command-like `Tasklet` steps. `Runnable`/`Callable` in executor services are essentially commands.

**When to use:** When you need undo/redo, command queuing, audit logging, or macro operations (batch of commands).  
**When NOT to use:** When operations are simple and don't need undo or queuing — adds unnecessary abstraction.

---

## Quick Reference: Pattern Selection Guide

| Situation | Pattern |
|-----------|---------|
| Need exactly one instance | Singleton |
| Object creation depends on a type/config | Factory Method |
| Families of related objects | Abstract Factory |
| Complex object with many optional fields | Builder |
| Incompatible third-party interface | Adapter |
| Add behavior without modifying original | Decorator |
| Lazy loading, caching, access control | Proxy |
| Simplify complex subsystem | Facade |
| Multiple algorithms, choose at runtime | Strategy |
| Notify multiple listeners of state changes | Observer |
| Same algorithm structure, different steps | Template Method |
| Object behavior changes with state | State |
| Encapsulate operations for undo/queue/log | Command |

---

## Frequently Asked Interview Meta-Questions

**Q: What's the difference between Strategy and State?**
> Strategy: The client chooses the algorithm. The strategy object doesn't change itself.
> State: The object transitions between states internally. Each state determines the next state.

**Q: What's the difference between Factory Method and Abstract Factory?**
> Factory Method: Creates one product. Uses inheritance (subclass decides).
> Abstract Factory: Creates families of related products. Uses composition (factory object).

**Q: What's the difference between Adapter and Facade?**
> Adapter: Makes one interface compatible with another (1-to-1).
> Facade: Simplifies a complex subsystem by providing a unified interface (1-to-many).

**Q: What's the difference between Decorator and Proxy?**
> Decorator: Adds new behavior (stacks functionality).
> Proxy: Controls access to the original (caching, lazy loading, security).
> Structurally identical — the intent differs.

**Q: What's the difference between Template Method and Strategy?**
> Template Method: Uses inheritance. The base class defines the skeleton, subclasses override steps.
> Strategy: Uses composition. The algorithm is injected from outside.
> Prefer Strategy when you want to swap algorithms at runtime without inheritance.
