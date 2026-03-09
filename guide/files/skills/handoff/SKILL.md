---
name: handoff
description: Generate a handoff document for continuing work in a new Claude Code session. Use when the user says /handoff or "create a handoff" or "hand off".
---

# Handoff — Session Continuity Document

**Purpose:** Generate a comprehensive handoff document and print a **ready-to-paste prompt** that kicks off a new Claude Code instance exactly where this session left off.

**Scope:** Global — works on any project.

---

## Invocation

User says `/handoff` or "create a handoff" or "hand off". Takes optional arguments:
- `/handoff` — full handoff with all context
- `/handoff --brief` — shorter version, just key files and next steps

---

## Process

### Step 1: Gather Context Automatically

Run these commands and collect data — don't ask the user for any of this:

1. **Git state** — run `git status`, `git branch --show-current`, `git log --oneline -10`, `git diff --stat` to capture branch, uncommitted changes, recent commits
2. **Project identification** — repo name from directory, working directory path
3. **What was accomplished** — summarize completed work from this session's conversation history
4. **What's in progress** — any unfinished tasks or partially implemented features discussed
5. **What's next** — pending tasks, known issues, user's stated intentions
6. **Key files touched** — files modified, created, or frequently referenced this session (check git status + conversation context)
7. **Architecture decisions made** — any design choices that a new instance needs to know
8. **Active services** — check for running servers/daemons: `lsof -iTCP -sTCP:LISTEN -P | grep -E 'python|node|ngrok'` and `ps aux | grep -E 'daemon|server\.py|ngrok' | grep -v grep`
9. **Environment state** — new env vars, dependencies installed, config changes
10. **Known bugs/issues** — anything broken or flaky the next session should watch for

### Step 2: Read Project Memory

Read the project's `MEMORY.md` and `CLAUDE.md` to include persistent context. Don't duplicate what's already there — the handoff supplements these files with session-specific state.

### Step 3: Generate the Handoff Prompt

**Print a single code block** that the user can copy-paste directly into a new Claude Code session as the opening message. The block must be self-contained — a new instance with no prior context should be able to read it and continue.

Format:

````
```
HANDOFF — [Project Name]
Date: [YYYY-MM-DD ~HH:MM]
Branch: [current branch]

## Session Summary
[2-3 sentences: what was done, what state things are in]

## Completed This Session
- [bullet list of completed work]

## In Progress / Next Steps
- [bullet list with priority order — first item = most important]
- [include any user-stated intentions for next session]
- [flag uncommitted changes prominently if they exist]

## Key Files
- `path/to/file.py` — [NEW|MODIFIED|KEY]: [one-line description of what it does + what changed]
- `path/to/other.js` — [NEW|MODIFIED|KEY]: [one-line description]
[... for ALL relevant files — err on the side of including more, not fewer]

## Active Services
- [service name] on port [N] (PID [X]) — [what it does]
[... or "None currently running"]

## Architecture Decisions
- [any design choices made this session that inform future work]

## Known Issues
- [bugs, flaky behavior, workarounds in place]
[... or "None identified"]

## Environment Changes
- [new env vars (names only, NOT values), installed packages, config changes]

## Key Context
[anything else a fresh instance needs — user preferences, project conventions,
gotchas discovered this session, links to relevant docs/URLs]
```
````

### Step 4: Save to Project Memory

After printing the handoff:
1. Write the handoff to `{project_memory_dir}/session-handoff.md` so it persists even if the user doesn't copy it
2. Update `MEMORY.md` with any new stable patterns or discoveries NOT already captured there

### Step 5: Confirm to User

After generating, tell the user:
- "Handoff ready — copy the block above into a new Claude Code session to continue."
- Mention the file was also saved to project memory as backup
- If there are uncommitted changes, remind them prominently

---

## Anti-Patterns

- NEVER include secrets, API keys, or token values in the handoff — env var names only
- NEVER include full file contents — just paths and summaries
- NEVER skip checking git status — uncommitted changes are the #1 thing that gets lost between sessions
- NEVER skip active services — a new instance that doesn't know about running servers will create port conflicts
- NEVER ask the user what to include — gather everything automatically, the whole point is this is one command
- NEVER generate a handoff shorter than ~30 lines — if the session was trivial, say so, but still capture git state and active services
