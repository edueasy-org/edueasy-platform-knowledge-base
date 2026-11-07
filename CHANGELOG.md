# KB Changelog

SemVer. Format: `[X.Y.Z] - date — summary`.

## [1.2.0] - 2026-11-06

### Changed
- §4.3 and §5.1 now reflect the implemented learner journey: select programmes before payment; payment or sponsorship is required before submitting institution forms; EFT proofs remain pending until reviewer approval.
- Added a current-code journey/security audit reference with verified findings and remediation acceptance criteria.

### Corrected
- Reclassified the 2026-08-22 frontend/backend contract findings that are already resolved in current code; retained authorization and error-handling gaps as actionable findings.

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
