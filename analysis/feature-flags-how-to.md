# Feature Flags — How-To Guide

## Overview

The app uses a **MongoDB-backed feature flag system** that allows dynamic toggling of features without redeployment. Flags are defined in code (a TypeScript enum), stored in MongoDB, and consumed both server-side and client-side through different mechanisms.

### Design Principles

- **Fail-safe**: If the database is unavailable, flags fall back to code-defined defaults.
- **Dual-layer**: Client-side uses a React Context + hooks; server-side uses a service class that talks directly to MongoDB.
- **Audit trail**: Every toggle records who made the change and when.
- **Admin UI**: Admins can view and toggle all flags from `/admin/feature-flags`.

---

## Architecture Diagram

```
                        +-----------------------+
                        |   FeatureFlag enum     |  (source of truth for flag names)
                        |   + DEFAULT values     |
                        +-----------+-----------+
                                    |
            +-----------------------+-----------------------+
            |                                               |
    CLIENT SIDE                                     SERVER SIDE
            |                                               |
  FeatureFlagProvider                            FeatureFlagService
  (React Context)                                (MongoDB operations)
      |         |                                       |
  useFeatureFlag   FeatureFlagWrapper          isFeatureEnabled()
  (hook)           (component)                 (async, via internal API)
      |                                                 |
  GET /api/feature-flags          GET /api/feature-flags/[flag]
      |                                                 |
      +------ FeatureFlagService.getAllFeatureFlags() ---+
                        |
                  MongoDB collection
                    "featureflags"
```

---

## Key Files

| File | Purpose |
|------|---------|
| `apps/web/utils/featureFlags/featureFlags.ts` | **FeatureFlag enum** (all flag names) + **DEFAULT_FEATURE_FLAGS** map + server-side helpers (`isFeatureEnabled`, `getAllFeatureFlags`) |
| `apps/web/utils/contexts/FeatureFlagContext.tsx` | **Client-side** React Context, `FeatureFlagProvider`, `useFeatureFlag` hook, `FeatureFlagWrapper` component, `withFeatureFlag` HOC |
| `apps/web/services/featureFlags/featureFlagService.ts` | **Server-side** `FeatureFlagService` class — CRUD operations against MongoDB |
| `apps/web/models/FeatureFlag.ts` | **Mongoose schema** for the `featureflags` collection |
| `apps/web/app/api/feature-flags/route.ts` | API route — GET all flags |
| `apps/web/app/api/feature-flags/[flag]/route.ts` | API route — GET single flag by name |
| `apps/web/app/api/admin/feature-flags/route.ts` | Admin API — GET flags with metadata, POST to toggle (admin-only) |
| `apps/web/app/admin/feature-flags/page.tsx` | Admin UI page for managing flags |
| `apps/web/components/Layout/Sidebar/Sidebar.tsx` | Primary example of feature-flagged navigation items |

---

## How to Add a New Feature Flag

### Step 1 — Add the flag to the enum

Open `apps/web/utils/featureFlags/featureFlags.ts` and add a new entry to the `FeatureFlag` enum:

```typescript
export enum FeatureFlag {
  // ... existing flags ...
  MY_NEW_FEATURE = 'MY_NEW_FEATURE',
}
```

### Step 2 — Set the default value

In the same file, add the default to `DEFAULT_FEATURE_FLAGS`. This value is used when the flag doesn't exist in MongoDB yet or when the database is unreachable:

```typescript
const DEFAULT_FEATURE_FLAGS: Record<FeatureFlag, boolean> = {
  // ... existing defaults ...
  [FeatureFlag.MY_NEW_FEATURE]: false, // disabled by default
};
```

> Convention: new features start as `false` (disabled). Logging/job flags that should fail-open use `true`.

### Step 3 — Use the flag in your code

#### Client-side (React component)

Use the `useFeatureFlag` hook:

```typescript
import { useFeatureFlag } from '@/utils/contexts/FeatureFlagContext';
import { FeatureFlag } from '@/utils/featureFlags/featureFlags';

const MyComponent = () => {
  const isEnabled = useFeatureFlag(FeatureFlag.MY_NEW_FEATURE);

  if (!isEnabled) return null;
  return <div>New feature content</div>;
};
```

Or use the `FeatureFlagWrapper` component for declarative gating:

```typescript
import { FeatureFlagWrapper } from '@/utils/contexts/FeatureFlagContext';

<FeatureFlagWrapper flag={FeatureFlag.MY_NEW_FEATURE} fallback={<OldComponent />}>
  <NewComponent />
</FeatureFlagWrapper>
```

#### Server-side (API route or job handler)

Use `isFeatureEnabled` (calls the internal API) or `FeatureFlagService.getFeatureFlag` (direct MongoDB):

```typescript
// Option A — via internal API (safe from any context)
import { isFeatureEnabled, FeatureFlag } from '@/utils/featureFlags/featureFlags';

const enabled = await isFeatureEnabled(FeatureFlag.MY_NEW_FEATURE);

// Option B — direct MongoDB (only in API routes / server code)
import { FeatureFlagService } from '@/services/featureFlags/featureFlagService';

const enabled = await FeatureFlagService.getFeatureFlag(FeatureFlag.MY_NEW_FEATURE);
```

### Step 4 — Add a sidebar entry (if applicable)

In `apps/web/components/Layout/Sidebar/Sidebar.tsx`:

1. Add a hook call at the top of the `Sidebar` component:

```typescript
const isMyFeatureEnabled = useFeatureFlag(FeatureFlag.MY_NEW_FEATURE);
```

2. Conditionally push a nav link:

```typescript
if (isMyFeatureEnabled) {
  conditionalLinks.push({
    name: 'My Feature',
    path: ROUTES.MY_FEATURE,
    icon: '...',
  });
}
```

### Step 5 — Enable the flag

Use the Admin UI at `/admin/feature-flags` to toggle the flag on. Alternatively, insert directly into MongoDB:

```javascript
db.featureflags.insertOne({
  flag: 'MY_NEW_FEATURE',
  enabled: true,
  createdBy: 'your.email@example.com',
  updatedBy: 'your.email@example.com',
  createdAt: new Date(),
  updatedAt: new Date(),
});
```

---

## How It Works — Data Flow

### Client-side flow

1. `FeatureFlagProvider` mounts and calls `GET /api/feature-flags`.
2. The API route calls `FeatureFlagService.getAllFeatureFlags()`, which queries MongoDB and merges results with code defaults.
3. The response is a `Record<string, boolean>` of all flag names and their status.
4. Flags are stored in React Context state.
5. Components call `useFeatureFlag(FeatureFlag.XYZ)` to read from context.
6. Changes take effect on next page load (or when `refetch()` is called on the context).

### Server-side flow

1. Code calls `isFeatureEnabled(FeatureFlag.XYZ)`.
2. This makes a `fetch()` to `GET /api/feature-flags/[flag]`.
3. The API route calls `FeatureFlagService.getFeatureFlag()` which does `FeatureFlagModel.findOne({ flag })`.
4. If the flag doesn't exist in MongoDB, the code-defined default is returned.
5. If the fetch or DB call fails, the code-defined default is returned.

---

## Sidebar Pattern — Detailed Breakdown

The Sidebar (`apps/web/components/Layout/Sidebar/Sidebar.tsx`) demonstrates three patterns:

### Pattern 1 — Pure feature flag

```typescript
const isDrugAdminPanelEnabled = useFeatureFlag(FeatureFlag.DRUG_ADMIN_PANEL);

if (isDrugAdminPanelEnabled) {
  conditionalLinks.push({ name: 'Drug Admin Panel', path: ROUTES.DRUG_ADMIN_PANEL, icon: '...' });
}
```

### Pattern 2 — Feature flag + role check

```typescript
const isChatEnabled = useFeatureFlag(FeatureFlag.CHAT_BUTTON);
const isChatAdminOnly = useFeatureFlag(FeatureFlag.CHAT_BUTTON_ADMIN_ONLY);

if (isChatEnabled && (!isChatAdminOnly || isPrivileged)) {
  conditionalLinks.push({ name: 'Chat', path: ROUTES.CHAT, icon: '...' });
}
```

### Pattern 3 — Permission-only (no feature flag)

```typescript
if (userObj && canManageHospitalSystems(userObj)) {
  conditionalLinks.push({ name: 'Hospital Systems', path: ROUTES.HOSPITAL_SYSTEM_AGREEMENTS, icon: '...' });
}
```

### Pattern 4 — Feature flag inside a role-gated section

```typescript
if (isPrivileged || isAuditor) {
  if (isProviderManagementEnabled) {
    auditorLinks.push({ name: 'Provider Management', path: ROUTES.PROVIDER_MANAGEMENT, icon: '...' });
  }
}
```

---

## MongoDB Schema

Collection: **`featureflags`**

| Field | Type | Description |
|-------|------|-------------|
| `flag` | String (unique, indexed) | The flag name (matches enum value) |
| `enabled` | Boolean | Whether the flag is on |
| `description` | String (optional) | Human-readable description |
| `createdBy` | String (optional) | Email of creator |
| `updatedBy` | String (optional) | Email of last updater |
| `createdAt` | Date (auto) | Mongoose timestamp |
| `updatedAt` | Date (auto) | Mongoose timestamp |

---

## Existing Flags (as of Feb 2026)

| Flag | Default | Category |
|------|---------|----------|
| `CHAT_BUTTON` | false | UI |
| `CHAT_BUTTON_ADMIN_ONLY` | true | UI / RBAC |
| `INTAKES` | false | UI |
| `INTAKE_PROCESSING` | true | Backend |
| `ORDERS_TRACKER` | false | UI |
| `APPOINTMENT_REMINDER` | false | Backend / Jobs |
| `PHI_MASKING` | false | Security |
| `RXPREFERRED_CLAIMS` | true | UI / Backend |
| `SEND_WHAT_TO_EXPECT` | false | Backend |
| `SEND_SYMPTOM` | false | Backend |
| `AI_CHAT_DASHBOARD` | false | UI |
| `CONTENT_LIBRARY` | false | UI |
| `CONTENT_LIBRARY_PREVIEW` | false | UI |
| `SPRUCE_FAX_ARCHIVE` | false | Backend |
| `SPRUCE_FAX_INTERNAL_NOTE` | false | Backend |
| `REVERIFICATION_CAMPAIGN` | false | Backend |
| `CLINICAL_REVIEWS` | false | UI |
| `CLINICAL_REVIEW_ALL_DRUGS` | false | UI |
| `PROVIDER_MANAGEMENT` | false | UI |
| `DRUG_ADMIN_PANEL` | true | UI |
| `CMS_NPI_JOB` | true | Jobs |
| `DOCUMENT_OCR_PROCESSING` | false | Jobs |
| `MRU_PATIENT_SEARCH` | false | UI |
| `MRU_PATIENT_HOME` | false | UI |
| `LOGS_FOR_*` (13 flags) | true | Webhook logging |

---

## Tips & Best Practices

1. **Always gate both UI and API**: Don't just hide a button — check the flag in the API route too (defense in depth).
2. **Choose defaults carefully**: Use `false` for new features (opt-in). Use `true` for jobs/logging where failure to check should not block processing.
3. **Combine with RBAC when needed**: The `CHAT_BUTTON` + `CHAT_BUTTON_ADMIN_ONLY` pattern shows how to progressively roll out to admins first, then all users.
4. **Clean up stale flags**: The admin UI marks flags that exist in the DB but are no longer defined in the enum (`isDefined: false`). Remove these periodically.
5. **No cache**: Flags are fetched with `cache: 'no-store'` so changes take effect immediately. Don't add caching without considering staleness implications.
