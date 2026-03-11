# Individual Task - Git Branch Strategy

## Overview

This document defines the git branch strategy for **standalone Jira tasks** (single MLID story/task, not part of a multi-task epic). The task branch is created directly from `develop` and merged back into `develop` via a Pull Request.

---

## Branch Hierarchy

```
develop (deployed to staging)
  |
  +-- feature/MLID-XXXX-short-description    (task branch)
```

There is no intermediate epic branch. The task branch is the only working branch.

---

## Flow

```
1. Task branch is created FROM develop
2. All commits for the task go into the task branch
3. When work is complete → PR from task branch → develop
4. After PR merges → delete the task branch
```

---

## Step-by-Step Workflow

### 1. Starting a new task

```bash
git checkout develop
git pull origin develop
git checkout -b feature/MLID-XXXX-short-description
```

### 2. Working on the task

```bash
# Make changes, commit often with descriptive messages
git add <files>
git commit -m "[MLID-XXXX] - type(scope): description"

# Push to remote regularly
git push origin feature/MLID-XXXX-short-description
```

### 3. Keeping up with develop (if needed)

If `develop` has moved ahead while you're working, rebase your branch:

```bash
git checkout develop
git pull origin develop
git checkout feature/MLID-XXXX-short-description
git rebase develop
git push origin feature/MLID-XXXX-short-description --force-with-lease
```

> **Tip:** Rebase early and often to avoid large merge conflicts.

### 4. Creating the Pull Request

```bash
# Ensure branch is up to date with develop
git checkout develop
git pull origin develop
git checkout feature/MLID-XXXX-short-description
git rebase develop
git push origin feature/MLID-XXXX-short-description --force-with-lease

# Create PR
gh pr create --base develop --head feature/MLID-XXXX-short-description \
  --title "[MLID-XXXX] type(scope): short description" \
  --body "## Summary
- What was done and why

## Test plan
- [ ] How to verify the changes

## Jira
- [MLID-XXXX](https://localinfusion.atlassian.net/browse/MLID-XXXX)"
```

### 5. After PR merges

```bash
# Clean up local branch
git checkout develop
git pull origin develop
git branch -d feature/MLID-XXXX-short-description

# Clean up remote branch (if not auto-deleted by GitHub)
git push origin --delete feature/MLID-XXXX-short-description
```

---

## Branch Naming Convention

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `feature/MLID-XXXX-short-description` | `feature/MLID-1600-add-patient-search` |
| Bug fix | `fix/MLID-XXXX-short-description` | `fix/MLID-1601-fix-pagination-offset` |
| Refactor | `refactor/MLID-XXXX-short-description` | `refactor/MLID-1602-extract-npi-validator` |

**Rules:**
- Always prefix with the branch type (`feature/`, `fix/`, `refactor/`)
- Always include the Jira key (`MLID-XXXX`)
- Use lowercase kebab-case for the description
- Keep the description short (3-5 words max)

---

## Commit Message Convention

Format: `[MLID-XXXX] - type(scope): description`

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
[MLID-1600] - feat(patient): add search by MRN
[MLID-1600] - test(patient): add unit tests for MRN search
[MLID-1601] - fix(pagination): correct off-by-one in page offset
[MLID-1602] - refactor(npi): extract validation to shared util
```

---

## PR Naming Convention

Format: `[MLID-XXXX] type(scope): short description`

**Examples:**
```
[MLID-1600] feat(patient): add search by MRN
[MLID-1601] fix(pagination): correct off-by-one in page offset
[MLID-1602] refactor(npi): extract validation to shared util
```

---

## Pre-PR Checklist

Before creating a PR, verify:

- [ ] All tests pass: `npm run test`
- [ ] Type checking passes: `npm run types:check`
- [ ] Linting passes: `npm run lint` (from `apps/web/`)
- [ ] Branch is rebased on latest `develop`
- [ ] Commit messages follow the convention
- [ ] No `console.log` or debug artifacts left in code
- [ ] No `any` types in new TypeScript code

---

## When to Use This Strategy vs. Epic Strategy

| Scenario | Strategy |
|----------|----------|
| Single Jira task, self-contained work | **This strategy** (task branch from develop) |
| Multiple related Jira tasks grouped into deliverables | [Epic strategy](../plans/epic-git-branch-strategy.md) (epic branch + sub-task branches) |

**Rule of thumb:** If the work can be reviewed and QA-tested in a single PR, use this strategy. If the work spans multiple tasks and needs to be delivered incrementally, use the epic strategy.

---

## Summary

```
develop ← PR ← feature/MLID-XXXX-short-description
```

One task, one branch, one PR. Keep it simple.
