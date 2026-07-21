# Financial Calculator — How the Patient Responsibility Is Calculated

This document explains the Financial Review calculator that appears on the order
detail screen, under the **Financial Review** tab.

It describes what the user enters, what the screen shows back, and the exact
arithmetic used to go from one to the other.

Note: there is a second, older financial calculator located on the patient screen.
It is not covered here. This document describes only the calculator on the order
detail screen.

---

## 1. What the calculator does

The user presses **Calculate**, fills in a short form describing the patient's
insurance benefits and the cost of this order, and saves it. The calculator then
estimates how much of that cost the patient is responsible for, split between the
drug itself and the cost of administering it.

Two things are worth knowing before reading further:

- **Every value is typed in by hand.** Nothing is pre-filled from the order, the
  patient record, or any insurance data source. The result is only as accurate as
  the benefits information the user transcribed into the form.
- **Each calculation is saved permanently.** Running a new calculation does not
  replace the previous one. The newest is marked **Latest**, and every earlier
  calculation stays visible as history, with the name of the person who created it
  and the date.

---

## 2. What the form captures

All nine fields are required.

| Field | Description |
| --- | --- |
| **Deductible met** | How much of the plan deductible the patient has already satisfied this benefit year. |
| **Deductible max** | The total plan deductible. Enter `0` if the plan has no deductible. |
| **Patient co-insurance** | The percentage of the cost the patient owes **after** the deductible has been satisfied. |
| **Out-of-pocket met** | How much of the out-of-pocket maximum the patient has already accumulated this benefit year. |
| **Out-of-pocket max** | The plan out-of-pocket maximum. **Entering `0` means "this plan has no out-of-pocket ceiling"** — it does not mean a ceiling of zero. |
| **Drug cost** | The billed cost of the drug for this order. |
| **Admin cost** | The billed cost of administering the drug: chair time, nursing, supplies. |
| **Assistance for drug** | Copay assistance or foundation support applied against the drug portion. |
| **Assistance for admin** | Copay assistance applied against the administration portion. |

---

## 3. What the screen shows back

### General view

| Shown | Meaning |
| --- | --- |
| **Total Responsibility** | The full amount the patient is estimated to owe for this order. |
| **Drug resp.** | The part of the total that comes from the drug. |
| **Admin resp.** | The part of the total that comes from administering the drug. |

### Detail view

The detail view is laid out in two columns: what was entered on the left, what the
calculator produced on the right. A single bar sits above both columns.

**Drug vs. Admin split** — spans the full width, above both columns. A bar showing
what share of the total comes from the drug and what share from administration. It
appears only when both parts are above zero.

#### Inputs column

| Section | Meaning |
| --- | --- |
| **INPUTS** | The nine values that were entered, repeated back as a permanent record of what this calculation was based on. |

#### Outputs column

| Section | Meaning |
| --- | --- |
| **DRUG** | The step-by-step trace of how the drug figure was reached. |
| **ADMIN** | The same trace for the administration figure. |
| **SUMMARY** | The three final numbers: admin responsibility, drug responsibility, total responsibility. |
| **REASON** | A short paragraph restating the calculation in prose. |

One clarification about the **REASON** paragraph: it is written by an artificial
intelligence service. That service is given the already-finished calculation steps
and asked to restate them in a few sentences. It never produces a number and never
changes one — it only rewords what the calculation already decided. If that service
is temporarily unavailable, the calculation is still saved and displayed correctly,
just without the paragraph.

---

## 4. The calculation

### Step 0 — three situations where the answer is immediately zero

Before any arithmetic runs, three situations are checked, in the order listed below.
If any of them applies, the drug responsibility, the admin responsibility and the
total are all **$0.00**, and the detail view shows a short explanation in the
SUMMARY section instead of the step-by-step DRUG and ADMIN traces, which are empty.

1. **The out-of-pocket maximum is already reached.** The patient has already paid
   their annual ceiling, so they owe nothing further this year.
2. **The deductible is met and co-insurance is 0%.** There is nothing left for the
   plan to charge the patient.
3. **There is no deductible and co-insurance is 0%.** The plan charges the patient
   nothing at all.

### Step 1 — work out how much room is left

Two running amounts are established. Both are shared across the drug portion and
the admin portion, and both are consumed as the calculation proceeds.

They work like a balance that gets spent down. Each step reads the amount left by
the previous step and produces a new, smaller amount for the next one. Wherever a
formula below says **"remaining ... before"** it means the value the step started
with, and **"remaining ... after"** means the value it leaves behind.

```
Remaining deductible     =  Deductible max  −  Deductible met
                            (never below 0)

Remaining out-of-pocket  =  Out-of-pocket max  −  Out-of-pocket met
                            (never below 0)
```

These two are the starting values. The administration portion runs first and uses
them as its "before" amounts; whatever it leaves behind becomes the "before"
amounts for the drug portion.

If **Out-of-pocket max** was entered as `0`, the remaining out-of-pocket is treated
as unlimited: the cap in step 2e never applies, and step 2g never reduces anything.

### Step 2 — the same seven steps are applied twice: **admin first, then drug**

The order matters. The administration portion is calculated **before** the drug
portion, so administration consumes the remaining deductible and the remaining
out-of-pocket room first, and the drug portion works with whatever is left.

A presentation note: the detail view lists the **DRUG** block above the **ADMIN**
block, which is the opposite of the order in which they are actually calculated.

For one portion — using that portion's cost and its assistance amount:

**2a. If the cost is zero, stop here.** The patient owes nothing for this portion
and nothing is consumed.

**2b. Apply the deductible.** The part of the cost that falls inside the remaining
deductible is charged to the patient **in full**. Co-insurance does not apply to it.

The starting point is the remaining deductible left by step 1, or by the previous
portion. First decide how much of this portion's cost falls inside it, then spend
it down:

```
Applied to deductible       =  the smaller of ( Cost , Remaining deductible before )

Cost above deductible       =  Cost  −  Applied to deductible

Remaining deductible after  =  Remaining deductible before  −  Applied to deductible
```

If the cost is smaller than the remaining deductible, the whole cost goes to the
deductible and nothing is left above it. If the remaining deductible is already
zero, nothing is applied and the full cost sits above the deductible.

**2c. Apply co-insurance to whatever is left above the deductible.**

```
Co-insurance amount  =  Cost above deductible  ×  Co-insurance percentage
```

**2d. Add the two together.**

```
Responsibility before assistance  =  Applied to deductible  +  Co-insurance amount
```

**2e. Limit it by the remaining out-of-pocket.** The patient can never be charged
past their annual ceiling.

```
Capped responsibility  =  the smaller of ( Responsibility before assistance , Remaining out-of-pocket before )
```

**2f. Apply copay assistance.** Assistance can never be larger than what the
patient actually owes. There is no refund and no leftover credit carried to the
next calculation.

```
Assistance applied  =  the smaller of ( Assistance , Capped responsibility )

Patient pays  =  Capped responsibility  −  Assistance applied
```

**2g. Reduce the remaining out-of-pocket — by the amount from step 2e, before
assistance was applied.**

```
Remaining out-of-pocket after  =  Remaining out-of-pocket before −  Capped responsibility
                                  (never below 0)
```

This is the single most important rule in the calculator, and the one most likely
to raise a question:

> **Copay assistance lowers what the patient pays, but it does not lower the
> credit applied toward the out-of-pocket maximum.** The patient still receives
> full out-of-pocket credit for the dollars that assistance covered.

### Step 3 — the drug portion, then the total

After the administration portion is finished:

- If administration used up all the remaining out-of-pocket, the drug portion is
  **skipped entirely** and set to **$0.00**. The patient has reached their ceiling.
- Otherwise the drug portion runs through the same steps 2a to 2g, starting from
  the amounts administration left behind.

```
Total responsibility  =  Drug responsibility  +  Admin responsibility
```

All amounts are rounded to two decimal places for display.

---

## 5. A worked example

These are the values from a real saved calculation.

**What was entered**

```
Deductible met      $1,200.00        Drug cost            $5,000.00
Deductible max        $200.00        Admin cost              $10.00
Co-insurance               10%       Assistance for drug     $10.00
Out-of-pocket met     $333.00        Assistance for admin     $0.00
Out-of-pocket max  $10,000.00
```

**Step 1 — room remaining**

```
Remaining deductible     =  $200.00  −  $1,200.00  =  −$1,000.00
                         →  never below 0          =       $0.00
                            (the deductible is already fully met)

Remaining out-of-pocket  =  $10,000.00  −  $333.00  =   $9,667.00
```

**Step 2 — administration first** (cost $10.00, assistance $0.00)

```
Applied to deductible    =  $0.00                       (deductible already met)
Co-insurance             =  $10.00  ×  10%    =  $1.00
Responsibility           =  $1.00
Limited by out-of-pocket =  smaller of $1.00 and $9,667.00   =  $1.00
Assistance applied       =  $0.00

Patient pays for admin   =  $1.00

Remaining out-of-pocket  =  $9,667.00  −  $1.00  =  $9,666.00
```

**Step 3 — drug second** (cost $5,000.00, assistance $10.00)

```
Applied to deductible    =  $0.00                       (deductible already met)
Co-insurance             =  $5,000.00  ×  10%  =  $500.00
Responsibility           =  $500.00
Limited by out-of-pocket =  smaller of $500.00 and $9,666.00  =  $500.00
Assistance applied       =  smaller of $10.00 and $500.00     =  $10.00

Patient pays for drug    =  $500.00  −  $10.00  =  $490.00

Remaining out-of-pocket  =  $9,666.00  −  $500.00  =  $9,166.00
```

Notice the final line. The out-of-pocket dropped by **$500.00**, not by the
$490.00 the patient actually paid. That is the assistance rule from step 2g.

**Result shown on screen**

```
Patient Admin Responsibility     $1.00
Patient Drug Responsibility    $490.00
Total Responsibility           $491.00

Drug vs. Admin split        100% / 0%     (rounded; the true split is 99.8% / 0.2%)
```

---

## 6. Decisions worth confirming

These are rules the calculator applies today that are not visible on the screen.
They are the most likely subjects of a follow-up question.

1. **Administration is calculated before the drug.** Whichever portion is
   calculated first consumes the remaining deductible and out-of-pocket room first.
   This changes how the total is split between the drug figure and the admin figure
   whenever the deductible or the out-of-pocket ceiling is only partly available.
2. **Copay assistance does not reduce out-of-pocket credit.** The patient gets full
   credit toward their annual ceiling for dollars that assistance paid.
3. **An out-of-pocket max of `0` means "no ceiling", not "a ceiling of zero".** If a
   user leaves that field at zero, the patient's responsibility is never capped.
4. **The deductible portion is charged at 100%.** Co-insurance applies only to the
   part of the cost above the remaining deductible.
5. **Assistance is capped at the amount owed.** It never produces a negative balance
   and never carries forward to another calculation.
6. **Nothing is pre-filled.** Every figure is entered manually, every time.
7. **The form does not validate the benefit figures against each other.** A user can
   enter a "deductible met" that is larger than the "deductible max" — as in the
   example above — and the calculator will accept it and treat the deductible as
   fully satisfied, without warning.
8. **The REASON paragraph is generated by artificial intelligence.** It restates the
   finished calculation in prose and should not be treated as the source of truth
   for any number.
