# Architecture Review — Edueasy Java APIs

Review date: 2026-09-06  ·  Last verified: 2026-09-07 (via the `clean-architecture-ddd` cline skill)
Rule-sets applied: Clean Architecture (Robert C. Martin), Domain-Driven Design (Evans/Vernon), Clean Code (Martin), Refactoring (Fowler) — `ciembor/agent-rules-books` nano edition, loaded from `.cline/skills/clean-architecture-ddd/`.
Scope: 9 Java API services + `common-starter-parent` (12 nested starter modules), **~514 production Java files across the 9 APIs** (641 incl. common-starter), **~250 test files** (181 across the 9 APIs + 69 in common-starter).
Status: re-verified 2026-09-07. No Java changes landed since 2026-09-06 — the findings stand. Figures below are the corrected, **measured** values (see §8).

---

## 1. Executive summary

Every service is built as a **framework-first, technically-layered Spring Boot application**:
`controller → service → repository → (entity | dto)` with `config` and `transformer` side packages.
There is **no Clean Architecture** (no `domain` / `application` / `infrastructure` / `interface` layers,
no use-case classes, dependency rule not enforced) and **no DDD tactical model**
(no aggregates, value objects, domain services, factories, or invariant-owning entities).

Positive signals that should be preserved and built on:

- Each API is already its own **Bounded Context** (`za.co.edueasy.api.<context>`) — good strategic boundary.
- **Transformers** exist in most APIs (a translation layer between persistence/DTO — the seed of an anti-corruption layer).
- **Read-only foreign entities** (`entity/readonly`) used for cross-context reads — an explicit, if weak, Conformist boundary.
- **Strategy pattern** for payment gateways (`GatewayContext` / `GatewayPaymentHandler`) is genuinely good.
- `LlmPort` in chatbot-api is a real **port** — the only one found.
- `common-starter-*` is a legitimate **generic subdomain** (idempotency, ratelimiting, http-client, tracing).
- Test suites exist in all APIs; `course-api`/`dashboard-api` are thin.

The dominant smell is the **god service**: `PaymentService` (794 LOC, 10 collaborators, 5 repositories),
`AuthenticationService` (746 LOC), `PDFProcessorService` (573), `CommsMessageListener` (525),
`ChatbotService` (472), `PayfastService` (348), fat controllers (`PaymentController` 431, `AdminController` 270).

---

## 2. Scorecard (per API)

Scored 1–5 per pillar; 5 = textbook.

| API | Clean Arch | DDD | Clean Code | Refactoring fit | Test depth | Notes |
|---|---|---|---|---|---|---|
| payment-api | 1 | 1 | 2 | 1 | 2 | Biggest god service (794); 5 repos + 5 collaborators; servlet import in PayfastService/controller, not the service |
| auth-api | 1 | 1 | 2 | 1 | 3 | God service (746); Spring JWT security types leak into service |
| document-processor-api | 1 | 1 | 2 | 1 | 3 | PDFProcessorService 573 LOC; util/ package of 6 generator classes |
| communication-api | 1 | 1 | 2 | 1 | 3 | Fat listeners (525/379/223); provider switch; decent entity shape |
| chatbot-api | 1 | 1 | 2 | 2 | 3 | LlmPort exists; 11-dependency orchestrator ChatbotService |
| course-api | 1 | 1 | 2 | 2 | 1 | Thin; cross-api REST client (good); only 5 tests |
| application-api | 1 | 1 | 2 | 1 | 2 | Freshly split from dashboard; ApplicationService 227 / JourneyStepService 182, fat StepDataController 229 |
| broadcast-api | 1 | 1 | 2 | 3 | 3 | Smallest; cleanest layering but BroadcastService still god-ish |
| dashboard-api | 1 | 1 | 2 | 2 | 1 | Thin facade over application-api by design; only 4 test files |
| common-starter-parent | n/a | n/a | 2 | 3 | 3 | Generic subdomain; `util/` + `api/base` escape hatches |

Average Clean Architecture and DDD scores: **1/5** across the platform.

---

## 3. Findings by rule-set

### 3.1 Clean Architecture violations

1. **No layer packages.** No `domain`, `application`, `infrastructure`, `interface` in any API.
   Code is organized by technical bucket (controller/service/repository), which Clean Architecture
   explicitly rejects in favour of use-case / business-capability organization.
2. **Dependency rule broken everywhere.** Framework types cross into policy:
   - `AuthenticationService` imports Spring JWT (`spring-security-oauth2-jwt` — `Jwt`, `JwtException`).
   - `PayfastService` (payment-api) imports `jakarta.servlet.http.HttpServletRequest` (IP validation for
     gateway callbacks).
   - Services across every API import JPA entities and Spring Data repositories directly.
   Policy and details are fused; the framework (Spring), ORM (JPA), and web (Servlet) are the center of design.
3. **God services own business rules.** Business rules are documented *in javadoc* (`PaymentService` lists
   BR1–BR11) but implemented as imperative code inside the god service, not as domain policy objects.
   Per the trigger rules: "When `*Service` classes grow business rules, move policy inward and split by use case."
4. **Repositories leak.** Spring Data `JpaRepository` interfaces are injected straight into services —
   no `port`/`adapter` seam, no repository interface owned by the application layer.
5. **`util` escape hatches.** `documentprocessor/util` (PDFReportGenerator, ExcelGenerator, PDFTextExtractor,
   ZipArchiver, GraphDataFormatter, InMemoryMultipartFile) and `chatbot/util` (CommonPublisherUtil, JsonUtility)
   are unowned, cross-cutting dumping grounds — Clean Architecture's "shared utility escape hatch".
6. **Cross-context persistence coupling.** Payment reads `ApplicationEntity`, `StudentEntity`,
   `InstitutionEntity`; chatbot reads readonly `ApplicationEntity`/`StudentEntity`. Contexts share a
   physical schema; no contract test or versioned anti-corruption layer protects these reads.

### 3.2 DDD violations

1. **Anaemic entities.** `PaymentEntity` is `@Data @Builder` — a JPA data holder, not a domain object.
   No invariants, no behavior, no state transitions. `paymentStatus` is a `String` column mapped field;
   status transition rules (BR8) live in the service, not on the Payment object.
2. **No Aggregates / Value Objects / Factories / Domain Services.** Searched all APIs for
   `ValueObject|Factory|Aggregate|Policy|UseCase|Port` — only `LlmPort` exists. Status codes live in
   `dto/code/` as enums, not in the domain.
3. **Persistence model = domain model.** Entities are named `*Entity` (procedural, storage-first) and are
   passed around the whole stack. DDD says design for the model first, storage second.
4. **Business behavior hidden in orchestration.** Chatbot pipeline (intent → LLM → FAQ → DB → response →
   escalation) exists only as imperative flow in `ChatbotService` (472 LOC, 11 collaborators). Comm pattern
   (`CommsMessageListener` 525 LOC) procedural. Per DDD trigger rule: "Procedural business rules in
   orchestration... trigger moving behavior into the model."
5. **Ubiquitous Language drift.** Domain terms (`payment status`, `plan type`, `conversation state`) are
   encoded as `*Code` enums in `dto` while tables use `EPY_*` names and JPA entities use `Entity` suffix —
   three vocabularies for one concept.
6. **Context mapping implicit.** The dashboard→application split is real (services renamed, clients
   delegated) but there is no explicit context map (Conformist for DB-reads, REST client for calls).
   The `entity/readonly` pattern is a reasonable half-step; it needs tests + documentation.

### 3.3 Clean Code violations

1. **God classes / functions.** 794-line and 746-line services defeat local reasoning (rule: "Write for
   local reasoning").
2. **Command/query mixing.** Services both mutate and answer — one god object spans initiate/query/
   verify paths and a transaction concern (write-sets only visible at method level, e.g. `PaymentService`
   `@Transactional` at 142/165/321).
3. **Boolean-flag / mixed-abstraction risk** — present in the big orchestrators; e.g. `ChatbotService`
   interleaves regex matching, DB lookups, LLM prompting, and response shaping in one method chain.
4. **Names.** `PaymentEntity`, `StudentEntity` encode storage; `InMemoryMultipartFile`, `CommonPublisherUtil`
   are vague. Transformed DTOs are well-named (`PaymentInitiationRequest`, `PaymentFinancialSummaryDTO`).
5. **Comments.** Javadocs are extensive and useful (rationale + BR list). Some comments explain the flow
   (e.g. `ChatbotService` "Escape hatch...") — those should become named policy, not prose.

### 3.4 Refactoring opportunities (Fowler lens)

The best news: these are classic, well-understood refactorings with small safe first steps. No rewrite needed.

---

## 4. Prioritized backlog

### P0 — stop the bleeding (small, behavior-preserving, high value)

1. **Sever web/security types from service policy.** In `PayfastService`, move `HttpServletRequest` IP
   validation behind a `GatewayCallbackSource` value object (or lift it into the controller). In
   `AuthenticationService`, hide raw Spring JWT types behind an application-owned `SessionToken` port.
   Pure wins, low risk.
2. **Split status logic into domain policy.** Extract string `paymentStatus` + transition rules into a
   `PaymentStatus` enum (or value object) with `canTransitionTo(code)` / `transition(...)` methods.
   Back with tests. (Same for `ConversationStateCode`, `ApplicationStatusCode`.)
3. **Extract the money/plan policy.** `PlanCatalog` already exists — promote to a proper domain concept
   (value objects `PlanType`, `Money`) and move BR2/BR3 onto it.
4. **Scope `@Transactional` explicitly.** `PaymentService` already uses method-level transactions
   (lines 142/165/321) — keep that discipline while splitting the use cases, and audit the other god
   services for class-level write-set sprawl (Refactoring: "keep mutation and call contracts clear").
### P1 — decompose the god services (each step buildable + testable)

5. **`PaymentService` (794 LOC)** → split by use case, Fowler-style:
   - `InitiatePaymentUseCase` (BR1, BR2, BR3, BR5)
   - `ProcessItnUseCase` (BR6, BR7 — idempotency exists, perfect seam)
   - `VerifyPaymentUseCase` (BR11 — proof-of-payment)
   - `ExpirePendingPaymentsUseCase` (BR9 — the `@Scheduled` method: extract a scheduled trigger → use case)
   Each gets its own port-backed repository slice; keep `PaymentEntity` behind a repository seam.
6. **`AuthenticationService` (746 LOC)** → split by capability: `LoginUseCase`, `TokenRefreshUseCase`,
   `PasswordResetUseCase`, `MfaUseCase`, `SessionIntrospectionUseCase`. The existing `MfaService` is already
   a good precedent — extend the pattern.
7. **`ChatbotService` (472)** → extract the pipeline steps as named collaborators around a
   `Conversation` aggregate: `HandleMessageUseCase`, `IntentRouter`, `AnswerComposer`. `LlmPort` should
   become the seam for the LLM adapter.
8. **`PDFProcessorService` (573)** → extract `DocumentIngestion`, `ReportGeneration`, `ExportPackage`
   use cases; move `util/PDFReportGenerator` + `ExcelGenerator` behind a `ReportGenerator` port.
9. **`CommsMessageListener` (525)** → listener should be a *humble* adapter: parse envelope,
   delegate to `SendCommunicationUseCase`. Move template/format logic to a `CommunicationComposer`.
10. **Fat controllers** (`PaymentController` 431, dashboard/application `AdminController` ~270) → thin:
    map HTTP → use-case call → HTTP response. Move `@PreAuthorize`/scope logic to a policy object where it grows.

### P2 — introduce Clean Architecture + DDD skeleton incrementally

11. Adopt the package skeleton per service (keep old packages during migration):
    ```
    za.co.edueasy.api.<ctx>.domain      (entities/aggregates, value objects, domain services, events)
    za.co.edueasy.api.<ctx>.application (use cases, ports, application DTOs)
    za.co.edueasy.api.<ctx>.infrastructure (jpa, rest-clients, adapters)
    za.co.edueasy.api.<ctx>.interface   (controllers, presenters)
    ```
12. Introduce **port interfaces** for repositories (`PaymentRepository` → `PaymentStore` port + JPA adapter).
    Spring Data impls stay; only the boundary moves. Enforce the dependency rule with ArchUnit `@ArchTest`
    (one test file per API bans `domain` → `infrastructure` imports).
13. Promote `entity/readonly` contracts to an **explicit anti-corruption layer**: read models + mappers +
    contract tests that break if the owning context changes a column name.
14. Domain tests in Ubiquitous Language: `payment-enters-PENDING-then-EXPIRED`, `student-cannot-apply-twice`,
    `conversation-escalates-after-N-failures`. The 225 existing tests are the safety net for the moves above.
---

## 5. Target architecture (one page)

Per bounded context (each API):

```
interface  (REST controllers, webhooks, listeners — humble, no rules)
   │  use-case calls + application DTOs (inward)
application (use cases = one class per business capability; ports; policies)
   │  domain events, port interfaces (inward)
domain     (entities, aggregates, value objects, domain services — framework-free)
   │
infrastructure (JPA adapters, gateway clients, LLM clients, message producers)
```

Cross-context communication only through:
- REST clients (`DashboardApsClient`, `ApplicationAdminClient` — already exist, keep) or
- domain events / outbox (`common-starter-outbox` — already exists, use it) or
- read-model projections with contract tests (`entity/readonly` — formalize).

Shared `za.co.common` stays as the generic subdomain (exceptions, idempotency, ratelimit, http-client,
tracing/metrics) — it is *infrastructure-only*; domain code never imports it.

---

## 6. Proof points (representative evidence)

- `PaymentService.java` — 794 LOC; 5 repositories + 5 more collaborators (10 total); javadoc lists
  BR1–BR11 implemented as procedural code; `@Transactional` is **method-level** (lines 142/165/321).
  No servlet import — the earlier attribution was wrong (see §8).
- `AuthenticationService.java` — 746 LOC; imports Spring JWT types, entities, repos, DTOs; no servlet import.
- `ChatbotService.java` — 472 LOC; 11 collaborators; pipeline orchestration in one class.
- `PaymentEntity.java` — `@Data @Builder @Entity @Table(name="EPY_PAYMENT")`; `paymentStatus` is a raw `String`.
- `PaymentController.java` — 431 LOC; `@PreAuthorize` + scope logic inside the controller.
- `PayfastService.java` — 348 LOC; imports `jakarta.servlet.http.HttpServletRequest` (`validateSourceIp`,
  `resolveClientIp`) — the real service-layer servlet leak.
- `CommsMessageListener.java` — 525 LOC; provider/template logic in a listener.
- `dashboard-api` — correct direction: thin facade delegating learner analytics to `application-api`
  via `ApplicationAdminClient`; mirrors the split done in the `338f595` commit family.

## 7. Suggested immediate first move

Pick **payment-api P0 (items 1–4)** as the pilot: small, fully test-covered, demonstrates the
pattern for every other service. Then apply the same P0 pass across auth + chatbot + document-processor.

---

## 8. Verification log (2026-09-07)

Re-ran the review against the current tree (commit `5dc896a` family, no Java changes since 2026-09-06)
using the `clean-architecture-ddd` skill. All figures in this doc were re-measured. Corrections to the
original pass:

| Claim (original) | Measured (2026-09-07) | Verdict |
|---|---|---|
| `PaymentService` 787 LOC | 794 LOC | ✓ god service (corrected figure) |
| `PaymentService` imports `HttpServletRequest` | **no servlet import** — servlet lives in `PayfastService` + `PaymentController` | ✗ corrected |
| `PaymentService` 6 repositories | **5** repositories of 10 collaborators | ✗ corrected |
| `PaymentService` class-level `@Transactional` | method-level (lines 142/165/321) | ✗ corrected |
| `AuthenticationService` 745 LOC / imports `HttpServletRequest` | 746 LOC; imports Spring JWT types, **no servlet** | ~ corrected (LOC), ✗ servlet attribution |
| `PayfastService` 325 LOC | 348 LOC | ✓ (corrected figure) |
| `PaymentController` 422 LOC | 431 LOC | ✓ (corrected figure) |
| `AdminController` ~270 LOC | 270 LOC (dashboard-api) | ✓ |
| `SponsorService` 410 LOC | **132 LOC** — the 410 was pre-split; current fat services: `ApplicationService` 227, `JourneyStepService` 182, `StepDataController` 229 | ✗ corrected |
| ~515 production Java files (9 APIs) | 514 | ✓ |
| ~225 test files | 250 total (181 across 9 APIs + 69 common-starter); `dashboard-api` has **4** test files, `course-api` 5 | ~ corrected |
| ArchUnit / domain-package checks | still absent; no `domain`/`application` packages in any API | ✓ finding stands |

Net conclusion unchanged: framework-first, no domain layer, avg Clean Arch / DDD = 1/5 per API. The
specific evidence pointers for the servlet-leak and transaction claims moved; the architecture verdict
does not. Narrative review kept in `docs/code-review/clean-architecture-ddd-review.md` (Grade D+);
frontend review in `docs/code-review/frontend-architecture-review.md`.