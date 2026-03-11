# SOLID Principles — Lending Platform Examples

---

## S — Single Responsibility Principle

> **A class should have only one reason to change.**

Every class should own exactly one piece of functionality. If a class has two responsibilities, changes to one can break the other.

### BAD — Violating SRP

```java
public class MandateService {

    public Mandate createMandate(LoanApplication app) {
        Mandate mandate = new Mandate();
        mandate.setLoanId(app.getLoanId());
        mandate.setAmount(app.getEmiAmount());
        mandate.setStatus(MandateStatus.CREATED);
        mandateRepository.save(mandate);

        // PROBLEM: Also handles webhook delivery — separate concern
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        String payload = objectMapper.writeValueAsString(mandate);
        restTemplate.postForEntity(app.getWebhookUrl(), new HttpEntity<>(payload, headers), String.class);

        // PROBLEM: Also handles audit logging — another separate concern
        AuditLog log = new AuditLog("MANDATE_CREATED", mandate.getId(), LocalDateTime.now());
        auditRepository.save(log);

        return mandate;
    }
}
```

Three reasons to change: mandate logic, webhook delivery mechanism, audit logging format.

### GOOD — Following SRP

```java
public class MandateService {
    private final MandateRepository mandateRepository;
    private final WebhookDispatcher webhookDispatcher;
    private final AuditService auditService;

    public Mandate createMandate(LoanApplication app) {
        Mandate mandate = new Mandate();
        mandate.setLoanId(app.getLoanId());
        mandate.setAmount(app.getEmiAmount());
        mandate.setStatus(MandateStatus.CREATED);
        mandateRepository.save(mandate);

        webhookDispatcher.dispatch(app.getWebhookUrl(), mandate);
        auditService.log("MANDATE_CREATED", mandate.getId());

        return mandate;
    }
}

public class WebhookDispatcher {
    public void dispatch(String url, Object payload) {
        // All webhook delivery logic isolated here
    }
}

public class AuditService {
    public void log(String event, Long entityId) {
        // All audit logging logic isolated here
    }
}
```

Each class has one reason to change. Webhook delivery can switch from REST to Kafka without touching MandateService.

### Real-world mapping

In a lending platform, the mandate creation service should not be responsible for notifying the partner bank, sending webhooks to the borrower app, or logging the event. Each of these is a separate concern that changes independently.

### When this principle matters most

- When a class grows beyond ~200 lines — a sign it has multiple responsibilities.
- When a bug fix in one area causes regression in another area of the same class.
- When multiple developers need to modify the same file for unrelated features.

### Common interview questions

1. "How would you refactor a God class?" → Identify responsibilities, extract into separate classes.
2. "Does SRP mean one method per class?" → No. It means one *reason to change*. A class can have many methods that serve the same responsibility.
3. "SRP vs separation of concerns?" → SRP operates at the class level; SoC is a broader architectural principle.

---

## O — Open/Closed Principle

> **Classes should be open for extension but closed for modification.**

You should be able to add new behavior without changing existing, tested code.

### BAD — Violating OCP

```java
public class LoanProcessor {

    public void processLoan(LoanApplication app) {
        if (app.getPartner().equals("ICICI_LOMBARD")) {
            // ICICI-specific processing
            iciciApi.submitApplication(app.getIciciPayload());
        } else if (app.getPartner().equals("ACKO")) {
            // Acko-specific processing
            ackoApi.submit(app.getAckoPayload());
        } else if (app.getPartner().equals("HDFC")) {
            // EVERY new partner requires modifying this class
            hdfcApi.process(app.getHdfcPayload());
        }
    }
}
```

Adding a new lending partner means modifying the `LoanProcessor` class every time — risk of breaking existing partners.

### GOOD — Following OCP

```java
public interface LendingPartner {
    String getPartnerCode();
    void processLoan(LoanApplication app);
}

@Component
public class IciciLombardPartner implements LendingPartner {
    public String getPartnerCode() { return "ICICI_LOMBARD"; }
    public void processLoan(LoanApplication app) {
        iciciApi.submitApplication(app.getIciciPayload());
    }
}

@Component
public class AckoPartner implements LendingPartner {
    public String getPartnerCode() { return "ACKO"; }
    public void processLoan(LoanApplication app) {
        ackoApi.submit(app.getAckoPayload());
    }
}

@Component
public class LendingPartnerFactory {
    private final Map<String, LendingPartner> partners;

    public LendingPartnerFactory(List<LendingPartner> partnerList) {
        this.partners = partnerList.stream()
            .collect(Collectors.toMap(LendingPartner::getPartnerCode, Function.identity()));
    }

    public LendingPartner getPartner(String code) {
        LendingPartner partner = partners.get(code);
        if (partner == null) throw new UnsupportedPartnerException(code);
        return partner;
    }
}

@Service
public class LoanProcessor {
    private final LendingPartnerFactory factory;

    public void processLoan(LoanApplication app) {
        LendingPartner partner = factory.getPartner(app.getPartnerCode());
        partner.processLoan(app);
    }
}
```

Adding a new partner = adding a new class implementing `LendingPartner`. Zero changes to `LoanProcessor` or the factory (Spring auto-discovers the new bean).

### Real-world mapping

In a lending platform integrating with ICICI Lombard, Acko, HDFC, etc., each partner has different API contracts, payloads, and authentication schemes. OCP ensures adding a new partner is a self-contained change.

### When this principle matters most

- When you frequently add new variants (partners, payment methods, product types).
- When multiple teams work on the same codebase — OCP minimizes merge conflicts.
- When you need to A/B test new behavior without risking stable code.

### Common interview questions

1. "How do you add new functionality without modifying existing code?" → Interface + new implementation + factory/registry.
2. "Isn't this over-engineering for 2 partners?" → Possibly. Apply when you *know* more variants are coming. Premature abstraction is also a cost.
3. "How does Spring Boot support OCP?" → Auto-wired lists of interface implementations, `@ConditionalOnProperty`, profiles.

---

## L — Liskov Substitution Principle

> **Subtypes must be substitutable for their base types without altering correctness.**

If code works with a base class/interface, it should work with any subclass without surprises.

### BAD — Violating LSP

```java
public interface PaymentMethod {
    void initiatePayment(BigDecimal amount);
    void refund(BigDecimal amount);
}

public class UpiPayment implements PaymentMethod {
    public void initiatePayment(BigDecimal amount) { /* UPI collect */ }
    public void refund(BigDecimal amount) { /* UPI refund */ }
}

public class NachPayment implements PaymentMethod {
    public void initiatePayment(BigDecimal amount) { /* NACH mandate execution */ }
    public void refund(BigDecimal amount) {
        // NACH doesn't support refunds!
        throw new UnsupportedOperationException("NACH does not support refunds");
    }
}
```

Code that calls `paymentMethod.refund(amount)` will blow up if it receives a `NachPayment`. The subtype is NOT substitutable.

### GOOD — Following LSP

```java
public interface PaymentMethod {
    void initiatePayment(BigDecimal amount);
}

public interface RefundablePayment extends PaymentMethod {
    void refund(BigDecimal amount);
}

public class UpiPayment implements RefundablePayment {
    public void initiatePayment(BigDecimal amount) { /* UPI collect */ }
    public void refund(BigDecimal amount) { /* UPI refund */ }
}

public class NachPayment implements PaymentMethod {
    public void initiatePayment(BigDecimal amount) { /* NACH mandate execution */ }
}

// Client code
public class PaymentService {
    public void processRefund(RefundablePayment payment, BigDecimal amount) {
        payment.refund(amount);  // guaranteed to work — type-safe
    }
}
```

Now `NachPayment` doesn't pretend to support refunds. The type system enforces correctness.

### Real-world mapping

In a lending platform, UPI, NACH, and API-based payments share some behaviors (initiate payment) but differ in others (refund, recurring). LSP forces you to model these differences in the type hierarchy rather than throwing runtime exceptions.

### When this principle matters most

- When you have a class hierarchy with varying capabilities.
- When client code uses the base type and expects uniform behavior.
- When `instanceof` checks or `UnsupportedOperationException` appear in your code — LSP violation smell.

### Common interview questions

1. "Is the Rectangle/Square problem a LSP violation?" → Yes. A Square that overrides setWidth to also change height breaks Rectangle's contract.
2. "How do you fix LSP violations?" → Redesign the hierarchy. Split interfaces. Prefer composition.
3. "How does LSP relate to polymorphism?" → LSP is what makes polymorphism *safe*. Without it, polymorphic code has hidden traps.

---

## I — Interface Segregation Principle

> **Clients should not be forced to depend on interfaces they don't use.**

Many small, focused interfaces are better than one large, do-everything interface.

### BAD — Violating ISP

```java
public interface LoanPartnerGateway {
    void disburseLoan(LoanApplication app);
    void collectEmi(String loanId, BigDecimal amount);
    void generateInsuranceQuote(String loanId);
    void reportToRegulator(String loanId);
    void sendSmsToCustomer(String phone, String message);
}
```

A partner gateway that only handles disbursement is forced to implement all five methods. Most will throw `UnsupportedOperationException`.

### GOOD — Following ISP

```java
public interface DisbursementGateway {
    void disburseLoan(LoanApplication app);
}

public interface CollectionGateway {
    void collectEmi(String loanId, BigDecimal amount);
}

public interface InsuranceGateway {
    void generateInsuranceQuote(String loanId);
}

public interface RegulatoryReporter {
    void reportToRegulator(String loanId);
}

// A partner can implement only what it supports
public class IciciPartnerGateway implements DisbursementGateway, CollectionGateway {
    public void disburseLoan(LoanApplication app) { /* ... */ }
    public void collectEmi(String loanId, BigDecimal amount) { /* ... */ }
}

public class AckoInsuranceGateway implements InsuranceGateway {
    public void generateInsuranceQuote(String loanId) { /* ... */ }
}
```

Each class only depends on the interfaces it actually needs.

### Real-world mapping

In a lending platform, different vendors handle different capabilities. An insurance partner shouldn't need to implement `collectEmi`. A payment partner shouldn't implement `generateInsuranceQuote`. ISP keeps integration classes clean and focused.

### When this principle matters most

- When you have "fat" interfaces that not all implementors can meaningfully support.
- When changes to one part of a large interface force recompilation/redeployment of unrelated modules.
- When you see many empty or exception-throwing method implementations.

### Common interview questions

1. "How is ISP different from SRP?" → SRP is about classes (one reason to change). ISP is about interfaces (clients shouldn't be forced to depend on methods they don't use). Related but distinct.
2. "Can a class implement multiple interfaces?" → Yes, and that's the point. A class picks the interfaces matching its actual capabilities.
3. "Real Java example of ISP?" → Java's `Iterable`, `Closeable`, `Serializable` — small, single-purpose interfaces that classes mix and match.

---

## D — Dependency Inversion Principle

> **High-level modules should not depend on low-level modules. Both should depend on abstractions.**

Code should depend on interfaces/abstract classes, not on concrete implementations.

### BAD — Violating DIP

```java
public class EmiCollectionService {
    private final NachApiClient nachApiClient = new NachApiClient();  // concrete dependency

    public void collectEmi(String mandateId, BigDecimal amount) {
        nachApiClient.executeMandate(mandateId, amount);
    }
}
```

`EmiCollectionService` is tightly coupled to `NachApiClient`. To switch to UPI autopay or add a mock for testing, you must modify this class.

### GOOD — Following DIP

```java
public interface PaymentGateway {
    PaymentResult executePayment(String referenceId, BigDecimal amount);
}

@Component
public class NachPaymentGateway implements PaymentGateway {
    public PaymentResult executePayment(String referenceId, BigDecimal amount) {
        // NACH API call
    }
}

@Component
public class UpiAutopayGateway implements PaymentGateway {
    public PaymentResult executePayment(String referenceId, BigDecimal amount) {
        // UPI Autopay API call
    }
}

@Service
public class EmiCollectionService {
    private final PaymentGateway paymentGateway;  // depends on abstraction

    public EmiCollectionService(@Qualifier("nachPaymentGateway") PaymentGateway gateway) {
        this.paymentGateway = gateway;  // injected, not created
    }

    public void collectEmi(String referenceId, BigDecimal amount) {
        paymentGateway.executePayment(referenceId, amount);
    }
}
```

The high-level service (`EmiCollectionService`) depends on an abstraction (`PaymentGateway`). The concrete implementation is injected by Spring. Switching from NACH to UPI is a configuration change.

### Real-world mapping

In a lending platform, EMI collection can happen via NACH, UPI autopay, or API-based debit. The collection service shouldn't know or care which mechanism is used — it depends on the `PaymentGateway` abstraction. This also makes unit testing trivial: inject a mock `PaymentGateway`.

### When this principle matters most

- When you need to swap implementations (NACH → UPI, vendor A → vendor B).
- When you want testable code — mocking an interface is far easier than mocking a concrete class.
- When multiple modules share a dependency — the abstraction provides a stable contract.

### Common interview questions

1. "How does Spring implement DIP?" → Spring's IoC container creates and injects dependencies. You declare interfaces, Spring wires implementations via `@Autowired`, `@Qualifier`, or profiles.
2. "DIP vs Dependency Injection?" → DIP is the *principle* (depend on abstractions). DI is the *technique* (inject dependencies from outside). DI is one way to implement DIP.
3. "What's the cost of DIP?" → More interfaces and classes. Over-abstraction for simple scenarios adds complexity without benefit. Apply when there's a genuine need for flexibility.

---

## SOLID Summary Table

| Principle | One-liner | Smell when violated |
|-----------|-----------|-------------------|
| **S**ingle Responsibility | One class, one reason to change | God classes, 500+ line files |
| **O**pen/Closed | Extend behavior, don't modify | if-else chains for variants |
| **L**iskov Substitution | Subtypes are safely swappable | `UnsupportedOperationException`, `instanceof` checks |
| **I**nterface Segregation | Small, focused interfaces | Fat interfaces, empty method implementations |
| **D**ependency Inversion | Depend on abstractions | `new ConcreteClass()` inside business logic |

---

## Common Holistic Interview Questions

**Q: Which SOLID principle do you think is most important?**
> All are interconnected, but SRP is foundational. If a class has a single responsibility, it's naturally easier to extend (OCP), substitute (LSP), and depend on via narrow interfaces (ISP/DIP).

**Q: Can you have too much SOLID?**
> Yes. Over-engineering violates YAGNI (You Aren't Gonna Need It). A `UserService` that only talks to one database doesn't need a `DatabaseGateway` interface with a factory. Apply SOLID when you see the need for flexibility — not preemptively.

**Q: How do SOLID principles map to microservices?**
> - SRP → each microservice owns one business capability.
> - OCP → new features as new services, not modifications to existing ones.
> - LSP → service contracts (APIs) must be backward compatible.
> - ISP → API endpoints should be client-specific (BFF pattern).
> - DIP → services communicate via contracts (REST/gRPC specs), not internal implementations.
