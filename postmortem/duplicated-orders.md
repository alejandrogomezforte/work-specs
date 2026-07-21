# Postmortem: Duplicated Orders in Production

## Summary

Two duplicated orders exist in production. The order was duplicated in WeInfuse
and the duplicate (the more recent one) was created with the **wrong drug**.

- **Original order** — display id `05k6`, `_id` = `6a58ee8cb2ed890709a7d70a`
- **Duplicate order** — display id `05k7`, `_id` = `6a58ee8db2ed890709a7d746` (created last, wrong drug)

Because this is production data (accessed through cloud.mongodb.com), all
investigation is done with **query filters only** — no scripts, no writes until a
remediation plan is agreed. No PHI is stored in this document.

## Environment

- Production MongoDB via cloud.mongodb.com (Atlas UI — filter-based querying only)

## Investigation

### Case 1 — Retrieve and compare the two orders

Fetched both order documents (PHI fields removed before comparison) and diffed
them field by field.

**Identical across both orders:** `createdBy` (`69160e05…7133`), `drug`
(`698507280c83fd8007e79da8`), `orderType` (`New Start`), `received` (2026-07-16),
`site` (`67b7083a…d625`), `provider`, `providerPracticeFax`, `dispenseAsWritten`.

**Differences:**

| Field | 05k6 `…d70a` (original) | 05k7 `…d746` (duplicate) |
|---|---|---|
| `createdAt` | 14:45:32.730 | 14:45:33.574 (0.844s later) |
| `lastUpdated` | 07-16 18:31:23 | 07-17 13:44:43 |
| `lastAssigned` | 07-16 15:18:52 | 07-16 14:45:33 |
| `updatedBy` | `6a2b43eb…8e32` | `69993725…b98b` |
| `assignedBy` | `67fe9b98…5284` (a user) | `"Auto Assigned"` |
| `assignedTo` | `6a2b43eb…8e32` | `69993725…b98b` |
| `displayId` | 05k6 | 05k7 |
| `weInfuseOrderSeriesId` | 0 | 1 |
| `status` | Working this week | No Go |
| `coverageStatus` | Buy & Bill | *(absent)* |
| `rtsOrNoGoDate` | *(absent)* | 07-17 16:00:00 |
| `notes` | benefits/financial review in progress | No-Go duplicate note (LISA AI) |

**Key observations:**

1. **`drug` is identical on both LISA orders** (`698507280c83fd8007e79da8`). The
   reported "wrong drug" on the WeInfuse duplicate did NOT propagate into LISA's
   `neworders.drug`. The wrong-drug discrepancy lives on the WeInfuse side; LISA
   mapped both order series to the same LISA drug. (Needs confirmation on the
   WeInfuse side.)
2. **Created 0.844 seconds apart by the same `createdBy` user**, with sequential
   `weInfuseOrderSeriesId` (0 then 1). This is not a human re-keying an order
   later — it indicates a rapid double-creation at intake/sync time. 05k7 was then
   `Auto Assigned`, while 05k6 was human-reassigned.
3. **05k7 is the one that was resolved**: marked `No Go` on 07-17 with the note
   "marked No Go due to a duplicate order being identified" (written by LISA AI),
   `rtsOrNoGoDate` set to 07-17 16:00. 05k6 remains the active order (`Working
   this week`, has `coverageStatus`).

## Timeline

- 2026-07-17 — Investigation started.

## Root Cause

_Pending._

## Remediation

_Pending._
