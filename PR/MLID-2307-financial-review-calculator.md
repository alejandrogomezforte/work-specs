# [MLID-2307] feat(financial-review): Order Details Financial Calculator with order-scoped persistence and graceful AI degradation

## Summary

- Adds a new **Financial Review** sub-tab to the Order Details v2 dashboard panel. The tab is a patient-responsibility calculator with two sub-views: a list of saved calculations for the current order (each expandable in place to reveal its breakdown) and a new-calculation form. Saved calculations persist on a new order-scoped MongoDB collection and survive page refreshes.
- Each calculation stores nine numeric inputs, three computed responsibility totals, and a `breakdown` object with deterministic per-step bullet arrays plus an AI-generated reason paragraph (Azure OpenAI, gpt-5-nano).
- The feature is gated behind a `ORDER_FINANCIAL_CALCULATION` feature flag at both the tab-visibility (UI) and route-rendering (URL) levels. Default is `false`; the surface is dark in production until QA sign-off.
- **Coexistence**: the legacy `/patient/[id]/financial-calculator` flow is **not modified**. Both calculators continue to work independently. Sunsetting the patient-scoped flow is a separate ticket.
- **Graceful degradation**: if Azure OpenAI is unavailable for any reason (missing credentials, network policy, API error), the save still completes with an empty `reason` field and a warning is logged. The UI conditionally omits the Reason section when empty. The save itself never fails because of an AI problem.
- **Infrastructure hygiene fix bundled**: `AZURE_OPENAI_API_VERSION` was being read from `process.env` by code dating back to 2025-06-27 but was never added to terraform. This PR codifies it in `terraform/main.tf` for both stage and prod container apps. **Anatoliy Kolodkin recommended as reviewer** for the infra-side change (he introduced the original code reference).

## Delivery Stages

The work was delivered in two stages because the original ticket lacked clear data-layer specifications and required a product-owner conversation before the persistence design could be finalised.

| Stage | Scope | Commit |
|---|---|---|
| Stage 1 — UI | Sub-tab routing, components, feature flag, in-memory seed for design validation, full TDD coverage | `cceefaf00` (2026-05-20) |
| Develop merge | Catch up with develop before Stage 2 | `8e9007de6` (2026-05-27) |
| Stage 2 — Data layer | Mongoose model, REST API, service layer, client hooks, Stage 1 UI rewire to use real backend, graceful Azure failure handling | `485eee93b` (2026-05-27) |
| Polish | Calculate button next to tab strip + terraform AZURE_OPENAI_API_VERSION pin | `cdaf6bfdf` (2026-05-28) |

Stage 1 was held locally and never shipped on its own; the data layer in Stage 2 replaced the in-memory mock before the branch ever went up for review.

## Architecture

### Data model — dedicated `ordersFinancialCalculations` collection

Decided unanimously by Engineer + Tech Lead + Principal Engineer during planning. The legacy patient-scoped flow writes to `patients.financialCalculationsHistory[]`, but the new calculator is tied to a specific order and the PO confirmed (2026-05-20) that history should be per-order, not per-patient.

A separate top-level Mongoose collection was preferred over embedding under `neworders` / `maintenanceorders` for three reasons:

1. The order collections (`neworders`, `maintenanceorders`, `leadorders`) do not have Mongoose schemas — they are read and written by the raw MongoDB driver. Retrofitting Mongoose onto them was out of scope.
2. A dedicated collection gets full Mongoose schema validation on every insert.
3. The collection scales independently of order documents; insert load is decoupled from order updates.

Foreign-key fields `orderId: ObjectId` and `orderCategory: 'neworders' | 'maintenanceorders'` link each calculation back to its source order. The two fields are always passed together at the service-layer boundary to avoid the discriminator footgun (a query with only `orderId` could return a calculation belonging to a different order category that happens to share the ObjectId by coincidence; the compound filter makes this impossible).

A compound index `{ orderId: 1, orderCategory: 1, calculationDate: -1 }` covers the only known read pattern (list one order's calculations newest first) and is built automatically by Mongoose on first insert. No migration script is needed.

### Calculation engine — deterministic core, AI for prose explanation only

The pure function `runOrdersFinancialCalculation(inputs)` produces the result and the structured per-step bullets. It is referentially transparent and unit-tested against every legacy seed scenario plus OOP-cap scaling, assistance overflow, and the OOP-exhausted-during-Drug edge case.

The Azure OpenAI call (`generateOrdersFinancialCalculationReason`) is invoked **only** to produce the human-readable `breakdown.reason` paragraph. The bullet arrays are deterministic; the AI never touches them. This separation:

- Keeps the pure function easy to test and reason about.
- Means a degraded AI save still has a fully-populated, accurate breakdown (only the prose paragraph is missing).
- Matches the prompt / model / max-tokens configuration of the legacy patient calculator exactly, so behaviour is predictable.

### Graceful AI degradation

The original Stage 2 plan called for synchronous Azure with no fallback (fail the whole POST if Azure fails). During implementation the user flagged that this was a poor reliability posture for what is essentially a non-essential AI nicety. The handler now wraps the Azure call in its own try-catch:

- Pre-check rejects clearly if `AZURE_OPENAI_ENDPOINT` or `AZURE_OPENAI_API_KEY` are missing (avoids constructing the SDK with empty strings, which produces opaque errors later).
- Any Azure error (missing config, network policy denial, API error, timeout) is caught, logged at warn level with the full error context, and `reason` defaults to `''`.
- The save then proceeds normally with an empty reason.
- The UI's `CalculationBreakdown` conditionally renders the Reason section, so an empty reason produces a clean visual outcome (no broken-looking placeholder).

This matches the behaviour the legacy patient calculator already had — the original Stage 2 plan's claim of "no fallback in legacy" was incorrect. The new flow is now aligned with the proven legacy pattern.

### Coexistence with the legacy patient flow

Hard rule, enforced throughout this PR: **no file under the legacy patient calculator is modified.** The following keep working unchanged:

- `apps/web/pages/api/patients/[id]/financialCalculation.ts`
- `apps/web/services/mongodb/financialCalculation.ts`
- `apps/web/components/Modules/financial-calculator/**`
- `apps/web/utils/api/patient/makeFinancialCalculation.ts`
- `apps/web/utils/hooks/useMakeFinancialCalculation.ts`
- `apps/web/types/financialCalculationDTO.ts`
- `patients.financialCalculationsHistory[]` data in MongoDB

All new code uses an `OrdersFinancialCalculation` / `ordersFinancialCalculation` naming prefix so there is no symbol or import collision with the legacy flow.

### Feature flag gating — belt-and-suspenders

The `ORDER_FINANCIAL_CALCULATION` flag is checked at two levels:

1. **Tab visibility** in `OrderClinicalReviewsPanel`: the Financial Review tab is conditionally inserted into the tabs array.
2. **Route gating** in the order-details `layout.tsx`: the `isFinancialReviewPath` children block only renders when the flag is enabled.

Typing the URL directly with the flag off renders blank. Anyone with order-detail access can use the feature when the flag is on; no additional role gate (the user-facing data is financial estimates, not PHI).

## Changes Overview

- **Files changed**: 38 (33 added or modified, 6 deleted)
- **Lines added / removed**: +4595 / -11 (Stage 1 deletions count net positive because the in-memory mock files were larger than the components that replaced them)
- **Commits**: 4 (one per logical phase — see Commits table below)
- **New MongoDB collection**: `ordersFinancialCalculations` (auto-creates on first insert; no migration script needed)
- **Terraform changes**: `AZURE_OPENAI_API_VERSION` env block added to both web and worker container apps. Requires `terraform apply` against both stage and prod tfvars after merge.

## Files Changed (summary)

### Backend — model, services, API routes

| File | Action | Description |
|---|---|---|
| `apps/web/models/OrdersFinancialCalculation.ts` | Create | Mongoose model with sub-schemas for `inputs`, `result`, `breakdown` (`_id: false` on each), compound index, hot-reload-safe export. |
| `apps/web/models/OrdersFinancialCalculation.test.ts` | Create | 15 schema-validation tests (required fields, enum, numeric ranges, defaults, sub-document _id suppression, index registration). |
| `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.ts` | Create | `runOrdersFinancialCalculation()` pure function (calc engine), `processOrdersFinancialCalculationPart()` per-leg helper, `fmt$` / `fmtPct` formatters, `generateOrdersFinancialCalculationReason()` Azure OpenAI service. |
| `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.test.ts` | Create | 25 tests covering every legacy seed scenario, OOP-cap scaling, assistance overflow, OOP-exhausted-during-Drug skip, env-var pre-check, mocked Azure responses. |
| `apps/web/services/ordersFinancialCalculation/handlers.ts` | Create | Shared GET and POST handlers used by both the `new` and `maintenance` route files. Validates input, composes calc → AI → persist, owns the graceful-degradation try-catch. |
| `apps/web/services/ordersFinancialCalculation/handlers.test.ts` | Create | 18 tests including graceful Azure degradation, validation failures, session fallback. |
| `apps/web/services/mongodb/ordersFinancialCalculation.ts` | Create | Mongoose service: `createOrdersFinancialCalculation()` (converts hex string to ObjectId), `findOrdersFinancialCalculationsByOrder()` (sorted, `.lean()`-ed). |
| `apps/web/services/mongodb/ordersFinancialCalculation.test.ts` | Create | 11 service-layer tests with mocked Mongoose model. |
| `apps/web/app/api/orders/new/[orderId]/financial-calculations/route.ts` | Create | Thin Next.js App Router GET / POST file, delegates to shared handler with `orderCategory: 'neworders'`. |
| `apps/web/app/api/orders/new/[orderId]/financial-calculations/route.test.ts` | Create | 4 smoke tests — auth and correct orderCategory forwarding. |
| `apps/web/app/api/orders/maintenance/[orderId]/financial-calculations/route.ts` | Create | Same shape, delegates with `orderCategory: 'maintenanceorders'`. |
| `apps/web/app/api/orders/maintenance/[orderId]/financial-calculations/route.test.ts` | Create | 4 smoke tests. |
| `apps/web/utils/constants/collections.ts` | Modify | Added `OrdersFinancialCalculations: 'ordersFinancialCalculations'` entry. |

### Client — API helpers and hooks

| File | Action | Description |
|---|---|---|
| `apps/web/utils/api/orders/financialCalculations.ts` | Create | `makeOrdersFinancialCalculation()` and `fetchOrdersFinancialCalculations()` fetch wrappers + the client-facing `OrdersFinancialCalculation` type. |
| `apps/web/utils/api/orders/financialCalculations.test.ts` | Create | 11 tests with mocked `global.fetch`. |
| `apps/web/utils/hooks/useOrdersFinancialCalculations.ts` | Create | List hook with auto-fetch on mount, exposes `{ calculations, isLoading, error, refetch }`. |
| `apps/web/utils/hooks/useOrdersFinancialCalculations.test.tsx` | Create | 6 hook tests using `@testing-library/react` `renderHook`. |
| `apps/web/utils/hooks/useMakeOrdersFinancialCalculation.ts` | Create | Save hook, exposes `{ execCalculation, isLoading, data, error }`. |
| `apps/web/utils/hooks/useMakeOrdersFinancialCalculation.test.tsx` | Create | 5 hook tests. |

### UI — Stage 1 UI rewired for the real backend, plus Calculate button

| File | Action | Description |
|---|---|---|
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.tsx` | Modify | Swapped the in-memory context for `useOrdersFinancialCalculations`. Added loading and error states. Per-order scoping now enforced server-side; the Stage 1 orderId-remap shim is gone. |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.tsx` | Modify | Swapped the in-memory context for `useMakeOrdersFinancialCalculation`. Live-estimate sidebar dropped pending PO direction. Save button shows "Saving…" while in flight; error message rendered inline; form stays open on error so user can retry. |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.tsx` | Modify | Prop type switched to `OrdersFinancialCalculation`. `calculationDate` formatted from ISO at render time. Copy-summary action now joins from `breakdown.summary`. |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.tsx` | Modify | Renders `breakdown.drug`, `breakdown.admin`, `breakdown.summary` as bullet sections, and `breakdown.reason` as a paragraph **only when non-empty** (graceful degradation visual). |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.tsx` | Modify | Added `showCalculateAction` boolean and a Calculate button next to the existing "New Review" button. Mirrors the playground design. |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.test.tsx` | Modify | Added 4 tests for the Calculate button (visibility, gating, click behaviour). |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.test.tsx` | Modify | Updated fixtures to the new schema shape; added tests for breakdown bullet rendering and the conditional Reason section. |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.test.tsx` | Modify | Updated fixtures and copy-summary assertions. |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.test.tsx` | Modify | Replaced context mock with hook mock; dropped live-estimate assertions. |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.test.tsx` | Modify | Replaced context mock with hook mock; added loading and error state tests. |

### UI — deletions (Stage 1 in-memory mock no longer needed)

| File | Action |
|---|---|
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.tsx` | Delete |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.test.tsx` | Delete |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/seed.ts` | Delete |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.ts` | Delete (server-side replacement lives in services/) |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.test.tsx` | Delete |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/layout.tsx` | Delete (no provider needed anymore) |

### Infrastructure

| File | Action | Description |
|---|---|---|
| `terraform/main.tf` | Modify | Added `AZURE_OPENAI_API_VERSION = "2024-08-01-preview"` env block to both container app definitions (web and worker). Matches the existing literal pattern for cross-environment constants. |
| `terraform/docs/ContainerAppSetup.md` | Modify | Documented the new env var. Added two rows that were already missing from the docs (`AZURE_OPENAI_TINY_DEPLOYMENT_NAME`) and corrected the source label for `AZURE_OPENAI_DEPLOYMENT_NAME` from "Key Vault" to "Terraform literal". |

## What Changed and Why

### Stage 1 was held locally on purpose

The first Stage 1 commit (`cceefaf00`) shipped a working UI against an in-memory seed. The plan explicitly forbade pushing or opening a PR with mock data in the branch. This was the right call — it gave us a complete UX to demo to the PO and validate the form layout, expand-in-place behaviour, copy-summary, and feature-flag gating without committing to a persistence design that had not yet been agreed. The data-layer decision was unblocked by a PO conversation on 2026-05-20 (per-order, not per-patient; same nine inputs as the legacy calculator; immutable saves).

### Why a new collection rather than embedding under the order

See **Architecture → Data model** above. Short version: the order collections do not have Mongoose schemas, retrofitting was out of scope, and a dedicated collection scales independently of order updates while gaining full Mongoose validation.

### Why graceful AI degradation

The original Stage 2 plan called for synchronous Azure with no fallback. During testing, the local dev environment surfaced two real failure modes: missing API key, and a `403 Public access is disabled` from APIM when reaching the production Azure OpenAI from outside the VNet. Either would have killed the POST entirely under the no-fallback design. The user (correctly) pushed back: AI prose is a nicety, not a correctness requirement, and losing the whole save because the prose generator is down is the wrong reliability posture. The handler now logs and continues. The legacy patient calculator had already adopted this same pattern (the original Stage 2 plan misread its source as "no fallback"). End state: identical resilience posture across both flows.

### Why bundle the terraform fix into this PR

While debugging the Azure 403 above, the user asked which env vars our code reads versus what is set in terraform. The cross-check surfaced an 11-month-old gap: every AI service in the repo reads `process.env.AZURE_OPENAI_API_VERSION` with a hardcoded code-side fallback of `'2024-08-01-preview'`, but the variable has never appeared anywhere in `terraform/main.tf`. The legacy patient calculator inherited the same omission. The fix is two lines per container-app block — trivially small — and bundling it here means MLID-2307 ships with the full set of env vars its code expects, rather than relying on a code default that someone could change without noticing the prod implication. Anatoliy Kolodkin introduced the original code reference (commit `96f52b3e5`, 2025-06-27, "Updates API to use Azure OpenAI integration") and is the natural reviewer for the infra-side change.

### Why drop the live-estimate sidebar

Stage 1 had a real-time client-side estimate as the user typed (computed by the now-deleted client `calculate.ts`). Stage 2 dropped it because the deterministic calc moved server-side and round-tripping the API on every keystroke is heavy-handed. The PO never validated the live-estimate feature — it was a UI designer proposal, not a written specification. If the PO does want it back later, a follow-up ticket can re-introduce a client-side calc that mirrors the server function. Not blocking for this PR.

## Commits

| Hash | Message |
|---|---|
| `cceefaf00` | `[MLID-2307] - feat(financial-review): Stage 1 — Order Details Financial Calculator UI (mock data, do not ship)` |
| `8e9007de6` | `[MLID-2307] - chore(merge): merge develop into feature/MLID-2307-financial-review` |
| `485eee93b` | `[MLID-2307] - feat(financial-review): Stage 2 — order-scoped data layer with Mongoose, REST API, and graceful AI degradation` |
| `cdaf6bfdf` | `[MLID-2307] - feat(financial-review): add Calculate button and codify AZURE_OPENAI_API_VERSION in terraform` |

## Test Plan

### Automated (all passing on this branch)

- [x] 184 tests across 15 suites covering the full MLID-2307 surface:
  - Model: 15 tests (schema validation, defaults, sub-document _id suppression, index registration)
  - Pure calculation: 22 tests (every legacy seed scenario, OOP-cap variants, assistance overflow)
  - AI reason generator: 12 tests (env-missing pre-check, prompt verbatim, model fallback, error propagation)
  - Mongo service: 11 tests (create + find chains with mocked model)
  - Handlers: 18 tests (happy path, validation 400s, graceful AI degradation, DB-failure 500, session fallback)
  - API route smoke: 8 tests (auth 401, correct orderCategory forwarding)
  - Client API helpers: 11 tests (URL construction, request shape, error handling)
  - Client hooks: 11 tests (mount fetch, loading, error, refetch)
  - UI components: 38 tests (CalculationBreakdown, CalculationCard, FinancialReviewIndex, Calculator)
  - Panel: 10 tests (tab visibility, active state, tab routing, Calculate button visibility and click)
- [x] TypeScript strict pass (`npm run types:check`)
- [x] Lint clean on all new and modified files
- [x] Coverage on new files above the 80% threshold (model and pure function at 100%)

### Manual verification (already exercised end-to-end)

Verified against a real test order (Alex Test, Actemra, neworders), 2026-05-28:

1. ✅ Financial Review tab appears in the panel header when `ORDER_FINANCIAL_CALCULATION` is enabled.
2. ✅ Empty state renders ("No financial calculations yet") with both the central CTA and the new Calculate button in the tab header.
3. ✅ Calculator form renders, accepts numeric input, masks non-numeric characters.
4. ✅ Save button shows "Saving…" during the request, disables until response.
5. ✅ POST returns 200 with the saved document; UI navigates back to the list.
6. ✅ Saved row appears at top with **Latest** chip and total `$15.00` (matches deterministic expected value for the S2 input set).
7. ✅ Expand the row → all four sections render (Inputs grid, Drug bullets, Admin bullets, Summary bullets, **Reason** paragraph from Azure).
8. ✅ Copy-summary action copies the structured summary; check icon flashes for 1.5 s.
9. ✅ Hard refresh — saved row still there (real persistence, not in-memory).
10. ✅ Three test saves end-to-end:
    - Save 1 (no Azure key): document landed with `breakdown.reason: ""`; warn log captured the error.
    - Save 2 (Azure 403 from APIM private-endpoint policy): document landed with `breakdown.reason: ""`; warn log captured the full Azure error including `policy-id: ThrowExceptionDueToTrafficDenied`.
    - Save 3 (Azure reachable after firewall allowlist): document landed with `breakdown.reason` populated by a real gpt-5-nano paragraph that correctly references all input values and calculation steps.
11. ✅ MongoDB document shape matches design: `orderId` as ObjectId, `orderCategory: 'neworders'`, structured `inputs` / `result` / `breakdown`, audit fields.
12. ✅ Compound index `{ orderId: 1, orderCategory: 1, calculationDate: -1 }` auto-built on first insert — no migration script needed.
13. ✅ Feature flag off → tab disappears from panel and direct URL renders blank.

The maintenance route was not exercised manually but uses the identical shared handler with a hardcoded `orderCategory: 'maintenanceorders'` value; route smoke tests cover the auth and forwarding behaviour.

### Reviewer-side verification suggestions

- Toggle the `ORDER_FINANCIAL_CALCULATION` feature flag in `/admin/feature-flags` and confirm the tab appears and disappears, and that the URL renders blank when the flag is off.
- Save a calculation on any new order and confirm: (a) saved row appears with all sections populated, (b) refresh preserves it, (c) a second save sorts newest-first and the Latest chip moves to the new entry.
- Sanity-check the deterministic outputs by trying the S1–S4 scenarios documented inline in the testing checklist: OOP-already-met → all zeros; partial deductible → drug $15 admin $0; no-deductible-no-coinsurance → all zeros; OOP-cap-exhausted-during-Drug → drug cap, Admin skipped.
- Confirm the Azure-down behaviour by temporarily breaking `AZURE_OPENAI_API_KEY` in `.env.local` and saving — expect the same 200 response, `breakdown.reason: ""` in the persisted document, and the Reason section absent from the expanded row.

### Post-merge

- [ ] Run `terraform apply -var-file=environments/stage.tfvars` against terraform to apply the `AZURE_OPENAI_API_VERSION` env var addition to stage container apps. Verify in the Azure portal.
- [ ] Then `terraform apply -var-file=environments/prod.tfvars` for prod. Coordinate timing with the team since the env-var change will trigger a rolling container-app revision.
- [ ] Enable `ORDER_FINANCIAL_CALCULATION` in stage `/admin/feature-flags` for QA exercise.
- [ ] After QA sign-off, enable the flag in prod.

## Reviewers

- **Tech Lead** — full feature review, schema decisions, REST contract.
- **Anatoliy Kolodkin** — recommended for the `terraform/main.tf` and `terraform/docs/ContainerAppSetup.md` portion of the diff. He introduced the original `process.env.AZURE_OPENAI_API_VERSION` reference back in commit `96f52b3e5` (2025-06-27); this PR codifies the matching infrastructure entry that was missing from terraform for the past 11 months.

## Out of Scope (explicit deferrals)

- **Edit / delete saved calculations**: PO confirmed immutability for v1; no `editedAt` / `deletedAt` fields on the schema. Add later if a workflow need surfaces.
- **Standalone calculation detail page**: unreachable in the playground UX; out of scope.
- **Live-estimate sidebar in the Calculator**: dropped in Stage 2; if PO requests it back, a follow-up ticket can re-introduce a client-side calc that mirrors the server function.
- **Author-filter index** on `ordersFinancialCalculations`: no UX need surfaced yet; easy to add later.
- **Sunset of the legacy `/patient/[id]/financial-calculator` UI**: separate ticket.
- **Auto-fill from upstream sources** (deductible / OOP from `patient_insurances`, drug cost from drug catalog): separate ticket.

## Jira

- [MLID-2307](https://localinfusion.atlassian.net/browse/MLID-2307) — primary tracking ticket. Was originally scoped as a single piece of work; split internally into Stage 1 (UI) and Stage 2 (data layer) during implementation because the data-layer design required a PO conversation that was not on the original ticket's specification.
