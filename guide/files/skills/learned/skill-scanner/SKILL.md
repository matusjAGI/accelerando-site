---
name: skill-scanner
description: Security scanner for AI agent skills (Cisco AI Defense). Use before installing any new skill or when auditing existing skills. Detects prompt injection, data exfiltration, obfuscation, backdoors, and malicious code patterns using multi-engine analysis.
---

# Skill Scanner (Cisco AI Defense)

Security scanner combining pattern-based detection, behavioral dataflow analysis, and LLM semantic analysis.

## When to Use

- **Before installing any new skill** — the PreToolUse hook runs automatically
- **Periodic audits** — scan all installed skills
- **When reviewing a skill manually** — deep analysis with all engines

## Usage

The scanner CLI is at `~/.claude/skills/skill-scanner/.venv/bin/skill-scanner`.

```bash
# Scan a single skill
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan /path/to/skill

# Scan all skills in a directory
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan-all ~/.claude/skills/

# Include behavioral analysis (slower, more thorough)
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan --behavioral /path/to/skill

# JSON output for parsing
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan --format json /path/to/skill

# SARIF output for CI/CD
~/.claude/skills/skill-scanner/.venv/bin/skill-scanner scan --format sarif /path/to/skill
```

## Threat Categories Detected

| Category | Examples |
|----------|----------|
| Prompt Injection | Jailbreak overrides, instruction manipulation |
| Data Exfiltration | SSH keys, AWS credentials, browser data, crypto wallets |
| Command Injection | eval(), exec(), piped curl, reverse shells |
| Obfuscation | Base64 payloads, hex encoding, variable obfuscation |
| Backdoors | Magic string triggers, persistent behavior changes |
| Supply Chain | Unpinned deps, remote binary downloads |
| Path Traversal | ../../../etc/passwd patterns |

## Interpreting Results

- **CRITICAL** — Immediate action required. Strong malware indicators.
- **HIGH** — Investigate before proceeding. May be false positive for legitimate tools.
- **MEDIUM** — Review context. Common in dev tools (e.g., `.env` examples).
- **LOW/INFO** — Informational. Structure warnings, unpinned versions.

## Automatic Hook

A PreToolUse hook scans SKILL.md files before installation and blocks if HIGH/CRITICAL findings exist. The hook is at `~/.claude/hooks/scan-skill-before-install.sh`.

## References

- [Cisco AI Defense](https://www.cisco.com/site/us/en/products/security/ai-defense/index.html)
- [Threat Taxonomy](https://github.com/cisco-ai-defense/skill-scanner/blob/main/docs/threat-taxonomy.md)
