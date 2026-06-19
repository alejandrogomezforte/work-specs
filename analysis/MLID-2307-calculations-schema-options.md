# MLID-2307 — Financial Calculations: Schema Options

## Objective

We need to persist patient-responsibility calculations from the new **Financial Review** tab on the Order Details v2 page.

- Scope is **order-level**, not patient-level (PO confirmed).
- Applies only to `neworders` and `maintenanceorders` (not `leadorders`).
- Each saved calculation stores the 9 user inputs, the 3 computed responsibility numbers, summary lines, author, and dates.

There are 3 viable shapes. Picking one before I build Stage 2.

---

## Reference: what we store today

For context, this is what one item in `patients.financialCalculationsHistory[]` looks like today (legacy patient-scoped calculator, PHI scrubbed). Note that the legacy shape only stores the 3 summary numbers + an AI-generated markdown blob — it does **not** preserve the raw inputs the user typed. Stage 2 should persist the full inputs + result regardless of which option below we pick, so the new UI's expandable inputs grid keeps working.

```jsonc
{
  "_id": ObjectId("678857bec3de489316307000"),
  "summary": {
    "calculationOwner": "jdoe@mylocalinfusion.com",
    "calculationDate": "2025-01-16T00:50:06.933Z",
    "totalResponsibility": {
      "title": "Total Responsibility",
      "value": "2100.00"
    },
    "patientDrugResponsibility": {
      "title": "Patient Drug Responsibility",
      "value": "2000.00"
    },
    "patientAdminResponsibility": {
      "title": "Patient Admin Responsibility",
      "value": "100.00"
    }
  },
  "resultBreakDownText": "Let's calculate the total patient responsibility step-by-step based on the provided information.\n\n### Insurance Information\n1. **Deductible Met to Date**: $1000\n2. **Deductible Max**: $5000\n3. **Patient Co-Insurance**: 10%\n4. **OOP Met to Date**: $1000\n5. **OOP Maximum**: $5000\n\n### Drug Cost\n6. **Drug Cost**: $3000\n\n### Admin Cost\n7. **Admin Cost**: $200\n\n### Copay Assistance Coverage\n8. **Copay Assistance for Drug**: $1000\n9. **Copay Assistance for Admin**: $100\n\n... (long AI-generated markdown breakdown continues; rendered as-is in the legacy UI) ...\n\n### Summary\n- **Total Responsibility: $2100.00**\n- **Patient Drug Responsibility: $2000.00**\n- **Patient Admin Responsibility: $100.00**\n- **Reason: The patient has not yet met the OOP maximum...**"
}
```

---

## Option A — Embedded array on each order

A new `ordersFinancialCalculations` array field on every document in `neworders` and `maintenanceorders`.

```jsonc
// neworders document (or maintenanceorders document)
{
  "_id": ObjectId("662f...a1"),
  // ... existing order fields ...
  "ordersFinancialCalculations": [
    {
      "_id": ObjectId("663a...01"),
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
          "Apply drug assistance: $5.00.",
          "Patient pays for drug: $15.00."
        ],
        "admin": [
          "Starting with cost $10.00.",
          "Apply to deductible: $10.00 (remaining deductible now $70.00).",
          "Responsibility before assistance: $10.00.",
          "Apply admin assistance: $10.00.",
          "Patient pays for admin: $0.00."
        ],
        "summary": [
          "Patient Drug Responsibility: $15.00",
          "Patient Admin Responsibility: $0.00",
          "Total Responsibility: $15.00"
        ],
        "reason": "Since $100 of the $200 deductible was already met, the $20 drug charge applies to the deductible. Drug assistance of $5 reduces the patient's drug payment to $15. The $10 admin charge is fully covered by $10 of admin assistance, so admin responsibility is $0."
      },
      "author": "jdoe@mylocalinfusion.com",
      "calculationDate": ISODate("2026-05-21T17:00:08.267Z"),
      "createdAt": ISODate("2026-05-21T17:00:08.267Z"),
      "updatedAt": ISODate("2026-05-21T17:00:08.267Z")
    }
  ]
}
```

- **Read**: rides along with the existing order fetch — no extra query.
- **Write**: `$push` onto the parent order document.
- Same shape duplicated across `neworders` and `maintenanceorders`.

---

## Option B — One dedicated collection with `orderCategory` discriminator

A new top-level collection `ordersFinancialCalculations`. Each document references back to the source order via `orderId` + `orderCategory`.

> **Naming note**: fields are `orderId` / `orderCategory` rather than generic `entityId` / `entityType` because orders are first-class citizens in LISA — the domain primary entity for this surface is the order, so the foreign-key fields should be spelled in domain terms. `entityType` invites the question "entity of what?"; `orderCategory` is self-documenting and matches how the rest of the codebase talks about orders.

```jsonc
// ordersFinancialCalculations document
{
  "_id": ObjectId("663a...01"),
  "orderId": ObjectId("662f...a1"),
  "orderCategory": "neworders",   // "neworders" | "maintenanceorders"
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
      "Apply drug assistance: $5.00.",
      "Patient pays for drug: $15.00."
    ],
    "admin": [
      "Starting with cost $10.00.",
      "Apply to deductible: $10.00 (remaining deductible now $70.00).",
      "Responsibility before assistance: $10.00.",
      "Apply admin assistance: $10.00.",
      "Patient pays for admin: $0.00."
    ],
    "summary": [
      "Patient Drug Responsibility: $15.00",
      "Patient Admin Responsibility: $0.00",
      "Total Responsibility: $15.00"
    ],
    "reason": "Since $100 of the $200 deductible was already met, the $20 drug charge applies to the deductible. Drug assistance of $5 reduces the patient's drug payment to $15. The $10 admin charge is fully covered by $10 of admin assistance, so admin responsibility is $0."
  },
  "author": "jdoe@mylocalinfusion.com",
  "calculationDate": ISODate("2026-05-21T17:00:08.267Z"),
  "createdAt": ISODate("2026-05-21T17:00:08.267Z"),
  "updatedAt": ISODate("2026-05-21T17:00:08.267Z")
}
```

- **Read**: indexed `find({ orderId, orderCategory })`.
- **Write**: insert one document — order doc untouched.
- Compound index on `{ orderId, orderCategory, calculationDate }`.

---

## Option C — Two dedicated collections, one per order type

Two new collections: `newOrdersFinancialCalculations` and `maintenanceOrdersFinancialCalculations`. Same document shape, no `orderCategory` field — the collection name is the discriminator.

```jsonc
// newOrdersFinancialCalculations document
{
  "_id": ObjectId("663a...01"),
  "orderId": ObjectId("662f...a1"),
  "inputs":          { /* ...same 9 fields as Option B... */ },
  "result":          { /* ...drugResp, adminResp, totalResp... */ },
  "breakdown":  { /* ...drug[], admin[], summary[], reason... */ },
  "author": "jdoe@mylocalinfusion.com",
  "calculationDate": ISODate("2026-05-21T17:00:08.267Z"),
  "createdAt": ISODate("2026-05-21T17:00:08.267Z"),
  "updatedAt": ISODate("2026-05-21T17:00:08.267Z")
}

// maintenanceOrdersFinancialCalculations document — same shape, different collection
{
  "_id": ObjectId("663a...02"),
  "orderId": ObjectId("8821...b4"),
  "inputs":          { /* ... */ },
  "result":          { /* ... */ },
  "breakdown":  { /* ... */ },
  "author":  "asmith@mylocalinfusion.com",
  "calculationDate": ISODate("2026-05-21T18:15:00Z"),
  "createdAt": ISODate("2026-05-21T18:15:00Z"),
  "updatedAt": ISODate("2026-05-21T18:15:00Z")
}
```

- Same read/write/index pattern as B, but per collection.

---

## Side-by-side

| Concern | A — embedded array | B — one collection w/ discriminator | C — two collections |
|---|---|---|---|
| Mongoose model | None (order collections are raw-driver today; would stay raw-driver) | One new model | Two new models |
| Aligns with team's "new code = Mongoose" direction | ❌ (raw driver only realistic option) | ✅ | ✅ |
| Read | Free (rides on existing order fetch) | One indexed `find` | One indexed `find` |
| Write | `$push` on order doc | Insert one doc | Insert one doc |
| Document growth | Order doc grows unboundedly; array rewritten on each append | Fixed-size new doc per calc | Fixed-size new doc per calc |
| `orderstrackercache` impact | Every save re-caches the entire order doc | Untouched | Untouched |
| Cross-order queries (by author, date, etc.) | Hard (`$unwind` required) | Easy | Easy (but per collection) |
| Add `leadorders` later | Add field to a 3rd collection | One enum value on `orderCategory` | A 3rd collection + model |
| Risk of mis-scoping | None (data is nested in the order) | Real — `orderCategory` must always be paired with `orderId` | None (collection is the namespace) |
| Migration cost from this shape to another later | High (have to unwind nested arrays + rewrite callers) | Low | Low |
| Stage 2 implementation effort (rough) | Small (raw-driver `$push` + read tweak + migration) | Small–medium (model + service + routes + migration) | Medium (≈ 2× of B) |

---

## My recommendation

**Option B.** Dedicated collection, one indexed lookup, no order-doc bloat, no `orderstrackercache` churn, easy to extend if PO ever wants `leadorders`. The only mild downside is the `orderCategory` discriminator — easy to mitigate at the service layer.

---

## Final resolution

**Decision: Option B.** Unanimous vote, 3 of 3 (Engineer + Tech Lead + Principal Engineer).

Naming amendments adopted after the vote:

- **Foreign-key fields** are spelled `orderId` and `orderCategory` (not the originally-proposed `entityId` / `entityType`) because orders are first-class citizens in LISA — see the naming note under [Option B](#option-b--one-dedicated-collection-with-ordercategory-discriminator).
- **Collection name** is `ordersFinancialCalculations` (not the originally-shown `financialCalculations`). Same principle — the prefix makes the domain explicit. All code identifiers in implementation (model class, service file, hook, API client) follow the matching `OrdersFinancialCalculation` / `ordersFinancialCalculation` convention.
