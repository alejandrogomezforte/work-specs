# LISA — High-Level Onboarding

> A glimpse of the system for a new software engineer joining the team.
> This is a map, not a manual. It points you at the big pieces, the data that matters,
> and the screens users live in. Once these click, the rest of the codebase reads much faster.

---

## 1. What LISA is

LISA is a **healthcare call-center and patient-management platform** for an infusion-therapy
business. Patients are referred for drug infusions; LISA helps the team intake the paperwork,
verify insurance, run clinical reviews of lab requirements, schedule and remind patients of
appointments, track the lifecycle of each order, and communicate with prescribing provider
offices.

The platform sits on top of an external EHR (**WeInfuse**) and pulls patient, appointment,
order, insurance, and location data from it. A lot of LISA's job is to enrich, automate, and
add workflow on top of that synced data.

---

## 2. Architecture & Stack

### 2.1 Monorepo layout (Turborepo + npm workspaces)

```
li-call-processor/
├── apps/
│   ├── web/        @repo/web        Next.js 14 main app          port 8080
│   ├── api/        li-public-api    NestJS public/webhook API    port 3001
│   ├── mcp/        li-mcp-server    NestJS MCP server (AI access) port 3002
│   └── storybook/  @repo/storybook  Design-system preview        port 6006
└── packages/
    ├── ui/                @repo/ui                 MUI 7 + Emotion design system
    ├── db/                @repo/db                 PostgreSQL / Drizzle schema (secondary)
    ├── shared/            @repo/shared             Shared types & utilities
    ├── eslint-config/     @repo/eslint-config      Shared lint rules
    └── typescript-config/ @repo/typescript-config  Shared tsconfig presets
```

Common commands run from the repo root through Turbo: `npm run dev` (all apps),
`npm run dev:web`, `npm run build`, `npm run lint`, `npm run test`, `npm run types:check`,
`npm run worker` (background job worker).

### 2.2 The web app (apps/web) — where ~90% of the work happens

| Concern | Choice |
|---|---|
| Framework | **Next.js 14** (App Router in `app/`, plus legacy Pages Router in `pages/api/`) |
| UI | React 18 + TypeScript, **MUI 7** + Emotion, MUI X Data Grid / Date Pickers |
| Primary DB | **MongoDB** via **Mongoose 8** (`local-infusion-db`) |
| Background jobs | **Pulse** (a fork of Agenda) — MongoDB-backed queue, separate `worker.ts` process |
| Realtime | **Socket.IO** (live call/order updates) and **SignalR** (client notifications) |
| Auth | **NextAuth** — Google OAuth + email/password credentials |
| Storage | **Azure Blob Storage** (primary), AWS S3, Google Cloud Storage |
| Secrets | **Azure Key Vault** |
| AI / LLM | OpenAI + Azure OpenAI, LangChain / LangGraph (clinical analysis, appeals, chat) |
| OCR | Azure Document Intelligence (intake document extraction) |
| Monitoring | Azure Application Insights + OpenTelemetry |

> Note on databases: MongoDB is the system of record. A newer **PostgreSQL/Drizzle**
> layer (`packages/db`) exists as a sync target — there is an active Mongo→Postgres sync
> engine. For day-to-day feature work you will almost always be in MongoDB/Mongoose.

### 2.3 Two component layers

1. **Design system** — `packages/ui/` (`@repo/ui`). Shared, reusable components on MUI 7 +
   Emotion, each with a Storybook story. **No CSS Modules.** Prefer this for new shared UI.
2. **App components** — `apps/web/components/`: `UI/` (app base, some legacy CSS Modules),
   `UI/MUI/` (form inputs — reuse these), `UICompositions/`, `Modules/` (feature-specific),
   `Layout/`.

### 2.4 Background jobs (Pulse)

Jobs are how LISA does everything asynchronous: syncing data from WeInfuse, sending
reminders, OCR'ing documents, running clinical-review analysis, exporting reports.

- Defined with `defineJob(name, handler, options)` and registered at startup.
- A dedicated **worker process** (`apps/web/worker.ts`) polls MongoDB (`pulseJobs` collection)
  every ~3s, runs up to 100 jobs concurrently, with retry/backoff.
- Definitions live in `apps/web/services/jobs/definitions/` (50+ jobs).
- Representative jobs: intake processing (`processIntake`), document OCR (`documentOcr`),
  clinical review analysis (`analyzeClinicalReview`), appointment reminders, RxPreferred
  claims, weekly provider reports, and pharma-partner exports (Abbvie, Argenx, Kedrion).
- You can trigger jobs from the **`/admin/jobs`** UI — no curl/JWT needed.

### 2.5 External integrations

| System | Role |
|---|---|
| **WeInfuse** | Source EHR — patients, appointments, orders, insurance, locations |
| **JotForm** | Patient intake form submissions (webhook → job) |
| **Spruce** | Fax / SMS / conversations with patients and provider offices |
| **RxPreferred** | Insurance claims processing (SFTP workflows) |
| **VIG** | "Virtual Infusion Guide" — healthcare AI chat assistant (LangChain adapter) |
| **Abbvie / Argenx / Kedrion** | Pharma-partner program data exports (dispense, inventory, status) |
| **Notion / Google Drive** | Knowledge-base / content sync |

### 2.6 The NestJS apps (smaller, focused)

- **`apps/api` (li-public-api, port 3001)** — public/webhook API. Receives intake webhooks,
  exposes drug-configuration and health endpoints. MongoDB via Mongoose.
- **`apps/mcp` (li-mcp-server, port 3002)** — a Model Context Protocol server that lets AI
  assistants query LISA data through structured tools (patient lookup, appointments,
  documents, orders, drug requirements, providers, locations). Two scopes: **non-PHI** and
  **PHI**, each behind auth + feature-flag guards.

---

## 3. Core Data Entities

LISA's domain is wide, but these are the entities a new engineer must understand first.
Almost everything is stitched together by **WeInfuse identifiers** — learn those linking
fields and the data model stops feeling random.

### The "spine" — the entities everything else hangs off

**Patient** (`patients`)
The central identity: demographics, contact, language preference, SMS opt-out, and embedded
call history. Key field: **`we_infuse_id_IV`** — the WeInfuse patient identifier used to link
to nearly everything else.

**Orders** — three collections: `neworders`, `maintenanceorders`, `leadorders`
A medication order — the drug, indication (ICD-10), status, administration details, assigned
location, and an audit trail. This is LISA's own **Order Tracker** workflow, and it is split
across **three sibling collections**, one per category:
- **`neworders`** — orders in the new-intake stage (the bulk of active workflow).
- **`maintenanceorders`** — ongoing/recurring orders.
- **`leadorders`** — early leads (often partial orders, before full data arrives).

These map to the three tabs you see on `/orders-tracker` and are typed in code as
`FullOrder<'new' | 'maintenance' | 'lead'>`. 
The collection names are registered in
`apps/web/utils/constants/collections.ts`.

**Intake** (`intakes`)
A submitted packet of paperwork (consent form, intake form, Spruce fax, eOrder, manual
upload). Tracks processing status (pending → processing → processed / needsReview / failed),
embeds the patient snapshot and the **documents** that came in, and links to orders via
`linkedOrderIds[]`. This is the front door for most patient data.

**Document** (`documents`)
An uploaded/extracted file (PDF, lab result, insurance card, referral). Belongs to an intake
(`intakeId`), carries OCR status/output (`ocrStatus`, `ocrData`), and vector-embedding state
for search. Links to orders via `linkedOrderIds[]`.

**Appointment** (`appointments`)
A scheduled infusion visit. Links patient + location + provider via WeInfuse IDs
(`we_infuse_patient_id`, `we_infuse_location_id`, `we_infuse_provider_npi`). Drives the
SMS-reminder workflow.

**Location** (`patientLocations`)
A clinic / infusion center. Key field **`we_infuse_id`**, plus NPI, tax ID, contact info,
and processing flags (`isIntakeProcessingEnabled`, `isPharmacyEligible`).

**User** (`users`)
A system user with a **role** (`user`, `admin`, `auditor`) and assigned `positions[]` and
`locations[]` that drive access. (Auditor is currently treated as Admin.)

### Insurance chain

**PatientInsurance** (`patient_insurances`) — a patient's coverage; links by
`we_infuse_patient_id`. Primary coverage is `priority: '1'`, `status: 'Active'`. Carries plan
name, member/group IDs, and pharmacy identifiers (`rx_bin`, `rx_grp`, `rx_pcn`).
→ **InsurancePlan** (`insurance_plans`) → **InsurancePayor** (`insurance_payors`).
Plan/payor master data also drives pharmacy eligibility and payor guideline links.

### Clinical & drug configuration

**ClinicalReview** (`clinicalReviews`)
An AI-assisted review of a patient's lab results against a drug's clinical requirements.
Links to Patient, the drug (`liDrugId`), the tracker order (`lisaOrderId`), and the documents
reviewed. Has a status workflow (draft → analyzing_labs → in_progress → approved/denied) and
an audit trail of document selection.

**LiDrug** (`lidrugs`) / **DrugRequirement** (`drugRequirements`)
LISA's internal drug configuration: clinical-review requirements, drug tier, order protocol,
pharmacy eligibility. This is what defines *what a clinical review checks*.

**Drug** (`drugs`)
Drug master data synced from WeInfuse (commercial/brand names).

### Providers

**ProviderMapping** / **ProviderOffice** / **CmsNpiProvider** / **HospitalSystem** — the
prescribing-provider side: NPI registry data, internal provider/office records with
communication preferences (fax vs. email, used for Spruce routing), and hospital-system
agreements grouping provider NPIs.

### The mental model to internalize

```
WeInfuse  ──sync──►  Patient ─┬─► Appointment
                              ├─► PatientInsurance ─► InsurancePlan ─► InsurancePayor
                              ├─► Intake ──► Document (OCR)
                              │      └─► linkedOrderIds ─┐
                              └─► Order Tracker  ────────┴──►  neworders
                                  (3 collections)              maintenanceorders
                                                               leadorders
                                                                 └─► ClinicalReview ─► LiDrug
```

> Linking gotcha worth remembering early: most cross-collection joins are on **WeInfuse IDs**
> (`we_infuse_id`, `we_infuse_patient_id`, …) rather than Mongo `_id`. And links from
> intakes/documents to tracker orders are stored as **plain hex strings**, not ObjectIds.

---

## 4. The Main Screens

These four are where users spend their day. Knowing them gives you the fastest path to
understanding what LISA actually *does*.

### 4.1 `/intakes` — the front door

**Route:** `apps/web/app/intakes/page.tsx` (legacy vs. new behind a feature flag:
`IntakesPageLegacy.tsx` / `IntakesPageNew.tsx`).

The team reviews incoming paperwork here. A paginated table of **Intake** records, filterable
by status (System Processing, Processed, Failed, Needs Review, Skipped), document type, and
category (Order, Insurance, Labs, Authorization, Supporting Medical). Users preview the
embedded documents, **approve/reject** an intake, **link it to orders**, or skip it.

**Data:** Intake (with OCR'd documents and extracted patient info).
**APIs:** `app/api/intakes/route.ts`, plus `[id]/review`, `[id]/link-orders`, `[id]/status`,
`[id]/document-intake`.

### 4.2 `/orders-tracker` — the operational heart

**Routes:** `app/orders-tracker/page.tsx` (redirects to `/new`) and
`app/orders-tracker/[category]/page.tsx`, where category is **new | maintenance | lead**.

**Components:** `Orders.tsx` (the data grid), with column sets in `NewOrders.tsx`,
`MaintenanceOrders.tsx`, `LeadOrder.tsx`.

The screen has **three tabs**, one per order collection:

| Tab label | Category | Collection |
|---|---|---|
| New Orders | `new` | `neworders` |
| Maintenance Orders | `maintenance` | `maintenanceorders` |
| Order Leads | `lead` | `leadorders` |

A tabbed MUI data grid of medication orders. Heavy inline editing (status, drug, assigned
location, notes), search/filter/sort, add/delete orders, plus AI-assisted flows: **drug
switch** context capture and **no-go** (blocked order) context capture. Expandable sub-views
show clinical reviews, financial review, documents, and order history. Each tab keeps its own
filter/sort/search state in the URL.

**Data:** `FullOrder<'new' | 'maintenance' | 'lead'>` — the three Order Tracker collections.
**APIs:** `app/api/orders/new|maintenance|lead/route.ts`, `orders/details`,
`orders/.../financial-calculations`, and the AI chat endpoints `orders/switch-context/chat`
and `orders/no-go-context/chat`.

> Performance note: the tracker is backed by an aggregation pipeline plus an
> `orderstrackercache` collection. The cache only refreshes when an order document changes —
> useful to know when a pipeline code change doesn't seem to take effect.

### 4.3 `/patient/[patientId]/...` — the 360° patient view

**Routes:** a tabbed area under `app/patient/[patientId]/` —
`details`, `history` (calls), `financial-calculator`, `clinical-reviews`, `prior-auth`,
`documents` / `document-records`, `insurance`, `appointments`, `orders`. Several tabs are
feature-flagged. **Components:** `PatientTabsModule` (the tab bar) wrapping per-tab modules;
`PatientDetailsInfo` for demographics.

Everything about one patient in one place: demographics and contact, insurance, call history,
clinical reviews, documents, appointments, and orders. PHI masking applies to sensitive
fields; client-side caching and socket updates keep call status fresh.

**Data:** Patient (joined to insurance, appointments, orders, documents, clinical reviews).
**APIs:** `app/api/patients/[id]/route.ts`, `app/api/orders/by-patient/route.ts`, and the
per-tab endpoints.

### 4.4 `/users-locations` — admin

**Route:** `app/users-locations/page.tsx`. **Component:**
`components/Modules/users-locations/UsersLocationsPanel.tsx`.

A dual-tab admin screen. **Users** tab: a table of users with role, positions, assigned
locations, and enabled/disabled status; add/edit via modal, set passwords, toggle status.
**Locations** tab: clinics/facilities with WeInfuse ID, names, NPI, tax ID, contact info, and
the pharmacy-eligible / intake-processing flags.

**Data:** User and Location.
**APIs:** `app/api/users-locations/route.ts`, `app/api/locations/route.ts`,
`app/api/admin/locations/route.ts`, plus user-management endpoints.

---

## 5. Feature Flags

Almost every new feature ships behind a flag. Flags are **MongoDB-backed** (not env vars), so
they can be toggled live with no redeploy — managed from the admin panel at
**`/admin/feature-flags`**.

**To add a flag:** add it to the `FeatureFlag` enum and `DEFAULT_FEATURE_FLAGS` in
`apps/web/utils/featureFlags/featureFlags.ts`. The code-defined default is used when the flag
has no MongoDB document yet (no row is created until an admin toggles it) or the DB is
unavailable.

**To read a flag:**

```ts
// Server (API routes, jobs, services)
import { getFeatureFlag } from '@/services/featureFlags';
if (await getFeatureFlag('ORDER_ELIGIBILITY')) { ... }

// Client (React components)
import { useFeatureFlag } from '@/utils/contexts/FeatureFlagContext';
import { FeatureFlag } from '@/utils/featureFlags/featureFlags';
const enabled = useFeatureFlag(FeatureFlag.ORDER_DETAILS);
```

Gate restricted features at **both** the server and the client — never rely on the client flag
alone. See `docs/FEATURE_FLAGS.md` for the full list and defaults.

---

## 6. Conventions worth knowing on day one

- **TypeScript:** no `any`. Use absolute imports (`@/...`); generated API clients via `@api/`.
- **Data access:** always go through the service layer in `apps/web/services/`, never raw
  Mongo. New code uses **Mongoose**, not the raw driver.
- **PHI everywhere:** this is HIPAA-regulated. Mask PHI in logs and responses, use
  `logger` (not `console`), and test that masking holds.
- **RBAC at both layers:** never rely on hidden UI — enforce role checks in the API/page too.
- **TDD is mandatory** here — red → green → refactor for all features and fixes.
- **Commits:** Conventional Commits with the Jira key —
  `[MLID-XXXX] - type(scope): description`. No Co-Authored-By lines.
- **Pagination shape:** `{ data, pagination: { page, pageSize, totalCount, totalPages } }`.

---

## 7. Where to go next

- `CLAUDE.md` (repo root) — the full development guide (commands, patterns, TDD rules).
- `docs/` — deeper documentation; `docs/FEATURE_FLAGS.md` for available flags.
- `apps/web/models/` — read the Mongoose schemas for the entities above; they are the
  most honest description of the domain.
- Run `npm run dev` and click through the four screens above with a test patient — the data
  model becomes concrete very quickly once you watch it move through intake → order →
  clinical review → appointment.
