---
name: knowledge-review
description: Review knowledge base entries for structure, format, and quality compliance
argument-hint: "[path] [--fix | --fix-all]"
---

# Knowledge Review Command

Review and validate knowledge base entries with optional auto-fixing capabilities.

## Usage

```bash
/knowledge-review [path] [--fix | --fix-all]
```

### Arguments

- **path** (optional): Target for review
  - Omit to review all entries in `.kb/`
  - Directory path: `.kb/typescript/` or `typescript/`
  - Filename: `functions.md` (searches all of `.kb/`)
  - Full path: `.kb/typescript/naming/functions.md`

- **--fix** (optional): Apply Level 1 auto-fixes only
  - YAML frontmatter corrections
  - Code block language tag detection
  - Heading hierarchy adjustments
  - No user confirmation needed (safe transformations)

- **--fix-all** (optional): Apply Level 1 + Level 2 AI-assisted fixes
  - All Level 1 auto-fixes (automatic)
  - Missing section generation (requires confirmation per file)
  - Additional examples (requires confirmation)

**Default behavior** (no flags): Report-only mode for safety. Identifies issues without modifying files.

## Examples

### Review all entries (report only)
```bash
/knowledge-review
```

### Review specific directory
```bash
/knowledge-review typescript/
```

### Review single file by name
```bash
/knowledge-review functions.md
```

### Review with auto-fixes
```bash
/knowledge-review --fix
```

### Review directory with all fixes
```bash
/knowledge-review typescript/ --fix-all
```

## Implementation

When this command is invoked:

1. **Parse arguments**:
   - Extract path (default: empty for all entries)
   - Extract fix mode:
     - No flags → `mode: "none"`
     - `--fix` → `mode: "fix"`
     - `--fix-all` → `mode: "fix-all"`

2. **Validate inputs**:
   - Check path exists if provided
   - Ensure only one fix flag is provided (--fix and --fix-all are mutually exclusive)

3. **Delegate to knowledge-reviewer agent**:
   ```
   Agent: knowledge-reviewer
   Parameters:
     path: <parsed_path or empty>
     fix_mode: <parsed_mode>
   ```

4. **Display results**:
   - For single file: Show detailed validation report with quality score
   - For batch: Show aggregated summary with files requiring attention

5. **Suggest next steps**:
   - If fixes applied: Remind user to run `/knowledge-reindex` to update catalog
   - If issues remain: Suggest running with `--fix` or `--fix-all`
   - If specific files need attention: Suggest `/knowledge-report <file>` for deep analysis

## Workflow

```
User → /knowledge-review [path] [--fix | --fix-all]
         ↓
  Parse arguments (path + fix_mode)
         ↓
  Validate inputs
         ↓
  Invoke knowledge-reviewer agent
         ↓
  Display formatted results
         ↓
  Suggest next steps
```

## Output Examples

### Single File Review (No Issues)

```
📋 Review: functions.md

File: .kb/typescript/naming/functions.md
Version: 1.0 | Tier: standard

── Validation ──────────────────────────
✓ YAML frontmatter: Valid
✓ Required sections: All present
✓ Code blocks: All tagged
✓ Heading hierarchy: Valid
✓ Cross-references: All valid

Fixes applied: 0

── Quality: 8.9/10 (Excellent ⭐⭐⭐⭐) ──
Clarity: 9 | Format: 9 | Structure: 9 | Completeness: 8 | Efficiency: 9

── Recommendations ─────────────────────
1. Add edge case for async functions → +0.4 pts

Potential: 9.3/10
```

### Batch Review with Fixes

```
📋 Batch Review: .kb/typescript/

Scanned: 17 files | Fixes: 8 across 5 files | Avg Quality: 8.2/10

── Issues ──────────────────────────────
❌ Critical (auto-fixed): 4
  • 2× missing language tags
  • 1× invalid YAML version format (1 → 1.0)
  • 1× heading hierarchy skip (h1→h3)

⚠ Structural (recommendations): 8
  • 3× insufficient examples (<3 for standard tier)
  • 2× missing Forbidden section
  • 2× cross-reference to non-existent file
  • 1× line count exceeds tier limit

── Files Requiring Attention ───────────
  6.5/10  patterns/error-handling.md    (3 issues)
  7.2/10  naming/variables.md           (2 issues)

── Next Steps ──────────────────────────
✓ All critical issues fixed automatically
⚠ Run /knowledge-reindex to update catalog
ℹ Run /knowledge-report <file> for deep analysis
```

## Error Handling

### Invalid Arguments

```bash
/knowledge-review --fix --fix-all
```

Output:
```
❌ Error: Cannot use --fix and --fix-all together
   Use --fix for Level 1 auto-fixes only
   Use --fix-all for Level 1 + Level 2 AI-assisted fixes
```

### File Not Found

```bash
/knowledge-review nonexistent.md
```

Output:
```
❌ Error: File "nonexistent.md" not found in .kb/
   Use /knowledge-review to see all available entries
```

### Invalid Path

```bash
/knowledge-review /outside/kb/file.md
```

Output:
```
❌ Error: Path must be within .kb/ directory
   Valid examples:
     /knowledge-review typescript/
     /knowledge-review functions.md
     /knowledge-review .kb/typescript/naming/functions.md
```

## Notes

- **Safe by default**: Report-only mode prevents accidental modifications
- **Explicit opt-in**: Users must explicitly request fixes with `--fix` or `--fix-all`
- **Idempotent**: Running review multiple times is safe - second run shows "0 fixes"
- **Non-destructive**: Auto-fixes never delete content, only add/correct structure
- **User control**: Level 2 fixes (fix-all mode) always ask for confirmation per file

## Integration

This command integrates with:
- **knowledge-reviewer agent**: Core orchestration and workflow logic
- **knowledge-review skill**: Validation checklists and fix strategies
- **knowledge-evaluate skill**: Quality assessment and recommendations
- **knowledge-draft skill**: AI-assisted content generation for Level 2 fixes
- **PostToolUse hook**: Auto-review trigger after Write operations

## Performance

- **Single file**: ~2-3 seconds (including quality evaluation)
- **Batch (20 files)**: ~40-60 seconds
- **Level 1 only**: <1 second per file
- **Level 3 (quality)**: ~1-2 seconds per file

## Suggested Workflows

### Weekly Quality Check
```bash
# Review all entries, no fixes
/knowledge-review

# Fix critical issues in specific scope
/knowledge-review typescript/ --fix

# Reindex after fixes
/knowledge-reindex
```

### Before Commit
```bash
# Review and fix recent changes
/knowledge-review --fix

# Deep analysis on low-scoring entries
/knowledge-report patterns/error-handling.md

# Update catalog
/knowledge-reindex
```

### New Entry Validation
```bash
# After creating new entry
/knowledge-add scope:python category:naming "Use snake_case for variables"

# Review auto-triggers via hook (Level 1 auto-fix)
# Manual review if needed:
/knowledge-review python/naming/variables.md

# Reindex to update catalog
/knowledge-reindex
```
