# 🔍 Code Review Cheat Sheet — Java 21 + Spring Boot Microservices

> Critical & major issues to catch in every pull request.

**Severity Guide:**
- 🚨 **CRITICAL** — Must fix before merge
- ⚠️ **HIGH** — Fix in this PR or next
- ℹ️ **MEDIUM** — Address soon

---

## Table of Contents

1. [☕ Java 21](#-java-21)
2. [🌱 Spring Boot](#-spring-boot)
3. [🔗 Microservices](#-microservices)
4. [🔐 Security](#-security)
5. [⚡ Performance](#-performance)
6. [🧪 Testing](#-testing)
7. [📊 Observability](#-observability)

---

## ☕ Java 21

### 🚨 CRITICAL — Use Records for DTOs

Prefer Java 21 records over mutable POJOs for data transfer objects. Records are immutable, concise, and auto-generate `equals`/`hashCode`/`toString`.

```java
// ✅ GOOD
public record UserDTO(Long id, String name, String email) {}

// ❌ BAD
public class UserDTO {
    private Long id;
    private String name;
    // boilerplate getters/setters...
}
```

---

### 🚨 CRITICAL — Pattern Matching & Sealed Classes

Use sealed classes with pattern matching in switch expressions for exhaustive type hierarchies. Missing branches = bugs.

```java
// ✅ GOOD — exhaustive, compiler-checked
sealed interface Shape permits Circle, Rectangle {}

public double area(Shape s) {
    return switch (s) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
    };
}
```

---

### ⚠️ HIGH — Virtual Threads (Project Loom)

Use virtual threads for I/O-bound tasks. Avoid pinning: never hold locks or `synchronized` blocks inside virtual threads across I/O calls.

```java
// ✅ GOOD — virtual thread executor
var executor = Executors.newVirtualThreadPerTaskExecutor();

// ❌ BAD — pins carrier thread
synchronized (lock) {
    someBlockingIOCall(); // PINS carrier thread!
}
```

---

### ⚠️ HIGH — Prefer Text Blocks for SQL/JSON

Use Java 15+ text blocks for multiline strings. Never concatenate SQL strings (SQL injection risk).

```java
// ✅ GOOD
String query = """
    SELECT u.id, u.name
    FROM users u
    WHERE u.active = true
    """;

// ❌ BAD
String query = "SELECT u.id " +
               "FROM users " + ...; // messy and error-prone
```

---

### ℹ️ MEDIUM — Use `var` Judiciously

Use `var` for local variables where the type is obvious from context. Avoid `var` when it obscures the type (e.g., method return values).

```java
// ✅ GOOD — type obvious
var users = new ArrayList<User>();
var response = userService.findAll();

// ❌ BAD — type unclear
var x = process(); // What does process() return?
```

---

### ℹ️ MEDIUM — Avoid Raw Types & Unchecked Casts

Always use generics. Suppress warnings only with justification comments. Unchecked casts are a `ClassCastException` waiting to happen.

```java
// ✅ GOOD
List<User> users = repository.findAll();

// ❌ BAD
List users = repository.findAll(); // raw type
@SuppressWarnings("unchecked")     // without justification
```

---

## 🌱 Spring Boot

### 🚨 CRITICAL — Never Use Field Injection

Always use constructor injection. Field injection hides dependencies, breaks immutability, and makes testing painful.

```java
// ✅ GOOD — constructor injection
@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}

// ❌ BAD — field injection
@Autowired
private UserRepository repo;
```

---

### 🚨 CRITICAL — Validate All Incoming Requests

Use `@Valid`/`@Validated` with Bean Validation annotations on every controller input. Never trust incoming data.

```java
// ✅ GOOD
@PostMapping("/users")
public ResponseEntity<UserDTO> create(
        @Valid @RequestBody CreateUserRequest req) { ... }

public record CreateUserRequest(
    @NotBlank String name,
    @Email    String email,
    @Min(18)  int age) {}
```

---

### 🚨 CRITICAL — Proper Exception Handling

Use `@ControllerAdvice` with `@ExceptionHandler`. Never let raw exceptions propagate to the client — they leak stack traces and implementation details.

```java
// ✅ GOOD
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

---

### ⚠️ HIGH — Use Spring Problem Details (RFC 7807)

Spring Boot 3.x supports RFC 7807 Problem Details natively. Enable it and use `ProblemDetail` for all error responses.

```yaml
# application.yml
spring:
  mvc:
    problemdetails:
      enabled: true

# Returns standardized JSON:
# { type, title, status, detail, instance }
```

---

### ⚠️ HIGH — Avoid N+1 Queries

Use `@EntityGraph` or `JOIN FETCH` for associated entities. Never lazy-load in a loop. Use projections for read-only queries.

```java
// ✅ GOOD — fetch join
@Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.id = :id")
Optional<User> findWithRoles(@Param("id") Long id);

// ❌ BAD — N+1
users.forEach(u -> u.getRoles().size()); // each call triggers a query
```

---

### ⚠️ HIGH — Transaction Boundaries

`@Transactional` on the service layer only — never on controllers or repositories. Set `readOnly = true` for read operations. Be aware of self-invocation proxy bypass.

```java
// ✅ GOOD
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public List<Order> findAll() { ... }

    @Transactional
    public Order create(CreateOrderRequest req) { ... }
}
```

---

### ⚠️ HIGH — Externalize Configuration

Use `@ConfigurationProperties` beans instead of `@Value` for grouped settings. Never hardcode URLs, credentials, or timeouts.

```java
// ✅ GOOD
@ConfigurationProperties(prefix = "payment")
public record PaymentConfig(
    String   apiUrl,
    Duration timeout,
    int      maxRetries) {}

// ❌ BAD
@Value("${payment.api.url}")
private String url; // scattered @Value annotations
```

---

### ℹ️ MEDIUM — Use Actuator Wisely

Expose only required actuator endpoints. Never expose `/actuator/env` or `/actuator/heapdump` in production without auth.

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized
```

---

## 🔗 Microservices

### 🚨 CRITICAL — Implement Circuit Breakers

Use Resilience4j for circuit breakers on all external service calls. A failing downstream service must not cascade failures.

```java
// ✅ GOOD
@CircuitBreaker(name = "userService", fallbackMethod = "fallback")
@Retry(name = "userService")
public UserDTO getUser(Long id) {
    return userClient.findById(id);
}

private UserDTO fallback(Long id, Exception ex) {
    return UserDTO.empty(id);
}
```

---

### 🚨 CRITICAL — Distributed Tracing

Propagate trace IDs across all services. Use Micrometer Tracing with Zipkin/Jaeger. Every log line must include `traceId` and `spanId`.

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0
logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

---

### 🚨 CRITICAL — Idempotent API Design

All POST/PUT/DELETE endpoints should be idempotent. Use idempotency keys for payment/order operations. Retries must be safe.

```java
// ✅ GOOD — idempotency key
@PostMapping("/payments")
public ResponseEntity<PaymentDTO> pay(
        @RequestHeader("Idempotency-Key") String key,
        @Valid @RequestBody PaymentRequest req) {
    return paymentService.processIdempotent(key, req);
}
```

---

### ⚠️ HIGH — Async Communication with Events

Prefer event-driven communication for non-critical cross-service calls. Avoid synchronous chains of 3+ services — use Kafka/RabbitMQ.

```java
// ✅ GOOD — publish event, let other services react
@Transactional
public Order createOrder(CreateOrderRequest req) {
    Order order = orderRepo.save(new Order(req));
    eventPublisher.publish(new OrderCreatedEvent(order));
    return order; // inventory service reacts asynchronously
}
```

---

### ⚠️ HIGH — API Versioning

Version your APIs from day one. Use URI versioning (`/v1/`) or header versioning. Never break existing consumers.

```java
// ✅ GOOD — URI versioning
@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller { ... }

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller { ... }
```

---

### ⚠️ HIGH — Health Checks & Readiness Probes

Implement `/actuator/health` with liveness and readiness groups. Readiness must fail if downstream dependencies are down.

```yaml
# application.yml
management:
  endpoint:
    health:
      group:
        readiness:
          include: db,redis,kafka
        liveness:
          include: ping
```

---

### ℹ️ MEDIUM — Service Discovery & Load Balancing

Use Spring Cloud LoadBalancer (not Ribbon). Register with Consul/Eureka. Never hardcode service URLs.

```java
// ✅ GOOD
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
// Uses: http://user-service/api/v1/users

// ❌ BAD
// http://192.168.1.10:8081/api/v1/users
```

---

## 🔐 Security

### 🚨 CRITICAL — Never Log Sensitive Data

Passwords, tokens, PII, and card numbers must never appear in logs. Use `@JsonIgnore`, masking utilities, and log scrubbers.

```java
// ✅ GOOD
log.info("User login attempt: userId={}", userId);

// ❌ BAD
log.info("Login: user={}, password={}", user, password);
log.debug("Token: {}", jwtToken); // token in logs!
```

---

### 🚨 CRITICAL — Use Parameterized Queries Only

Never build SQL/JPQL with string concatenation. Always use named parameters, `@Query` with `:param`, or the Criteria API.

```java
// ✅ GOOD
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// ❌ CRITICAL BUG — SQL Injection
String q = "SELECT * FROM users WHERE email = '" + email + "'";
entityManager.createNativeQuery(q);
```

---

### 🚨 CRITICAL — Secure JWT Configuration

Use strong signing keys (256-bit minimum). Validate issuer, audience, and expiry. Refresh tokens must be rotated on use.

```java
// ✅ GOOD
return Jwts.builder()
    .subject(userId)
    .issuer("my-service")
    .audience().add("my-clients").and()
    .expiration(Date.from(Instant.now().plus(15, MINUTES)))
    .signWith(secretKey) // HS256, min 256-bit key
    .compact();
```

---

### ⚠️ HIGH — Method-Level Security

Use `@PreAuthorize`/`@PostAuthorize` on service methods. Don't rely solely on URL-based security — defence in depth is required.

```java
// ✅ GOOD
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
public UserDTO getUser(Long userId) { ... }

@PostAuthorize("returnObject.ownerId == authentication.name")
public Document getDocument(Long docId) { ... }
```

---

### ⚠️ HIGH — CORS Configuration

Never use `allowedOrigins("*")` in production. Explicitly list allowed origins, methods, and headers.

```java
// ✅ GOOD
@Bean
public CorsConfigurationSource corsConfig() {
    var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.mycompany.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    // ❌ BAD: config.addAllowedOrigin("*");
}
```

---

### ⚠️ HIGH — Secrets Management

Use Vault, AWS Secrets Manager, or Kubernetes Secrets. Never store secrets in `application.yml`, environment variables in code, or Git.

```yaml
# ✅ GOOD — Spring Cloud Vault
spring:
  cloud:
    vault:
      uri: https://vault.internal
      authentication: kubernetes

# ❌ BAD — hardcoded
spring:
  datasource:
    password: myS3cretP@ss
```

---

## ⚡ Performance

### 🚨 CRITICAL — Paginate All Collection Endpoints

Never return unbounded collections. Always use `Pageable` with a maximum page size. Set a default if not provided.

```java
// ✅ GOOD
@GetMapping("/users")
public Page<UserDTO> list(
        @PageableDefault(size = 20, max = 100) Pageable pageable) {
    return userRepo.findAll(pageable).map(mapper::toDTO);
}

// ❌ BAD
public List<UserDTO> list() {
    return userRepo.findAll(); // could return millions of records!
}
```

---

### ⚠️ HIGH — Connection Pool Sizing

Configure HikariCP pool size properly: `pool = Tn × (Cm - 1) + 1`. Too many connections = worse performance. Monitor pool metrics.

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

---

### ⚠️ HIGH — Use Projections for Read Queries

Don't load full entities for read-only/list operations. Use interface projections or DTO projections to select only needed columns.

```java
// ✅ GOOD — projection
public interface UserSummary {
    Long getId();
    String getName();
}
List<UserSummary> findAllProjectedBy(); // SELECT id, name only

// ❌ BAD
List<User> findAll(); // loads all columns + lazy relations
```

---

### ⚠️ HIGH — Cache Strategically

Use `@Cacheable` for expensive, rarely-changing reads. Always define TTL and eviction. Cache at the service layer, not the repository.

```java
// ✅ GOOD
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public UserDTO findById(Long id) { ... }

@CacheEvict(value = "users", key = "#user.id")
public UserDTO update(User user) { ... }
```

```yaml
# application.yml — always set TTL!
spring:
  cache:
    redis:
      time-to-live: 300s
```

---

### ℹ️ MEDIUM — Async Methods for Non-Critical Work

Use `@Async` for fire-and-forget operations (emails, notifications, audit logs). Always use a custom thread pool, never the default.

```java
// ✅ GOOD
@Async("notificationExecutor")
public CompletableFuture<Void> sendWelcomeEmail(User user) { ... }

@Bean
public Executor notificationExecutor() {
    var exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(5);
    exec.setMaxPoolSize(20);
    exec.setQueueCapacity(100);
    return exec;
}
```

---

## 🧪 Testing

### 🚨 CRITICAL — Use `@SpringBootTest` Selectively

Full `@SpringBootTest` is slow. Use slices: `@WebMvcTest` for controllers, `@DataJpaTest` for repositories, `@JsonTest` for serialization.

```java
// ✅ GOOD — fast slice test
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockBean UserService userService;
    // tests only the web layer
}

// Use @SpringBootTest only for true integration tests
```

---

### ⚠️ HIGH — Test Edge Cases & Error Paths

Don't just test happy paths. Cover null inputs, empty collections, boundary values, concurrent access, and all exception paths.

```java
// ✅ GOOD — covers error path
@Test
void shouldReturn404WhenUserNotFound() {
    when(service.findById(99L))
        .thenThrow(new ResourceNotFoundException("User not found"));

    mockMvc.perform(get("/api/v1/users/99"))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.detail").value("User not found"));
}
```

---

### ⚠️ HIGH — Use Testcontainers for DB Tests

Never use H2 for production DB tests. Use Testcontainers with the real database (PostgreSQL, MySQL). H2 dialect differences hide real bugs.

```java
// ✅ GOOD
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> pg =
        new PostgreSQLContainer<>("postgres:16");
}
```

---

### ℹ️ MEDIUM — Assert Response Structure, Not Just Status

Validate response body, headers, content type, and pagination metadata — not just HTTP status codes.

```java
// ✅ GOOD
mockMvc.perform(get("/api/v1/users"))
    .andExpect(status().isOk())
    .andExpect(content().contentType(APPLICATION_JSON))
    .andExpect(jsonPath("$.content").isArray())
    .andExpect(jsonPath("$.totalElements").isNumber())
    .andExpect(jsonPath("$.content[0].id").exists());
```

---

## 📊 Observability

### ⚠️ HIGH — Structured Logging

Use structured JSON logging in production. Include `requestId`, `userId`, and service name in every log. Use MDC for context propagation.

```java
// ✅ GOOD — MDC context
MDC.put("userId", String.valueOf(userId));
MDC.put("requestId", requestId);
log.info("Processing order"); // context included automatically
```

```xml
<!-- logback-spring.xml — use JSON encoder in prod -->
<encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
```

---

### ⚠️ HIGH — Custom Metrics with Micrometer

Instrument business-critical operations with custom metrics. Track order counts, payment failures, cache hit rates — not just JVM metrics.

```java
// ✅ GOOD
@Service
public class OrderService {
    private final Counter orderCounter;
    private final Timer   orderTimer;

    public Order create(CreateOrderRequest req) {
        return orderTimer.record(() -> {
            Order o = processOrder(req);
            orderCounter.increment();
            return o;
        });
    }
}
```

---

### ℹ️ MEDIUM — Graceful Shutdown

Configure graceful shutdown to drain in-flight requests. Set the timeout to the maximum expected request duration.

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```yaml
# Kubernetes preStop hook
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]
```

---

*Java 21 · Spring Boot 3.x · Microservices · Resilience4j · Micrometer · Testcontainers*
