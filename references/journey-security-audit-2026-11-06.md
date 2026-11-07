# EduEasy Journey and Security Verification

**Verified:** 2026-11-06  
**Scope:** Current student payment/application flow, the related course and payment API routes, and the MFE/central-ui service contracts.  
**Method:** Compared controllers, use cases, service clients, frontend service calls, and current README/knowledge-base claims. Findings below are source-verified; this was not a production penetration test or a complete runtime test of every platform journey.

## Executive summary

The canonical student flow is **choose institutions/programmes → pay or obtain an active sponsorship → submit institution forms → track decisions**. course-api records choices before payment and enforces the payment check only in `SubmitInstitutionApplicationsUseCase`. EFT proof uploads remain `PENDING_VERIFICATION` until an authorized reviewer approves them.

The 2026-08-22 MFE audit contains stale findings: current frontend payment paths and initiation payloads match payment-api, the dashboard analytics routes cited there now exist, and sponsor search was previously corrected. Route presence does not by itself establish complete response-schema compatibility.

## Current-code findings

### JSEC-01 — Learner application ownership is not enforced consistently in course-api

**Priority:** P1 · **Confidence:** 9/10 · **Status:** Open

- `CourseRecommendationController.getRecommendations` accepts an `applicationId` without receiving or checking the authenticated learner.
- `ApplicationChoiceService.applyForCourse` records the caller as `createdBy` but does not verify that the application belongs to that caller.
- `SubmitInstitutionApplicationsUseCase` verifies payment by the supplied application ID, then loads all choices for that ID without filtering them to the authenticated learner. It creates forms using the caller's ID.
- The status-dashboard path filters choices by creator, but that does not establish ownership of the underlying application record.

**Risk:** An authenticated learner with the normal application scopes can use another learner's application ID to retrieve recommendations, write choices, or attempt final submission. The submission path can associate choices with a different caller.

**Jira-ready title:** `Enforce learner ownership across course recommendations, choices, and submission`

**Acceptance criteria:**
1. Every learner-facing course route verifies that the application belongs to the authenticated user before returning or changing data.
2. Final submission includes only the authenticated learner's choices and rejects an application not owned by that learner.
3. Ownership failures do not reveal whether another learner's application exists.
4. Tests cover two distinct learners for recommendations, choice creation, status dashboard, and submission.
5. Valid course choice remains allowed before payment; the existing payment/sponsorship gate remains at final institution-form submission.

### JSEC-02 — Institution application routes lack institution-level authorization

**Priority:** P1 · **Confidence:** 9/10 · **Status:** Open

- `InstitutionApplicationController.listByInstitution` requires only `SCOPE_APPLICATION_READ`.
- `InstitutionApplicationController.decide` requires only `SCOPE_APPLICATION_WRITE`.
- `DecideInstitutionApplicationUseCase` loads a form by public tracking reference and updates it; it does not verify that the caller represents the form's institution.
- Those generic application scopes are also used by learner-facing routes.

**Risk:** A principal with ordinary application read/write scopes may view applications for an arbitrary institution or change an admission decision.

**Jira-ready title:** `Restrict institution application review and decisions to authorized institution staff`

**Acceptance criteria:**
1. Institution list access is checked against an institution identity/role bound to the requested `institutionId`.
2. Decision updates verify that the caller is authorized for the target form's institution.
3. Learner and unrelated-institution principals receive 403/404 and cannot read applicant PII or alter decisions.
4. Tests cover authorized and unauthorized institution users, learners, and administrators; decisions remain auditable.

### JSEC-03 — Payment routes do not consistently enforce record ownership

**Priority:** P1 · **Confidence:** 9/10 · **Status:** Open

- `PaymentController` grants payment reads through `PAYMENT_READ_AUTHORITY`, including the standard `edueasy-payment.user` role and `SCOPE_payment:read`. The collection, payment-ID, status, institution, and application-ID read routes do not perform an owner check.
- Payment initiation and proof upload accept an `applicationId`; their use cases load that application but do not compare its `authUserId` with the caller.
- The explicit self/staff check on `/payments/students/{authUserId}` treats any authority beginning with `ROLE_edueasy-` as payment staff, not just defined payment-staff roles.

**Risk:** An authenticated standard user may read another learner's financial/payment details or create a payment/proof record against an application they do not own.

**Jira-ready title:** `Enforce owner-or-explicit-payment-staff authorization on payment records`

**Acceptance criteria:**
1. A learner can initiate payment, upload proof, and read only their own application/payment records.
2. Cross-learner access is allowed only for explicitly defined payment staff/admin authorities; unrelated EduEasy roles are not treated as staff.
3. Global payment list and other bulk reads are limited to explicitly authorized back-office principals.
4. Tests cover cross-user read/write attempts for every payment route, owner access, and each approved staff role.

### JSEC-04 — Proof-of-payment upload lacks server-side file constraints

**Priority:** P2 · **Confidence:** 8/10 · **Status:** Open

- The upload endpoint reads the entire multipart file into memory and has no endpoint-level allowlist, signature/content validation, or explicit file-size check.
- The storage call records every uploaded file as `application/pdf`, even though the UI also accepts images.

**Risk:** Unexpected or oversized content can be persisted as a proof document, and recorded content type may not match the actual file.

**Jira-ready title:** `Validate proof-of-payment uploads before storage`

**Acceptance criteria:**
1. Server-side file size and allowed-type limits are explicit and tested.
2. Validation uses file content/signature, not only the submitted extension or MIME header.
3. The stored content type reflects validated content; invalid files are rejected with a clear client error.
4. Malware scanning is applied or an approved compensating control is documented before reviewer download.

### JSEC-05 — Student payment screens mask status-service failures

**Priority:** P2 · **Confidence:** 9/10 · **Status:** Open

- Payment and institution-selection pages convert any payment-status request error into `UNPAID`. The payment API already returns a default `PENDING` response when no payment exists, so network/authorization failures are not equivalent to “no payment.”
- Payment initiation suppresses status-dashboard errors and continues with an empty institution list, even though the payment create contract requires at least one institution.

**Risk:** The learner can be shown an incorrect unpaid state or receive an avoidable payment-validation failure instead of the actual service error.

**Jira-ready title:** `Surface payment status and institution lookup failures in student journeys`

**Acceptance criteria:**
1. A missing payment uses the API's explicit default response; transport, authorization, and server failures are surfaced as errors.
2. Payment initiation does not continue if selected institutions cannot be loaded.
3. Error, retry, and loading states are covered by frontend tests in both central-ui and student-portal.

### JSEC-06 — Payment-by-student does not return the latest payment across applications

**Priority:** P2 · **Confidence:** 9/10 · **Status:** Open

- `PaymentQueryService.getPaymentsByStudent` iterates the learner's applications and returns the first application with any payment. It chooses the last element only within that application's payment list.
- The controller contract describes this as the learner's most recent payment; applications/payment history may contain multiple records.

**Risk:** The displayed learner payment status can be stale or tied to a different application than the one the learner is viewing.

**Jira-ready title:** `Return payment status for the correct application or latest learner payment`

**Acceptance criteria:**
1. The endpoint contract clearly specifies whether status is per application or global per learner.
2. Queries select the correct application/latest payment deterministically; ordering does not depend on repository iteration order.
3. Tests cover multiple applications and multiple payments, including pending and completed records.

### JSEC-07 — Learner document upload trusts client-supplied MIME type

**Priority:** P2 · **Confidence:** 8/10 · **Status:** Open

- application-api checks document size (10 MB) and allows selected MIME types, but the MIME type is supplied by the multipart client and the service does not verify the file signature.
- The reviewed upload path stores the bytes and marks the document index pending; no malware scan is invoked before persistence in this path.

**Risk:** A crafted upload can be stored under a misleading content type and later handled as a permitted document.

**Jira-ready title:** `Validate and scan learner document uploads before processing`

**Acceptance criteria:**
1. Validate file signatures against the permitted document types in addition to the submitted MIME type.
2. Scan uploads before making them available to downstream processing/review, or document and approve an equivalent control.
3. Keep the 10 MB limit enforced server-side and add tests for spoofed MIME, malformed files, oversized content, and valid PDFs/images.

## Corrected claims and resolved contract checks

| Claim in older material | Current evidence | Disposition |
|---|---|---|
| Student payment-status route does not exist | Both frontends use `/payments/students/{authUserId}`; payment-api maps that route | Stale finding |
| Payment-initiation payload is incompatible | Both frontends send `applicationId`, `planType`, and `institutionIds`; the UI reads hosted-checkout form data | Stale finding |
| Dashboard analytics routes are absent | The previously cited analytics routes now exist | Stale finding; full response-schema parity was not revalidated |
| Sponsor search contract is mismatched | The prior audit records that the service was corrected to supported parameters and response fields | Resolved in earlier work |
| Payment unlocks course selection | Actual course-api flow allows choices first and checks payment at final submission | Corrected in KB and both payment pages |
| EFT proof upload immediately completes payment | Backend creates `PENDING_VERIFICATION`; reviewer approval is required | Corrected in API/frontend comments and documentation |
| Non-`DRAFT` application status proves payment | The legacy payment-check overload used to bypass payment verification | Fixed during this review; regression test now requires an actual payment/sponsorship result |

## Verification limits and follow-up

Authentication endpoints include rate limits, and application-api document operations verify application ownership and enforce a size/MIME allowlist. This source review did not certify the full MFA policy, production TLS termination, malware-scanner deployment, every API response schema, or every admin/sponsor/chatbot/document journey at runtime. Those claims must not be reported as verified from this review alone.

The seven open findings above are the Jira backlog candidates. Create one Jira issue per finding, retain the indicated priorities, and link each issue to this audit and the affected repository paths. The application-status payment bypass was fixed in this review and is not an open Jira item.
