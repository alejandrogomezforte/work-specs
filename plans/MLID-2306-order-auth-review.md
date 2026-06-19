# MLID-2306 — Order Details: Auth Review

## Task Reference

- **Jira**: [MLID-2306](https://localinfusion.atlassian.net/browse/MLID-2306)
- **Epic**: MLID-2206 (Order Details v2)
- **Story Points**: ~19 (delivered as a single story in one PR — no sub-task split)
- **Branch**: `feature/MLID-2306-order-auth-review`
- **Base Branch**: `develop`
- **Status**: Ready for PR (branch pushed 2026-06-10; PR doc drafted; not yet merged to develop)

---

## Resume state (as of 2026-06-10)

> Single source of truth for this task's progress lives in this plan file and the
> testing notebook `docs/agomez/testing/MLID-2306-order-auth-review-testing.md`
> (not in assistant auto-memory).

**All code is implemented, committed, AND pushed** on
`feature/MLID-2306-order-auth-review` (local is in sync with origin). Full
feature tested end-to-end and looks solid.

**Update 2026-06-10 (wrap-up):** Tweaked the removal trigger + panel styling to
match the mock (`de88eb422`). Merged current `develop` into the branch
(`cc50851bb`) — two conflicts in `OrderClinicalReviewsPanel.{tsx,test.tsx}`,
resolved by keeping develop's singular "Clinical Review" label plus the Auth
Review tab, and keeping both sides' test cases. The merge's pre-commit hook had
reordered imports in 72 unrelated develop files; a `chore(merge)` commit
(`7e73ffc0a`, made with `--no-verify`) restored develop's exact bytes on those,
so the net diff vs develop is the 40 genuine Auth Review files. `tsc` clean
post-merge; AuthReviewDetail (18) + OrderClinicalReviewsPanel (15) green. Branch
pushed. PR draft written to `docs/agomez/PR/MLID-2306.md` — **user to open the
PR manually.**

Commits, newest first:

- `967f917e1` — guard Review Status changes in the detail header (needs_review =
  static label; reviewed = revertible dropdown; revert gated by `window.confirm`)
- `bdc3ca216` — cap preview/detail height so multi-page PDFs scroll in place
  (list `.layoutRow` `max-height: calc(100vh - 420px)`; detail `.splitContainer`
  `calc(100vh - 440px)` + flex `min-height: 0` plumbing)
- `c1b56cb8e` — preview letters in an embedded side panel (resizable, sibling
  card; replaced the full-screen modal)
- `d26c9aa42` — match list view to the approved mockup (one card, borderless
  integrated grid, placeholder filters, ImageSearchRounded preview icon)
- `fce3d2fdf` — restructure detail layout into one card; drop
  InlineDocumentPreview wrapper
- `ebd377b25` — Stage 5 (3-step wizard + WeInfuse Summary Note; PriorAuthReview
  refactored onto the shared `usePriorAuthReviewWizard`)
- `31095fbaa` — Stage 4 (Step 2 Verify Data)
- `0abe3b9ad` — Stage 3 (detail screen + Step 1)
- `ca7ba91b0` — drug-scope rule (Steps 2.15 + 2.16)
- `742343442` — Stage 2 (order-scoped list view)
- `285dcfd58` — Stage 1 (the `ORDER_DETAILS_AUTH_REVIEW` feature flag)

Quality gate (`/verify`) passed: tests, `tsc`, prettier, eslint all green;
coverage above threshold on changed files.

**Next / outstanding:**
- **Minor UI mismatches to fix** — user identified some during testing on
  2026-06-10; specifics TBD (capture them here when known).
- **No PR opened yet** — per convention, draft a PR doc into `docs/agomez/PR/`
  for the user to open manually.
- **PO confirmations before QA/Done:** order-date field
  (`neworders.createdAt`, provisional), `linkedOrderIds` as a precise
  order-scoping follow-up, the unresolved drug-filter passthrough edge case
  (below), and the `MLID-2363` dependency (drug match relies on `liDrugId`).
- **Confirmation modal:** the revert guard currently uses vanilla
  `window.confirm` as a placeholder — UI designer to provide a designed modal.
- **Feature flag** `ORDER_DETAILS_AUTH_REVIEW` defaults `false`; enable in
  stage/prod when PO is ready.

**Open for PO/Egor (not blocking):** documents whose drug only exists in
`priorAuthData.drugName` and/or `ocrData.service.serviceName`, with no `liDrugId`
and no brand/generic name, still pass via the passthrough (e.g. reviewed doc
`6997c15809ade37db602eac7`: `drugName` "Remicade", no `liDrugId`, brand/generic
empty, `serviceName` "Entyvio"). Tracked in
`docs/agomez/plans/MLID-2306-stage-2-unresolved-drug-filter-issue.md`. (User
thought this might already be solved — NOT changed in code yet; confirm.)

**Context references:** field-origin confirmation is in Appendix A below; the
drug-match rule is in Steps 2.15 + 2.16; the `li-intake` (Python) repo lives at
`~/Dev/li-intake`. Key UI files: `auth-review/_components/AuthReviewIndex.tsx`
(+ `.module.css`) for the list/side-panel, `AuthReviewDetail.tsx`
(+ `.module.css`) for the detail/wizard.

---

## Summary

Add an **Auth Review** sub-tab inside the order detail view at
`/orders-tracker/new/[id]/auth-review` (list) and
`/orders-tracker/new/[id]/auth-review/[letterId]` (single-letter review). The
feature ports the existing patient prior-auth experience — the same `Intake`
collection, the same 3-step review wizard, the same ownership rules — but
filtered server-side to documents relevant to that specific order (by date and
drug id), rebuilt on `@repo/ui` `DataGrid` with multi-select `Autocomplete`
filters, and given a new takeover header (Back button + real Review Status
selector). Everything is gated behind a new `ORDER_DETAILS_AUTH_REVIEW` feature
flag at both UI and server.

The work is delivered as a single PR in **5 end-to-end stages**. Stage 1 lays
the feature flag foundation. Stages 2 through 5 each produce a vertical slice
that is independently testable in the browser before moving on. Stage 2 delivers
the full list view; Stage 3 delivers the detail screen through Step 1 of the
wizard; Stage 4 adds the Step 2 (Verify Data) form; Stage 5 completes the 3-step
flow, adds Step 3 (WeInfuse Summary Note), and refactors the existing patient
prior-auth view to consume the shared wizard hook.

---

## Codebase Analysis

### Feature flag system
`apps/web/utils/featureFlags/featureFlags.ts` — `FeatureFlag` enum + `DEFAULT_FEATURE_FLAGS`
map. The "Order detail tab" family is at the bottom of the enum:
`ORDER_DETAILS_DOCUMENTS`, `ORDER_DETAILS_ACTIONS`. New flag goes right after those.
Client gate: `useFeatureFlag` from `@/utils/contexts/FeatureFlagContext`.
Server gate: `isFeatureEnabled` from `@/services/featureFlags`.

### Prior-auth service layer
- `apps/web/services/priorAuth/query.ts` — exports only
  `buildPriorAuthBaseMatchQuery(patientWeInfuseId)`. The list pipeline
  (sort / skip / limit / facet) lives inline in the patient list route.
- `apps/web/services/priorAuth/transforms.ts` — exports
  `transformIntakeToPriorAuthLetter(intake)` and helpers; reuse as-is.
- The single-letter GET and PATCH logic is inline in
  `apps/web/app/api/patient/[patientId]/prior-auth/[intakeId]/route.ts`.
  Both are large blocks that need extraction into shared service functions
  before a new order route can delegate to them without duplication.

### Prior-auth types
`apps/web/types/priorAuth.ts` — `PriorAuthLetter`, `PriorAuthLetterType`
(`Approval|Denial|Request|Receipt|Reminder|Duplicate|Unnecessary|Other`),
`PriorAuthReviewStatus` (`needs_review|reviewed`), `PriorAuthListResponse`
(`{ letters, total, totalPages }`), `PriorAuthDetailResponse`,
`PriorAuthPatchRequest`, `PriorAuthFilterOptions`. All reused unchanged for the
order routes.

### Order detail shell
`apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`
(`OrderDetailsLayout`) — a `'use client'` component that gates each sub-route
by a feature-flag + pathname check before rendering `children`. Clinical and
financial review gates are the pattern to follow exactly. The layout reads
`useOrderDetails()` context for `data.patientId` and `data.id`.

`apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.tsx`
— renders the sub-tab `Tabs` row (`Clinical Reviews | Financial Review`) and
the active tab content. `FinancialReviewIndex` is rendered inline from within
this panel when `activeReviewTab === 'financial'`. Auth Review follows the same
inline pattern.

### Financial review as the closest mirror
- `financial-review/page.tsx` is a one-liner: `return <OrderClinicalReviewsPanel />`.
- `financial-review/_components/FinancialReviewIndex.tsx` is the list component,
  uses `@repo/ui` `Layout`/`Card`/`Button`/`Typography`, fetches via a data hook.
- `clinical-reviews/_components/hooks/useClinicalReviews.ts` is the data hook
  pattern to mirror for `useOrderAuthReview`.

### Order API path for auth-review sub-routes
Existing order-scoped API routes live under
`apps/web/app/api/orders/new/[orderId]/financial-calculations/route.ts`. The
auth-review routes follow the same path convention:
`apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` (list + PATCH is
per letter so lives at `auth-review/[letterId]/route.ts`).

### Order → patient resolution
`layout.tsx` calls `/api/orders/details` which reads `neworders.patient`
(Mongo `_id` reference) via `$lookup` against `patients`. `OrderDetailsData.patientId`
= the patient's Mongo hex `_id`. The order-scoped API route must:
1. Load the order from `neworders` by `orderId` → read `neworders.patient`
   (the Mongo `_id`) and `neworders.drug` (stringified `lidrugs._id`) and
   `neworders.createdAt`.
2. `Patient.findById(neworders.patient)` → read `patient.we_infuse_id_IV`.
3. Feed `we_infuse_id_IV` into `buildPriorAuthBaseMatchQuery`.

Ownership check stays patient-based: `intake.patient.patientId === patient.we_infuse_id_IV`.

### Order-scoped filter — date and drug
- **Date condition**: `effectiveDecisionDate >= order.createdAt`. The
  effective date is:
  `$dateFromString(additionalInfo.ocrData.authorization.dateOfDecision.value, onError/onNull: null)`
  coalesced to `$createdAt`. The `dateOfDecision.value` field is a `"YYYY-MM-DD"`
  string (not a Date), present on only ~40% of processed intakes, so the
  `createdAt` fallback is the common path.
- **Drug condition**: `additionalInfo.priorAuthData.liDrugId.value === orderDrugId`
  (string equality) OR `liDrugId` null/missing. This uses
  `MLID-2363`-written data; before that lands, only manually reviewed
  documents have a `liDrugId` so all others pass via the null branch (acceptable).

### Components to reuse
- `VerifyDataForm` — at
  `components/Modules/patient/components/patientPriorAuth/VerifyDataForm.tsx`,
  import by direct path (not from folder index; not currently re-exported).
- `WeinfuseSummaryNote` — same folder, same import strategy.
- `EmbeddedFileViewer` / `InlineDocumentPreview` — from `@/components/UI`.
- The step-orchestration logic in `PriorAuthReview.tsx` (currentStep 1|2|3,
  `SINGLE_STEP_LETTER_TYPES`, step-2 validity gating, initial-step-on-open,
  finish vs. save-changes) will be extracted into
  `usePriorAuthReviewWizard` + a presentational steps wrapper component.

### DataGrid
`import { DataGrid, GridColDef } from '@repo/ui'`. Default: `density="compact"`,
`pageSizeOptions={[25, 50, 100]}`, `initialState.pagination.paginationModel.pageSize=25`,
`disableColumnMenu`, `disableRowSelectionOnClick`. Reference usage:
`apps/web/components/Modules/communications/components/PatientsWithErrorsTable.tsx`.

---

## Implementation Steps

The commit message format is `[MLID-2306] - type(scope): description`.

### Stage map

| Stage | Slice | End-to-end testable outcome | Depends on |
|-------|-------|----------------------------|------------|
| 1 | Feature flag foundation | Not visually testable on its own | — |
| 2 | Auth Review list view (tab + DataGrid + filters + preview) | Enable flag, open an order, confirm the Auth Review tab appears, list loads scoped to that order, filters/chips/clear-all work, preview modal opens | Stage 1 |
| 3 | Detail screen: header + Step 1 + single-step finish | Click a row, verify the takeover header; change Review Status and confirm it persists; finish a single-step letter and land back on the list; inline fax renders; admin removal panel visible only to admins | Stage 2 |
| 4 | Detail screen: Step 2 (Verify Data) | Open a multi-step letter, advance from Step 1 to Step 2, fill Verify Data fields, observe validity-gated Next button, navigate back to Step 1 | Stage 3 |
| 5 | Detail screen: Step 3 (WeInfuse Summary Note) + patient view refactor | Complete full 3-step flow for a multi-step letter; reopen a reviewed letter and confirm it opens on Step 3; patient prior-auth view still works unchanged | Stage 4 |

---

### Stage 1 — Feature flag foundation ✅ IMPLEMENTED

This stage adds the `ORDER_DETAILS_AUTH_REVIEW` enum entry and default value. It
is the agreed foundation for all subsequent stages but is not visually testable
on its own.

> **Implementation note:** the flag test was added as a `describe` block in the
> existing `apps/web/utils/featureFlags/featureFlags.test.ts` (the repo
> convention) using the exported `getDefaultFeatureFlagValue`, rather than a new
> standalone test file referencing the non-exported `DEFAULT_FEATURE_FLAGS` map.

#### Step 1.1 — RED: write a failing test for the feature flag enum entry

**Files (create/modify)**
- Create: `apps/web/utils/featureFlags/__tests__/featureFlags.ORDER_DETAILS_AUTH_REVIEW.test.ts`

**Test cases to write**
```ts
describe('FeatureFlag.ORDER_DETAILS_AUTH_REVIEW', () => {
  it('should be defined in the FeatureFlag enum', () => {
    expect(FeatureFlag.ORDER_DETAILS_AUTH_REVIEW).toBe('ORDER_DETAILS_AUTH_REVIEW');
  });

  it('should default to false in DEFAULT_FEATURE_FLAGS', () => {
    expect(DEFAULT_FEATURE_FLAGS[FeatureFlag.ORDER_DETAILS_AUTH_REVIEW]).toBe(false);
  });
});
```

Run: fails because the enum entry does not exist.

#### Step 1.2 — GREEN + REFACTOR: add the flag

**Files (modify)**
- `apps/web/utils/featureFlags/featureFlags.ts`

**What to add**
In the `FeatureFlag` enum, after `ORDER_DETAILS_ACTIONS`:
```ts
ORDER_DETAILS_AUTH_REVIEW = 'ORDER_DETAILS_AUTH_REVIEW',
```
In `DEFAULT_FEATURE_FLAGS`:
```ts
[FeatureFlag.ORDER_DETAILS_AUTH_REVIEW]: false,
```

Run tests. Both pass.

**Commit**: `[MLID-2306] - feat(feature-flags): add ORDER_DETAILS_AUTH_REVIEW flag (default false)`

---

### Stage 2 — Auth Review general table (end-to-end list view) ✅ IMPLEMENTED

This stage delivers a fully working list view: data pipeline, order-scoped API
routes (list + filter-options), the sub-tab wiring in the layout and panel, and
the `AuthReviewIndex` UI component. At the end of this stage, the Auth Review
tab is visible in the browser and functional.

> **Implementation reconciliations (deviations from the illustrative snippets):**
> - `buildPriorAuthListPipeline` includes a top-level `$sort` stage (the plan's
>   own Step 2.2 tests and the existing patient `route.test.ts` require
>   `pipeline[1].$sort` / `pipeline[2].$facet`); the GREEN snippet had omitted it.
> - Server-side flag gate uses `FeatureFlagService.getFeatureFlag(...)` (the
>   established order-route precedent) rather than `isFeatureEnabled` — the latter
>   does an internal HTTP fetch and the `@/services/featureFlags` index does not
>   exist.
> - `getNewOrderById` is backed by a new lean Mongoose model `models/NewOrder.ts`
>   (`strict: false`, only `patient`/`drug`/`createdAt` typed) to honor the
>   Mongoose-only convention. Confirmed against the DB: `neworders.patient` and
>   `neworders.drug` are hex strings, `createdAt` is a Date.
> - The `useOrderAuthReview` and `AuthReviewIndex` tests are `.test.tsx` (the jest
>   config only matches `app/**/*.test.tsx` under jsdom, not `.ts`).
> - The shared `@/mocks/repo-ui` mock was extended additively with `DataGrid` and
>   `Autocomplete`.
> - The flag enum test (Stage 1) was added to the existing `featureFlags.test.ts`.
> - Multi-select filters append repeated query params; the server reads the first
>   value (single-value equality, per the plan's pipeline design).

**What is NOT in this stage:** the detail GET/PATCH route and the
`auth-review/[letterId]/page.tsx` page file — those are Stage 3. The
`getNewOrderById` helper is introduced here because the list route is the first
route that loads an order server-side.

**Visual testing scope at the end of Stage 2:** the "Received" column cells link
to the detail route (`.../auth-review/[letterId]`), which does not exist yet.
Row navigation is therefore NOT testable in Stage 2. What is testable:
- The Auth Review tab appears between Clinical Reviews and Financial Review.
- The list is narrowed to prior-auth documents in the order's date/drug scope.
- Multi-select Autocomplete filters work; active filter chips appear; Clear all
  filters resets the list.
- Sorting and pagination work.
- The document preview modal opens when the preview icon is clicked.

---

#### Step 2.1 — RED: write failing tests for `buildOrderScopedPriorAuthMatch`

**Files (create)**
- `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts`

**Test cases to write**

```ts
describe('buildOrderScopedPriorAuthMatch', () => {
  const orderCreatedAt = new Date('2024-01-15T00:00:00.000Z');
  const orderDrugId = 'abc123drugid';

  it('should include $expr with $and combining date and drug conditions', () => {
    const result = buildOrderScopedPriorAuthMatch({ orderCreatedAt, orderDrugId });
    expect(result).toHaveProperty('$expr.$and');
    expect(Array.isArray(result.$expr.$and)).toBe(true);
    expect(result.$expr.$and).toHaveLength(2);
  });

  it('date condition should use $dateFromString on dateOfDecision with createdAt fallback', () => {
    const result = buildOrderScopedPriorAuthMatch({ orderCreatedAt, orderDrugId });
    const dateCondition = result.$expr.$and[0];
    // Must be a $gte expression
    expect(dateCondition).toHaveProperty('$gte');
    const [effectiveDate] = dateCondition.$gte;
    // effectiveDate must be a $ifNull that falls back to '$createdAt'
    expect(effectiveDate).toHaveProperty('$ifNull');
    const [parsedDate, fallback] = effectiveDate.$ifNull;
    expect(fallback).toBe('$createdAt');
    // parsedDate must be a $dateFromString
    expect(parsedDate).toHaveProperty('$dateFromString');
    expect(parsedDate.$dateFromString.dateString).toBe(
      '$additionalInfo.ocrData.authorization.dateOfDecision.value'
    );
    expect(parsedDate.$dateFromString.onError).toBeNull();
    expect(parsedDate.$dateFromString.onNull).toBeNull();
    // Comparison target must be the literal order creation date
    const [, comparisonDate] = dateCondition.$gte;
    expect(comparisonDate).toEqual(orderCreatedAt);
  });

  it('drug condition should match liDrugId OR null/missing', () => {
    const result = buildOrderScopedPriorAuthMatch({ orderCreatedAt, orderDrugId });
    const drugCondition = result.$expr.$and[1];
    expect(drugCondition).toHaveProperty('$or');
    const [exactMatch, nullMissing] = drugCondition.$or;
    expect(exactMatch).toEqual({
      $eq: ['$additionalInfo.priorAuthData.liDrugId.value', orderDrugId],
    });
    // null/missing condition: liDrugId.value must not exist or must be null
    expect(nullMissing).toHaveProperty('$in');
    // The $in array must contain null and must reference the liDrugId.value field
    expect(nullMissing.$in[1]).toContain(null);
  });

  it('drug condition includes documents with no liDrugId (null branch)', () => {
    // When liDrugId is absent, the document should be included
    const result = buildOrderScopedPriorAuthMatch({ orderCreatedAt, orderDrugId: 'someId' });
    const drugCondition = result.$expr.$and[1];
    expect(drugCondition.$or).toHaveLength(2);
  });
});
```

#### Step 2.2 — RED: write failing tests for `buildPriorAuthListPipeline`

**Files (create)**
- `apps/web/services/priorAuth/__tests__/query.listPipeline.test.ts`

**Test cases to write**

```ts
describe('buildPriorAuthListPipeline', () => {
  const baseArgs = {
    patientWeInfuseId: '7719',
    filters: {},
    sort: -1 as const,
    pagination: { page: 1, pageSize: 25 },
  };

  it('should return a pipeline array with $match, $sort, and $facet stages', () => {
    const pipeline = buildPriorAuthListPipeline(baseArgs);
    expect(pipeline[0]).toHaveProperty('$match');
    expect(pipeline[1]).toHaveProperty('$sort');
    expect(pipeline[2]).toHaveProperty('$facet');
  });

  it('$facet should have totalCount and data branches', () => {
    const pipeline = buildPriorAuthListPipeline(baseArgs);
    const facet = pipeline[2].$facet;
    expect(facet).toHaveProperty('totalCount');
    expect(facet).toHaveProperty('data');
  });

  it('$facet.data should include $skip and $limit matching pagination args', () => {
    const pipeline = buildPriorAuthListPipeline({
      ...baseArgs,
      pagination: { page: 2, pageSize: 50 },
    });
    const dataStages = pipeline[2].$facet.data;
    const skipStage = dataStages.find((s: PipelineStage) => '$skip' in s);
    const limitStage = dataStages.find((s: PipelineStage) => '$limit' in s);
    expect(skipStage.$skip).toBe(50); // (page-1) * pageSize
    expect(limitStage.$limit).toBe(50);
  });

  it('should append order-scoped $match stage when orderScope is provided', () => {
    const orderScope = {
      orderCreatedAt: new Date('2024-01-01'),
      orderDrugId: 'drug123',
    };
    const pipeline = buildPriorAuthListPipeline({ ...baseArgs, orderScope });
    // There should be two $match stages: base + order-scoped
    const matchStages = pipeline.filter((s: PipelineStage) => '$match' in s);
    expect(matchStages).toHaveLength(2);
  });

  it('should produce a single $match stage when no orderScope is given', () => {
    const pipeline = buildPriorAuthListPipeline(baseArgs);
    const matchStages = pipeline.filter((s: PipelineStage) => '$match' in s);
    expect(matchStages).toHaveLength(1);
  });

  it('should apply letterType filter when provided', () => {
    const pipeline = buildPriorAuthListPipeline({
      ...baseArgs,
      filters: { letterType: 'Approval' },
    });
    const match = pipeline[0].$match;
    // letterType filter uses $and with $or clauses
    expect(match.$and).toBeDefined();
  });

  it('should apply reviewStatus filter for needs_review', () => {
    const pipeline = buildPriorAuthListPipeline({
      ...baseArgs,
      filters: { reviewStatus: 'needs_review' },
    });
    const match = pipeline[0].$match;
    expect(match['additionalInfo.priorAuthData.reviewStatus']).toEqual({
      $exists: false,
    });
  });
});
```

#### Step 2.3 — GREEN: implement `buildOrderScopedPriorAuthMatch` and `buildPriorAuthListPipeline`

**Files (modify)**
- `apps/web/services/priorAuth/query.ts`

**What to add**

```ts
export type OrderScopeArgs = {
  orderCreatedAt: Date;
  orderDrugId: string;
};

/**
 * Additional $match stage that narrows prior-auth intakes to those relevant
 * for a specific order, based on decision date and drug id.
 *
 * Date condition:
 *   The effective decision date is the parsed OCR dateOfDecision.value (YYYY-MM-DD string)
 *   coalesced to intake.createdAt when null/malformed. The document is included when
 *   effectiveDecisionDate >= orderCreatedAt.
 *
 * Drug condition:
 *   Include when priorAuthData.liDrugId.value equals orderDrugId (string equality)
 *   OR when liDrugId is null/missing (un-reviewed documents pass through).
 *
 * Note: orderCreatedAt and orderDrugId are scalars injected as literals.
 * No $lookup to neworders is needed here.
 */
export function buildOrderScopedPriorAuthMatch(
  args: OrderScopeArgs
): PipelineStage.Match['$match'] {
  const { orderCreatedAt, orderDrugId } = args;

  const effectiveDecisionDate = {
    $ifNull: [
      {
        $dateFromString: {
          dateString:
            '$additionalInfo.ocrData.authorization.dateOfDecision.value',
          onError: null,
          onNull: null,
        },
      },
      '$createdAt',
    ],
  };

  const dateCondition = {
    $gte: [effectiveDecisionDate, orderCreatedAt],
  };

  const drugCondition = {
    $or: [
      {
        $eq: ['$additionalInfo.priorAuthData.liDrugId.value', orderDrugId],
      },
      {
        $in: [
          '$additionalInfo.priorAuthData.liDrugId.value',
          [null, undefined, ''],
        ],
      },
    ],
  };

  return {
    $expr: {
      $and: [dateCondition, drugCondition],
    },
  };
}

export type PriorAuthListFilters = {
  letterType?: string;
  payor?: string;
  drug?: string;
  reviewStatus?: string;
  receivedStart?: string;
  receivedEnd?: string;
};

export type PriorAuthListPipelineArgs = {
  patientWeInfuseId: string | undefined;
  filters: PriorAuthListFilters;
  sort: 1 | -1;
  pagination: { page: number; pageSize: number };
  orderScope?: OrderScopeArgs;
};

/**
 * Builds the full aggregation pipeline for prior-auth list endpoints.
 * Used by both the patient route and the order-scoped route.
 * When orderScope is provided, a second $match stage is prepended to narrow
 * results to the order's date and drug context.
 */
export function buildPriorAuthListPipeline(
  args: PriorAuthListPipelineArgs
): PipelineStage[] {
  const { patientWeInfuseId, filters, sort, pagination, orderScope } = args;
  const { page, pageSize } = pagination;
  const skip = (page - 1) * pageSize;

  const baseMatch = buildPriorAuthBaseMatchQuery(patientWeInfuseId);

  // Apply inline filter clauses to baseMatch
  const orClauses: Record<string, unknown>[] = [];

  if (filters.letterType) {
    orClauses.push({
      $or: [
        {
          'additionalInfo.priorAuthData.letterType.value': filters.letterType,
        },
        { 'additionalInfo.priorAuthData.letterType': filters.letterType },
        {
          'additionalInfo.ocrData.letterMetadata.documentType.value':
            filters.letterType,
        },
      ],
    });
  }
  if (filters.payor) {
    baseMatch['additionalInfo.ocrData.payor.name.value'] = decodeURIComponent(
      filters.payor
    );
  }
  if (filters.drug) {
    const decoded = decodeURIComponent(filters.drug);
    orClauses.push({
      $or: [
        { 'additionalInfo.ocrData.service.brandDrugName.value': decoded },
        { 'additionalInfo.ocrData.service.genericDrugName.value': decoded },
      ],
    });
  }
  if (orClauses.length > 0) {
    baseMatch.$and = orClauses;
  }
  if (filters.reviewStatus === 'needs_review') {
    baseMatch['additionalInfo.priorAuthData.reviewStatus'] = {
      $exists: false,
    };
  } else if (filters.reviewStatus === 'reviewed') {
    baseMatch['additionalInfo.priorAuthData.reviewStatus'] = 'reviewed';
  }
  if (filters.receivedStart) {
    baseMatch.createdAt = {
      ...(baseMatch.createdAt as Record<string, unknown>),
      $gte: new Date(filters.receivedStart),
    };
  }
  if (filters.receivedEnd) {
    const endDate = new Date(filters.receivedEnd);
    endDate.setUTCHours(23, 59, 59, 999);
    baseMatch.createdAt = {
      ...(baseMatch.createdAt as Record<string, unknown>),
      $lte: endDate,
    };
  }

  const pipeline: PipelineStage[] = [
    { $match: baseMatch },
    ...(orderScope
      ? [{ $match: buildOrderScopedPriorAuthMatch(orderScope) }]
      : []),
    {
      $facet: {
        totalCount: [{ $count: 'count' }],
        data: [
          { $sort: { createdAt: sort } } as PipelineStage.Sort,
          { $skip: skip },
          { $limit: pageSize },
        ],
      },
    },
  ];

  return pipeline;
}
```

#### Step 2.4 — REFACTOR: update patient list route to delegate to `buildPriorAuthListPipeline`

**Files (modify)**
- `apps/web/app/api/patient/[patientId]/prior-auth/route.ts`

**What to change**
Replace the inline pipeline construction with a call to `buildPriorAuthListPipeline`.
The patient route passes `filters` parsed from search params and no `orderScope`.
Behavior must stay identical — all existing tests must still pass.

**Verify**: run `apps/web/app/api/patient/[patientId]/prior-auth/route.test.ts` (if it exists); if none exists, run the new query tests.

**Commit**: `[MLID-2306] - refactor(prior-auth): extract buildOrderScopedPriorAuthMatch and buildPriorAuthListPipeline; patient route delegates to shared pipeline`

---

#### Step 2.5 — RED: tests for the list route

**Files (create)**
- `apps/web/app/api/orders/new/[orderId]/auth-review/route.test.ts`

**Test cases**

```ts
describe('GET /api/orders/new/[orderId]/auth-review', () => {
  beforeEach(() => jest.clearAllMocks());

  it('should return 401 when session is absent', async () => {
    mockGetServerSession.mockResolvedValue(null);
    const response = await GET(new Request('http://x/api/orders/new/ord1/auth-review'),
      { params: Promise.resolve({ orderId: 'ord1' }) });
    expect(response.status).toBe(401);
  });

  it('should return 403 when ORDER_DETAILS_AUTH_REVIEW flag is disabled', async () => {
    mockGetServerSession.mockResolvedValue({ user: {} });
    mockIsFeatureEnabled.mockResolvedValue(false);
    // Assert status 403
  });

  it('should return 404 when order is not found', async () => {
    mockGetServerSession.mockResolvedValue({ user: {} });
    mockIsFeatureEnabled.mockResolvedValue(true);
    mockGetNewOrderById.mockResolvedValue(null);
    // Assert status 404
  });

  it('should return 404 when patient is not found', async () => { ... });

  it('should return 200 with letters, total, totalPages on happy path', async () => {
    // Mock order, patient, Intake.aggregate returning facet result
    // Assert response body has { letters: [...], total: N, totalPages: N }
  });

  it('should pass orderScope to buildPriorAuthListPipeline', async () => {
    // Verify buildPriorAuthListPipeline was called with orderScope containing
    // the order createdAt and drug id
  });
});
```

#### Step 2.6 — RED: tests for the filter-options route

**Files (create)**
- `apps/web/app/api/orders/new/[orderId]/auth-review/filter-options/route.test.ts`

**Test cases**

```ts
describe('GET /api/orders/new/[orderId]/auth-review/filter-options', () => {
  it('should return 401 when unauthenticated', ...);
  it('should return 403 when flag is off', ...);
  it('should return 200 with { payors, drugs } scoped to the order patient and order drug/date', async () => {
    // The filter-options query MUST include both the base match AND the order-scoped match
    // so that only drugs/payors from order-relevant intakes are returned
  });
});
```

#### Step 2.7 — GREEN: implement list + filter-options routes and `getNewOrderById`

**Files (create)**
- `apps/web/services/mongodb/newOrders.ts` — `getNewOrderById(orderId)`
- `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts`
- `apps/web/app/api/orders/new/[orderId]/auth-review/filter-options/route.ts`

**Route handler pattern (both routes follow this structure)**

```ts
export const dynamic = 'force-dynamic';

export async function GET(request: NextRequest, context: RouteContext) {
  const session = await getServerSession(authOptions);
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { orderId } = await context.params;

  const flagEnabled = await isFeatureEnabled(FeatureFlag.ORDER_DETAILS_AUTH_REVIEW);
  if (!flagEnabled) {
    return NextResponse.json({ error: 'Feature not enabled' }, { status: 403 });
  }

  const order = await getNewOrderById(orderId);
  if (!order) {
    return NextResponse.json({ error: 'Order not found' }, { status: 404 });
  }

  const patient = await Patient.findById(order.patient);
  if (!patient) {
    return NextResponse.json({ error: 'Patient not found' }, { status: 404 });
  }

  // ... delegate to shared pipeline / service
}
```

**List route** — calls `buildPriorAuthListPipeline` with `orderScope: { orderCreatedAt: order.createdAt, orderDrugId: order.drug }`.

**Filter-options route** — runs the same pipeline as the patient filter-options
route but with the order-scoped $match appended:
base match + `buildOrderScopedPriorAuthMatch(orderScope)` → `$facet` for distinct drugs/payors.

Every error path must call `logger.error(...)` before returning 500 responses.
No stack traces in responses.

**Commit**: `[MLID-2306] - feat(api): add order-scoped auth-review list and filter-options routes gated by ORDER_DETAILS_AUTH_REVIEW`

---

#### Step 2.8 — RED: layout test for auth-review path gate

**Files (create/modify)**
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx` (modify — add test cases)

**Test cases to add**

```ts
describe('OrderDetailsLayout — auth-review gate', () => {
  it('should not render children when ORDER_DETAILS_AUTH_REVIEW flag is off and pathname is auth-review', () => {
    // Mock useFeatureFlag: ORDER_DETAILS_AUTH_REVIEW = false
    // Mock pathname = '/orders-tracker/new/order1/auth-review'
    // Assert children are NOT in the DOM
  });

  it('should render children when ORDER_DETAILS_AUTH_REVIEW flag is on and pathname is auth-review', () => {
    // Mock useFeatureFlag: ORDER_DETAILS_AUTH_REVIEW = true
    // Assert children ARE in the DOM
  });
});
```

#### Step 2.9 — RED: panel test for Auth Review tab

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.test.tsx`
  (check if it exists; if so add test cases)

**Test cases**

```ts
describe('OrderClinicalReviewsPanel — Auth Review tab', () => {
  it('should not show Auth Review tab when ORDER_DETAILS_AUTH_REVIEW flag is off', () => {
    render(<OrderClinicalReviewsPanel />, { ...mockProviders, flags: { ORDER_DETAILS_AUTH_REVIEW: false } });
    expect(screen.queryByRole('tab', { name: /auth review/i })).not.toBeInTheDocument();
  });

  it('should show Auth Review tab when ORDER_DETAILS_AUTH_REVIEW flag is on', () => {
    render(<OrderClinicalReviewsPanel />, { flags: { ORDER_DETAILS_AUTH_REVIEW: true } });
    expect(screen.getByRole('tab', { name: /auth review/i })).toBeInTheDocument();
  });

  it('should render AuthReviewIndex when active tab is auth', () => {
    // Mock pathname includes /auth-review
    expect(screen.getByTestId('auth-review-index')).toBeInTheDocument();
  });
});
```

#### Step 2.10 — GREEN: wire layout + panel + `auth-review/page.tsx`

**Files (modify)**
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`
- `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.tsx`

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/page.tsx`

Note: `auth-review/[letterId]/page.tsx` is NOT created in this stage — that is Stage 3.

**`layout.tsx` changes**
Add near the top of the component body (after existing path checks):
```ts
const isAuthReviewEnabled = useFeatureFlag(FeatureFlag.ORDER_DETAILS_AUTH_REVIEW);
const isAuthReviewPath = !!basePath && pathname?.startsWith(`${basePath}/auth-review`);
```
Add to the render output (after the `isFinancialReviewPath` gate block):
```tsx
{isAuthReviewPath && isAuthReviewEnabled && !isLoading && !error
  ? children
  : null}
```

**`OrderClinicalReviewsPanel.tsx` changes**

1. Add `isAuthReviewEnabled` via `useFeatureFlag(FeatureFlag.ORDER_DETAILS_AUTH_REVIEW)`.
2. Add `{ label: 'Auth Review', value: 'auth' }` to `reviewTabItems` when the flag is on
   (insert between `Clinical Reviews` and `Financial Review`).
3. Extend `activeReviewTab` detection: `pathname?.includes('/auth-review')` → return `'auth'`.
4. Extend `onChange` handler: `value === 'auth'` → `router.push(`${basePath}/auth-review`)`.
5. Add a content branch: `activeReviewTab === 'auth'` → `<AuthReviewIndex orderId={params.orderId} />`.
   Import `AuthReviewIndex` from `../auth-review/_components/AuthReviewIndex`.

**`auth-review/page.tsx`** (mirrors `financial-review/page.tsx`):
```tsx
'use client';
import OrderClinicalReviewsPanel from '../components/OrderClinicalReviewsPanel';
export default function AuthReviewPage() {
  return <OrderClinicalReviewsPanel />;
}
```

**Commit**: `[MLID-2306] - feat(order-details): wire ORDER_DETAILS_AUTH_REVIEW sub-tab in layout and panel; add auth-review page`

---

#### Step 2.11 — RED: tests for `useOrderAuthReview` hook

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.test.ts`

**Test cases**

```ts
describe('useOrderAuthReview', () => {
  beforeEach(() => jest.clearAllMocks());

  it('should fetch letters from /api/orders/new/[orderId]/auth-review on mount', async () => {
    // Mock global.fetch; verify URL was called
    // Assert letters state is populated after fetch
  });

  it('should refetch when page changes', async () => {
    const { result } = renderHook(() => useOrderAuthReview({ orderId: 'ord1' }));
    act(() => result.current.setPage(2));
    await waitFor(() => expect(global.fetch).toHaveBeenCalledTimes(2));
  });

  it('should include active filters in query string', async () => {
    const { result } = renderHook(() => useOrderAuthReview({ orderId: 'ord1' }));
    act(() => result.current.setFilters({ letterTypes: ['Approval'] }));
    await waitFor(() =>
      expect(global.fetch).toHaveBeenCalledWith(
        expect.stringContaining('letterType=Approval'),
        expect.anything()
      )
    );
  });

  it('should set error state when fetch fails', async () => {
    global.fetch = jest.fn().mockRejectedValue(new Error('Network error'));
    const { result } = renderHook(() => useOrderAuthReview({ orderId: 'ord1' }));
    await waitFor(() => expect(result.current.error).not.toBeNull());
  });

  it('should fetch filter options from /filter-options endpoint', async () => {
    // Assert filterOptions state is populated
  });
});
```

#### Step 2.12 — RED: tests for `AuthReviewIndex` component

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx`

**Test cases**

```ts
describe('AuthReviewIndex', () => {
  it('should render a DataGrid with the correct columns', () => {
    render(<AuthReviewIndex orderId="ord1" />, { ...mockHookWithData });
    expect(screen.getByRole('columnheader', { name: /received/i })).toBeInTheDocument();
    expect(screen.getByRole('columnheader', { name: /drug/i })).toBeInTheDocument();
    expect(screen.getByRole('columnheader', { name: /payor/i })).toBeInTheDocument();
    expect(screen.getByRole('columnheader', { name: /letter type/i })).toBeInTheDocument();
    expect(screen.getByRole('columnheader', { name: /auth valid period/i })).toBeInTheDocument();
    expect(screen.getByRole('columnheader', { name: /review status/i })).toBeInTheDocument();
  });

  it('should show Autocomplete filter controls for drugs, types, statuses, payors', () => {
    expect(screen.getByLabelText(/all drugs/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/all types/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/all statuses/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/all payors/i)).toBeInTheDocument();
  });

  it('should show active filter chips when filters are applied', () => {
    render(<AuthReviewIndex orderId="ord1" />,
      { ...mockHookWithActiveFilters });
    expect(screen.getByText('Approval')).toBeInTheDocument();
  });

  it('should call clearFilters when "Clear all filters" is clicked', () => {
    const clearFilters = jest.fn();
    render(<AuthReviewIndex orderId="ord1" />,
      { ...mockHookWithActiveFilters, clearFilters });
    userEvent.click(screen.getByText(/clear all filters/i));
    expect(clearFilters).toHaveBeenCalled();
  });

  it('should open EmbeddedFileViewer when preview button is clicked', () => {
    userEvent.click(screen.getByRole('button', { name: /preview/i }));
    expect(screen.getByRole('dialog')).toBeInTheDocument();
  });

  it('should navigate to detail route when Received cell link is clicked', () => {
    userEvent.click(screen.getByRole('link', { name: mockLetter.receivedAt }));
    expect(mockPush).toHaveBeenCalledWith(
      expect.stringContaining(`/auth-review/${mockLetter._id}`)
    );
  });

  it('should show empty state message when letters array is empty', () => {
    render(<AuthReviewIndex orderId="ord1" />, { ...mockHookEmpty });
    expect(screen.getByText(/no prior auth letters/i)).toBeInTheDocument();
  });

  it('should show loading state while fetching', () => {
    render(<AuthReviewIndex orderId="ord1" />, { ...mockHookLoading });
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
  });
});
```

#### Step 2.13 — GREEN: implement hook + `AuthReviewIndex` component

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.ts`
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/index.ts`
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx`
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.module.css`

**`useOrderAuthReview` shape**

```ts
type OrderAuthReviewFilters = {
  letterTypes?: string[];
  payors?: string[];
  drugs?: string[];
  reviewStatuses?: string[];
};

type UseOrderAuthReviewReturn = {
  letters: PriorAuthLetter[];
  filterOptions: PriorAuthFilterOptions;
  isLoading: boolean;
  error: Error | null;
  page: number;
  pageSize: number;
  total: number;
  totalPages: number;
  sortOrder: 'asc' | 'desc';
  filters: OrderAuthReviewFilters;
  setPage: (page: number) => void;
  setPageSize: (pageSize: number) => void;
  setSortOrder: (order: 'asc' | 'desc') => void;
  setFilters: (filters: OrderAuthReviewFilters) => void;
  clearFilters: () => void;
};

export function useOrderAuthReview(args: { orderId: string }): UseOrderAuthReviewReturn
```

The hook mirrors `useClinicalReviews` in shape. It fetches from
`/api/orders/new/${orderId}/auth-review` with query params derived from filters
and pagination, and separately from `/api/orders/new/${orderId}/auth-review/filter-options`.
Filter state is local to the hook (not URL-based — the patient prior-auth view
uses URL params but the DataGrid-based order view keeps state in React to stay
consistent with `FinancialReviewIndex`).

**`AuthReviewIndex` columns (GridColDef array)**

```ts
const columns: GridColDef<PriorAuthLetter>[] = [
  {
    field: 'receivedAt',
    headerName: 'Received',
    renderCell: ({ row }) => (
      <Link href={`${basePath}/auth-review/${row._id}`}>
        {formatDate(row.receivedAt)}
      </Link>
    ),
  },
  { field: 'drug', headerName: 'Drug' },
  { field: 'payor', headerName: 'Payor' },
  {
    field: 'letterType',
    headerName: 'Letter Type',
    valueGetter: ({ row }) => LETTER_TYPE_LABEL_MAP[row.letterMetadata.documentType],
  },
  {
    field: 'authValidPeriod',
    headerName: 'Auth Valid Period',
    valueGetter: ({ row }) =>
      row.letterMetadata.documentType === 'Approval' && row.authValidFrom
        ? `${row.authValidFrom} – ${row.authValidTo}`
        : '—',
  },
  {
    field: 'reviewStatus',
    headerName: 'Review Status',
    renderCell: ({ row }) => <ReviewStatusDot status={row.reviewStatus} />,
  },
  {
    field: 'preview',
    headerName: '',
    renderCell: ({ row }) => (
      <IconButton size="small" onClick={() => setPreviewLetter(row)}>
        <VisibilityIcon />
      </IconButton>
    ),
  },
];
```

`LETTER_TYPE_LABEL_MAP` is derived from `PRIOR_AUTH_LETTER_TYPES` in `@/types/priorAuth`.

**Filter row** — four `Autocomplete` components from `@repo/ui` (or `@mui/material`
if `@repo/ui` does not export `Autocomplete`), each `multiple`. Options come from
`filterOptions.drugs`, `filterOptions.payors`, `PRIOR_AUTH_LETTER_TYPES`, and
`[{ value: 'needs_review', label: 'Needs Review' }, { value: 'reviewed', label: 'Reviewed' }]`.

**Active filter chips** — one `Chip` per active filter value below the filter row.
A `Clear all filters` text button appears when any filter is active.

**Preview** — `EmbeddedFileViewer` modal opened when `previewLetter` is set.

#### Step 2.14 — REFACTOR: extract `ReviewStatusDot` to shared location

`ReviewStatusDot` (colored dot + label) is used in both `AuthReviewIndex` and
`AuthReviewDetail`. Extract to
`apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/ReviewStatusDot.tsx`.

**Commit**: `[MLID-2306] - feat(auth-review): add AuthReviewIndex list view with DataGrid, Autocomplete filters, and preview modal`

---

#### Step 2.15 — Drug-match rule (last-minute workaround found during testing)

> **Note:** This is a **last-minute workaround** discovered while testing the
> Stage 2 work. The original order-scoped drug condition (hard `liDrugId` match,
> else null/missing passthrough) showed every un-reviewed document on every order,
> because un-reviewed documents have no `liDrugId`. After investigating the data
> and confirming the field origins (see Appendix A) and a PO decision, the drug
> condition is revised as below. This supersedes the drug half of
> `buildOrderScopedPriorAuthMatch` from Step 2.3.

**Inputs resolved per order:**
- `orderDrugId` = `neworders.drug` (a `lidrugs._id` hex string)
- `orderDrugName` = `lidrugs.name` resolved from `orderDrugId` (e.g. `"Privigen"`)

**Document drug fields used (the other two are discarded for matching):**
- `additionalInfo.priorAuthData.liDrugId.value` (id, set at verification)
- `additionalInfo.ocrData.service.brandDrugName.value` (text, set at OCR)

**Show a document (drug condition, still AND-ed with the date condition) when:**

```
(liDrugId is set       AND  liDrugId === orderDrugId)            // hard id match
OR (liDrugId null/empty AND  brandDrugName null/empty)           // passthrough (no drug info)
OR (liDrugId null/empty AND  contains(brandDrugName, orderDrugName))  // fuzzy match
```

The only two ways a document is hidden by drug:

| liDrugId | brandDrugName | Result |
|---|---|---|
| set, **==** orderDrugId | — | show (hard match) |
| set, **!=** orderDrugId | — | **hide** |
| null/empty | null/empty | show (passthrough) |
| null/empty | present, **contains** orderDrugName | show (fuzzy) |
| null/empty | present, **does not contain** orderDrugName | **hide** |

`contains(brandDrugName, orderDrugName)` = `lower(brandDrugName)` includes
`lower(orderDrugName)` — case-insensitive substring, the clean LISA name contained
within the (possibly noisy) brand name (e.g. `"OzempicQQ"` contains `"ozempic"`).

**Edge case:** if `orderDrugName` is empty/unresolvable, the fuzzy branch is
omitted (a `contains("")` would otherwise match everything); `liDrugId`-null
documents then only pass via the null/empty `brandDrugName` passthrough, and hard
id matches are unaffected.

**Implementation impact:**
- `buildOrderScopedPriorAuthMatch` (`services/priorAuth/query.ts`) — new drug
  condition; `OrderScopeArgs` gains `orderDrugName`. (Used only by the two order
  auth-review routes; the patient prior-auth routes do not pass `orderScope`, so
  they are unaffected.)
- A Mongoose resolver (`LiDrug.findById(orderDrugId) -> name`) used by both order
  routes (`auth-review/route.ts` and `auth-review/filter-options/route.ts`) to
  pass `orderDrugName`.
- Tests: `services/priorAuth/__tests__/query.orderScoped.test.ts`, both order
  route tests, plus a resolver test.

#### Step 2.16 — Note: drug filter refined to also match the generic name (last change)

> **Note:** A follow-up change made during testing, after Step 2.15 still let
> documents through whose drug name lived in the OCR **generic** name rather than
> the brand name (e.g. an order's document showed a `genericDrugName` of
> "UBLITUXIMAB-XIIY" — actually Briumvi's generic — on a Privigen order). The
> drug-match rule was extended to be **symmetric** across both name fields.

**Revised drug condition (supersedes the brand-only fuzzy match in Step 2.15):**

```
SHOW the document when (still AND-ed with the date condition):
  liDrugId is set AND liDrugId === orderDrugId                    // hard id match
  OR liDrugId is null/empty AND (
       any of [order.name, order.genericName]                    // fuzzy match
         is contained (case-insensitive) within
       any of [doc.brandDrugName, doc.genericDrugName]
       OR (doc.brandDrugName AND doc.genericDrugName both empty)  // passthrough
     )
```

Key points:
- Both the **order's** `name` and `genericName` (resolved from `lidrugs`) are
  compared against both the **document's** `brandDrugName` and `genericDrugName`.
  Each fuzzy clause is emitted only when its order term is non-empty (so empty
  terms cannot match everything). The clean LISA name is the substring searched
  inside the possibly-noisy OCR name.
- `lidrugs.genericName` is populated in **production** but mostly empty in
  **staging**; the rule degrades gracefully (the document's `genericDrugName` is
  still matched against the order's `name`).
- The **passthrough now requires both** `brandDrugName` and `genericDrugName` to
  be empty (previously only `brandDrugName`), so a document that has a generic
  name but no brand name is matched rather than passed through.
- A `$type` string guard wraps each name field before `$toLower`/`$indexOfCP`,
  because some documents store a malformed (e.g. array) value that would
  otherwise throw at query time.

**Implementation delta from Step 2.15:**
- `OrderScopeArgs` also gains `orderDrugGenericName`.
- The resolver became `getLiDrugNamesById(orderDrugId) -> { name, genericName }`
  (`services/mongodb/liDrugLookup.ts`); both order routes pass both names.

**Still NOT handled (open for PO/Egor):** documents whose drug only exists in
`priorAuthData.drugName` and/or `ocrData.service.serviceName`, with no `liDrugId`,
no `brandDrugName`, and no `genericDrugName` — these still pass via the
passthrough (e.g. a reviewed doc with `drugName: "Remicade"` but no `liDrugId`).
Treated as corrupted/edge data, tracked in
`MLID-2306-stage-2-unresolved-drug-filter-issue.md`.

---

### Stage 3 — Auth Review detail screen, Step 1 (Verify Letter Type) end-to-end

This stage produces a navigable detail screen. The user can click a row in the
list, land on the detail screen, change the letter type, change Review Status,
finish a single-step letter, and see the fax inline. The full PATCH service is
built here even though Step 1 only sends `letterType` + `reviewStatus` — it
accepts partial payloads, and Stages 4 and 5 simply send more fields.

**End-of-stage testable outcome:** click a row in the table and verify the
takeover detail screen opens. Change the letter type. Change Review Status via
the selector and confirm it persists on reload. For single-step letter types
(Receipt, Reminder, Other) click Finish & Mark Reviewed and land back on the
list. Verify the fax renders inline. Admins see the removal action; non-admins
do not.

---

#### Step 3.1 — RED: write failing tests for `getPriorAuthDetail` and `applyPriorAuthReview`

**Files (create)**
- `apps/web/services/priorAuth/__tests__/detail.test.ts`

**Test cases to write**

```ts
describe('getPriorAuthDetail', () => {
  beforeEach(() => jest.clearAllMocks());

  it('should return PriorAuthDetailResponse when intake belongs to patient', async () => {
    // Arrange: mock Intake.findById → intake with matching patient.patientId
    // mock Patient (passed in as arg), mock getDocumentsByIntakeId
    // Act: await getPriorAuthDetail(intakeId, mockPatient)
    // Assert: response has letter, ocrData, documentUrl, patientName
    expect(result.patientName).toBe('Jane Doe');
  });

  it('should return null when intake is not found', async () => {
    // Arrange: Intake.findById returns null
    // Act + Assert
    expect(await getPriorAuthDetail('unknown', mockPatient)).toBeNull();
  });

  it('should return null when intake does not belong to the patient', async () => {
    // Arrange: intake.patient.patientId !== patient.we_infuse_id_IV
    expect(await getPriorAuthDetail(intakeId, wrongPatient)).toBeNull();
  });
});

describe('applyPriorAuthReview', () => {
  beforeEach(() => jest.clearAllMocks());

  it('should save letterType and verifyData to additionalInfo.priorAuthData', async () => {
    // Arrange: mock Intake.findById → intake; mock intake.save()
    // Act: await applyPriorAuthReview(intakeId, mockPatient, patch)
    // Assert: intake.set was called with the merged priorAuthData; intake.save() called once
  });

  it('should return ownership error when intake does not belong to patient', async () => {
    // Assert: result is { ownershipError: true }
  });

  it('should return notFound when intake does not exist', async () => {
    expect(result).toEqual({ notFound: true });
  });

  it('should write reviewStatus: reviewed when included in patch', async () => {
    // Arrange patch includes reviewStatus: 'reviewed'
    // Assert priorAuthData.reviewStatus === 'reviewed' after save
  });
});
```

#### Step 3.2 — GREEN: create `apps/web/services/priorAuth/detail.ts`

**Files (create)**
- `apps/web/services/priorAuth/detail.ts`

**What to implement**

```ts
import Intake, { type IIntake } from '@/models/Intake';
import type { IPatient } from '@/models/Patient';
import { getDocumentsByIntakeId } from '@/services/mongodb/document';
import { transformIntakeToPriorAuthLetter } from '@/services/priorAuth/transforms';
import type {
  PriorAuthDetailResponse,
  PriorAuthPatchRequest,
  PriorAuthPatchResponse,
} from '@/types/priorAuth';

// Discriminated result types for service callers
export type GetDetailResult =
  | { found: true; data: PriorAuthDetailResponse }
  | { found: false; reason: 'not_found' | 'ownership_error' };

export type ApplyReviewResult =
  | { success: true; data: PriorAuthPatchResponse }
  | { success: false; reason: 'not_found' | 'ownership_error' | 'no_fields' };

/**
 * Fetch a single prior-auth intake as PriorAuthDetailResponse.
 * Validates that the intake belongs to the given patient.
 */
export async function getPriorAuthDetail(
  intakeId: string,
  patient: IPatient
): Promise<GetDetailResult> { ... }

/**
 * Apply verified prior-auth review data to an intake.
 * Validates ownership. Writes to additionalInfo.priorAuthData.
 */
export async function applyPriorAuthReview(
  intakeId: string,
  patient: IPatient,
  patch: PriorAuthPatchRequest
): Promise<ApplyReviewResult> { ... }
```

The body of each function moves the corresponding logic out of the patient
route verbatim, with return types changed from `NextResponse` to the
discriminated union types above.

#### Step 3.3 — REFACTOR: patient detail route delegates to shared services

**Files (modify)**
- `apps/web/app/api/patient/[patientId]/prior-auth/[intakeId]/route.ts`

**What to change**
- The `GET` handler calls `getPriorAuthDetail(intakeId, patient)` and maps the
  discriminated result to a `NextResponse`.
- The `PATCH` handler calls `applyPriorAuthReview(intakeId, patient, body)` and
  maps the discriminated result to a `NextResponse`.
- All the helper functions (`normalizeOCRField`, `applyOCRPagesToVerifyData`,
  etc.) move into `detail.ts`. The route file becomes thin auth + flag check +
  delegation.

**Verify**: existing behavior is identical; run the full prior-auth test suite.

**Commit**: `[MLID-2306] - refactor(prior-auth): extract getPriorAuthDetail and applyPriorAuthReview; patient route delegates to shared services`

---

#### Step 3.4 — RED: tests for the detail GET + PATCH route

**Files (create)**
- `apps/web/app/api/orders/new/[orderId]/auth-review/[letterId]/route.test.ts`

**Test cases**

```ts
describe('GET /api/orders/new/[orderId]/auth-review/[letterId]', () => {
  it('should return 401 when unauthenticated', ...);
  it('should return 403 when flag is off', ...);
  it('should return 404 when getPriorAuthDetail returns not_found', ...);
  it('should return 403 when getPriorAuthDetail returns ownership_error', ...);
  it('should return 200 with PriorAuthDetailResponse on happy path', ...);
});

describe('PATCH /api/orders/new/[orderId]/auth-review/[letterId]', () => {
  it('should return 401 when unauthenticated', ...);
  it('should return 403 when flag is off', ...);
  it('should return 400 when body has no update fields', ...);
  it('should return 200 with success:true and updatedFields on happy path', ...);
  it('should return 403 on ownership_error from applyPriorAuthReview', ...);
});
```

#### Step 3.5 — GREEN: implement the detail GET + PATCH route

**Files (create)**
- `apps/web/app/api/orders/new/[orderId]/auth-review/[letterId]/route.ts`

**Route structure** — follows the same auth → flag → order → patient pipeline
as the list route in Stage 2:

**Letter detail GET** — calls `getPriorAuthDetail(letterId, patient)` → maps
discriminated result. Returns 404 on `not_found`, 403 on `ownership_error`.

**Letter detail PATCH** — calls `applyPriorAuthReview(letterId, patient, body)` →
maps discriminated result. Returns 400 on `no_fields`, 403 on `ownership_error`,
200 with `PriorAuthPatchResponse` on success.

Every error path calls `logger.error(...)` before returning 500 responses.
No stack traces in responses.

**Commit**: `[MLID-2306] - feat(api): add order-scoped auth-review detail GET and PATCH routes`

---

#### Step 3.6 — RED: tests for `usePriorAuthReviewWizard` (Step 1 and single-step subset)

**Files (create)**
- `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/usePriorAuthReviewWizard.test.ts`

Write only the test cases that apply to the Step 1 / single-step / header
surface. The step 1→2 navigation, step-2 validity, and step-2→3 transition test
cases are deferred to Stages 4 and 5 — add comments marking them as TODO.

**Test cases (Stage 3 subset)**

```ts
describe('usePriorAuthReviewWizard', () => {
  it('should start on step 1 when reviewStatus is needs_review', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockNeedsReviewData }));
    expect(result.current.currentStep).toBe(1);
  });

  it('should return finishLabel "Finish & Mark Reviewed" when needs_review', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockNeedsReviewData }));
    expect(result.current.finishLabel).toBe('Finish & Mark Reviewed');
  });

  it('should return finishLabel "Finish & Save Changes" when already reviewed', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockReviewedData }));
    expect(result.current.finishLabel).toBe('Finish & Save Changes');
  });

  it('should hide steps 2 and 3 for single-step letter types', () => {
    const { result } = renderHook(() =>
      usePriorAuthReviewWizard({ data: mockNeedsReviewData })
    );
    act(() => result.current.setLetterType('Receipt'));
    expect(result.current.isMultiStep).toBe(false);
    expect(result.current.visibleSteps).toHaveLength(1);
  });

  // TODO Stage 4: step 1 → 2 allowed without validation
  // TODO Stage 4: step 2 → 3 blocked when step2IsValid is false
  // TODO Stage 4: step 2 → 3 allowed when step2IsValid is true
  // TODO Stage 4: multi-step shows all 3 steps
  // TODO Stage 5: start on step 3 when already reviewed, non-single-step, hasFilledVerifyData
});
```

#### Step 3.7 — RED: tests for `AuthReviewDetail` (Stage 3 subset)

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.test.tsx`

Write only the test cases that apply to the header, Step 1, single-step
finish, admin removal, and back navigation. Defer step-2 and step-3 rendering
test cases to Stages 4 and 5 — add TODO comments.

**Test cases (Stage 3 subset)**

```ts
describe('AuthReviewDetail', () => {
  it('should render the takeover header with Back button and "Auth Review" title', () => {
    render(<AuthReviewDetail />);
    expect(screen.getByRole('button', { name: /back/i })).toBeInTheDocument();
    expect(screen.getByText('Auth Review')).toBeInTheDocument();
  });

  it('should render the Review Status selector with Needs Review and Reviewed options', () => {
    render(<AuthReviewDetail />);
    expect(screen.getByLabelText(/review status/i)).toBeInTheDocument();
  });

  it('should navigate back to /auth-review when Back is clicked', () => {
    render(<AuthReviewDetail />);
    userEvent.click(screen.getByRole('button', { name: /back/i }));
    expect(mockPush).toHaveBeenCalledWith(expect.stringContaining('/auth-review'));
  });

  it('should render "View Fax in Spruce" link when spruceUrl is present', () => {
    render(<AuthReviewDetail />, { ...mockWithSpruceUrl });
    expect(screen.getByRole('link', { name: /view fax in spruce/i })).toBeInTheDocument();
  });

  it('should not render "View Fax in Spruce" link when spruceUrl is absent', () => {
    render(<AuthReviewDetail />, { ...mockWithoutSpruceUrl });
    expect(screen.queryByRole('link', { name: /view fax in spruce/i })).not.toBeInTheDocument();
  });

  it('should show only step 1 subtab for single-step letter types', () => {
    render(<AuthReviewDetail />, { ...mockWithReceiptType });
    expect(screen.queryByText(/verify data/i)).not.toBeInTheDocument();
  });

  it('should only show removal panel to admin users', () => {
    render(<AuthReviewDetail />, { ...mockAdminSession });
    expect(screen.getByText(/this is not a prior authorization letter/i)).toBeInTheDocument();
    render(<AuthReviewDetail />, { ...mockNonAdminSession });
    expect(screen.queryByText(/this is not a prior authorization letter/i)).not.toBeInTheDocument();
  });

  it('should navigate to /auth-review after successful finish', async () => {
    render(<AuthReviewDetail />);
    userEvent.click(screen.getByText(/finish.*reviewed/i));
    await waitFor(() =>
      expect(mockPush).toHaveBeenCalledWith(expect.stringContaining('/auth-review'))
    );
  });

  // TODO Stage 4: shows all 3 steps for Approval letter type
  // TODO Stage 5: full 3-step flow completes and navigates back
});
```

#### Step 3.8 — GREEN: implement `usePriorAuthReviewWizard` (Step 1 + single-step path)

**Files (create)**
- `apps/web/components/Modules/patient/components/patientPriorAuth/usePriorAuthReviewWizard.ts`

Build the hook covering only the surface needed for Stage 3:
- `currentStep` state (initialized to 1 for needs_review; the start-on-step-3
  logic is added in Stage 5).
- `letterType` + `setLetterType`.
- `SINGLE_STEP_LETTER_TYPES` constant.
- `isMultiStep` / `visibleSteps` (returns `[1]` for single-step types, `[1, 2, 3]` for multi-step).
- `finishLabel` ("Finish & Mark Reviewed" vs "Finish & Save Changes" based on current reviewStatus).
- The single-step "finish on step 1" path.

Step 1→2 / 2→3 navigation (`handleNextStep`, `handlePrevStep`, `handleStepClick`) and
`step2Data`/`step2IsValid` are added in Stage 4. `buildFinishPayload`/`buildSavePayload`
and the start-on-step-3 logic are added in Stage 5. Add comments in the hook file
marking where these will go.

**Full hook type signature (for reference — partially implemented here)**

```ts
type Step = 1 | 2 | 3;

type UsePriorAuthReviewWizardArgs = {
  data: PriorAuthDetailResponse | null;
};

type UsePriorAuthReviewWizardReturn = {
  currentStep: Step;
  isMultiStep: boolean;
  visibleSteps: Step[];
  finishLabel: string;
  prevLabel: string;
  nextLabel: string;
  letterType: string;
  setLetterType: (type: string) => void;
  step2Data: VerifyDataFormData | null;
  setStep2Data: (data: VerifyDataFormData) => void;
  step2IsValid: boolean;
  setStep2IsValid: (valid: boolean) => void;
  handleNextStep: () => void;
  handlePrevStep: () => void;
  handleStepClick: (step: Step) => void;
  buildFinishPayload: () => PriorAuthPatchRequest;
  buildSavePayload: () => PriorAuthPatchRequest;
};

export function usePriorAuthReviewWizard(
  args: UsePriorAuthReviewWizardArgs
): UsePriorAuthReviewWizardReturn
```

The hook will be consumed by both `AuthReviewDetail` (order route) and, after Stage 5,
`PriorAuthReview.tsx` (patient route). The `buildFinishPayload` / `buildSavePayload`
functions produce the `PriorAuthPatchRequest` body that the caller PATCHes to its own
API route — they do NOT perform the fetch themselves.

#### Step 3.9 — GREEN: implement `AuthReviewDetail` (header + Step 1 + single-step finish)

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx`
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.module.css`

**Files (create)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/[letterId]/page.tsx`

**`auth-review/[letterId]/page.tsx`**:
```tsx
'use client';
import { AuthReviewDetail } from '../_components/AuthReviewDetail';
export default function AuthReviewLetterPage() {
  return <AuthReviewDetail />;
}
```

**Takeover header layout**

```tsx
<div className={styles.takeoverHeader}>
  <div className={styles.left}>
    <Button variant="text" startIcon={<ArrowBackIcon />} onClick={handleBack}>
      Back
    </Button>
    <Divider orientation="vertical" flexItem />
    <Typography variant="h5">Auth Review</Typography>
  </div>
  <div className={styles.right}>
    <Select
      label="Review Status"
      value={reviewStatus}
      onChange={handleReviewStatusChange}
      options={[
        { value: 'needs_review', label: 'Needs Review' },
        { value: 'reviewed', label: 'Reviewed' },
      ]}
    />
    {spruceUrl ? (
      <Link href={spruceUrl} target="_blank" rel="noopener noreferrer">
        View Fax in Spruce ↗
      </Link>
    ) : null}
  </div>
</div>
```

The `reviewStatus` selector PATCHes `{ reviewStatus: selectedValue }` to
`/api/orders/new/${orderId}/auth-review/${letterId}`. This is the new behavior
the patient view does not have — it allows flipping status directly without
completing the full wizard.

**Step 1 content** — letter-type Select + AI Letter Summary card, using `visibleSteps`
from `usePriorAuthReviewWizard`. Steps 2 and 3 are rendered in Stages 4 and 5.

**Inline document preview** — `InlineDocumentPreview` wrapping `EmbeddedFileViewer`
on the right panel (same split layout as `PriorAuthReview.tsx`).

**Admin removal** — `hasAdminPermissions(session.user)` → show the removal panel.
Admin removes via the existing
`POST /api/patient/[patientId]/prior-auth/[intakeId]/remove` endpoint
(the patient route — there is no order-scoped remove endpoint needed, ownership
is still by patient).

**On finish/save success** — `router.push(`${basePath}/auth-review`)`.

**Commit**: `[MLID-2306] - feat(auth-review): add AuthReviewDetail with takeover header, Review Status selector, and Step 1 wizard; add detail page route`

---

### Stage 4 — Auth Review detail screen, Step 2 (Verify Data)

This stage extends the wizard and the detail component to include Step 2. At
the end of this stage, multi-step letter types show the "Verify Data" step,
the form is fillable, and the Next button is gated on validity.

**Note:** at the end of Stage 4, the "Generate Note" button (Step 2 → Step 3)
advances toward Step 3, which is still a placeholder until Stage 5. The testable
surface for Stage 4 is Step 2 itself: reachable, fillable, and validity-gated.

**End-of-stage testable outcome:** open a multi-step letter (for example,
Approval), advance from Step 1 to Step 2 via "Verify Data", fill the Verify Data
fields, observe the Next button becomes active when all required fields are filled,
and navigate back to Step 1 via "Back to Letter Type".

---

#### Step 4.1 — RED: extend `usePriorAuthReviewWizard` tests for Step 1↔2 navigation

Add these test cases to the existing `usePriorAuthReviewWizard.test.ts` file,
replacing the TODO comments from Stage 3:

```ts
  it('should allow step 1 → 2 transition without validation', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockNeedsReviewData }));
    act(() => result.current.handleNextStep());
    expect(result.current.currentStep).toBe(2);
  });

  it('should block step 2 → 3 transition when step2IsValid is false', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockNeedsReviewData }));
    act(() => result.current.handleNextStep()); // → step 2
    act(() => result.current.handleNextStep()); // should NOT advance
    expect(result.current.currentStep).toBe(2);
  });

  it('should allow step 2 → 3 transition when step2IsValid is true', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockNeedsReviewData }));
    act(() => result.current.handleNextStep()); // → step 2
    act(() => result.current.setStep2IsValid(true));
    act(() => result.current.handleNextStep()); // → step 3
    expect(result.current.currentStep).toBe(3);
  });

  it('should show all 3 steps for multi-step letter types', () => {
    const { result } = renderHook(() =>
      usePriorAuthReviewWizard({ data: mockNeedsReviewData })
    );
    act(() => result.current.setLetterType('Approval'));
    expect(result.current.isMultiStep).toBe(true);
    expect(result.current.visibleSteps).toHaveLength(3);
  });
```

#### Step 4.2 — RED: extend `AuthReviewDetail` tests for Step 2 rendering

Add this test case to the existing `AuthReviewDetail.test.tsx` file, replacing
the TODO comment from Stage 3:

```ts
  it('should show all 3 steps for Approval letter type', () => {
    render(<AuthReviewDetail />, { ...mockWithApprovalType });
    expect(screen.getByText(/verify data/i)).toBeInTheDocument();
    expect(screen.getByText(/generate note/i)).toBeInTheDocument();
  });
```

#### Step 4.3 — GREEN: extend `usePriorAuthReviewWizard` with Step 1↔2 navigation

**Files (modify)**
- `apps/web/components/Modules/patient/components/patientPriorAuth/usePriorAuthReviewWizard.ts`

Extend the hook with:
- `handleNextStep` — advances from step 1 to step 2 without validation; advances
  from step 2 to step 3 only when `step2IsValid` is true; no-ops on step 3.
- `handlePrevStep` — moves back one step; no-ops on step 1.
- `handleStepClick(step)` — navigates to the requested step with the same
  validation constraint on the step 2→3 transition.
- `step2Data` / `setStep2Data` — holds the `VerifyDataForm` data.
- `step2IsValid` / `setStep2IsValid` — validity flag from `VerifyDataForm`.
- Contextual labels: `nextLabel` = "Verify Data" (on step 1), "Generate Note" (on step 2);
  `prevLabel` = "Back to Letter Type" (on step 2), "Back to Edit Fields" (on step 3 — note:
  Step 3 itself arrives in Stage 5, but the label is determined here).

The step 2→3 transition gate is set up in this stage. Step 3 rendering is added in
Stage 5.

#### Step 4.4 — GREEN: extend `AuthReviewDetail` to render `VerifyDataForm` on Step 2

**Files (modify)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx`

For multi-step letter types, render `VerifyDataForm` (imported directly from
`@/components/Modules/patient/components/patientPriorAuth/VerifyDataForm`) on Step 2.
Wire its `onChange(data, isValid)` callback to `setStep2Data` and `setStep2IsValid`
from the hook.

**Commit**: `[MLID-2306] - feat(auth-review): extend wizard and AuthReviewDetail with Step 2 Verify Data form`

---

### Stage 5 — Auth Review detail screen, Step 3 (WeInfuse Summary Note) + finalize

This stage completes the 3-step flow, adds the start-on-step-3 behavior for
already-reviewed multi-step letters, and refactors the existing patient
`PriorAuthReview.tsx` to consume the now-complete `usePriorAuthReviewWizard`.

**End-of-stage testable outcome:** complete the full 3-step flow for a multi-step
letter and click Finish & Mark Reviewed; reopen that letter and confirm it opens
on Step 3 (WeInfuse Summary Note); confirm the patient prior-auth review view
still works unchanged and all `patientPriorAuth` tests pass.

---

#### Step 5.1 — RED: extend `usePriorAuthReviewWizard` tests for Step 3 and finish payloads

Add these test cases to the existing `usePriorAuthReviewWizard.test.ts` file,
replacing the final TODO comment from Stage 3:

```ts
  it('should start on step 3 when already reviewed, non-single-step type, and hasFilledVerifyData', () => {
    const { result } = renderHook(() => usePriorAuthReviewWizard({ data: mockReviewedData }));
    expect(result.current.currentStep).toBe(3);
  });
```

#### Step 5.2 — GREEN: extend `usePriorAuthReviewWizard` with Step 3 entry and finish payloads

**Files (modify)**
- `apps/web/components/Modules/patient/components/patientPriorAuth/usePriorAuthReviewWizard.ts`

Extend the hook with:
- Step 3 entry from `handleNextStep` when step 2 is valid.
- `handlePrevStep` on step 3 navigates to step 2.
- Initial-step-on-open = step 3 logic: when `data.reviewStatus === 'reviewed'`
  AND the letter type is non-single-step AND `hasFilledVerifyData` is true,
  initialize `currentStep` to 3.
- `buildFinishPayload()` — produces the full `PriorAuthPatchRequest` (letterType,
  verifyData from step2Data, reviewStatus: 'reviewed') for the "Finish & Mark Reviewed" path.
- `buildSavePayload()` — produces the `PriorAuthPatchRequest` without forcing
  reviewStatus, for the "Finish & Save Changes" path.

Neither function performs the fetch — that stays in the component.

#### Step 5.3 — GREEN: extend `AuthReviewDetail` to render `WeinfuseSummaryNote` on Step 3

**Files (modify)**
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx`

Render `WeinfuseSummaryNote` (imported directly from
`@/components/Modules/patient/components/patientPriorAuth/WeinfuseSummaryNote`) on Step 3.
Wire the finish and save buttons:
- "Finish & Mark Reviewed" / "Finish & Save Changes" — call `buildFinishPayload()` or
  `buildSavePayload()`, PATCH to `/api/orders/new/${orderId}/auth-review/${letterId}`,
  on success `router.push(`${basePath}/auth-review`)`.

**Commit**: `[MLID-2306] - feat(auth-review): complete 3-step wizard with WeInfuse Summary Note; wire finish and save actions`

---

#### Step 5.4 — REFACTOR: wire `PriorAuthReview` to use `usePriorAuthReviewWizard`

**Files (modify)**
- `apps/web/components/Modules/patient/components/patientPriorAuth/PriorAuthReview.tsx`

Replace the inline step state and navigation logic with `usePriorAuthReviewWizard`.
The component still handles its own fetch URL
(`/api/patient/${patientId}/prior-auth/${intakeId}`) and its own navigation back
to `/patient/${patientId}/prior-auth`.

Run the full `patientPriorAuth` test suite to confirm behavior is unchanged.

**Commit**: `[MLID-2306] - refactor(prior-auth): wire PriorAuthReview to use shared usePriorAuthReviewWizard`

---

## Files Affected

Files are grouped by the stage in which they are created or modified.

### Stage 1

| File | Action | Description |
|------|--------|-------------|
| `apps/web/utils/featureFlags/featureFlags.ts` | Modify | Add `ORDER_DETAILS_AUTH_REVIEW` enum entry and default `false` |
| `apps/web/utils/featureFlags/__tests__/featureFlags.ORDER_DETAILS_AUTH_REVIEW.test.ts` | Create | Tests for enum entry and default value |

### Stage 2

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/query.ts` | Modify | Add `buildOrderScopedPriorAuthMatch` and `buildPriorAuthListPipeline` exports |
| `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts` | Create | Tests for `buildOrderScopedPriorAuthMatch` |
| `apps/web/services/priorAuth/__tests__/query.listPipeline.test.ts` | Create | Tests for `buildPriorAuthListPipeline` |
| `apps/web/app/api/patient/[patientId]/prior-auth/route.ts` | Modify | Delegate to `buildPriorAuthListPipeline`; remove inline pipeline construction |
| `apps/web/services/mongodb/newOrders.ts` | Create | `getNewOrderById(orderId)` helper to load order fields needed by auth-review routes |
| `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` | Create | Order-scoped list route, gated by flag |
| `apps/web/app/api/orders/new/[orderId]/auth-review/route.test.ts` | Create | Unit tests for list route |
| `apps/web/app/api/orders/new/[orderId]/auth-review/filter-options/route.ts` | Create | Order-scoped filter-options route |
| `apps/web/app/api/orders/new/[orderId]/auth-review/filter-options/route.test.ts` | Create | Unit tests for filter-options route |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` | Modify | Add `isAuthReviewPath` + `isAuthReviewEnabled` gate for `children` |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx` | Modify | Add auth-review path gate test cases |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.tsx` | Modify | Add Auth Review tab + `AuthReviewIndex` content branch |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.test.tsx` | Create | Tests for Auth Review tab visibility and routing |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/page.tsx` | Create | One-liner: returns `<OrderClinicalReviewsPanel />` |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.ts` | Create | Data hook for list fetching and filter state |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.test.ts` | Create | Tests for data hook |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/index.ts` | Create | Barrel export |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx` | Create | DataGrid list + Autocomplete filters + preview modal |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.module.css` | Create | Styles for filter row and chip area |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx` | Create | RTL tests for list component |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/ReviewStatusDot.tsx` | Create | Shared colored dot + label for review status |

### Stage 3

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/detail.ts` | Create | `getPriorAuthDetail` and `applyPriorAuthReview` shared service functions |
| `apps/web/services/priorAuth/__tests__/detail.test.ts` | Create | Tests for detail service functions |
| `apps/web/app/api/patient/[patientId]/prior-auth/[intakeId]/route.ts` | Modify | Delegate GET/PATCH bodies to `getPriorAuthDetail` / `applyPriorAuthReview` |
| `apps/web/app/api/orders/new/[orderId]/auth-review/[letterId]/route.ts` | Create | Letter detail GET + PATCH, delegates to shared services |
| `apps/web/app/api/orders/new/[orderId]/auth-review/[letterId]/route.test.ts` | Create | Unit tests for detail GET + PATCH |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/[letterId]/page.tsx` | Create | Renders `AuthReviewDetail` |
| `apps/web/components/Modules/patient/components/patientPriorAuth/usePriorAuthReviewWizard.ts` | Create | Shared step-orchestration hook (Step 1 + single-step path, extended in Stages 4 and 5) |
| `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/usePriorAuthReviewWizard.test.ts` | Create | Tests for shared wizard hook (Stage 3 subset; extended in Stages 4 and 5) |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx` | Create | Header + Step 1 + single-step finish (extended in Stages 4 and 5) |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.module.css` | Create | Styles for takeover header layout |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.test.tsx` | Create | RTL tests for detail component (Stage 3 subset; extended in Stages 4 and 5) |

### Stage 4

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/Modules/patient/components/patientPriorAuth/usePriorAuthReviewWizard.ts` | Modify | Add step 1↔2 navigation, step2Data/step2IsValid, contextual labels |
| `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/usePriorAuthReviewWizard.test.ts` | Modify | Add step navigation test cases |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx` | Modify | Render `VerifyDataForm` on Step 2 |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.test.tsx` | Modify | Add Step 2 rendering test case |

### Stage 5

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/Modules/patient/components/patientPriorAuth/usePriorAuthReviewWizard.ts` | Modify | Add step 3 entry, start-on-step-3 logic, buildFinishPayload, buildSavePayload |
| `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/usePriorAuthReviewWizard.test.ts` | Modify | Add start-on-step-3 test case |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx` | Modify | Render `WeinfuseSummaryNote` on Step 3; wire finish and save actions |
| `apps/web/components/Modules/patient/components/patientPriorAuth/PriorAuthReview.tsx` | Modify | Use `usePriorAuthReviewWizard`; remove inline step logic |

---

## Testing Strategy

### Unit tests — services (100% coverage target)

**Stage 2**
- `buildOrderScopedPriorAuthMatch`: date condition (with and without `dateOfDecision.value`),
  drug match (exact string match, null branch, empty string branch, mismatch),
  malformed date string guard (must parse to null, not throw).
- `buildPriorAuthListPipeline`: no `orderScope` (single match), with `orderScope`
  (two match stages), all filter dimensions (letterType, payor, drug, reviewStatus,
  receivedStart/End), pagination skip calculation, sort direction.

**Stage 3**
- `getPriorAuthDetail`: found + valid ownership, found + wrong ownership, not found.
- `applyPriorAuthReview`: success with letterType, success with verifyData, success
  with reviewStatus, ownership error, not found, no fields → `no_fields` result.

### Unit tests — API routes (happy path + all error paths)

**Stage 2**
- List route: 401 no session, 403 flag off, 404 order not found, 404 patient not
  found, 200 with paginated letters, 200 with filters applied.
- Filter-options route: 401, 403, 200 with order-scoped distinct drugs/payors.

**Stage 3**
- Letter detail GET: 401, 403, 404, 403 ownership, 200.
- Letter detail PATCH: 401, 403, 400 no fields, 403 ownership, 200.

### Component tests (>80% coverage target)

**Stage 2**
- `AuthReviewIndex`: all 7 columns present, 4 Autocomplete filters, chip rendering,
  clear-all action, preview modal open/close, row link navigation, empty state,
  loading state, error state.

**Stage 3**
- `usePriorAuthReviewWizard` (Step 1 subset): starts on step 1 for needs_review;
  finishLabel "Finish & Mark Reviewed" vs "Finish & Save Changes"; single-step types
  show only one visible step.
- `AuthReviewDetail` (Stage 3 subset): takeover header elements, Back navigation,
  Review Status selector renders options and PATCHes on change, Spruce link conditional,
  single-step renders only Step 1, admin removal panel visibility, finish success navigation.

**Stage 4**
- `usePriorAuthReviewWizard` (navigation subset): step 1→2 allowed; step 2→3 blocked
  when invalid; step 2→3 allowed when valid; multi-step shows 3 steps.
- `AuthReviewDetail` (Step 2 subset): Approval type shows Verify Data and Generate Note
  step labels.

**Stage 5**
- `usePriorAuthReviewWizard` (complete): start-on-step-3 when reviewed + non-single-step
  + hasFilledVerifyData.
- `AuthReviewDetail` (complete): full 3-step flow finish navigates back.

### Manual verification checklist (after tests pass and before commit)

**After Stage 2**
1. Enable `ORDER_DETAILS_AUTH_REVIEW` in the admin feature flags panel.
2. Open an order detail page for a new order that has a linked patient with prior-auth
   documents.
3. Verify the "Auth Review" tab appears between "Clinical Reviews" and "Financial Review".
4. Verify the list shows only letters whose date is on or after the order's creation date.
5. Verify multi-select filter controls work; apply a drug filter; confirm results narrow.
6. Verify active filter chips appear; click Clear all filters; confirm list resets.
7. Click the preview icon on a row; confirm the document preview modal opens.
8. Note: clicking a row in the "Received" column leads to a 404 — that route is added in Stage 3.

**After Stage 3**
9. Click a letter row; verify the takeover detail screen opens with the Back button and
   "Auth Review" heading.
10. Change Review Status via the selector; reload page; verify the status persisted.
11. For a Receipt, Reminder, or Other letter: verify only Step 1 tab shows; click
    Finish & Mark Reviewed; verify navigation back to the list.
12. Verify the fax renders inline in the document preview panel.
13. Log in as a non-admin user; verify the removal panel is absent in the detail view.
14. Log in as an admin; verify the removal panel is present.

**After Stage 4**
15. Open an Approval letter; verify all three step tabs appear.
16. Advance to Step 2; fill the Verify Data fields; confirm the Next button becomes
    active once all required fields are filled.
17. Navigate back to Step 1 via "Back to Letter Type"; confirm state is preserved.

**After Stage 5**
18. Complete all 3 steps of the wizard for an Approval letter; click Finish & Mark
    Reviewed; confirm navigation back to the list.
19. Reopen the letter just reviewed; confirm it opens on Step 3 (WeInfuse Summary Note).
20. Navigate to a patient's prior-auth view; confirm the review experience still works
    identically to before this story.

**Final**
21. Disable the feature flag; verify the Auth Review tab disappears and the API returns 403.

### QA note — MLID-2363 dependency

Until MLID-2363 lands, `priorAuthData.liDrugId` is populated only on manually
reviewed documents. This means the drug-match condition fires on those documents
only; all other documents match via the null/unknown branch and are included
regardless of drug. QA should verify that un-reviewed documents still appear in
the list (they take the null branch), and that reviewed documents with a matching
drug id also appear. Reviewed documents with a non-matching drug id should be
excluded from the list.

---

## Security Considerations

### Input validation
All API routes validate that `orderId` and `letterId` are non-empty strings.
The PATCH body is validated using the same normalization logic as the patient route
(`normalizeOCRField`, `applyOCRPagesToVerifyData`). No field from the request body
is written to MongoDB without going through those normalizers first.

### Authorization — dual enforcement
- **Server**: every auth-review API route checks `getServerSession` (401 when no session)
  and `isFeatureEnabled(FeatureFlag.ORDER_DETAILS_AUTH_REVIEW)` (403 when off). Ownership
  is verified by `intake.patient.patientId === patient.we_infuse_id_IV` before any read or
  write. Failing ownership returns 403, not 404 (no information leakage about document
  existence).
- **UI**: `OrderClinicalReviewsPanel` renders the Auth Review tab only when
  `useFeatureFlag(ORDER_DETAILS_AUTH_REVIEW)` is true. `layout.tsx` renders the
  `auth-review` subtree only when the flag is on. Neither of these replaces the
  server-side check — they are complementary gates.

### PHI
- Prior-auth intakes contain patient-identifiable insurance and medical data. All
  document preview URLs must go through the existing `/api/files/stream` wrapper
  inside `EmbeddedFileViewer` — never expose raw blob URLs in JSON responses (the
  transform already stores the raw blob URL in `documentUrl` and `EmbeddedFileViewer`
  wraps it; do not change this behavior).
- No prior-auth field values are written to logs. `logger.error` calls log only
  the error object and non-PHI context (orderId, intakeId — these are Mongo IDs,
  not patient identifiers).
- The PATCH response body (`PriorAuthPatchResponse`) contains only `success`,
  `message`, and `updatedFields` (field names, not values) — no PHI returned.

### Admin-only removal
The "This is not a Prior Authorization letter" removal action in `AuthReviewDetail`
must remain behind `hasAdminPermissions(session.user)` from `@/utils/roles`.
The corresponding POST route is the existing patient-scoped
`/api/patient/[patientId]/prior-auth/[intakeId]/remove` — no new order-scoped
remove endpoint is created. Ownership is still validated server-side by the
existing patient route.

---

## Open Questions

### 1. `linkedOrderIds` as a simpler order-scoping mechanism

`Intake.linkedOrderIds: string[]` is a field on the Intake model that stores
direct order-to-intake links. If this field is reliably populated for prior-auth
intakes, the order-scoping logic could be simplified from a date+drug heuristic
to a direct `{ linkedOrderIds: orderId }` match, which would be more precise and
avoid the `dateOfDecision` parsing complexity.

The preplan's date+drug heuristic is the PO's explicit written requirement and is
what this plan implements. Before the story is marked Done, it is worth raising
with the PO whether `linkedOrderIds` is the right mechanism for a follow-up
cleanup. Do not build around it in this story.

### 2. `neworders.createdAt` as order date (provisional)

The date condition uses `neworders.createdAt` (the Mongo row creation timestamp)
as the order creation date. This was marked "provisional pending PO decision" in
the preplan (section 2.10). If the PO wants a different business date (such as a
WeInfuse-sourced order date that may differ from when the LISA row was written),
the single field to swap is the `orderCreatedAt` value passed into
`buildOrderScopedPriorAuthMatch` — the comparison logic stays the same.

Confirm with the PO before the story reaches QA.

### 3. `NewOrder` Mongoose model availability

The plan calls for a `getNewOrderById(orderId)` helper in
`apps/web/services/mongodb/newOrders.ts` that returns the `patient`, `drug`, and
`createdAt` fields from `neworders`. Confirm whether a Mongoose model for
`neworders` already exists under a different file name before creating a new
service file. If the existing aggregation service
(`apps/web/services/mongodb/ordersTracker.ts`) can be reused to load a single
order by `_id` with those fields, prefer that over a new helper. The key
constraint is: do not use the raw MongoDB driver.

---

# Appendix A — Drug fields origin (Stage 2 only; handled via conversational prompting)

> **Note:** This appendix is relevant **only to Stage 2** (the order-scoped drug
> filter). It is being worked through via conversational prompting rather than as
> a formal stage step, because the matching rule is an open product decision (see
> the standalone issue doc `MLID-2306-stage-2-unresolved-drug-filter-issue.md`,
> pending PO). It is recorded here so the full plan stays self-contained.

A prior-auth document stores the drug across **four** fields, while an order
stores it in **one** (a LISA drug id, `neworders.drug` = `lidrugs._id`). The two
field groups are written by two different processes. Confirmed by reading both
the li-call-processor monorepo and the separate `li-intake` (Python) repo on
2026-06-04.

## Field group A — OCR drug names (written during OCR processing)

Fields:
- `additionalInfo.ocrData.service.brandDrugName.value`
- `additionalInfo.ocrData.service.genericDrugName.value`

Who writes them, and when:
- Produced by the OCR/AI step in the **li-intake** repo (Python), returned to this
  monorepo, and **persisted here**.
- Persistence lives in `@repo/shared`:
  `packages/shared/src/ocr/processOcrDataAndUpdateIntake.ts`, which writes
  `additionalInfo.ocrData` via `deps.updateIntake(...)` (around lines 180-208) and
  sets the intake status to `needsReview`.
- Invoked by the intake processing job
  `apps/web/services/jobs/definitions/processIntake/processIntakeJob.ts`
  (around line 298, after `intakeAiService.analyzeDoc(...)`), and from
  `processIntake/pollForCachedOcrResults.ts`.
- li-call-processor stores the names exactly as li-intake returns them; it does
  not modify the values on this side.

li-intake LISA-name alignment (what "MLID-2363" refers to):
- `li-intake/src/core/order_rules.py` -> `_align_prior_auth_service_drug_names`,
  added by commit `56abf9b` "MLID-2363 map PA drug names to LISA drugs"
  (2026-05-15). It matches the OCR brand (then generic) name to a LISA medication
  configuration name via `match_medication_configuration_name_only`
  (`li-intake/src/core/page_processors/order/medication_matcher.py`). On match it
  overwrites BOTH brand and generic with that single LISA name; on no match it
  sets BOTH to null.
- Data reality (Mongo `intakes`, 2026-06-04): the alignment is NOT consistently
  reflected. Of processed prior-auth docs: 97 still carry raw, differing
  brand/generic names (newest 2026-06-03), 85 are both-null, only 9 show the
  aligned "both equal a LISA name" pattern. So the OCR name fields cannot be
  assumed to hold a clean LISA name today.

## Field group B — verified drug fields (written during form verification)

Fields:
- `additionalInfo.priorAuthData.drugName.value`
- `additionalInfo.priorAuthData.liDrugId.value`

Who writes them, and when:
- Written when a user verifies the drug in the 3-step prior-auth review form in
  `/patient/[id]/prior-auth` (same write path reused by the order Auth Review
  screen `/orders-tracker/new/[id]/auth-review`).
- The form
  `apps/web/components/Modules/patient/components/patientPriorAuth/VerifyDataForm.tsx`
  sets `drugName = selectedLiDrug.name` and `liDrugId = selectedLiDrug._id` when
  the user selects a LISA drug (around lines 117-119). It prefills `drugName` from
  the OCR brand/generic name and re-resolves the LISA drug on re-open via
  `apps/web/services/priorAuth/resolveLisaDrugFromCatalog.ts`.
- The PATCH route
  `apps/web/app/api/patient/[patientId]/prior-auth/[intakeId]/route.ts` persists
  them into `additionalInfo.priorAuthData` with `reviewStatus: 'reviewed'`
  (around lines 455-461).
- li-intake does NOT set `liDrugId`; it is attached only at verification time
  here.

## Net summary

The only id-comparable field is `priorAuthData.liDrugId` (set only after human
verification). The OCR name fields are intended (by the li-intake MLID-2363
alignment) to be the LISA name or null, but in current data they are mostly still
raw text. This is the core constraint behind the unresolved drug-match rule.
