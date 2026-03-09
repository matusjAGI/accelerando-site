---
name: tech-debt
description: >
  Systematic technical debt inventory and prioritization. Scans codebase for code quality debt,
  test gaps, dependency rot, documentation holes, design debt, and performance issues.
  Use when the user says "/tech-debt", "audit the codebase", "what needs fixing", or
  "how healthy is this codebase?". Also runs as part of /ship pre-flight (lightweight mode)
  and can be invoked from Richter reviews.
---

# Tech Debt — Codebase Health Inventory

**Purpose:** Transform invisible code health problems into a prioritized, actionable list. Not a lecture about best practices — a concrete inventory of what's wrong, how bad it is, and what to fix first.

---

## Invocation

- **Manual:** `/tech-debt` — full inventory of the current project
- **Scoped:** `/tech-debt tests` — test debt only
- **Scoped:** `/tech-debt deps` — dependency debt only
- **Scoped:** `/tech-debt hotspots` — git churn analysis only
- **Lightweight (auto):** When called from `/ship`, runs hotspot + dependency checks only (fast mode)

---

## Step 0: Discover the Project

1. Read `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` for stack info
2. Identify test framework and coverage tools
3. Get project scale: `find . -name '*.ts' -o -name '*.js' -o -name '*.py' | wc -l` (adapt to language)
4. Check for existing quality tools (ESLint config, pylint, rustfmt, etc.)

---

## The 7 Categories

### Category 1: Code Quality Debt

**Detection — run these commands:**

```bash
# Git churn — most-changed files in 90 days (highest debt risk)
git log --format=format: --name-only --since="90 days ago" | sort | uniq -c | sort -rn | head -20

# Long files (>300 lines) — likely need splitting
find . -name '*.ts' -o -name '*.js' -o -name '*.py' | xargs wc -l | sort -rn | head -20

# TODO/FIXME/HACK count
grep -rn "TODO\|FIXME\|HACK\|XXX" --include='*.ts' --include='*.js' --include='*.py' | wc -l
```

**What to flag:**
- Files with high churn AND high complexity (the real debt hotspots)
- Functions > 50 lines or > 4 nesting levels
- Files > 300 lines that aren't generated/config
- God objects with too many responsibilities
- Duplicated logic across files (grep for similar patterns)
- Untracked TODOs — count them, list the top 10 by recency

### Category 2: Test Debt

**Detection:**

```bash
# Coverage report (if available)
npx vitest run --coverage 2>/dev/null || npx jest --coverage 2>/dev/null || pytest --cov 2>/dev/null

# Files with no corresponding test
# Compare src files to test files — find the gaps
```

**What to flag:**
- Source files with zero test coverage
- Critical paths (auth, payments, data mutation) without integration tests
- Test files that only test happy paths — grep for error/edge assertions
- Flaky tests — check CI history if available
- Test execution time if > 2 minutes

### Category 3: Documentation Debt

**What to flag:**
- Missing or outdated README setup instructions
- Public functions/APIs without any documentation
- Stale TODOs that reference completed work
- CHANGELOG not maintained
- Architecture decisions not recorded anywhere

### Category 4: Dependency Debt

**Detection:**

```bash
# Outdated packages
npm outdated 2>/dev/null || pip list --outdated 2>/dev/null

# Unused dependencies
npx depcheck 2>/dev/null

# Security vulnerabilities
npm audit 2>/dev/null || pip-audit 2>/dev/null
```

**What to flag:**
- Packages > 2 major versions behind
- Known CVEs (with severity)
- Unused dependencies bloating the install
- Deprecated packages still in use
- Conflicting or duplicated transitive dependencies

### Category 5: Design Debt

**What to flag:**
- Circular dependencies between modules
- Tight coupling — modules that import from many unrelated modules
- Missing abstraction layers (business logic in route handlers, DB queries in UI code)
- Inconsistent patterns (some endpoints use middleware, others don't)
- Monolithic files that should be split

### Category 6: Infrastructure Debt

**What to flag:**
- Outdated runtime versions (Node < LTS, Python < supported)
- Missing or broken CI/CD pipeline
- Manual deployment steps
- Missing environment variable documentation
- No health check endpoints

### Category 7: Performance Debt

**What to flag:**
- N+1 queries (grep for DB calls inside loops)
- Missing database indexes on frequently queried columns
- Synchronous I/O in async contexts
- Unbounded data fetching (no pagination)
- Missing caching on expensive operations

---

## Severity Formula

For each debt item, calculate priority:

```
Priority = (Churn × Impact × Complexity) / Safety

Where:
- Churn = git commits touching this area in last 90 days (0-5 scale)
- Impact = how many users/features affected (0-5 scale)
- Complexity = how hard it is to work with this code (0-5 scale)
- Safety = test coverage + type safety (1-5 scale, higher = safer)
```

**Priority levels from the score:**
- **CRITICAL** (score > 15): Actively causing bugs or blocking development. Fix this sprint.
- **HIGH** (score 8-15): Slowing the team down. Fix next sprint.
- **MEDIUM** (score 3-8): Annoying but manageable. Fix this quarter.
- **LOW** (score < 3): Minor. Backlog it.

---

## Output Format

### Hotspot Map (always show first)

The top 10 debt hotspots — files with the worst combination of churn + complexity + low safety:

```
DEBT HOTSPOTS (highest risk first):

 1. src/services/PaymentService.ts
    Churn: 47 commits (90d) | Tests: 12% coverage | Complexity: high
    Issues: God object (800 lines), 3 TODOs, no error handling on refund path
    Priority: CRITICAL

 2. src/handlers/auth.ts
    Churn: 31 commits (90d) | Tests: 0% coverage | Complexity: medium
    Issues: No rate limiting, password comparison uses == not timing-safe
    Priority: CRITICAL

[...]
```

### Full Inventory (by category)

For each category, list findings with severity:

```
CATEGORY 1: Code Quality Debt

[CRITICAL] PaymentService.ts — 800 lines, 47 commits in 90d, 12% test coverage
  → Split into PaymentProcessor, RefundHandler, PaymentValidator
  → Effort: 2-3 days | Risk: medium (needs careful test migration)

[HIGH] auth.ts — timing-unsafe password comparison at line 45
  → Replace with crypto.timingSafeEqual
  → Effort: 30 min | Risk: low

[MEDIUM] 47 untracked TODOs across 23 files
  → Top 5 by recency: [list them]
  → Convert to issues or delete stale ones
```

### Summary Dashboard

```
TECH DEBT INVENTORY — [project name]
Date: [date]

Category          | Critical | High | Medium | Low | Total
Code Quality      |    2     |  4   |   8    |  3  |  17
Test Coverage     |    1     |  3   |   5    |  2  |  11
Documentation     |    0     |  1   |   4    |  6  |  11
Dependencies      |    1     |  2   |   3    |  1  |   7
Design            |    0     |  2   |   3    |  1  |   6
Infrastructure    |    0     |  1   |   2    |  2  |   5
Performance       |    0     |  1   |   2    |  1  |   4
─────────────────────────────────────────────────────────
TOTAL             |    4     | 14   |  27    | 16  |  61

TOP 3 ACTIONS (highest ROI):
1. [action] — fixes N issues, effort: X, unblocks: [what]
2. [action] — fixes N issues, effort: X, unblocks: [what]
3. [action] — fixes N issues, effort: X, unblocks: [what]
```

---

## Lightweight Mode (for /ship integration)

When called from `/ship` or `/qa-sentinel`, run only:
1. Hotspot analysis (Category 1 — churn + complexity)
2. Dependency vulnerabilities (Category 4 — CVEs only)
3. Critical TODOs in changed files

Report as a compact block within the parent skill's output, not a full inventory.

---

## What Tech Debt Does NOT Do

- Does not auto-fix anything — inventories and prioritizes
- Does not give time estimates (effort is S/M/L, not "3.5 days")
- Does not lecture about best practices — reports concrete findings
- Does not overlap with Richter (Richter reviews a specific diff; tech-debt audits the whole codebase)
- Does not overlap with security-scan (tech-debt flags dependency CVEs but defers deep security analysis)
