# MLID-2306 — Auth Review: Prior-Auth Filter Rules

How the Auth Review list decides which prior-auth letters to show for a given order.

Each letter is checked against two things: **date** and **drug**. Both must match for the letter to appear.

---

## Date check

The letter must be dated on or after the order was created.

The "date" we use is:
- `additionalInfo.ocrData.authorization.dateOfDecision.value` if the OCR extracted one
- otherwise we fall back to the letter's own `createdAt`

---

## Drug check

This is a waterfall. We stop at the first rule that gives a clear answer.

### Rule 1 — Hard match (most reliable)

If the letter has `additionalInfo.priorAuthData.liDrugId.value` set, we compare it directly to `neworders.drug`.
- Same value → show the letter
- Different value → hide the letter, no further checks

### Rule 2 — Fuzzy match (when liDrugId is absent)

If `additionalInfo.priorAuthData.liDrugId.value` is empty or missing, we look at the OCR drug names on the letter:
- `additionalInfo.ocrData.service.brandDrugName.value`
- `additionalInfo.ocrData.service.genericDrugName.value`

And we compare them against the order's drug names:
- `lidrugs.name` (resolved from `neworders.drug`)
- `lidrugs.genericName` (resolved from `neworders.drug`)

A match means: any order drug name appears inside any letter drug name, case-insensitive. For example, order name `"Ocrevus"` found inside letter brand name `"OCREVUS 300MG"` is a match.
- At least one combination matches → show the letter
- No combination matches → hide the letter

### Rule 3 — Not enough evidence (passthrough)

If `additionalInfo.priorAuthData.liDrugId.value` is absent AND both `brandDrugName.value` and `genericDrugName.value` are also empty — there is simply no drug information to compare. The letter is shown with the understanding that we cannot confirm or deny it belongs to this order.

---

**The short version: verified id beats everything; OCR names are the fallback; no drug info at all means we let it through.**
