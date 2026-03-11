# MLID-1780 — Cherry-pick to `release/2026-03-04`

## Context

Branch `fix/MLID-1780-intakes-search-focus` contains fixes for the intakes search feature (MLID-1780). The work was merged to `develop` across three PRs (#755, #799, #803). The `release/2026-03-04` branch represents the code going to production on 2026-03-05. We need to bring the MLID-1780 fixes into the release without pulling in the other 24 unrelated commits sitting on `develop`.

## Why cherry-pick

- `develop` is **28 commits ahead** of `release/2026-03-04` — merging develop would bring unwanted, untested-for-release code
- Cherry-pick applies only the specific diffs we need — no extra history, no risk from unrelated changes
- The feature branch itself is behind both `develop` and `release/2026-03-04`, so merging it directly is not an option

## All PRs and commits (branch history)

### PR #755 — Initial fix (search input focus)
| Hash | Message | Cherry-pick needed? |
|------|---------|-------------------|
| `58f12e6c` | fix(intakes): search input loses focus after each keystroke | **No** — already in `release/2026-03-04` |
| `ece97b44` | fix(intakes): address PR review comments | **No** — already in `release/2026-03-04` |
| `c9b08da7` | Merge branch 'develop' into fix/MLID-1780-intakes-search-focus | **No** — merge commit, skip |

### PR #799 — WeInfuse patient ID search
| Hash | Message | Cherry-pick needed? |
|------|---------|-------------------|
| `193ec2bf` | fix(intakes): enable search by WeInfuse patient ID | **Yes** |

### PR #803 — Full name search, regex escaping, cleanup
| Hash | Message | Cherry-pick needed? |
|------|---------|-------------------|
| `2ee60028` | fix(intakes): support full name search by concatenating firstName + lastName | **Yes** |
| `b2e963b1` | fix(intakes): escape regex metacharacters in search input | **Yes** |
| `eb1fe7c5` | fix(intakes): remove redundant .trim() call | **Yes** |

### Why PR #755 is excluded

PR #755 commits (`58f12e6c`, `ece97b44`) were merged to `develop` **before** the `release/2026-03-04` branch was cut. Verified with:

```bash
git branch -r --contains 58f12e6c | grep release/2026-03-04  # found
git branch -r --contains ece97b44 | grep release/2026-03-04  # found
```

Both are already in the release branch. Only **4 commits** need cherry-picking.

## Commits to cherry-pick (in chronological order)

```
193ec2bf  [MLID-1780] - fix(intakes): enable search by WeInfuse patient ID
2ee60028  [MLID-1780] - fix(intakes): support full name search by concatenating firstName + lastName
b2e963b1  [MLID-1780] - fix(intakes): escape regex metacharacters in search input
eb1fe7c5  [MLID-1780] - fix(intakes): remove redundant .trim() call
```

Files affected: `apps/web/app/api/intakes/route.ts`, `apps/web/app/api/intakes/route.test.ts`

---

## Strategy: Plan A vs Plan B

### Plan A — Copy branch, cherry-pick, reproduce

1. Copy release branch into `release/2026-03-04/copy`
2. Cherry-pick the 4 commits into the copy
3. Resolve conflicts if any
4. Run tests locally, manual UI QA
5. If everything passes, redo the cherry-picks on the real `release/2026-03-04` branch
6. Push to remote

**Pros:**
- Release branch is untouched until you're confident

**Cons:**
- Cherry-pick happens **twice** — the second attempt on the real branch could differ if someone pushes to `release/2026-03-04` in between
- No GitHub safety net: no PR means no revert button, no CI checks, no reviewable diff
- The copy branch is throwaway noise in the repo

### Plan B — PR-based cherry-pick (chosen)

1. Create a branch off `release/2026-03-04` named `cherry-pick/MLID-1780-to-release`
2. Cherry-pick the 4 commits (oldest first), resolve conflicts
3. Run tests locally, manual UI QA on that branch
4. Push the branch and open a PR targeting `release/2026-03-04`
5. CI runs, diff is reviewable
6. Merge the PR when everything is green

**Pros:**
- Cherry-pick happens **once** — the PR merges the exact code you tested
- GitHub **Revert button** — if anything goes wrong in production, one click creates a revert PR
- **CI runs automatically** on the PR — automated safety before merge
- **Reviewable diff** — you or a teammate can inspect exactly what enters the release
- If unhappy, just delete the branch — release is untouched
- Standard industry workflow for hotfixes into release branches

**Cons:**
- None significant — all the safety of Plan A is preserved, with additional guardrails

### Why Plan B wins

The core concern is: *"I don't want to commit risky code that can break production."*

Plan B addresses this better than Plan A because:

1. **Same local safety** — you still test locally before anything touches the release branch
2. **Extra GitHub safety** — PR provides CI, code review, and a one-click revert mechanism
3. **No double work** — cherry-pick once, not twice
4. **No divergence risk** — no gap between "what I tested" and "what I merged"

---

## Execution steps (Plan B)

```bash
# 1. Fetch latest
git fetch origin

# 2. Create branch off release
git checkout -b cherry-pick/MLID-1780-to-release origin/release/2026-03-04

# 3. Cherry-pick the 4 commits (chronological order)
git cherry-pick 193ec2bf
git cherry-pick 2ee60028
git cherry-pick b2e963b1
git cherry-pick eb1fe7c5

# If conflicts on any step:
#   - resolve conflicts
#   - git add <resolved-files>
#   - git cherry-pick --continue

# 4. Run tests locally
cd apps/web && npm run test -- --testPathPattern="intakes"

# 5. Manual UI QA (start dev server, test intakes search)
npm run dev:web

# 6. Push and open PR → release/2026-03-04
git push -u origin cherry-pick/MLID-1780-to-release
# Open PR on GitHub targeting release/2026-03-04

# 7. Merge PR when CI is green and QA passes
```

---

## Execution log

### Step 1-2: Branch created — DONE

```
git fetch origin
git checkout -b cherry-pick/MLID-1780-to-release origin/release/2026-03-04
# -> Switched to a new branch 'cherry-pick/MLID-1780-to-release'
# -> branch set up to track 'origin/release/2026-03-04'
```

### Step 3: Cherry-picks — DONE (all clean, zero conflicts)

| Order | Original hash | New hash | Message | Result |
|-------|---------------|----------|---------|--------|
| 1 | `193ec2bf` | `7a12c183` | fix(intakes): enable search by WeInfuse patient ID | Clean |
| 2 | `2ee60028` | `e040e848` | fix(intakes): support full name search by concatenating firstName + lastName | Clean |
| 3 | `b2e963b1` | `b23ff58a` | fix(intakes): escape regex metacharacters in search input | Clean |
| 4 | `eb1fe7c5` | `0a56ce92` | fix(intakes): remove redundant .trim() call | Clean |

### Step 4: Intakes route tests — DONE (13/13 passing)

```
PASS node app/api/intakes/route.test.ts
  GET /api/intakes
    √ should return 401 when not authenticated
    √ should return intakes with pagination when authenticated
    search by WeInfuse patient ID
      √ should include patient.patientId as exact match when search is numeric
      √ should NOT include patient.patientId when search is non-numeric
      √ should reject non-numeric values for patient ID search (no regex injection)
      √ should use exact match (not regex) for patient ID
    search by ObjectId
      √ should search by _id when search term is a valid ObjectId
    search by patient name
      √ should search by firstName and lastName with case-insensitive regex
      √ should treat search with trailing/leading spaces as single word
      √ should match full name against concatenated firstName + lastName
      √ should respect name order — "Johnson Kristen" should not match "Kristen Johnson"
      √ should escape regex special characters in single-word search
      √ should escape regex special characters in full name search

Test Suites: 1 passed, 1 total
Tests:       13 passed, 13 total
```

### Step 4b: Full test suite — DONE (606/607 suites, 10,381/10,383 tests)

```
Test Suites: 1 failed, 606 passed, 607 total
Tests:       2 skipped, 10381 passed, 10383 total
Time:        79.734 s
```

**The single failure is pre-existing and unrelated to our changes:**

- `app/api/jobs/orders-tracker/export/route.test.ts` — V8 memory crash (`Fatal JavaScript invalid size error 174895934`). Jest worker process runs out of memory and crashes before any test assertion runs. Reproduced in isolation with the same result. This is a known flaky test on the release branch, not caused by the cherry-picks.

### Step 5: Manual UI QA — DONE

Verified locally: search by patient ID, full name, partial name, special characters — all working.

### Step 6: Push & open PR → `release/2026-03-04` — DONE

- Branch pushed: `origin/cherry-pick/MLID-1780-to-release`
- PR message saved at: `docs/agomez/PR/MLID-1780-to-release.md`
- Create PR at: https://github.com/LocalInfusion/li-call-processor/pull/new/cherry-pick/MLID-1780-to-release

### Step 7: Merge PR — DONE

PR merged successfully into `release/2026-03-04`.

---

## Final status: SUCCESS

All 4 MLID-1780 commits cherry-picked, tested, QA'd, and merged into `release/2026-03-04` via PR. Ready for production deployment on 2026-03-05.
