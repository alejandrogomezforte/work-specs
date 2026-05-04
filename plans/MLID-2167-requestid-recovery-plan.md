# Plan: MLID-2167 — Spruce Archive requestId Recovery

**Branch**: `fix/MLID-2167-spruce-archive-requestid-recovery`
**Parent branch**: `develop`
**Postmortem**: `docs/agomez/postmortem/unarchived-spruce-messages.md`

---

## Context

The original MLID-2167 fix (commit `a5d014cb`) narrowed the race condition between `sendMessageJob` and the Spruce outbound webhook, but didn't eliminate it. Messages that lose the race end up permanently orphaned: no `spruce.conversationId`, no archival, no Spruce link in the UI.

Investigation revealed two additional problems:
1. The `conversationitems` collection (which stores raw Spruce webhook payloads) overwrites `requestId` with an empty string when the `conversationItem.updated` webhook arrives after `.created`.
2. The `spruceArchiveSweeper` only looks for messages that already have `conversationId` — it can't recover orphaned messages.

This fix addresses both problems.

---

## Fix 1: Preserve `requestId` in `conversationitems`

**File**: `apps/web/services/webhooks/handlers/spruce_conversation.ts`
**Test file**: `apps/web/services/webhooks/handlers/spruce_conversation.test.ts`

### Problem

At line 264-276, the `findOneAndUpdate` uses `$set: { data }` which replaces the entire `data` field. When Spruce sends the `.updated` webhook (which has `requestId: ""`), it overwrites the `requestId` that the `.created` webhook had saved.

**Data impact**: 317,215 out of 366,669 outbound conversation items (86%) have lost their `requestId`.

### Change

Modify the `findOneAndUpdate` to preserve `requestId` when the incoming value is empty. After the upsert, if the new payload has an empty `requestId` but the doc already had a non-empty one, restore it.

Approach — use two operations:
1. Keep the existing `$set: { data }` upsert (preserves current behavior for inserts and for webhooks that have `requestId`).
2. After the upsert, if `data.data.object.requestId` is falsy in the incoming payload, run a targeted update to restore `requestId` from the previous value.

Alternative simpler approach:
- Before the `findOneAndUpdate`, check if `requestId` is empty in the incoming payload.
- If empty, use `$set` on individual `data` subfields but exclude `data.data.object.requestId` (use dot notation).
- If non-empty, use the current `$set: { data }` as-is.

### TDD

**RED tests to write first:**
1. `should preserve requestId when conversationItem.updated webhook has empty requestId` — upsert with `.created` (has requestId), then upsert with `.updated` (empty requestId), verify requestId is preserved.
2. `should save requestId when conversationItem.created webhook has non-empty requestId` — verify first upsert saves requestId correctly.
3. `should overwrite requestId when conversationItem.updated has a different non-empty requestId` — edge case, non-empty should still update.

---

## Fix 2: Expand sweeper with `conversationitems` fallback

**File**: `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts`
**Test file**: `apps/web/services/jobs/definitions/spruceArchiveSweeper.test.ts` (NEW)

### Problem

The sweeper query at line 67-75 requires `spruce.conversationId: { $exists: true }`. Messages that lost the race condition have no `conversationId` — the sweeper ignores them entirely.

### Change

Add a **second pass** to `runSpruceArchiveSweeper` after the existing pass:

**Pass 1** (existing, unchanged): Find messages with `conversationId` but no `archivedAt`. Proceed with archival as before.

**Pass 2** (new): Find orphaned messages — have `spruce.requestId` but no `spruce.conversationId`:

```
Query messages:
  source: { $in: archivableSources }
  spruce.requestId: { $exists: true, $ne: "" }
  spruce.conversationId: { $exists: false }
  sentDate: within lookback window
```

For each orphaned message:
1. Query `conversationitems` by `data.data.object.requestId` matching `message.spruce.requestId`.
2. If found and `conversationId` is non-empty:
   - Backfill `spruce.conversationId` and `spruce.appUrl` on the message.
   - Proceed with the existing `noteAndArchiveConversation` flow.
   - Stamp `spruce.archivedAt` on success.
3. If not found (or `requestId` was blanked — pre-Fix-1 data), log and skip.

### Lookback window consideration

The current `LOOKBACK_MS` is 1 hour. For Pass 2, we may want a longer lookback (e.g., 24 hours) since these are messages that were missed entirely and could be older. Add a separate constant `ORPHAN_LOOKBACK_MS` for Pass 2.

### Result tracking

Extend `SpruceArchiveSweeperResult` with new fields:
- `orphansScanned`: number of orphaned messages found
- `orphansRecovered`: number successfully backfilled and archived
- `orphansMissing`: number where `conversationitems` lookup failed

### TDD

**RED tests to write first:**
1. `should find and recover orphaned message using conversationitems lookup` — message with `requestId` but no `conversationId`, matching `conversationitems` doc exists → should backfill and archive.
2. `should skip orphaned message when conversationitems has no matching requestId` — no match in `conversationitems` → should log and skip.
3. `should skip orphaned message when conversationitems requestId was blanked` — match exists but `requestId: ""` → should skip.
4. `should not interfere with Pass 1 (existing behavior)` — messages with `conversationId` already set should still be processed by Pass 1 as before.
5. `should use ORPHAN_LOOKBACK_MS for Pass 2` — verify the longer lookback window.

---

## Implementation Order

```
Fix 1: Preserve requestId in conversationitems
  │
  ├── RED:  Write tests for requestId preservation
  ├── GREEN: Modify findOneAndUpdate logic
  └── REFACTOR: Clean up
  │
Fix 2: Expand sweeper with Pass 2
  │
  ├── RED:  Write tests for orphan recovery
  ├── GREEN: Add Pass 2 to runSpruceArchiveSweeper
  └── REFACTOR: Clean up, extract helpers if needed
```

Fix 1 must land first — it stops the data loss so that Fix 2's `conversationitems` lookup actually works for future messages.

---

## Files to Modify

| File | Change |
|------|--------|
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Fix 1: preserve `requestId` on upsert |
| `apps/web/services/webhooks/handlers/spruce_conversation.test.ts` | Fix 1: add tests for requestId preservation |
| `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts` | Fix 2: add Pass 2 orphan recovery |
| `apps/web/services/jobs/definitions/spruceArchiveSweeper.test.ts` | Fix 2: new test file for sweeper |

## Files NOT to Modify

| File | Reason |
|------|--------|
| `sendMessageJob.ts` | MLID-2167 immediate-save fix already in place, no changes needed |
| `ConversationItem.ts` (model) | Schema is `Mixed`, no changes needed |
| `spruceArchiveSweeper` job registration | No schedule/config changes needed |

---

## Verification

After implementation:
1. Run `npm run test` from `apps/web/` for all new/modified test files
2. Run `npm run types:check` from `apps/web/`
3. Run `npm run lint:fix` from `apps/web/`
4. Manual verification: check the Bangor message in the `/messages` UI after sweeper runs
