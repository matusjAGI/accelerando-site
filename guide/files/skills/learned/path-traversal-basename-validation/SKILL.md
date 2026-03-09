---
name: path-traversal-basename-validation
description: |
  Prevent path traversal attacks by validating reconstructed paths, not just parent directories.
  Use when: (1) implementing path allowlist/sandbox, (2) validating file paths for write operations,
  (3) paths don't exist yet (e.g., file creation). Addresses attack vector: "allowed_dir/../../../etc/passwd"
  where parent directory is valid but final path escapes allowed root. Critical for: filesystem sandboxes,
  file upload validation, path allowlists in any language (Zig, Rust, Go, C++, Python, Node.js).
author: Claude Code
version: 1.0.0
date: 2026-02-16
---

# Path Traversal Basename Validation Gap

## Problem

Path validation functions often check if a parent directory is allowed, but fail to validate the
reconstructed full path (parent + basename). This creates a subtle path traversal vulnerability.

**Common Vulnerable Pattern:**
```zig
fn isPathAllowed(path: []const u8) bool {
    const canonical = realpathAlloc(path) catch {
        // Path doesn't exist - check parent only
        if (dirname(path)) |parent| {
            const parent_canon = realpathAlloc(parent);
            return isUnderAllowedRoot(parent_canon);  // BUG: Only checks parent!
        }
    };
    return isUnderAllowedRoot(canonical);
}
```

## Context / Trigger Conditions

**When This Vulnerability Exists:**
- Path validation for non-existent files (write operations, file creation)
- Canonical path resolution fails, falls back to parent directory check
- Allowlist enforcement based on string prefix matching
- Sandbox implementations, file upload handlers, path-based access control

**Attack Vector:**
```
Allowed directory: /home/user/workspace/allowed_dir
Attack path: "allowed_dir/../../../etc/passwd"

1. Path doesn't exist → realpathAlloc(path) fails
2. Code checks dirname("allowed_dir/../../../etc")
3. If "allowed_dir" exists, parent resolves to some real directory
4. Parent passes allowlist check
5. BUT: Final path would be /etc/passwd (outside allowed root)
```

**Symptoms:**
- Path allowlist bypassed with crafted relative paths
- Sandbox escape via file write operations
- Arbitrary file read/write outside intended directory

## Solution

**Defense-in-Depth Approach:**

### Step 1: Validate Parent Directory
```zig
const dirname = std.fs.path.dirname(path) orelse ".";
const parent_canon = realpathAlloc(dirname) catch return false;
defer free(parent_canon);

if (!isUnderAllowedRoot(parent_canon)) return false;
```

### Step 2: Reconstruct Full Path
```zig
const basename = std.fs.path.basename(path);
const reconstructed = path.join(&.{ parent_canon, basename }) catch return false;
defer free(reconstructed);
```

### Step 3: Validate Reconstructed Path
```zig
// CRITICAL: Verify final path is under allowed root
return isUnderAllowedRoot(reconstructed);
```

**Complete Secure Implementation:**

```zig
fn isPathAllowed(path: []const u8) bool {
    // Layer 1: Quick reject on obvious traversal
    if (indexOf(path, "..") != null) return false;

    // Layer 2: Try canonical resolution (if path exists)
    const canonical = realpathAlloc(path) catch {
        // Layer 3: Path doesn't exist - validate parent + basename
        const dirname = std.fs.path.dirname(path) orelse ".";
        const basename = std.fs.path.basename(path);

        // Resolve parent directory
        const parent_canon = realpathAlloc(dirname) catch return false;
        defer free(parent_canon);

        // Check parent is allowed
        if (!isUnderAllowedRoot(parent_canon)) return false;

        // CRITICAL: Reconstruct and validate full path
        const reconstructed = path.join(&.{ parent_canon, basename }) catch return false;
        defer free(reconstructed);

        return isUnderAllowedRoot(reconstructed);
    };
    defer free(canonical);

    return isUnderAllowedRoot(canonical);
}
```

## Verification

**Test Cases:**

1. **Normal file in allowed directory:**
   ```
   Input: "allowed_dir/file.txt"
   Expected: true (if under allowed root)
   ```

2. **Path traversal attempt:**
   ```
   Input: "allowed_dir/../../../etc/passwd"
   Expected: false (reconstructed path escapes)
   ```

3. **Symlink to outside directory:**
   ```
   Input: "allowed_dir/symlink_to_etc"
   Expected: false (canonical resolution detects escape)
   ```

4. **Non-existent file in allowed directory:**
   ```
   Input: "allowed_dir/new_file.txt"
   Expected: true (parent + basename validates correctly)
   ```

**Automated Test:**
```bash
# Create test structure
mkdir -p /tmp/allowed_dir
touch /tmp/allowed_dir/legit.txt

# Test cases
test_path "/tmp/allowed_dir/legit.txt"                  # Should ALLOW
test_path "/tmp/allowed_dir/new_file.txt"               # Should ALLOW
test_path "/tmp/allowed_dir/../../../etc/passwd"        # Should BLOCK
test_path "/tmp/allowed_dir/../outside.txt"             # Should BLOCK
```

## Language-Specific Implementations

### Rust
```rust
use std::path::{Path, PathBuf};
use std::fs;

fn is_path_allowed(path: &Path, allowed_root: &Path) -> Result<bool, std::io::Error> {
    // Try to canonicalize the full path
    match fs::canonicalize(path) {
        Ok(canonical) => Ok(canonical.starts_with(allowed_root)),
        Err(_) => {
            // Path doesn't exist - check parent + basename
            let parent = path.parent().unwrap_or(Path::new("."));
            let basename = path.file_name().ok_or_else(||
                std::io::Error::new(std::io::ErrorKind::InvalidInput, "Invalid path"))?;

            let parent_canonical = fs::canonicalize(parent)?;
            if !parent_canonical.starts_with(allowed_root) {
                return Ok(false);
            }

            // Reconstruct and validate
            let reconstructed = parent_canonical.join(basename);
            Ok(reconstructed.starts_with(allowed_root))
        }
    }
}
```

### Python
```python
from pathlib import Path

def is_path_allowed(path: str, allowed_root: str) -> bool:
    try:
        # Try to resolve the path
        canonical = Path(path).resolve(strict=True)
        return canonical.is_relative_to(allowed_root)
    except (FileNotFoundError, RuntimeError):
        # Path doesn't exist - check parent + basename
        path_obj = Path(path)
        parent = path_obj.parent or Path(".")
        basename = path_obj.name

        try:
            parent_canonical = parent.resolve(strict=True)
        except (FileNotFoundError, RuntimeError):
            return False

        if not parent_canonical.is_relative_to(allowed_root):
            return False

        # Reconstruct and validate
        reconstructed = parent_canonical / basename
        return reconstructed.is_relative_to(allowed_root)
```

### Go
```go
import (
    "path/filepath"
    "strings"
)

func isPathAllowed(path, allowedRoot string) bool {
    // Quick reject
    if strings.Contains(path, "..") {
        return false
    }

    // Try to resolve
    canonical, err := filepath.EvalSymlinks(path)
    if err == nil {
        return strings.HasPrefix(canonical, allowedRoot)
    }

    // Path doesn't exist - check parent + basename
    dir := filepath.Dir(path)
    base := filepath.Base(path)

    parentCanon, err := filepath.EvalSymlinks(dir)
    if err != nil {
        return false
    }

    if !strings.HasPrefix(parentCanon, allowedRoot) {
        return false
    }

    // Reconstruct and validate
    reconstructed := filepath.Join(parentCanon, base)
    return strings.HasPrefix(reconstructed, allowedRoot)
}
```

## Example: Real-World Exploit

**YoctoClaw Vulnerability (CVE-2026-XXXXX - Hypothetical):**

```zig
// BEFORE FIX (Vulnerable)
fn isPathAllowed(path: []const u8) bool {
    const canonical = std.fs.cwd().realpathAlloc(allocator, path) catch {
        if (std.fs.path.dirname(path)) |parent| {
            const parent_canon = std.fs.cwd().realpathAlloc(allocator, parent) catch return false;
            defer allocator.free(parent_canon);
            return isCanonicalPathAllowed(parent_canon);  // BUG HERE
        }
        return false;
    };
    defer allocator.free(canonical);
    return isCanonicalPathAllowed(canonical);
}

// EXPLOIT
write_file("workspace/allowed/../../../home/victim/.ssh/id_rsa", malicious_key)
→ dirname("workspace/allowed/../../../home/victim/.ssh") resolves
→ If "workspace/allowed" exists, parent check passes
→ File written to /home/victim/.ssh/id_rsa (SSH key compromise)
```

**After Fix:**
```zig
// Now validates: parent_canon + basename = /home/victim/.ssh/id_rsa
// This path is NOT under allowed root → BLOCKED
```

## Notes

### Edge Cases

1. **Empty Basename:** Paths ending in `/` have empty basename
   - Solution: Treat as directory, validate parent only

2. **Root Directory:** Paths like `/` have no parent
   - Solution: Special case check, usually disallow

3. **Current Directory:** Paths like `.` or `./file.txt`
   - Solution: Resolve `.` to cwd before validation

4. **Windows Paths:** Drive letters and UNC paths
   - Solution: Use platform-specific canonicalization

### Performance Considerations

- Canonical path resolution involves filesystem operations (slow)
- Cache allowed root canonical path (resolve once)
- Consider allowlist of canonical paths (hash set lookup)

### Defense-in-Depth Layers

This fix is ONE layer. Complete protection requires:

1. ✅ String-based quick reject (`..` check)
2. ✅ Canonical path resolution (symlink, `..` resolution)
3. ✅ Parent directory validation
4. ✅ **Reconstructed path validation (this fix)**
5. ✅ Tool-level enforcement (see related skill: tool-level-path-allowlist)

### Common Mistakes

❌ **Only checking string prefix:**
```python
if path.startswith(allowed_root):  # WRONG
    return True
```

❌ **Only checking parent directory:**
```python
if Path(path).parent.resolve().is_relative_to(allowed_root):  # INCOMPLETE
    return True
```

✅ **Correct: Check reconstructed path:**
```python
parent_canon = Path(path).parent.resolve()
reconstructed = parent_canon / Path(path).name
return reconstructed.is_relative_to(allowed_root)
```

## References

- [CWE-22: Improper Limitation of a Pathname to a Restricted Directory](https://cwe.mitre.org/data/definitions/22.html)
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [POSIX realpath(3)](https://man7.org/linux/man-pages/man3/realpath.3.html)
- YoctoClaw Security Audit (2026-02-16) - Internal finding
