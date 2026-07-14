# MLID-364 — R10 Runtime Verification Checklist (browser)

**Purpose:** confirm the app runs correctly in a real browser on the migrated stack (**Next.js 16 + React 19 + Auth.js v5**), route by route. Tests and `tsc` are green, but they do not catch runtime render breakage — that is what this pass is for.

**Why re-do it now (even though you spot-checked ~3 days ago):**
1. Your earlier pass predates **two `develop` syncs (~103 commits)** merged into the epic since then — the **Messages / Weekly Provider Report** and **Communications** features especially were NOT verified on the migrated stack.
2. R10 is broader than the PoC spot-check: it must deliberately exercise the libraries flagged as risky because they declare **React ≤18 peers** (same trap as `react-input-mask`).

**Environment:** run the epic branch (`epic/MLID-364-nextjs-v16-migration`) locally, log in, keep the **browser console open** the whole time.

**On every page, watch for:**
- Console errors/warnings: `findDOMNode`, hydration mismatch, `Maximum update depth exceeded`, `reactRender is not a function`.
- Blank sections, components that fail to render, broken layout.
- The specific risk item flagged on that route (see legend).

**Legend:** ⚠️PDF = `react-pdf` · ⚠️MODAL = `react-modal` · ⚠️MASK = masked input (`@mona-health/react-input-mask`) · ⚠️TOAST = `react-simple-toasts` · ⚠️EXPAND = `react-collapsible` · 🆕 = newly synced, never verified on this stack.

---

## 0. Cross-cutting checks (do these ONCE — they touch every page)

These are the migration's highest-risk surfaces. If any fails, it likely fails app-wide.

- [ ] **Login** (Google OAuth) — Auth.js v5 replaced NextAuth v4. This is the single biggest change.
- [ ] **Login** (credentials / password) if used.
- [ ] **Logout** works and clears the session.
- [ ] **Session persists** across a full page refresh.
- [ ] **Protected-route redirect:** while logged out, open a deep URL (e.g. `/orders-tracker`) → should redirect to login, not error.
- [ ] **Role-based access:** an Admin sees the admin nav; a non-admin/User account does not, and cannot open an admin URL directly.
- [ ] ⚠️TOAST **A save action shows a toast.** `react-simple-toasts` was a confirmed React 19 break (fixed by bumping to 6.1.0) — edit + save anything and confirm the toast appears.
- [ ] ⚠️MODAL **Open + close any dialog** (`react-modal` via the base `Dialog`).
- [ ] ⚠️EXPAND **Expand + collapse any section** (`react-collapsible` via the base `Expandable`).
- [ ] ⚠️PDF **Open a document / PDF** and confirm it renders (`react-pdf` under Next 16's forced SWC minify).
- [ ] ⚠️MASK **A phone / SSN field renders and masks** correctly as you type (`@mona-health/react-input-mask` fork).

---

## Tier 1 — Core workflows + highest migration risk + newly synced (DO FIRST)

### 1A. Orders Tracker (core daily driver)
- [ ] `/orders-tracker` — list loads, filters/sorting work
- [ ] `/orders-tracker/[category]` — pick a category; list renders
- [ ] `/orders-tracker/[category]/[orderId]` — order detail; all tabs load
- [ ] `.../[orderId]/documents` — ⚠️PDF document viewer renders
- [ ] `.../[orderId]/clinical-reviews` — ⚠️PDF ⚠️TOAST history list renders
- [ ] `.../[orderId]/clinical-reviews/new` — review wizard (multi-step) completes
- [ ] `.../[orderId]/clinical-reviews/[reviewId]` — single review detail
- [ ] `.../[orderId]/auth-review` — ⚠️PDF auth letters list
- [ ] `.../[orderId]/auth-review/[letterId]` — ⚠️PDF letter renders
- [ ] `.../[orderId]/financial-review` and `.../financial-review/new`
- [ ] `.../[orderId]/order-history`

### 1B. Patient
- [ ] `/patient/[patientId]/details` — ⚠️MASK SSN/phone fields render + mask
- [ ] `.../insurance` — ⚠️MASK insurance forms
- [ ] `.../documents` and `.../document-records` — ⚠️PDF viewer
- [ ] `.../prior-auth` and `.../prior-auth/[intakeId]`
- [ ] `.../orders`
- [ ] `.../appointments`
- [ ] `.../history` and `.../history/[historyId]`
- [ ] `.../financial-calculator`
- [ ] `.../clinical-reviews`

### 1C. Intakes
- [ ] `/intakes` — ⚠️PDF list with document preview
- [ ] `/intakes/[id]` — intake detail
- [ ] `/intakes/[id]/review` — ⚠️PDF review screen + forms

### 1D. Messages / Weekly Provider Report — 🆕 NEWLY SYNCED (unverified on this stack)
> Merged from `develop` in the last 3 days; not covered by your earlier pass. Verify carefully.
- [ ] 🆕 `/messages`
- [ ] 🆕 `/messages/weekly-report/configure`
- [ ] 🆕 `/messages/weekly-report/[id]/detail`
- [ ] 🆕 `/messages/weekly-report/[id]/review` — ReviewResolve table + resolve dialog + edit-note modal (new components)
- [ ] 🆕 `/messages/weekly-report/[id]/approve`
- [ ] 🆕 `/messages/weekly-report/[id]/practice/[practiceId]` — delivery tables (new)

---

## Tier 2 — Important, moderate risk

### 2A. Content Library
- [ ] `/content-library` — ⚠️PDF document previews
- [ ] `/content-library/payor-guidelines`

### 2B. Insurance
- [ ] `/insurance-payors`
- [ ] `/insurance-payors/audit-log`
- [ ] `/insurance-payors/[id]/guidelines`
- [ ] `/insurance-plans`
- [ ] `/insurance-plans/[id]` — ⚠️MODAL edit pharmacy-eligible modal

### 2C. Drug Admin Panel
- [ ] `/drug-admin-panel` — ⚠️MODAL treatment/biosimilar/add-drug modals
- [ ] `/drug-admin-panel/settings`
- [ ] `/drug-admin-panel/drugs/[id]` — ⚠️MODAL edit-drug modal

### 2D. Hospital System Agreements
- [ ] `/hospital-system-agreements`
- [ ] `/hospital-system-agreements/[id]` — ⚠️MASK NPI input ⚠️MODAL add/edit modal + bulk-update tab

### 2E. Provider Management & Appointments
- [ ] `/provider-management`
- [ ] `/appointments`
- [ ] `/appointments/[id]`

---

## Tier 3 — Admin, config, low-traffic (verify last)

### 3A. Admin
- [ ] `/admin` (home)
- [ ] `/admin/feature-flags`
- [ ] `/admin/jobs`
- [ ] `/admin/users`
- [ ] `/admin/locations`, `/admin/locations/new`, `/admin/locations/[id]`
- [ ] `/admin/holidays`
- [ ] `/admin/configuration`
- [ ] `/admin/drug-settings`
- [ ] `/admin/data-export-stats`
- [ ] `/admin/messages`
- [ ] `/admin/migration-status`
- [ ] `/admin/reverification-campaign` — ⚠️MASK form
- [ ] `/admin/ai-chat-settings`, `/admin/ai-chat-settings/[id]`
- [ ] `/admin/api-keys`, `/admin/api-keys/[id]`
- [ ] `/admin/orders-tracker/assignment`
- [ ] `/admin/mcp-instructions`
- [ ] `/admin/integrations` and each: `abbvie`, `argenx`, `google-drive`, `jotform`, `kedrion`, `notion`, `spruce`

### 3B. Users
- [ ] `/users-management`
- [ ] `/users-locations`

### 3C. Other
- [ ] `/rxpreferred-claims` — ⚠️MODAL edit-claim modal
- [ ] `/chat`
- [ ] `/ai-chat-dashboard`
- [ ] `/auditor/accepted`
- [ ] `/home`
- [ ] `/checkin`
- [ ] `/` (root)

---

## Where the flagged libraries live (reference)

| Library | Base component | Appears on |
|---|---|---|
| `react-pdf` | `components/UI/FileViewer/*` | order documents, auth-review, clinical-reviews, patient documents, intakes review, content-library |
| `react-modal` | `components/UI/Dialog/Dialog.tsx` | app-wide (every modal) |
| `react-collapsible` | `components/UI/Expandable/Expandable.tsx` | app-wide (expandable sections) |
| `@mona-health/react-input-mask` | `components/UI/MUI/MuiPhoneField.tsx`, `components/UI/MaskedInput/MaskedInput.tsx` | patient details/insurance, hospital NPI, admin forms, users, intake |
| `react-simple-toasts` | `components/UI/Toast/Toast.tsx` | app-wide (save/action toasts) |

---

## Sign-off

| Tier | Result | Notes / issues found |
|---|---|---|
| 0 — Cross-cutting | ⬜ | |
| 1 — Core + newly synced | ⬜ | |
| 2 — Important | ⬜ | |
| 3 — Admin / low-traffic | ⬜ | |

If everything is green, R10 can be marked ✅ in the plan and we can move to R12 (team decisions) → R13 (epic → develop PR). Any issue found → note the route + the console error here so it can be triaged.
