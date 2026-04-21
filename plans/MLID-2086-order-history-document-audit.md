# MLID-2086 — D3-T3: Order History — Document Audit Trail Entries

## Task Reference

- **Jira**: [MLID-2086](https://localinfusion.atlassian.net/browse/MLID-2086)
- **Story Points**: 3
- **Branch**: `feature/MLID-2086-order-history-document-audit`
- **Base Branch**: `epic/MLID-2011-order-document-notifications`
- **Status**: In Development
- **Epic Plan**: `docs/agomez/plans/MLID-2011-plan-progress.md` (D3-T3 Refinement section)

---

## Summary

When a document is added to an order, write a structured audit log entry to the existing `auditLogs` collection so that document additions appear in the Order History tab. Uses existing `action: 'update'` with `changes: [{ field: 'documents', from: null, to: { _id, category, source } }]`. No schema extensions — the existing `AuditLogChange.to: unknown` already accepts objects. Gated behind `ORDER_DOCUMENT_NOTIFICATIONS` feature flag.

---

## Architecture (from epic plan-progress — source of truth)

### Data shape stored in `auditLogs`

```typescript
{
  entityType: 'order',
  entityId: '<orderId>',
  action: 'update',                    // existing action, no new type
  actorId: '<userId>' | '',            // userId for uploads, '' for system
  source: 'user' | 'webhook' | 'system', // who wrote the audit row
  changes: [{
    field: 'documents',                // relation name
    from: null,                        // document did not exist before
    to: {                              // snapshot of what was added
      _id: '<documentId>',
      category: 'Lab Results',         // human-readable doc category
      source: 'fax',                   // 'fax' | 'e-order' | 'upload'
    },
  }],
  timestamp: Date
}
```

### Renderer — `renderChange` hook on `AuditFieldConfig`

`AuditLogTable.tsx` is `'use client'` — cannot import server-side `getFieldConfig()`. Solution: add `renderChange` to the config, pre-compute `renderedChange` string server-side during enrichment in `getAuditLog()`, pass it through on `EnrichedAuditLogChange`.

Config entry:
```typescript
documents: {
  label: 'New Document',
  category: 'Documents',
  renderChange: (change) => {
    const to = change.to as { category?: string } | null;
    return `New Document received (Document type = ${to?.category ?? 'Unknown'})`;
  },
}
```

Display: `New Document received (Document type = Lab Results)`. Source stored but **NOT displayed** this iteration per designer reference.

### actorId and audit-log source per entry point

| Entry point | `actorId` | `source` |
|---|---|---|
| Upload (manual) | `session.user.id` | `'user'` |
| E-Order (JotForm) | `''` | `'webhook'` |
| Fax (Spruce, `intake.source === 'spruce'`) | `''` | `'webhook'` |

### Feature flag

**Gated behind `ORDER_DOCUMENT_NOTIFICATIONS`** — consistent with the rest of the epic.

### Fax detection

Fax and e-order share the same `weinfuseUploader` code path. Distinguish via `intake.source === 'spruce'` → document source `'fax'`, otherwise `'e-order'`.

---

## Codebase Analysis

**Existing audit service (`apps/web/services/mongodb/auditLog.ts`):**
- `writeAuditLog(params)` — inserts to `auditLogs`, swallows errors. Already handles `changes[]`.
- `getAuditLog(params)` — enrichment step at lines 144-155 calls `getFieldConfig(change.field)` per change, producing `EnrichedAuditLogChange` with `fieldLabel` and `category`. **This is where `renderedChange` gets pre-computed.**
- `getAuditLogFilterOptions()` — derives categories from `distinct('changes.field')`. The `'documents'` field will appear in the config as `category: 'Documents'`, so it will show up in the filter dropdown automatically.

**Config (`apps/web/services/mongodb/auditLog.config.ts`):**
- `AuditFieldConfig = { label: string; category: string }` — needs `renderChange?: (change: AuditLogChange) => string`
- `ORDER_FIELD_CONFIG` — needs `documents` entry
- `getFieldConfig()` — returns config or falls through to `humanizeFieldName` + `'Other'` category

**Types (`apps/web/types/auditLog.ts`):**
- `EnrichedAuditLogChange` — needs `renderedChange?: string` (pre-computed server-side)
- `AuditLogChange.to: unknown` — already accepts objects, **no change needed**
- `AuditAction`, `AuditSource`, `WriteAuditLogParams` — **no changes needed**

**UI (`apps/web/app/orders-tracker/[category]/[orderId]/order-history/components/AuditLogTable.tsx`):**
- `flattenEntries()` line 48: currently formats `Original value: X\nNew value: Y`
- Needs to check `change.renderedChange` first — if present, use it; otherwise fall through to default format
- `displayValue()` coerces via `String()` — objects become `[object Object]`, so the `renderedChange` pre-computation avoids this

**Notification service (`apps/web/services/notifications/documentNotification.ts`):**
- `notifyDocumentCreated()` performs patient→orders lookup (lines 54-80). Extract `findOrdersForPatient()` as shared helper.
- `DocumentNotificationSource = 'upload' | 'e-order'` — needs `'fax'` added

**Call sites:**
- Upload: `apps/web/app/api/documents/split-pdf/route.ts` ~lines 339, 582
- E-order/Fax: `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` ~line 341

**Test files to extend:**
- `apps/web/services/mongodb/__tests__/auditLog.test.ts` — mocks `connectToDatabase`
- `apps/web/app/orders-tracker/.../AuditLogTable.test.tsx` — RTL tests with `EnrichedAuditLogEntry` objects
- `apps/web/app/api/documents/split-pdf/__tests__/route.document.test.ts` — mocks `notifyDocumentCreated`
- `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.document.test.ts` — same pattern

---

## Implementation Steps

### Step 1 — Types: add `renderedChange` to `EnrichedAuditLogChange`

- **Files**: `apps/web/types/auditLog.ts`
- **What**: Add `renderedChange?: string` to `EnrichedAuditLogChange`. This is the only type change — no new action, no new source, no metadata field.
- **Test**: `npm run types:check` from `apps/web/`. No Jest test for pure types.

### Step 2 — RED/GREEN/REFACTOR: `renderChange` hook on `AuditFieldConfig`

- **Files**: `apps/web/services/mongodb/auditLog.config.ts`, test in `apps/web/services/mongodb/__tests__/auditLog.test.ts`
- **What (RED)**: Add test:
  ```
  it('should return renderChange hook for documents field')
  ```
  Calls `getFieldConfig('documents')` and asserts `config.label === 'New Document'`, `config.category === 'Documents'`, `typeof config.renderChange === 'function'`. Also test the hook output: `config.renderChange({ field: 'documents', from: null, to: { category: 'Lab Results' } })` returns `'New Document received (Document type = Lab Results)'`. Run — fails.
- **What (GREEN)**: Add `renderChange?: (change: AuditLogChange) => string` to `AuditFieldConfig`. Add `documents` entry to `ORDER_FIELD_CONFIG`:
  ```typescript
  documents: {
    label: 'New Document',
    category: 'Documents',
    renderChange: (change: AuditLogChange) => {
      const to = change.to as { category?: string } | null;
      return `New Document received (Document type = ${to?.category ?? 'Unknown'})`;
    },
  },
  ```
- **What (REFACTOR)**: Verify existing config tests still green.

### Step 3 — RED/GREEN/REFACTOR: `getAuditLog` pre-computes `renderedChange`

- **Files**: `apps/web/services/mongodb/__tests__/auditLog.test.ts` (RED), `apps/web/services/mongodb/auditLog.ts` (GREEN)
- **What (RED)**: Add test:
  ```
  it('should populate renderedChange when field config has renderChange hook')
  ```
  Seeds `mockAggregateToArray` with an entry having `changes: [{ field: 'documents', from: null, to: { _id: 'doc-1', category: 'Lab Results', source: 'upload' } }]`. Asserts `result.data[0].changes[0].renderedChange === 'New Document received (Document type = Lab Results)'`. Run — fails.
- **What (GREEN)**: In the enrichment map (lines 144-155), after getting `config = getFieldConfig(change.field)`, add:
  ```typescript
  renderedChange: config.renderChange ? config.renderChange(change) : undefined,
  ```
- **What (REFACTOR)**: Verify existing `getAuditLog` tests still green.

### Step 4 — RED/GREEN/REFACTOR: `AuditLogTable` uses `renderedChange`

- **Files**: `apps/web/app/orders-tracker/.../AuditLogTable.test.tsx` (RED), `AuditLogTable.tsx` (GREEN)
- **What (RED)**: Add test:
  ```
  it('renders renderedChange text instead of default from/to format when present')
  ```
  Constructs an `EnrichedAuditLogEntry` with `action: 'update'`, `changes: [{ field: 'documents', fieldLabel: 'New Document', category: 'Documents', from: null, to: { _id: 'doc-1', category: 'Lab Results', source: 'upload' }, renderedChange: 'New Document received (Document type = Lab Results)' }]`. Asserts:
  - `screen.getByText('New Document')` — Field column
  - `screen.getByText('Documents')` — Category column
  - `screen.getByText('New Document received (Document type = Lab Results)')` — Change column
  - Does NOT contain `'Original value:'` or `'[object Object]'`
  Also add test confirming existing entries WITHOUT `renderedChange` still use the default format.
  Run — fails.
- **What (GREEN)**: In `flattenEntries()`, update the change text construction at line 48:
  ```typescript
  change: change.renderedChange ?? `Original value: ${displayValue(change.from)}\nNew value: ${displayValue(change.to)}`,
  ```
- **What (REFACTOR)**: All existing `AuditLogTable` tests still green.

### Step 5 — RED/GREEN/REFACTOR: Extract `findOrdersForPatient` + add `'fax'` source

- **Files**: `apps/web/services/notifications/documentNotification.ts`, existing notification tests
- **What**: Extract lines 54-80 of `notifyDocumentCreated()` into:
  ```typescript
  export async function findOrdersForPatient(
    patientId: string
  ): Promise<Array<{ _id: unknown; assignedTo?: string }>>
  ```
  Performs: `Patient.findById(patientId).select('we_infuse_id_IV').lean()` → returns `[]` if patient not found or no `we_infuse_id_IV` → `db.collection(COLLECTION.NewOrders).find(...)`.
  
  Refactor `notifyDocumentCreated()` to call this helper. Run existing notification tests — should remain green (behavior unchanged).
  
  Also add `'fax'` to `DocumentNotificationSource` and `SOURCE_LABELS`:
  ```typescript
  export type DocumentNotificationSource = 'upload' | 'e-order' | 'fax';
  const SOURCE_LABELS: Record<DocumentNotificationSource, string> = {
    upload: 'Upload',
    'e-order': 'E-Order',
    fax: 'Fax',
  };
  ```

### Step 6 — RED/GREEN/REFACTOR: New service `recordDocumentAddedToOrders`

- **Files**: `apps/web/services/audit/documentAudit.test.ts` (RED — new), `apps/web/services/audit/documentAudit.ts` (GREEN — new)
- **What (RED)**: Create test file. Mock `findOrdersForPatient`, `writeAuditLog`, `FeatureFlagService.getFeatureFlag`, `logger`. Test cases:
  - `should write one audit entry per linked order` — stubs 2 orders; asserts `writeAuditLog` called twice with `action: 'update'`, `changes: [{ field: 'documents', from: null, to: { _id: documentId, category, source } }]`
  - `should use actorId and source correctly for upload` — `actorId: 'user-abc'`, `source: 'user'`
  - `should use actorId and source correctly for system` — `actorId: ''`, `source: 'webhook'`
  - `should no-op when patientId is undefined`
  - `should no-op when feature flag is disabled`
  - `should not throw when writeAuditLog rejects`
  Run — all fail.
- **What (GREEN)**: Create `apps/web/services/audit/documentAudit.ts`:
  ```typescript
  import { FeatureFlagService } from '@/services/featureFlags/featureFlagService';
  import { writeAuditLog } from '@/services/mongodb/auditLog';
  import { findOrdersForPatient } from '@/services/notifications/documentNotification';
  import { AuditSource } from '@/types/auditLog';
  import { FeatureFlag } from '@/utils/featureFlags/featureFlags';
  import { logger } from '@/utils/logger';

  export interface RecordDocumentAddedParams {
    documentId: string;
    patientId: string | undefined;
    category: string;        // human-readable document category
    documentSource: 'fax' | 'e-order' | 'upload';
    actorId: string;         // userId for uploads, '' for system
    auditSource: AuditSource; // 'user' | 'webhook' | 'system'
  }

  export async function recordDocumentAddedToOrders(
    params: RecordDocumentAddedParams
  ): Promise<void> {
    try {
      const { documentId, patientId, category, documentSource, actorId, auditSource } = params;

      const isEnabled = await FeatureFlagService.getFeatureFlag(
        FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS
      );
      if (!isEnabled) return;

      if (!patientId) return;

      const orders = await findOrdersForPatient(patientId);
      if (orders.length === 0) return;

      for (const order of orders) {
        await writeAuditLog({
          entityType: 'order',
          entityId: String(order._id),
          action: 'update',
          actorId,
          source: auditSource,
          changes: [{
            field: 'documents',
            from: null,
            to: { _id: documentId, category, source: documentSource },
          }],
        });
      }
    } catch (error) {
      logger.error('Failed to record document added audit entries', {
        error,
        documentId: params.documentId,
      });
    }
  }
  ```
  Run tests — pass.

### Step 7 — RED/GREEN/REFACTOR: Upload call site (`split-pdf/route.ts`)

- **Files**: `apps/web/app/api/documents/split-pdf/__tests__/route.document.test.ts` (RED), `route.ts` (GREEN)
- **What (RED)**: Add mock:
  ```typescript
  jest.mock('@/services/audit/documentAudit', () => ({
    recordDocumentAddedToOrders: jest.fn().mockResolvedValue(undefined),
  }));
  ```
  Add test asserting `recordDocumentAddedToOrders` called with `{ documentSource: 'upload', actorId: '<session-user-id>', auditSource: 'user' }`.
- **What (GREEN)**: At both call sites (~339, ~582), after `void notifyDocumentCreated(...)`:
  ```typescript
  void recordDocumentAddedToOrders({
    documentId: String(createdDoc._id),
    patientId: patient?._id?.toString(),
    category: uploadData.category,
    documentSource: 'upload',
    actorId: session.user?.id || '',
    auditSource: 'user',
  });
  ```
- **What (REFACTOR)**: Existing `notifyDocumentCreated` assertions still pass.

### Step 8 — RED/GREEN/REFACTOR: E-Order / Fax call site (`weinfuseUploader.ts`)

- **Files**: `weinfuseUploader.document.test.ts` (RED), `weinfuseUploader.ts` (GREEN)
- **What (RED)**: Add mock for `recordDocumentAddedToOrders`. Add tests:
  - When `intake.source !== 'spruce'`: assert `documentSource: 'e-order'`
  - When `intake.source === 'spruce'`: assert `documentSource: 'fax'`
  Both with `actorId: ''`, `auditSource: 'webhook'`.
- **What (GREEN)**: After `void notifyDocumentCreated(...)`:
  ```typescript
  const docSource = intake.source === 'spruce' ? 'fax' : 'e-order';
  void recordDocumentAddedToOrders({
    documentId: String(createdDoc._id),
    patientId: patientDbId,
    category: uploadData.category,
    documentSource: docSource,
    actorId: '',
    auditSource: 'webhook',
  });
  ```
  Also update the existing `notifyDocumentCreated` call to use `source: docSource` so fax notifications are labeled correctly too.
- **What (REFACTOR)**: All existing `weinfuseUploader` tests green.

### Step 9 — Coverage sweep

- From `apps/web/`:
  ```bash
  npm run test -- --coverage --testPathPattern="auditLog|documentAudit|documentNotification|split-pdf|weinfuseUploader|AuditLogTable"
  ```
  All modified files must meet 80% threshold. Fix gaps.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/types/auditLog.ts` | Modify | Add `renderedChange?: string` to `EnrichedAuditLogChange` |
| `apps/web/services/mongodb/auditLog.config.ts` | Modify | Add `renderChange` to `AuditFieldConfig`; add `documents` entry to `ORDER_FIELD_CONFIG` |
| `apps/web/services/mongodb/auditLog.ts` | Modify | Pre-compute `renderedChange` in `getAuditLog` enrichment step |
| `apps/web/services/mongodb/__tests__/auditLog.test.ts` | Modify | Tests for config hook, `renderedChange` enrichment |
| `apps/web/services/notifications/documentNotification.ts` | Modify | Extract `findOrdersForPatient()`; add `'fax'` to `DocumentNotificationSource` + `SOURCE_LABELS` |
| `apps/web/services/audit/documentAudit.ts` | Create | `recordDocumentAddedToOrders()` — feature-flag gated, patient→orders lookup via shared helper, `writeAuditLog` per order |
| `apps/web/services/audit/documentAudit.test.ts` | Create | Unit tests for all branches + error paths |
| `apps/web/app/api/documents/split-pdf/route.ts` | Modify | Add `void recordDocumentAddedToOrders(...)` at both call sites |
| `apps/web/app/api/documents/split-pdf/__tests__/route.document.test.ts` | Modify | Mock + assert audit call with upload params |
| `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` | Modify | Derive `docSource` from `intake.source`; add audit call; update notification source for fax |
| `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.document.test.ts` | Modify | Mock + assert audit call for e-order and fax paths |
| `apps/web/app/orders-tracker/.../AuditLogTable.tsx` | Modify | Use `change.renderedChange` when present in `flattenEntries()` |
| `apps/web/app/orders-tracker/.../AuditLogTable.test.tsx` | Modify | Test `renderedChange` rendering + confirm default format preserved |

---

## Testing Strategy

**Unit tests — `recordDocumentAddedToOrders`:**
- Writes one `writeAuditLog` per linked order with `action: 'update'`, `changes: [{ field: 'documents', from: null, to: { _id, category, source } }]`
- Correct `actorId` + `source` per entry point
- No-op when `patientId` undefined, feature flag off, or no orders found
- Does not throw on `writeAuditLog` failure

**Unit tests — config + enrichment:**
- `getFieldConfig('documents')` returns label, category, and `renderChange` hook
- Hook produces correct display text
- `getAuditLog` enrichment populates `renderedChange` from hook

**Unit tests — `AuditLogTable`:**
- Entry with `renderedChange` renders that text (not `Original value:` format)
- Entry without `renderedChange` renders default format (no regression)

**Unit tests — call sites:**
- Upload: `recordDocumentAddedToOrders` called with `documentSource: 'upload'`, `auditSource: 'user'`
- E-order: called with `documentSource: 'e-order'`, `auditSource: 'webhook'`
- Fax: called with `documentSource: 'fax'`, `auditSource: 'webhook'`

**Manual verification:**
1. Upload a document via split-PDF for a patient with active new orders
2. Navigate to Order Tracker → that order → Order History tab
3. Verify row: Field=`New Document`, Category=`Documents`, Change=`New Document received (Document type = <category>)`, User=logged-in user
4. Trigger e-order intake; verify row with User=`System`
5. Order History filter → Category=`Documents` → only document rows shown
6. Edit order's `assignedTo` → verify existing "Assigned to" rows still render normally

---

## Security Considerations

- `documentTitle` (`Document.description`) and `documentType` (`Document.category`) are document metadata, not patient PHI. Consistent with how existing notification flow uses `category`.
- No new API endpoints — reading Order History uses existing auth middleware.
- `patientId` is NOT stored in the audit entry — only `entityId` (the order ID).

---

## Open Questions

1. ~~Fax ingestion entry point~~ — **Resolved:** fax shares `weinfuseUploader` path; distinguished via `intake.source === 'spruce'`.
2. **Document category format** — need to verify `uploadData.category` at call sites is the human-readable label (e.g., `"Lab Results"`) and not an enum key. If it's an enum key, add a label map. (Verify during Step 7/8 implementation.)
