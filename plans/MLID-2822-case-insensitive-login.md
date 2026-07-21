# MLID-2822 — Make login email matching case-insensitive (hotfix)

## Task Reference

- **Jira**: [MLID-2822](https://localinfusion.atlassian.net/browse/MLID-2822)
- **Story Points**: 2–3
- **Branches (both pushed)**:
  - `hotfix/MLID-2822-case-insensitive-login` (off `main`) → PR to **`main`** — commit `c25bcee92`
  - `hotfix/MLID-2822-case-insensitive-login-develop` (off `develop`) → PR to **`develop`** — cherry-pick `3a5cb3912` (code-identical)
- **Delivery**: developed in a git worktree; two independent PRs (PR doc at `docs/agomez/PR/MLID-2822.md`; PRs opened manually).
- **Status**: **DONE — merged to `main` and `develop`, deployed.** Stage 1 (code) on both branches; 25 unit tests green + browser-verified; CI green after a follow-up fix to `__tests__/users.test.ts`. **No migration shipped.** The 3 mixed-case duplicate accounts were **DISABLED in production on 2026-07-16** (manually, after the code deploy):
  - `699f1d9db3bff5f408cc73f3`
  - `69a0c0f0ac42a5b48e1cc985`
  - `69f8985a50ec5fb019e9662e`

  Their enabled lowercase twins now match at login via the case-insensitive lookup. The code is self-sufficient without this step.
- **Authoritative sections**: 1b (decided approach) and 5c Stage 1 (implementation). **5c Stage 2 (the migrations) was dropped** in favor of the manual PO disable above; ignore the migration steps.

---

## 1. The problem (from the ticket)

A user cannot log in. The likely cause is that the email stored on her account contains uppercase characters while she signs in with a lowercased address. The login flow performs an **exact, case-sensitive** match on email at every step, so any difference in letter case makes the account lookup return nothing and the user is denied with `/?error=notAuthorized`.

Case-sensitivity is not introduced deliberately anywhere. It is the **absence of normalization** at three layers: incoming lookups, stored data, and future writes. All three must be addressed for the fix to be complete — normalizing the incoming email alone is not enough, because a lowercased input still will not match a mixed-case stored value.

For Google OAuth users, Google normally returns a lowercase address, which is why this has been mostly invisible. The problem appears when the **stored** email has mixed case.

## 1b. Decided approach (updated 2026-07-15, after production audit)

The production reference audit (recorded in the user-reference scan audit file) established a clear pattern across the three collision pairs: **the mixed-case (uppercase-containing) email is always the empty duplicate**, and any real footprint — always only HIPAA `fileaccesses` rows, never operational/blocking data — sits on the **lowercase** twin, which is also the account Google OAuth resolves to.

Given that, the chosen fix has two parts and deliberately defers the heavier data-normalization work:

1. **Ship the case-insensitive read patch** (the read-side + allow-list changes in Section 4). Login email lookups become case-insensitive so case can no longer block a valid user.
2. **Migration to disable the confirmed mixed-case duplicate accounts**, by hardcoded `_id` (soft, reversible — sets `disabled: true`, does not delete). The surviving lowercase account keeps all history.

The two parts are not interchangeable. Part 1 (the case-insensitive read) is the **permanent, forward-looking** fix: it makes login work for the current standalone mixed-case account and for any account created with uppercase letters in the future — we cannot control whether a future human admin, a data import, or an external identity provider stores an email with mixed case, so the lookup itself must tolerate any case. Part 2 (the disable migration) is a **one-time data cleanup** of the 3 known empty duplicates; it does not, and cannot, prevent future case mismatches. Even if Part 2 were skipped, Part 1 would still be required.

### Hardcoded ids to disable (3 confirmed clean duplicates)

These three mixed-case accounts were scanned and are empty duplicates whose lowercase twin holds the activity:

- `699f1d9db3bff5f408cc73f3`
- `69a0c0f0ac42a5b48e1cc985`
- `69f8985a50ec5fb019e9662e`

### The 4th uppercase account — NOT a disable target (and fixed by Stage 1 anyway)

The prod audit found **4** accounts with a non-lowercase email, but only **3** are collision duplicates. The **4th is standalone** (no lowercase twin) — it is that person's only account.

Good news from the collation approach: the Stage 1 case-insensitive read **fixes login for this standalone on its own** — a user typing lowercase now matches the mixed-case stored email regardless of case, and since it is the single enabled match it is returned. So the standalone is not a blocker.

Therefore: **do NOT disable the 4th account, and it does not need to be normalized for login to work.** Lowercasing its stored email is optional data hygiene only. The disable migration targets 3 ids, never 4.

### Design note — case-insensitive read must exclude disabled AT THE QUERY level

Confirmed behavior in code (does the `disabled` flag block login? yes):

- `getUserByEmail` (`apps/web/services/mongodb/users.ts`) ends with `if (!user || user.disabled) return null;` — a post-fetch guard. But its Mongo `findOne` filter is only `{ email, deleted: { $ne: true } }` (no `disabled` in the query). The Google OAuth `signInCallback` uses this function, so a disabled user resolves to `null` and is denied (unless their email is in the `AUTHORIZED_EMAILS` env allowlist, which bypasses the DB check).
- `getUserWithPasswordHash` (`apps/web/services/mongodb/password.ts`) filters `disabled: { $ne: true }` in the query itself, so the credentials/password path denies disabled users directly.

The trap: with an exact-match email, `findOne` returns exactly one document, so the post-fetch guard in `getUserByEmail` is enough. Once the lookup is **case-insensitive**, a collision pair (disabled mixed-case + enabled lowercase) yields **two** matches; `findOne` returns one non-deterministically. If it returns the disabled mixed-case doc, the post-fetch guard nulls the whole result and the user is **denied even though an enabled twin exists**.

Fix (required): add `disabled: { $ne: true }` to the query filter in the case-insensitive `getUserByEmail` — both the Mongo `findOne` and the Postgres `findByEmailForRead` path — not just the post-fetch guard. Then `findOne` only ever considers enabled accounts and deterministically returns the enabled lowercase account. `getUserWithPasswordHash` already filters `disabled` at the query, so it just needs the case-insensitive match added.

### Deployment ordering and sessions

- Run the disable migration **together with, or before,** the case-insensitive read deploy. Otherwise there is a window where both twins are still enabled and the case-insensitive `findOne` could resolve to the empty mixed-case account.
- Session strategy is JWT: disabling a user **blocks new sign-ins but does not terminate an existing active session** until the token expires. Not a concern for the empty duplicate accounts (no one is signed into them), noted for completeness.

### Scope change vs the original plan

This approach **defers** the full "lowercase every stored email" data migration and the destructive case-insensitive unique index (Section 4). Reads are normalized now; only the specific duplicate accounts are disabled. Write-side `lowercase: true` enforcement for future accounts is still recommended, but the unique-index rebuild can wait until the data is fully cleaned. Revisit the full normalization as a follow-up once these targeted duplicates are resolved.

### Environment note

The disable migration hardcodes **production** `_id`s, so it no-ops in staging (those ids do not exist there). Staging has its own separate collision (one pair, found earlier) that needs its own handling. Also: `fileaccesses` rows on the lowercase accounts are append-only compliance data and are retained — disabling the mixed-case duplicate does not touch them. Role (admin vs user) must be confirmed on each pair before disabling, so the surviving account keeps the required access.

## 2. Confirmed root cause (case-sensitive by omission)

Every email comparison in the auth chain is exact-match, and nothing normalizes case:

- NextAuth config: `apps/web/auth.ts` (Auth.js v5) — Google provider with `allowDangerousEmailAccountLinking: true`, plus a Credentials provider. Route handler at `apps/web/app/api/auth/[...nextauth]/route.ts`.
- Sign-in callback: `signInCallback` in `apps/web/app/api/auth/[...nextauth]/callbacks.ts` calls `getUserByEmail(user.email)` with the raw email, and checks `authorizedEmails.includes(user.email)` (exact JS match). `sessionCallback` repeats the raw lookup and also checks `AUDITOR_EMAILS.includes(session.user.email)`.
- Mongo read: `getUserByEmail` in `apps/web/services/mongodb/users.ts` runs `findOne({ email, deleted: { $ne: true } })` — plain equality, no `$regex`, no `.collation()`. This function can route to Postgres via the `PG_READ_COMPLEX_2` feature flag.
- Postgres read: `packages/db/src/users/repository.ts` uses Drizzle `eq(users.email, email)` — exact match.
- Credentials path: `apps/web/app/api/auth/[...nextauth]/credentials.ts` → `getUserWithPasswordHash` in `apps/web/services/mongodb/password.ts` runs `findOne({ email, deleted: { $ne: true }, disabled: { $ne: true } })` — exact match.
- User model: `apps/web/models/User.ts` — `email: { type: String, required: true }` (no `lowercase: true`, no `trim: true`); `UserSchema.index({ email: 1 }, { unique: true })` (plain index, no collation).
- Postgres schema: `packages/db/src/users/schema/users.ts` — plain `text('email')` column (not `citext`, no `COLLATE`), unique index on `email`.

The only normalization anywhere in the repo is `isUserWhiteListed` in `apps/web/services/mongodb/auth.ts` (`email.toLowerCase().trim()`), but it is **dead code** with no callers, targeting a collection dropped by an earlier db-update migration. It does not help here.

## 3. Security assessment

**This change does NOT create an external hacking vector.** In this app the email is a lookup key / authorization allow-list entry, **not** the authentication secret:

- Google OAuth identity is verified by Google (verified email + signed token). A case-insensitive lookup on our side does not weaken Google's verification.
- The Credentials provider is still gated by the password hash. Case-insensitive email lookup is the industry-standard behavior (email is case-insensitive in practice) and does not bypass the password check.

**The real risk is internal data integrity — pre-existing case-variant duplicate accounts.** The current unique index is case-sensitive, so two accounts differing only by case (for example `John@x.com` and `john@x.com`) may already coexist. Lowercasing everything could:

- (a) fail the new case-insensitive unique index during migration, or
- (b) cause a login to match the **wrong** account — an identity mix-up (internal, not an outside attack).

Mitigation: run the duplicate audit (Step 5) in **both stage and production** and resolve any collision manually **before** the data migration.

Secondary notes:
- `allowDangerousEmailAccountLinking: true` (in `apps/web/auth.ts`) links accounts by email. Once normalized, mixed-case variants become the same identity for linking — the desired behavior, but another reason to clear duplicates first.
- Locale casing (minor): JavaScript `toLowerCase()` uses Unicode default casing, safe for normal ASCII emails. Apply the same normalization on both input and stored data so the two sides always agree.

## 4. Affected files

> **SUPERSEDED (analysis only).** Follow Section 1b (decided approach) and Section 5c (implementation steps) for what to build. This original table includes the **deferred, out-of-scope** write-side work — `lowercase: true` / `trim: true` on the schema, the case-insensitive unique index, and the lowercase-all data migration — which this hotfix does NOT implement. The read-side rows (`getUserByEmail`, `getUserWithPasswordHash`, `packages/db` repository, `callbacks.ts`) remain valid and map to Stage 1. Kept for reference.

| File | Change |
|------|--------|
| `apps/web/services/mongodb/users.ts` (`getUserByEmail`) | Lowercase + trim `email` before the query. Central Mongo read used by both sign-in callbacks — the best single choke point. |
| `apps/web/services/mongodb/password.ts` (`getUserWithPasswordHash`) | Same normalization before the query (Credentials-provider path). |
| `packages/db/src/users/repository.ts` | Lowercase `email` before `eq(users.email, email)` (Postgres read path via `PG_READ_COMPLEX_2`). Must match Mongo behavior exactly. |
| `apps/web/app/api/auth/[...nextauth]/callbacks.ts` | Lowercase before the `AUTHORIZED_EMAILS.includes(...)` and `AUDITOR_EMAILS.includes(...)` checks, and lowercase the env-var lists themselves. |
| `apps/web/models/User.ts` | Add `lowercase: true, trim: true` to the `email` field; make the unique index case-insensitive (collation `{ locale: 'en', strength: 2 }`). |
| `packages/db/src/users/schema/users.ts` + Drizzle migration | Normalize the Postgres email column (lowercase existing values; consider `citext` or a `lower(email)` unique index). |
| New MongoDB db-update migration (`apps/web/db-update/migration/`) | One-time data cleanup: lowercase all existing stored `email` values in the `users` collection. |

The two most important items are the **central read normalization** and the **one-time data migration**. The data migration is what actually fixes the reported user; everything else keeps behavior consistent going forward.

## 5. Duplicate audit (Step 1 of execution — start here, staging first)

Run in **staging first**, then production. The queries return only counts and non-identifying groupings so they can be run without exposing PHI — prefer the count/aggregate forms when sharing output. A zero result on the collision query means the data migration is safe to run.

### MongoDB (collection `users`)

List of accounts whose stored email is not already lowercase (the population the migration will touch). Returns the full documents so the email values are visible; the number of rows returned is the count:

```
db.users.find({
  deleted: { $ne: true },
  $expr: { $ne: ["$email", { $toLower: "$email" }] }
})
```

Case-collision groups — emails that become identical once lowercased but are stored with different case (the risky ones; must be resolved before migrating):

```
db.users.aggregate([
  { $match: { deleted: { $ne: true } } },
  { $group: {
      _id: { $toLower: "$email" },
      count: { $sum: 1 },
      variants: { $addToSet: "$email" },
      ids: { $addToSet: "$_id" }
  }},
  { $match: { $expr: { $gt: [ { $size: "$variants" }, 1 ] } } },
  { $sort: { count: -1 } }
])
```

### Postgres (table `users`)

List of non-lowercase stored emails (the number of rows is the count):

```sql
SELECT * FROM users WHERE email <> lower(email) AND deleted = false;
```

Case-collision groups:

```sql
SELECT lower(email) AS normalized, count(*) AS cnt
FROM users
WHERE deleted = false
GROUP BY lower(email)
HAVING count(*) > 1;
```

### Running these in the MongoDB Atlas Cloud UI (Data Explorer — no mongo shell)

The `db.users.*` snippets above are mongo-shell syntax and will not work in the `cloud.mongodb.com` web UI. In Atlas there is no shell; you use either the **Find** tab (a filter box) or the **Aggregation** tab (a pipeline builder). Navigate to your cluster, click **Browse Collections**, then select database `local-infusion-db` and collection `users`.

Important: an aggregation `$count` returns nothing (zero rows) when the matched set is empty. So "no result row" in the queries below means a count of zero — for the collision query that is the good outcome.

#### A. List accounts with a non-lowercase stored email (shows the emails)

Easiest: the **Find** tab. Paste this into the filter box; it lists the full documents, so you see each email, and the header shows how many matched:

```
{ "deleted": { "$ne": true }, "$expr": { "$ne": ["$email", { "$toLower": "$email" }] } }
```

If you prefer the **Aggregation** tab (text mode), a single `$match` stage returns the same documents:

```
[
  { "$match": { "deleted": { "$ne": true }, "$expr": { "$ne": ["$email", { "$toLower": "$email" }] } } }
]
```

Stage-by-stage (older UI): pick `$match`, paste `{ "deleted": { "$ne": true }, "$expr": { "$ne": ["$email", { "$toLower": "$email" }] } }`.

If you want just the number instead of the list, append a `$count` stage to the aggregation: `{ "$count": "nonLowercaseEmails" }` — the output `{ "nonLowercaseEmails": N }` is the count.

#### B. Case-collision groups (the risky ones) — full detail

**Aggregation** tab, text mode, paste the whole array:

```
[
  { "$match": { "deleted": { "$ne": true } } },
  { "$group": {
      "_id": { "$toLower": "$email" },
      "count": { "$sum": 1 },
      "variants": { "$addToSet": "$email" },
      "ids": { "$addToSet": "$_id" }
  } },
  { "$match": { "$expr": { "$gt": [ { "$size": "$variants" }, 1 ] } } }
]
```

Each output document is one collision group: `count` accounts, `variants` = the differently-cased email strings, `ids` = the account ObjectIds to resolve. No output = no collisions (safe).

Stage-by-stage equivalent (older UI): `$match` → `{ "deleted": { "$ne": true } }`; then `$group` → `{ "_id": { "$toLower": "$email" }, "count": { "$sum": 1 }, "variants": { "$addToSet": "$email" }, "ids": { "$addToSet": "$_id" } }`; then `$match` → `{ "$expr": { "$gt": [ { "$size": "$variants" }, 1 ] } }`.

#### C. Count of collision groups only (PHI-safe — no emails in output)

**Aggregation** tab, text mode:

```
[
  { "$match": { "deleted": { "$ne": true } } },
  { "$group": { "_id": { "$toLower": "$email" }, "variants": { "$addToSet": "$email" } } },
  { "$match": { "$expr": { "$gt": [ { "$size": "$variants" }, 1 ] } } },
  { "$count": "collisionGroups" }
]
```

Output `{ "collisionGroups": N }`, or no row at all if N is zero.

## 5c. Implementation steps (TDD)

Two stages, both shipped in the same hotfix so the read change and the disable land together (see the deployment-ordering note in Section 1b).

### Stage 1 — Case-insensitive email lookup (the login fix)

Approach: because we are deferring the "lowercase every stored email" data migration, do NOT rely on normalizing stored data. Instead make the lookup itself case-insensitive (a MongoDB collation `{ locale: 'en', strength: 2 }` on the query, and a `lower()`-based match on the Postgres side), and exclude disabled accounts at the query level. The `users` collection is small, so the collation collection-scan cost is negligible.

RED (write failing tests first):
- `getUserByEmail` returns the user when the queried email differs only in case from the stored value.
- `getUserByEmail` returns the **enabled** account, not the disabled one, when a case-variant pair both match — i.e. `disabled` is filtered in the query, not only in the post-fetch guard.
- `getUserWithPasswordHash` matches case-insensitively and still excludes `disabled`.
- `signInCallback` allows a user whose stored email case differs from the provider-supplied email; the `AUTHORIZED_EMAILS` / `AUDITOR_EMAILS` allow-list comparisons are case-insensitive.
- Postgres `findByEmailForRead` matches case-insensitively and excludes `disabled`.

GREEN (minimal implementation):
- `apps/web/services/mongodb/users.ts` — `getUserByEmail`: add `disabled: { $ne: true }` to the Mongo `findOne` filter and pass `{ collation: { locale: 'en', strength: 2 } }`. Keep the existing post-fetch `if (user.disabled) return null` guard as a belt-and-suspenders.
- `apps/web/services/mongodb/password.ts` — `getUserWithPasswordHash`: add the same collation to the `findOne` (it already filters `disabled` and `deleted`).
- `packages/db/src/users/repository.ts` — `findByEmailForRead`: change `eq(users.email, email)` to a case-insensitive match (`sql\`lower(${users.email}) = lower(${email})\`` or `ilike`) and AND in `eq(users.disabled, false)` (and keep `deleted = false`).
- `apps/web/app/api/auth/[...nextauth]/callbacks.ts` — lowercase both sides of the `AUTHORIZED_EMAILS.includes(...)` and `AUDITOR_EMAILS.includes(...)` comparisons.

REFACTOR: centralize the normalization/collation so Mongo and Postgres branches cannot drift.

### Stage 2 — Disable the 3 mixed-case duplicate accounts (migration, both stores)

The disable must be applied to Mongo AND Postgres, because reads can be served from either (`PG_READ_COMPLEX_2`). Ids are hardcoded (confirmed clean duplicates from the production reference audit):

```
699f1d9db3bff5f408cc73f3
69a0c0f0ac42a5b48e1cc985
69f8985a50ec5fb019e9662e
```

2a. **Mongo migration** — `apps/web/db-update/migration/044.disable-mixed-case-duplicate-users.mjs`, modeled on `030.add-is-service-account-to-users.mjs`:
- `up(session, db)`: `updateMany({ _id: { $in: [ObjectId(...x3)] } }, { $set: { disabled: true } }, { session })`; log `modifiedCount`.
- `down(db)`: set `disabled: false` on the same three `_id`s (audit confirmed all three were `disabled: false` before, so the rollback is exact).
- `runDBUpdater({ up, down })`. Hardcode the three ids as `ObjectId` literals.

2b. **Postgres migration** — a custom drizzle migration in `packages/db/migrations` (generated with `drizzle-kit generate --custom`, offline, per repo convention):
- The Postgres `users` table (confirmed from the live table definition) has its own `id bigserial` primary key and a **separate `mongo_id text` column** that stores the Mongo `_id` hex. So target rows by **`mongo_id`**, NOT `id`:
  - `UPDATE users SET disabled = true WHERE mongo_id IN ('699f1d9db3bff5f408cc73f3','69a0c0f0ac42a5b48e1cc985','69f8985a50ec5fb019e9662e');`
- `disabled` is a `bool` column. The up migration sets it **true** (disable the duplicates); the rollback sets it back to **false**.
- `email` is plain `text` with default (case-sensitive) collation — this is why the Stage 1 Postgres read needs a `lower()` match, not a plain `eq`.
- Drizzle migrations are forward-only; document the rollback SQL (`SET disabled = false WHERE mongo_id IN (...)`) in the migration file comment.

Ordering in the job: drizzle (Postgres) runs before the Mongo migration automatically, so both apply in one release. No separate run needed. Because migrations run before the new code rolls out, Stage 1 code and Stage 2 migration must each be backward-compatible with the currently deployed code (expand/contract) — disabling three unused duplicate accounts and adding a query collation both satisfy this.

Verification per environment (stage, then prod): confirm the three ids show `disabled: true` in both Mongo and Postgres, and that a case-insensitive login for one of the collision addresses resolves to the enabled lowercase (history-bearing) account, not the disabled duplicate.

Note: these hardcoded ids are production values, so the Mongo migration no-ops in staging (ids absent) and the Postgres `UPDATE` affects 0 rows there. Staging's own separate collision (one pair) is handled outside this migration.

## 6. Recommended execution order

> **SUPERSEDED.** The order of record is the two-stage flow in Section 5c: Stage 1 (case-insensitive read) + Stage 2 (disable-3 migration, Mongo + Postgres), built under TDD and shipped together in one release (see the deployment-ordering note in Section 1b). The steps below describe the original lowercase-everything approach and are kept for reference only — do not follow them.

1. **Run the duplicate audit in staging** (this session's starting point), then production.
2. Resolve any collisions manually.
3. Apply read-side normalization + write-side `lowercase: true` + case-insensitive unique index (Mongo and Postgres), under TDD.
4. Run the one-time data migration to lowercase stored emails.

## 7. Branch / deployment strategy

Hotfix deployed to both stage and production, developed in an isolated git worktree so the current `develop` working tree is left untouched.

- **Worktree:** do the work in a dedicated git worktree (a separate checkout), not the main working directory.
- **Base branch:** the hotfix branch is cut off **`main`** (production line), e.g. `hotfix/MLID-2822-case-insensitive-login`. Not off `develop`, not off an epic.
- **Two independent PRs from the same hotfix branch:**
  - hotfix branch → **`main`** (ships to production).
  - hotfix branch → **`develop`** (keeps develop in sync so the fix is not lost on the next release).
- The two PRs are independent — neither depends on the other merging first.
- Follow the usual workflow: prepare a PR draft document for each target rather than opening PRs automatically; the PRs are opened manually. Both PR titles/bodies reference MLID-2822.

## 8. Pre-implementation checklist (status)

Reflects the decided approach (case-insensitive read + disable the 3 mixed-case duplicates). Most items are resolved by the production audit and verification.

1. **Audit — RESOLVED.** Prod: 4 non-lowercase emails = 3 collision pairs (all scanned) + 1 standalone. Staging: 3 non-lowercase, 1 collision pair. Full results in the audit files.
2. **Disable ids + roles — RESOLVED (2026-07-16).** The 3 `mixedCase: true` ids match the migration's hardcoded ids; their lowercase survivors are confirmed; all 6 accounts are `role: user`, so disabling strips no elevated access.
3. **Postgres schema — RESOLVED.** The `users` table has `id bigserial` PK plus a separate `mongo_id text` link to Mongo; the disable migration targets `mongo_id`. `disabled` is `bool`; `email` is `text` with default (case-sensitive) collation, so the read uses a `lower()` match.
4. **Two-database read parity — handled, not a separate task.** `getUserByEmail` can read from Postgres via `PG_READ_COMPLEX_2`; parity is guaranteed by applying both the Stage 1 case-insensitive+disabled read change AND the Stage 2 disable to Postgres. Mongo remains the source of truth.
5. **Email-case consumers — accepted low risk.** Stored email data is NOT changed by this hotfix (the duplicates keep their mixed-case email, just disabled), so display / WeInfuse sync / notifications are unaffected.
6. **Standalone 4th account — no action required.** Fixed for login by Stage 1 (case-insensitive read matches it regardless of case). Lowercasing it is optional data hygiene only; it is never disabled.
7. **Deferred follow-up (separate ticket).** The full "lowercase every stored email + case-insensitive unique index" normalization is out of scope for this hotfix; track it separately.

## 9. Effort summary

| Item | Scope | Risk |
|------|-------|------|
| Stage 1 — case-insensitive read | `getUserByEmail` (collation + `disabled` in query), `getUserWithPasswordHash` (collation), Postgres `findByEmailForRead` (`lower()` match + `disabled`), allow-list lowercasing | Low |
| Stage 2 — disable migration | Mongo `044.disable-mixed-case-duplicate-users.mjs` (3 ids) + Postgres custom drizzle migration (`UPDATE ... WHERE mongo_id IN (...)`) | Low (3 empty, verified-clean duplicates; reversible) |
| Verification | Done — prod audit + id/role confirmation | — |
| Delivery | Worktree off `main`; two independent PRs (→ `main`, → `develop`) | Low |

Estimated 2–3 SP. Scope is narrower than the original plan (no lowercase-all data migration, no unique-index rebuild). The previously gating risk — case-variant duplicates — is retired: the duplicates are verified empty and all roles are `user`.
