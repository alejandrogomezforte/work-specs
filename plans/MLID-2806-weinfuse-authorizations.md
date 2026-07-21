# MLID-2806 — Bring WeInfuse authorizations into LISA and link them to orders

## Task Reference

- **Jira**: [MLID-2806](https://localinfusion.atlassian.net/browse/MLID-2806)
- **Issue type**: epic — this ticket is itself the epic, no parent
- **Story Points**: not yet estimated
- **Branches**: none yet
- **Status**: **Stage 0 ready to detail.** Stages 1–3 deliberately high level.

### How this document works

This plan is written **progressively**, not end to end. The epic touches five areas that are
new to this work — Looker webhooks, PostgreSQL tables, the Drizzle data access layer,
Next.js API routes, and Next.js user interface components — so detailing all four stages up
front would produce guesses that later stages would have to undo.

The rule for this document:

- Every stage has a section from day one, at the altitude of "what it is and what it depends
  on".
- **Only the stage being worked on is detailed.** Detail is added to its section when that
  stage starts, not before.
- Stages are worked **sequentially**. A stage is not detailed until its predecessor has
  landed, because each stage's design depends on what the previous one actually produced
  rather than what it was expected to produce.
- Nothing is deleted as the epic progresses. A finished stage keeps its detail and gains an
  outcome note, so the document doubles as the implementation history.

Superseded by nothing; supersedes the pre-plan at `docs/agomez/preplans/MLID-2806.md`, whose
content is carried into this document. That file is kept for reference but is no longer the
working artifact.

---

## 1. The problem (from the ticket)

Infusion Guides record authorizations in WeInfuse, but LISA cannot see them. Today the only
authorization information inside LISA is the Auth Letter Review section, and the
"Authorization Submission Date" on a New Order is typed in by hand — an extra manual step
that can be wrong or forgotten.

The epic as written by the Product Owner describes the work as three user interface
deliverables. That framing assumes the data is already inside LISA, which it is not. Moving
the authorizations and their notes from WeInfuse into LISA is real engineering work with its
own storage, its own entry points and its own failure modes. It is therefore a stage of its
own — **Stage 0** — and the three original deliverables are reduced to display and behaviour
only.

A further piece — showing authorization changes in Order History — is named in the ticket
only as context and is tracked in a different epic.

## 2. Decided approach

Two facts change the shape of the epic relative to how it was written.

**The transfer mechanism already exists in the product.** Looker reports run against WeInfuse
and call LISA webhooks, which insert the rows into LISA tables. Stage 0 replicates that
pattern for two new feeds: authorizations, and authorization notes.

**The order linking already exists.** On the patient record's Orders tab a user can already
link a LISA order to a WeInfuse order series, and the chosen series identifier is stored on
the LISA order itself. The epic's phrase "this will come from the Order Linking" points at
this existing capability. Nothing in Stages 1–3 has to build it.

Together these mean the authorization-to-order relationship is a **read-time join** on a
value that is already stored. No new write path, no synchronisation job, and no code reacting
to the three "triggers" the ticket lists — a link or an unlink is simply reflected the next
time the data is read.

**PostgreSQL only.** Decided with the Principal Engineer: no new MongoDB structures are
created for this epic. The PostgreSQL migration is close to finished and MongoDB is expected
to be dropped around the time this epic is delivered, so anything new born in MongoDB would
be migration debt on the day it ships.

## 3. Stage order and dependencies

| Stage | Deliverable | Depends on | Note |
|---|---|---|---|
| Stage 0 | Data transfer from WeInfuse to LISA | nothing | Can start immediately. Root of everything else. |
| Stage 1 | Authorizations on the patient record | Stage 0 | Nothing to display until the authorizations are in LISA. |
| Stage 2 | Authorizations on the New Order | Stage 0, Stage 1 read layer | Reuses Stage 1's read layer, table and notes modal. |
| Stage 3 | System-driven Authorization Submission Date | Stage 0, Stage 1 read layer | Needs the join only, not Stage 1's interface. Can run beside Stage 2. |

Seeded data is an accepted option, not a decision to make now: if it helps, Stage 1 may be
built against seeded records before Stage 0 is verified in an environment.

### Jira task map

The epic's sub-tasks already exist in Jira. Each Dn-Tm pair in this document maps to one task
under epic MLID-2806. The `[Dn-Tm]` prefix in each Jira summary matches the label used here.

| Stage | Dn-Tm | Jira | Summary |
|---|---|---|---|
| Stage 0 | D0-T1 | [MLID-2856](https://localinfusion.atlassian.net/browse/MLID-2856) | Create PostgreSQL tables for WeInfuse authorizations and notes |
| Stage 0 | D0-T2 | [MLID-2857](https://localinfusion.atlassian.net/browse/MLID-2857) | Create webhooks to receive the WeInfuse authorization and notes feeds |
| Stage 0 | D0-T3 | [MLID-2858](https://localinfusion.atlassian.net/browse/MLID-2858) | Set up the WeInfuse Looker reports and point them at the new webhooks |
| Stage 0 | D0-T4 | [MLID-2859](https://localinfusion.atlassian.net/browse/MLID-2859) | Run the one-time historical load of WeInfuse authorizations and notes |
| Stage 1 | D1-T1 | [MLID-2860](https://localinfusion.atlassian.net/browse/MLID-2860) | Read layer and API for a patient's WeInfuse authorizations |
| Stage 1 | D1-T2 | [MLID-2861](https://localinfusion.atlassian.net/browse/MLID-2861) | WeInfuse Auths tab on the patient record |
| Stage 1 | D1-T3 | [MLID-2862](https://localinfusion.atlassian.net/browse/MLID-2862) | Update the patient record header |
| Stage 2 | D2-T1 | [MLID-2863](https://localinfusion.atlassian.net/browse/MLID-2863) | Order-scoped read and API for an order's WeInfuse authorizations |
| Stage 2 | D2-T2 | [MLID-2864](https://localinfusion.atlassian.net/browse/MLID-2864) | WeInfuse Auths tab on the New Order |
| Stage 3 | D3-T1 | [MLID-2865](https://localinfusion.atlassian.net/browse/MLID-2865) | Make the Authorization Submission Date system generated |

There is also a grooming task, [MLID-2807](https://localinfusion.atlassian.net/browse/MLID-2807)
("Grooming - Deliverable #1"), which is not one of the Dn-Tm work items.

---

# Stage 0 — Data transfer from WeInfuse to LISA

**Status**: ready to detail — this is the current stage.
**Type**: backend / data pipeline. No user interface.

## 0.1 Goal

Two new PostgreSQL tables receiving two new Looker feeds — authorizations, and the notes that
belong to them — with the authorizations carrying the WeInfuse order series identifier that
every later stage joins on.

**Done when**: both feeds deliver on their cadence into their tables, the historical load has
run, and a patient's authorizations can be retrieved from PostgreSQL with their notes and
their order series identifier intact.

## 0.2 Tasks

| Task | Jira | What it covers |
|---|---|---|
| D0-T1 | [MLID-2856](https://localinfusion.atlassian.net/browse/MLID-2856) | Design and create the two PostgreSQL tables that receive the authorizations and the authorization notes. |
| D0-T2 | [MLID-2857](https://localinfusion.atlassian.net/browse/MLID-2857) | Design and create the two webhooks that receive the report payloads and write them into those tables, logging success and errors. |
| D0-T3 | [MLID-2858](https://localinfusion.atlassian.net/browse/MLID-2858) | Set up the two Looker reports against WeInfuse and point them at the new webhooks on the required cadence. |
| D0-T4 | [MLID-2859](https://localinfusion.atlassian.net/browse/MLID-2859) | Run the one-time full load of all authorizations and notes already in WeInfuse. |

## 0.3 Sequencing — read before scheduling

The task numbering is not the execution order.

**The tables cannot be designed until the payload is known.** T1 is written as "design and
create the tables", but the columns come from whatever the Looker report emits. The existing
orders feed receives flat rows with dotted keys, and the exact key set is a property of the
report, not something this side chooses. Until there is a column list — ideally a real sample
delivery — table design is guesswork.

**T3 is partly external and has the longest lead time.** Someone configures the reports on the
WeInfuse side. That conversation both unblocks T1 and is the item most likely to sit waiting
on another person, so it should start first even though the ticket lists it third.

Suggested order: agree the payload contract → T1 tables and schemas → T2 ingest paths → T4
historical load, with T3's configuration running in parallel from the beginning.

## 0.4 Reference implementation

The `looker_orders` webhook is the closest existing example and should be read before
designing the new ones. It brings WeInfuse orders into LISA and is documented in
`docs/agomez/analysis/weinfuse-orders-overview.md`. Its shape:

- An endpoint per feed under the webhooks API folder, authenticated by a shared token supplied
  in a request header and compared against an environment secret.
- A thin handler that does nothing but delegate, plus optional payload archiving behind a
  feature flag for troubleshooting.
- A separate transform-and-upsert utility holding all the real logic. This is where a
  replicated webhook spends its effort.
- A persistence entry point handing prepared rows to the repository in the data access
  package.

Five behaviours in that utility are worth carrying over, because each exists to fix a real
incident or data quirk:

1. **Delta, not snapshot.** An empty payload is a no-op. Treating absence as "no longer
   active" once mass-deactivated thousands of order rows from a single empty delivery. Any
   active/inactive state must be derived from a field in the payload, never from presence in
   it.
2. **Merge, do not replace.** A report delivery is a partial payload — fields arriving empty
   are deliberately skipped so stored values survive. A blind overwrite nulls columns on every
   run.
3. **Deduplicate before writing.** Report joins fan out: one logical record can arrive as
   several rows differing only in the joined columns. A single upsert statement cannot touch
   the same row twice, so duplicates must be collapsed and logged.
4. **Resolve foreign keys at write time.** Incoming rows carry WeInfuse identifiers, not LISA
   ones. The repository resolves them to the parent table's identifier and stores null when
   the parent is unknown, keeping the raw value so resolution can be re-run later.
5. **Fail loudly.** A persistence failure is re-thrown so the endpoint answers non-2xx. A
   swallowed failure reads as a healthy sync while nothing is being written.

The authorizations feed is the closer analogue of the two, since it carries the order series
identifier everything downstream joins on. The notes feed is a child feed and mainly needs its
parent resolved, which is behaviour 4.

## 0.5 Data

| Table | Action | Why |
|---|---|---|
| WeInfuse authorizations (new, PostgreSQL) | create | Receives the authorizations feed. Carries the order series identifier used for the join. |
| WeInfuse authorization notes (new, PostgreSQL) | create | Receives the notes feed. Each note belongs to one authorization. |

**Row identity.** Each table needs a natural key the feed can upsert on, decided in T1. Two
warnings from the existing orders feed. First, the older tables carry a transitional
identifier column inherited from the MongoDB migration; the data access package's own guidance
says never to add one to a new table, so these two tables must not have it. Second, the
WeInfuse identifier on the existing orders table is deliberately *not* unique, because the
same WeInfuse record can arrive as more than one row — so a WeInfuse identifier cannot be
assumed unique without checking the actual feed first.

## 0.6 Code

| Area | Action | Why |
|---|---|---|
| Webhook entry point for authorizations | create | Receive the authorizations feed. Modelled on the existing `looker_orders` endpoint, including its shared-token header check. |
| Webhook entry point for authorization notes | create | Same, for the notes feed. |
| Handler per feed | create | Thin delegation layer, mirroring `handleLookerOrders`. Optional payload archiving behind a flag. |
| Schemas and repositories for the two new tables | create | Drizzle schemas in the `packages/db` data access package, each with its own repository, following that package's own guidance file. |
| Ingestion service layer | create | Transform the report rows, deduplicate, resolve foreign keys and upsert. The equivalent of `webhookUpsertOrders`, and where the bulk of T2 lives. |

**Where the unfamiliarity will cost most**: the Drizzle schema conventions in `packages/db`
and the upsert semantics. The endpoint and handler are close to boilerplate and can follow
`looker_orders` almost directly. New webhooks go wrong in the transform-and-upsert layer —
the five behaviours above. Front-load a careful reading of `packages/db/CLAUDE.md` and one
existing repository.

## 0.7 Deploy

The two tables are created through the project's migration system, the two entry points must
be deployed and reachable, the two Looker reports configured on the WeInfuse side, and the
historical load run once. Part of that work is outside this repository.

Following the existing Looker webhooks, each endpoint authenticates against a shared token
held in an environment variable. Those are new environment variables and must be wired into
Azure **before** the reports are pointed at the endpoints — otherwise every delivery is
rejected. Whether the two feeds share one token or take one each is still open.

## 0.8 Open questions — Stage 0

- What does the authorizations report actually emit, and the notes report? Blocking for T1.
- What is the natural key of each table, given a WeInfuse identifier may not be unique?
- What happens when an incoming authorization references a patient or an order series LISA
  does not have? — options: store unlinked and let a later link resolve it / reject and report
  / quarantine. Precedent: the existing orders feed stores the row with a null link and keeps
  the raw WeInfuse value, so resolution can be re-run later.
- Does a repeated feed run replace existing rows or append? The existing orders feed upserts
  and merges rather than replacing, and that is the safer default.
- Does the required refresh cadence describe how often the reports run, or how often the
  interface re-reads what is stored?
- Who configures the Looker reports, and on what cadence?
- One shared token for both feeds, or one each? Who wires them into Azure?
- Is the historical load a script run once by an engineer, or a re-runnable job exposed in the
  admin jobs interface?
- What is the deploy order between the migration, the entry points, the historical load and
  the later interface releases?

## 0.9 Detail

Task-level detail lives in its own file once a task is being worked, so this section stays
readable as the stage overview.

| Task | Jira | Detail document | Status |
|---|---|---|---|
| D0-T1 — both tables | [MLID-2856](https://localinfusion.atlassian.net/browse/MLID-2856) | `docs/agomez/plans/MLID-2856-D0-T1-tables.md` | Draft, proposal for review |
| D0-T2 — webhooks and ingest | [MLID-2857](https://localinfusion.atlassian.net/browse/MLID-2857) | not yet | — |
| D0-T3 — Looker reports | [MLID-2858](https://localinfusion.atlassian.net/browse/MLID-2858) | not yet | — |
| D0-T4 — historical load | [MLID-2859](https://localinfusion.atlassian.net/browse/MLID-2859) | not yet | — |

Note: D0-T1 (MLID-2856) is a single Jira task covering **both** tables — authorizations and
notes. Its detail document holds both, in Part A and Part B, because the two feeds have different
payloads (the notes feed has its own note id and a parent-authorization reference, but no patient
id).

**D0-T1 in one paragraph**: an `authorizations` table keyed on the WeInfuse authorization id,
carrying the order series id as a plain indexed integer rather than a foreign key, the patient as
a raw source value plus a resolved foreign key, and the authorization's dates as date-only
columns; and a child `authorization_notes` table keyed on the WeInfuse note id, linked to its
parent authorization by both an internal foreign key and a preserved WeInfuse-id reference, with
the patient reached through that parent rather than stored. No resolved LISA order is stored on
either table, because links change from the Orders tab and a stored pointer would go stale — the
epic's read-time join stands. The task is gated on obtaining full report exports to confirm the
natural keys, the meaning of the authorizations updated-date column, and true nullability.

---

# Stage 1 — WeInfuse authorizations on the patient record

**Status**: high level. Do not detail until Stage 0 has landed.
**Type**: full stack, display only.

## 1.1 Goal

A WeInfuse Auths surface on the patient record showing that patient's authorizations, each
resolved to its LISA order, with their notes reachable from the table.

## 1.2 Tasks

| Task | Jira | What it covers |
|---|---|---|
| D1-T1 | [MLID-2860](https://localinfusion.atlassian.net/browse/MLID-2860) | Read layer and API route serving a patient's authorizations, including the join resolving each one to a LISA order. Reused by Stages 2 and 3. |
| D1-T2 | [MLID-2861](https://localinfusion.atlassian.net/browse/MLID-2861) | The WeInfuse Auths surface: two-tab structure beside the existing Auth Letter Review, the authorizations table, and the notes modal. Built so Stage 2 can reuse the table with a reduced column set. |
| D1-T3 | [MLID-2862](https://localinfusion.atlassian.net/browse/MLID-2862) | Patient record header: name, date of birth with age, and the copyable WeInfuse identifier. Separate so it is tested on its own. |

## 1.3 Known at this altitude

- The join is read-time, on the order series identifier already stored on the LISA order.
- The table and notes modal are shared components — Stage 2 reuses them without the LISA Order
  ID column.
- Check what the design system already provides for table, tabs and modal before creating
  anything.
- Existing behaviour that must not regress: the current Auth Letter Review tab.
- Expected to be gated by a feature flag.

## 1.4 Open questions — Stage 1

- Which roles may see the WeInfuse Auths tab, and is the rule the same here and on the order?
- Is the patient record header change in scope for this epic's acceptance, or a separate design
  ticket?
- Will the orders and patients data be served from PostgreSQL by the time this is built, or
  must the join cross two databases in the meantime?
- One feature flag for both surfaces, or one each?

## 1.5 Detail

_Not yet. Detail this section when Stage 0 has landed._

---

# Stage 2 — WeInfuse authorizations on the New Order

**Status**: high level. Do not detail until Stage 1 has landed.
**Type**: full stack, display only.

## 2.1 Goal

The same authorizations surface on the New Order, narrowed to that order.

## 2.2 Tasks

| Task | Jira | What it covers |
|---|---|---|
| D2-T1 | [MLID-2863](https://localinfusion.atlassian.net/browse/MLID-2863) | Order-scoped read and API route: the same authorizations, narrowed to the patient and to the order series linked to this order. |
| D2-T2 | [MLID-2864](https://localinfusion.atlassian.net/browse/MLID-2864) | The WeInfuse Auths surface on the New Order: two-tab structure in the order's prior authorization section, reusing the Stage 1 table and notes modal without the LISA Order ID column. |

## 2.3 Known at this altitude

- Reuses Stage 1's read layer rather than introducing a second join.
- Same feature-flag question as Stage 1.

## 2.4 Open questions — Stage 2

- Does Stage 3 wait for this stage or run beside it?

## 2.5 Detail

_Not yet. Detail this section when Stage 1 has landed._

---

# Stage 3 — System-driven Authorization Submission Date

**Status**: high level. Do not detail until Stage 1 has landed.
**Type**: full stack.

## 3.1 Goal

The Authorization Submission Date on a New Order stops being typed in by hand and is derived
from the authorizations linked to that order.

## 3.2 Tasks

| Task | Jira | What it covers |
|---|---|---|
| D3-T1 | [MLID-2865](https://localinfusion.atlassian.net/browse/MLID-2865) | Derive the date from the linked authorizations, define what it shows when there are none, remove the user's ability to edit it, and settle what happens to values users have already typed in. |

## 3.3 Known at this altitude

- Needs Stage 1's join, not Stage 1's interface.
- Touching an existing editable field, so the regression surface is wider than Stages 1 and 2.

## 3.4 Open questions — Stage 3

- Is the derived date computed at read time or stored on the order? Read time is simpler and
  self-correcting; stored is queryable and filterable.
- What happens to hand-entered dates that disagree with the derived value? — options: overwrite
  / keep and mark / report only.
- Does anything else in LISA read this field today — reporting, filters, exports — that changes
  behaviour once it becomes system driven?
- Does Stage 3 need its own feature flag, so the derived date can be switched off without
  hiding the tabs?

## 3.5 Detail

_Not yet. Detail this section when Stage 1 has landed._

---

## Cross-stage open questions

- Does the Product Owner need the epic restructured in Jira to show Stage 0, or does it stay an
  engineering-side breakdown under the existing three deliverables?
- What happens when an authorization arrives for a patient LISA does not know? Affects Stage 0
  storage and Stage 1 display.
