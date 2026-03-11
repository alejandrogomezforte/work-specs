# MLID-1603: Fix Header Overlapping Sidebar Panel

> **Status**: Merged and Done

## Context

**Bug**: On the Intake Analyser page, when the sidebar (hamburger menu) is opened, the DocumentIntake header block overlaps the left side panel.

**Root cause**: The `.header` class in `DocumentIntake.module.css` has `position: sticky; top: 0; z-index: 100`. This creates an unnecessary stacking context that competes with the sidebar (also z-index 100, `position: fixed`).

These properties are **redundant** because:
- The `.container` parent has a fixed height (`calc(100vh - 160px)`) and `overflow: hidden` — it never scrolls
- The `.header` is a flex child with `flex-shrink: 0` — it naturally stays at the top
- Scrolling happens inside the `.content` sibling (`overflow: auto`), not in the container

So `position: sticky` has no effect on layout, but its `z-index: 100` creates a stacking context that causes the overlap.

## Fix

### 1. File: `apps/web/components/Modules/document-intake/DocumentIntake.module.css`

**Lines 25-27**: Remove the three unnecessary properties from `.header`:

```css
/* REMOVE these lines */
position: sticky;
top: 0;
z-index: 100;
```

### 2. File: `apps/web/components/Layout/Sidebar/styles.module.css`

**Line 9**: Raise sidebar `z-index: 100` → `z-index: 1100` to prevent future overlap issues with other components that use z-index 100-1000.

- Sidebar starts at `top: 80px` (below the global header), so it won't visually cover the header area
- The ChatHistoryDialog overlay also uses z-index 1100, but it's a modal that covers the full viewport — no conflict since the sidebar would be behind it when the dialog is open

## Verification

1. Create branch `fix/MLID-1603-header-overlaps-side-panel` from `develop`
2. Apply the CSS changes
3. Run lint from `apps/web/`
4. Manual E2E test: open Intake Analyser → click hamburger → sidebar should not be overlapped by the DocumentIntake header
5. Verify scrolling inside the DocumentIntake content area still works correctly (header stays at top, content scrolls)
6. Verify sidebar works on other pages (Home, Appointments, etc.)
