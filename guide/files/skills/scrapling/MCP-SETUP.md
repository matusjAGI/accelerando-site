# Scrapling MCP Server Setup Guide

## What This Gives You

Scrapling's MCP server connects Claude Code/Desktop to advanced web scraping with:
- **Cloudflare Turnstile/Interstitial bypass** out of the box
- **CSS selector pre-filtering** to extract target content BEFORE passing to AI (massive token savings)
- **Parallel URL scraping** - concurrent batch operations
- **Adaptive element tracking** - selectors survive website redesigns
- **Browser fingerprint spoofing** - TLS impersonation, real headers
- **Proxy support** for geo-targeting

Critical advantage: Other MCP scrapers dump entire pages to Claude. Scrapling extracts specific elements FIRST, then passes minimal content to AI.

## Installation

```bash
# Install with AI/MCP capabilities
pip install "scrapling[ai]" --break-system-packages

# Install browser dependencies (REQUIRED - large downloads ~500MB)
scrapling install
```

## Claude Code Integration

Add to your Claude Code MCP configuration:

```bash
# For local/project scope (current project only)
claude mcp add --transport stdio --scope local scrapling -- scrapling mcp

# For global scope (all projects)
claude mcp add --transport stdio --scope global scrapling -- scrapling mcp
```

**Verify installation:**
```bash
claude mcp list
```

You should see `scrapling` in the output.

## Claude Desktop Integration

Edit your Claude Desktop config file:

**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

Add this configuration:

```json
{
  "mcpServers": {
    "scrapling": {
      "command": "scrapling",
      "args": ["mcp"]
    }
  }
}
```

**If you need full path** (recommended for reliability):

```bash
# Find scrapling path
which scrapling
# Example output: /Users/jonathan/.venv/bin/scrapling
```

Then use:

```json
{
  "mcpServers": {
    "scrapling": {
      "command": "/Users/jonathan/.venv/bin/scrapling",
      "args": ["mcp"]
    }
  }
}
```

**Restart Claude Desktop** for changes to take effect.

## Available MCP Tools

Once configured, Claude can use these tools:

1. **get** - Simple HTTP fetch, converts to markdown
2. **fetch** - Dynamic fetch (JavaScript rendering)
3. **stealth** - Anti-bot bypass (Cloudflare, etc.)
4. **get_all** - Batch parallel simple fetches
5. **fetch_all** - Batch parallel dynamic fetches
6. **stealth_all** - Batch parallel stealth fetches

## Usage Patterns

### Simple page scrape
```
Claude: Scrape the main content from https://example.com
```

Claude calls `get` tool, returns markdown.

### JavaScript-heavy site
```
Claude: Extract product listings from https://example.com/products
```

Claude automatically escalates to `fetch` (JavaScript rendering) if `get` fails.

### Bot-protected site
```
Claude: Get past Cloudflare and scrape https://example.com
```

Claude uses `stealth` tool with anti-detection.

### Extract specific elements (TOKEN SAVER!)
```
Claude: Scrape only the article content (CSS selector: .article-body) from these 3 URLs: [list]
```

Claude uses `stealth_all` with CSS selectors, extracts target content BEFORE passing to AI.

**Key advantage:** Without CSS pre-filtering, entire pages dump to Claude (massive token waste). With Scrapling MCP, you extract `.article-body` only, then Claude processes just that.

## Pro Tips

1. **Always specify CSS selectors for large pages** - saves massive tokens
2. **Use batch tools (_all variants) for multiple URLs** - faster parallel execution
3. **Let Claude decide tool escalation** - it will try `get` → `fetch` → `stealth` automatically
4. **Enable adaptive mode for frequently-scraped sites** - survives redesigns

## Troubleshooting

**MCP server not appearing:**
1. Check `scrapling install` completed (browser dependencies)
2. Verify path to scrapling executable is correct
3. Restart Claude Desktop/Code completely
4. Check `claude mcp list` shows scrapling

**Scraping fails:**
- Let Claude retry with higher protection mode
- Add explicit CSS selectors to reduce content
- Check if proxy is needed for geo-restrictions

## When to Use MCP vs Writing Custom Code

**Use MCP when:**
- Quick data extraction tasks
- You want Claude to handle scraping strategy
- Minimal custom processing needed
- Token efficiency matters (CSS pre-filtering)

**Write custom code when:**
- Complex multi-step pipelines (e.g., accelerandoAI article→video)
- Custom data transformation logic
- Integration with other systems (databases, APIs)
- You need the scrapling skill (see SKILL.md)

## Security Note

Scrapling MCP can access any URL Claude requests. Use with caution in:
- Shared development environments
- Projects with untrusted collaborators
- Automated pipelines (could be prompt-injected to scrape unintended URLs)

Consider using project-scope (`--scope local`) rather than global scope for sensitive work.
