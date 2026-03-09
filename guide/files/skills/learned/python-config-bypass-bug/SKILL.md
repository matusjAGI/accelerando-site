---
name: python-config-bypass-bug
description: |
  Detect and fix env var config bypasses in Python projects. Use when: (1) you fix
  an env var loading issue in a centralized config module but the bug persists in
  production, (2) grep reveals os.environ.get() or os.getenv() calls outside the
  config module, (3) a library or scraper class loads its own API key directly from
  os.environ in __init__. Root cause: individual modules read env vars directly,
  bypassing centralized config sanitization (strip, validation, defaults).
author: Claude Code
version: 1.0.0
date: 2026-02-05
---

# Python Config Bypass Bug

## Problem
You fix an environment variable issue in a centralized config module (e.g., adding
`.strip()` to remove trailing newlines), but the bug persists because other modules
load the same env var directly via `os.environ.get()` or `os.getenv()`, bypassing
the fix entirely.

## Context / Trigger Conditions
- Fixed an env var issue in `config.py` but production still shows the old behavior
- A module's `__init__` has `self.api_key = api_key or os.environ.get("API_KEY")`
- Multiple files load the same env var independently
- Trailing whitespace, newlines (`%0A` in URLs), or missing defaults persist after
  a config fix

## Solution

### 1. Audit all env var loading sites
After fixing config, always grep for direct env var access:

```bash
grep -rn "os\.environ\|os\.getenv" bot/ --include="*.py" | grep -v config.py
```

### 2. Fix at every usage site
Either:
- **Option A**: Make the module use the centralized config instead of `os.environ`
- **Option B**: Apply the same sanitization (`.strip()`, validation) at each direct
  usage site

Option B is simpler when the module has its own `__init__(api_key=None)` pattern
that falls back to `os.environ`:

```python
# Before (bug: trailing newline passes through)
self.api_key = api_key or os.environ.get("SCRAPINGBEE_API_KEY")

# After (stripped at source)
self.api_key = (api_key or os.environ.get("SCRAPINGBEE_API_KEY") or "").strip() or None
```

### 3. Prevent recurrence
Add a comment in the config module listing known bypass sites:

```python
# NOTE: These modules also load env vars directly (not through config):
# - bot/content/scrapingbee.py:28
# - bot/content/browserbase_scraper.py:35
```

## Verification
1. Deploy the fix
2. Check production logs for the specific symptom (e.g., `%0A` in URLs, auth failures)
3. Confirm the sanitized value is being used at the actual call site, not just in config

## Example

**Scenario**: ScrapingBee API calls fail with 401. Logs show `api_key=...KEY%0A` (trailing
newline URL-encoded). You add `.strip()` to `config.py`. Deploy. Bug persists.

**Root cause**: `scrapingbee.py` line 28 reads `os.environ.get("SCRAPINGBEE_API_KEY")`
directly, never touching `config.py`.

**Fix**: Add `.strip()` at `scrapingbee.py:28`. Deploy. Logs now show clean API key.

## Notes
- This pattern is common in Python projects where individual service classes have
  `__init__(self, api_key=None)` with env var fallbacks
- The `FallbackExtractor` pattern (instantiating scrapers in its own `__init__`) means
  the scraper reads env vars at import time, not when config is accessed
- `dotenv` / `load_dotenv()` does NOT strip values — it preserves whitespace from `.env`
- Railway, Heroku, and similar platforms can introduce trailing newlines in env vars
  depending on how they're set (web UI paste, CLI, config files)
