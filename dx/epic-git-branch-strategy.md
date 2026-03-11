# Epic - Git Branch Strategy

## Overview

This document defines the git branch strategy for **epics** — large bodies of work composed of multiple Jira tasks that are grouped into **deliverables**. Each deliverable represents a deployable increment that QA can test end-to-end in staging.

An intermediate **epic branch** sits between `develop` and individual sub-task branches, allowing incremental integration without polluting the main branch.

---

## Branch Hierarchy

```
develop (deployed to staging)
  |
  +-- epic/MLID-XXXX-short-epic-name              (epic integration branch)
        |
        +-- feature/MLID-YYYY-short-description    (sub-task branch)
        +-- feature/MLID-ZZZZ-short-description    (sub-task branch)
        +-- fix/MLID-WWWW-short-description         (sub-task branch)
```

- **Epic branch**: long-lived branch that accumulates all sub-task work
- **Sub-task branches**: short-lived branches for individual Jira tasks, created from and merged back into the epic branch

---

## Flow

```
1. Epic branch is created FROM develop
2. Sub-task branches are created FROM the epic branch
3. Sub-task branches are merged INTO the epic branch (via merge --no-ff)
4. When a deliverable group is complete on the epic branch → PR from epic → develop
5. After PR merges to develop → rebase epic branch on develop before starting next deliverable
6. Repeat steps 2-5 for each deliverable
7. After the final deliverable merges → delete the epic branch
```

---

## Deliverable Groups

Before starting development, break the epic into deliverable groups:

| Property | Description |
|----------|-------------|
| **Self-contained** | Each deliverable produces something QA can test in staging |
| **Ordered** | Deliverables are merged to `develop` in sequence (D1 → D2 → D3 → ...) |
| **Scope** | Each deliverable bundles related sub-tasks that together form a testable feature slice |

**Example grouping:**

```
D1: Backend schemas + API + list page + search + create modal     → "List View"
D2: Detail page + edit modal + related data tab                   → "Detail View"
D3: Bulk operations + validation + confirmation flow              → "Bulk Update"
```

> **Tip:** A sub-task that can't be tested in isolation (e.g., a schema or service layer) should be grouped with the sub-tasks that make it testable.

---

## Step-by-Step Workflow

### 1. Creating the epic branch

```bash
git checkout develop
git pull origin develop
git checkout -b epic/MLID-XXXX-short-epic-name
git push origin epic/MLID-XXXX-short-epic-name
```

### 2. Starting a sub-task

```bash
git checkout epic/MLID-XXXX-short-epic-name
git pull origin epic/MLID-XXXX-short-epic-name
git checkout -b feature/MLID-YYYY-short-description
```

### 3. Working on a sub-task

```bash
# Make changes, commit often
git add <files>
git commit -m "[MLID-YYYY] - type(scope): description"

# Push to remote regularly
git push origin feature/MLID-YYYY-short-description
```

### 4. Finishing a sub-task (merge into epic)

```bash
git checkout epic/MLID-XXXX-short-epic-name
git pull origin epic/MLID-XXXX-short-epic-name
git merge --no-ff feature/MLID-YYYY-short-description
git push origin epic/MLID-XXXX-short-epic-name

# Clean up sub-task branch
git branch -d feature/MLID-YYYY-short-description
git push origin --delete feature/MLID-YYYY-short-description
```

> **Why `--no-ff`?** It preserves the sub-task as a distinct merge commit in the epic branch history, making it easy to identify what each sub-task contributed.

### 5. Finishing a deliverable (PR to develop)

Once all sub-tasks for a deliverable are merged into the epic branch:

```bash
# Rebase epic on latest develop before creating PR
git checkout develop
git pull origin develop
git checkout epic/MLID-XXXX-short-epic-name
git rebase develop
git push origin epic/MLID-XXXX-short-epic-name --force-with-lease

# Create PR
gh pr create --base develop --head epic/MLID-XXXX-short-epic-name \
  --title "[MLID-XXXX] feat: Deliverable Name (DN - component list)" \
  --body "## Summary
- What this deliverable includes

## Sub-tasks included
- [MLID-YYYY] Task description
- [MLID-ZZZZ] Task description

## QA test scenarios
- [ ] Scenario 1
- [ ] Scenario 2

## Jira
- [MLID-XXXX](https://localinfusion.atlassian.net/browse/MLID-XXXX)"
```

### 6. After PR merges (prepare for next deliverable)

```bash
git checkout develop
git pull origin develop
git checkout epic/MLID-XXXX-short-epic-name
git rebase develop
git push origin epic/MLID-XXXX-short-epic-name --force-with-lease
```

> This ensures the epic branch starts the next deliverable with a clean base that includes everything already merged to `develop`.

### 7. After the final deliverable merges

```bash
git checkout develop
git pull origin develop
git branch -d epic/MLID-XXXX-short-epic-name
git push origin --delete epic/MLID-XXXX-short-epic-name
```

---

## Branch Naming Convention

| Type | Pattern | Example |
|------|---------|---------|
| Epic | `epic/MLID-XXXX-short-epic-name` | `epic/MLID-1021-hospital-systems-340B` |
| Feature sub-task | `feature/MLID-YYYY-short-description` | `feature/MLID-1548-hospital-list-page` |
| Fix sub-task | `fix/MLID-YYYY-short-description` | `fix/MLID-1560-fix-provider-count` |
| Refactor sub-task | `refactor/MLID-YYYY-short-description` | `refactor/MLID-1561-extract-npi-util` |

**Rules:**
- Epic branch uses the **epic Jira key** (`MLID-XXXX`)
- Sub-task branches use the **sub-task Jira key** (`MLID-YYYY`)
- Use lowercase kebab-case for descriptions
- Keep descriptions short (3-5 words max)

---

## Commit Message Convention

Format: `[MLID-YYYY] - type(scope): description`

Commits always reference the **sub-task** Jira key, not the epic key.

| Type | When to use |
|------|-------------|
| `feat` | New feature or functionality |
| `fix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `test` | Adding or updating tests |
| `docs` | Documentation changes |
| `chore` | Tooling, config, or maintenance |

**Examples:**
```
[MLID-1543] - feat(hospital): add MongoDB schema for hospital systems
[MLID-1544] - feat(hospital): add CRUD API endpoints
[MLID-1548] - feat(hospital): create list page component
[MLID-1549] - test(hospital): add unit tests for search filter
```

---

## PR Naming Convention

Deliverable PRs reference the **epic** Jira key and include the deliverable number:

```
[MLID-XXXX] feat: Deliverable Name (DN - component list)
```

**Examples:**
```
[MLID-1021] feat: Hospital List View (D1 - nav, list page, search, add modal)
[MLID-1021] feat: Hospital Detail View (D2 - detail page, edit modal, providers tab)
[MLID-1021] feat: Bulk Update (D3 - CSV upload, paste NPIs, confirm & replace)
```

---

## Pre-PR Checklist (per deliverable)

Before creating a deliverable PR, verify:

- [ ] All sub-tasks for the deliverable are merged into the epic branch
- [ ] Epic branch is rebased on latest `develop`
- [ ] All tests pass: `npm run test`
- [ ] Type checking passes: `npm run types:check`
- [ ] Linting passes: `npm run lint` (from `apps/web/`)
- [ ] No `console.log` or debug artifacts left in code
- [ ] No `any` types in new TypeScript code
- [ ] QA test scenarios documented in the PR description
- [ ] Sub-task Jira keys listed in the PR description

---

## Keeping the Epic Branch Healthy

| Situation | Action |
|-----------|--------|
| `develop` has moved ahead during sub-task work | Rebase the epic branch on `develop`, then rebase sub-task on epic |
| Merge conflict between sub-task and epic | Resolve on the sub-task branch before merging into epic |
| A sub-task turns out to be unnecessary | Close the Jira task with a note; do not create an empty branch |
| A sub-task was started before the epic branch existed | Merge existing sub-task branch into the epic branch to align |

---

## When to Use This Strategy vs. Task Strategy

| Scenario | Strategy |
|----------|----------|
| Multiple related Jira tasks grouped into deliverables | **This strategy** (epic branch + sub-task branches) |
| Single Jira task, self-contained work | [Task strategy](./task-git-branch-strategy.md) (task branch from develop) |

**Rule of thumb:** If the work requires multiple PRs to `develop` delivered incrementally, use this strategy. If the work fits in a single PR, use the task strategy.

---

## Summary

```
develop ← PR (per deliverable) ← epic/MLID-XXXX ← merge --no-ff ← feature/MLID-YYYY
```

One epic branch, multiple sub-task branches, one PR per deliverable. Deliver incrementally, keep `develop` stable.
