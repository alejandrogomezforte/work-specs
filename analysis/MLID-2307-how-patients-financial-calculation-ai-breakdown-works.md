# How the patients financial-calculation AI breakdown works (legacy flow)

Reference investigation for MLID-2307. The new Order Details Financial Review (MLID-2307) is replacing the legacy patient-scoped calculator at `/patient/[id]/financial-calculator`. Before we port the AI breakdown to the order-scoped flow, we need to understand exactly how the existing one is built. This document captures that, end-to-end.

---

## Component architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│ UI  (apps/web/components/Modules/financial-calculator/)                │
│                                                                        │
│   financialCalculator.tsx                                              │
│      └─ reads patient.financialCalculationsHistory from PatientProvider│
│      └─ paginates & renders <CalculatorHistoryItem>s                   │
│      └─ button → opens <CalculatorSidebar> (the drawer form)           │
│                                                                        │
│   calculatorSidebar.tsx (the form)                                     │
│      └─ react-hook-form with 9 fields (calculatorDynamicFormItems.tsx) │
│      └─ submit → useMakeFinancialCalculation().execCalculation(data)   │
│                                                                        │
│   calculatorHistoryItem.tsx (the result card)                          │
│      └─ shows summary numbers + <Expandable>                           │
│      └─ <MarkdownReader> renders the resultBreakDownText blob          │
└─────────────────────────────────┬──────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼──────────────────────────────────────┐
│ Client hook + fetch wrapper                                            │
│                                                                        │
│   utils/hooks/useMakeFinancialCalculation.ts                           │
│      └─ pulls calculationOwner from useSession().data.user.email       │
│      └─ wraps makeFinancialCalculation() in useFetchData()             │
│      └─ on success → fetchUser(patientId) to refresh PatientProvider   │
│                                                                        │
│   utils/api/patient/makeFinancialCalculation.ts                        │
│      └─ plain fetch(): POST /api/patients/{id}/financialCalculation    │
│      └─ body: { ...9 form fields, calculationOwner }                   │
└─────────────────────────────────┬──────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼──────────────────────────────────────┐
│ Server (single file — no service layer for the calc)                   │
│                                                                        │
│   pages/api/patients/[id]/financialCalculation.ts  (350 lines, all-in) │
│      handler                                                           │
│        ├─ getServerSession() check                                     │
│        ├─ calculateMedicalCosts(details)  ◄── DOES THE WORK            │
│        │     ├─ parseFloat all 9 string inputs                         │
│        │     ├─ branch: OOP-already-met / deductible-met-no-coins / …  │
│        │     ├─ processPart('Drug', ...) → mutates `lines: string[]`   │
│        │     ├─ processPart('Admin', ...) → mutates `lines: string[]`  │
│        │     ├─ generateAiReason(lines)   ◄── ONLY AI CALL             │
│        │     └─ wrapResult(drugPay, adminPay, lines)                   │
│        └─ insertFCItem({ patientId, newFCItem })                       │
└─────────────────────────────────┬──────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼──────────────────────────────────────┐
│ MongoDB write                                                          │
│                                                                        │
│   services/mongodb/financialCalculation.ts                             │
│      insertFCItem({ patientId, newFCItem })                            │
│        └─ raw driver: db.collection('patients').updateOne(             │
│              { _id: ObjectId(patientId) },                             │
│              { $push: { financialCalculationsHistory: { _id, ...item }}})│
└────────────────────────────────────────────────────────────────────────┘
```

---

## Function pipeline (server-side, sequential)

The handler runs these in order, all synchronous-await:

1. `getServerSession()` — auth gate, 401 if missing.
2. `calculateMedicalCosts(details)` — the orchestrator (`pages/api/patients/[id]/financialCalculation.ts:77`).
3. Inside it:
    - **Parse**: `parseFloat()` each of the 9 incoming string fields (legacy types them as strings). Coinsurance is divided by 100 to get a 0–1 fraction.
    - **Header bullets pushed onto `lines: string[]`**: `### Inputs:` block with the 9 values formatted as `$X` / `%`.
    - **Three early-exit branches** (all push `### Summary:` + a one-liner and return zero):
        1. OOP already met (`oopMet ≥ oopMax`).
        2. Deductible exists, met, coinsurance = 0%.
        3. No deductible, coinsurance = 0%.
    - **`processPart('Drug', drugCost, drugAssist)`** — closure mutating `remainingDeductible`, `remainingOOP`, and `lines`:
        - Apply cost to remaining deductible.
        - Apply coinsurance to the leftover.
        - Cap `responsibilityBeforeAssist` at `remainingOOP`.
        - Subtract assistance (`Math.min(assistance, cappedResponsibility)`).
        - Push 6–8 bullet lines describing each step.
        - Return `patientPays`.
    - **`processPart('Admin', adminCost, adminAssist)`** — same shape; skipped entirely if `remainingOOP` hit zero during Drug.
    - **`### Summary:` block pushed** with the three final numbers.
    - **`generateAiReason(lines)`** — `await` (synchronous in the request path).
    - **`### Reason:` block** spread onto `lines`.
4. `wrapResult(drugPay, adminPay, lines)` — packages output as:
    ```ts
    {
      explanation: lines.join('\n'),         // ← the markdown blob
      totalResponsibility:        { title, value: total.toFixed(2) },
      patientDrugResponsibility:  { title, value: drugPay.toFixed(2) },
      patientAdminResponsibility: { title, value: adminPay.toFixed(2) },
    }
    ```
5. Handler builds `newFCItem = { summary: { calculationOwner, calculationDate, ...result }, resultBreakDownText: explanation }`.
6. `insertFCItem({ patientId, newFCItem })` — raw-driver `$push`. Done.

---

## The Azure OpenAI call — specifics

Located at `pages/api/patients/[id]/financialCalculation.ts:41-75` (`generateAiReason`).

### Client construction — instantiated per-call (no singleton):

```ts
new AzureOpenAI({
  endpoint:   process.env.AZURE_OPENAI_ENDPOINT,
  apiKey:     process.env.AZURE_OPENAI_API_KEY,
  apiVersion: process.env.AZURE_OPENAI_API_VERSION || '2024-08-01-preview',
});
```

Note: 7 other places in the codebase use `AzureOpenAI` (e.g. `services/clinicalReview/summary/summaryAiService.ts`), and at least some wrap construction in a `getAzureOpenAiClient()` factory with proper error handling. This file does NOT — it would `throw` mid-handler if the env vars are blank.

### Deployment — env-overridable, defaults to `'gpt-5-nano'`:

```ts
process.env.AZURE_OPENAI_TINY_DEPLOYMENT_NAME || 'gpt-5-nano'
```

### Prompt

- **System**: `"You generate a concise, informative reasonings for based on insurance calculation steps and results."` (yes, that's literally what's in the code — there's a typo)
- **User**: `"Give a SHORT reason for the calculation results (LESS THAN 6 SENTENCES WITH NO BULLET POINTS), including the numerical values (no vague language):\n\n${summary.join('\n')}"` — where `summary` is the entire `lines: string[]` accumulated so far (Inputs + Drug + Admin + Summary blocks).

### Request

```ts
{
  model: deployment,
  max_completion_tokens: 4096,
  messages: [
    { role: 'system', content: systemPrompt },
    { role: 'user',   content: userPrompt },
  ],
}
```

No temperature, no top_p, no streaming, no response_format. Default settings everywhere else.

### Response handling

```ts
const content = completion.choices[0].message.content || '';
return content.split('\n');
```

Returns a `string[]`, but immediately downstream:

```ts
lines.push('', '### Reason:', ...aiReason);
return wrapResult(..., lines);
// wrapResult: explanation: lines.join('\n')
```

So the `string[]` is split-then-spread-then-joined — the array structure is essentially round-tripped to nothing. The persisted value is just one big markdown string.

### No retry, no timeout, no fallback

If Azure errors, the whole `POST` errors (handler's outer `try/catch` returns 500).

---

## Data flow timeline (one save)

```
T0    User clicks Calculate in CalculatorSidebar
T0    useMakeFinancialCalculation.execCalculation(data) called
T0    LoadingSpinner shown (calculationExecData.isLoading=true)
T0    fetch POST /api/patients/{id}/financialCalculation
T0+   Server validates session
T0+   Server runs deterministic math (microseconds) — Inputs/Drug/Admin/Summary bullets
T0+s  ⚠️ Server awaits Azure OpenAI gpt-5-nano (typically 1-4s) — blocking
T0+s  Server appends ### Reason: aiReason to lines
T0+s  Server insertFCItem — raw-driver $push (tens of ms)
T0+s  Server returns 200
T0+s  Hook calls fetchUser(patientId) — refetches patient doc
T0+s  PatientProvider updates → list re-renders with new item
T0+s  Drawer closes
```

---

## Five observations that matter for our Stage 2 schema

1. **Only the "Reason" paragraph is actually AI-generated.** The Inputs / Drug / Admin / Summary bullets are produced by the deterministic `processPart()` closure. So a planned `aiCalculations: { drug[], admin[], summary[], reason }` field is misleadingly named — three of those four fields are NOT AI output, they're produced by `calculate.ts` (the Stage 1 pure function). Only `reason` is the AI piece.

2. **The legacy schema persists the bullets as one markdown blob** (`resultBreakDownText`). To structure them as arrays (which is already decided for Stage 2), we need to capture them at generation time from inside `processPart()`, not parse the markdown back out.

3. **Inputs arrive as strings, are `parseFloat`'d.** The legacy `FinancialCalculationDetails` type has all-string fields. The new schema has all-number fields, so the `POST` body should be numbers — Mongoose will reject strings via the schema validators.

4. **Sync AI call is the dominant request latency.** Deterministic math is microseconds; Azure OpenAI is 1–4 seconds (sometimes more). The legacy UI just shows a `LoadingSpinner` for that whole time. Two routes for Stage 2:
    - **Sync** (mirror legacy) — `POST` waits on Azure, returns a fully-populated calculation. Simple, brittle, user waits.
    - **Deferred enrichment** — `POST` saves with `aiCalculations.reason = null`, returns immediately. A background job (or `aiCalculations` endpoint) populates the reason later, optionally pushed via SignalR. Two-step UX but resilient if Azure is slow/down.

5. **No graceful degradation today.** If Azure fails, the calculation is lost (the deterministic math result is discarded along with the AI error). Stage 2 should at least persist the calc and mark the AI as failed/pending — never lose the deterministic data because the LLM hiccuped.

---

## Implications for Stage 2 design

- **Rename `aiCalculations`** to something more accurate. Candidates: `breakdown` (for the deterministic bullets) + `aiReason: string` (for the LLM output) as two sibling fields. Or keep one block but split it internally as `{ deterministic: { drug, admin, summary }, ai: { reason } }`. Worth a quick design call.
- **Move the Azure client to a shared service** following `summaryAiService.ts`'s factory pattern — `services/openai/getAzureOpenAiClient.ts` already kind of exists as a pattern in 7 other places. Don't inline another per-call construction.
- **Pick sync vs. deferred** for the AI call. My read: sync is fine for Stage 2 (matches legacy UX, users already accept the wait), but the handler must save the deterministic calc even if AI fails (graceful degradation that legacy lacks).
- **Fix or rewrite the prompt** — there's a typo (`reasonings for based on`) and no structured-output guidance. Worth tightening when we port it.
