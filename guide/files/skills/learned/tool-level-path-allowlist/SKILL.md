---
name: tool-level-path-allowlist
description: |
  Prevent sandbox bypass by enforcing path allowlists at tool entry points, not just file I/O.
  Use when: (1) implementing plugin/tool systems with filesystem access, (2) search/list/find tools
  that traverse directories, (3) sandbox mode restricts file access. Addresses attack: tools that
  read directory contents bypass file-level restrictions. Critical for: agent frameworks, plugin systems,
  sandboxed environments, file browser tools, search utilities. Applies to: Claude Code, LangChain tools,
  VSCode extensions, any tool system with filesystem access.
author: Claude Code
version: 1.0.0
date: 2026-02-16
---

# Tool-Level Path Allowlist Enforcement

## Problem

Path allowlists are often enforced at the file I/O level (read_file, write_file, edit_file) but
NOT at the tool level for operations like search, list_files, or find. This creates a **complete
sandbox bypass** where directory traversal tools can access arbitrary filesystem locations despite
allowlist restrictions.

**Vulnerable Architecture:**
```
┌─────────────────────────────────────┐
│ Tool Layer                          │
│  ├─ read_file()  ✅ Checks allowlist│
│  ├─ write_file() ✅ Checks allowlist│
│  ├─ search()     ❌ NO CHECK        │  ← VULNERABILITY
│  └─ list_files() ❌ NO CHECK        │  ← VULNERABILITY
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ File I/O Layer (isPathAllowed)     │
│  ✅ Validates paths for read/write  │
│  ❌ Never reached by search/list    │
└─────────────────────────────────────┘
```

## Context / Trigger Conditions

**When This Vulnerability Exists:**

1. **Sandbox Mode Implementation:**
   - Restricts file operations to working directory
   - Uses path allowlist for security boundary
   - Assumes all filesystem access goes through validated paths

2. **Tool Systems with Directory Operations:**
   - search(pattern, path) - searches files for content
   - list_files(path, pattern) - lists directory contents
   - find(path, name) - finds files by name
   - glob(pattern) - expands glob patterns

3. **Attack Scenario:**
   ```
   Sandbox Mode: Restricts to /home/user/workspace
   Attacker Goal: Read /etc/passwd

   BLOCKED: read_file("/etc/passwd")
   → Path allowlist check → DENIED

   BYPASSED: search("root", "/etc")
   → No path allowlist check → SUCCESS
   → Returns: "/etc/passwd:1:root:x:0:0:..."
   ```

**Symptoms:**
- Sandbox can be completely bypassed via directory traversal tools
- Data exfiltration possible despite path restrictions
- Security boundary only protects direct file I/O, not discovery

## Solution

**Principle:** Enforce path allowlist at EVERY tool entry point that accepts a path parameter.

### Step 1: Identify All Filesystem-Accessing Tools

Audit tool definitions for any that accept path/directory parameters:

```zig
// File I/O tools (usually protected)
read_file(path)     → Needs allowlist check ✅
write_file(path)    → Needs allowlist check ✅
edit_file(path)     → Needs allowlist check ✅

// Directory traversal tools (often unprotected!)
search(pattern, path)         → Needs allowlist check ❌
list_files(path, pattern)     → Needs allowlist check ❌
find(path, name)              → Needs allowlist check ❌
glob(pattern)                 → Needs allowlist check ❌
watch(path)                   → Needs allowlist check ❌
```

### Step 2: Add Allowlist Check at Tool Entry Point

**BEFORE (Vulnerable):**
```zig
fn executeSearch(allocator: std.mem.Allocator, input: []const u8) ToolResult {
    const pattern = json.extractString(input, "pattern") orelse return error;
    const search_path = json.extractString(input, "path") orelse ".";
    // NO VALIDATION - proceeds to search arbitrary paths
    var results = searchDirectory(search_path, pattern);
    return .{ .output = results };
}
```

**AFTER (Secure):**
```zig
fn executeSearch(allocator: std.mem.Allocator, input: []const u8) ToolResult {
    const pattern = json.extractString(input, "pattern") orelse return error;
    const search_path = json.extractString(input, "path") orelse ".";

    // CRITICAL: Enforce path allowlist at tool entry point
    if (!isPathAllowed(search_path)) {
        return .{
            .output = "Search path not allowed",
            .is_error = true,
        };
    }

    // Now safe to proceed
    var results = searchDirectory(search_path, pattern);
    return .{ .output = results };
}
```

### Step 3: Apply to All Filesystem Tools

**Complete Fix Pattern:**

```zig
// 1. read_file - Already protected
fn executeReadFile(allocator: Allocator, input: []const u8) ToolResult {
    const path = extractPath(input);
    if (!isPathAllowed(path)) return error.PathNotAllowed;  // ✅
    return readFileContents(path);
}

// 2. write_file - Already protected
fn executeWriteFile(allocator: Allocator, input: []const u8) ToolResult {
    const path = extractPath(input);
    if (!isPathAllowed(path)) return error.PathNotAllowed;  // ✅
    return writeFileContents(path, content);
}

// 3. search - ADD PROTECTION
fn executeSearch(allocator: Allocator, input: []const u8) ToolResult {
    const path = extractPath(input) orelse ".";
    if (!isPathAllowed(path)) return error.PathNotAllowed;  // ← ADD THIS
    return searchDirectory(path, pattern);
}

// 4. list_files - ADD PROTECTION
fn executeListFiles(allocator: Allocator, input: []const u8) ToolResult {
    const path = extractPath(input) orelse ".";
    if (!isPathAllowed(path)) return error.PathNotAllowed;  // ← ADD THIS
    return listDirectory(path, pattern);
}

// 5. Any other filesystem tools - ADD PROTECTION
fn executeFind(allocator: Allocator, input: []const u8) ToolResult {
    const path = extractPath(input) orelse ".";
    if (!isPathAllowed(path)) return error.PathNotAllowed;  // ← ADD THIS
    return findInDirectory(path, name);
}
```

## Verification

### Manual Testing

**Test Case 1: Direct File Access (Should be blocked already)**
```bash
# In sandbox mode, restricted to /home/user/workspace
read_file("/etc/passwd")
→ Expected: "Path not allowed" ✅

read_file("/home/user/.ssh/id_rsa")
→ Expected: "Path not allowed" ✅
```

**Test Case 2: Search/List Bypass (Should NOW be blocked)**
```bash
# BEFORE FIX: These would succeed
search("password", "/etc")
→ Expected (BEFORE): Returns /etc/passwd contents ❌
→ Expected (AFTER): "Search path not allowed" ✅

list_files("/home/user/.ssh")
→ Expected (BEFORE): Returns "id_rsa, id_rsa.pub, ..." ❌
→ Expected (AFTER): "Directory path not allowed" ✅

# AFTER FIX: Only allowed paths work
search("password", ".")
→ Expected: Searches current directory (if allowed) ✅

list_files("/home/user/workspace/subdir")
→ Expected: Lists subdir (if under allowed root) ✅
```

### Automated Test Suite

```python
import pytest

def test_search_respects_sandbox(agent, sandbox_root):
    """Verify search tool enforces path allowlist"""
    agent.enable_sandbox(sandbox_root)

    # Should be blocked
    result = agent.call_tool("search", {
        "pattern": "password",
        "path": "/etc"
    })
    assert result.is_error
    assert "not allowed" in result.output.lower()

    # Should succeed (within sandbox)
    result = agent.call_tool("search", {
        "pattern": "test",
        "path": "."
    })
    assert not result.is_error

def test_list_files_respects_sandbox(agent, sandbox_root):
    """Verify list_files tool enforces path allowlist"""
    agent.enable_sandbox(sandbox_root)

    # Should be blocked
    result = agent.call_tool("list_files", {
        "path": "/home/victim/.ssh"
    })
    assert result.is_error
    assert "not allowed" in result.output.lower()

    # Should succeed (within sandbox)
    result = agent.call_tool("list_files", {
        "path": "./src"
    })
    assert not result.is_error
```

### Security Audit Checklist

When reviewing tool systems for this vulnerability:

- [ ] List all tools that accept path/directory parameters
- [ ] Verify EACH tool calls `isPathAllowed()` or equivalent
- [ ] Test sandbox bypass via search/list/find tools
- [ ] Check glob expansion doesn't bypass allowlist
- [ ] Verify recursive directory operations stay within bounds
- [ ] Test with absolute paths, relative paths, and `.`
- [ ] Test with symlinks pointing outside sandbox

## Example: Real-World Exploit

**YoctoClaw Sandbox Bypass (Fixed 2026-02-16):**

```zig
// BEFORE FIX (Vulnerable Code)
fn executeSearch(allocator: std.mem.Allocator, input: []const u8) ToolResult {
    const pattern = json.extractString(input, "pattern") orelse {
        return .{ .output = "Missing 'pattern' parameter", .is_error = true };
    };
    const search_path = json.extractString(input, "path") orelse ".";
    // BUG: No isPathAllowed check!
    const unescaped_pattern = json.unescape(allocator, pattern) catch pattern;
    var results = std.ArrayList(u8).init(allocator);
    searchDir(allocator, search_path, unescaped_pattern, &results, ...) catch |err| {
        return .{ .output = "Search failed", .is_error = true };
    };
    return .{ .output = results.toOwnedSlice(), .is_error = false };
}
```

**Exploit:**
```python
# Attacker in sandbox mode (restricted to /tmp/yoctoclaw-sandbox)
agent.tool("search", {
    "pattern": "BEGIN RSA PRIVATE KEY",
    "path": "/home/victim/.ssh"
})
→ Returns: "/home/victim/.ssh/id_rsa:1:-----BEGIN RSA PRIVATE KEY-----\n..."
→ SSH private key exfiltrated despite sandbox!

agent.tool("list_files", {
    "path": "/etc"
})
→ Returns: "/etc/passwd\n/etc/shadow\n/etc/sudoers\n..."
→ Complete filesystem disclosure!
```

**After Fix:**
```zig
fn executeSearch(allocator: std.mem.Allocator, input: []const u8) ToolResult {
    const pattern = json.extractString(input, "pattern") orelse { ... };
    const search_path = json.extractString(input, "path") orelse ".";

    // CRITICAL FIX: Enforce allowlist at tool entry
    if (!isPathAllowed(search_path)) {
        return .{ .output = "Search path not allowed", .is_error = true };
    }

    // ... rest of implementation
}
```

## Language-Specific Implementations

### Python (LangChain Tool Example)

```python
from langchain.tools import BaseTool
from pathlib import Path

class SecureSearchTool(BaseTool):
    name = "search"
    description = "Search files for a pattern"
    allowed_root: Path

    def _run(self, pattern: str, path: str = ".") -> str:
        # CRITICAL: Check path allowlist FIRST
        if not self._is_path_allowed(path):
            return "Error: Search path not allowed"

        # Safe to proceed
        return self._search_directory(path, pattern)

    def _is_path_allowed(self, path: str) -> bool:
        try:
            resolved = Path(path).resolve()
            return resolved.is_relative_to(self.allowed_root)
        except (ValueError, RuntimeError):
            return False
```

### TypeScript (VSCode Extension Example)

```typescript
import * as vscode from 'vscode';
import * as path from 'path';
import * as fs from 'fs';

export class SecureFileTool {
    private allowedRoot: string;

    constructor(workspaceRoot: string) {
        this.allowedRoot = fs.realpathSync(workspaceRoot);
    }

    async search(pattern: string, searchPath: string): Promise<string[]> {
        // CRITICAL: Validate path first
        if (!this.isPathAllowed(searchPath)) {
            throw new Error('Search path not allowed');
        }

        // Safe to proceed with actual search
        return this.performSearch(searchPath, pattern);
    }

    private isPathAllowed(targetPath: string): boolean {
        const resolved = path.resolve(targetPath);
        const real = fs.existsSync(resolved)
            ? fs.realpathSync(resolved)
            : fs.realpathSync(path.dirname(resolved));

        return real.startsWith(this.allowedRoot);
    }
}
```

### Rust (Cargo Plugin Example)

```rust
use std::path::{Path, PathBuf};
use std::fs;

pub struct SecureSearchTool {
    allowed_root: PathBuf,
}

impl SecureSearchTool {
    pub fn search(&self, pattern: &str, path: &Path) -> Result<Vec<String>, String> {
        // CRITICAL: Validate path first
        if !self.is_path_allowed(path)? {
            return Err("Search path not allowed".to_string());
        }

        // Safe to proceed
        self.perform_search(path, pattern)
    }

    fn is_path_allowed(&self, path: &Path) -> Result<bool, String> {
        let canonical = fs::canonicalize(path)
            .or_else(|_| fs::canonicalize(path.parent().unwrap()))?;

        Ok(canonical.starts_with(&self.allowed_root))
    }
}
```

## Notes

### Why This Vulnerability Is Common

1. **Assumption of File I/O Gatekeeping:**
   - Developers assume path validation at file read/write is sufficient
   - Directory traversal tools seem "read-only" and harmless
   - Not obvious that listing files can exfiltrate data

2. **Separation of Concerns:**
   - Tool layer handles business logic
   - File I/O layer handles security
   - **Gap:** Tools that don't go through file I/O layer

3. **Incremental Development:**
   - Initial file I/O tools get security review
   - Later added search/list tools bypass existing checks

### Defense-in-Depth Layers

This fix is ONE layer. Complete protection requires:

1. ✅ Tool-level enforcement (this fix)
2. ✅ File I/O level enforcement (existing)
3. ✅ Canonical path resolution (prevent symlink escape)
4. ✅ Basename validation (prevent parent dir bypass)
5. ✅ Process-level sandboxing (seccomp, pledge, sandbox)

### Related Vulnerabilities

- **Glob Expansion:** `glob("../**/*.txt")` might bypass allowlist
- **Recursive Operations:** `watch("/etc")` starts recursive monitor
- **Archive Extraction:** `unzip("archive.zip", "/tmp")` might extract to `../../../../etc`

### Testing Checklist

When implementing this fix:

- [ ] Test with absolute paths outside sandbox
- [ ] Test with relative paths that resolve outside
- [ ] Test with `.` (current directory)
- [ ] Test with `..` (parent directory)
- [ ] Test with symlinks
- [ ] Test recursive operations stay within bounds
- [ ] Test error messages don't leak path information

## References

- [OWASP: Insecure Direct Object References](https://owasp.org/www-project-top-ten/2017/A5_2017-Broken_Access_Control)
- [CWE-552: Files or Directories Accessible to External Parties](https://cwe.mitre.org/data/definitions/552.html)
- YoctoClaw Security Audit (2026-02-16) - CODEX-AUDIT-REPORT.md Finding #1
- Related Skill: `path-traversal-basename-validation` for complete path validation
