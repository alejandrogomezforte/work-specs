# MLID-2011 — Test Data for Document Notifications

## Resumption Context (2026-04-14)

This file is ready to execute from Step 5 onward. Quick state check for the next session:

- **Current branch**: `epic/MLID-2011-order-document-notifications` (has latest `develop` merged in plus the MLID-2159 Documents-tab bugfix).
- **MLID-2159 bugfix is in this branch**, so manual verification on the Documents tab will actually show documents (was always empty before).
- **D3-T1 badge code** (MLID-2084) is committed on `feature/MLID-2084-order-tracker-badges` but **not yet merged into epic**. Merge that branch into epic before UI verification so the badges render.
- **Stashed**: `stash@{0}` holds an unrelated `.mcp.json` change — ignore here; handled in a separate tiny PR.
- **Data below is populated** — tables under "Data Found" reflect real DB state as of 2026-04-14. All 4 patients are QA test accounts (no PHI risk).

## Purpose

Insert demo notification data into MongoDB so the Order Tracker notification badges can be tested in the UI without requiring SignalR to be fully set up in the cloud. The badges read from `notifications` + `notificationreads` collections — if data exists there, the UI will show it.

## Prerequisites

- VPN enabled (to reach staging MongoDB)
- Connection string from `apps/web/.env.local` → `MONGODB_URI`
- MongoDB MCP server configured, OR `mongosh` available

---

## Step 1 — Find 5 New Orders with Assigned Users

Query the `neworders` collection for orders that have `assignedTo` set (so there's a user to target the notification at).

**Note:** In `neworders`, `patient` is a **string reference** to `patients._id`, not an embedded subdocument. So `we_infuse_id_IV` is resolved via the patient join in Step 2, not filtered here.

**Note on `assignedTo`:** Legacy rows store an email address; current writes store the user's `_id` (24-hex ObjectId string). Per the tech lead, email-shaped rows are old orders — we filter them out via a regex so the notification query (`targetUserId === user._id`) will actually match.

```javascript
db.neworders.aggregate([
  {
    $match: {
      assignedTo: { $exists: true, $nin: [null, ''], $regex: /^[a-f0-9]{24}$/ },
      patient: { $exists: true, $ne: null },
      deleted: { $ne: true }
    }
  },
  { $sort: { createdAt: -1 } },
  { $limit: 5 },
  {
    $project: {
      _id: 1,
      displayId: 1,
      assignedTo: 1,
      patient: 1,
      orderType: 1,
      status: 1,
      createdAt: 1
    }
  }
])
```

**Save**: `_id`, `displayId`, `assignedTo`, `patient` (ID string) for each order.

---

## Step 2 — Look Up the Patients

Using the `we_infuse_id_IV` values from Step 1, find the corresponding patient MongoDB `_id` values.

```javascript
db.patients.find(
  { we_infuse_id_IV: { $in: [/* values from step 1 */] } },
  { _id: 1, we_infuse_id_IV: 1, first_name: 1, last_name: 1 }
)
```

**Save**: `_id` (MongoDB patient ID), `we_infuse_id_IV` for each patient.

---

## Step 3 — Find Documents for Those Patients

Using the patient MongoDB `_id` values from Step 2, find existing documents.

**Note on `documents.patientId` storage format:** Mixed. Some rows (Jotform intake webhook) store `we_infuse_id_IV` (numeric string, e.g. `"1078388"`); other rows (manual upload, most recent writes) store the patient's MongoDB `_id` as string. For orders in our test set, the MongoDB `_id` variant matches — that's what we query with.

```javascript
db.documents.find(
  { patientId: { $in: ['/* patient _id strings from step 2 */'] } },
  { _id: 1, patientId: 1, category: 1, description: 1, createdAt: 1 }
).sort({ createdAt: -1 }).limit(20)
```

**Save**: `_id`, `patientId`, `category` for each document.

---

## Step 4 — Look Up the Assigned Users

Using the `assignedTo` values from Step 1, find user details (so you know who to log in as when testing).

```javascript
db.users.find(
  { _id: { $in: [/* ObjectId values from assignedTo */] } },
  { _id: 1, name: 1, email: 1, role: 1 }
)
```

**Save**: `_id`, `name`, `email` — you need to log in as one of these users to see badges.

---

## Step 5 — Insert Demo Notifications

For each order+document pair, insert a `user`-type notification targeting the order's `assignedTo` user. This mimics what `notifyDocumentCreated()` would create.

```javascript
// Repeat for each order+document combination
db.notifications.insertOne({
  title: 'New Document: <category from step 3>',
  message: 'A new document was added via Upload.',
  type: 'user',
  priority: 'normal',
  targetUserId: '<assignedTo from step 1>',
  targetLocationId: null,
  targetRole: null,
  entities: [
    { entityType: 'order', entityId: '<order _id string from step 1>' },
    { entityType: 'document', entityId: '<document _id string from step 3>' }
  ],
  isRead: false,
  readBy: null,
  readAt: null,
  expiresAt: null,
  createdBy: 'system',
  createdAt: new Date(),
  updatedAt: new Date()
})
```

**Important**: 
- `targetUserId` = the `assignedTo` field from the order (string, not ObjectId)
- `entities[].entityId` = string representations of the `_id` values
- `type` must be `'user'` (not `'task'`) — this matches the epic's architecture decision
- Do NOT insert into `notificationreads` — that's the "read receipt" collection. Leaving it empty means all notifications show as unread (which is what we want for testing badges).

---

## Step 6 — Verify in UI

1. Enable `ORDER_DOCUMENT_NOTIFICATIONS` flag in `/admin/feature-flags`
2. Log in as one of the users from Step 4
3. Navigate to `/orders-tracker/new`
4. Orders from Step 1 should show a document icon badge with the unread count next to the Order ID

---

## Step 7 — Clean Up (after testing)

```javascript
// Remove all demo notifications created by this test
db.notifications.deleteMany({
  createdBy: 'system',
  message: 'A new document was added via Upload.'
})

// Remove any read receipts created during testing
db.notificationreads.deleteMany({
  notificationId: { $in: [/* notification _ids from step 5 */] }
})
```

---

## Data Found (fill in after running queries)

**Discovery method:** Single aggregation starting from `documents`, `$lookup`ing `neworders` by `patient == patientId`, filtering for `assignedTo` matching `^[a-f0-9]{24}$` and `deleted != true`, and taking the 15 most recent docs that have at least one assigned order. This surfaced **4 unique patients** (all QA test accounts — no PHI concerns) tied to **36 unique assigned orders**. Orders are capped below at the 3 most recent per patient for readability; expand as needed.

**Dataset at a glance:**

| Metric                              | Count |
|-------------------------------------|:-----:|
| Documents surfaced                  | 15    |
| Unique patients (all QA test data)  | 4     |
| Total assigned orders (full set)    | 36    |
| Orders included in table (capped)   | 10    |
| Unique assignees (users)            | 7     |

**Why this dataset is good for testing notifications:**

- All 4 patients are **QA test records** (`Test Request`, `Justin Tracey QA`, `LISA TEST Q`, `Amber Testing eOrder`) → **zero PHI risk** when seeding notifications.
- **LISA TEST Q** — 18 assigned orders tied to a single doc. Great for testing the "same doc shown on N orders" behavior (since docs are linked to patients, not orders).
- **Justin Tracey QA** — 8 docs tied to 1 order. Great for testing "many unread docs on one order" badge counts.
- **Kierra Fulford** and **Callie Avila** each own 2 orders → multi-order badge testing per user.
- **alex lukashevich** is `admin` → lets us contrast admin vs regular-user role behavior.

### Documents

15 recent documents that belong to patients with at least one assigned order. Sorted desc by `createdAt`.

| # | _id                        | patientId                    | category                      | description                                                             | createdAt (UTC)           |
|---|----------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------|---------------------------|
|  1 | 69de22d0db1282f388db61d0  | 69b2b217deaef8ec97bbbf0a     | Prior Authorization           | Copay                                                                   | 2026-04-14 11:19:44.730   |
|  2 | 69d3a9fb082b98de0f34bacc  | 69b2b217deaef8ec97bbbf0a     | Lab Results                   | Quest Diagnostics — CMP/CBC/STRATIFY JCV                                | 2026-04-06 12:41:31.496   |
|  3 | 69cf8e9a73f54d46ff950cf2  | 69cf89d202323d384fe6dc8e     | Insurance Card                | QA Test                                                                 | 2026-04-03 09:55:38.439   |
|  4 | 69cf8c7573f54d46ff950c5e  | 69cf89d202323d384fe6dc8e     | Order                         | Order                                                                   | 2026-04-03 09:46:29.185   |
|  5 | 69cf8aee73f54d46ff9507bb  | 69cf89d202323d384fe6dc8e     | Order                         | Order                                                                   | 2026-04-03 09:39:58.115   |
|  6 | 69cf8aec73f54d46ff9507b4  | 69cf89d202323d384fe6dc8e     | Lab Results                   | Cumulative BMP/CMP/Lipid panel reports                                  | 2026-04-03 09:39:56.266   |
|  7 | 69cf8aeb73f54d46ff9507af  | 69cf89d202323d384fe6dc8e     | Supporting Medical Document   | BMD test — osteoporotic T-score                                         | 2026-04-03 09:39:55.745   |
|  8 | 69cf8aeb73f54d46ff9507aa  | 69cf89d202323d384fe6dc8e     | Insurance Card                | Anthem BCBS + Medicare cards                                            | 2026-04-03 09:39:55.063   |
|  9 | 69cf8aea73f54d46ff9507a4  | 69cf89d202323d384fe6dc8e     | Referral Copy                 | Referral Copy                                                           | 2026-04-03 09:39:54.076   |
| 10 | 69cf89d373f54d46ff950572  | 69cf89d202323d384fe6dc8e     | Supporting Medical Document   | It is labs                                                              | 2026-04-03 09:35:15.345   |
| 11 | 69cbc7edc62d1cbfc4f198a6  | 6989d559e2d8c6414306d17b     | Order                         | Order                                                                   | 2026-03-31 13:11:09.288   |
| 12 | 69cbb855c62d1cbfc4f17af0  | 69cbb84e02323d384fe622e5     | Order                         | Stelara (ustekinumab) IV order for migraine                             | 2026-03-31 12:04:37.649   |
| 13 | 69cbb854c62d1cbfc4f17ae0  | 69cbb84e02323d384fe622e5     | Demographics                  | Patient demographics (DOB, address, contacts)                           | 2026-03-31 12:04:36.099   |
| 14 | 69cbb853c62d1cbfc4f17adb  | 69cbb84e02323d384fe622e5     | Lab Results                   | Northwell Health — CBC, metabolic, lipid profile                        | 2026-03-31 12:04:35.586   |
| 15 | 69cbb852c62d1cbfc4f17ad5  | 69cbb84e02323d384fe622e5     | Supporting Medical Document   | Referral checklist, cardiac cath report, progress notes                 | 2026-03-31 12:04:34.592   |

### Patients

4 unique patients (all QA test accounts).

| # | _id (MongoDB)            | we_infuse_id_IV | name                    | # docs | # assigned orders |
|---|--------------------------|-----------------|-------------------------|:------:|:-----------------:|
| 1 | 69b2b217deaef8ec97bbbf0a | 8529            | Test Request            | 2      | 4                 |
| 2 | 69cf89d202323d384fe6dc8e | 8905            | Justin Tracey QA        | 8      | 1                 |
| 3 | 6989d559e2d8c6414306d17b | 8081            | LISA TEST Q             | 1      | 18                |
| 4 | 69cbb84e02323d384fe622e5 | 8809            | Amber Testing eOrder    | 4      | 13                |

### Orders (assigned only — top 3 most recent per patient)

| # | _id                        | displayId | assignedTo                 | patient (ID)              | createdAt (UTC)           | orderType | status    | patient name           |
|---|----------------------------|-----------|----------------------------|---------------------------|---------------------------|-----------|-----------|------------------------|
|  1 | 69de2b8a0971dd1aa4b82441  | 05HP      | 67fe5a27c4d62cde04add191   | 69b2b217deaef8ec97bbbf0a  | 2026-04-14 11:56:58.737   | Transfer  | —         | Test Request           |
|  2 | 69de2b6b0971dd1aa4b8243f  | 05HO      | 67fe5a27c4d62cde04add191   | 69b2b217deaef8ec97bbbf0a  | 2026-04-14 11:56:27.711   | New Start | —         | Test Request           |
|  3 | 69ba7923688c98bb931496ef  | 05Ex      | 67fe9b98a5133e710d955294   | 69b2b217deaef8ec97bbbf0a  | 2026-03-18 10:06:27.233   | Transfer  | —         | Test Request           |
|  4 | 69cf9aee03a194bd1a65a199  | 05Gy      | 686eb246db18cb88c3956078   | 69cf89d202323d384fe6dc8e  | 2026-04-03 10:48:14.818   | New Start | —         | Justin Tracey QA       |
|  5 | 69a55c9cd67d0c3ea0432544  | 05Dy      | 67fe9b98a5133e710d95527e   | 6989d559e2d8c6414306d17b  | 2026-03-02 09:47:08.927   | New Start | —         | LISA TEST Q            |
|  6 | 69a55c9cd67d0c3ea0432545  | 05Dy-1    | 67fe9b98a5133e710d95527e   | 6989d559e2d8c6414306d17b  | 2026-03-02 09:47:08.927   | New Start | —         | LISA TEST Q            |
|  7 | 69a55af6d67d0c3ea043253e  | 05Dv      | 67fe9b98a5133e710d955286   | 6989d559e2d8c6414306d17b  | 2026-03-02 09:40:06.933   | New Start | —         | LISA TEST Q            |
|  8 | 69cc126a86c104551909ddd5  | 05Gi      | 682625f5f91fa5e600a51f0a   | 69cbb84e02323d384fe622e5  | 2026-03-31 18:28:58.338   | New Start | —         | Amber Testing eOrder   |
|  9 | 69cc11ce86c104551909ddd3  | 05Gh      | 682625f5f91fa5e600a51f0a   | 69cbb84e02323d384fe622e5  | 2026-03-31 18:26:22.364   | New Start | —         | Amber Testing eOrder   |
| 10 | 69cc116986c104551909ddd0  | 05Gg      | 696020a37fd0f57284c5b121   | 69cbb84e02323d384fe622e5  | 2026-03-31 18:24:41.426   | New Start | —         | Amber Testing eOrder   |

### Users (log in as)

7 unique assignees across the 10 capped orders above.

| # | _id                        | name                  | email                              | role  | orders               |
|---|----------------------------|-----------------------|------------------------------------|-------|----------------------|
| 1 | 67fe5a27c4d62cde04add191   | Kierra Fulford        | kfulford@mylocalinfusion.com       | user  | #1, #2 (Test Req)    |
| 2 | 67fe9b98a5133e710d955294   | Destiny Gray          | dgray@mylocalinfusion.com          | user  | #3 (Test Req)        |
| 3 | 686eb246db18cb88c3956078   | Olga Turovych         | olha.turovych@fortegrp.com         | user  | #4 (Justin)          |
| 4 | 67fe9b98a5133e710d95527e   | Callie Avila          | cavila@mylocalinfusion.com         | user  | #5, #6 (LISA)        |
| 5 | 67fe9b98a5133e710d955286   | Kimberly Ayala        | kayala@mylocalinfusion.com         | user  | #7 (LISA)            |
| 6 | 682625f5f91fa5e600a51f0a   | alex lukashevich      | alex.lukashevich@fortegrp.com      | admin | #8, #9 (Amber)       |
| 7 | 696020a37fd0f57284c5b121   | Melissa Waterman      | mwaterman@mylocalinfusion.com      | user  | #10 (Amber)          |

### Notifications Inserted

| # | notification _id           | targetUserId             | order _id (displayId)            | document _id (category)                            |
|---|----------------------------|--------------------------|----------------------------------|----------------------------------------------------|
| 1 | 69dfd3ecadc73ef87c73c047   | 67fe5a27c4d62cde04add191 | 69de2b8a0971dd1aa4b82441 (05HP)  | 69de22d0db1282f388db61d0 (Prior Authorization)     |
| 2 | 69dfe246adc73ef87c73c048   | 67fe5a27c4d62cde04add191 | 69de2b6b0971dd1aa4b8243f (05HO)  | 69d3a9fb082b98de0f34bacc (Lab Results)              |
| 3 | 69dfe246adc73ef87c73c049   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8e9a73f54d46ff950cf2 (Insurance Card)           |
| 4 | 69dfe246adc73ef87c73c04a   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8c7573f54d46ff950c5e (Order)                    |
| 5 | 69dfe246adc73ef87c73c04b   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8aee73f54d46ff9507bb (Order)                    |
| 6 | 69dfe246adc73ef87c73c04c   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8aec73f54d46ff9507b4 (Lab Results)              |
| 7 | 69dfe246adc73ef87c73c04d   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8aeb73f54d46ff9507af (Supporting Medical Doc)   |
| 8 | 69dfe246adc73ef87c73c04e   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8aeb73f54d46ff9507aa (Insurance Card)           |
| 9 | 69dfe246adc73ef87c73c04f   | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf8aea73f54d46ff9507a4 (Referral Copy)            |
| 10 | 69dfe246adc73ef87c73c050  | 695c25a530f7a75a8b1ed8c8 | 69cf9aee03a194bd1a65a199 (05Gy)  | 69cf89d373f54d46ff950572 (Supporting Medical Doc)   |
| 11 | 69dfe246adc73ef87c73c051  | 67fe9b98a5133e710d95527e | 69a55c9cd67d0c3ea0432544 (05Dy)  | 69cbc7edc62d1cbfc4f198a6 (Order)                    |
| 12 | 69dfe246adc73ef87c73c052  | 682625f5f91fa5e600a51f0a | 69cc126a86c104551909ddd5 (05Gi)  | 69cbb855c62d1cbfc4f17af0 (Order)                    |
| 13 | 69dfe246adc73ef87c73c053  | 682625f5f91fa5e600a51f0a | 69cc11ce86c104551909ddd3 (05Gh)  | 69cbb854c62d1cbfc4f17ae0 (Demographics)             |
| 14 | 69dfe246adc73ef87c73c054  | 696020a37fd0f57284c5b121 | 69cc116986c104551909ddd0 (05Gg)  | 69cbb853c62d1cbfc4f17adb (Lab Results)              |
| 15 | 69dfe246adc73ef87c73c055  | 682625f5f91fa5e600a51f0a | 69cc126a86c104551909ddd5 (05Gi)  | 69cbb852c62d1cbfc4f17ad5 (Supporting Medical Doc)   |
