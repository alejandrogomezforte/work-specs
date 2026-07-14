# MLID-2669 — Prior Auth Verify Data: Back button frozen by render loop

## Task Reference

- **Jira**: [MLID-2669](https://localinfusion.atlassian.net/browse/MLID-2669)
- **Type**: Bug (Low priority)
- **Branch**: `fix/MLID-2669-prior-auth-verify-back-button`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

On the Order Details → Prior Auth tab, opening a letter and advancing to step 2
("Verify Data") freezes the screen: the header **Back** button (and every other
control) stops responding, and the URL never changes.

The ticket describes this as a broken Back button, but the Back handler is
correct. The real defect is an **infinite render loop** in the shared
`VerifyDataForm` component that fires the moment step 2 mounts. React aborts with
*"Maximum update depth exceeded"* and the component tree is frozen, so the click
on Back can never take effect.

The fix breaks the loop inside `VerifyDataForm` so it only reports changes upward
when the form content actually changes.

---

## Root Cause Analysis

Component: `apps/web/components/Modules/patient/components/patientPriorAuth/VerifyDataForm.tsx`

1. `VerifyDataForm` reads all form values with react-hook-form's `watch()`
   (assigned to `formValues`). Called with no arguments, `watch()` returns a
   **new object reference on every render**.
2. The final `useEffect` in the component depends on `formValues` and
   unconditionally calls `onChange(formValues, checkValidity())`. Because
   `formValues` has a new identity every render, this effect runs on **every**
   render.
3. `onChange` is the parent's `handleStep2Change`, which does
   `setStep2Data(formData)` + `setStep2IsValid(isValid)`. `setStep2Data` always
   receives a brand-new object reference, so the parent state always "changes"
   and the parent always re-renders.
4. The parent re-render re-renders `VerifyDataForm` → `watch()` yields a new
   `formValues` object → the effect fires again → `onChange` again → loop with no
   fixed point. React bails out with "Maximum update depth exceeded".

`checkValidity` is a `useCallback` that also depends on `formValues`, so its
identity churns every render as well, reinforcing the loop.

**Scope:** `VerifyDataForm` is shared. Both consumers use the identical
`handleStep2Change` = `setStep2Data(...)` pattern:
- `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.tsx`
  (order-scoped — the surface in this ticket)
- `apps/web/components/Modules/patient/components/patientPriorAuth/PriorAuthReview.tsx`
  (patient-scoped)

Both therefore carry this latent loop. Fixing the shared component resolves both.

**Why the symptom differs between the two screens.** Both screens loop, but their
Back controls navigate differently:
- `AuthReviewDetail` (order) uses `router.push(...)` — a client-side App Router
  navigation that React must process and commit. While the tree is stuck in
  "Maximum update depth exceeded", the push cannot complete, so Back looks dead
  and the URL never changes. This is the reported symptom.
- `PriorAuthReview` (patient) uses `router.back()` (when `window.history.length > 1`)
  — the browser's native history navigation, which happens at the browser level
  independent of React. It escapes the frozen tree, so "Back to list" *appears*
  to work even though the same loop and console error are present.

The differing navigation only masks the symptom on the patient screen; both
screens hit the same underlying loop, and the shared fix addresses both.

Observed error (browser console), abbreviated call stack:
```
Maximum update depth exceeded ...
AuthReviewDetail.useCallback[handleStep2Change]  (AuthReviewDetail.tsx)
VerifyDataForm.useEffect                          (VerifyDataForm.tsx)
AuthReviewDetail
AuthReviewLetterPage
```

There is currently **no test** covering `VerifyDataForm`, so nothing guards
against this regression.

---

## Fix Design

Guard the reporting effect in `VerifyDataForm` so it only calls `onChange` when
the serialized form values **and** validity differ from what was last reported.
A `useRef` holds the last-sent signature; the effect still runs each render (that
is inherent to the `watch()` snapshot usage and is fine for a small form), but it
no longer pushes a redundant state update into the parent, so the feedback loop
has a fixed point and terminates.

```ts
const lastSentRef = useRef<string | null>(null);

useEffect(() => {
  const isValid = checkValidity();
  const signature = JSON.stringify([formValues, isValid]);
  if (signature === lastSentRef.current) {
    return; // nothing changed — do not trigger a parent state update
  }
  lastSentRef.current = signature;
  onChange(formValues, isValid);
}, [formValues, checkValidity, onChange]);
```

Why this option (vs. alternatives):
- **Self-contained** in the shared component — it does not rely on either parent
  passing a referentially stable `onChange`.
- No `eslint-disable` needed; the dependency array stays honest.
- Preserves existing behavior: real edits still propagate value + validity to the
  parent; first mount still emits once so the wizard's Next/validity gating works.

`JSON.stringify` over this small, flat-ish form on each render is negligible.
`formValues.selectedLiDrug` is a plain object and serializes deterministically
alongside the string fields.

No parent changes are required. `handleStep2Change` in both `AuthReviewDetail`
and `PriorAuthReview` can stay as-is.

---

## Implementation Steps (TDD)

### Step 1 — RED: failing regression test for the loop

- **File**: `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/VerifyDataForm.test.tsx` (new)
- **What**: Add a test that mirrors the real parent wiring — a small stateful
  harness component that stores the `onChange` output in state and feeds it back
  as `initialData`, mimicking `setStep2Data` + `initialData={step2Data}`.
  - Test A (the loop): rendering the harness with `letterType="Approval"` must
    **not** throw and must not exceed the update depth; assert `onChange`
    settles (called a small, bounded number of times rather than unbounded).
  - Against the current code this reproduces "Maximum update depth exceeded" and
    fails.
- Run: `npm run test -- VerifyDataForm`

### Step 2 — GREEN: add the ref-guard to the reporting effect

- **File**: `apps/web/components/Modules/patient/components/patientPriorAuth/VerifyDataForm.tsx`
- **What**: Introduce `lastSentRef` and guard the final `useEffect` as shown in
  Fix Design so `onChange` is only invoked when the serialized
  `[formValues, isValid]` signature changes. Keep the dependency array
  `[formValues, checkValidity, onChange]`.
- Run: `npm run test -- VerifyDataForm` → Test A passes.

### Step 3 — REFACTOR + guard against regressing propagation

- **File**: `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/VerifyDataForm.test.tsx`
- **What**: Add Test B — editing a field (e.g. a required text field for the
  chosen `letterType`) still calls `onChange` with the updated value, and
  validity flips from invalid → valid once required fields are satisfied. This
  proves the guard did not suppress legitimate updates.
- Run: `npm run test -- VerifyDataForm && npm run types:check && npm run lint:fix`
  (from `apps/web`).

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/Modules/patient/components/patientPriorAuth/VerifyDataForm.tsx` | Modify | Ref-guard the reporting `useEffect` so it only calls `onChange` when form values + validity change; breaks the infinite loop. |
| `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/VerifyDataForm.test.tsx` | Create | Regression test: no update-depth loop with a stateful parent; edits still propagate value + validity. |

---

## Testing Strategy

- **Unit tests**: New `VerifyDataForm.test.tsx` — (A) no infinite loop under a
  stateful parent; (B) field edits still propagate value + validity.
- **Manual verification** (user performs in browser):
  1. Order Details → Prior Auth tab → open a letter → click **Verify Data**.
  2. Confirm no "Maximum update depth exceeded" error in the console.
  3. Click header **Back** → returns to the Prior Auth tab (URL changes to
     `…/auth-review`).
  4. Repeat the same on the patient Prior Auth review step 2 to confirm the
     shared fix holds there too.

---

## Security Considerations

- **PHI handling**: No change to what data is rendered or transmitted; the fix
  only controls when in-memory form state is reported to the parent. No new
  logging of form contents.
- **Authorization**: Unchanged.

---

## Open Questions

- None. Approach approved (ref-guard signature). Fix is localized to the shared
  `VerifyDataForm`; no parent or route changes needed.
