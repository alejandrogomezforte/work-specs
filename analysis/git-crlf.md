# Git CRLF Line Ending Issues on Windows

## Problem Summary

Files in `apps/api/` appear as modified in `git status` even though they have no actual content changes. This is a "phantom modification" caused by Git's line ending handling on Windows.

## Root Cause

### Configuration
```bash
git config --get core.autocrlf
# Returns: true
```

With `core.autocrlf=true`:
- Git stores files with **LF** internally (in the repository)
- Git checks out files with **CRLF** on Windows (working copy)
- Git converts CRLF→LF when committing

### What Happened

1. Git's index recorded that these files should have CRLF endings on disk
2. Some tool (turbo, npm scripts, etc.) wrote/touched the files with LF endings
3. Git detects the mismatch and shows files as "modified"
4. `git diff` shows no actual content changes, only warnings:
   ```
   warning: in the working copy of 'apps/api/eslint.config.mjs', LF will be replaced by CRLF the next time Git touches it
   ```

### Verification Commands

```bash
# Check line endings in working copy (shows \n = LF)
head -1 apps/api/eslint.config.mjs | od -c

# Check line endings in Git HEAD (also shows \n = LF)
git show HEAD:apps/api/eslint.config.mjs | head -1 | od -c

# Check for actual content diff (shows nothing, just warnings)
git diff --numstat apps/api/

# Check diff stats (shows no insertions/deletions)
git diff --stat apps/api/
```

## Commands That Can Trigger This

These commands may touch files and cause line ending mismatches:

| Command | Tool | Why |
|---------|------|-----|
| `npm run types:check` | turbo | Turbo may touch/cache files |
| `npm run format` | prettier | May write files with different line endings |
| `npm run build` | turbo/webpack | Build processes touch many files |
| `npm test` | jest | May touch config files |

## Solutions

### Quick Fix: Restore Files
Discard the phantom changes without affecting actual content:
```bash
git restore apps/api/
```

### Permanent Fix: Re-normalize Repository
Apply Git's normalization rules to all files and commit:
```bash
git add --renormalize .
git status  # Review what changed
git commit -m "Normalize line endings"
```

### Prevent Future Issues

1. **Add `.gitattributes`** to enforce consistent line endings:
   ```gitattributes
   # Set default behavior to automatically normalize line endings
   * text=auto

   # Force LF for specific files
   *.ts text eol=lf
   *.tsx text eol=lf
   *.js text eol=lf
   *.json text eol=lf
   *.md text eol=lf
   *.css text eol=lf
   ```

2. **Configure editors** to use LF:
   - VS Code: `"files.eol": "\n"`
   - .editorconfig: `end_of_line = lf`

3. **Consider changing autocrlf**:
   ```bash
   # Option 1: Input mode (convert CRLF→LF on commit, don't convert on checkout)
   git config core.autocrlf input

   # Option 2: Disable (no automatic conversion)
   git config core.autocrlf false
   ```

## Understanding `git add --renormalize`

This command re-applies Git's clean/smudge filters (including line ending normalization) to all tracked files.

**What it does:**
1. Reads each tracked file from disk
2. Applies normalization rules from `.gitattributes` and `core.autocrlf`
3. Updates the Git index with the normalized version

**When to use:**
- After changing `.gitattributes` rules
- To fix line ending inconsistencies permanently
- When migrating a repository to consistent line endings

**Caution:** This may create a commit that touches many files if line endings were inconsistent.

## References

- [Git Documentation: gitattributes](https://git-scm.com/docs/gitattributes)
- [GitHub: Configuring Git to handle line endings](https://docs.github.com/en/get-started/getting-started-with-git/configuring-git-to-handle-line-endings)
- [Git Documentation: core.autocrlf](https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration)
