---
name: formalize
description: Normalize and structure informal task descriptions into clear,
  executable specifications. Splits compound requests into ordered tasks
  with dependency graphs. Scores confidence and routes to appropriate
  output mode (silent, summary, or clarify).
allowed-tools: Read, Grep, Glob
disable-model-invocation: true
context: fork
---

# Task Formalization Pipeline

Analyze and formalize the following raw input into a structured task
specification.


## Raw Input

$ARGUMENTS

If the input references files or code in the current project,
use Read/Grep/Glob to gather context before formalizing.

For examples of formalized output, see [examples.md](examples.md).


## Step 1: Parse

Analyze the raw message and identify:

1. **Distinct objectives** — separate tasks bundled in one message
2. **Implicit context** — assumptions the user didn't state
3. **Dependencies** — ordering constraints between tasks
4. **Scope boundaries** — what's in scope vs. out of scope
5. **Success criteria** — how to know when the task is done

Common patterns to detect:

| Pattern        | Example                              | Action                    |
| -------------- | ------------------------------------ | ------------------------- |
| Multi-task     | "Fix bug and update docs and deploy" | Split into ordered tasks  |
| Implicit ref   | "Make it work like the other one"    | Flag for clarification    |
| Vague scope    | "Clean up the code"                  | Request boundaries        |
| Missing DoD    | "Add authentication"                 | Infer or ask for criteria |
| Mixed concerns | Context + requirements interleaved   | Separate into sections    |


## Step 2: Decompose

If the input contains multiple tasks:

1. Extract each distinct task
2. Identify dependencies between them
3. Determine execution order
4. Flag tasks that can run in parallel


## Step 3: Formalize

Transform each task into this structure:

```markdown
## Task: [Clear, action-oriented title]

**Objective**: [One sentence — what needs to be done]

**Context**: [Why this task exists]

**Requirements**:

- [Specific, verifiable requirement]

**Constraints**: [Limitations, boundaries]

**Dependencies**: [What must be true/done first]

**Definition of Done**:

- [ ] [Concrete criterion]

**Ambiguities**: [Unclear items with suggested defaults]
```

For simple tasks, use compact format:

```markdown
**Task**: [Title] — [Objective in one line]
**DoD**: [Completion criteria]
```


## Step 4: Confidence Evaluation

Score 0.0–1.0 based on:

| Factor       | High (+0.2)             | Low (−0.2)              |
| ------------ | ----------------------- | ----------------------- |
| Goal clarity | Single, clear objective | Vague or multiple goals |
| Requirements | Explicit, testable      | Implied, subjective     |
| Scope        | Well-bounded            | Open-ended              |
| Terminology  | Specific, unambiguous   | Jargon, no referent     |
| Dependencies | Stated or none          | Implied, circular       |


## Step 5: Output

Format:

```markdown
CONFIDENCE: [0.0-1.0]
MODE: [silent|summary|clarify]
TASKS_COUNT: [number]
---
[Formalized tasks in markdown]
```

**≥ 0.8 — Silent**: Return formalized task only. No preamble.

**0.5–0.79 — Summary**:

```markdown
> Interpreting as: [1-2 sentence summary]

[Formalized task]
```

**< 0.5 — Clarify**:

```markdown
> ⚠️ Need clarification:
> 1. [Specific question]
>
> Current best interpretation:

[Formalized task with [ASSUMED: ...] markers]
```


## Principles

- **Preserve intent** — never add scope the user didn't imply
- **Make implicit explicit** — surface hidden assumptions
- **Be concrete** — replace "improve" with specific metrics
- **Maintain voice** — keep the user's technical terms
- **Mark inferences** — label as [INFERRED] or [ASSUMED]
- **Minimal for clear input** — don't over-formalize "run tests"
- **Respect expertise** — don't dumb down technical specs


## Special Cases

**Single clear task** (confidence ≥ 0.9): Return as-is with minimal formatting.

**Planning requests**: Formalize the planning scope, not implementation steps.

**Follow-up messages**: Use Read/Grep to check recent files for context.


## Anti-Patterns

- Adding scope the user didn't request
- Splitting a naturally atomic task
- Bureaucratic formatting for simple requests
- Losing the user's technical specificity
- Asking questions that can be reasonably inferred

ultrathink

