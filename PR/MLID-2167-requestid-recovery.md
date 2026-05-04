# PR: [MLID-2167] Spruce Archive requestId Recovery

**Branch**: `fix/MLID-2167-spruce-archive-requestid-recovery`
**Base**: `develop`
**Postmortem**: `docs/agomez/postmortem/unarchived-spruce-messages.md`
**Plan**: `docs/agomez/plans/MLID-2167-requestid-recovery-plan.md`

---

## Background

The original MLID-2167 fix (immediate `message.save()` after Spruce send) and the safety-net sweeper closed most archival gaps, but investigation of a Bangor patient's unarchived SMS revealed two remaining problems:

1. **`conversationitems` requestId overwrite**: The `conversationItem.updated` webhook arrives with `requestId: ""`. The existing `$set: { data }` replaces the entire document, blanking the `requestId` that `.created` had saved. 86% of outbound conversation items (317,215 of 366,669) lost their `requestId`.

2. **Sweeper blind spot**: The sweeper query requires `spruce.conversationId: { $exists: true }`. Messages that lost the webhook race (sendMessageJob hadn't saved `requestId` before the webhook fired `Message.updateMany`) have no `conversationId` — the sweeper ignores them permanently.

---

## Changes

### Fix 1: Preserve `requestId` in `conversationitems` collection

**File**: `apps/web/services/webhooks/handlers/spruce_conversation.ts`

The upsert now branches on whether the incoming `requestId` is truthy:

- **Non-empty requestId**: `$set: { data }` as before — full overwrite is safe because the payload contains the real `requestId`.
- **Empty requestId**: Strips `requestId` from the payload via destructuring before `$set`, so the existing value in MongoDB is preserved. `$setOnInsert` seeds `requestId` to `""` for brand-new docs only.

This stops future data loss. Historical recovery depends on the `.created` webhooks that did arrive with valid `requestId` values.

### Fix 2: Expand `spruceArchiveSweeper` with orphan recovery (Pass 2)

**File**: `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts`

Added a second pass after the existing sweep:

- **Pass 1** (unchanged): Finds messages with `spruce.conversationId` but no `spruce.archivedAt` within a 1-hour lookback. Proceeds with `noteAndArchiveConversation`.

- **Pass 2** (new): Finds orphaned messages — have `spruce.requestId` but no `spruce.conversationId` — within a 24-hour lookback (`ORPHAN_LOOKBACK_MS`). For each orphan:
  1. Looks up `conversationitems` by `data.data.object.requestId` matching the message's `spruce.requestId`.
  2. If found with a non-empty `conversationId`: backfills `spruce.conversationId` and `spruce.appUrl` on the message, then runs the standard `noteAndArchiveConversation` flow.
  3. If not found or `conversationId` is empty: logs a warning and increments `orphansMissing`.

Extended `SpruceArchiveSweeperResult` with:
- `orphansScanned`: number of orphaned messages found
- `orphansRecovered`: number successfully backfilled and archived
- `orphansMissing`: number where `conversationitems` lookup failed

---

## Files changed

| File | Change |
|------|--------|
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Fix 1: branch upsert on requestId presence |
| `apps/web/services/webhooks/handlers/spruce_conversation.test.ts` | +3 tests for requestId preservation |
| `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts` | Fix 2: Pass 2 orphan recovery, extended result interface |
| `apps/web/services/jobs/definitions/spruceArchiveSweeper.test.ts` | +5 tests for orphan recovery scenarios (new file) |

---

## Test plan

- [x] `spruce_conversation.test.ts` — 3 new tests: save requestId on .created, preserve on .updated with empty, overwrite on .updated with non-empty. 29 tests total, all pass.
- [x] `spruceArchiveSweeper.test.ts` — 5 new tests: recover orphan via conversationitems lookup, skip when no match, skip when conversationId is blanked, Pass 1 non-interference, combined Pass 1 + Pass 2 stats. All 5 pass.
- [x] Types check clean (`npx tsc --noEmit` — only pre-existing `@repo/intake-ocr` and `docx` errors).
- [x] Prettier applied.
- [ ] Staging: trigger a new SMS, verify `.created` webhook saves requestId, `.updated` webhook preserves it (check `conversationitems` doc in MongoDB).
- [ ] Staging: insert a test message with `spruce.requestId` but no `spruce.conversationId` inside the 24h lookback; confirm sweeper Pass 2 recovers it and archives.
- [ ] Staging: verify Pass 1 still works independently (message with `conversationId` set, no `archivedAt`).
- [ ] Staging: verify the Bangor patient message (`69e8c652752417d8e83f2e4c`) gets recovered on next sweeper run.
