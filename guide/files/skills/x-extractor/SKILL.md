---
name: x-extractor
description: "Extract full content from X/Twitter posts using the finishreading.it API. Use when the user shares an x.com or twitter.com URL, asks to extract/read/summarize a tweet, or needs tweet content for analysis. Handles single posts and threads."
---

# X Post Extractor

Extract full content from X/Twitter posts via the finishreading.it API.

## When to Use

- User shares an x.com or twitter.com URL
- User asks to "extract", "read", "summarize", or "analyze" a tweet/post
- User wants tweet content (text, media, engagement stats) for any purpose
- User provides multiple X URLs for batch extraction

## Authentication

The API requires a session cookie. Authenticate first, then use the authenticated endpoint.

### Step 1: Login (once per session)

```bash
curl -s -X POST 'https://www.finishreading.it/api/extract/auth' \
  -H "Content-Type: application/json" \
  -d '{"password": "extract"}' \
  -c /tmp/finishreading-cookies.txt
```

This stores a cookie (`extract_auth=authenticated`) valid for 30 days.

### Step 2: Extract

```bash
curl -s -X POST 'https://www.finishreading.it/api/extract' \
  -H "Content-Type: application/json" \
  -b /tmp/finishreading-cookies.txt \
  -d '{"url": "https://x.com/user/status/1234567890"}'
```

**Always use the authenticated endpoint** (`/api/extract`), not the public one (`/api/extract/public`). The public endpoint often returns false 404s.

### Response Format

```json
{
  "success": true,
  "id": 55,
  "title": "Post title or first line",
  "text": "Full formatted post content with markdown...",
  "username": "handle"
}
```

Fields:
- `success` — Boolean
- `id` — Internal extraction ID
- `title` — Title extracted from the post/thread
- `text` — Full post content, formatted as markdown. Includes author, date, engagement stats, and the complete text (including long-form articles/threads)
- `username` — Author's X handle

### Error Responses

| Code | Meaning |
|------|---------|
| 400 | Bad request — invalid URL format |
| 401 | Not authenticated — run login step first |
| 404 | Tweet deleted, private, or author suspended |
| 429 | Rate limited — wait and retry |
| 500 | Server error |

Error format:
```json
{ "success": false, "error": "description" }
```

## Usage Patterns

### Single Post Extraction

```bash
# Login
curl -s -X POST 'https://www.finishreading.it/api/extract/auth' \
  -H "Content-Type: application/json" \
  -d '{"password": "extract"}' \
  -c /tmp/finishreading-cookies.txt

# Extract
curl -s -X POST 'https://www.finishreading.it/api/extract' \
  -H "Content-Type: application/json" \
  -b /tmp/finishreading-cookies.txt \
  -d '{"url": "https://x.com/user/status/1234567890"}' | python3 -m json.tool
```

### Batch Extraction

Login once, then extract multiple URLs in parallel using multiple curl calls. Present results in a unified format.

### After Extraction

Once content is extracted, offer to:
1. **Summarize** — Key points from the post/thread
2. **Analyze** — Engagement metrics, sentiment, audience response
3. **Extract quotes** — Pull notable quotes for reference
4. **Save** — Write content to a file for later use
5. **Compare** — If multiple posts, compare perspectives/positions

## Implementation Notes

- Always use `curl` via Bash tool (not WebFetch — this is a POST API)
- **Always authenticate first** — the cookie file persists across calls
- If you get a 401, re-run the login step
- Parse JSON response with `python3 -m json.tool` for readability
- If rate limited (429), wait 5 seconds and retry once
- URL normalization: accept `twitter.com` URLs, they work the same as `x.com`
- The `text` field can be very long (30K+ chars) for article-style posts — truncate when summarizing
- Cookie file at `/tmp/finishreading-cookies.txt` — reuse across extractions in the same session
