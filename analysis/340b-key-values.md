# 340B Eligibility — Key Test Values

Reference IDs for manual testing of the 340B Eligible column.

## Eligibility Outcomes

The 340B eligibility calculation returns a **string** with 3 possible outcomes, evaluated in order:

| # | Outcome | Provider NPI in Hospital System? | Patient Has Primary Active Insurance? | Insurance Type | Displayed Value |
|---|---------|----------------------------------|---------------------------------------|----------------|-----------------|
| 1 | Eligible | Yes | Yes | `Commercial` | Comma-separated hospital system names (e.g., `"Advocate Health Care, Endeavor Health"`) |
| 2 | Waiting on insurance | Yes | No | (not evaluated) | `"Waiting on Insurance Input"` |
| 3a | Not eligible (wrong insurance) | Yes | Yes | Any other type (Medicaid, Medicare Advantage, etc.) | `"No"` |
| 3b | Not eligible (no hospital match) | No | (not evaluated) | (not evaluated) | `"No"` |

**Evaluation order:** Hospital system match is checked first. If no match, the result is `"No"` regardless of insurance. If there is a match, insurance is checked next — no insurance means `"Waiting on Insurance Input"`, Commercial means eligible (show hospital names), anything else means `"No"`.

---

## Reference Collections

### Hospital Systems

| Name | _id | Provider NPI(s) | Notes |
|------|-----|-----------------|-------|
| Advocate Health Care | 69aafc3887e5b1cb843ace14 | 1679576722, 1588667638, 1497758544, 1215930367, 1932102084, 1841293990 | |
| Endeavor Health | 69aafc5087e5b1cb843ace6d | 1679576722, 1588667638, 1497758544 | |
| Northwestern Memorial Hospital | 69aafc2387e5b1cb843acddc | 0123456789 | Fake NPI, not in cmsnpiproviders |

### Providers (with NPI)

| Name | NPI | In Hospital System? | Notes |
|------|-----|---------------------|-------|
| David Wiebe | 1679576722 | Yes | Advocate Health Care, Endeavor Health |
| William Pilcher | 1588667638 | Yes | Advocate Health Care, Endeavor Health |
| (unknown) | 1497758544 | Yes | Advocate Health Care, Endeavor Health; No name in record |
| Laurent Gressot | 1215930367 | Yes | Advocate Health Care |
| Ravi Adusumilli | 1932102084 | Yes | Advocate Health Care; Cardiovascular Disease |
| Susan Wortsman | 1841293990 | Yes | Advocate Health Care |
| (lookup in CMS) | 1306849450 | No | Not in any hospital system |
| (lookup in CMS) | 1023011178 | No | Not in any hospital system |

### Patients

| Name | _id | we_infuse_id_IV | Insurance Type | Notes |
|------|-----|-----------------|----------------|-------|
| Chelsea Beaulieu | 6807cc0684836fa1a2683876 | 452282 | Commercial | Anthem BCBS NH Commercial |
| Kevin Angelli | 6807cc0084836fa1a268330b | 755870 | Commercial | Mass General Brigham |
| John Tolochko | 681cff934baefb304675c4fa | 886357 | Commercial | UnitedHealthcare Commercial |
| Reid Pizzonia | 68baedcd76aacd981d9e8cf3 | 977503 | Medicaid | Connecticut Medicaid |
| Jose Velez | 6933562fdf5bf0f4dd0d58d4 | 1049678 | Medicaid | WellSense Health Plan |
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | 1012283 | Medicare Advantage | Aetna Health Plans MedicareAdvantage |
| Anthony Giguere | 666c5a6fe20aab1b7a20bb6c | 987246 | (none) | No primary active insurance |
| Sarah Ryder | 666c59f0e20aab1b7a20bb6b | 767632 | (none) | No primary active insurance |
| Mark Lapierre | 666b51d137d60e8d291683bc | 989898 | (none) | No primary active insurance |

### Patient Insurances

| Patient (we_infuse_patient_id) | patient_id | _id | priority | status | payer_type_name | Notes |
|-------------------------------|------------|-----|----------|--------|-----------------|-------|
| 452282 | 6807cc0684836fa1a2683876 | 694304dcf0484dead5cae72a | 1 | Active | Commercial | Anthem BCBS NH Commercial |
| 755870 | 6807cc0084836fa1a268330b | 694304dcf0484dead5cae72d | 1 | Active | Commercial | Mass General Brigham |
| 886357 | 681cff934baefb304675c4fa | 694304dcf0484dead5cae732 | 1 | Active | Commercial | UnitedHealthcare Commercial |
| 977503 | 68baedcd76aacd981d9e8cf3 | 694304dcf0484dead5cae740 | 1 | Active | Medicaid | Connecticut Medicaid |
| 1049678 | 6933562fdf5bf0f4dd0d58d4 | 694304dcf0484dead5cae754 | 1 | Active | Medicaid | WellSense Health Plan |
| 1012283 | 68f1411f886cee3aa11e1690 | 694304dcf0484dead5cae75c | 1 | Active | Medicare Advantage | Aetna Health Plans MedicareAdvantage |

---

## Orders to Create

### Outcome 1: "Advocate Health Care, Endeavor Health" (NPI in hospital system + Commercial insurance)

| Patient | Patient _id | Provider | NPI | Expected Result |
|---------|-------------|----------|-----|-----------------|
| Chelsea Beaulieu | 6807cc0684836fa1a2683876 | David Wiebe | 1679576722 | Hospital names list |
| Kevin Angelli | 6807cc0084836fa1a268330b | William Pilcher | 1588667638 | Hospital names list |
| John Tolochko | 681cff934baefb304675c4fa | Laurent Gressot | 1215930367 | Hospital names list |

### Outcome 2: "Waiting on Insurance Input" (NPI in hospital system + no primary active insurance)

| Patient | Patient _id | Provider | NPI | Expected Result |
|---------|-------------|----------|-----|-----------------|
| Anthony Giguere | 666c5a6fe20aab1b7a20bb6c | David Wiebe | 1679576722 | Waiting on Insurance Input |
| Sarah Ryder | 666c59f0e20aab1b7a20bb6b | Ravi Adusumilli | 1932102084 | Waiting on Insurance Input |
| Mark Lapierre | 666b51d137d60e8d291683bc | Susan Wortsman | 1841293990 | Waiting on Insurance Input |

### Outcome 3a: "No" — NPI in hospital system but insurance is not Commercial

| Patient | Patient _id | Provider | NPI | Expected Result |
|---------|-------------|----------|-----|-----------------|
| Reid Pizzonia | 68baedcd76aacd981d9e8cf3 | David Wiebe | 1679576722 | No (Medicaid) |
| Jose Velez | 6933562fdf5bf0f4dd0d58d4 | William Pilcher | 1588667638 | No (Medicaid) |
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | Laurent Gressot | 1215930367 | No (Medicare Advantage) |

### Outcome 3b: "No" — NPI not in any hospital system

| Patient | Patient _id | Provider | NPI | Expected Result |
|---------|-------------|----------|-----|-----------------|
| Chelsea Beaulieu | 6807cc0684836fa1a2683876 | (lookup in CMS) | 1306849450 | No (NPI not in hospital system) |
| Kevin Angelli | 6807cc0084836fa1a268330b | (lookup in CMS) | 1023011178 | No (NPI not in hospital system) |
| John Tolochko | 681cff934baefb304675c4fa | (lookup in CMS) | 1306849450 | No (NPI not in hospital system) |

