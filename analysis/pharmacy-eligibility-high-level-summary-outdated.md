# Pharmacy Eligibility Column — High-Level Summary

> Part of [MLID-1492](https://localinfusion.atlassian.net/browse/MLID-1492) — Referral Tracker: Add 340B & Pharmacy Eligible on Orders

---

## What is this column?

A new **Pharmacy Eligible** column on the Orders Tracker (both New Orders and Maintenance Orders) that tells Infusion Guides whether an order qualifies for processing through the pharmacy workflow.

The column is **automatically calculated** based on existing data — no manual input is needed from the user.

---

## What does it display?

The column shows one of three possible values:

| Value | Meaning |
|-------|---------|
| **"Yes"** | The order is pharmacy eligible. The drug, location, and insurance all meet the criteria. |
| **"Waiting on Insurance Input"** | We can't determine eligibility yet because the patient doesn't have a primary active insurance on file. |
| **"No"** | The order is not pharmacy eligible. At least one of the three criteria (drug, location, or insurance) is not met. |

---

## How is it calculated?

The calculation checks three criteria. **All three must pass** for the order to be pharmacy eligible.

### Criteria 1: Is the drug pharmacy eligible?

We check whether the drug on the order has been marked as pharmacy eligible by an admin in the Drug Admin panel.

- If the drug is **not** marked as pharmacy eligible → **"No"**
- If the drug **is** pharmacy eligible → continue checking

### Criteria 2: Is the location pharmacy eligible?

We check whether the infusion location on the order has been marked as pharmacy eligible by an admin in the Locations panel.

- If the location is **not** marked as pharmacy eligible → **"No"**
- If the location **is** pharmacy eligible → continue checking

### Criteria 3: Does the patient's insurance qualify?

We look up the patient's primary active insurance and check its type. The rules depend on the insurance category:

- **Medicare or Medicare Advantage** → **automatically qualifies** (no further check needed)
- **Commercial or Medicaid** → qualifies **only if** the specific insurance plan has also been marked as pharmacy eligible by an admin
- **No primary active insurance on file** → **"Waiting on Insurance Input"**
- **Any other insurance type** → **"No"**

### In short

> **Pharmacy Eligible = Drug is eligible + Location is eligible + Insurance qualifies**

---

## What data does it depend on?

The calculation pulls from six data sources — more than 340B eligibility:

| Data source | What we use | Managed by |
|-------------|-------------|------------|
| **Order** | Linked drug, location, and patient | Infusion Guides (via Orders Tracker) |
| **Drug** | `pharmacyEligible` flag | Admins (via Drug Admin panel) |
| **Location** | `isPharmacyEligible` flag | Admins (via Locations panel) |
| **Patient record** | Link to the patient's insurance records | Synced from WeInfuse |
| **Patient insurance** | Primary insurance type and plan name | Synced from WeInfuse via Looker webhooks |
| **Insurance plan** | `isPharmacyEligible` flag | Admins (via Insurance Plans panel), also synced via Looker webhooks |

---

## When does it recalculate?

The eligibility value is stored on each order and updated automatically whenever the underlying data changes. There are seven scenarios that trigger a recalculation — more than 340B because pharmacy depends on more data sources.

### From user actions in the app (UI)

| # | What happens | Source | Collection changed | What gets recalculated | Impact |
|---|-------------|--------|-------------------|----------------------|--------|
| 1 | **An order is created** | UI | `neworders` | The new order | Instant, lightweight |
| 2 | **An order's drug, location, or patient is changed** | UI | `neworders` | The edited order | Instant, lightweight |
| 3 | **An admin toggles a drug's pharmacy eligible flag** | UI | `lidrugs` | All orders using that drug | Runs in the background. Popular drugs may affect many orders. |
| 4 | **An admin toggles a location's pharmacy eligible flag** | UI | `patientLocations` | All orders at that location | Runs in the background. High-volume locations may affect many orders. |
| 5 | **An admin toggles an insurance plan's pharmacy eligible flag** | UI | `insurance_plans` | All orders belonging to patients whose primary insurance uses that plan | Runs in the background. Has the deepest lookup chain — the system must trace from the plan back through patient insurances and patients to find affected orders. |

### From external systems (automated)

| # | What happens | Source | Collection changed | What gets recalculated | Impact |
|---|-------------|--------|-------------------|----------------------|--------|
| 6 | **Patient insurance is updated via Looker webhook** (insurance added, changed, or deactivated in WeInfuse) | Webhook | `patient_insurances` | All orders belonging to the affected patient(s) | Runs in the background. Can affect multiple patients in a single webhook delivery. |
| 7 | **Insurance plan data is synced via Looker webhook** (plan's pharmacy eligible flag may change) | Webhook | `insurance_plans` | All orders belonging to patients whose primary insurance uses any plan that changed | Runs in the background. The system compares old and new values and only recalculates for plans where the flag actually changed. |

---

## Triggers summary

| # | Trigger | Source | Collection | Complexity |
|---|---------|--------|------------|------------|
| 1 | Order created | UI | `neworders` | Trivial (inline) |
| 2 | Order updated (drug/site/patient) | UI | `neworders` | Trivial (inline) |
| 3 | Drug pharmacyEligible toggled | UI | `lidrugs` | Moderate |
| 4 | Location isPharmacyEligible toggled | UI | `patientLocations` | Moderate |
| 5 | Insurance plan isPharmacyEligible toggled | UI | `insurance_plans` | Moderate-High (3-hop reverse) |
| 6 | Patient insurance webhook | Webhook | `patient_insurances` | Moderate-High |
| 7 | Insurance plan webhook | Webhook | `insurance_plans` | High (3-hop + bulk) |

---

## Complexity overview

| Trigger | Source | Collection | Frequency | Orders affected | Speed | Runs as |
|---------|--------|------------|-----------|----------------|-------|---------|
| Order created | UI | `neworders` | Frequent (daily) | 1 order | Instant | Part of the save action |
| Order edited | UI | `neworders` | Frequent (daily) | 1 order | Instant | Part of the save action |
| Drug flag toggled | UI | `lidrugs` | Rare (admin action) | All orders using that drug | Seconds to minutes | Background job |
| Location flag toggled | UI | `patientLocations` | Rare (admin action) | All orders at that location | Seconds to minutes | Background job |
| Insurance plan flag toggled | UI | `insurance_plans` | Rare (admin action) | Potentially many — must trace plan → patients → orders | Seconds to minutes | Background job |
| Patient insurance webhook | Webhook | `patient_insurances` | Periodic (WeInfuse syncs) | A few per patient, but many patients per webhook | Seconds to minutes | Background job |
| Insurance plan webhook | Webhook | `insurance_plans` | Periodic (WeInfuse syncs) | Potentially many — only for plans where the flag actually changed | Seconds to minutes | Background job |

**Key optimization:** When the insurance plan webhook syncs data, the system doesn't blindly recalculate orders for every plan in the payload. It compares old and new `isPharmacyEligible` values and only recalculates orders tied to plans where the flag actually **changed**. If no flags changed, no recalculation happens at all.

---

## Comparison with 340B Eligibility

| Aspect | 340B Eligibility | Pharmacy Eligibility |
|--------|-----------------|---------------------|
| Data sources | 4 (order, hospital systems, patient, insurance) | 6 (order, drug, location, patient, insurance, insurance plan) |
| Criteria checked | 2 (provider in hospital system + Commercial insurance) | 3 (drug eligible + location eligible + insurance qualifies) |
| Recalculation triggers | 5 | 7 |
| Cost per calculation | 3 DB reads | 4–5 DB reads |
| Most expensive trigger | Hospital NPI list update | Insurance plan webhook (deepest chain + bulk scope) |
| Shared triggers | Order create/update, patient insurance webhook | Order create/update, patient insurance webhook |

**Shared recalculation:** When an order is created/updated or a patient insurance webhook fires, both 340B and pharmacy eligibility should be calculated together to avoid redundant work.

---

## Dependencies

This feature relies on existing admin-managed flags across three different panels:

- **Drug Admin** — each drug must have its `pharmacyEligible` flag configured
- **Locations Admin** — each location must have its `isPharmacyEligible` flag configured
- **Insurance Plans Admin** — each plan must have its `isPharmacyEligible` flag configured (also synced from Looker)

The feature will be gated behind the **ORDER_ELIGIBILITY** feature flag and can be enabled independently of the 340B Eligibility column (Deliverable 1).
