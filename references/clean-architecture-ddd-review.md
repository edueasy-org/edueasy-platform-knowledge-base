# Harsh Code Review: Clean Architecture / DDD / Clean Code / Refactoring

> Reviewed: 2026-09-06
> Scope: All Java microservices in `edueasy-org` monorepo (~627 production Java files)
> Standards: Clean Architecture (Robert C. Martin), DDD (Eric Evans), Clean Code (Robert C. Martin), Refactoring (Martin Fowler)

---

## Executive Summary

**Grade: D+**

The codebase works. It does not embody Clean Architecture, DDD, or Clean Code. It is a textbook example of **database-shaped, framework-first, anemic domain model** organized by technical layers with god services, primitive obsession, and zero domain encapsulation. Every service repeats the same anti-patterns. The few bright spots (gateway strategy pattern in payment-api, provider strategy in communication-api) prove the team *can* do better — but these are exceptions, not the rule.

---

## 1. Clean Architecture Violations

### 1.1 No Domain Layer Exists

**Rule violated:** *"Domain and use cases must not import frameworks, databases, web, UI, queues, service clients, device, vendor, or infrastructure details."*

**Reality:** There is no domain layer. The package structure is purely technical:

```
src/main/java/za/co/edueasy/api/authentication/
├── controller/    ← REST
├── service/       ← "business logic" (all of it)
├── repository/    ← Spring Data JPA
├── entity/        ← JPA entities (pure data)
├── dto/           ← data transfer
├── transformer/   ← field mapping
└── config/        ← Spring config
```

No `domain/` package. No `usecase/` package. No `port/` or `adapter/` separation. The "domain" is scattered across services that import everything: JPA, Spring Security, RabbitMQ, RestClient, MeterRegistry, etc.

**Evidence:**
- `AuthenticationService` imports: `PasswordEncoder`, `Jwt`, `JwtException`, `RabbitTemplate` (via event service), `AuthenticationTransformer`, Spring Data repositories
- `PaymentService` imports: `PaymentEntity`, `PaymentRepository`, `RabbitTemplate` (via event service), `PlanCatalog`, `GatewayPaymentHandler`

### 1.2 Entities Are JPA Data Bags, Not Domain Objects

**Rule violated:** *"Entities guard enterprise invariants."*

**Reality:** Entities are `@Data @Builder @NoArgsConstructor @AllArgsConstructor` JPA mappings with zero behavior. Lombok generates getters, setters, `equals`, `hashCode`, `toString` — exposing every field for mutation from anywhere.

**Evidence — `AuthUserEntity`:**
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuthUserEntity implements Serializable {
    @Id
    private Long id;
    private String email;
    private String passwordHash;
    private String userTypeCode;
    private String accountStatusCode;
    private Integer failedLoginCount;
    private LocalDateTime lockedUntil;
    // ... 15+ more fields, all publicly settable
}
```

Any code can do `user.setAccountStatusCode("LOCKED")` without going through any invariant check. The entity does not protect its own state.
## 2.6 No Bounded Context Boundaries

**Rule violated:** *"Define context boundaries and relationships explicitly before sharing terms, data, or behavior across systems."*

**Reality:** Multiple services share concepts without explicit boundaries:
- `Student` exists in chatbot-api, payment-api, course-api, application-api
- `Application` exists in chatbot-api, payment-api, course-api, application-api
- `Institution` exists in chatbot-api, payment-api, course-api, application-api

Each service has its own entity, repository, and DTO for the same concept. No anti-corruption layer. No shared kernel. No context map.

### 1.3 Services Are God Classes With All Business Rules

**Rule violated:** *"When controllers, jobs, handlers, gateways, repositories, SQL, presenters, service listeners, or `*Service` classes grow business rules, move policy inward and split by use case."*

**Reality:** Services contain ALL business logic, validation, persistence orchestration, event publishing, and sometimes even HTTP concerns.

**Evidence — `AuthenticationService` (746 lines):**
- `login()` — ~100 lines of credential validation, account lockout, token generation, audit logging
- `register()` — ~80 lines of validation, password hashing, user creation, event publishing
- `changePassword()` — ~50 lines
- `validateToken()`, `refreshToken()`, `revokeToken()`, `rotateKeys()`, `getUserSessions()`, `validateScopes()`
- All in ONE class, directly depending on repositories, JWT service, transformer, event service

**Evidence — `PaymentService` (794 lines):**
- `initiatePayment()` — ~150 lines
- `processItn()` — ~100 lines
- `verifyPayment()`, `getPaymentStatus()`, `getPaymentInstitutions()`, `expirePendingPayments()`
- Plus private helpers: `generateMerchantPaymentId()`, `getPlanPrice()`, `getPlanLimit()`, `mapPayfastStatus()`, `parseBigDecimal()`

### 1.4 No Use Case Layer

**Rule violated:** *"Organize by use case, feature, or business capability."*

**Reality:** No use case classes exist. The service layer IS the use case layer, but it's not organized by use case — it's organized by entity/aggregate root. `AuthenticationService` handles login, registration, password change, token management, MFA, sessions — all because they relate to "authentication" as a technical concept, not as distinct use cases.

---

## 4. Refactoring Opportunities

### 4.1 Extract Domain Model from Persistence

**Current:** JPA entity = domain object = persistence model
**Target:** Separate domain model from persistence mapping

```
domain/
├── model/
│   ├── User.java           ← domain entity with behavior
│   ├── Email.java          ← value object
│   ├── Password.java       ← value object
│   └── AccountStatus.java  ← value object/enum
persistence/
├── UserEntity.java         ← JPA mapping
└── UserRepositoryImpl.java ← implements domain UserRepository
```

### 4.2 Extract Use Cases

**Current:** `AuthenticationService` (746 lines, 7+ responsibilities)
**Target:** One class per use case

```
usecase/
├── LoginUseCase.java
├── RegisterUseCase.java
├── ChangePasswordUseCase.java
├── RefreshTokenUseCase.java
└── RevokeTokenUseCase.java
```

Each use case: input port → orchestration → output port. Max 50 lines.

### 4.3 Extract Value Objects

**Current:** `String email`, `String passwordHash`, `String accountStatusCode`
**Target:**

```java
public record Email(String value) {
    public Email {
        if (value == null || !value.matches("^[\w.-]+@[\w.-]+\.[a-zA-Z]{2,}$")) {
            throw new IllegalArgumentException("Invalid email: " + value);
        }
    }
}

public record Password(String hash) {
    public boolean matches(Password raw, PasswordEncoder encoder) {
        return encoder.matches(raw.hash, this.hash);
    }
}
```

### 4.4 Break Up God Services

**Priority targets:**
1. `AuthenticationService` → 5+ classes
2. `PaymentService` → 4+ classes
3. `StudentStatusService` → 3+ classes (already borderline)

### 4.5 Introduce Ports and Adapters

**Current:** `Service → JpaRepository`
**Target:**

```java
// Domain layer owns the port
public interface UserRepository {
    User save(User user);
    Optional<User> findById(UserId id);
    Optional<User> findByEmail(Email email);
}

// Infrastructure layer provides the adapter
@Repository
public class JpaUserRepository implements UserRepository {
    private final SpringDataUserRepository springDataRepo;
    private final UserMapper mapper;
    // ...
}
```

### 4.6 Replace String Codes with Enums/Value Objects

**Current:** `String accountStatusCode = "LOCKED"`
**Target:**

```java
public enum AccountStatus {
    ACTIVE, LOCKED, PASSWORD_EXPIRED, DISABLED;

    public boolean canLogin() {
        return this == ACTIVE;
    }

    public boolean isLocked() {
        return this == LOCKED;
    }
}
```

### 4.7 Extract Domain Events

**Current:** `eventService.publishPaymentUpdatedEvent(dto)`
**Target:**

```java
public class Payment {
    private final List<DomainEvent> events = new ArrayList<>();

    public void complete() {
        if (this.status == PaymentStatus.COMPLETE) {
            throw new IllegalStateException("Payment already complete");
        }
        this.status = PaymentStatus.COMPLETE;
        this.completedAt = LocalDateTime.now();
        events.add(new PaymentCompletedEvent(this.id, this.amount, this.completedAt));
    }

    public List<DomainEvent> pullEvents() {
        List<DomainEvent> pulled = new ArrayList<>(events);
        events.clear();
        return pulled;
    }
}
```

---

## 5. Bright Spots (Do More of This)

### 5.1 Gateway Strategy Pattern (payment-api)

```java
public interface GatewayPaymentHandler {
    GatewayCode gateway();
    PaymentInitiationResponse buildInitiation(GatewayContext context);
}
```

This is proper Clean Architecture. The domain defines the interface; infrastructure provides implementations (`PayFastGatewayHandler`, `StripeGatewayHandler`, etc.). **Apply this pattern everywhere.**

### 5.2 Provider Strategy Pattern (communication-api)

```java
public interface WhatsAppProvider {
    WhatsAppProviderCode code();
    boolean isConfigured();
    String send(WhatsAppDTO dto);
    boolean verifyWebhook(HttpServletRequest request, String rawBody);
}
```

Same good pattern. The `WhatsAppProviderSelector` handles fallback logic cleanly.

### 5.3 Value Object Attempt (communication-api)

```java
public record WhatsAppSendResult(WhatsAppProviderCode provider, String messageId) {}
```

Using Java records for immutable value objects. More of this.

### 5.4 Code Objects (Multiple APIs)

```java
public abstract class Code {
    private final String externalCode;
    private final String description;
    // ...
}
```

The `Code` hierarchy (`BroadcastStatusCode`, `PaymentStatusCode`, `WhatsAppProviderCode`) is a good start at type safety. But they're still just string wrappers without behavior.

---

## 6. Priority Refactoring Plan

### Phase 1: Foundation (Week 1-2)
1. Extract value objects: `Email`, `Password`, `UserId`, `Money`, `PhoneNumber`
2. Replace string status codes with enums: `AccountStatus`, `PaymentStatus`, `ApplicationStatus`
3. Add domain-owned repository interfaces (ports)

### Phase 2: Use Case Extraction (Week 3-4)
1. Split `AuthenticationService` into individual use cases
2. Split `PaymentService` into individual use cases
3. Create input/output port interfaces for each use case

### Phase 3: Domain Model (Week 5-6)
1. Extract domain entities from JPA entities
2. Move business logic from services into domain entities
3. Introduce domain events

### Phase 4: Adapter Separation (Week 7-8)
1. Move repository implementations to infrastructure layer
2. Move event publishing to infrastructure layer
3. Move external API clients to infrastructure layer

### Phase 5: Package Reorganization (Week 9-10)
1. Reorganize packages by feature, not by technical layer
2. Enforce dependency rules with ArchUnit tests
3. Document bounded contexts and context map

---

## 7. Metrics

| Metric | Current | Target |
|---|---|---|
| Avg service class length | ~300 lines | ~80 lines |
| Max service class length | 794 lines | ~150 lines |
| Dependencies per service | 5-11 | 2-4 |
| Domain behavior in entities | 0% | 60%+ |
| Value objects | 3 (records) | 20+ |
| Use case classes | 0 | 30+ |
| Domain events | 0 | 15+ |
| Port interfaces | 0 | 15+ |
| Adapter implementations | 0 | 15+ |

---

## 8. Conclusion

The codebase is a **working prototype that was never refactored into a proper architecture**. It delivers value, but at a high cost:

- **Change risk:** Modifying `AuthenticationService.login()` risks breaking registration, password reset, and token management
- **Test difficulty:** God services require 10+ mocks for unit tests
- **Onboarding cost:** New developers must read 700+ line files to understand a single business operation
- **Duplication:** The same patterns (CRUD + validation + event publish) are copied across every service

The team has demonstrated they CAN write good code (gateway strategy, provider strategy). The problem is systemic, not skill-based. Apply the patterns from payment-api and communication-api consistently across all services, and extract domain logic from services into a proper domain model.

**The code works. It is not clean. It is not maintainable at scale. Refactor now before the next feature.**
---

## Update 2026-09-07 — Re-verification (skill: clean-architecture-ddd)

Re-ran the review pass against the current tree after the review was restored (commit `5dc896a`).
Rule-set: same four books via the `clean-architecture-ddd` compound skill
(`.cline/skills/clean-architecture-ddd/`), loaded at review time.

### Scope confirmed — all 10 Maven modules covered

Verified the module inventory: `common-starter-parent` (12 starter modules), `document-processor-api`,
`edueasy-application-api`, `edueasy-auth-api`, `edueasy-broadcast-api`, `edueasy-chatbot-api`,
`edueasy-communication-api`, `edueasy-course-api`, `edueasy-dashboard-api`, `edueasy-payment-api`.
641 production `src/main/java` files across those Maven modules (9 APIs = 514 + common-starter = 127;
a further 14 stray files sit in the unclaimed repo-root `src/main/java`).
No Java changes committed since the original 2026-09-06 pass — findings hold.

### Evidence re-verified (line counts, current tree)

| Class | Original claim | Re-verified | Verdict |
|---|---|---|---|
| `PaymentService` | 787 LOC | 794 LOC | ✓ god service confirmed |
| `AuthenticationService` | 745 LOC | 746 LOC | ✓ god service confirmed |
| `PDFProcessorService` | 573 LOC | 573 LOC | ✓ |
| `CommsMessageListener` | 525 LOC | 525 LOC | ✓ |
| `ChatbotService` | 472 LOC | 472 LOC | ✓ |
| `PaymentEntity` `@Data @Builder`, raw `String paymentStatus` | — | lines 24/25/50 | ✓ anemic entity confirmed |
| `AuthenticationService` imports Spring JWT (`spring-security-oauth2-jwt.Jwt`) | — | imports 13-14 | ✓ framework types leak into service |
| `PaymentService` injects 6 repositories | **5** repositories of 10 collaborators (plus `PayfastService`, transformer, event service, `PlanCatalog`, gateway handlers) | ✗ corrected |
| `PaymentService` uses class-level `@Transactional` | **method-level** (`@Transactional` at lines 142/165, `readOnly` at 321) | ✗ corrected |

### One correction

The original pass attributed `jakarta.servlet.http.HttpServletRequest` to `PaymentService`. Current
tree shows the servlet import lives in **`PaymentController` (line 11, used lines 381/388) and
`PayfastService`** — not in `PaymentService`. The Clean Architecture verdict is unchanged (a web type
on the interface layer controller is *less* egregious than in a service, and `PayfastService` is still
a service-layer servlet leak), but the evidence pointer moves. Corrected here so the next reader
searches the right files.

### Bottom line unchanged

Average Clean Architecture / DDD score remains 1/5 per API. The gateway strategy (payment-api) and
provider strategy (communication-api) remain the two patterns worth propagating. No P0 was fixed in the
intervening commits; the throttle is structural (no `domain`/`application` skeleton, god services,
anemic entities). Next concrete step is still the P0 pilot on payment-api.
