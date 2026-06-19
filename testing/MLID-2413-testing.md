# MLID-2413 — Data Testing: Document unlink notification cleanup

Working notebook for pre-fix verification and post-fix regression of the order-document unlink flow. The bug (reported by Olha) is that when a document is unlinked from a new order via Patient Card → Documents tab, the notification badge on the order's Documents tab stays stale — and we need to confirm what *else* the unlink touches (or fails to touch), specifically the **order history log**.

- **Plan**: `docs/agomez/plans/MLID-2413-document-unlink-notification-cleanup.md`
- **Jira**: [MLID-2413](https://localinfusion.atlassian.net/browse/MLID-2413)
- **Branch**: `fix/MLID-2413-document-unlink-notification-cleanup` (not yet created — pre-testing first)
- **Feature flag**: `ORDER_DOCUMENT_NOTIFICATIONS` — must be ON locally to exercise the notification surface

---

## Test fixture (snapshot 2026-05-26)

> Captured via `node scripts/lookup-MLID-2413-fixture.mjs 6a15dda68c4205d331775425`.

### Patient

| Field | Value |
|---|---|
| `_id` | `6a15dda68c4205d331775425` |
| Name | Alex Test (Gomez) |
| `we_infuse_id_IV` | `9562` |
| `we_infuse_location_id` | `252` |
| Email | `alex.test@demo.com` |
| DOB | 27 May 1970 |
| Drug under review | Hydration (`6989fa7be13040640aae9f50`) |
| `treatment_authorized_date` | `2026-05-26T17:51:34.047Z` |

### Patient Documents (3 attached to the patient)

> Verified via MongoDB MCP against `local-infusion-db.documents` on 2026-05-26. Schema note: documents reference the patient via `patientId` (String), not `patient` (ObjectId).

| `_id` | `category` | `description` | `linkedOrderIds` | `pageCount` |
|---|---|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | Referral | `[]` | 2 |
| `6a15dda86b5a32726f4b0bf1` | Other | Other | `[]` | 1 |
| `6a15dda96b5a32726f4b0bf8` | **Order** | Order | `[]` | 1 |

All three share `intakeId: 6a15d3615bcb388f22e5be06` (same fax intake), `createdBy: 695c25a530f7a75a8b1ed8c8`, `ocrStatus: completed`, and were created on 2026-05-26 between 17:51:35 and 17:51:37 UTC.

> **Linkage constraint:** Only documents with `category: "Order"` can be linked to an order (i.e., have an order ID appended to `linkedOrderIds`). `Referral Copy` and `Other` documents cannot be linked. This means **`6a15dda96b5a32726f4b0bf8` is the only document in this fixture that participates in the link/unlink flow under test.**

#### Full document records (for hotfix verification)

**Document 1 — Referral Copy (`6a15dda76b5a32726f4b0be1`)**

| Field | Value |
|---|---|
| `_id` | `6a15dda76b5a32726f4b0be1` |
| `intakeId` | `6a15d3615bcb388f22e5be06` |
| `patientId` | `6a15dda68c4205d331775425` |
| `category` | `Referral Copy` |
| `description` | `Referral` |
| `pageCount` | `2` |
| `linkedOrderIds` | `[]` |
| `createdBy` | `695c25a530f7a75a8b1ed8c8` |
| `createdAt` | `2026-05-26T17:51:35.227Z` |
| `ocrStatus` | `completed` |
| `ocrCompletedAt` | `2026-05-26T17:51:48.313Z` |
| `blobPath` | `https://listageapp.blob.core.windows.net/documents/intakes/2026/05/26/6a15d3615bcb388f22e5be06/Fax.pdf` |
| `ocrJsonPath` | `intakes/2026/05/26/6a15d3615bcb388f22e5be06/Fax_ocr.json` |
| `ocrMarkdownPath` | `intakes/2026/05/26/6a15d3615bcb388f22e5be06/Fax_ocr.md` |

**Document 2 — Other (`6a15dda86b5a32726f4b0bf1`)**

| Field | Value |
|---|---|
| `_id` | `6a15dda86b5a32726f4b0bf1` |
| `intakeId` | `6a15d3615bcb388f22e5be06` |
| `patientId` | `6a15dda68c4205d331775425` |
| `category` | `Other` |
| `description` | `Other` |
| `pageCount` | `1` |
| `linkedOrderIds` | `[]` |
| `createdBy` | `695c25a530f7a75a8b1ed8c8` |
| `createdAt` | `2026-05-26T17:51:36.126Z` |
| `ocrStatus` | `completed` |
| `ocrCompletedAt` | `2026-05-26T17:51:42.905Z` |
| `blobPath` | `https://listageapp.blob.core.windows.net/documents/intakes/2026/05/26/6a15d3615bcb388f22e5be06/other_pages-2-2.pdf` |
| `ocrJsonPath` | `intakes/2026/05/26/6a15d3615bcb388f22e5be06/other_pages-2-2_ocr.json` |
| `ocrMarkdownPath` | `intakes/2026/05/26/6a15d3615bcb388f22e5be06/other_pages-2-2_ocr.md` |

**Document 3 — Order (`6a15dda96b5a32726f4b0bf8`)** — *target for link/unlink repro*

| Field | Value |
|---|---|
| `_id` | `6a15dda96b5a32726f4b0bf8` |
| `intakeId` | `6a15d3615bcb388f22e5be06` |
| `patientId` | `6a15dda68c4205d331775425` |
| `category` | `Order` |
| `description` | `Order` |
| `pageCount` | `1` |
| `linkedOrderIds` | `[]` |
| `createdBy` | `695c25a530f7a75a8b1ed8c8` |
| `createdAt` | `2026-05-26T17:51:37.084Z` |
| `ocrStatus` | `completed` |
| `ocrCompletedAt` | `2026-05-26T17:51:46.089Z` |
| `blobPath` | `https://listageapp.blob.core.windows.net/documents/intakes/2026/05/26/6a15d3615bcb388f22e5be06/order_pages-1-1.pdf` |
| `ocrJsonPath` | `intakes/2026/05/26/6a15d3615bcb388f22e5be06/order_pages-1-1_ocr.json` |
| `ocrMarkdownPath` | `intakes/2026/05/26/6a15d3615bcb388f22e5be06/order_pages-1-1_ocr.md` |

## Order snapshots — Order `05WF` (pre-fix baseline, against develop)

Tracking order `6a15df236b5a32726f4b0fd7` (displayId `05WF`) over time as link/unlink actions are exercised. Each snapshot captures the order's DB state, what `/orders-tracker/new/[id]/documents/` renders, and the notification surface counts at that moment.

> The original test order `6a15dda96b5a32726f4b0bfc` (displayId `05WE`) was retired in favor of this one. `05WF` is auto-assigned at creation, which lets us skip the manual assignment step.

### Snapshot 1 — 2026-05-26T17:57:55 UTC (order just created, no docs linked yet)

**Order state (from `neworders`):**

| Field | Value |
|---|---|
| `_id` | `6a15df236b5a32726f4b0fd7` |
| `displayId` | `05WF` |
| `orderType` | New Start |
| `drug` | `693885079879dc15c6a1de62` |
| `site` | `67b7083a8532e12fdce9d61a` |
| `provider` | `1710368154` (Dr. JOHN JOHN) |
| `providerPracticeFax` | `+12126627954` |
| `patient` | `6a15dda68c4205d331775425` |
| `received` | `2026-05-26T17:57:25.609Z` |
| `createdAt` / `lastUpdated` | `2026-05-26T17:57:55.565Z` |
| `assignedTo` | `695c25a530f7a75a8b1ed8c8` (auto-assigned) |
| `assignedBy` | `Auto Assigned` |
| `lastAssigned` | `2026-05-26T17:57:55.665Z` |

**Documents (`linkedOrderIds` in `documents` collection):**

| Document `_id` | `category` | `linkedOrderIds` |
|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | `[]` |
| `6a15dda86b5a32726f4b0bf1` | Other | `[]` |
| `6a15dda96b5a32726f4b0bf8` | Order | `[]` |

No document has `linkedOrderIds` referencing `6a15df236b5a32726f4b0fd7` in the DB.

**Visible in `/orders-tracker/new/6a15df236b5a32726f4b0fd7/documents/`: 2 of 3**

The order's Documents tab renders **2 documents** (Referral Copy + Other) — these surface automatically via patient/intake association and are not link-eligible. The **Order**-category document `6a15dda96b5a32726f4b0bf8` is **not** shown because it has not yet been linked, and only `category: "Order"` documents participate in the explicit link flow (see Linkage constraint above).

**Notifications / NotificationReads:**

- `db.notifications` for `(order 6a15df236b5a32726f4b0fd7 × document *)`: **2** (one per auto-attached non-Order document)
- `db.notificationreads` referencing this order: **0**

| `_id` | `title` | `entities` (doc) | `targetUserId` | `isRead` | `createdAt` |
|---|---|---|---|---|---|
| `6a15df236b5a32726f4b0fdc` | `New Document: Other` | `6a15dda86b5a32726f4b0bf1` (Other) | `695c25a530f7a75a8b1ed8c8` | `false` | `2026-05-26T17:57:55.762Z` |
| `6a15df236b5a32726f4b0fdf` | `New Document: Referral Copy` | `6a15dda76b5a32726f4b0be1` (Referral Copy) | `695c25a530f7a75a8b1ed8c8` | `false` | `2026-05-26T17:57:55.802Z` |

> **Correction to earlier hypothesis:** Notifications are NOT gated on the explicit link event. They are also emitted at order creation for every non-Order patient document that gets auto-attached to the new order (i.e., propagation fires on auto-attachment, not just explicit link). All notifications share `type: user`, `priority: normal`, `createdBy: system`, `targetLocationId: null`, `targetRole: null`.

**Audit log (`auditLogs` collection, what `/orders-tracker/new/6a15df236b5a32726f4b0fd7/order-history` renders): 3 entries**

Query: `db.auditLogs.find({ entityId: "6a15df236b5a32726f4b0fd7" }).sort({ timestamp: 1 })`. Schema reminder: `auditLogs.entityId` is a String, not ObjectId.

| # | `timestamp` (UTC) | `action` | `source` | `actorId` | `changes[0].field` | `changes[0].from` → `changes[0].to` |
|---|---|---|---|---|---|---|
| 1 | `2026-05-26T17:57:55.745Z` | `create` | `user` | `695c25a530f7a75a8b1ed8c8` | — | (no `changes` array — order create event) |
| 2 | `2026-05-26T17:57:55.769Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda86b5a32726f4b0bf1", category: "Other", source: "upload" }` |
| 3 | `2026-05-26T17:57:55.808Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda76b5a32726f4b0be1", category: "Referral Copy", source: "upload" }` |

Observations:

- The order's own `create` event is logged with `source: user, actorId: 695c25a530f7a75a8b1ed8c8`.
- The two **non-Order** patient documents (Referral Copy, Other) were each recorded as an `update` on the order with `field: documents` and `from: null`, `to: { _id, category, source }`. These are tagged `source: system` (auto-attachment, not user-initiated), with an empty `actorId`.
- **No entry exists for the Order-category document `6a15dda96b5a32726f4b0bf8`** — consistent with the linkage constraint (only Order-category docs are linked via the explicit flow, and that flow has not run yet).
- Expected behavior on link (Snapshot 2): a new `update` row with `field: documents`, `from: null`, `to: { _id: "6a15dda96b5a32726f4b0bf8", category: "Order", source: ? }`, `source: user`, `actorId: 695c25a530f7a75a8b1ed8c8`.
- Expected behavior on unlink (Snapshot 3) — **this is the open question driving MLID-2413**: does an audit entry get written (e.g., `from: {...}, to: null`)? Today's propagation orchestrator (`propagateDocumentNotificationToOrder` → `writeOrderDocumentAuditEntry`) only writes audit entries on the *add* side; we need to confirm what Snapshot 3 shows here.

### Snapshot 2 — 2026-05-26T18:52:13 UTC (after linking the Order-category doc)

Action performed between snapshots: linked document `6a15dda96b5a32726f4b0bf8` (category `Order`) to order `6a15df236b5a32726f4b0fd7` from Patient Card → Documents tab.

**Order state (from `neworders`):** unchanged from Snapshot 1.

The link does **not** touch the order document itself — `lastUpdated` is still `2026-05-26T17:57:55.665Z`, `updatedBy` is still `695c25a530f7a75a8b1ed8c8`. The link state lives on the `documents` collection (`linkedOrderIds`), not on the order.

**Documents (`linkedOrderIds` in `documents` collection):**

| Document `_id` | `category` | `linkedOrderIds` |
|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | `[]` |
| `6a15dda86b5a32726f4b0bf1` | Other | `[]` |
| `6a15dda96b5a32726f4b0bf8` | Order | `["6a15df236b5a32726f4b0fd7"]` |

The Order-category doc now references this order in its `linkedOrderIds`. Non-Order docs remain with empty arrays (confirms: only Order-category docs get an entry in `linkedOrderIds`).

**Visible in `/orders-tracker/new/6a15df236b5a32726f4b0fd7/documents/`: 3 of 3**

The order's Documents tab now renders **3 documents** — the Order-category doc shows up because it is now linked. The 2 non-Order docs continue to surface via patient/intake association.

**Notifications / NotificationReads:**

- `db.notifications` for `(order 6a15df236b5a32726f4b0fd7 × document *)`: **3** (+1 vs Snapshot 1; the link triggered propagation for the Order-category doc)
- `db.notificationreads` referencing this order: **0** (still no reads)

| `_id` | `title` | `entities` (doc) | `targetUserId` | `isRead` | `createdAt` |
|---|---|---|---|---|---|
| `6a15df236b5a32726f4b0fdc` | `New Document: Other` | `6a15dda86b5a32726f4b0bf1` (Other) | `695c25a530f7a75a8b1ed8c8` | `false` | `2026-05-26T17:57:55.762Z` |
| `6a15df236b5a32726f4b0fdf` | `New Document: Referral Copy` | `6a15dda76b5a32726f4b0be1` (Referral Copy) | `695c25a530f7a75a8b1ed8c8` | `false` | `2026-05-26T17:57:55.802Z` |
| **`6a15ebdd49e2fad58ff868b8`** | **`New Document: Order`** | **`6a15dda96b5a32726f4b0bf8` (Order)** | `695c25a530f7a75a8b1ed8c8` | `false` | **`2026-05-26T18:52:13.959Z`** |

New row is bolded. Unread badge count is now **3**.

**Audit log (`auditLogs` collection, what `/orders-tracker/new/6a15df236b5a32726f4b0fd7/order-history` renders): 4 entries (+1 vs Snapshot 1)**

| # | `timestamp` (UTC) | `action` | `source` | `actorId` | `changes[0].field` | `changes[0].from` → `changes[0].to` |
|---|---|---|---|---|---|---|
| 1 | `2026-05-26T17:57:55.745Z` | `create` | `user` | `695c25a530f7a75a8b1ed8c8` (String) | — | (order create event) |
| 2 | `2026-05-26T17:57:55.769Z` | `update` | `system` | `""` | `documents` | `null` → `{ _id: "6a15dda86b5a32726f4b0bf1", category: "Other", source: "upload" }` |
| 3 | `2026-05-26T17:57:55.808Z` | `update` | `system` | `""` | `documents` | `null` → `{ _id: "6a15dda76b5a32726f4b0be1", category: "Referral Copy", source: "upload" }` |
| **4** | **`2026-05-26T18:52:13.967Z`** | **`update`** | **`user`** | **`695c25a530f7a75a8b1ed8c8` (ObjectId)** | **`documents`** | **`null` → `{ _id: "6a15dda96b5a32726f4b0bf8", category: "Order", source: "upload" }`** |

New row is bolded. Schema note: `actorId` is stored as a **String** in the order `create` row but as an **ObjectId** in the link `update` row — same value, different BSON type. Worth flagging if the order-history renderer assumes one or the other.

### Snapshot 3 — TBD (after unlinking the Order-category doc — the bug repro)

> Run after the unlink action in "Pre-fix observations" Step 2 below.

_<observations>_

---

## Order snapshots — Order `05WL` (abandoned mid-test — surfaced the unassigned-to-assigned bug)

> **Status: abandoned.** Order `6a161fabd11fe1b093dc47da` was created via the manual-creation flow with `assignedTo: ""`. After manual assignment, no notifications appeared for the auto-attached non-Order docs — exposed a pre-existing reassignment-cascade bug (notification backfill missing for the empty-to-populated transition). Bug captured separately in `docs/agomez/postmortem/MLID-2391-unassigned-to-assigned-skips-notification-backfill.md`. Continued testing of the MLID-2413 fix moved to order `05WM` below.

### Snapshot 1 — 2026-05-26T22:33:15 UTC (order just created, UNASSIGNED, no docs linked)

**Order state (from `neworders`):**

| Field | Value |
|---|---|
| `_id` | `6a161fabd11fe1b093dc47da` |
| `displayId` | `05WL` |
| `orderType` | New Start |
| `drug` | `693885079879dc15c6a1de62` |
| `site` | `67b7083a8532e12fdce9d61d` |
| `provider` | `1710368154` (Dr. JOHN JOHN) |
| `providerPracticeFax` | `+12126627954` |
| `patient` | `6a15dda68c4205d331775425` |
| `received` | `2026-05-26T22:31:58.250Z` |
| `createdAt` / `lastUpdated` | `2026-05-26T22:33:15.954Z` |
| `assignedTo` | `""` (**empty — unassigned, manual assignment required**) |
| `assignedBy` | `695c25a530f7a75a8b1ed8c8` |
| `lastAssigned` | `2026-05-26T22:33:15.954Z` |
| `audit` | `"Yes"` (new field, not seen on 05WF) |

**Documents (`linkedOrderIds` in `documents` collection):**

| Document `_id` | `category` | `linkedOrderIds` |
|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | `[]` |
| `6a15dda86b5a32726f4b0bf1` | Other | `[]` |
| `6a15dda96b5a32726f4b0bf8` | Order | `["6a15df236b5a32726f4b0fd7"]` (still linked to 05WF) |

**Expected UI in `/orders-tracker/new/6a161fabd11fe1b093dc47da/documents/`: 2 of 3**

Same as the equivalent state on 05WF — the 2 non-Order docs surface via patient/intake association, the Order-category doc is not visible because it's not linked to THIS order.

**Notifications / NotificationReads:**

- `db.notifications` for `(order 6a161fabd11fe1b093dc47da × document *)`: **0**
- `db.notificationreads` referencing this order: **0**

Both zero, even though 2 non-Order docs were auto-attached. This is **different from 05WF's Snapshot 1**: 05WF had `assignedTo` populated at creation, which triggered notifications for the auto-attached non-Order docs. 05WL has `assignedTo: ""`, so the propagation step that writes notifications (`writeOrderDocumentNotification`, gated on `order.assignedTo` truthy) was skipped — confirming the `assignedTo` truthy gate in the orchestrator.

**Audit log (`auditLogs` for `6a161fabd11fe1b093dc47da`): 3 entries**

| # | `timestamp` (UTC) | `action` | `source` | `actorId` | `changes[0].field` | `changes[0].from` → `changes[0].to` |
|---|---|---|---|---|---|---|
| 1 | `2026-05-26T22:33:16.416Z` | `create` | `user` | `695c25a530f7a75a8b1ed8c8` | — | (order create event) |
| 2 | `2026-05-26T22:33:16.524Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda86b5a32726f4b0bf1", category: "Other", source: "upload" }` |
| 3 | `2026-05-26T22:33:18.146Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda76b5a32726f4b0be1", category: "Referral Copy", source: "upload" }` |

Observations:

- Audit entries for the auto-attached non-Order docs ARE written even when `assignedTo` is empty. Confirms: audit writes are independent of the notification write step (both run inside the same orchestrator, but only the notification step gates on `assignedTo`).
- On the order-history page, the "Change" column should render entries 2 and 3 as `New Document received (Document type = Other)` and `New Document received (Document type = Referral Copy)` (legacy format — non-Order category, unchanged by the fix).

### Snapshot 2+ — n/a (abandoned)

The remaining 05WL snapshots (assignment, link, unlink, re-link) were not captured against this order — testing moved to `05WM` once the unassigned-to-assigned bug was confirmed.

---

## Order snapshots — Order `05WM` (active post-fix verification on `fix/MLID-2413-document-unlink-notification-cleanup`)

Tracking order `6a162261d11fe1b093dc48f4` (displayId `05WM`). This order is **auto-assigned at creation** (matching the 05WF baseline pattern), which sidesteps the unassigned-to-assigned bug surfaced on 05WL. Each subsequent snapshot captures the order's DB state, what `/orders-tracker/new/[id]/documents/` renders, and the notification surface counts at that moment.

> **Carry-over state to be aware of**: the Order-category document `6a15dda96b5a32726f4b0bf8` has `linkedOrderIds: ["6a15df236b5a32726f4b0fd7"]` from the earlier 05WF test session. When linked to 05WM, the field becomes `["6a15df236b5a32726f4b0fd7", "6a162261d11fe1b093dc48f4"]`. The fix's behavior is unaffected — `handleOrderLink` and `handleOrderUnlink` operate on the `(order, document)` tuple, so the carry-over link to 05WF stays untouched.

### Snapshot 1 — 2026-05-26T22:44:49 UTC (order created, auto-assigned, 2 docs auto-attached, 0 docs linked)

**Order state (from `neworders`):**

| Field | Value |
|---|---|
| `_id` | `6a162261d11fe1b093dc48f4` |
| `displayId` | `05WM` |
| `orderType` | New Start |
| `drug` | `693885079879dc15c6a1de62` |
| `site` | `67b7083a8532e12fdce9d61a` |
| `provider` | `1710368154` (Dr. JOHN JOHN) |
| `providerPracticeFax` | `+12126627954` |
| `patient` | `6a15dda68c4205d331775425` |
| `received` | `2026-05-26T22:37:17.876Z` |
| `createdAt` / `lastUpdated` | `2026-05-26T22:44:49.712Z` / `2026-05-26T22:44:50.208Z` |
| `assignedTo` | `67fe9b98a5133e710d95527e` |
| `assignedBy` | `Auto Assigned` |
| `lastAssigned` | `2026-05-26T22:44:50.208Z` |
| `createdBy` | `695c25a530f7a75a8b1ed8c8` |
| `audit` | `"Yes"` |

> Note: `assignedTo` (`67fe9b98a5133e710d95527e`) is different from `createdBy` (`695c25a530f7a75a8b1ed8c8`). The 2 notifications below target the assignee, not the creator — relevant for the "New" chip / "Mark as Read" per-row controls, which only render for the current user when they match `targetUserId`.

**Documents (`linkedOrderIds` in `documents` collection):**

| Document `_id` | `category` | `linkedOrderIds` |
|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | `[]` |
| `6a15dda86b5a32726f4b0bf1` | Other | `[]` |
| `6a15dda96b5a32726f4b0bf8` | Order | `["6a15df236b5a32726f4b0fd7"]` (carry-over to 05WF, NOT to this order yet) |

**Expected UI in `/orders-tracker/new/6a162261d11fe1b093dc48f4/documents/`: 2 of 3**

The 2 non-Order docs surface via patient/intake association. The Order-category doc is not visible because it is not linked to this order.

**Notifications: 2**

| `_id` | `title` | `entities` (doc) | `targetUserId` | `isRead` | `createdAt` |
|---|---|---|---|---|---|
| `6a162262d11fe1b093dc48fb` | `New Document: Other` | `6a15dda86b5a32726f4b0bf1` (Other) | `67fe9b98a5133e710d95527e` | `false` | `2026-05-26T22:44:50.561Z` |
| `6a162264d11fe1b093dc48fe` | `New Document: Referral Copy` | `6a15dda76b5a32726f4b0be1` (Referral Copy) | `67fe9b98a5133e710d95527e` | `false` | `2026-05-26T22:44:52.355Z` |

Both notifications target the assignee. Created within ~2 seconds of order creation — the auto-attach propagation fired with `assignedTo` populated.

**NotificationReads: 0** for these 2 notifications (the assignee has not opened the order yet, and the docs were just attached).

**Audit log (`auditLogs` for `6a162261d11fe1b093dc48f4`): 3 entries**

| # | `timestamp` (UTC) | `action` | `source` | `actorId` | `changes[0].field` | `changes[0].from` → `changes[0].to` |
|---|---|---|---|---|---|---|
| 1 | `2026-05-26T22:44:50.440Z` | `create` | `user` | `695c25a530f7a75a8b1ed8c8` | — | (order create event) |
| 2 | `2026-05-26T22:44:50.615Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda86b5a32726f4b0bf1", category: "Other", source: "upload" }` |
| 3 | `2026-05-26T22:44:52.415Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda76b5a32726f4b0be1", category: "Referral Copy", source: "upload" }` |

The audit shape on 05WM matches 05WF's Snapshot 1 exactly (1 create + 2 system updates for auto-attached non-Order docs). Order-history "Change" column should render entries 2 and 3 as `New Document received (Document type = Other)` and `New Document received (Document type = Referral Copy)` (legacy format — unchanged by the fix).

**Pre-link badge counter**: `2` on the orders-tracker table row + Documents tab badge.

### Snapshot 2 — 2026-05-26T22:50:33 UTC (after linking `6a15dda96b5a32726f4b0bf8` to `05WM` — `handleOrderLink` ran)

**Action performed**: Patient Card → Documents tab → linked the Order-category doc `6a15dda96b5a32726f4b0bf8` to order `05WM`. The doc was already linked to `05WF` from a prior test session.

**Documents (`linkedOrderIds` in `documents` collection):**

| Document `_id` | `category` | `linkedOrderIds` |
|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | `[]` |
| `6a15dda86b5a32726f4b0bf1` | Other | `[]` |
| `6a15dda96b5a32726f4b0bf8` | **Order** | **`["6a15df236b5a32726f4b0fd7", "6a162261d11fe1b093dc48f4"]`** |

**Notifications: 3 (+1 vs Snapshot 1)**

| `_id` | `title` | `entities` (doc) | `targetUserId` | `isRead` | `createdAt` |
|---|---|---|---|---|---|
| `6a162262d11fe1b093dc48fb` | `New Document: Other` | `6a15dda86b5a32726f4b0bf1` (Other) | `67fe9b98a5133e710d95527e` | `false` | `2026-05-26T22:44:50.561Z` |
| `6a162264d11fe1b093dc48fe` | `New Document: Referral Copy` | `6a15dda76b5a32726f4b0be1` (Referral Copy) | `67fe9b98a5133e710d95527e` | `false` | `2026-05-26T22:44:52.355Z` |
| **`6a1623b9d11fe1b093dc4934`** | **`New Document: Order`** | **`6a15dda96b5a32726f4b0bf8` (Order)** | `67fe9b98a5133e710d95527e` | `false` | **`2026-05-26T22:50:33.475Z`** |

**Audit log (`auditLogs` for `6a162261d11fe1b093dc48f4`): 4 entries (+1 vs Snapshot 1)**

| # | `timestamp` (UTC) | `action` | `source` | `actorId` | `changes[0].field` | `changes[0].from` → `changes[0].to` |
|---|---|---|---|---|---|---|
| 1 | `2026-05-26T22:44:50.440Z` | `create` | `user` | `695c25a530f7a75a8b1ed8c8` (String) | — | (order create event) |
| 2 | `2026-05-26T22:44:50.615Z` | `update` | `system` | `""` | `documents` | `null` → `{ _id: "6a15dda86b5a32726f4b0bf1", category: "Other", source: "upload" }` |
| 3 | `2026-05-26T22:44:52.415Z` | `update` | `system` | `""` | `documents` | `null` → `{ _id: "6a15dda76b5a32726f4b0be1", category: "Referral Copy", source: "upload" }` |
| **4** | **`2026-05-26T22:50:33.529Z`** | **`update`** | **`user`** | **`695c25a530f7a75a8b1ed8c8` (ObjectId)** | **`documents`** | **`null` → `{ _id: "6a15dda96b5a32726f4b0bf8", category: "Order", source: "upload" }`** |

> The new audit entry mirrors the equivalent row written for `05WF` at `18:52:13.967Z` — same shape, same `actorId` (the user performing the link), same `source: user`. Type inconsistency observed earlier still holds: `create` rows store `actorId` as String, `update` rows as ObjectId.

**No-duplicate verification on `05WF` (`6a15df236b5a32726f4b0fd7`)**

`handleOrderLink` should only fire for the *newly-added* delta (`05WM` here). The pre-existing link to `05WF` must not be re-triggered. Confirmed:

| Surface | Count before linking to 05WM | Count after linking to 05WM | Δ |
|---|:--:|:--:|---|
| `auditLogs` for `05WF` | 4 | 4 | **0** (no duplicate; last entry still `2026-05-26T18:52:13.967Z`) |
| Notifications for `05WF` | 3 | 3 | **0** (no duplicate; latest still `6a15ebdd49e2fad58ff868b8` at `2026-05-26T18:52:13.959Z`) |

This is the add-delta correctness check that the route handler enforces: `newlyAddedOrderIds = sanitizedIds.filter(id => !previousLinkedOrderIds.includes(id))`. Only `05WM` was in the newly-added set; `05WF` (already in `previousLinkedOrderIds`) was excluded.

### Snapshot 3 — 2026-05-26T22:57:48 + 22:58:10 UTC (after UNLINKING from `05WF` AND `05WM` — `handleOrderUnlink` ran twice — the fix in action)

**Action performed**: two separate unlink events from Patient Card → Documents tab. Order of operations:

1. `22:57:48.201Z` — unlinked `6a15dda96b5a32726f4b0bf8` from `05WF` (accidental, but useful — gives a parallel test on the carry-over order).
2. `22:58:10.722Z` — unlinked `6a15dda96b5a32726f4b0bf8` from `05WM` (the intentional fix-verification unlink).

Both went through the link-orders route's removed-delta path, each triggering `handleOrderUnlink` for a single orderId.

**Document state — fully unlinked from both orders**

| Document `_id` | `category` | `linkedOrderIds` |
|---|---|---|
| `6a15dda96b5a32726f4b0bf8` | Order | **`[]`** (was `["6a15df236b5a32726f4b0fd7", "6a162261d11fe1b093dc48f4"]`) |

**05WM — final state**

| Surface | Before (Snapshot 2) | After (Snapshot 3) | Δ |
|---|:--:|:--:|---|
| Notifications | 3 | **2** | -1 (deleted: `6a1623b9d11fe1b093dc4934`) |
| AuditLogs | 4 | **5** | +1 (new remove-side row) |
| NotificationReads orphans | 0 | 0 | no orphans |

**New 05WM remove-side audit entry**:

| `_id` | `timestamp` | `action` | `source` | `actorId` | `changes[0]` |
|---|---|---|---|---|---|
| `6a162582d11fe1b093dc4995` | `2026-05-26T22:58:10.722Z` | `update` | `user` | `695c25a530f7a75a8b1ed8c8` (ObjectId) | `{ field: 'documents', from: { _id: '6a15dda96b5a32726f4b0bf8', category: 'Order', source: 'upload' }, to: null }` |

**05WF — same shape, no cross-contamination**

| Surface | Pre-unlink | Post-unlink | Δ |
|---|:--:|:--:|---|
| Notifications | 3 | **2** | -1 (deleted: `6a15ebdd49e2fad58ff868b8`) |
| AuditLogs | 4 | **5** | +1 (new remove-side row) |

**New 05WF remove-side audit entry**:

| `_id` | `timestamp` | `action` | `source` | `actorId` | `changes[0]` |
|---|---|---|---|---|---|
| `6a16256cd11fe1b093dc4984` | `2026-05-26T22:57:48.201Z` | `update` | `user` | `695c25a530f7a75a8b1ed8c8` (ObjectId) | `{ field: 'documents', from: { _id: '6a15dda96b5a32726f4b0bf8', category: 'Order', source: 'upload' }, to: null }` |

**Acceptance criteria verification**

- [x] Unlinking decreases the badge count by exactly the number of unread notifications for that `(order, document)` pair. (Both orders: -1 notification each → -1 badge each.)
- [x] After unlinking, the "New" chip and "Mark as Read" button disappear for that document (UI-verified by user).
- [x] On unlink, the matching notification row(s) are deleted from `notifications`. (Verified: `6a1623b9d11fe1b093dc4934` and `6a15ebdd49e2fad58ff868b8` both deleted.)
- [x] Associated `notificationreads` rows (for any user) are also deleted. (Verified: 0 orphans referencing the deleted notification IDs.)
- [x] Unlinking writes a new `auditLogs` entry; "Change" column renders `A Document has been removed (Document type = Order)`. (Verified: both orders gained a remove-side row with the correct `from/to` shape; UI rendering to be visually confirmed on the order-history page.)
- [x] No-duplicate / no-cross-contamination: the 2 unlink calls (one per order) wrote one row each, not multiple per order; the 2 non-Order auto-attach notifications survived on each order (unaffected).
- [x] Re-linking creates fresh records — covered by Snapshot 4 below if you choose to test.

**Per-orderId loop correctness**

The fix processes `removedOrderIds` sequentially via `handleOrderUnlink`'s `for...of` loop. Both unlinks happened as separate route invocations (each with `removedOrderIds.length === 1`), so the loop body ran once per invocation. The 22-second gap between the two events shows clearly in the audit timestamps — no interleaving, no race.

### Snapshot 4 — TBD (after RE-LINKING — fresh notification + fresh audit entry)

> Run after linking `6a15dda96b5a32726f4b0bf8` to `05WM` again.

Expected diffs vs Snapshot 3:

- A **fresh** `(order × document)` notification row appears in `db.notifications` (different `_id` than the one deleted in Snapshot 3).
- A **fresh** `A Document has been added (Document type = Order)` audit entry appears.
- UI: badge `2 → 3`; "New" chip + "Mark as Read" button visible for the Order-category doc row.

_<observations>_

---

