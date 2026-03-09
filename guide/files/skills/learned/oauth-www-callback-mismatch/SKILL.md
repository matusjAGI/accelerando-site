---
name: oauth-www-callback-mismatch
description: |
  Fix OAuth redirect_uri_mismatch and "You weren't able to give access to the App" errors
  caused by www vs non-www domain mismatch. Use when: (1) OAuth fails with redirect_uri_mismatch
  (Google) or "couldn't give access" (X/Twitter), (2) callback URLs are correctly configured
  but OAuth still fails, (3) site uses Vercel or other hosting that auto-redirects www/non-www,
  (4) NextAuth OAuth providers fail in production but work locally. Root cause is usually
  that the browser redirects to www.domain.com but OAuth provider only has domain.com callback.
author: Claude Code
version: 1.0.0
date: 2026-01-28
---

# OAuth www vs non-www Callback URL Mismatch

## Problem
OAuth authentication fails in production with errors like:
- Google: `redirect_uri_mismatch` (Error 400)
- X/Twitter: "You weren't able to give access to the App"

This happens even when callback URLs appear correctly configured.

## Context / Trigger Conditions
- OAuth works locally but fails in production
- Using NextAuth.js with Twitter/X or Google providers
- Site is hosted on Vercel (or similar) which auto-redirects between www and non-www
- Callback URL in OAuth provider console matches NEXTAUTH_URL but OAuth still fails
- Error appears immediately on the OAuth provider's authorization page (not after returning)

## Root Cause
Hosting platforms like Vercel often redirect users between `www.domain.com` and `domain.com`.
When a user visits `domain.com`, they may be redirected to `www.domain.com`. NextAuth then
constructs the callback URL using the current hostname (`www.domain.com`), but the OAuth
provider only has `domain.com` registered.

## Solution

### Step 1: Identify the Actual Redirect URI
Check the OAuth authorization URL in the browser address bar. Look for the `redirect_uri`
parameter (URL-encoded). Decode it to see what callback URL is being requested.

Example:
```
redirect_uri=https%3A%2F%2Fwww.example.com%2Fapi%2Fauth%2Fcallback%2Ftwitter
```
Decodes to: `https://www.example.com/api/auth/callback/twitter`

### Step 2: Add Both Callback URLs to OAuth Providers

**For X/Twitter Developer Portal:**
1. Go to your app → User authentication settings → Edit
2. Under "Callback URI / Redirect URL", click "Add another"
3. Add both:
   - `https://domain.com/api/auth/callback/twitter`
   - `https://www.domain.com/api/auth/callback/twitter`
4. Save changes

**For Google Cloud Console:**
1. Go to APIs & Services → Credentials → OAuth 2.0 Client ID
2. Under "Authorized redirect URIs", add both:
   - `https://domain.com/api/auth/callback/google`
   - `https://www.domain.com/api/auth/callback/google`
3. Save (may take 5 minutes to propagate)

### Step 3: Verify
Test OAuth login again. The authorization screen should now appear instead of the error.

## Alternative Solution
Instead of adding both URLs, you can force consistent domain usage:

1. Set `NEXTAUTH_URL` in Vercel to match your canonical domain exactly
2. Configure Vercel to redirect all traffic to one version (www or non-www)
3. Ensure DNS and hosting settings are consistent

However, adding both callback URLs is faster and more resilient.

## Verification
- OAuth authorization screen appears (shows "App wants to access your account")
- After authorizing, user is redirected back to the app successfully
- No redirect_uri_mismatch or access errors

## Example
```
# Before (broken)
X callback URL: https://finishreading.ai/api/auth/callback/twitter
Browser URL: https://www.finishreading.ai/login
Result: Mismatch error

# After (working)
X callback URLs:
  - https://finishreading.ai/api/auth/callback/twitter
  - https://www.finishreading.ai/api/auth/callback/twitter
Result: OAuth works regardless of www redirect
```

## Notes
- This issue is common with Vercel, Netlify, and Cloudflare Pages deployments
- The error message doesn't indicate the www mismatch—you must inspect the URL
- Google shows `redirect_uri_mismatch` explicitly; X/Twitter shows a generic error
- Some OAuth providers (like GitHub) are more lenient; X and Google are strict
- Always add both www and non-www callbacks proactively when setting up OAuth

## Related Issues
- Missing NEXTAUTH_URL environment variable in production
- Using wrong OAuth app credentials (e.g., bot app vs web app)
- OAuth 2.0 not enabled on the app (X requires explicit setup)

## References
- NextAuth.js Twitter Provider: https://next-auth.js.org/providers/twitter
- NextAuth.js Google Provider: https://next-auth.js.org/providers/google
- Vercel Domain Configuration: https://vercel.com/docs/projects/domains
