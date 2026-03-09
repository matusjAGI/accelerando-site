---
name: security-scan
description: >
  Security audit for codebases. Scans for dependency vulnerabilities, hardcoded secrets,
  input validation gaps, auth issues, and OWASP Top 10 risks. Use when the user says
  "/security-scan", "security audit", "check for vulnerabilities", or "is this secure?".
  Also runs automatically as part of /ship pre-flight and can be invoked from /qa-sentinel.
---

# Security Scan

**Purpose:** Systematic security audit of the current codebase. Finds real vulnerabilities, not theoretical risks. Every finding must reference a specific file and line.

---

## Invocation

- **Manual:** User says `/security-scan`, `"security audit"`, `"check security"`, or `"is this secure?"`
- **Auto-trigger:** Runs as part of `/ship` pre-flight checks
- **Scoped run:** `/security-scan auth` — focus on authentication only
- **Scoped run:** `/security-scan deps` — dependency scan only
- **Scoped run:** `/security-scan secrets` — secrets detection only

---

## Step 0: Detect Stack

1. Identify languages, frameworks, package managers
2. Check for existing security config (`.github/workflows/security.yml`, `.snyk`, `.npmrc`, `bandit.yml`)
3. Identify what changed recently: `git diff --stat HEAD~10` to focus the scan

---

## The 7 Checks

### Check 1: Dependency Vulnerabilities

```bash
# Node.js
npm audit --audit-level=moderate 2>/dev/null

# Python
pip-audit 2>/dev/null || pip check 2>/dev/null

# Rust
cargo audit 2>/dev/null

# Go
govulncheck ./... 2>/dev/null
```

Also check:
- Outdated packages with known CVEs (`npm outdated`, `pip list --outdated`)
- Packages > 2 major versions behind
- Unused dependencies that expand the attack surface

### Check 2: Hardcoded Secrets

Scan for patterns — grep the codebase, not just `.env` files:

**Patterns to detect:**
- API keys: `(?i)(api[_-]?key|apikey)\s*[:=]\s*['"][A-Za-z0-9]{16,}`
- AWS keys: `AKIA[0-9A-Z]{16}`
- Private keys: `-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----`
- JWT secrets: `(?i)(jwt[_-]?secret|token[_-]?secret)\s*[:=]\s*['"].+['"]`
- Database URLs: `(?i)(postgres|mysql|mongodb|redis)://[^/\s]+:[^/\s]+@`
- Generic secrets: `(?i)(password|passwd|secret|token)\s*[:=]\s*['"][^'"]{8,}['"]`

**Check `.env` files:** If present, verify they're in `.gitignore`. If `.env` is tracked in git, that's a CRITICAL finding.

**Check git history:** `git log --all --diff-filter=A -- '*.env' '.env*'` — were secrets ever committed and "removed"?

### Check 3: Authentication & Session Management

- How are passwords stored? (bcrypt/argon2 = good, MD5/SHA1/plaintext = CRITICAL)
- Session tokens: are they cryptographically random? What's the expiry?
- JWT: is the secret strong? Is `alg: none` rejected? Are tokens validated on every request?
- OAuth: is the state parameter used? Are redirect URIs validated?
- Rate limiting: is there brute-force protection on login endpoints?

### Check 4: Input Validation (OWASP Top 10)

For each user-facing endpoint or input handler:

- **SQL injection:** Are queries parameterized? Look for string concatenation in SQL.
- **XSS:** Is user input escaped before rendering in HTML? Check template engines.
- **Command injection:** Is user input passed to `exec()`, `spawn()`, `system()`, `os.popen()`?
- **Path traversal:** Is user input used in file paths without sanitization?
- **SSRF:** Are user-provided URLs fetched without allowlist validation?
- **Deserialization:** Is untrusted data passed to `eval()`, `pickle.loads()`, `JSON.parse()` on unvalidated input?

### Check 5: Data Protection

- Is HTTPS enforced? Check for `http://` URLs in production config.
- Are sensitive fields (passwords, tokens, PII) excluded from logs?
- Is data encrypted at rest where required?
- Are API responses leaking internal data (stack traces, DB schemas, internal IDs)?

### Check 6: Security Headers & CORS

For web apps, check:
- `Content-Security-Policy` header
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options` or CSP `frame-ancestors`
- `Strict-Transport-Security` (HSTS)
- CORS: is `Access-Control-Allow-Origin: *` used? Is it scoped correctly?
- Cookie attributes: `HttpOnly`, `Secure`, `SameSite`

### Check 7: Infrastructure & Config

- Are debug modes disabled in production config?
- Are default credentials changed?
- Is the `.git` directory excluded from deployment?
- Are CI/CD secrets stored securely (not in plaintext config)?
- Docker: is the container running as non-root? Are images pinned to digests?

---

## Output Format

For each finding:

```
FINDING [N]: [title]
Check: [1-7]
Severity: CRITICAL / HIGH / MEDIUM / LOW
[file:line] — [description]

Risk:
  [what could happen if exploited]

Evidence:
  [the specific code, config, or pattern found]

Fix:
  [concrete remediation — code example preferred]
```

**Severity levels:**
- **CRITICAL:** Actively exploitable. Hardcoded secrets, SQL injection, auth bypass. Fix immediately.
- **HIGH:** Exploitable with effort. Missing rate limiting, weak crypto, unvalidated input. Fix before shipping.
- **MEDIUM:** Defense-in-depth gap. Missing headers, overly permissive CORS, verbose errors. Fix soon.
- **LOW:** Best practice violation. Minor config issues, informational findings. Fix when convenient.

---

## Summary

After all checks, render:

```
SECURITY SCAN SUMMARY:

Check 1 (Dependencies):    PASS / FAIL — [N vulns: X critical, Y high]
Check 2 (Secrets):         PASS / FAIL — [N findings]
Check 3 (Auth/Sessions):   PASS / FAIL — [summary]
Check 4 (Input Validation): PASS / FAIL — [N injection risks]
Check 5 (Data Protection):  PASS / FAIL — [summary]
Check 6 (Headers/CORS):    PASS / FAIL — [summary]
Check 7 (Infra/Config):    PASS / FAIL — [summary]

Total findings: N (X critical, Y high, Z medium, W low)

VERDICT: SECURE / AT RISK / CRITICAL ISSUES
```

A single CRITICAL finding = CRITICAL ISSUES verdict.
Any HIGH finding = AT RISK verdict.

---

## What Security Scan Does NOT Do

- Does not run penetration tests or active exploitation
- Does not scan network infrastructure or cloud config (unless in the repo)
- Does not review business logic vulnerabilities (that's QA Sentinel's domain)
- Does not auto-fix — reports findings with remediation guidance
- Does not replace a professional security audit for production systems

---

## Anti-Sycophancy

- Never say "the codebase looks secure" without evidence
- "No findings" requires explaining exactly what was checked
- If the project is too small to have auth/sessions, say so — don't skip checks silently
- If a scan tool isn't available, note it: "npm audit not available — dependency check is incomplete"
