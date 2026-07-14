# MLID-364 (Next.js 16 upgrade) — Decisions needed before we deploy

This upgrade (LISA to Next.js 16 / React 19, with a new authentication system) is code-complete. Before it goes out, there are **three decisions** that are not about code — they are about risk and timing, and the team should agree on them together.

This document explains each one in plain language: what it is, why it matters, and a recommendation.

| # | Decision | Who decides | User impact | Recommendation |
|---|----------|-------------|-------------|----------------|
| 1 | Ship on a pre-release (beta) auth library | Engineering (PE) | None | Ship it, pin the version, upgrade later |
| 2 | All users get logged out on deploy | PO + Engineering | One-time re-login for everyone | Deploy off-hours + tell users first |
| 3 | How to merge such a large change | PE | None | One reviewed pull request, planned timing |

---

## Decision 1 — We depend on a "beta" (pre-release) version of the login library

**What it is.**
The upgrade requires a new version of the login/authentication library (Auth.js v5). The people who build that library have not yet published a final, "stable" version — only a **pre-release (beta)** version exists today. We are using one specific, fixed beta build.

**Why this matters.**
Beta software is feature-complete but officially still "in progress," so it carries slightly more risk than a final release. However, this beta version is the **only** one that works with Next.js 16's new way of handling web requests — the old login library is not compatible. Waiting for the final "stable" release is not a realistic option, because that timeline is decided by an outside team and could be months away.

**The reality.**
This particular beta is already used in production by a very large number of applications and is stable in practice. The risk is low, but it is not zero, and we should acknowledge it rather than hide it.

**The decision.**
- **Option A (recommended):** Ship now on the exact pinned beta version, write down that we are on a beta, and upgrade to the stable release later when it exists (a small, low-risk change at that point).
- **Option B:** Wait for the stable release. This would block the entire upgrade indefinitely, for a timeline we do not control.

**Recommendation:** Option A. Pin the version, document it, move on.

---

## Decision 2 — Everyone will be logged out the moment we deploy

**This is the one with real user impact. It needs a plan, not just a yes/no.**

**What it is.**
When someone logs in, the app saves a small token in their browser (a "session cookie") so they stay logged in. The new login system stores that token in a **different format**. The tokens created by the old system and the new system are not compatible with each other.

**What happens on deploy.**
The instant the new version goes live, **every user who is currently logged in will be signed out.** They do **not** lose any data or work saved on the server — they simply have to log in one more time. After that single re-login, everything works normally and this never happens again.

**Why this matters.**
It is a one-time disruption, but it hits **all active users at the same moment**. If we deploy in the middle of a busy workday with no warning, a lot of people suddenly get kicked out and may think something is broken, which creates confusion and support tickets — over what is actually a harmless, expected, one-time event.

**The decision (how to handle it).**
- **Timing:** Deploy during a **low-traffic window** (evening or weekend) so the fewest people are interrupted.
- **Communication:** Give users a **short heads-up beforehand** — for example, "On [date/time] you will be signed out and will need to log in again. This is expected; nothing is wrong."
- Best result: **do both.**

**Recommendation:** Deploy off-hours **and** send a brief advance notice. Handled this way it is a minor, expected inconvenience. Handled silently, it looks like an outage.

---

## Decision 3 — How to land such a large change safely

**What it is.**
This upgrade is a very large change: over one hundred commits, and the authentication change alone touched hundreds of files. All of it lives on a separate branch and eventually has to merge into the main development branch in one step.

**Why this matters.**
- A very large change is harder to review thoroughly, and merging it is a significant moment — a lot moves at once.
- The main branch keeps moving because the team ships quickly, so the longer we wait to merge, the more we have to keep re-synchronizing our branch with everyone else's work.

**Why we cannot simply split it into small pieces.**
The changes are deeply connected. The new login system, in particular, touches almost every part of the app, so there is no clean way to cut it into small, independent pieces without breaking things in between. This realistically has to land as **one** change.

**The decision.**
- **Reviewer:** who owns the review (recommended: the Principal Engineer).
- **Timing:** pick a planned, low-risk moment to merge — not right before a release, and after the browser verification pass is complete.
- **Keep in sync:** continue synchronizing our branch with the main branch right up until the merge, so the final step is small.

**Recommendation:** One pull request, reviewed by the PE, merged at a planned low-risk time, with the browser verification done first and the branch kept in sync until then.

---

## In short

- **Decision 1 (beta library):** low risk, just needs a conscious "yes." → Recommend: ship it.
- **Decision 2 (everyone logged out):** the important one. Needs a deploy window and a user heads-up. → Recommend: off-hours deploy + advance notice.
- **Decision 3 (how to merge):** one big reviewed pull request, planned timing, kept in sync. → Recommend as stated.

None of these block finishing the work — they are about **how and when we release it safely.**
