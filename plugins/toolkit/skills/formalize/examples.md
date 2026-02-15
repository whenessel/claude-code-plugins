# Formalize — Examples

Real-world examples showing how the formalizer processes different input types.


## Example 1: Multi-task Bundle (Confidence: 0.72 → Summary)


### Raw Input

```markdown
> fix the auth bug, update API docs, and deploy to staging if tests pass
```


### Formalized Output

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


## Example 2: Vague Scope (Confidence: 0.35 → Clarify)


### Raw Input

```markdown
> clean up the code, make it like the other service, you know what I mean
```

### Formalized Output

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


## Example 3: Clear Single Task (Confidence: 0.95 → Silent)


### Raw Input

```markdown
> run tests in packages/playwright-integration
```

### Formalized Output

```markdown
CONFIDENCE: 0.95
MODE: silent
TASKS_COUNT: 1
---
**Task**: Run tests — Execute test suite in packages/playwright-integration
**DoD**: All tests execute, results reported
```


## Example 4: Interleaved Context (Confidence: 0.65 → Summary)


### Raw Input

```markdown
> We have rrweb that records the DOM, and I need the Playwright integration
> to auto-capture recordings, like Cypress already does with before/after
> hooks. For Playwright it should probably use fixtures. Config should be
> minimal. Also need to handle parallel tests so recordings don't mix.
```

### Formalized Output

```markdown
CONFIDENCE: 0.65
MODE: summary
TASKS_COUNT: 1
---
> Interpreting as: Implement Playwright integration for rrweb recording
> with auto-setup via fixtures, minimal config, and parallel test isolation.

## Task: Implement Playwright rrweb Integration

**Objective**: Create a Playwright integration that automatically captures
rrweb DOM recordings during test execution

**Context**: rrweb records DOM changes. Cypress integration already exists
using before/after hooks. Playwright uses fixtures as its extension mechanism.

**Requirements**:
- Auto-inject rrweb recording via Playwright fixtures
- Minimal user configuration (≤ 3 lines)
- Parallel test isolation — separate recording per worker
- Follow patterns from existing Cypress integration

**Constraints**:
- Must not require extensive user configuration
- Recordings must not mix between parallel test workers

**Definition of Done**:
- [ ] Playwright fixture captures rrweb recording per test
- [ ] Works with parallel test execution
- [ ] User config is ≤ 3 lines
- [ ] Pattern consistent with Cypress approach

**Ambiguities**:
- [INFERRED] Output format same as Cypress integration
- [INFERRED] Fixture-based approach (user said "probably")
```
