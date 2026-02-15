---
name: knowledge-report
description: Evaluate knowledge entry quality with detailed analysis and recommendations
argument-hint: "[file-path or entry-name]"
---

> **EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Knowledge Quality Report Command

Generate comprehensive quality evaluation reports for knowledge entries.

## Usage

```bash
/knowledge-report [file-path or knowledge entry-name]
```

**Arguments**:
- `file-path`: Path to knowledge entry file (absolute, relative, or name only)
- If omitted: Evaluates most recently created/modified knowledge entry

**Examples**:
```bash
/knowledge-report typescript/naming/functions.md
/knowledge-report functions.md
/knowledge-report functions
/knowledge-report
```

## Implementation

### Step 1: Parse Arguments

Parse the command arguments as follows:

- If no arguments are provided (empty or whitespace only), set mode to `"most_recent"` to find the most recently modified knowledge entry
- Otherwise, trim whitespace and remove surrounding quotes (single or double) from the input to get the file path, and set mode to `"explicit"`

### Step 2: Resolve File Path

Resolve the file path based on mode:

- **most_recent mode**: Use **Glob** with pattern `**/*.md` in the `.kb` directory. Glob returns files sorted by modification time, so take the first result as the most recently modified entry. If no files are found, report that no knowledge entries exist.
- **explicit mode**: Pass the provided file path directly to the evaluator agent (it handles its own path resolution).

### Step 3: Invoke Convention Evaluator Agent

Use the **Task** tool to delegate to the `knowledge-evaluator` agent, passing the resolved file path. The agent performs deep analysis and returns a formatted report.

### Step 4: Display Results

The evaluator agent returns a formatted report. Display it directly to the user.

## Error Handling

**No Arguments + No Conventions**:
```
❌ No knowledge entrys found in .knowledge entrys/

Create your first knowledge entry with:
/knowledge entry "your coding guidelines here"
```

**Invalid File Path**:
```
❌ Convention not found: invalid-path.md

Searched:
- .knowledge entrys/invalid-path.md
- .knowledge entrys/**/invalid-path.md

List available knowledge entrys:
ls .knowledge entrys/**/*.md
```

**Evaluation Failure**:
```
❌ Failed to evaluate knowledge entry

This may indicate a malformed knowledge entry file.
Check YAML frontmatter and section structure.
```

## Workflow Example

```
User: /knowledge-report functions

1. Parse arguments → file_path = "functions"
2. Invoke knowledge-evaluator agent
3. Agent resolves "functions" → .knowledge entrys/typescript/naming/functions.md
4. Agent reads file and validates
5. Agent invokes knowledge-evaluate skill (deep mode)
6. Agent generates detailed report
7. Display report to user

Output:
# Convention Quality Report

**File**: .knowledge entrys/typescript/naming/functions.md
**Version**: 1.0
**Scope**: typescript | **Category**: naming
...

[Full detailed report]
```

## Integration

This command delegates all processing to the `knowledge-evaluator` agent, which:
- Handles file path resolution
- Validates knowledge entry files
- Invokes the evaluation skill
- Generates detailed reports
- Updates project memory

The command itself is a thin wrapper for user-friendly invocation.

## Notes

- Always runs in **deep mode** for detailed analysis
- Takes 20-30 seconds to complete
- Results are stored in project memory
- Can be run repeatedly without side effects (read-only)
