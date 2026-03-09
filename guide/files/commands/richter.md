## Richter — Code Review

Load the Richter skill from `~/.claude/skills/richter/SKILL.md` and begin a code review of the current branch.

1. Read `~/.claude/skills/richter/SKILL.md` for full persona and process
2. Gather context:
   - Run `git diff` against the base branch to identify all changes
   - Run `git diff --stat` to get file count and LOC summary
   - Read project CLAUDE.md if it exists
   - Read the task spec if one exists in `ralph/tasks.md`
3. Ask the user which mode (BIG CHANGE or SMALL CHANGE)
4. Execute the review per the SKILL.md process
5. Log findings to `ralph/logs/` per the logging format in SKILL.md
