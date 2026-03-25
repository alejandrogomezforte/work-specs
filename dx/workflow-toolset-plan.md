# Workflow Toolset Plan

## Overview

Design for a Claude Code toolset (skills + agents) that standardizes the daily development workflow described in [sw-workflow.md](./sw-workflow.md). The toolset maps to 3 workflow phases, each with a human interruption point.

---

## Architecture: 4 Skills + 2 Agents

### Flow Diagram

```
/plan MLID-XXXX     → investigate, plan, approve
/implement MLID-XXXX → write code + tests (TDD)
  ↕ user tests UI, asks questions, queries MongoDB
/verify MLID-XXXX   → quality gate before wrap-up
/wrap-up MLID-XXXX  → merge, PR, update tracking
```

---

## Skills (4)

### 1. `/plan MLID-XXXX`

**Phase:** Context Gathering + Codebase Analysis + Implementation Plan (Steps 1-5)

**What it does:**

1. Reads the Jira ticket via Atlassian MCP (`MLID-XXXX` passed as argument)
2. Determines standalone task vs epic sub-task
3. Reads the appropriate branch strategy doc (`task-git-branch-strategy.md` or `epic-git-branch-strategy.md`)
4. Forks to `codebase-investigator` agent for deep code exploration
5. Forks to `solution-architect` agent to craft the plan
6. Writes plan file to `docs/agomez/plans/`
7. Presents the plan inline and **stops for user approval**

**Interruption:** User reviews, adjusts, and approves the plan before any code is written.

**Key details:**

- Uses `$ARGUMENTS` for the Jira ticket ID
- Dynamic context injection: `!git branch --show-current` to determine current branch context
- Reads branch strategy docs as supporting files
- Plan file naming: `MLID-XXXX-short-description.md` (individual) or `MLID-XXXX-plan-progress.md` (epic)
- Templates: `individual-task-workflow.md` or `epic-workflow.md`

---

### 2. `/implement MLID-XXXX`

**Phase:** Implementation (Steps 6-7)

**What it does:**

1. Reads the approved plan file from `docs/agomez/plans/` (matched by `MLID-XXXX`)
2. Creates the feature branch (following branch strategy)
3. Executes TDD red-green-refactor cycles **inline** (user can watch)
4. Runs tests after each cycle
5. Commits frequently with `[MLID-XXXX] - type(scope): description` format
6. **Stops** when implementation is complete

**Interruption:** User tests the UI manually, asks questions, queries MongoDB, interacts with Claude Code.

**Why inline (not an agent):**

- TDD is iterative — red-green-refactor cycles need back-and-forth with test runner
- User interrupts this phase to test UI and ask questions
- CLAUDE.md already defines all coding standards, TDD rules, and patterns
- Code changes must happen in the working tree, not an isolated worktree

---

### 3. `/verify MLID-XXXX`

**Phase:** Quality Gate (Step 8)

**What it does:**

1. Uses `MLID-XXXX` to identify the feature branch and base branch
2. Identifies changed files vs base branch
2. Runs tests on changed files only
3. Runs `types:check`
4. Runs prettier on changed files only (not whole repo)
5. Runs eslint on changed files only (not whole repo)
6. Checks for `console.log`, `any` types
7. Checks test coverage on modified files (80% threshold)
8. Reports a pass/fail checklist

**Dynamic context injection:**

```yaml
- Changed files: !`git diff --name-only develop`
- Current branch: !`git branch --show-current`
```

**Key detail:** Never runs `npm run format` or `npm run lint` globally — always targets only changed files, as documented in sw-workflow.md Step 8.

---

### 4. `/wrap-up MLID-XXXX`

**Phase:** Delivery (Steps 9-10)

**What it does:**

1. Uses `MLID-XXXX` to locate plan file, branch, and Jira ticket
2. Determines if task is standalone or epic sub-task
2. **If epic sub-task:**
   - Merges feature branch into epic branch
   - Updates plan-progress tracking table
   - If deliverable group is complete → continues to PR creation
   - If more sub-tasks remain → stops after merge
3. **If standalone or deliverable complete:**
   - Generates PR message document at `docs/agomez/PR/MLID-XXXX.md`
   - Includes: summary, files changed table, commits, test plan, Jira link
   - Creates the GitHub PR
4. Updates plan document status to Done

**Key details:**

- PR body follows the template in sw-workflow.md Step 9
- Jira ticket transitions only when explicitly requested
- Reads plan doc + git log + diff for PR content generation

---

## Agents (2)

### 1. `codebase-investigator`

**Type:** Explore agent
**Model:** sonnet (fast, cost-effective for exploration)
**Tools:** Read, Grep, Glob, Bash

**Purpose:** Deep codebase analysis that produces verbose output. Runs in isolation so file reads, grep results, and analysis don't pollute the main conversation context.

**What it does:**

Given a feature description or Jira context, investigates:

| Area | What to look for |
|------|------------------|
| Pages / Routes | Existing pages related to the feature |
| Components | UI components, modules, compositions |
| API Routes | Endpoints, request/response shapes, middleware |
| Services | MongoDB service layer, business logic, integrations |
| Models / Schemas | Mongoose schemas and TypeScript types |
| Utilities | Helpers, permissions, feature flags, contexts |
| Tests | Existing test patterns and coverage |
| Styles | CSS module patterns |

**Returns:** A structured summary with:

- Reusable components and patterns found
- Naming conventions in the relevant area
- API response shapes and pagination patterns
- Permission/authorization patterns in use
- How similar features were implemented (precedent)
- Potential areas of impact or risk
- Key file paths and their purposes

**Invoked by:** `/plan` skill (forked via `context: fork`)

---

### 2. `solution-architect`

**Type:** Custom agent
**Model:** opus (needs deep reasoning for architectural decisions)
**Tools:** Read, Grep, Glob

**Purpose:** Given the investigator's findings + Jira requirements, designs a structured implementation plan. Reasoning through alternatives and trade-offs happens in isolation — only the final plan is surfaced.

#### Design Philosophy: Stack-Aware, Best-Practice-Driven

The solution architect is **not** a pattern-copier. The existing codebase contains both good and bad practices — legacy code, inconsistent patterns, and shortcuts that accumulated over time. The agent must **not** learn "how to code" from the codebase.

Instead, the agent operates as a **senior engineer who brings industry best practices to the team**, constrained only by the tech stack already in use. The codebase analysis (from the investigator) tells the agent **what tools are on the workbench** — the agent decides **how to use them correctly**.

**What the agent learns from the codebase:**
- The tech stack (what libraries, frameworks, and tools are available)
- File locations (where things live in the directory structure)
- Integration points (what existing code it needs to interface with)
- Data shapes (existing models, DTOs, API contracts it must respect)

**What the agent does NOT learn from the codebase:**
- Code style or quality patterns — it applies its own standards
- Architecture decisions — it evaluates from first principles
- Testing approaches — it follows TDD best practices, not existing test shortcuts
- Error handling — it applies proper patterns regardless of what exists

#### Tech Stack (What the Agent Works With)

| Layer | Tech |
|-------|------|
| Framework | Next.js 14 (App Router) |
| UI | React 18, TypeScript (strict) |
| Styling | CSS Modules, MUI components + `sx` props |
| Forms | React Hook Form (`useForm`, `Controller`) |
| Database | MongoDB with Mongoose ODM (raw driver is legacy — never use for new code) |
| Auth | NextAuth with Google OAuth |
| RBAC | Role + Position-based, dual enforcement (API + UI) |
| Feature flags | MongoDB-backed, enum-defined (`FeatureFlag`) |
| Logging | Custom logger (Application Insights), never `console` |
| Testing | Jest, React Testing Library, TDD mandatory |
| Job processing | Pulse (Agenda fork) |
| Real-time | Socket.IO |

#### Engineering Standards the Agent Enforces

**TypeScript:**
- Strict mode, no `any` — use `unknown` + type guards when type is uncertain
- Discriminated unions over optional fields for variant types
- Exhaustive switch/case with `never` for compile-time safety
- Proper generics over type assertions

**React:**
- Single responsibility per component
- Composition over prop drilling — use render props, children, or context where appropriate
- Custom hooks to encapsulate stateful logic — components should be thin
- Memoization only when profiling justifies it, not by default

**API Design:**
- Consistent auth → permission → validate → execute → respond pipeline
- Input validation at the boundary (API route), trust internal code
- Proper HTTP status codes: 400 (bad input), 401 (no auth), 403 (no permission), 404 (not found), 500 (unexpected)
- Error responses that are safe for production (no stack traces, no internal details)

**Mongoose / Database:**
- Lean queries (`.lean()`) for read operations
- Explicit field selection when full documents aren't needed
- Aggregation pipelines via Mongoose model `.aggregate()`
- Index-aware query design — don't design queries that require full collection scans
- Soft delete pattern: `deletedAt` field

**Testing (TDD):**
- Tests describe behavior, not implementation ("should return 403 when user lacks permission", not "should call checkRole")
- Arrange-Act-Assert structure
- One assertion per logical concept (multiple `expect` calls are fine if testing one behavior)
- Mock at boundaries (database, external services), not internal functions
- Every code path tested: happy path, error paths, edge cases, authorization

**Security:**
- RBAC enforced at both API and UI levels — never UI-only
- Feature flags gated at both server (API route) and client (page)
- Input sanitization at API boundaries
- PHI awareness — no sensitive data in logs, error messages, or client responses
- No secrets in code — environment variables or Key Vault

**Code Organization:**
- Files do one thing — avoid god-files
- Co-locate tests with source
- Named exports (no default exports)
- Absolute imports (`@/...`)

#### What It Does

1. Receives: Jira ticket details + codebase investigation summary
2. Designs the implementation approach using best practices for the stack
3. Identifies files to create/modify, specifying the correct layer and responsibility
4. Defines the TDD test strategy (what to test, in what order)
5. Considers security, permissions, feature flags
6. Writes the plan file using the appropriate template

#### Returns

A plan file written to `docs/agomez/plans/` containing:

- Task overview and acceptance criteria
- Implementation steps in order
- Files to create/modify with descriptions
- TDD strategy: tests to write and in what order
- Risk areas and edge cases
- Dependencies and prerequisites

**Invoked by:** `/plan` skill (forked via `context: fork`, after investigator completes)

---

## File Locations

| Item | Location | Scope |
|------|----------|-------|
| Skills | `~/.claude/skills/<name>/SKILL.md` | User-scoped (reusable across repos, not shared with team) |
| Agents | `~/.claude/agents/<name>.md` | User-scoped (reusable across repos, not shared with team) |

### Skill files to create

```
~/.claude/skills/
├── plan/
│   └── SKILL.md
├── implement/
│   └── SKILL.md
├── verify/
│   └── SKILL.md
└── wrap-up/
    └── SKILL.md
```

### Agent files to create

```
~/.claude/agents/
├── codebase-investigator.md
└── solution-architect.md
```

---

## Design Decisions

### Why `/implement` is a skill, not an agent

| Concern | Why inline is better |
|---------|---------------------|
| TDD cycles | Iterative red-green-refactor needs test runner feedback |
| User interaction | User interrupts to test UI, ask questions, query MongoDB |
| Coding standards | CLAUDE.md already defines all rules — no duplication needed |
| File access | Code changes must be in working tree, not isolated worktree |
| Visibility | User wants to observe implementation progress |

### Why `solution-architect` uses opus model

Plan generation involves reasoning through architectural trade-offs, evaluating multiple approaches, and making design decisions. This benefits from the strongest reasoning model. The cost is justified because it runs once per task, not iteratively.

### Why `codebase-investigator` uses sonnet model

Exploration is about reading and searching, not deep reasoning. Sonnet is faster and cheaper, which matters when scanning many files across the codebase.

---

## Implementation Order

```
1. codebase-investigator agent  (standalone, no dependencies)
2. solution-architect agent     (standalone, no dependencies)
3. /plan skill                  (depends on both agents)
4. /implement skill             (depends on /plan existing)
5. /verify skill                (standalone)
6. /wrap-up skill               (standalone)
```

Steps 1-2 can be built in parallel.
Steps 5-6 can be built in parallel.

---

## Open Questions

- [x] ~~Should `/implement` auto-read the most recent plan file, or require the user to specify which plan?~~ **Resolved:** All skills take `MLID-XXXX` — plan file is located by task ID.
- [x] ~~Should `/verify` auto-detect the base branch (develop vs epic branch), or require it as argument?~~ **Resolved:** Task ID is passed; base branch can be inferred from the plan file or branch naming convention.
- [ ] Should `/wrap-up` auto-create the PR, or just generate the PR doc and let the user create it?
- [ ] Should agents preload any skills for additional context (e.g., coding conventions)?
