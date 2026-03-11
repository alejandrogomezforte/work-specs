# Pharmacy Eligibility — Analysis

> Part of [MLID-1492 Plan](../plans/MLID-1492-plan-progress.md) — Deliverable 2: Pharmacy Eligible Calculation

---

## Involved Collections

Six collections participate in the pharmacy eligibility calculation. Only fields relevant to the calculation are listed.

### `neworders` (source)

| Field | Type | Purpose |
|-------|------|---------|
| `drug` | string | ObjectId hex referencing `lidrugs._id` |
| `site` | string | ObjectId hex referencing `patientLocations._id` |
| `patient` | string | ObjectId hex referencing `patients._id` |

### `lidrugs` (hop 1 — from order.drug)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.drug` |
| `pharmacyEligible` | boolean \| undefined | Whether the drug is pharmacy eligible |

### `patientLocations` (hop 1 — from order.site)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.site` |
| `isPharmacyEligible` | boolean \| undefined | Whether the location is pharmacy eligible |

### `patients` (hop 1 — from order.patient)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.patient` |
| `we_infuse_id_IV` | number | Bridge to `patient_insurances` |

### `patient_insurances` (hop 2 — from patients.we_infuse_id_IV)

| Field | Type | Purpose |
|-------|------|---------|
| `we_infuse_patient_id` | number | Matched from `patients.we_infuse_id_IV` |
| `priority` | string | `'1'` = primary insurance |
| `status` | string | `'Active'` = current active record |
| `payer_type_name` | string | Insurance type (e.g., `"Medicare"`, `"Commercial"`) |
| `insurance_plan_name` | string | Bridge to `insurance_plans.planName` |

### `insurance_plans` (hop 3 — from patient_insurances.insurance_plan_name)

| Field | Type | Purpose |
|-------|------|---------|
| `planName` | string | Matched from `patient_insurances.insurance_plan_name` |
| `isPharmacyEligible` | boolean | Whether the insurance plan is pharmacy eligible |

### Connections

```
neworders.drug     ─── ObjectId     ──► lidrugs._id
neworders.site     ─── ObjectId     ──► patientLocations._id
neworders.patient  ─── ObjectId     ──► patients._id
patients.we_infuse_id_IV ─── number ──► patient_insurances.we_infuse_patient_id
                                         (filtered by priority = '1' AND status = 'Active')
patient_insurances.insurance_plan_name ─── string match ──► insurance_plans.planName
```

---

## Eligibility Resolution

The pharmacy eligibility calculation returns a **string** with 3 possible outcomes:

| Outcome | Condition |
|---------|-----------|
| `"Waiting on Insurance Input"` | Patient has no primary active insurance |
| `"Yes"` | Drug is pharmacy eligible **AND** location is pharmacy eligible **AND** insurance condition is met (see below) |
| `"No"` | Any of the above criteria is not met |

```
isPharmacyEligible = (order) => string
```

The insurance condition has two paths:
- `payer_type_name` is `"Medicare"` or `"Medicare Advantage"` → **automatically passes** (no further check needed)
- `payer_type_name` is `"Commercial"` or `"Medicaid"` → passes **only if** `insurance_plans.isPharmacyEligible == true`

The function depends on three sub-checks:

```
isPharmacyEligible(order) = f(
    drugIsPharmacyEligible(order),
    locationIsPharmacyEligible(order),
    insuranceMeetsPharmacyCriteria(order)
)
```

---

### `drugIsPharmacyEligible(order)`

**Question:** Is the order's drug marked as pharmacy eligible?

**Input:** `order.drug` (string, ObjectId hex)

**Collections:** `neworders` → `lidrugs`

**Chain:** `neworders.drug` → `lidrugs._id` (ObjectId match)

```typescript
function drugIsPharmacyEligible(order): boolean {
    const drug = lidrugs.findOne({ _id: ObjectId(order.drug) })
    return drug.pharmacyEligible === true
}
```

---

### `locationIsPharmacyEligible(order)`

**Question:** Is the order's location marked as pharmacy eligible?

**Input:** `order.site` (string, ObjectId hex)

**Collections:** `neworders` → `patientLocations`

**Chain:** `neworders.site` → `patientLocations._id` (ObjectId match)

```typescript
function locationIsPharmacyEligible(order): boolean {
    const location = patientLocations.findOne({ _id: ObjectId(order.site) })
    return location.isPharmacyEligible === true
}
```

---

### `insuranceMeetsPharmacyCriteria(order)`

**Question:** Does the patient's primary active insurance meet the pharmacy eligibility criteria?

**Input:** `order.patient` (string, ObjectId hex)

**Collections:** `neworders` → `patients` → `patient_insurances` → `insurance_plans`

**Chain:**
1. `neworders.patient` → `patients._id` (ObjectId match)
2. `patients.we_infuse_id_IV` → `patient_insurances.we_infuse_patient_id` (number match, filtered by `priority = '1'` AND `status = 'Active'`)
3. `patient_insurances.insurance_plan_name` → `insurance_plans.planName` (string match) — **only when payer type is `"Commercial"` or `"Medicaid"`**

```typescript
function insuranceMeetsPharmacyCriteria(order): 'yes' | 'no' | 'no_insurance' {
    // Hop 1: order.patient → patients
    const patient = patients.findOne({ _id: ObjectId(order.patient) })

    // Hop 2: patients.we_infuse_id_IV → patient_insurances (primary + active)
    const primaryInsurance = patient_insurances.findOne({
        we_infuse_patient_id: patient.we_infuse_id_IV,
        priority: '1',
        status: 'Active'
    })

    if (primaryInsurance == null) {
        return 'no_insurance'
    }

    const type = primaryInsurance.payer_type_name

    // Medicare or Medicare Advantage → automatically passes
    if (type === 'Medicare' || type === 'Medicare Advantage') {
        return 'yes'
    }

    // Commercial or Medicaid → need to check insurance plan's pharmacy eligibility
    if (type === 'Commercial' || type === 'Medicaid') {
        // Hop 3: patient_insurances.insurance_plan_name → insurance_plans.planName
        const plan = insurance_plans.findOne({
            planName: primaryInsurance.insurance_plan_name
        })

        if (plan != null && plan.isPharmacyEligible === true) {
            return 'yes'
        }
    }

    return 'no'
}
```

---

### `isPharmacyEligible(order)` — combined calculation

```typescript
function isPharmacyEligible(order): string {
    const insurance = insuranceMeetsPharmacyCriteria(order)

    // No primary active insurance → can't determine
    if (insurance === 'no_insurance') {
        return "Waiting on Insurance Input"
    }

    // All three must be true for "Yes"
    if (
        drugIsPharmacyEligible(order)
        && locationIsPharmacyEligible(order)
        && insurance === 'yes'
    ) {
        return "Yes"
    }

    return "No"
}
```

### Resolution diagram

```
order
│
├── 1. drugIsPharmacyEligible()
│   │   order.drug: "64abc123def456"
│   └──► lidrugs._id → pharmacyEligible: true
│
├── 2. locationIsPharmacyEligible()
│   │   order.site: "64abc123def789"
│   └──► patientLocations._id → isPharmacyEligible: true
│
├── 3. insuranceMeetsPharmacyCriteria()
│   │   order.patient: "64abc123def012"
│   │
│   ├──► patients._id → we_infuse_id_IV: 378038
│   ├──► patient_insurances (WHERE we_infuse_patient_id == 378038 AND priority == '1' AND status == 'Active')
│   │    ├── payer_type_name: "Commercial"
│   │    └── insurance_plan_name: "Aetna HMO Gold"
│   │
│   │   payer_type_name == "Medicare" or "Medicare Advantage"?  ──► 'yes' (skip hop 3)
│   │   payer_type_name == "Commercial" or "Medicaid"?          ──► check hop 3:
│   │
│   └──► insurance_plans (WHERE planName == "Aetna HMO Gold")
│        └── isPharmacyEligible: true  ──► 'yes'
│
└── isPharmacyEligible()
    drug eligible?      ── true
    location eligible?  ── true
    insurance criteria? ── 'yes'
    ──► "Yes"
```

---

## Recalculation Triggers

The pharmacy eligibility result is stored as a field on each order document (`neworders`).

**Tables involved:**
- **Destiny:** `neworders` (where the calculated result is stored)
- **Sources:** `lidrugs`, `patientLocations`, `patient_insurances`, `insurance_plans` (lookup tables used during calculation)

Triggers fire when the destiny table or any source table is updated. Each trigger has different **known data** at trigger time and different **fan-out** characteristics.

**Note:** Each `isPharmacyEligible()` call costs **4–5 DB reads** (1 `lidrugs` + 1 `patientLocations` + 1 `patients` + 1 `patient_insurances` + conditionally 1 `insurance_plans`), compared to 3 reads for 340B eligibility.

---

### Trigger 1 — Order created

**When:** A new order is inserted via the web app.

**Source:** UI action (user creates an order in the Orders Tracker)

**Entry points:**
- `POST /api/orders/new` → `createOrder()` in `services/mongodb/ordersTracker.ts` (line 779)
- `POST /api/orders/maintenance` → same `createOrder()` function

**Known data at trigger time:** The full order document (including `drug`, `site`, and `patient`).

**Lookup chain:** None — we already have the order, go straight to `isPharmacyEligible(order)`.

**Volume:**

| Metric | Value |
|--------|-------|
| Orders affected | 1 |
| `isPharmacyEligible()` calls | 1 |
| DB reads (inside calculation) | 1 `lidrugs` + 1 `patientLocations` + 1 `patients` + 1 `patient_insurances` + (conditionally 1 `insurance_plans`) = **4–5** |
| DB writes | 1 (`$set` on the order) |
| **Total DB operations** | **5–6** |

**Strategy:** Inline — calculate during the API request. Negligible cost.

---

### Trigger 2 — Order updated (key fields changed)

**When:** An existing order's `drug`, `site`, or `patient` field changes via the web app.

**Source:** UI action (user edits an order in the Orders Tracker)

**Entry points:**
- `PUT /api/orders/new` → `updateOrder()` in `services/mongodb/ordersTracker.ts` (line 854)
- `PUT /api/orders/maintenance` → same `updateOrder()` function

**Known data at trigger time:** The updated order document and the changed fields.

**Lookup chain:** None — we already have the order. Only recalculate if `drug`, `site`, or `patient` changed.

**Volume:**

| Metric | Value |
|--------|-------|
| Orders affected | 1 (or 2 if protocol partner is linked) |
| `isPharmacyEligible()` calls | 1–2 |
| DB reads (inside calculation) | 4–10 |
| DB writes | 1–2 |
| **Total DB operations** | **5–12** |

**Strategy:** Inline — calculate during the API request. Negligible cost.

---

### Trigger 3 — Drug `pharmacyEligible` flag updated

**When:** An admin toggles the "L.I. Pharmacy Eligible" flag on a drug in the Drug Admin panel.

**Source:** UI action (admin toggles drug pharmacy eligibility)

**Entry point:** `PUT /api/admin/lidrugs/[id]` → `updateLiDrug()` in `services/mongodb/liDrugs.ts` (line 387)

**Known data at trigger time:** The drug ID. The update is a single drug toggle (no bulk update endpoint exists for this flag).

**Lookup chain:**

```
drugId
│
├──► find all neworders WHERE drug = drugId → N orders
└──► for each of N orders → isPharmacyEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Drugs affected | 1 |
| Orders to find | `neworders` WHERE `drug` = drugId |
| Orders affected | N (depends on how many orders use this drug) |
| `isPharmacyEligible()` calls | N |
| DB reads (find orders) | 2 queries |
| DB reads (inside calculation) | (N) × 4–5 |
| DB writes | N |
| **Total DB operations** | **2 + (N) × 5–6** |

**Concerns:**
- Popular drugs could be referenced by many orders, leading to high fan-out
- Single drug update, so no diff needed — just recalculate all orders using that drug

**Strategy:** Background job (Pulse). Single drug but potentially high fan-out.

---

### Trigger 4 — Location `isPharmacyEligible` flag updated

**When:** An admin toggles the "Pharmacy Eligible" checkbox on a location in the Locations admin panel.

**Source:** UI action (admin toggles location pharmacy eligibility)

**Entry point:** `PUT /api/admin/locations/[id]` → `updateLocationInDB()` in `services/locations/locationDbService.ts` (line 154)

**Known data at trigger time:** The location ID. The update is a single location toggle.

**Lookup chain:**

```
locationId
│
├──► find all neworders WHERE site = locationId → N orders
└──► for each of N orders → isPharmacyEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Locations affected | 1 |
| Orders to find | `neworders` WHERE `site` = locationId |
| Orders affected | N (depends on how many orders are at this location) |
| `isPharmacyEligible()` calls | N |
| DB reads (find orders) | 2 queries |
| DB reads (inside calculation) | (N) × 4–5 |
| DB writes | N |
| **Total DB operations** | **2 + (N) × 5–6** |

**Concerns:**
- High-volume locations could have many orders, leading to significant fan-out

**Strategy:** Background job (Pulse). Single location but potentially high fan-out.

---

### Trigger 5 — Insurance plan `isPharmacyEligible` flag updated (UI)

**When:** An admin toggles the pharmacy eligibility flag on an insurance plan in the Insurance Plans admin panel.

**Source:** UI action (admin toggles insurance plan pharmacy eligibility)

**Entry point:** `PUT /api/insurance-plans/[id]` → `updateInsurancePlanPharmacyEligible()` in `services/insurancePlans/insurancePlanService.ts` (line 132)

**Known data at trigger time:** The plan ID and the new `isPharmacyEligible` value.

**Lookup chain:**

This trigger has the deepest reverse lookup chain — we need to go from plan → patients → orders (3-hop reverse):

```
planId
│
├──► fetch insurance plan → planName
│
├──► find patient_insurances WHERE insurance_plan_name = planName
│    AND priority = '1' AND status = 'Active'
│    → list of we_infuse_patient_id values (D distinct patients)
│
├──► find patients WHERE we_infuse_id_IV IN [...D patient IDs]
│    → list of patient._id values
│
├──► find neworders WHERE patient IN [...patient._id list] → N orders
└──► for each of N orders → isPharmacyEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Plans affected | 1 |
| Patients using this plan as primary | D (could be many for popular plans) |
| Orders to find | `neworders` WHERE `patient` IN [D patient IDs] |
| Orders affected | N |
| `isPharmacyEligible()` calls | N |
| DB reads (fetch plan) | 1 |
| DB reads (find patient_insurances) | 1 query |
| DB reads (find patients) | 1 query with `$in` over D IDs |
| DB reads (find orders) | 2 queries with `$in` over D patient IDs |
| DB reads (inside calculation) | (N) × 4–5 |
| DB writes | N |
| **Total DB operations** | **5 + (N) × 5–6** |

**Concerns:**
- Popular insurance plans (e.g., "Aetna HMO Gold") could be the primary plan for hundreds of patients
- The 3-hop reverse lookup means 5 DB reads just to find affected orders before calculation even starts
- Only affects patients with Commercial or Medicaid payer types (Medicare/Medicare Advantage auto-pass, so changing the plan flag doesn't affect them) — but we still recalculate all matched orders since the full calculation handles this logic

**Strategy:** Background job (Pulse). Deepest lookup chain and potentially high fan-out.

---

### Trigger 6 — Patient insurance updated via webhook

**When:** Looker sends a patient insurance webhook (insurance created, updated, or status changed in WeInfuse).

**Source:** Backend webhook (Looker → WeInfuse sync)

**Entry point:** `POST /api/webhooks/looker_patient_insurances` → `webhookUpsertPatientInsurances()` in `services/webhooks/handlers/util/webhookUpsertPatientInsurances.ts`

**Known data at trigger time:** An array of `WebhookPatientInsurance` records. Each record contains `orders.patient_id` (which maps to `we_infuse_patient_id`). The payload has **no batch size limit** and records are processed one at a time via `findOneAndUpdate`.

**Lookup chain:**

```
webhook payload (R records, belonging to D distinct patients)
│
├──► deduplicate by orders.patient_id → D unique patient IDs (we_infuse_patient_id)
│
├──► for each patient_id:
│    ├──► query patients WHERE we_infuse_id_IV = patient_id → patient._id
│    ├──► query neworders WHERE patient = patient._id → N orders
│    └──► for each of N orders → isPharmacyEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Records in webhook | R (no limit) |
| Distinct patients | D (≤ R, after dedup by `orders.patient_id`) |
| Orders per patient | Typically small (1–5), but unbounded |
| `isPharmacyEligible()` calls | D × (avg orders per patient) |
| DB reads (find patients) | D × 1 query on `patients` |
| DB reads (find orders) | D × 2 queries on `neworders` |
| DB reads (inside calculation) | total_orders × 4–5 |
| DB writes | total_orders |
| **Total DB operations** | **D × 3 + total_orders × 5–6** |

**Concerns:**
- The webhook payload can contain many records (hundreds) across many patients
- Each distinct patient requires 3 lookup queries before calculation even starts
- Pharmacy calculation costs more per order (4–5 reads) than 340B (3 reads)

**Strategy:** Background job (Pulse). One job per distinct patient to parallelize.

---

### Trigger 7 — Insurance plan `isPharmacyEligible` updated via webhook

**When:** Looker sends an insurance plans webhook that syncs plan data from WeInfuse, including the `isPharmacyEligible` flag.

**Source:** Backend webhook (Looker → WeInfuse sync)

**Entry point:** `POST /pages/api/webhooks/insurance_plans` → `webhookUpsertInsurancePlans()` in `services/webhooks/handlers/util/webhookUpsertInsurancePlans.ts` (line 18)

**Known data at trigger time:** A bulk payload of insurance plan records. The webhook handler does a `bulkWrite` with `$set: { isPharmacyEligible: plan.isPharmacyEligible }` for each plan, matched by `planName`. The payload can contain many plans at once.

**Diff strategy:**

Since the webhook does a full `$set` on `isPharmacyEligible` for each plan, we need to detect which plans actually had their value **changed**. Only plans where the old and new `isPharmacyEligible` differ need to trigger recalculation.

```
webhook payload (W plans)
│
├──► fetch current insurance_plans WHERE planName IN [...W planNames]
│    → build map of planName → current isPharmacyEligible
│
├──► compare old vs new for each plan
│    changedPlans = plans WHERE old.isPharmacyEligible ≠ new.isPharmacyEligible
│
│    changedPlans is empty? ──► no-op
│
├──► perform the bulk upsert (write new values)
│
├──► for each changed plan:
│    ├──► find patient_insurances WHERE insurance_plan_name = planName
│    │    AND priority = '1' AND status = 'Active'
│    │    → list of we_infuse_patient_id values
│    ├──► find patients WHERE we_infuse_id_IV IN [...]
│    │    → list of patient._id values
│    ├──► find neworders WHERE patient IN [...] → N orders
│    └──► for each of N orders → isPharmacyEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Plans in webhook | W |
| Plans with changed `isPharmacyEligible` | C (≤ W, after diff) |
| Patients per changed plan | D_i (varies per plan) |
| Total patients affected | sum of D_i across C plans |
| Orders affected | total patients × avg orders per patient |
| DB reads (fetch old state) | 1 query (bulk fetch current plans) |
| DB reads per changed plan | 3 (patient_insurances + patients + orders) |
| DB reads (inside calculation) | total_orders × 4–5 |
| DB writes | total_orders |
| **Total DB operations** | **1 + C × 3 + total_orders × 5–6** |

**Concerns:**
- Combines the deepest lookup chain (Trigger 5) with bulk scope (multiple plans)
- Each changed plan triggers a full 3-hop reverse lookup to find affected orders
- The diff detection requires fetching all current plan states before the webhook write
- Most webhook syncs likely don't change `isPharmacyEligible` values, so the diff typically results in few or zero recalculations

**Strategy:** Background job (Pulse). Most expensive trigger — deep chain with bulk scope.

---

### Volume comparison

```
Trigger    Source     Known Data          Fan-out        Lookups to find orders    Cost per trigger
────────   ────────  ──────────────────  ─────────────  ────────────────────────  ─────────────────────
1. Order   UI        order               1:1            0 (already have it)       5–6 ops (trivial)
   create

2. Order   UI        order + changed     1:1            0 (already have it)       5–12 ops (trivial)
   update              fields

3. Drug    UI        drugId              1 drug         2 queries (orders by       2 + N × 5–6 ops
   flag                                  → N        drug)                     ⚠️ MODERATE
   update                                orders

4. Loc.    UI        locationId          1 location     2 queries (orders by       2 + N × 5–6 ops
   flag                                  → N        site)                     ⚠️ MODERATE
   update                                orders

5. Plan    UI        planId              1 plan         5 queries (plan →          5 + N × 5–6 ops
   flag                                  → D patients   patient_insurances →      ⚠️ MODERATE-HIGH
   update                                → N        patients → orders)        (deep chain)
                                          orders

6. Ins.    Webhook   R records with      R → D          D reads (patients) +      D×3 + total×5–6 ops
   webhook            we_infuse_patient   patients       D×2 reads (orders by      ⚠️ MODERATE-HIGH
                      _id                 → orders       patient)

7. Plan    Webhook   W plans with        W → C changed  1 read (old state) +      1 + C×3 + total×5–6
   webhook            isPharmacy          → D patients   C × 3-hop reverse         ⚠️ HIGH
                      Eligible            → orders       lookups                   (deep chain + bulk)
```

### Indexes creation

The `neworders` collection is a raw MongoDB collection (no Mongoose model) with no existing indexes on the fields used by recalculation triggers. The following indexes must be created:

| Index | Used by | Query pattern |
|-------|---------|---------------|
| `neworders: { drug: 1 }` | Trigger 3 (drug flag update) | `WHERE drug = drugId` |
| `neworders: { site: 1 }` | Trigger 4 (location flag update) | `WHERE site = locationId` |

**Note:** The `patient` and `provider` indexes needed by shared triggers (order update, insurance webhook) are covered in the [340B eligibility analysis](./340B-eligibility.md#indexes-creation).

---

### Optimization notes (for later)

The volume analysis reveals several areas that will need attention during implementation:

1. **~~Missing indexes on `neworders.drug` and `neworders.site`~~** — Addressed in the Indexes creation section above.
2. **Higher per-order cost than 340B** — Each `isPharmacyEligible()` call costs 4–5 DB reads (vs 3 for 340B). This amplifies the cost of high fan-out triggers.
3. **Deep reverse lookup for insurance plan changes** — Triggers 5 and 7 require a 3-hop reverse lookup (plan → patient_insurances → patients → orders) just to find affected orders. This is the deepest chain in either eligibility calculation.
4. **Unbounded webhook payloads** — Triggers 6 and 7 can contain many records. Trigger 7 is especially costly because each changed plan triggers the 3-hop reverse lookup.
5. **Shared trigger with 340B** — Trigger 6 (patient insurance webhook) affects both 340B and pharmacy eligibility. The recalculation for both should happen together to avoid processing the same webhook twice.
6. **Drug and location fan-out** — Triggers 3 and 4 are simple lookups but popular drugs or high-volume locations could affect many orders. Consider batching the recalculations.
