# Installing Scrapling Skill on Your Machines

## Context
This skill is synced across machines via the `~/.claude-shared/` git repo (github.com/matusjAGI/claude-global-config). Cron auto-pulls every 5 minutes.

## Installation Steps

### 1. Skill Files (Auto-Synced)

Skill files live at `~/.claude-shared/skills/scrapling/`. They sync automatically via the cron job. No manual action needed on secondary machines.

### 2. Install Scrapling

Choose your integration mode:

**For MCP Server (recommended):**
```bash
pip install "scrapling[ai]" --break-system-packages
scrapling install  # Downloads ~500MB of browser dependencies
claude mcp add --transport stdio --scope global scrapling -- scrapling mcp
claude mcp list  # Verify scrapling appears
```

**For Python Skill:**
```bash
pip install "scrapling[fetchers]" --break-system-packages
scrapling install
python3 -c "from scrapling.fetchers import Fetcher; print(Fetcher().get('https://example.com').css('h1::text').get())"
```

**For Both:**
```bash
pip install "scrapling[all]" --break-system-packages
scrapling install
claude mcp add --transport stdio --scope global scrapling -- scrapling mcp
```

### 3. Verify Installation

**MCP mode:**
```bash
claude mcp list
# Should show 'scrapling' in the list
```

**Python mode:**
```bash
python3 << 'PYEOF'
from scrapling.fetchers import StealthyFetcher
StealthyFetcher.adaptive = True
page = StealthyFetcher.fetch('https://example.com', headless=True)
print(page.css('h1::text').get())
PYEOF
```

### 4. Sync Across Your Fleet

Files auto-sync via `~/.claude-shared/auto-sync.sh` (cron every 5 min, git pull --rebase).

To force immediate sync on another machine:
```bash
cd ~/.claude-shared && git pull
```

**Note:** Only skill files sync automatically. Each machine still needs `pip install` and `scrapling install` run locally (browser binaries are platform-specific).

### 5. Test on Each Machine

**Quick MCP test:**
Ask Claude Code: "Use the scrapling MCP tool to scrape the main heading from https://example.com"

**Quick Python test:**
```bash
python3 -c "from scrapling.fetchers import Fetcher; print(Fetcher().get('https://example.com').css('h1::text').get())"
```

## Troubleshooting

**MCP server not appearing:**
1. Check `scrapling install` completed successfully
2. Verify path: `which scrapling`
3. Restart Claude Desktop/Code
4. Check `claude mcp list`

**Browser dependencies failing:**
```bash
# Manual browser installation
scrapling install

# If that fails, try with specific browsers
pip show scrapling  # Shows installation path
```

**Import errors in Python:**
```bash
# Verify installation
pip show scrapling

# Check dependencies
pip install "scrapling[fetchers]" --break-system-packages --upgrade
```

## Machine-Specific Notes

1. **Install on Mac Mini first** (primary development machine)
2. **Test thoroughly** before syncing to other machines
3. **Browser dependencies are ~500MB** per machine - expect long first install
4. **MCP config location**: `~/Library/Application Support/Claude/claude_desktop_config.json` (if using Claude Desktop)

## What You Get

After installation across your fleet:
- ✅ Global skill available in Claude Code on all machines
- ✅ MCP tools available in Claude Desktop (if configured)
- ✅ Synchronized via existing git-based skills system
- ✅ Automatic updates when cron pulls on any machine
