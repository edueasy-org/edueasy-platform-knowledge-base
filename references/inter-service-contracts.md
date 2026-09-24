# EduEasy Inter-Service Contracts

Machine-verified inventory of every **service-to-service** HTTP call in the
platform: what the caller sends, what the provider returns, and how failures
are translated.

This is deliberately *not* the browser-facing API. Those are already documented
per service via springdoc — see
[per-api/api-specification.md](../per-api/api-specification.md). This document
covers only the calls where one backend talks to another backend.

Last verified: 2026-09-24, against the running `docker` profile.

---

## 1. Topology

```
                      ┌──────────────────┐
  browser ──────────► │  auth-api  :8089 │  (identity, JWT issuance)
                      └────────▲─────────┘
                               │ C1 sponsor register/login (public, no token)
                               │
            ┌──────────────────┴───────────────┐
            │                                  │
  ┌─────────▼──────────┐        ┌──────────────▼──────────┐
  │ application-api    │◄───────┤ dashboard-api           │
  │ :8088              │  C4    │ :8087                   │
  │  (learner data)    │  admin │  (admin screens)        │
  └─────────▲──────────┘        └─────────────────────────┘
            │  C2 APS / C3 payment-status (service token)
            │
  ┌─────────┴──────────┐
  │ course-api  :8091  │  (course selection, gated on APS + payment)
  └────────────────────┘
```

`communication-api` talks to **external** providers (SMS gateway, WhatsApp via
Meta / Twilio / Clickatell). Those are outbound integrations, not internal
contracts, and are listed in section 5 for completeness only.

---

## 2. C1 — application-api → auth-api

**Client:** `za.co.edueasy.api.application.integration.AuthApiClient`
**Base URL:** `${sponsor.auth-api.base-url}` (default `http://localhost:8089`)
**Transport:** `RestClient` via `common-starter-http-client`
**Auth:** none — both operations are public entry points

### C1.1 Register a sponsor

`POST {sponsor.auth-api.register-path}` (default `/auth/register`)

Request body — `name` is split on the first space by the client before sending:

| Field               | Type   | Notes                                    |
|---------------------|--------|------------------------------------------|
| `email`             | string | sponsor's email                          |
| `password`          | string | raw, unhashed — auth-api does the hashing |
| `firstName`         | string | portion of `name` before the first space  |
| `lastName`          | string | remainder; `""` when `name` has no space  |
| `userTypeCode`      | string | hard-coded `SPONSOR`                      |
| `accountStatusCode` | string | hard-coded `ACTIVE`                      |

```json
{
  "email": "sponsor@example.co.za",
  "password": "s3cret",
  "firstName": "Thandi",
  "lastName": "Mokoena",
  "userTypeCode": "SPONSOR",
  "accountStatusCode": "ACTIVE"
}
```

Response `200` — `SponsorLoginResponseDTO`:

```json
{ "accessToken": "<jwt>", "user": { "userId": "...", "...": "..." } }
```

A `null` body **or a `null` `accessToken`** is treated as provider outage and
raised as `AUTH_UNAVAILABLE`.

### C1.2 Log a sponsor in

`POST {sponsor.auth-api.login-path}` (default `/auth/login`)

```json
{ "email": "sponsor@example.co.za", "password": "s3cret" }
```

Response is the same `SponsorLoginResponseDTO` as C1.1.

### C1.3 Error translation

| auth-api status | application-api error code | Surfaced as       |
|-----------------|----------------------------|-------------------|
| `409 Conflict`  | `EMAIL_ALREADY_EXISTS`     | 409 to the caller |
| `401` / `403`   | `AUTH_INVALID_CREDENTIALS` | 401 to the caller |
| anything else   | `AUTH_UNAVAILABLE`         | 503 to the caller |

`application-api` re-exposes these publicly as
`POST /api/v1/sponsors/register` and `POST /api/v1/sponsors/login`, so the
browser contract is the same shape with a translated status code.


---

## 3. C2 / C3 — course-api → application-api

Both use `ServiceTokenProvider` (OAuth2 client-credentials) and share the same
resilience policy: **3 attempts, 200 ms fixed backoff**. Failures degrade
gracefully — the caller receives `Optional.empty()` rather than an exception, so
a dashboard outage cannot block course selection.

**Base URLs:** `course.aps.dashboard-base-url` and `course.dashboard.base-url`
(both default `http://localhost:8088`)

### C2 — resolve authoritative APS

**Client:** `DashboardApsClient`
`GET {course.aps.internal-aps-path}` (default
`/api/v1/internal/applications/{applicationId}/aps`)

- Response: an untyped map — the APS payload. See the risk note in section 6.
- The **dashboard is the source of truth**. If it is unreachable or has no APS
  yet, the caller may fall back to a client-supplied APS for UX only.

### C3 — resolve payment status

**Client:** `DashboardPaymentStatusClient`
`GET {course.dashboard.payment-status-path}` (default
`/api/v1/applications/{applicationId}/journey/payment-status`)

- Response: an untyped map describing direct payment **or** active sponsorship.
- Used to gate course selection.

### C2/C3 metrics exported to Prometheus

| Metric                                | Type    | Meaning                    |
|---------------------------------------|---------|----------------------------|
| `course.aps.fetch`                    | counter | APS fetch attempts         |
| `course.aps.fetch.success`            | counter | successful resolutions    |
| `course.aps.fetch.failure`            | counter | network / timeout failures |
| `course.aps.fetch.duration`           | timer   | latency histogram          |
| `course.payment_status.fetch`         | counter | payment-status attempts    |
| `course.payment_status.fetch.success` | counter | successful resolutions    |
| `course.payment_status.fetch.failure` | counter | failures                   |

---

## 4. C4 — dashboard-api → application-api

**Client:** `za.co.edueasy.api.dashboard.integration.ApplicationAdminClient`
**Base URL:** `${application-api.base-url}` (default `http://localhost:8088`)

The dashboard **no longer owns learner data**. Admin screens delegate to the
application service and **forward the caller's JWT unchanged**, so
application-api performs its own `ADMIN` / `SUPPORT` authorization. A service
token is also attached for gateway-level authentication.

### C4.1 Search applications

`GET /api/v1/admin/applications/search`

| Query param      | Required | Notes                   |
|------------------|----------|-------------------------|
| `limit`          | yes      | page size               |
| `offset`         | yes      | page offset             |
| `status`         | no       | omitted when null/blank |
| `authUserId`     | no       | omitted when null/blank |
| `applicationRef` | no       | omitted when null/blank |

Response: `PaginatedListDTO<Map<String, Object>>`.

### C4.2 View a journey

`GET /api/v1/admin/applications/{applicationId}/journey` (used by
`viewJourney`) — response is an untyped map.

Any `RestClientException` becomes
`ServiceException("DELEGATION_FAILED", "Application service unavailable")`.

---

## 5. External outbound integrations (not internal contracts)

`communication-api` only. These are third-party, not EduEasy services, so they
are **not** governed by this document — but they are the most likely source of
production incidents, and they are centralised behind `WA_META_BASE_URL`,
`WA_TWILIO_BASE_URL`, `WA_CLICKATELL_BASE_URL` and `SMS_GATEWAY_URL` so a
provider change is a config change.

---

## 6. Known risks

1. **Three of the four contracts are untyped on the consumer side.** C2, C3 and
   C4.2 deserialize into `Map<String, Object>`. Nothing at compile time detects
   a renamed or removed field, so a provider-side breaking change surfaces as a
   `null` at runtime. Replacing these with shared DTOs in a published
   `common-starter-contracts` artifact is the single highest-value hardening
   step for this layer.
2. **No contract tests.** Nothing asserts that auth-api still honours the shape
   application-api expects. Pact, or a simple WireMock-based consumer test,
   would catch drift in CI.
3. **C1 sends the sponsor password in cleartext to auth-api** and relies on
   transport security. Under the current Docker network that is plain HTTP —
   acceptable on a private bridge, **not** acceptable once split across hosts
   without TLS.
4. **The internal endpoints are path-configurable**, which is good for
   deployment, but a typo in `*-path` fails silently into the
   `Optional.empty()` fallback rather than loudly.

---

## 7. Testing these contracts

Every provider operation above is a documented springdoc operation, so it can be
exercised directly from Swagger UI at the provider's port:

| Contract     | Provider        | Provider spec                                |
|--------------|-----------------|----------------------------------------------|
| C1.1 / C1.2  | auth-api :8089  | `http://localhost:8089/swagger-ui/index.html` |
| C2           | application-api | `http://localhost:8088/swagger-ui/index.html` |
| C3           | application-api | same                                         |
| C4.1 / C4.2  | application-api | same                                         |

C2/C3/C4 require a service token; obtain one from auth-api's service-client
token endpoint and send it as `Authorization: Bearer <token>`.
