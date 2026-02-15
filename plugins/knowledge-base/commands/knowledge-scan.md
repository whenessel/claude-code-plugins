---
name: knowledge-scan
description: Scan codebase to discover and generate knowledge entries from existing code patterns
argument-hint: "[path] [--auto] [--scope SCOPE] [--min-confidence N] | generate <ids>"
---

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

```python
def parse_arguments(args: list[str]) -> dict:
    """Parse command arguments."""

    result = {
        "mode": "discover",      # discover | generate
        "path": None,            # optional path filter
        "auto": False,           # --auto flag
        "scope": None,           # --scope filter
        "min_confidence": 60,    # --min-confidence threshold
        "generate_ids": None     # ids for generate mode
    }

    # Check for generate subcommand
    if args and args[0] == "generate":
        result["mode"] = "generate"
        if len(args) > 1:
            target = args[1].strip()
            if target == "all":
                result["generate_ids"] = "all"
            elif target == "high":
                result["generate_ids"] = "high"  # high-confidence only
            else:
                # Parse comma-separated IDs: "1,2,5"
                try:
                    result["generate_ids"] = [
                        int(x.strip()) for x in target.split(",")
                    ]
                except ValueError:
                    return {"error": f"Invalid IDs: {target}. Use numbers like: 1,2,5"}
        else:
            return {"error": "Missing IDs. Usage: /knowledge-scan generate 1,2,5 | all | high"}
        return result

    # Parse flags and positional args
    i = 0
    while i < len(args):
        arg = args[i]

        if arg == "--auto":
            result["auto"] = True

        elif arg == "--scope" and i + 1 < len(args):
            i += 1
            result["scope"] = args[i]

        elif arg == "--min-confidence" and i + 1 < len(args):
            i += 1
            try:
                result["min_confidence"] = int(args[i])
            except ValueError:
                return {"error": f"Invalid confidence value: {args[i]}"}

        elif not arg.startswith("--"):
            # Positional argument = path
            result["path"] = arg

        else:
            return {"error": f"Unknown flag: {arg}"}

        i += 1

    return result
```

### Route by Mode

#### Discovery Mode (default)

```python
if parsed["mode"] == "discover":
    prompt = f"""
    Scan codebase to discover conventions and patterns.

    {"Scan path: " + parsed["path"] if parsed["path"] else "Scan path: project root"}
    {"Scope filter: " + parsed["scope"] if parsed["scope"] else "Scope filter: auto-detect all"}
    Minimum confidence: {parsed["min_confidence"]}%
    Auto mode: {parsed["auto"]}
    """

    Task(
        description="Discover codebase conventions",
        prompt=prompt,
        subagent_type="knowledge-scanner"
    )
```

#### Generate Mode (after discovery)

```python
if parsed["mode"] == "generate":
    # Verify discovery plan exists in session
    # (agent checks this via project memory)

    prompt = f"""
    Generate knowledge entries from discovery plan.

    Target: {parsed["generate_ids"]}
    """

    Task(
        description="Generate knowledge entries from scan plan",
        prompt=prompt,
        subagent_type="knowledge-scanner"
    )
```

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
