---
name: google-sheets-chrome-mcp
description: |
  Read Google Sheets data via the Chrome MCP extension when direct API access isn't available.
  Use when: (1) user asks you to read a Google Sheet and you have Chrome MCP tools available,
  (2) you need repeatable access to a live Google Sheet from Claude Code sessions,
  (3) Google Sheets DOM scraping fails (it always will — Sheets uses an opaque canvas/custom renderer),
  (4) CSV export via navigation triggers a download instead of rendering content.
  Covers: gviz API endpoint, CSV parsing with multiline fields, Chrome MCP output sanitization,
  multi-sheet workbooks, and data range discovery.
author: Claude Code
version: 1.0.0
date: 2026-02-09
---

# Reading Google Sheets via Chrome MCP Extension

## Problem
You need to read data from a Google Sheet during a Claude Code session. The user has the Chrome
MCP extension available but no Google Sheets API credentials. Google Sheets' DOM is opaque
(no standard HTML table elements), and several obvious approaches fail silently.

## Context / Trigger Conditions
- User shares a Google Sheets URL and asks you to read/analyze the data
- Chrome MCP tools (`Claude_in_Chrome`) are available in the session
- You don't have Google Sheets API credentials or service account access
- Symptoms of hitting this: `read_page` returns toolbar elements but no cell data,
  `get_page_text` says "body is too large", DOM queries for `table.waffle tbody tr` return empty

## What Doesn't Work (Save Time)

1. **DOM scraping** (`document.querySelectorAll('table.waffle tbody tr')`) — Returns empty.
   Google Sheets renders cells on a canvas, not in standard HTML table elements.

2. **`get_page_text`** — Fails with "page body is too large (likely contains CSS/scripts)".

3. **`read_page` accessibility tree** — Only returns toolbar/menu elements, not cell data.

4. **Navigating to CSV export URL** (`/export?format=csv&gid=0`) — Triggers a file download
   instead of rendering the CSV in the browser tab.

5. **JS `fetch()` to `/export?format=csv`** — The fetch succeeds but returning the result
   through Chrome MCP gets blocked by the cookie/query string security filter.

## Solution

### Step 1: Navigate to the Sheet
```
navigate(tabId, "https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit")
```

### Step 2: Discover Data Range
Press `Ctrl+A` (or `Cmd+A`) then check the Name Box (ref_53 typically) — it shows
the selected range like `A1:L90`, telling you how many rows/columns have data.

### Step 3: Fetch via gviz API (THE KEY INSIGHT)
Use JavaScript `fetch()` from within the sheet's page context to hit the **gviz endpoint**:

```javascript
// For first sheet (gid=0):
fetch('https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:csv&gid=0')
  .then(r => r.text())
  .then(text => { window._sheetCSV = text; })

// For named sheets:
fetch('https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:csv&sheet=Sheet%20Name')
  .then(r => r.text())
  .then(text => { window._sheet2 = text; })
```

This works because:
- The gviz API is a different endpoint than the export URL
- It returns CSV data as a text response (not a file download)
- It inherits the user's Google auth cookies from the page context
- It works on sheets shared with "anyone with the link"

### Step 4: Parse the CSV
Google Sheets CSV has quoted fields with embedded newlines. Use this parser:

```javascript
function parseCSV(csv) {
  const rows = [];
  let row = [];
  let field = '';
  let inQuotes = false;
  for (let i = 0; i < csv.length; i++) {
    const c = csv[i];
    if (inQuotes) {
      if (c === '"' && csv[i+1] === '"') { field += '"'; i++; }
      else if (c === '"') { inQuotes = false; }
      else { field += c; }
    } else {
      if (c === '"') { inQuotes = true; }
      else if (c === ',') { row.push(field); field = ''; }
      else if (c === '\n') { row.push(field); field = ''; rows.push(row); row = []; }
      else { field += c; }
    }
  }
  if (field || row.length) { row.push(field); rows.push(row); }
  return rows;
}
```

### Step 5: Sanitize Before Returning
**CRITICAL**: The Chrome MCP extension blocks responses containing URLs and cookie-like
strings. You MUST strip these before returning data:

```javascript
function sanitize(v, maxLen) {
  return (v || '')
    .replace(/https?:\/\/[^\s"]+/g, '')        // Strip URLs
    .replace(/[^\w\s.,;:!?*()\-\n#]/g, '')     // Strip special chars
    .trim()
    .substring(0, maxLen || 300);
}
```

Without sanitization, you'll see `[BLOCKED: Cookie/query string data]` as the response.

### Step 6: Extract Data in Batches
For large sheets, extract 15-20 rows at a time to stay within response limits:

```javascript
const r = window._parsedRows;
const out = [];
for (let i = 1; i <= 20 && i < r.length; i++) {
  out.push(i + '. ' + sanitize(r[i][2], 70) + ' | ' + sanitize(r[i][3], 60));
}
out.join('\n');
```

## Verification
- `window._sheetCSV.length` should return a positive number
- `window._parsedRows.length` should match the row count from the Name Box range
- Returned data should not show `[BLOCKED: Cookie/query string data]`

## Multi-Sheet Workbooks
- List sheet tabs: `document.querySelectorAll('.docs-sheet-tab')` → returns tab names
- Fetch by name: append `&sheet=URL_ENCODED_NAME` to the gviz URL
- Fetch by gid: append `&gid=NUMBER` (gid=0 is first sheet; other gids vary)

## Edge Cases and Notes
- **Private sheets**: This only works if the user is logged into Google in Chrome and has
  access to the sheet. The fetch inherits their session cookies.
- **Very large sheets (>10K rows)**: The gviz endpoint may time out. Consider using
  `&tq=SELECT%20*%20LIMIT%201000%20OFFSET%200` for pagination.
- **Formulas**: The gviz endpoint returns computed values, not formulas.
- **The `javascript_tool` action must be `javascript_exec`** — other action values fail.
- **Header row**: The gviz CSV always includes column headers as the first row.

## Complete Working Example

```javascript
// 1. Fetch
fetch('https://docs.google.com/spreadsheets/d/SHEET_ID/gviz/tq?tqx=out:csv&gid=0')
  .then(r => r.text())
  .then(text => { window._csv = text; });

// 2. (Next call) Parse
function parseCSV(csv) { /* ... parser above ... */ }
window._rows = parseCSV(window._csv);
'Parsed ' + window._rows.length + ' rows';

// 3. (Next call) Extract with sanitization
function s(v,n) { return (v||'').replace(/https?:\/\/[^\s"]+/g,'').replace(/[^\w\s.,;:!?*()\-\n#]/g,'').trim().substring(0,n||300); }
const out = [];
for (let i = 1; i <= 15; i++) {
  out.push('Row ' + (i+1) + ': ' + s(window._rows[i][0], 70) + ' | ' + s(window._rows[i][1], 70));
}
out.join('\n');
```
