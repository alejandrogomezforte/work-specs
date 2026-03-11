# PR: Cherry-pick MLID-1780 intakes search fixes into release/2026-03-04

## Title

[MLID-1780] - fix(intakes): cherry-pick search improvements to release

## Body

### Summary

Cherry-picks 4 commits from `develop` into `release/2026-03-04` to bring the MLID-1780 intakes search fixes into the upcoming production release.

**Commits cherry-picked (chronological order):**

- `193ec2bf` — Enable search by WeInfuse patient ID (exact numeric match)
- `2ee60028` — Support full name search by concatenating firstName + lastName
- `b2e963b1` — Escape regex metacharacters in search input (prevents crashes on special characters)
- `eb1fe7c5` — Remove redundant `.trim()` call

**What these fixes do:**

- **Patient ID search**: Users can now search intakes by WeInfuse patient ID. Numeric input is matched as an exact value against `patient.patientId`, preventing regex injection.
- **Full name search**: Searching "Kristen Johnson" now matches by concatenating `firstName + " " + lastName`, so multi-word queries work as expected.
- **Regex safety**: Special characters like `(`, `)`, `.`, `*` in search input are escaped before being used in regex queries, preventing server errors.
- **Cleanup**: Removed a redundant `.trim()` call since the input is already trimmed upstream.

### Cherry-pick strategy

`develop` is 28 commits ahead of `release/2026-03-04`. A direct merge from develop would bring in unrelated, untested-for-release changes. Cherry-picking applies only the specific diffs needed.

**Why not merge the feature branch directly?**
The feature branch `fix/MLID-1780-intakes-search-focus` is behind both `develop` and the release branch, so merging it would create conflicts with commits already in release.

**Approach chosen (PR-based cherry-pick):**
1. Created branch `cherry-pick/MLID-1780-to-release` off `release/2026-03-04`
2. Cherry-picked the 4 commits (all applied cleanly, zero conflicts)
3. Ran full test suite locally — 606/607 suites passing, 10,381 tests passing (1 pre-existing V8 memory crash unrelated to these changes)
4. Manual UI QA passed — all search scenarios verified
5. Opened this PR for CI validation and reviewable diff before merging

This approach gives us GitHub's revert button as a safety net if anything goes wrong post-merge.

**Note:** PR #755 commits (`58f12e6c`, `ece97b44`) were already in `release/2026-03-04` before the branch was cut, so only PR #799 and PR #803 commits needed cherry-picking.

### Files changed

- `apps/web/app/api/intakes/route.ts`
- `apps/web/app/api/intakes/route.test.ts`

### Test plan

- [x] Intakes route unit tests (13/13 passing)
- [x] Full test suite (606/607 passing — 1 pre-existing failure unrelated to changes)
- [x] Manual UI QA: search by patient ID, full name, partial name, special characters
