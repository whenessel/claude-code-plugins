# Task Formalization — CLAUDE.md Integration

Add this section to your `CLAUDE.md` (project-level) or `~/.claude/CLAUDE.md`
(user-level). Handles simple cases inline, delegates complex ones to the
task-formalizer subagent automatically.

---

## ✂️ Copy below this line into your CLAUDE.md ✂️

### Task Formalization Protocol

Before executing any non-trivial task, apply this preprocessing.

#### When to Formalize

Formalize when ANY of these are true:
- Message contains 2+ distinct objectives ("fix X and also Y")
- Vague scope without boundaries ("clean up", "improve", "refactor")
- Implicit dependencies between steps
- Missing definition of done
- Mixed context/requirements/examples in one message
- Unresolved references ("make it like the other one")

Skip when:
- Single, clear, atomic action ("run tests", "format src/utils.ts")
- Input is a question, not a task
- User explicitly says "just do it"

#### Inline Formalization (simple cases)

For inputs with 1-2 tasks and confidence ≥ 0.5:

1. **Parse**: Identify objectives, context, dependencies, scope
2. **Structure** each task as:
   - **Objective**: One sentence
   - **Requirements**: Specific, verifiable items
   - **Definition of Done**: How to verify completion
3. **Confidence check**:
   - ≥ 0.8 → Proceed silently with formalized version
   - 0.5–0.79 → Show 1-line interpretation, then proceed

Do this inline — no subagent needed.

#### Delegate to task-formalizer subagent (complex cases)

Delegate to the task-formalizer subagent when ANY of these are true:
- 3+ distinct tasks bundled in one message
- Confidence < 0.5 (too ambiguous to interpret alone)
- Complex dependency graph between tasks
- Input mixes multiple languages or contexts that need untangling
- Formalization requires reading project files to resolve references

The subagent runs in its own context and returns a clean specification.

#### Principles

- Preserve intent — don't add scope
- Make implicit explicit — mark as [INFERRED]
- Be concrete — replace "improve" with specifics
- Minimal for clear inputs — don't over-formalize
