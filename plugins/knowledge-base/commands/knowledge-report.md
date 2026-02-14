---
name: knowledge-report
description: Evaluate knowledge entry quality with detailed analysis and recommendations
argument-hint: "[file-path or entry-name]"
---

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

```python
def parse_arguments(args: str) -> dict:
    """Parse command arguments."""

    if not args or args.strip() == '':
        # No arguments - find most recent knowledge entry
        return {
            "mode": "most_recent"
        }

    # Clean input
    file_path = args.strip()

    # Remove quotes if present
    file_path = file_path.strip('"\'')

    return {
        "mode": "explicit",
        "file_path": file_path
    }
```

### Step 2: Resolve File Path

```python
def resolve_file_path(mode: str, file_path: str = None) -> str:
    """Resolve file path based on mode."""

    if mode == "most_recent":
        # Find most recently modified knowledge entry
        knowledge entrys = Glob(pattern="**/*.md", path=".knowledge entrys")

        if not knowledge entrys:
            return None

        # Get most recent
        most_recent = max(knowledge entrys, key=lambda f: os.path.getmtime(f))
        return most_recent

    else:
        # Explicit path provided - will be resolved by evaluator agent
        return file_path
```

### Step 3: Invoke Convention Evaluator Agent

```python
def execute_evaluation(file_path: str):
    """Invoke evaluator agent for deep analysis."""

    # Invoke knowledge-evaluator agent
    result = invoke_agent(
        agent="knowledge-evaluator",
        params={
            "file_path": file_path
        }
    )

    return result
```

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
