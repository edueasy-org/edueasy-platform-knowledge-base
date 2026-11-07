# KB Changelog

SemVer. Format: `[X.Y.Z] - date — summary`.

## [1.1.1] - 2026-11-07

### Corrected
- §3.1 documents container vs host-default ports (broadcast-api defaults to 8086 and document-processor-api's `local` profile to 8082 when `SERVER_PORT` is unset).
- §3.1 documents the implemented schema layout: `edu_pay` is a shared core schema for payment, application, course and dashboard tables; `edu_course` / `edu_dash` exist but are empty; schema changes are owned by `edueasy-db-pipeline-api`.

## [1.1.0] - 2026-09-07

### Changed
- **KB extracted into its own repository** (`edueasy-platform-knowledge-base`) — no longer versioned inside `edueasy-db-pipeline-api`. KB §9 conflict-order and tagging unchanged.

### Added
- `per-api/`: absorbed the service-level documentation suite (chatbot-api: api-specification, api-usage-guide, architecture, security-guide, deployment-guide, setup-and-configuration-guide, testing-guide, test-cases-specification, production-readiness-checklist, quick-reference, application-env-template.yml).

## [1.0.1] - 2026-09-07

### Changed
- §6.2: `common-starter-archrules` is **live** — payment-api pilot verified end-to-end (baseline seeded in `archunit_violations/`, enforcement green, deliberate `HttpServletRequest` violation correctly failed the build). Uses plain ArchUnit-core + Jupiter (archunit-junit5 engine incompatible with JUnit platform 6.0.3).

## [1.0.0] - 2026-09-07

### Added
- Initial consolidated knowledge base: product definition, intent, platform topology, context map, golden flows, user journeys (student/admin/ops), architecture law + compliance gates (§6), observability (§7), governance (§9).
- Compliance gate design: `common-starter-archrules` ArchUnit module (frozen baselines per API), CI `mvn verify` gate, debt register via `archunit_violations/`.
- Reference copies of all point-in-time architecture/code reviews under `references/`.
