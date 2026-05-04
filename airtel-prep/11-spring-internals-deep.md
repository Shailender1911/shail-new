# File 11: Spring + Spring Boot Internals — Deep

> When R1 goes beyond "what does `@SpringBootApplication` do."
> All anchored to your shipped patterns.

---

## SECTION 1 — BEAN LIFECYCLE (full sequence)

### The 9 phases (memorize)
```
1. Instantiation              (Constructor called)
2. Property population        (DI — setters / fields / autowire)
3. BeanNameAware.setBeanName
4. BeanFactoryAware.setBeanFactory
5. ApplicationContextAware.setApplicationContext
6. BeanPostProcessor.postProcessBeforeInitialization   (e.g. @PostConstruct invocation lives here)
7. InitializingBean.afterPropertiesSet  /  @PostConstruct  /  init-method
8. BeanPostProcessor.postProcessAfterInitialization    (this is where AOP proxies wrap the bean)
9. (use)
10. DisposableBean.destroy   /  @PreDestroy  /  destroy-method
```

### Talking points
- **`BeanPostProcessor.postProcessAfterInitialization` is where Spring AOP proxies wrap your bean** — that's why self-invocation bypasses the proxy: `this.method()` is on the raw bean, not the proxy.
- **`@PostConstruct` runs after DI but before AOP wrapping** — don't rely on AOP-cross-cutting behaviour in `@PostConstruct`.
- **Constructor injection happens before any `Aware` callback** — so don't expect ApplicationContext in your constructor.

---

## SECTION 2 — APPLICATIONCONTEXT vs BEANFACTORY

| Aspect | BeanFactory | ApplicationContext |
|---|---|---|
| Bean creation | Lazy (on `getBean`) | Eager (on startup, except `@Lazy`) |
| Auto-wiring | Manual | Automatic |
| AOP | Manual config | Automatic |
| Event publishing | No | `ApplicationEventPublisher` |
| Internationalization | No | `MessageSource` |
| Common impl | (rarely used directly) | `AnnotationConfigApplicationContext`, `WebApplicationContext` |

**ApplicationContext is BeanFactory + extra.** Always use ApplicationContext in real code.

---

## SECTION 3 — `@SpringBootApplication` UNPACKED

```java
@Target(TYPE) @Retention(RUNTIME)
@Documented @Inherited
@SpringBootConfiguration   // = @Configuration with metadata for Boot
@EnableAutoConfiguration   // imports auto-config classes via spring.factories / .imports
@ComponentScan(...)        // scans current package + subpackages
public @interface SpringBootApplication { ... }
```

### Auto-configuration internals (the deep cut)
1. `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`.
2. Selector reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (post 2.7) or `spring.factories` (legacy).
3. For each candidate class, evaluates `@ConditionalOn*`:
   - `@ConditionalOnClass(X)` — only if X is on classpath.
   - `@ConditionalOnMissingBean(Y)` — only if user hasn't defined Y.
   - `@ConditionalOnProperty(prefix=..., name=..., havingValue=...)` — gated by config.
   - `@ConditionalOnWebApplication` — only in web context.
4. Surviving configs are registered.

### Concrete example (your code)
```java
@Configuration
@ConditionalOnProperty(prefix="pariksha", name="enabled", havingValue="true")
public class ParikshaConfig {
  @Bean
  public Unleash unleash(...) { ... }
}
```
- Pariksha is **opt-in per environment**. Non-prod can disable it without code change.

### Debug auto-config
- `--debug` flag → prints positive/negative matches at startup.
- `actuator/conditions` endpoint → live snapshot.

---

## SECTION 4 — `@Configuration` vs `@Component`

### `@Configuration`
- Class is a "factory" of beans (via `@Bean` methods).
- Spring CGLIB-proxies the class so `@Bean` method calls within the class return the **same singleton instance** (call `bean()` twice → same instance).

### `@Component` (and `@Service`/`@Repository`/`@Controller`)
- Class itself becomes a bean.
- No CGLIB proxy by default.

### `proxyBeanMethods = false` optimization
```java
@Configuration(proxyBeanMethods = false)   // since Spring 5.2
public class FastConfig {
  @Bean
  public Foo foo() { return new Foo(); }
}
```
- Skips CGLIB proxy → faster startup.
- Trade-off: each `foo()` call inside the class returns a new instance (you must `@Autowired` if you need the singleton).

---

## SECTION 5 — `@Transactional` INTERNALS

### How it works
- Spring proxies the bean (JDK proxy if interfaces present; CGLIB otherwise).
- Proxy wraps method invocation in `TransactionInterceptor`.
- Interceptor consults `PlatformTransactionManager` to begin/commit/rollback.

### The 4 gotchas
1. **Self-invocation bypasses proxy.** `this.method()` calls the raw bean. Fix: split into separate beans, OR inject self-reference, OR `AopContext.currentProxy()`.
2. **Only public methods.** Private/package-private methods aren't advised by JDK or default CGLIB.
3. **Checked exceptions don't roll back by default.** Need `@Transactional(rollbackFor = Exception.class)`.
4. **Final classes / methods** can't be CGLIB-proxied. Spring 5.2+ relaxes this for some cases but stay safe — don't make `@Transactional` classes/methods `final`.

### Propagation in your shop
- `REQUIRED` (default) — join existing or create.
- `REQUIRES_NEW` — used in `MandateCronServiceImpl` retry log so it persists even if outer rolls back.
- `MANDATORY` — must already be in tx; throws if not.
- `NEVER` — must NOT be in tx; throws if so. Useful for explicit "don't accidentally enroll this in a tx."

### Isolation levels
| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ_UNCOMMITTED | ✓ allowed | ✓ allowed | ✓ allowed |
| READ_COMMITTED   | ✗ | ✓ | ✓ |
| REPEATABLE_READ  | ✗ | ✗ | ✓ (mostly) |
| SERIALIZABLE     | ✗ | ✗ | ✗ |

- MySQL default: `REPEATABLE_READ`. Postgres default: `READ_COMMITTED`. Be aware which DB you're on.
- We use READ_COMMITTED for almost everything; explicit higher level only when needed.

---

## SECTION 6 — FILTER vs INTERCEPTOR vs HandlerMethodArgumentResolver vs ARGUMENTRESOLVER

### Order in a request lifecycle
```
HTTP Request
  ↓
Tomcat / Netty
  ↓
Servlet Filter chain                    ← Filters here (servlet level, before Spring)
  ↓
DispatcherServlet
  ↓
HandlerInterceptor.preHandle             ← Interceptors (Spring MVC level)
  ↓
HandlerMethodArgumentResolver           ← Resolves method args (e.g. @RequestBody)
  ↓
Controller method
  ↓
HandlerInterceptor.postHandle
  ↓
HandlerInterceptor.afterCompletion
  ↓
Servlet Filter chain (response side)
```

### When to use which
- **Filter** — cross-cutting at servlet level (CORS, request logging, security at servlet boundary). Runs before Spring sees the request.
- **HandlerInterceptor** — Spring MVC-level cross-cutting (auth, metrics, MDC setup). Has access to handler method.
- **ControllerAdvice / ExceptionHandler** — exception → response mapping.
- **HandlerMethodArgumentResolver** — custom parameter injection (e.g. `@CurrentUser` → user from JWT).
- **AOP `@Around`** — method-level cross-cutting on any bean (your `DataSourceAspect`).

### Real example (your code)
- **Filter:** JWT authentication filter on `/v1/gpay/*` paths — populates `GooglePayContextHolder`, clears in `finally`.
- **Interceptor:** request id MDC setup so all logs in a request share a correlation id.
- **AOP:** `DataSourceAspect` for read-write split.
- **ControllerAdvice:** `OrchestrationExceptionHandler` mapping exceptions to API v4 envelope.

---

## SECTION 7 — EMBEDDED TOMCAT vs UNDERTOW vs JETTY vs NETTY

| Server | Default? | Stack | When to swap |
|---|---|---|---|
| **Tomcat** | Yes | Servlet, blocking IO with NIO connector | Default, fine for most. |
| **Undertow** | No | Non-blocking, async-friendly | Higher concurrency / lower memory |
| **Jetty** | No | Servlet | Niche; some long-poll patterns |
| **Netty** | Yes for WebFlux | Async event-loop | Reactive Spring (WebFlux) |

### Tomcat tuning (real flags)
```
server.tomcat.threads.max=200            # max worker threads
server.tomcat.threads.min-spare=10       # always-warm threads
server.tomcat.accept-count=100           # OS-level accept queue size
server.tomcat.max-connections=10000      # max concurrent connections
server.tomcat.connection-timeout=20s
```

---

## SECTION 8 — CIRCULAR DEPENDENCY (interview classic)

### Spring's resolution
- Spring uses a 3-stage cache:
  1. `singletonObjects` — fully initialized.
  2. `earlySingletonObjects` — populated but pre-init.
  3. `singletonFactories` — factory references.
- When A depends on B which depends on A, Spring exposes A's early reference (stage 2) so B can complete; then A completes.
- **Works for setter / field injection** because Spring can construct A first, then inject.
- **Does NOT work for constructor injection** — both need to be constructed before injection. Spring throws `BeanCurrentlyInCreationException`.

### What we do
- **Refactor.** A circular dep is a code smell — extract a third bean both depend on.
- `@Lazy` on one side is a workaround that hides the smell. Don't use as default.

### Concrete real-world cause
- A controller injecting a service that injects a repository which injects… and somewhere in the chain, a service injects the controller for an event-publishing path. Fix: extract an event publisher bean.

---

## SECTION 9 — APPLICATION EVENTS

### Pattern
```java
// Publisher
@Autowired ApplicationEventPublisher publisher;
publisher.publishEvent(new LoanApprovedEvent(loanId));

// Subscriber
@Component
class LoanApprovedListener {
  @EventListener
  public void onApproved(LoanApprovedEvent event) { ... }

  @EventListener
  @Async                      // off the publisher's thread
  public void onApprovedAsync(LoanApprovedEvent event) { ... }

  @TransactionalEventListener(phase = AFTER_COMMIT)   // only after tx commits
  public void onApprovedAfterCommit(LoanApprovedEvent event) { ... }
}
```

### `@TransactionalEventListener` (the killer feature)
- Default phase: `AFTER_COMMIT`. Listener runs only if the tx successfully commits.
- Solves: "publish event after DB write succeeded" without dual-write.
- Trade-off: in-process only. For cross-service, use outbox pattern.

---

## SECTION 10 — ACTUATOR (production hygiene)

### Endpoints we expose
- `/actuator/health` — liveness; cheap, doesn't hit downstream.
- `/actuator/health/readiness` — readiness; checks DB, Bedrock for Eklavya.
- `/actuator/info` — build info, git commit (via `git-commit-id-plugin`).
- `/actuator/prometheus` — Micrometer metrics for Prometheus scrape.
- `/actuator/loggers` — runtime log-level changes (admin-only).
- `/actuator/heapdump` — emergency heap snapshot (admin-only).
- `/actuator/threaddump` — thread snapshot (admin-only).

### K8s probe rule
- **Liveness** — never check downstreams. Else one downstream outage cascades into pod restart storms.
- **Readiness** — can check critical downstreams. Outage → traffic stops to this pod, but pod isn't restarted.

### Custom health indicator
```java
@Component
public class BedrockHealthIndicator implements HealthIndicator {
  @Override
  public Health health() {
    try {
      bedrockClient.ping();
      return Health.up().build();
    } catch (Exception e) {
      return Health.down(e).build();
    }
  }
}
```

---

## SECTION 11 — REAL Q&A FOR THE ROUND

### Q: "What's the difference between Spring Boot and Spring?"
> "Spring Boot is opinionated configuration on top of Spring. Auto-configuration eliminates most of the XML and `@Configuration` classes you'd hand-write. Embedded server (Tomcat by default) ships in the JAR — no external app server. Starter dependencies bundle common combinations. Production-ready features (Actuator, Micrometer, externalized config) are first-class. Underneath it's still the Spring core — IoC, AOP, transaction management. Boot is the productivity layer."

### Q: "Walk me through how you'd add a new endpoint."
> "Define DTOs at the boundary (request/response), don't expose JPA entities. Validate via `@Valid` on the request DTO. Constructor-inject the service via `@RequiredArgsConstructor`. `@RestController` returns DTO/`ResponseEntity`. Service layer is `@Transactional` where needed; repository extends `JpaRepository`. Exception path goes through `@ControllerAdvice`. Tests: WebMvcTest for the controller, `@DataJpaTest` for the repository, integration test for happy + 1 edge case."

### Q: "Why constructor injection?"
> "Immutable fields with `final`, fail-fast at startup if a dep is missing, easy to test (no reflection — pass mocks via constructor), and forces you to acknowledge if a class has too many dependencies (constructor gets unwieldy → time to refactor). Lombok `@RequiredArgsConstructor` makes it boilerplate-free. We've moved every new class to constructor injection; field `@Autowired` is allowed in legacy only."

### Q: "How do you handle exceptions globally?"
> "`@ControllerAdvice` with `@ExceptionHandler` methods mapping each exception type to a response. Concrete: `OrchestrationExceptionHandler` maps `BusinessException` / `ValidationException` / `IntegrationException` to API v4 envelope. For GPay, a separate handler returns `GpayErrorResponse` matching Google's contract — different envelope from internal API."

### Q: "What's the order of `@ControllerAdvice` resolution?"
> "Most specific exception type wins. If two advices catch the same exception, `@Order` decides — lower order runs first. We explicitly `@Order` our GPay-specific advice ahead of the generic one."
