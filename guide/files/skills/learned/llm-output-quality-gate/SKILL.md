---
name: llm-output-quality-gate
description: |
  Detect and reject LLM refusals/errors before storing or displaying output in automated pipelines.
  Use when: (1) an LLM generates content that gets stored in a database or shown to users without
  human review, (2) the LLM input might be incomplete/garbage (web scraping, extraction failures),
  (3) you see "I apologize" or "I cannot" in production output that should be article summaries,
  descriptions, or other content. Covers refusal pattern detection, quality gates for LLM-in-the-loop
  architectures, and input validation before LLM calls.
author: Claude Code
version: 1.0.0
date: 2026-02-11
---

# LLM Output Quality Gate

## Problem

When an LLM is used inside an automated pipeline (e.g., summarizing scraped articles, generating
descriptions, creating metadata), the LLM can produce refusals or error messages instead of the
expected content. If the pipeline doesn't check for this, the refusal text gets stored in the
database and displayed to users as if it were real content.

## Context / Trigger Conditions

- LLM output stored in database without human review
- LLM input comes from unreliable sources (web scraping, user uploads, extraction pipelines)
- Users report seeing "I apologize, but..." or "I cannot..." in places where article summaries,
  descriptions, or generated content should appear
- Pipeline has no validation between LLM response and storage/display

## Solution

### 1. Add a quality gate after every LLM call in automated pipelines

Check the LLM output for refusal patterns before using it:

```python
REFUSAL_PATTERNS = [
    "i apologize",
    "i cannot",
    "i'm unable",
    "article text you intended",
    "article is incomplete",
    "without the full",
    "could you please",
    "please provide",
    "paste the complete",
    "share a working link",
    "i don't have access",
    "i'm not able to",
]

def validate_llm_output(text: str, input_context: str = "") -> str:
    """Validate LLM output is actual content, not a refusal."""
    text_lower = text.lower()
    for pattern in REFUSAL_PATTERNS:
        if pattern in text_lower:
            raise ValueError(
                f"LLM refused to generate content "
                f"(pattern: '{pattern}', context: {input_context})"
            )
    return text
```

### 2. Validate inputs BEFORE sending to the LLM

Don't send garbage to the LLM — validate upstream:

- **Minimum content length**: Set a realistic minimum (e.g., 100 words for articles, not 50)
- **Content type validation**: Is this actually article text, or is it tweet metadata?
- **Extraction success check**: Did the scraper actually get the article body?

### 3. Handle refusal gracefully

When the quality gate catches a refusal:
- **Don't store it**: Set the field to NULL, not the refusal text
- **Fall back**: Use the original content, a default, or skip the feature
- **Log it**: Record the failure for monitoring (input length, source, pattern matched)

## Key Insight

The failure chain is always:
1. Upstream produces bad/insufficient input (scraping failure, timeout, auth wall)
2. Bad input passes weak validation (minimum too low, wrong checks)
3. LLM correctly refuses but the refusal is treated as valid output
4. Refusal text gets stored and displayed to users

Fix at ALL three points, not just one:
- Tighten upstream validation (raise minimums, add content-type checks)
- Add quality gate on LLM output (pattern matching)
- Handle the failure gracefully (NULL, fallback, skip)

## Verification

1. Check database for existing bad content: `SELECT * FROM table WHERE LOWER(column) LIKE '%i apologize%'`
2. Fix existing bad data: `UPDATE table SET column = NULL WHERE ...`
3. Test with intentionally short/garbage input to verify the gate catches it
4. Verify the fallback path works (e.g., video uses full content instead of TLDR)

## Example

From FinishReading.it (the actual bug):

```
Input: 54 words of tweet text (scraper failed to get article body)
LLM prompt: "Summarize this article in 400 words..."
LLM output: "I apologize, but the article text you intended to share is incomplete..."
Result: Refusal text stored as tldr_content, shown in video reply to user
```

Fix applied:
- Raised MIN_WORD_COUNT from 50 → 100 (catches tweet-only extractions)
- Added refusal pattern detection in summarizer (raises ValueError)
- Callers catch ValueError and fall back to full content for video
- Cleared existing bad data from database

## Notes

- Refusal patterns should be lowercase-matched — LLMs vary capitalization
- The patterns list needs periodic updating as LLM behavior evolves
- "i apologize" is better than just "apologize" — avoids false positives on articles
  about customer service that discuss apologies
- This pattern applies to ANY LLM-in-the-loop architecture: summarizers, classifiers,
  metadata generators, content moderators, etc.
- Consider also checking for extremely short outputs relative to the requested length
  (e.g., asked for 400 words but got 20)
