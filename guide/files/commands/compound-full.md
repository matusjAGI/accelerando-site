## Compound Review Phase

Before execution, run the compound review:

1. Check `ralph/logs/` for recent session findings
2. Look for recurring patterns (3+ occurrences across recent sessions)
3. Flag recurring findings for potential promotion to Dave's checklist

---

## Pre-Flight Gate (Dave Mode 2)

After compound review, run the Dave architect pre-flight:

1. Read `ralph/tasks.md`
2. For each task, evaluate against the Dave Mode 2 checklist (from `~/.claude/skills/dave-architect/SKILL.md`):
   - **Clarity** — Would a fresh Claude instance know exactly what to do?
   - **Scope** — Is this one task or three pretending to be one?
   - **Exit criteria** — How does Ralph know it's done?
   - **Failure modes** — What if it fails halfway?
   - **Dependencies** — Does this need something that doesn't exist yet?
   - **Complexity budget** — Effort proportional to value?
3. Produce pre-flight report:

```
DAVE PRE-FLIGHT: ralph/tasks.md
TASK 1: [name] — STATUS: APPROVED / REWRITTEN / BLOCKED
TASK 2: [name] — STATUS: …
SUMMARY: ✓ N approved, ✏️ N rewritten, ✗ N blocked
```

4. Only proceed with APPROVED or REWRITTEN tasks
5. Log blocked tasks with reasons to `ralph/logs/`
6. If tasks were REWRITTEN, update `ralph/tasks.md` with the revised specs before proceeding

---

## Pull Latest

Pull the latest changes from the remote repository before starting work.

## Execute Tasks

For each APPROVED or REWRITTEN task in `ralph/tasks.md`:

1. Create a feature branch for the task
2. Execute the task per its spec
3. Commit changes with clear commit messages
4. Log progress to `ralph/logs/`

## Post-Execution

1. Run the 3-pass code review on all changes
2. Extract learnings and update project `CLAUDE.md`
3. Flag cross-project learnings for potential promotion to global `~/.claude/CLAUDE.md`
4. Log results to `ralph/logs/`
5. Update `ralph/tasks.md` with completion status
