# [MLID-2306] style(auth-review): use brand green for the active step-tab underline

> Follow-up to the already-merged MLID-2306 Auth Review PR. A small visual fix
> caught after merge — scoped to a single CSS line.

## Summary

- The Auth Review detail step tabs (1. Verify Letter Type / 2. Verify Data /
  3. Weinfuse Summary Note) showed a **blue** underline (`#6466fa`) on the
  active tab, which did not match the mock and clashed with the design system.
- Changed the active indicator to the brand green `#134649` — the same
  `@repo/ui` `interactive.primary` token that drives the active indicator on the
  main orders-tracker Tabs (Order Dashboard / Order History / Documents), so the
  two tab rows now read consistently.

## Why not reuse the `@repo/ui` Tabs component

The step-tab row is the left half of a split card whose bottom border must align
with the right preview-panel header at a fixed 52px height. The `@repo/ui` Tabs
component brings its own height/border model that would have required several
overrides and risked breaking that alignment. The only visual discrepancy was
the indicator color, so a one-value token change gives a pixel-accurate match
with no alignment risk.

## Changes Overview

- **Files changed**: 1
- **Lines added**: +3
- **Lines removed**: -1

## Files Changed

| File | Change | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewDetail.module.css` | Modified | Active step-tab `border-bottom-color` `#6466fa` → `#134649` (brand green) |

## Commits

| Hash | Message |
|------|---------|
| `4195ad9ba` | `[MLID-2306] - style(auth-review): use brand green for the active step-tab underline` |

## Test Plan

- CSS-only change; no test or type impact.
- [ ] With `ORDER_DETAILS_AUTH_REVIEW` enabled, open an order's Auth Review
      detail for a multi-step letter; confirm the active step tab's underline is
      the dark brand green, matching the main orders-tracker tabs.

## Jira

- [MLID-2306](https://localinfusion.atlassian.net/browse/MLID-2306) — Order
  Details: Auth Review
