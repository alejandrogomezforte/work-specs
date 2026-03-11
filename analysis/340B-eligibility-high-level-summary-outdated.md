# 340B Eligibility Column — High-Level Summary

> Part of [MLID-1492](https://localinfusion.atlassian.net/browse/MLID-1492) — Referral Tracker: Add 340B & Pharmacy Eligible on Orders

---

## What is this column?

A new **340B Eligibility** column on the Orders Tracker (both New Orders and Maintenance Orders) that tells Infusion Guides whether an order qualifies for processing through a hospital system's 340B drug pricing program.

The column is **automatically calculated** based on existing data — no manual input is needed from the user.

---

## What does it display?

The column shows one of three possible values:

| Value | Meaning |
|-------|---------|
| **Hospital system name(s)** | The order is 340B eligible. Shows which hospital system(s) it qualifies through (e.g., "Memorial Health, St. Joseph's"). |
| **"Waiting on Insurance Input"** | The order's provider is associated with a hospital system, but we can't determine eligibility yet because the patient doesn't have a primary active insurance on file. |
| **"No"** | The order is not 340B eligible. |

---

## How is it calculated?

The calculation answers two questions in sequence:

### Question 1: Is the order's provider part of a hospital system?

We check whether the provider (identified by their NPI number) appears in any active hospital system's provider list. These lists are managed by admins in the Hospital System Agreements feature.

- If the provider is **not** in any hospital system's list → the answer is **"No"** and we stop here.
- If the provider **is** in one or more hospital system lists → we continue to Question 2.

### Question 2: Does the patient have Commercial insurance?

We look up the patient's primary active insurance to check its type.

- If the patient has **no primary active insurance** → **"Waiting on Insurance Input"**
- If the primary insurance type is **Commercial** → the order is **eligible**, and we display the matching hospital system name(s)
- If the primary insurance type is **anything other than Commercial** (Medicare, Medicaid, etc.) → **"No"**

### In short

> **340B Eligible = Provider is in a hospital system's list + Patient has Commercial insurance**

---

## What data does it depend on?

The calculation pulls from four data sources:

| Data source | What we use | Managed by |
|-------------|-------------|------------|
| **Order** | Provider NPI number and linked patient | Infusion Guides (via Orders Tracker) |
| **Hospital System Agreements** | Provider NPI lists per hospital system | Admins (via Hospital System Agreements page) |
| **Patient record** | Link to the patient's insurance records | Synced from WeInfuse |
| **Patient insurance** | Primary insurance type (Commercial, Medicare, etc.) | Synced from WeInfuse via Looker webhooks |

---

## When does it recalculate?

The eligibility value is stored on each order and updated automatically whenever the underlying data changes. There are five scenarios that trigger a recalculation:

### From user actions in the app (UI)

| # | What happens | Source | Collection changed | What gets recalculated | Impact |
|---|-------------|--------|-------------------|----------------------|--------|
| 1 | **An order is created** | UI | `neworders` | The new order | Instant, lightweight |
| 2 | **An order's provider or patient is changed** | UI | `neworders` | The edited order | Instant, lightweight |
| 3 | **An admin updates a hospital system's NPI list** | UI | `hospitalSystems` | Only orders tied to providers that were added to or removed from the list | Runs in the background. The system compares the old and new lists and only recalculates orders affected by the changes — not all orders in the system. |
| 4 | **An admin renames a hospital system** | UI | `hospitalSystems` | All orders tied to that hospital system's providers | Runs in the background. Infrequent but can affect many orders since the displayed name needs to be refreshed. |

### From external systems (automated)

| # | What happens | Source | Collection changed | What gets recalculated | Impact |
|---|-------------|--------|-------------------|----------------------|--------|
| 5 | **Patient insurance is updated via Looker webhook** (insurance added, changed, or deactivated in WeInfuse) | Webhook | `patient_insurances` | All orders belonging to the affected patient(s) | Runs in the background. Can affect multiple patients in a single webhook delivery. |

---

## Triggers summary

| # | Trigger | Source | Collection | Complexity |
|---|---------|--------|------------|------------|
| 1 | Order created | UI | `neworders` | Trivial (inline) |
| 2 | Order updated (provider/patient) | UI | `neworders` | Trivial (inline) |
| 3 | Hospital NPI list updated | UI | `hospitalSystems` | Moderate (diff reduces scope) |
| 4 | Hospital system renamed | UI | `hospitalSystems` | Moderate (rare trigger) |
| 5 | Patient insurance webhook | Webhook | `patient_insurances` | Moderate-High |

---

## Complexity overview

| Trigger | Source | Collection | Frequency | Orders affected | Speed | Runs as |
|---------|--------|------------|-----------|----------------|-------|---------|
| Order created | UI | `neworders` | Frequent (daily) | 1 order | Instant | Part of the save action |
| Order edited | UI | `neworders` | Frequent (daily) | 1 order | Instant | Part of the save action |
| Hospital NPI list updated | UI | `hospitalSystems` | Rare (admin action) | Potentially many, but optimized to only target changes | Seconds to minutes | Background job |
| Hospital system renamed | UI | `hospitalSystems` | Very rare | Potentially many (all orders tied to that system) | Seconds to minutes | Background job |
| Insurance webhook | Webhook | `patient_insurances` | Periodic (WeInfuse syncs) | A few per patient, but many patients per webhook | Seconds to minutes | Background job |

**Key optimization:** When an admin updates a hospital system's NPI list, the system doesn't blindly recalculate all orders. It detects which providers were **added** and which were **removed** from the list, and only recalculates orders tied to those specific changes. If nothing actually changed, no recalculation happens at all.

---

## Dependencies

This feature builds on the **Hospital System Agreements** feature ([MLID-1021](https://localinfusion.atlassian.net/browse/MLID-1021)), which established the ability for admins to manage hospital systems and their provider NPI lists. Without that data in place, no order can be 340B eligible.

The feature will be gated behind the **ORDER_ELIGIBILITY** feature flag and can be enabled independently of the Pharmacy Eligibility column (Deliverable 2).
