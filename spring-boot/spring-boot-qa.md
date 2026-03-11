# Spring Boot Interview Questions & Answers

> Comprehensive guide covering Spring Core, Spring Boot, Spring Security, and Advanced topics.
> Real-world examples drawn from fintech/lending platforms.

---

## Table of Contents

1. [Spring Core](#1-spring-core)
2. [Spring Boot Specifics](#2-spring-boot-specifics)
3. [Spring Security](#3-spring-security)
4. [Advanced Spring Boot](#4-advanced-spring-boot)

---

## 1. Spring Core

---

### Q1. What is IoC Container and how does Dependency Injection work internally? [Medium]

**IoC (Inversion of Control)** means the framework controls object creation and wiring, not your code.

**How DI works internally (simplified flow):**

1. **Classpath scanning** — Spring scans packages for `@Component`, `@Service`, `@Repository`, `@Controller` annotated classes.
2. **BeanDefinition creation** — For each discovered class, a `BeanDefinition` object is created holding metadata (class name, scope, constructor args, dependencies).
3. **BeanDefinitionRegistry** — All definitions are registered in the registry.
4. **BeanFactory instantiation** — When `getBean()` is called (eagerly for singletons at startup), the factory:
   - Reads the `BeanDefinition`
   - Resolves dependencies recursively
   - Creates the instance via constructor/reflection
   - Injects dependencies (constructor/setter/field)
   - Applies `BeanPostProcessor`s (proxies for AOP, `@Transactional`, etc.)
5. **Singleton cache** — Singleton beans are cached in a `ConcurrentHashMap` (`singletonObjects`) for reuse.

```
ApplicationContext startup
  └─> ClassPathBeanDefinitionScanner
        └─> Registers BeanDefinitions
              └─> DefaultListableBeanFactory
                    └─> createBean()
                          ├─> resolveConstructorArgs() ← recursive DI
                          ├─> instantiate via Constructor
                          ├─> populateBean() ← setter/field injection
                          └─> initializeBean() ← lifecycle callbacks
```

**Interview Follow-up:** "What's the difference between `BeanFactory` and `ApplicationContext`?" — See Q8.

---

### Q2. Constructor vs Setter vs Field Injection — why is constructor preferred? [Medium]

| Aspect | Constructor | Setter | Field (`@Autowired`) |
|---|---|---|---|
| Immutability | ✅ Fields can be `final` | ❌ Mutable | ❌ Mutable |
| Required deps | ✅ Enforced at compile time | ❌ Can be null at runtime | ❌ Can be null |
| Testability | ✅ Easy — pass via constructor | ⚠️ Need setter calls | ❌ Need reflection |
| Circular dep detection | ✅ Fails fast at startup | ❌ Hides it | ❌ Hides it |
| Spring coupling | ✅ No annotation needed | ⚠️ `@Autowired` | ❌ `@Autowired` required |

```java
// ✅ PREFERRED: Constructor Injection
@Service
public class LoanService {

    private final CreditScoreClient creditScoreClient;
    private final LoanRepository loanRepository;
    private final NotificationService notificationService;

    // Since Spring 4.3, @Autowired is optional when there's a single constructor
    public LoanService(CreditScoreClient creditScoreClient,
                       LoanRepository loanRepository,
                       NotificationService notificationService) {
        this.creditScoreClient = creditScoreClient;
        this.loanRepository = loanRepository;
        this.notificationService = notificationService;
    }
}

// ⚠️ Setter Injection — use for optional dependencies
@Service
public class ReportService {

    private MetricsCollector metricsCollector;

    @Autowired(required = false)
    public void setMetricsCollector(MetricsCollector metricsCollector) {
        this.metricsCollector = metricsCollector;
    }
}

// ❌ AVOID: Field Injection
@Service
public class PaymentService {
    @Autowired
    private PaymentGateway paymentGateway; // untestable without Spring context
}
```

> **Interview Tip:** If someone asks "Why not field injection?" — say: "It hides dependencies, makes the class untestable without reflection, and couples the code to Spring's DI container."

---

### Q3. Explain the complete Bean Lifecycle in Spring. [Hard]

```
1.  Instantiation (constructor called)
2.  Populate Properties (DI — setter/field injection)
3.  BeanNameAware.setBeanName()
4.  BeanFactoryAware.setBeanFactory()
5.  ApplicationContextAware.setApplicationContext()
6.  BeanPostProcessor.postProcessBeforeInitialization()
7.  @PostConstruct method
8.  InitializingBean.afterPropertiesSet()
9.  Custom init-method (from @Bean(initMethod = "init"))
10. BeanPostProcessor.postProcessAfterInitialization()  ← Proxies created here (AOP, @Transactional)
    ──── Bean is READY ────
11. @PreDestroy method
12. DisposableBean.destroy()
13. Custom destroy-method (from @Bean(destroyMethod = "cleanup"))
```

```java
@Component
public class PaymentProcessor implements BeanNameAware, InitializingBean, DisposableBean {

    private String beanName;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("1. BeanNameAware: " + name);
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("2. @PostConstruct: Initialize payment gateway connection");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("3. InitializingBean: Validate configuration");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("4. @PreDestroy: Close payment gateway connection");
    }

    @Override
    public void destroy() {
        System.out.println("5. DisposableBean: Final cleanup");
    }
}
```

**Real-world use case (PayU Lending Platform):**
- `@PostConstruct` — Warm up caches, validate API keys, establish connection pools.
- `@PreDestroy` — Gracefully close DB connections, flush pending audit logs.

**Interview Follow-up:** "In which phase are AOP proxies created?" — In `postProcessAfterInitialization()` by `AbstractAutoProxyCreator`.

---

### Q4. Explain Bean Scopes with use cases. [Medium]

| Scope | Instances | Lifecycle | Use Case |
|---|---|---|---|
| `singleton` (default) | One per ApplicationContext | App lifetime | Stateless services, repositories |
| `prototype` | New instance per injection/request | Not managed after creation | Stateful beans, command objects |
| `request` | One per HTTP request | Request lifetime | Request-scoped audit context |
| `session` | One per HTTP session | Session lifetime | Shopping cart, user preferences |
| `application` | One per ServletContext | App lifetime (like singleton in web) | Shared counters |
| `websocket` | One per WebSocket session | WebSocket lifetime | Chat session state |

```java
@Component
@Scope("prototype")
public class LoanApplication {
    private String applicantId;
    private BigDecimal amount;
    // Each loan application gets its own instance
}

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestAuditContext {
    private String correlationId;
    private Instant requestTime;
}
```

> **Anti-Pattern:** Injecting a `prototype` bean into a `singleton` — the prototype is resolved once and the same instance is reused. Fix: use `ObjectProvider<T>` or `@Lookup`.

```java
@Service
public class LoanProcessingService {

    private final ObjectProvider<LoanApplication> loanAppProvider;

    public LoanProcessingService(ObjectProvider<LoanApplication> loanAppProvider) {
        this.loanAppProvider = loanAppProvider;
    }

    public void process() {
        LoanApplication app = loanAppProvider.getObject(); // fresh instance each time
    }
}
```

---

### Q5. @Component vs @Service vs @Repository vs @Controller — what's the difference? [Easy]

All are **stereotype annotations** and are specializations of `@Component`. Spring treats them the same for component scanning, but they carry semantic meaning and some have extra behavior:

| Annotation | Layer | Special Behavior |
|---|---|---|
| `@Component` | Generic | Base annotation — no extras |
| `@Service` | Business logic | None — purely semantic |
| `@Repository` | Data access | **Exception translation**: catches platform-specific exceptions (JDBC, JPA) and wraps them as Spring's `DataAccessException` |
| `@Controller` | Web / MVC | Enables `@RequestMapping`, returns view names |
| `@RestController` | Web / REST | `@Controller` + `@ResponseBody` on every method |

```java
@Repository  // enables PersistenceExceptionTranslationPostProcessor
public class LoanRepositoryImpl implements LoanRepository {
    @PersistenceContext
    private EntityManager em;

    public Loan findById(Long id) {
        return em.find(Loan.class, id); // JPA exception → DataAccessException
    }
}
```

**Interview Follow-up:** "Can I use `@Component` everywhere?" — Yes, it works, but you lose semantic clarity and `@Repository`'s exception translation.

---

### Q6. @Autowired vs @Inject vs @Resource — what's the difference? [Easy]

| Feature | `@Autowired` (Spring) | `@Inject` (JSR-330) | `@Resource` (JSR-250) |
|---|---|---|---|
| Package | `org.springframework` | `javax.inject` | `javax.annotation` |
| Match by | **Type** first, then qualifier | **Type** first, then qualifier | **Name** first, then type |
| `required` attribute | ✅ `@Autowired(required=false)` | ❌ Use `Optional<T>` | ❌ |
| Qualifier | `@Qualifier("name")` | `@Named("name")` | `@Resource(name="name")` |

```java
// All three do the same thing here:
@Autowired
private PaymentGateway gateway;

@Inject
private PaymentGateway gateway;

@Resource(name = "paymentGateway")
private PaymentGateway gateway;
```

> **Interview Tip:** Prefer `@Autowired` with constructor injection in Spring projects. Use `@Inject` if you want framework-agnostic code (rare in practice).

---

### Q7. How does Spring handle Circular Dependencies? How to avoid them? [Hard]

**What is it?** Bean A depends on Bean B, and Bean B depends on Bean A.

**How Spring handles it (for singleton + setter/field injection):**

Spring uses a **three-level cache** system:
1. `singletonObjects` — Fully initialized beans
2. `earlySingletonObjects` — Early references (partially initialized)
3. `singletonFactories` — Object factories that produce early references

```
Creating A → A needs B → Creating B → B needs A
  → A is in singletonFactories (partially created)
  → B gets early reference of A → B completes
  → A gets fully initialized B → A completes
```

**This does NOT work with constructor injection** — Spring throws `BeanCurrentlyInCreationException` because neither bean can be constructed without the other.

**Solutions:**

```java
// 1. Use @Lazy on one side
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(@Lazy PaymentService paymentService) {
        this.paymentService = paymentService; // injects a proxy, resolved on first use
    }
}

// 2. Redesign: extract common logic into a third service
@Service
public class OrderPaymentMediator {
    private final OrderRepository orderRepo;
    private final PaymentGateway gateway;
    // Both OrderService and PaymentService depend on this instead of each other
}

// 3. Use ApplicationEventPublisher (event-driven decoupling)
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public void completeOrder(Order order) {
        publisher.publishEvent(new OrderCompletedEvent(order));
    }
}
```

> **Anti-Pattern:** Circular dependencies are a design smell. If you have them, your services are too tightly coupled. Refactor.

---

### Q8. ApplicationContext vs BeanFactory [Easy]

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Bean instantiation | **Lazy** (on first `getBean()`) | **Eager** (singletons at startup) |
| Event publishing | ❌ | ✅ `ApplicationEventPublisher` |
| i18n (MessageSource) | ❌ | ✅ |
| AOP integration | ❌ | ✅ Auto-proxy creation |
| `@PostConstruct` / `@PreDestroy` | Manual registration needed | ✅ Automatic |
| `Environment` & profiles | ❌ | ✅ |

`ApplicationContext` extends `BeanFactory` and adds enterprise features. In modern Spring Boot, you always use `ApplicationContext` (specifically `AnnotationConfigServletWebServerApplicationContext`).

---

## 2. Spring Boot Specifics

---

### Q9. How does Spring Boot Auto-Configuration work? [Hard]

`@SpringBootApplication` is a composed annotation:

```java
@SpringBootApplication
// is equivalent to:
@SpringBootConfiguration   // → @Configuration
@EnableAutoConfiguration    // → triggers auto-config
@ComponentScan              // → scans current package & sub-packages
```

**Auto-configuration flow:**

1. `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`.
2. The selector reads from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x) or `META-INF/spring.factories` (pre-3.x).
3. Each listed class (e.g., `DataSourceAutoConfiguration`) is loaded.
4. `@Conditional*` annotations on each class are evaluated:
   - Class present on classpath? (`@ConditionalOnClass`)
   - Property set? (`@ConditionalOnProperty`)
   - No user-defined bean? (`@ConditionalOnMissingBean`)
5. Only matching auto-configurations are applied.

```java
// Example: How DataSource auto-config works
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean  // user can override by defining their own DataSource bean
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
                .url(properties.getUrl())
                .username(properties.getUsername())
                .password(properties.getPassword())
                .build();
    }
}
```

**Debugging auto-config:**
- Run with `--debug` flag or set `debug=true` → prints `CONDITIONS EVALUATION REPORT`
- Lists: Positive matches, Negative matches, Exclusions

> **Interview Tip:** "Auto-configuration is opinionated defaults that back off when you provide your own beans."

---

### Q10. Explain @Conditional annotations with examples. [Medium]

| Annotation | Condition |
|---|---|
| `@ConditionalOnClass` | Class is on the classpath |
| `@ConditionalOnMissingClass` | Class is NOT on the classpath |
| `@ConditionalOnBean` | A specific bean exists |
| `@ConditionalOnMissingBean` | A specific bean does NOT exist |
| `@ConditionalOnProperty` | A property has a specific value |
| `@ConditionalOnExpression` | SpEL expression evaluates to true |
| `@ConditionalOnWebApplication` | Running in a web context |
| `@ConditionalOnJava` | Specific Java version |

```java
@Configuration
public class CacheConfig {

    @Bean
    @ConditionalOnProperty(name = "app.cache.type", havingValue = "redis")
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory).build();
    }

    @Bean
    @ConditionalOnProperty(name = "app.cache.type", havingValue = "caffeine", matchIfMissing = true)
    public CacheManager caffeineCacheManager() {
        return new CaffeineCacheManager("loans", "users");
    }
}

// Custom conditional
public class OnKubernetesCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return System.getenv("KUBERNETES_SERVICE_HOST") != null;
    }
}

@Bean
@Conditional(OnKubernetesCondition.class)
public ServiceDiscovery k8sServiceDiscovery() { ... }
```

---

### Q11. How do Spring Profiles work? [Easy]

Profiles allow environment-specific configuration.

**Activation methods:**
```properties
# application.yml
spring.profiles.active=dev

# JVM argument
-Dspring.profiles.active=prod

# Environment variable
SPRING_PROFILES_ACTIVE=staging

# Programmatic
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("dev");
```

**Profile-specific config files:**
```
application.yml              ← common/default
application-dev.yml          ← dev overrides
application-staging.yml      ← staging overrides
application-prod.yml         ← prod overrides
```

**Profile-specific beans:**
```java
@Configuration
public class NotificationConfig {

    @Bean
    @Profile("dev")
    public NotificationService mockNotificationService() {
        return new MockNotificationService(); // logs instead of sending
    }

    @Bean
    @Profile("prod")
    public NotificationService smsNotificationService(SmsGateway gateway) {
        return new SmsNotificationService(gateway);
    }
}
```

**Profile groups (Spring Boot 2.4+):**
```yaml
spring:
  profiles:
    group:
      prod:
        - proddb
        - prodmq
        - prodmetrics
```

---

### Q12. @ConfigurationProperties vs @Value — when to use which? [Medium]

```java
// ❌ @Value — fine for 1-2 simple properties
@Component
public class SimpleConfig {
    @Value("${app.name}")
    private String appName;

    @Value("${app.timeout:30}")  // default value
    private int timeout;

    @Value("${app.features.enabled:false}")
    private boolean featuresEnabled;
}

// ✅ @ConfigurationProperties — structured, type-safe, validated
@ConfigurationProperties(prefix = "lending")
@Validated
public class LendingProperties {

    @NotBlank
    private String apiKey;

    @Min(1000) @Max(5000000)
    private BigDecimal maxLoanAmount;

    @Valid
    private RetryConfig retry = new RetryConfig();

    @Valid
    private List<PartnerConfig> partners = new ArrayList<>();

    // getters/setters

    public static class RetryConfig {
        private int maxAttempts = 3;
        private Duration backoff = Duration.ofSeconds(2);
        // getters/setters
    }

    public static class PartnerConfig {
        @NotBlank
        private String name;
        private String baseUrl;
        // getters/setters
    }
}
```

```yaml
# application.yml
lending:
  api-key: ${LENDING_API_KEY}
  max-loan-amount: 1000000
  retry:
    max-attempts: 5
    backoff: 3s
  partners:
    - name: CIBIL
      base-url: https://api.cibil.com
    - name: Experian
      base-url: https://api.experian.com
```

Enable it in your config class:
```java
@Configuration
@EnableConfigurationProperties(LendingProperties.class)
public class AppConfig { }
```

| Aspect | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Type safety | ❌ String-based | ✅ Strongly typed |
| Validation | ❌ | ✅ JSR-303 (`@Valid`, `@NotBlank`) |
| Nested properties | ❌ Painful | ✅ Natural with inner classes |
| IDE support | ❌ | ✅ With `spring-boot-configuration-processor` |
| Relaxed binding | ❌ | ✅ `max-loan-amount` = `maxLoanAmount` |

---

### Q13. What is Spring Boot Actuator and how to create custom endpoints? [Medium]

Actuator exposes production-ready operational endpoints:

| Endpoint | Purpose |
|---|---|
| `/actuator/health` | App health status |
| `/actuator/info` | App information |
| `/actuator/metrics` | Micrometer metrics |
| `/actuator/env` | Environment properties |
| `/actuator/beans` | All registered beans |
| `/actuator/mappings` | All `@RequestMapping` endpoints |
| `/actuator/threaddump` | Thread dump |
| `/actuator/prometheus` | Prometheus-formatted metrics |

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

**Custom health indicator:**
```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGateway gateway;

    public PaymentGatewayHealthIndicator(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    @Override
    public Health health() {
        try {
            gateway.ping();
            return Health.up()
                    .withDetail("gateway", "reachable")
                    .withDetail("latency", gateway.getLatencyMs() + "ms")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("gateway", "unreachable")
                    .withException(e)
                    .build();
        }
    }
}
```

**Custom actuator endpoint:**
```java
@Component
@Endpoint(id = "loan-stats")
public class LoanStatsEndpoint {

    private final LoanRepository loanRepository;

    public LoanStatsEndpoint(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    @ReadOperation
    public Map<String, Object> loanStatistics() {
        return Map.of(
            "totalLoans", loanRepository.count(),
            "activeLoans", loanRepository.countByStatus(LoanStatus.ACTIVE),
            "defaultRate", loanRepository.calculateDefaultRate()
        );
    }

    @ReadOperation
    public Map<String, Object> loanStatsByPartner(@Selector String partnerId) {
        return Map.of(
            "partner", partnerId,
            "disbursed", loanRepository.countByPartner(partnerId)
        );
    }
}
```

---

### Q14. How to configure the embedded server? [Easy]

```yaml
server:
  port: 8443
  servlet:
    context-path: /api
  compression:
    enabled: true
    mime-types: application/json,text/html
    min-response-size: 1024
  http2:
    enabled: true
  ssl:
    key-store: classpath:keystore.p12
    key-store-type: PKCS12
    key-store-password: ${SSL_PASSWORD}
  tomcat:
    max-threads: 200
    min-spare-threads: 20
    max-connections: 10000
    accept-count: 100
    connection-timeout: 5000
```

**Programmatic customization:**
```java
@Component
public class ServerCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(9090);
        factory.addConnectorCustomizers(connector -> {
            connector.setMaxPostSize(10 * 1024 * 1024); // 10MB
        });
    }
}
```

**Switching to Jetty/Undertow:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

---

## 3. Spring Security

---

### Q15. Explain the Spring Security Filter Chain architecture. [Hard]

**Request flow through the security filter chain:**

```
HTTP Request
  │
  ▼
DelegatingFilterProxy (Servlet Filter — bridges to Spring)
  │
  ▼
FilterChainProxy (Spring Security entry point)
  │
  ▼
SecurityFilterChain (matched by URL pattern)
  │
  ├── DisableEncodeUrlFilter
  ├── WebAsyncManagerIntegrationFilter
  ├── SecurityContextHolderFilter
  ├── HeaderWriterFilter
  ├── CorsFilter
  ├── CsrfFilter
  ├── LogoutFilter
  ├── UsernamePasswordAuthenticationFilter / JwtAuthFilter (custom)
  ├── RequestCacheAwareFilter
  ├── SecurityContextHolderAwareRequestFilter
  ├── AnonymousAuthenticationFilter
  ├── SessionManagementFilter
  ├── ExceptionTranslationFilter
  └── AuthorizationFilter (formerly FilterSecurityInterceptor)
  │
  ▼
DispatcherServlet → Controller
```

**Modern Spring Security 6.x configuration (lambda DSL):**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/loans/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new Http401EntryPoint())
                .accessDeniedHandler(new Http403Handler())
            )
            .build();
    }
}
```

> **Interview Tip:** "Spring Security is fundamentally a chain of servlet filters. Each filter handles one concern (CSRF, auth, authz, etc.). You can insert custom filters at any position."

---

### Q16. Explain JWT Authentication flow in Spring Security. [Hard]

**Complete JWT flow:**

```
1. Client sends POST /api/auth/login { username, password }
2. AuthController → AuthenticationManager.authenticate()
3. AuthenticationManager → UserDetailsService.loadUserByUsername()
4. Password verified via PasswordEncoder.matches()
5. JWT generated with claims (sub, roles, exp) and signed
6. Token returned to client

── For subsequent requests ──

7. Client sends: Authorization: Bearer <token>
8. JwtAuthFilter intercepts request
9. Extract token from header
10. Validate token (signature, expiration, issuer)
11. Extract username + authorities from claims
12. Create UsernamePasswordAuthenticationToken
13. Set in SecurityContextHolder
14. Request proceeds to controller
```

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);

        try {
            String username = jwtService.extractUsername(token);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {
                    var authToken = new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration:86400000}") // 24 hours
    private long expiration;

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList());

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        Claims claims = Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
        return resolver.apply(claims);
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }
}
```

**Interview Follow-up:** "How do you handle token refresh?" — Use a separate refresh token with longer expiry stored in HttpOnly cookie. Access token is short-lived (15 min), refresh token is long-lived (7 days).

---

### Q17. Explain OAuth2 / SSO integration in Spring Boot. [Hard]

**OAuth2 Authorization Code Flow:**

```
1. User clicks "Login with Google"
2. Redirect to Google's /authorize endpoint
3. User authenticates & consents
4. Google redirects back with authorization code
5. Spring exchanges code for access token (server-to-server)
6. Spring fetches user info from Google's /userinfo
7. Session created, user logged in
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth -> oauth
                .userInfoEndpoint(info -> info
                    .userService(customOAuth2UserService())
                )
                .successHandler(oAuth2SuccessHandler())
            )
            .build();
    }

    @Bean
    public OAuth2UserService<OAuth2UserRequest, OAuth2User> customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }
}

@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    public CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public OAuth2User loadUser(OAuth2UserRequest request) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(request);

        String email = oAuth2User.getAttribute("email");
        String provider = request.getClientRegistration().getRegistrationId();

        User user = userRepository.findByEmail(email)
                .orElseGet(() -> createNewUser(oAuth2User, provider));

        return new CustomOAuth2User(user, oAuth2User.getAttributes());
    }
}
```

---

### Q18. Explain method-level security annotations. [Medium]

```java
@Configuration
@EnableMethodSecurity // replaces @EnableGlobalMethodSecurity in Spring Security 6
public class MethodSecurityConfig { }
```

```java
@Service
public class LoanService {

    // SpEL-based — most flexible
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public LoanDetails getLoanDetails(Long userId, Long loanId) { ... }

    // Check AFTER method execution — access to return value
    @PostAuthorize("returnObject.userId == authentication.principal.id")
    public Loan findLoan(Long loanId) { ... }

    // Simple role check — no SpEL
    @Secured("ROLE_ADMIN")
    public void deleteLoan(Long loanId) { ... }

    // JSR-250
    @RolesAllowed({"ADMIN", "MANAGER"})
    public void approveLoan(Long loanId) { ... }

    // Filter collections
    @PreFilter("filterObject.status != 'CLOSED'")
    public void processLoans(List<Loan> loans) { ... }

    @PostFilter("filterObject.userId == authentication.principal.id")
    public List<Loan> getAllLoans() { ... }
}
```

> **Interview Tip:** `@PreAuthorize` is the most commonly used. `@PostAuthorize` is useful when authorization depends on the returned data. Avoid `@Secured` in new code — it doesn't support SpEL.

---

### Q19. How to configure CORS in Spring Boot? [Easy]

```java
// Method 1: Global CORS via SecurityFilterChain
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.mycompany.com", "https://admin.mycompany.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Request-Id"));
    config.setExposedHeaders(List.of("X-Total-Count", "X-Correlation-Id"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}

// Method 2: Per-controller
@RestController
@CrossOrigin(origins = "https://app.mycompany.com", maxAge = 3600)
@RequestMapping("/api/loans")
public class LoanController { ... }
```

> **Anti-Pattern:** Never use `allowedOrigins("*")` with `allowCredentials(true)` — browsers block this. Use `allowedOriginPatterns("*")` if you must.

---

### Q20. Password Encoding in Spring Security [Medium]

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // BCrypt — most common, good default
    return new BCryptPasswordEncoder(12); // strength 12 (2^12 rounds)
}

// Or use DelegatingPasswordEncoder for migration support
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // Stores as: {bcrypt}$2a$12$... or {argon2}$argon2id$...
    // Allows migrating from one algorithm to another
}

// Argon2 — recommended for new projects (memory-hard, resistant to GPU attacks)
@Bean
public PasswordEncoder passwordEncoder() {
    return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
    // saltLength, hashLength, parallelism, memory (KB), iterations
}
```

```java
@Service
public class AuthService {

    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public void register(RegisterRequest request) {
        User user = new User();
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        userRepository.save(user);
    }

    public boolean authenticate(String rawPassword, String storedHash) {
        return passwordEncoder.matches(rawPassword, storedHash);
    }
}
```

> **Anti-Pattern:** Never store passwords in plain text. Never use MD5/SHA for password hashing — they're fast algorithms designed for checksums, not password storage.

---

## 4. Advanced Spring Boot

---

### Q21. How does @Async work? What are common pitfalls? [Hard]

`@Async` makes a method execute in a separate thread. Spring achieves this via **AOP proxy** — it wraps the method call to submit it to a `TaskExecutor`.

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "notificationExecutor")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("notification-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {

    @Async("notificationExecutor")
    public CompletableFuture<Boolean> sendEmail(String to, String subject, String body) {
        // runs in a separate thread from notificationExecutor pool
        emailClient.send(to, subject, body);
        return CompletableFuture.completedFuture(true);
    }

    @Async("notificationExecutor")
    public void sendSms(String phone, String message) {
        // fire-and-forget
        smsGateway.send(phone, message);
    }
}
```

**Common Pitfalls:**

```java
// ❌ PITFALL 1: Calling @Async method from the SAME class — bypasses proxy!
@Service
public class OrderService {

    public void processOrder(Order order) {
        sendConfirmation(order); // ← Direct call, NOT async! Proxy is bypassed.
    }

    @Async
    public void sendConfirmation(Order order) { ... }
}

// ✅ FIX: Inject self or move to another service
@Service
public class OrderService {

    private final NotificationService notificationService; // separate service

    public void processOrder(Order order) {
        notificationService.sendConfirmation(order); // goes through proxy → async
    }
}

// ❌ PITFALL 2: Swallowed exceptions in void @Async methods
// ✅ FIX: Configure AsyncUncaughtExceptionHandler
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async error in {}: {}", method.getName(), ex.getMessage(), ex);
        };
    }
}

// ❌ PITFALL 3: Using @Async on private methods — AOP proxies can't intercept private methods
// ✅ FIX: Always use public methods
```

> **Interview Tip:** "The #1 mistake with `@Async` is calling the async method from within the same bean. Since Spring uses proxy-based AOP, the call bypasses the proxy and runs synchronously."

---

### Q22. Explain @Scheduled — cron expressions, fixedRate vs fixedDelay. [Medium]

```java
@Configuration
@EnableScheduling
public class SchedulingConfig { }

@Component
public class LoanScheduler {

    // fixedRate: runs every 5 seconds REGARDLESS of previous execution
    // If execution takes 3s → next starts 2s after it finishes
    // If execution takes 7s → next starts immediately after
    @Scheduled(fixedRate = 5000)
    public void checkPendingRepayments() { ... }

    // fixedDelay: waits 5 seconds AFTER previous execution completes
    // If execution takes 3s → next starts 5s after finish = 8s apart
    @Scheduled(fixedDelay = 5000)
    public void syncPartnerData() { ... }

    // initialDelay: wait before first execution
    @Scheduled(fixedRate = 60000, initialDelay = 10000)
    public void warmUpCache() { ... }

    // Cron: second minute hour day-of-month month day-of-week
    @Scheduled(cron = "0 0 2 * * ?")     // Every day at 2:00 AM
    public void generateDailyReport() { ... }

    @Scheduled(cron = "0 */15 * * * ?")   // Every 15 minutes
    public void pollExternalService() { ... }

    @Scheduled(cron = "0 0 9 1 * ?")      // 1st of every month at 9:00 AM
    public void monthlyInterestCalculation() { ... }

    // Externalized cron expression
    @Scheduled(cron = "${loan.overdue.check.cron:0 0 */6 * * ?}")
    public void checkOverdueLoans() { ... }
}
```

| `fixedRate` | `fixedDelay` |
|---|---|
| Timer starts when method **starts** | Timer starts when method **finishes** |
| Can overlap if execution > interval | Never overlaps |
| Use for periodic polling | Use for sequential processing |

> **Anti-Pattern:** Don't run `@Scheduled` tasks in clustered environments without coordination — all instances will execute the same task. Use ShedLock or a distributed scheduler.

```java
// ShedLock integration to prevent duplicate execution in clusters
@Scheduled(cron = "0 0 2 * * ?")
@SchedulerLock(name = "dailyReport", lockAtMostFor = "PT30M", lockAtLeastFor = "PT5M")
public void generateDailyReport() { ... }
```

---

### Q23. Explain AOP concepts: @Aspect, @Before, @After, @Around. [Hard]

**AOP Terminology:**
- **Aspect** — Cross-cutting concern (logging, security, transactions)
- **Join Point** — A point during execution (method call)
- **Advice** — Action taken at a join point (before, after, around)
- **Pointcut** — Predicate that matches join points
- **Weaving** — Linking aspects with target objects (Spring uses runtime proxy-based weaving)

```java
@Aspect
@Component
@Order(1) // execution order among aspects
public class PerformanceAspect {

    // @Before — runs before the target method
    @Before("execution(* com.lending.service.*.*(..))")
    public void logMethodEntry(JoinPoint jp) {
        log.info("Entering: {}.{}()", jp.getTarget().getClass().getSimpleName(),
                jp.getSignature().getName());
    }

    // @After — runs after the method (regardless of outcome)
    @After("execution(* com.lending.service.*.*(..))")
    public void logMethodExit(JoinPoint jp) {
        log.info("Exiting: {}.{}", jp.getTarget().getClass().getSimpleName(),
                jp.getSignature().getName());
    }

    // @AfterReturning — runs only on successful return
    @AfterReturning(pointcut = "execution(* com.lending.service.LoanService.disburse(..))",
                    returning = "result")
    public void afterDisbursement(JoinPoint jp, DisbursementResult result) {
        log.info("Loan {} disbursed: {}", result.getLoanId(), result.getAmount());
    }

    // @AfterThrowing — runs only when exception is thrown
    @AfterThrowing(pointcut = "execution(* com.lending.service.*.*(..))",
                   throwing = "ex")
    public void afterException(JoinPoint jp, Exception ex) {
        log.error("Exception in {}: {}", jp.getSignature().getName(), ex.getMessage());
    }

    // @Around — wraps the method (most powerful)
    @Around("@annotation(com.lending.annotation.Timed)")
    public Object measureExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed(); // actually call the method
            return result;
        } finally {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.info("{}.{} took {}ms", pjp.getTarget().getClass().getSimpleName(),
                    pjp.getSignature().getName(), elapsed);
        }
    }
}

// Custom annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timed { }

// Usage
@Service
public class LoanService {
    @Timed
    public Loan processLoanApplication(LoanRequest request) { ... }
}
```

**Common Pointcut Expressions:**
```java
execution(* com.lending.service.*.*(..))       // any method in service package
execution(public * *(..))                       // any public method
execution(* com.lending..*Repository.find*(..)) // find* methods in repositories
@annotation(org.springframework.transaction.annotation.Transactional) // methods with @Transactional
within(com.lending.service.*)                   // all methods within service package
bean(*Service)                                  // all beans ending with "Service"
```

---

### Q24. Filters vs Interceptors vs AOP — comparison [Medium]

| Aspect | Servlet Filter | HandlerInterceptor | AOP |
|---|---|---|---|
| **Level** | Servlet container | Spring MVC | Spring bean |
| **Scope** | All requests (inc. static) | Only `@Controller` requests | Any Spring bean method |
| **Access to** | `HttpServletRequest/Response` | `Handler` + `ModelAndView` | Method args, return value |
| **Use cases** | Auth, logging, CORS, compression | Logging, locale, tenant | Transactions, caching, metrics |
| **Order** | `@Order` / `FilterRegistrationBean` | `@Order` / `registry.addInterceptor().order()` | `@Order` on `@Aspect` |
| **Exception** | No `@ControllerAdvice` | No `@ControllerAdvice` | ✅ Can use with `@ControllerAdvice` |

```
Request → [Filters] → DispatcherServlet → [Interceptors preHandle] → Controller
Controller → [Interceptors postHandle] → ViewResolver → [Interceptors afterCompletion] → [Filters] → Response

AOP wraps service-layer method calls (independent of web layer)
```

```java
// Filter
@Component
@Order(1)
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-Id"))
                .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId);
        response.setHeader("X-Correlation-Id", correlationId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("correlationId");
        }
    }
}

// Interceptor
@Component
public class RequestTimingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        long duration = System.currentTimeMillis() - (long) request.getAttribute("startTime");
        log.info("{} {} completed in {}ms", request.getMethod(), request.getRequestURI(), duration);
    }
}
```

---

### Q25. Exception Handling with @ControllerAdvice and @ExceptionHandler [Medium]

```java
// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.lending.com/errors/not-found"));
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", Instant.now());
        problem.setProperty("errorCode", "LOAN_NOT_FOUND");
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");

        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
                errors.put(error.getField(), error.getDefaultMessage()));
        problem.setProperty("fieldErrors", errors);
        return problem;
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ProblemDetail handleConflict(OptimisticLockingFailureException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.CONFLICT,
                "The resource was modified by another user. Please refresh and try again.");
        problem.setTitle("Concurrent Modification");
        problem.setProperty("errorCode", "CONCURRENT_UPDATE");
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR,
                "An unexpected error occurred. Please contact support.");
        problem.setTitle("Internal Server Error");
        return problem;
    }
}
```

**ProblemDetail response (RFC 7807):**
```json
{
  "type": "https://api.lending.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Loan with ID 12345 not found",
  "instance": "/api/loans/12345",
  "timestamp": "2026-03-12T10:30:00Z",
  "errorCode": "LOAN_NOT_FOUND"
}
```

**Custom exception hierarchy:**
```java
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;

    protected BusinessException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String resource, Object id) {
        super(resource + " with ID " + id + " not found", resource.toUpperCase() + "_NOT_FOUND");
    }
}

public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException(BigDecimal requested, BigDecimal available) {
        super("Requested " + requested + " but only " + available + " available",
              "INSUFFICIENT_FUNDS");
    }
}
```

> **Interview Tip:** Spring Boot 3+ supports RFC 7807 `ProblemDetail` natively. Enable it with `spring.mvc.problemdetails.enabled=true`. Always use structured error responses in APIs — never return plain strings.

---

### Q26. Explain @Retryable and @Recover. [Medium]

Spring Retry provides declarative retry logic for transient failures (network timeouts, DB deadlocks, rate limits).

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

```java
@Configuration
@EnableRetry
public class RetryConfig { }

@Service
public class CreditScoreService {

    @Retryable(
        retryFor = { RestClientException.class, TimeoutException.class },
        noRetryFor = { IllegalArgumentException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
    )
    public CreditScore fetchCreditScore(String panNumber) {
        log.info("Fetching credit score for PAN: {}", panNumber);
        return creditBureauClient.getScore(panNumber); // may throw on timeout
    }

    @Recover
    public CreditScore recoverCreditScore(RestClientException ex, String panNumber) {
        log.warn("All retries exhausted for PAN: {}. Using cached score.", panNumber);
        return creditScoreCache.getLastKnownScore(panNumber)
                .orElseThrow(() -> new ServiceUnavailableException(
                    "Credit bureau unavailable and no cached score for " + panNumber));
    }

    @Recover
    public CreditScore recoverFromTimeout(TimeoutException ex, String panNumber) {
        log.warn("Timeout fetching score for PAN: {}. Queuing for async fetch.", panNumber);
        asyncScoreFetcher.queue(panNumber);
        return CreditScore.pending();
    }
}
```

**Retry flow:**
```
Attempt 1 → RestClientException → wait 1s
Attempt 2 → RestClientException → wait 2s (1s × multiplier 2)
Attempt 3 → RestClientException → @Recover method called
```

**Programmatic retry (RetryTemplate):**
```java
@Bean
public RetryTemplate retryTemplate() {
    return RetryTemplate.builder()
            .maxAttempts(4)
            .exponentialBackoff(500, 2, 8000)
            .retryOn(RestClientException.class)
            .traversingCauses()
            .build();
}

// Usage
retryTemplate.execute(ctx -> externalApiClient.call(request));
```

> **Anti-Pattern:** Don't retry non-idempotent operations (like payment deduction) without ensuring idempotency keys.

---

### Q27. Event-driven architecture with Spring Events. [Medium]

```java
// 1. Define Event
public class LoanApprovedEvent {
    private final Long loanId;
    private final String userId;
    private final BigDecimal amount;
    private final Instant approvedAt;

    public LoanApprovedEvent(Long loanId, String userId, BigDecimal amount) {
        this.loanId = loanId;
        this.userId = userId;
        this.amount = amount;
        this.approvedAt = Instant.now();
    }
    // getters
}

// 2. Publish Event
@Service
public class LoanService {

    private final ApplicationEventPublisher eventPublisher;
    private final LoanRepository loanRepository;

    public LoanService(ApplicationEventPublisher eventPublisher, LoanRepository loanRepository) {
        this.eventPublisher = eventPublisher;
        this.loanRepository = loanRepository;
    }

    @Transactional
    public Loan approveLoan(Long loanId) {
        Loan loan = loanRepository.findById(loanId)
                .orElseThrow(() -> new ResourceNotFoundException("Loan", loanId));
        loan.setStatus(LoanStatus.APPROVED);
        loanRepository.save(loan);
        eventPublisher.publishEvent(
                new LoanApprovedEvent(loan.getId(), loan.getUserId(), loan.getAmount()));
        return loan;
    }
}

// 3. Listen to Event
@Component
public class LoanEventListeners {

    @EventListener
    public void onLoanApproved(LoanApprovedEvent event) {
        log.info("Loan {} approved for user {}", event.getLoanId(), event.getUserId());
        notificationService.sendApprovalNotification(event.getUserId(), event.getAmount());
    }

    // Only runs AFTER the transaction commits successfully
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onLoanApprovedAfterCommit(LoanApprovedEvent event) {
        disbursementService.initiateDisbursement(event.getLoanId());
    }

    // Runs if the transaction rolls back
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onLoanApprovalFailed(LoanApprovedEvent event) {
        alertService.notifyApprovalRollback(event.getLoanId());
    }

    // Async event handling
    @Async
    @EventListener
    public void onLoanApprovedAsync(LoanApprovedEvent event) {
        analyticsService.trackLoanApproval(event);
    }
}
```

**@EventListener vs @TransactionalEventListener:**

| Feature | `@EventListener` | `@TransactionalEventListener` |
|---|---|---|
| When it runs | Immediately when published | After transaction phase (default: AFTER_COMMIT) |
| Transaction context | Inside publisher's transaction | After publisher's transaction |
| Use case | In-process notifications | Side effects that must only happen if TX succeeds |

> **Interview Tip:** In a lending platform, always use `@TransactionalEventListener(phase = AFTER_COMMIT)` for disbursement triggers. You don't want to initiate a bank transfer if the loan approval transaction rolls back.

---

### Q28. How does @Transactional work with proxy mechanism? [Hard]

See the JPA file for comprehensive `@Transactional` coverage. Here's the proxy mechanism overview:

```
CallerClass → $Proxy (TransactionInterceptor) → Actual Bean

The proxy:
1. Checks @Transactional attributes (propagation, isolation, etc.)
2. Gets a DB connection from DataSource
3. Sets connection.setAutoCommit(false)
4. Calls the actual method
5. If no exception → connection.commit()
6. If RuntimeException → connection.rollback()
7. Returns connection to pool
```

**Key gotcha: self-invocation bypasses the proxy**

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Order order) {
        saveOrder(order);
        updateInventory(order); // ← Direct call — @Transactional on this method is IGNORED
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateInventory(Order order) {
        // This does NOT run in a new transaction when called from processOrder()
        // because the call bypasses the proxy
    }
}

// ✅ Fix: Extract to a separate service
@Service
public class InventoryService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateInventory(Order order) { ... }
}
```

---

### Q29. Explain Filters in Spring Boot — how to register and order them. [Easy]

```java
// Method 1: @Component (auto-registered)
@Component
@Order(1)
public class LoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        log.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        chain.doFilter(request, response);
        log.info("Response: {} for {} {}", response.getStatus(),
                request.getMethod(), request.getRequestURI());
    }
}

// Method 2: FilterRegistrationBean (more control)
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        FilterRegistrationBean<RateLimitFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new RateLimitFilter());
        registration.addUrlPatterns("/api/*");
        registration.setOrder(2);
        registration.setName("rateLimitFilter");
        return registration;
    }
}
```

---

### Q30. What is ProblemDetail (RFC 7807) and why should you use it? [Medium]

RFC 7807 defines a standard format for HTTP API error responses. Spring Boot 3+ supports it natively.

```java
// Enable globally
// application.yml
// spring.mvc.problemdetails.enabled: true

// Spring Boot 3 auto-configures ProblemDetail responses for:
// - MethodArgumentNotValidException → 400
// - HttpRequestMethodNotSupportedException → 405
// - NoHandlerFoundException → 404
// - TypeMismatchException → 400

// Custom ProblemDetail
@RestControllerAdvice
public class LoanExceptionHandler {

    @ExceptionHandler(LoanLimitExceededException.class)
    public ProblemDetail handleLoanLimit(LoanLimitExceededException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
                HttpStatus.UNPROCESSABLE_ENTITY,
                ex.getMessage());
        pd.setTitle("Loan Limit Exceeded");
        pd.setType(URI.create("https://docs.lending.com/errors/loan-limit"));
        pd.setProperty("currentLimit", ex.getCurrentLimit());
        pd.setProperty("requestedAmount", ex.getRequestedAmount());
        pd.setProperty("suggestion", "Complete existing loans or request a limit increase");
        return pd;
    }
}
```

**Standard ProblemDetail fields:**
| Field | Required | Description |
|---|---|---|
| `type` | ✅ | URI identifying the problem type |
| `title` | ✅ | Short human-readable summary |
| `status` | ✅ | HTTP status code |
| `detail` | ❌ | Human-readable explanation specific to this occurrence |
| `instance` | ❌ | URI identifying the specific occurrence |
| + extensions | ❌ | Custom fields (errorCode, timestamp, etc.) |

---

### Q31. How to create a custom Spring Boot Starter? [Hard]

**Real-world scenario:** In a lending platform, you have 15 microservices that all need the same audit logging, error handling, and security configuration. Create a shared starter.

**Project structure:**
```
lending-spring-boot-starter/
├── src/main/java/com/lending/starter/
│   ├── audit/
│   │   ├── AuditAutoConfiguration.java
│   │   ├── AuditFilter.java
│   │   └── AuditProperties.java
│   └── error/
│       ├── ErrorAutoConfiguration.java
│       └── GlobalErrorHandler.java
├── src/main/resources/
│   └── META-INF/
│       └── spring/
│           └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── pom.xml
```

```java
// AuditProperties.java
@ConfigurationProperties(prefix = "lending.audit")
public class AuditProperties {
    private boolean enabled = true;
    private String serviceName;
    private List<String> excludePaths = List.of("/actuator/**", "/health");
    // getters/setters
}

// AuditAutoConfiguration.java
@AutoConfiguration
@ConditionalOnProperty(prefix = "lending.audit", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public AuditFilter auditFilter(AuditProperties properties) {
        return new AuditFilter(properties);
    }
}
```

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.lending.starter.audit.AuditAutoConfiguration
com.lending.starter.error.ErrorAutoConfiguration
```

**Usage in any microservice:**
```xml
<dependency>
    <groupId>com.lending</groupId>
    <artifactId>lending-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

```yaml
lending:
  audit:
    enabled: true
    service-name: loan-service
```

---

## Quick Reference — Annotations Cheat Sheet

| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@RequestMapping` | Map HTTP requests to methods |
| `@PathVariable` | Extract value from URI path |
| `@RequestParam` | Extract query parameter |
| `@RequestBody` | Deserialize request body |
| `@ResponseStatus` | Set HTTP response status |
| `@Valid` | Trigger JSR-303 validation |
| `@Transactional` | Declarative transaction management |
| `@Async` | Execute method asynchronously |
| `@Scheduled` | Schedule method execution |
| `@Cacheable` | Cache method result |
| `@CacheEvict` | Remove entries from cache |
| `@ConditionalOnProperty` | Conditional bean creation |
| `@ConfigurationProperties` | Bind external config to POJO |
| `@Profile` | Activate bean for specific profile |
| `@PreAuthorize` | Method-level authorization |
| `@EnableWebSecurity` | Enable Spring Security |
| `@ControllerAdvice` | Global exception handling |

---

*Last updated: March 2026*
