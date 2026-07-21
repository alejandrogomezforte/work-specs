# [MLID-2822] User-reference scan — collections that track a user id

Companion to the email case-sensitivity audit. Purpose: before deleting/merging a duplicate user account, find every record in the system that references that user, so we know if the account has "trailing" data. Suppress PHI when sharing output (ids are fine; emails/names are not).

## The critical fact: three reference shapes, no single convention

A user id is stored three different ways depending on the collection. You cannot tell which from the field name — it was verified from the code that writes each field.

| Shape | How it is stored | Example collections |
|-------|------------------|---------------------|
| **String = user `_id` hex** | 24-char hex string of `users._id` | orders, notifications, notificationreads, intakes.processedBy |
| **Real `ObjectId`** | true `ObjectId` ref to `users._id` | weeklyproviderreports, weeklyproviderreportentries, apiKeys.serviceUserId |
| **String = email** | the user's email address, NOT the id | almost all admin-config collections |

Consequence: to scan a user fully you must search by **id hex**, by **email**, and (because these accounts have mixed-case emails) by **both case variants of the email**. Searching by id alone is not enough.

## Collection map

### A. Keyed by user `_id` hex string

| Collection | Field(s) |
|------------|----------|
| `neworders`, `maintenanceorders`, `leadorders` | `createdBy`, `updatedBy`, `assignedBy`, `assignedTo`, `drugSwitchContext[].user_id`, `noGoContext[].user_id` |
| `orderstrackercache` | `createdBy`, `updatedBy`, `assignedBy`, `assignedTo` (derived cache of the orders) |
| `notifications` | `targetUserId`, `readBy`, `createdBy` |
| `notificationreads` | `userId` |
| `intakes` | `processedBy.userId` |

### B. Keyed by real `ObjectId`

| Collection | Field(s) |
|------------|----------|
| `weeklyproviderreports` | `createdBy`, `deletedBy` |
| `weeklyproviderreportentries` | `editHistory[].editedBy` |
| `apiKeys` | `serviceUserId` (points at an auto-created service-account user) |

### C. Keyed by email string

| Collection | Field(s) |
|------------|----------|
| `hospitalSystems`, `insurance_payors`, `payor_guideline_links`, `orderprotocols`, `libiosimilars`, `drugRequirements`, `drugSettings`, `googleDriveConnections`, `integrations`, `featureFlags`, `holidays`, `chatConfigurations` | `createdBy`, `updatedBy` |
| `lidrugs` | `createdBy`, `updatedBy`, `discontinuedBy` |
| `configurations` | `createdBy` |
| `contentlibraryitems` | `uploadedBy` |
| `chatthreads` | `userId`, `lastFeedback.submittedBy` |
| `chatmessages` | `userId`, `feedback.submittedBy`, `feedback.reviewed_by`, `feedback.rating_modified_by` |
| `weeklyproviderreportsends` | `resolvedBy` (ops free-text, format unconfirmed) |

### D. Mixed per-row (id hex OR email) — search BOTH

| Collection | Field(s) |
|------------|----------|
| `auditLogs` | `actorId` (id hex for order/patient rows, email for payor/guideline rows); `actor` (email, only on `api_key_lifecycle` rows) |
| `clinicalReviews` | `createdBy`, `updatedBy`, `completedBy`, `labResults[].userFeedback.reviewedBy`, `otherDocuments[].createdBy` |
| `documents` | `createdBy` |
| `fileaccesses` | `userId` |
| `users` | `deletedBy` (self-referential) |

### Not a user reference (do NOT scan)

`patients.callsHistory[].targetUserId` looks like a user ref but is set to the patient's own `_id` (call-automation target), not a staff user.

### Sentinel values that are not users (will not false-match an id/email)

`"Auto Assigned"`, `"system"`, `"admin"`, `"unknown"`, `anonymous_*`.

## The scan query (counts summary)

Run in the **Aggregation** tab on the `neworders` collection (the base). It uses `$unionWith` to sweep every other collection and returns one row per collection that has a match, as `{ n: <count>, source: "<collection>" }`. Collections with zero matches simply do not appear. An unknown/misspelled collection name in `$unionWith` is treated as empty (no error), so a missing row means zero.

Before running, find-and-replace all three tokens in the text:
- `<<USER_ID>>` → the account's 24-char `_id` hex
- `<<USER_EMAIL>>` → the account's email (run once per case variant if the stored email is mixed-case: once with the exact stored value, once lowercased)

```
[
  { "$match": { "$or": [
      { "createdBy": "<<USER_ID>>" }, { "updatedBy": "<<USER_ID>>" },
      { "assignedBy": "<<USER_ID>>" }, { "assignedTo": "<<USER_ID>>" },
      { "drugSwitchContext.user_id": "<<USER_ID>>" }, { "noGoContext.user_id": "<<USER_ID>>" }
  ] } },
  { "$count": "n" }, { "$addFields": { "source": "neworders" } },

  { "$unionWith": { "coll": "maintenanceorders", "pipeline": [
      { "$match": { "$or": [
          { "createdBy": "<<USER_ID>>" }, { "updatedBy": "<<USER_ID>>" },
          { "assignedBy": "<<USER_ID>>" }, { "assignedTo": "<<USER_ID>>" },
          { "drugSwitchContext.user_id": "<<USER_ID>>" }, { "noGoContext.user_id": "<<USER_ID>>" }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "maintenanceorders" } }
  ] } },

  { "$unionWith": { "coll": "leadorders", "pipeline": [
      { "$match": { "$or": [
          { "createdBy": "<<USER_ID>>" }, { "updatedBy": "<<USER_ID>>" },
          { "assignedBy": "<<USER_ID>>" }, { "assignedTo": "<<USER_ID>>" },
          { "drugSwitchContext.user_id": "<<USER_ID>>" }, { "noGoContext.user_id": "<<USER_ID>>" }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "leadorders" } }
  ] } },

  { "$unionWith": { "coll": "orderstrackercache", "pipeline": [
      { "$match": { "$or": [
          { "createdBy": "<<USER_ID>>" }, { "updatedBy": "<<USER_ID>>" },
          { "assignedBy": "<<USER_ID>>" }, { "assignedTo": "<<USER_ID>>" }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "orderstrackercache" } }
  ] } },

  { "$unionWith": { "coll": "notifications", "pipeline": [
      { "$match": { "$or": [
          { "targetUserId": "<<USER_ID>>" }, { "readBy": "<<USER_ID>>" }, { "createdBy": "<<USER_ID>>" }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "notifications" } }
  ] } },

  { "$unionWith": { "coll": "notificationreads", "pipeline": [
      { "$match": { "userId": "<<USER_ID>>" } },
      { "$count": "n" }, { "$addFields": { "source": "notificationreads" } }
  ] } },

  { "$unionWith": { "coll": "intakes", "pipeline": [
      { "$match": { "processedBy.userId": "<<USER_ID>>" } },
      { "$count": "n" }, { "$addFields": { "source": "intakes" } }
  ] } },

  { "$unionWith": { "coll": "clinicalReviews", "pipeline": [
      { "$match": { "$or": [
          { "createdBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } },
          { "updatedBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } },
          { "completedBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } },
          { "labResults.userFeedback.reviewedBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } },
          { "otherDocuments.createdBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "clinicalReviews" } }
  ] } },

  { "$unionWith": { "coll": "documents", "pipeline": [
      { "$match": { "createdBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } } },
      { "$count": "n" }, { "$addFields": { "source": "documents" } }
  ] } },

  { "$unionWith": { "coll": "fileaccesses", "pipeline": [
      { "$match": { "userId": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } } },
      { "$count": "n" }, { "$addFields": { "source": "fileaccesses" } }
  ] } },

  { "$unionWith": { "coll": "auditLogs", "pipeline": [
      { "$match": { "$or": [
          { "actorId": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } },
          { "actor": "<<USER_EMAIL>>" }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "auditLogs" } }
  ] } },

  { "$unionWith": { "coll": "users", "pipeline": [
      { "$match": { "deletedBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } } },
      { "$count": "n" }, { "$addFields": { "source": "users.deletedBy" } }
  ] } },

  { "$unionWith": { "coll": "weeklyproviderreports", "pipeline": [
      { "$match": { "$or": [
          { "createdBy": ObjectId("<<USER_ID>>") }, { "deletedBy": ObjectId("<<USER_ID>>") }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "weeklyproviderreports" } }
  ] } },

  { "$unionWith": { "coll": "weeklyproviderreportentries", "pipeline": [
      { "$match": { "editHistory.editedBy": ObjectId("<<USER_ID>>") } },
      { "$count": "n" }, { "$addFields": { "source": "weeklyproviderreportentries" } }
  ] } },

  { "$unionWith": { "coll": "apiKeys", "pipeline": [
      { "$match": { "$or": [
          { "serviceUserId": ObjectId("<<USER_ID>>") },
          { "createdBy": "<<USER_EMAIL>>" }, { "revokedBy": "<<USER_EMAIL>>" }
      ] } },
      { "$count": "n" }, { "$addFields": { "source": "apiKeys" } }
  ] } },

  { "$unionWith": { "coll": "weeklyproviderreportsends", "pipeline": [
      { "$match": { "resolvedBy": { "$in": ["<<USER_ID>>", "<<USER_EMAIL>>"] } } },
      { "$count": "n" }, { "$addFields": { "source": "weeklyproviderreportsends" } }
  ] } },

  { "$unionWith": { "coll": "hospitalSystems",   "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "hospitalSystems" } } ] } },
  { "$unionWith": { "coll": "insurance_payors",  "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "insurance_payors" } } ] } },
  { "$unionWith": { "coll": "payor_guideline_links", "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "payor_guideline_links" } } ] } },
  { "$unionWith": { "coll": "orderprotocols",   "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "orderprotocols" } } ] } },
  { "$unionWith": { "coll": "lidrugs",          "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" }, { "discontinuedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "lidrugs" } } ] } },
  { "$unionWith": { "coll": "libiosimilars",    "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "libiosimilars" } } ] } },
  { "$unionWith": { "coll": "drugRequirements", "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "drugRequirements" } } ] } },
  { "$unionWith": { "coll": "drugSettings",     "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "drugSettings" } } ] } },
  { "$unionWith": { "coll": "googleDriveConnections", "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "googleDriveConnections" } } ] } },
  { "$unionWith": { "coll": "integrations",     "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "integrations" } } ] } },
  { "$unionWith": { "coll": "featureFlags",     "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "featureFlags" } } ] } },
  { "$unionWith": { "coll": "holidays",         "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "holidays" } } ] } },
  { "$unionWith": { "coll": "configurations",   "pipeline": [ { "$match": { "createdBy": "<<USER_EMAIL>>" } }, { "$count": "n" }, { "$addFields": { "source": "configurations" } } ] } },
  { "$unionWith": { "coll": "chatConfigurations", "pipeline": [ { "$match": { "$or": [ { "createdBy": "<<USER_EMAIL>>" }, { "updatedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "chatConfigurations" } } ] } },
  { "$unionWith": { "coll": "contentlibraryitems", "pipeline": [ { "$match": { "uploadedBy": "<<USER_EMAIL>>" } }, { "$count": "n" }, { "$addFields": { "source": "contentlibraryitems" } } ] } },
  { "$unionWith": { "coll": "chatthreads",      "pipeline": [ { "$match": { "$or": [ { "userId": "<<USER_EMAIL>>" }, { "lastFeedback.submittedBy": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "chatthreads" } } ] } },
  { "$unionWith": { "coll": "chatmessages",     "pipeline": [ { "$match": { "$or": [ { "userId": "<<USER_EMAIL>>" }, { "feedback.submittedBy": "<<USER_EMAIL>>" }, { "feedback.reviewed_by": "<<USER_EMAIL>>" }, { "feedback.rating_modified_by": "<<USER_EMAIL>>" } ] } }, { "$count": "n" }, { "$addFields": { "source": "chatmessages" } } ] } }
]
```

Note on `ObjectId`: the three `ObjectId`-keyed blocks use `ObjectId("<<USER_ID>>")`. The Atlas Aggregation editor runs in JS mode and rejects the extended-JSON form `{ "$oid": "..." }` with `unknown operator: $oid`, so `ObjectId(...)` is required there. This makes the pipeline JS rather than strict JSON, which is fine for the Atlas Aggregation editor.

## Getting the actual records (not just counts)

The summary tells you which collections have hits. To see the real documents in a collection that showed a nonzero `n`, open that collection's **Find** tab and paste the same `$match` filter used in its block above (the object inside `"$match"`). That returns the whole documents. Doing it per-collection keeps the output readable and avoids pulling large `intakes`/order documents into one giant result.

## Caveats

1. Run the scan once per email case variant (stored value and lowercased) for accounts whose email is mixed-case.
2. Collection names come from the models; if you suspect a name is off, a `$unionWith` on a non-existent collection returns nothing silently, so double-check any collection you specifically care about against `apps/web/utils/constants/collections.ts`.
3. `orderstrackercache` is a derived cache of the orders — a hit there is not independent data, it mirrors `neworders`/`maintenanceorders`.
4. `apiKeys.serviceUserId` points at a service-account user, not a human login; a hit there means the account is wired to an API key and needs care beyond a simple delete.

## Scan results

Environment: **Production**. Date: 2026-07-15. Emails omitted (PHI); account `_id` kept. Each pair below are the two case-variants of a single address.

**Ids and roles verified 2026-07-16** via the collision query (returning `id`/`role`/`mixedCase` per member, no emails). The three `mixedCase: true` ids — `699f1d9db3bff5f408cc73f3`, `69f8985a50ec5fb019e9662e`, `69a0c0f0ac42a5b48e1cc985` — match exactly the three disable-migration ids, and their lowercase survivors (`699f1c9bb3bff5f408cc73f0`, `6a3c3d67c901ad3ec374abcb`, `69a0e190ac42a5b48e1cc994`) are confirmed. **All six accounts are `role: user`** (no admin), so disabling the mixed-case duplicates strips no elevated access. The earlier "`_id` user-supplied, unconfirmed" caveats in the tables below are now resolved.

**RESOLVED — DISABLED 2026-07-16 (production).** After the MLID-2822 case-insensitive code shipped, the three mixed-case duplicate accounts were disabled (`disabled: true`) in production — applied manually, no migration:

- `699f1d9db3bff5f408cc73f3`
- `69a0c0f0ac42a5b48e1cc985`
- `69f8985a50ec5fb019e9662e`

Their lowercase twins (`699f1c9bb3bff5f408cc73f0`, `69a0e190ac42a5b48e1cc994`, `6a3c3d67c901ad3ec374abcb`) remain enabled and now match at login via the case-insensitive lookup.

### Collision pair 1

| Account `_id` | Email case | Footprint (`source` : `n`) | Notes |
|---------------|-----------|----------------------------|-------|
| `699f1d9db3bff5f408cc73f3` | mixed-case | *(no rows)* — clean, zero references | Empty duplicate. Safe to remove/merge. |
| `699f1c9bb3bff5f408cc73f0` | lowercase | `fileaccesses` : 21 | The used account. 21 HIPAA file-access audit rows only; no orders/intakes/notifications/config/chat/apiKeys. `_id` was user-supplied and NOT yet confirmed via the `users` email lookup, so the empty id-keyed collections are only fully trustworthy once that id is verified; the `fileaccesses` hit is reliable via the email match. |

Reading: of this pair the lowercase account holds all activity (audit-log only, no operational/blocking data); the mixed-case account is an empty duplicate. `fileaccesses` is append-only compliance data — retain it; it does not block removing/merging the user record. Recommendation: keep the lowercase account, remove/merge the mixed-case one, after confirming the surviving account has the required role (check admin-vs-user on both first).

### Collision pair 2

| Account `_id` | Email case | Footprint (`source` : `n`) | Notes |
|---------------|-----------|----------------------------|-------|
| `69a0c0f0ac42a5b48e1cc985` | mixed-case | *(no rows)* — clean, zero references | Empty duplicate. |
| `69a0e190ac42a5b48e1cc994` | lowercase | *(no rows)* — clean, zero references | `_id` user-supplied, unconfirmed. |

Reading: **both** accounts of this pair are clean — no operational, audit, config, or chat references on either. This pair has no trailing data to preserve on either side, so the merge is low-risk once roles are checked; keep whichever account has the required role and remove the other.

### Collision pair 3

| Account `_id` | Email case | Footprint (`source` : `n`) | Notes |
|---------------|-----------|----------------------------|-------|
| `69f8985a50ec5fb019e9662e` | mixed-case | *(no rows)* — clean, zero references | Empty duplicate. |
| `6a3c3d67c901ad3ec374abcb` | lowercase | `fileaccesses` : 1 | The used account. 1 HIPAA file-access audit row only; no operational/blocking data. `fileaccesses` hit reliable via email; `_id` user-supplied, unconfirmed. |

Reading: same shape as pair 1 — the lowercase account holds the only footprint (a single HIPAA file-access row, append-only compliance data, non-blocking); the mixed-case account is an empty duplicate. Recommendation: keep the lowercase account, remove/merge the mixed-case one, after confirming role.

