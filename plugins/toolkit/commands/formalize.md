---
name: formalize
description: Transform vague or complex requests into clear, structured task specifications
argument-hint: "<task description or question>"
---

# Formalize Command

Normalize informal, vague, or multi-objective user input into a structured task specification that can be executed without ambiguity.

## Usage

```bash
/formalize fix the auth bug and update docs
/formalize clean up the code and make it work like the other service
/formalize We need better error handling, maybe use try-catch everywhere?
```

## When to Use

Use `/formalize` when your request:
- Contains multiple objectives bundled together
- Has vague scope ("improve", "clean up", "refactor")
- Lacks clear definition of done
- Has implicit dependencies between steps
- References code/features without context

## How It Works

1. **Parses** your input to identify objectives, context, dependencies
2. **Evaluates** clarity with a confidence score (0.0-1.0)
3. **Structures** into clear tasks with:
   - Objective (one sentence, action-oriented)
   - Requirements (specific, verifiable)
   - Definition of Done (completion criteria)
   - Dependencies (what must happen first)
   - Ambiguities (assumptions made, flagged)
4. **Routes** based on confidence:
   - ≥ 0.8 (high) → Returns structured spec silently
   - 0.5-0.79 (medium) → Shows interpretation summary
   - < 0.5 (low) → Asks clarifying questions

## Implementation

Invoke the `formalizer` agent with user arguments:

```python
Task(
  description="Formalize task request",
  prompt=f"Formalize this user input: {arguments}",
  subagent_type="toolkit:formalizer"
)
```

The agent uses the `formalize` skill which:
- Parses multi-task bundles
- Scores confidence based on clarity factors
- Generates structured specifications
- Flags ambiguities with [ASSUMED] or [INFERRED] markers

## Output Examples

**Simple task (confidence: 0.95)**
```markdown
**Task**: Run tests — Execute test suite in packages/playwright-integration
**DoD**: All tests execute, results reported
```

**Multi-task with dependencies (confidence: 0.72)**
```markdown
## Task 1: Fix authorization bug
**Objective**: Identify and fix the bug in the authorization module
**Requirements**:
- Locate the failing auth flow
- Implement fix
- Add/update tests for the fixed behavior
**Done When**: Auth tests pass, no regression in related tests

## Task 2: Update API documentation (parallel with Task 1)
**Objective**: Update API docs to reflect current state
**Done When**: Documentation matches current API behavior

## Task 3: Deploy to staging (depends on: Task 1 tests passing)
**Objective**: Deploy current branch to staging environment
**Constraints**: Only proceed if all tests pass
**Done When**: Staging deployment successful, smoke tests pass

Dependency graph: [Task 1] → [Task 3]
                  [Task 2] → (parallel)
```

**Vague input requiring clarification (confidence: 0.35)**
```markdown
> ⚠️ Need clarification:
> 1. Which code/module needs cleanup?
> 2. Which "other service" are you referring to?
> 3. What aspects should match? (architecture? API design? code style?)
>
> Current best interpretation:

## Task: Code cleanup
**Objective**: Refactor code to match patterns from [UNKNOWN service]
**Requirements**: [Cannot determine without answers above]
**Done When**: [Cannot determine]
```

## Error Handling

**No arguments provided:**
```
Usage: /formalize <task description>

Examples:
  /formalize fix bug and update docs
  /formalize clean up the authentication code
  /formalize make it work like the checkout flow

Transforms vague requests into structured specifications with clear objectives,
requirements, and definition of done.
```

## Notes

- The formalizer agent has access to Read/Grep/Glob to resolve references to existing code
- Output is optimized for execution by another agent or human developer
- Preserves user's technical terminology and intent
- Marks all assumptions and inferences explicitly
- For very simple requests ("run tests"), returns minimal formatting

## See Also

- `/build-context` — Analyze codebase structure before formalizing tasks
- `skills/formalize/examples.md` — More formalization examples
- `skills/formalize/CLAUDE-md-snippet.md` — Integration instructions for CLAUDE.md
