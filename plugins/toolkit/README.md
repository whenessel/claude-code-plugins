# Toolkit

**Essential productivity utilities for Claude Code.**

A collection of intelligent tools that enhance your development workflow through automated quality improvements, smart analysis, and workflow optimization. Designed to be extensible — more utilities added over time.

## Current Utilities

This plugin bundles multiple productivity utilities. More tools will be added in future releases.

### 🎯 Task Formalization

Automatically structures vague or complex requests into clear, executable specifications.

**Key capabilities:**
- Auto-detects low-quality prompts and reformulates them
- Decomposes multi-objective requests into ordered tasks
- Identifies dependencies and creates execution graphs
- Works transparently via UserPromptSubmit hook

**Commands:** `/formalize`

### 📚 Context Building

Analyzes codebase architecture and generates structured documentation.

**Key capabilities:**
- Maps architectural layers and module boundaries
- Extracts public APIs with signatures and line numbers
- Visualizes dependency graphs and data flow
- Caches patterns with project-level memory

**Commands:** `/build-context`, `/task-context`

### 🔄 Memory-Backed Learning

Agents use project-level memory to:
- Cache architectural patterns across sessions
- Avoid re-analyzing unchanged code
- Build cumulative understanding of the codebase
- Improve accuracy over time

## Installation

### From Marketplace (Recommended)

```bash
/plugin marketplace add whenessel/claude-code-plugins
```

Then select the `toolkit` plugin from the list.

### Manual Installation

1. Clone the repository:
```bash
git clone https://github.com/whenessel/claude-code-plugins.git
cd claude-code-plugins/plugins/toolkit
```

2. Copy to your project:
```bash
cp -r . ~/.claude/plugins/toolkit/
```

Or for project-level installation:
```bash
cp -r . ./.claude/plugins/toolkit/
```

## Commands

### `/formalize <task description>`

Transform vague or complex requests into structured task specifications.

**Use when:**
- Request has multiple objectives
- Scope is unclear ("improve", "clean up")
- Missing definition of done
- Implicit dependencies

**Example:**
```bash
/formalize fix auth bug and update docs and deploy if tests pass
```

**Output:**
```markdown
## Task 1: Fix authorization bug
**Objective**: Identify and fix auth bug
**Requirements**: Locate failing flow, implement fix, add tests
**Done When**: Auth tests pass, no regressions

## Task 2: Update API documentation (parallel)
**Done When**: Docs match current behavior

## Task 3: Deploy to staging (depends on: Task 1)
**Constraints**: Only if tests pass
**Done When**: Deployment successful, smoke tests pass

Dependency graph: [Task 1] → [Task 3]
                  [Task 2] → (parallel)
```

### `/build-context [scope] [--deep|--shallow]`

Analyze project architecture and generate structured context documentation.

**Use when:**
- Starting in unfamiliar codebase
- Planning major refactoring
- Onboarding to new project area
- Documenting architecture

**Arguments:**
- `scope` — Focus on specific area (e.g., `src/auth/`, `the payment layer`)
- `--deep` — Full API signatures with line numbers
- `--shallow` — Module names and paths only (fast)
- `--output <path>` — Custom output location

**Example:**
```bash
/build-context src/auth/ --deep
```

**Output:** `.claude/context/auth.md` with layers, APIs, dependencies

### `/task-context <task description>`

Create focused context for a specific task, feature, or issue.

**Use when:**
- Starting work on a feature/bug
- Need to know "what files to touch"
- Working on a feature branch
- Investigating specific issue

**Auto-detection:**
- Detects current git branch
- Extracts issue IDs (JIRA-123, #456)
- Auto-names output file

**Example:**
```bash
/task-context Add JWT authentication
```

**Output:** `.claude/context/branch-feature-auth.md` with:
- Files to modify (with line numbers)
- Relevant APIs to use
- Risks and dependencies
- Testing requirements

## How Automatic Formalization Works

### UserPromptSubmit Hook

The plugin includes a `UserPromptSubmit` hook that evaluates every user request:

```
User Request
    ↓
Quality Evaluation (0.0-1.0 confidence score)
    ↓
Decision:
  - confidence ≥ 0.6 → Proceed directly
  - confidence < 0.6 → Invoke formalizer agent
    ↓
  Structured Task Spec
    ↓
  Execute Implementation
```

### Confidence Scoring

| Factor | High (+0.2) | Low (−0.2) |
|--------|-------------|------------|
| Goal clarity | Single, clear objective | Vague or multiple goals |
| Requirements | Explicit, testable | Implied, subjective |
| Scope | Well-bounded | Open-ended ("improve") |
| Dependencies | Stated or none | Implicit, circular |
| Definition of Done | Clear criteria | Missing or ambiguous |

### Skip Formalization

Automatically skipped when:
- User asks a question (not requesting action)
- Single atomic action ("run tests in src/")
- User says "just do it" or "no planning"
- Request already has clear DoD

## Architecture

### Components

```
toolkit/
├── commands/
│   ├── formalize.md         # Explicit formalization command
│   ├── build-context.md     # Project analysis command
│   └── task-context.md      # Task-scoped context command
├── agents/
│   ├── formalizer.md        # Task normalization engine
│   └── context-builder.md   # Codebase analyzer
├── skills/
│   ├── formalize/
│   │   ├── SKILL.md         # Formalization pipeline
│   │   ├── examples.md      # Real-world examples
│   │   └── CLAUDE-md-snippet.md  # Manual integration guide
│   ├── build-context/
│   │   └── SKILL.md         # Context analysis logic
│   └── build-task-context/
│       └── SKILL.md         # Task-scoped analysis
└── hooks/
    └── hooks.json           # UserPromptSubmit hook for auto-formalization
```

### Agent Routing

**formalizer agent:**
- Uses `formalize` skill for all requests
- Scores confidence, structures tasks
- Returns clean specifications

**context-builder agent:**
- Routes to `build-context` for general analysis
- Routes to `build-task-context` for task-specific focus
- Uses project memory to cache patterns

## Configuration

### Optional: Manual Integration

If you prefer explicit control over formalization, add the instructions from `skills/formalize/CLAUDE-md-snippet.md` to your `CLAUDE.md`:

```markdown
## Task Formalization Protocol

Before executing any non-trivial task, apply this preprocessing...
[see CLAUDE-md-snippet.md for full content]
```

**Note:** This is **not required** — the UserPromptSubmit hook handles formalization automatically.

### Memory Location

Project-level memory is stored in:
```
.claude/memory/toolkit/
├── formalizer/
│   └── [project-specific patterns]
└── context-builder/
    └── [architectural cache]
```

## Examples

### Example 1: Auto-formalization

```
User: "fix the login bug and also update the readme"

[Hook detects multi-objective, confidence: 0.55]
[Auto-invokes formalizer agent]

Output:
## Task 1: Fix login bug
**Objective**: Debug and fix authentication issue
**Requirements**: Locate bug, implement fix, add test
**Done When**: Login works, test passes

## Task 2: Update README (parallel)
**Objective**: Update documentation
**Done When**: README reflects current state
```

### Example 2: Context before feature

```bash
# First, understand the auth architecture
/build-context src/auth/ --deep

# Then, create task-specific context
/task-context Add OAuth 2.0 support

# Now implement with full context
```

### Example 3: Explicit formalization

```bash
/formalize We need better error handling, maybe try-catch everywhere?

# Output includes:
# - Clarifying questions
# - Current best interpretation
# - Marked assumptions [ASSUMED: ...]
```

## Use Cases

### 🚀 Starting in Unfamiliar Codebase

```bash
/build-context
# → Full project overview with layers, APIs, dependencies

/task-context Implement feature X
# → Focused context for your specific task
```

### 🎯 Clarifying Vague Requests

Automatic (via hook):
```
User: "make it faster"
→ Auto-formalized into specific performance targets
```

Manual (explicit command):
```bash
/formalize improve performance
→ Structured spec with metrics, constraints, DoD
```

### 🔍 Feature Development Workflow

```bash
# 1. Analyze relevant code
/build-context src/payments/

# 2. Create task context
/task-context Add Stripe integration

# 3. Work with full context loaded
# Context files are in .claude/context/
```

### 📋 Multi-task Decomposition

```bash
/formalize Implement user authentication: login, signup, password reset, email verification

# → Returns 4 separate tasks with:
#   - Dependencies (signup before login)
#   - Parallel opportunities (email verification)
#   - Shared components (email service)
```

## Limitations

- **Large codebases**: Analysis may be slow for 1000+ files. Use scoped analysis (`/build-context src/module/`)
- **Non-standard structures**: Works best with conventional project layouts
- **Context window**: Very large projects may exceed context limits. Use `--shallow` or focus on specific modules
- **Formalization accuracy**: Complex domain-specific requests may need manual refinement

## Tips

1. **Scope your analysis**: Use `/build-context src/specific-area/` instead of analyzing everything
2. **Use task context for features**: `/task-context` is faster than `/build-context` when you know what you're building
3. **Check formalization output**: Review auto-formalized specs before proceeding with complex tasks
4. **Leverage memory**: The agents get better over time as they cache architectural patterns
5. **Combine with other plugins**: Use with `knowledge-base` plugin to document discovered conventions

## Troubleshooting

**"Cannot detect project structure"**
- Ensure you're in a valid project root with `package.json`, `Cargo.toml`, etc.
- Try scoping to a specific directory: `/build-context src/`

**"Formalization confidence too low"**
- Provide more context in your request
- Reference specific files or features
- Use `/build-context` first to help the agent understand your codebase

**"Too many files to analyze"**
- Use `--shallow` for quick overview
- Scope to specific module: `/build-context src/auth/`
- Split into multiple focused analyses

## Contributing

Found a bug or have a feature request? Please open an issue at:
https://github.com/whenessel/claude-code-plugins/issues

## License

MIT License - see [LICENSE](../../LICENSE) file for details

## Author

**Artem Demidenko**

## Related Plugins

- **knowledge-base**: Build AI-driven knowledge entries from coding guidelines
- **feature-dev**: Guided feature development with architecture focus
- **ralph-loop**: Autonomous task execution loop

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and updates.
