# Ship — Automated Push + PR + CI Retry

**Purpose:** Automate the push → PR → CI check → auto-fix loop. Fire and (mostly) forget.

**Scope:** Global — works on any project with GitHub Actions CI.

---

## Invocation

User says `/ship` or "ship it". Takes optional arguments:
- `/ship` — push current branch, create PR against default base, monitor CI
- `/ship --base <branch>` — specify PR base branch
- `/ship --pr-only` — skip push (branch already pushed), just create PR + monitor
- `/ship --no-pr` — push only, no PR creation, still monitor CI

---

## Process

### Step 1: Pre-flight checks (local)

Before pushing, run the same checks as the pre-commit hook:
1. `npx tsc --noEmit` — TypeScript clean
2. `npm run build` — build succeeds
3. `npx vitest run` — tests pass (or project-equivalent test command)
4. **Security scan (lightweight):** Run `/security-scan deps` — check for dependency CVEs. Any CRITICAL CVE = block the ship.
5. **Tech debt check (lightweight):** Run `/tech-debt hotspots` on changed files only — flag any CRITICAL debt items in the diff. This is informational (reported to the user) not blocking, unless the hotspot is in a file being shipped.

6. **Mobile checkpoint (conditional):** If changed files include viewport meta tags, responsive CSS (`@media`), Tailwind responsive classes, or mobile-specific components, run `/julie checkpoint`. Any FAIL items on checks 1-4 (viewport, zoom, touch targets, font size) block the ship.

If checks 1-3 fail, fix the issue before pushing. Check 4 blocks on CRITICAL CVEs. Check 5 warns but doesn't block. Check 6 blocks on FAIL items (mobile breakage).

**Detect test command:** Check `package.json` scripts for `test`, fall back to `npx vitest run`. For non-Node projects, check for `Makefile`, `pytest`, `cargo test`, etc.

### Step 2: Push + Create PR

1. Push current branch: `git push -u origin <branch>`
2. Create PR with `gh pr create`:
   - Title: derive from branch name + recent commits
   - Body: summary of changes + test plan
   - Base: `main` (or user-specified `--base`)
3. Report PR URL to user

### Step 3: Monitor CI (up to 3 retry cycles)

Poll CI status using `gh run list --branch <branch> --limit 1` every 30 seconds until completion.

**If CI passes:** Report success. Move to Step 4.

**If CI fails (retry cycle):**
1. Fetch the failure logs: `gh run view <run-id> --log-failed`
2. Read the error output — identify which step failed and why
3. Fix the issue in code
4. Run pre-flight checks again (Step 1)
5. Commit the fix with message: `fix: CI failure — <description>`
6. Push — CI re-triggers automatically
7. Resume polling

**Max 3 retry cycles.** If CI still fails after 3 fixes, stop and report:
- What failed
- What was tried
- What the user should look at

### Step 4: Report final status

Once CI passes (or retries exhausted), report to user:
```
SHIP: PR #123 — CI passed
URL: https://github.com/user/repo/pull/123
Retries: 1 (fixed TypeScript error in foo.ts)
Security: PASS (0 critical CVEs)
Tech Debt: 2 hotspots flagged (see pre-flight output)
Mobile: PASS / FAIL / N/A (no mobile files changed)
Status: Ready for review
```

---

## What Ship Does NOT Do (Yet)

- Does not monitor for review comments (that's Phase 2 — see memory/automated-pr-workflow-prd.md)
- Does not auto-merge
- Does not handle branch cleanup after merge
- Does not run in the background after the session ends

---

## Error Handling

- **No remote configured:** Ask user which remote to push to
- **PR already exists:** Skip PR creation, just monitor CI on existing PR
- **No CI workflow:** Warn user, skip monitoring, just push + create PR
- **Auth issues with `gh`:** Tell user to run `gh auth login`

---

## Anti-Patterns

- NEVER force-push unless the user explicitly says to
- NEVER push to main/master directly — always via PR
- NEVER merge the PR automatically — that's the user's (or reviewer's) decision
- NEVER skip pre-flight checks "to save time"
