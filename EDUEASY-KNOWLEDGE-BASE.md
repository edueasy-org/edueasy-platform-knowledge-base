# EduEasy Knowledge Base

> **KB Version: 1.1.0** · Status: AUTHORITATIVE · Last updated: 2026-09-07
> This document is the **single source of truth** for what EduEasy is, what it intends to do, and how it works.
> It lives in its own repository — `edueasy-platform-knowledge-base` — not inside any service repo.
> Every refactoring or feature that changes architecture, journeys, topology, or policy **must** update this file and bump the version (see §9 Governance and `CHANGELOG.md`).

---

## Contents

1. [What EduEasy is](#1-what-edueasy-is)
2. [What EduEasy intends to do](#2-what-edueasy-intends-to-do)
3. [Platform topology — the how](#3-platform-topology--the-how)
4. [How the APIs fit together](#4-how-the-apis-fit-together)
5. [User journeys](#5-user-journeys)
6. [Architecture law — target and enforcement](#6-architecture-law--target-and-enforcement)
7. [Observability](#7-observability)
8. [Reference documents](#8-reference-documents)
9. [KB governance and versioning](#9-kb-governance-and-versioning)

---

## 1. What EduEasy is

EduEasy is a **South African higher-education application platform**. It lets prospective
students discover universities and TVET colleges, submit one application journey (personal
details, documents, programme choices, sponsors/funding), pay application fees in ZAR through
the PayFast gateway, and track each application's status end to end. Institutions and
administrators use the same platform to triage applications, communicate decisions, and
broadcast notices.

One sentence: **one place where a South African student applies, pays, and tracks; one place
where institutions respond.**

Domain language (Ubiquitous Language — used verbatim in code, docs, and tests):

| Term | Meaning |
|---|---|
| Application | A student's submission to one institution/programme; the central aggregate of the platform |
| Student / Learner | The applicant; owner of the journey |
| Institution | University/TVET college receiving applications |
| Sponsor | Funding party attached to an application (bursary, employer, parent) |
| Payment | Application-fee payment in ZAR via PayFast (form initiation, ITN webhook, proof of payment) |
| ITN | PayFast's Instant Transaction Notification webhook — source of truth for payment outcomes |
| Plan type | Fee plan selection (e.g. application fee tiers) governed by `PlanCatalog` |
| Communication | Email/SMS/push message rendered from templates and delivered via providers |
| Broadcast | Mass notification to audiences of students/institutions |
| Conversation | Chatbot session with a student, ending in answer or escalation to a human |
| Document | PDF/Excel artefact produced or ingested by the document pipeline |

## 2. What EduEasy intends to do

- **Grow application volume** without per-institution integrations blocking onboarding — new institutions are data, not code.
- **Own the full student funnel**: registration → application → payment → status tracking → decision → enrolment hand-off.
- **Make payment trustworthy**: idempotent ITN handling, no double process, auditable status transitions, proof-of-payment verification.
- **Communicate reliably**: every state change reaches the student (email/SMS) with template governance and delivery tracking.
- **Serve institutions** with triage, analytics dashboards, and mass broadcasts.
- **Scale the team safely**: each API is an independently deployable Bounded Context with an enforced architecture, so new engineers work inside one context without breaking others.

Non-goals (explicit): being an LMS, owning student-records systems of institutions, general accounting.

## 3. Platform topology — the how

### 3.1 Services

| API | Port | Bounded context responsibility |
|---|---|---|
| edueasy-auth-api | 8089 | Identity: login, JWT/OAuth2 tokens, MFA, password reset, session introspection |
| edueasy-application-api | 8088 | Applications, students, sponsors, institutions; learner analytics (absorbed from dashboard) |
| edueasy-payment-api | 8080 | PayFast integration: initiate, ITN webhook, status, proof-of-payment, expiry |
| edueasy-course-api | 8091 | Course/programme catalogue; cross-API REST client consumer |
| edueasy-communication-api | 8085 | Email/SMS delivery, template rendering, provider switching |
| edueasy-broadcast-api | 8080 | Mass notifications to audiences |
| edueasy-chatbot-api | 8090 | Student-facing assistant: intent → FAQ/DB → LLM → escalation |
| edueasy-document-processor-api | 8081 | PDF/Excel generation, ingestion, zip export packages |
| edueasy-dashboard-api | 8087 | Thin facade over application-api analytics (delegation by design) |

Shared infrastructure: PostgreSQL (primary store), Redis (cache/session), RabbitMQ (events, DLQ), Mailhog (local SMTP), Kong (gateway), Otelf/OpenShift tracing sink.

### 3.2 Shared platform starters (`common-starter-parent`, generic subdomain)

All 9 APIs consume: `common-starter-metrics` (Actuator + Micrometer Prometheus), `common-starter-logging` (Logstash JSON to stdout), `common-starter-tracing` (OTLP), `common-starter-security`, `common-starter-exception`, `common-starter-validation`, `common-starter-idempotency`, `common-starter-outbox`, `common-starter-ratelimit`, `common-starter-http-client`, `common-starter-api-client`, `common-starter-amqp`, `common-starter-redis`, `common-starter-cache`, `common-starter-sentry`, `common-starter-vault`, `common-starter-test`.

Rule: **domain code never imports `za.co.common.*`** — starters are infrastructure.

### 3.3 Clients and delivery

- `edueasy-central-ui` — React SPA (nginx container) for the student/admin web experience.
- `edueasy-central-mfe` — module-federation shell (`edueasy_shell/`) with `student-portal`, `admin-portal` remotes.
- `grafana/` — Prometheus + Loki + Grafana + blackbox-exporter + postgres-exporter monitoring plane.
- `edueasy-github-workflows` — org-level reusable CI/CD (JDK 21/Temurin/Maven, thin per-repo hooks).
## 4. How the APIs fit together

### 4.1 Context map (strategic DDD)

Each API is a **Bounded Context** under `za.co.edueasy.api.<context>`. Relationships:

| Upstream → Downstream | Mode | Mechanism |
|---|---|---|
| auth-api → all | Open Host Service | JWT validation via `common-starter-security` |
| application-api → dashboard-api | Customer/Supplier (dashboard is conformist) | `ApplicationAdminClient` REST — dashboard owns no data |
| payment-api → application/institution/student | Conformist (read-only) | `entity/readonly` shared-schema reads |
| chatbot-api → application/student | Conformist (read-only) | `entity/readonly` reads |
| payment-api ↔ comms/broadcast | Event-driven | RabbitMQ topics via `common-starter-amqp` + outbox |
| course-api → institutions | Open Host Service | REST client |
| all → PayFast | Anti-corruption layer (planned) | `GatewayPaymentHandler` strategy + transformers |

### 4.2 Communication rules

1. **Synchronous** cross-context = REST client only (`common-starter-api-client`, `common-starter-http-client`). Never a shared datasource write.
2. **Asynchronous** = RabbitMQ with `common-starter-outbox` for atomic state+event; DLQ mandatory (`common-starter-amqp`).
3. **Read-only foreign entities** (`entity/readonly`) are a transitional Conformist pattern — allowed, but must gain contract tests (§6) and must never be written to.
4. No API may import another API's internal packages — only published clients.

### 4.3 The two golden flows

**Payment flow (BR1–BR11, payment-api):**
`initiate (validate plan/fee → create PENDING payment → signed PayFast form)` →
student pays at PayFast → `ITN webhook (idempotent; signature verified; source-IP validated)` →
`PaymentStatus` transitions `PENDING → COMPLETE | FAILED | EXPIRED` (expiry scheduled job) →
proof-of-payment verification endpoint → events to comms.

**Application flow (application-api ← dashboard-api):**
student submits/edits application (with sponsor + documents) → institutions/admin triage via
admin endpoints → status changes emit events (comms notifies) → learner analytics read by
dashboard-api through `ApplicationAdminClient` (dashboard stores nothing).

## 5. User journeys

### 5.1 Student journey (primary)

1. **Discover** — lands on central-ui (or MFE student-portal), browses institutions/courses (course-api).
2. **Register / login** — auth-api issues JWT; MFA available; session persisted.
3. **Apply** — creates application(s) in application-api: personal details, programme selection, document upload (document-processor-api validates/stores PDFs).
4. **Sponsor (optional)** — attaches sponsor/funding details to the application.
5. **Pay** — payment-api returns signed PayFast form → PayFast payment in ZAR → ITN webhook confirms → status PENDING→COMPLETE on the UI.
6. **Track** — dashboard/analytics show per-application status; comms-api emails/SMSes every state change.
7. **Assist** — chatbot-api answers "where is my application / what do I still owe" questions; escalates to human after N failures.
8. **Decision** — institution accepts/rejects; student notified via comms + broadcast; documents (acceptance letters) generated by document-processor-api.

### 5.2 Institution / admin journey

1. **Login** (admin-portal MFE) → 2. **Triage** applications in application-api admin endpoints → 3. **Communicate** decisions individually (comms-api) or en masse (broadcast-api) → 4. **Monitor** funnel via dashboard-api analytics (Grafana for ops-level health).

### 5.3 Ops journey

`grafana/` dashboards: service health (blackbox `probe_success`), uptime, JVM, request rate/latency p95, DB connections/Hikari pool, per-container logs (Loki). Bring-up: `docker compose -f docker-compose-complete.yml up -d --build` then `docker compose -f grafana/docker-compose.yml up -d`.
## 6. Architecture law — target and enforcement

### 6.1 Target architecture (every API converges to this)

```
za.co.edueasy.api.<ctx>.interface        — controllers, webhooks, listeners (humble; no rules)
za.co.edueasy.api.<ctx>.application      — use cases (one per business capability), ports, policies
za.co.edueasy.api.<ctx>.domain           — entities/aggregates, value objects, domain services (framework-free)
za.co.edueasy.api.<ctx>.infrastructure   — JPA adapters, REST clients, LLM/ PayFast clients, producers
```

Dependency rule: `interface → application → domain ← infrastructure`. Domain imports nothing framework/`za.co.common`.

### 6.2 Compliance enforcement (how "all 9 APIs comply" is guaranteed, not hoped for)

| Gate | What it checks | Status |
|---|---|---|
| `common-starter-archrules` (ArchUnit 1.4.1 core, tests-jar) | Dependency rule, layer purity, servlet/Spring-web ban in `*Service/*Listener/*Controller`, service-to-service coupling ban, package cycles. Frozen baseline (`archunit_violations/`, committed per API) means existing debt is allowed but **new violations fail the build**. Plain Jupiter test (not the archunit-junit5 engine) — the engine is incompatible with the platform's JUnit 6.0.3 | **Live in payment-api (verified: baseline seeded, enforcement green, negative control red)**. Rollout to remaining 8 = 2-line pom dep + 1-line `ArchitectureTest` each |
| CI gate (`api-ci.yml` / `api-pull-request.yml` step: `mvn verify`) | ArchUnit runs as part of test phase — red build = non-compliant PR | Org workflow step (documented in `edueasy-github-workflows`) |
| PR checklist | Feature/bug PRs touching `src/main/java` must state: KB updated? architecture deviation justified? | Manual, review-enforced |
| Contract tests for `entity/readonly` reads | Break the build when the owning context changes a shared column | Backlog (P2, review item 13) |
| Debt register | `archunit_violations/` files ARE the register — shrinking sets tracked per release | Active (payment-api) |

Commands (per API): seed/shrink baseline `mvn test -Dtest=ArchitectureTest -Darchunit.freeze.refreeze=true`; enforce `mvn test -Dtest=ArchitectureTest`.

Rollout order (matches review §P1): payment-api → auth-api → chatbot-api → document-processor-api → communication-api → broadcast-api → course-api → application-api → dashboard-api.

### 6.3 Non-negotiables (amend only via KB version bump + review)

1. No `jakarta.servlet.*` / Spring web types in service classes.
2. `@Transactional` at method level, never class level.
3. New business capability = new use-case class, not a bigger god service.
4. Cross-context data access only via §4.2 mechanisms.
5. Status transitions live on domain status types (`PaymentStatus` etc.), not scattered string checks.
6. Every API keeps the three starters (metrics/logging/tracing) and its Actuator endpoints internal-only.

## 7. Observability

Single plane: **Grafana + Prometheus (metrics) + Loki (logs)**. No ELK — `LogstashEncoder` is field compatibility, not a dependency. Producing side: `common-starter-metrics` (`/actuator/prometheus`), `common-starter-logging` (JSON logs with `traceId/spanId/requestId/userId`), `common-starter-tracing` (OTLP). Consuming side: `grafana/docker-compose.yml` — Prometheus scrapes all 9 APIs + blackbox-exporter probes `/actuator/health` per API (`probe_success`, `probe_duration_seconds`, `probe_http_status_code`), postgres-exporter for DB, Loki+promtail for logs. Bring-up documented in §5.3. Correlation key across all three signals: `traceId` (logs+traces) and `application`/`instance` labels (metrics).

## 8. Reference documents

Point-in-time audits; superseded by this KB where they conflict. Kept in `references/`:

| Document | Date | Content |
|---|---|---|
| `architecture-review.md` | 2026-09-07 (verified) | Clean Architecture/DDD scorecard per API, findings, P0–P2 backlog |
| `clean-architecture-ddd-review.md` | 2026-09-06 | Detailed per-API evidence tables |
| `frontend-architecture-review.md` | 2026-09-07 | central-ui / central-mfe review (grade C-, P0 blockers) |
| `agent-rules-placement.md` | — | Where Cline skills/rules live |

Per-API `docs/` suites (`api-specification`, `architecture`, `security-guide`, …) remain the fine-grained reference **inside each service repo**; this KB is the cross-cutting truth.

## 9. KB governance and versioning

**This KB is the source of truth.** Rules:

1. **Any refactoring or feature that changes**: architecture (§6), topology (§3), context map / flows (§4), journeys (§5), observability (§7), or domain language (§1) → **must update this file in the same PR**.
2. **Version bump** (SemVer, recorded in header + `CHANGELOG.md`):
   - **MAJOR** — architecture law change, new/retired API, journey-breaking change.
   - **MINOR** — new capability, new integration, new compliance gate, new reference doc.
   - **PATCH** — corrections, clarifications, refreshed metrics.
3. **Tags**: every version bump is committed and tagged `kb-vX.Y.Z` in this repo.
4. **Reviews**: point-in-time audits go to `references/` with date; conclusions are folded into the body (§§1–7), never left only in the reference.
5. **Conflict resolution**: KB body > reference docs > per-API docs > code comments.
6. Owners: platform/architecture team. Review cadence: at every release train, and on any `common-starter-parent` minor bump.

