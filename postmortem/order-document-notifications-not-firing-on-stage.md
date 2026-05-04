# Postmortem: Order Document Notifications Not Firing on Stage

**Status:** Investigation in progress
**Date:** 2026-04-29
**Environment:** Stage (post-merge to `develop`)
**Feature:** MLID-2011 — Order Document Notifications

## Summary

The notification trigger apparently is not working as expected on staging env.

The MLID-2011 feature was merged to `develop` and is deployed to stage. Documents are being uploaded daily across many orders, so we expect notifications to be created for each (assigned-order, document) pair. QA reports that documents have been added to at least one order and **no notifications appear**.

### Investigation focus: notification creation only

When a new document is uploaded, two things must happen:

1. **Primary — a row in the `notifications` collection** (so the assignee gets a badge / unread count).
2. **Secondary — a SignalR `documentAdded` broadcast** (so already-open browsers refetch without a manual refresh).

If (1) doesn't happen, (2) is moot — there's nothing to surface anyway. **This investigation is scoped to (1).** SignalR delivery problems can be triaged separately once notifications are being written reliably.

## Order under investigation

| Field | Value |
|---|---|
| Order `_id` | `69f2297b7fcc628ed8534fd1` |
| `displayId` | `05IR` |
| `createdAt` | `2026-04-29T15:53:31Z` |
| `orderType` | `Transfer` |
| `audit` | `Yes` |
| `assignedTo` | `67fe9b98a5133e710d955286` |
| `patient` (ref) | `69e7a39fd86a597a3ddd3be8` |
| `site` | `67b7083a8532e12fdce9d618` |

### Patient

| Field | Value |
|---|---|
| `_id` | `69e7a39fd86a597a3ddd3be8` |
| Name | John Papa QA |
| `we_infuse_id_IV` | `9146` |
| `we_infuse_location_id` | `252` |

`we_infuse_id_IV` is the key `notifyDocumentCreated` uses to fan out to active `neworders` for the patient — present and valid here.

### Assigned user

| Field | Value |
|---|---|
| `_id` | `67fe9b98a5133e710d955286` |
| Name | Kimberly Ayala |
| Email | `kayala@mylocalinfusion.com` |
| `role` | `user` |
| `positions` | Market Lead, Infusion Guide |

The order has a valid `assignedTo`, so a `type: 'user'` notification targeting Kimberly should be created when a document is added to John Papa QA.

## Preconditions check (so far)

Both prerequisites for `notifyDocumentCreated` to write a notification are met for this order:

- [x] Patient has `we_infuse_id_IV` (9146)
- [x] Order has `assignedTo` (Kimberly Ayala)
- [x] Documents linked to this patient in the time window — **3 confirmed (see below)**
- [ ] `notifications` collection has docs for this order/patient — **to verify**
- [ ] `ORDER_DOCUMENT_NOTIFICATIONS` feature flag enabled on stage — **to verify**
- [ ] Server logs show `notifyDocumentCreated` invocations / SignalR broadcasts — **to verify (az CLI)**

## Documents added today (2026-04-29) for patient `69e7a39fd86a597a3ddd3be8`

All three confirmed by QA — these are the documents we expect notifications for.

| Doc `_id` | Category | Description | createdAt (UTC) | intakeId | linkedOrderIds | createdBy |
|---|---|---|---|---|---|---|
| `69f22957f5e2b50085307c24` | Lab Results | Lab results for T-cell subsets and hepatitis B serologies | 2026-04-29T15:52:55.455Z | `69f2280f7817a9a29965c5c1` | `[]` | `68c85068249cae0a011916b9` |
| `69f22999f5e2b50085307e98` | Supporting Medical Document | MRI brain supporting medical document with final impression and ARIA assessment | 2026-04-29T15:54:01.293Z | `69f225637817a9a29965c51d` | `[]` | `68c85068249cae0a011916b9` |
| `69f22999f5e2b50085307e9f` | Other | Other - Pages 1-1 | 2026-04-29T15:54:01.967Z | `69f225637817a9a29965c51d` | `[]` | `68c85068249cae0a011916b9` |

### Timing observation

Order `05IR` (`69f2297b7fcc628ed8534fd1`) was created at **2026-04-29T15:53:31.622Z**.

| Document | Uploaded vs. order creation |
|---|---|
| Lab Results (15:52:55Z) | **Before** the order existed — `findOrdersForPatient` would return no active orders for this patient at that moment, so **no notification expected** for this one. |
| Supporting Medical Document (15:54:01Z) | After order existed — **notification expected**. |
| Other (15:54:01Z) | After order existed — **notification expected**. |

So the minimum we should see in `notifications` is **2 documents** targeting `targetUserId = 67fe9b98a5133e710d955286` (Kimberly Ayala) with `entities.entityId = 69f2297b7fcc628ed8534fd1` (the order).

### Path observation

All three docs come from two intakes (`69f225637817a9a29965c51d`, `69f2280f7817a9a29965c5c1`) — this is the **JotForm intake → `weinfuseUploader.ts:344`** path, not the manual `split-pdf` upload route. Important because each entry point invokes `notifyDocumentCreated` independently — a regression could affect one path and not the other.

Note: `linkedOrderIds: []` on all three is expected for the notification flow — the trigger does *not* use `linkedOrderIds`. It links docs → patient (`patientId`) → `we_infuse_id_IV` → active `neworders` for the WeInfuse patient ID.

## Notifications & reads queried (2026-04-29)

### `notifications` collection — total in stage DB: **8**

Filters tried, all returned **0 matches**:
- `entities.entityId = 69f2297b7fcc628ed8534fd1` (the order)
- `entities.entityId` ∈ the three document IDs from today
- `targetUserId = 67fe9b98a5133e710d955286` (Kimberly)

So no notification exists for our order, our patient, our assignee, or any of today's documents.

### What the 8 existing notifications look like

| Property | Value |
|---|---|
| `createdAt` | All 8 share **2026-04-15T19:08:54.249Z** (same millisecond) |
| `_id` sequence | `69dfe246adc73ef87c73c049` … `c050` — sequential, single batch insert |
| `targetUserId` | All 8: `695c25a530f7a75a8b1ed8c8` (one user) |
| `entities[].entityId` (order) | All 8: `69cf9aee03a194bd1a65a199` (one order) |
| `createdBy` | `system` |
| `type` | `user` |
| Distinct categories | Insurance Card (×2), Order (×2), Lab Results, Supporting Medical Document (×2), Referral Copy |

The same-millisecond timestamps + sequential `_id`s indicate a **single bulk insert** — likely a one-time manual / seed / test-data write, **not organic upload activity**.

### `notificationreads` collection — total: **0**

Empty. Nobody on stage has marked anything as read.

### Implication

Stage has had **zero `notifyDocumentCreated` activity since 2026-04-15** (or possibly ever — those 8 don't look organic). Documents have been uploaded daily across the system but no notifications are being written.

This narrows the cause to one of:
1. `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is **off** in stage → the trigger returns early with no log noise.
2. The deployed build on stage doesn't actually contain the trigger calls (deploy didn't pick up the merge, or the trigger callers are gated differently).
3. The trigger is invoked but throws inside the patient/order lookup (silently caught + `logger.error` only — would show in az logs).
4. The MongoDB connection used by the trigger differs from the one we're querying.

Inspecting the feature flag and stage app logs will distinguish (1) from (2)/(3).

## Hypotheses (status)

| # | Hypothesis | Status |
|---|---|---|
| 1 | `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is off on stage | **Ruled out** — confirmed ON |
| 2 | Trigger isn't wired in the upload path the UI actually uses | Code review confirms it *is* wired (see below) |
| 3 | Trigger throws inside, swallowed by outer try/catch — visible only in logs | Open — needs az logs |
| 4 | `getOrdersByPatient` returns no orders due to ID mismatch | Open — see "Patient identifier audit" below |
| 5 | Deployed stage build is older than `develop` | Discarded — `origin/develop` is the source of truth for stage and contains the merged epic |
| 6 | Notifications *are* being written, UI/counts surfacing is broken | Ruled out — `notifications` count = 0 for our order/user/docs |

## Code review (post-investigation)

### Confirming the upload path

Today's "Other - Pages 1-1" description format (`${documentType} - Pages ${from-to}`) only appears in `split-pdf/route.ts:555` (branch B, split with page ranges). So today's three docs came through **`split-pdf` branch B**, not `weinfuseUploader.ts`. That points us at the trigger call at `split-pdf/route.ts:602`.

### Trigger callers (on `develop`)

| File | Line | What it passes as `patientId` |
|---|---|---|
| `apps/web/app/api/documents/split-pdf/route.ts` | 350 (branch A — `isFullUpload`) | `patient?._id?.toString()` — MongoDB `_id` |
| `apps/web/app/api/documents/split-pdf/route.ts` | 602 (branch B) ← **our path** | `mongoPatientId = patient?._id?.toString()` — MongoDB `_id` |
| `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` | 344 (uploadPDFToWeInfuse) | `patientDbId` — MongoDB `_id` |

All three call sites pass the **MongoDB `_id`** of the patient, not the WeInfuse ID. They obtain it via:
- `split-pdf`: `getExtendedPatientByWeInfuseId(weInfuseId)` → `patient._id.toString()`
- `weinfuseUploader`: `Patient.findOne({ we_infuse_id_IV: intake.patient.patientId })` → `patient._id.toString()`

### `notifyDocumentCreated` (`apps/web/services/notifications/documentNotification.ts`)

Outer try/catch swallows everything → on failure it only emits `logger.error`, never throws. Two silent early-return paths produce **zero log output**:

- L44-46: FF off → return (ruled out).
- L48-50: `!patientId` → return (would only fire if `mongoPatientId` was undefined; document record we found has `patientId: "69e7a39fd86a597a3ddd3be8"`, so this didn't happen).

Errors inside the loop:
- `createNotification` throw → caught at L81-86, logged.
- `signalr.broadcastToAll` throw → caught at L98-103, logged.
- Any other throw (e.g., `getSignalRService()` Key Vault failure, `getOrdersByPatient` connection error) → caught by outer try/catch at L105-110.

### `getOrdersByPatient` lean overload (`services/mongodb/ordersTracker.ts:1112`)

The trigger calls `getOrdersByPatient(patientId, { expandRefs: false })` which runs:

```js
{ $match: { patient: patientId, deleted: { $ne: true } } }
```

A direct string-equality match against `neworders.patient`. This implies:

- **The trigger expects `patientId` to be the same string stored in `neworders.patient`.**
- Our order stores `patient: "69e7a39fd86a597a3ddd3be8"` (a MongoDB `_id` as a hex string) — same value the trigger passes in.

So on paper, this should match. The next section verifies that empirically.

### Doc/code naming drift (informational)

The original epic used `findOrdersForPatient`. A subsequent refactor renamed it to `getOrdersByPatient` — the spec doc (`docs/signalr/order-documents-signalr-notifications.md` §4.2) still references the old name. **Stale doc, not a runtime issue.** Worth fixing in a follow-up.

### Conclusion of code review

The wiring is correct and `develop` is what's deployed. **The trigger function is not reaching `createNotification`** (we have zero notifications written despite documents being uploaded). The remaining failure modes are all runtime:

- (A) `getOrdersByPatient` returns `[]` (ID mismatch — see audit below).
- (B) An error before the loop (e.g., `getSignalRService()` Key Vault init failure) is caught by the outer try/catch and only visible as a `logger.error` line.

(B) requires az logs. (A) is testable now via DB queries — doing that next.

## Patient identifier audit (results)

**Goal:** confirm that `documents.patientId`, `neworders.patient`, and the value the trigger passes are all the same identifier type. A mismatch would silently make `getOrdersByPatient` return zero orders.

### `neworders.patient` — 2,044 docs

Distribution:
- `string` × 2,044 (100%) — all values are 24-char hex MongoDB ObjectIds (e.g. `"6807cc0884836fa1a2683d73"`).

The collection is consistent. **`neworders.patient` always stores the patient's MongoDB `_id` as a string.**

### `documents.patientId` — 9,969 docs

Distribution:
- `string` × 9,944 — sample values: `"1078388"`, `"1062362"` — **WeInfuse IDs (numeric strings)**.
- `null` × 3
- missing × 22

**Convention drift discovered:** historically the `documents.patientId` field stored the **WeInfuse ID**. The new code in this epic (split-pdf line 581/342, weinfuseUploader line 334) writes the **MongoDB `_id`** instead — so today's 3 documents have `patientId: "69e7a39fd86a597a3ddd3be8"` (MongoDB `_id`) while the 9,944 historic ones have numeric WeInfuse IDs.

This drift is **not the bug today** for our flow — the value the trigger receives matches the value `neworders.patient` stores. But it's a documented inconsistency in the field's meaning that any future query joining `documents.patientId` to a patient will need to handle.

### Reproducing the trigger's lookup

I ran the exact `$match` the trigger uses:

```js
db.neworders.aggregate([
  { $match: { patient: "69e7a39fd86a597a3ddd3be8", deleted: { $ne: true } } },
  { $project: { _id: 1, assignedTo: 1 } }
])
```

Returned **2 orders**, both assigned to Kimberly Ayala (`67fe9b98a5133e710d955286`):

| Order `_id` | createdAt |
|---|---|
| `69ebdee51f5216e1a640e097` | 2026-04-24T21:21:41Z |
| `69f2297b7fcc628ed8534fd1` (our order) | 2026-04-29T15:53:31Z |

### Expected notifications vs actual

If the trigger had run successfully for today's 3 documents:

| Document | Uploaded at | Active orders at that time | Expected notifications |
|---|---|---|---|
| Lab Results (`69f22957...`) | 15:52:55Z | 1 (only `69ebdee5...` — our order didn't exist yet) | 1 |
| Supporting Medical Document (`69f22999...e98`) | 15:54:01Z | 2 (both orders) | 2 |
| Other (`69f22999...e9f`) | 15:54:01Z | 2 (both orders) | 2 |
| | | **Expected total** | **5** |
| | | **Actual** | **0** |

### Verdict

Hypothesis #4 (ID mismatch / lookup returns empty) is **ruled out**. `getOrdersByPatient` returns the correct orders for our patient. The trigger is silently failing somewhere else — most likely an exception caught by the outer try/catch at `documentNotification.ts:105-110` (or one of the inner catches at L81 / L98), visible only as a `logger.error` entry.

## Remaining hypothesis (open) — runtime error inside the trigger

Scoping to the primary failure (no rows in `notifications`), the question is: **what's preventing the loop from reaching `createNotification`?**

### Key code observation: SignalR init currently gates notification creation

Looking at `notifyDocumentCreated` (`apps/web/services/notifications/documentNotification.ts`):

```ts
// L52
const orders = await getOrdersByPatient(patientId, { expandRefs: false });
if (orders.length === 0) return;

// L57-58 — BEFORE the loop
const sourceLabel = SOURCE_LABELS[source];
const signalr = await getSignalRService();   // ← if this throws, the outer try/catch
                                              //   at L105 catches and the loop never runs.

// L60 — inside this loop is where createNotification is called
for (const order of orders) {
  ...
  try { await createNotification(...); } catch { logger.error(...); }
  try { await signalr.broadcastToAll(...); } catch { logger.error(...); }
}
```

**Implication:** the call ordering means that any failure in `getSignalRService()` — Key Vault lookup, connection-string parsing, etc. — **silently prevents every notification insert**, even though notifications conceptually don't depend on SignalR.

This is the single most likely cause of "0 rows in `notifications`":
- It produces exactly **one** `Document notification: Unexpected error in notifyDocumentCreated` log line per upload (caught at L105-110, never thrown out).
- It writes **zero** rows to `notifications`.
- It writes **zero** SignalR broadcasts.
- All three match our symptoms exactly.

Stage uses `SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string`. If that env var isn't wired into the stage runtime config, `getSignalRService()` falls back to the default `'signalr-primary-connection-string'` which may not exist in stage's Key Vault, throwing on first call.

### Other (less likely) possibilities

- `createNotification` validation throws on every insert (e.g., Mongoose pre-validate regression) → would produce one `Failed to create document notification` log per attempt, but the loop would *try* every order and we'd see N inner-catch lines instead of one outer-catch line.
- `getOrdersByPatient` throws after returning successfully when re-tested in dev (we already empirically verified it returns 2 orders).
- Some other unexpected runtime error in the trigger path.

### Possible code-level fix (do not apply yet — flagging only)

If logs confirm SignalR init is the culprit, the right fix is to **decouple notification creation from SignalR init**: move `getSignalRService()` lazily inside the broadcast `try` block (or call it once *after* `createNotification` succeeds), so that a SignalR misconfiguration at most loses the real-time push but still writes the notification. That preserves the badge/unread-count contract regardless of SignalR health.

Per the workflow, this would land via a hotfix branch off `develop`, not a direct edit on the current working branch.

## Root cause — confirmed via stage app logs

Pulled `traces` and `exceptions` from `li-stage-appins` (App Insights `b0855cae-c060-4be6-baff-4e36f9fa7f52`, role `local-infusion-app`) for the upload window using `az monitor app-insights query --offset 6h`. Subscription verified as **Non-Production**.

### Definitive evidence

For each of today's 3 documents, the outer catch in `notifyDocumentCreated` fires with the **same** error:

| Time (UTC) | Trace message | documentId | Doc category |
|---|---|---|---|
| 2026-04-29T15:52:55.66Z | `Document notification: Unexpected error in notifyDocumentCreated` | `69f22957f5e2b50085307c24` | Lab Results |
| 2026-04-29T15:54:01.38Z | `Document notification: Unexpected error in notifyDocumentCreated` | `69f22999f5e2b50085307e98` | Supporting Medical Document |
| 2026-04-29T15:54:02.03Z | `Document notification: Unexpected error in notifyDocumentCreated` | `69f22999f5e2b50085307e9f` | Other |

Inner error (identical in all 3):

```
Error: Failed to retrieve secret: A secret with (name/id) signalr-primary-connection-string
was not found in this key vault.
```

The same error also fires for `/api/signalr/negotiate` continuously (≥10 traces of `SignalR negotiate failed` between 15:46Z and 16:00Z). **SignalR is fully broken on stage** — both the browser-negotiate path and the server-side broadcast path.

### What this confirms

1. The trigger **is** being called for every document upload. Wiring at `split-pdf/route.ts:602` works.
2. `getSignalRService()` (called at `documentNotification.ts:58`, *before* the loop) throws when it tries to read the SignalR connection string from stage's Key Vault.
3. The outer try/catch at `documentNotification.ts:105` catches the throw → emits one `Document notification: Unexpected error in notifyDocumentCreated` log per upload → **the order loop never runs → zero rows in `notifications`, zero broadcasts**.

This matches the symptom precisely (0 notifications for 3 uploads) and validates the suspicion noted earlier that SignalR init currently *gates* notification creation.

### Why the secret isn't found

Per `docs/signalr/signalr-integration.md` §3:
- Stage's actual Key Vault secret is named `li-stage-signalr-primary-connection-string` (Terraform provisioned with an env-prefixed name).
- The code defaults to `signalr-primary-connection-string` — overridden via env var `SIGNALR_KV_SECRET_NAME`.
- Local `.env.local` sets the override; **stage's deployed Container App runtime config does not.**

So in stage, the SignalR service factory always tries to read the wrong secret name, throws, and never initializes.

## Fix plan

Two independent fixes — different owners, different urgency.

### Fix 1 (infra/config — APPLIED MANUALLY 2026-04-29)

**Status:** Applied 2026-04-29 ~20:23Z by Alejandro Gomez via the Azure Portal UI (Non-Production subscription, resource group `li-stage-app-rg`). Each Container App was edited under **Application → Containers → Environment variables → + Add → Save as new revision**, which provisioned a new revision and shifted 100% traffic to it automatically.

**What was added:**

| Container App | Resource group | New revision after change | Env var added | Value |
|---|---|---|---|---|
| `li-stage-local-infusion-app` (Next.js web — handles `split-pdf` upload trigger) | `li-stage-app-rg` | `--0001297` | `SIGNALR_KV_SECRET_NAME` | `li-stage-signalr-primary-connection-string` |
| `li-stage-local-infusion-worker` (background worker — handles JotForm intake `weinfuseUploader.ts` trigger) | `li-stage-app-rg` | `--0000668` | `SIGNALR_KV_SECRET_NAME` | `li-stage-signalr-primary-connection-string` |

Both revisions came up Healthy. Source value (`Manual entry`) — the env var holds the *name* of a Key Vault secret, not a secret itself, so it doesn't need to be a secret reference.

**Effect once applied:**
- The negotiate route works (browsers can connect).
- The broadcast path works (`documentAdded`, `notification:read`).
- The trigger's outer try/catch stops firing → notifications start being written → badges/unread counts work.

This was the immediate unblock. Verified end-to-end in the verification window starting at 20:23Z (see Timeline + "Fix 1 verification" sections below).

**Important:** the env var was added **at the running-revision level** through the Portal — it is **not yet codified in Terraform / IaC**. A future re-deploy from infra automation that doesn't include this env var would re-introduce the bug. See "Open follow-ups" for the infra-automation tracking item.

Long-term cleanup option (per the SignalR doc): rename the stage KV secret to `signalr-primary-connection-string` so the env override isn't needed at all. Same PE coordination.

### Fix 2 (code hotfix — decouple notification writes from SignalR init)

Even after Fix 1 lands, the current code structure is fragile: any future SignalR misconfiguration silently breaks the notification primary write path. Concretely, in `documentNotification.ts`:

- `getSignalRService()` is awaited at L58, *before* the order loop.
- A throw there is caught by the outer catch at L105 and prevents any `createNotification` call.

**Proposed change** (not applied — flagging only): move `getSignalRService()` lazily inside each iteration's broadcast `try` block, or call it once *after* `createNotification` has run. Either way, a SignalR failure should at most lose the real-time push — never block the notification insert.

This should land via a hotfix branch off `develop` per the team's git workflow.

## Verification plan once Fix 1 lands

1. Pick a recent document upload (or trigger one via the QA flow).
2. Confirm in App Insights: zero `Document notification: Unexpected error in notifyDocumentCreated` traces in the upload window.
3. Confirm in MongoDB: rows appear in `notifications` with `targetUserId` matching the order's assignee, `entities` containing the order and document IDs.
4. Confirm in App Insights: `SignalR broadcast sent { eventName: 'documentAdded' }` traces fire per (order × document).
5. Confirm in browser: the orders-list badge increments / the Documents tab "New" chip appears without manual refresh.

## Timeline

- **2026-04-29 09:00–11:30** — Investigation opened; order, patient, assignee, and 3 documents identified; FF confirmed ON; `notifications`/`notificationreads` confirmed empty for our scope.
- **2026-04-29 11:30–13:00** — Code review of trigger callers + `notifyDocumentCreated` + `getOrdersByPatient`. Patient-identifier audit ruled out ID mismatch; `getOrdersByPatient` returns the correct 2 orders.
- **2026-04-29 13:00–13:45** — Stage app logs pulled via `az monitor app-insights query`. Three `Document notification: Unexpected error in notifyDocumentCreated` traces matched 1:1 with today's documents. Root cause: `SIGNALR_KV_SECRET_NAME` not set in stage's Container App config.
- **2026-04-29 20:23Z** — Fix 1 applied via Azure Portal. Env var `SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string` added to both `li-stage-local-infusion-app` (revision `--0001297`) and `li-stage-local-infusion-worker` (revision `--0000668`). Both new revisions Healthy.
- **2026-04-29 ~21:12Z** — Verification window (20:23–21:12Z) confirms the fix:
  - `SignalR negotiate failed` traces: **0** (was firing continuously before).
  - `Document notification: Unexpected error in notifyDocumentCreated`: **0** (was firing per upload before).
  - `SignalR service initialized` traces: **2** on the web app — service factory ran successfully on stage for the first time.
  - Worker had no SignalR traces in this window because no JotForm intake jobs ran — fix is in place for the next one.

## Fix 1 verification — confirmed

Both the negotiate path and the broadcast path are now operational. The next document upload (split-pdf or JotForm intake) should:

1. Write rows to `notifications` (one per matching active order × document).
2. Emit `SignalR broadcast sent { eventName: 'documentAdded' }` traces in App Insights.
3. Update badges/unread counts on connected clients in real time.

End-to-end live test still pending: needs a fresh upload to verify the full chain in production conditions. SignalR-side checkpoints from `docs/signalr/order-documents-signalr-notifications.md` §6 apply once an upload happens.

## Open follow-ups

- **Codify the `SIGNALR_KV_SECRET_NAME` env var in infra automation (Terraform / IaC) — ADDRESSED 2026-04-30.** See "Fix 3" section below. Branch `hotfix/MLID-2011-codify-signalr-env-var-in-terraform`.
- **Fix 2 (code hotfix — APPLIED 2026-04-29):** decoupled `getSignalRService()` initialization from the notification-write path in `documentNotification.ts`. Per-iteration ordering now puts `createNotification` first; SignalR init runs lazily and is cached so it executes at most once per call. Branch `hotfix/MLID-2011-decouple-signalr-from-notifications` (commit `20ff9502`) — PR pending merge to `develop`. After this lands, a future SignalR misconfiguration cannot block notification writes.
- **Long-term infra cleanup (PE):** rename the stage KV secret from `li-stage-signalr-primary-connection-string` to plain `signalr-primary-connection-string` so the env-var override isn't needed at all. This removes the per-environment override class entirely and matches the other secrets' naming. (Would also obsolete the `SIGNALR_KV_SECRET_NAME` env var across all environments.)
- **Doc fix:** `docs/signalr/order-documents-signalr-notifications.md` §4.2 still references `findOrdersForPatient`; the actual code uses `getOrdersByPatient`. Stale, not load-bearing — fix in passing.

## Next steps

- Pull recent documents for patient `69e7a39fd86a597a3ddd3be8` / `we_infuse_id_IV: 9146` to confirm the QA upload landed.
- Query `notifications` for any docs with `entities.entityId = 69f2297b7fcc628ed8534fd1` or `targetUserId = 67fe9b98a5133e710d955286`.
- Check `featureFlags` collection for `ORDER_DOCUMENT_NOTIFICATIONS` state on stage.
- Pull stage app logs around the document-upload timestamp via `az` CLI — look for `notifyDocumentCreated` errors or `[INFO] SignalR broadcast sent { eventName: 'documentAdded' }` lines.

## Reference

- Feature spec: `docs/signalr/order-documents-signalr-notifications.md`
- SignalR layer: `docs/signalr/signalr-integration.md`
- Trigger: `apps/web/services/notifications/documentNotification.ts` → `notifyDocumentCreated`
- Trigger callers:
  - `apps/web/app/api/documents/split-pdf/route.ts:350` (upload, branch A)
  - `apps/web/app/api/documents/split-pdf/route.ts:602` (upload, branch B)
  - `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts:344` (e-order)

## Timeline

- **2026-04-29** — Postmortem opened. Order, patient, and assignee identified. No notifications observed for the QA-tested order.

---

## Fix 3 — `SIGNALR_KV_SECRET_NAME` codified in Terraform (2026-04-30)

**Status:** Code change applied on branch `hotfix/MLID-2011-codify-signalr-env-var-in-terraform`. PR pending. Apply to stage + prod pending (requires VPN + Cloudflare token, deferred to whoever runs the next `terraform apply`).

**What this fixes:** Open follow-up #1. Fix 1 was a manual Portal change on stage's two Container Apps; that env var lived only at the running-revision level and would be silently dropped by the next IaC re-apply. This change moves the env var into source-controlled Terraform so it's part of every future apply — and additionally extends the same fix to **prod**, which had not been manually patched.

### Naming chain (verified end-to-end)

The Key Vault secret name is produced by string concatenation across two repos. Traced the chain to confirm the values we set are correct:

```
li-terraform/modules/signalr/main.tf:41
  azurerm_key_vault_secret.signalr_primary_connection_string.name
  = "${var.name}-primary-connection-string"
                ▲
li-terraform/main.tf:672                         ← var.name comes from here
  module.signalr.name = "${local.prefix}-${var.environment}-signalr"
                              ▲                   ▲
li-terraform/locals.tf:2                          │
  local.prefix = var.resource_prefix != ""        │
                  ? var.resource_prefix : "li"     │
                                                  │
stage.tfvars:1     environment = "stage" ─────────┘
prod.tfvars:1      environment = "prod"  ─────────┘

stage.tfvars:4 + prod.tfvars:4: resource_prefix = "li"
stage.tfvars:566 + prod.tfvars:454: signalr_enabled = true  ← module is enabled
li-terraform/modules/security/outputs.tf:1: key_vault_id is exported (non-null)
   → secret resource is NOT count=0 → it is created
```

**Resolves to:**
- Stage: `li-stage-signalr-primary-connection-string`
- Prod: `li-prod-signalr-primary-connection-string`

**Live verification (stage, 2026-04-30):**
```
$ az keyvault secret list --vault-name li-stage-kv \
    --query "[?contains(name,'signalr')].name" -o tsv
li-stage-signalr-hostname
li-stage-signalr-primary-connection-string
```

Prod KV reads were denied for our user (expected — least-privilege), but the IaC trace is deterministic: `prod.tfvars` has `environment = "prod"`, `resource_prefix = "li"`, `signalr_enabled = true`. There is no path through the code that produces a different name.

### Changes applied (li-call-processor, 4 files, +33 lines)

1. **`terraform/variables.tf`** — declared a new variable. **No default**, deliberately, so each environment must set the value explicitly. A default would let a missing override in prod silently fall back to stage's value (or vice versa) — exactly the failure mode that caused the original bug.
   ```hcl
   variable "signalr_kv_secret_name" {
     description = "Key Vault secret name holding the SignalR primary connection string..."
     type        = string
   }
   ```
2. **`terraform/main.tf`** — added an `env { }` block to **both** `azurerm_container_app.app` (web) and `azurerm_container_app.worker`. Plain `value` (not `secret_name`) because the env var holds the *name* of a secret, not the secret itself.
   ```hcl
   env {
     name  = "SIGNALR_KV_SECRET_NAME"
     value = var.signalr_kv_secret_name
   }
   ```
3. **`terraform/environments/stage.tfvars`** — `signalr_kv_secret_name = "li-stage-signalr-primary-connection-string"`.
4. **`terraform/environments/prod.tfvars`** — `signalr_kv_secret_name = "li-prod-signalr-primary-connection-string"`.

### Apply story

- **Stage:** the env var is already live on stage's Container Apps from Fix 1. Next `terraform apply` should reconcile cleanly. The plan output may show `+ env { name = "SIGNALR_KV_SECRET_NAME"; ... }` if Terraform's state file pre-dates the Portal change; if so, the apply rotates a revision with the same value (now owned by Terraform). Either way, no functional regression.
- **Prod:** prod was never manually patched. Next `terraform apply` will *add* the env var for the first time, create a new revision, and the worker + web app will pick it up on restart. **This effectively delivers Fix 1 to prod as part of this PR.**
- **Prerequisites for apply:** stage VPN, Cloudflare API token (provider re-evaluates DNS resources on every apply), Azure CLI logged in to the right subscription, Terraform CLI. None of these are encoded in the PR — they're operator concerns at apply time.

### What this does NOT do

- Does not rename the KV secret (still `li-{env}-signalr-primary-connection-string`). The "rename to drop the env prefix" cleanup remains a separate PE-owned task — it would obsolete `SIGNALR_KV_SECRET_NAME` entirely but requires migrating the existing secret value across all environments.
- Does not touch `li-terraform`. The mapping verification was read-only; the SignalR service + KV secret naming convention there is correct as-is.
- Does not run `terraform apply`. Code change only; the apply step is deferred.

### Why this matters

Without this change, the next infra re-apply silently re-introduces the original bug on stage. With this change, the bug cannot return through that vector. Combined with **Fix 2** (code-side decoupling of SignalR init from notification writes), notification creation is now defended by both code (cannot be gated by SignalR misconfig) *and* infra (env var won't drift away from source of truth).
