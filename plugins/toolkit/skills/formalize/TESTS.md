# Formalizer Testing Guide

This document provides comprehensive test cases to validate the Formalizer functionality across all modes and triggering mechanisms.

## Test Environment

- **Plugin Location**: `.claude/plugins/toolkit/` or `plugins/toolkit/`
- **Components Tested**:
  - Skill: `formalize` (5-step pipeline)
  - Command: `/formalize` (manual invocation)
  - Agent: `formalizer` (orchestration)
  - Hook: `UserPromptSubmit` (automatic triggering)
- **Claude Code Version**: Latest
- **Model**: Sonnet (agent default)

## Pre-Test Checklist

- [ ] Toolkit plugin installed
- [ ] `/formalize` command available (run `/help` to verify)
- [ ] `formalizer` agent registered
- [ ] `UserPromptSubmit` hook active in `plugins/toolkit/hooks/hooks.json`
- [ ] Test project with sample code files (for codebase research tests)
- [ ] Claude Code CLI is running

## Test Cases

---

### Test 1: Simple Clear Task → Silent Mode

**Objective**: Verify high-confidence tasks return minimal formatted output without preamble

**Command**:
```bash
/formalize run tests in packages/playwright-integration
```

**Expected Behavior**:
1. Skill parses input: single clear objective, no ambiguity
2. Confidence score: 0.90-0.95
3. Mode selected: `silent` (≥ 0.8 threshold)
4. Returns compact format with no interpretation preamble

**Expected Output**:
```markdown
CONFIDENCE: 0.95
MODE: silent
TASKS_COUNT: 1
---
**Task**: Run tests — Execute test suite in packages/playwright-integration
**DoD**: All tests execute, results reported
```

**Validation**:
- [ ] Confidence score is ≥ 0.90
- [ ] Mode is `silent`
- [ ] No "> Interpreting as:" preamble present
- [ ] Compact format used (not full structure)
- [ ] Task count is 1
- [ ] Execution takes < 3 seconds

**Check**:
```bash
# Output should be minimal, no extra sections
```

---

### Test 2: Standard Task → Summary Mode

**Objective**: Validate medium-confidence tasks include interpretation summary

**Command**:
```bash
/formalize fix the auth bug, update API docs, deploy to staging if tests pass
```

**Expected Behavior**:
1. Skill detects 3 distinct tasks bundled together
2. Decomposes into ordered tasks with dependencies
3. Confidence score: 0.65-0.75 (some ambiguity in "the auth bug")
4. Mode selected: `summary`
5. Includes "> Interpreting as:" preamble
6. Dependency graph shown

**Expected Output**:
```markdown
CONFIDENCE: 0.72
MODE: summary
TASKS_COUNT: 3
---
> Interpreting as: 3 tasks with sequential dependencies
> (fix → test → deploy), plus parallel docs update.

## Task 1: Fix authorization bug
**Objective**: Identify and fix the bug in the authorization module
**Requirements**:
- Locate the failing auth flow
- Implement fix
- Add/update tests for the fixed behavior
**Done When**: Auth tests pass, no regression in related tests
**Ambiguities**: [ASSUMED: most recent auth-related issue]

## Task 2: Update API documentation (parallel with Task 1)
**Objective**: Update API docs to reflect current state
**Ambiguities**: [ASSUMED: docs related to auth changes]
**Done When**: Documentation matches current API behavior

## Task 3: Deploy to staging (depends on: Task 1 tests passing)
**Objective**: Deploy current branch to staging environment
**Constraints**: Only proceed if all tests pass
**Done When**: Staging deployment successful, smoke tests pass

Dependency graph: [Task 1] → [Task 3]
                  [Task 2] → (parallel)
```

**Validation**:
- [ ] Confidence score is 0.65-0.75
- [ ] Mode is `summary`
- [ ] Preamble with interpretation present
- [ ] 3 tasks identified correctly
- [ ] Dependencies detected (Task 1 → Task 3)
- [ ] Parallel task flagged (Task 2)
- [ ] [ASSUMED] markers present for ambiguities
- [ ] Execution takes < 5 seconds

**Check**:
```bash
# Verify dependency graph is correct
# Check that "if tests pass" condition is in Task 3 constraints
```

---

### Test 3: Vague Task → Clarify Mode

**Objective**: Confirm low-confidence tasks request clarification with marked assumptions

**Command**:
```bash
/formalize clean up the code and make it like the other service
```

**Expected Behavior**:
1. Skill identifies multiple vague terms: "clean up", "the other service"
2. Confidence score: 0.30-0.40
3. Mode selected: `clarify` (< 0.5 threshold)
4. Output includes "⚠️ Need clarification:" section
5. Specific questions listed
6. Best-guess interpretation with [UNKNOWN] markers

**Expected Output**:
```markdown
CONFIDENCE: 0.35
MODE: clarify
TASKS_COUNT: 1
---
> ⚠️ Need clarification:
> 1. Which code/module needs cleanup? (specific files or directories?)
> 2. Which "other service" are you referring to?
> 3. What aspects should match? (code style? architecture? API design?)
>
> Current best interpretation:

## Task: Code cleanup
**Objective**: Refactor code to match patterns from [UNKNOWN service]
**Requirements**: [Cannot determine without answers above]
**Done When**: [Cannot determine]
```

**Validation**:
- [ ] Confidence score is < 0.5
- [ ] Mode is `clarify`
- [ ] Warning emoji and "Need clarification:" present
- [ ] Specific numbered questions provided (≥ 2)
- [ ] [UNKNOWN] or similar markers for indeterminate values
- [ ] "Current best interpretation" section included
- [ ] Execution takes < 5 seconds

**Check**:
```bash
# Verify questions are specific and actionable
# Confirm [UNKNOWN] markers are used appropriately
```

---

### Test 4: Multi-Task Bundle → Decomposition

**Objective**: Verify complex decomposition with 3+ tasks and dependency analysis

**Command**:
```bash
/formalize update dependencies, fix breaking changes, run integration tests, generate migration guide if any breaking changes found, deploy to production
```

**Expected Behavior**:
1. Skill identifies 5 distinct tasks
2. Detects sequential and conditional dependencies
3. Confidence score: 0.60-0.70 (clear tasks, but complex dependencies)
4. Mode selected: `summary`
5. Dependency graph shows proper sequencing
6. Conditional task flagged (migration guide)

**Expected Output**:
```markdown
CONFIDENCE: 0.67
MODE: summary
TASKS_COUNT: 5
---
> Interpreting as: 5-step release workflow with conditional documentation task

## Task 1: Update dependencies
**Objective**: Update project dependencies to latest compatible versions
**Done When**: Dependencies updated, lock file regenerated

## Task 2: Fix breaking changes (depends on: Task 1)
**Objective**: Address any breaking changes from dependency updates
**Requirements**: Code compiles, no runtime errors
**Done When**: All existing functionality works with new dependencies

## Task 3: Run integration tests (depends on: Task 2)
**Objective**: Execute full integration test suite
**Done When**: All integration tests pass

## Task 4: Generate migration guide (conditional: if breaking changes in Task 2)
**Objective**: Document migration steps for breaking changes
**Constraints**: Only needed if Task 2 identified breaking changes
**Done When**: Migration guide written and reviewed

## Task 5: Deploy to production (depends on: Task 3 passing)
**Objective**: Deploy current version to production environment
**Constraints**: Only proceed if Task 3 tests pass
**Done When**: Production deployment successful, health checks pass

Dependency graph:
[Task 1] → [Task 2] → [Task 3] → [Task 5]
           [Task 2] ⟶ [Task 4] (conditional)
```

**Validation**:
- [ ] All 5 tasks identified
- [ ] Dependencies correctly sequenced (1→2→3→5)
- [ ] Conditional task (Task 4) marked
- [ ] Mode is `summary` (confidence 0.6-0.7)
- [ ] Dependency graph clearly shows relationships
- [ ] Execution takes < 7 seconds

---

### Test 5: File References → Codebase Research

**Objective**: Verify skill uses Read/Grep/Glob when code references are detected

**Setup**:
Create test file:
```bash
mkdir -p test-code
cat > test-code/auth.ts << 'EOF'
export async function login(email: string, password: string) {
  // TODO: Add proper error handling
  const response = await fetch('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ email, password })
  });
  return response.json();
}
EOF
```

**Command**:
```bash
/formalize add error handling to the login function in test-code/auth.ts like we do in the checkout flow
```

**Expected Behavior**:
1. Skill detects file reference: `test-code/auth.ts`
2. Uses Read tool to examine the file
3. Searches for "checkout" patterns using Grep
4. References existing code patterns in formalized output
5. Confidence score: 0.60-0.70 (specific file, but "checkout flow" needs resolution)

**Expected Output**:
```markdown
CONFIDENCE: 0.68
MODE: summary
TASKS_COUNT: 1
---
> Interpreting as: Add error handling to login function following
> existing checkout flow patterns

## Task: Add error handling to login function
**Objective**: Implement error handling in test-code/auth.ts login function

**Context**: Current login function lacks error handling (has TODO comment).
Should follow patterns from checkout flow.

**Requirements**:
- Add try-catch block around fetch call
- Handle network errors
- Handle API error responses
- [ASSUMED: Follow checkout flow error handling patterns]

**Done When**:
- [ ] Try-catch implemented
- [ ] Error cases handled
- [ ] Pattern matches checkout flow
- [ ] TODO comment removed
```

**Validation**:
- [ ] File was read (verify Read tool was called)
- [ ] Code context included in formalization
- [ ] Reference to "checkout flow" preserved
- [ ] [ASSUMED] marker used for pattern matching
- [ ] Execution takes < 10 seconds (includes file I/O)

**Cleanup**:
```bash
rm -rf test-code
```

---

### Test 6: Low Confidence Auto-Formalize (Hook Trigger)

**Objective**: Verify UserPromptSubmit hook automatically invokes formalizer when confidence < 0.6

**Command** (enter as normal user prompt, NOT as `/formalize`):
```
clean up the messy code, you know what I mean
```

**Expected Behavior**:
1. UserPromptSubmit hook evaluates prompt quality
2. Scores confidence < 0.6 (extremely vague)
3. Hook automatically invokes formalizer agent via Task tool
4. Agent returns formalized output in clarify mode
5. Claude proceeds with implementation using formalized spec

**Expected Hook Output** (visible in conversation):
```markdown
[Formalizer agent invoked automatically due to low prompt quality]

CONFIDENCE: 0.25
MODE: clarify
TASKS_COUNT: 1
---
> ⚠️ Need clarification:
> 1. Which code/files are "messy"?
> 2. What specific issues need addressing?
> 3. What does "clean up" mean in this context? (refactor? format? remove unused code?)
>
> Current best interpretation:

## Task: Code cleanup
**Objective**: [Cannot determine without clarification]
**Scope**: [UNKNOWN: which files/modules]
**Requirements**: [UNKNOWN: specific cleanup actions]
```

**Validation**:
- [ ] Hook triggered automatically (no manual `/formalize` needed)
- [ ] Formalizer agent invocation visible
- [ ] Confidence score < 0.6
- [ ] Clarify mode activated
- [ ] Claude asks user questions before proceeding
- [ ] Total execution time < 8 seconds (hook + formalization)

**Check**:
```bash
# Verify the hook did NOT execute the vague request directly
# Confirm formalization happened before any code changes
```

---

### Test 7: High Confidence Bypass (No Hook Trigger)

**Objective**: Confirm hook skips formalization for high-quality prompts (confidence ≥ 0.6)

**Command** (enter as normal user prompt):
```
add a console.log statement at line 42 in src/utils.ts to debug the API response structure
```

**Expected Behavior**:
1. UserPromptSubmit hook evaluates prompt quality
2. Scores confidence ≥ 0.8 (very specific, clear DoD)
3. Hook bypasses formalization
4. Claude proceeds directly with the request
5. No formalizer agent invoked

**Expected Output**:
```markdown
[No formalization - proceeding directly]

I'll add the console.log statement to src/utils.ts at line 42.
```

**Validation**:
- [ ] No "[Formalizer agent invoked]" message
- [ ] Claude proceeds immediately to implementation
- [ ] No CONFIDENCE/MODE/TASKS_COUNT output
- [ ] Execution faster (< 3 seconds, no formalization overhead)
- [ ] Task completed successfully

**Check**:
```bash
# Verify src/utils.ts was modified
# Confirm no formalization step occurred
```

---

### Test 8: Threshold Boundary Testing

**Objective**: Test behavior at exact confidence threshold boundaries (0.59 vs 0.61)

**Test 8A: Just Below Threshold (0.59)**

**Command**:
```
update the auth flow to handle edge cases better
```

**Expected Behavior**:
- Confidence: ~0.55-0.59 (some vagueness: "edge cases", "better")
- Hook triggers formalization
- Summary or clarify mode

**Test 8B: Just Above Threshold (0.61)**

**Command**:
```
update the login function in auth.ts to validate email format before API call
```

**Expected Behavior**:
- Confidence: ~0.61-0.65 (specific file, clear action)
- Hook bypasses formalization
- Proceeds directly

**Validation**:
- [ ] 0.59 triggers formalization
- [ ] 0.61 bypasses formalization
- [ ] Threshold is consistently 0.6
- [ ] Behavior changes correctly at boundary

---

### Test 9: Question Detection → Skip Formalization

**Objective**: Verify hook skips formalization when user asks questions (not requesting action)

**Command** (enter as normal user prompt):
```
what's the best way to handle authentication in React apps?
```

**Expected Behavior**:
1. Hook detects question pattern (no action requested)
2. Confidence set to 1.0 (always skip for questions)
3. No formalization triggered
4. Claude answers the question directly

**Expected Output**:
```markdown
There are several popular approaches for handling authentication in React apps:

1. JWT tokens with localStorage/sessionStorage
2. OAuth2/OpenID Connect flows
3. Session-based authentication with HTTP-only cookies
...
```

**Validation**:
- [ ] No formalization triggered
- [ ] Question answered directly
- [ ] No task specification generated
- [ ] Fast response (< 3 seconds)

---

### Test 10: Silent Mode Structure Validation

**Objective**: Verify silent mode uses compact format with no preamble

**Command**:
```bash
/formalize format all TypeScript files in the src directory
```

**Expected Behavior**:
1. Clear single task, high confidence (≥ 0.85)
2. Mode: silent
3. Compact format (no full structure)
4. No interpretation preamble

**Expected Output**:
```markdown
CONFIDENCE: 0.88
MODE: silent
TASKS_COUNT: 1
---
**Task**: Format TypeScript files — Run formatter on all .ts files in src/
**DoD**: All TypeScript files in src/ are formatted
```

**Validation**:
- [ ] No "> Interpreting as:" line
- [ ] Compact single-line format used
- [ ] Only essential fields (Task, DoD)
- [ ] No Context, Requirements, Constraints sections
- [ ] Output is ≤ 5 lines

---

### Test 11: Summary Mode Preamble Validation

**Objective**: Confirm summary mode includes clear interpretation statement

**Command**:
```bash
/formalize refactor the data layer to use TypeScript instead of JavaScript, update tests
```

**Expected Behavior**:
1. Two related tasks detected
2. Confidence: 0.65-0.75
3. Mode: summary
4. Preamble explains interpretation

**Expected Output**:
```markdown
CONFIDENCE: 0.71
MODE: summary
TASKS_COUNT: 2
---
> Interpreting as: 2-step migration from JavaScript to TypeScript in data layer,
> with corresponding test updates

## Task 1: Migrate data layer to TypeScript
**Objective**: Convert data layer files from .js to .ts with proper typing
**Requirements**:
- Rename .js files to .ts
- Add TypeScript type annotations
- Fix type errors
**Done When**: All data layer files are TypeScript, no compilation errors

## Task 2: Update tests (depends on: Task 1)
**Objective**: Update test files to work with TypeScript data layer
**Done When**: All tests pass with migrated code
```

**Validation**:
- [ ] Preamble starts with "> Interpreting as:"
- [ ] Interpretation is 1-2 sentences
- [ ] Interpretation accurately summarizes the tasks
- [ ] Preamble separated from tasks with blank line

---

### Test 12: Clarify Mode Markers Validation

**Objective**: Verify clarify mode uses [ASSUMED] and [INFERRED] markers correctly

**Command**:
```bash
/formalize improve the error messages to be more user-friendly
```

**Expected Behavior**:
1. Vague scope: "the error messages" (which ones?)
2. Subjective criteria: "user-friendly" (how to measure?)
3. Confidence: 0.40-0.50
4. Mode: clarify
5. Assumptions marked explicitly

**Expected Output**:
```markdown
CONFIDENCE: 0.45
MODE: clarify
TASKS_COUNT: 1
---
> ⚠️ Need clarification:
> 1. Which error messages? (API errors? validation? system errors?)
> 2. What makes an error message "user-friendly"? (tone? detail level? suggested actions?)
> 3. Are there existing patterns to follow?
>
> Current best interpretation:

## Task: Improve error message user-friendliness
**Objective**: Update error messages to be more helpful to users

**Requirements**: [ASSUMED: Replace technical jargon with plain language]

**Scope**: [INFERRED: All user-facing error messages]

**Success Criteria**: [ASSUMED: Users can understand and act on errors]

**Done When**:
- [ ] Error messages reviewed and updated
- [ ] [ASSUMED: Technical terms replaced with plain language]
- [ ] [ASSUMED: Each error suggests a corrective action]
```

**Validation**:
- [ ] [ASSUMED] markers present (≥ 2 instances)
- [ ] [INFERRED] markers present (≥ 1 instance)
- [ ] Markers indicate specific assumptions made
- [ ] Alternatives or questions provided for each marker

---

### Test 13: Already Structured Input → Passthrough

**Objective**: Verify formalizer recognizes and preserves pre-structured input

**Command**:
```bash
/formalize Objective: Implement rate limiting for the API endpoint. Requirements: Use Redis for rate limit storage, 100 requests per hour per IP, return 429 status when exceeded. Definition of Done: Rate limiting active, tested with load tests showing correct throttling behavior.
```

**Expected Behavior**:
1. Skill detects structured format already present
2. Confidence: ≥ 0.85 (clear structure provided)
3. Mode: silent
4. Preserves user's structure with minimal changes

**Expected Output**:
```markdown
CONFIDENCE: 0.92
MODE: silent
TASKS_COUNT: 1
---
**Task**: Implement API rate limiting

**Objective**: Implement rate limiting for the API endpoint

**Requirements**:
- Use Redis for rate limit storage
- Limit: 100 requests per hour per IP
- Return 429 status when limit exceeded

**Definition of Done**:
- [ ] Rate limiting active
- [ ] Load tests verify correct throttling behavior
```

**Validation**:
- [ ] Original content preserved
- [ ] Minimal reformatting applied
- [ ] No new requirements added
- [ ] Structure recognized and enhanced
- [ ] Confidence score ≥ 0.85

---

### Test 14: Complex Dependency Graph

**Objective**: Test handling of 5+ tasks with multiple dependency types

**Command**:
```bash
/formalize create database schema, set up migrations, implement user model, add authentication endpoints, write unit tests for auth, integration tests for user flow, update API docs
```

**Expected Behavior**:
1. Identifies 7 distinct tasks
2. Detects complex dependencies:
   - Sequential: schema → migrations → model → auth
   - Parallel: unit tests + integration tests (both depend on auth)
   - Independent: docs (can happen anytime)
3. Confidence: 0.62-0.68
4. Mode: summary
5. Dependency graph visualizes relationships

**Expected Output**:
```markdown
CONFIDENCE: 0.66
MODE: summary
TASKS_COUNT: 7
---
> Interpreting as: Full authentication system implementation with
> sequential setup (DB → model → endpoints) and parallel testing

## Task 1: Create database schema
**Objective**: Define database schema for user authentication
**Done When**: Schema designed and documented

## Task 2: Set up migrations (depends on: Task 1)
**Objective**: Create migration files for schema deployment
**Done When**: Migration scripts created and tested

## Task 3: Implement user model (depends on: Task 2)
**Objective**: Create User model with authentication fields
**Done When**: User model implemented, migrations applied

## Task 4: Add authentication endpoints (depends on: Task 3)
**Objective**: Implement login/logout/register API endpoints
**Done When**: Auth endpoints functional

## Task 5: Write unit tests for auth (depends on: Task 4, parallel with Task 6)
**Objective**: Create unit tests for authentication logic
**Done When**: Unit tests pass, coverage ≥ 80%

## Task 6: Write integration tests (depends on: Task 4, parallel with Task 5)
**Objective**: Create integration tests for complete user flow
**Done When**: Integration tests pass

## Task 7: Update API documentation (independent, can be done anytime)
**Objective**: Document new authentication endpoints
**Done When**: API docs reflect auth endpoints

Dependency graph:
[Task 1] → [Task 2] → [Task 3] → [Task 4] → [Task 5]
                                            → [Task 6]
[Task 7] (independent)
```

**Validation**:
- [ ] All 7 tasks identified
- [ ] Sequential dependencies correct (1→2→3→4)
- [ ] Parallel tasks marked (5 and 6)
- [ ] Independent task flagged (7)
- [ ] Dependency graph shows all relationships
- [ ] Execution time < 7 seconds

---

### Test 15: Edge Case Battery

**Objective**: Test multiple edge cases in a single comprehensive test

**Test 15A: Empty Input**

**Command**:
```bash
/formalize
```

**Expected Output**:
```
Usage: /formalize <task description>

Examples:
  /formalize fix bug and update docs
  /formalize clean up the authentication code
  /formalize make it work like the checkout flow
```

**Validation**:
- [ ] Usage message displayed
- [ ] No error thrown
- [ ] Examples provided

---

**Test 15B: Single Word Input**

**Command**:
```bash
/formalize refactor
```

**Expected Output**:
```markdown
CONFIDENCE: 0.20
MODE: clarify
TASKS_COUNT: 1
---
> ⚠️ Need clarification:
> 1. What should be refactored? (files? modules? functions?)
> 2. What kind of refactoring? (extract? rename? restructure?)
> 3. What problem does this refactoring solve?
```

**Validation**:
- [ ] Very low confidence (< 0.3)
- [ ] Clarify mode activated
- [ ] Specific questions asked
- [ ] Does not attempt to proceed

---

**Test 15C: Already Formalized Bypass**

**Command** (as normal user prompt, not `/formalize`):
```
CONFIDENCE: 0.95
MODE: silent
TASKS_COUNT: 1
---
**Task**: Run tests
```

**Expected Behavior**:
- Hook detects already formalized format
- Confidence = 1.0 (bypass)
- No re-formalization

**Validation**:
- [ ] Input recognized as formalized
- [ ] Not re-processed
- [ ] Proceeds directly

---

**Test 15D: Circular Dependencies**

**Command**:
```bash
/formalize set up module A which depends on module B, and module B which depends on module A
```

**Expected Output**:
```markdown
CONFIDENCE: 0.50
MODE: clarify
TASKS_COUNT: 2
---
> ⚠️ Circular dependency detected:
> Module A depends on Module B
> Module B depends on Module A
>
> Need clarification:
> 1. Should one module be refactored to break the cycle?
> 2. Is there a shared dependency that should be extracted?
> 3. Can one dependency be inverted or made optional?
```

**Validation**:
- [ ] Circular dependency detected
- [ ] Clarify mode activated
- [ ] Specific resolution strategies suggested
- [ ] Does not attempt invalid sequencing

---

## Validation Checklist

After running all tests, verify:

### Skill Functionality
- [ ] 5-step pipeline executes correctly (Parse → Decompose → Formalize → Confidence → Output)
- [ ] Confidence scoring is consistent and meaningful
- [ ] Mode selection respects thresholds (≥0.8 silent, 0.5-0.79 summary, <0.5 clarify)
- [ ] Codebase research tools (Read/Grep/Glob) used when appropriate

### Command Interface
- [ ] `/formalize` command registered and accessible
- [ ] Accepts arguments correctly
- [ ] Returns formatted output
- [ ] Error handling for empty input

### Agent Orchestration
- [ ] `formalizer` agent invokes skill correctly
- [ ] Agent has access to Read/Grep/Glob tools
- [ ] Project memory persists across invocations
- [ ] Agent output format is consistent

### Hook Triggering
- [ ] UserPromptSubmit hook evaluates prompt quality
- [ ] Hook triggers formalization when confidence < 0.6
- [ ] Hook bypasses formalization when confidence ≥ 0.6
- [ ] Hook skips questions (no action requests)
- [ ] Hook does not trigger for already formalized input

### Output Modes
- [ ] **Silent mode** (≥0.8): Compact format, no preamble
- [ ] **Summary mode** (0.5-0.79): Includes "> Interpreting as:" preamble
- [ ] **Clarify mode** (<0.5): Shows warning, asks questions, marks assumptions

### Task Decomposition
- [ ] Multi-task bundles split correctly
- [ ] Sequential dependencies identified
- [ ] Parallel tasks flagged
- [ ] Conditional tasks marked
- [ ] Circular dependencies detected

### Assumption Marking
- [ ] [ASSUMED] markers used for inferred requirements
- [ ] [INFERRED] markers used for scope decisions
- [ ] [UNKNOWN] markers used when information missing
- [ ] Markers are specific and actionable

### Edge Cases
- [ ] Empty input handled gracefully
- [ ] Single-word input requests clarification
- [ ] Already structured input preserved
- [ ] Circular dependencies detected and flagged
- [ ] Questions bypass formalization

## Performance Benchmarks

Expected execution times:

| Operation | Expected Time | Notes |
|-----------|---------------|-------|
| Simple task (confidence ≥0.8) | < 3s | Minimal processing, silent mode |
| Standard task (0.5-0.8) | < 5s | Full pipeline, summary mode |
| Complex decomposition (3-5 tasks) | < 7s | Dependency analysis |
| Complex decomposition (6+ tasks) | < 10s | Extended dependency analysis |
| Codebase research | < 10s | File reading + analysis |
| Hook evaluation (bypass) | < 1s | Fast path, no formalization |
| Hook evaluation (trigger) | < 8s | Includes formalization overhead |

## Troubleshooting

### Common Issues

**Issue**: Command not recognized
```
Error: Unknown command: /formalize
```
**Solution**:
- Verify toolkit plugin is installed in `.claude/plugins/toolkit/`
- Check: `ls -la .claude/plugins/toolkit/commands/formalize.md`
- Reload CLI if recently installed

---

**Issue**: Hook not triggering automatically
```
[User enters vague prompt, but no formalization occurs]
```
**Solution**:
- Check hook is active: `cat .claude/plugins/toolkit/hooks/hooks.json`
- Verify UserPromptSubmit hook is present
- Test explicitly with `/formalize` to isolate issue
- Check Claude Code version supports UserPromptSubmit hooks

---

**Issue**: Confidence scores seem inconsistent
```
CONFIDENCE: 0.85 for a vague task
CONFIDENCE: 0.40 for a clear task
```
**Solution**:
- Review scoring factors in SKILL.md
- Check if codebase research provided context (raises confidence)
- Verify input matches expected clarity patterns
- File issue if scoring appears systematically wrong

---

**Issue**: Formalizer adds unwanted scope
```
User: "run tests"
Output: [50 lines of detailed test execution strategy]
```
**Solution**:
- Check confidence score (should be ≥0.9 for simple tasks)
- Verify silent mode is activated for high-confidence tasks
- Review anti-patterns in SKILL.md (avoid over-formalization)
- Simple tasks should get compact format

---

**Issue**: Codebase research not happening
```
User: "fix the bug in auth.ts"
Output: [No file context included]
```
**Solution**:
- Verify agent has access to Read/Grep/Glob tools
- Check agent frontmatter: `tools: Read, Grep, Glob`
- Ensure file references are explicit (not just "the file")
- Test with explicit file path in command

---

**Issue**: Hook triggers on questions
```
User: "what's the best way to handle auth?"
[Formalization triggered incorrectly]
```
**Solution**:
- Check hook logic for question detection
- Verify question patterns are recognized (what/how/why at start)
- Update hook if question detection is faulty
- Questions should always bypass formalization

---

## Reporting Issues

If tests fail, collect the following information:

1. **Command/Input Used**:
   ```
   /formalize <exact input>
   ```

2. **Expected Behavior**:
   - Confidence score range
   - Expected mode (silent/summary/clarify)
   - Task count

3. **Actual Behavior**:
   - Actual confidence score
   - Actual mode
   - Output diff from expected

4. **Error Messages** (if any):
   ```
   [Paste full error output]
   ```

5. **Environment**:
   - Claude Code version: `claude --version`
   - Plugin version: Check `plugins/toolkit/.claude-plugin/plugin.json`
   - Test date and tester name

6. **Reproducibility**:
   - Can issue be reproduced consistently?
   - Does it occur with variations of input?
   - Does it occur in fresh Claude Code session?

**Report to**: GitHub Issues in claude-code-plugins repository

---

## Test Summary Template

Use this template to document test results:

```markdown
# Formalizer Test Results

**Date**: 2026-02-15
**Tester**: [Your Name]
**Plugin Version**: 1.0.0
**Claude Code Version**: [Version]

## Tests Passed

### Category 1: Manual Command Invocation
- [ ] Test 1: Simple Clear Task → Silent Mode
- [ ] Test 2: Standard Task → Summary Mode
- [ ] Test 3: Vague Task → Clarify Mode
- [ ] Test 4: Multi-Task Bundle → Decomposition
- [ ] Test 5: File References → Codebase Research

### Category 2: Automatic Hook Triggering
- [ ] Test 6: Low Confidence Auto-Formalize
- [ ] Test 7: High Confidence Bypass
- [ ] Test 8: Threshold Boundary Testing
- [ ] Test 9: Question Detection → Skip Formalization

### Category 3: Output Mode Validation
- [ ] Test 10: Silent Mode Structure
- [ ] Test 11: Summary Mode Preamble
- [ ] Test 12: Clarify Mode Markers

### Category 4: Integration & Advanced
- [ ] Test 13: Already Structured Input → Passthrough
- [ ] Test 14: Complex Dependency Graph

### Category 5: Edge Cases
- [ ] Test 15A: Empty Input
- [ ] Test 15B: Single Word Input
- [ ] Test 15C: Already Formalized Bypass
- [ ] Test 15D: Circular Dependencies

## Issues Found

1. **Issue**: [Description]
   - **Severity**: [Critical/High/Medium/Low]
   - **Test**: [Test number]
   - **Reproduction**: [Steps]

2. **Issue**: [Description]
   - **Severity**: [Critical/High/Medium/Low]
   - **Test**: [Test number]
   - **Reproduction**: [Steps]

## Performance Notes

| Test | Expected Time | Actual Time | Status |
|------|---------------|-------------|--------|
| Test 1 | < 3s | | |
| Test 2 | < 5s | | |
| ... | | | |

## Overall Assessment

**Pass Rate**: X/15 tests passed (X%)

**Critical Issues**: [Number]

**Recommendations**:
- [Any suggestions for improvements]
- [Areas needing attention]
- [Feature requests discovered during testing]

## Notes

[Any additional observations, edge cases discovered, or suggestions for future test coverage]
```

---

## Advanced Testing Scenarios

For comprehensive validation, consider these additional scenarios:

### Scenario A: Iterative Refinement
1. Start with vague input (clarify mode)
2. Provide clarification answers
3. Re-run formalization with refined input
4. Verify confidence improves

### Scenario B: Multi-Language Input
1. Test with non-English technical terms
2. Test with mixed language input
3. Verify output is always in English
4. Check technical terms preserved

### Scenario C: Context Accumulation
1. Run formalization in a session
2. Reference previous task implicitly
3. Verify formalizer uses session context
4. Test memory persistence

### Scenario D: Large-Scale Decomposition
1. Input: 10+ tasks in one prompt
2. Verify correct decomposition
3. Check dependency graph complexity
4. Validate performance (< 15s)

---

## Continuous Testing

Recommended testing frequency:

- **After skill changes**: Run all 15 tests
- **After agent updates**: Run Tests 6-9 (hook integration)
- **After hook changes**: Run Tests 6-9 (triggering logic)
- **Before releases**: Full test suite + advanced scenarios
- **Weekly**: Random sample of 5 tests for regression checking

## Success Criteria

The formalizer is working correctly when:

1. ✅ All 15 core tests pass
2. ✅ Confidence scores correlate with input clarity
3. ✅ Mode selection matches expected thresholds
4. ✅ Hook triggers/bypasses correctly
5. ✅ Assumptions are marked explicitly
6. ✅ Dependencies are detected accurately
7. ✅ Performance meets benchmarks
8. ✅ Edge cases handled gracefully
9. ✅ No over-formalization of simple tasks
10. ✅ No under-formalization of complex tasks
