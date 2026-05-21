# Postmortem: orderstrackercache Holds Stale Eligibility After an Upstream Lookup Field Changes

**Status**: Resolved per-order by touching the affected order to fire the change stream. No code change made. Same operational pattern as the prior post-deploy variant ([orderstrackercache-stale-after-eligibility-pipeline-deploy.md](./orderstrackercache-stale-after-eligibility-pipeline-deploy.md)), different trigger.
**Date**: 2026-05-18
**Related ticket**: None — surfaced by a Product Owner question about a specific order in production.
**Related files**:

- `apps/web/services/jobs/definitions/orders-tracker-cache/ordersTrackerCache.ts` (the change-stream-driven cache job)
- `apps/web/services/mongodb/ordersTracker.ts` (`getEligibilityPipelineStages`, the pipeline whose output is cached)
- `apps/web/worker.ts` (worker process that runs the cache job)
- `docs/orders/pharmacy-eligibility.md` (canonical pharmacy-eligibility rule reference)

---

## Symptom

Product Owner asked why order `04dT` was showing `pharmacyEligible: "No"` in the Orders Tracker UI when, by manual UI inspection, every gate of the pharmacy-eligibility rule appeared to pass:

- Patient `69f3785bd5d0cfeda77db9e8` has an active Commercial primary insurance.
- Insurance plan `697355c258c03cfecc353f4d` (Meritain Health) is marked pharmacy-eligible.
- Drug `698507630c83fd8007e79e6c` is not flagged ineligible (`pharmacyEligible !== false`).
- Site `67b7083a8532e12fdce9d61c` is pharmacy-eligible.

Expected: `"Yes"`. Observed in UI: `"No"`.

---

## Diagnostic walkthrough

### Step 1 — Run the live aggregation pipeline against the order

The Orders Tracker UI does **not** invoke the eligibility pipeline live — it reads from `orderstrackercache`. To verify what the pipeline produces _right now_ against current data, we bypass the cache by running the equivalent aggregation directly against `neworders` in the production Atlas UI.

```javascript
[
  // 1. Match the order — use displayId regex OR replace with _id match
  { $match: { displayId: { $regex: "04dT", $options: "i" } } },

  // 2. Hydrate the three string-stored ObjectId refs (site/drug/patient)
  {
    $addFields: {
      _siteId:    { $convert: { input: "$site",    to: "objectId", onError: null } },
      _drugId:    { $convert: { input: "$drug",    to: "objectId", onError: null } },
      _patientId: { $convert: { input: "$patient", to: "objectId", onError: null } }
    }
  },
  { $lookup: { from: "patientLocations", localField: "_siteId",    foreignField: "_id", as: "_site"    } },
  { $lookup: { from: "lidrugs",          localField: "_drugId",    foreignField: "_id", as: "_drug"    } },
  {
    $lookup: {
      from: "patients",
      localField: "_patientId",
      foreignField: "_id",
      pipeline: [{ $project: { _id: 1, we_infuse_id_IV: 1 } }],
      as: "_patient"
    }
  },
  { $unwind: { path: "$_site",    preserveNullAndEmptyArrays: true } },
  { $unwind: { path: "$_drug",    preserveNullAndEmptyArrays: true } },
  { $unwind: { path: "$_patient", preserveNullAndEmptyArrays: true } },

  // 3. Primary active insurance: priority='1' AND status IN ['Active','Undetermined']
  {
    $lookup: {
      from: "patient_insurances",
      let: { wiId: "$_patient.we_infuse_id_IV" },
      pipeline: [
        { $match: { $expr: { $and: [
          { $eq: ["$we_infuse_patient_id", "$$wiId"] },
          { $eq: ["$priority", "1"] },
          { $in: ["$status", ["Active", "Undetermined"]] }
        ] } } },
        { $limit: 1 },
        { $project: { _id: 1, priority: 1, status: 1, payer_type_name: 1, insurance_plan_name: 1 } }
      ],
      as: "_primaryInsurance"
    }
  },

  // 4. Insurance plan by plan name (only consulted for Commercial/Medicaid)
  {
    $lookup: {
      from: "insurance_plans",
      let: { planName: { $arrayElemAt: ["$_primaryInsurance.insurance_plan_name", 0] } },
      pipeline: [
        { $match: { $expr: { $eq: ["$planName", "$$planName"] } } },
        { $limit: 1 },
        { $project: { _id: 1, planName: 1, isPharmacyEligible: 1 } }
      ],
      as: "_insurancePlan"
    }
  },

  // 5. Per-gate diagnostic flags (mirrors prod regex exactly)
  {
    $addFields: {
      checks: {
        gate1_hasPrimaryInsurance: {
          $gt: [{ $size: { $ifNull: ["$_primaryInsurance", []] } }, 0]
        },
        gate2_drugIsTreatment: {
          $eq: [{ $ifNull: ["$_drug.is_treatment", false] }, true]
        },
        gate3_drugNotExplicitlyIneligible: {
          $ne: [{ $ifNull: ["$_drug.pharmacyEligible", null] }, false]
        },
        gate4_siteIsPharmacyEligible: {
          $eq: [{ $ifNull: ["$_site.isPharmacyEligible", false] }, true]
        },
        payerTypeName: { $arrayElemAt: ["$_primaryInsurance.payer_type_name", 0] },
        gate5_pathA_medicareMatches: {
          $regexMatch: {
            input: { $ifNull: [{ $arrayElemAt: ["$_primaryInsurance.payer_type_name", 0] }, ""] },
            regex: "^\\s*medicare(\\s+(advantage|supplement))?\\s*$",
            options: "i"
          }
        },
        gate5_pathB_commercialOrMedicaidMatches: {
          $regexMatch: {
            input: { $ifNull: [{ $arrayElemAt: ["$_primaryInsurance.payer_type_name", 0] }, ""] },
            regex: "commercial|medicaid",
            options: "i"
          }
        },
        gate5_pathB_planIsPharmacyEligible: {
          $eq: [{ $arrayElemAt: ["$_insurancePlan.isPharmacyEligible", 0] }, true]
        }
      }
    }
  },

  // 6. Final verdict — identical $switch logic to prod
  {
    $addFields: {
      pharmacyEligible: {
        $switch: {
          branches: [
            {
              case: { $eq: [{ $size: { $ifNull: ["$_primaryInsurance", []] } }, 0] },
              then: "Waiting on Insurance Input"
            },
            {
              case: { $eq: [{ $ifNull: ["$_drug.is_treatment", false] }, true] },
              then: "No"
            },
            {
              case: {
                $and: [
                  { $ne: [{ $ifNull: ["$_drug.pharmacyEligible", null] }, false] },
                  { $eq: [{ $ifNull: ["$_site.isPharmacyEligible", false] }, true] },
                  {
                    $or: [
                      {
                        $regexMatch: {
                          input: { $ifNull: [{ $arrayElemAt: ["$_primaryInsurance.payer_type_name", 0] }, ""] },
                          regex: "^\\s*medicare(\\s+(advantage|supplement))?\\s*$",
                          options: "i"
                        }
                      },
                      {
                        $and: [
                          {
                            $regexMatch: {
                              input: { $ifNull: [{ $arrayElemAt: ["$_primaryInsurance.payer_type_name", 0] }, ""] },
                              regex: "commercial|medicaid",
                              options: "i"
                            }
                          },
                          { $eq: [{ $arrayElemAt: ["$_insurancePlan.isPharmacyEligible", 0] }, true] }
                        ]
                      }
                    ]
                  }
                ]
              },
              then: "Yes"
            }
          ],
          default: "No"
        }
      }
    }
  },

  // 7. Compact diagnostic projection
  {
    $project: {
      _id: 1,
      displayId: 1,
      orderStatus: 1,
      pharmacyEligible: 1,
      checks: 1,
      site:    { _id: "$_site._id",    name: "$_site.name",    isPharmacyEligible: "$_site.isPharmacyEligible" },
      drug:    { _id: "$_drug._id",    is_treatment: "$_drug.is_treatment", pharmacyEligible: "$_drug.pharmacyEligible" },
      patient: { _id: "$_patient._id", we_infuse_id_IV: "$_patient.we_infuse_id_IV" },
      primaryInsurance: { $arrayElemAt: ["$_primaryInsurance", 0] },
      insurancePlan:    { $arrayElemAt: ["$_insurancePlan",    0] }
    }
  }
]
```

### Step 2 — Inspect the diagnostic result

The pipeline returned this document for the order in question:

```json
{
  "_id": { "$oid": "69f4b91f50ec5fb019e960fb" },
  "displayId": "04dT",
  "site": {
    "_id": { "$oid": "67b7083a8532e12fdce9d61c" },
    "isPharmacyEligible": true
  },
  "patient": {
    "_id": { "$oid": "69f3785bd5d0cfeda77db9e8" },
    "we_infuse_id_IV": "[redacted]"
  },
  "drug": {
    "_id": { "$oid": "698507630c83fd8007e79e6c" },
    "is_treatment": false
  },
  "checks": {
    "gate1_hasPrimaryInsurance": true,
    "gate2_drugIsTreatment": false,
    "gate3_drugNotExplicitlyIneligible": true,
    "gate4_siteIsPharmacyEligible": true,
    "payerTypeName": "Commercial",
    "gate5_pathA_medicareMatches": false,
    "gate5_pathB_commercialOrMedicaidMatches": true,
    "gate5_pathB_planIsPharmacyEligible": true
  },
  "pharmacyEligible": "Yes",
  "primaryInsurance": {
    "_id": { "$oid": "69f4ce506eb92d6bba0d94bb" },
    "insurance_plan_name": "Meritain Health",
    "payer_type_name": "Commercial",
    "priority": "1",
    "status": "Active"
  },
  "insurancePlan": {
    "_id": { "$oid": "697355c258c03cfecc353f4d" },
    "planName": "Meritain Health",
    "isPharmacyEligible": true
  }
}
```

Every gate passes — the live verdict is `"Yes"`. UI shows `"No"`. The discrepancy is between the live pipeline and the cached value in `orderstrackercache`.

### Step 3 — Confirm the cache is stale

```js
db.orderstrackercache.findOne({ _originalId: "69f4b91f50ec5fb019e960fb" })
```

> Note: `_originalId` is stored as a **string** in `orderstrackercache`, not an `ObjectId` — do not wrap it in `ObjectId(...)` when querying.

The cached row's `pharmacyEligible` is `"No"` — frozen from a prior computation when one of the upstream lookup fields (in this case almost certainly `insurance_plans.isPharmacyEligible` for `Meritain Health`) was `false`. The flag has since been flipped to `true`, but the cache row never recomputed.

---

## Conclusion

**This is not a pipeline bug.** The eligibility rule and the underlying data are both correct. The bug is in the cache-invalidation strategy.

`orderstrackercache` is populated by a change-stream listener on the `neworders` collection (`ordersTrackerCacheJob`). The listener fires only when an order document itself is modified. It does **not** fire when any of the upstream lookup sources change:

- `lidrugs.is_treatment` or `lidrugs.pharmacyEligible`
- `patientLocations.isPharmacyEligible`
- `patient_insurances.priority` / `status` / `payer_type_name` / `insurance_plan_name`
- `insurance_plans.isPharmacyEligible`

So whenever someone toggles a flag on any of those collections, every cached order that references it stays frozen at the pre-toggle value until the order itself is touched. From the UI's perspective, the column appears stuck.

This is the same operational class as the post-code-deploy variant ([orderstrackercache-stale-after-eligibility-pipeline-deploy.md](./orderstrackercache-stale-after-eligibility-pipeline-deploy.md)) — different trigger (data change vs. code change), identical symptom and remediation.

---

## Suggested fix

### Immediate (single order)

Edit any field on the affected order to fire the `neworders` change stream — for example, update `lastUpdated` to now in Atlas, or save any field through the UI. The cache job will recompute that one row within a few seconds. Refresh the Orders Tracker to confirm the column flips to `"Yes"`.

### Workaround (many orders affected by a recent flag toggle)

Trigger `ordersTrackerCacheJob` from `/admin/jobs` to force a full rebuild. This recomputes every cached row from the live pipeline. Slow (touches every order), but appropriate when a flag toggle plausibly affects a large slice of the order set (e.g., a payer-type fix, a plan-eligibility toggle that's on many patients).

### Long-term (the real fix, design conversation needed)

Extend cache invalidation to the four upstream collections. Two viable shapes:

1. **Additional change-stream listeners** on `lidrugs`, `patientLocations`, `patient_insurances`, `insurance_plans`. On change, look up the affected orders (reverse-lookup chains: e.g., `insurance_plans.planName` → `patient_insurances.insurance_plan_name` → `patients.we_infuse_id_IV` → `neworders.patient`) and re-cache them. Highest correctness, highest implementation cost — some of those reverse chains are 3-hop.
2. **TTL on cache rows** — accept eventual staleness, set a short TTL (e.g., 15 minutes) so cache rows automatically refresh in the background. Lower correctness floor (briefly stale during the TTL window), much cheaper to implement and reason about.

Worth a design conversation with the tech lead before picking one. The trade-off is correctness latency vs. implementation surface area. Until then, this postmortem and the operational runbook (touch the order or rebuild) is the documented remediation.
