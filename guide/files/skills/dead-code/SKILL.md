---
name: dead-code
description: >
  Detect and safely remove unused code — imports, functions, classes, exports, and dependencies.
  Framework-aware: won't delete React hooks, Django models, Flask routes, Spring beans, or
  dynamically referenced code. Use when the user says "/dead-code", "clean up unused code",
  "find dead code", or "remove unused imports". Also runs as a check in /tech-debt.
---

# Dead Code — Framework-Aware Unused Code Removal

**Purpose:** Find and safely remove dead code. Every removal is validated — syntax check + test run after each batch. When in doubt, preserve and flag for manual review.

---

## Invocation

- **Manual:** `/dead-code` — full scan of the project
- **Scoped:** `/dead-code src/services/` — scan a specific directory
- **Dry run:** `/dead-code --dry` — report findings without removing anything
- **Imports only:** `/dead-code imports` — just unused imports

---

## Step 0: Map the Project

1. **Detect languages and frameworks** — read package.json, requirements.txt, Cargo.toml, etc.
2. **Identify entry points** — these are NEVER dead code:
   - `main.ts`, `index.ts`, `app.ts`, `server.ts` (and `.js`/`.py`/`.go` equivalents)
   - `__main__.py`, `manage.py`, `wsgi.py`, `asgi.py`
   - Test files (`*.test.*`, `*.spec.*`, `test_*`)
   - Config files, migration files, seed files
   - Files referenced in `package.json` scripts, Dockerfile, CI config
3. **Build the dependency graph** — what imports what

---

## Detection Rules

### Unused Imports

Scan every source file:
- Is the imported symbol used anywhere in the file?
- Watch for: re-exports (`export { X } from './y'`), type-only imports, side-effect imports

**Safe to remove:** Import present, symbol never referenced in file body.
**Never remove:**
- Side-effect imports (`import './polyfill'`, `import 'dotenv/config'`)
- CSS/style imports (`import './styles.css'`)
- Type-only imports used in JSDoc or declaration files

### Unused Functions & Classes

- Is the function/class called, referenced, or exported anywhere in the project?
- Check: direct calls, callback references, class inheritance, decorator usage

**Never remove if:**
- Exported from an entry point (could be used externally)
- Referenced via dynamic patterns (see Dynamic Usage Safety below)
- Part of a public API or SDK

### Unused Exports

- Is the export imported anywhere else in the project?
- `grep -r "from.*moduleName" --include='*.ts' --include='*.js'`

**Never remove if:**
- The module is an entry point or public API
- The export is referenced in config files, scripts, or test fixtures

### Unused Dependencies

```bash
# Node.js
npx depcheck 2>/dev/null

# Python
# Check each import against installed packages
```

**Never remove if:**
- Used as a CLI tool (in `package.json` scripts)
- A peer dependency required by another package
- A type-only dependency (`@types/*`)
- Referenced in config files (babel plugins, eslint plugins, etc.)

---

## Dynamic Usage Safety — NEVER Remove If:

These patterns mean the code might be referenced at runtime in ways static analysis can't detect:

**JavaScript/TypeScript:**
- `window[varName]`, `this[varName]`, `obj[computedKey]`
- `eval()`, `new Function()`
- Dynamic `import()` with variable paths
- `Reflect.get()`, `Proxy` handlers
- String references in template engines, route configs, DI containers

**Python:**
- `getattr()`, `setattr()`, `__getattr__`
- `globals()`, `locals()`, `vars()`
- `importlib.import_module()`, `__import__()`
- Metaclasses, decorators that register classes
- String references in Django URL configs, Celery tasks

**Java:**
- Reflection (`Class.forName()`, `Method.invoke()`)
- Annotations (`@Component`, `@Service`, `@Bean`, `@Entity`)
- Spring dependency injection, JPA entities

---

## Framework Preservation Rules

### React / Next.js
- Keep: Components (even if not imported — could be lazy-loaded or routed)
- Keep: Hooks (`use*` functions) — may be used conditionally
- Keep: Context providers, HOCs
- Keep: `getServerSideProps`, `getStaticProps`, page components in `pages/` or `app/`

### Django
- Keep: Models, migrations, admin registrations, management commands
- Keep: Anything in `urls.py`, `views.py` referenced by URL patterns
- Keep: Signal handlers, middleware classes

### Flask / FastAPI
- Keep: Route handlers (decorated with `@app.route`, `@router.get`, etc.)
- Keep: Blueprints, app factories, middleware
- Keep: CLI commands (`@app.cli.command`)

### Express / Koa / Hono
- Keep: Route handlers registered with `app.get()`, `app.post()`, `router.use()`, etc.
- Keep: Middleware functions passed to `app.use()`
- Keep: Error handlers (4-argument functions)

### Vue / Angular / Svelte
- Keep: Components registered in templates or routing config
- Keep: Directives, pipes, services registered in modules
- Keep: Lifecycle hooks, composables

---

## Execution Process

### 1. Scan (read-only)

Build the full inventory of candidates:
```
DEAD CODE CANDIDATES:

Unused imports (23):
  src/auth.ts:3     — import { hash } from 'bcrypt'  (never used)
  src/utils.ts:1    — import lodash from 'lodash'     (never used)
  [...]

Unused functions (7):
  src/helpers.ts:45  — function formatLegacyDate()    (0 references in project)
  src/api.ts:120     — function retryWithBackoff()     (0 references in project)
  [...]

Unused exports (4):
  src/types.ts:23    — export type OldUserSchema       (0 imports in project)
  [...]

Unused dependencies (3):
  moment             — 0 imports found
  chalk              — 0 imports found (but used in package.json scripts ← PRESERVED)
  [...]
```

### 2. Classify

For each candidate:
- **SAFE** — no dynamic usage patterns, no framework preservation rules, clear evidence of disuse
- **UNCERTAIN** — might be dynamically used, or framework patterns detected → flag for manual review
- **PRESERVED** — framework rule applies, entry point, or dynamic usage detected → do not remove

### 3. Remove (if not --dry)

For **SAFE** candidates only:
1. Remove in batches grouped by file
2. After each file: run syntax check (`node -c`, `python -m py_compile`, `tsc --noEmit`)
3. After all removals: run the project's test suite
4. If tests fail: revert the last batch, flag those items as UNCERTAIN

### 4. Report

```
DEAD CODE REMOVAL SUMMARY:

Removed:
  - 23 unused imports across 14 files
  - 5 unused functions (187 lines)
  - 2 unused exports
  - 1 unused dependency (moment)

Preserved (framework/dynamic):
  - 3 React hooks (may be conditionally used)
  - 1 Express middleware (registered via string reference)

Flagged for manual review:
  - src/legacy.ts:formatOldData() — 0 references but in a utils barrel export
  - src/types.ts:InternalConfig — 0 imports but referenced in JSDoc comments

Lines removed: 234
Validation: tests pass ✓ | syntax clean ✓
```

---

## What Dead Code Does NOT Do

- Does not remove commented-out code (that's a style choice)
- Does not remove unused CSS (different tooling needed)
- Does not remove unused database columns or tables
- Does not remove code that is "unused but planned" — if you're not sure, flag don't delete
- Does not run without test validation — if no tests exist, it operates in dry-run mode only and warns
