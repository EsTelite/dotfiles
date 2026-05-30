# Creating a Basic Kiro Agent with Markdown & Skills

This guide walks through creating a custom Kiro agent with markdown support and skills integration, based on the `python-dev-expert` example.

## Agent File Structure

Store agent configurations in `~/.kiro/agents/` (global) or `.kiro/agents/` (workspace).

```
.kiro/agents/
└── my-agent.json
```

## Basic Agent Configuration

```json
{
  "name": "my-agent",
  "description": "Brief description of what this agent does",
  "prompt": "Detailed instructions for the agent's behavior and capabilities",
  "allowedTools": ["fs_read", "fs_write", "execute_bash"],
  "toolsSettings": {
    "execute_bash": {
      "autoAllowReadonly": true
    }
  },
  "resources": [
    "file://README.md",
    "skill://.kiro/skills/*/SKILL.md",
    "skill://~/.kiro/skills/*/SKILL.md"
  ],
  "welcomeMessage": "Custom greeting message"
}
```

## Core Fields

| Field | Purpose |
|-------|---------|
| `name` | Unique agent identifier (lowercase, hyphens allowed) |
| `description` | What the agent does |
| `prompt` | Detailed behavioral instructions (supports markdown) |
| `allowedTools` | Array of tools the agent can use without prompting |
| `toolsSettings` | Per-tool configuration (e.g., auto-allow readonly bash) |
| `resources` | File and skill paths using `file://` and `skill://` URIs |
| `welcomeMessage` | Initial message when agent starts |

## Tools Reference

Available tools for `allowedTools`:
- `fs_read` - Read files
- `fs_write` - Write/create files
- `execute_bash` - Run shell commands
- `code` - Code intelligence (search symbols, AST parsing)
- `grep` - Text pattern search
- `glob` - File pattern matching
- `web_fetch` - Fetch URL content
- `web_search` - Search the web
- `use_aws` - AWS CLI operations
- `query-docs` - Query documentation
- `knowledge` - Knowledge base operations

## Resources

### File Resources
Load project files and documentation:
```json
"resources": [
  "file://README.md",
  "file://.amazonq/rules/**/*.md",
  "file://src/architecture.md"
]
```

### Skill Resources
Enable skills from workspace and global locations:
```json
"resources": [
  "skill://.kiro/skills/*/SKILL.md",
  "skill://~/.kiro/skills/*/SKILL.md"
]
```

Glob patterns (`*`) match all skill folders. Specific paths also work:
```json
"skill://.kiro/skills/pr-review/SKILL.md"
```

## Example: Python Developer Agent

```json
{
  "name": "python-dev",
  "description": "Expert Python developer for building and debugging Python projects",
  "prompt": "You are an expert Python developer. Write clean, minimal code. Use `uv` for package management and work within virtual environments. When installing packages, use `uv pip install`. When creating environments, use `uv venv`.",
  "allowedTools": ["fs_read", "fs_write", "execute_bash", "code"],
  "toolsSettings": {
    "execute_bash": {
      "autoAllowReadonly": true
    }
  },
  "resources": [
    "file://README.md",
    "file://requirements.txt",
    "skill://.kiro/skills/*/SKILL.md",
    "skill://~/.kiro/skills/*/SKILL.md"
  ],
  "welcomeMessage": "Ready for Python development. What's the task?"
}
```

## Using Your Agent

### Start a Session
```bash
kiro chat --agent my-agent
```

### Invoke Skills
Skills activate automatically when matched to your request, or manually:
```
> /skill-name arguments here
```

View available skills:
```
> /context show
```

## Best Practices

1. **Specific Prompts** - Write detailed behavioral instructions in the `prompt` field using markdown
2. **Minimal Tools** - Only add tools to `allowedTools` that the agent needs
3. **Smart Resources** - Include documentation and project files relevant to the agent's workflow
4. **Glob Patterns** - Use `*/` to load all skills and adapt as new ones are added
5. **Version Control** - Commit workspace agents (`.kiro/agents/`) to share with your team

## Common Patterns

### Research Agent
```json
{
  "allowedTools": ["fs_read", "web_fetch", "web_search", "grep"],
  "resources": ["file://docs/**/*.md", "skill://~/.kiro/skills/*/SKILL.md"]
}
```

### Infrastructure Agent
```json
{
  "allowedTools": ["fs_read", "fs_write", "execute_bash", "use_aws"],
  "resources": ["file://terraform/**/*.tf", "skill://.kiro/skills/*/SKILL.md"]
}
```

### Code Review Agent
```json
{
  "allowedTools": ["fs_read", "code", "web_fetch"],
  "resources": ["file://.kiro/skills/pr-review/SKILL.md"]
}
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Agent not found | Verify JSON is in `~/.kiro/agents/` or `.kiro/agents/` with `.json` extension |
| Invalid JSON | Use a JSON validator; check commas and quotes |
| Skills not loading | Confirm `skill://` URIs in `resources` field; verify SKILL.md exists in skill folder |
| Tool not working | Check if tool is in `allowedTools` array |
