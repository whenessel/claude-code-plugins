# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **Claude Code plugins marketplace** repository containing custom plugins that extend Claude Code with skills, commands, agents, and hooks. The repository follows the standard Claude Code marketplace structure with individual plugins in the `plugins/` directory.

**Available Plugins:**
- **knowledge-base**: AI-driven knowledge entry generation and management system
- **toolkit**: Context building and formalization utilities

## Plugin Architecture

### Core Components

Each plugin follows a standardized structure with four optional component types:

1. **Commands** (`commands/*.md`): Slash commands invoked by users (e.g., `/knowledge-add`)
   - YAML frontmatter with `name`, `description`, `argument-hint`
   - Markdown documentation with implementation details
   - Often delegate to agents for complex workflows

2. **Agents** (`agents/*.md`): Autonomous subprocesses with specialized capabilities
   - YAML frontmatter defines `name`, `description`, `tools`, `model`, `memory`, `skills`
   - System prompt in markdown body defines behavior and workflows
   - Can invoke skills and use project-level memory persistence

3. **Skills** (`skills/<skill-name>/SKILL.md`): Reusable capabilities invoked by agents or commands
   - YAML frontmatter with `name`, `description`, `allowed-tools`
   - Detailed instructions for processing and output generation
   - Support progressive disclosure with additional files (`template.md`, `examples.md`)

4. **Hooks** (`hooks/hooks.json`): Event-driven automation triggered by tool use or lifecycle events
   - PostToolUse, PreToolUse, Stop, SessionStart, etc.
   - File pattern matching with conditions
   - Command execution or prompt injection

### Plugin Manifest

Each plugin has `.claude-plugin/plugin.json` with metadata:
```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "...",
  "author": { "name": "..." },
  "keywords": [...]
}
```

### MCP Server Integration

Plugins can integrate Model Context Protocol servers via `.mcp.json` files at plugin or repository root. This repository uses:
- `chakra-ui-v3`: Chakra UI documentation and migration guidance
- `sequential-thinking`: Multi-step reasoning capabilities
- `context7`: Library documentation lookup

## Working with Plugins

### Plugin Installation Paths

Plugins can be installed at multiple levels:
- **Project-level**: `.claude/plugins/<plugin-name>/` (recommended for project-specific plugins)
- **Global**: `~/.claude/plugins/<plugin-name>/` (for universal utilities)
- **Marketplace**: Via `/plugin marketplace add whenessel/claude-code-plugins`

### Testing Plugins Locally

Test plugin changes by running Claude Code with the plugin directory:
```bash
claude --plugin-dir ./plugins/<plugin-name>
```

Or copy to `.claude/plugins/` for testing in a specific project context.

### Plugin Component Auto-Discovery

Claude Code automatically discovers:
- All `.md` files in `commands/` as commands
- All `.md` files in `agents/` as agents
- All directories in `skills/` with `SKILL.md` as skills
- `hooks/hooks.json` for event hooks

No additional registration needed beyond placing files in correct locations.

## Knowledge Base Plugin Architecture

### Workflow Pipeline

The knowledge-base plugin implements a sophisticated multi-stage pipeline:

1. **User Input** → `/knowledge-add` command (with argument parsing)
2. **Command Layer** → Detects file/URL/directory sources, extracts explicit scope/category
3. **Agent Orchestration** → `knowledge-writer` agent coordinates the workflow
4. **Skill Invocation** → `knowledge-draft` skill transforms guidelines into structured entries
5. **Research Phase** (conditional):
   - Codebase analysis via Grep/Glob when triggered by keywords
   - Web research when "best practices" mentioned or input quality is low
6. **Structure Generation** → YAML frontmatter + markdown with Format section
7. **Validation & Evaluation** → `knowledge-evaluate` skill provides quality scores
8. **File Writing** → Outputs to `.kb/{scope}/{category}/{name}.md`
9. **Hook Notification** → PostToolUse hook reminds to run `/knowledge-reindex`

### Key Patterns

**Multi-language Support**: Detects Russian/English input via Cyrillic characters, always outputs English

**Versioning Strategy**: Semantic versioning (major.minor) with auto-increment logic
- Minor updates: append/modify rules without confirmation
- Major updates: require user confirmation via AskUserQuestion

**Batch Processing**: Supports directory input with per-file success/failure tracking

**Memory Persistence**: Agents use `memory: project` to maintain context across sessions

**Conditional Research**: Trigger-based analysis (keywords, quality threshold) rather than always-on

## Toolkit Plugin Architecture

### Context Building Pattern

The toolkit plugin follows an analyzer-only pattern:
- Agents produce documentation, never modify code
- Skills focus on structured output generation
- Memory stores architectural patterns discovered in prior analysis

**Agent Routing**: `context-builder` agent routes to different skills based on input:
- `build-context`: General project overview or scope-based analysis
- `build-task-context`: Task-specific file identification

## Common Development Patterns

### Skill Progressive Disclosure

Skills use multiple files to avoid overwhelming prompts:
- `SKILL.md`: Core instructions (always loaded)
- `template.md`: Output format templates (referenced when needed)
- `examples.md`: Usage examples (for clarification)

### Hook File Pattern Matching

Hooks use JSON configuration with precise file patterns:
```json
{
  "matcher": "Write|Edit",
  "condition": {
    "filePattern": ".kb/**/*.md",
    "excludePatterns": [".kb/README.md", ...]
  }
}
```

### Agent-Skill Delegation

Commands delegate complex logic to agents, agents invoke skills:
- **Command**: Argument parsing, input validation, user interaction
- **Agent**: Workflow orchestration, decision-making, memory management
- **Skill**: Focused transformation/generation logic

### Explicit vs Auto-Detected Parameters

Pattern seen in knowledge-base: Allow users to override auto-detection with explicit parameters:
- Auto-detect scope/category from content analysis
- Support explicit `scope:X category:Y` overrides in command args
- Use confidence thresholds (70%) to trigger AskUserQuestion when ambiguous

## Repository Conventions

### File Organization
- Each plugin is self-contained in `plugins/<plugin-name>/`
- MCP servers configured at `.mcp.json` (root) or plugin-level
- Documentation: README.md (overview), USAGE.md (examples), TESTING.md (test cases)

### Documentation Standards
- All components use YAML frontmatter for metadata
- Markdown body for detailed instructions
- Code examples in language-specific blocks
- Version tracking in CHANGELOG.md

### Git Workflow
- MIT License (see LICENSE file)
- Author: Artem Demidenko
- Standard gitignore for Node/Python/IDE files
- Keep `.claude-plugin/`, `plugin.json`, `*.md`, `hooks.json`, `.mcp.json` in version control

## MCP Server Usage

When working with MCP-enabled features:
- **chakra-ui-v3**: Use for React UI component examples, theme customization, migration scenarios
- **context7**: Use to fetch up-to-date library documentation (requires library ID resolution first)
- **sequential-thinking**: Use for complex multi-step reasoning tasks

MCP servers are auto-configured and available to all plugins in the workspace.

## Development Workflow

When creating or modifying plugins:

1. **Structure First**: Set up `.claude-plugin/plugin.json` with proper metadata
2. **Component Design**: Decide which component types are needed (commands/agents/skills/hooks)
3. **Progressive Implementation**: Start with core skill logic, then add agent orchestration, finally wrap with user-facing commands
4. **Testing**: Use local plugin directory or copy to `.claude/plugins/` for testing
5. **Documentation**: Update README.md with usage examples and architecture notes
6. **Validation**: Test with various inputs, verify auto-discovery, check hook triggers

When modifying existing plugins:
1. Check component dependencies (agents → skills, commands → agents)
2. Update version in `plugin.json` if making breaking changes
3. Test entire workflow from command invocation to final output
4. Update CHANGELOG.md with changes