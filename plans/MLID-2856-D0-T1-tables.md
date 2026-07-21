# MLID-2806 Stage 0 / D0-T1 (MLID-2856) — WeInfuse authorizations & notes tables

## Task Reference

- **Jira**: [MLID-2856](https://localinfusion.atlassian.net/browse/MLID-2856) — "[D0-T1] Create
  PostgreSQL tables for WeInfuse authorizations and notes"
- **Epic**: [MLID-2806](https://localinfusion.atlassian.net/browse/MLID-2806)
- **Parent plan**: `docs/agomez/plans/MLID-2806-weinfuse-authorizations.md`, Stage 0
- **Story Points**: not yet estimated
- **Branch**: none yet
- **Status**: **Draft — proposal for review.** Not approved, not started.

## Goal

Create the two PostgreSQL tables that receive the WeInfuse feeds — `authorizations` and its
child `authorization_notes` — together with their Drizzle schemas and one migration.

**Done when**: both tables exist in stage through the project's migration system, their Drizzle
schemas are exported from the data access package, and the shapes have been checked against full
report exports rather than samples.

**Not in this task**: the webhook endpoints, the transform-and-upsert layer, and the repositories
used to write rows. Those are D0-T2 (MLID-2857). This task delivers the receiving structures only.

The two tables are one Jira task but two feeds with different payloads: the authorizations feed
(Part A) and its child notes feed (Part B). Each has its own part below.

---

# Part A — `authorizations`

## A.1 Source payload

The Looker report delivers an array of flat objects. Keys are human-readable strings with
spaces, matching how the existing order feed is shaped.

The sample below is **synthetic** — structure and value types are faithful to the real export,
the values are invented. One representative row is shown; the properties that drive the design
are listed underneath, because a single row cannot display all of them.

```json
{
  "Authorizations ID": 1000001,
  "Authorizations Order Series ID": 1500001,
  "Order Series Patient ID": 1900001,
  "Authorizations Order of Auth": 1,
  "Authorizations Status": "Approved",
  "Authorizations Decision Date": "2026-01-15",
  "Authorizations Expiry Date": "2027-01-15",
  "Authorizations Treatments Approved": 12,
  "Authorizations Treatments Received": 3,
  "Authorizations Authorized Dosage": 500,
  "Authorization Insurances Insurance Plan Name": "Example Health Plan",
  "Authorizations Authorization Number": "AB12345678",
  "Authorizations Requested Date": "2026-01-10",
  "Authorizations Created Date": "2026-01-10",
  "Authorizations Updated Date": "2026-07-21"
}
```

Four properties of the real payload that drive the design. The single row above cannot show
them all, so they are stated as facts about the full export:

- **Nulls are common and meaningful.** Expiry date, treatments approved and authorized dosage
  are all null on many real rows, including approved ones. A denied row typically has a null
  expiry date, null treatments approved and null authorized dosage.
- **`0` is a real value** for treatments received, distinct from null.
- **Plan names arrive dirty** — leading and trailing whitespace, and backslashes (for example
  `" Example Regional Plan \\ COMM"`).
- **Date ordering is not reliable.** Decision date can precede requested date on real rows (for
  example decided `2026-03-20`, requested `2026-03-25`).

## A.2 Proposed table — `authorizations`

Table name decided: `authorizations`. No existing table in the data access package uses that
name or `authorization_notes`, so there is no collision today. One thing to keep in mind: LISA
has its own prior-authorization concepts (the Auth Letter Review), so anyone reading this table
name should understand it holds the **WeInfuse** authorizations feed specifically — the
`we_infuse_*` columns inside make the source explicit.

| Column | Type | Null | Source key | Notes |
|---|---|---|---|---|
| `id` | `bigserial` PK | no | — | Package convention |
| `we_infuse_authorization_id` | `integer` | no | Authorizations ID | Natural key of the feed |
| `we_infuse_order_series_id` | `integer` | yes | Authorizations Order Series ID | The join key. Not a foreign key — see A.3.1 |
| `we_infuse_patient_id` | `integer` | yes | Order Series Patient ID | Raw source value, always preserved |
| `patient_id` | `bigint` → `patients.id` | yes | *(resolved at write time)* | `ON DELETE SET NULL` |
| `auth_order` | `integer` | yes | Authorizations Order of Auth | Sequence within the order series |
| `status` | `text` | yes | Authorizations Status | Not an enum — see A.3.6 |
| `decision_date` | `date` | yes | Authorizations Decision Date | |
| `expiry_date` | `date` | yes | Authorizations Expiry Date | Null on denied rows |
| `requested_date` | `date` | yes | Authorizations Requested Date | |
| `treatments_approved` | `integer` | yes | Authorizations Treatments Approved | Null even on approved rows |
| `treatments_received` | `integer` | yes | Authorizations Treatments Received | `0` distinct from null |
| `authorized_dosage` | `numeric` | yes | Authorizations Authorized Dosage | Not integer — see A.3.5 |
| `we_infuse_authorization_number` | `text` | yes | Authorizations Authorization Number | Not numeric — see A.3.4 |
| `insurance_plan_name` | `text` | yes | Authorization Insurances Insurance Plan Name | Raw and unresolved — see A.3.7 |
| `source_created_at` | `date` | yes | Authorizations Created Date | WeInfuse's own dates, kept apart from ours |
| `source_updated_at` | `date` | yes | Authorizations Updated Date | Suspicious — see A.4.2 |
| `created_at` | `timestamptz` | no | — | Row insert time, defaults to now |
| `updated_at` | `timestamptz` | no | — | Row update time, defaults to now |

**No `mongo_id`.** This is a new table and the data access package's guidance forbids adding
one.

**Indexes**

| Index | Columns | Why |
|---|---|---|
| unique | `we_infuse_authorization_id` | Feed identity; upsert target |
| plain | `we_infuse_order_series_id` | Every downstream read joins on it |
| plain | `patient_id` | Patient-scoped reads in Stage 1 |
| plain | `we_infuse_patient_id` | Resolution and reconciliation passes |
| plain | `status` | Expected filter on both display surfaces |

Follow the package's existing index naming — `<table>_<columns>_idx`.

## A.3 Design decisions

### A.3.1 `we_infuse_order_series_id` cannot be a foreign key

It is the value everything joins on, but its target — `orders.order_series_id` — is deliberately
non-unique, because the same WeInfuse order arrives as more than one row. There is nothing
unique to reference. It stays a plain indexed integer, joined by value, exactly as
`lisa_orders.we_infuse_order_series_id` already is.

### A.3.2 Do not store a resolved LISA order

Adding a `lisa_order_id` FK alongside is tempting and wrong. A user can re-link or unlink an
order at any time from the patient Orders tab, and that action writes only to the `orders`
table — nothing would come back and correct a pointer stored here, so it would go stale
silently. The epic already decided the authorization-to-order relationship is a **read-time
join**; this column is how that decision would quietly get broken.

### A.3.3 Dates are date-only, so the columns are `date`

Every date in the payload is `YYYY-MM-DD`, with no time and no zone. Storing them as
`timestamptz` invents a midnight and a timezone, which is how authorizations end up displaying
one day off. This intentionally differs from the `orders` table, which uses `timestamptz`
because its source carries real timestamps. This feed does not.

### A.3.4 `we_infuse_authorization_number` is text

Values are alphanumeric and mixed case. A numeric type fails on the first delivery.

### A.3.5 `authorized_dosage` is `numeric`

Observed values are whole numbers, but dosage is a clinical quantity that is fractional in
general. `numeric` costs nothing now and avoids a migration the first time a half-unit arrives.

### A.3.6 `status` stays text, not an enum

The observed set is small, but the real set certainly includes pending, expired and cancelled
states, and it is WeInfuse-managed — we cannot control when it grows. The package uses enums
only where the complete value set was verified against real data. This does not qualify yet,
and may never.

### A.3.7 `insurance_plan_name` is stored raw

Values carry leading and trailing whitespace and backslashes. An `insurance_plans` table exists,
but matching on a dirty display name would be guesswork. Store the raw string; a resolved
foreign key can be added later if matching proves reliable.

### A.3.8 Two-column rule for the patient

Raw `we_infuse_patient_id` always preserved, plus a resolved `patient_id` foreign key that is
null when the patient is unknown. This mirrors how `orders` handles the same problem, keeps
resolution re-runnable, and answers the epic's open question about authorizations arriving for
patients LISA does not have: **store the row, leave the foreign key null.**

## A.4 Verify before building

### A.4.1 Confirm the natural key against a full export

`Authorizations ID` looks unique and the proposal enforces a unique index on it. The `orders`
table teaches the opposite lesson — its WeInfuse identifier is non-unique because report joins
fan out into duplicate rows. If this report joins to anything that fans out, duplicates will
appear here too.

Run the full export and check for repeated identifiers before committing to the constraint.
Enforcing it and letting it fail loudly is the safer choice, **provided** the T2 ingest
deduplicates first.

### A.4.2 Establish what `Authorizations Updated Date` means

In the real export this value is identical on every row and equals the export date. Either every
authorization genuinely changed that day, or the column is the report run date rather than a
modification timestamp.

This matters beyond curiosity: a real modification timestamp is a natural delta cursor for the
feed; an export date is useless for that and mildly misleading to store. Ask whoever configures
the report.

### A.4.3 No CHECK constraints on date ordering

Decision dates precede requested dates on real rows. Do not add constraints assuming an order,
and do not write read logic that assumes one.

### A.4.4 Confirm nullability against the full export

The nullability above is inferred from a small sample. Before the migration is written, confirm
against a full export which columns are genuinely never null. Anything uncertain stays nullable
— widening a column later is a cheap migration, and a false `NOT NULL` rejects real deliveries.

---

# Part B — `authorization_notes`

Child of `authorizations`. One note belongs to one authorization; an authorization has a history
of many notes.

## B.1 Source payload

Two shape differences from the authorizations feed matter immediately:

- **Every value is a string**, including the identifiers. The authorizations feed delivered
  identifiers as JSON numbers; this feed quotes them (`"5169752"`). The transform must parse
  them.
- **No patient id.** Unlike the authorizations feed, this payload carries no patient identifier
  at all. The patient is reached only through the parent authorization — see B.3.6.

The feed provides two distinct authorization references per note:

- **`Authorization Review Notes Parent ID`** — the authorization the note is written against.
  This is the real parent and the FK source.
- **`Latest Authorization ID`** — the most recent authorization in the same order series. It is
  contextual, not the note's parent. In the sample rows the two happen to be equal, but they are
  different fields and can diverge once a series has more than one authorization.

The sample below is **synthetic**. Field structure, string typing, embedded quotes, newlines and
URLs are faithful to the real export; all values — identifiers, note text, author names — are
invented. **The real notes contain PHI** (patient references, appointment dates, authorization
numbers, Spruce message links) and are deliberately not reproduced here.

```json
{
  "Authorization Review Notes ID": "5100001",
  "Authorization Review Notes Parent ID": "1000001",
  "Latest Authorization ID": "1000001",
  "Latest Authorization Order Series ID": "1500001",
  "Authorization Review Notes Created Time": "2026-07-09 15:28:06",
  "Authorization Review Notes Note": "updated authorization to \"Approved\"\n\nDOS 0/0/00-0/0/00, quantity approved was 1, asked for that to be updated to 4.\nhttps://example.com/thread/[id]",
  "Authorization Review Notes User Name": "Demo User"
}
```

Properties of the real payload that drive the design. The single row cannot show them all, so
they are stated as facts about the full export:

- **A real note identifier exists.** `Authorization Review Notes ID` is the note's own unique id.
  This is the natural key — the note table upserts on it, and no surrogate is needed (contrast
  the earlier assumption; see B.3.1).
- **Two authorization references.** Parent ID (the note's authorization) and Latest Authorization
  ID (latest in series), as described above.
- **The note body is free text**, multi-line, hand-typed. It can contain **embedded double
  quotes**, blank lines (`\n\n`), and URLs. It is not reliably structured; store it whole.
- **Whitespace is dirty** inside the body — doubled spaces and stray line breaks are common, the
  same quirk seen in the plan names on the authorizations feed.
- **The created time carries a real time of day**, unlike the date-only authorizations feed.

## B.2 Proposed table — `authorization_notes`

| Column | Type | Null | Source key | Notes |
|---|---|---|---|---|
| `id` | `bigserial` PK | no | — | Package convention |
| `we_infuse_note_id` | `integer` | no | Authorization Review Notes ID | The note's own WeInfuse id. Natural key; upsert target — see B.3.1 |
| `authorization_id` | `bigint` → `authorizations.id` | yes | *(resolved at write time)* | Internal FK. `ON DELETE SET NULL`. Resolved from `we_infuse_authorization_id` — see B.3.3 |
| `we_infuse_authorization_id` | `integer` | no | Authorization Review Notes **Parent ID** | The parent authorization's WeInfuse id — the WeInfuse-side link and the resolution source for `authorization_id`. Always preserved — see B.3.3 |
| `we_infuse_latest_authorization_id` | `integer` | yes | Latest Authorization ID | Latest authorization in the series. Context only, not the parent — see B.3.4 |
| `we_infuse_order_series_id` | `integer` | yes | Latest Authorization Order Series ID | Carried for the same join everything else uses |
| `note` | `text` | yes | Authorization Review Notes Note | Stored whole, not parsed — see B.3.2 |
| `author_name` | `text` | yes | Authorization Review Notes User Name | LISA user who wrote the note; a plain name string, not a resolvable id — see B.3.7 |
| `note_created_at` | `timestamptz` | yes | Authorization Review Notes Created Time | Real time of day — zone must be established, see B.3.5 |
| `created_at` | `timestamptz` | no | — | Row insert time, defaults to now |
| `updated_at` | `timestamptz` | no | — | Row update time, defaults to now |

**No patient columns.** The feed carries no patient id; patient is reached through the parent
authorization (`authorization_id` → `authorizations.patient_id`). See B.3.6.

**No `mongo_id`.** New table; the data access package's guidance forbids it.

**Indexes**

| Index | Columns | Why |
|---|---|---|
| unique | `we_infuse_note_id` | Feed identity; upsert target |
| plain | `authorization_id` | The parent lookup — fetch a note's authorization / a note history for an authorization |
| plain | `we_infuse_authorization_id` | Resolution and reconciliation before the FK is resolved; WeInfuse-side join |
| plain | `we_infuse_order_series_id` | Order-scoped reads in Stage 2 |

## B.3 Design decisions

### B.3.1 `we_infuse_note_id` is the natural key

The feed provides the note's own identifier (`Authorization Review Notes ID`), so the table
upserts on it directly — one unique index, re-deliveries are idempotent, and no content hash or
composite key is needed. (An earlier draft of this plan, written before the corrected payload,
assumed there was no note id and proposed a content-hash surrogate; that is no longer required.)

Confirm across a full export that the id is genuinely unique — the same lesson as the
authorizations feed (A.4.1): if the report joins to anything that fans out, one note id could
appear on several rows and the unique index would reject the batch. Enforcing the constraint and
letting it fail loudly is the safer choice, provided the D0-T2 ingest deduplicates first.

### B.3.2 The note body is stored whole, and sanitized on display

The body is hand-typed free text — sometimes a short sentence, sometimes several lines with
embedded quotes and URLs. It has no reliable structure to parse into columns, and doing so would
be fragile and lossy. Store the raw text as delivered; the ingest treats it as opaque text and
never tokenizes it.

Because it is untrusted free text that later stages will render in the UI, **sanitization is a
display concern, not a storage one.** Keep the stored value faithful to the source, and escape /
sanitize it at the point of rendering so embedded markup, quotes or URLs cannot inject into the
page. Sanitizing before storage would corrupt the record and is not this table's job — the
Stage 1 and Stage 2 display tasks own the render-time sanitization.

### B.3.3 Two links to the authorization — internal FK and WeInfuse reference

Each note carries both references to its parent authorization, as the epic requires:

- **`authorization_id`** — the internal Postgres foreign key to `authorizations.id`, resolved at
  write time from the parent's WeInfuse id. Nullable, `ON DELETE SET NULL`. **This is the id the
  rest of LISA joins on** — every table in this epic links by internal Postgres id, and the
  WeInfuse ids are kept only as the source values that resolution maps from.
- **`we_infuse_authorization_id`** — the parent authorization's WeInfuse id, taken from
  `Authorization Review Notes Parent ID`. Always stored, so resolution stays re-runnable.

**Working assumption (unverified): the parent is `Authorization Review Notes Parent ID`, not
`Latest Authorization ID`.** The two are equal in every sample row, so the sample cannot settle
which one is the true link, and Looker's report model does not expose the underlying table
relationship the way a SQL schema would — so this cannot be read off the payload with certainty.
The guess is Parent ID, because the name says "parent" and it reads as the note's own
authorization. Proceed on that basis, but treat it as a decision to confirm (B.4) before the
ingest is trusted; if it turns out `Latest Authorization ID` is the real link, the FK source
flips to that column and `we_infuse_latest_authorization_id` / `we_infuse_authorization_id` swap
roles.

`authorizations.id` is unique, so the internal FK is genuine and correct — unlike
`we_infuse_order_series_id`, whose target is non-unique.

Resolution follows the two-column rule: keep the raw `we_infuse_authorization_id` always, and set
`authorization_id` when the parent exists. The internal FK is null when the parent authorization
is not yet in the table — an ordinary timing case: the notes feed runs before the authorizations
feed, or references an authorization the authorizations report has not delivered yet.
`ON DELETE SET NULL` and a re-runnable resolution keep this recoverable.

**One decision to confirm — whether `we_infuse_authorization_id` is a hard FK constraint.**
`authorizations.we_infuse_authorization_id` is unique, so Postgres *could* enforce a foreign key
on it. This plan deliberately does **not**, for two reasons that both stem from the raw column
being written unconditionally: a hard constraint would reject any note whose parent authorization
has not been ingested yet, and it would forbid storing the raw parent id at all when the parent
is missing — defeating the re-runnable resolution above. Keeping it an indexed reference (not a
constraint) lets notes arrive in any order and always preserves the WeInfuse-side link. If the
D0-T2 ingest can guarantee the authorizations feed always lands first and every parent exists, it
could be promoted to a hard FK; until then, indexed reference is the safe default.

### B.3.4 `we_infuse_latest_authorization_id` — the other candidate, stored as context

`Latest Authorization ID` reads as the most recent authorization in the note's order series —
a different thing from the authorization the note was written against, which we are assuming is
`Parent ID` (B.3.3). Under that assumption it is context, not the parent link: stored because the
feed provides it and it may matter for "which authorization is currently active", but not the FK
source and not joined on for parentage.

The caveat: this reading is the flip side of the B.3.3 guess. If the confirmation there shows
`Latest Authorization ID` is actually the note's authorization, this column becomes the FK source
instead. Until then, both WeInfuse ids are stored, so whichever way the answer lands the ingest
can resolve the FK without a re-migration — only the resolution source changes.

### B.3.5 `note_created_at` — establish the source time zone before ingesting

Unlike the date-only authorizations feed, these timestamps carry a real time of day
(`2026-07-09 15:28:06`) and no zone. Storing them without knowing the source zone risks an
off-by-hours error on every note. Confirm what zone WeInfuse emits, and have the transform
localize on the way in so the stored `timestamptz` is correct. This is the most important verify
item for this table.

### B.3.6 Patient is reached through the authorization, not stored on the note

The feed has no patient identifier, so the table has no patient column. A note's patient is its
parent authorization's patient (`authorization_id` → `authorizations.patient_id`). Patient-scoped
reads in Stage 1 therefore join notes to authorizations rather than filtering notes directly.
This is a consequence of the payload, not a choice — there is simply nothing to store. It also
means a note whose parent has not resolved yet has no reachable patient until resolution runs.

### B.3.7 `author_name` stays a plain string

The note author is a LISA user, but the feed gives only a display name, not an id. Matching it to
the users table by name would be unreliable. Store the string; a resolved user reference could be
added later if it proves worthwhile, following the same raw-plus-resolved pattern used elsewhere.

## B.4 Verify before building

- **Is `Authorization Review Notes ID` unique across a full export?** Gates the unique index
  (B.3.1).
- **Which of Parent ID / Latest Authorization ID is the note's authorization?** The working
  assumption is Parent ID (B.3.3), but this is a guess — the sample cannot settle it and Looker
  hides the relationship. Confirm before the ingest is trusted, e.g. by finding an export row
  where the two diverge and checking which one points at the authorization the note belongs to,
  or by asking whoever builds the Looker report what each field joins on.
- **What time zone are the created times in?** (B.3.5).
- **Do all `Parent ID` values exist in the authorizations feed?** If a large fraction never
  resolve, the join is weaker than expected and worth understanding before Stage 1 relies on it.
- **Are notes ever edited after creation?** Less critical now that the note id is the key (an
  edit updates the same row), but still worth knowing for the `updated_at` semantics.

---

## Implementation steps (both tables)

1. **Get full exports of both reports** and settle the verify items — for `authorizations`,
   A.4.1, A.4.2 and A.4.4; for `authorization_notes`, the note-id uniqueness (B.3.1), the
   Parent-vs-Latest divergence (B.3.4) and the created-time zone (B.3.5).
2. **Read the reference material first**: the data access package's own guidance file, and one
   existing parent+child schema pair with its foreign key and repository as a shape to copy.
3. **Create both Drizzle schemas**, each in its own entity folder, with `authorization_notes`
   carrying the `authorization_id` foreign key to `authorizations` (resolved from Parent ID).
4. **Export both from the schema barrel** so migration generation sees them.
5. **Generate the migration** (offline, no database connection needed). Known cross-branch
   hazard: migration numbers collide when two branches generate in parallel — regenerate rather
   than hand-edit the number.
6. **Review the generated SQL by hand** before it is applied — types, nullability, the unique
   indexes, and the foreign key action in particular.
7. **Apply to stage** and confirm both tables, the foreign key and the indexes exist as intended.

The repositories that write into these tables are deliberately not part of this task; they arrive
with the D0-T2 ingest so their methods are shaped by real usage.

## Open questions (both tables)

**`authorizations`**
- Is `Authorizations ID` unique across a full export? Gates the unique index (A.4.1).
- Is `Authorizations Updated Date` a real modification timestamp or the report run date (A.4.2)?
- Which columns are genuinely never null across a full export (A.4.4)?
- Does the feed ever delete authorizations, or only add and update them? Determines whether the
  table needs a soft-delete or active marker, which the sample gives no basis to decide.
- Does `Authorizations Order of Auth` restart per order series, and can two rows in one series
  share a value?

**`authorization_notes`**
- Is `Authorization Review Notes ID` unique across a full export? Gates the unique index (B.3.1).
- Should `we_infuse_authorization_id` be a hard FK constraint on
  `authorizations.we_infuse_authorization_id`, or stay an indexed reference? Depends on whether
  the ingest can guarantee the parent authorization is always loaded first (B.3.3).
- Do Parent ID and Latest Authorization ID ever diverge, confirming Parent ID as the FK source
  (B.3.4)?
- What time zone are the note created times in (B.3.5)?
- Are notes ever edited after creation (affects `updated_at` semantics)?
- Should the note author be resolved to a user reference, or is the display name enough?
