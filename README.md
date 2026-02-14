# Claude Code Plugins

Custom Claude Code plugins marketplace with skills, commands, and agents for AI-assisted development.

## Installation

Add this marketplace to Claude Code:
```bash
/plugin marketplace add whenessel/claude-code-plugins
```

## Available Plugins

Coming soon...

## Usage

Install a plugin from this marketplace:
```bash
/plugin install plugin-name@claude-code-plugins
```

## Plugin Structure

Each plugin follows the standard Claude Code structure:
```
plugin-name/
├── .claude-plugin/
│   └── plugin.json
├── skills/          # Agent Skills (optional)
├── commands/        # Slash commands (optional)
├── agents/          # Agents (optional)
└── hooks/           # Event hooks (optional)
```

## Development

To test plugins locally:
```bash
claude --plugin-dir ./plugin-name
```

## License

MIT
