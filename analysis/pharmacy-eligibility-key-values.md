# Pharmacy Eligibility — Key Test Values

Reference IDs for manual testing of the Pharmacy Eligible column.

## Eligibility Outcomes

The pharmacy eligibility calculation returns a **string** with 3 possible outcomes, evaluated in order:

| # | Outcome | Has Primary Insurance? | Drug Pharmacy Eligible? | Site Pharmacy Eligible? | Insurance Condition | Displayed Value |
|---|---------|------------------------|-------------------------|-------------------------|---------------------|-----------------|
| 1 | Waiting | No | (not evaluated) | (not evaluated) | (not evaluated) | `"Waiting on Insurance Input"` |
| 2 | Yes | Yes | Not explicitly `false` | `true` | Met (see below) | `"Yes"` |
| 3 | No | Yes | Any fails | Any fails | Any fails | `"No"` |

**Insurance condition:**
- Medicare / Medicare Advantage -> **auto-pass** (case-insensitive regex)
- Commercial / Medicaid -> pass **only if** `insurance_plans.isPharmacyEligible == true`
- Any other payer type -> **fail**

**Evaluation order:** Insurance presence is checked first. If none, result is "Waiting on Insurance Input". Then all three conditions (drug, site, insurance) must pass for "Yes". If any fails, result is "No".

### Decision Flow Diagram

```
                        ┌─────────────────────┐
                        │   Order enters      │
                        │   pharmacy check    │
                        └──────────┬──────────┘
                                   │
                                   ▼
                  ┌──────────────────────────────────┐
                  │  Does the patient have a         │
                  │  primary active insurance?       │
                  └───────────┬────────────┬─────────┘
                              │            │
                           No │            │ Yes
                              ▼            ▼
               ┌──────────────────┐   ┌─────────────────────────┐
               │ "Waiting on      │   │  Is the drug pharmacy   │
               │  Insurance Input"│   │  eligible?              │
               │  ■ STOP          │   │                         │
               └──────────────────┘   └──────┬──────────┬───────┘
                                             │          │
                                          No │          │ Yes
                                             ▼          ▼
                                    ┌──────────┐  ┌───────────────────────┐
                                    │ "No"     │  │  Is the location      │
                                    │  ■ STOP  │  │  pharmacy eligible?   │
                                    └──────────┘  │                       │
                                                  └──────┬──────┬─────────┘
                                                         │      │
                                                      No │      │ Yes
                                                         ▼      ▼
                                                ┌──────────┐  ┌──────────────────┐
                                                │ "No"     │  │  What is the     │
                                                │  ■ STOP  │  │  insurance type? │
                                                └──────────┘  └──┬──────┬──────┬─┘
                                                                 │      │      │
                                      ┌──────────────────────────┘      │      └─────────────┐
                                      │                                 │                    │
                                      ▼                                 ▼                    ▼
                            ┌────────────────────┐      ┌───────────────────┐      ┌──────────────┐
                            │ Medicare /         │      │ Commercial /      │      │ Other type   │
                            │ Medicare Advantage │      │ Medicaid          │      │              │
                            └────────┬───────────┘      └────────┬──────────┘      └──────┬───────┘
                                     │                           │                        │
                                     │ auto-pass                 ▼                        ▼
                                     │                  ┌───────────────────┐      ┌──────────┐
                                     │                  │ Is the insurance  │      │ "No"     │
                                     │                  │ plan pharmacy     │      │  ■ STOP  │
                                     │                  │ eligible?         │      └──────────┘
                                     │                  └──────┬──────┬────┘
                                     │                         │      │
                                     │                      No │      │ Yes
                                     │                         ▼      │
                                     │                ┌──────────┐    │
                                     │                │ "No"     │    │
                                     │                │  ■ STOP  │    │
                                     │                └──────────┘    │
                                     │                                │
                                     └──────────────┬─────────────────┘
                                                    │
                                                    ▼
                                             ┌─────────────┐
                                             │   "Yes"     │
                                             │    ■ STOP   │
                                             └─────────────┘
```

**4 gates (in order):**

| Gate | Check | Pass condition | Fail result |
|------|-------|----------------|-------------|
| 1. Insurance exists | Patient has primary active insurance | Found | "Waiting on Insurance Input" |
| 2. Drug eligible | Drug is pharmacy eligible | Not explicitly `false` (true/null/undefined all pass) | "No" |
| 3. Location eligible | Location is pharmacy eligible | `isPharmacyEligible == true` | "No" |
| 4. Insurance type | Insurance type meets criteria | Medicare/MA: auto-pass. Commercial/Medicaid: insurance plan must be pharmacy eligible. Other: fail | "No" |

All 4 gates must pass to reach **"Yes"**. Any single failure short-circuits to its respective result.

---

## Reference Collections

### Drugs

| Name | _id | pharmacyEligible | Treated As |
|------|-----|------------------|------------|
| TREMFYA | 69839ec25313bcc28e0ecc81 | `true` | Eligible |
| QA Drug (Sodium, Billirubin) | 698c536eef01c5ef8e6059b9 | `true` | Eligible |
| TEST01 | 699ccfa692d3290d3bceb871 | `true` | Eligible |
| Actemra | 693885079879dc15c6a1de62 | *(undefined)* | Eligible (not explicitly `false`) |
| Skyrizi | 6938863d9879dc15c6a1de68 | *(undefined)* | Eligible (not explicitly `false`) |
| Asceniv | 693896d59879dc15c6a1de68 | *(undefined)* | Eligible (not explicitly `false`) |
| QA Test drug | 698f717d26577db9d23befd8 | `false` | Not eligible |
| agomez test drug | 69af60e4cfaf3469b51668a2 | `false` | Not eligible |

### Sites (Patient Locations)

| Name | _id | isPharmacyEligible | Notes |
|------|-----|--------------------|-------|
| Alexandria, Virginia | 67b7083a8532e12fdce9d618 | `true` | Eligible |
| Augusta, Maine | 67b7083a8532e12fdce9d619 | `true` | Eligible |
| Concord, New Hampshire | 67b7083a8532e12fdce9d61c | `true` | Eligible |
| Bangor, Maine | 67b7083a8532e12fdce9d61a | `false` | Not eligible |
| Pharmacy (Test), not Clinic | 698470b34b23b46fd1565ec6 | `false` | Not eligible |
| Test Clinic | 698476f94b23b46fd1565ec7 | `false` | Not eligible |

### Insurance Plans

> Most insurance plans have `isPharmacyEligible: false`. Anthem BCBS NH Commercial has been set to `true` for testing.

| Plan Name | _id | isPharmacyEligible | Notes |
|-----------|-----|--------------------|-------|
| Anthem BCBS NH Commercial | 6977d66701fc42e5479df79c | `true` | Matches patient 452282 (Chelsea Beaulieu) |
| Mass General Brigham | 6977d66701fc42e5479df6e5 | `false` | Matches patient 755870 (Kevin Angelli) |
| UnitedHealthcare Commercial | 6977d66701fc42e5479df6a0 | `false` | Matches patient 886357 (John Tolochko) |
| Transamerica Life Insurance | 6977d66701fc42e5479df691 | `false` | |
| Mutual of Omaha | 6977d66701fc42e5479df692 | `false` | |

### Patients (reused from 340B key values)

| Name | _id | we_infuse_id_IV | Payer Type | Plan Name | Plan Pharm Eligible? |
|------|-----|-----------------|------------|-----------|----------------------|
| Chelsea Beaulieu | 6807cc0684836fa1a2683876 | 452282 | Commercial | Anthem BCBS NH Commercial | `true` |
| Kevin Angelli | 6807cc0084836fa1a268330b | 755870 | Commercial | Mass General Brigham | `false` |
| John Tolochko | 681cff934baefb304675c4fa | 886357 | Commercial | UnitedHealthcare Commercial | `false` |
| Reid Pizzonia | 68baedcd76aacd981d9e8cf3 | 977503 | Medicaid | Connecticut Medicaid | `false` |
| Jose Velez | 6933562fdf5bf0f4dd0d58d4 | 1049678 | Medicaid | WellSense Health Plan | `false` |
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | 1012283 | Medicare Advantage | Aetna Health Plans MedicareAdvantage | *(auto-pass)* |
| Anne Bailey | 69407711f308335625c6cb66 | 1055632 | Medicare Advantage | Martin's Point Medicare Advantage | *(auto-pass)* |
| Robin Stocks | 6807cc0684836fa1a2683a57 | 767045 | Medicare | Medicare MD | *(auto-pass)* |
| Anthony Giguere | 666c5a6fe20aab1b7a20bb6c | 987246 | *(none)* | - | - |
| Sarah Ryder | 666c59f0e20aab1b7a20bb6b | 767632 | *(none)* | - | - |
| Mark Lapierre | 666b51d137d60e8d291683bc | 989898 | *(none)* | - | - |

---

## Orders to Create

### Outcome 1: "Waiting on Insurance Input" (no primary active insurance)

| Patient | Patient _id | Drug | Site | Expected Result |
|---------|-------------|------|------|-----------------|
| Anthony Giguere | 666c5a6fe20aab1b7a20bb6c | TREMFYA | Alexandria, Virginia | Waiting on Insurance Input |
| Sarah Ryder | 666c59f0e20aab1b7a20bb6b | Actemra | Augusta, Maine | Waiting on Insurance Input |

### Outcome 2a: "Yes" — Medicare/Medicare Advantage auto-pass

Use a pharmacy-eligible drug + pharmacy-eligible site + Medicare patient. Medicare auto-passes without checking insurance plan.

| Patient | Patient _id | Drug | Site | Payer Type | Expected Result |
|---------|-------------|------|------|------------|-----------------|
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | TREMFYA | Alexandria, Virginia | Medicare Advantage | Yes |
| Robin Stocks | 6807cc0684836fa1a2683a57 | Actemra | Concord, New Hampshire | Medicare | Yes |
| Anne Bailey | 69407711f308335625c6cb66 | Skyrizi | Augusta, Maine | Medicare Advantage | Yes |

### Outcome 2b: "Yes" — Commercial/Medicaid with eligible plan

| Patient | Patient _id | Drug | Site | Payer Type | Plan | Expected Result |
|---------|-------------|------|------|------------|------|-----------------|
| Chelsea Beaulieu | 6807cc0684836fa1a2683876 | TREMFYA | Alexandria, Virginia | Commercial | Anthem BCBS NH Commercial | Yes |

### Outcome 3a: "No" — Commercial/Medicaid with ineligible plan

| Patient | Patient _id | Drug | Site | Payer Type | Plan Pharm? | Expected Result |
|---------|-------------|------|------|------------|-------------|-----------------|
| Kevin Angelli | 6807cc0084836fa1a268330b | Actemra | Augusta, Maine | Commercial | `false` | No |
| Reid Pizzonia | 68baedcd76aacd981d9e8cf3 | TREMFYA | Concord, New Hampshire | Medicaid | `false` | No |

### Outcome 3b: "No" — site not pharmacy eligible

| Patient | Patient _id | Drug | Site | Site Pharm? | Expected Result |
|---------|-------------|------|------|-------------|-----------------|
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | TREMFYA | Bangor, Maine | `false` | No |
| Robin Stocks | 6807cc0684836fa1a2683a57 | Actemra | Pharmacy (Test), not Clinic | `false` | No |
| Anne Bailey | 69407711f308335625c6cb66 | Skyrizi | Test Clinic | `false` | No |

### Outcome 3c: "No" — drug explicitly ineligible

| Patient | Patient _id | Drug | Drug Pharm? | Site | Expected Result |
|---------|-------------|------|-------------|------|-----------------|
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | QA Test drug | `false` | Alexandria, Virginia | No |
| Robin Stocks | 6807cc0684836fa1a2683a57 | agomez test drug | `false` | Augusta, Maine | No |
