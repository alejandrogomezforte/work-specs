# MLID-2670 — Prior Auth Drug Name Separator-Agnostic Matching

## Task Reference

- **Jira**: [MLID-2670](https://localinfusion.atlassian.net/browse/MLID-2670)
- **Story Points**: 4
- **Branch**: `fix/MLID-2670-prior-auth-drug-name-normalization`
- **Base Branch**: `develop`
- **Status**: In QA

---

## Summary

The order-scoped prior-auth fuzzy match in `buildOrderScopedPriorAuthMatch` drops legitimate intakes when the catalogue drug name and the OCR drug name differ only in separator characters. LISA `lidrugs.name` = "Ocrevus-Zunovo" (hyphen) fails to match OCR `brandDrugName.value` = "Ocrevus Zunovo" (space) because the underlying `$indexOfCP` substring test is sensitive to those separators.

The fix treats the catalogue name (the source of truth) as a sequence of alphanumeric words and matches it against the OCR field with a **case-insensitive regular expression** that allows any run of separator characters — including none — between words. So "Ocrevus Zunovo", "Ocrevus-Zunovo", and "OcrevusZunovo" all match the catalogue "Ocrevus-Zunovo".

---

## Root Cause

In `apps/web/services/priorAuth/query.ts`, `buildContainsClause` performs a raw case-insensitive substring test:

```
{ $gte: [ { $indexOfCP: [ { $toLower: <fieldExpr> }, orderTermLower ] }, 0 ] }
```

The order term is prepared as `orderDrugName.trim().toLowerCase()` — lowercased but separators intact. The document field is also only lowercased via `$toLower`. "ocrevus-zunovo" is never a substring of "ocrevus zunovo", so `$indexOfCP` returns -1 and the intake is excluded.

Ocrevus-Zunovo has no `genericName` in LISA, the order has no generic drug name, and the OCR brand IS present — so neither the generic-fuzzy clause nor the not-enough-evidence passthrough applies. The intake is silently dropped.

---

## Design Decision

### Principle: the catalogue name is the source of truth

The order links to the LISA drug catalogue via `order.drug`; `lidrugs.name` is the correct, canonical drug name. The OCR `brandDrugName.value` / `genericDrugName.value` are approximations an LLM extracted from a faxed PDF. The match must therefore be forgiving of how the OCR text is punctuated, while anchoring on the catalogue name.

### Why `$regexMatch` and not "normalize both sides"

MongoDB aggregation has **no regex-replace operator**. `$replaceAll` swaps only literal strings; it cannot collapse "any run of non-alphanumeric characters" into a single separator. Normalizing both strings to a canonical form (e.g. "ocrevus.zunovo") is therefore awkward to express in aggregation.

MongoDB does have `$regexMatch`, which accepts `options: 'i'` for case-insensitive matching. So instead of normalizing the document field, we build a regular expression from the trusted catalogue name and match it against the raw OCR field:

```
catalogue name "Ocrevus-Zunovo"
  -> lowercase, split on [^a-z0-9]+ -> words ["ocrevus", "zunovo"]
  -> join with the separator class [^a-z0-9]* -> "ocrevus[^a-z0-9]*zunovo"

$regexMatch: {
  input: <ocr brandDrugName, coerced to string>,
  regex: "ocrevus[^a-z0-9]*zunovo",
  options: "i"
}
```

Properties:

- **Separator-agnostic** — `[^a-z0-9]*` between words matches a space, hyphen, underscore, period, repeated separators, or no separator at all (glued words).
- **Case-insensitive** — `options: 'i'`.
- **Contains semantics preserved** — `$regexMatch` is unanchored, so OCR extra words (a dosage suffix, a leading label) still match, matching the previous `$indexOfCP` "contains" behavior.
- **Injection-safe** — after splitting on `[^a-z0-9]+`, each word contains only `[a-z0-9]`, so the built pattern carries no regular-expression metacharacters. The order term is also resolved server-side from the LISA catalogue, not from request input.

### Separator strictness (decided)

Between two catalogue words the regex uses `[^a-z0-9]*` (zero or more separators), so a catalogue "Ocrevus-Zunovo" also matches an OCR value where the separator was dropped ("OcrevusZunovo"). This is the most forgiving choice for genuine separator-only OCR differences while still requiring the catalogue's words to appear, in order. (A single mid-word OCR split such as "Ocre-vus" is intentionally NOT covered — that is a character-level OCR error, out of scope for this hotfix.)

### Worked example (the reported case)

The catalogue drug name (source of truth) for order `6a3d1465f86aefc0df04ca26` is `Ocrevus-Zunovo`. The reported intake `69a59a75a592eac284432c44` has OCR `brandDrugName.value` = `Ocrevus Zunovo`.

How the catalogue name becomes the match expression:

| Step | Value |
|------|-------|
| Catalogue name (`lidrugs.name`) | `Ocrevus-Zunovo` |
| Lowercase | `ocrevus-zunovo` |
| Split on `[^a-z0-9]+` into words | `["ocrevus", "zunovo"]` |
| Join words with `[^a-z0-9]*` | `ocrevus[^a-z0-9]*zunovo` |
| Final `$regexMatch` | `{ input: <ocr brandDrugName>, regex: "ocrevus[^a-z0-9]*zunovo", options: "i" }` |

How the old and new comparison behave for the same catalogue name `Ocrevus-Zunovo` against different OCR values:

| OCR `brandDrugName.value` | Old (`$indexOfCP` substring of `ocrevus-zunovo`) | New (`/ocrevus[^a-z0-9]*zunovo/i`) | Note |
|---|---|---|---|
| `Ocrevus Zunovo` (**the reported case**) | no match | **match** | space vs hyphen — this was the bug |
| `Ocrevus-Zunovo` | match | match | identical separator |
| `Ocrevus_Zunovo` | no match | match | different separator |
| `OCREVUS ZUNOVO` | no match | match | case difference handled by `options: 'i'` |
| `OcrevusZunovo` | no match | match | separator dropped (glued words) |
| `Ocrevus Zunovo 920mg` | no match | match | extra trailing text — unanchored "contains" |
| `Ocre-vus Zunovo` | no match | no match | mid-word split (character-level OCR error, out of scope) |
| `Rituximab` | no match | no match | unrelated drug — not a false positive |

Confirmed on stage: evaluating the new `$regexMatch` against the real intake returns `true` for `Ocrevus Zunovo`, where the old `$indexOfCP` logic returned `false`.

---

## Codebase Analysis

- `apps/web/services/priorAuth/query.ts` — contains `buildContainsClause` (private helper) and `buildOrderScopedPriorAuthMatch` (exported). All production changes are confined to this file.
- `apps/web/utils/helpers/text.ts` — exports `normalizeText(text) => text.toLowerCase().replace(/[^a-z0-9]/g, '')`. Reused here only for the empty-term guard (decide whether a fuzzy clause should be emitted), not for building the match. Already used in `apps/web/services/priorAuth/resolveLisaDrugFromCatalog.ts`.
- `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts` — existing test harness. Uses `build(overrides)` and `getFallbackOr(overrides)` helpers; all assertions are on the returned aggregation-expression object shape (`toEqual`, `toHaveProperty`, `JSON.stringify().toContain`, array length). No in-memory MongoDB — behavioral proof is via the stage aggregation after deploy.
- Two callers of `buildOrderScopedPriorAuthMatch`:
  - `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` (via `buildPriorAuthListPipeline`)
  - `apps/web/app/api/orders/new/[orderId]/auth-review/filter-options/route.ts` (directly)
  - The fix is entirely internal to `query.ts`; neither route file changes.

---

## Implementation Steps

### Step 1 (RED) — Write failing tests for separator-agnostic regex matching

- **Files**: `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts`
- **What**: Add two new test cases inside the existing `describe('buildOrderScopedPriorAuthMatch')` block.

**Test A — brand clause builds a separator-agnostic, case-insensitive regex**

```typescript
it('builds a separator-agnostic regex for the brand-to-brand clause (hyphen vs space)', () => {
  const fallbackOr = getFallbackOr({ orderDrugName: 'Ocrevus-Zunovo', orderDrugGenericName: '' });
  // passthrough + brand fuzzy + not-enough-evidence = 3
  expect(fallbackOr).toHaveLength(3);

  const brandClause = fallbackOr[1];
  const brandJson = JSON.stringify(brandClause);

  // Uses a regex, not a literal substring test
  expect(brandJson).toContain('$regexMatch');
  // Words joined by the zero-or-more separator class
  expect(brandJson).toContain('ocrevus[^a-z0-9]*zunovo');
  // Case-insensitive
  expect(brandJson).toContain('"options":"i"');
  // The old hyphen-sensitive literal must NOT be present
  expect(brandJson).not.toContain('"ocrevus-zunovo"');
});
```

**Test B — generic clause builds the same regex shape**

```typescript
it('builds a separator-agnostic regex for the generic-to-generic clause', () => {
  const fallbackOr = getFallbackOr({ orderDrugGenericName: 'Drug-Generic' });
  // passthrough + brand fuzzy + generic fuzzy = 3
  expect(fallbackOr).toHaveLength(3);

  const genericClause = fallbackOr[2];
  const genericJson = JSON.stringify(genericClause);

  expect(genericJson).toContain('$regexMatch');
  expect(genericJson).toContain('drug[^a-z0-9]*generic');
  expect(genericJson).toContain('"options":"i"');
});
```

Run (from `apps/web/`):
```
npx jest --testPathPattern="query.orderScoped" --no-coverage
```

Expected: the two new tests FAIL (current code emits `$indexOfCP`, not `$regexMatch`).

---

### Step 2 (GREEN) — Implement `$regexMatch` matching in `query.ts`

- **Files**: `apps/web/services/priorAuth/query.ts`

**Change 1 — import `normalizeText` (for the empty-term guard only)**

```typescript
import { normalizeText } from '@/utils/helpers/text';
```

**Change 2 — rewrite `buildContainsClause` to build a separator-agnostic regex**

```typescript
/**
 * Case-insensitive, separator-agnostic "contains" expression.
 *
 * The catalogue drug name (orderTerm — the source of truth) is split into
 * alphanumeric words and rejoined into a regular expression where any run of
 * non-alphanumeric characters, including none, is allowed between words. OCR
 * punctuation differences ("Ocrevus Zunovo", "Ocrevus-Zunovo", "OcrevusZunovo")
 * therefore all match. The match is unanchored (contains) and case-insensitive.
 *
 * The field is coerced to '' unless it is genuinely a string — $regexMatch
 * throws on non-string input (some documents store an array).
 *
 * Each word contains only [a-z0-9] after the split, so the built pattern has no
 * regex metacharacters: it is injection-safe.
 */
function buildContainsClause(
  docFieldPath: string,
  orderTerm: string
): Record<string, unknown> {
  const fieldAsString = {
    $cond: [{ $eq: [{ $type: docFieldPath }, 'string'] }, docFieldPath, ''],
  };

  const words = orderTerm
    .toLowerCase()
    .split(/[^a-z0-9]+/)
    .filter(Boolean);
  const pattern = words.join('[^a-z0-9]*');

  return {
    $regexMatch: { input: fieldAsString, regex: pattern, options: 'i' },
  };
}
```

**Change 3 — normalize the order terms only for the empty-term guard, pass the raw name to the builder**

In `buildOrderScopedPriorAuthMatch`, replace:

```typescript
const orderNameLower = orderDrugName.trim().toLowerCase();
const orderGenericLower = orderDrugGenericName.trim().toLowerCase();
```

with:

```typescript
const orderNameNormalized = normalizeText(orderDrugName);
const orderGenericNormalized = normalizeText(orderDrugGenericName);
```

Then update the two fuzzy emissions and the not-enough-evidence branch to guard on the normalized values and pass the raw names to `buildContainsClause`:

```typescript
if (orderNameNormalized !== '') {
  fallbackOrClauses.push(buildContainsClause(BRAND_DRUG_NAME_PATH, orderDrugName));
}

if (orderGenericNormalized !== '') {
  fallbackOrClauses.push(buildContainsClause(GENERIC_DRUG_NAME_PATH, orderDrugGenericName));
} else {
  // No generic name on the order: the generic-vs-generic comparison cannot be
  // made. When the intake also has no brand name, there is no basis to exclude
  // the document — show it.
  fallbackOrClauses.push({ $in: [brandDrugNameValue, [null, '']] });
}
```

Guarding on `normalizeText(...) !== ''` prevents emitting a clause for an order term that has no alphanumeric content (which would otherwise produce an empty pattern that matches everything).

Run the same test command. Expected: the two new tests pass.

---

### Step 3 (REFACTOR) — Update existing shape-assertion tests to the `$regexMatch` shape

- **Files**: `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts`
- **What**: The fuzzy-match assertions in the pre-existing tests look for the old `$indexOfCP` / `$toLower` / `$gte` shape and for the lowercased literal order term. Update them to the new shape:
  - Replace assertions that reference `$indexOfCP` (or `$toLower` inside a fuzzy clause) with `$regexMatch`.
  - The existing single-word drug names "Privigen" and "Globulin" produce single-word patterns ("privigen", "globulin") with no separator class, so assertions like `toContain('privigen')` and the cross-contamination checks (`brandClause` not containing "globulin", `genericClause` not containing "privigen") remain valid.
  - Keep all non-fuzzy assertions (date condition, hard-id branch, passthrough, not-enough-evidence, branch counts) unchanged — those expressions are untouched by this fix.

Run the order-scoped suite, then the broader prior-auth suite:

```
npx jest --testPathPattern="query.orderScoped" --no-coverage
npx jest --testPathPattern="priorAuth" --no-coverage
```

Expected: all tests pass.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts` | Modify | Add two RED tests (Step 1); update existing fuzzy-clause assertions to `$regexMatch` (Step 3) |
| `apps/web/services/priorAuth/query.ts` | Modify | Import `normalizeText`; rewrite `buildContainsClause` to build a separator-agnostic case-insensitive regex; guard order terms with `normalizeText` |

No route files change. No new files are created.

---

## Testing Strategy

### Unit tests (shape assertions — no in-memory MongoDB)

All in `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts`.

**New tests (RED phase):**

- "builds a separator-agnostic regex for the brand-to-brand clause (hyphen vs space)" — with `orderDrugName` = "Ocrevus-Zunovo", asserts the brand clause uses `$regexMatch`, contains the pattern `ocrevus[^a-z0-9]*zunovo`, sets `options: 'i'`, and does NOT contain the old hyphen-sensitive literal.
- "builds a separator-agnostic regex for the generic-to-generic clause" — same for the generic path with `orderDrugGenericName` = "Drug-Generic".

**Existing tests that must continue to pass (regression):**

- "combines date and drug conditions under `$expr.$and`"
- "date condition parses dateOfDecision with createdAt fallback"
- "hard id match compares (liDrugId ?? null) to orderDrugId"
- "fallback is gated by liDrugId being null/empty"
- "passthrough requires BOTH brand and generic to be null/empty"
- "fuzzy-matches brand-to-brand and generic-to-generic when both order names are set" (fuzzy assertions updated to `$regexMatch` in Step 3)
- "replaces the generic fuzzy clause with a not-enough-evidence passthrough when the order has no generic name"
- "emits the complete passthrough and a not-enough-evidence passthrough when the order has no names at all"
- "omits the hard id branch when the order has no drug id"

**Test command:**

```
# from apps/web/
npx jest --testPathPattern="query.orderScoped" --no-coverage
```

### Manual / stage verification

Re-run the aggregation that proved the bug, against the **`intakes`** collection (the route uses `Intake.aggregate`), scoped to:

- Order: `6a3d1465f86aefc0df04ca26` (catalogue drug "Ocrevus-Zunovo", `createdAt` 2026-01-01)
- Intake: `69a59a75a592eac284432c44` (OCR brand "Ocrevus Zunovo", no `dateOfDecision`)

Before the fix the intake was excluded because the brand fuzzy clause failed. After the fix the regex `ocrevus[^a-z0-9]*zunovo` (case-insensitive) matches "Ocrevus Zunovo", so the intake survives the order-scoped match.

Also verify in the UI: open the order's Auth Review tab in stage and confirm the Ocrevus-Zunovo intake is listed. The user runs the dev server and performs all browser verification (no Playwright).

---

## Security Considerations

- **Regex injection**: The pattern is built from words that contain only `[a-z0-9]` after splitting on `[^a-z0-9]+`, so no regular-expression metacharacters reach `$regexMatch`. The order term is resolved server-side from the LISA catalogue, not from request input.
- **Non-string field safety**: The `$type`-string guard remains; non-string OCR fields are coerced to `''` so `$regexMatch` does not throw.
- **Empty-pattern safety**: Fuzzy clauses are only emitted when `normalizeText(orderTerm) !== ''`, preventing an empty pattern that would match every document.
- **Authorization / PHI**: No changes to auth, permissions, or PHI handling. Drug names are not PHI.
- **False-positive risk**: The match still requires the catalogue's words to appear, in order, within the OCR field; `[^a-z0-9]*` only relaxes the separators between those words. It does not match unrelated drugs.

---

## Acceptance Criteria

From the Jira ticket: "If NULL decision date, fax received date after order creation date + drug name matches then a document shall be added to prior auth tab."

Mapped to the implementation:

1. An intake with `dateOfDecision` null and `createdAt` >= `orderCreatedAt` (date gate passes) whose `brandDrugName.value` = "Ocrevus Zunovo" appears in the Auth Review tab for an order whose catalogue drug name is "Ocrevus-Zunovo".
2. All existing prior-auth matching behaviors (hard id match, complete passthrough, not-enough-evidence passthrough, generic-to-generic fuzzy, hard id branch omission) remain unchanged.
3. The fix is transparent to both callers (`auth-review/route.ts` and `auth-review/filter-options/route.ts`); no route changes are required.
