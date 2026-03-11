# Commits on `develop` not in `release/2026-03-04`

28 commits total. `develop` is 28 ahead, `release/2026-03-04` is 0 ahead.

## Commit list (newest first)

| # | Hash | Message | Author | Date |
|---|------|---------|--------|------|
| 1 | 697e25d4 | Merge pull request #803 from LocalInfusion/fix/MLID-1780-intakes-search-focus | Alejandro Gomez | 2026-03-04 |
| 2 | eb1fe7c5 | [MLID-1780] - fix(intakes): remove redundant .trim() call | Alejandro Gomez | 2026-03-04 |
| 3 | b2e963b1 | [MLID-1780] - fix(intakes): escape regex metacharacters in search input | Alejandro Gomez | 2026-03-04 |
| 4 | 2ee60028 | [MLID-1780] - fix(intakes): support full name search by concatenating firstName + lastName | Alejandro Gomez | 2026-03-04 |
| 5 | 330bd528 | Merge pull request #802 from LocalInfusion/feat/MLID-1847-update-provider-management-tables | rosacamargo-fortegrp | 2026-03-04 |
| 6 | 042055ec | [MLID-1847] - fix(provider-management): remove dead onView prop and fix index-based React keys | Rosa Camargo | 2026-03-04 |
| 7 | 601c4295 | Merge branch 'feat/MLID-1847-update-provider-management-tables' into feat/MLID-1847-update-provider-management-tables | Rosa Camargo | 2026-03-04 |
| 8 | 79391611 | [MLID-1847] - feat(provider-management): migrate provider office and mapping tables to MUI DataGrid | Rosa Camargo | 2026-03-04 |
| 9 | cedc416b | [MLID-1847] - feat(provider-management): migrate provider office and mapping tables to MUI DataGrid | Rosa Camargo | 2026-03-04 |
| 10 | 3ed7e2d0 | Merge pull request #799 from LocalInfusion/fix/MLID-1780-intakes-search-focus | Alejandro Gomez | 2026-03-04 |
| 11 | 193ec2bf | [MLID-1780] - fix(intakes): enable search by WeInfuse patient ID | Alejandro Gomez | 2026-03-04 |
| 12 | 1f3468d9 | Merge pull request #792 from LocalInfusion/claude/coverage-autopilot-22648602986 | Ruslan Kryvosheiev | 2026-03-04 |
| 13 | d0ee11d2 | Merge branch 'develop' into claude/coverage-autopilot-22648602986 | Ruslan Kryvosheiev | 2026-03-04 |
| 14 | 39d9fe5f | Merge pull request #798 from LocalInfusion/copilot/sub-pr-792 | Ruslan Kryvosheiev | 2026-03-04 |
| 15 | f4300f9f | Merge pull request #797 from LocalInfusion/MLID-1442 | Anatoliy Kolodkin | 2026-03-04 |
| 16 | 746aa9ed | [MLID-791] - fix(tests): use larger token values to avoid rounding totalCostUsd to 0 | copilot-swe-agent[bot] | 2026-03-04 |
| 17 | b91e7ae7 | Merge branch 'claude/coverage-autopilot-22648602986' into copilot/sub-pr-792 | Ruslan Kryvosheiev | 2026-03-04 |
| 18 | 7feb8a5d | Merge branch 'develop' into claude/coverage-autopilot-22648602986 | Ruslan Kryvosheiev | 2026-03-04 |
| 19 | 29fe0756 | Initial plan | copilot-swe-agent[bot] | 2026-03-04 |
| 20 | 5e40dd49 | Merge pull request #796 from LocalInfusion/MLID-1863b | albertobeltran-fortegrp | 2026-03-04 |
| 21 | a880ffb1 | [MLID-1442] - feat(clinical-review): populate dateReceived from document upload date | Anatoliy Kolodkin | 2026-03-04 |
| 22 | e14db58b | MLID-1863b feature: Desctivate auto submit for descriptions with feature flag - Unittests remove dead code | Alberto Beltran | 2026-03-04 |
| 23 | c1374411 | MLID-1863b feature: Desctivate auto submit for descriptions with feature flag - Unittests | Alberto Beltran | 2026-03-04 |
| 24 | 66cdc7dc | MLID-1863b feature: Desctivate auto submit for descriptions with feature flag | Alberto Beltran | 2026-03-04 |
| 25 | e1efe0ba | [Coverage Autopilot] Add unit tests: ai-chat-dashboard routes | claude[bot] | 2026-03-04 |
| 26 | 8c931cc8 | Merge pull request #795 from LocalInfusion/mlid-1804-tab | Egor Dezhic | 2026-03-04 |
| 27 | 02f234ec | [MLID-1442] - feat(clinical-review): move Source into Result column as clickable navigation link | Anatoliy Kolodkin | 2026-03-04 |
| 28 | d8f30525 | MLID-1804 same update for prior auth tab preview | Egor Dezhic | 2026-03-04 |

## Grouped by PR / feature

### MLID-1780 — Intakes search fix (PR #803, PR #799)
- `193ec2bf` — enable search by WeInfuse patient ID
- `2ee60028` — support full name search by concatenating firstName + lastName
- `b2e963b1` — escape regex metacharacters in search input
- `eb1fe7c5` — remove redundant .trim() call

### MLID-1847 — Provider management tables (PR #802)
- `cedc416b` — migrate provider office and mapping tables to MUI DataGrid
- `79391611` — migrate provider office and mapping tables to MUI DataGrid
- `042055ec` — remove dead onView prop and fix index-based React keys

### MLID-1442 — Clinical review (PR #797)
- `02f234ec` — move Source into Result column as clickable navigation link
- `a880ffb1` — populate dateReceived from document upload date

### MLID-1863b — Deactivate auto submit for descriptions (PR #796)
- `66cdc7dc` — deactivate auto submit for descriptions with feature flag
- `c1374411` — unittests
- `e14db58b` — unittests remove dead code

### MLID-1804 — Prior auth tab preview (PR #795)
- `d8f30525` — same update for prior auth tab preview

### Coverage autopilot (PR #792, PR #798)
- `e1efe0ba` — add unit tests: ai-chat-dashboard routes
- `29fe0756` — initial plan
- `746aa9ed` — fix(tests): use larger token values to avoid rounding totalCostUsd to 0

## Cherry-pick status

| Feature | Commits to cherry-pick | Status |
|---------|----------------------|--------|
| MLID-1780 | `193ec2bf`, `2ee60028`, `b2e963b1`, `eb1fe7c5` | Pending |
| MLID-1847 | TBD | Pending |
| MLID-1442 | TBD | Pending |
| MLID-1863b | TBD | Pending |
| MLID-1804 | TBD | Pending |
| Coverage autopilot | TBD | Pending |
