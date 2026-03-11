# Software Engineering Workflow

## Overview

This document defines the workflow cycle for picking up, planning, implementing, and delivering work. It applies to both **individual tasks** and **sub-tasks within an epic** — the cycle is the same, the difference is iteration:

- **Individual task**: Run the cycle once
- **Epic**: Run the cycle once per sub-task, with a master plan tracking progress across iterations

---

## Agent Identity

When working on tasks, the agent operates as:

- An **expert Software Engineer** with deep experience in Full Stack Development
- **Tech stack**: React, Next.js v14 (App Router), MongoDB, JavaScript, TypeScript
- Proficient in **Front End and Back End design patterns** for modern web applications
- Follows **Full Stack Web Application standards** for building and designing apps
- Adheres to **security standard patterns** (OWASP, input sanitization, RBAC, PHI handling)
- Practices **Test-Driven Development** (red-green-refactor)

---

## The Workflow Cycle

### Phase 1: Context Gathering

#### Step 1 — Read the Jira ticket

Given a task ID (`MLID-XXXX`), read the Jira issue to understand:

- **Summary and description**: What needs to be built or fixed
- **Acceptance criteria**: What defines "done"
- **Story points**: Scope and complexity estimate
- **Parent/epic**: Whether this is standalone or part of an epic
- **Links and attachments**: Design references, related issues, dependencies

#### Step 2 — Determine the git branch strategy

Based on the task type, read the appropriate branch strategy:

| Task type | Strategy document |
|-----------|-------------------|
| Standalone task | [task-git-branch-strategy.md](./task-git-branch-strategy.md) |
| Sub-task within an epic | [epic-git-branch-strategy.md](./epic-git-branch-strategy.md) |

If part of an epic, also read the **epic plan file** to understand overall progress and dependencies between tasks.

---

### Phase 2: Codebase Analysis

#### Step 3 — Investigate the codebase

Perform a comprehensive analysis of the existing code relevant to the task:

| Area | What to look for |
|------|------------------|
| **Pages / Routes** | Existing pages related to the feature (`apps/web/app/`) |
| **Components** | UI components, modules, and compositions (`apps/web/components/`) |
| **API Routes** | Existing endpoints, request/response shapes, middleware (`apps/web/app/api/`, `apps/web/pages/api/`) |
| **Services** | MongoDB service layer, business logic, external integrations (`apps/web/services/`) |
| **Models / Schemas** | Mongoose schemas and TypeScript types (`apps/web/models/`, `apps/web/types/`) |
| **Utilities** | Shared helpers, permissions, feature flags, contexts (`apps/web/utils/`) |
| **Tests** | Existing test patterns, coverage for related files (`*.test.ts`, `*.test.tsx`) |
| **Styles** | CSS module patterns, shared styles (`*.module.css`) |

**What to capture:**

- Reusable components and patterns (don't reinvent what exists)
- Naming conventions in this part of the codebase
- API response shapes and pagination patterns
- Permission and authorization patterns in use
- How similar features were implemented (precedent)
- Potential areas of impact or risk

---

### Phase 3: Implementation Plan

#### Step 4 — Write the plan

Create a plan file at `docs/agomez/plans/` using the appropriate template:

| Scope | Template | Output file |
|-------|----------|-------------|
| Individual task | [individual-task-workflow.md](./individual-task-workflow.md) | `docs/agomez/plans/MLID-XXXX-short-description.md` |
| Epic | [epic-workflow.md](./epic-workflow.md) | `docs/agomez/plans/MLID-XXXX-plan-progress.md` |

For an **individual task**, the plan covers the single task's implementation.

For an **epic**, the plan is created once at the start and serves as the master document. Each sub-task iteration updates the plan's tracking tables as work progresses.

> **Important:** The `docs/agomez/` directory is gitignored. Plan files, analysis documents, and workflow docs are **local working documents only** — never stage, commit, or version them. They live outside the codebase on purpose.

#### Step 5 — Wait for approval

**Do not write any implementation code until the plan is approved.**

Present the plan and wait for explicit approval. During review:

- **Approve as-is** → proceed to Phase 4
- **Request changes** → update the plan and re-present
- **Ask questions** → clarify and update accordingly

---

### Phase 4: Implementation

#### Step 6 — Create the feature branch

Follow the branch strategy determined in Step 2:

**Standalone task:**
```bash
git checkout develop
git pull origin develop
git checkout -b feature/MLID-XXXX-short-description
```

**Epic sub-task:**
```bash
git checkout epic/MLID-YYYY-epic-name
git pull origin epic/MLID-YYYY-epic-name
git checkout -b feature/MLID-XXXX-short-description
```

#### Step 7 — Implement using TDD

Follow the red-green-refactor cycle:

1. **RED** — Write a failing test for the next piece of functionality
2. **GREEN** — Write the minimum code to make the test pass
3. **REFACTOR** — Clean up while keeping tests green
4. Repeat for each piece of the implementation

Commit frequently:
```bash
git commit -m "[MLID-XXXX] - type(scope): description"
```

#### Step 8 — Verify quality

Before considering the task complete:

```bash
npm run test                    # All tests pass
npm run types:check             # No type errors
```

**Formatting — target only changed files:**

> `npm run format` runs Prettier across the **entire repo**. This can reformat unrelated files (line endings, spacing, etc.), polluting your git diff with dozens of unwanted changes. Always use `npx prettier --write` targeting only the files you changed.

```bash
# Format specific files
npx prettier --write apps/web/path/to/file1.ts apps/web/path/to/file2.ts

# Format a specific directory
npx prettier --write apps/web/services/orderEligibility/
```

**Never run `npm run format` without targeting** — it will touch files outside your scope.

**Linting — target only changed files:**

> `npm run lint` runs `next lint --fix` which scans and auto-fixes the **entire app**. This can modify unrelated files. Always use `npx eslint --fix` targeting only the files you changed.

```bash
# Lint specific files
npx eslint --fix apps/web/path/to/file1.ts apps/web/path/to/file2.ts

# Lint a specific directory
npx eslint --fix apps/web/services/orderEligibility/
```

**Never run `npm run lint` or `next lint --fix` without targeting** — it will touch files outside your scope.

Checklist:
- [ ] No `console.log` or debug artifacts
- [ ] No `any` types in new TypeScript code
- [ ] Test coverage meets thresholds (80%+ on modified files)
- [ ] Inputs validated, authorization enforced at API + UI levels

---

### Phase 5: Delivery

#### Step 9 — Write the Pull Request message

Create a comprehensive PR message document at `docs/agomez/PR/MLID-XXXX.md`. The PR message should be detailed and thorough, drawing from:

- **Individual task**: The plan document (`docs/agomez/plans/MLID-XXXX-short-description.md`)
- **Epic sub-task**: The epic plan-progress document (`docs/agomez/plans/MLID-XXXX-plan-progress.md`)

The PR message must include:

| Section | Content |
|---------|---------|
| **Title** | `[MLID-XXXX] type(scope): short description` |
| **Summary** | Bullet points of what was done and why |
| **Root Cause** | (for bug fixes) Detailed explanation of the underlying problem |
| **Changes Overview** | Total files changed, lines added/removed |
| **Files Changed** | Table of each file and what changed |
| **What Changed and Why** | Detailed breakdown of each change with before/after examples where applicable, alternatives considered, and design decisions |
| **Commits** | List of commit hashes and messages |
| **Test Plan** | Checklist of manual verification steps |
| **Jira** | Link to the Jira ticket |

> **Note:** The actual GitHub PR is created manually by the developer using this document as the PR body. This step only produces the message file.

#### Step 10 — Update tracking

- Update the plan document status (mark as Done)
- If part of an epic, update the plan-progress file (mark task as Done in the tracking table)
- Note any follow-up items or discoveries in the plan document

> **Note:** Jira ticket updates (status transitions, comments) are only performed when explicitly requested by the developer. Do not update Jira automatically as part of this step.

---

## Quick Reference

```
Phase 1: Context Gathering
  Step 1 → Read the Jira ticket
  Step 2 → Determine git branch strategy

Phase 2: Codebase Analysis
  Step 3 → Investigate related code comprehensively

Phase 3: Implementation Plan
  Step 4 → Write the plan (pick template by scope)
  Step 5 → Wait for approval              ← GATE: no code before approval

Phase 4: Implementation
  Step 6 → Create the feature branch
  Step 7 → Implement using TDD (red-green-refactor)
  Step 8 → Verify quality (tests, types, lint, security)

Phase 5: Delivery
  Step 9  → Write the PR message (docs/agomez/PR/MLID-XXXX.md)
  Step 10 → Update Jira and tracking docs
```

---

## Iteration Model

```
Individual task:
  → Cycle x1 → Done

Epic:
  → Create master plan (epic-workflow template)
  → For each sub-task in each deliverable:
      → Cycle x1 (steps 1-10) → Update master plan
  → When deliverable group complete → PR to develop
  → Repeat for next deliverable
  → Epic done when all deliverables merged
```

---

## When to Deviate

| Situation | Adjustment |
|-----------|------------|
| Trivial fix (typo, config, 1-line change) | Skip Phase 2 depth; plan can be verbal |
| Spike / investigation task | Phase 3 output is a findings document, not an implementation plan |
| Urgent hotfix | Abbreviated plan, but still get verbal approval before coding |
| Task depends on another incomplete task | Document the dependency; wait or adjust scope |
