# MLID-1868 — Intake Analyzer - Skipped Notes

## Task Reference

- **Jira**: [MLID-1868](https://localinfusion.atlassian.net/browse/MLID-1868)
- **Story Points**: 2
- **Branch**: `feature/MLID-1868-intake-skipped-notes`
- **Base Branch**: `develop`
- **Status**: In QA (v2 — UI revisions applied per PO feedback)

---

## Summary

When an intake is skipped in the intake analyzer, require a mandatory reason note via a popup dialog before allowing the skip. Store the note on the intake document and display it in the intakes list. Keep the "Errors" column for errors only. Display the skip reason below the status text for skipped intakes, with expandable/collapsible truncation for long notes.

---

## Codebase Analysis

Key findings from the investigation:

- **Skip flow**: User clicks "Skip Intake" in the actions menu on `/intakes/[id]`. This calls `triggerIntakeAction()` which does `PATCH /api/intakes/{id}` with `{ status: 'skipped' }`. No user input is collected.
- **Skip action definition**: In `intakeActions.ts`, `skipIntake` is a `isDirectAction: true` action — no background job, direct API call.
- **"Errors" column**: In `/intakes/page.tsx` line 909, header is `<th>Errors</th>`. Cell renders `ErrorIndicator` component only for non-processed intakes with errors.
- **Intake model**: `apps/web/models/Intake.ts` — has `intakeErrors` array and `status` enum (includes `'skipped'`). No `skipReason` or `note` field exists.
- **API PATCH**: `apps/web/app/api/intakes/[id]/route.ts` — accepts `{ status }` in body, validates against allowed statuses, saves.
- **Client service**: `apps/web/services/intakes/intakeService.ts` — `skipIntake(intakeId)` sends `{ status: 'skipped' }`.
- **Existing tests**: Test files exist for the intakes list page, individual intake API route, intake service, and intake error service.
- **UI patterns**: The codebase uses MUI components for dialogs (e.g., `IntakeSubmissionDialog.tsx` uses Dialog/DialogTitle/DialogContent).

---

## Implementation Steps

### Step 1 — Add `skipReason` field to Intake model

- **Files**: `apps/web/models/Intake.ts`
- **What**: Add optional `skipReason: string` field to the Intake schema. This stores the mandatory reason note when an intake is skipped.

### Step 2 — Update PATCH API to accept and store `skipReason`

- **Files**: `apps/web/app/api/intakes/[id]/route.ts`, `apps/web/app/api/intakes/[id]/route.test.ts`
- **What**: When PATCH receives `status: 'skipped'`, require `skipReason` in the body. If missing, return 400. Store `skipReason` on the intake document alongside the status update. Clear `skipReason` if status transitions away from skipped.

### Step 3 — Update client service to pass `skipReason`

- **Files**: `apps/web/services/intakes/intakeService.ts`, `apps/web/services/intakes/intakeService.test.ts`
- **What**: Update `skipIntake(intakeId, skipReason)` to accept and send the reason in the PATCH body.

### Step 4 — Create SkipReasonDialog component

- **Files**: `apps/web/app/intakes/[id]/SkipReasonDialog.tsx`, `apps/web/app/intakes/[id]/SkipReasonDialog.test.tsx`
- **Pattern**: MUI Dialog (Pattern 2) — matches `DeleteConfirmModal` and `AddEditHospitalModal`
  - Uses `Dialog`, `DialogTitle`, `DialogContent`, `DialogActions` from `@mui/material`
  - `maxWidth="sm"`, `fullWidth`
  - TextField for reason input, Cancel + Skip buttons in `DialogActions`
- **What**: Create a modal dialog that:
  - Opens when the user selects "Skip Intake"
  - Contains a text field for the reason (mandatory — submit button disabled when empty)
  - Has Cancel and Skip buttons
  - Calls `skipIntake(intakeId, reason)` on submit
  - Prevents close during submission (`disableEscapeKeyDown`)

### Step 5 — Wire the dialog into the intake detail page

- **Files**: `apps/web/app/intakes/[id]/page.tsx`
- **What**: When user clicks "Skip Intake" action, instead of directly calling the skip API, open the SkipReasonDialog. On dialog submit, perform the skip with the reason. On success, refetch the intake and show success message.

### Step 6 — Keep "Errors" column as-is, show skip reason below status

- **Files**: `apps/web/app/intakes/page.tsx`, `apps/web/app/intakes/page.test.tsx`
- **What**:
  - **Do NOT rename the "Errors" column** — leave it as "Errors" showing `ErrorIndicator` for all intakes with errors, regardless of status.
  - **Show skip reason in the Status column**: For rows where `status === 'skipped'` and `skipReason` exists, render the skip reason text below the status badge/text.
  - **Expandable truncation**: The skip reason text must be truncated to 1 line with a "show more" / "show less" toggle link. Implementation:
    1. Create an `ExpandableText` component (inline in the page file or as a small shared component):
       - Accept a `text: string` prop
       - Use `useState` for `expanded` (boolean) and `overflows` (boolean)
       - Use `useRef` on the text element and `useEffect` to detect overflow: `el.scrollHeight > el.clientHeight + 1`
       - When collapsed, apply CSS truncation: `display: '-webkit-box'`, `WebkitLineClamp: 1`, `WebkitBoxOrient: 'vertical'`, `overflow: 'hidden'`
       - When expanded, remove the clamping styles (render with no inline style override)
       - Render a "show more" / "show less" link below the text, visible only when `overflows || expanded`
       - Wrap the component in a `<div onMouseDown={(e) => e.stopPropagation()}>` to prevent the click from triggering row selection in the table
    2. In the Status column cell renderer, after the existing status content, conditionally render:
       ```tsx
       {intake.status === 'skipped' && intake.skipReason && (
         <ExpandableText text={intake.skipReason} />
       )}
       ```
    3. Ensure the row height accommodates the expanded text — if using a fixed row height, switch to auto row height for rows with skip reasons, or use `getRowHeight={() => 'auto'}` if the table supports it.
  - Ensure the `skipReason` field is returned in the intakes list API response (it comes from the model, so no API changes needed for the list endpoint)

### Step 7 — Display skip reason on the intake detail page

- **Files**: `apps/web/app/intakes/[id]/page.tsx`
- **What**: When the intake status is `skipped` and `skipReason` exists, display the skip reason in a visible section (e.g., near the status indicator or in the errors/warnings area). Use the same `ExpandableText` component from Step 6 if the text could be long, or display it in full if space allows on the detail page.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/models/Intake.ts` | Modify | Add `skipReason` optional string field to schema |
| `apps/web/app/api/intakes/[id]/route.ts` | Modify | Handle `skipReason` in PATCH, require when status='skipped' |
| `apps/web/app/api/intakes/[id]/route.test.ts` | Modify | Add tests for skipReason validation and storage |
| `apps/web/services/intakes/intakeService.ts` | Modify | Update `skipIntake` to accept and send reason |
| `apps/web/services/intakes/intakeService.test.ts` | Modify | Update tests for new skipIntake signature |
| `apps/web/app/intakes/[id]/SkipReasonDialog.tsx` | Create | Modal dialog for mandatory skip reason |
| `apps/web/app/intakes/[id]/SkipReasonDialog.test.tsx` | Create | Tests for SkipReasonDialog |
| `apps/web/app/intakes/[id]/page.tsx` | Modify | Wire dialog to skip action, show skip reason in detail view |
| `apps/web/app/intakes/page.tsx` | Modify | Add `ExpandableText` component, show skip reason below status for skipped intakes |
| `apps/web/app/intakes/page.test.tsx` | Modify | Add tests for skip reason display in status column and expand/collapse behavior |

---

## Testing Strategy

- **Unit tests**: SkipReasonDialog rendering, validation (empty reason disables submit), submit/cancel callbacks
- **API integration tests**: PATCH with skipReason, 400 when skipping without reason, skipReason cleared on status change
- **Service tests**: Updated skipIntake function sends reason
- **Page tests**: Skip reason shown below status for skipped intakes, "Errors" column unchanged, expand/collapse toggle works, error indicator still shown for intakes with errors
- **Manual verification**: Skip an intake, verify dialog appears, enter reason, verify it shows in "Note" column and detail page

---

## Security Considerations

- **Input validation**: `skipReason` must be a non-empty string, trimmed, with reasonable max length (e.g., 500 chars)
- **Authorization**: Existing intake PATCH authorization applies — no new permissions needed
- **PHI handling**: Skip reasons should not contain PHI; no special masking needed but input should be sanitized

---

## QA Testing

**Date**: 2026-03-11

### Test Document

| Field | Value |
|-------|-------|
| **_id** | `69b1e7f3f3cbe0589e5ed5bf` |
| **Patient** | Sarah Kowalczyk |
| **Type** | spruceFax |
| **Original status** | `needsReview` |
| **skipReason entered** | `testing from Alejandro Gomez` |

### Verification

- Confirmed via MongoDB MCP that after skipping, the document had `status: 'skipped'` and `skipReason: 'testing from Alejandro Gomez'`.
- Skip is safe and fully reversible — it only updates `status` and `skipReason` fields. No jobs, webhooks, or external API calls are triggered.

### Queries

**Find the test document:**
```js
db.intakes.find({ _id: ObjectId('69b1e7f3f3cbe0589e5ed5bf') }, { status: 1, skipReason: 1, 'patient.firstName': 1, 'patient.lastName': 1 })
```

**Rollback to original status:**
```js
db.intakes.updateOne({ _id: ObjectId('69b1e7f3f3cbe0589e5ed5bf') }, { $set: { status: 'needsReview' }, $unset: { skipReason: '' } })
```

**Find second test document (demo test):**
```js
db.intakes.find({ _id: ObjectId('69b2ceeaf09c0a6980f3bca2') }, { status: 1, skipReason: 1, type: 1, 'patient.firstName': 1, 'patient.lastName': 1 })
```

**Rollback second test document:**
```js
db.intakes.updateOne({ _id: ObjectId('69b2ceeaf09c0a6980f3bca2') }, { $set: { status: 'needsReview' }, $unset: { skipReason: '' } })
```

**Set long skipReason (~500 chars) for testing truncation/expand UI:**
```js
db.intakes.updateOne({ _id: ObjectId('69b2ceeaf09c0a6980f3bca2') }, { $set: { skipReason: 'Lorem ipsum dolor sit amet consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident sunt in culpa qui officia.' } })
```

---

## Open Questions

- None — the task is well-defined. The Jira mentions "Currently there is no edit note function" which appears to be stating current state, not requesting one. If edit functionality is needed, it can be a follow-up task.
