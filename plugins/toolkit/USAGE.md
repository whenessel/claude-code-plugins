# Usage Guide

Practical examples and workflows for the **toolkit** plugin.

## Table of Contents

- [Quick Start](#quick-start)
- [Automatic Formalization](#automatic-formalization)
- [Manual Formalization](#manual-formalization)
- [Context Building Workflows](#context-building-workflows)
- [Advanced Patterns](#advanced-patterns)
- [Integration with Other Tools](#integration-with-other-tools)
- [Best Practices](#best-practices)

---

## Quick Start

### Installation Verification

After installing the plugin, verify it's active:

```bash
# Check plugin status
/plugin list

# Verify hooks are loaded
cat .claude/plugins/toolkit/hooks/hooks.json
```

### First Test: Auto-Formalization

Try a vague request to see auto-formalization in action:

```
User: "clean up the auth code and add some tests"

[Hook evaluates: confidence ~0.45]
[Auto-invokes formalizer agent]

Output:
> ⚠️ Need clarification:
> 1. Which specific files in auth need cleanup?
> 2. What type of cleanup? (code style, architecture, naming)
> 3. What should the tests cover?
>
> Current best interpretation:
>
## Task 1: Auth code cleanup
**Objective**: Refactor authentication module
**Requirements**: [Cannot determine without answers above]
...
```

### First Context Build

Analyze your project:

```bash
/build-context
```

Output: `.claude/context/project-overview.md` with full architectural analysis.

---

## Automatic Formalization

### How It Works

The UserPromptSubmit hook evaluates every request:

```
Your Input → Confidence Scoring → Decision
                   ↓
         ≥ 0.6: Proceed directly
         < 0.6: Auto-formalize first
```

### Example 1: Multi-Objective Request

**Input:**
```
fix the login bug, update API docs, and deploy to staging if tests pass
```

**What Happens:**
1. Hook detects 3 objectives (confidence: ~0.55)
2. Auto-invokes `formalizer` agent
3. Returns structured spec:

```markdown
## Task 1: Fix login bug
**Objective**: Debug and resolve authentication issue
**Requirements**:
- Locate failing login flow
- Implement fix
- Add regression test
**Done When**: Login works, tests pass

## Task 2: Update API documentation (parallel)
**Objective**: Sync docs with current API
**Done When**: Documentation matches behavior

## Task 3: Deploy to staging (depends on: Task 1)
**Constraints**: Only if tests pass
**Done When**: Deployment successful, smoke tests pass

Dependency graph:
  [Task 1] → [Task 3]
  [Task 2] → (parallel)
```

### Example 2: Vague Scope Detection

**Input:**
```
make it work like the checkout flow
```

**What Happens:**
1. Hook detects implicit reference (confidence: ~0.35)
2. Auto-formalized with clarification request:

```markdown
> ⚠️ Need clarification:
> 1. What is "it"? Which feature/module?
> 2. Which aspects of checkout flow? (UX, error handling, validation)
> 3. What specifically should match?
>
> Current best interpretation:
## Task: Replicate checkout flow patterns
**Objective**: [UNKNOWN target] should match checkout flow behavior
**Requirements**: [Needs answers to questions above]
```

### Example 3: Clear Request (No Formalization)

**Input:**
```
run tests in packages/auth
```

**What Happens:**
1. Hook evaluates: single, clear objective (confidence: 0.95)
2. **No formalization** - proceeds directly
3. Executes: `pytest packages/auth/`

---

## Manual Formalization

Use `/formalize` for explicit control:

### Example 1: Complex Feature Request

```bash
/formalize We need better error handling. Maybe use try-catch everywhere? Also log errors to Sentry. Should probably add user-facing error messages too.
```

**Output:**

```markdown
CONFIDENCE: 0.58
MODE: summary
TASKS_COUNT: 3
---
> Interpreting as: Implement comprehensive error handling system
> with exception handling, logging, and user notifications.

## Task 1: Add try-catch blocks
**Objective**: Wrap async operations in error handling
**Requirements**:
- Identify all async functions without error handling
- Add try-catch blocks with appropriate error types
- Ensure no silent failures
**Done When**: All async operations have error handling

## Task 2: Integrate Sentry logging
**Objective**: Send errors to Sentry for monitoring
**Requirements**:
- Install/configure Sentry SDK
- Add Sentry.captureException() calls
- Include context (user ID, request data)
**Dependencies**: Requires Task 1 (error handling in place)
**Done When**: Errors appear in Sentry dashboard

## Task 3: User-facing error messages
**Objective**: Display helpful error messages to users
**Requirements**:
- Create error message templates
- Map error types to user messages
- Avoid exposing internal details
**Done When**: Users see clear error messages, no stack traces

**Ambiguities**:
- [ASSUMED] "everywhere" means all API routes and critical functions
- [INFERRED] User messages should be non-technical
```

### Example 2: Refactoring Request

```bash
/formalize Refactor the payment processing to use the same pattern as subscriptions
```

**Output:**

```markdown
CONFIDENCE: 0.62
MODE: summary
TASKS_COUNT: 1
---
> Interpreting as: Align payment processing architecture with
> existing subscription module patterns.

## Task: Refactor payment processing
**Objective**: Restructure payment code to match subscription patterns
**Context**: Subscription module exists and works well

**Requirements**:
- Read subscription module to understand pattern
- Identify key architectural differences
- Refactor payment processing to match
- Ensure API contract unchanged (no breaking changes)

**Definition of Done**:
- [ ] Payment processing uses same layering as subscriptions
- [ ] Payment tests still pass
- [ ] No breaking API changes
- [ ] Code review approval

**Files to examine first**:
- `src/subscriptions/` — reference architecture
- `src/payments/` — code to refactor

**Risks**:
- Payment processing may have domain-specific requirements
- Existing payment integrations must continue working
```

---

## Context Building Workflows

### Workflow 1: New Feature Development

**Scenario:** Implementing OAuth 2.0 support in an unfamiliar codebase.

```bash
# Step 1: Understand the overall architecture
/build-context

# Step 2: Deep dive into auth module
/build-context src/auth/ --deep

# Step 3: Create task-specific context
/task-context Add OAuth 2.0 provider support

# Step 4: Review generated contexts
cat .claude/context/project-overview.md
cat .claude/context/auth.md
cat .claude/context/add-oauth-2-0-provider-support.md

# Step 5: Implement with full context
# Now you have complete understanding of:
# - Project structure
# - Auth module APIs
# - Files to modify
# - Integration points
```

### Workflow 2: Bug Investigation

**Scenario:** Memory leak in recording service.

```bash
# Step 1: Create task context for the bug
/task-context Fix memory leak in recording service

# Output includes:
# - Files related to recording
# - Memory management patterns
# - Dependencies that might leak
# - Testing approaches

# Step 2: Review context
cat .claude/context/task-fix-memory-leak.md

# Step 3: Investigate with focused file list
# Context file tells you exactly which files to examine
```

### Workflow 3: Refactoring Planning

**Scenario:** Refactoring authentication system.

```bash
# Step 1: Document current architecture
/build-context src/auth/ --deep --output docs/architecture/auth-current.md

# Step 2: Formalize refactoring goals
/formalize Refactor auth to support multiple providers (OAuth, SAML, LDAP) with plugin architecture

# Step 3: Create refactoring-specific context
/task-context Implement multi-provider auth architecture

# Step 4: Compare current vs. planned
# Use both context documents to plan migration path
```

### Workflow 4: Onboarding to New Codebase

**Scenario:** First day on a new project.

```bash
# Day 1, Hour 1: High-level overview
/build-context --shallow

# Output: Quick scan of all modules, 2-3 minutes

# Hour 2: Identify key modules
/build-context src/core/ --deep
/build-context src/api/ --deep

# Day 2: Task-specific deep dive
/task-context Implement feature X

# Result: Full understanding without reading entire codebase
```

---

## Advanced Patterns

### Pattern 1: Iterative Formalization

**Scenario:** Request is too vague initially.

```bash
# First attempt
/formalize improve performance

# Output: Clarification questions
# Answer them in refined request:

/formalize Improve API response time: reduce p95 latency from 500ms to 200ms for /api/users endpoint

# Output: Concrete, measurable task spec
```

### Pattern 2: Context Layering

**Scenario:** Large monorepo, need focused analysis.

```bash
# Layer 1: Top-level structure
/build-context --shallow

# Layer 2: Specific workspace
/build-context packages/backend/

# Layer 3: Specific module
/build-context packages/backend/src/auth/ --deep

# Layer 4: Task focus
/task-context Add JWT refresh token support
```

### Pattern 3: Branch-Based Context

**Scenario:** Working on feature branch.

```bash
# Switch to feature branch
git checkout feature/add-stripe-integration

# Create branch-specific context
/task-context Integrate Stripe payment gateway

# Output auto-named: .claude/context/branch-feature-add-stripe-integration.md

# Benefits:
# - Context persists with branch
# - Git checkout automatically switches context
# - Multiple features tracked separately
```

### Pattern 4: Confidence Tuning

**Scenario:** Fine-tune formalization sensitivity.

The hook uses confidence < 0.6 as threshold. To adjust:

**Option A: Edit hook directly**
```json
// plugins/toolkit/hooks/hooks.json
// Change: "If confidence < 0.6" → "If confidence < 0.5"
```

**Option B: Use explicit /formalize when needed**
```bash
# For ambiguous requests where you want formalization
/formalize <request>

# For clear requests where hook might over-trigger
just do it: <request>
```

---

## Integration with Other Tools

### With knowledge-base Plugin

**Workflow:** Discover and document coding conventions.

```bash
# Step 1: Analyze codebase
/build-context src/ --deep

# Step 2: Extract patterns from context
cat .claude/context/src.md | grep "Patterns\|Conventions"

# Step 3: Formalize as knowledge entry
/knowledge-add Naming conventions: functions are camelCase, classes are PascalCase, constants are UPPER_SNAKE_CASE
```

### With Git Hooks

**Workflow:** Auto-update context on branch switch.

Create `.git/hooks/post-checkout`:
```bash
#!/bin/bash
BRANCH=$(git branch --show-current)

if [[ $BRANCH != "main" ]]; then
  echo "Updating context for branch: $BRANCH"
  # Auto-create task context (requires claude CLI)
  # claude "/task-context Work on $BRANCH"
fi
```

### With CI/CD

**Workflow:** Generate architecture docs on deploy.

```yaml
# .github/workflows/docs.yml
- name: Generate architecture docs
  run: |
    claude "/build-context --output docs/architecture.md"
    git add docs/architecture.md
    git commit -m "Update architecture docs"
```

---

## Best Practices

### ✅ Do

1. **Use scoped analysis for large projects**
   ```bash
   /build-context src/auth/  # Not entire project
   ```

2. **Review auto-formalized specs before proceeding**
   - Check assumptions marked [ASSUMED]
   - Verify inferences marked [INFERRED]

3. **Combine context commands**
   ```bash
   /build-context src/payments/
   /task-context Add Stripe integration
   # Two complementary contexts
   ```

4. **Leverage branch naming**
   ```bash
   git checkout feature/add-oauth
   /task-context Add OAuth support
   # Auto-detects branch, names output accordingly
   ```

5. **Use --shallow for quick scans**
   ```bash
   /build-context --shallow  # Fast overview
   ```

### ❌ Don't

1. **Don't analyze everything at once**
   ```bash
   # Bad: Slow, overwhelming
   /build-context

   # Good: Focused, fast
   /build-context src/module-im-working-on/
   ```

2. **Don't ignore formalization output**
   - If confidence is low, there's ambiguity
   - Resolve before implementing

3. **Don't mix task and general context**
   ```bash
   # Bad: Conflates purposes
   /build-context "fix bug X"

   # Good: Use task context
   /task-context Fix bug X
   ```

4. **Don't skip explicit formalization for complex requests**
   ```bash
   # If hook doesn't trigger but request is complex:
   /formalize <complex request>
   ```

---

## Troubleshooting

### Issue: Hook Not Triggering

**Symptoms:** Vague requests proceed without formalization.

**Solutions:**
1. Verify hook file exists:
   ```bash
   cat .claude/plugins/toolkit/hooks/hooks.json
   ```

2. Check plugin is loaded:
   ```bash
   /plugin list | grep toolkit
   ```

3. Use explicit formalization:
   ```bash
   /formalize <your request>
   ```

### Issue: Context File Too Large

**Symptoms:** Analysis takes very long, output exceeds limits.

**Solutions:**
1. Use scoped analysis:
   ```bash
   /build-context src/specific-module/
   ```

2. Use --shallow mode:
   ```bash
   /build-context --shallow
   ```

3. Split into multiple analyses:
   ```bash
   /build-context src/auth/
   /build-context src/api/
   # Separate files, easier to manage
   ```

### Issue: Formalization Too Aggressive

**Symptoms:** Simple requests get over-formalized.

**Solutions:**
1. Prefix with "just do it":
   ```
   just do it: run tests
   ```

2. Edit hook threshold (increase from 0.6 to 0.5):
   ```json
   "If confidence < 0.5"  // Was 0.6
   ```

3. Use explicit phrasing:
   ```
   # Clear, won't trigger formalization:
   "Execute pytest in packages/auth/"

   # Vague, will trigger:
   "test the auth stuff"
   ```

---

## Real-World Examples

### Example 1: E-commerce Feature

**Initial request:**
```
Add shopping cart with checkout flow, similar to Amazon
```

**Auto-formalized output:**
```markdown
> ⚠️ Need clarification:
> 1. Which aspects of Amazon checkout? (one-page? guest checkout? address validation?)
> 2. Payment integration requirements?
> 3. Scope: cart UI, backend API, or both?

## Task 1: Shopping cart data model
[Details...]

## Task 2: Cart UI components
[Dependencies: Task 1]

## Task 3: Checkout flow API
[Dependencies: Task 1]

## Task 4: Payment integration
[Dependencies: Task 3]
```

**Context building:**
```bash
/build-context src/products/  # Understand existing product system
/task-context Implement shopping cart and checkout
```

### Example 2: Performance Optimization

**Initial request:**
```
Make the dashboard faster
```

**Auto-formalized output:**
```markdown
> ⚠️ Need clarification:
> 1. Current performance metrics? (load time, p95 latency)
> 2. Target metrics?
> 3. Which dashboard? (user dashboard, admin dashboard)

## Task: Dashboard performance optimization
**Objective**: Improve dashboard load time
**Requirements**:
- Profile current performance
- Identify bottlenecks
- Implement optimizations (specific to findings)
- Measure improvements
**Done When**: Load time reduced by [TARGET]% and p95 < [TARGET]ms
**Ambiguities**:
- [ASSUMED] User dashboard (not admin)
- [ASSUMED] Focus on initial load, not interactions
```

**Workflow:**
```bash
/build-context src/dashboard/ --deep
/task-context Optimize dashboard performance
# Context reveals API calls, rendering logic, data fetching patterns
```

---

## Summary

The toolkit plugin provides two main capabilities:

1. **Automatic Formalization**: Transparent quality analysis and structuring of vague requests
2. **Context Building**: Architectural analysis and task-focused file identification

**Key Commands:**
- `/formalize` — Explicit task structuring
- `/build-context` — Architecture documentation
- `/task-context` — Task-scoped file discovery

**Key Hook:**
- **UserPromptSubmit** — Automatic quality gate with confidence-based routing

Use together for maximum productivity: let the hook handle routine formalization, use explicit commands for complex analysis, and leverage memory to improve accuracy over time.
