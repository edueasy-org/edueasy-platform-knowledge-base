# Harsh Code Review: Frontend Design / Clean Architecture / DDD / Clean Code

> Reviewed: 2026-09-07
> Scope: `edueasy-central-ui` (legacy SPA, React 19 + Module Federation, ~9,900 LOC src) and
> `edueasy-central-mfe` (federation target: shell + student-portal + admin-portal remotes)
> Standards: Frontend design & architecture review, Clean Architecture (R. C. Martin), DDD (Evans),
> Clean Code (R. C. Martin), Refactoring (Fowler) — `clean-architecture-ddd` skill rules
> Note: the `frontend-design` skill directory (`.cline/skills/frontend-design/`) is **empty** — no
> SKILL.md/rules installed. This review applies the frontend-design discipline from first principles
> and the loaded `clean-architecture-ddd` nano rules. Install the skill content or the discipline
> checklist is not reproducible (see `docs/agent-rules-placement.md`).
> State reviewed: working tree of `edueasy-central-ui` includes an **uncommitted refactor** (12 files,
> +806/−1,695) that was under active change during this review.

---

## Executive Summary

**Grade: C-**

The frontend is structurally ahead of the Java APIs reviewed on 2026-09-06 (D+): pages/services/components
are separated, a shared UI kit is emerging, error normalization is centralized in the axios layer, and the
MFE federation design (promise-based dynamic remotes, error boundary, graceful degradation, singleton
shared scopes) is genuinely good.

But the review found: the working tree **does not type-check** (33 `tsc` errors), the uncommitted diff
**breaks three import paths, drops an auth safety guard, and collapses readable code into unreadable
single lines**, the payment flow **cannot compile against its own service layer**, and the MFE shell is a
**divergent fork** of the legacy app (4,011 differing lines, no `node_modules` in any app, unverified
build). Test suite is green (28 tests) but covers ~5% of the surface and misses every bug above.

The bright spot is `resumableUploadService.ts` (chunked, resumable, offline-guarded, throughput-aware) —
proof the team can ship careful engineering when it wants to.

---

## Verified Status (run, not claimed)

| Check | central-ui | central-mfe (shell) |
|---|---|---|
| `tsc --noEmit` | **FAIL — 33 errors** | **FAIL — 33+ errors** (inflated by missing deps) |
| `npm test` | PASS — 3 suites / 28 tests | not run — no `node_modules` in any app |
| `npm run lint` | PASS — **but blind** (no type-aware rules; zero findings on 33 compiler errors) | not run |

`tsconfig.json` sets `"strict": true`; the 25 `TS7006 implicit any` failures are therefore genuine
type errors, not style. ESLint uses `tseslint.configs.recommended` **without project service**, so
`TS2307`/`TS7006` never surface in lint — lint green means nothing about type health.

---

## P0 — Blocks merge / blocks production (fix first)

### 1. Broken import paths introduced by the uncommitted diff (3 files)

`applicationService.ts`, `journeyService.ts`, `resumableUploadService.ts` all import `"../api"`.
From `src/services/` that resolves to `src/api` — **no such module**. HEAD had `'./api'` (correct).

```
src/services/applicationService.ts:1    TS2307 Cannot find module '../api'
src/services/journeyService.ts:1        TS2307 Cannot find module '../api'
src/services/resumableUploadService.ts:1 TS2307 Cannot find module '../api'
```

Whoever did the quote-collapse pass (`'` → `"`) also rewrote relative paths. Fix: `./api` in all three.
The 20+ `TS7006 implicit any` errors in consumers cascade from this one break (`.then((r) => ...)` where
the axios instance became `any`).

### 2. Payment flow cannot compile against its own service layer

`src/pages/Payment.tsx` calls APIs that do not exist in the current services:

- `institutionApplicationService.getStatusDashboard(applicationId)` — method is named `statusDashboard`
  (TS2551).
- `paymentService.initiate({ applicationId, planType, institutionIds })` — signature is
  `initiate(studentId, gateway: "PAYFAST" | "STRIPE")` (TS2554: expected 2 args, got 1).
- reads `initiation.formData?.processUrl ?? initiation.paymentFormData?.processUrl ?? initiation.redirectUrl`
  — return type has only `redirectUrl` (TS2339 ×2).

This is a half-done contract migration: the caller was updated toward the backend's real
`{applicationId, planType, institutionIds}` create contract (documented in the MFE README audit) but the
service was never updated to match. **The page can neither compile nor run.** Decide the contract,
update the service to match the caller, inline the `initiate`/`statusDashboard` renames.

### 3. AuthContext dropped its corrupted-session guard

HEAD parsed `sessionStorage` user inside try/catch and removed the poisoned key; the diff replaced it
with bare `JSON.parse(stored)` and `setUser(JSON.parse(stored))` at startup:

```ts
-    let parsedUser ... try { parsedUser = JSON.parse(stored) } catch { /* drop + remove */ }
+    setUser(JSON.parse(stored));   // throws on any malformed payload → white screen at boot
```

Corrupt `edueasy_user` (old schema, partial write, hand-edited) now crashes the whole provider instead of
logging the user out. Restore the guard; also `JSON.parse` result is untyped `any` — cast to `AuthUser`.

---

## P1 — Clean Architecture / DDD findings

### 4. God pages are the frontend's god services

Same disease as the backend, same grade of it. Domain + workflow + persistence + presentation live in one
default-export component:

- `pages/Landing.tsx` — **2,597 lines**
- `pages/Journey.tsx` — 725 lines: 6-step wizard, field state, SA-ID date/gender derivation, step
  save/complete orchestration, resume-upload bookkeeping, draft autosave, per-step validation, and the
  entire step render — one component.
- `pages/SponsorDashboard.tsx` — 732 lines.

**Clean Architecture:** a page is the outermost adapter. Domain behavior (which institution an application
is "started" against — `Journey.startApplication` hardcodes `INSTITUTIONS[0]` from the types module, no
learner choice), APS derivation, and step-completion rules are policy and belong inward of the view.
Today the "domain" is JS inside `pages/`.

**Refactoring first move per page:** extract (1) form-state hooks or stores per wizard step, (2) a
use-case-shaped `journeyFlow` module (start → saveStep → completeStep → submit) with a plain state model,
(3) keep the component as a humble view. Start with `Journey` — highest-value flow, currently
unreviewable in a single pass.

### 5. Service layer is a nameless RPC bag; contracts are silent

```ts
export const paymentService = {
  initiate: (studentId: string, gateway: "PAYFAST" | "STRIPE") =>
    paymentApi.post<{ redirectUrl: string }>(API_PATHS.PAYMENTS.BASE, { studentId, gateway }).then(r => r.data),
```

Every service is the same shape: `const xService = { method: (...) => api...then(r => r.data) }`.
No explicit request/response contracts beyond axios generics, no error policy (normalization lives in
`api.ts` interceptors — good — but services silently depend on it), no unit boundary. The `payments`
mismatch in P0-2 is the direct cost.

Recommend: keep the interceptor strategy, but give every service **typed method-level input/output
models** (`PaymentInitiationRequest`, `PaymentInitiationResult`) and one service per bounded backend.
`"PAYFAST" | "STRIPE"` literals and `redirectUrl` strings then surface as one-line compile breaks,
not runtime 404s.

### 6. DTO duplication and fabricated domain values

- `types/index.ts` vs `types/dto.ts` split is inconsistent (some pages import `PaymentStatus` from
  `../types`, others from `../types/dto`).
- Status strings duplicate everywhere: `StatusBadge` carries a 17-key status map, `utils/payment.ts`
  normalizes again, `ProtectedRoute` re-derives role flags already exposed by `AuthContext`. One status
  vocabulary, one place.
- `Journey.startApplication` fabricates `studentId`/`institutionName` from `INSTITUTIONS[0]` — domain
  magic numbers in the view. The backend owns application creation; pages should call a
  `startApplication(user)` use case, not assemble an entity.

### 7. Auth/session is hand-rolled sessionStorage protocol

Five magic keys (`edueasy_token`, `edueasy_refresh_token`, `edueasy_user`, `edueasy_token_expiry`) are
spelled out in `AuthContext.tsx` *and* `services/api.ts` and mutated from both; plus `edueasy_reg_info`,
`edueasy_journey_*`, `edueasy_upload_*`, `edueasy_active_uploads`. No single owner; source of truth and
interceptor can drift. Extract a `session.ts` module owning key names, validation and read/write; let
`AuthContext` and the interceptors both import it.
## P2 — Frontend design findings

### 8. The uncommitted diff is a readability regression

The working-tree refactor swapped multi-line structure for single-line density. Contrasts from
`Register.tsx` / `Payment.tsx` / `Journey.tsx` in the current diff:

```ts
// before (readable)
if (!consent.privacy || !consent.terms || !consent.idProcessing) {
  setError('Please accept all required agreements.');
  return;
}
// after (current tree)
if (!consent.privacy || !consent.terms || !consent.idProcessing) { setError("Please accept all
required agreements."); return; }
```

JSX was likewise collapsed element-by-element (the `Set` helper, the `input`/`label`/`p` blocks in
`Register.tsx`), and `Journey.updateSubject` maps with inline casts on one line. This is the exact
anti-pattern Clean Code targets: **local reasoning destroyed**. Formatting is not refactoring.

The intent was a real refactor (new `common/ui` kit, `resumableUploadService`, `allowedRoles` on
`ProtectedRoute` — all good). The quote/indent collapse was a separate bad patch. Keep the functional
changes; restore `prettier --write` output and re-map the multi-line shapes. `lint-staged` already runs
prettier; the tree was committed around it. Run `npx prettier --write src` in central-ui and the diff
shrinks to near-zero.

### 9. Two copies of the same components, half-migrated

The new kit landed in `components/common/ui/` (`Button`, `Card`, `Input`, `Alert`, `StatusBadge`,
`ErrorAlert`, `Spinner`), but the old independent copies are still live and still imported:

- `components/ErrorAlert.tsx` + `components/common/StatusBadge.tsx` — still used by pages
  (`ApplicationStatus`, `AdminDashboard`, `CourseSelection`, `InstitutionApplications`, `LearnerInsights`,
  `StudentDashboard`); `Payment.tsx` imports `StatusBadge`/`ErrorAlert` from `common/ui` while
  `InstitutionApplications.tsx` imports both old and new.
- Result: two status-color maps, two error parsers, two button semantics — the drift the migration was
  meant to kill.

Finish the migration: delete the old files, re-point remaining pages at `common/ui`.
### 10. MFE topology is right; the code duplication is not

Good, keep: promise-based dynamic remotes with `onerror` fallback, `RemoteErrorBoundary` + retry,
singleton `react`/`react-dom`/`react-router-dom` shared scopes, shell-owned `AuthContext`/`api`/`ui`
exposed and consumed by both portals via `edueasy_shell/*`. That is a correct host/remote split.

Problems:
- **Shell is a divergent fork of central-ui**: `diff` of `central-ui/src` vs `shell/src` = 4,011
  differing lines, while `api.ts` is byte-identical. The legacy SPA is still production while the shell
  keeps being patched — every change must land twice. The README's migration plan says new features land
  only in the MFE; the working-tree refactor in central-ui (P0-1), plus the new `resumableUploadService`
  + `documentApi` already in the shell, show drift compounding both ways.
- **Contract stubs**: each portal carries a hand-written `types/remotes.d.ts` declaring `edueasy_shell/*`
  modules with duplicated shapes for `AuthContext`, `api`, `ui`. One shell-side change (e.g. `isSupport`),
  silently mismatches the other copies. Generate declarations from the shell build (`dts: false` today) or
  publish one shared package.
- **No local verification possible**: no `node_modules` in any of the three apps, so claimed MFE
  type-check/test/build green is unverified in this review. `shell` `tsconfig` does not know
  `JSX.IntrinsicElements` and `@sentry/react`/`axios` cannot resolve. Install deps and run suites in CI
  before trusting the federation cutover.
- **Remotes keep duplicating per-API URL/`getBaseUrl` logic** and hard-coded `http://localhost:808x`
  defaults between shell, student and admin copies (the payment/dashboard contract gaps are already
  documented in the README audit). The three `edueasy_shell/*` interfaces should be the only hand-off
  points.

### 11. Tests: green, near-useless

- 3 suites / 28 tests total (central-ui): `AuthContext.test`, `api.test`, `AdminDashboard.test`.
- **Zero tests** for: `Journey`, `Payment`, `Register`, `Login`, `ProtectedRoute`, the journey/
  application/sponsor/student services, and `resumableUploadService` (the most complex new unit — a
  chunking/resume state machine — untested).
- The suite passes *while the app fails to compile* and *while the payment flow calls methods that do
  not exist* — it never touches either path. Tests assert the past, not the product.
- MFE ports the same 3 suites verbatim into shell; `admin-portal` has 0 tests for 5 of its 6 pages.

Target the P0s first, then: `ProtectedRoute` role-matrix tests (`allowedRoles`/`isSupport` logic is pure
and cheap), a `ResumableUploadService` state-machine test with stubbed `documentApi`/`sessionStorage`,
and one smoke test per wizard step of `Journey`.
---

## Metrics

| Metric | central-ui | MFE shell+portals |
|---|---|---|
| `tsc --noEmit` | 33 errors | unverified (deps missing) |
| Test suites / total | 3 / 28 | 3 (+5th) / 28+ |
| Pages untested (of N) | 16 / 19 | 7 / 15 (incl. 5/6 admin) |
| God page max LOC | Landing 2,597 | same source copy |
| Shared UI kit | present, half-migrated | exposed as `edueasy_shell/ui` |
| Service contract types | none | none |
| Duplicate component sets | 2 (old vs new) | shell only |
| Domain behavior in pages | ~100% | same |

---

## Priority plan

**P0 (before merge — all four are mechanical):**
1. Fix `../api` → `./api` in 3 service files.
2. Reconcile `Payment.tsx` with a payment service matching the chosen backend contract (or revert the
   caller — pick one, compile it).
3. Restore the AuthContext corrupted-session guard (`try/catch` around `JSON.parse`).
4. `npx prettier --write src` to unwind the density regression.

**P1 (2 weeks):** extract `Journey` into flow module + step hooks + humble view; typed request/result
models per service; single `session.ts` key owner; delete legacy duplicate components.

**P2 (month):** generate `remotes.d.ts` from the shell build; add tests for `ProtectedRoute`,
`resumableUploadService`, wizard steps; install + build + test the MFE in CI before cutover; stop
adding features to `edueasy-central-ui` — land only in the MFE per the migration plan.

---

## Conclusion

The platform's frontend is **better-structured than its backend but is not clean**. The federation
design is genuinely thoughtful; the code around it is an accumulation of good intentions undone by
unchecked formatting passes, silent contract drift between pages and services, and a test suite that is
green while the app is red. Everything in P0 is a mechanical fix that a CI gate (`tsc --noEmit` + a
lint that is type-aware + a payment flow test) would have caught. Add the gate, finish the `common/ui`
migration, de-fork the shell from the legacy SPA, and the rest follows.