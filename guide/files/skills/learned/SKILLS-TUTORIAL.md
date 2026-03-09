# Global Skills Tutorial

Your installed global skills and when to use them.

---

## 1. planning-with-files

**What:** Creates `task_plan.md`, `findings.md`, `progress.md` in your project for complex tasks. Based on Manus AI's "filesystem as memory" pattern.

**When to use:**
- Starting a multi-step task (3+ phases)
- Research projects requiring persistence across sessions
- Any task requiring >5 tool calls
- When you need to track decisions and errors

**How to use:**
```
"Plan this refactoring using planning-with-files"
"Create a task plan for implementing the new auth system"
```

**What happens:**
1. Creates `task_plan.md` with phases, checkboxes, decisions
2. Creates `findings.md` for research notes
3. Creates `progress.md` for session logs
4. Hooks remind me to read/update these files

**Note:** Creates files in your *project* directory, not the skill directory.

---

## 2. agent-browser

**What:** Headless browser automation CLI. Navigate, click, fill forms, screenshot, scrape.

**When to use:**
- Web scraping or data extraction
- Filling forms programmatically
- Testing web apps
- Taking screenshots of pages
- Any browser automation task

**How to use:**
```
"Open example.com and fill out the contact form"
"Take a screenshot of the dashboard after login"
"Scrape all product prices from this page"
```

**Core workflow:**
1. `agent-browser open <url>` — Navigate
2. `agent-browser snapshot -i` — Get element refs (@e1, @e2...)
3. `agent-browser fill @e1 "text"` — Interact using refs
4. Re-snapshot after navigation (refs invalidate)

**Prerequisites:** Requires `npm install -g agent-browser && agent-browser install`

**Note:** You already have Chrome MCP tools. Use this when you need headless/programmatic control vs interactive browser use.

---

## 3. mem-search

**What:** Search persistent memory database across all past sessions (requires claude-mem infrastructure).

**When to use:**
- "Did we already solve this?"
- "How did we fix X last time?"
- "What did we discover about Y last week?"
- Finding past work, decisions, or discoveries

**How to use:**
```
"Search memory for authentication bugs we fixed"
"What did we learn about the payment system?"
"Find all decisions we made about the database schema"
```

**3-layer workflow:**
1. `search(query="...")` — Get index with IDs
2. `timeline(anchor=ID)` — Get context around result
3. `get_observations(ids=[...])` — Fetch full details

**Prerequisites:** Requires claude-mem backend running (SQLite + Chroma). The skill is installed but won't work without the backend service.

---

## 4. skill-scanner (Cisco AI Defense)

**What:** Security scanner for agent skills. Multi-engine detection: pattern matching, behavioral analysis, LLM semantic analysis.

**When to use:**
- **Automatic:** Hook scans SKILL.md before any installation
- Before manually installing a new skill
- Auditing all installed skills
- Reviewing a suspicious skill

**How to use:**
```bash
# Scan single skill
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan /path/to/skill

# Scan all skills
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan-all ~/.claude/skills/

# With behavioral analysis (slower, more thorough)
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan --behavioral /path/to/skill
```

**Auto-protection:** The PreToolUse hook at `~/.claude/hooks/scan-skill-before-install.sh` automatically scans any SKILL.md being written and blocks if HIGH/CRITICAL findings.

---

## Quick Reference

| Skill | Trigger Phrases |
|-------|-----------------|
| planning-with-files | "plan this", "create a task plan", "track this project" |
| agent-browser | "open website", "fill form", "screenshot", "scrape" |
| mem-search | "did we solve", "how did we", "find past work" |
| skill-scanner | Automatic on skill install, or "scan this skill" |

---

## Skill Locations

All skills installed at `~/.claude/skills/`:

```
~/.claude/skills/
├── planning-with-files/   # File-based planning
├── agent-browser/         # Browser automation
├── mem-search/            # Memory search
├── skill-scanner/         # Security scanner
├── claudeception/         # Knowledge extraction
└── ...
```

---

## Prerequisites Summary

| Skill | Setup Required |
|-------|----------------|
| planning-with-files | None — works immediately |
| agent-browser | `npm install -g agent-browser && agent-browser install` |
| mem-search | claude-mem backend (Bun + SQLite + Chroma service) |
| skill-scanner | None — venv pre-installed |
