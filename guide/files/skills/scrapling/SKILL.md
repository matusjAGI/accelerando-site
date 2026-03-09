# Scrapling - Adaptive Web Scraping

## Overview

Scrapling is a high-performance Python web scraping library that adapts to website changes, bypasses anti-bot systems, and provides multiple fetching strategies. Use this skill when building custom scraping pipelines, especially for accelerandoAI's article-to-video workflow or other projects requiring programmatic web extraction.

## When to Use This Skill

**Use Scrapling skill for:**
- Custom scraping pipelines with complex data transformation
- Integration with databases, APIs, or other systems
- Projects like accelerandoAI (X/Twitter article → cliff notes video)
- Automated workflows requiring specific scraping logic
- When you need fine-grained control over fetching strategy

**Use Scrapling MCP instead when:**
- Quick one-off data extraction
- You want Claude to handle scraping strategy autonomously
- Minimal custom processing needed
- See `scrapling-mcp-setup.md` for MCP approach

## Installation

```bash
# Basic installation (parser only, no fetchers)
pip install scrapling --break-system-packages

# With fetchers for browser automation
pip install "scrapling[fetchers]" --break-system-packages

# Install browser dependencies (REQUIRED for fetchers)
scrapling install

# Optional: All extras (AI/MCP, shell, fetchers)
pip install "scrapling[all]" --break-system-packages
scrapling install
```

## Core Concepts

### Three Fetching Strategies

Scrapling provides three main fetchers with progressive anti-detection:

1. **Fetcher** - Fast HTTP requests, no JavaScript rendering
   - Use for: Static HTML sites, APIs, simple scraping
   - Speed: Fastest (milliseconds)
   - Detection risk: Medium

2. **DynamicFetcher** - Chromium browser automation
   - Use for: JavaScript-heavy sites, SPAs, dynamic content
   - Speed: Moderate (1-3 seconds)
   - Detection risk: Medium-High

3. **StealthyFetcher** - Modified Camoufox browser with anti-detection
   - Use for: Cloudflare-protected sites, heavy bot detection
   - Speed: Slower (2-5 seconds)
   - Detection risk: Very Low

### Adaptive Element Tracking

Scrapling can learn element locations and relocate them when websites redesign:

```python
from scrapling.fetchers import StealthyFetcher

# Enable adaptive mode
StealthyFetcher.adaptive = True

fetcher = StealthyFetcher()
page = fetcher.fetch('https://example.com')

# Auto-save selector mapping for future use
products = page.css('.product', auto_save=True)

# Later, if site redesigns and .product class changes:
products = page.css('.product', adaptive=True)  # Still finds elements!
```

**How it works:** Scrapling creates a fingerprint of surrounding elements, then uses similarity matching to relocate targets when structure changes.

## Basic Usage Patterns

### Simple Static Scraping

```python
from scrapling.fetchers import Fetcher

# Simple GET request (Fetcher uses .get(), not .fetch())
fetcher = Fetcher()
page = fetcher.get('https://example.com')

# Extract with CSS selectors
title = page.css('h1::text').get()
products = page.css('.product-card')

# Extract with XPath
price = page.xpath('//span[@class="price"]/text()').get()

# Get all matching elements
all_links = page.css('a::attr(href)').getall()
```

### Dynamic Content (JavaScript Rendering)

```python
from scrapling.fetchers import DynamicFetcher

# Fetch page with JavaScript execution
fetcher = DynamicFetcher()
page = fetcher.fetch(
    'https://example.com/products',
    headless=True,           # Run browser in headless mode
    network_idle=True,       # Wait for network to be idle
    wait_after_load=2000     # Additional wait in milliseconds
)

# JavaScript-rendered content now available
products = page.css('.product-card')
```

### Stealth Mode (Anti-Bot Bypass)

```python
from scrapling.fetchers import StealthyFetcher

# Fetch with maximum stealth
fetcher = StealthyFetcher()
page = fetcher.fetch(
    'https://cloudflare-protected-site.com',
    headless=True,
    network_idle=True,
    disable_resources=True         # Block images/fonts for speed
)

# Bypass Cloudflare Turnstile automatically
content = page.css('.main-content').get()
```

### Persistent Sessions

```python
from scrapling.fetchers import StealthySession

# Create persistent session (maintains cookies, headers)
with StealthySession(headless=True) as session:
    # First request (login)
    login_page = session.fetch('https://example.com/login')

    # Subsequent requests maintain session
    dashboard = session.fetch('https://example.com/dashboard')
    profile = session.fetch('https://example.com/profile')

# Session auto-closes, cleans up browsers
```

### Batch Scraping via async_fetch

```python
import asyncio
from scrapling.fetchers import StealthyFetcher

urls = [
    'https://example.com/page1',
    'https://example.com/page2',
    'https://example.com/page3'
]

# Use async_fetch for concurrent scraping
async def scrape_all(urls):
    fetcher = StealthyFetcher()
    tasks = [fetcher.async_fetch(url, headless=True) for url in urls]
    return await asyncio.gather(*tasks)

results = asyncio.run(scrape_all(urls))

for page in results:
    title = page.css('h1::text').get()
    print(title)
```

> **Note:** `fetch_all` does not exist in v0.4.1. Use `async_fetch` with `asyncio.gather` for concurrency.

## Advanced Features

### Proxy Support

```python
fetcher = StealthyFetcher()
page = fetcher.fetch(
    'https://example.com',
    proxy='http://user:pass@proxy.example.com:8080'
)
```

### Custom Headers

```python
fetcher = Fetcher()
page = fetcher.get(
    'https://api.example.com',
    headers={
        'Authorization': 'Bearer TOKEN',
        'Accept': 'application/json'
    }
)
```

### Element Extraction Helpers

```python
# Get text content, cleaned and normalized
text = page.css('.article').get_clean_text()

# Extract all attributes
all_hrefs = page.css('a::attr(href)').getall()

# Regex search within extracted text
matches = page.css('.content').re(r'price: \$(\d+)')

# Get HTML of element
html = page.css('.product').get()

# Get text only (no tags)
text_only = page.css('.product::text').get()
```

### Selector Generation

```python
# Generate CSS selector for an element
selector = page.generate_css_selector(element)

# Generate XPath for an element
xpath = page.generate_xpath(element)
```

## AccelerandoAI Integration Pattern

Example for X/Twitter article scraping pipeline:

```python
from scrapling.fetchers import StealthyFetcher
import json

def scrape_twitter_article(url: str) -> dict:
    """
    Scrape X/Twitter article for accelerandoAI cliff notes pipeline.

    Returns structured data ready for Remotion video generation.
    """
    # Fetch with anti-detection
    fetcher = StealthyFetcher()
    page = fetcher.fetch(
        url,
        headless=True,
        network_idle=True,
        disable_resources=True,  # Speed up by blocking images
    )

    # Extract article components (adapt selectors to actual Twitter structure)
    article_data = {
        'title': page.css('article h1::text', auto_save=True).get(),
        'author': page.css('[data-testid="User-Name"]::text', auto_save=True).get(),
        'content': page.css('[data-testid="tweetText"]', auto_save=True).getall(),
        'publish_date': page.css('time::attr(datetime)', auto_save=True).get(),
        'images': page.css('article img::attr(src)', auto_save=True).getall()
    }

    # Clean and structure for downstream processing
    article_data['content'] = ' '.join([
        elem.get_clean_text() for elem in article_data['content']
    ])

    return article_data

# Use in pipeline
article = scrape_twitter_article('https://x.com/user/status/123456')
# Pass to next stage: article → summarize → generate video
```

## Performance Optimization

### 1. Disable Resources for Speed

```python
fetcher = StealthyFetcher()
page = fetcher.fetch(
    url,
    disable_resources=True,  # Block images, fonts, stylesheets
    headless=True
)
# 2-3x faster for text-only scraping
```

### 2. Use Appropriate Fetcher

```python
# Fast path for static content
page = Fetcher().get(url)  # Milliseconds

# Only use DynamicFetcher if JavaScript required
if page.css('.content').get() is None:
    page = DynamicFetcher().fetch(url)  # 1-3 seconds

# Only use StealthyFetcher for bot-protected sites
if 'cloudflare' in page.text.lower():
    page = StealthyFetcher().fetch(url)  # 2-5 seconds
```

### 3. Async Concurrent Operations

```python
# Scrape multiple URLs concurrently with async_fetch
import asyncio
fetcher = StealthyFetcher()
results = asyncio.run(asyncio.gather(*[fetcher.async_fetch(u, headless=True) for u in urls]))
```

### 4. Reuse Sessions

```python
# Bad: Creates new browser for each request
for url in urls:
    page = StealthyFetcher().fetch(url)  # Slow!

# Good: Reuse browser session
with StealthySession(headless=True) as session:
    for url in urls:
        page = session.fetch(url)  # Fast!
```

## Error Handling

```python
from scrapling.fetchers import StealthyFetcher
from scrapling.exceptions import (
    ScrapingError,
    NavigationError,
    SelectorError
)

try:
    fetcher = StealthyFetcher()
    page = fetcher.fetch(url, headless=True)
    content = page.css('.article-body').get()

    if content is None:
        # Selector didn't match - website structure changed?
        # Try adaptive mode
        content = page.css('.article-body', adaptive=True).get()

except NavigationError as e:
    # Failed to load page (timeout, network error)
    print(f"Navigation failed: {e}")

except SelectorError as e:
    # CSS/XPath selector syntax error
    print(f"Invalid selector: {e}")

except ScrapingError as e:
    # General scraping error
    print(f"Scraping failed: {e}")
```

## Debugging

### Enable Debug Logging

```python
import logging

logging.getLogger("scrapling").setLevel(logging.DEBUG)

# Now see detailed logs of what Scrapling is doing
page = StealthyFetcher().fetch(url)
```

### Interactive Shell

```python
# Launch interactive IPython shell for exploration
# (Requires: pip install "scrapling[shell]")

from scrapling import Shell

shell = Shell()
shell.start()

# Now you can interactively test selectors:
# >>> page = fetch('https://example.com')
# >>> page.css('h1::text').get()
```

### View Response in Browser

```python
# Save fetched page for inspection
page = StealthyFetcher().fetch(url, headless=True)
page.save('debug_page.html')
# Open debug_page.html in browser to see what Scrapling sees
```

## Best Practices for Jonathan's Workflow

1. **Start simple, escalate as needed**
   ```python
   # Try fast path first
   page = Fetcher().get(url)
   if not page.css('.content').get():
       # Escalate to JavaScript rendering
       page = DynamicFetcher().fetch(url)
   ```

2. **Always enable adaptive mode for production scrapers**
   ```python
   StealthyFetcher.adaptive = True
   # Survives website redesigns
   ```

3. **Use CSS selector pre-filtering**
   ```python
   # Extract only what you need - saves memory and processing
   article = page.css('article', auto_save=True).get()
   # Don't extract entire page if you only need one section
   ```

4. **Async for batch operations**
   ```python
   # Scrape multiple articles concurrently
   fetcher = StealthyFetcher()
   pages = asyncio.run(asyncio.gather(*[fetcher.async_fetch(u, headless=True) for u in article_urls]))
   ```

5. **Persistent sessions for multi-step flows**
   ```python
   # Login once, scrape many pages
   with StealthySession() as session:
       session.fetch('login_url')  # Cookies maintained
       for url in dashboard_urls:
           page = session.fetch(url)  # Already authenticated
   ```

## Integration with accelerandoAI Pipeline

```python
# article-to-video pipeline
def process_article_to_video(article_url: str):
    # 1. Scrape article with Scrapling
    article = scrape_twitter_article(article_url)

    # 2. Extract cliff notes (use Claude API or other LLM)
    cliff_notes = extract_cliff_notes(article['content'])

    # 3. Generate video with Remotion
    video_path = generate_remotion_video({
        'title': article['title'],
        'author': article['author'],
        'notes': cliff_notes,
        'images': article['images']
    })

    return video_path
```

## Common Pitfalls

1. **Not installing browser dependencies**
   - Always run `scrapling install` after pip install
   - Required for DynamicFetcher and StealthyFetcher

2. **Using wrong fetcher for task**
   - Don't use StealthyFetcher for simple static sites (overkill, slow)
   - Don't use Fetcher for JavaScript-rendered content (won't work)

3. **Not handling selector failures**
   - Always check if `.get()` returns None
   - Use `.get(default='fallback')` or try/except

4. **Creating too many browser instances**
   - Use persistent sessions for multiple requests
   - Use `async_fetch` with `asyncio.gather` for concurrent scraping, not sequential loops

5. **Ignoring adaptive mode benefits**
   - Use `auto_save=True` on selectors for resilience

## Summary

Scrapling is your high-performance, anti-detection scraping library. Key capabilities:

- **Three fetching strategies**: Fetcher (fast), DynamicFetcher (JavaScript), StealthyFetcher (anti-bot)
- **Adaptive element tracking**: Survives website redesigns
- **Cloudflare bypass**: Built-in Turnstile/Interstitial handling
- **Async scraping**: Concurrent operations via `async_fetch`
- **Sessions**: Persistent cookies/auth across requests

For accelerandoAI and similar pipelines, Scrapling handles the scraping layer while you focus on data transformation and video generation logic.

**Next steps:**
1. Install: `pip install "scrapling[fetchers]" --break-system-packages && scrapling install`
2. Test simple scrape: `page = Fetcher().get('https://example.com')`
3. Build custom pipeline integrating with your existing tools

**MCP alternative:** See `scrapling-mcp-setup.md` for zero-code integration with Claude Code/Desktop.
