# [MLID-2822] Email case-sensitivity audit

Duplicate / case audit for the login case-insensitivity hotfix. PHI suppressed: email addresses and names are intentionally omitted. Account ObjectIds are kept (not PHI, and needed to locate records for manual resolution). The email domain shown is the shared company domain, not a personal identifier.

## Environment

| | |
|---|---|
| Environment | **Staging** |
| Database | `local-infusion-db` |
| Collection | `users` |
| Date run | 2026-07-15 |
| Production audit | **NOT yet run** — must be run separately before implementation (prod DB access is held by the senior engineer). Same queries below. |

## Queries used (read-only)

Count of accounts whose stored email is not already lowercase:

```
db.users.countDocuments({
  deleted: { $ne: true },
  $expr: { $ne: ["$email", { $toLower: "$email" }] }
})
```

Case-collision groups (emails identical once lowercased but stored with different case):

```
db.users.aggregate([
  { $match: { deleted: { $ne: true } } },
  { $group: {
      _id: { $toLower: "$email" },
      count: { $sum: 1 },
      variants: { $addToSet: "$email" },
      ids: { $addToSet: "$_id" }
  }},
  { $match: { $expr: { $gt: [ { $size: "$variants" }, 1 ] } } }
])
```

## Findings (staging)

- **3** active accounts have a stored email that is not all-lowercase (the population the data migration would rewrite).
- **1** case-collision group (2 accounts that collapse to the same email once lowercased).

### Accounts with non-lowercase stored email (3)

| Account id | Role | Disabled | Auth type | Domain | In collision? |
|------------|------|----------|-----------|--------|---------------|
| `696019617fd0f57284c5b120` | user | no | Google OAuth (no password) | mylocalinfusion.com | No — safe to lowercase |
| `69d51fec6ec108bf68071efa` | user | no | Password (has hash) | mylocalinfusion.com | No — safe to lowercase |
| `6a1f2343b2a12cd1dd4ad23b` | admin | no | Google OAuth (no password) | mylocalinfusion.com | **Yes** — see collision below |

Two of these three lowercase cleanly (no other account shares their lowercased email). The third is one half of the collision group.

### Collision group (1) — REQUIRES MANUAL RESOLUTION

Two active accounts share the same email address ignoring case, on domain `mylocalinfusion.com`:

| Account id | Role | Disabled | Auth type | Stored email case |
|------------|------|----------|-----------|-------------------|
| `6a1f2343b2a12cd1dd4ad23b` | **admin** | no | Google OAuth (no password) | mixed-case |
| `6a1f279ab2a12cd1dd4ae16a` | **user** | no | Google OAuth (no password) | already lowercase |

Once emails are lowercased, both records resolve to the identical email. This would (a) violate the new case-insensitive unique index at migration time, and (b) before that, make login ambiguous between an **admin** account and a **user** account — an access-level change, not just a cosmetic one.

## Required action before the data migration

1. A human (PO / senior) must decide which of the two collision accounts is the account of record, and disable / delete / merge the other. The admin-vs-user role difference makes this an access decision, not a mechanical one.
2. Re-run the collision query in **production** and resolve any collisions found there the same way.
3. Only after each environment's collisions are cleared is the lowercase data migration safe to run in that environment.

## Notes

- The `users` collection holds internal staff / application accounts, not patient records; these are employee emails rather than patient PHI. Addresses and names are still suppressed here as a precaution.
- All three non-lowercase accounts and both collision members are on the internal `mylocalinfusion.com` domain, consistent with staff accounts.

## Verification queries — items 1 and 2 (confirm ids + roles before the disable migration)

Run in the production **Aggregation** tab on the `users` collection. This one query returns every collision pair with the **full member documents**, so you get in a single result: each account's `_id` (verify the mixed-case ids to disable), its `email` (confirm which member is the mixed-case one), its `role` (item 2 — admin vs user), and `disabled`. No output means no collisions.

```
[
  { "$match": { "deleted": { "$ne": true } } },
  { "$addFields": { "emailLower": { "$toLower": "$email" } } },
  { "$group": {
      "_id": "$emailLower",
      "variants": { "$addToSet": "$email" },
      "members": { "$push": "$$ROOT" }
  } },
  { "$match": { "$expr": { "$gt": [ { "$size": "$variants" }, 1 ] } } }
]
```

For each group in the output: the member whose `email` contains uppercase letters is the one to disable (its `_id` must match one of the three hardcoded migration ids); the lowercase member is the one to keep. Compare the two members' `role` — if the mixed-case (to-be-disabled) member is `admin` while the lowercase survivor is `user`, raise the survivor's role before disabling.

To verify a single account by id instead, use the **Find** tab: `{ "_id": { "$oid": "<id>" } }` (returns the whole document, including `role` and `email`).

## Resolution — 2026-07-16 (production)

The three mixed-case duplicate accounts were **disabled** (`disabled: true`) in production after the MLID-2822 case-insensitive code shipped — applied manually, no migration:

- `699f1d9db3bff5f408cc73f3`
- `69a0c0f0ac42a5b48e1cc985`
- `69f8985a50ec5fb019e9662e`

Their lowercase twins stay enabled and now match at login. The detailed per-pair record (twin ids, footprint) is in `MLID-2822-user-reference-scan.md`.
