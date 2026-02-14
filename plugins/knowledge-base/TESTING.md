# Conventions Plugin Testing Guide

This document provides comprehensive test cases to validate the Conventions Plugin functionality.

## Test Environment

- **Plugin Location**: `.claude/plugins/conventions/`
- **Test Conventions**: `.conventions/`
- **Claude Code Version**: Latest

## Pre-Test Checklist

- [ ] Plugin installed in `.claude/plugins/conventions/`
- [ ] All 9 plugin files present (run `tree .claude/plugins/conventions`)
- [ ] `.conventions/` directory exists (created during testing)
- [ ] Claude Code CLI is running

## Test Cases

### Test 1: Simple Convention (Direct Input)

**Objective**: Verify basic convention creation with minimal input

**Command**:
```bash
/convention TypeScript files should not exceed 800 lines
```

**Expected Behavior**:
1. Convention-writer agent activates
2. Draft-convention skill processes input
3. Detects tier: `simple` (20-50 lines)
4. Auto-generates path: `.conventions/typescript/rules/file-size.md`
5. Creates directory structure if needed
6. Writes convention file
7. Validates YAML and structure

**Expected Output**:
```
✓ Convention file created: .conventions/typescript/rules/file-size.md

Tier: simple
Lines: ~40
Sections: 3
```

**Validation**:
- [ ] File exists at expected path
- [ ] YAML front matter is valid
- [ ] Contains: Title, Description, Format, Allowed sections
- [ ] Line count is 20-50

**Check File**:
```bash
cat .conventions/typescript/rules/file-size.md
```

---

### Test 2: Standard Convention (Multiple Rules)

**Objective**: Verify standard-tier convention with multiple related rules

**Command**:
```bash
/convention TypeScript functions: camelCase, verb-based, descriptive, max 3 words, boolean functions start with is/has/should
```

**Expected Behavior**:
1. Agent detects 5 rules mentioned
2. Tier detection: `standard` (50-150 lines)
3. Creates comprehensive structure with Allowed/Forbidden/Rules sections
4. Auto-generates examples based on rules
5. Path: `.conventions/typescript/naming/functions.md`

**Expected Output**:
```
✓ Convention file created: .conventions/typescript/naming/functions.md

Tier: standard
Lines: ~95
Sections: 6
Examples: 6
```

**Validation**:
- [ ] Contains Allowed section with good examples
- [ ] Contains Forbidden section with anti-patterns
- [ ] Rules section with numbered guidelines
- [ ] Line count is 50-150
- [ ] Code blocks have `typescript` language tags

**Check File**:
```bash
cat .conventions/typescript/naming/functions.md
head -50 .conventions/typescript/naming/functions.md
```

---

### Test 3: Interactive Mode

**Objective**: Verify interactive prompts and user input collection

**Command**:
```bash
/convention interactive
```

**Expected Prompts**:
```
1. Topic: Error Handling
2. Scope: TypeScript
3. Category: patterns
4. Guidelines (multiline):
   > try-catch blocks around async operations
   > custom Error classes extending Error
   > log errors with context
   > user-friendly error messages
   >
5. Tags: error, exceptions, async, logging
```

**Expected Behavior**:
1. AskUserQuestion prompts appear sequentially
2. Collects all inputs
3. Constructs structured guidelines
4. Detects tier: `comprehensive` (7+ related concepts)
5. Creates detailed convention with subsections

**Expected Output**:
```
✓ Convention file created: .conventions/typescript/patterns/error-handling.md

Tier: comprehensive
Lines: ~220
Sections: 10
Examples: 8
```

**Validation**:
- [ ] All user inputs are reflected in the file
- [ ] Comprehensive structure with subsections
- [ ] Summary table included
- [ ] Multiple domain sections
- [ ] Line count is 150-300

---

### Test 4: Custom Path

**Objective**: Verify --path argument overrides auto-generated path

**Command**:
```bash
/convention --path .conventions/custom/my-rules.md React hooks: start with use, camelCase, descriptive
```

**Expected Behavior**:
1. Parses --path argument
2. Validates path (must end with .md, no ..)
3. Uses custom path instead of auto-generated
4. Creates directory structure if needed

**Expected Output**:
```
✓ Convention file created: .conventions/custom/my-rules.md

Tier: standard
Custom path used
```

**Validation**:
- [ ] File created at `.conventions/custom/my-rules.md`
- [ ] Not created at default path (react/naming/hooks.md)
- [ ] YAML front matter still correct
- [ ] Scope is `react` (from input)

**Check**:
```bash
ls -la .conventions/custom/
cat .conventions/custom/my-rules.md
```

---

### Test 5: File Conflict Handling

**Objective**: Verify behavior when convention file already exists

**Setup**:
```bash
# First, create a convention
/convention TypeScript variables should use camelCase
```

**Command** (same topic):
```bash
/convention TypeScript variables: camelCase, descriptive, const for constants
```

**Expected Behavior**:
1. Detects file exists: `.conventions/typescript/naming/variables.md`
2. Returns conflict status
3. Agent prompts user for action

**Expected Prompt**:
```
⚠ File already exists: .conventions/typescript/naming/variables.md

Options:
1. Overwrite existing file
2. Rename new file (variables-v2.md)
3. Merge with existing content

What would you like to do? [1/2/3]
```

**Test Each Option**:
- [ ] **Option 1 (Overwrite)**: Replaces file with new content
- [ ] **Option 2 (Rename)**: Creates variables-v2.md
- [ ] **Option 3 (Merge)**: Combines rules from both (future enhancement)

---

### Test 6: Insufficient Input

**Objective**: Verify handling when user input is too vague

**Command**:
```bash
/convention error handling
```

**Expected Behavior**:
1. Skill detects <2 specific rules
2. Returns `insufficient_input` status
3. Agent asks clarifying questions

**Expected Output**:
```
⚠ Need more information to create convention

The following questions need answers:
- What specific rules should this convention enforce?
- Can you provide an example of correct usage?
- What problem does this convention solve?

Would you like me to research best practices for this topic?
```

**User Response Options**:
- Provide more details
- Request research: "Yes, research best practices"
- Cancel and try interactive mode

---

### Test 7: Russian Input

**Objective**: Verify multi-language support (Russian input, English output)

**Command**:
```bash
/convention Правила обработки ошибок: try-catch блоки, логирование, пользовательские классы ошибок
```

**Expected Behavior**:
1. Detects Cyrillic characters
2. Sets input_language: russian
3. Processes correctly
4. Output is in English

**Expected Output**:
```
✓ Convention file created: .conventions/typescript/patterns/error-handling.md

Language: Russian detected, output in English
Tier: standard
```

**Validation**:
- [ ] Convention content is in English
- [ ] Rules correctly translated/interpreted
- [ ] Metadata accurate
- [ ] Agent memory stores language preference

**Check**:
```bash
cat .conventions/typescript/patterns/error-handling.md | head -30
# Verify content is in English
```

---

### Test 8: Rebuild Index

**Objective**: Verify index generation from existing conventions

**Setup**:
```bash
# Create 2-3 test conventions first
/convention TypeScript functions: camelCase, verb-based
/convention TypeScript files: max 800 lines
/convention Python modules: snake_case, lowercase
```

**Command**:
```bash
/rebuild-index
```

**Expected Behavior**:
1. Scans `.conventions/**/*.md`
2. Excludes README.md, TEMPLATE.md
3. Extracts YAML front matter and titles
4. Groups by scope → category
5. Generates README.md with:
   - Directory structure
   - Grouped listings
   - Statistics
   - Last updated date

**Expected Output**:
```
Scanning conventions directory...
Found 3 convention files

Organizing by scope and category...
- typescript: 2 conventions
- python: 1 convention

✓ Index rebuilt: .conventions/README.md

Summary:
- Total conventions: 3
- Scopes: 2
- Categories: 2
```

**Validation**:
- [ ] `.conventions/README.md` exists
- [ ] Contains all 3 conventions
- [ ] Grouped by scope (typescript, python)
- [ ] Within scope, grouped by category
- [ ] Version numbers displayed
- [ ] Tags listed
- [ ] Statistics accurate
- [ ] Relative paths correct

**Check**:
```bash
cat .conventions/README.md
```

---

### Test 9: PostToolUse Hook

**Objective**: Verify automatic notification after convention file changes

**Command**:
```bash
/convention TypeScript async functions: use try-catch, await properly
```

**Expected Behavior**:
1. Convention file created successfully
2. PostToolUse hook triggers
3. Echo notification displayed

**Expected Notification**:
```
✓ Convention file updated. Run /rebuild-index to update catalog.
```

**Validation**:
- [ ] Notification appears after file write
- [ ] Does NOT trigger for README.md writes
- [ ] Does NOT trigger for TEMPLATE.md writes
- [ ] Only triggers for convention files

**Test Negative Case**:
```bash
# Manually edit .conventions/README.md
# Hook should NOT trigger
```

---

### Test 10: Research Integration

**Objective**: Verify codebase research when "best practices" mentioned

**Setup**:
Create some TypeScript files with patterns:
```bash
mkdir -p test-code
echo "async function fetchData() { try { ... } catch (e) { ... } }" > test-code/sample.ts
```

**Command**:
```bash
/convention TypeScript error handling best practices
```

**Expected Behavior**:
1. Detects "best practices" trigger
2. Runs codebase research:
   - Searches for try-catch patterns
   - Checks tsconfig.json
   - Looks for existing conventions
3. Optionally runs web research (if configured)
4. Merges findings with knowledge
5. Generates comprehensive convention

**Expected Output**:
```
Researching codebase patterns...
Found 15 try-catch blocks in TypeScript files
Found tsconfig.json with strict mode enabled

✓ Convention file created: .conventions/typescript/patterns/error-handling.md

Tier: comprehensive
Research: codebase patterns included
```

**Validation**:
- [ ] Convention includes codebase examples
- [ ] References project patterns
- [ ] Notes any conflicts between project and best practices
- [ ] More detailed than without research

---

## Validation Checklist

After running all tests, verify:

### Plugin Structure
- [ ] All 9 files present in `.claude/plugins/conventions/`
- [ ] No syntax errors in markdown files
- [ ] YAML front matter valid in all files

### Convention Files
- [ ] All created conventions have valid YAML
- [ ] Required fields present: type, version, scope, category, tags
- [ ] Versions follow semantic versioning
- [ ] Code blocks have language tags
- [ ] Line counts within tier ranges (±20%)

### Index
- [ ] README.md generated successfully
- [ ] All conventions listed
- [ ] Correct grouping by scope/category
- [ ] Statistics accurate
- [ ] Relative paths work

### Commands
- [ ] `/convention` works with direct input
- [ ] `/convention interactive` prompts correctly
- [ ] `/convention --path` respects custom paths
- [ ] `/rebuild-index` generates valid catalog

### Hooks
- [ ] PostToolUse notification triggers correctly
- [ ] Only triggers for convention file changes
- [ ] Message is clear and actionable

### Edge Cases
- [ ] File conflicts handled gracefully
- [ ] Insufficient input prompts for more info
- [ ] Invalid paths rejected
- [ ] Russian input processed correctly

## Performance Benchmarks

Expected performance:

| Operation | Expected Time | Notes |
|-----------|---------------|-------|
| Simple convention | <5 seconds | No research |
| Standard convention | <8 seconds | Basic research |
| Comprehensive convention | <15 seconds | Full research |
| Rebuild index (10 files) | <2 seconds | Fast glob + parse |
| Rebuild index (100 files) | <5 seconds | Should scale |

## Troubleshooting

### Common Issues

**Issue**: Command not recognized
```
Solution: Verify plugin is in .claude/plugins/conventions/
Check: ls -la .claude/plugins/
```

**Issue**: File not created
```
Solution: Check for error messages in output
Check: Provide more detailed guidelines
Try: Interactive mode for step-by-step
```

**Issue**: YAML validation fails
```
Solution: Check YAML syntax in generated file
Look for: Missing colons, incorrect array syntax
Auto-fix: Plugin should generate valid YAML
```

**Issue**: Index not updating
```
Solution: Ensure files are in .conventions/
Check: YAML front matter is valid
Verify: File not in excluded paths
```

## Reporting Issues

If tests fail, collect:
1. Command used
2. Expected behavior
3. Actual behavior
4. Error messages (if any)
5. Generated file content (first 50 lines)
6. Plugin version

Report to: GitHub issues in ai-knowledge-base repository

---

## Test Summary Template

```
# Test Results

Date: YYYY-MM-DD
Tester: [Name]
Plugin Version: 1.0.0

## Tests Passed
- [ ] Test 1: Simple Convention
- [ ] Test 2: Standard Convention
- [ ] Test 3: Interactive Mode
- [ ] Test 4: Custom Path
- [ ] Test 5: File Conflict
- [ ] Test 6: Insufficient Input
- [ ] Test 7: Russian Input
- [ ] Test 8: Rebuild Index
- [ ] Test 9: PostToolUse Hook
- [ ] Test 10: Research Integration

## Issues Found
1. [Issue description]
2. [Issue description]

## Notes
- [Any observations or suggestions]
```
