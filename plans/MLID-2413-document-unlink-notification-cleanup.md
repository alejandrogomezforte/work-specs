# MLID-2413 — Order notification badge: counter mismatch on document unlink

## Task Reference

- **Jira**: [MLID-2413](https://localinfusion.atlassian.net/browse/MLID-2413)
- **Story Points**: 4
- **Priority**: High
- **Sprint**: 28 (active)
- **Branch**: `fix/MLID-2413-document-unlink-notification-cleanup` (2 commits — feat + chore/quality)
- **Base Branch**: `develop` (D0 tickets branch off develop directly, not the epic branch)
- **Status**: **Done** — implementation complete, quality gates passed, local UI verification confirmed, PR doc written to `docs/agomez/PR/MLID-2413.md`. Ready to push and open the PR.
- **Parent Epic**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417) — D0-T2 in that epic's plan

---

## Summary

When a `category: "Order"` document is unlinked from a new order via `PUT /api/documents/[documentId]/link-orders`, the route only fires propagation for the *added* delta of order IDs. Removed order IDs leave their `notifications` rows orphaned — still counted by badge queries, still rendered as "New" chips — and no `notificationreads` cleanup runs. As a result, the badge count on the Documents tab and the orders-tracker table row does not decrease after an unlink. Additionally, no `auditLogs` entry is written for the remove side, so the order-history "Change" column shows no record of the unlink.

This fix introduces a `handleOrderUnlink(documentId, removedOrderIds, { actorId })` orchestrator (modelled after `handleOrderReassignment`) that performs four steps for each removed `(orderId, documentId)` pair in sequence: delete matching `notifications` rows, delete the associated `notificationreads` rows, write a remove-side audit-log entry on the order, and broadcast `documentUnlinked { orderId, documentId }` via SignalR so open clients refetch badges and clear "New" chips. The `auditLog.config.ts` `documents.renderChange` function is also updated to produce the PO-specified "Change" column text for `category: "Order"` documents only — add-side renders `A Document has been added (Document type = Order)` and the new remove-side renders `A Document has been removed (Document type = Order)`. **Documents of every other category continue to render the existing `New Document received (Document type = <category>)` text — the renderer change is Order-category-scoped, not global.** D4-T4 (MLID-2453) reuses `handleOrderUnlink` for maintenance orders without modification — the helper is collection-agnostic.

---

## Codebase Analysis

### `apps/web/app/api/documents/[documentId]/link-orders/route.ts`

Current state (lines confirmed by read):
- Line 75: `previousLinkedOrderIds` snapshot — `doc.linkedOrderIds ?? []`
- Line 85-87: `newlyAddedOrderIds` computed as `sanitizedIds.filter(id => !previousLinkedOrderIds.includes(id))`
- Line 89-103: `handleOrderLink` fire-and-forget for added delta only
- Lines 98-99: `actorId = session.user?.id || ''` (inline, reuse same expression)
- **Insertion point**: between line 87 (end of add-delta block) and line 89 (`if (newlyAddedOrderIds.length > 0)`) — compute `removedOrderIds` here and fire `handleOrderUnlink` fire-and-forget
- `updated` is the return value of `linkDocumentToOrders`; `updated.category` and `updated._id` are available for the unlink helper
- No test file exists today — `route.test.ts` must be created from scratch

### `apps/web/services/mongodb/notification.ts`

Contains all notification service helpers. Last function is `isReadByUser` at line 461. New functions are added adjacent to `reassignUserNotificationsForOrder` (line 262):
- `deleteNotificationsForOrderDocument(orderId, documentId)`: must use `$and` with two `$elemMatch` clauses (single-`$elemMatch` with dot-notation would cross-match array elements — documented in `notification.test.ts:319-342`). Must `find().select('_id').lean()` first to capture IDs, then `deleteMany()` with the same filter. `deleteMany` does not return document IDs.
- `deleteNotificationReadsByNotificationIds(notificationIds: string[])`: `NotificationRead.deleteMany({ notificationId: { $in: notificationIds } })` — strings auto-coerce to ObjectId per existing Mongoose patterns in this file

Current mock setup in `notification.test.ts` (lines 18-38): `Notification` mock lacks `deleteMany`; `NotificationRead` mock lacks `deleteMany`. Both must be added before the new service tests can pass.

### `apps/web/services/audit/documentAudit.ts`

Contains `writeOrderDocumentAuditEntry` (add side, `from: null, to: { _id, category, source }`). The new `writeOrderDocumentUnlinkAuditEntry` is a sibling export. Same `WriteOrderDocumentAuditEntryParams` type can be reused (identical shape — `actorId`, `orderId`, `documentId`, `category`, `documentSource`, `auditSource`). The only structural difference is the `changes` entry: `from: { _id, category, source }`, `to: null`. Decision: create a sibling function (not parameterize the existing one) to keep each function's shape explicit and avoid a boolean discriminator argument. The test file already has a `beforeEach(() => jest.clearAllMocks())` pattern to follow.

### `apps/web/services/mongodb/auditLog.config.ts`

Line 34-41: `documents` field config. Current `renderChange`:
```
(change) => `New Document received (Document type = ${to?.category ?? 'Unknown'})`
```
Must be replaced with a renderer that **scopes the new PO-specified text to `category: "Order"` only**, and leaves every other category on the existing `New Document received (Document type = <category>)` format. Discriminator: read category from `change.from?.category ?? change.to?.category`; if category is `"Order"`, render the add/remove text based on whether `from` or `to` is null; otherwise fall through to the legacy format. The `AuditLogTable.test.tsx:220` test uses `"Lab Results"` (non-Order) — its existing string remains valid and **does not need to be modified**. A new test case for `category === "Order"` is added separately.

### `apps/web/services/notifications/handleOrderReassignment.ts`

Structural template for `handleOrderUnlink`. Identical shape: outer `try/catch`, feature-flag early return, guard clauses, inner `try/catch` per step (delete notifications, delete reads, audit write, broadcast), each step failure logged individually. The retarget step in `handleOrderReassignment` aborts the broadcast if it fails; the same pattern applies: notification-delete failure skips the read-cleanup and broadcast, but audit and broadcast have separate try/catch from each other.

Difference from reassignment: `handleOrderUnlink` receives `documentId` + `removedOrderIds: string[]` and iterates over the array, calling the four-step sequence per `orderId`. The iteration is sequential (not `Promise.all`) to keep error isolation per pair.

### `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts`

Lines 100-123: SignalR event registrations. Currently registers `documentAdded`, `notification:read`, and `order:reassigned`. The new `documentUnlinked` handler mirrors `handleDocumentAdded` — targeted per-`orderId` refetch using `payload?.orderId` with the same `visibleSet.has(orderId)` guard.

### `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts`

Lines 116-124: SignalR event registrations. Currently registers `documentAdded`, `notification:read`, and `order:reassigned`. The new `documentUnlinked` handler is identical to the existing `documentAdded` handler: `connection.on('documentUnlinked', fetchNotifications)`. No payload inspection needed — `fetchNotifications` already re-queries for the current `orderId`.

### Test files for hooks

- `useOrderNotificationCounts.test.tsx` lines 282-411: `documentAdded` handler tests are the pattern. Add parallel block for `documentUnlinked` covering: targeted refetch, merge into existing counts, ignore non-visible orderId, ignore missing orderId in payload.
- `useDocumentsTabNotifications.test.tsx` lines 342-444: `documentAdded` / `notification:read` / `order:reassigned` listener tests are the pattern. Add a `documentUnlinked` block following the same structure.

---

## Implementation Steps

### Step 1 — `deleteNotificationsForOrderDocument` service helper (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/services/mongodb/notification.test.ts` (modify — add `deleteMany` to mocks + new test describe block)
- `apps/web/services/mongodb/notification.ts` (modify — add new exported function)

**RED**: Add `deleteMany: jest.fn()` to the `Notification` mock (line 29 area) and `NotificationRead` mock (line 37 area). Add a new `describe('deleteNotificationsForOrderDocument', ...)` block with tests that will fail because the function does not exist:
- Returns `{ deletedCount: 0, deletedNotificationIds: [] }` when `find` returns empty array
- Returns `{ deletedCount: 2, deletedNotificationIds: ['id-1', 'id-2'] }` when `find` returns two docs
- Passes `$and` of two `$elemMatch` clauses (not dot-notation) to both `find` and `deleteMany`
- Calls `find().select('_id').lean()` first, then `deleteMany` with the same filter
- Throws and logs on `find` error; throws and logs on `deleteMany` error
- Guards: returns empty result when `orderId` or `documentId` is empty string

**GREEN**: Add to `notification.ts` adjacent to `reassignUserNotificationsForOrder`:
```typescript
export async function deleteNotificationsForOrderDocument(
  orderId: string,
  documentId: string
): Promise<{ deletedCount: number; deletedNotificationIds: string[] }> {
  if (!orderId || !documentId) {
    return { deletedCount: 0, deletedNotificationIds: [] };
  }
  const filter = {
    $and: [
      { entities: { $elemMatch: { entityType: 'order', entityId: orderId } } },
      { entities: { $elemMatch: { entityType: 'document', entityId: documentId } } },
    ],
  };
  const found = (await Notification.find(filter).select('_id').lean()) as Array<{ _id: unknown }>;
  const deletedNotificationIds = found.map((doc) => String(doc._id));
  if (deletedNotificationIds.length === 0) {
    return { deletedCount: 0, deletedNotificationIds: [] };
  }
  const result = await Notification.deleteMany(filter);
  return { deletedCount: result.deletedCount ?? 0, deletedNotificationIds };
}
```

**REFACTOR**: Extract the filter into a named variable `orderDocumentEntityFilter` to avoid duplication across the two calls. Add `logger.info` before the delete and `logger.error` on thrown paths, matching the style of `reassignUserNotificationsForOrder`.

**Commit**: `[MLID-2413] - feat(notification): add deleteNotificationsForOrderDocument service helper`

---

### Step 2 — `deleteNotificationReadsByNotificationIds` service helper (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/services/mongodb/notification.test.ts` (modify — add new describe block)
- `apps/web/services/mongodb/notification.ts` (modify — add new exported function)

**RED**: Add a `describe('deleteNotificationReadsByNotificationIds', ...)` block. Tests:
- Returns 0 when `notificationIds` is empty array (guard — must not call `deleteMany`)
- Returns `deletedCount` from the `deleteMany` result on success
- Passes `{ notificationId: { $in: notificationIds } }` as filter to `NotificationRead.deleteMany`
- Throws and logs on `deleteMany` error

**GREEN**: Add to `notification.ts`:
```typescript
export async function deleteNotificationReadsByNotificationIds(
  notificationIds: string[]
): Promise<number> {
  if (notificationIds.length === 0) {
    return 0;
  }
  const result = await NotificationRead.deleteMany({
    notificationId: { $in: notificationIds },
  });
  return result.deletedCount ?? 0;
}
```

**REFACTOR**: Add `logger.info` on success and `logger.error` catch/rethrow, matching existing service patterns.

**Commit**: `[MLID-2413] - feat(notification): add deleteNotificationReadsByNotificationIds service helper`

---

### Step 3 — `writeOrderDocumentUnlinkAuditEntry` audit helper (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/services/audit/documentAudit.test.ts` (modify — add new describe block)
- `apps/web/services/audit/documentAudit.ts` (modify — add new exported function)

**RED**: Add a `describe('writeOrderDocumentUnlinkAuditEntry', ...)` block (after the existing `writeOrderDocumentAuditEntry` describe). Tests — mirror the existing test cases:
- Calls `writeAuditLog` once with `entityType: 'order'`, `entityId: orderId`, `action: 'update'`, `actorId`, `source: auditSource`, and `changes: [{ field: 'documents', from: { _id: documentId, category, source: documentSource }, to: null }]`
- Forwards `documentSource: 'fax'` correctly in `from.source`
- Forwards `documentSource: 'e-order'` correctly in `from.source`
- Logs info on success with `orderId`, `documentId`, `category`
- Swallows error and logs via `logger.error` when `writeAuditLog` rejects (never throws)

Note: `WriteOrderDocumentAuditEntryParams` already covers every required field — reuse the same type, no new type needed.

**GREEN**: Add to `documentAudit.ts` adjacent to `writeOrderDocumentAuditEntry`:
```typescript
export async function writeOrderDocumentUnlinkAuditEntry(
  params: WriteOrderDocumentAuditEntryParams
): Promise<void> {
  const { orderId, documentId, category, documentSource, actorId, auditSource } = params;
  try {
    await writeAuditLog({
      entityType: 'order',
      entityId: orderId,
      action: 'update',
      actorId,
      source: auditSource,
      changes: [
        {
          field: 'documents',
          from: { _id: documentId, category, source: documentSource },
          to: null,
        },
      ],
    });
    logger.info('Document unlink audit entry written', { orderId, documentId, category });
  } catch (error) {
    logger.error('Failed to write document unlink audit entry', { orderId, documentId, error });
  }
}
```

**REFACTOR**: No structural changes needed — the function is already at the right abstraction level.

**Commit**: `[MLID-2413] - feat(audit): add writeOrderDocumentUnlinkAuditEntry helper`

---

### Step 4 — `handleOrderUnlink` orchestrator (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/services/notifications/handleOrderUnlink.test.ts` (create)
- `apps/web/services/notifications/handleOrderUnlink.ts` (create)

**RED**: Create `handleOrderUnlink.test.ts` using `handleOrderReassignment.test.ts` as the structural template. Jest environment directive `@jest-environment node` at top. Mock all four boundary imports: `FeatureFlagService`, `deleteNotificationsForOrderDocument`, `deleteNotificationReadsByNotificationIds`, `writeOrderDocumentUnlinkAuditEntry`, `getSignalRService`, `logger`. Test cases:

- Feature flag off → returns without calling any step
- `removedOrderIds` empty array → returns without calling any step (guard)
- `documentId` empty string → returns without calling any step (guard)
- Happy path (one `orderId`): calls delete-notifications → gets IDs back → calls delete-reads with those IDs → calls audit write with correct params → broadcasts `documentUnlinked { orderId, documentId }` in that order
- Happy path (two `orderIds`): processes each sequentially; both get all four steps
- `deleteNotificationsForOrderDocument` rejects → logs, skips delete-reads and broadcast for that orderId (but continues to next orderId if present)
- `deleteNotificationReadsByNotificationIds` rejects → logs, skips broadcast for that orderId (audit still runs — see design note below)
- `writeOrderDocumentUnlinkAuditEntry` never throws by contract — no failure test needed for this step
- SignalR resolution fails → logs, does not throw; the three DB steps still ran
- `broadcastToAll` rejects → logs, does not throw
- `actorId` is forwarded to `writeOrderDocumentUnlinkAuditEntry` as-is

Design note on step ordering: the four steps are: (1) delete notifications, (2) delete reads, (3) write audit, (4) broadcast. Steps 1-2 can fail independently. If step 1 fails, steps 2-4 are skipped for that orderId to preserve consistency (no orphaned reads without a parent notification). Step 3 (audit) is always attempted regardless of step 2's success — a failed reads-delete should still produce a human-readable history record. Step 4 is attempted after step 3. Each of steps 1, 2, 4 has its own inner `try/catch` matching the `handleOrderReassignment` pattern; step 3 never throws.

**GREEN**: Create `handleOrderUnlink.ts`:

```typescript
import { FeatureFlagService } from '@/services/featureFlags/featureFlagService';
import {
  deleteNotificationReadsByNotificationIds,
  deleteNotificationsForOrderDocument,
} from '@/services/mongodb/notification';
import { writeOrderDocumentUnlinkAuditEntry } from '@/services/audit/documentAudit';
import { getSignalRService } from '@/services/signalr';
import { DocumentAuditSource } from '@/services/audit/documentAudit';
import { FeatureFlag } from '@/utils/featureFlags/featureFlags';
import { logger } from '@/utils/logger';

interface HandleOrderUnlinkOptions {
  actorId: string;
  documentCategory: string;
  documentSource: DocumentAuditSource;
}

export async function handleOrderUnlink(
  documentId: string,
  removedOrderIds: string[],
  options: HandleOrderUnlinkOptions
): Promise<void> {
  try {
    const isEnabled = await FeatureFlagService.getFeatureFlag(
      FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS
    );
    if (!isEnabled) return;
    if (!documentId || removedOrderIds.length === 0) return;

    for (const orderId of removedOrderIds) {
      await processOrderUnlink(documentId, orderId, options);
    }
  } catch (error) {
    logger.error('handleOrderUnlink: unexpected error', {
      error,
      documentId,
      removedOrderIds,
    });
  }
}

async function processOrderUnlink(
  documentId: string,
  orderId: string,
  options: HandleOrderUnlinkOptions
): Promise<void> {
  const { actorId, documentCategory, documentSource } = options;

  let deletedNotificationIds: string[] = [];

  try {
    const result = await deleteNotificationsForOrderDocument(orderId, documentId);
    deletedNotificationIds = result.deletedNotificationIds;
  } catch (error) {
    logger.error(
      'handleOrderUnlink: failed to delete notifications — skipping reads cleanup and broadcast',
      { error, orderId, documentId }
    );
    return;
  }

  try {
    await deleteNotificationReadsByNotificationIds(deletedNotificationIds);
  } catch (error) {
    logger.error(
      'handleOrderUnlink: failed to delete notification reads — continuing to audit and broadcast',
      { error, orderId, documentId }
    );
  }

  await writeOrderDocumentUnlinkAuditEntry({
    orderId,
    documentId,
    category: documentCategory,
    documentSource,
    actorId,
    auditSource: 'user',
  });

  try {
    const signalr = await getSignalRService();
    await signalr.broadcastToAll('documentUnlinked', { orderId, documentId });
  } catch (error) {
    logger.error(
      'handleOrderUnlink: failed to broadcast documentUnlinked — DB cleanup and audit were completed',
      { error, orderId, documentId }
    );
  }
}
```

**REFACTOR**: Extract the inner function `processOrderUnlink` into the file body (already done above). Verify log messages are consistent with `handleOrderReassignment` style.

**Commit**: `[MLID-2413] - feat(notifications): add handleOrderUnlink orchestrator`

---

### Step 5 — Update `auditLog.config.ts` renderer (Order-category scoped) (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/services/mongodb/auditLog.config.test.ts` (create)
- `apps/web/services/mongodb/auditLog.config.ts` (modify — `documents.renderChange`)
- `apps/web/app/orders-tracker/[category]/[orderId]/order-history/components/AuditLogTable.test.tsx` (modify — add a new remove-side test case for Order category; the existing line 220 Lab Results test stays unchanged)

**Scope note**: the new add/remove text is **only** applied when `category === 'Order'`. For every other document category (Referral Copy, Other, Lab Results, etc.), the renderer falls through to the existing `New Document received (Document type = <category>)` format. The existing `AuditLogTable.test.tsx:220` test uses `Lab Results` — that string is preserved as-is.

**RED**: Create `auditLog.config.test.ts`. Test cases for the `documents.renderChange` function (accessed via `getFieldConfig('documents').renderChange`):

Order-category branch (new behavior):
- `change.from === null`, `change.to = { category: 'Order' }` → returns `"A Document has been added (Document type = Order)"`
- `change.to === null`, `change.from = { _id: 'doc-1', category: 'Order', source: 'upload' }` → returns `"A Document has been removed (Document type = Order)"`

Non-Order categories (legacy behavior preserved):
- `change.from === null`, `change.to = { category: 'Lab Results' }` → returns `"New Document received (Document type = Lab Results)"` (unchanged)
- `change.from === null`, `change.to = { category: 'Referral Copy' }` → returns `"New Document received (Document type = Referral Copy)"` (unchanged)
- `change.from === null`, `change.to = { category: 'Other' }` → returns `"New Document received (Document type = Other)"` (unchanged)

Edge cases:
- Both `from` and `to` are null → returns a safe fallback string with `Unknown`
- `to?.category` is missing → falls back to `Unknown`

Also update `AuditLogTable.test.tsx`: **do NOT modify the existing line 220 Lab Results test** — it remains valid because Lab Results is a non-Order category. Add a new test case below it for the Order-category remove side: pass a `change` row with `from = { _id: 'doc-1', category: 'Order', source: 'upload' }, to: null` and verify the rendered string reads `"A Document has been removed (Document type = Order)"`. Optionally also add an Order-category add-side row that asserts the new `"A Document has been added (Document type = Order)"` text.

**GREEN**: Update `auditLog.config.ts` lines 34-41:
```typescript
documents: {
  label: 'New Document',
  category: 'Documents',
  renderChange: (change: AuditLogChange) => {
    const from = change.from as { category?: string } | null;
    const to = change.to as { category?: string } | null;
    const category = from?.category ?? to?.category ?? 'Unknown';

    // PO-specified text — Order category only
    if (category === 'Order') {
      if (change.from === null) {
        return `A Document has been added (Document type = Order)`;
      }
      if (change.to === null) {
        return `A Document has been removed (Document type = Order)`;
      }
    }

    // Legacy format preserved for every other category
    return `New Document received (Document type = ${category})`;
  },
},
```

**REFACTOR**: No structural changes needed. If the `category === 'Order'` branch grows in future tickets, consider extracting it; for now the inline form is more readable.

**Commit**: `[MLID-2413] - fix(audit): order-category renderChange text for add and remove`

---

### Step 6 — Wire `handleOrderUnlink` into `PUT /api/documents/[documentId]/link-orders` + create route test file (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/app/api/documents/[documentId]/link-orders/route.test.ts` (create)
- `apps/web/app/api/documents/[documentId]/link-orders/route.ts` (modify — compute removed delta, call `handleOrderUnlink` fire-and-forget)

**RED**: Create `route.test.ts` from scratch, following the `apps/web/app/api/orders/new/route.test.ts` integration-test pattern. Mock list: `next-auth` (for `getServerSession`), `@/services/mongodb/document` (`getDocumentById`, `linkDocumentToOrders`), `@/services/notifications/documentNotification` (`handleOrderLink`), `@/services/notifications/handleOrderUnlink` (`handleOrderUnlink`), `@/utils/logger`.

Auth / validation test cases:
- No session → 401
- Session with disallowed role → 403
- Malformed JSON body → 400
- Non-ObjectId `documentId` path param → 400
- `linkedOrderIds` not an array → 400
- `linkedOrderIds` containing invalid ObjectId strings → 400
- Document not found (`getDocumentById` returns null) → 404
- Document category is not `"Order"` → 422
- `linkDocumentToOrders` returns null → 404

Add-delta path (linked orders increase):
- `newlyAddedOrderIds.length > 0` → `handleOrderLink` called fire-and-forget; response is 200 with updated doc
- `handleOrderLink` throws → response is still 200 (fire-and-forget swallows)

Remove-delta path (core of this fix):
- `removedOrderIds.length > 0` → `handleOrderUnlink` called with `(documentId, removedOrderIds, { actorId, documentCategory: updated.category, documentSource: 'upload' })` fire-and-forget
- `handleOrderUnlink` throws → response is still 200 (fire-and-forget swallows)
- No overlap between added and removed deltas — verify each only receives its own IDs

No-change case:
- Payload equals current `linkedOrderIds` exactly → neither helper is called; response is 200

Internal error:
- `getDocumentById` throws → 500

**GREEN**: Modify `route.ts` — insert between line 87 and 89:
```typescript
const removedOrderIds = previousLinkedOrderIds.filter(
  (id) => !sanitizedIds.includes(id)
);

if (removedOrderIds.length > 0) {
  handleOrderUnlink(documentId, removedOrderIds, {
    actorId: session.user?.id || '',
    documentCategory: updated.category,
    documentSource: 'upload',
  }).catch((error) => {
    logger.error('Unlink cleanup failed', { documentId, error });
  });
}
```

Add the import: `import { handleOrderUnlink } from '@/services/notifications/handleOrderUnlink';`

Note: `documentSource` is hardcoded to `'upload'` here because the link-orders route is always a user-driven action from the UI (never a fax or e-order path). D4-T4 will inherit the same value.

**REFACTOR**: No structural changes needed. Verify the existing `handleOrderLink` block is unmodified.

**Commit**: `[MLID-2413] - fix(link-orders): delete orphaned notifications on document unlink`

---

### Step 7 — `useDocumentsTabNotifications` `documentUnlinked` SignalR listener (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx` (modify — add `documentUnlinked` test cases)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` (modify — register `documentUnlinked`)

**RED**: Add a `'documentUnlinked SignalR handler'` test block after the `order:reassigned` block (around line 445). Test structure mirrors the `documentAdded` test at lines 342-373:
- Verifies `connection.on` is called with `'documentUnlinked'` during mount
- When the `documentUnlinked` handler fires, `fetchNotifications` is called (verified via `fetch` being called once more for `by-order/order-1`)
- `connection.off` is called with `'documentUnlinked'` on unmount (cleanup)

**GREEN**: In `useDocumentsTabNotifications.ts`, in the SignalR `useEffect` (lines 111-125), add:
```typescript
connection.on('documentUnlinked', fetchNotifications);
```
and in the cleanup return:
```typescript
connection.off('documentUnlinked', fetchNotifications);
```

**REFACTOR**: No structural changes needed. The `fetchNotifications` callback is already stable (wrapped in `useCallback`).

**Commit**: `[MLID-2413] - feat(useDocumentsTabNotifications): refetch on documentUnlinked SignalR event`

---

### Step 8 — `useOrderNotificationCounts` `documentUnlinked` SignalR listener (RED → GREEN → REFACTOR)

**Files**:
- `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.test.tsx` (modify — add `documentUnlinked` test cases)
- `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` (modify — register `documentUnlinked`)

**RED**: Add a `'documentUnlinked SignalR handler'` test block after the `documentAdded` block (around line 412). Tests mirror the `documentAdded` pattern (lines 282-411):
- Targeted refetch for only the event's `orderId` (not the full visible list)
- Merges targeted refetch into existing counts without wiping other badges
- Ignores `documentUnlinked` events whose `orderId` is not in the visible list
- Ignores `documentUnlinked` events with no `orderId` in the payload

The `documentUnlinked` payload is `{ orderId, documentId }` — `orderId` is present, so the same `payload?.orderId` extraction used for `documentAdded` applies.

**GREEN**: In `useOrderNotificationCounts.ts`, in the SignalR `useEffect` (lines 93-131), add a handler and registration:
```typescript
const handleDocumentUnlinked = (payload?: { orderId?: string }) => {
  const orderId = payload?.orderId;
  if (!orderId || !visibleSet.has(orderId)) {
    return;
  }
  fetchCountsForOrders([orderId]);
};
```
Register and deregister it alongside `handleDocumentAdded`:
```typescript
connection.on('documentUnlinked', handleDocumentUnlinked);
// cleanup:
connection.off('documentUnlinked', handleDocumentUnlinked);
```

**REFACTOR**: No structural changes needed. The handler is a direct mirror of `handleDocumentAdded`.

**Commit**: `[MLID-2413] - feat(useOrderNotificationCounts): refetch on documentUnlinked SignalR event`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/notification.ts` | Modify | Add `deleteNotificationsForOrderDocument` and `deleteNotificationReadsByNotificationIds` exports |
| `apps/web/services/mongodb/notification.test.ts` | Modify | Add `deleteMany` to Notification/NotificationRead mocks; add test blocks for both new functions |
| `apps/web/services/audit/documentAudit.ts` | Modify | Add `writeOrderDocumentUnlinkAuditEntry` export (remove-side sibling of existing add-side helper) |
| `apps/web/services/audit/documentAudit.test.ts` | Modify | Add test block for `writeOrderDocumentUnlinkAuditEntry` mirroring existing add-side tests |
| `apps/web/services/notifications/handleOrderUnlink.ts` | Create | New orchestrator: feature-flag gate, per-orderId four-step cleanup loop, never throws |
| `apps/web/services/notifications/handleOrderUnlink.test.ts` | Create | Unit tests covering all four steps, early returns, per-step failure isolation, sequential ordering |
| `apps/web/services/mongodb/auditLog.config.ts` | Modify | Update `documents.renderChange` — Order category renders new add/remove text; all other categories keep legacy `New Document received` format |
| `apps/web/services/mongodb/auditLog.config.test.ts` | Create | Unit tests for the Order-category branch (add + remove) and the legacy fallback for non-Order categories |
| `apps/web/app/orders-tracker/[category]/[orderId]/order-history/components/AuditLogTable.test.tsx` | Modify | Add Order-category remove-side test case (existing Lab Results add-side test stays unchanged) |
| `apps/web/app/api/documents/[documentId]/link-orders/route.ts` | Modify | Compute `removedOrderIds` delta; fire `handleOrderUnlink` fire-and-forget for non-empty removed set |
| `apps/web/app/api/documents/[documentId]/link-orders/route.test.ts` | Create | Full route test coverage: auth, validation, add-delta, remove-delta, no-change, internal error |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | Modify | Register `documentUnlinked` listener that calls `fetchNotifications` |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx` | Modify | Add `documentUnlinked` handler test block |
| `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` | Modify | Register `documentUnlinked` listener with targeted per-`orderId` refetch |
| `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.test.tsx` | Modify | Add `documentUnlinked` handler test block mirroring `documentAdded` tests |

---

## Testing Strategy

### Service helpers (Steps 1-3)

Unit tests in `notification.test.ts` and `documentAudit.test.ts`. Mocks at the Mongoose model boundary (`Notification`, `NotificationRead`) and `writeAuditLog`.

- `deleteNotificationsForOrderDocument`: verify `$and` / dual-`$elemMatch` filter shape (not dot-notation), `find().select('_id').lean()` called before `deleteMany`, returned IDs match `find` results, empty-guard early return
- `deleteNotificationReadsByNotificationIds`: verify `{ notificationId: { $in: [...] } }` filter, empty-array guard, `deletedCount` forwarded
- `writeOrderDocumentUnlinkAuditEntry`: verify `changes[0]` shape is `{ field: 'documents', from: { _id, category, source }, to: null }`, error-swallowing on `writeAuditLog` failure

### Orchestrator (Step 4)

Unit tests in `handleOrderUnlink.test.ts`. All boundaries mocked (notification service, audit helper, SignalR). Key behaviors:
- Feature flag off → no-op
- Empty `removedOrderIds` or empty `documentId` → no-op
- Sequential per-orderId processing when multiple IDs given
- Deleted notification IDs are passed from step 1 to step 2
- Step 1 failure → steps 2 and 4 skipped; step 3 (audit) still runs for that orderId (audit is non-throwing, always proceeds)
- Step 2 failure → step 4 skipped; orchestrator does not throw
- Step 4 (broadcast) failure → orchestrator does not throw; steps 1-3 already ran

### Route (Step 6)

Integration tests in `route.test.ts`. All service boundaries mocked. Covers auth (401/403), input validation (400/422), not-found (404), add-delta fires `handleOrderLink`, remove-delta fires `handleOrderUnlink`, both are fire-and-forget (response is 200 even on helper rejection), no-overlap between added and removed delta calls.

### Renderer (Step 5)

Unit tests in `auditLog.config.test.ts` cover both branches of the discriminated renderer and the fallback for missing `category`. The `AuditLogTable.test.tsx` regression-guards the rendered text end-to-end in a component test.

### Hooks (Steps 7-8)

React Testing Library / `renderHook` tests. Mock `useSignalR` connection, `useSession`, `useFeatureFlag`, `fetch`. Verify that when `documentUnlinked` fires on the mock connection, the hook calls `fetch` with the correct URL/params. Verify `connection.off` is called on cleanup.

### Manual verification

After all steps pass automated tests, verify against the testing notebook at `docs/agomez/testing/MLID-2413-testing.md` (fixture data prepared). Steps to verify manually:
1. Enable `ORDER_DOCUMENT_NOTIFICATIONS` feature flag in `/admin/feature-flags`
2. Navigate to an order with an assigned user and a linked `category: "Order"` document — confirm badge count > 0 and "New" chip visible
3. Unlink the document via the link-orders dialog
4. Confirm the badge on the Documents tab decreases by the correct amount without page reload
5. Confirm the "New" chip and "Mark as Read" button disappear for the unlinked document
6. Navigate to the order History tab — confirm a `"A Document has been removed (Document type = Order)"` entry appears with the correct timestamp and actor
7. Re-link the same document — confirm fresh notification and `"A Document has been added (Document type = Order)"` entry appear
8. In MongoDB, confirm `notifications` and `notificationreads` contain no orphaned rows for the `(order, document)` pair after unlink

---

## Security Considerations

**Authorization**: The trigger site (`PUT /api/documents/[documentId]/link-orders`) already enforces `getServerSession` authentication (401 when no session) and `hasAllowedRole` role check (403 for disallowed roles) before any document operations. `handleOrderUnlink` is called only after both checks pass. `actorId` is sourced from `session.user?.id` — the same pattern as the existing `handleOrderLink` invocation.

**Input validation**: `documentId` is validated as a 24-char hex ObjectId string before the route body is processed. `sanitizedIds` filters any non-ObjectId entries from the payload. `removedOrderIds` is derived from the difference between the sanitized incoming array and the document's existing `linkedOrderIds` — both already validated — so no additional validation is needed at the helper boundary.

**PHI handling**: No patient PHI is present in the `notifications` or `notificationreads` collections. Audit-log entries store only `orderId`, `documentId`, `category`, `source`, and `actorId` — no patient-identifying data. The `logger` calls in `handleOrderUnlink` log only `orderId` and `documentId`, not patient fields. This is consistent with the existing `handleOrderReassignment` logging pattern.

**Fire-and-forget safety**: `handleOrderUnlink(...).catch(...)` at the route boundary ensures the HTTP response (200 + updated document) is returned immediately regardless of cleanup outcome. The helper itself never throws; the outer `.catch` is belt-and-suspenders only. A cleanup failure cannot leak into a 500 response.

**Feature flag gating**: `FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` is checked as the first operation inside `handleOrderUnlink` — when off, no DB mutations or SignalR broadcasts occur. The feature flag is not checked at the UI level for this task because `handleOrderUnlink` is a server-side fire-and-forget with no UI-visible gating point; the existing `handleOrderLink` follows the same pattern.

---

## Open Questions

None. The architecture is fully specified in the MLID-2417 epic plan. The step-ordering decision (audit always runs regardless of step-2 failure) is documented inline in Step 4.
