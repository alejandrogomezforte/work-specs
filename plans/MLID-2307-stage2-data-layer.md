# MLID-2307 — Stage 2: Financial Review Data Layer

## Task Reference

- **Jira**: [MLID-2307](https://localinfusion.atlassian.net/browse/MLID-2307) (Stage 2)
- **Parent epic**: [MLID-2206](https://localinfusion.atlassian.net/browse/MLID-2206) — "Order Details v2 - Show Each Step Under Order Tracker"
- **Story Points**: 5 (to be confirmed; pencil-estimate based on step count and complexity)
- **Branch**: `feature/MLID-2307-financial-review` (continues Stage 1's branch)
- **Base Branch**: `develop` (Stage 1 branched off develop; Stage 2 continues on the same branch)

> **MANDATORY — branch directive for `/implement`**: do **NOT** create a new branch off `develop`. Stage 1 is already committed locally on `feature/MLID-2307-financial-review`; Stage 2 must continue on that same branch. Before running any implementation step, confirm with `git rev-parse --abbrev-ref HEAD` that the current branch is `feature/MLID-2307-financial-review`. If it is not, `git checkout feature/MLID-2307-financial-review` first. Do NOT run `git checkout develop && git pull && git checkout -b feature/...` — that would destroy or hide the Stage 1 commit.
- **Stage 1 plan**: [`MLID-2307-financial-review-calculator.md`](./MLID-2307-financial-review-calculator.md)
- **Schema decision record**: [`../analysis/MLID-2307-calculations-schema-options.md`](../analysis/MLID-2307-calculations-schema-options.md)
- **Legacy AI flow reference**: [`../analysis/MLID-2307-how-patients-financial-calculation-ai-breakdown-works.md`](../analysis/MLID-2307-how-patients-financial-calculation-ai-breakdown-works.md)

---

## Summary

Replace the Stage 1 in-memory mock (`FinancialCalculationsContext` + `seed.ts`) with a real, order-scoped persistence layer. New Mongoose collection `ordersFinancialCalculations` stores each calculation as a structured document — 9 numeric `inputs`, 3 numeric `result` totals, a `breakdown` object whose bullet arrays come from a deterministic calculation function and whose `reason` paragraph comes from Azure OpenAI. New REST endpoints under `/api/orders/{new,maintenance}/[orderId]/financial-calculations`. Stage 1 UI is rewired to call the API; the in-memory context is deleted. Legacy patient-scoped flow (`/patient/[id]/financial-calculator`) is **not modified** — both flows coexist.

---

## Architecture Context (locked)

- **Collection**: `ordersFinancialCalculations` (single dedicated collection, Option B).
- **Foreign-key fields**: `orderId: ObjectId` + `orderCategory: 'neworders' | 'maintenanceorders'`.
- **Per-document shape**: `inputs`, `result`, `breakdown` (`drug[]`, `admin[]`, `summary[]`, `reason`), `author`, `calculationDate`, `createdAt`, `updatedAt`. No `editedAt` / `deletedAt` (calculations are immutable per PO).
- **Index**: compound `{ orderId: 1, orderCategory: 1, calculationDate: -1 }`.
- **Authorization**: any authenticated user with order-detail access can `POST`. Role gate deferred to a follow-up ticket.
- **AI scope**: only `breakdown.reason` is AI-generated. `breakdown.drug`, `breakdown.admin`, `breakdown.summary` are produced deterministically by the calculation function (mirroring how the legacy `processPart()` closure built its bullets, just captured into arrays instead of joined into a markdown blob).
- **AI sync vs. deferred**: synchronous in the `POST` request path, mirroring the legacy patient flow exactly. If Azure fails, the whole `POST` fails (same behavior as legacy — no graceful degradation, no second-guessing the proven pattern).

The full PE/TL decision record (rejected alternatives, vote, naming amendments) lives in the schema-options analysis doc.

---

## Coexistence with the legacy patient flow

Hard rule: **no file under the legacy patient calculator is modified.** The following keep working unchanged:

- `apps/web/pages/api/patients/[id]/financialCalculation.ts`
- `apps/web/services/mongodb/financialCalculation.ts`
- `apps/web/components/Modules/financial-calculator/**`
- `apps/web/utils/api/patient/makeFinancialCalculation.ts`
- `apps/web/utils/hooks/useMakeFinancialCalculation.ts`
- `apps/web/types/financialCalculationDTO.ts`
- `patients.financialCalculationsHistory[]` data in MongoDB

All Stage 2 code lives in new files using the `OrdersFinancialCalculation` / `ordersFinancialCalculation` prefix, avoiding any import / name collision with the legacy flow.

---

## Naming reference

| Concept | Identifier |
|---|---|
| MongoDB collection | `ordersFinancialCalculations` |
| Mongoose model file | `apps/web/models/OrdersFinancialCalculation.ts` |
| Mongoose model class + default export | `OrdersFinancialCalculation` |
| Document interface | `OrdersFinancialCalculationDocument` |
| Sub-schema vars | `CalculationInputsSchema`, `CalculationResultSchema`, `BreakdownSchema`, `OrdersFinancialCalculationSchema` |
| COLLECTION constant entry | `OrdersFinancialCalculations: 'ordersFinancialCalculations'` |
| Server orchestration folder | `apps/web/services/ordersFinancialCalculation/` |
| Orchestration file (calc + AI + helpers, single file) | `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.ts` |
| Pure calculation function | `runOrdersFinancialCalculation(inputs)` → `{ result, breakdown: { drug, admin, summary } }` (no `reason`) |
| Per-leg helper | `processOrdersFinancialCalculationPart()` |
| AI reason generator | `generateOrdersFinancialCalculationReason(breakdown)` → `Promise<string>` |
| Shared POST/GET handler | `handleOrdersFinancialCalculationPost()`, `handleOrdersFinancialCalculationGet()` |
| Mongo service file | `apps/web/services/mongodb/ordersFinancialCalculation.ts` |
| Mongo service functions | `createOrdersFinancialCalculation()`, `findOrdersFinancialCalculationsByOrder()` |
| Client API helper file | `apps/web/utils/api/orders/financialCalculations.ts` (no `Orders` prefix in filename since parent folder is `orders/`) |
| Client API helper functions | `makeOrdersFinancialCalculation()`, `fetchOrdersFinancialCalculations()` (function names keep the prefix to avoid collision with legacy `makeFinancialCalculation()`) |
| Client list hook | `useOrdersFinancialCalculations()` — replaces Stage 1 `useFinancialCalculations` |
| Client action hook | `useMakeOrdersFinancialCalculation()` |
| Domain type (client + server shared) | `OrdersFinancialCalculation` (replaces Stage 1 `FinancialCalculation`) |

URL routes:

- `GET  /api/orders/new/[orderId]/financial-calculations`
- `POST /api/orders/new/[orderId]/financial-calculations`
- `GET  /api/orders/maintenance/[orderId]/financial-calculations`
- `POST /api/orders/maintenance/[orderId]/financial-calculations`

URL category param uses the frontend-facing `'new'` / `'maintenance'` (matching the existing `[category]` segment in `app/orders-tracker/...`). The API handler translates to the DB-side `orderCategory` field values `'neworders'` / `'maintenanceorders'` before persisting.

---

## Implementation Steps

Each step is one commit, RED → GREEN → REFACTOR within the step.

---

### Step 1 — COLLECTION constant + Mongoose model

**Files**
- `apps/web/utils/constants/collections.ts` — Modify (add `OrdersFinancialCalculations: 'ordersFinancialCalculations'`)
- `apps/web/models/OrdersFinancialCalculation.ts` — Create
- `apps/web/models/OrdersFinancialCalculation.test.ts` — Create

**What**

Implement the Mongoose schema designed in the Stage 1 plan's "Mongoose model design" section. Sub-schemas for `inputs`, `result`, `breakdown` with `_id: false` and inline validators. Compound index on `{ orderId: 1, orderCategory: 1, calculationDate: -1 }`. Hot-reload-safe export pattern matching `apps/web/models/Order.ts`.

**RED phase tests:**

```
describe('OrdersFinancialCalculation model', () => {
  it('should accept a fully-populated valid document')
  it('should reject a document missing orderId')
  it('should reject a document missing orderCategory')
  it('should reject orderCategory values outside the enum')
  it('should reject negative numeric inputs (e.g. deductibleMet: -1)')
  it('should reject coinsurance > 100')
  it('should default breakdown.drug, .admin, .summary to []')
  it('should default breakdown.reason to ""')
  it('should set createdAt / updatedAt via timestamps option')
  it('should set calculationDate via default if not provided')
  it('should not mint _id on sub-documents (inputs, result, breakdown)')
})
```

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add OrdersFinancialCalculation Mongoose model and COLLECTION entry
```

---

### Step 2 — Orchestration file: pure calc + AI reason + helpers (one file)

**Files**
- `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.ts` — Create
- `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.test.ts` — Create

This single file holds the deterministic calculation (`runOrdersFinancialCalculation` + `processOrdersFinancialCalculationPart` + `fmt$` / `fmtPct` helpers) **and** the AI reason generator (`generateOrdersFinancialCalculationReason`). Matches the legacy pattern where `calculateMedicalCosts`, `processPart`, `generateAiReason`, and the formatters all sit in one place. Splitting them across files would be over-decomposition.

Step 2 below covers the deterministic functions. Step 3 covers the AI reason generator added to the same file.

**What**

Port `calculateMedicalCosts` + `processPart` from the legacy `pages/api/patients/[id]/financialCalculation.ts`. Adaptations:

- **Take numbers, not strings** — type is `CalculationInputs` (the 9 numeric fields). No `parseFloat`.
- **Return structured arrays, not a markdown blob** — instead of mutating a shared `lines: string[]` and joining at the end, `processOrdersFinancialCalculationPart` returns `{ patientPays, bullets: string[], newState }` where `state` tracks `remainingDeductible` and `remainingOOP`. The orchestrator threads state explicitly through Drug then Admin.
- **No AI inside** — this function is pure. The `reason` field is added by the API handler, not the calculator.
- **Output shape**:
  ```ts
  type Output = {
    result: { drugResp: number; adminResp: number; totalResp: number };
    breakdown: { drug: string[]; admin: string[]; summary: string[] };
  };
  ```

Helper formatters (`fmt$`, `fmtPct`) are colocated.

**RED phase tests** — port the legacy edge cases verbatim:

```
describe('runOrdersFinancialCalculation', () => {
  describe('OOP-already-met branch', () => {
    it('should return zero responsibility when oopMet >= oopMax')
    it('should include the "OOP max already met" line in summary[]')
  })

  describe('deductible-met no-coinsurance branch', () => {
    it('should return zero when deductible is fully met and coinsurance is 0')
  })

  describe('no-deductible no-coinsurance branch', () => {
    it('should return zero when there is no deductible and coinsurance is 0')
  })

  describe('partial deductible + coinsurance', () => {
    it('should apply cost to remaining deductible then coinsurance on the leftover')
    it('should include the deductible-applied bullet in breakdown.drug')
    it('should include the coinsurance bullet in breakdown.drug')
  })

  describe('assistance application', () => {
    it('should subtract drug assistance from drug responsibility, clamped at responsibility')
    it('should subtract admin assistance from admin responsibility, clamped at responsibility')
    it('should not let admin assistance overflow into drug')
  })

  describe('OOP cap', () => {
    it('should cap responsibility-before-assistance at remaining OOP per leg')
    it('should skip the Admin leg entirely when remainingOOP hits zero during Drug')
    it('should mark "OOP max reached" in breakdown.drug when remainingOOP drops to 0')
  })

  describe('breakdown shape', () => {
    it('should emit a "Starting with cost $X" first bullet for both drug and admin')
    it('should emit summary[] with the three final totals')
    it('should not emit any AI-generated content')
  })

  describe('fmt$', () => {
    it('should format a number with two decimal places and dollar sign')
    it('should return $0.00 for 0')
  })
})
```

Use fixtures derived from the seed scenarios already validated in Stage 1's `calculate.test.ts` so the new function's outputs are checkable against the existing Stage 1 expectations.

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add runOrdersFinancialCalculation pure function

Ports the deterministic calculation logic from the legacy patient flow
(calculateMedicalCosts + processPart) to a structured-output, number-typed
function. Returns inputs/result + a breakdown object with drug/admin/summary
arrays — no AI call inside. The reason field is added later by the API handler.
```

---

### Step 3 — Azure OpenAI reason generator (added to the same file)

**Files**
- `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.ts` — Modify (append `generateOrdersFinancialCalculationReason`)
- `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.test.ts` — Modify (append tests)

**What**

Mirror the legacy patient flow's `generateAiReason` behavior exactly — same client construction (per-call `new AzureOpenAI({...})`), same env vars, same system + user prompts (verbatim, including the typo), same `max_completion_tokens: 4096`, same model selection (`process.env.AZURE_OPENAI_TINY_DEPLOYMENT_NAME || 'gpt-5-nano'`). Errors propagate to the caller (no `try/catch`, no fallback). This is a deliberate "don't reinvent" — keep the proven communication pattern; we can evolve it in a follow-up if needed.

One shape difference: takes the deterministic `breakdown` object as input (so the user prompt joins `breakdown.drug ∪ breakdown.admin ∪ breakdown.summary` as the calculation steps) and returns `Promise<string>` (legacy returns `Promise<string[]>` via `content.split('\n')`; we want a single prose string for `breakdown.reason`).

**RED phase tests:**

```
describe('generateOrdersFinancialCalculationReason', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should construct AzureOpenAI with endpoint/apiKey/apiVersion from env')
  it('should call chat.completions.create with the configured deployment')
  it('should send the legacy system prompt verbatim')
  it('should include the breakdown bullets in the user prompt')
  it('should pass max_completion_tokens: 4096')
  it('should return the response content as a single string')
  it('should propagate the error if Azure throws (no fallback)')
})
```

Mock `AzureOpenAI` constructor + the `chat.completions.create` method.

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add generateOrdersFinancialCalculationReason Azure OpenAI service

Mirrors the legacy patient generateAiReason behavior — same client setup,
same prompts, same model env-var fallback, same error propagation. Only
difference is the input shape (takes the deterministic breakdown object)
and the return type (single string instead of legacy's split-then-rejoin
through string[]).
```

---

### Step 4 — Mongo service layer

**Files**
- `apps/web/services/mongodb/ordersFinancialCalculation.ts` — Create
- `apps/web/services/mongodb/ordersFinancialCalculation.test.ts` — Create

**What**

Mongoose-based service exposing:

```ts
export async function createOrdersFinancialCalculation(input: {
  orderId: string;            // accepts hex string, converts to ObjectId internally
  orderCategory: 'neworders' | 'maintenanceorders';
  inputs: CalculationInputs;
  result: CalculationResult;
  breakdown: Breakdown;
  author: string;
}): Promise<OrdersFinancialCalculationDocument>

export async function findOrdersFinancialCalculationsByOrder(
  orderId: string,
  orderCategory: 'neworders' | 'maintenanceorders'
): Promise<OrdersFinancialCalculationDocument[]>
```

Both always take `orderId` + `orderCategory` together — the discriminator footgun mitigation called out in the locked architecture.

`find` returns the list sorted newest-first by `calculationDate`. `.lean()` is applied for read performance.

**RED phase tests** — integration-style with `mongodb-memory-server` (or whatever the repo uses for mongoose tests; check existing model tests for pattern):

```
describe('createOrdersFinancialCalculation', () => {
  it('should persist a document with all fields set')
  it('should set createdAt / updatedAt automatically')
  it('should reject when orderCategory is invalid')
  it('should reject when a required numeric input is missing')
  it('should accept orderId as a hex string and store it as ObjectId')
})

describe('findOrdersFinancialCalculationsByOrder', () => {
  it('should return all calculations for the given orderId+orderCategory pair')
  it('should return them sorted by calculationDate descending')
  it('should return an empty array when no calculations exist')
  it('should NOT return calculations for a different orderCategory even if orderId matches')
  it('should NOT return calculations for a different orderId even if orderCategory matches')
})
```

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add Mongoose service layer for CRUD operations
```

---

### Step 5 — API routes (two thin wrappers + shared handlers)

**Files**
- `apps/web/services/ordersFinancialCalculation/handlers.ts` — Create
- `apps/web/services/ordersFinancialCalculation/handlers.test.ts` — Create
- `apps/web/app/api/orders/new/[orderId]/financial-calculations/route.ts` — Create
- `apps/web/app/api/orders/new/[orderId]/financial-calculations/route.test.ts` — Create
- `apps/web/app/api/orders/maintenance/[orderId]/financial-calculations/route.ts` — Create
- `apps/web/app/api/orders/maintenance/[orderId]/financial-calculations/route.test.ts` — Create

**What**

`handlers.ts` exports two functions that the two route files call (passing in a hardcoded `orderCategory`):

```ts
export async function handleOrdersFinancialCalculationPost(
  orderId: string,
  orderCategory: 'neworders' | 'maintenanceorders',
  body: unknown,
  session: Session,
): Promise<Response>

export async function handleOrdersFinancialCalculationGet(
  orderId: string,
  orderCategory: 'neworders' | 'maintenanceorders',
  session: Session,
): Promise<Response>
```

`handleOrdersFinancialCalculationPost`:
1. Validate `body` shape (9 numeric inputs).
2. `result, breakdown = runOrdersFinancialCalculation(inputs)`.
3. `reason = await generateOrdersFinancialCalculationReason(breakdown)`.
4. `saved = await createOrdersFinancialCalculation({ orderId, orderCategory, inputs, result, breakdown: { ...breakdown, reason }, author: session.user.email })`.
5. Return 200 with `saved` as JSON.
6. If any step throws, return 500 with logged error — same all-or-nothing behavior as the legacy patient handler.

`handleOrdersFinancialCalculationGet`:
1. `list = await findOrdersFinancialCalculationsByOrder(orderId, orderCategory)`.
2. Return 200 with `list` as JSON.

Each `route.ts` is ~20 lines:

```ts
// app/api/orders/new/[orderId]/financial-calculations/route.ts
import { handleOrdersFinancialCalculationGet, handleOrdersFinancialCalculationPost } from '@/services/ordersFinancialCalculation/handlers';
// ... auth import ...

export async function GET(req: Request, { params }: { params: { orderId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session) return new Response('Unauthorized', { status: 401 });
  return handleOrdersFinancialCalculationGet(params.orderId, 'neworders', session);
}

export async function POST(req: Request, { params }: { params: { orderId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session) return new Response('Unauthorized', { status: 401 });
  const body = await req.json();
  return handleOrdersFinancialCalculationPost(params.orderId, 'neworders', body, session);
}
```

The `maintenance/` route is identical except the hardcoded `'maintenanceorders'` value.

**RED phase tests** — split between `handlers.test.ts` (the meat) and `route.test.ts` (thin smoke tests that the route correctly forwards):

`handlers.test.ts`:
```
describe('handleOrdersFinancialCalculationPost', () => {
  it('should call runOrdersFinancialCalculation with inputs from body')
  it('should call generateReason with the breakdown')
  it('should save with the AI reason included in breakdown.reason')
  it('should return 200 with the saved document')
  it('should return 400 when body is missing a required numeric input')
  it('should return 400 when an input is a string instead of a number')
  it('should return 500 when generateReason throws (error propagates per legacy behavior)')
  it('should return 500 when createOrdersFinancialCalculation throws')
})

describe('handleOrdersFinancialCalculationGet', () => {
  it('should return 200 with the calculations list')
  it('should return 200 with an empty array when no calculations exist')
  it('should pass through orderId + orderCategory to the service function')
})
```

`route.test.ts` (per category):
```
describe('GET /api/orders/new/[orderId]/financial-calculations', () => {
  it('should return 401 when session is missing')
  it('should pass orderCategory="neworders" to the shared handler')
})
describe('POST /api/orders/new/[orderId]/financial-calculations', () => {
  it('should return 401 when session is missing')
  it('should pass orderCategory="neworders" to the shared handler')
})
```

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add API routes for GET/POST financial calculations

Two thin route files (new + maintenance) delegate to shared handlers. Handler
composes runOrdersFinancialCalculation -> generateReason -> createOrdersFinancialCalculation.
Errors propagate (all-or-nothing save), matching legacy patient handler behavior.
```

---

### Step 6 — Client API helpers

**Files**
- `apps/web/utils/api/orders/financialCalculations.ts` — Create
- `apps/web/utils/api/orders/financialCalculations.test.ts` — Create

**What**

Two `fetch` wrappers:

```ts
export async function makeOrdersFinancialCalculation(args: {
  orderId: string;
  orderCategory: 'new' | 'maintenance';
  inputs: CalculationInputs;
}): Promise<OrdersFinancialCalculation>

export async function fetchOrdersFinancialCalculations(args: {
  orderId: string;
  orderCategory: 'new' | 'maintenance';
}): Promise<OrdersFinancialCalculation[]>
```

URLs constructed via existing `buildOrdersUrl` utility (or inline if no helper exists). Body of POST is just `{ inputs: { … } }` — server pulls `author` from the session.

Note that the client passes the frontend-facing `'new'` / `'maintenance'` strings; URL routing puts those in the path, so the API helper just inserts them as URL segments.

**RED phase tests:**

```
describe('makeOrdersFinancialCalculation', () => {
  it('should POST to /api/orders/new/[orderId]/financial-calculations for orderCategory=new')
  it('should POST to /api/orders/maintenance/[orderId]/financial-calculations for orderCategory=maintenance')
  it('should send inputs in the request body')
  it('should return the saved document JSON')
  it('should throw with the response message on non-2xx')
})

describe('fetchOrdersFinancialCalculations', () => {
  it('should GET the correct URL per orderCategory')
  it('should return the list JSON')
  it('should throw with the response message on non-2xx')
})
```

Use `jest.spyOn(global, 'fetch')`.

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add client API helpers
```

---

### Step 7 — Client hooks

**Files**
- `apps/web/utils/hooks/useOrdersFinancialCalculations.ts` — Create
- `apps/web/utils/hooks/useOrdersFinancialCalculations.test.ts` — Create
- `apps/web/utils/hooks/useMakeOrdersFinancialCalculation.ts` — Create
- `apps/web/utils/hooks/useMakeOrdersFinancialCalculation.test.ts` — Create

**What**

`useOrdersFinancialCalculations({ orderId, orderCategory })` — list hook. Uses `useFetchData` (the project's existing data-fetching pattern, same as legacy patient hooks). Exposes `{ calculations, isLoading, error, refetch }`.

`useMakeOrdersFinancialCalculation({ orderId, orderCategory })` — action hook. Mirrors the legacy `useMakeFinancialCalculation` shape: pulls `author` from `useSession()`, uses `useFetchData` for the request, exposes `{ execCalculation, isLoading, data, error }`. On success, the consumer calls the list hook's `refetch`.

No React Context, no global state. Each hook owns its fetch lifecycle. Stage 1's `FinancialCalculationsProvider` was needed because seed state had to be shared across the list and form via `addCalculation` — with an API backend, the form POSTs and the list refetches; no shared state needed.

**RED phase tests:**

```
describe('useOrdersFinancialCalculations', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should fetch calculations on mount with the given orderId+orderCategory')
  it('should set isLoading=true while fetching')
  it('should expose the fetched list as `calculations`')
  it('should expose `error` when the fetch fails')
  it('should refetch when refetch() is called')
})

describe('useMakeOrdersFinancialCalculation', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should call makeOrdersFinancialCalculation with inputs + orderId + orderCategory')
  it('should pull the author email from useSession')
  it('should expose `data` after successful save')
  it('should expose `error` when the save fails')
})
```

**Commit message**
```
[MLID-2307] - feat(orders-financial-calculation): add client hooks for list and create
```

---

### Step 8 — Stage 1 UI integration + cleanup

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.tsx` — Modify
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.tsx` — Modify
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.tsx` — **Delete**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.test.tsx` — **Delete**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/seed.ts` — **Delete**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.ts` — **Delete** (moves to server; client gets the result via the API)
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.test.ts` — **Delete**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/layout.tsx` — **Delete** (no longer wrapping in a provider)
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.tsx` — Modify (props use the new domain type)
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.tsx` — Modify (renders `breakdown.drug` / `.admin` / `.summary` arrays + the `.reason` paragraph)
- Stage 1 component tests — update / replace mocks of `useFinancialCalculations` with mocks of `useOrdersFinancialCalculations` + `useMakeOrdersFinancialCalculation`

**What**

`FinancialReviewIndex.tsx` swaps `useFinancialCalculations` + `getCalculationsForOrder` for `useOrdersFinancialCalculations({ orderId, orderCategory })`. The `orderCategory` is derived from the page's `useParams().category`. Loading state shows a `LoadingSpinner` (legacy pattern). Errors are shown via a `Snackbar` from `@repo/ui` with `severity="error"` and the error message — same pattern Clinical Reviews uses in this subtree.

`Calculator.tsx`:
- Removes the local "live estimate" calculation (which depended on the now-deleted `calculate.ts` client function). For Stage 2, the form submits to the API and the server returns the calculation. The "live estimate" sidebar is dropped per PO ambiguity (see Risks section).
- `addCalculation` flow becomes `execCalculation` from `useMakeOrdersFinancialCalculation`. On success, navigates back to the list (which refetches on mount). On error, surfaces a `Snackbar` and leaves the form open so the user can retry.

`CalculationCard.tsx` and `CalculationBreakdown.tsx` switch their props from the Stage 1 `FinancialCalculation` type to the new `OrdersFinancialCalculation` type, and `CalculationBreakdown` now renders `breakdown.drug` + `breakdown.admin` + `breakdown.summary` as bullet lists, plus the `breakdown.reason` paragraph below.

`FinancialCalculationsContext.tsx`, `seed.ts`, the layout wrapper, and the client `calculate.ts` (+ its test) are deleted.

**RED phase tests** — adjust Stage 1 tests rather than rewrite from scratch:

- `FinancialReviewIndex.test.tsx` — replace `useFinancialCalculations` mock with `useOrdersFinancialCalculations` mock. Add tests for loading and error states.
- `Calculator.test.tsx` — replace context mock with `useMakeOrdersFinancialCalculation` mock. Remove live-estimate assertions (the feature is gone).
- `CalculationCard.test.tsx` / `CalculationBreakdown.test.tsx` — update fixtures to use the new schema shape.

**Commit message**
```
[MLID-2307] - refactor(financial-review): replace in-memory mock with API-backed hooks

Stage 1's FinancialCalculationsContext + seed.ts + client calculate.ts are
deleted. FinancialReviewIndex and Calculator now use useOrdersFinancialCalculations
and useMakeOrdersFinancialCalculation. CalculationBreakdown renders the new
breakdown.drug/admin/summary/reason shape.

Note: the live-estimate sidebar in Calculator is dropped for Stage 2 (out of
scope, was advisory). Tracked separately if PO wants it back.
```

---

### Step 9 — Visual verification (pre-PR gate)

**No code changes.** Mandatory browser-side verification before the PR.

1. Run `npm run dev:web` from repo root.
2. Enable `ORDER_FINANCIAL_CALCULATION` in `/admin/feature-flags`.
3. Open a new order — `/orders-tracker/new/[orderId]`.
4. Click **Financial Review** tab — verify it loads (empty state initially, since no calculations exist yet for this real order).
5. Click **New Calculation** — verify the form renders.
6. Enter values, click **Save** — verify the request fires, the document is saved, and the list refreshes showing the new entry.
7. Open MongoDB Compass (or use the MCP) and confirm a document exists in `ordersFinancialCalculations` with the expected shape — `orderId`, `orderCategory: 'neworders'`, structured `inputs`, structured `result`, `breakdown` with `drug[]`, `admin[]`, `summary[]`, and a non-empty `reason` paragraph from Azure.
8. Expand the calculation card — verify the inputs grid, the bullet sections, and the reason paragraph all render.
9. Repeat steps 3-8 with a **maintenance** order to verify the second route and `orderCategory: 'maintenanceorders'` value.
10. Trigger an error (e.g., temporarily break `AZURE_OPENAI_API_KEY` to force a 500, then save) — verify the Snackbar appears with an error message and the form stays open so the user can retry. Restore the env var.
11. Refresh the page — verify the saved calculations persist (the in-memory bug from Stage 1 is gone).
12. Disable the feature flag — verify the tab disappears and the route renders blank.

**Wait for user's visual sign-off before Step 10.**

---

### Step 10 — PR draft

**Files**
- `docs/agomez/PR/MLID-2307-financial-review-calculator.md` — Create

**What**

The PR draft covers both Stage 1 and Stage 2 (one Jira ticket → one PR). Highlight:

- **What ships**: new Financial Review tab on Order Details v2, gated behind `ORDER_FINANCIAL_CALCULATION` (default `false`).
- **What's new in the DB**: new `ordersFinancialCalculations` collection with documented schema.
- **Coexistence**: legacy `/patient/[id]/financial-calculator` untouched.
- **Notable changes vs. legacy**: structured arrays not markdown blob, numeric not string inputs, graceful AI degradation, fixed prompt typo.
- **Out of scope** (call out explicitly so reviewers don't ask): edit/delete saved calculations, legacy data migration, Azure-deferred enrichment, finance-only role gate, live-estimate sidebar.

**Commit message**
```
[MLID-2307] - docs(financial-review): add PR draft document
```

---

## Files Affected

| File | Action |
|------|--------|
| `apps/web/utils/constants/collections.ts` | Modify (add `OrdersFinancialCalculations` entry) |
| `apps/web/models/OrdersFinancialCalculation.ts` | Create |
| `apps/web/models/OrdersFinancialCalculation.test.ts` | Create |
| `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.ts` | Create (deterministic calc + AI reason + helpers, single file matching legacy structure) |
| `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.test.ts` | Create |
| `apps/web/services/ordersFinancialCalculation/handlers.ts` | Create |
| `apps/web/services/ordersFinancialCalculation/handlers.test.ts` | Create |
| `apps/web/services/mongodb/ordersFinancialCalculation.ts` | Create |
| `apps/web/services/mongodb/ordersFinancialCalculation.test.ts` | Create |
| `apps/web/app/api/orders/new/[orderId]/financial-calculations/route.ts` | Create |
| `apps/web/app/api/orders/new/[orderId]/financial-calculations/route.test.ts` | Create |
| `apps/web/app/api/orders/maintenance/[orderId]/financial-calculations/route.ts` | Create |
| `apps/web/app/api/orders/maintenance/[orderId]/financial-calculations/route.test.ts` | Create |
| `apps/web/utils/api/orders/financialCalculations.ts` | Create |
| `apps/web/utils/api/orders/financialCalculations.test.ts` | Create |
| `apps/web/utils/hooks/useOrdersFinancialCalculations.ts` | Create |
| `apps/web/utils/hooks/useOrdersFinancialCalculations.test.ts` | Create |
| `apps/web/utils/hooks/useMakeOrdersFinancialCalculation.ts` | Create |
| `apps/web/utils/hooks/useMakeOrdersFinancialCalculation.test.ts` | Create |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.test.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.test.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.test.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.test.tsx` | Modify |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.tsx` | **Delete** |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.test.tsx` | **Delete** |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/seed.ts` | **Delete** |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.ts` | **Delete** |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.test.ts` | **Delete** |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/layout.tsx` | **Delete** |
| `docs/agomez/PR/MLID-2307-financial-review-calculator.md` | Create |

**No file under the legacy patient calculator is touched.**

---

## Testing Strategy

- **Mongoose model tests** (Step 1) — schema validation, defaults, sub-document `_id: false`. Use either real connection via `mongodb-memory-server` or pure schema-level validation; check what existing model tests (e.g., `apps/web/models/HospitalSystem.test.ts`) use.
- **Pure function tests** (Step 2) — input/output assertions; deterministic. Port the seed scenarios from Stage 1's `calculate.test.ts` so the new function's outputs match what the Stage 1 UI was already rendering.
- **AI service tests** (Step 3) — mock `AzureOpenAI` constructor + the chat method. Assert prompt content, response handling, graceful failure.
- **Mongo service tests** (Step 4) — integration with the model. Use `mongodb-memory-server` if available.
- **Handlers tests** (Step 5) — unit tests with mocked service-layer dependencies. Cover all branches: success, validation failure, AI failure with calc-save success, total failure.
- **Route tests** (Step 5) — thin smoke tests asserting auth + correct `orderCategory` forwarding to the shared handler.
- **Client API helper tests** (Step 6) — `jest.spyOn(global, 'fetch')`.
- **Hook tests** (Step 7) — `@testing-library/react`'s `renderHook` with mocked API helpers.
- **Stage 1 component test rewrites** (Step 8) — replace context mocks with hook mocks.
- **Manual visual** (Step 9) — 12-step browser walkthrough including the Azure-failure graceful-degradation test.

---

## Security Considerations

- **Authorization**: every route handler requires a NextAuth session (401 otherwise). No role check beyond authenticated — finance-only gate is deferred per the open question.
- **PHI**: calculator inputs are insurance financial parameters; they are not PHI. No patient name, DOB, SSN, or any identifying field is in `inputs`. `author` is the staff user's email, not the patient. No PHI logging needed.
- **Validation**: server-side input validation rejects non-numeric or out-of-range values before they hit Mongoose. Mongoose `min`/`max`/`enum` provides a second line of defense.
- **Azure secrets**: `AZURE_OPENAI_API_KEY` is read from env, never logged or returned. If missing, the handler logs the absence and returns a calculation with empty `reason` — never crashes.
- **Rate limiting**: not implemented; relies on existing app-wide rate limiting (if any). Flag for follow-up if Azure costs spike.

---

## Risks and Open Questions

### Risk — Azure cost / latency spike

Every `POST` triggers an Azure OpenAI call (sync, 1–4s). If a user spam-saves calculations, that's a real cost. Mitigation: rely on existing auth + the natural UX friction (the form takes effort to fill). Flag for monitoring.

### Deferred — live-estimate sidebar

Stage 1's `Calculator.tsx` had a real-time client-side estimate as the user typed (computed by the now-deleted client `calculate.ts`). Stage 2 drops this because the calculation lives server-side. PO has not validated the live-estimate feature — it was a UI-designer proposal, not a spec. **Decision: leave unresolved for now.** If/when PO weighs in we revisit; not a blocker for Stage 2 shipping.

---

## Definition of Done — Stage 2

- [ ] `OrdersFinancialCalculation` Mongoose model created and tested
- [ ] `COLLECTION.OrdersFinancialCalculations` entry added
- [ ] `runOrdersFinancialCalculation` pure function ported with all legacy edge cases passing
- [ ] `generateOrdersFinancialCalculationReason` service with mocked-Azure tests and graceful degradation
- [ ] Mongo service layer with full CRUD tests
- [ ] Both `/api/orders/{new,maintenance}/[orderId]/financial-calculations` routes (GET + POST) with handler + route tests
- [ ] Client API helpers + hooks with tests
- [ ] Stage 1 UI rewired to use new hooks; in-memory context, seed, client calculate, layout deleted
- [ ] All new + modified tests passing: `npm run test`
- [ ] TypeScript strict pass: `npm run types:check` from `apps/web/`
- [ ] Lint clean: `npm run lint`
- [ ] Manual visual sign-off from user (12-step walkthrough including Azure-failure graceful-degradation test)
- [ ] PR draft written to `docs/agomez/PR/MLID-2307-financial-review-calculator.md`
- [ ] PR explicitly notes mock data has been removed, feature flag still defaults to `false` for production rollout, and legacy patient flow is unchanged
