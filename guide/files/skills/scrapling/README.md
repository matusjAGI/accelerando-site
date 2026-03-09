# Scrapling Skill

## Two Integration Modes

This skill provides **two ways** to use Scrapling:

### 1. MCP Server (Recommended for Most Cases) 🚀

**Best for:** Zero-code web scraping, letting Claude handle strategy autonomously

**Setup:** `MCP-SETUP.md`

**Key benefits:**
- No Python code needed - Claude calls tools directly
- CSS selector pre-filtering saves massive tokens
- Automatic escalation: get → fetch → stealth
- Parallel batch operations built-in

**Use when:**
- Quick data extraction tasks
- You want Claude to autonomously decide scraping strategy
- Minimal custom processing needed
- Token efficiency matters (CSS pre-filtering extracts target content BEFORE passing to Claude)

### 2. Python Skill (For Custom Pipelines) 🛠️

**Best for:** Complex scraping workflows with custom logic

**Setup:** `SKILL.md`

**Key benefits:**
- Full programmatic control
- Integration with databases, APIs, other systems
- Custom data transformation pipelines
- Fine-grained error handling

**Use when:**
- Building accelerandoAI (article → summarize → video generation)
- Multi-step data processing pipelines
- Database integration required
- Custom business logic around scraping

## Quick Start

### If you want MCP integration (recommended):
```bash
# Read MCP-SETUP.md for full instructions
pip install "scrapling[ai]" --break-system-packages
scrapling install
claude mcp add --transport stdio --scope global scrapling -- scrapling mcp
```

### If you want Python scripting:
```bash
# Read SKILL.md for full documentation
pip install "scrapling[fetchers]" --break-system-packages
scrapling install

# Test it
python3 -c "from scrapling.fetchers import Fetcher; print(Fetcher().get('https://example.com').css('h1::text').get())"
```

## Files in This Skill

- **README.md** (this file) - Overview and quick start
- **MCP-SETUP.md** - MCP server integration guide for Claude Code/Desktop
- **SKILL.md** - Python library documentation for custom pipelines
- **skill.json** - Metadata and triggers

## Key Capabilities

- ✅ **Cloudflare Turnstile/Interstitial bypass** out of the box
- ✅ **Adaptive element tracking** - selectors survive website redesigns
- ✅ **Three fetching strategies** - Fetcher (fast), DynamicFetcher (JS), StealthyFetcher (stealth)
- ✅ **Parallel batch scraping** - concurrent URL processing
- ✅ **Browser fingerprint spoofing** - TLS impersonation, real headers
- ✅ **CSS selector pre-filtering** (MCP) - extract target content before AI processing

## Decision Matrix

| Scenario | Use MCP | Use Python Skill |
|----------|---------|------------------|
| Extract article text from URL | ✅ | |
| Scrape 100 product pages | ✅ | |
| Article → summarize → video pipeline | | ✅ |
| Quick data exploration | ✅ | |
| Integration with database | | ✅ |
| Custom transformation logic | | ✅ |
| Let Claude handle scraping | ✅ | |
| Need full programmatic control | | ✅ |

## Common Questions

**Q: Which should I use for accelerandoAI?**
A: Python skill (`SKILL.md`). You need custom pipeline: scrape → summarize → video generation.

**Q: Can I use both?**
A: Yes! MCP for quick extraction, Python for complex pipelines.

**Q: Does this work with X/Twitter?**
A: Yes, use StealthyFetcher (Python) or stealth tool (MCP) for anti-detection.

**Q: What about Cloudflare protection?**
A: Built-in bypass. StealthyFetcher/stealth tool handles Turnstile automatically.

**Q: How do I save tokens with MCP?**
A: Specify CSS selectors: "Scrape .article-body from URL" extracts target content BEFORE passing to Claude.

## Next Steps

1. **Choose your mode** (MCP or Python)
2. **Read the appropriate guide** (MCP-SETUP.md or SKILL.md)
3. **Install dependencies** (follow guide instructions)
4. **Test with simple scrape** (examples in guides)
5. **Build your pipeline** (integrate with accelerandoAI or other projects)

## Support

This skill is based on [D4Vinci/Scrapling](https://github.com/D4Vinci/Scrapling).

For issues:
- MCP setup problems → check MCP-SETUP.md troubleshooting section
- Python usage questions → see SKILL.md examples and patterns
- Scrapling bugs → report to upstream GitHub repo
