# Changelog

All notable changes to the **toolkit** plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-02-15

### Added

#### Core Features
- **Automatic Task Formalization**: UserPromptSubmit hook that transparently analyzes user requests and structures vague/complex inputs
- **Confidence-based Routing**: 0.0-1.0 scoring system determines when to auto-invoke formalizer agent (threshold: 0.6)
- **Context Building**: Automated codebase analysis with architectural layer mapping, API extraction, and dependency graphs
- **Memory Persistence**: Project-level memory for both agents to cache patterns and improve over time

#### Commands
- `/formalize` - Explicit task formalization command for manual control
- `/build-context` - Full project or module-scoped architecture analysis
- `/task-context` - Focused context generation for specific tasks/features

#### Agents
- **formalizer** - Task normalization engine with confidence scoring and dependency detection
- **context-builder** - Codebase analyzer with language-agnostic discovery and memory caching

#### Skills
- **formalize** - Multi-step formalization pipeline with parse/decompose/structure/score/output phases
- **build-context** - Architectural analysis with 3-depth modes (shallow/normal/deep)
- **build-task-context** - Task-scoped file and API identification

#### Hooks
- **UserPromptSubmit** - Automatic quality evaluation and formalization trigger
- Multi-factor confidence scoring (goal clarity, requirements, scope, dependencies, DoD)
- Smart skip logic for questions, atomic actions, and explicit "no planning" requests

#### Documentation
- Comprehensive README with examples, use cases, troubleshooting
- Command documentation with argument parsing and error handling
- CLAUDE-md-snippet for optional manual integration
- Real-world formalization examples

### Implementation Details

#### Formalization Pipeline
- Parses multi-objective requests into ordered tasks
- Identifies implicit dependencies and creates dependency graphs
- Scores confidence based on 5 factors (clarity, requirements, scope, terminology, dependencies)
- Routes output mode: silent (≥0.8), summary (0.5-0.79), clarify (<0.5)
- Marks assumptions with [ASSUMED] and inferences with [INFERRED]

#### Context Analysis
- Language-agnostic discovery strategy (works with TypeScript, Python, Rust, Go, etc.)
- Architectural layer classification (core, infrastructure, adapters, utilities, config, tests)
- Public API surface extraction with signatures and line numbers
- Cross-module dependency mapping with circular dependency detection
- Auto-detects project type (monorepo, single-package, workspace)

#### Memory Management
- Agents cache architectural patterns, conventions, gotchas
- Memory stored in `.claude/memory/toolkit/{agent-name}/`
- Incremental updates to avoid full re-analysis
- Session continuity across conversations

#### Output Formats
- Context documents: `.claude/context/{scope}.md`
- Task contexts: Auto-named by branch/issue ID
- Formalized tasks: Structured markdown with objectives, requirements, DoD, dependencies

### Technical Specifications

#### Hook Configuration
```json
{
  "UserPromptSubmit": [
    {
      "type": "prompt",
      "prompt": "Quality evaluation with confidence scoring..."
    }
  ]
}
```

#### Agent Configuration
- **formalizer**: Tools: Read, Grep, Glob | Model: sonnet | Memory: project | Skills: formalize
- **context-builder**: Tools: Read, Grep, Glob, Bash, Write | Model: sonnet | Memory: project | Skills: build-context, build-task-context

#### Skill Features
- `context: fork` for isolated execution environments
- `disable-model-invocation: true` for deterministic processing
- Progressive disclosure with separate files (SKILL.md, examples.md)

### Known Limitations
- Large codebases (1000+ files) may require scoped analysis
- Non-standard project structures may need manual guidance
- Context window limits for very large projects
- Formalization accuracy depends on domain-specific terminology clarity

### Dependencies
- None (uses built-in Claude Code tools only)

### Compatibility
- Claude Code CLI 1.0+
- All operating systems (macOS, Linux, Windows)

---

## Future Roadmap

### Planned Features
- [ ] Incremental context updates (track file changes)
- [ ] Integration with `knowledge-base` plugin for convention discovery
- [ ] Customizable confidence thresholds via `.local.md` settings
- [ ] Export context to Markdown/PDF/HTML
- [ ] Multi-language formalization (preserve user's language in output)
- [ ] Git integration for branch-based context caching
- [ ] Visual dependency graph generation (Mermaid diagrams)
- [ ] API change impact analysis

### Under Consideration
- Webhook integration for CI/CD context updates
- LLM-based code similarity analysis for "like the other service" references
- Automatic DoD generation from test files
- Context diff view for architectural changes over time

---

## Migration Guide

### From Manual CLAUDE.md Integration

If you previously used `CLAUDE-md-snippet.md` instructions in your `CLAUDE.md`:

1. Remove the Task Formalization Protocol section from `CLAUDE.md`
2. Install toolkit plugin (hooks now handle formalization automatically)
3. Verify hook is active: check `.claude/plugins/toolkit/hooks/hooks.json`
4. Test with a vague request to confirm auto-formalization works

The hook-based approach is more reliable and doesn't require maintaining custom instructions.

---

## Contributors

- **Artem Demidenko** - Initial release and core architecture

## License

MIT License - see [LICENSE](../../LICENSE) file
