# Pharmacy Eligibility — Pipeline Validation Results

> Ran against `local-infusion-db.neworders` on 2026-03-09 using MongoDB MCP aggregation.

---

## Aggregation Pipeline

```javascript
[
  { $match: { deleted: { $ne: true } } },
  { $sort: { _id: -1 } },
  { $limit: 10 },

  // Safely convert string IDs to ObjectId (skip non-24-char values like NPI strings)
  {
    $addFields: {
      _patientOid: {
        $cond: [{ $eq: [{ $strLenCP: { $ifNull: ['$patient', ''] } }, 24] }, { $toObjectId: '$patient' }, null],
      },
      _drugOid: {
        $cond: [{ $eq: [{ $strLenCP: { $ifNull: ['$drug', ''] } }, 24] }, { $toObjectId: '$drug' }, null],
      },
      _siteOid: {
        $cond: [{ $eq: [{ $strLenCP: { $ifNull: ['$site', ''] } }, 24] }, { $toObjectId: '$site' }, null],
      },
      _providerOid: {
        $cond: [{ $eq: [{ $strLenCP: { $ifNull: ['$provider', ''] } }, 24] }, { $toObjectId: '$provider' }, null],
      },
    },
  },

  // Lookup patient
  {
    $lookup: {
      from: 'patients',
      let: { patientId: '$_patientOid' },
      pipeline: [{ $match: { $expr: { $eq: ['$_id', '$$patientId'] } } }, { $project: { we_infuse_id_IV: 1, name: 1 } }],
      as: 'patient_doc',
    },
  },
  { $unwind: { path: '$patient_doc', preserveNullAndEmptyArrays: true } },

  // Lookup drug
  {
    $lookup: {
      from: 'lidrugs',
      let: { drugId: '$_drugOid' },
      pipeline: [{ $match: { $expr: { $eq: ['$_id', '$$drugId'] } } }, { $project: { name: 1, pharmacyEligible: 1 } }],
      as: 'drug_doc',
    },
  },
  { $unwind: { path: '$drug_doc', preserveNullAndEmptyArrays: true } },

  // Lookup site
  {
    $lookup: {
      from: 'patientLocations',
      let: { siteId: '$_siteOid' },
      pipeline: [{ $match: { $expr: { $eq: ['$_id', '$$siteId'] } } }, { $project: { name: 1, isPharmacyEligible: 1 } }],
      as: 'site_doc',
    },
  },
  { $unwind: { path: '$site_doc', preserveNullAndEmptyArrays: true } },

  // Lookup provider
  {
    $lookup: {
      from: 'providers',
      let: { providerId: '$_providerOid' },
      pipeline: [{ $match: { $expr: { $eq: ['$_id', '$$providerId'] } } }, { $project: { npi: 1, name: 1 } }],
      as: 'provider_doc',
    },
  },
  { $unwind: { path: '$provider_doc', preserveNullAndEmptyArrays: true } },

  // === ELIGIBILITY STAGES (mirrors getEligibilityPipelineStages()) ===

  // Stage 1: Extract provider NPI
  { $addFields: { _providerNpi: { $ifNull: ['$provider_doc.npi', { $toString: '$provider' }] } } },

  // Stage 2: Lookup hospital systems by provider NPI
  {
    $lookup: {
      from: 'hospitalSystems',
      let: { npi: '$_providerNpi' },
      pipeline: [
        { $match: { $expr: { $and: [{ $eq: ['$deletedAt', null] }, { $in: ['$$npi', '$providers.npi'] }] } } },
        { $project: { name: 1 } },
      ],
      as: '_340b_hospitals',
    },
  },

  // Stage 3: Lookup primary active insurance (shared)
  {
    $lookup: {
      from: 'patient_insurances',
      let: { weInfuseId: '$patient_doc.we_infuse_id_IV' },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [{ $eq: ['$we_infuse_patient_id', '$$weInfuseId'] }, { $eq: ['$priority', '1'] }, { $eq: ['$status', 'Active'] }],
            },
          },
        },
        { $limit: 1 },
        { $project: { payer_type_name: 1, insurance_plan_name: 1 } },
      ],
      as: '_primaryInsurance',
    },
  },

  // Stage 4: Compute 340B eligibility
  {
    $addFields: {
      hospital340BEligible: {
        $switch: {
          branches: [
            { case: { $eq: [{ $size: '$_340b_hospitals' }, 0] }, then: 'No' },
            { case: { $eq: [{ $size: { $ifNull: ['$_primaryInsurance', []] } }, 0] }, then: 'Waiting on Insurance Input' },
            {
              case: {
                $regexMatch: {
                  input: { $ifNull: [{ $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] }, ''] },
                  regex: 'commercial',
                  options: 'i',
                },
              },
              then: {
                $reduce: {
                  input: '$_340b_hospitals.name',
                  initialValue: '',
                  in: { $cond: [{ $eq: ['$$value', ''] }, '$$this', { $concat: ['$$value', ', ', '$$this'] }] },
                },
              },
            },
          ],
          default: 'No',
        },
      },
    },
  },

  // Stage 5: Lookup insurance plan for pharmacy eligibility
  {
    $lookup: {
      from: 'insurance_plans',
      let: { planName: { $arrayElemAt: ['$_primaryInsurance.insurance_plan_name', 0] } },
      pipeline: [
        { $match: { $expr: { $eq: ['$planName', '$$planName'] } } },
        { $limit: 1 },
        { $project: { isPharmacyEligible: 1 } },
      ],
      as: '_insurancePlan',
    },
  },

  // Stage 6: Compute pharmacy eligibility
  {
    $addFields: {
      pharmacyEligible: {
        $switch: {
          branches: [
            // No primary insurance
            { case: { $eq: [{ $size: { $ifNull: ['$_primaryInsurance', []] } }, 0] }, then: 'Waiting on Insurance Input' },
            // All three conditions met
            {
              case: {
                $and: [
                  // Drug NOT explicitly ineligible
                  { $ne: [{ $ifNull: ['$drug_doc.pharmacyEligible', null] }, false] },
                  // Site is pharmacy eligible
                  { $eq: [{ $ifNull: ['$site_doc.isPharmacyEligible', false] }, true] },
                  // Insurance criteria
                  {
                    $or: [
                      // Medicare / Medicare Advantage (case-insensitive)
                      {
                        $regexMatch: {
                          input: { $ifNull: [{ $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] }, ''] },
                          regex: 'medicare(\\s+advantage)?',
                          options: 'i',
                        },
                      },
                      // Commercial / Medicaid with eligible plan (case-insensitive)
                      {
                        $and: [
                          {
                            $regexMatch: {
                              input: { $ifNull: [{ $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] }, ''] },
                              regex: 'commercial|medicaid',
                              options: 'i',
                            },
                          },
                          { $eq: [{ $arrayElemAt: ['$_insurancePlan.isPharmacyEligible', 0] }, true] },
                        ],
                      },
                    ],
                  },
                ],
              },
              then: 'Yes',
            },
          ],
          default: 'No',
        },
      },
    },
  },

  // Final projection for readability
  {
    $project: {
      displayId: 1,
      drug_name: '$drug_doc.name',
      drug_pharmacyEligible: '$drug_doc.pharmacyEligible',
      site_name: '$site_doc.name',
      site_isPharmacyEligible: '$site_doc.isPharmacyEligible',
      provider_npi: '$_providerNpi',
      hospital_systems: '$_340b_hospitals.name',
      insurance_payer_type: { $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] },
      insurance_plan_name: { $arrayElemAt: ['$_primaryInsurance.insurance_plan_name', 0] },
      plan_isPharmacyEligible: { $arrayElemAt: ['$_insurancePlan.isPharmacyEligible', 0] },
      hospital340BEligible: 1,
      pharmacyEligible: 1,
    },
  },
]
```

> **Note:** The lookups for patient/drug/site/provider use safe `$toObjectId` conversion (checking string length = 24) because some `provider` fields store NPI strings directly instead of ObjectIds. In the real pipeline, `expandIdsAggregate` handles this — this test pipeline replicates that behavior manually.

---

## Results (10 most recent new orders)

| Order | 340B Eligible | Pharmacy Eligible | Payer Type | Drug | Site Pharm? | Plan Pharm? | Hospital Systems |
|-------|---------------|-------------------|------------|------|-------------|-------------|------------------|
| 05EM | No | Waiting on Insurance Input | *(none)* | Skyrizi | true | - | - |
| 05EL | No | Waiting on Insurance Input | *(none)* | QA Drug (MRI) | true | - | - |
| 05EK | No | No | Commercial | Hydration | true | false | - |
| 05EJ | Advocate Health Care | No | Commercial | Hydration | true | false | Advocate Health Care |
| 05EI | Advocate Health Care, Endeavor Health | No | Commercial | Hydration | true | false | Advocate Health Care, Endeavor Health |
| 05EH | Waiting on Insurance Input | Waiting on Insurance Input | *(none)* | Hydration | true | - | Advocate Health Care, Endeavor Health |
| 05EG | No | No | Medicaid | Hydration | true | false | Advocate Health Care, Endeavor Health |
| 05EF | Advocate Health Care, Endeavor Health | No | Commercial | Hydration | true | false | Advocate Health Care, Endeavor Health |
| 05EE | No | No | Commercial | Hydration | true | *(no match)* | - |
| 05ED | No | No | Medicaid | Hydration | true | false | Advocate Health Care, Endeavor Health |

---

## Analysis

### Scenarios covered

| Scenario | Orders | Correct? |
|----------|--------|----------|
| No primary insurance → both show "Waiting on Insurance Input" | 05EM, 05EL, 05EH | Yes |
| Provider NPI in hospital system + Commercial → 340B shows hospital name(s) | 05EJ, 05EI, 05EF | Yes |
| Provider NPI in hospital system + no insurance → 340B "Waiting on Insurance Input" | 05EH | Yes |
| Provider NPI NOT in hospital system → 340B "No" | 05EM, 05EL, 05EK, 05EE | Yes |
| Commercial + plan.isPharmacyEligible = false → Pharmacy "No" | 05EK, 05EJ, 05EI, 05EF | Yes |
| Medicaid + plan.isPharmacyEligible = false → Pharmacy "No" | 05EG, 05ED | Yes |
| Commercial + no matching plan (leading space in name) → Pharmacy "No" | 05EE | Yes |

### Scenarios NOT covered in this sample

| Scenario | Why |
|----------|-----|
| Medicare / Medicare Advantage → Pharmacy auto-pass | No Medicare patients in sample |
| drug.pharmacyEligible = false → Pharmacy "No" | All drugs have undefined pharmacyEligible |
| site.isPharmacyEligible = false → Pharmacy "No" | All sites have isPharmacyEligible = true |
| Commercial/Medicaid + plan.isPharmacyEligible = true → Pharmacy "Yes" | No eligible plans in sample |

### Notes

- **05EE**: Insurance plan name has leading whitespace (`" Anthem BCBS of Maine \ COMM"`) — the `insurance_plans` lookup returns no match because `planName` comparison is exact. This is correct behavior (data quality issue in source, not a pipeline bug).
- **05EG**: Provider NPI matches hospital systems but payer type is Medicaid (not Commercial), so 340B correctly returns "No" — 340B only passes for Commercial insurance.
