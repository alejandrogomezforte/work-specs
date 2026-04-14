# MLID-2011 — Test Data for Document Notifications

## Purpose

Insert demo notification data into MongoDB so the Order Tracker notification badges can be tested in the UI without requiring SignalR to be fully set up in the cloud. The badges read from `notifications` + `notificationreads` collections — if data exists there, the UI will show it.

## Prerequisites

- VPN enabled (to reach staging MongoDB)
- Connection string from `apps/web/.env.local` → `MONGODB_URI`
- MongoDB MCP server configured, OR `mongosh` available

---

## Step 1 — Find 5 New Orders with Assigned Users

Query the `neworders` collection for orders that have `assignedTo` set (so there's a user to target the notification at).

```javascript
db.neworders.find(
  {
    assignedTo: { $exists: true, $nin: [null, ''] },
    'patient.we_infuse_id_IV': { $exists: true, $ne: null },
    deleted: { $ne: true }
  },
  {
    _id: 1,
    displayId: 1,
    assignedTo: 1,
    'patient.we_infuse_id_IV': 1,
    'patient.full_name': 1,
    'patient.name': 1,
    status: 1
  }
).limit(5)
```

**Save**: `_id`, `displayId`, `assignedTo`, `patient.we_infuse_id_IV` for each order.

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

### Orders

| # | _id | displayId | assignedTo | patient.we_infuse_id_IV | patient name |
|---|-----|-----------|------------|------------------------|--------------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |

### Patients

| # | _id (MongoDB) | we_infuse_id_IV | name |
|---|---------------|-----------------|------|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |

### Documents

| # | _id | patientId | category | for order(s) |
|---|-----|-----------|----------|-------------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |

### Users (log in as)

| # | _id | name | email | role |
|---|-----|------|-------|------|
| 1 | | | | |
| 2 | | | | |

### Notifications Inserted

| # | notification _id | targetUserId | order _id | document _id |
|---|-----------------|--------------|-----------|-------------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |
