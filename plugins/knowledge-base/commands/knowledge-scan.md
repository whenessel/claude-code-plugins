---
name: knowledge-scan
description: Scan codebase to discover and generate knowledge entries from existing code patterns
argument-hint: "[path] [--auto] [--scope SCOPE] [--min-confidence N] | generate <ids>"
---

> **EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Knowledge Scan Command

Analyze project codebase to discover de-facto conventions and generate knowledge entries.

## Usage

```bash
# Phase 1: Discovery (default)
/knowledge-scan
/knowledge-scan src/
/knowledge-scan --scope typescript
/knowledge-scan src/backend/ --scope python --min-confidence 70

# Phase 2: Generation (after discovery)
/knowledge-scan generate 1,2,5
/knowledge-scan generate all
/knowledge-scan generate high

# Full auto mode
/knowledge-scan --auto
/knowledge-scan src/ --auto --min-confidence 80
```

## Implementation

### Parse Arguments

```text
ARGUMENT PARSING LOGIC (reference only — apply mentally, do NOT execute as code):

Default values:
  mode: "discover"
  path: null
  auto: false
  scope: null
  min_confidence: 60
  generate_ids: null

GENERATE SUBCOMMAND:
  If first argument is "generate":
    mode = "generate"
    Second argument determines generate_ids:
      "all"   → generate_ids = "all"
      "high"  → generate_ids = "high" (high-confidence only)
      "1,2,5" → parse as comma-separated integers; error if non-numeric
    If no second argument → error: "Missing IDs. Usage: /knowledge-scan generate 1,2,5 | all | high"

DISCOVERY FLAGS (iterate through args):
  --auto                → auto = true
  --scope <value>       → scope = next argument
  --min-confidence <N>  → min_confidence = parse as integer; error if non-numeric
  (no -- prefix)        → path = argument (positional)
  (unknown flag)        → error: "Unknown flag: {arg}"
```

### Route by Mode

#### Discovery Mode (default)

If mode is `"discover"`, use the **Task** tool to delegate to the `knowledge-scanner` agent with description "Discover codebase conventions". Include in the prompt:
- Scan path (parsed path, or "project root" if none)
- Scope filter (parsed scope, or "auto-detect all" if none)
- Minimum confidence percentage
- Auto mode flag

#### Generate Mode (after discovery)

If mode is `"generate"`, use the **Task** tool to delegate to the `knowledge-scanner` agent with description "Generate knowledge entries from scan plan". Include the target IDs in the prompt. The agent verifies that a discovery plan exists in session via project memory.

### Error Handling

**No arguments (default discovery):**
```
Starts full project scan with default settings.
Equivalent to: /knowledge-scan --min-confidence 60
```

**Generate without prior discovery:**
```
❌ No scan plan found in current session.

Run discovery first:
  /knowledge-scan

Then generate from results:
  /knowledge-scan generate 1,2,5
```

**Invalid path:**
```
❌ Path not found: src/nonexistent/

Available directories:
  src/
  backend/
  frontend/
```

**No conventions discovered:**
```
Codebase scan complete.

No conventions discovered with confidence ≥ {min_confidence}%.

Suggestions:
- Lower threshold: /knowledge-scan --min-confidence 40
- Scan specific area: /knowledge-scan src/components/
- Add conventions manually: /knowledge-add <guidelines>
```

### Display Results

#### Discovery Phase Output

```
🔍 Codebase Scan Complete

Project stack: TypeScript, React, Python, FastAPI
Files analyzed: 342 (187 .ts/.tsx, 98 .py, 57 other)
Linter configs: .eslintrc.json, tsconfig.json, .prettierrc

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Discovered conventions (12):

  High confidence (≥80%):
  ✓ [1]  typescript/naming/functions — camelCase, verb-prefix (94%, 156 files)
  ✓ [2]  typescript/naming/components — PascalCase (91%, 78 files)
  ✓ [3]  typescript/structure/imports — grouped by type (87%, 145 files)
  ✓ [4]  python/naming/functions — snake_case (96%, 89 files)
  ✓ [5]  python/naming/modules — lowercase, underscores (88%, 42 files)

  Medium confidence (60-79%):
  ? [6]  typescript/patterns/error-handling — try-catch in async (72%, 45 files)
  ? [7]  react/patterns/hooks — custom hooks extraction (68%, 32 files)
  ? [8]  python/structure/modules — __init__.py exports (65%, 28 files)
  ? [9]  typescript/docs/jsdoc — function documentation (62%, 67 files)

  Low confidence (<60%):
  ○ [10] react/patterns/state — zustand stores (51%, 41 files)
  ○ [11] typescript/naming/files — kebab-case (48%, 89 files)
  ○ [12] python/patterns/error-handling — custom exceptions (43%, 18 files)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Existing knowledge entries detected:
  ⚡ [1] conflicts with .kb/typescript/naming/functions.md (v1.2)
  ⚡ [6] conflicts with .kb/typescript/patterns/error-handling.md (v1.0)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next steps:
  /knowledge-scan generate all          — Generate all ≥60% confidence
  /knowledge-scan generate high         — Generate only ≥80% confidence
  /knowledge-scan generate 1,3,6,7      — Generate specific entries
```

#### Generation Phase Output

```
⚙️  Generating knowledge entries...

  [1/4] typescript/naming/functions
        ⚡ Conflict with existing .kb/typescript/naming/functions.md (v1.2)
        Comparison:
          Existing: camelCase, verb-based, max 3 words
          Discovered: camelCase, verb-prefix, descriptive (94%, 156 files)
          Diff: +verb-prefix pattern, +async naming rules, ~similar core rules
        → Update existing entry? [y/n/skip]

  [2/4] typescript/structure/imports ✓
        Created: .kb/typescript/structure/imports.md (v1.0)
        Quality: 8.2/10

  [3/4] typescript/patterns/error-handling
        ⚡ Conflict with existing .kb/typescript/patterns/error-handling.md (v1.0)
        Comparison:
          Existing: basic try-catch patterns (v1.0)
          Discovered: try-catch + custom errors + logging (72%, 45 files)
          Diff: +custom error classes, +structured logging, +error boundaries
        → Update existing entry? [y/n/skip]

  [4/4] react/patterns/hooks ✓
        Created: .kb/react/patterns/hooks.md (v1.0)
        Quality: 7.8/10

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Summary:
  Created: 2 new entries
  Updated: 1 entry (v1.2 → v1.3)
  Skipped: 1 (user choice)

Run /knowledge-reindex to update catalog.
```

#### Auto Mode Output

```
🤖 Auto-scan mode (confidence ≥ 60%)

[Discovery phase...]
Found 9 conventions above threshold.

[Generation phase...]
  ✓ [1] .kb/typescript/naming/functions.md (v1.0) — 8.4/10
  ✓ [2] .kb/typescript/naming/components.md (v1.0) — 8.1/10
  ⚡ [3] .kb/typescript/structure/imports.md — CONFLICT, skipped (use interactive mode)
  ✓ [4] .kb/python/naming/functions.md (v1.0) — 8.6/10
  ...

Summary:
  Created: 7 new entries
  Skipped: 2 (conflicts — resolve with interactive /knowledge-scan)

Run /knowledge-reindex to update catalog.
```

## Notes

- **Session-scoped plan**: Discovery results are NOT persisted to disk. Re-run `/knowledge-scan` for fresh analysis.
- **Auto mode skips conflicts**: In `--auto` mode, conflicting entries are skipped (not overwritten). Use interactive mode to resolve.
- **Delegated generation**: Each entry is generated via `knowledge-writer` agent with enriched context from discovery.
- **Incremental friendly**: Safe to run multiple times. Existing entries are detected and handled.
- **Path filter**: Limits file scanning, not output paths. Output always goes to `.kb/{scope}/{category}/`.

## Examples

**Example 1: Full project discovery**
```bash
/knowledge-scan
# → Scans entire project, shows plan
/knowledge-scan generate high
# → Generates entries for ≥80% confidence items
```

**Example 2: Focused scan**
```bash
/knowledge-scan src/api/ --scope python
# → Scans only Python files in src/api/
/knowledge-scan generate 1,2,3
```

**Example 3: Quick auto-generate**
```bash
/knowledge-scan --auto --min-confidence 80
# → Discovers and generates all high-confidence entries
```

**Example 4: Conservative scan**
```bash
/knowledge-scan --min-confidence 85
# → Only shows very consistent patterns
/knowledge-scan generate all
```
