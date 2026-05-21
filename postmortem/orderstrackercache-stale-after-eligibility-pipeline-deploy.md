# Postmortem: orderstrackercache Holds Stale Eligibility After a Pipeline Code Deploy

**Status**: Resolved per-order by manually touching each affected order. No code change made; this is a documented operational pattern, not a bug to fix.
**Date**: 2026-05-05
**Related ticket**: [MLID-2192](https://localinfusion.atlassian.net/browse/MLID-2192) — surfaced this issue during end-to-end testing
**Related files**:

- `apps/web/services/jobs/definitions/orders-tracker-cache/ordersTrackerCache.ts` (the change-stream-driven cache job)
- `apps/web/services/mongodb/ordersTracker.ts` (`getEligibilityPipelineStages`, the pipeline whose output is cached)
- `apps/web/worker.ts` (worker process that runs the cache job)
- `docs/agomez/testing/MLID-2192-data-testing.md` (the testing notebook where the discrepancy first showed up)

---

## Symptom

After MLID-2192 was merged to `develop` (commit `c8da369d`, ~8 hours before testing), an end-to-end test run against the deployed environment showed that the New Orders UI **still** reported `pharmacyEligible: "Waiting on Insurance Input"` for an order whose primary insurance had `status: "Undetermined"`, `priority: "1"`. With the MLID-2192 fix in place, the expected value was `"Yes"`.

Test order:

| Field | Value |
|-------|-------|
| Order `_id` | `69fa76b753008d6c2de629c2` |
| Created | `2026-05-05T23:01:11.592Z` |
| Patient | Jane Barker (`we_infuse_id_IV: 1045568`) |
| Drug | Skyrizi (`is_treatment: false`, `pharmacyEligible: true`) |
| Site | Toms River, NJ (`isPharmacyEligible: true`) |
| Insurance | VA Commercial, `status: "Undetermined"`, `priority: "1"` |
| Insurance plan | Department of Veterans Affairs (`isPharmacyEligible: true` after manual seed) |
| **UI reported** | `"Waiting on Insurance Input"` |
| **Expected** | `"Yes"` |

---

## Diagnostic walkthrough

### Step 1 — Verify the data shape

The 5 patient_insurances candidates with `priority: "1"` + `status: "Undetermined"` were already documented in `docs/agomez/testing/MLID-2192-data-testing.md`. The chosen patient + insurance + drug + site combination satisfied every gate of the pharmacy `$switch` (`ordersTracker.ts:127–217`). The data was correct.

### Step 2 — Verify the deployed code

Confirmed `c8da369d` (the MLID-2192 merge) was on `develop`:

```
git log --oneline --grep="MLID-2192" -10
c8da369d Merge pull request #1460 from LocalInfusion/fix/MLID-2192-undetermined-insurance
e6ef199d Merge branch 'develop' into fix/MLID-2192-undetermined-insurance
a5427f02 [MLID-2192] - fix(orders-tracker): include "Undetermined" insurance status...
```

Code-side change was definitely on `develop`.

### Step 3 — Run the live aggregation against this exact order

Bypassed the cache by running the patient_insurances `$lookup` directly via the MCP `aggregate` tool, with both the new (`$in: [Active, Undetermined]`) and old (`$eq: Active`) filters side-by-side:

| Filter | Result |
|--------|--------|
| **New (MLID-2192)** `$in: ['$status', ['Active', 'Undetermined']]` | ✅ Returns 1 doc — Jane Barker's VA Commercial insurance |
| **Old** `$eq: ['$status', 'Active']` | ❌ Returns `[]` (empty) |

So if the running pipeline used the new filter, the order would resolve to `"Yes"`. The empty result on the old filter exactly matches the `"Waiting on Insurance Input"` outcome the UI was showing.

### Step 4 — Inspect the cache row

UI reads from `orderstrackercache` (a precomputed materialized view), not from the live pipeline. The cache row for this order existed but was stale:

```js
db.orderstrackercache.findOne({ _originalId: "69fa76b753008d6c2de629c2" })
```

```
{
  _originalId: "69fa76b753008d6c2de629c2",
  _lastCacheUpdated: 2026-05-05T23:01:23.754Z,    // ← 12s after order creation
  pharmacyEligible: "Waiting on Insurance Input",  // ← STALE
  hospital340BEligible: "No",
  patient: { we_infuse_id_IV: 1045568, ... },
  // patient.primaryInsurance ABSENT — lookup returned []
}
```

### Step 5 — Check the change-stream is alive

```js
db.resumetokens.findOne({ _id: "ordersTrackerCache" })
// updatedAt: 2026-05-05T23:01:23.854Z — same instant the order was cached
```

The cache job *was* running and *did* process this order. It just produced a stale value.

### Step 6 — Force a recompute

Touched the order via UI (reassigned to a different user). The change stream fired, the handler re-ran the eligibility pipeline, and the cache row was rewritten with `pharmacyEligible: "Yes"`. The UI immediately reflected the correct value.

---

## Root cause

The `orderstrackercache` collection is a **precomputed materialized view** populated by a change-stream-driven background job (`apps/web/services/jobs/definitions/orders-tracker-cache/ordersTrackerCache.ts`). UI reads come from the cache, not from the live aggregation.

Two relevant invariants of how this cache is invalidated:

1. **Per-row invalidation requires an order document change.** Each cache row is rewritten only when the source order in `neworders` / `maintenanceorders` / `leadorders` is updated, fires a change-stream event, and the handler runs the eligibility pipeline against that one document. There is no scheduled refresh.
2. **Worker-process code is held in memory.** The cache job runs in `worker.ts`, a long-lived Node process. When `develop` is deployed, the new build only takes effect on the next worker restart/redeploy. Any cache rows written *before* that restart still reflect whatever code the worker happened to be running at the time.

The combination of (1) and (2) creates a stale-cache window after any deploy that changes pipeline output:

- Existing orders that aren't subsequently edited keep their pre-deploy `pharmacyEligible` / `hospital340BEligible` values **forever**, even though the data and the code both say a different answer.
- New orders created in the small window between merge-to-`develop` and the worker actually picking up the new build also get cached with the old logic.
- The change stream and resume token are unaffected — the job *looks* perfectly healthy from a monitoring standpoint.

In this specific case, the test order `69fa76b753008d6c2de629c2` was created and cached at `23:01:23Z` with `"Waiting on Insurance Input"`. The cache row simply held that value until the order was edited, at which point the (by-then upgraded) worker recomputed it as `"Yes"`.

---

## Why the test result looked like a code bug

Two failure modes converge on the same observable symptom:

| Failure mode | What's wrong |
|--------------|--------------|
| Worker is running pre-MLID-2192 code | Pipeline runs, but with the old `$eq: 'Active'` filter |
| Worker is on new code, but the cache row was written before the worker upgrade | Pipeline never re-runs for that order until it's touched |

Both write the same `"Waiting on Insurance Input"` value to the cache and produce identical UI behavior. There is no monitoring signal distinguishing them. The only distinguishing test is to (a) bypass the cache via direct aggregation against `neworders`, or (b) touch the order and check whether the cache row gets rewritten with a different value.

---

## Resolution (this incident)

Touched the order via the UI (reassigned to a different user). The change stream fired, the cache row was rewritten with the correct `"Yes"` value within a second. End-to-end MLID-2192 verified.

For testing with multiple orders, the same trick scales:

```js
db.neworders.updateMany(
  { _id: { $in: [ /* test order ids */ ] } },
  { $set: { _cacheBust: new Date() } }
)
```

Each `updateOne` triggers one change-stream event → one cache row rewrite. The custom field is harmless and ignored by the eligibility pipeline.

---

## Recommended pattern for future deploys that affect the eligibility pipeline

Any change to `getEligibilityPipelineStages()` or any of its inputs (drug fields, site fields, insurance lookup, plan lookup) should be followed by a **full rebuild** of `orderstrackercache` after the worker is redeployed. The job exposes a full-rebuild path; trigger it from `/admin/jobs` once the new worker is confirmed up.

For ad-hoc testing of a deployed pipeline change against a single order:

1. Run the live aggregation against `neworders` directly (bypass cache) to confirm the *data* would produce the expected value with the new pipeline. This proves the pipeline change is correct independent of cache state.
2. Inspect `db.orderstrackercache.findOne({ _originalId: <orderId> })` — compare `_lastCacheUpdated` to the deploy time. Older = stale row.
3. Either edit the order (any field) to force a re-cache, or trigger a full rebuild.

---

## Lessons

- **A precomputed materialized view is a deployment surface.** Code changes that change pipeline output don't take effect for existing rows without an explicit invalidation step. Treat the cache as a separate artifact that needs its own deploy step.
- **Change-stream-driven caches are silently stale on deploys.** They only invalidate on source-document change, never on time or code-version. There's no built-in way to detect that a cache row predates a relevant code deploy.
- **Two failure modes (old worker, stale cache) produce identical UI output.** Don't conflate them. The aggregate-against-source diagnostic is the cheapest way to disambiguate.
- **Collection-name casing trap.** The cache collection is `orderstrackercache` (lowercase), not `ordersTrackerCache`. The Mongoose `COLLECTION.OrdersTrackerCache` constant resolves to the lowercase name.
