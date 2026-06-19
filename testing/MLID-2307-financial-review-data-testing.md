# MLID-2307 — Data Testing: Order Details Financial Review

Working notebook for end-to-end verification of the Financial Review tab on Order Details v2, backed by the new `ordersFinancialCalculations` collection.

- **Branch:** `feature/MLID-2307-financial-review`
- **Plans:**
  - Stage 1 (UI): `docs/agomez/plans/MLID-2307-financial-review-calculator.md`
  - Stage 2 (data layer): `docs/agomez/plans/MLID-2307-stage2-data-layer.md`
- **Feature flag:** `ORDER_FINANCIAL_CALCULATION` (default `false`)
- **New collection:** `ordersFinancialCalculations`
- **Azure OpenAI graceful degradation:** if `AZURE_OPENAI_ENDPOINT` / `AZURE_OPENAI_API_KEY` are missing or Azure returns an error, the handler logs a `warn` and saves the calculation with `breakdown.reason = ''`. The UI omits the "Reason" section when empty. The save itself never fails for an AI problem; only DB / validation failures return 500.
- **API routes:**
  - `GET  /api/orders/new/[orderId]/financial-calculations`
  - `POST /api/orders/new/[orderId]/financial-calculations`
  - `GET  /api/orders/maintenance/[orderId]/financial-calculations`
  - `POST /api/orders/maintenance/[orderId]/financial-calculations`

---

## Pre-flight

- [ ] `npm run dev:web` running
- [ ] `ORDER_FINANCIAL_CALCULATION` toggled **ON** in `/admin/feature-flags`
- [ ] MongoDB MCP connected to local dev DB

---

## Test orders

> Fill in `_id` (Mongo ObjectId hex string), `displayId` (human-readable), and any notable fields as orders are created. We need at least one `neworders` order and one `maintenanceorders` order to exercise both routes.

### Order 1 — `neworders`

| Field | Value |
|---|---|
| `_id` | `6a178fd41a8f1c052bc0c7b8` |
| `displayId` | `05X7` |
| `orderType` | `New Start` |
| Patient | Alex Test (DOB 1970-05-27, WeInfuse ID 9562) |
| Drug | Actemra (lidrugs `693885079879dc15c6a1de62`) |
| Provider | NPI `1710368154` (JOHN JOHN, Internal Medicine) |
| Created | 2026-05-28T00:44:04Z |
| URL | `/orders-tracker/new/6a178fd41a8f1c052bc0c7b8` |

### Order 2 — `maintenanceorders`

| Field | Value |
|---|---|
| `_id` | _(to fill)_ |
| `displayId` | _(to fill)_ |
| Patient | _(to fill)_ |
| Drug | _(to fill)_ |
| URL | `/orders-tracker/maintenance/<_id>` |

---

## Scenario fixtures

Pre-built input sets to copy into the Calculator form. Expected values reference the deterministic output from `runOrdersFinancialCalculation()`.

### S1 — OOP already met (zero responsibility, fast exit)

| Field | Value |
|---|---|
| Deductible met | `100` |
| Deductible max | `200` |
| Patient co-insurance | `20` |
| OOP met | `500` |
| OOP max | `500` |
| Drug cost | `1000` |
| Admin cost | `200` |
| Assistance for drug | `0` |
| Assistance for admin | `0` |

**Expected result:** `drugResp = 0`, `adminResp = 0`, `totalResp = 0`. `breakdown.summary` contains "OOP max already met". `breakdown.drug` and `breakdown.admin` are empty.

### S2 — Partial deductible + coinsurance (typical case)

| Field | Value |
|---|---|
| Deductible met | `100` |
| Deductible max | `200` |
| Patient co-insurance | `20` |
| OOP met | `50` |
| OOP max | `500` |
| Drug cost | `20` |
| Admin cost | `10` |
| Assistance for drug | `5` |
| Assistance for admin | `15` |

**Expected result (Azure available):** Drug applies $20 to deductible (remaining $80) → responsibility $20 → after $5 assist → `drugResp = 15`. Admin applies $10 to deductible (remaining $70) → responsibility $10 → after $15 assist clamped to $10 → `adminResp = 0`. `totalResp = 15`. `breakdown.reason` is a non-empty paragraph from Azure.

**Expected result (Azure unavailable — graceful degradation):**
- Lands on the list view with the saved row at top, Total Responsibility **$15.00** (Drug $15, Admin $0)
- Expand the row → Inputs grid, Drug bullets, Admin bullets, Summary bullets
- **No "Reason" section** (empty `reason` is conditionally omitted in `CalculationBreakdown.tsx`)
- Server log contains `logger.warn` line: `"generateOrdersFinancialCalculationReason failed — saving calculation without AI reason"` with `orderId`, `orderCategory`, and the underlying Azure error
- MongoDB document in `ordersFinancialCalculations` has `breakdown.reason: ""`

### S3 — No deductible, no coinsurance (everything covered)

| Field | Value |
|---|---|
| Deductible met | `0` |
| Deductible max | `0` |
| Patient co-insurance | `0` |
| OOP met | `0` |
| OOP max | `0` |
| Drug cost | `800` |
| Admin cost | `300` |
| Assistance for drug | `0` |
| Assistance for admin | `0` |

**Expected result:** `drugResp = 0`, `adminResp = 0`, `totalResp = 0`. `breakdown.summary` mentions "No deductible and coinsurance = 0%".

### S4 — OOP cap reached during Drug (Admin skipped)

| Field | Value |
|---|---|
| Deductible met | `0` |
| Deductible max | `0` |
| Patient co-insurance | `100` |
| OOP met | `0` |
| OOP max | `50` |
| Drug cost | `500` |
| Admin cost | `200` |
| Assistance for drug | `0` |
| Assistance for admin | `0` |

**Expected result:** `drugResp = 50` (capped at OOP). `adminResp = 0` (skipped — `breakdown.admin` contains "Skipped calculation because OOP max was reached during Drug"). `totalResp = 50`. `breakdown.drug` contains "OOP max reached".

---

## Walkthrough — Order 1 (neworders)

- [ ] Navigate to Order 1 URL
- [ ] **Financial Review** tab visible alongside Clinical Reviews
- [ ] Click **Financial Review** — empty-state card renders ("No financial calculations yet")
- [ ] Click **Start a new calculation** — Calculator form renders
- [ ] Fill in **S2** values, click **Save calculation** (~1–4 s for Azure round-trip)
- [ ] Lands back on the list — saved row appears at top with **Latest** chip
- [ ] Expand the row — verify Inputs grid, Drug bullets, Admin bullets, Summary bullets, **Reason** paragraph from Azure
- [ ] Click the copy icon — check icon appears for ~1.5 s
- [ ] Hard refresh — saved row persists (real backend, not in-memory anymore)
- [ ] **MongoDB MCP**: verify the document in `ordersFinancialCalculations` matches expected shape (Order 1 `_id` as `orderId`, `orderCategory: 'neworders'`, 9 numeric `inputs`, 3 numeric `result`, `breakdown.{drug,admin,summary,reason}`, `author`, `calculationDate`, `createdAt`, `updatedAt`)
- [ ] **MongoDB MCP**: verify compound index `{ orderId: 1, orderCategory: 1, calculationDate: -1 }` exists on the collection
- [ ] Save **S1**, **S3**, **S4** in sequence — newest at top each time, **Latest** chip moves to the newest

## Walkthrough — Order 2 (maintenanceorders)

- [ ] Navigate to Order 2 URL
- [ ] Click **Financial Review** tab — empty state (Order 2 has no calculations yet)
- [ ] Confirm Order 1's calculations do NOT appear here (per-order scoping verified)
- [ ] Save **S2** — verify it lands and that **MongoDB MCP** shows `orderCategory: 'maintenanceorders'`

## Error path — Azure OpenAI failure (graceful degradation, not a 500)

With the graceful-degradation contract in place, an Azure outage is **not** an error path for the user — the calculation still saves with `reason: ""` and the UI omits the Reason section. The only visible artifact is a `logger.warn` in server output. To exercise the **true** error path (DB-level failure), kill the Mongo connection mid-save instead.

- [x] No `AZURE_OPENAI_API_KEY` set → save still landed with `breakdown.reason: ""` and a `warn` log (verified 2026-05-28, see "Confirmed run" below)
- [ ] (DB-failure variant) Stop Mongo, click Save → expect 500 from `POST` → Calculator shows "Failed to save calculation. Please try again." → form stays open with values intact

## Flag gating

- [ ] Disable `ORDER_FINANCIAL_CALCULATION` in `/admin/feature-flags`
- [ ] Financial Review tab disappears from the panel
- [ ] Direct URL to `/orders-tracker/new/<id>/financial-review` renders blank

---

## Confirmed run — 2026-05-28 (S2 against Order 1, Azure-unavailable path)

End-to-end POST flow exercised against Alex Test's Actemra order with **no `AZURE_OPENAI_API_KEY` set**. The save still landed cleanly, proving the graceful-degradation contract.

### Stored document (`ordersFinancialCalculations`)

```jsonc
{
  "_id": ObjectId("6a1795c2ac322eae929bfb16"),
  "orderId": ObjectId("6a178fd41a8f1c052bc0c7b8"),
  "orderCategory": "neworders",
  "inputs": {
    "deductibleMet": 100,
    "deductibleMax": 200,
    "coinsurance": 20,
    "oopMet": 50,
    "oopMax": 500,
    "drugCost": 20,
    "adminCost": 10,
    "drugAssist": 5,
    "adminAssist": 15
  },
  "result": {
    "drugResp": 15,
    "adminResp": 0,
    "totalResp": 15
  },
  "breakdown": {
    "drug": [
      "Starting with cost $20.00.",
      "Apply to deductible: $20.00 (remaining deductible now $80.00).",
      "Responsibility before assistance: $20.00.",
      "Responsibility counted toward OOP (pre-assist): $20.00.",
      "Apply drug assistance: $5.00.",
      "Patient pays for drug: $15.00.",
      "OOP remaining decreased by $20.00: $450.00 → $430.00."
    ],
    "admin": [
      "Starting with cost $10.00.",
      "Apply to deductible: $10.00 (remaining deductible now $70.00).",
      "Responsibility before assistance: $10.00.",
      "Responsibility counted toward OOP (pre-assist): $10.00.",
      "Apply admin assistance: $10.00.",
      "Patient pays for admin: $0.00.",
      "OOP remaining decreased by $10.00: $430.00 → $420.00."
    ],
    "summary": [
      "Patient Drug Responsibility: $15.00",
      "Patient Admin Responsibility: $0.00",
      "Total Responsibility: $15.00"
    ],
    "reason": ""
  },
  "author": "alejandro.gomez@fortegrp.com",
  "calculationDate": ISODate("2026-05-28T01:09:22.489Z"),
  "createdAt": ISODate("2026-05-28T01:09:22.494Z"),
  "updatedAt": ISODate("2026-05-28T01:09:22.494Z"),
  "__v": 0
}
```

### Indexes auto-built on first insert

```
_id_                                              { _id: 1 }
orderId_1_orderCategory_1_calculationDate_-1     { orderId: 1, orderCategory: 1, calculationDate: -1 }
```

### Verifications

- ✅ Inputs stored as numbers (not strings) — schema validation passing
- ✅ ObjectId conversion from hex string in `createOrdersFinancialCalculation` works
- ✅ `breakdown.reason` = `""` confirms the graceful-degradation try/catch in `handlers.ts` fired
- ✅ `author` pulled from NextAuth session
- ✅ `createdAt` / `updatedAt` populated by Mongoose `timestamps: true`
- ✅ Compound index registered automatically — no migration needed

---

## Notes / findings

### PR-time follow-ups (carry into the PR draft)

- **AZURE_OPENAI_API_VERSION terraform gap fix** — discovered while wiring local Azure auth: `process.env.AZURE_OPENAI_API_VERSION` has been read by code since 2025-06-27 (Anatoliy Kolodkin, commit `96f52b3e5`, "Updates API to use Azure OpenAI integration") but was never added to `terraform/main.tf`. The legacy patient calculator (Dvir Rassovsky, commit `2f0817062`, MLID-925) inherited the same omission. Stage and prod have been running on the code-side default `'2024-08-01-preview'` for ~11 months — functionally fine but a latent risk if the default ever drifts. This PR fixes it: adds `AZURE_OPENAI_API_VERSION = "2024-08-01-preview"` to both container app blocks (web + worker) in `terraform/main.tf` and to `terraform/docs/ContainerAppSetup.md`. **Add Anatoliy Kolodkin as a reviewer** to surface the catch.

_(Free space for additional issues you spot during testing — add subsections as needed.)_
