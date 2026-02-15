---
name: knowledge-review
description: Multi-level structural and quality compliance checks for knowledge base entries
allowed-tools:
  - Read
  - Grep
  - Glob
context: fork
---

> **EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Knowledge Review Skill

This skill provides comprehensive validation checklists and fix strategies for knowledge base entries, organized into three validation levels.

## Overview

Knowledge entries are validated across three levels with different fix strategies:

- **Level 1 — Structural**: Safe auto-fixes (YAML, headings, code tags)
- **Level 2 — Content Structure**: AI-assisted fixes with confirmation (missing sections, examples)
- **Level 3 — Quality**: Recommendations only (delegated to knowledge-evaluate skill)

## Input Format

Expected input parameters:
```
file_path: <path to .kb/**/*.md file>
fix_mode: "none" | "fix" | "fix-all"
```

## Output Format

Return validation results as structured text:

```
VALIDATION RESULT
mode: <fix_mode>
file: <file_path>

FIXES APPLIED: <count>
- [<type>] <description>
- [<type>] <description>

ISSUES FOUND: <count>
- [level1] [critical|high|medium|low] <description>
- [level2] [medium|low] <description>

STATUS: pass | partial | fail
```

## Level 1 — Structural Compliance (Safe Auto-Fix)

### YAML Frontmatter Checks

| Check | Auto-Fix Strategy | Example |
|-------|-------------------|---------|
| Missing YAML block | Generate from content | Add `---\n...\n---` with title → scope/category detection |
| YAML syntax error | Repair quotes, colons | `scope typescript` → `scope: typescript` |
| Missing field: `type` | Detect from content, fallback to `convention` | Analyze keywords → `type: rule` or `type: convention` |
| Invalid `type` value | Suggest closest match | `type: convent` → `type: convention` |
| Missing field: `version` | Add default | `version: 1.0` |
| Missing field: `scope` | Detect from file path | `.kb/typescript/...` → `scope: typescript` |
| Missing field: `category` | Detect from file path | `.kb/.../naming/...` → `category: naming` |
| Missing field: `tags` | Extract from content | Analyze keywords → `tags: [naming, functions]` |
| Invalid version format | Normalize to X.Y | `version: 1` → `version: 1.0` |
| Tags as string | Convert to array | `tags: naming` → `tags: [naming]` |

**Auto-fix behavior (fix_mode = "fix" or "fix-all")**:
- Apply all fixes automatically using Edit tool
- Log each fix in FIXES APPLIED section
- No user confirmation needed (safe transformations)

### Markdown Structure Checks

| Check | Auto-Fix Strategy | Example |
|-------|-------------------|---------|
| No H1 title | Generate from YAML/filename | Add `# {title}` |
| Multiple H1 titles | Convert 2nd+ to H2 | `# T1\n# T2` → `# T1\n## T2` |
| Heading hierarchy skip | Adjust levels | `# H1\n### H3` → `# H1\n## H2` |
| Code block without tag | Detect language, add tag | ` ```\nconst x` → ` ```typescript\nconst x` |
| Missing `## Format` | Add placeholder section | `## Format\n\n*Define format here.*` |

**Code Language Detection Logic**:

```text
CODE LANGUAGE DETECTION LOGIC (reference only — apply mentally, do NOT execute as code):

Check code block content against these patterns in order (first match wins):

1. TypeScript/JavaScript: matches \b(const|let|interface|type)\b
   - If also matches \b(interface|type )\b → "typescript"
   - Otherwise → "javascript"

2. Python: matches \b(def |import |class |print\()
   → "python"

3. Bash: starts with #! OR matches \b(echo|ls|cd|mkdir)\b
   → "bash"

4. Go: matches \b(func |package |fmt\.)
   → "go"

5. React/JSX: matches (<\w+|useState|useEffect)
   - If code contains "interface" → "tsx"
   - Otherwise → "jsx"

6. Fallback: use the file's scope as the language tag
```

### Path Consistency Checks

| Check | Fix Strategy | Example |
|-------|--------------|---------|
| Filename mismatch | Report only (suggest rename) | `functions.md` but title "Classes" → warn |
| Path scope mismatch | Report only (suggest move) | `.kb/python/...` but `scope: typescript` → warn |
| Path category mismatch | Report only (suggest move) | `.kb/.../rules/...` but `category: naming` → warn |

**Report-only behavior**: Add to ISSUES FOUND section with severity `[level1] [low]`

## Level 2 — Content Structure (AI-Assisted Fix)

### Tier Detection

Automatically detect entry tier based on content metrics:

```text
TIER DETECTION LOGIC (reference only — apply mentally, do NOT execute as code):

Determine tier from content metrics (check in order, first match wins):

1. SIMPLE tier: rules_count <= 3 AND code_blocks_count <= 2
   Thresholds: min_lines=50, max_lines=100, min_examples=1

2. COMPREHENSIVE tier: rules_count >= 7 AND code_blocks_count >= 12 AND sections >= 8
   Thresholds: min_lines=400, max_lines=650, min_examples=5

3. STANDARD tier (default): everything else
   Thresholds: min_lines=150, max_lines=300, min_examples=3
```

### Tier Compliance Checks

| Check | AI-Assisted Fix Strategy (fix-all mode) | Confirmation Required |
|-------|------------------------------------------|----------------------|
| Missing Allowed section | Generate examples from Rules via knowledge-draft | Yes (per-file) |
| Missing Forbidden section | Generate anti-patterns from Allowed via knowledge-draft | Yes (per-file) |
| Missing Rules section | Extract rules from Description via knowledge-draft | Yes (per-file) |
| Missing Summary table | Generate table from sections via knowledge-draft | Yes (per-file) |
| Insufficient examples | Generate additional examples via knowledge-draft | Yes (per-file) |
| Line count outside tier range | Report only (may indicate wrong tier) | No |

**Fix-all behavior (fix_mode = "fix-all")**:
- For each Level 2 issue, use AskUserQuestion to request confirmation
- If approved, invoke knowledge-draft skill to regenerate missing content
- If denied, add to ISSUES FOUND section

**Report-only behavior (fix_mode = "none" or "fix")**:
- Add all Level 2 issues to ISSUES FOUND with severity `[level2] [medium]` or `[level2] [low]`

### Cross-Reference Validation

| Check | Fix Strategy | Example |
|-------|--------------|---------|
| Link to non-existent file | Report only (warn broken link) | `[Pattern](../patterns/logging.md)` → check file exists |
| Link to README | Report only (suggest direct entry) | Link to category README → suggest specific entry |
| External links | Report only (verify accessibility) | Check HTTP links return 200 |

**Report-only behavior**: Add to ISSUES FOUND with severity `[level2] [low]`

## Level 3 — Quality Assessment (Recommendations Only)

Delegate to existing `knowledge-evaluate` skill in **fast mode**:

Use the **Task** tool to delegate to the `knowledge-evaluate` skill in fast mode, passing the resolved file path and `mode: "fast"` (quick structural analysis, under 2 seconds).

**Quality criteria evaluated**:
- **Clarity**: Must vs should consistency, vague language detection
- **Completeness**: Edge cases, example diversity
- **Efficiency**: Information density, redundancy analysis
- **Format**: Visual structure quality
- **Structure**: Logical flow, concept separation

**Output parsing**: Extract scores from text-based evaluation result:
Parse the evaluation result text mentally: look for the pattern `Quality: {score}/10 (Clarity: {n}, Format: {n}, ...)` and extract the overall score and individual criterion scores.

**No auto-fix**: Quality assessment produces recommendations only, never modifies files.

## Workflow Instructions

### Single File Review

1. **Read file** using Read tool
2. **Parse YAML** frontmatter to extract metadata
3. **Run Level 1 checks**:
   - Validate YAML structure and required fields
   - Check markdown heading hierarchy
   - Find code blocks without language tags
   - Collect fixes and errors
4. **Apply Level 1 fixes** if `fix_mode` is "fix" or "fix-all":
   - Use Edit tool to apply safe transformations
   - Log each fix in output
5. **Run Level 2 checks**:
   - Detect tier from content metrics
   - Check tier compliance (sections, examples, line count)
   - Validate cross-references
6. **Apply Level 2 fixes** if `fix_mode` is "fix-all":
   - For each issue, ask user for confirmation
   - If approved, invoke knowledge-draft skill
7. **Run Level 3 evaluation** (if no critical errors):
   - Invoke knowledge-evaluate skill in fast mode
   - Parse quality scores from output
8. **Format output** according to VALIDATION RESULT template
9. **Return result** to calling agent

### Batch File Review

For batch processing (handled by knowledge-reviewer agent):

1. **Glob files** using pattern `.kb/**/*.md` (exclude README.md, TEMPLATE.md)
2. **For each file**:
   - Run single file review workflow
   - On failure: log error, continue to next (continue-on-failure)
   - Collect: `{path, level1_errors[], level2_warnings[], level3_score}`
3. **Aggregate statistics**:
   - Total files processed
   - Fixes applied by type (yaml, code_block, heading, section)
   - Average quality score
   - Issue distribution (critical vs warnings)
4. **Sort files** by quality ascending (worst first)
5. **Format summary** report
6. **Return aggregated results**

## Error Handling

- **File not found**: Return error status with message
- **Invalid YAML**: Attempt repair, if fails report in ISSUES FOUND
- **Read permission denied**: Return error status with message
- **Skill invocation failure**: Log error, continue with partial results
- **Continue-on-failure**: In batch mode, always continue processing remaining files

## Performance Targets

- Level 1 validation: <500ms per file
- Level 2 validation: <1s per file (without fixes)
- Level 3 evaluation (fast mode): ~1-2s per file
- Total single file review: ~2-3s
- Batch processing 20 files: ~40-60s

## Idempotency

Reviews are idempotent - running multiple times on the same file should show:
- First run: Apply fixes, report issues
- Subsequent runs: "0 fixes applied" if already corrected
- Quality score should remain stable unless content changes

## Examples

### Example 1: Valid Entry (No Issues)

Input:
```
file_path: .kb/typescript/naming/functions.md
fix_mode: fix
```

Output:
```
VALIDATION RESULT
mode: fix
file: .kb/typescript/naming/functions.md

FIXES APPLIED: 0

ISSUES FOUND: 0

STATUS: pass
```

### Example 2: Auto-Fixed Issues

Input:
```
file_path: .kb/python/patterns/error-handling.md
fix_mode: fix
```

Output:
```
VALIDATION RESULT
mode: fix
file: .kb/python/patterns/error-handling.md

FIXES APPLIED: 3
- [yaml] version: "1" → "1.0"
- [code_block] line 45: added language tag "python"
- [code_block] line 89: added language tag "python"

ISSUES FOUND: 2
- [level2] [medium] Missing Forbidden section (recommended for standard tier)
- [level2] [low] Only 2 examples in Allowed (target: 3+ for standard tier)

STATUS: partial
```

### Example 3: Report-Only Mode

Input:
```
file_path: .kb/react/hooks/useState.md
fix_mode: none
```

Output:
```
VALIDATION RESULT
mode: none
file: .kb/react/hooks/useState.md

FIXES APPLIED: 0

ISSUES FOUND: 4
- [level1] [high] Missing field: version
- [level1] [medium] Code block at line 67 missing language tag
- [level2] [medium] Missing Forbidden section
- [level2] [low] Cross-reference "../patterns/hooks.md" — file not found

STATUS: fail
```
