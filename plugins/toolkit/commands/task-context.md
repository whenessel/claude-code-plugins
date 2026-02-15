---
name: task-context
description: Create a focused context document for a specific task, feature, or issue
argument-hint: "<task or feature description> [--output <path>]"
---

# Task Context Command

Create a laser-focused context document scoped to a specific task, feature branch, or issue. Unlike `/build-context` which analyzes the entire project, this command identifies only the files and APIs relevant to completing a particular task.

## Usage

```bash
/task-context Add user authentication with JWT
/task-context Fix memory leak in recording service
/task-context Implement dark mode toggle
/task-context JIRA-123 --output .claude/context/jira-123.md
```

## When to Use

Use `/task-context` when:
- Starting work on a specific feature or bug fix
- Working on a feature branch
- Investigating a specific issue or ticket
- Need to know "what files do I need to touch for X"
- Want focused context without full project analysis

## Arguments

- **task description** (required): The specific task, feature, or issue
  - Natural language description
  - Issue/ticket ID (JIRA-123, #456)
  - Feature branch name

- **--output <path>**: Custom output location (auto-detected by default)

## How It Works

The command invokes the `context-builder` agent with task-scoping mode:

1. **Auto-detect branch**: Gets current git branch name if applicable
2. **Identify relevant files**: Searches for code related to the task
3. **Extract key APIs**: Documents only the interfaces this task touches
4. **Analyze dependencies**: What could break, what depends on this
5. **Write focused doc**: Task-specific context with actionable guidance

## Implementation

```python
# Get current branch
branch = bash("git branch --show-current 2>/dev/null || echo 'unknown'")

# Determine output path
if "--output" in arguments:
    output_path = extract_output_path(arguments)
elif branch != "unknown" and branch != "main":
    output_path = f".claude/context/branch-{branch}.md"
else:
    task_id = extract_task_id(arguments)  # JIRA-123, #456, etc.
    if task_id:
        output_path = f".claude/context/task-{task_id}.md"
    else:
        slug = slugify(arguments)
        output_path = f".claude/context/{slug}.md"

# Invoke agent with task scope
Task(
  description="Build task-scoped context",
  prompt=f"""
  Create a focused context document for this specific task:

  Task: {arguments}
  Branch: {branch}
  Output: {output_path}

  Use the build-task-context skill to identify relevant files, APIs,
  and dependencies. Focus ONLY on what's needed for this specific task.
  Make the output actionable — tell the reader exactly which files to modify,
  which APIs to use, and what risks exist.
  """,
  subagent_type="toolkit:context-builder"
)
```

## Output Structure

The generated task context follows this format:

```markdown
# Task Context: [task title]

> Branch: [branch name]
> Created: [date]
> Status: active

## Task Summary

[1-2 sentences: what needs to happen]

## Relevant Files

### Files to Modify

- `src/auth/login.ts` (lines 42-68) — Add JWT token generation
- `src/middleware/auth.ts` (new file) — Create JWT verification middleware

### Related Files

- `src/types/user.ts` (line 12) — User interface, may need token field
- `config/auth.ts` — JWT secret configuration

## Key APIs & Interfaces

### Existing APIs to Use

- `UserService.authenticate(credentials)` → User
- `TokenService.sign(payload, secret)` → string

### New APIs to Create

- `AuthMiddleware.verifyToken(req, res, next)` → void
- `TokenService.verify(token)` → TokenPayload | null

## Risks & Dependencies

### Breaking Changes
- Adding `token` field to User type may affect existing serialization

### Dependencies
- Requires `jsonwebtoken` package (not currently installed)
- Database migration may be needed for token storage

### Testing Requirements
- Unit tests for token generation/verification
- Integration tests for protected routes

## Notes

[Decisions made, approaches considered, gotchas found]
```

## Output Path Logic

The command automatically determines the output path:

1. **Explicit flag**: If `--output <path>` provided, use that
2. **Feature branch**: If on a non-main branch → `.claude/context/branch-<name>.md`
3. **Task ID detected**: If input contains "JIRA-123" or "#456" → `.claude/context/task-<id>.md`
4. **Default**: Slugified task description → `.claude/context/<slug>.md`

## Examples

**Feature branch context:**
```bash
# On branch: feature/add-authentication
/task-context Implement JWT authentication
# Output: .claude/context/branch-feature-add-authentication.md
```

**Issue-specific context:**
```bash
/task-context Fix JIRA-456 memory leak in recording service
# Output: .claude/context/task-JIRA-456.md
```

**Ad-hoc task context:**
```bash
/task-context Add dark mode toggle to settings page
# Output: .claude/context/add-dark-mode-toggle-to-settings-page.md
```

**Custom output path:**
```bash
/task-context Refactor payment gateway --output docs/tasks/payment-refactor.md
```

## Comparison: task-context vs build-context

| Feature | `/task-context` | `/build-context` |
|---------|----------------|------------------|
| **Scope** | Single task/feature | Full project or module |
| **Files analyzed** | Only task-relevant | All files in scope |
| **API detail** | Only touched APIs | All public APIs |
| **Output focus** | "What to change" | "How it's structured" |
| **Speed** | Fast (focused) | Slower (comprehensive) |
| **Use case** | Starting a task | Onboarding/planning |

## Error Handling

**No task description:**
```
Usage: /task-context <task description> [--output <path>]

Examples:
  /task-context Add user authentication
  /task-context Fix JIRA-123 memory leak
  /task-context Implement dark mode toggle

Description:
  Create a focused context document for a specific task or feature.
  Identifies relevant files, APIs, and dependencies needed to complete the task.
```

**Cannot identify relevant files:**
```
Warning: Could not identify files clearly related to this task.

Suggestions:
- Use more specific terminology from the codebase
- Reference existing modules or features
- Try /build-context first to understand project structure
- Provide file paths if known: /task-context Fix bug in src/auth/login.ts
```

## See Also

- `/build-context` — Analyze full project or module structure
- `/formalize` — Structure vague task descriptions before creating context
- `skills/build-task-context/SKILL.md` — Skill implementation details
