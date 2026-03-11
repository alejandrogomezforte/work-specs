# MLID-1868 — Intake Analyzer - Skipped Notes

## Task Reference

- **Jira**: [MLID-1868](https://localinfusion.atlassian.net/browse/MLID-1868)
- **Story Points**: 2
- **Branch**: `feature/MLID-1868-intake-skipped-notes`
- **Base Branch**: `develop`
- **Status**: To Do

---

## Summary

When an intake is skipped in the intake analyzer, require a mandatory reason note via a popup dialog before allowing the skip. Store the note on the intake document and display it in the intakes list. Rename the "Errors" column to "Note" and show the skip reason there for skipped intakes (while still showing error indicators for non-skipped intakes with errors).

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
- **What**: Create a modal dialog that:
  - Opens when the user selects "Skip Intake"
  - Contains a text field for the reason (mandatory — submit button disabled when empty)
  - Has Cancel and Skip buttons
  - Calls `skipIntake(intakeId, reason)` on submit
  - Uses MUI Dialog components consistent with existing patterns

### Step 5 — Wire the dialog into the intake detail page

- **Files**: `apps/web/app/intakes/[id]/page.tsx`
- **What**: When user clicks "Skip Intake" action, instead of directly calling the skip API, open the SkipReasonDialog. On dialog submit, perform the skip with the reason. On success, refetch the intake and show success message.

### Step 6 — Rename "Errors" column to "Note" and display skip reason

- **Files**: `apps/web/app/intakes/page.tsx`, `apps/web/app/intakes/page.test.tsx`
- **What**:
  - Rename column header from "Errors" to "Note"
  - For skipped intakes: display the `skipReason` text
  - For non-skipped intakes with errors: continue showing `ErrorIndicator` as before
  - Ensure the `skipReason` field is returned in the intakes list API response (it comes from the model, so no API changes needed for the list endpoint)

### Step 7 — Display skip reason on the intake detail page

- **Files**: `apps/web/app/intakes/[id]/page.tsx`
- **What**: When the intake status is `skipped` and `skipReason` exists, display the skip reason in a visible section (e.g., near the status indicator or in the errors/warnings area).

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
| `apps/web/app/intakes/page.tsx` | Modify | Rename "Errors" to "Note", show skip reason for skipped intakes |
| `apps/web/app/intakes/page.test.tsx` | Modify | Update tests for column rename and skip reason display |

---

## Testing Strategy

- **Unit tests**: SkipReasonDialog rendering, validation (empty reason disables submit), submit/cancel callbacks
- **API integration tests**: PATCH with skipReason, 400 when skipping without reason, skipReason cleared on status change
- **Service tests**: Updated skipIntake function sends reason
- **Page tests**: Column renamed to "Note", skip reason shown for skipped intakes, error indicator shown for non-skipped with errors
- **Manual verification**: Skip an intake, verify dialog appears, enter reason, verify it shows in "Note" column and detail page

---

## Security Considerations

- **Input validation**: `skipReason` must be a non-empty string, trimmed, with reasonable max length (e.g., 500 chars)
- **Authorization**: Existing intake PATCH authorization applies — no new permissions needed
- **PHI handling**: Skip reasons should not contain PHI; no special masking needed but input should be sanitized

---

## Open Questions

- None — the task is well-defined. The Jira mentions "Currently there is no edit note function" which appears to be stating current state, not requesting one. If edit functionality is needed, it can be a follow-up task.
