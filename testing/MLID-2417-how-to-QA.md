# MLID-2417 — Document Notifications V2 (Maintenance Orders) — How to QA

This is the master QA guide for epic **MLID-2417 — Orders: Document Notifications
V2 (Maintenance & Order Linking)**. It covers the maintenance-orders feature work
only, deliverables **D1 through D5**. (D0 was an independent set of new-orders
hotfixes that shipped earlier off `develop`; it is out of scope here.)

The epic ports the document-notification + SignalR machinery that already exists
for **new orders** so it now also works for **maintenance orders**. Functionally,
every notification affordance you already know from new orders (the per-row badge
on the orders table, the Documents-tab badge, the per-document "New" chip, the
"Mark as read" / "Mark all as read" buttons, and the History-tab document
entries) must now behave identically on maintenance orders, and the back-end
triggers (upload, e-order/fax, link, unlink, reassignment) must fan out to
maintenance orders too.

QA chooses the concrete data (which patient, which maintenance order, which
documents). This guide only describes **where to click, which route to visit, and
what to observe** to prove each feature works.

---

## How the deliverables map to what you test

| Deliverable | Ticket | What to prove | Type |
|---|---|---|---|
| D1 | MLID-2447 | Unread badge per row on the maintenance orders table | UI |
| D2 | MLID-2448 | Maintenance order detail **Documents** tab: tab badge, "New" chip, "Mark as read", "Mark all as read" | UI |
| D3 | MLID-2449 | Maintenance order detail **History** tab: a log entry per document | UI |
| D4-T1 | MLID-2450 | Intake-review upload propagates a notification to maintenance orders | Back end |
| D4-T2 | MLID-2451 | E-order / fax upload (background job) propagates to maintenance orders | Back end |
| D4-T3 | MLID-2452 | Linking a document to a maintenance order propagates | Back end |
| D4-T4 | MLID-2453 | Unlinking a document from a maintenance order deletes the notification | Back end |
| D4-T5 | MLID-2454 | Reassigning a maintenance order retargets notifications to the new assignee | Back end |
| D5-T1 | MLID-2628 | The link request now carries the order `category` end-to-end | UI / network |

The D4 back-end triggers have **no UI of their own** — you verify them through the
D1/D2/D3 surfaces they feed (the table badge, the Documents tab, the History tab).
So a natural QA order is: confirm a trigger fires (D4), then watch the badge / chip
/ history update (D1/D2/D3).

---

## Global prerequisites (read this first)

1. **Environment.** All of D1–D5 is merged to `develop` and deployed to **stage**.
   Run QA against the stage app.

2. **Feature flag must be ON.** Go to **`/admin/feature-flags`** and turn
   **`ORDER_DOCUMENT_NOTIFICATIONS`** **ON**. This single flag gates every
   surface and every trigger in this epic. With it OFF, no badge / chip / history
   document-entry / notification is produced — that is the expected "flag OFF"
   behavior, not a bug.
   - For the **D5 intake-submission path only**, also turn **`ORDER_LINK_INTAKE`**
     ON (it gates the order-type selection inside the intake review wizard).

3. **You need TWO user accounts.** Most checks require the person performing the
   action (uploader / linker / reassigner) to be a **different** user than the
   maintenance order's **assignee**. See the "self-action suppression rule" below.
   Pick one account to own (be assigned) the test order, and a second account to
   perform the actions.

4. **Live updates use SignalR.** Badges, chips, and counts update **without a page
   reload**. When a test says "live", keep the page open and watch it change. If
   something does not update live, a hard refresh should still show the correct
   final state — note the difference if you see it.

5. **Maintenance orders need an assignee.** A notification is only written when the
   order has a non-empty `assignedTo`. Use a maintenance order that is assigned to
   your first account. Orders with no assignee are skipped by design.

### Two rules that will otherwise look like bugs

- **Self-action suppression (rule from MLID-2434).** If the person who performs
  the action (uploads, links, or reassigns) **is** the order's assignee, the
  in-app **notification is intentionally suppressed** — no badge, no "New" chip —
  even though the audit-log entry and the live SignalR event still fire. This is
  product-intended (you are not notified about something you did yourself).
  **To see a notification appear, perform the action as the second account and
  check the result while logged in as the assignee.**

- **Order-category documents only notify when linked.** A document whose category
  is **"Order"** does **not** produce a notification at upload time — it only
  notifies once it is explicitly **linked** to an order (that is the D4-T3 case).
  Documents of every other category (Labs, Other, Referral, etc.) notify on upload
  unconditionally. Keep this in mind when choosing a test document.

### Routes you will use

- Maintenance orders **table**: **`/orders-tracker/maintenance`** (or open
  `/orders-tracker` and click the **Maintenance** tab).
- Maintenance order **Documents** tab:
  **`/orders-tracker/maintenance/<orderId>/documents`**
- Maintenance order **History** tab:
  **`/orders-tracker/maintenance/<orderId>/order-history`**
- The new-orders equivalents (`/orders-tracker/new/...`) are used only for the
  regression checks ("new orders still behave as before").

---

## D1 — Maintenance orders table unread badge (MLID-2447)

**Goal:** each row on the maintenance orders table shows a document-notification
badge (a small document icon with an unread count), exactly like the new-orders
table. The badge appears only when the order has at least one unread notification.

**Steps:**

1. Make sure your test maintenance order has at least one unread notification.
   The cleanest way is to trigger one through D4 (for example, upload a non-Order
   document for the order's patient as the second account — see D4-T1). If QA has
   DB access, a notification row on the order also works.
2. Go to **`/orders-tracker/maintenance`** (or the **Maintenance** tab on
   `/orders-tracker`).
3. Find your order's row and look at the **Order ID** cell.

**What to observe:**

- A badge (document icon + number) renders next to the Order ID, with a count
  equal to the order's unread notifications.
- The badge styling matches the new-orders table badge.
- Clicking the badge / Order ID navigates to
  `/orders-tracker/maintenance/<orderId>/documents`.
- The badge updates **live**: when a new notification is created for that order,
  the count rises without reload; when notifications are marked read, it drops.
- Badge visibility is **global** — every authenticated user sees the same count
  for the order, regardless of who it is assigned to. (The count is not per-user;
  the per-user behavior is the "New" chip on the Documents tab, in D2.)

**Flag-OFF check:** with `ORDER_DOCUMENT_NOTIFICATIONS` OFF, no badge renders on
any row.

**Regression:** the `/orders-tracker/new` table badge still behaves exactly as
before.

---

## D2 — Maintenance order Documents tab (MLID-2448)

**Goal:** the **Documents** tab of a maintenance order detail page shows a tab
badge with the unread count, a **"New"** chip on each document the current user
has an unread notification for, a per-document **"Mark as read"** button, and a
**"Mark all as read"** button.

**Steps:**

1. Ensure the order has at least one unread notification **targeting your first
   account** (create it as the second account so it is not self-suppressed — see
   D4-T1 or D4-T3).
2. Log in as the **first account (the assignee)**.
3. Go to **`/orders-tracker/maintenance/<orderId>/documents`**.

**What to observe:**

- The **"Documents" tab label** shows a badge with the order's unread count in its
  top-right corner.
- Each document row that has an unread notification **for you** shows a **"New"**
  chip and a **"Mark as read"** button.
  - "For you" is strict: the chip shows only when the notification's target user is
    you. A different user, even the order's assignee viewing someone else's
    notification, does not see a chip for it.
- A **"Mark all as read"** button sits at the top of the documents table.
- Click a single **"Mark as read"**: that row's chip and button disappear
  immediately, and the tab badge (and the table badge from D1) drop by one — live.
- Click **"Mark all as read"**: every chip / button on the page clears and the tab
  badge goes to zero — live.
- Counts and chips also update **live** when documents are added, unlinked, marked
  read elsewhere, or the order is reassigned (SignalR events
  `documentAdded`, `documentUnlinked`, `notification:read`, `order:reassigned`).

**Second-account check:** logged in as the **second account** (not the
notification's target), open the same Documents tab. The **tab badge count is the
same** (it is global), but you see **no "New" chips and no "Mark as read"
buttons** — those are per-user.

**Flag-OFF check:** with the flag OFF, no tab badge, no chip, and no mark-as-read
controls render.

**Regression:** the `/orders-tracker/new/<id>/documents` tab is unchanged.

---

## D3 — Maintenance order History tab (MLID-2449)

**Goal:** the **History** tab of a maintenance order shows a log entry for each
document propagated or linked to that order, with the same layout, sorting, and
"Change" text as the new-orders History tab.

**Steps:**

1. Produce some document history on the order by running D4 triggers: upload a
   non-Order document for the patient (D4-T1), and link / unlink an Order document
   (D4-T3 / D4-T4). Each of these writes an audit entry.
2. Go to **`/orders-tracker/maintenance/<orderId>/order-history`**.

**What to observe (the "Change" column text):**

- A non-Order document (Labs, Other, etc.) added →
  **"New Document received (Document type = <category>)"**.
- An **Order**-category document **linked** →
  **"A Document has been added (Document type = Order)"**.
- An **Order**-category document **unlinked** →
  **"A Document has been removed (Document type = Order)"**.
- The **Field** column reads **"New Document"**, the **Category** column reads
  **"Documents"**.
- The **User** column resolves to the actor's name for user actions, and to
  **"System"** for system-generated rows.
- Rows are sorted newest-first; the Date/Time header toggles the sort; pagination
  matches the new-orders History tab.
- Order-category documents appear **only when they were actually linked** to this
  order (consistent with the notification eligibility rule). Non-Order documents
  appear unconditionally.

**Flag note:** the History tab renders the audit log **regardless** of the feature
flag (the flag gates notifications and the Documents-tab controls, not the audit
log itself). Toggling the flag OFF should **not** remove the History entries.

**Regression:** the `/orders-tracker/new/<id>/order-history` tab is unchanged.

---

## D4 — Back-end triggers (verified through the D1/D2/D3 surfaces)

All five D4 tasks share the same proof pattern: perform the trigger as the second
account, then confirm the notification appears on the assignee's surfaces (table
badge, Documents tab chip/badge, History entry). For each, also keep DevTools →
Network open if you want to watch the API call.

### D4-T1 — Intake-review upload (MLID-2450)

**Trigger:** uploading a document through the intake-review wizard (the split-pdf
flow) must fan out a notification to **every maintenance order** of the matched
patient (not just new orders).

**Steps:**

1. Have a maintenance order assigned to the **first account**, on a patient you
   can match in an intake.
2. Log in as the **second account**. Go to **`/intakes`**, open an intake of type
   **`spruceFax`** or **`eOrder`** that is matched (or that you match) to that
   patient, and click **"Review Documents"** → `/intakes/<id>/review`.
3. Walk the 3-step wizard: assign a page range to a **non-Order** section (for
   example "Labs", type "Lab Results"), fill the Review-Data fields, click
   **"Complete Intake"**, then **"Submit Intake"** and **"Done"**. The
   `POST /api/documents/split-pdf` call fires on submit.

**What to observe (log in as the assignee / first account):**

- The maintenance order's **table badge** (D1) and **Documents-tab badge + "New"
  chip** (D2) appear for the uploaded document.
- A **History** entry (D3) reads "New Document received (Document type = Lab
  Results)".
- An **Order**-category section uploaded the same way produces **no** notification
  (eligibility gate — it is not linked yet). Other categories do notify.

**Negative / regression:**

- An unassigned maintenance order receives nothing.
- A **new** order of the same patient still receives its notification (the
  existing path is unchanged).
- Flag OFF → nothing propagates.

### D4-T2 — E-order / fax upload via background job (MLID-2451)

**Trigger:** the same propagation, but driven by the e-order / fax background job
(`weinfuseUploader`) instead of the interactive intake-review flow. It routes
through the same code as D4-T1, so the observable result is identical.

**Steps:**

1. Cause an e-order or fax document to be uploaded for a patient who has an
   assigned maintenance order (through whatever stage mechanism feeds the
   e-order/fax job — the background job processes it).
2. Wait for the job to process the document.

**What to observe:**

- The maintenance order shows the same badge / chip / History entry as D4-T1.
- The notification's message reflects the source (E-Order or Fax).
- Note: on the e-order/fax path the uploader is the system (no human actor), so
  the self-action suppression rule never applies here — the notification always
  lands.
- Regression: new orders on the same patient still receive their notification.

### D4-T3 — Link a document to a maintenance order (MLID-2452)

**Trigger:** linking a document to a maintenance order propagates a notification
(plus audit entry + live event) to that order.

**Steps:**

1. Log in as the **second account**.
2. Open the patient's **Documents** area and use the **"Link to Order"** action on
   an **Order-category** document (the link route only accepts Order documents).
3. In the Link Orders modal, check the **maintenance order**, then **Update**.

**What to observe (log in as the assignee):**

- The maintenance order's table badge / Documents-tab badge + "New" chip appear
  for the linked document — live.
- The **History** tab shows "A Document has been added (Document type = Order)".
- Self-action check: if you link as the **assignee**, no notification appears (the
  audit entry still does) — that is the suppression rule, not a failure.
- Unassigned order → nothing. Flag OFF → nothing. New orders unchanged.

### D4-T4 — Unlink a document from a maintenance order (MLID-2453)

**Trigger:** unlinking a document from a maintenance order must **delete** the
matching notification(s) for that (order, document) pair and update the surfaces.

**Steps:**

1. Start from a document already linked to the maintenance order with a live
   notification (from D4-T3).
2. **Unlink** it — open the Link Orders modal again, uncheck the maintenance order,
   and Update (the same `PUT /api/documents/<documentId>/link-orders` route, now
   with the order removed from the set).

**What to observe:**

- The order's **table badge** and **Documents-tab badge** drop by the number of
  unread notifications for that document; the document's **"New" chip** and
  **"Mark as read"** button disappear — live (SignalR `documentUnlinked`).
- The **History** tab gains a "A Document has been removed (Document type = Order)"
  entry.
- Re-linking the same document creates a fresh notification and a fresh "added"
  history entry (symmetric round-trip).

### D4-T5 — Reassign a maintenance order (MLID-2454)

**Trigger:** reassigning a maintenance order to a different user must **retarget**
the order's existing notifications from the previous assignee to the new one, and
broadcast a live refresh.

**Steps:**

1. Start with a maintenance order assigned to the **first account** that has one or
   more unread notifications (chips visible to that account on the Documents tab).
2. Reassign the order to the **second account** (through the order's assignment
   control / the maintenance order edit flow that calls
   `PUT /api/orders/maintenance`).

**What to observe:**

- **Previous assignee (first account):** their "New" chips and "Mark as read"
  buttons for this order disappear (the notifications no longer target them) —
  live.
- **New assignee (second account):** they now see every affected document as
  **unread** — "New" chips and "Mark as read" appear — **even documents the
  previous assignee had already marked read** (the new assignee starts fresh).
- The order-table badge count for the order is unchanged in total (it is global);
  only *who* the per-row controls belong to changes.
- The HTTP response for the reassignment is not blocked by the cascade (it is
  fire-and-forget); the order saves normally even if the cascade had an issue.
- Partner orders (the related `-1` suffix orders) are retargeted too.
- Regression: new-order reassignment (the D0 behavior) is unchanged. Flag OFF → no
  cascade.

---

## D5 — Link-orders category pass-through (MLID-2628)

**Goal:** this is a refactor — the user-visible behavior is the **same as before**.
The proof is that linking a **maintenance** order still works *and* that the link
request now carries the order **category** in its payload (so the server resolves
the order in one lookup instead of guessing across collections).

**The key thing to verify is the request shape.** Open **DevTools → Network**
before saving and filter on `link-orders`.

**Steps (sender 1 — patient Documents tab modal):**

1. Log in as the **second account** (linker ≠ assignee, so you also see the
   notification land on the assignee afterward).
2. On a patient with a maintenance order, open the **Documents** tab, choose an
   **Order-category** document, and use **"Link to Order"**.
3. With Network open, check the **maintenance** order and click **Update**.

**What to observe:**

- The request is **`PUT /api/documents/<documentId>/link-orders`** and its
  **payload** now has the new shape, including the category:

  ```json
  { "linkedOrders": [ { "id": "<orderId>", "category": "maintenance" } ] }
  ```

  (The old shape was `{ "linkedOrderIds": ["<orderId>"] }` — it must no longer be
  sent.)
- Response is **200**, and the link propagates exactly like D4-T3 (badge / "New"
  chip appear for the assignee).
- Linking a **new** order sends `"category": "new"` and behaves as before.

**Steps (sender 2 — document-intake submission, optional, needs
`ORDER_LINK_INTAKE` ON):**

1. From `/intakes`, open a `spruceFax` / `eOrder` intake → **"Review Documents"**.
2. In the review wizard, on the order tab, in the **"Order Type & Review"** card,
   select the **"Update Order"** pill and pick an existing **maintenance** order.
3. Complete and submit the intake; in Network, find the
   `PUT /api/documents/<documentId>/link-orders` call fired by the
   "Link Document to LISA Order" step.

**What to observe:**

- The payload carries `{ "linkedOrders": [ { "id": "<orderId>", "category":
  "maintenance" } ] }` (the category comes from the order you selected).
- The "Create" variants send the matching category as well.

**Negative cases (optional — also covered by automated tests):**

- A body using the legacy `{ "linkedOrderIds": [...] }` shape → **400**.
- A `linkedOrders` entry with an invalid 24-char hex `id` → **400**.
- A `category` other than `"new"` / `"maintenance"` → **400**.
- A non-Order document → **422** (unchanged).

---

## Quick regression checklist (new orders must be unaffected)

The whole epic extends shared code paths, so after testing maintenance, confirm
the new-orders surfaces still behave exactly as before:

- `/orders-tracker/new` table badge renders and updates.
- `/orders-tracker/new/<id>/documents` tab badge, "New" chip, "Mark as read",
  "Mark all as read" all work.
- `/orders-tracker/new/<id>/order-history` document entries render with the same
  "Change" text.
- New-order upload, link, unlink, and reassignment all still notify correctly.

---

## What "pass" looks like for the epic

- Every maintenance-order surface (table badge, Documents tab, History tab) behaves
  identically to its new-orders counterpart.
- Each D4 trigger (upload, e-order/fax, link, unlink, reassign) produces the
  correct notification effect on those surfaces, live.
- The D5 link request carries the order category in its payload.
- The feature flag cleanly gates everything (OFF = silent, no errors).
- New-orders behavior is unchanged.
- No errors in the browser console or server logs during any of the flows.
