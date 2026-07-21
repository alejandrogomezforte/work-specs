# How to run the web dev server and redirect its output to a log file

Run `npm run dev:web` and capture **all** output (stdout + stderr) into a date-stamped
log file under `docs/agomez/logs/`, while still watching it live in the terminal.

Then paste the log file contents (or just its name) into a Claude prompt for debugging.

- `2>&1` — merge stderr into stdout so build/runtime errors are captured too.
- `tee` — print to the terminal **and** write to the file at the same time.
- `$(date +%Y-%m-%d)` — names the file `apps-web-2026-07-15.txt` (today's date).
- Shell: **Git Bash** (these are bash commands, not PowerShell).

> Note: both worktrees' `dev:web` use port **8080**. Stop one server (Ctrl+C) before
> starting another, or they will collide.

---

## Option A — main checkout (`li-call-processor`)

Run from the repository root `C:\Users\alejandro.gomez\Dev\li-call-processor`.
The logs folder is a normal relative path from here.

```bash
cd /c/Users/alejandro.gomez/Dev/li-call-processor
npm run dev:web 2>&1 | tee "docs/agomez/logs/apps-web-$(date +%Y-%m-%d).txt"
```

---

## Option B — a custom worktree (e.g. `../worktrees/MLID-2740`)

Run from the worktree root. The log still goes to the **main checkout's**
`docs/agomez/logs/` folder (so all logs live in one place), reached with the
relative path `../../li-call-processor/docs/agomez/logs/`.

```bash
cd /c/Users/alejandro.gomez/Dev/worktrees/MLID-2740
npm run dev:web 2>&1 | tee "../../li-call-processor/docs/agomez/logs/apps-web-$(date +%Y-%m-%d).txt"
```

Path math for the relative target (from `Dev/worktrees/<branch>`):
`../` → `Dev/worktrees`, `../../` → `Dev`, then `li-call-processor/docs/agomez/logs/`.

---

## Optional: avoid overwriting when running more than once per day

Add the time to the filename (`apps-web-2026-07-15_151954.txt`):

```bash
# Option A
npm run dev:web 2>&1 | tee "docs/agomez/logs/apps-web-$(date +%Y-%m-%d_%H%M%S).txt"

# Option B
npm run dev:web 2>&1 | tee "../../li-call-processor/docs/agomez/logs/apps-web-$(date +%Y-%m-%d_%H%M%S).txt"
```
