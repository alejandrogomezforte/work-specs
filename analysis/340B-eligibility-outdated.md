# 340B Eligibility — Analysis

> Part of [MLID-1492 Plan](../plans/MLID-1492-plan-progress.md) — Deliverable 1: 340B Calculation

---

## Involved Collections

Four collections participate in the 340B eligibility calculation. Only fields relevant to the calculation are listed.

### `neworders` (source)

| Field | Type | Purpose |
|-------|------|---------|
| `provider` | string | 10-digit NPI (e.g., `"1234567890"`) |
| `patient` | string | ObjectId hex referencing `patients._id` |

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
| `payer_type_name` | string | Insurance type (e.g., `"Commercial"`) |

### `hospitalSystems` (independent lookup — from order.provider)

| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Hospital system name (displayed in result) |
| `providers` | `{ npi: string, name: string }[]` | NPI list to match against `order.provider` |
| `deletedAt` | Date \| null | `null` = active (soft-delete filter) |

### Connections

```
neworders.provider ─── string match ──► hospitalSystems.providers[].npi
neworders.patient  ─── ObjectId     ──► patients._id
patients.we_infuse_id_IV ─── number ──► patient_insurances.we_infuse_patient_id
                                         (filtered by priority = '1' AND status = 'Active')
```

---

## Eligibility Resolution

The 340B eligibility calculation returns a **string** with 3 possible outcomes:

| Outcome | Condition |
|---------|-----------|
| `"Waiting on Insurance Input"` | Provider NPI is in a hospital system list, but the patient has no primary active insurance |
| `"Hospital Name 1, Hospital Name 2"` | Provider NPI is in a hospital system list **AND** primary active insurance type is `"Commercial"` |
| `"No"` | Provider NPI is not in any hospital system list, **OR** primary insurance type is not `"Commercial"` |

```
is340BEligible = (order) => string
```

The function depends on two sub-checks:

```
is340BEligible(order) = f( providerIsIn340BHospitalsList(order), getPatientPrimaryInsuranceType(order) )
```

---

### `providerIsIn340BHospitalsList(order)`

**Question:** Is the order's provider NPI found in at least one active hospital system's providers list?

**Input:** `order.provider` (string, 10-digit NPI)

**Collections:** `neworders` → `hospitalSystems`

**Chain:** `neworders.provider` == `hospitalSystems.providers[].npi` (direct string match)

```typescript
function providerIsIn340BHospitalsList(order): { matched: boolean, systemNames: string[] } {
    // Find all active hospital systems that include this NPI in their providers list
    const matchingSystems = hospitalSystems.find({
        deletedAt: null,
        'providers.npi': order.provider
    })

    return {
        matched: matchingSystems.length > 0,
        systemNames: matchingSystems.map(hs => hs.name)
        // e.g., ["Hospital System ABC", "Hospital System XYZ"]
    }
}
```

**Notes:**
- One NPI can match 0, 1, or many hospital systems.
- Only active systems (`deletedAt == null`) are considered.
- The result carries the system names because the UI column displays them.

---

### `getPatientPrimaryInsuranceType(order)`

**Question:** What is the payer type of the order's patient's primary active insurance?

**Input:** `order.patient` (string, ObjectId hex)

**Collections:** `neworders` → `patients` → `patient_insurances`

**Chain:**
1. `neworders.patient` → `patients._id` (ObjectId match)
2. `patients.we_infuse_id_IV` → `patient_insurances.we_infuse_patient_id` (number match, filtered by `priority = '1'` AND `status = 'Active'`)

```typescript
function getPatientPrimaryInsuranceType(order): string | null {
    // Hop 1: order.patient → patients
    const patient = patients.findOne({ _id: ObjectId(order.patient) })

    // Hop 2: patients.we_infuse_id_IV → patient_insurances (primary + active)
    const primaryInsurance = patient_insurances.findOne({
        we_infuse_patient_id: patient.we_infuse_id_IV,
        priority: '1',
        status: 'Active'
    })

    if (primaryInsurance == null) {
        return null  // no primary active insurance
    }

    return primaryInsurance.payer_type_name  // e.g., "Commercial", "Medicare", etc.
}
```

---

### `is340BEligible(order)` — combined calculation

```typescript
function is340BEligible(order): string {
    const hospitals = providerIsIn340BHospitalsList(order)

    // If provider NPI is not in any hospital system → "No" regardless of insurance
    if (!hospitals.matched) {
        return "No"
    }

    // Provider IS in a hospital system — now check insurance
    const insuranceType = getPatientPrimaryInsuranceType(order)

    // No primary active insurance → we can't determine yet
    if (insuranceType == null) {
        return "Waiting on Insurance Input"
    }

    // Provider in hospital list AND insurance is Commercial → eligible
    if (insuranceType === "Commercial") {
        return hospitals.systemNames.join(", ")
        // e.g., "Hospital System ABC, Hospital System XYZ"
    }

    // Provider in hospital list but insurance is not Commercial
    return "No"
}
```

### Resolution diagram

```
order
│
├── 1. providerIsIn340BHospitalsList()
│   │   order.provider: "2222222222"
│   │
│   └──► hospitalSystems (WHERE providers[].npi == "2222222222" AND deletedAt == null)
│        ├── match: "Hospital System ABC"
│        └── match: "Hospital System XYZ"
│        → matched: true, systemNames: ["Hospital System ABC", "Hospital System XYZ"]
│
│   matched == false? ──► return "No"
│   matched == true?  ──► continue to step 2
│
├── 2. getPatientPrimaryInsuranceType()
│   │   order.patient: "64abc123def456"
│   │
│   ├──► patients._id → we_infuse_id_IV: 378038
│   └──► patient_insurances (WHERE we_infuse_patient_id == 378038 AND priority == '1' AND status == 'Active')
│        └── payer_type_name: "Commercial"
│
│   insuranceType == null?         ──► return "Waiting on Insurance Input"
│   insuranceType == "Commercial"? ──► return "Hospital System ABC, Hospital System XYZ"
│   insuranceType != "Commercial"? ──► return "No"
│
└── Result: "Hospital System ABC, Hospital System XYZ"
```

---

## Recalculation Triggers

The 340B eligibility result is stored as a field on each order document (`neworders`).

**Tables involved:**
- **Destiny:** `neworders` (where the calculated result is stored)
- **Sources:** `hospitalSystems`, `patient_insurances` (lookup tables used during calculation)

Triggers fire when the destiny table or either source table is updated. Each trigger has different **known data** at trigger time and different **fan-out** characteristics.

---

### Trigger 1 — Order created

**When:** A new order is inserted via the web app.

**Source:** UI action (user creates an order in the Orders Tracker)

**Entry points:**
- `POST /api/orders/new` → `createOrder()` in `services/mongodb/ordersTracker.ts` (line 779)
- `POST /api/orders/maintenance` → same `createOrder()` function

**Known data at trigger time:** The full order document (including `provider` and `patient`).

**Lookup chain:** None — we already have the order, go straight to `is340BEligible(order)`.

**Volume:**

| Metric | Value |
|--------|-------|
| Orders affected | 1 |
| `is340BEligible()` calls | 1 |
| DB reads (inside calculation) | 1 `hospitalSystems` query + 1 `patients` query + 1 `patient_insurances` query = **3** |
| DB writes | 1 (`$set` on the order) |
| **Total DB operations** | **4** |

**Strategy:** Inline — calculate during the API request. Negligible cost.

---

### Trigger 2 — Order updated (key fields changed)

**When:** An existing order's `patient` or `provider` field changes via the web app.

**Source:** UI action (user edits an order in the Orders Tracker)

**Entry points:**
- `PUT /api/orders/new` → `updateOrder()` in `services/mongodb/ordersTracker.ts` (line 854)
- `PUT /api/orders/maintenance` → same `updateOrder()` function

**Known data at trigger time:** The updated order document and the changed fields.

**Lookup chain:** None — we already have the order. Only recalculate if `patient` or `provider` changed.

**Volume:** Same as Trigger 1.

| Metric | Value |
|--------|-------|
| Orders affected | 1 (or 2 if protocol partner is linked) |
| `is340BEligible()` calls | 1–2 |
| DB reads (inside calculation) | 3–6 |
| DB writes | 1–2 |
| **Total DB operations** | **4–8** |

**Strategy:** Inline — calculate during the API request. Negligible cost.

---

### Trigger 3 — Hospital system NPI list updated

**When:** An admin modifies a hospital system's providers list via the bulk update endpoint.

**Source:** UI action (admin updates NPI list in Hospital System Agreements)

**Entry point:** `PUT /api/hospitals/[id]/providers/bulk` → `bulkUpdateProviders()` in `services/mongodb/hospitalSystem.ts`

**Known data at trigger time:** The hospital system ID. The providers list is a **full replacement** (not a diff), with up to **10,000 NPIs** (enforced API limit).

#### NPI diff strategy

Since the API receives a full replacement list, we must compute the diff between the old and new NPI lists to determine which orders are affected. Only orders tied to **changed** NPIs need recalculation — orders tied to unchanged NPIs are not affected.

```
oldNPIs = current hospitalSystem.providers[].npi   (before update)
newNPIs = incoming request payload NPIs             (after update)

added   = newNPIs − oldNPIs   // NPIs present in new list but not in old
removed = oldNPIs − newNPIs   // NPIs present in old list but not in new

affectedNPIs = added ∪ removed
```

**Why each change type matters:**

| Change type | Affected orders | Effect |
|-------------|----------------|--------|
| **Added NPI** | Orders where `provider` matches an added NPI | May **gain** 340B eligibility (this hospital system now covers their provider) |
| **Removed NPI** | Orders where `provider` matches a removed NPI | May **lose** 340B eligibility (this hospital system no longer covers their provider — but another system might still match) |
| **Unchanged NPI** | Orders where `provider` matches an unchanged NPI | **No effect** — the hospital system still covers their provider, no recalculation needed |

In both added and removed cases, the resolution is the same: recalculate `is340BEligible(order)`. The full calculation checks ALL active hospital systems, so it naturally handles:
- An added NPI appearing in the results for the first time
- A removed NPI disappearing from one system but still existing in another

**Example:**

```
Hospital System "ABC" old providers: [001, 002, 003]
Hospital System "ABC" new providers: [002, 003, 004]

added   = [004]       ← orders with provider 004 might now be eligible via "ABC"
removed = [001]       ← orders with provider 001 might no longer be eligible via "ABC"
                        (but could still be eligible via another hospital system)
unchanged = [002, 003] ← no recalculation needed

affectedNPIs = [001, 004]  (2 NPIs instead of all 4)
```

**Lookup chain:**

```
hospitalSystemId
│
├──► fetch current hospitalSystem → oldNPIs = providers[].npi
│
├──► compute diff:
│    added   = newNPIs − oldNPIs
│    removed = oldNPIs − newNPIs
│    affectedNPIs = added ∪ removed
│
│    affectedNPIs is empty? ──► no-op (nothing changed)
│
├──► perform the bulk update (write new providers list)
│
├──► query neworders WHERE provider IN [...affectedNPIs] → N orders
└──► for each of N orders → is340BEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Old NPIs | M_old (up to 10,000) |
| New NPIs | M_new (up to 10,000) |
| Added NPIs | A = \|newNPIs − oldNPIs\| |
| Removed NPIs | R = \|oldNPIs − newNPIs\| |
| Affected NPIs | A + R (typically much smaller than M_old or M_new) |
| Orders to find | `neworders` WHERE `provider` IN [A + R NPIs] |
| Orders affected | N |
| `is340BEligible()` calls | N |
| DB reads (fetch old state) | 1 (read hospital system before update) |
| DB reads (find orders) | 2 queries with `$in` over (A + R) NPIs |
| DB reads (inside calculation) | (N) × 3 |
| DB writes | N |
| **Total DB operations** | **3 + (N) × 4** |

**Improvement over naive approach:**
- Naive: `$in` query over all M NPIs (up to 10,000)
- Diff: `$in` query over only (A + R) changed NPIs
- If 10,000 NPIs but only 5 changed → `$in` shrinks from 10,000 to 5 elements
- If nothing changed (same list re-uploaded) → **no recalculation at all**

**Concerns:**
- **No index on `neworders.provider`** — the `$in` query still does a collection scan, but over a much smaller set of values
- Each matched order still triggers 3 additional reads for the calculation
- Worst case (full list replacement — all old NPIs removed, all new NPIs added): A + R = M_old + M_new, similar to naive approach but this is rare in practice

**Strategy:** Background job (Pulse). Still the most expensive trigger, but the diff optimization significantly reduces the typical case.

---

### Trigger 4 — Patient insurance updated via webhook

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
│    └──► for each of N orders → is340BEligible(order)
```

**Volume:**

| Metric | Value |
|--------|-------|
| Records in webhook | R (no limit) |
| Distinct patients | D (≤ R, after dedup by `orders.patient_id`) |
| Orders per patient | Typically small (1–5), but unbounded |
| `is340BEligible()` calls | D × (avg orders per patient) |
| DB reads (find patients) | D × 1 query on `patients` |
| DB reads (find orders) | D × 2 queries on `neworders` |
| DB reads (inside calculation) | total_orders × 3 |
| DB writes | total_orders |
| **Total DB operations** | **D × 3 + total_orders × 4** |

**Concerns:**
- The webhook payload can contain many records (hundreds) across many patients
- Each distinct patient requires 3 lookup queries before calculation even starts
- The chain is deeper: payload → `we_infuse_patient_id` → `patients._id` → orders (2-hop lookup to find affected orders)

**Strategy:** Background job (Pulse). One job per distinct patient to parallelize.

---

### Trigger 5 — Hospital system name renamed

**When:** An admin renames a hospital system via the admin panel.

**Source:** UI action (admin renames hospital system in Hospital System Agreements)

**Entry point:** `PUT /api/hospitals/[id]` → `updateHospitalSystem()` in `services/mongodb/hospitalSystem.ts` (line 201)

**Known data at trigger time:** The hospital system ID and the new name. The current `updateHospitalSystem()` uses `findOneAndUpdate({ new: true })`, which returns the updated document — the old name is not available after the update. To detect a name change, the old document must be fetched before updating.

**Why this matters:** The stored 340B eligibility result is a string containing the hospital system name (e.g., `"Hospital System ABC, Hospital System XYZ"`). If the name changes, the stored string on every affected order becomes stale. The eligibility outcome itself doesn't change — only the display value.

**Lookup chain:**

```
hospitalSystemId + newName
│
├──► fetch current hospitalSystem → oldName, providers[].npi (M NPIs)
│
│    oldName === newName? ──► no-op (nothing changed)
│
├──► perform the name update
│
├──► query neworders WHERE provider IN [...M NPIs] → N orders
└──► for each of N orders → is340BEligible(order)
     (recalculation will use the new name from the updated hospitalSystem document)
```

**Volume:**

| Metric | Value |
|--------|-------|
| NPIs in hospital system | M (up to 10,000) |
| Orders to find | `neworders` WHERE `provider` IN [M NPIs] |
| Orders affected | N |
| `is340BEligible()` calls | N |
| DB reads (fetch old state) | 1 (read hospital system before update) |
| DB reads (find orders) | 2 queries with `$in` over M NPIs |
| DB reads (inside calculation) | (N) × 3 |
| DB writes | N |
| **Total DB operations** | **3 + (N) × 4** |

**Note:** Unlike Trigger 3, no diff optimization is possible here — all NPIs in the system are affected because the name applies to the entire hospital system. However, name renames are rare, so the higher cost per trigger is acceptable.

**Strategy:** Background job (Pulse). Low frequency but potentially high fan-out.

---

### Volume comparison

```
Trigger    Source     Known Data          Fan-out        Lookups to find orders    Cost per trigger
────────   ────────  ──────────────────  ─────────────  ────────────────────────  ─────────────────────
1. Order   UI        order               1:1            0 (already have it)       4 ops (trivial)
   create

2. Order   UI        order + changed     1:1            0 (already have it)       4 ops (trivial)
   update              fields

3. Hosp.   UI        hospitalSystemId    1 → (A+R)      1 read (old state) +     3 + N × 4 ops
   NPI                A=added, R=removed  changed NPIs   diff computation +       ⚠️ MODERATE
   update             (typically ≪ 10K)   → N        2 queries (orders by     (diff reduces scope)
                                           orders         changed NPIs only)

4. Ins.    Webhook   R records with      R → D           D reads (patients) +     D×3 + total×4 ops
   webhook            we_infuse_patient   patients        D×2 reads (orders by     ⚠️ MODERATE-HIGH
                      _id                 → orders        patient)

5. Hosp.   UI        hospitalSystemId    1 → M NPIs     1 read (old state) +     3 + N × 4 ops
   name               + newName           → N        2 queries (orders by     ⚠️ MODERATE
   rename             (no diff possible)  orders         all system NPIs)         (rare trigger)
```

### Indexes creation

The `neworders` collection is a raw MongoDB collection (no Mongoose model) with no existing indexes on the fields used by recalculation triggers. The following indexes must be created:

| Index | Used by | Query pattern |
|-------|---------|---------------|
| `neworders: { provider: 1 }` | Trigger 3 (NPI list update), Trigger 5 (hospital rename) | `WHERE provider IN [affectedNPIs]` |
| `neworders: { patient: 1 }` | Trigger 4 (insurance webhook) | `WHERE patient = patient._id` |

---

### Optimization notes (for later)

The volume analysis reveals two bottlenecks that will need attention during implementation:

1. **~~Missing index on `neworders.provider`~~** — Addressed in the Indexes creation section above.
2. **Unbounded webhook payload** — Trigger 4 can contain hundreds of records, each requiring a 2-hop lookup chain to find affected orders. The sequential `findOneAndUpdate` pattern means a large payload already takes time before recalculation even starts.
3. **Redundant calculations** — If both `patient` and `provider` are used as 340B inputs, an order could be recalculated multiple times if Triggers 3 and 4 fire close together for the same order.
4. **~~No diff on hospital update~~** — Addressed by the NPI diff strategy in Trigger 3. The diff (`added ∪ removed`) is computed before the update, and only orders tied to changed NPIs are recalculated. Same-list re-uploads result in zero recalculations.
