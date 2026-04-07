# 340B Eligibility (Maintenance Orders) — Key Test Values

Reference IDs for manual testing of the 340B Eligible column on the Maintenance Orders table.

## Eligibility Outcomes

Same formula as New Orders — see [340b-key-values.md](./340b-key-values.md) for full logic.

| # | Outcome | Provider NPI in Hospital System? | Patient Has Primary Active Insurance? | Insurance Type | Displayed Value |
|---|---------|----------------------------------|---------------------------------------|----------------|-----------------|
| 1 | Eligible | Yes | Yes | `Commercial` | Comma-separated hospital system names |
| 2 | Waiting on insurance | Yes | No | (not evaluated) | `"Waiting on Insurance Input"` |
| 3a | Not eligible (wrong insurance) | Yes | Yes | Any other type | `"No"` |
| 3b | Not eligible (no hospital match) | No | (not evaluated) | (not evaluated) | `"No"` |

---

## Provider Storage in Maintenance Orders

Unlike new orders (which store provider as an NPI string or ObjectId), maintenance orders have **3 provider formats**:

| Format | Count | Example | NPI Extraction |
|--------|-------|---------|----------------|
| String (doctor name) | 3,093 | `"Dr. William Monaco"` | Falls back to `$toString` — not a valid NPI, no hospital match → `"No"` |
| String (NPI) | ~10 | `"1225011968"` | Falls back to `$toString` — works correctly, matched against hospital systems |
| Object (resolved provider) | 21 | `{ npi: "1770860157", first_name: "MARY", last_name: "MARK" }` | Uses `$provider.npi` — works correctly |

The pipeline stage `$addFields: { _providerNpi: { $ifNull: ['$provider.npi', { $toString: '$provider' }] } }` handles all 3 formats.

---

## Existing Maintenance Orders with Hospital System NPIs

### NPI as Object (provider.npi)

| displayId | Provider NPI | Provider Name | Hospital System(s) | Patient _id | Patient Name | Insurance |
|-----------|-------------|---------------|---------------------|-------------|--------------|-----------|
| 0o7 | 1770860157 | MARY MARK | QA Hospital in Silver Spring | 68e900e8629c6322a9ec4300 | Fito Test 7 | None |
| 0oD | 1770860157 | MARY MARK | QA Hospital in Silver Spring | 6989d559e2d8c6414306d17b | (unnamed, 8081) | None |
| 0oH | 1770860157 | MARY MARK | QA Hospital in Silver Spring | 6989d559e2d8c6414306d17b | (unnamed, 8081) | None |
| 0oV | 1770860157 | MARY MARK | QA Hospital in Silver Spring | 6971e73af002e9d03851a4bd | LISA TEST | None |
| 0oV-1 | 1770860157 | MARY MARK | QA Hospital in Silver Spring | 6971e73af002e9d03851a4bd | LISA TEST | None |
| 0oZ | 1225011968 | THOMAS THOMAS | QA Hospitals (5 locations) | 6971e73af002e9d03851a4bd | LISA TEST | None |
| 0od | 1225011968 | THOMAS THOMAS | QA Hospitals (5 locations) | 69b057bd797f3cace48bf785 | (unnamed, 8514) | None |
| 0oe | 1225011968 | THOMAS THOMAS | QA Hospitals (5 locations) | 6971e73af002e9d03851a4bd | LISA TEST | None |

### NPI as String (provider = "NPI")

| displayId | Provider (string) | Hospital System(s) | Patient _id | Patient Name | Insurance |
|-----------|-------------------|---------------------|-------------|--------------|-----------|
| 0oM | 1225011968 | QA Hospitals (5 locations) | 6989d559e2d8c6414306d17b | (unnamed, 8081) | None |
| 0oM-1 | 1225011968 | QA Hospitals (5 locations) | 6989d559e2d8c6414306d17b | (unnamed, 8081) | None |

**All 10 orders above will show: `"Waiting on Insurance Input"`** — because the NPI matches a hospital system but none of these patients have primary active insurance records.

---

## Existing Maintenance Orders WITHOUT Hospital System NPIs

### NPI as Object (not in hospital systems)

| displayId | Provider NPI | Provider Name | Patient _id | Patient Name |
|-----------|-------------|---------------|-------------|--------------|
| 0o4 | 1134556921 | MARK TERESA | 6989d559e2d8c6414306d17b | (unnamed, 8081) |
| 0o5 | 1134556921 | MARK TERESA | 68e50815629c6322a910932a | - |
| 0o6 | 1134556921 | MARK TERESA | 68e3a124f5a2e19abcaf7766 | - |
| 0o8 | 1497789507 | JOHN JOHN | 6807cc0084836fa1a268309c | - |
| 0oK | 1649975202 | TE ZHAO | 6971e8ea1522a6fbea4865c1 | - |

### String providers (doctor names — no NPI match possible)

| displayId | Provider | Patient Name | Notes |
|-----------|----------|--------------|-------|
| 0Gl | (empty) | Michele Alshouli | Empty string provider |
| 0GE | (empty) | Margarita Sanchez-Ceballos | Empty string provider |
| 0JO | `-` | Mairead Ferrie | Dash string provider |

**All orders above will show: `"No"`** — because the provider NPI does not match any hospital system.

---

## Test Coverage Status

| Outcome | Covered? | Source |
|---------|----------|-------|
| 1 — Eligible (hospital names) | **Yes** | Order `0oh` (Chelsea Beaulieu + David Wiebe) |
| 2 — Waiting on Insurance Input | **Yes** | 10 pre-existing orders (hospital-NPI patients without insurance) |
| 3a — No (wrong insurance type) | **Yes** | Order `0oi` (Reid Pizzonia + David Wiebe, Medicaid) |
| 3b — No (no hospital match) | **Yes** | 3,000+ orders with string doctor names, plus ~13 with non-hospital NPIs |

### Remaining Orders to Create

Create orders via the UI at `/orders-tracker/maintenance` using the providers and patients listed below.

**Requirements per order:**
- Any location/site is fine
- Any drug is fine
- Any change type is fine (e.g., "Maintenance")
- The **provider** must be one whose NPI is registered in a hospital system (for outcomes 1/2/3a) or NOT registered (for outcome 3b)
- The **patient** must match the insurance criteria for the target outcome

#### Providers

| Name | NPI | Hospital System(s) |
|------|-----|--------------------|
| David Wiebe | 1679576722 | Advocate Health Care, Endeavor Health |
| William Pilcher | 1588667638 | Advocate Health Care, Endeavor Health |
| Laurent Gressot | 1215930367 | Advocate Health Care |
| (any non-hospital NPI) | 1306849450 | None — not in any hospital system |

#### Patients

| Name | _id | we_infuse_id_IV | Primary Insurance | payer_type_name |
|------|-----|-----------------|-------------------|-----------------|
| Chelsea Beaulieu | 6807cc0684836fa1a2683876 | 452282 | Anthem BCBS NH Commercial | Commercial |
| Kevin Angelli | 6807cc0084836fa1a268330b | 755870 | Mass General Brigham | Commercial |
| Reid Pizzonia | 68baedcd76aacd981d9e8cf3 | 977503 | Connecticut Medicaid | Medicaid |
| Remedios Jandusay | 68f1411f886cee3aa11e1690 | 1012283 | Aetna Health Plans MedicareAdvantage | Medicare Advantage |
| Anthony Giguere | 666c5a6fe20aab1b7a20bb6c | 987246 | (none) | (none) |

#### Outcome 1: Eligible — "Advocate Health Care, Endeavor Health"

**Criteria:** Provider NPI in hospital system + patient has primary active insurance with `payer_type_name = "Commercial"`

| Patient | Provider NPI | Order _id | displayId | Status |
|---------|-------------|-----------|-----------|--------|
| Chelsea Beaulieu | 1679576722 (David Wiebe) | 69c42af80b7b64289bb4e346 | 0oh | Created |
| Kevin Angelli | 1588667638 (William Pilcher) | - | - | To create |

#### Outcome 2: Waiting on Insurance Input — "Waiting on Insurance Input"

**Criteria:** Provider NPI in hospital system + patient has **no** primary active insurance

Covered by 10 pre-existing orders (see above). Optionally create one more for a clean test:

| Patient | Provider NPI | Order _id | displayId | Status |
|---------|-------------|-----------|-----------|--------|
| Anthony Giguere | 1679576722 (David Wiebe) | - | - | Optional |

#### Outcome 3a: No — NPI in hospital system but wrong insurance type

**Criteria:** Provider NPI in hospital system + patient has primary active insurance with `payer_type_name != "Commercial"` (e.g., Medicaid, Medicare Advantage)

| Patient | Provider NPI | Order _id | displayId | Status |
|---------|-------------|-----------|-----------|--------|
| Reid Pizzonia | 1679576722 (David Wiebe) | 69c431650b7b64289bb4e348 | 0oi | Created |
| Remedios Jandusay | 1215930367 (Laurent Gressot) | - | - | To create |

#### Outcome 3b: No — NPI not in any hospital system

**Criteria:** Provider NPI not registered in any hospital system (or provider is a string name, not an NPI)

Covered by 3,000+ pre-existing orders. Optionally create one for a clean test:

| Patient | Provider NPI | Order _id | displayId | Status |
|---------|-------------|-----------|-----------|--------|
| Chelsea Beaulieu | 1306849450 (not in hospital system) | - | - | Optional |

---

## Hospital Systems Reference

Same as new orders — see [340b-key-values.md](./340b-key-values.md#hospital-systems).

| Name | _id | Provider NPI(s) |
|------|-----|-----------------|
| Advocate Health Care | 69aafc3887e5b1cb843ace14 | 1679576722, 1588667638, 1497758544, 1215930367, 1932102084, 1841293990 |
| Endeavor Health | 69aafc5087e5b1cb843ace6d | 1679576722, 1588667638, 1497758544 |
| Northwestern Memorial Hospital | 69aafc2387e5b1cb843acddc | 0123456789 (fake NPI) |
| QA Hospital in Silver Spring | - | 1225011968, 1770860157 |
| QA Hospitals (5 locations) | - | 1225011968 |
