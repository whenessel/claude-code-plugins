---
name: build-task-context
description: >
  Create a context file scoped to a specific task, feature branch, or issue.
  Focuses only on files and APIs relevant to the task at hand.
  Output goes to .claude/context/ with branch or task ID in the filename.
allowed-tools: Read, Grep, Glob, Bash, Write
disable-model-invocation: true
---

<!-- prettier-ignore-start -->

# Task-Scoped Context Analysis

Create a focused context file for a specific task or feature.


## Task

$ARGUMENTS


## Scope Rules

This is a **task-scoped** build. Focus ONLY on files and modules relevant
to this specific task. Do not produce a full project overview.


## Output Location

Determine the output path (in priority order):

1. If `--output <path>` was specified, use that
2. If on a feature branch: `.claude/context/branch-<branch-name>.md`
3. If a task/issue ID is mentioned: `.claude/context/task-<id>.md`
4. Otherwise: `.claude/context/<slugified-task-description>.md`

Get the branch name:

```bash
git branch --show-current 2>/dev/null || echo "unknown"
```


## Output Format

```markdown
# Task Context: [task title]

> Branch: [branch name]
> Created: [date]
> Status: active

## Task Summary

[1-2 sentences: what needs to happen]

## Relevant Files

[files that need changes, with line numbers for key locations]

## Key APIs & Interfaces

[only the APIs this task touches]

## Risks & Dependencies

[what could break, what depends on this]

## Notes

[decisions made, approaches considered, gotchas found]
```


## Emphasis

The document must be actionable — tell the reader exactly:

- Which files to modify and why
- Which interfaces are relevant
- What constraints and risks exist
- What dependencies might be affected

ultrathink

<!-- prettier-ignore-end -->
