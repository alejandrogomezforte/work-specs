# Pharmacy Eligibility — Decision Flow Diagram

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
