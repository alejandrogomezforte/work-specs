# PR: [MLID-2167] Add `conversationitems` index for `data.data.object.requestId`

**Branch**: `feature/MLID-2167-conversationitems-requestid-index`
**Base**: `develop`
**Related PRs**: #1212 (Pass 2 orphan recovery), #1215 (sweeper cadence)

---

## Background

PR #1212 added a second pass to `spruceArchiveSweeper` that recovers orphaned Spruce messages by looking up `conversationitems` via the Spruce `requestId`:

```ts
// apps/web/services/jobs/definitions/spruceArchiveSweeper.ts:180
const conversationItem = await ConversationItem.findOne({
  'data.data.object.requestId': requestId,
});
```

The `conversationitems` collection currently holds **~1.78M documents** and has no index covering that field. Every orphan lookup is a full collection scan, and Pass 2 runs one lookup per orphan in the batch, so the sweeper's cost grows linearly with collection size for every run.

Current indexes on `conversationitems` (before this change):

| Name | Key |
|------|-----|
| `_id_` | `{ _id: 1 }` |
| `conversation_item_id_1` (unique) | `{ conversation_item_id: 1 }` |
| `conversation_id_1` | `{ conversation_id: 1 }` |
| `conversation_id_1_conversation_item_id_1` | `{ conversation_id: 1, conversation_item_id: 1 }` |

---

## Changes

### Declare the index in the Mongoose schema

**File**: `apps/web/models/ConversationItem.ts`

Adds a single-field index declaration for `data.data.object.requestId` to match the index that was already created manually on staging (and will be created on production at release time):

```ts
ConversationItemSchema.index(
  { 'data.data.object.requestId': 1 },
  { name: 'data.data.object.requestId_1', background: true }
);
```

Options are mirrored byte-for-byte against the staging index so Mongoose's `ensureIndexes` on app boot (driven by the global `autoIndex: true` in `services/mongodb/connect.ts`) is a no-op against an already-built index instead of creating a duplicate.

| Option | Value | Reason |
|--------|-------|--------|
| `name` | `data.data.object.requestId_1` | Matches the default name `createIndex` assigned on staging. |
| `background` | `true` | Default on Mongo 4.2+; made explicit for readers. |
| partial filter | none | Staging index has no partial filter — matched for consistency. |
| unique | `false` | Spruce emits multiple events per `requestId` (`.created`, `.updated`), so the value is not unique per doc. |

### Why declare it in the schema at all (given it's already in the DB)?

1. **Source of truth** — the next person reading `ConversationItem.ts` sees which indexes back which queries. Without the declaration, the index is tribal knowledge that lives only in prod/staging.
2. **New environments** — local dev, CI integration tests, and any future restored environment start without this index. Declaring it in the schema ensures they pick it up on boot via `autoIndex`, so behavior matches prod.
3. **Git trail** — the index is tied to the sweeper change in history. Six months from now, `git blame` on the schema line points to the PR that needed it.
4. **Safe in Mongoose's default behavior** — `ensureIndexes` only *creates missing* indexes; it does not drop or rebuild mismatched ones. Nothing in this repo calls `syncIndexes()` on boot (verified with grep). The declaration is additive as long as the options match what was already built.

---

## Rollout

The index was created manually by the tech lead in staging via the mongo shell:

```js
db.conversationitems.createIndex({ 'data.data.object.requestId': 1 })
```

The same command will be run by the ops team on production at release time next week. This PR only adds the schema declaration — it does **not** trigger any production-side index creation beyond what ops will run manually.

---

## Files changed

| File | Change |
|------|--------|
| `apps/web/models/ConversationItem.ts` | Declare `data.data.object.requestId_1` index matching the one built manually on staging. |

---

## Test plan

- [x] Types check clean for the changed file (`npx tsc --noEmit` — only pre-existing `@repo/intake-ocr` and `docx` errors, unrelated to this change).
- [x] Prettier applied.
- [x] Verified no existing test file for `ConversationItem.ts`; a declarative index line doesn't require new tests.
- [x] Verified no `syncIndexes()` calls anywhere in `apps/web` — boot behavior is safe (`ensureIndexes` is idempotent against an existing exact-match index).
- [x] Verified `autoIndex: true` is set globally in `services/mongodb/connect.ts`, so the declaration will actually be consulted on boot in non-prod environments.
- [ ] After merge: confirm the staging index name and options exactly match the declaration (name `data.data.object.requestId_1`, no partial filter, no uniqueness). If they differ, follow up with a one-line option adjustment before the ops team builds the prod index.
- [ ] After prod release: verify `db.conversationitems.getIndexes()` on production shows the new index and that `spruceArchiveSweeper` Pass 2 `findOne` goes from COLLSCAN to IXSCAN in query profiling.

---

## Not doing

- **No migration script under `apps/web/db-update/`** — the index is being created manually by ops for both staging and prod (correct call for a collection this size). The schema declaration is just documentation + safety net for non-prod environments.
- **No partial filter on the index** — earlier analysis suggested a partial filter excluding empty-string `requestId` values to shrink index size. We matched the staging index (which has no partial filter) for consistency; a partial-filter optimization can be a separate follow-up if index size becomes a concern.
