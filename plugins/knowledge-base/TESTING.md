# Knowledge Base Plugin — Testing Guide

Comprehensive test cases for the knowledge-base plugin v1.4.0.

## Test Environment

- **Plugin Location**: `plugins/knowledge-base/` (or installed via `.claude/plugins/knowledge-base/`)
- **Knowledge Base Directory**: `.kb/`
- **Claude Code Version**: Latest
- **Plugin Version**: 1.4.0

## How to Run

Launch Claude Code with the plugin:

```bash
claude --plugin-dir ./plugins/knowledge-base
```

Or copy to `.claude/plugins/` for testing in a specific project context.

## Pre-Test Checklist

- [ ] Plugin installed and accessible
- [ ] All plugin files present:
  - [ ] `.claude-plugin/plugin.json` (version 1.4.0)
  - [ ] 4 commands: `knowledge-add.md`, `knowledge-reindex.md`, `knowledge-report.md`, `knowledge-scan.md`
  - [ ] 3 agents: `knowledge-writer.md`, `knowledge-evaluator.md`, `knowledge-scanner.md`
  - [ ] 4 skills: `knowledge-draft/`, `knowledge-evaluate/`, `codebase-discover/`, `knowledge-review/`
  - [ ] `hooks/hooks.json`
- [ ] Claude Code CLI is running

---

## Critical: Python Script Prevention Tests

> These tests verify the core fix in v1.4.0 — the plugin must **NEVER** create or execute Python scripts.

### Test P1: No Python Scripts During Entry Creation

**Command**:
```bash
/knowledge-add TypeScript functions: camelCase, verb-based, max 3 words
```

**Watch for**:
- [ ] No `.py` files created anywhere (check `/tmp`, working directory)
- [ ] No `python` or `python3` commands in Bash tool calls
- [ ] No `import json`, `import yaml`, `import os` in any generated content
- [ ] All YAML/JSON parsing done "mentally" by Claude (via Read tool output)
- [ ] Entry created successfully in `.kb/`

**Post-test verification**:
```bash
# Check no .py files were created in the last 5 minutes
find /tmp -name "*.py" -mmin -5 2>/dev/null
find . -name "*.py" -mmin -5 -not -path "./node_modules/*" 2>/dev/null
```

### Test P2: No Python Scripts During Codebase Scan

**Command**:
```bash
/knowledge-scan
```

**Watch for**:
- [ ] Discovery phase uses Grep/Glob/Read tools (not Python)
- [ ] No `python` Bash commands
- [ ] Stack detection works via Glob (config files) and Read (parsing mentally)
- [ ] Pattern analysis uses Grep tool with regex patterns

### Test P3: No Python Scripts During Quality Evaluation

**Command**:
```bash
/knowledge-report
```

**Watch for**:
- [ ] Scoring algorithms applied "mentally" (not via Python execution)
- [ ] Flesch-Kincaid formula applied without Python scripts
- [ ] Quality report returned in expected format

### Test P4: Execution Constraint Block Presence

**Verification** (run outside Claude Code):
```bash
# All 11 files must contain the constraint block
for file in \
  plugins/knowledge-base/skills/knowledge-draft/SKILL.md \
  plugins/knowledge-base/skills/knowledge-evaluate/SKILL.md \
  plugins/knowledge-base/skills/codebase-discover/SKILL.md \
  plugins/knowledge-base/skills/knowledge-review/SKILL.md \
  plugins/knowledge-base/agents/knowledge-scanner.md \
  plugins/knowledge-base/agents/knowledge-evaluator.md \
  plugins/knowledge-base/agents/knowledge-writer.md \
  plugins/knowledge-base/commands/knowledge-add.md \
  plugins/knowledge-base/commands/knowledge-reindex.md \
  plugins/knowledge-base/commands/knowledge-scan.md \
  plugins/knowledge-base/commands/knowledge-report.md; do
  if grep -q "EXECUTION CONSTRAINT" "$file"; then
    echo "OK: $file"
  else
    echo "MISSING: $file"
  fi
done

# No python code blocks should remain
grep -rn '```python' plugins/knowledge-base/ --include="*.md"
# Expected: 0 results

# No Python imports should remain
grep -rn 'import os\|import json\|import yaml\|import re' plugins/knowledge-base/ --include="*.md"
# Expected: 0 results
```

---

## Functional Tests

### Test 1: Create Entry from Text (Basic)

**Objective**: Verify basic knowledge entry creation with minimal input

**Command**:
```bash
/knowledge-add TypeScript files should not exceed 800 lines
```

**Expected Behavior**:
1. `knowledge-writer` agent activates
2. `knowledge-draft` skill processes input
3. Detects scope: `typescript`, category: `rules`
4. Creates file at `.kb/typescript/rules/file-size.md` (or similar)
5. YAML frontmatter includes: `type`, `version: 1.0`, `scope`, `category`, `tags`
6. Quality evaluation runs (non-blocking)

**Expected Output**:
```
✓ Knowledge entry created: .kb/typescript/rules/file-size.md
Version: 1.0
Quality: X.X/10 (Clarity: N, Format: N, Structure: N, Completeness: N, Efficiency: N)
```

**Validation**:
- [ ] File exists at expected path
- [ ] YAML front matter is valid (starts with `---`, ends with `---`)
- [ ] Required fields present: type, version, scope, category, tags
- [ ] Type is one of: convention, rule, pattern, guide, documentation, reference, style, environment
- [ ] Content is comprehensive tier (150-300 lines)
- [ ] No Python scripts created

---

### Test 2: Create Entry with Multiple Rules

**Objective**: Verify comprehensive entry creation with rich input

**Command**:
```bash
/knowledge-add TypeScript functions: camelCase, verb-based, descriptive, max 3 words, boolean functions start with is/has/should
```

**Expected Behavior**:
1. Detects scope: `typescript`, category: `naming`
2. Generates comprehensive content with multiple sections
3. Path: `.kb/typescript/naming/functions.md` (or similar)
4. Includes Format section, Allowed/Forbidden examples, Rules

**Validation**:
- [ ] Contains Format section with bullet points
- [ ] Contains Allowed section with good examples
- [ ] Contains Forbidden section with anti-patterns
- [ ] Rules section with guidelines
- [ ] Code blocks have `typescript` language tags
- [ ] Quality score provided

---

### Test 3: Create Entry with Explicit Parameters

**Objective**: Verify explicit type/scope/category override auto-detection

**Command**:
```bash
/knowledge-add type:rule scope:react category:patterns Always use ErrorBoundary components around route-level components
```

**Expected Behavior**:
1. Parses `type:rule`, `scope:react`, `category:patterns`
2. Uses explicit values (confidence=1.0, no ambiguity questions)
3. Creates at `.kb/react/patterns/error-boundary.md` (or similar)

**Validation**:
- [ ] YAML `type: rule`
- [ ] YAML `scope: react`
- [ ] YAML `category: patterns`
- [ ] No AskUserQuestion prompts for type/scope/category

---

### Test 4: Create Entry from URL

**Objective**: Verify URL source detection and content fetching

**Command**:
```bash
/knowledge-add https://example.com/style-guide.md scope:typescript
```

**Expected Behavior**:
1. Detects URL pattern (`https?://`)
2. Fetches content via WebFetch
3. Processes fetched content as guidelines
4. Creates knowledge entry

**Validation**:
- [ ] URL correctly detected as source
- [ ] Content fetched and used
- [ ] Entry created with fetched content as base

---

### Test 5: Create Entry from Local File

**Objective**: Verify local file source detection

**Command**:
```bash
/knowledge-add ./docs/guidelines.md scope:typescript
```

**Expected Behavior**:
1. Detects file path (ends with `.md`)
2. Reads file content via Read tool
3. Processes file content as guidelines
4. Security check: rejects paths with `..`

**Validation**:
- [ ] File content correctly read
- [ ] Entry created from file content
- [ ] Path with `..` is rejected with error

---

### Test 6: Batch Processing from Directory

**Objective**: Verify batch processing of multiple files

**Command**:
```bash
/knowledge-add ./docs/guidelines/ scope:typescript
```

**Expected Behavior**:
1. Detects directory source
2. Globs `**/*.md` in directory
3. Excludes `README.md` and hidden files
4. Processes each file independently
5. Shows aggregate results

**Expected Output**:
```
✓ Batch processing complete

Success: N files
  - ./docs/file1.md → .kb/typescript/.../entry1.md (v1.0)
  ...

Skipped: N files (if any ambiguous)
Failed: N files (if any errors)
```

**Validation**:
- [ ] README.md excluded from processing
- [ ] Hidden files excluded
- [ ] Per-file success/failure tracking
- [ ] Batch continues on individual file failure

---

### Test 7: Russian Input

**Objective**: Verify multi-language support (Russian input, English output)

**Command**:
```bash
/knowledge-add Правила обработки ошибок в TypeScript: try-catch блоки, логирование, пользовательские классы ошибок
```

**Expected Behavior**:
1. Detects Cyrillic characters
2. Processes correctly
3. Output convention is in English
4. Scope correctly detected as `typescript`

**Validation**:
- [ ] Entry content is in English
- [ ] Rules correctly interpreted from Russian
- [ ] YAML metadata accurate

---

### Test 8: Ambiguous Scope Detection

**Objective**: Verify AskUserQuestion triggers when scope is ambiguous

**Command**:
```bash
/knowledge-add Error handling best practices
```

**Expected Behavior**:
1. Scope confidence < 0.7 (could be typescript, python, or general)
2. AskUserQuestion prompts user to choose scope
3. After user selects, entry is created

**Validation**:
- [ ] AskUserQuestion prompt appears
- [ ] Options include relevant scopes
- [ ] After selection, correct scope used

---

### Test 9: Update Existing Entry (Minor)

**Objective**: Verify minor update without confirmation

**Setup**: First create an entry:
```bash
/knowledge-add TypeScript functions should use camelCase
```

**Command** (add rule to same topic):
```bash
/knowledge-add Add rule: TypeScript functions should document thrown errors
```

**Expected Behavior**:
1. Detects existing file at same path
2. Intent: `append_rules` (minor update)
3. No confirmation needed
4. Version increments: 1.0 → 1.1

**Validation**:
- [ ] File updated (not replaced)
- [ ] Version incremented to 1.1
- [ ] New rule added to Rules section
- [ ] Existing content preserved

---

### Test 10: Update Existing Entry (Major — Requires Confirmation)

**Command**:
```bash
/knowledge-add Completely rewrite TypeScript function naming conventions
```

**Expected Behavior**:
1. Detects existing file
2. Intent: `full_overwrite` (major update)
3. AskUserQuestion for confirmation
4. If confirmed, version increments: 1.x → 2.0

**Validation**:
- [ ] Confirmation prompt appears
- [ ] If confirmed: file overwritten, version = 2.0
- [ ] If rejected: file unchanged

---

### Test 11: Rebuild Index

**Objective**: Verify index generation from existing entries

**Setup**: Create 2-3 test entries first (Tests 1-3)

**Command**:
```bash
/knowledge-reindex
```

**Expected Behavior**:
1. Scans `.kb/**/*.md` (excludes README.md, TEMPLATE.md)
2. Extracts YAML frontmatter from each file
3. Groups by scope → category
4. Generates hierarchy of README files:
   - `.kb/README.md` (root summary)
   - `.kb/{scope}/README.md` (scope details)
   - `.kb/{scope}/{category}/README.md` (category details)

**Expected Output**:
```
Scanning knowledge entries directory...
Found N knowledge entries

Organizing by scope and category...
- typescript: N entries (naming: X, rules: Y)

Generating category READMEs...
✓ .kb/typescript/naming/README.md (X entries)
...

Generating scope READMEs...
✓ .kb/typescript/README.md (N entries)

Generating root README...
✓ .kb/README.md (summary with links)

Summary:
- Total knowledge entries: N
- Scopes: X
- Categories: Y
- README files generated: Z
```

**Validation**:
- [ ] `.kb/README.md` exists and contains summary
- [ ] Scope READMEs exist
- [ ] Category READMEs exist
- [ ] All entries listed
- [ ] Navigation links work (relative paths)
- [ ] Statistics accurate
- [ ] Scopes sorted alphabetically (`general` last)

---

### Test 12: Rebuild Index with Review

**Command**:
```bash
/knowledge-reindex --review
```

**Expected Behavior**:
1. Standard reindex (same as Test 11)
2. Additionally runs Level 1 validation on all entries
3. Reports structural issues found

**Validation**:
- [ ] Index rebuilt normally
- [ ] Validation report appended
- [ ] Issues listed (if any)
- [ ] No auto-fixes applied (report-only mode)

---

### Test 13: Quality Report

**Objective**: Verify deep quality evaluation

**Command**:
```bash
/knowledge-report
```

**Expected Behavior**:
1. Finds most recently modified entry (Glob sorted by mtime)
2. Delegates to `knowledge-evaluator` agent
3. Agent invokes `knowledge-evaluate` skill in deep mode
4. Returns detailed report

**Expected Output Format**:
```
# Knowledge Entry Quality Report

**File**: .kb/typescript/naming/functions.md
**Version**: 1.0
**Scope**: typescript | **Category**: naming
**Lines**: N | **Evaluated**: YYYY-MM-DD

---

## Overall Score: X.X/10 {grade}

{summary}

## Criteria Breakdown

### 1. Clarity: X/10
...

### 2. Format: X/10
...

### 3. Structure: X/10
...

### 4. Completeness: X/10
...

### 5. Utility/Efficiency: X/10
...

## Summary
...
```

**Validation**:
- [ ] All 5 criteria scored
- [ ] Grade assigned (Outstanding/Excellent/Good/Adequate/Needs Improvement)
- [ ] Strengths listed
- [ ] Recommendations provided
- [ ] Potential score calculated

---

### Test 14: Quality Report for Specific File

**Command**:
```bash
/knowledge-report functions
```

**Expected Behavior**:
1. Resolves "functions" to a file in `.kb/`
2. Tries multiple resolution strategies (exact path, glob pattern)
3. Evaluates found file

**Validation**:
- [ ] File correctly resolved from partial name
- [ ] Report generated for correct file

---

### Test 15: Codebase Scan — Discovery Phase

**Objective**: Verify codebase convention discovery

**Command**:
```bash
/knowledge-scan
```

**Expected Behavior**:
1. `knowledge-scanner` agent activates
2. Delegates to `codebase-discover` skill
3. Detects project stack (languages, frameworks, tools)
4. Analyzes multiple dimensions (naming, structure, patterns, docs)
5. Calculates confidence scores
6. Presents discovery plan to user

**Expected Output**:
```
Project Stack:
  Languages: typescript, ...
  Frameworks: react, ...
  Tools: eslint, prettier, ...
  Files analyzed: N

Discovered Conventions:
  #  | Scope      | Category  | Dimension  | Confidence | Files
  1  | typescript | naming    | functions  | 94%        | 156
  2  | typescript | naming    | classes    | 88%        | 89
  ...

Options:
  /knowledge-scan generate all    — Generate all entries
  /knowledge-scan generate high   — Generate high-confidence only (>=80%)
  /knowledge-scan generate 1,3,5  — Generate specific entries
```

**Validation**:
- [ ] Project stack correctly detected
- [ ] Dimensions analyzed with Grep patterns
- [ ] Confidence scores calculated (0.0–1.0)
- [ ] Entries sorted by confidence descending
- [ ] No Python scripts created during scan

---

### Test 16: Codebase Scan — Generation Phase

**Command** (after Test 15):
```bash
/knowledge-scan generate high
```

**Expected Behavior**:
1. Filters discovered entries with confidence >= 0.80
2. Checks for conflicts with existing `.kb/` entries
3. Generates entries via `knowledge-writer` agent
4. Reports created/updated/skipped counts

**Validation**:
- [ ] Only high-confidence entries generated
- [ ] Conflicts detected and handled
- [ ] Created entries have proper YAML frontmatter
- [ ] Progress reported per entry

---

### Test 17: Codebase Scan — Auto Mode

**Command**:
```bash
/knowledge-scan auto
```

**Expected Behavior**:
1. Runs discovery
2. Automatically generates all high-confidence entries
3. Skips entries with conflicts (no user prompts)
4. Reports results

**Validation**:
- [ ] No interactive prompts
- [ ] Conflicts auto-skipped
- [ ] Summary shows created/skipped counts

---

### Test 18: Review Single Entry

**Command**:
```bash
/knowledge-review functions.md
```

**Expected Behavior**:
1. Finds file in `.kb/`
2. Runs Level 1 structural validation
3. Reports issues without modifying file
4. Includes quality score

**Validation**:
- [ ] File NOT modified (report-only mode)
- [ ] Structural issues identified (if any)
- [ ] Quality score calculated
- [ ] Recommendations provided

---

### Test 19: Review with Auto-Fix

**Command**:
```bash
/knowledge-review functions.md --fix
```

**Expected Behavior**:
1. Detects structural issues
2. Applies safe Level 1 fixes:
   - Add missing YAML fields (e.g., `version: 1.0`)
   - Add language tags to code blocks
   - Fix invalid version format
3. Reports fixes applied

**Validation**:
- [ ] File IS modified
- [ ] Fixes correctly applied
- [ ] Fix details reported
- [ ] Quality score improved after fixes

---

### Test 20: Batch Review

**Command**:
```bash
/knowledge-review typescript/ --fix
```

**Expected Behavior**:
1. Globs all `.md` files in scope directory
2. Processes each file independently
3. Continues on failure
4. Aggregates statistics

**Validation**:
- [ ] All files processed
- [ ] Per-file results reported
- [ ] Aggregate statistics accurate
- [ ] Batch continues on individual errors

---

### Test 21: PostToolUse Hook

**Objective**: Verify automatic notification after entry creation

**Command**:
```bash
/knowledge-add TypeScript async functions: use try-catch, await properly
```

**Expected Notification** (after file write):
```
✓ Convention file updated. Run /knowledge-reindex to update catalog.
```

**Validation**:
- [ ] Notification appears after Write to `.kb/**/*.md`
- [ ] Does NOT trigger for `.kb/README.md` writes
- [ ] Does NOT trigger for `.kb/TEMPLATE.md` writes

---

### Test 22: Research Integration

**Objective**: Verify codebase/web research triggers

**Command**:
```bash
/knowledge-add TypeScript error handling best practices
```

**Expected Behavior**:
1. Detects "best practices" trigger → web research
2. Low rule count (< 3) → additional research trigger
3. Searches codebase for try-catch patterns
4. Optionally runs WebSearch
5. Merges findings into comprehensive entry

**Validation**:
- [ ] Research triggered (codebase or web)
- [ ] Entry enriched with discovered patterns
- [ ] Quality higher than without research

---

## Edge Cases

### Test E1: No Arguments

**Command**:
```bash
/knowledge-add
```

**Expected**: Usage help displayed with examples

### Test E2: Invalid Type

**Command**:
```bash
/knowledge-add type:invalid Some guidelines
```

**Expected**: Error listing valid types: convention, rule, pattern, guide, documentation, reference, style, environment

### Test E3: Path Traversal Attempt

**Command**:
```bash
/knowledge-add ../../etc/passwd
```

**Expected**: Error: "Path cannot contain '..'"

### Test E4: Empty Knowledge Base

**Command** (with no `.kb/` directory):
```bash
/knowledge-report
```

**Expected**: Error indicating no knowledge entries found, with suggestion to create first entry

### Test E5: Non-Existent File Report

**Command**:
```bash
/knowledge-report nonexistent-file
```

**Expected**: Error with search paths tried and usage hint

---

## Performance Benchmarks

| Operation | Expected Time | Notes |
|-----------|---------------|-------|
| Create entry (simple text) | < 15 seconds | Includes quality evaluation |
| Create entry (with research) | < 30 seconds | Codebase + web research |
| Rebuild index (10 entries) | < 5 seconds | Glob + Read + Write |
| Rebuild index (100 entries) | < 15 seconds | Should scale linearly |
| Quality report (deep mode) | < 30 seconds | Full 5-criteria evaluation |
| Codebase scan (discovery) | < 60 seconds | Depends on project size |
| Codebase scan (generation, 5 entries) | < 120 seconds | 5x entry creation |
| Review single file | < 10 seconds | Level 1 + quality score |
| Review batch (20 files) | < 60 seconds | Level 1 per file |

---

## Validation Checklist

After running all tests, verify:

### Plugin Integrity
- [ ] Version is 1.4.0 in `plugin.json`
- [ ] All 11 instruction files contain EXECUTION CONSTRAINT block
- [ ] Zero `\`\`\`python` code blocks in any instruction file
- [ ] Zero Python imports in any instruction file

### Knowledge Entries
- [ ] All created entries have valid YAML frontmatter
- [ ] Required fields present: type, version, scope, category, tags
- [ ] Versions follow semantic format (major.minor)
- [ ] Code blocks have language tags
- [ ] Comprehensive tier content (150-300 lines)

### Commands
- [ ] `/knowledge-add` — creates entries from text, file, URL, directory
- [ ] `/knowledge-reindex` — rebuilds README hierarchy
- [ ] `/knowledge-reindex --review` — adds validation check
- [ ] `/knowledge-report` — generates detailed quality report
- [ ] `/knowledge-report <name>` — resolves partial names
- [ ] `/knowledge-scan` — discovers codebase conventions
- [ ] `/knowledge-scan generate` — creates entries from discoveries
- [ ] `/knowledge-review` — validates entries

### Agents
- [ ] `knowledge-writer` — orchestrates entry creation
- [ ] `knowledge-evaluator` — deep quality analysis
- [ ] `knowledge-scanner` — codebase convention discovery

### Skills
- [ ] `knowledge-draft` — transforms guidelines into structured entries
- [ ] `knowledge-evaluate` — scores entries across 5 criteria
- [ ] `codebase-discover` — detects project stack and patterns
- [ ] `knowledge-review` — structural validation and auto-fix

### Hooks
- [ ] PostToolUse triggers on `.kb/**/*.md` writes
- [ ] Excludes README.md and TEMPLATE.md
- [ ] Shows reminder to run `/knowledge-reindex`

### Python Prevention (Critical)
- [ ] No `.py` files created during any test
- [ ] No `python`/`python3` Bash commands executed
- [ ] No Python library imports (`json`, `yaml`, `os`, `re`) used
- [ ] All YAML/JSON parsed "mentally" from Read output
- [ ] All `invoke_skill()`/`invoke_agent()` references are pseudocode only

---

## Test Results Template

```
# Test Results

Date: YYYY-MM-DD
Tester: [Name]
Plugin Version: 1.4.0
Claude Code Version: [Version]

## Python Prevention Tests
- [ ] P1: No Python during entry creation
- [ ] P2: No Python during codebase scan
- [ ] P3: No Python during quality evaluation
- [ ] P4: Execution constraint blocks present

## Functional Tests
- [ ] Test 1: Basic entry creation
- [ ] Test 2: Multiple rules entry
- [ ] Test 3: Explicit parameters
- [ ] Test 4: URL source
- [ ] Test 5: Local file source
- [ ] Test 6: Batch directory
- [ ] Test 7: Russian input
- [ ] Test 8: Ambiguous scope
- [ ] Test 9: Minor update
- [ ] Test 10: Major update (confirmation)
- [ ] Test 11: Rebuild index
- [ ] Test 12: Rebuild index + review
- [ ] Test 13: Quality report (latest)
- [ ] Test 14: Quality report (by name)
- [ ] Test 15: Codebase scan (discovery)
- [ ] Test 16: Codebase scan (generate high)
- [ ] Test 17: Codebase scan (auto mode)
- [ ] Test 18: Review single entry
- [ ] Test 19: Review with auto-fix
- [ ] Test 20: Batch review
- [ ] Test 21: PostToolUse hook
- [ ] Test 22: Research integration

## Edge Cases
- [ ] E1: No arguments
- [ ] E2: Invalid type
- [ ] E3: Path traversal
- [ ] E4: Empty knowledge base
- [ ] E5: Non-existent file

## Issues Found
1. [Issue description]
2. [Issue description]

## Notes
- [Any observations or suggestions]
```

---

## Troubleshooting

### Command Not Recognized

```
Solution: Verify plugin is installed
Check: claude --plugin-dir ./plugins/knowledge-base
Or: ls .claude/plugins/knowledge-base/
```

### Entry Not Created

```
Solution: Check output for errors
Common: Ambiguous scope/category → answer AskUserQuestion prompt
Try: Add explicit parameters: /knowledge-add scope:typescript category:naming ...
```

### Python Script Created (v1.4.0 regression)

```
This should NEVER happen in v1.4.0.
If it does:
1. Note which command triggered it
2. Check which agent/skill was active
3. Verify EXECUTION CONSTRAINT block is present in the relevant .md file
4. Report as critical bug
```

### Quality Score Too Low

```
Solution: Provide richer input with more rules and examples
Or: Run /knowledge-report for detailed recommendations
Auto-improve: /knowledge-review <file> --fix for structural fixes
```

### Index Out of Date

```
Solution: Run /knowledge-reindex after creating/modifying entries
The PostToolUse hook should remind you automatically
```
