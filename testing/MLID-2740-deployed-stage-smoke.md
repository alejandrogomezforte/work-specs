# MLID-2740 — Deployed-stage smoke test (plain step-by-step)

## What this is, in one sentence

After your PR merges into `develop`, the app **automatically deploys to the STAGING environment**. This test = open the staged app and click around to confirm the webpack→Turbopack change did not break anything that only shows up on a real server (things that look fine on your laptop but could fail once deployed).

**You do NOT deploy anything by hand.** Merging into `develop` triggers the deploy for you.

---

## Fill this in first

- **Staging URL of LISA** (the address you normally use to open the staged app): `__________________________`
- You log in the same way you always do (Google).

> If you tell me the staging URL, I will paste it into this file so it is written down for next time.

---

## Step 0 — Wait for the deploy to finish

1. Make sure your PR is **merged into `develop`** (you already opened the PR).
2. Open GitHub in the browser → the `li-call-processor` repo → the **"Actions"** tab (top menu).
3. Look for the most recent run on the **`develop`** branch named **"Build and Deploy to Azure Container Apps"**.
4. Wait until it turns **green** (a green check). It runs several jobs — the ones that matter here are the **main app (web)** and the **worker**. This usually takes several minutes.

> If the Actions run turns **red** (fails), stop and tell me — do not continue; we look at the failure first.

When it is green, the staging environment is now running your change. Now do the checks below.

---

## Step 1 — The app opens and login works  ⬅ most important

**Why:** we removed an old setting (`esmExternals`). This confirms Google login still works after that.

**Do:**
1. Open the **staging URL** in your browser.
2. Log in with Google, the normal way.

**Good result:** you land on the normal home/dashboard, logged in.
**Bad result:** login fails, or you get redirected to a page with `api/auth/error`, or a "server configuration" error.
→ If bad: take a screenshot and tell me.

---

## Step 2 — Click through the main pages

**Why:** confirms pages still render correctly under the new bundler.

**Do:**
1. Press **F12** in the browser → click the **"Console"** tab (this shows errors).
2. Visit a few pages you use every day, for example: Patients, Orders, Intakes, Documents.

**Good result:** pages load normally. In the Console there are **no red errors** that mention `ssh2`, `applicationinsights`, or `Module not found`.
**Bad result:** a page is blank/broken, or a red error mentions one of those words.
→ If bad: screenshot the page + the Console, and tell me.

> **Ignore this one known message:** on the Intakes page you may see `AbortError: signal is aborted without reason`. That is the already-known, harmless issue tracked separately as **MLID-2837**. It is NOT part of this test.

---

## Step 3 — Background jobs / worker is alive

**Why:** the SFTP export jobs (Alnylam, Kedrion, Abbvie) run in a separate "worker". This confirms that worker is healthy after the deploy.

**Do:**
1. Open **`<staging URL>/admin/jobs`** in the browser.
2. Look at the list of jobs and their recent runs.

**Good result:** the page loads and you can see jobs, with recent runs that **succeeded** (not stuck or all failing). That means the worker is running and processing jobs.
**Bad result:** the page errors, or every recent job is failing/stuck.
→ If bad: screenshot and tell me.

> **Do you need to trigger a real export?** Not required for this test. Triggering a real SFTP export actually sends files to the drug companies, so **do not** fire one just to test. Seeing that scheduled jobs are running successfully is enough. If the team specifically wants a live export tested, coordinate that with the PE — don't do it solo.

---

## Step 4 — Application Insights (monitoring) still works  — *optional, skip if unsure*

**Why:** we changed how the monitoring library is loaded. This confirms monitoring still starts.

**Easiest check (needs Azure portal access):**
1. Open the **Azure Portal** → find the staging web Container App named **`li-stage-local-infusion-app`**.
2. Open its **"Log stream"** (live logs).
3. Look near the startup lines for: `[instrumentation] Application Insights initialized`.

**Good result:** you see that "initialized" line and **no** crash right after it.
**If you don't have access or are not sure:** skip this step and tell me — this is one the PE can confirm quickly. It does not block the rest of the test.

---

## If anything looks broken

Tell me:
- which step,
- which page,
- and a screenshot of the error.

Because the change is already on `develop`, we fix it "forward": I help you make a small hotfix branch, or we revert. Do not panic — just report it.

---

## When everything passes

Tell me "stage smoke passed" and I will mark MLID-2740 as fully verified in the plan. Your team's QA moves the Jira ticket after it is on `develop` — you do not move it yourself.

---

## Quick checklist (copy for your notes)

- [ ] Step 0 — GitHub Actions deploy on `develop` is green
- [ ] Step 1 — App opens and Google login works
- [ ] Step 2 — Main pages load, no red console errors about `ssh2` / `applicationinsights` / `Module not found`
- [ ] Step 3 — `/admin/jobs` loads and recent jobs are succeeding (worker alive)
- [ ] Step 4 — (optional) App Insights "initialized" line in staging web logs
